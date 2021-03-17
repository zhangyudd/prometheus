# prometheus

## prometheus 相关exporter
K8S 生态的组件都会提供/metric接口以提供自监控，这里列下我们正在使用的：

    cadvisor: 集成在 Kubelet 中。
    kubelet: 10255为非认证端口，10250为认证端口。
    apiserver: 6443端口，关心请求数、延迟等。
    scheduler: 10251端口。
    controller-manager: 10252端口。
    etcd: 如etcd 写入读取延迟、存储容量等。
    docker: 需要开启 experimental 实验特性，配置 metrics-addr，如容器创建耗时等指标。
    kube-proxy: 默认 127 暴露，10249端口。外部采集时可以修改为 0.0.0.0 监听，会暴露：写入 iptables 规则的耗时等指标。
    kube-state-metrics: K8S 官方项目，采集pod、deployment等资源的元信息。
    node-exporter: Prometheus 官方项目，采集机器指标如 CPU、内存、磁盘。
    blackbox_exporter: Prometheus 官方项目，网络探测，dns、ping、http监控
    process-exporter: 采集进程指标
    nvidia exporter: 我们有 gpu 任务，需要 gpu 数据监控
    node-problem-detector: 即 npd，准确的说不是 exporter，但也会监测机器状态，上报节点异常打 taint
    应用层 exporter: mysql、nginx、mq等，看业务需求。

`四个黄金信号`：延迟、流量、错误数、饱和度


## Prometheus 采集外部 K8S 集群、多集群
Prometheus 如果部署在K8S集群内采集是很方便的，用官方给的Yaml就可以，但我们因为权限和网络需要部署在集群外，二进制运行，采集多个 K8S 集群。

以 Pod 方式运行在集群内是不需要证书的（In-Cluster 模式），但集群外需要声明 token之类的证书，并替换address，即使用 Apiserver Proxy采集，以 Cadvisor采集为例，Job 配置为：

```
- job_name: cluster-cadvisor
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: https
  kubernetes_sd_configs:
  - api_server: https://xx:6443
    role: node
    bearer_token_file: token/cluster.token
    tls_config:
      insecure_skip_verify: true
  bearer_token_file: token/cluster.token
  tls_config:
    insecure_skip_verify: true
  relabel_configs:
  - separator: ;
    regex: __meta_kubernetes_node_label_(.+)
    replacement: $1
    action: labelmap
  - separator: ;
    regex: (.*)
    target_label: __address__
    replacement: xx:6443
    action: replace
  - source_labels: [__meta_kubernetes_node_name]
    separator: ;
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    action: replace
  metric_relabel_configs:
  - source_labels: [container]
    separator: ;
    regex: (.+)
    target_label: container_name
    replacement: $1
    action: replace
  - source_labels: [pod]
    separator: ;
    regex: (.+)
    target_label: pod_name
    replacement: $1
    action: replace
```

bearer_token_file 需要提前生成，这个参考官方文档即可。记得 base64 解码。

对于 cadvisor 来说，__metrics_path__可以转换为/api/v1/nodes/{1}/proxy/metrics/cadvisor，代表Apiserver proxy 到 Kubelet，如果网络能通，其实也可以直接把 Kubelet 的10255作为 target，可以直接写为：{1}:10255/metrics/cadvisor，代表直接请求Kubelet，规模大的时候还减轻了 Apiserver 的压力，即服务发现使用 Apiserver，采集不走 Apiserver

因为 cadvisor 是暴露主机端口，配置相对简单，如果是 kube-state-metric 这种 Deployment，以 endpoint 形式暴露，写法应该是：
```
- job_name: cluster-service-endpoints
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: https
  kubernetes_sd_configs:
  - api_server: https://xxx:6443
    role: endpoints
    bearer_token_file: token/cluster.token
    tls_config:
      insecure_skip_verify: true
  bearer_token_file: token/cluster.token
  tls_config:
    insecure_skip_verify: true
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    separator: ;
    regex: "true"
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
    separator: ;
    regex: (https?)
    target_label: __scheme__
    replacement: $1
    action: replace
  - separator: ;
    regex: (.*)
    target_label: __address__
    replacement: xxx:6443
    action: replace
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_endpoints_name,
      __meta_kubernetes_service_annotation_prometheus_io_port]
    separator: ;
    regex: (.+);(.+);(.*)
    target_label: __metrics_path__
    replacement: /api/v1/namespaces/${1}/services/${2}:${3}/proxy/metrics
    action: replace
  - separator: ;
    regex: __meta_kubernetes_service_label_(.+)
    replacement: $1
    action: labelmap
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: kubernetes_namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: kubernetes_name
    replacement: $1
    action: replace
```
对于 endpoint 类型，需要转换__metrics_path__为/api/v1/namespaces/{1}/services/{2}:${3}/proxy/metrics，需要替换 namespace、svc 名称端口等，这里的写法只适合接口为/metrics的exporter，如果你的 exporter 不是/metrics接口，需要替换这个路径。或者像我们一样统一约束都使用这个地址。

这里的__meta_kubernetes_service_annotation_prometheus_io_port来源就是 exporter 部署时写的那个 annotation，大多数文章中只提到prometheus.io/scrape: ‘true’，但也可以定义端口、路径、协议。以方便在采集时做替换处理。

其他的一些 relabel 如kubernetes_namespace 是为了保留原始信息，方便做 promql 查询时的筛选条件。

如果是多集群，同样的配置多写几遍就可以了，一般一个集群可以配置三类job：

* role:node 的，包括 cadvisor、 node-exporter、kubelet 的 summary、kube-proxy、docker 等指标

* role:endpoint 的，包括 kube-state-metric 以及其他自定义 Exporter

* 普通采集：包括Etcd、Apiserver 性能指标、进程指标等。
