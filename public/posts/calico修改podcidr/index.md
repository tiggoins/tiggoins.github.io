# calico修改podCIDR

前言：pod CIDR是指Kubernetes为pod分配的ip地址段，默认情况下使用kubesparay部署时默认的CIDR是10.233.64.0/16。换算出来的可用地址是10.233.64.1-10.233.127.254。可为64个节点分配pod的ip地址。如果集群扩容超过了64，如何修改podCIDR以允许大量的主机呢？
<!--more-->

ip地址在线换算： "[ip地址在线计算器 (520101.com)](https://tool.520101.com/wangluo/ipjisuan/)" 

> 修改pod CIDR会触及工作负载，需在业务量较小时进行。

### 配置calicoctl

calicoctl的默认配置文件是`/etc/calico/calicoctl.cfg`，如果发现使用`calicoctl get node`提示`no etcd endpoints specified`则需要手动添加该配置文件，样式如下：

```yaml
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "etcdv3"
  etcdEndpoints: "https://10.4.7.11:2379,https://10.4.7.12:2379,https://10.4.7.21:2379"
  etcdKeyFile: "/opt/etcd/ssl/server-key.pem"
  etcdCertFile: "/opt/etcd/ssl/server.pem"
  etcdCACertFile: "/opt/etcd/ssl/ca.pem"
```

如果不清楚etcd的证书路径可以使用`ps -ef | grep etcd`查看具体的证书路径。



### 配置ippool

配置好calicoctl后可以使用`calicoctl get ippool`查看默认的CIDR范围

```yaml
[root@master-1 ~]# calicoctl get ippool 
NAME           CIDR           
default-pool   10.233.64.0/18
```

导出后复制为自定义配置并应用

```shell
[root@master-1 ~]# calicoctl get ippool -o yaml > default-ippool.yaml

[root@master-1 ~]# cat default-ippool.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-pool
spec:
  blockSize: 24
  cidr: 10.233.64.0/18
  ipipMode: Never
  natOutgoing: true
  disabled: true #应用my-ippool后添加改行将该ippool禁用

[root@master-1 ~]# cp default-ippool.yaml my-ippool.yaml

[root@master-1 ~]# 修改手动my-ippool中的cidr值：如10.10.10.0/16，地址范围是10.10.0.1-10.10.255.254,可容纳255台主机

#应用my-ippool.yaml
[root@master-1 ~] calicoctl apply -f my-ippool.yaml

#查看当前ippool状态
[root@master-1 roles]# calicoctl get ippool -o wide
NAME           CIDR             NAT    IPIPMODE   DISABLED   
default-pool   10.233.64.0/18   true   Never      true       
my-ippool      10.10.10.0/16    true   Never      false 
```



> 新建的ippool与默认的ippool的cidr不能有地址重叠，否则会无法创建。

修改cni中定义的地址段根据cni及calico规范，calico部署成功后会在`/etc/cni/net.d/`生成`10-calico.conflist`文件。修改其中的`ipv4_pools`值为my-ippool中定义的CIDR范围。注意各个节点都需要修改并且`calico.conflist.template`需同步修改。修改后保存退出即可。



### 重启所有pod

修改pod CIDR后为使配置生效，手动重启所有pod。deployment，daemonset等控制器会自动拉起pod。

```shell
kubectl delete pods --all
```


