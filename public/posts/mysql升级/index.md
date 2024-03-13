# MySQL升级


随着mysql不断演进，旧的版本不断地会发现新的漏洞，为修复漏洞体验新版本的功能，就需要对数据库进行升级操作。

<!--more-->

### 升级注意点

> 备份！备份！备份！



+ 从5.6升级到5.7需首先升级到5.6最新版；不支持跨版本升级，如直接从5.5升级到5.7

+ 系统初始化时会默认创建`root@localhost`账户，但如果启用了`skip_name_resolve`选项，事先需要给`127.0.0.1`单独授权

+ `mysql_install_db`命令在5.7.5之后被`mysqld --initialize-insecure`取代

+ `mysql.user`的`password`字段在5.7.6版本后已经被`authentication_string`取代，如果选择**In-place**升级，注意运行`mysql_upgrade`命令进行字段转换。如果是**Logical**升级，注意执行`mysqldump`时**必须**包含`--add-drop-table`并且**不**能添加`--flush-privileges`选项

+ 5.7.6之前的版本升级到最新初次启动需要禁用授权表`--skip-grant-tables`





### 升级方法

#### In-Place upgrade

一次in-place升级主要包括停止老的数据库，二进制文件更新，启动新数据库，执行数据库升级命令等

完整操作步骤如下：

1、执行mysql慢速关闭命令，此步骤是为了让脏页刷新到磁盘，避免直接关闭造成数据丢失

```
mysql -u root -p --execute="SET GLOBAL innodb_fast_shutdown=0"
```

2、关闭旧数据库

```
mysqladmin -u root -p shutdown
```

3、如果新的数据库需要和旧数据库在同目录，将旧数据库所在文件夹备份

4、将新的数据库安装包解压

5、如果旧配置文件`/etc/my.cnf`中定义的数据目录不需要更改，将旧数据库的数据目录下所有文件移动到新的数据库数据目录中

6、运行新的数据库

```
mysqld_safe --user=mysql &
```

7、执行`mysql_upgrade`命令，该命令会检查旧数据与新版本不兼容的地方并自动修正同时升级系统数据库以应用新特性。

````shell
mysql_upgrade -uroot -p 
````

8、重新启动数据库使`mysql_upgrade`的变更生效

```
mysqladmin -uroot -p shutdown
mysqld_safe --user=mysql &
```

9、升级完成



#### Logical upgrade

一次逻辑升级，包括从旧数据库导出SQL文件、安装新的数据库、导入SQL文件到新数据库中等

完整操作步骤如下：

1、从旧数据库导出SQL文件，包括系统表。如果用到了存储程序，还需在选项中添加`--events`和`--routines`参数，不推荐在导出时`gtid_mode`处于开启状态。

```shell
[root@bochs ~]# mysql -uroot -p -e "show variables like 'gtid_mode%';"
Enter password: 
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | OFF   |
+---------------+-------+
```

```shell
mysqldump -u root -p
 --add-drop-table --routines --events
 --all-databases --force > data-for-upgrade.sql
```


2、关闭旧数据库

```shell
mysqladmin -uroot -p shutdown
```

3、安装新的数据库实例，可以使用旧的配置文件，与新版本不兼容的地方需要手动更改

4、初始化mysql，手动执行数据目录

```shell
mysqld --initialize-insecure
```

> 该步骤会生成临时密码，显示在终端或error日志中,注意保存！

5、启动新数据库

```
mysqld_safe --user=mysql &
```

6、重置root密码

```shell
shell> mysql -u root -p
Enter password: **** <- 输入第4步的临时密码
mysql> ALTER USER USER() IDENTIFIED BY 'your new password';
```

7、将旧数据库导出的SQL文件导入新数据库中

```shell
mysql -u root -p --force < data-for-upgrade.sql
```

8、执行`mysql_upgrade`命令

```shell
mysql_upgrade -uroot -p
```

9、重启新数据库，使`mysql_upgrade`命令生效

```shell
mysqladmin -uroot -p shutdown
mysqld_safe --user=mysql &
```

10、升级完成



### 方法比较

+ In-place升级方式不改变数据文件，升级速度快，但不支持跨操作系统升级
+ Logical升级方式需要导出导入数据，在数据量较大的情况下速度会十分缓慢；支持跨操作系统升级






