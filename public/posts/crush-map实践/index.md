# CRUSH map实践


**前言**:CRUSH算法是Ceph的核心算法，全称为`可扩展散列下的智能分发机制`(Controlled Replication Under Scalable Hashing)。是整个Ceph数据存储机制的核心。默认安装的Ceph集群会根据当前集群自动生成一套CRUSH map规则，但是默认的CRUSH map可能不符合预期，可以手动修改以满足实际需求。
<!--more-->

> 生产环境中，我们的应用场景可能需要在多种类型设备上创建多个存储池。如基于SSD磁盘创建池创建CephFS的metadata存储池，基于HDD磁盘创建CephFS的data存储池。分而治之，实现Ceph集群的最佳性能。



### 主机列表

| 外部ip         | 集群ip        | host      | 角色         |
| -------------- | ------------- | --------- | ------------ |
| 172.16.200.101 | 10.16.200.101 | ceph-mon1 | mon、osd节点 |
| 172.16.200.102 | 10.16.200.102 | ceph-mon2 | mon、osd节点 |
| 172.16.200.103 | 10.16.200.103 | ceph-mon3 | mon、osd节点 |
| 172.16.200.104 | 10.16.200.104 | ceph-osd4 | osd节点      |

这里假设`ceph-osd4`节点上的磁盘是SSD磁盘，其余节点是HDD磁盘。创建两个存储池：ssd及sata,分别应用到应用到SSD磁盘和HDD磁盘上。需要手动修改CRUSH map以应用此效果。

### 调整CRUSH map

#### 获取现有CRUSH map

登录任一mon节点导出现有的CRUSH map

```
root  /etc/ceph >>> ceph osd getcrushmap -o crushmap_compiled_file

#反编译
root  /etc/ceph >>> crushtool -d crushmap_compiled_file -o crushmap_decompiled_file
#经过反编译后的CRUSH map可以使用任意的编辑器打开

#为防止错误修改CRUSH map导致集群故障，首先备份CRUSH map
root  /etc/ceph >>> cp crushmap_decompiled_file crushmap_fix
```

#### 修改CRUSH map

打开导出的CRUSH map文件，当前的CRUSH map如下：

```bash
...
# buckets
root default {
	id -1		# do not change unnecessarily
	id -2 class hdd		# do not change unnecessarily
	# weight 0.156
	alg straw2
	hash 0	# rjenkins1
	item ceph-mon1 weight 0.039
	item ceph-mon2 weight 0.039
	item ceph-mon3 weight 0.039
	item ceph-osd4 weight 0.039
}

# rules
rule replicated_rule {
	id 0
	type replicated
	min_size 1
	max_size 10
	step take default
	step chooseleaf firstn 0 type host
	step emit
}
```

root default bucket中定义了所有的节点，每个节点的权重一致。创建两个bucket取代default bucket。**注意**id不要与之前出现的id重复。

```bash
root sata {
    id -11 
    alg straw2
    hash 0   
    item ceph-mon1 weight 0.039
    item ceph-mon2 weight 0.039
    item ceph-mon3 weight 0.039
}

root ssd {
    id -12    
    alg straw2
    hash 0
    item ceph-osd4 weight 0.039
}
```

创建两条规则`sata_rule`及`ssd_rule`取代默认的`crush_rule`。

```
rule sata_rule {
    id 0
	type replicated
	min_size 3
	max_size 10
	step take sata
	step chooseleaf firstn 0 type host
	step emit
}

rule ssd_rule {
    id 1
	type replicated
	min_size 3
	max_size 10
	step take ssd
	step chooseleaf firstn 0 type host
	step emit
}
```

**注意**：`sata_rule`的id是`0`需要与之前默认的rule的id保持一致，否则无法应用修改后的CRUSH map。因为之前创建的池找不到对应的`crush_rule`。

#### 应用新的CRUSH map

编译修改后的CRUSH map

```
root  /etc/ceph >>> crushtool -c crushmap_fix -o crushmap_fix_compiled
```

应用新的CRUSH map

```
root  /etc/ceph >>> ceph osd setcrushmap -i crushmap_fix_compiled
```

> 应用新的CRUSH map后。集群将开始数据调整和数据恢复，过一段时间后集群才会重新回到`HEALTH_OK`状态。

### 新建存储池

一旦Ceph集群回到`HEALTH_OK`状态，就可以创建存储池了。这里定义两个存储池：sata池和ssd池，并且分别定义`crush_rule`为`sata_rule`和`ssd_rule`，即CRUSH map中新定义的`sata_rule`和`ssd_rule`。

#### 创建存储池

创建存储池sata及ssd

```
root  /etc/ceph >>> ceph osd pool create sata 64 64
pool 'sata' created

root  /etc/ceph >>> ceph osd pool create ssd 64 64    
pool 'ssd' created
```

#### 为存储池指定crush rule

分别为存储池sata及ssd指定crush_rule

```
root  /etc/ceph >>> ceph osd pool set sata crush_rule sata_rule
set pool 11 crush_rule to sata_rule

root  /etc/ceph >>> ceph osd pool set ssd crush_rule ssd_rule     
set pool 12 crush_rule to ssd_rule
```

#### 查看现有的池规则

```
root  /etc/ceph >>> ceph osd dump | egrep -i "sata|ssd"
pool 11 'sata' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 106 flags hashpspool stripe_width 0
pool 12 'ssd' replicated size 3 min_size 2 crush_rule 1 object_hash rjenkins pg_num 64 pgp_num 64 last_change 107 flags hashpspool stripe_width 0
```

sata池与ssd池对应的`crush_rule`分别为0和1，就是修改后的CRUSH map中`sata_rule`和`ssd_rule`中的id。

### 测试存储效果

#### 创建新文件

```
root  /etc/ceph >>> dd if=/dev/zero of=sata.file bs=1M count=64 conv=fsync
64+0 records in
64+0 records out
67108864 bytes (67 MB) copied, 0.167555 s, 401 MB/s

root  /etc/ceph >>> dd if=/dev/zero of=ssd.file bs=1M count=64 conv=fsync    
64+0 records in
64+0 records out
67108864 bytes (67 MB) copied, 0.151092 s, 444 MB/s
```

#### 上传文件到指定的池

rados为对象存储的命令行工具，上传对象的子命令为`rados put <obj-name> <infile> -p <pool-name>`

```bash
root  /etc/ceph >>> rados put sata.pool.object sata.file -p sata

root  /etc/ceph >>> rados put ssd.pool.object ssd.file -p ssd 
```

#### 检查池中的对象信息

```bash
root  /etc/ceph >>> ceph osd map sata sata.pool.object     
osdmap e110 pool 'sata' (11) object 'sata.pool.object' -> pg 11.f71bcbc2 (11.2) -> up ([0,2,4], p0) acting ([0,2,4], p0)

root  /etc/ceph >>> ceph osd map ssd ssd.pool.object 
osdmap e110 pool 'ssd' (12) object 'ssd.pool.object' -> pg 12.82fd0527 (12.27) -> up ([6], p6) acting ([6,0,3], p6)
```

由于副本数为3。查看两个对象`acting`字段中的值，`sata.pool.object`的第一副本位于osd.0,第二第三副本位于osd.2,osd.4上，符合预期。`ssd.pool.object`的第一副本为osd.6为SSD磁盘，其余两个副本位于HDD磁盘上。

