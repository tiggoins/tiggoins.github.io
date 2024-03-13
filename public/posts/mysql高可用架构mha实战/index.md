# MySQL高可用架构MHA实战


MHA（Master High Availability）目前在 MySQL 高可用方面是一个相对成熟的解决方案，在 MySQL 故障切换过程中，MHA 能做到在0~30秒之内自动完成数据库的故障切换操作，并且在进行故障切换的过程中，MHA 能在最大程度上保证数据的一致性，以达到真正意义上的高可用。
<!--more-->
### 服务器信息

MHA架构要求需至少采用三台服务器进行部署

|    主机名    |    服务器IP    |         数据库角色         | 操作系统版本 | MHA角色  |
| :----------: | :------------: | :------------------------: | ------------ | -------- |
| mysql-master | 172.19.158.154 |            主库            | CentOS 8.2   | 管理节点 |
| mysql-node1  | 172.19.158.155 | 从库(主库故障时提升为主库) | CentOS 8.2   | 数据节点 |
| mysql-node2  | 172.19.158.156 |            从库            | CentOS 8.2   | 数据节点 |



### 配置主从同步

配置主从复制,确保两台从库的IO线程以及SQL线程均为YES。

```bash
[root@mysql-node1 ~]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.19.158.155  netmask 255.255.240.0  broadcast 172.19.159.255
        inet6 fe80::216:3eff:fe02:2ee5  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:02:2e:e5  txqueuelen 1000  (Ethernet)
        RX packets 478224  bytes 719808102 (686.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 50004  bytes 3882744 (3.7 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@mysql-node1 ~]# mysql -uroot -p'12344' -e'show slave status\G;'
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.19.158.154
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 896
               Relay_Log_File: relaylog.000003
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```



```bash
[root@mysql-node2 ~]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.19.158.156  netmask 255.255.240.0  broadcast 172.19.159.255
        inet6 fe80::216:3eff:fe10:39b3  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:10:39:b3  txqueuelen 1000  (Ethernet)
        RX packets 496558  bytes 747619292 (712.9 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 53124  bytes 4116370 (3.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@mysql-node2 ~]# mysql -uroot -p'12344' -e'show slave status\G;'
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.19.158.154
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 896
               Relay_Log_File: relaylog.000003
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

如果目前的主库发生宕机，那么需要将其中一台从库提升为主库。这里我选择为`mysql-node1`节点开启`binlog`日志，在配置文件中添加下面两行即可实现。

```mysql
log_bin=/opt/sudytech/mysql/data/mysql-node1-bin
log-slave-updates=1
```



### 配置host及ssh免密登录

各台服务器添加host配置，以`mysql-master`节点为例。其余两台node节点也需要进行同样的配置

```bash
[root@mysql-master ~]# cat >>/etc/hosts<<EOF
> 172.19.158.154 mysql-master 
> 172.19.158.155 mysql-node1 
> 172.19.158.156 mysql-node2
> EOF
```

配置ssh免密登录，以`mysql-master`节点为例，其余node节点也需要进行同样的操作实现ssh免密登录。

> 这里一定要保证各服务器均能免密登录其他服务器，否则mha检测免密登录会失败



```bash
[root@mysql-master ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:mMhr30tVc4QQLlrzH+wEmk8c5QWHallgdtTv4G3PhQE root@mysql-master
The key's randomart image is:
+---[RSA 3072]----+
|          *===+  |
|         + +E+.  |
|        + +++... |
|   . . = B+= oo .|
|    o + S.= +. * |
|     .   + + .o =|
|    o   . . o  oo|
|   . . o        o|
|      . o.       |
+----[SHA256]-----+
[root@mysql-master ~]# ssh-copy-id mysql-master
输入密码后完成登录
[root@mysql-master ~]# ssh-copy-id mysql-node1
输入密码后完成登录
[root@mysql-master ~]# ssh-copy-id mysql-node2
输入密码后完成登录
```

> 配置完上述后测试登录，确保可以登录成功，比较坑的一点就是自己要给自己进行免密的登录验证

### 安装MHA

MHA（Master High Availability）目前在 MySQL 高可用方面是相对成熟的解决方案，是一套优秀的作为 MySQL 高可用性环境下故障切换和主从提升的高可用软件。

项目位于作者的Github，地址如下：

https://github.com/yoshinorim/mha4mysql-manager/releases

https://github.com/yoshinorim/mha4mysql-node/releases

找到其中的`rpm`文件安装到相应的节点即可，**注意**管理节点两个rpm文件均需要安装。

#### 数据节点安装

安装mha4mysql-node

```bash
[root@mysql-node1 ~]# yum install -y https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```

```bash
[root@mysql-node2 ~]# yum install -y https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```

#### 管理节点安装

> 管理节点所需要的包众多且依赖关系复杂，很多软件在阿里云epel都找不到，我在安装时遇到了到很多问题，好在最终是解决了，下面分享我的安装方法。

安装libvirt

```
[root@mysql-master ~]# yum install -y ftp://ftp.pbone.net/mirror/ftp.centos.org/8.1.1911/PowerTools/x86_64/os/Packages/libvirt-libs-4.5.0-35.3.module_el8.1.0+297+df420408.i686.rpm
```

安装perl系列包

```bash
[root@mysql-master ~]# wget -O /tmp/perl.tar.gz https://siteofhexo.oss-cn-shanghai.aliyuncs.com/perl.tar.gz
[root@mysql-master ~]# cd /tmp && tar -xzvf perl.tar.gz
[root@mysql-master ~]# yum install -y *.rpm
```

安装mha4mysql-node

```
[root@mysql-master ~]# yum install -y https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```

安装mha4mysql-manager

```bash
[root@mysql-master ~]# yum install -y https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
```



### MHA配置 

#### 配置MHA管理账户

所有数据库授权给mha用户所有权限

```mysql
mysql> grant all privileges on *.* to 'mha'@'172.19.158.%' identified by 'mha12344';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> set global read_only=1;（仅数据节点设置，防止误写入数据）
```

#### 创建MHA配置文件

仅管理节点配置mha管理用的日志，配置文件等

```bash
[root@mysql-master ~]# mkdir /mha/{work,conf,log} -pv
[root@mysql-master ~]# cat /mha/conf/mha.conf
[root@mysql-master ~]# cat /mha/conf/mha.conf    
[server default]
user=mha                                    #授权给mha的用户名称
password=mha12344                           #授权给mha的用户密码
ping_interval=2                             #检查时间间隔为2s
ssh_user=root                               #远程登录用户

manager_workdir=/mha/work/mha.log
manager_log=/mha/log/mha.log
master_binlog_dir=/opt/sudytech/mysql/data
repl_user=rep                               #主从同步账号
repl_password=12344                         #主从同步密码

[server1]
hostname=mysql-master
port=3306
candidate_master=1

[server2]
hostname=mysql-node1
port=3306
candidate_master=1

[server3]
hostname=mysql-node2
port=3306
no_master=1                                #该数据库永远不会成为主节点
```

#### 检测MHA配置

验证MHA的SSH免密登录

```bash
[root@mysql-master ~]# masterha_check_ssh --conf=/mha/conf/mha.conf 
Sat Sep  5 17:10:10 2020 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Sat Sep  5 17:10:10 2020 - [info] Reading application default configuration from /mha/conf/mha.conf..
...
Sat Sep  5 17:10:12 2020 - [info] All SSH connection tests passed successfully.
Use of uninitialized value in exit at /usr/bin/masterha_check_ssh line 44.
```

出现`All SSH connection tests passed successfully.`表示所有主机间的SSH连接处于正常状态

验证主从同步

```bash
[root@mysql-master ~]# masterha_check_repl --conf=/mha/conf/mha.conf 
...
MySQL Replication Health is OK!
```

出现`MySQL Replication Health is OK!`即表示主从同步状态正常

两个报错的修复方式:

> - Error happened on checking configurations. Redundant argument in sprintf at /usr/share/perl5/MHA/NodeUtil.pm line 201.
> 解决方法：修改`/usr/share/perl5/vendor_perl/MHA/NodeUtil.pm`中的`parse_mysql_major_version`函数为下面的方式
> 
> sub parse_mysql_major_version($) {
        my $str = shift;
        $str =~ /(\d+)\.(\d+)/;
        my $strmajor = "$1.$2";
        my $result = sprintf( '%03d%03d', $strmajor =~ m/(\d+)/g );
        return $result;
> }
>
> 参考连接：https://blog.csdn.net/ctypyb2002/article/details/88344274
>
>
> - Can't exec "mysqlbinlog": No such file or directory at /usr/share/perl5/vendor_perl/MHA/BinlogManager.pm line 106
>
> 解决方法：找到mysql的安装目录将mysql与mysqlbinlog建立软连接到/usr/bin目录下
>
> [root@mysql-master ~]# ln -s /opt/sudytech/mysql/bin/mysqlbinlog /usr/bin/mysqlbinlog
> [root@mysql-master ~]# ln -s /opt/sudytech/mysql/bin/mysql /usr/bin/mysql
>
> 其余各节点执行相同的操作



#### 启动MHA

```bash
[root@mysql-master ~]# masterha_manager --conf=/mha/conf/mha.conf &
```
复制终端会话检查启动日志
```bash
[root@mysql-master ~]# less /mha/log/mha.log
mysql-master(172.19.158.154:3306) (current master)
 +--mysql-node1(172.19.158.155:3306)
 +--mysql-node2(172.19.158.156:3306)

Sat Sep  5 17:38:54 2020 - [warning] master_ip_failover_script is not defined.
Sat Sep  5 17:38:54 2020 - [warning] shutdown_script is not defined.
Sat Sep  5 17:38:54 2020 - [info] Set master ping interval 2 seconds.
Sat Sep  5 17:38:54 2020 - [warning] secondary_check_script is not defined. It is highly recommended setting it to check master reachability from two or more routes.
Sat Sep  5 17:38:54 2020 - [info] Starting ping health check on mysql-master(172.19.158.154:3306)..
Sat Sep  5 17:38:54 2020 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..
```

从日志可以看出目前是mysql-master是主库，mysql-node1和mysql-node2是两个从库

### 测试故障切换

MHA的特点能自动监控当前数据库集群的状态，发生故障时及时的进行转移，MHA部署完成需进行故障切换测试以保证该功能可用。

手动停止主数据库模拟现场环境主库发生宕机等情形。

```bash
[root@mysql-master ~]# service mysql stop
[root@mysql-master ~]# less /mha/log/mha.log
...
----- Failover Report -----

mha: MySQL Master failover mysql-master(172.19.158.154:3306) to mysql-node1(172.19.158.155:3306) succeeded

Master mysql-master(172.19.158.154:3306) is down!

Check MHA Manager logs at mysql-master:/mha/log/mha.log for details.

Started automated(non-interactive) failover.
The latest slave mysql-node1(172.19.158.155:3306) has all relay logs for recovery.
Selected mysql-node1(172.19.158.155:3306) as a new master.
mysql-node1(172.19.158.155:3306): OK: Applying all logs succeeded.
mysql-node2(172.19.158.156:3306): This host has the latest relay log events.
Generating relay diff files from the latest slave succeeded.
mysql-node2(172.19.158.156:3306): OK: Applying all logs succeeded. Slave started, replicating from mysql-node1(172.19.158.155:3306)
mysql-node1(172.19.158.155:3306): Resetting slave info succeeded.
Master failover to mysql-node1(172.19.158.155:3306) completed successfully.
```

MHA检测到了主库发生了宕机，并自动开启了故障转移，将node1节点提升为主库(与mha.conf描述一致)。

> 故障转移完成后masterha_manager会自动停止

测试mysql-node1与mysql-node2主从同步是否正常。

mysql-node1写入数据

```mysql
mysql> INSERT INTO `employees` VALUES 
    -> (10010,'1963-06-01','Duangkaew','Piveteau','F','1989-08-24');
Query OK, 1 row affected (0.00 sec)
```

登录mysql-node2查看，主从同步正常

```mysql
mysql> select * from db_test.employees;
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |
+--------+------------+------------+-----------+--------+------------+
2 rows in set (0.00 sec)
```

当老的主库(mysql-master)完成故障修复后，需要手动将其加入到集群中去。按照方法手动配置主从复制，将老的主库设置为从库并接受主库的同步数据。可以在加入集群后手动将其设置为主库，但是除非必要否则不推荐这么做。

