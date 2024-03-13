---
title: Prometheus-Operator监控Nginx Ingress
date: 2021-03-07 11:50:47    
tags:    
 - Kubernetes  
 - Monitoring
categories:    
 - Kubernetes
 - Monitoring
toc: true
disableToC: false
disableAutoCollapse: true
---

**前言**：最近客户需要监控ingress流量，在查阅资料后成功部署，记录下部署的过程及遇到的问题。
<!--more-->



### 暴露ingress的监控端口

默认情况下nginx-ingress的监控指标端口为10254，监控路径为其下的/metrics。调整配置ingress-nginx的配置文件，打开service及pod的10254端口。

在官方的部署文件中的`Service`段中添加一段监听10254端口的配置，命名为`https-metrics`

```yaml
spec:
  type: ClusterIP
  ports:
    - name: https-webhook
      port: 443
      targetPort: webhook
    - name: https-metrics
      port: 10254
      targetPort: 10254
```

同时在`deployment`段中开放pod的10254端口,命名为`metrics`

```yaml
ports:
  - name: http
    containerPort: 80
    protocol: TCP 
  - name: https
    containerPort: 443 
    protocol: TCP 
  - name: webhook
    containerPort: 8443
    protocol: TCP 
  - name: metrics
    containerPort: 10254
    protocol: TCP
```



### 静态抓取

在`prometheus-prometheus.yaml`的配置文件中，官方内置了一个字段`additionalScrapeConfigs`用于添加自定义的抓取目标

https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/api.md#PrometheusSpec

所以只需要按照官方的要求写入配置即可。新建额外抓取的信息`prometheus-additional.yaml`：

```yaml
- job_name: nginx-ingress
  metrics_path: /metrics
  scrape_interval: 5s
  static_configs:
  - targets:
    - 172.16.200.102:10254
    - 172.16.200.103:10254
    - 172.16.200.104:10254
```

抓取的目标制定为targets列表，创建secret对象`ingress-nginx-additional-configs`引用该配置

```bash
$ kubectl create secret generic ingress-nginx-additional-configs --from-file=./prometheus-additional.yaml -n monitoring
```

在`promethes-prometheus.yaml`中添加`additionalScrapeConfigs`引用创建的secret

```yaml
serviceAccountName: prometheus-k8s
   serviceMonitorNamespaceSelector: {}
   serviceMonitorSelector: {}
   version: v2.11.0
   additionalScrapeConfigs:
     name: ingress-nginx-additional-configs
     key: prometheus-additional.yaml
```

重建promethes配置:

```bash
$ kubectl delete -f ./prometheus-prometheus.yaml && kubectl apply -f ./prometheus-prometheus.yaml
```

打开promethes的UI页面，访问/targets路径发现相关的节点的状态已经为up状态

![image-20210309161147110](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210309161147110.png)

在grafana中导入`9614`号监控模板,数据显示正常：

![prometheus-ingress](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/prometheus-ingress.png)

静态配置的方式比较固定，不能针对主机列表动态变化。新增加边缘节点需要重建secret`ingress-nginx-additional-configs`,十分不便。



### servicemonitors抓取

针对使用Operator模式部署的prometheus，有一个名为`servicemonitors.monitoring.coreos.com`的CRD资源，可以仿照这个资源中某个项进行ingress-nginx的数据抓取

```bash
[root@master-1 ~]# kubectl -n monitoring get servicemonitors.monitoring.coreos.com 
NAME                      AGE
alertmanager              4d22h
coredns                   4d22h
grafana                   4d22h
kube-apiserver            4d22h
kube-controller-manager   4d22h
kube-scheduler            4d22h
kube-state-metrics        4d22h
kubelet                   4d22h
node-exporter             4d22h
prometheus                4d22h
prometheus-operator       4d22h
```

定义ingress-nginx的配置文件：

```yaml
$ cat ingress-nginx.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress
  namespace: monitoring
  labels:
    app.kubernetes.io/component: controller
spec:
  jobLabel: app.kubernetes.io/component
  endpoints:
  - port: https-metrics                        #之前定义的ingress的监控端口，一定要用名称。不能使用数字
    interval: 10s 
  selector:
    matchLabels:
      app.kubernetes.io/component: controller  #该标签匹配ingress-nginx-controller的pod
  namespaceSelector:
    matchNames:
    - ingress-nginx                            #指定ingress控制器的名称空间为ingress-nginx
```

创建该资源

```
$ kubectl apply -f ./ingress-nginx.yaml
```

打开prometheues的监控面板下的/targets,相关资源已经生成：

![image-20210309170151240](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210309170151240.png)

参考静态抓取的方式导入监控面板即可。



### 参考文档

- https://www.amd5.cn/atang_4421.html
- https://www.cnblogs.com/lvcisco/p/12574532.html
- https://prometheus.io/docs/prometheus/latest/configuration/configuration
