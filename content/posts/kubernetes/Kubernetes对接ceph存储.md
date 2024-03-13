---
title: Kubernetes对接Ceph存储
date: 2020-12-15
tags:
 - Kubernetes
 -  Ceph
categories:
 - Kubernetes
 - Ceph
toc: true
disableToC: false
disableAutoCollapse: true
---

前言：本篇博客主要介绍kubernetes集群如何与ceph集群进行对接，将ceph作为kubernetes的后端存储实现pvc的动态供应。本文中的ceph和kubernetes为一套集群。

<!--more-->



### 主机列表

| K8s集群角色 | ceph集群角色 | IP             | 内核                        |
| ----------- | ------------ | -------------- | --------------------------- |
| master-1    | mon、osd节点 | 172.16.200.101 | 4.4.247-1.el7.elrepo.x86_64 |
| master-2    | mon、osd节点 | 172.16.200.102 | 4.4.247-1.el7.elrepo.x86_64 |
| master-3    | mon、osd节点 | 172.16.200.103 | 4.4.247-1.el7.elrepo.x86_64 |
| node-1      | osd节点      | 172.16.200.104 | 4.4.247-1.el7.elrepo.x86_64 |

> 升级内核的原因：ceph官方推荐4.1.4版本以上的版本

对接前需要保证ceph集群已经处于健康状态

```bash
root ~ >>> ceph health
HEALTH_OK
```

### 对接过程

#### 创建存储池mypool

```bash
root ~ >>> ceph osd pool create mypool 128 128 #设置PG和PGP为128
pool 'mypool' created

root ~ >>> ceph osd pool ls
mypool
```

#### 创建secret对象

在ceph中创建kube用户并授予相应的权限

```bash
root ~ >>> ceph auth get-or-create client.kube mon 'allow r' osd 'allow class-read object_prefix rbd_children,allow rwx pool=mypool'
```

```bash
root ~/k8s/ceph >>> cat ceph-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-kube-secret
  namespace: default
data: 
  key: "ceph auth get-key client.kube | base64" 
type: kubernetes.io/rbd
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: default
data:
  key: "ceph auth get-key client.admin | base64"
type: kubernetes.io/rbd
```

创建上述资源

```bash
root ~/k8s/ceph >>> kubectl apply -f ceph-secret.yaml
```

#### 创建StorageClass

```bash
root ~/k8s/ceph >>> cat ceph-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-storageclass
provisioner: kubernetes.io/rbd
parameters:
  monitors: 172.16.200.101:6789,172.16.200.102:6789,172.16.200.103:6789  #定义mon节点，ceph-mon默认监听6789端口
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: default
  pool: mypool                                                           #定义ceph存储池
  userId: kube
  userSecretName: ceph-kube-secret
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

```bash
root ~/k8s/ceph >>> kubectl apply -f ceph-storageclass.yaml
root ~/k8s/ceph >>> kubectl get sc
NAME                PROVISIONER    AGE
ceph-storageclass   kubernetes.io/rb   87s
```

#### 创建pvc

```bash
root ~/k8s/ceph >>> cat ceph-storageclass-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-test-claim
spec:
  storageClassName: ceph-storageclass  #对应storageclass的名称
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi                    #申请5G空间的存储空间
```

> - ReadWriteOnce: 读写权限，并且只能被单个Node挂载
> - ReadOnlyMany:只读权限，允许被多个Node挂载
> - ReadWriteMany:读写权限，允许被多个Node挂载


#### 创建测试pod

```shell
root ~/k8s/ceph >>> cat ceph-busybox-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod
spec:
  containers:
  - name: ceph-busybox
    image: busybox:1.32.0
    command: ["/bin/sh","-c","tail -f /etc/resolv.conf"]
    volumeMounts:
    - name: ceph-volume
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-volume
    persistentVolumeClaim:
      claimName: ceph-test-claim
```

查看pod的状态，如果发现pod并未处于`running`状态并且事件中有如下的报错。并且k8s集群的各个节点确认已安装与ceph版本一致的`ceph-common`软件包，这时需要更新`storageclass`的`Provisioner`，用外部的提供者代替k8s自带的`kubernetes.io/rbd`以实现pvc的动态供应。

```
rbd: create volume failed, err: failed to create rbd image: executable file not found in $PATH:
```

### 排障步骤

出现这个报错问题的原因其实很简单：`gcr.io`中自带的kube-controller-manager镜像没有自带`rbd`子命令。所以如果是镜像方式启动的`kube-controller-manager`就会遇到这个问题。

Github相关issue：[Error creating rbd image: executable file not found in $PATH · Issue #38923 · kubernetes/kubernetes · GitHub](https://github.com/kubernetes/kubernetes/issues/38923)

#### 部署外部provisioner

定义外部的provisioner，由这个provisioner拿之前定义的秘钥创建ceph镜像。同时对外提供`ceph.com/rbd`的供应者。相关的定义如下：

```shell
root ~/k8s/ceph >>> cat storageclass-fix-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rbd-provisioner
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rbd-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "quay.io/external_storage/rbd-provisioner:latest"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccountName: persistent-volume-binder               
```

> 这里必须明确指明`ServiceAccountName`为`persistent-volume-binder`，默认的`default`账户没有权限列出相应资源

创建该deployment资源

```shell
root ~/k8s/ceph >>> kubectl apply -f storageclass-fix-deployment.yaml
```

#### 更新原有storageclass

```yaml
...
metadata:
  name: ceph-storageclass
#provisioner: kubernetes.io/rbd
provisioner: ceph.com/rbd
...
```

```bash
root ~/k8s/ceph >>> kubectl get sc
NAME                PROVISIONER    AGE
ceph-storageclass   ceph.com/rbd   64m
```

#### 重新创建pvc及pod

```shell
root ~/k8s/ceph >>> kubectl delete -f ceph-storageclass-pvc.yaml && kubectl apply -f ceph-storageclass-pvc.yaml

root ~/k8s/ceph >>> kubectl delete -f ceph-busybox-pod.yaml && kubectl apply -f ceph-busybox-pod.yaml
```

pod成功进入running状态

```yaml
root ~/k8s/ceph >>> kubectl get pods              
NAME       READY   STATUS    RESTARTS   AGE
ceph-pod   1/1     Running   0          47s
```

#### 验证PV

```bash
root ~/k8s/ceph >>> kubectl get pv
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS        REASON   AGE
persistentvolume/pvc-9579436e-5122-48c1-871d-2e18b8e42863   5Gi        RWO            Delete           Bound    default/ceph-test-claim   ceph-storageclass            112s
```
相应的image已映射为块设备挂载到`/usr/share/busybox/`目录下

```
root ~/k8s/ceph >>> kubectl exec -it ceph-pod -- /bin/sh
/ # mount | grep share
/dev/rbd0 on /usr/share/busybox type ext4 (rw,seclabel,relatime,stripe=1024,data=ordered)
```

mypool池中顺利生成相应的image

```bash
root ~/k8s/ceph >>> rbd ls -p mypool
kubernetes-dynamic-pvc-54b5fec8-3e1d-11eb-ab76-925ce2bb963b
```

### 常见错误

1、rbd image无法映射为块设备

```bash
Warning  FailedMount             72s (x2 over 2m13s)  kubelet, node-1          MountVolume.WaitForAttach failed for volume "pvc-c0683658-308a-4fc0-b330-c813c1cad850" : rbd: map failed exit status 110, rbd output: rbd: sysfs write failed
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (110) Connection timed out

#对应节点的dmesg日志：
[843372.414046] libceph: mon1 172.16.200.102:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[843372.415663] libceph: mon1 172.16.200.102:6789 missing required protocol features
[843382.430314] libceph: mon1 172.16.200.102:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[843382.432031] libceph: mon1 172.16.200.102:6789 missing required protocol features
[843393.566967] libceph: mon1 172.16.200.102:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[843393.569506] libceph: mon1 172.16.200.102:6789 missing required protocol features
[843403.421791] libceph: mon2 172.16.200.103:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[843403.424892] libceph: mon2 172.16.200.103:6789 missing required protocol features
[843413.405831] libceph: mon0 172.16.200.101:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
```

**原因**：linux内核版本<4.5时不支持feature flag 400000000000000，手动关闭即可。`ceph osd crush tunables hammer`
