# CentOS编译安装MySQL8.0


编译安装mysql8.0.18作为测试。顺便记录下安装过程。
<!--more-->

### GCC版本
mysql8.0要求gcc版本要5.5以上，CentOS7默认的gcc版本为4.8.5，CentOS8默认gcc版本为8.1.0。为了方便，本次选用CentOS8.0安装mysql8.0。



### 下载mysql8.0

为了方便，直接下载boost版本
```bash
wget https://cdn.mysql.com//Downloads/MySQL-8.0/mysql-boost-8.0.18.tar.gz #大小为185MB左右
```


### 安装配套软件

```bash
yum install -y gcc gcc-c++ cmake openssl openssl-devel ncurses ncurses-devel libaio-devel 
```


### 关于缺少rpcgen问题的解决

1）下载[rpc.tar.gz](https://www.dmesg.top:456/Middleware/mysql/rpc.tar.gz)至/usr/include解压
2）下载[rpcsvc-proto-1.4.tar.gz](https://www.dmesg.top:456/Middleware/mysql/rpcsvc-proto-1.4.tar.gz)执行以下操作

```bash
tar -xzvf rpcsvc-proto-1.4.tar.gz &&cd rpcsvc-proto-1.4 && ./configure && make && make install
```


### 编译安装

```bash
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/data/mysql/ -DWITH_BOOST=boost -DFORCE_INSOURCE_BUILD=ON 
#安装路径为/usr/local/mysql  数据文件目录/data/mysql/
 
make && make install （make过程长达一小时左右）
```



### 增加配置文件

```bash
cat /etc/my.cnf
[mysqld]
server-id=1
port=3306
basedir=/usr/local/mysql
datadir=/data/mysql
#有其他需要可以另行添加
```


### 权限修改

```bash
chown -R mysql:mysql /usr/local/mysql
chown -R mysql:mysql /data/mysql
chmod -R 755 /usr/local/mysql
chmod -R 755 /data/mysql
```



### mysql初始化

```shell
/usr/local/mysql/bin/mysqld --initialize --user=mysql --datadir=/data/mysql/
```

> 此步骤执行完会产生一个临时的密码用于登录mysql，请保存此密码。

### 启动并登录mysql

```mysql
/usr/local/mysql/bin/mysqld_safe --user=mysql &
/usr/local/mysql/bin/mysql -u root -p"初始密码";
```


### 修改密码

登录进mysql后执行show databases会提示修改密码。为了安全考虑。mysql5.7初始化完成会允许空密码登录，而mysql8.0会生成默认的密码进行登录

在mysql下执行
```shell
alter user 'root'@'localhost' identified by "你要改的密码";
```



### 添加到启动

```shell
cp support-files/mysql.server /etc/init.d/
```

可用选项:  
```bash
service mysql.server start
service mysql.server stop
service mysql.server restart
service mysql.server reload #重载配置文件
service mysql.server status
```


