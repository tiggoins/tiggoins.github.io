---
title: Docker网络中的Veth pair
date: 2020-08-30 22:45:47
tags:
 - Docker
categories:
 - Docker
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/MakeHay_ZH-CN5759590757_1920x1080.jpg
---

Docker中的veth pair是一种特殊的网络设备，目的是让容器内能顺利访问外部网路
<!--more-->

### 复习docker三种网络

docker在安装后会默认生成三种网络，none、bridge及host。

```yaml
[root@k8s-master ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
089c60c71261        bridge              bridge              local
cc6b4177daf6        host                host                local
4d8e46882a41        none                null                local
```



#### none网络

顾名思义，就是挂载这个网络下的容器仅有lo网络，没有其他网卡，也就不能与外界通信，适合对安全性要求较高且不需要联网的容器。

```shell
[root@k8s-master ~]# docker run -it --network=none busybox        
/ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

  

#### host网络

连接到host网络的容器会共享宿主机的网络栈,容器内的网络配置与宿主机一致

```shell
[root@k8s-master ~]# docker run -it --network=host busybox
/ # ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
   link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq qlen 1000
    link/ether 00:0c:29:cf:da:70 brd ff:ff:ff:ff:ff:ff
...
```

直接使用host网络好处就是性能，如果容器对网络有较高要求可以直接使用host网络。*但是*：使用host网络也会牺牲灵活性，host已经使用的端口容器中将无法使用。



#### bridge网络

默认网络，运行容器时如果不指定网络类型就会使用bridge网络

当前docker0网卡上没有任何网络设备

   ```shell
[root@k8s-master ~]# bridge link show dev docker0 | grep docker0
[root@k8s-master ~]# 
   ```

在运行一个busybox容器后再次查看会发现一个新的网络接口vetha8b45fa@if30挂载到了docker0，这个就是busybox的虚拟网卡

```shell
[root@k8s-master ~]# docker run -itd --name busybox busybox
99dfbb8efef7be8f7c5e85d7d8f7860c26c269e003c8ba36a93d0c5d2f1f0a89
[root@k8s-master ~]# bridge link show | grep docker0       
31: vetha8b45fa@if30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2 
```

但是查看busybox的网络配置却没有找到这个vetha8b45fa@if30设备

```shell
[root@k8s-master ~]# docker exec -it busybox ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
30: eth0@if31: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
```

> 这里的eth0与vetha8b45fa@if30就是一对veth pair设备。可以把veth设备看做是一对网卡，一块网卡(eth0)在容器内，另一块(vetha8b45fa@if30)挂载在docker0上。目的就是把容器的eth0挂到docker0上。



查看容器的网关,这个网关就是docker0的ip地址

```shell
[root@k8s-master ~]# docker exec -it busybox ip route
default via 172.17.0.1 dev eth0 
[root@k8s-master ~]# docker network inspect bridge | jq '.[0].IPAM.Config[0].Gateway'
"172.17.0.1"
```

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/veth.png)

### Veth详解

如上面的介绍，Veth设备的作用主要用来连接两个网络命名空间，如同一对网卡中间连着一条网线。既然是一对网卡，那么其中一块网卡称作另一块的peer。在Veth设备的一端发送数据会直接发送到另一端。



#### Veth设备的操作

创建Veth设备对veth0-veth1

```shell
[root@k8s-master ~]# ip link add veth0 type veth peer name veth1
```

创建完成后使用`ip link show`查看新建出来的veth设备信息

```shell
[root@k8s-master ~]# ip link show
...
32: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 86:d5:ad:c4:43:27 brd ff:ff:ff:ff:ff:ff
33: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether da:a4:fc:43:09:d9 brd ff:ff:ff:ff:ff:ff
```

从命名来看就可以看出veth设备的特殊关系了，你中有我，我中有你。目前创建的veth pair都在同一网络空间下，还不算是真正的veth设备。接下来就是创建新的网络命名空间，并且把其中的一块网卡甩到到另一网络空间。

创建新的命名空间ns2

```shell
[root@k8s-master ~]# ip netns add ns2
[root@k8s-master ~]# ip netns list
ns2
```

将veth1置入ns2网络空间中再查看本机中的网卡

```shell
[root@k8s-master ~]# ip link set veth1 netns ns2
[root@k8s-master ~]# ip link show
...
33: veth0@if32: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether da:a4:fc:43:09:d9 brd ff:ff:ff:ff:ff:ff link-netns ns2
```

进入ns2名称空间查看网卡

```shell
[root@k8s-master ~]# ip netns exec ns2 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
32: veth1@if33: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 86:d5:ad:c4:43:27 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

 现在我们就使用veth pair连接本空间与ns2空间，但目前还不能直接通信，必须为两块网卡配置各自的ip地址并启动

```shell
[root@k8s-master ~]# ip netns exec ns2 ip addr add 10.1.1.2/24 dev veth1
[root@k8s-master ~]# ip addr add 10.1.1.1/24 dev veth0
[root@k8s-master ~]# ip netns exec ns2 ip link set dev veth1 up
[root@k8s-master ~]# ip link set veth0 up
```

现在两个网络空间就能直接通信了

```shell
[root@k8s-master ~]# ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.146 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.115 ms
^C
--- 10.1.1.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 54ms
rtt min/avg/max/mdev = 0.115/0.130/0.146/0.019 ms
[root@k8s-master ~]# ip netns exec ns2 ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.101 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.131 ms
^C
--- 10.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 19ms
rtt min/avg/max/mdev = 0.101/0.116/0.131/0.015 ms
```

**总结**:docker就是这样使用veth pair设备连接宿主机网络与容器网络。理解veth设备的工作原理对于理解bridge网络起到至关重要的作用。
