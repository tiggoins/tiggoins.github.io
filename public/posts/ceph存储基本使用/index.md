# Ceph存储基本使用


前言:Ceph能提供三大存储接口，即块存储、文件存储和对象存储。本篇博客主要介绍Ceph实现三种存储的步骤。
<!--more-->

### 主机列表

| 外部ip         | 集群ip        | host      | 角色         |
| -------------- | ------------- | --------- | ------------ |
| 172.16.200.101 | 10.16.200.101 | ceph-mon1 | mon、osd节点 |
| 172.16.200.102 | 10.16.200.102 | ceph-mon2 | mon、osd节点 |
| 172.16.200.103 | 10.16.200.103 | ceph-mon3 | mon、osd节点 |
| 172.16.200.104 | 10.16.200.104 | ceph-osd4 | osd节点      |

### 块存储

块存储可以为客户端提供基于块的持久化存储，通常是格式化成单独的磁盘使用。客户可以灵活的使用这个磁盘，就像新加的磁盘一样，直接作为裸设备使用或者格式化成文件系统挂载到特定的目录。

#### 基本使用

1、创建存储池

```bash
$ ceph osd pool create mypool 128 128
```

2、创建RBD镜像，名称为myimage，大小为10G。feature为layering

```bash
$ rbd create myimage -s 10G --image-feature layering -p mypool 
$ rbd ls -p mypool
myimage
```

3、查看myimage的详细信息

```bash
$ root ~ >>> rbd info myimage -p mypool
rbd image 'myimage':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        id: 16966b8b4567
        block_name_prefix: rbd_data.16966b8b4567
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Thu Dec 17 16:26:28 2020
```

4、将myimage映射为块设备

```bash
$ $  rbd map myimage -p mypool
/dev/rbd0
```

myimage映射为`/dev/rbd0`块设备

同时`showmapped`子命令可以查看所有已经映射的卷

```bash
$ $  rbd showmapped
id pool   image   snap device    
0  mypool myimage -    /dev/rbd0 
```

5、一旦RBD的image映射到了系统上，就应该在其上创建文件系统以使用这个设备

```bash
$ mkfs.xfs /dev/rbd0    #格式化为xfs格式
Discarding blocks...Done.
meta-data=/dev/rbd0              isize=512    agcount=16, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none

$ mkdir /ceph/rbd -p   #创建挂载点

$ mount /dev/rbd0 /ceph/rbd #挂载
$ df -h 
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G   74M  3.9G   2% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   44G   17G   28G  37% /
/dev/sda1               1014M  132M  883M  13% /boot
tmpfs                    799M     0  799M   0% /run/user/0
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-0
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-1
/dev/rbd0                 10G   33M   10G   1% /ceph/rbd
```

6、写入数据

```bash
$ dd if=/dev/zero of=/ceph/rbd/file count=100 bs=1M
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0.135906 s, 772 MB/s
$ cd /ceph/rbd/
$  ls -lh
total 100M
-rw-r--r--. 1 root root 100M Dec 17 16:45 file
```

#### 扩展RBD镜像

ceph块设备支持动态的增加或者减少RBD的大小，但需要底层的文件系统同时作出修改才能使容量生效。**但是**：一般不建议做缩盘操作，可能会造成数据丢失

1、扩展RBD镜像myimage至20G

```bash
$ rbd resize myimage -s 20G -p mypool    
Resizing image: 100% complete...done.

$ rbd info myimage -p mypool 
rbd image 'myimage':
        size 20 GiB in 5120 objects
        order 22 (4 MiB objects)
        id: 16966b8b4567
        block_name_prefix: rbd_data.16966b8b4567
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Thu Dec 17 16:26:28 2020
```

扩展后size大小变成了20G

2、调整文件系统。虽然rbd image已经调整为20G，但是对于文件系统还是之前的10G，需手动扩展文件系统中的大小。由于之前格式为xfs文件系统，故这里使用`xfs_growfs`来调整。

```bash
$ xfs_growfs /ceph/rbd
meta-data=/dev/rbd0              isize=512    agcount=16, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 2621440 to 5242880

$ df -h 
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G   74M  3.9G   2% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   44G   17G   28G  37% /
/dev/sda1               1014M  132M  883M  13% /boot
tmpfs                    799M     0  799M   0% /run/user/0
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-0
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-1
/dev/rbd0                 20G  134M   20G   1% /ceph/rbd
```

#### RBD快照

ceph支持快照，它是一个基于时间点只读的RBD镜像副本。可以通过创建快照并恢复其原始数据，报错RBD镜像的状态。

1、创建测试文件

```bash
$ echo "This is a test file before snapshot" > ./snap-test.txt
$ cat snap-test.txt 
This is a test file before snapshot
$ ls -ltrh
total 101M
-rw-r--r--. 1 root root 100M Dec 17 16:45 file
-rw-r--r--. 1 root root   36 Dec 17 17:14 snap-test.txt
```

2、创建快照:语法是`rbd snap create [--pool <pool>] [--image <image>] [--snap <snap>]`或者`rbd <pool-name>/]<image-name>@<snapshot-name>`

```bash
$ rbd snap create mypool/myimage@snap01

$ rbd snap ls myimage -p mypool
SNAPID NAME     SIZE TIMESTAMP                
     4 snap01 20 GiB Thu Dec 17 17:16:33 2020 
```

3、为了测试快照的恢复的功能，从文件系统删除文件

```bash
$ rm -rf file snap-test.txt 
$ ls
$  
```

4、回滚快照实现文件的回滚

```bash
$ rbd snap rollback mypool/myimage@snap01
Rolling back to snapshot: 100% complete...done.
$ ls
 
$ cd 
$ umount /ceph/rbd
$ mount /dev/rbd0 /ceph/rbd/
$ cd /ceph/rbd/
$ ls
file  snap-test.txt
$ cat snap-test.txt 
This is a test file before snapshot
$  
```

> 回滚快照会导致当前版本及数据被覆盖。应该慎重的使用这个命令。

5、快照删除

当不在需要某个快照版本时可以手动将其删除，删除快照不会删除RBD镜像中的数据

```bash
$ rbd snap rm mypool/myimage@snap01
Removing snap: 100% complete...done.

$ rbd snap purge mypool/myimage     #snap purge可以用于删除这个镜像的所有快照
$ rbd snap ls myimage -p mypool
$ 
```

#### 快照分层

ceph分层特性允许在快照中创建写时副本(COW),这就是快照的分层特性。快照是只读的，但是COW副本是可写的。ceph的分层特性为云平台带来了巨大的便捷性。

1、创建rbd image快照

```bash
$ rbd snap create mypool/myimage@snap_for_clone
$ rbd snap ls mypool/myimage
SNAPID NAME             SIZE TIMESTAMP                
     6 snap_for_clone 20 GiB Fri Dec 18 10:45:38 2020 
```

2、创建COW副本需要先将快照置于保护状态。如果快照被删除，所有连接到这个快照的COW副本都会被删除。处于protect状态的snap无法删除

```bash
$ rbd snap protect mypool/myimage@snap_for_clone
$ rbd snap rm mypool/myimage@snap_for_clone
Removing snap: 0% complete...failed.2020-12-18 16:19:52.377 7fc06bacc840 -1 librbd::Operations: snapshot is protected
rbd: snapshot 'snap_for_clone' is protected from removal.
```

3、复制快照需要父存储池、RBD镜像以及快照的名称：

```bash
$ rbd clone mypool/myimage@snap_for_clone mypool/myimage-cloned
$ rbd ls mypool
myimage
myimage-cloned
```

4、查看`myimage-cloned`的详细信息，会发现特殊的字段，这个字段指明了父镜像的池、镜像及快照信息

```bash
$ rbd info myimage-cloned -p mypool
rbd image 'myimage-cloned':
        size 20 GiB in 5120 objects
        order 22 (4 MiB objects)
        id: 17206b8b4567
        block_name_prefix: rbd_data.17206b8b4567
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Fri Dec 18 16:22:52 2020
        parent: mypool/myimage@snap_for_clone
        overlap: 20 GiB
```

5、扁平化(可选)

要让复制出来的镜像不依赖于父镜像，需要进行扁平化。扁平化会从父镜像中复制数据到子镜像中，所需时间取决于镜像中的数据大小。

```bash
$ rbd flatten mypool/myimage-cloned
Image flatten: 100% complete...done.
$ rbd info mypool/myimage-cloned
rbd image 'myimage-cloned':
        size 20 GiB in 5120 objects
        order 22 (4 MiB objects)
        id: 17206b8b4567
        block_name_prefix: rbd_data.17206b8b4567
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Fri Dec 18 16:22:52 2020
```

扁平化执行完成后将完全与父镜像脱离关系。

```bash
$ rbd map mypool/myimage-cloned
/dev/rbd1
$ mkdir /ceph/rbd-cloned
$ mount /dev/rbd1 /ceph/rbd-cloned
mount: wrong fs type, bad option, bad superblock on /dev/rbd1,
       missing codepage or helper program, or other error

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
$ dmesg | tail
[441827.686175] IPVS: rr: UDP 10.233.0.3:53 - no destination available
[441828.183596] IPVS: rr: UDP 10.233.0.3:53 - no destination available
[441828.183870] IPVS: rr: UDP 10.233.0.3:53 - no destination available
[441828.186889] IPVS: rr: UDP 10.233.0.3:53 - no destination available
[442996.431560] libceph: osd3 172.16.200.102:6802 socket closed (con state OPEN)
[444104.690687] IPVS: Creating netns size=2200 id=135
[444118.448382] IPVS: Creating netns size=2200 id=136
[444796.411407] libceph: osd3 172.16.200.102:6802 socket closed (con state OPEN)
[447089.970167] rbd: rbd1: added with size 0x500000000
[447115.521387] XFS (rbd1): Filesystem has duplicate UUID cb2b61cf-decf-49c9-a302-a1710b0e2187 - can't mount
$ xfs_admin -U generate /dev/rbd1
Clearing log and setting UUID
writing all SBs
new UUID = 79b0e509-c596-4996-ba37-32b7b5f9dd75
$ mount /dev/rbd1 /ceph/rbd-cloned
$ ls /ceph/rbd-cloned
file  snap-test.txt
```

### 文件存储

Ceph的文件存储全称是CephFS,它是一个与POSIX兼容的分布式文件系统，底层使用Ceph rados存储数据。要能正常使用CephFS，集群中应至少包含一个元数据服务器MDS(MataData Server)。

#### 创建MDS

创建MDS可以直接使用ceph-deploy中的`mds create`子命令进行创建

```bash
$ ceph-deploy mds create ceph-mon1 ceph-mon2 ceph-mon3
```

#### 添加MDS节点

1、在需要作为新MDS的节点创建新的目录,${id}可以直接使用`hostname`

```bash
$ mkdir -pv /var/lib/ceph/mds/ceph-${id}
```

2、使用cephx生成秘钥

```bash
$ sudo ceph auth get-or-create mds.${id} mon 'profile mds' mgr 'profile mds' mds 'allow *' osd 'allow *' > /var/lib/ceph/mds/ceph-${id}/keyring
```

3、在新的MDS节点启动MDS服务

```bash
$ sudo systemctl start ceph-mds@${id}
```

4、MDS集群的状态应当如下

```bash
$ ceph mds stat
cephfs-1/1/1 up  {0=ceph-mon1=up:active}, 1 up:standby
```

#### 剔除MDS节点

1、停止对应节点的ceph-mds服务

```bash
$ sudo systemctl stop ceph-mds@${id}
```

2、删除创建的MDS文件夹

```bash
$ sudo rm -rf /var/lib/ceph/mds/ceph-${id}
```

#### 创建存储池

一个ceph文件系统至少需要包含两个存储池，一个用来存储数据，另一个则是用来存储元数据。配置这些存储池时需考虑：

- 为元数据存储池设置较高的副本水平，因为此存储池丢失任何数据都会导致整个文件系统失效。
- 为元数据存储池分配低延时存储器（像 SSD ），因为它会直接影响到客户端的操作延时。

接下来创建两个存储池，一个叫`cephfs_data`,一个叫`ceph_metadata`:

```bash
$  ceph osd pool create cephfs_data 128
$  ceph osd pool create cephfs_metadata 128
```

#### 创建文件系统

在MDS和存储池创建完成后就可以创建文件系统了。创建文件系统的子命令是`fs new`。ceph fs new <fs_name> <metadata> <data>

```bash
$ ceph fs new cephfs cephfs_metadata cephfs_data
$ ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

使用`ceph fs status`查看新建的文件系统的状态

```bash
root /etc/ceph >>> ceph fs status
cephfs - 0 clients
======
+------+--------+-----------+---------------+-------+-------+
| Rank | State  |    MDS    |    Activity   |  dns  |  inos |
+------+--------+-----------+---------------+-------+-------+
|  0   | active | ceph-mon1 | Reqs:    0 /s |   10  |   13  |
+------+--------+-----------+---------------+-------+-------+
+-----------------+----------+-------+-------+
|       Pool      |   type   |  used | avail |
+-----------------+----------+-------+-------+
| cephfs_metadata | metadata | 2635  | 34.1G |
|   cephfs_data   |   data   |    0  | 34.1G |
+-----------------+----------+-------+-------+
+-------------+
| Standby MDS |
+-------------+
|  ceph-mon3  |
|  ceph-mon2  |
+-------------+
MDS version: ceph version 13.2.10 (564bdc4ae87418a232fc901524470e1a0f76d641) mimic (stable)
```

 文件系统创建完成后，MDS服务器就能达到`active`的状态了

```bash
$ ceph mds stat
cephfs-1/1/1 up  {0=ceph-mon1=up:active}, 2 up:standby
```

#### 挂载文件存储

要挂载 Ceph 文件系统，如果你知道mon节点ip地址可以用 `mount` 命令、或者用 `mount.ceph` 工具来自动解析监视器 IP 地址。由于`mount`命令与`mount.ceph`基本一致，所以仅介绍`mount`命令挂载。

1、创建挂载点

```bash
$ mkdir -p /ceph/cephfs/
```

2、挂载文件系统

默认情况下，ceph部署完成后会自动开启cephx认证，所以在挂载时需要指明用户名及秘钥。

```bash
$ cat ./ceph.client.admin.keyring 
[client.admin]
        key = AQA7vdVfBt21JBAAOL2oY8iHCaqNc2Fctv588w==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
$ mount -t ceph 172.16.200.101:/ /ceph/cephfs/ -o name=admin,secret=AQA7vdVfBt21JBAAOL2oY8iHCaqNc2Fctv588w==
root /etc/ceph >>> df -h | grep cephfs
172.16.200.101:/         160G   37G  124G  23% /ceph/cephfs
```

> ceph集群总共是160G

官方建议以更加安全的方式挂载cephFS，就需要把秘钥文件导出到一个文件中，挂载时指明文件名。这样就避免了`secret`残留在bash中。

```bash
$ echo "AQA7vdVfBt21JBAAOL2oY8iHCaqNc2Fctv588w==" > /etc/cepg/ceph-admin.secret
$ mount -t ceph 172.16.200.101:/ /ceph/cephfs/ -o name=admin,secretfile=/etc/ceph/ceph-admin.secret
```

3、写入/etc/fstab

为了保证服务器重启后挂载仍然有效，需要将挂载信息写入到/etc/fstab中。
```bash
$ echo "172.16.200.101:/  /ceph/cephfs/ ceph name=admin,secretfile=/etc/ceph/ceph-admin.secret,noatime 0 2" >>/etc/fstab
```
### 对象存储

Ceph 对象网关是一个构建在 `librados` 之上的对象存储接口，它为应用程序访问Ceph 存储集群提供了一个 RESTful 风格的网关 。同时支持Amazon S3风格及Openstack Swift风格的对象存储接口。从F版开始Ceph运行在Civetweb，不再需要Apache+FastCGI。造成的影响是Civetweb不支持HTTPS,想要以HTTPS的方式访问对象网关，需要在外层部署Nginx等反向代理软件。

#### 开启对象存储

对象存储需要本地安装`ceph-radosgw`软件包，运行的方法也很简单，直接使用`ceph-deploy rgw create ceph-mon1`来开启对象网关服务。默认情况下，对象网关监听7480端口。未授权的情况下访问该端口会提示匿名用户。

```shell
$ curl http://127.0.0.1:7480/      
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
	<Owner>
		<ID>anonymous</ID>
		<DisplayName>
		</DisplayName>
	</Owner>
	<Buckets>
	</Buckets>
</ListAllMyBucketsResult>
```

#### 修改默认端口

修改默认端口需要在`ceph.conf`中添加定义指明新的rados网关的端口。假设修改为8080，修改前确保没有其他程序占用8080端口

```shell
$ cat << EOF >> /etc/ceph/ceph.conf
[client.rgw.ceph-mon1]
rgw_frontends = "civetweb port=8080"
EOF

#推送配置到其他节点
$ ceph-deploy --overwrite-conf config push ceph-mon2 ceph-mon3 ceph-osd4

#重启对象网关服务
$ systemctl restart ceph-radosgw@rgw.ceph-mon1.service
```

#### 使用对象存储

##### S3风格

###### 新建用户

在mon节点(这里是`ceph-mon1`节点)创建新用户`ceph-rgw-testuser`

```shell
$ radosgw-admin user create --uid="ceph-rgw-testuser" --display-name="First RGW User"      
{
    "user_id": "ceph-rgw-testuser",
    "display_name": "First RGW User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "ceph-rgw-testuser",
            "access_key": "3TOIAIEP4JFT0SJKLWY0",
            "secret_key": "7wZ3Kp2qAj03PAgB35JxdYMcanYJRe8XOLqMzTDZ"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```

> `keys`中的`access_key`及`secret_key`是客户端访问对象网关必不可少的，注意保存！

> `access_key`或者`secret_key`中如果出现了`\`,需要手动将其删除。防止部分客户端无法处理转义的值。

###### 测试S3访问

1、安装`python-boto`包

```bash
$ yum install -y python-boto
```

2、创建测试文件

```bash
$ cat s3.py       
#!/usr/bin/env python
import boto
import boto.s3.connection

access_key = '3TOIAIEP4JFT0SJKLWY0'
secret_key = '7wZ3Kp2qAj03PAgB35JxdYMcanYJRe8XOLqMzTDZ'
conn = boto.connect_s3(
        aws_access_key_id = access_key,
        aws_secret_access_key = secret_key,
        host = 'ceph-mon1', port = 8080,
        is_secure=False, calling_format = boto.s3.connection.OrdinaryCallingFormat(),
        )

bucket = conn.create_bucket('ceph-s3-bucket')
for bucket in conn.get_all_buckets():
        print ("{name}".format(name = bucket.name))

```

3、运行测试文件

```bash
$ python s3.py 
ceph-s3-bucket
```

##### swift风格

###### 新建子用户

```bash
$ radosgw-admin subuser create --uid=ceph-rgw-testuser --subuser=ceph-rgw-testuser:swift --access=full
```
> 这里为了方便直接使用上面创建的用户`ceph-rgw-testuser`

###### 生成secret key

```bash
$ radosgw-admin key create --subuser=ceph-rgw-testuser:swift --key-type=swift --gen-secret               
{
    "user_id": "ceph-rgw-testuser",
    "display_name": "First RGW User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [
        {
            "id": "ceph-rgw-testuser:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "ceph-rgw-testuser",
            "access_key": "3TOIAIEP4JFT0SJKLWY0",
            "secret_key": "7wZ3Kp2qAj03PAgB35JxdYMcanYJRe8XOLqMzTDZ"
        }
    ],
    "swift_keys": [
        {
            "user": "ceph-rgw-testuser:swift",
            "secret_key": "ldUzAqsrzyGJohYNC57sB3oOmPqRtazniCnKCXPq"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```

> `swift_keys`中的`secret_key`是客户端访问对象网关必不可少的，注意保存！

###### 测试swift客户端访问

1、安装swift客户端

```bash
$ pip install --upgrade setuptools
$ pip install --upgrade python-swiftclient
```

2、测试访问,成功显示刚才s3客户端创建的bucket

```bash
$ swift -A http://172.16.200.101:8080/auth/1.0 -U ceph-rgw-testuser:swift -K 'ldUzAqsrzyGJohYNC57sB3oOmPqRtazniCnKCXPq' list
ceph-s3-bucket
```

