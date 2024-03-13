# Kubernetes集群部署


Oh! Kubernetes!

<!--more-->



### 基础配置

|    角色    |       IP       | 系统版本  |
| :--------: | :------------: | :-------: |
| k8s-master | 172.19.158.107 | CentOS8.2 |
| k8s-node1  | 172.19.158.108 | CentOS8.2 |
| k8s-node2  | 172.19.158.109 | CentOS8.2 |

在每台机器的hosts文件中添加所有角色名称及对应的ip



#### 各节点进行时间同步



#### 内核参数调整

```bash
cat << EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sysctl -p /etc/sysctl.d/k8s.conf
centos8如果上面两个提示未找到，则需装载netfilter模块 modprobe br_netfilter
```



#### 关闭swap并注释相关启动项

```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```



### 安装Docker

```bash
# 安装必要的一些系统工具
[root@k8s-master ~]# yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加软件源信息
[root@k8s-master ~]# yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 构建镜像缓存
[root@k8s-master ~]# yum makecache

# 安装containerd.io
[root@k8s-master ~]# yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/edge/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm

#安装docker
[root@k8s-master ~]# yum -y install docker-ce

#开启Docker服务并设置为自启动
[root@k8s-master ~]# systemctl start docker 
```



### 安装Kubeadm kubectl kubelet

```bash
#导入阿里云k8s仓库
[root@k8s-master ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#禁用selinux
[root@k8s-master ~]# setenforce 0

#安装kubelet kubeadm kubectl
[root@k8s-master ~]# yum install kubelet-1.18.3 kubeadm-1.18.3 kubectl-1.18.3 -y

#启动kubectl并设置为自启动
[root@k8s-master ~]# systemctl enable kubelet && systemctl start kubelet
```



### 初始化Kubernetes集群

```bash
[root@k8s-master ~]# kubeadm init --apiserver-advertise-address 172.19.158.107 --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version=v1.18.3  
W0729 10:11:06.680899    5631 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
        [WARNING FileExisting-tc]: tc not found in system path
        [WARNING Hostname]: hostname "k8s-master" could not be reached
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
```

>
> 1、detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd"解决方法：官方建议使用systemd作为Docker的cgroup的驱动，需要在/etc/docker/daemon.json添加 "exec-opts": ["native.cgroupdriver=systemd"]
> 2、--pod-network-cidr必须是10.244.0.0/16,这个值是和flannel中的Network值一致的。
> 3、--image-repository=registry.aliyuncs.com/google_containers   指明k8s镜像的拉取地址为阿里云k8s镜像地址，不指定会默认从谷歌镜像仓库拉取。由于众所周知的原因造成一直初始化停滞


> Your Kubernetes control-plane has initialized successfully!
> 出现上述内容表示初始化完成



### 配置kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 这项必须执行，否则会无法使用kubectl命令



### 加入集群

在初始化结束后，最后会提示其他节点加入集群需要的命令


```
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.19.158.107:6443 --token hh3dh6.brp9nfb2s0soby9h \
    --discovery-token-ca-cert-hash sha256:42b48189d75ac6e145873d1248b8e3b2b3354db3516e57f1159d9ab49928fba0 
```

> 其他节点加入主节点需要需首先安装docker及kubernetes的三组件才能执行





### 安装pod网络

要使kubernetes集群正常工作，必须安装pod网络，否则pod之间无法通信。kubernetes支持多种网络组件，flannel，calico等，这里安装flannel网络

在master节点上部署flannel网络插件：

```bash
[root@aliyun ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml 
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```



### 添加工作节点

按上面的加入集群将工作节点加入到集群中，如果master节点未保存相应命令，可以使用`kudeadm token list`查看。

```bash
[root@k8s-node1 ~]# kubeadm join 172.19.158.107:6443 --token hh3dh6.brp9nfb2s0soby9h
```

如果结果中出现了表示加入成功

```bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

在master节点运行`kubectl get nodes`查看已添加工作节点

```bash
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS     ROLES    AGE    VERSION
k8s-master   NotReady   master   105m   v1.18.3
k8s-node1    NotReady   <none>   104m   v1.18.3
k8s-node2    NotReady   <none>   104m   v1.18.3
```

此时会发现所有节点都是**NotReady**状态，原因可能是多种多样的，这是需要借助命令查看详细原因

`kubectl get pod --all-namespaces`:查看所有名称空间下的pod及运行状态

通过`kubectl describe pod <podname> --namespace=<namespace>`查看分析该pod不能运行的详细原因。

等处理完上面的问题，再次执行kubectl get nodes之后，就会发现所有节点都是Running状态，此时Kubernetes集群才算构建完成。

```bash
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   26m   v1.18.3
k8s-node1    Ready    <none>   17m   v1.18.3
k8s-node2    Ready    <none>   17m   v1.18.3
```







