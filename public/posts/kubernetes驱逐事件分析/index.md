# Kubernetes驱逐事件分析


**前言**：本文主要分析了针对pod`custom-metrics-apiserver`的驱逐事件，分析相关成因并给出解决措施。

<!--more-->



### 问题

在学习HPA自动伸缩时，部署完`custom-metrics-apiserver`这个的Deployment后资源后经过一段时间总能观察到大量的驱逐事件:

```bash
$ kubectl get pods -n custom-metrics 
NAME                                       READY   STATUS    RESTARTS   AGE
custom-metrics-apiserver-f884776d4-9x9br   0/1     Evicted   0          6h34m
custom-metrics-apiserver-f884776d4-cztcr   0/1     Evicted   0          8h
custom-metrics-apiserver-f884776d4-fpr25   0/1     Evicted   0          4h46m
custom-metrics-apiserver-f884776d4-lcvtt   1/1     Running   0          33m
custom-metrics-apiserver-f884776d4-pb58n   0/1     Evicted   0          5h28m
custom-metrics-apiserver-f884776d4-plzr5   0/1     Evicted   0          99m
custom-metrics-apiserver-f884776d4-q7fpz   0/1     Evicted   0          10h
custom-metrics-apiserver-f884776d4-tllrm   0/1     Evicted   0          154m
custom-metrics-apiserver-f884776d4-wqnw9   0/1     Evicted   0          7h41m
custom-metrics-apiserver-f884776d4-xh9wz   0/1     Evicted   0          8h
custom-metrics-apiserver-f884776d4-xrs44   0/1     Evicted   0          3h40m
```

一开始并不了解是什么情况也是第一次见到`STATUS=Evicted`的状态，只是简单的删除被驱逐的pod`kubectl delete pods --all -n custom-metrics`。但是经过一段时间后又会出现同样的情况，本着发生驱逐就意味着节点资源紧缺的判断，有必要分析下驱逐的原因并且梳理下避免驱逐的措施。



### 分析

从kubelet日志可以发现，是由于`ephemeral-storage`不足而引发的驱逐。那么这个`ephemeral-storage`是什么呢？带着问题去[kubernetes官网]([Managing Resources for Containers | Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#setting-requests-and-limits-for-local-ephemeral-storage))搜索，发现这是类似于cpu、内存的一种资源，暂时翻译为**临时性存储**。

```
node-1 kubelet[7082]: W 7082 eviction_manager.go:330] eviction manager: attempting to reclaim ephemeral-storage
node-1 kubelet[7082]: I 7082 container_gc.go:85] attempting to delete unused containers
node-1 kubelet[7082]: I 7082 image_gc_manager.go:317] attempting to delete unused images
node-1 kubelet[7082]: I 7082 eviction_manager.go:341] eviction manager: must evict pod(s) to reclaim ephemeral-storage
node-1 kubelet[7082]: I 7082 eviction_manager.go:359] eviction manager: pods ranked for eviction: custom-metrics-apiserver-747fd5d6f6-9dg8t_custom-metrics(e8ea2efe-7e5f-4098-a9a6-359f7e79cdef), node-exporter-khwxn_monitoring(5f8ffd48-e9fe-4273-9937-1e4e6788e437), nginx-proxy-node-1_kube-system(1e396843ff781cfa662c1e9f2423a988), rbd-provisioner-bc84fd48f-nckhq_kube-system(ec731329-782f-4768-b989-b07f04724c0f), ceph-pod_default(bd698243-bea0-4175-993c-00de365a9d19), metrics-server-cc9d976bf-h8jqh_kube-system(d06ea89f-f8ea-474d-9864-82d336621a8b), nodelocaldns-8d9ml_kube-system(ccfcf424-3f70-4a3e-8525-f0fddebc301f), calico-node-kkfsx_kube-system(386b8857-3354-43e9-827d-b7245a4a6e81), kube-proxy-vcw5x_kube-system(a7db076a-a9ff-41b6-9bf5-0f190b99abc9)

node-1 kubelet[7082]: I 7082 kubelet_node_status.go:486] Recording NodeHasDiskPressure event message for node node-1
node-1 kubelet[7082]: I 7082 kubelet_pods.go:1102] Killing unwanted pod "custom-metrics-apiserver-747fd5d6f6-9dg8t"
node-1 kubelet[7082]: I 7082 eviction_manager.go:566] eviction manager: pod custom-metrics-apiserver-747fd5d6f6-9dg8t_custom-metrics(e8ea2efe-7e5f-4098-a9a6-359f7e79cdef) is evicted successfully
#清理残留数据过程...
node-1 kubelet[7082]: I 7082 eviction_manager.go:403] eviction manager: pods custom-metrics-apiserver-747fd5d6f6-9dg8t_custom-metrics(e8ea2efe-7e5f-4098-a9a6-359f7e79cdef) successfully cleaned up
```

日志中提到了需要回收`ephemeral-storage`,并且尝试删除未使用的容器及镜像无果，发现必须要驱逐pod才能完成对于`ephemeral-storage`的回收操作。然后对所有运行中的pod进行打分，`custom-metrics-apiserver-747fd5d6f6-9dg8t_custom-metrics`的分数最高从而被驱逐。在此之间kubelet向apiserver报告了`NodeHasDiskPressure`的事件，并且kubelet在驱逐完pod后对节点资源进行回收等操作。

#### 关于驱逐的误区

一开始我以为驱逐是把k8s把资源紧张的节点上的某个pod移动到另外的节点上，但后来发现所谓的驱逐仅仅是终止pod。在别的节点拉起的动作是由Deployment、ReplicaSet等高层控制器完成的。

- 定义测试pod

```bash
$ cat test-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  hostNetwork: true
  containers:
  - name: nginx
    image: reg.kolla.org/library/nginx:1.18
    imagePullPolicy: IfNotPresent
    ports:
    - name: test-pod
      containerPort: 80
      
$ kubectl get pods 
NAME     READY   STATUS    RESTARTS   AGE
pod      1/1     Running   0          118s
```

- 定义驱逐文件

```bash
$ cat eviction.json 
{
  "apiVersion": "policy/v1beta1",
  "kind": "Eviction",
  "metadata": {
    "name": "pod",
    "namespace": "default"
  }
}
```

- 测试驱逐

```bash
$ kubectl proxy
Starting to serve on 127.0.0.1:8001

#复制终端后执行
$ curl -X POST -d @eviction.json -H 'Content-type: application/json' http://127.0.0.1:8001/api/v1/namespaces/default/pods/pod/eviction
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Success",
  "code": 201
}
```

- 查看结果

```bash
$ kubectl get pods 
No resources found in default namespace.
```

从结果来看，由于pod资源上层没有更高级的副本控制器，在pod被驱逐后pod就无法运行了。所以**驱逐**仅是完成对pod的终止操作。

#### 临时存储的问题

我们知道volume中的`emptyDir`就是一种临时存储，这种存储不能用来存放持久化数据。当pod被删除后，`emptyDir`中的数据就会被清空，所以`emptyDir`这种类型的volume只是用来存储pod运行过程中产生的临时数据，或者是pod内多个container共享临时数据的一种方式。那么是否有可能是`emptyDir`的问题呢？查看配置发现，配置中的确有`emptyDir`类型的卷：

```bash
$ cat custom-metrics-apiserver-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: custom-metrics-apiserver
  name: custom-metrics-apiserver
  namespace: custom-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-metrics-apiserver
  template:
    metadata:
      labels:
        app: custom-metrics-apiserver
      name: custom-metrics-apiserver
    spec:
      serviceAccountName: custom-metrics-apiserver
      containers:
      - name: custom-metrics-apiserver
        image: reg.kolla.org/prometheus/k8s-prometheus-adapter-amd64 
        args:
        - --secure-port=6443
        - --tls-cert-file=/var/run/serving-cert/serving.crt
        - --tls-private-key-file=/var/run/serving-cert/serving.key
        - --logtostderr=true
        - --prometheus-url=http://prometheus-k8s.monitoring.svc:9090/
        - --metrics-relist-interval=1m
        - --v=10
        - --config=/etc/adapter/config.yaml
        ports:
        - containerPort: 6443
        volumeMounts:
        - mountPath: /var/run/serving-cert
          name: volume-serving-cert
          readOnly: true
        - mountPath: /etc/adapter/
          name: config
          readOnly: true
        - mountPath: /tmp
          name: tmp-vol
      volumes:
      - name: volume-serving-cert
        secret:
          secretName: cm-adapter-serving-certs
      - name: config
        configMap:
          name: adapter-config
      - name: tmp-vol
        emptyDir: {}
```

但是仔细分析配置文件发现这个emptyDir卷的作用不大可以忽略，尝试注释这个名为tmp-vol的emptyDIr以及上面的挂载配置之后重新应用后发现过一段时间后仍然会出现大量的Eviction，那就证明与配置中定义的emptyDir没有关系。

#### 资源配额的问题

思前想后，翻阅有关驱逐的定义：驱逐是指当某个节点的资源紧张时。既然是资源紧张，那么我设置一个ResourceQuota试试看：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: custom-metrics
spec:
  hard:
    pods: "2" 
    requests.cpu: "1" 
    requests.memory: 512Mi 
    requests.ephemeral-storage: 4Gi 
    limits.cpu: "2" 
    limits.memory: 1Gi 
    limits.ephemeral-storage: 8Gi 
```

由于日志中是`ephemeral-storage`资源紧张，那么这段配置实际的上也是看`requests.ephemeral-storage`和`limits.ephemeral-storage`两个配置。这样设置后就能保证`custom-metrics`名称空间下只能申请最大8Gi的`ephemeral-storage`,由于该名称空间下只有这一个pod，所以也是限制了这个pod的`ephemeral-storage`是8Gi。由于设置ResourceQuota必须要在Deployment的配置中添加资源配额，所以在deployment中增加资源配额的定义:

```yaml
...
   resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
            ephemeral-storage: "4Gi"
          limits:
            memory: "512Mi"
            cpu: "1"
            ephemeral-storage: "8Gi"
...
```

删除并重新应用这个deployment之后发现，不仅没有解决，驱逐的数量反而大大增加

仔细想想其实很快就能理解：原本没有限制ResourceQuota时，这个`ephemeral-storage`肯定是大于8Gi的。加了限制后肯定很快就达到了限值从而被驱逐。

#### 转机

走投无路之后，本着资源紧张才会发生驱逐的想法。在node-1节点部署简单的监控脚本，每分钟输出一次`free -m`、`df -h /`以及`du -sh /var/lib/docker/`,在pod发生驱逐后查看日志果然发现了问题：

```
------------------------------------------
2021-01-05 22:59:01
              total        used        free      shared  buff/cache   available
Mem:           7983        1037        2565         283        4380        6186
Swap:             0           0           0

Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   44G  8.7G   36G  20% /

6.4G	/var/lib/docker
------------------------------------------

...
------------------------------------------
2021-01-05 23:22:01
              total        used        free      shared  buff/cache   available
Mem:           7983        1019        1544         283        5419        6243
Swap:             0           0           0

Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   44G   13G   32G  29% /

11G	/var/lib/docker
------------------------------------------

------------------------------------------
2021-01-06 00:52:01
              total        used        free      shared  buff/cache   available
Mem:           7983         939        1258         283        5785        6324
Swap:             0           0           0

Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   44G   18G   27G  40% /

15G	/var/lib/docker
------------------------------------------

------------------------------------------
2021-01-06 06:43:01
              total        used        free      shared  buff/cache   available
Mem:           7983        1023         626         291        6332        6231
Swap:             0           0           0

Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   44G   38G  6.8G  85% /

35G	/var/lib/docker
------------------------------------------
2021-01-06 07:16:01
              total        used        free      shared  buff/cache   available
Mem:           7983         853        6159         291         970        6423
Swap:             0           0           0

Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   44G  4.6G   40G  11% /

2.2G	/var/lib/docker
------------------------------------------
```

根分区磁盘使用率达到85%后开始回落到11%，查阅了官网中有关于硬驱逐的定义中发现，当`imagefs.available`<15%时就会触发硬驱逐,正好与85%对应：

> The `kubelet` has the following default hard eviction threshold:
>
> - `memory.available<100Mi`
> - `nodefs.available<10%`
> - `nodefs.inodesFree<5%`
> - `imagefs.available<15%`

关于nodefs和imagefs的区别：官网中的说法是这样：

> `kubelet` supports only two filesystem partitions.
>
> 1. The `nodefs` filesystem that kubelet uses for volumes, daemon logs, etc.
> 2. The `imagefs` filesystem that container runtimes uses for storing images and container writable layers.

翻译过来就是：

`nodefs`是kubelet用来存储卷及日志的文件系统，而`imagefs`是容器运行时用来存储镜像及容器可写层的文件系统。我的理解是`imagefs`实际上也和`nodefs`一样使用系统的磁盘空间，干脆直接理解为磁盘的使用率，仅仅是用途不一致。回到监控的输出可以发现`/var/lib/docker/`所占用的磁盘空间越来越大，想了想，什么东西会在容器运行的时候越来越大？只可能是日志了。回到配置文件看到容器的参数`- --v=10`一下子恍然大悟：容器日志级别太低导致疯狂输出日志，大量占用了`/var/lib/docker`目录的空间，逐步积累到磁盘85%的使用率引发kubelet驱逐。被驱逐后由于Deployment的作用在别的节点拉起相同的pod，如此反复造成大量的驱逐事件。



### 解决

联系上文的分析，断定是日志的问题，解决方法也很简单：1、修改日志的级别  2、docker中配置日志相关参数

- 将日志的级别修改为1后重新应用，运行一天后不再观察到驱逐事件：

```bash
$ kubectl get pods -n custom-metrics                   
NAME                                        READY   STATUS    RESTARTS   AGE
custom-metrics-apiserver-6594b9d597-t84q2   1/1     Running   0          28h
```

虽然调整日志级别的方法很管用，但是当容器长期运行时随着日志量的逐步累积仍然可能被驱逐。并且日志级别太高不能很好的分析问题。

- docker启动参数中添加最大日志及日志数量的限制：

```bash
$ cat >> /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "500m",
    "max-file": "3"
  }
}

$ systemctl daemon-reload
$ systemctl restart docker
```

限制最多有3个日志并且每个日志最大500M。长期运行后未发现驱逐事件。其实在部署k8s集群就应该将日志的限制加到docker的配置项中，但我这个是测试环境懒得加，直到出现这个问题才发现是这个原因，生产环境应添加日志配置的参数。

限制前单个日志不断增大造成磁盘紧张引发驱逐：

```bash
$ ls -lh
total 1.0G
-rw-r-----. 1 root root 610M Jan  9 17:01 a9d63a1a2c48765ac9a962d269b5ab59aae2617f6954280826dbdc8cfd22fcd0-json.log
drwx------. 2 root root    6 Jan  9 16:50 checkpoints
-rw-------. 1 root root 7.2K Jan  9 16:50 config.v2.json
-rw-r--r--. 1 root root 2.4K Jan  9 16:50 hostconfig.json
drwx------. 2 root root    6 Jan  9 16:50 mounts
```

限制后日志不断回滚，最大不超过1.5G：

```bash
$ ls -lh
total 970M
-rw-r-----. 1 root root  15M Jan  9 17:02 86acee999a12760622fc7983944ecfe44d8e944b5cdd5905d9d999ff21f9aa4e-json.log
-rw-r-----. 1 root root 477M Jan  9 17:01 86acee999a12760622fc7983944ecfe44d8e944b5cdd5905d9d999ff21f9aa4e-json.log.1
-rw-r-----. 1 root root 477M Jan  9 16:53 86acee999a12760622fc7983944ecfe44d8e944b5cdd5905d9d999ff21f9aa4e-json.log.2
drwx------. 2 root root    6 Jan  9 16:10 checkpoints
-rw-------. 1 root root 7.2K Jan  9 16:10 config.v2.json
-rw-r--r--. 1 root root 2.4K Jan  9 16:10 hostconfig.json
drwx------. 2 root root    6 Jan  9 16:10 mounts
```





