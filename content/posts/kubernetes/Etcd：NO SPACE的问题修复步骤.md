---
title: Etcd：NO SPACE的问题修复步骤
date: 2021-01-17
tags:
 - Kubernetes
 - Etcd
categories:
 - Kubernetes
 - Etcd
toc: true
disableToC: false
disableAutoCollapse: true
---
`kubectl get cs`时发现所有的`etcd`均返回503报错，查看etcd的告警发现有`NO SPACE`的信息且`etcdctl endpoints status`中的DB SIZE大于2GiB时。表示etcd的空间超过最大值需进行压缩。不进行压缩api-server无法继续写入数据，影响集群正常工作。

<!--more-->

> 注意：修复会导致所有节点Not Ready，需重启各个节点的`kubelet`

- kubernetes版本：v1.17.0

- etcd版本：3.3.10

- etcd运行方式：docker容器

### 解决步骤

1、将API版本调整为3

```shell
$ export ETCDCTL_API=3
```

2、声明变量ETCD_ENDPOINT

```bash
$ ETCD_ENDPOINT=(https://xx:xx:xx:xx:2379,https://xx:xx:xx:xx:2379,https://xx:xx:xx:xx:2379)
$ ETCD_CAFILE=(/etc/ssl/etcd/ssl/ca.pem)
$ ETCD_CERTFILE=(/etc/ssl/etcd/ssl/node-master-1.pem)
$ ETCD_KEYFILE=(/etc/ssl/etcd/ssl/node-master-1-key.pem)
```
> 如果不清楚etcd证书存放的位置可以使用`ps`命令查看
>
> ```bash
> $ ps -ef | grep -v grep | grep apiserver |sed 's/ /\n/g' | grep etcd
> --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem
> --etcd-certfile=/etc/ssl/etcd/ssl/node-master-1.pem
> --etcd-keyfile=/etc/ssl/etcd/ssl/node-master-1-key.pem
> --etcd-servers=https://172.16.200.101:2379,https://172.16.200.102:2379,https://172.16.200.103:2379
> --storage-backend=etcd3
> ```

3、备份etcd

```shell
$ etcdctl --endpoints=${ETCD_ENDPOINT} --cert=${ETCD_CERTFILE} --key=${ETCD_KEYFILE}  \
--cacert=${ETCD_CAFILE} snapshot save /var/lib/etcd/my-snapshot.db
```

4、获取当前的修订编号,选取需要截断的历史

```bash
$ rev=$(etcdctl --cert=${ETCD_CERTFILE} --key=${ETCD_KEYFILE} --cacert=${ETCD_CAFILE}  \
endpoint status -w json |  egrep -o '"revision":[0-9]*' |  \
egrep -o '[0-9]*'| awk 'NR==1{print $1}')
```

5、按第四步得到的编号进行压缩

```bash
$ etcdctl --endpoints=${ETCD_ENDPOINT} --cert=${ETCD_CERTFILE}   \
--key=${ETCD_KEYFILE} --cacert=${ETCD_CAFILE} compact $rev 
```

6、释放空间

```bash
$ etcdctl --endpoints=${ETCD_ENDPOINT} --cert=${ETCD_CERTFILE}  \
--key=${ETCD_KEYFILE} --cacert=${ETCD_CAFILE}  defrag
```

> 此步骤最好将endpoints列表中的节点拆开依次执行，否则容易引起集群震荡

7、解除告警

```bash
$ etcdctl --endpoints=${ETCD_ENDPOINT} --cert=${ETCD_CERTFILE}  \
--key=${ETCD_KEYFILE} --cacert=${ETCD_CAFILE} alarm disarm 
```

> 此命令可能需执行两次，确保`etcdctl --endpoints=${ETCD_ENDPOINT} --cert=${ETCD_CERTFILE} --key=${ETCD_KEYFILE} --cacert=${ETCD_CAFILE} alarm list`再无输出

8、重启kubelet

经过一段时候后，如果发现所有节点都是`Not Ready`状态,只需要将每个节点的kubelet重启即可

```bash
$ for node in `kubectl get node --no-headers | awk '{print $1}' |  \
xargs`;do ssh ${node} 'systemctl restart kubelet';done
```

9、观察一段时间的DBsize，确保无大幅度增长

```bash
$ etcdctl --endpoints=${ETCD_ENDPOINT} --cert=${ETCD_CERTFILE}  \
--key=${ETCD_KEYFILE} --cacert=${ETCD_CAFILE} endpoints status -w table
```

> 参考：压缩完DB SIZE是30MiB，一段时间涨到132MiB后持续不动
#### KeySpace与存储空间

在解决步骤的第6步，会执行`defrag`命令，这是用来释放碎片文件的。碎片文件是etcd进行`compaction`操作后历史的数据，这部分的空间是可以被用来存放新键的。所以在进行完`compaction`操作后是没有必要进行`defrag`的。只有当etcd的空间满了进行释放空间的操作时才需要进行`defrag`操作。

#### 如何查看etcd空间的实际使用量?
通过`etcdctl`命令查看的DB SIZE是etcd历史最大的数据量大小，不是目前的数据量。etcd对外暴露了mertics提供监控信息，其中名为`etcd_mvcc_db_total_size_in_use_in_bytes`的值即是目前数据量的大小。查询方法可使用下面的命令:
```
curl -s --cacert ${ETCD_CA_FILE} --cert ${ETCD_CERT_FILE} --key ${ETCD_KEY_FILE} `echo ${ETCD_ENDPOINTS}   \
| awk -F, '{print $1}'`/metrics -o-| egrep ^etcd_mvcc_db_total_size_in_use_in_bytes | awk '{print $2}' | awk '{printf("%.2fMiB",$0/1024/1024)}'
```

### 参考文档

- https://etcd.io/docs/v3.3.12/op-guide/maintenance/

