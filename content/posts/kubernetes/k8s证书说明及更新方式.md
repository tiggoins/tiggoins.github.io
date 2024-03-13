---
title: k8s证书说明及更新方式
date: 2023-05-09
tags:
 - Kubernetes
categories:
 - Kubernetes
---
本篇文章主要介绍k8s集群中证书的作用位置，并且以创建pod和exec为例说明证书调用，最后写了部署监控证书有效期的步骤。
<!--more-->

### 证书概述  

在 Kubernetes 中包含多个以独立进程形式运行的组件，这些组件之间通过 HTTPS/gRPC 相互通信，以协同完成集群中应用的部署和管理工作。

{{< figure src="../../images/posts/k8s/a57c868383c5fe1a3af1481da60e30ab5df8c4.jpg" title="k8s多master架构" >}}

从图中可以看到，Kubernetes控制平面中包含了etcd，kube-apiserver，kube-scheduler，kube-controller-manager等组件，这些组件会相互进行远程调用，例如 kube-apiserver会调用etcd接口查询或数据，kube-controller-manager会调用kube-apiserver接口查询集群中的对象状态；同时kube-apiserver也会和在工作节点上的kubelet和kube-proxy进行通信，以在工作节点上部署和管理应用。

### 证书路径及用途

#### 集群CA证书

|   签发者(CN)   |                  路径                  |          作用          |
| :------------: | :------------------------------------: | :--------------------: |
|   kubernetes   |       /etc/kubernetes/ssl/ca.crt       |   Kubernetes 通用 CA   |
|    etcd-ca     |       /etc/ssl/etcd/ssl/ca.pem       | 与 etcd 相关的所有功能 |
| front-proxy-ca | /etc/kubernetes/ssl/front-proxy-ca.crt |  用于前端代理证书签署  |

#### 服务端证书

|            签发者             |                证书路径                 |      启动命令       | 相关参数或环境变量 |                            用途                             |
| :---------------------------: | :-------------------------------------: | :-----------------: | :----------------: | :---------------------------------------------------------: |
|            etcd-ca            | /etc/ssl/etcd/ssl/member-<nodeName>.pem | /usr/local/bin/etcd |   ETCD_CERT_FILE   |                   提供etcd集群的加密访问                    |
|          kubernetes           |    /etc/kubernetes/ssl/apiserver.crt    |   kube-apiserver    |  {?-}{?-}tls-cert-file   |                提供kube-apiserver的加密访问                 |
| <nodeName>-ca@[UnixTimeStamp] |    /var/lib/kubelet/pki/kubelet.crt     |         无          |         无         | 节点自签证书，提供kubelet的加密访问，**过期不影响集群运行** |

#### 客户端证书

|     签发者     |                     证书路径                     |    启动命令    |      相关参数或环境变量      |                             用途                             |
| :------------: | :----------------------------------------------: | :------------: | :--------------------------: | :----------------------------------------------------------: |
|    etcd-ca     |      /etc/ssl/etcd/ssl/node-[NodeName].pem       | kube-apiserver |       {?-}{?-}etcd-certfile        |              kube-apiserver访问etcd的客户端证书              |
|   kubernetes   | /etc/kubernetes/ssl/apiserver-kubelet-client.crt | kube-apiserver | {?-}{?-}kubelet-client-certificate |            kube-apiserver访问kubelet的客户端证书             |
| front-proxy-ca |    /etc/kubernetes/ssl/front-proxy-client.crt    | kube-apiserver |   {?-}{?-}proxy-client-cert-file   | 前端代理的客户端证书，只有需要支持扩展API服务器时才需要该证书 |

#### 服务账户证书(另一种形式的客户端证书)

|   签发者   |                文件路径                 |        启动命令         |     相关参数或环境变量      |                             用途                             |
| :--------: | :-------------------------------------: | :---------------------: | :-------------------------: | :----------------------------------------------------------: |
| kubernetes |       /etc/kubernetes/admin.conf        |         kubectl         |             无              | 提供给集群管理员操作集群时使用，一般需要复制到`$HOME/.kube/config` |
| kubernetes | /etc/kubernetes/controller-manager.conf | kube-controller-manager | {?-}{?-}authentication-kubeconfig |  提供给kube-controller-manager组件访问kube-apiserver时使用   |
| kubernetes |     /etc/kubernetes/scheduler.conf      |     kube-scheduler      | {?-}{?-}authentication-kubeconfig |       提供给kube-scheduler组件访问kube-apiserver时使用       |
| kubernetes |     /etc/kubenernetes/kubelet.conf      |         kubelet         |        {?-}{?-}kubeconfig         |          提供给kubelet组件访问kube-apiserver时使用           |

> 注：kubelet.conf中的证书文件实际指向`/var/lib/kubelet/pki/kubelet-client-current.pem`，该文件是链接文件在kubelet证书滚动更新后自动指向新的文件。

#### 其他证书

| 签发者  |                文件路径                | 启动命令 | 相关参数或环境变量 |                用途                 |
| :-----: | :------------------------------------: | :------: | :----------------: | :---------------------------------: |
| etcd-ca | /etc/ssl/etcd/ssl/admin-<nodeName>.pem | etcdctl  |       {?-}{?-}cert       | 提供给管理员直接访问/操作etcd的证书 |



### 证书交互过程

以创建Pod和执行kubectl exec为例说明证书交互的过程

#### 以创建Pod为例

{{< mermaid >}}
%%{init:{"theme":"netural"}}%%
sequenceDiagram
    actor user
    participant apiSrv as control plane<br><br>kube-apiserver
    participant etcd as control plane<br><br>etcd
    participant cntrlMgr as control plane<br><br>kube-controller-manager
    participant sched as control plane<br><br>kube-scheduler
    participant kubelet as node<br><br>kubelet
    participant container as node<br><br>container<br>runtime
    user->>apiSrv: 1. kubectl create -f pod.yaml
    apiSrv-->>etcd: 2. 保存到etcd
    cntrlMgr->>apiSrv: 3. kube-controller-manager检查更新
    sched-->apiSrv: 4. kube-scheduler观察到需要创建新的pod
    sched->>apiSrv: 5. kube-scheduler通过算法算出要调度的节点
    apiSrv-->>etcd: 6. 保存pod及节点映射到etcd中
    kubelet->>apiSrv: 7. 对应节点的kubelet观察到需要在本节点创建新的pod
    apiSrv->>kubelet: 8. 绑定pod到对应节点的kubelet
    kubelet->>container: 9. 通知容器运行时运行pod
    kubelet->>apiSrv: 10. 向kube-apiserver更新状态
    apiSrv-->>etcd: 11. 保存最新状态到etcd中
{{< /mermaid >}}

步骤解析：
1. 用户通过kubectl提交资源创建请求，此时使用的集群管理证书即$HOME/.kube/config访问kube-apiserver
2. kube-apiserver使用客户端证书保存资源到etcd中，此时使用的证书为/etc/ssl/etcd/ssl/node-<nodeName>.pem
3. kube-controller-manager检查更新，此时使用kube-controller-manager的客户端证书，即/etc/kubernetes/controller-manager.conf中内嵌的证书
4. kube-scheduler观察到需要调度新的Pod，使用客户端证书连接到`kube-apiserver`，证书位于/etc/kubernetes/scheduler.conf中内嵌的证书
5. kube-scheduler通过各种算法计算出调度的节点，计算完成后提交给kube-apiserver，此时使用的证书同第4步中证书
6. kube-apiserver保存pod与节点的映射到etcd中，此时使用的证书同第2步中的证书
7. kubelet观察到需要在本节点创建新的Pod，证书位于/etc/kubernetes/kubelet.conf中内嵌的证书
8. kube-apiserver绑定pod到对应节点的kubelet，此时使用的证书位于/etc/kubernetes/ssl/apiserver-kubelet-client.crt
9. kubelet通知容器运行时创建pod
10. 创建完成后kubelet向kube-apiserver更新状态，证书同第8步
11. kube-apiserver保存最新的状态到etcd中，证书同第2步



#### 以执行kubectl exec为例

{{< mermaid >}}
%%{init:{"theme":"netural"}}%%
sequenceDiagram
    actor user
    participant apiSrv as control plane<br><br>kube-apiserver
    participant kubelet as node<br><br>kubelet
    participant container as node<br><br>container<br>runtime
    user->>apiSrv: 1. kubectl exec -it pod -- /bin/bash
    apiSrv ->> kubelet: 2.连接到kubelet
    kubelet ->> container: 3.连接到容器运行时
    container ->> kubelet: 4.返回执行结果
    kubelet->>apiSrv: 5.返回执行结果
    apiSrv ->> user: 6.呈现执行结果
{{< /mermaid >}}

步骤解析：

1. 用户执行kubectl exec子命令请求进入容器中执行指令，此时使用的集群管理证书即$HOME/.kube/config访问kube-apiserver
2. kube-apiserver使用客户端证书访问kubelet，证书位于/etc/kubernetes/ssl/apiserver-kubelet-client.crt
3. kubelet通过gRPC接口访问容器运行时，无需证书
4. 容器运行时返回执行结果给kubelet
5. kubelet返回执行结果给kube-apiserver，此时使用kubelet客户端证书访问kube-apiserver，证书位于/etc/kubernetes/kubelet.conf中内嵌的证书
6. 呈现执行的结果


### 证书有效期监控

为防止证书过期造成集群不可用，可部署exporter采集各个证书的有效期并且对小于7天的证书进行告警及时通知集群管理员更新证书。

#### 部署ssl_exporter

部署类型为daemonset

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: ssl-exporter
  name: ssl-exporter
spec:
  selector:
    matchLabels:
      app: ssl-exporter
  template:
    metadata:
      labels:
        app: ssl-exporter
    spec:
      containers:
      - image: reg.kolla.org/library/ssl-exporter:2.4.2
        imagePullPolicy: IfNotPresent
        name: ssl-exporter
        resources: 
          limits:
            cpu: 250m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - mountPath: /host/etc/kubernetes
          name: k8s-certs
          readOnly: true
        - mountPath: /host/var/lib/kubelet/pki
          name: kubelet-certs
          readOnly: true
      hostNetwork: true
      volumes:
      - hostPath:
          path: /etc/kubernetes
          type: ""
        name: k8s-certs
      - hostPath:
          path: /var/lib/kubelet/pki
          type: ""
        name: kubelet-certs
```

#### 配置Prometheus抓取规则

由于服务证书暂不能使用通配符，各个服务账户证书需手动配置策略。

```yaml
scrape_configs:
- job_name: "ssl-k8s-file"
  metrics_path: /probe
  params:
    module: ["file"]
    target: ["/host/etc/kubernetes/ssl/*.crt"]
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: ^(.*):(.*)$
    target_label: __address__
    replacement: ${1}:9219

- job_name: "ssl-kubelet-file"
  metrics_path: /probe
  params:
    module: ["file"]
    target: 
      - "/host/var/lib/kubelet/pki/*.crt"
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: ^(.*):(.*)$
    target_label: __address__
    replacement: ${1}:9219

- job_name: "kubeconfig-admin"
  metrics_path: /probe
  params:
    module: ["kubeconfig"]
    target: ["/host/etc/kubernetes/admin.conf"]
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: ^(.*):(.*)$
    target_label: __address__
    replacement: ${1}:9219

- job_name: "kubeconfig-controller-manager"
  metrics_path: /probe
  params:
    module: ["kubeconfig"]
    target: ["/host/etc/kubernetes/controller-manager.conf"]
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: ^(.*):(.*)$
    target_label: __address__
    replacement: ${1}:9219

- job_name: "kubeconfig-kubelet"
  metrics_path: /probe
  params:
    module: ["kubeconfig"]
    target: ["/host/etc/kubernetes/kubelet.conf"]
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: ^(.*):(.*)$
    target_label: __address__
    replacement: ${1}:9219

- job_name: "kubeconfig-scheduler"
  metrics_path: /probe
  params:
    module: ["kubeconfig"]
    target: ["/host/etc/kubernetes/scheduler.conf"]
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: ^(.*):(.*)$
    target_label: __address__
    replacement: ${1}:9219
```



#### 配置Prometheus告警策略

```yaml
groups:
- name: check_ssl_validity
  rules:
  - alert: "K8S集群证书在7天后过期"
    expr: (ssl_file_cert_not_after - time()) / 86400 < 7
    for: 1h
    labels:
      severity: critical
    annotations:
      description: 'K8S集群的证书还有{{ printf "%.1f" $value }}天就过期了,请尽快更新证书'
      summary: "K8S集群证书证书过期警告"
```

#### 配置Grafana面板

服务账户证书展示

![image-20230508111309091](../../images/posts/k8s/image-20230508111309091.png)

ssl证书展示

![image-20230508111341724](../../images/posts/k8s/image-20230508111341724.png)
