# pod调度策略


一般而言pod的调度都是通过RC、Deployment等控制器自动完成，但是仍可以通过手动配置的方式进行调度，目的就是让pod的调度符合我们的预期。

<!--more-->



### 定向调度:nodeSelector

定向调度是把pod调度到具有特定标签的node节点的一种调度方式，比如把MySQL数据库调度到具有SSD的node节点以优化数据库性能。此时需要首先给指定的node打上标签，并在pod中设置nodeSelector属性以完成pod的指定调度。



#### 给指定的node打上标签

```shell
[root@k8s-master deployment]# kubectl label nodes <node-name> <key>:<value>

#比如给k8s-master节点打上disk=ssd的属性
[root@k8s-master deployment]# kubectl label nodes k8s-master disk=ssd

#查看k8s-master节点的所有标签
[root@k8s-master deployment]# kubectl label node k8s-master --list=true
beta.kubernetes.io/os=linux
disk=ssd
kubernetes.io/arch=amd64
kubernetes.io/hostname=k8s-master
kubernetes.io/os=linux
node-role.kubernetes.io/master=
beta.kubernetes.io/arch=amd64
```



> pod默认不会调度到master节点，如果需要将其调度到master上，需要为master节点取消污点:<br />  kubectl taint node k8s-master node-role.kubernetes.io/master- 



#### 定义测试文件

```yaml
#nginx-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
     matchLabels:
       app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.18.0
        name: nginx
        ports:
        - containerPort: 80
```



使用该配置创建pod，pod会默认调度到**k8s-node1**节点

```shell
[root@k8s-master deployment]# kubectl apply -f nginx-deployment.yml 
deployment.apps/nginx-deployment deployed

[root@k8s-master deployment]# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-75ddd4d4b4-89rbn   1/1     Running   0          17m   10.244.1.157   k8s-node1   <none>           <none>
nginx-deployment-75ddd4d4b4-jrkmj   1/1     Running   0          17m   10.244.1.159   k8s-node1   <none>           <none>
nginx-deployment-75ddd4d4b4-pffxc   1/1     Running   0          17m   10.244.1.158   k8s-node1   <none>           <none>
```



在配置文件中添加**disk:ssd**的nodeSelector属性

```yaml
...
    spec:
      containers:
...
      nodeSelector:
        disk: ssd
```



重新应用配置文件，就会发现pod全部被调度到具有**disk:ssd**属性的master节点了

```yml
[root@k8s-master deployment]# kubectl apply -f nginx-deployment.yml 
deployment.apps/nginx-deployment configured
[root@k8s-master deployment]# kubectl get pods -o wide                 
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
nginx-deployment-7c4d94f56b-8v8st   1/1     Running   0          91s   10.244.0.35   k8s-master   <none>           <none>
nginx-deployment-7c4d94f56b-9jhcd   1/1     Running   0          87s   10.244.0.37   k8s-master   <none>           <none>
nginx-deployment-7c4d94f56b-blstp   1/1     Running   0          89s   10.244.0.36   k8s-master   <none>           <none>
```



> 定向调度可以把pod调度到特定的node节点，但随之而来的缺点就是如果集群中不存在响应的node，即使有基本满足条件的node节点，pod也不会被调度



### Node亲和性调度:nodeAffinity

NodeAffinity是作为NodeSelector的全新调度策略，相比于NodeSelector而言更具表达力。目前有两种亲和性调度策略：

- requiredDuringSchedulingIgnoredDuringExecution:相当于nodeSelector定向调度，硬限制
- preferredDuringSchedulingIgnoredDuringExecution:软限制尝试调度pod到node上。还可以设置多个软限制并定义权重以实现执行的先后顺序

> IgnoredDuringExecution为如果pod在运行期间node的属性发生了变更，则系统会忽略变更，该pod可以继续在该节点运行。



该配置要求只能运行在k8s-master节点上，并且节点尽可能是sshd(不是sshd也可以运行)

```yaml
#nodeAffinity
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - k8s-master
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx:1.18.0
```

operator支持的操作包括：In,NotIn,Exists,DoesNotExist,Gt,Lt。

NodeAffinity规则注意事项如下：

- 如果同时定义了**NodeSelector**和**NodeAffinity**，则必须两者同时满足pod才会调度到该节点
- 如果定义了多个**nodeSelectorTerms**，则其中一个满足匹配即可成功调度
- 如果定义了多个**matchExpressions**,则必须所有的条件都满足才能调度到该节点

比如flannel网络插件的定义中就指明了必须是amd64架构的linux操作系统才可运行此pod

```yaml
...
spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
...
```





### Pod亲和与排斥调度：podAffinity与podAntiAffinity

相比较NodeAffinity，podAffinity是实现pod与pod间亲和或者互斥调度的策略，亲和性调度可以将pod调度到已经运行具有某种特性的node节点的pod同节点上，而互斥性调度则反之。这里说的某种特性的node节点可以是集群中的节点名称、区域等概念，在定义文件中用**topology**进行表示。

- kubernetes.io/hostname (节点名称)
- failure-domain.beta.kubernetes.io/zone （区域）
- failure-domain.beta.kubernetes.io/region （区域）

与NodeAffinity相似，podAffinity也用**requiredDuringSchedulingIgnoredDuringExecution**及**preferredDuringSchedulingIgnoredDuringExecution**进行亲和性配置，亲和性配置位于Pod.Spec.affinity的PodAffinity子字段下，互斥性配置与podAffinity同级的podAntiAffinity中定义。



#### 定义参照pod

```yaml
[root@k8s-master podAffinity]# cat flag.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: flag
  labels:
    foo: "bar"
    security: "high"
spec:
  containers:
  - image: nginx:1.18.0
    name: nginx
    ports:
    - containerPort: 80
  nodeSelector:
    disk: sshd
```

根据nodeSelector定义的标签，该pod会被调度到k8s-master节点上，同时该pod具有*foo=bar*和*security=high*两个属性。

```shell
[root@k8s-master podAffinity]# kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
flag   1/1     Running   0          89s   10.244.1.183   k8s-master   <none>       <none>
```



#### 亲和性调度

```yaml
[root@k8s-master podAffinity]# cat affinity.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: foo
            operator: In
            values:
            - bar
        topologyKey: kubernetes.io/hostname
  containers:
  - name: affinity
    image: busybox:latest
    command: ["/bin/sh","-c","tail -f /dev/null"]
```

创建该pod之后会发现这两个pod在同一节点运行

```shell
[root@k8s-master ~]# kubectl get pods -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
affinity   1/1     Running   0          22m   10.244.1.186   k8s-node1   <none>           <none>
flag       1/1     Running   0          34m   10.244.1.183   k8s-node1   <none>           <none>
```

> topologyKey: kubernetes.io/hostname 在这里是做为一种参照，两个pod的运行节点的hostname必须相同，后来的pod才能被调度，如果删去，那么新的pod将始终无法被调度而一直处于pending状态





#### 互斥性调度

互斥性调度是保证新pod不会调度到具有运行某标签pod的node节点的调度策略，即保证两个pod不会在同一node节点运行。参照前面的*flag*pod的两个标签*foo=bar*及*security=high*设置互斥策略，关键字**podAntiAffinity**



测试用例仍然采用亲和性调度的列子，只把**podAffinity**改为**podAntiAffinity**。运行该pod发现该pod被调度到与*flag*pod不同的节点上。

```shell
[root@k8s-master podAffinity]# kubectl get pods -o wide
NAME               READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
affinity       1/1     Running   0          42m     10.244.1.186   k8s-node1     <none>           <none>
antiaffinity   1/1     Running   0          3m34s   10.244.0.53    k8s-master   <none>           <none>
flag           1/1     Running   0          54m     10.244.1.183   k8s-node1     <none>           <none>
```

应用：调度三个redis副本到三个不同的节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```



### 容忍和污点：Taints及Tolerations

想比于nodeAffinity期望把某个pod调度到具有某属性node的趋势，容忍和污点是node主动拒绝pod调度到其上的措施。污点是对于node设定的，容忍是对于pod定义的。节点在设置了污点（Taints）后除非pod明确声明了相对应的容忍（Tolerations），那么pod不会被调度到该node上。

默认情况下：master节点不会调度任何pod，因为其中定义了NoSchedule的效果

```shell
[root@k8s-master k8s]# kubectl describe node k8s-master | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule
```

取消的方法：

```shell
[root@k8s-master k8s]# kubectl taint node k8s-master  node-role.kubernetes.io/master- 
node/k8s-master untainted
```

取消后k8s就会允许pod调度该node节点上了，当然也可以在pod中设定相应的容忍。

使用kubectl taint子命令为node设置污点

```
kubectl taint node [node] key=value:[effect]
     其中effect可以有三种取值：
     NoSchedule:不允许调度到该节点上
     PreferNoschedule：尽量避免调度pod到该节点上
     NoExecute：不会调度到该节点，并且该节点上没有设置这个容忍的pod会被驱逐
如： kubectl taint node k8s-master foo=bar:NoSchedule
```

相应的容忍可以用两种设定方式：

```yaml
    tolerations:
    - key: "foo"
      operator: "Equal"
      value: "bar"
      effect: "NoSchedule"
或者：
    tolerations:
    - key: "foo"
      operator: "Exists"
      effect: "NoSchedule"
```

> - operator的值是Exists时不需要执行value
> - operator的值是Equal时pod的value必须和node的value相一致
> 两个特例：
> - 空的key配合Exists可以匹配所有的键值
> - 空的effect匹配所有的effect



k8s允许在一个node上设置多个Taint，也可以在pod上设置多个Toleration。k8s的处理顺序是先列出所有Taint，然后忽略pod中相应的Tolerations，最后没有被匹配到的就是Taint对pod的效果了。

- 剩余Taint中存在effect=NoSchedule，那么调度器不会把pod调度到该node上
- 剩余Taint中存在effect=PreferNoScheduler，则调度器尝试不会调度pod到该node上
- 剩余Taint上存在有NoExecute，并且Pod已经在这个节点运行，那么该pod会被立即驱逐；没有运行，那么该pod不会再被调度到该节点上

例如：对k8s-node1设置两个Taint

```shell
[root@k8s-master k8s]# kubectl taint node k8s-node1 a=b:NoSchedule
[root@k8s-master k8s]# kubectl taint node k8s-node1 c=d:NoExecute
```

在pod中设置了一个容忍：

```shell
    tolerations:
    - key: "a"
      operator: "Equal"
      value: "b"
      effect: "NoSchedule"
```

这样的匹配结果是该pod无法调度到node-1上，因为第二个Taint没有对应的Tolerations。并且由于第二个Taint的效果是NoExecute，那么即使在设置Taint前该pod已在该node运行也会被驱逐。

如果Node上有NoExcute的effect，那么节点上所有没有相应容忍的pod都会被驱逐。而有相应容忍的可以一直在该node上运行。此外系统可以在NoExcute添加可选字段tolerationSeconds字段，用于定义Pod在节点添加相应NoExcute的effect之后还能在Node上运行多久，如

```shell
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

拥有该Tolerations的pod会在运行该pod的Node加上Noexcute后继续运行3600s,随后被驱逐。但是没有定义`tolerationSeconds`则永远不会被驱逐。
