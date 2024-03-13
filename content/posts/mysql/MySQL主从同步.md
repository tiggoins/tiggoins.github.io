---
title: MySQL主从同步
date: 2020-07-25 20:29:47
tags:
 - MySQL
categories:
 - MySQL
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/LofotenIslands_ZH-CN0114482586_1920x1080.jpg
toc: true
disableToC: false
disableAutoCollapse: true
---

单台服务器在企业业务不断发展中必然会遇到访问压力与单点故障，主从复制的架构在一定程度上解决了这个问题。一方面可以实现流量分流，可以让外部请求直接访问主库，内部人员访问从库；另一方面，从库可以当做备份服务器实现主库故障时的手动切换，以满足正常运行的基本要求。

<!--more-->

### 主从同步基本原理

- 主库打开binlog日志记录功能，主从同步就是从库去主库上获取这个binlog日志，并且将binlog日志中记录执行的SQL语句在从库上执行一次

- 从库上执行`start slave`命令即可开始主从同步，主从的数据复制就开始了

- 主从同步开始进行后，从库上的I/O线程就会发送验证请求到主库请求建立连接，并指定位置（这个位置是在从库上执行`chage master`时指定的）后的binlog日志

- 主库在收到从库的验证请求后会返回从库请求的内容，这些内容包括binlog日志的内容，主库中新纪录binlog日志的名称以及新的binlog日志中下一个指定位置点等信息

- 从库收到主库发送来的binlog日志后会将其存在relay log中，并将新的binlog文件名以及位置点信息存储在master.info文件中

- 从库的SQL线程会实时查看本地relay log线程新增的内容，然后把relay log文件中的内容解析为SQL语句并顺序的在从库一一执行，最后将当前服务器中的中继日志的文件名及位置点信息存储在relay-log.info中以便下次同步需要

  ![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/主从复制.png)



### 主从同步部署

1、服务器环境配置：

|       IP       |   角色   | 操作系统  |
| :------------: | :------: | :-------: |
| 106.14.14.122  | 主数据库 | CentOS7.8 |
| 101.132.38.235 | 从数据库 | CentOS7.8 |

> 本篇博客仅仅是部署测试，现实环境应最大可能使用内网ip。



2、数据库安装：

[数据库安装脚本](https://www.dmesg.top:456/Shellscripts/mysql_installation.sh)



3、主数据库开始binlog日志

在主数据库的配置文件中添加log_bin,开启记录bin_log日志，文件名为mysql-bin

```mysql
log_bin=/opt/mysql/data/mysql-bin
```

重启数据库后会发现在`/opt/mysql/data`目录中会生成两个文件`mysql-bin.000001`和`mysql-bin.index `,`mysql-bin.000001`是记录binlog日志的文件，而index是存放mysql-bin文件名的文件

```bash
[root@mysql-master data]# cat mysql-bin.index 
/opt/mysql/data/mysql-bin.000001
```



实际同步中，为了安全考虑，主数据库的用户授权信息是不会同步给从库的，这就必须显式的在配置文件中告诉mysql在不要同步mysql库 。如果binlog-format是ROW模式，那么忽略同步mysql库的语法是`replicate-wild-ignore-table=mysql.%`;如果是STATEMENT模式，语法为`replicate-ignore-db=mysql`。

> 此项配置需写在从库的配置文件中

```mysql
mysql> show variables like '%binlog_format%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)
```



> ## binlog 三种格式
>
> ### ROW
>
> ROW格式会记录每行记录修改的记录，这样可能会产生大量的日志内容,比如一条update语句修改了100条记录，那么这100条记录的修改都会被记录在binlog日志中，这样造成binlog日志量会很大，这种日志格式会占用大量的系统资源，mysql5.7和myslq8.0安装后默认就是这种格式。
>
> ### STATEMENT
>
> 记录每一条修改数据的SQL语句（批量修改时，记录的不是单条SQL语句，而是批量修改的SQL语句事件）所以大大减少了binlog日志量，节约磁盘IO，提高性能,看上面的图解可以很好的理解row和statement 两种模式的区别。但是STATEMENT对一些特殊功能的复制效果不是很好，比如：函数、存储过程的复制。由于row是基于每一行的变化来记录的，所以不会出现类似问题
>
> ### MIXED
>
> 实际上就是前两种模式的结合。在Mixed模式下，MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选择一种。



4、主库配置同步用户并对从库授权

```mysql
mysql> GRANT REPLICATION SLAVE ON *.* To 'rep'@'101.132.38.235' IDENTIFIED BY '123456';               
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```

查看主库状态

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      896 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```



5、配置从库

```mysql
mysql> CHANGE MASTER TO
    -> MASTER_HOST='106.14.14.122',
    -> MASTER_PORT=3306,
    -> MASTER_USER='rep',
    -> MASTER_PASSWORD='123456',
    -> MASTER_LOG_FILE='mysql-bin.000002',
    -> MASTER_LOG_POS=896;
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

查看从库同步状态

```
mysql> show slave status\G;
*************************** 1. row ***************************
*************************** 略***************************
             Slave_IO_Running: No
            Slave_SQL_Running: Yes
*************************** 略***************************
1 row in set (0.00 sec)
```

发现`Slave_IO_Running: No`报错，查看日志发现

```
2020-07-25T14:18:43.822184Z 6 [ERROR] Slave I/O for channel '': Fatal error: The slave I/O thread stops because master and slave have equal MySQL server ids; these ids must be different for replication to work (or the --replicate-same-server-id option must be used on slave but this does not always make sense; please check the manual before using it). Error_code: 1593
```

关键信息是说主库与从库的server id一样，因为是脚本批量安装，造成了所有服务器server-id=1，手动将从数据库的server-id改为2并重启，再次检查从库状态，发现两个均为YES。至此，主从同步部署完成。



### 主从同步测试

主库创建新库新表并写入测试数据

```mysql
mysql> create database db_test;
Query OK, 1 row affected (0.00 sec)

mysql> use db_test;
Database changed
mysql> CREATE TABLE employees (
    ->     emp_no      INT             NOT NULL COMMENT '主键',
    ->     birth_date  DATE            NOT NULL COMMENT '生日',
    ->     first_name  VARCHAR(14)     NOT NULL COMMENT '用户-姓',
    ->     last_name   VARCHAR(16)     NOT NULL COMMENT '用户-名',
    ->     gender      ENUM ('M','F')  NOT NULL COMMENT '性别',
    ->     hire_date   DATE            NOT NULL COMMENT '入职时间',
    ->     PRIMARY KEY (emp_no)
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO `employees` VALUES 
    -> (10001,'1953-09-02','Georgi','Facello','M','1986-06-26'),
    -> (10002,'1964-06-02','Bezalel','Simmel','F','1985-11-21'),
    -> (10003,'1959-12-03','Parto','Bamford','M','1986-08-28'),
    -> (10004,'1954-05-01','Chirstian','Koblick','M','1986-12-01'),
    -> (10005,'1955-01-21','Kyoichi','Maliniak','M','1989-09-12'),
    -> (10006,'1953-04-20','Anneke','Preusig','F','1989-06-02'),
    -> (10007,'1957-05-23','Tzvetan','Zielinski','F','1989-02-10'),
    -> (10008,'1958-02-19','Saniya','Kalloufi','M','1994-09-15'),
    -> (10009,'1952-04-19','Sumant','Peac','F','1985-02-18'),
    -> (10010,'1963-06-01','Duangkaew','Piveteau','F','1989-08-24');
Query OK, 10 rows affected (0.01 sec)
Records: 10  Duplicates: 0  Warnings: 0
```



登录从库查看

```mysql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| db_test            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use db_test;

mysql> show tables;
+-------------------+
| Tables_in_db_test |
+-------------------+
| employees         |
+-------------------+
1 row in set (0.00 sec)

mysql> select * from employees;
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |
+--------+------------+------------+-----------+--------+------------+
10 rows in set (0.00 sec)
```

数据同步完成







