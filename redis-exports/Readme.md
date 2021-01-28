## 参考网站
https://www.kubernetes.org.cn/7073.html
https://www.cnblogs.com/Dev0ps/p/11315713.html

## 部署步骤
* 添加仓库
  * helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
* 下载repo
  * helm fetch prometheus-community/prometheus-redis-exporter
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
env: {}
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