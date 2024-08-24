---
title: 基于自定义指标的HPA实战
date: 2021-03-10 23:50:47    
tags:    
 - Kubernetes    
categories:    
 - Kubernetes    
toc: true
---
### 概念

Kubernetes自1.11版本开始引入了名为"HorizontalPodAutoscaler"的控制器用于完成Pod基于CPU使用率进行水平扩展。所谓水平扩展是在Pod中CPU的使用率达到设定的某个值时，HPA告知Pod对应的高层控制器(如Deployment、RS等控制器)。高层控制器随即创建出多个pod副本平衡负载使Pod的平均CPU使用率低于我们设定的值。当Pod的平均CPU使用率变小时，又能回收多余的Pod以避免资源浪费。
<!--more-->

![image-20210313152925460](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313152925460.png)

除了CPU使用率这个指标外，还可以采用自定义的指标。通常自定义的指标需要容器以某种方式提供，如URL路径"/metrics"提供，即http://[podIP]:[podPort]/metrics获取相应的指标数据。当然如果kubernetes构建在公有云上，亦可以采用公有云服务商提供的指标数据如负载均衡器的QPS等作为HPA控制器的指标来源。

### HPA控制器配置

目前HPA(HorizontalPodAutoscaler)资源对象处于Kubernetes的API组"autoscaling"中，包括v1、v2beta1及v2beta2三个API版本，其中"autoscaling/v1"仅支持基于CPU的自动扩缩容，而"autoscaling/v2"则支持基于自定义指标类型的数据。
为了使用HPA需要实现部署Metrics-Server。项目地址https://github.com/kubernetes-sigs/metrics-server，由于我本地环境是用[Operator部署的Prometheus](https://github.com/prometheus-operator/kube-prometheus)，部署完自带`metrics-server`,所以不再手动部署。

![image-20210313161330100](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313161330100.png)

部署完成metrics-server后稍等一会后就可以使用`kubectl top node/pod`命令查看节点或者pod的CPU及内存使用率了。

![image-20210313161538317](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313161538317.png)

(1)基于autoscaling/v1版本的HPA配置，**仅可以设置CPU使用率**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  miniReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

- scaleTargetRef:定义目标作用对象，可以是Deployment、RS
- targetCPUUtilizationPercentage：目标pod的CPU阈值，超过这个值即触发扩容操作
- miniReplicas及maxReplicas：最小及最大副本数量，HPA会自动在这个范围内进行Pod的自动伸缩同时维持各个Pod的CPU使用率为50%

(2)基于autoscaling/v2beta2的HPA配置

#### type=Resource

type=Resource的指标数据由metrics-server采集并提供给HPA控制器

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sample-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu / memory
      target:
        type: Utilization
        averageUtilization: 50  / averageValue: 200MiB
```

其中type字段可为：

- Resource：基于资源的指标值，指标数据可以通过metrics.k8s.io这个API查询。

- Pods：基于Pod的指标

- Object：基于某种资源对象,如ingress或者任意自定义的指标
  Pods与Object是自定义指标，需要搭建自定义的Metrics Server和监控工具进行指标采集。指标数据通过custom.metrics.k8s.io进行查询，必须先启动自定义的Metrics Server。

#### type=pods(本次采用的类型)

类型为Pods的类型指标数据来源于Pod对象本身，target指标只能是**AverageValue**。

```yaml
  metrics:
  - type: Pods
    pods:
      metrics:
        name: packet-per-second
      target:
        type: AverageValue
        averageValue: 1k
```



#### type=object

type=object的指标数据来自其他资源或者是自定义的指标。target指标为**Value**或者是**AverageValue**

```yaml
#指标名称为requests-per-second，其值来源于Ingress"main-route"的目标值，即Ingress每秒请求量达到2000时触发扩容操作
    metrics：
    - type: Object
      object:
        metric:
          name: requests-per-second
        describeObject:
          apiVersion: extensions/v1beta1
          kind: Ingress
          name: main-route
        target:
          type: Value
          value: 2k
#指标名称为requests-per-second，其值来源于Ingress"main-route"的目标值，即Ingress每秒请求量达到2000时触发扩容操作
    metrics：
    - type: Object
      object:
        metric:
          name: 'http_requests'
          selector: 'verb=GET'
        target:
          type: AverageValue
          averageValue: 500
```

### 基于自定义指标的HPA实战

#### 整体概述

![image-20210313164205820](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313164205820.png)

按照《Kubernetes权威指南(第四版)》所述的架构图，由[Prometheus](https://www.prometheus.io)采集各个pod的指标通过prometheus-adapter这个适配器将数据提交给指标聚合器，并最终注册到api-server中，以`/apis/custom.metrics.k8s.io`的路径提供指标数据。HPA控制器从该路径下获取数据作为水平自动扩缩容的依据。

#### 配置controller-manager

一般而言通过kubeadm部署的k8s集群不需要进行api-server的手动配置，所需要的配置已经默认开启。我们只需要配置kube-controller-manager的参数即可。

![image-20210313164735322](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313164735322.png)

编辑`/etc/kubernetes/manifests/kube-controller-manager.yaml`添加上图所示的启动参数。为了加快试验的效果，这里缩短了默认的配置。生产环境推荐以官方默认配置为准。

#### 创建对象pod

该pod会在/metrics路径下提供名为http_requests_total的指标。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - image: luxas/autoscale-demo:v0.1.2
        name: metrics-provider
        ports:
        - name: http
          containerPort: 8080
```

![image-20210313165453521](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313165453521.png)

#### 创建ServiceMonitor对象，用于监控程序提供的指标

```yaml
$ cat service-monitor.yaml 
kind: ServiceMonitor
apiVersion: monitoring.coreos.com/v1
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints: 
  - port: http
```

**注意**:这里的port:http一定要和上面部署的Service对象中port的名称一致。应用该对象后稍等2~3分钟打开promethes的页面，即可发现相应的监控项已经是up状态：

![image-20210313165930266](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313165930266.png)

#### 配置Adapter

以上配置的指标作http_requests_total为一种持续增长的值，不能反映单位时间内的增长量，不能用作HPA的依据。需要进行适当地转换，打开adapter的配置，添加如下配置并应用：

![image-20210313170429702](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313170429702.png)

```yaml
config.yaml: |
    rules:
    #sum(rate(http_requests_total{namespace="xx",pod="xx"}[1m])) by pod:1分钟内全部pod指标http_requests_total的总和的每秒平均值
    - metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)                                       
      #将metricsQuery计算的结果赋给新的指标`http_requests`并提供给HPA控制器
      name:
        as: "${1}"
        matches: ^(.*)_total$
      resources:
        template: <<.Resource>>
      seriesFilters:
      - isNot: .*_seconds_total$
      seriesQuery: '{namespace!="",__name__=~"http_requests_.*"}'
```

应用configmap完成后删除adapter Pod，使新的配置生效。

#### 自定义API资源

创建名为v1beta1.custom.metrics.k8s.io的自定义聚合API资源

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  service:
    name: prometheus-adapter
    namespace: monitoring
  group: custom.metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

#### 创建HPA控制器资源

在以上资源创建完成后即可以创建HPA控制器了

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta2
metadata:
  name: sample-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests
      target:
        type: AverageValue
        averageValue: 500m
```

- type=pods:表示从Pods自身获取指标
- name=http_requests：即adapter中的配置，由http_requests_total计算而来
- averageValue=500m：即http_requests的目标值为500m。

通过聚合API查询指标数据,pod指标成功采集

![image-20210313172852633](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313172852633.png)

![image-20210313172954061](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313172954061.png)

#### 验证动态扩缩容

在其他终端请求sample-app的地址进行压测：

```bash
$ for i in {1..100000};do wget -q -O- 10.233.38.160 > /dev/null;done
```

![image-20210313173159598](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313173159598.png)

自动扩容成功

停止访问请求一分钟后(`kube-controller-manager`中设定的参数`--horizontal-pod-autoscaler-downscale-stabilization=1min`进行缩容操作)pod数量成功回落到1个

![image-20210313173612060](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210313173612060.png)

### 参考

- 《Kubernetes权威指南(第四版)》（p241-p264）

