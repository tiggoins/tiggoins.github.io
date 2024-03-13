---
title: 初试Pod垂直扩缩容VPA
date: 2021-03-11 23:50:47    
tags:    
 - Kubernetes
categories:    
 - Kubernetes   
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/LongIsland_ZH-CN7089248815_1920x1080.jpg       
toc: true
disableAutoCollapse: false
---
### 概念

相对于水平自动扩缩容(HPA)在pod资源紧张时扩充pod个数来平衡负载。Pod的垂直扩容会*自动*调整Pod资源申请的requests值及limits值，它会依据pod当前运行状况动态地为Pod资源申请CPU及内存使用量。解放了手动设置request值及limits值的难点，使Pod运行更加智能。**目前为Beta阶段**
<!--more-->

垂直扩缩容项目地址位于：https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler



### 安装VPA控制器

与HPA一样，VPA在运行时的指标同样是由[Metrics Server](https://github.com/kubernetes-sigs/metrics-server)提供,所以安装VPA控制器前首先先运行好Metrics Server

目前最新版本为0.9，考虑到部署0.9版本需要升级`OpenSSL`。本次实验采用0.8版本的部署包。

#### 下载部署文件

```bash
$ wget https://codeload.github.com/kubernetes/autoscaler/tar.gz/vertical-pod-autoscaler-0.8.0
```

解压后进入vertical-pod-autoscaler目录

```bash
$ cd vertical-pod-autoscaler
```

批量修改镜像地址为registry.cn-shanghai.aliyuncs.com/ltzhang

```bash
$ sed -i 's@k8s.gcr.io/@registry.cn-shanghai.aliyuncs.com/ltzhang/@g' `egrep -r "\<image\>" deploy/ | awk -F: '{print $1}'`
```

#### 安装VPA控制器

```bash
$ ./hack/vpa-up.sh  
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalers.autoscaling.k8s.io created
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalercheckpoints.autoscaling.k8s.io created
clusterrole.rbac.authorization.k8s.io/system:metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:vpa-actor created
...
deployment.apps/vpa-admission-controller created
service/vpa-webhook created
```

安装完成后会生成自定义API资源`autoscaling.k8s.io`并在其下生成两个资源`VerticalPodAutoscalerCheckpoint`及`VerticalPodAutoscaler`

![image-20210314120620126](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210314120620126.png)

#### 创建测试应用

```bash
$ kubectl apply -f examples/hamster.yaml 
verticalpodautoscaler.autoscaling.k8s.io/hamster-vpa created
deployment.apps/hamster created
```

测试pod会申请100m的CPU及50M内存并且其中会运行shell命令不断来消耗CPU及内存：

```yaml
resources:
  requests:
    cpu: 100m
    memory: 50Mi
command: ["/bin/sh"]
args:
  - "-c"
  - "while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done"
```

在pod运行中，VPA控制器会发现应用所需要的CPU及内存不断增加。根据当前的**updatePolicy**会不断重建pod给予Pod更多的CPU及内存申请值

![image-20210314123312300](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210314123312300.png)

Pod中的资源申请已经上调为587m个CPU及256MiB内存了，这都是VPA控制器自动帮我们完成的。在应用负载降低时，同样会删除Pod并给予相对资源申请值。

![image-20210314123815146](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/typora/image-20210314123815146.png)

#### 配置文件解析

回到测试Pod配置文件查看有关VPA的定义

```yaml
---
apiVersion: "autoscaling.k8s.io/v1beta2"
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hamster
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
```

- targetRef：定义了VPA控制器作用的资源名称即相应的类型及API组

- resourcePolicy：定义指定资源(这里是CPU及内存)的计算策略。CPU在100m-1间浮动，内存在50MiB-500MiB之间浮动

- updatePolicy：这个字段可以通过`describe vpa hamster-vpa`看到，用于定义更新策略,目前有以下四种更新策略：

  1、Off：仅会在VPA控制器中发现建议的资源申请值，Pod中的依然是原来的值。相当于`dry run`模式

  2、Initial：仅会在Pod被创建时给予一个推荐的值，在Pod运行过程中不会修改。

  3、Recreate：默认模式。除了在Pod创建时给予一个推荐值外，在Pod运行过程中会反复调整。调整的方式就是删除旧Pod新建新的Pod

  4、Auto：相当于Recreate

#### 已知问题

https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#known-limitations
