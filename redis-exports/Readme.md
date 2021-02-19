## 参考网站
https://www.kubernetes.org.cn/7073.html
https://www.cnblogs.com/Dev0ps/p/11315713.html


# redis高可用哨兵模式
## 参考连接
  https://blog.csdn.net/jesse919/article/details/102605178
  https://www.cnblogs.com/Dev0ps/p/11259401.html

## 添加仓库并下载chart
  [root@k8s-master1]# helm repo add mirror-stable http://mirror.azure.cn/kubernetes/charts/
  [root@k8s-master1]# helm fetch mirror-stable/redis-ha --version 3.8.0 --untar --untardir ./
## 修改value.yaml
  [root@k8s-master1]# vi value.yaml

    auth
    redisPassword
    storageClass
    修改 “hardAntiAffinity: true” 为 “hardAntiAffinity: false” (仅限当replicas > worker node 节点数时修改)
## 修改redis监控及自动发现相关配置
  [root@k8s-master1]# vi value.yaml

      exporter:   # 开启监控
    enabled: true

  [root@k8s-master1]# vi templates/redis-ha-announce-service.yaml  

    annotations下添加：prometheus.io/probe: "true" 
    修改port_name：redis-exporter
## 部署
  [root@k8s-master1]# helm install redis-ha -f values.yaml ./
    
  [root@k8s-master1]# kubectl exec -it redis-ha-server-0  sh

    /data $ redis-cli
    127.0.0.1:6379> auth password

# 集群监控部署步骤
* 添加仓库
  * [root@k8s-master1]# helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
* 下载repo
  * [root@k8s-master1]# helm fetch prometheus-community/prometheus-redis-exporter
* 配置values.yaml
```
rbac:
  create: true
  pspEnabled: true
serviceAccount:
  create: true
  name:
replicaCount: 1
image:
  repository: oliver006/redis_exporter
  tag: v1.11.1
  pullPolicy: IfNotPresent
extraArgs: {}
env: 
  - name: REDIS_PASSWORD    ###---------replace---------###
    value: pass    ###---------replace---------###
service:
  type: ClusterIP
  port: 9121
  annotations: 
    prometheus.io/scrape: "true"    ###---------replace---------###
  labels: {}
resources: {}
nodeSelector: {}
tolerations: []
affinity: {}
redisAddress: 
  - redis://redis-cluster-0.default.redis-cluster.svc.cluster.local:6379   ###---------replace---------###
  - redis://redis-cluster-1.default.redis-cluster.svc.cluster.local:6379   ###---------replace---------###
  - redis://redis-cluster-2.default.redis-cluster.svc.cluster.local:6379   ###---------replace---------###
  - redis://redis-cluster-3.default.redis-cluster.svc.cluster.local:6379   ###---------replace---------###
  - redis://redis-cluster-4.default.redis-cluster.svc.cluster.local:6379   ###---------replace---------###
  - redis://redis-cluster-5.default.redis-cluster.svc.cluster.local:6379   ###---------replace---------###
 # - redis://pod_name.namespace_name.svc_name.svc.cluster.local:6379
annotations: 
  prometheus.io/path: /metrics   ###---------replace---------###
  prometheus.io/port: "9121"    ###---------replace---------###
  prometheus.io/scrape: "true"    ###---------replace---------###
serviceMonitor:
  enabled: true    ###---------replace---------###
  namespace: devops    ###---------replace---------###
  interval: 30s    ###---------replace---------###
  telemetryPath: /metrics    ###---------replace---------###
  labels:    ###---------replace---------###
  timeout: 10s    ###---------replace---------###
prometheusRule:
  enabled: false
  additionalLabels: {}
  namespace: ""
  rules: []
auth:
  enabled: false
  secret:
    name: ""
    key: ""
  redisPassword: "" 
```
* 部署
  [root@k8s-master1]# helm install redis-exporter -f values.yaml ./
# Prometheus自动发现配置
    ##（需在svc中配置annotations: prometheus.io/probe: "true"而且port_name改为redis-exporter ）

    ######### redis 自动发现配置 ################
    - job_name: 'kube_svc_redis-exporter'
      kubernetes_sd_configs:
      - role: service
      metrics_path: /metrics
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
        # 匹配svc.metadata.annotation字段包含key/value(prometheus.io/probe: "true")
      - source_labels: [__meta_kubernetes_service_port_name]
        action: keep
        regex: "redis-exporter"
        # 匹配svc.spec.ports.name 是否包含redis-exporter
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name