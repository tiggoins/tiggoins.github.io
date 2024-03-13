---

title: OpenSSH升级
date: 2020-06-26 16:05:47
tags:
 - Linux
categories:
 - Linux
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/SantaCruzRiver_ZH-CN0935957996_1920x1080.jpg
toc: true
disableToC: false
disableAutoCollapse: true
---

OpenSSH是SSH协议的免费开源实现,经常会曝出安全漏洞，由于CentOS7自带的OpenSSH版本（OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017）太低，有必要进行新服务器的OpenSSH版本升级。升级[OpenSSH](https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/)升级前首先需要升级[OpenSSL](https://www.openssl.org/source/)。

<!--more-->



> 本升级教程仅针对CentOS7

### 启用Telnet

> 此步骤是为了防止一旦升级失败导致无法连接到服务器的措施，由于Telnet是明文传输数据，再使用完后务必关闭。

```bash
#安装telnet服务
yum install -y telnet-server

#启动telnet服务
systemctl status telnet.socket

#防火墙开启23端口
firewall-cmd --permanent --add-port=23/tcp
firewall-cmd --reload

#windows打开cmd窗口,telnet即可登陆服务器
telnet [服务器ip]

linux7 因安全原因不支持ROOT账户直接登录，解决方法mv /etc/securetty /etc/securetty_bak
```



### 停止服务并卸载原有的OpenSSH

```shell
systemctl stop sshd
#查看rpm安装的ssh
rpm -qa | grep openssh
#卸载rpm安装的ssh
rpm -e openssh --nodeps && rpm -e openssh-clients --nodeps && rpm -e openssh-server --nodeps
#查看rpm安装的ssh是否卸载
rpm -qa | grep openssh         
```



### 预操作

```shell
#安装相关依赖
yum install -y pam* zlib*
#备份原ssh配置
mv /etc/ssh /etc/ssh_bak
```



### 安装OpenSSL(1.1.1g)

```shell
mkdir ./sshupdate
cd ./sshupdate
wget https://www.openssl.org/source/openssl-1.1.1g.tar.gz
tar -xzvf openssl-1.1.1g.tar.gz
cd openssl-1.1.1g
./config --prefix=/usr/ --openssldir=/usr/ shared
make && make install
#完成后看下openssl版本
openssl version
OpenSSL 1.1.1g  21 Apr 2020
```



### 安装OpenSSH(8.3p1)

```shell
wget https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-8.3p1.tar.gz
tar -xzvf openssh-8.3p1.tar.gz
cd openssh-8.3p1
./configure --with-zlib --with-ssl-dir --with-pam --bindir=/usr/bin --sbindir=/usr/sbin --sysconfdir=/etc/ssh
make && make install
cp contrib/redhat/sshd.init /etc/init.d/sshd
#完成后看下ssh版本
ssh -V
OpenSSH_8.3p1, OpenSSL 1.1.1g  21 Apr 2020
```



### 修改配置文件

```shell
vim /etc/ssh/sshd_config
查找#PermitRootLogin prohibit-password 改成 PermitRootLogin yes 并取消注释
同时如果端口非22端口，需要更改为对应端口

#关闭selinux
sed -i.bak 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
setenforce 0
```



### 重启OpenSSH

```
nohup service sshd restart
nohup systemctl restart sshd
#添加到自启动
chkconfig --add sshd
```



### 测试

重开窗口连接对应服务器，如ssh跳转登录失败的，清空/root/.ssh/下面的文件即可。

