# 初探GlusterFS


GlusterFS是一种分布式文件系统。分布式文件系统可以有效解决数据的存储和管理难题。将固定于某个地点的某个文件系统，扩展到任意多个节点组成一个文件系统网络。
<!--more-->


### GlusterFS中的一些概念

|      名词      |                             解释                             |
| :------------: | :----------------------------------------------------------: |
|  trusted pool  |                  指加入GlsterFs集群的主机组                  |
|      node      |                   指trusted pool中一台主机                   |
|     brick      |      指代被GlusterFS存储任何设备，是GlusterFS的基本单位      |
|     export     |                指给定服务器上程序块的装入路径                |
| Gluster volume |           指多块brick的集合，如同NFS的/etc/exports           |
|   GNFS和kNFS   | kNFS是指系统自带的NFS，一般而言在GlusterFS中需要禁用。GNFS是指是GlusterFS使用的类NFS，并且无需多余的配置 |



### 安装GlusterFS



> 确保防火墙，selinux已关闭，服务器时间正确。

#### 集群信息

|    服务器ip    | 主机名 | 系统版本  |      硬盘分布      |
| :------------: | :----: | :-------: | :----------------: |
| 172.19.158.161 | node1  | CentOS8.2 | vdb：20GB vdc:20GB |
| 172.19.158.162 | node2  | CentOS8.2 | vdb：20GB vdc:20GB |
| 172.19.158.163 | node3  | CentOS8.2 | vdb：20GB vdc:20GB |
| 172.19.158.164 | client | CentOS8.2 |       客户端       |



#### 节点信息写入Host

各节点均需要操作

```shell
echo "172.19.158.161 node1" >> /etc/hosts
echo "172.19.158.162 node2" >> /etc/hosts
echo "172.19.158.163 node3" >> /etc/hosts
```



#### 安装GlusterFS仓库

```shell
yum install centos-release-gluster
```



#### 格式化并挂载brick



> 这里格式化的磁盘必须要是未分区的磁盘

```shell
#我这里租用的阿里云服务器，三个节点的vdb与vdc是没有分区的磁盘
Disk /dev/vdb: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/vdc: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

#以下操作在三个节点均需要操作

mkfs.xfs -i size=512 /dev/vdb
mkfs.xfs -i size=512 /dev/vdc


mkdir -p /data/brick1  
mkdir -p /data/brick2

echo "/dev/vdb /data/brick1 xfs defaults 1 2" >> /etc/fstab
echo "/dev/vdc /data/brick2 xfs defaults 1 2" >> /etc/fstab

mount -a

#如果你的机器系统是CentOS6，那么需要添加XFS文件格式支持
yum install -y xfsprogs
```



#### 安装GlusterFS

```shell
yum install -y glusterfs-server
#CentOS8安装此步骤会遇到依赖问题：nothing provides shell3-pyxattr needed by glusterfs-server-7.7-1.el8.x86_64
#解决方法：从第三方安装shell3-pyxattr 
#yum install -y ftp://ftp.pbone.net/mirror/archive.fedoraproject.org/fedora/linux/releases/27/Everything/x86_64/os/Packages/p/shell3-pyxattr-0.5.3-12.fc27.x86_64.rpm

systemctl start glusterd && systemctl enable glusterd

#各节点检查是否正常启动，GlusterFS默认运行于24007端口
#[root@node1 ~]# ss -tnlp
```



#### 配置trusted pool

```shell
#在任一节点操作，将各节点加入信任主机池中
#比如在node1节点操作
gluster peer probe node2
peer probe: success.

gluster peer probe node3
peer probe: success.
    
#查看信任池状态，任一节点仅能查看到其余节点的信息
[root@node1 ~]# gluster peer status
Number of Peers: 2

Hostname: node2
Uuid: 2dbd2fb1-186b-4217-be86-04488771f133
State: Peer in Cluster (Connected)

Hostname: node3
Uuid: eff68c51-0462-479e-9f4d-327d6e77dc02
State: Peer in Cluster (Connected)
    
#注意：一旦建立了信任池，外部的主机想要加入到集群中，必须在任一主机添加。新的服务器无法探测到已存在的信任池
```



#### 设置GlusterFs卷

```shell
#所有节点
mkdir /data/brick1/gv0

#在任一节点创建Gluster卷
gluster volume create gv0 replica 3 node1:/data/brick1/gv0 node2:/data/brick1/gv0 node3:/data/brick1/gv0
            
#启用gv0
gluster volume start gv0

#查看gv0信息
[root@node1 ~]# gluster volume info
 
Volume Name: gv0
Type: Replicate                                    #创建的是复制卷
Volume ID: 39ae23b7-c1f6-49b2-8096-6c4fab4265dc
Status: Started                                    #状态是已启动
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp                                #传输方式：tcp
Bricks:                                            #各节点对应的brick
Brick1: node1:/data/brick1/gv0
Brick2: node2:/data/brick1/gv0
Brick3: node3:/data/brick1/gv0
Options Reconfigured:
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on                                    #已禁用kNFS
performance.client-io-threads: off
```



#### 配置客户端并测试

```shell
#安装gluster-client
yum install -y glusterfs-client 

#从任一节点挂载gluster volume gv0
mount -t glusterfs node1:/gv0 /mnt
    
#创建测试数据
cd /mnt;for i in `seq 1 150`; do touch `date +%F`-$i; done

#查看所有节点的数据，因为默认创建的是复制卷，所以会发现节点数据都是一致的
ls /data/brick1/gv0
```


