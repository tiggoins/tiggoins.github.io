---
title: 记一次zabbix报警邮件无法接收的问题
date: 2020-02-02 10:29:38
tags:
 - Monitoring
 - TroubleShooting
categories:
 - TroubleShooting
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/GreatReefDay_ZH-CN1185297376_1920x1080.jpg
toc: true
disableToC: false
disableAutoCollapse: true
---

学习zabbix报警媒介，尝试调用shell脚本来完成邮件的发送操作时，在触发动作后，报警邮件顺利发出，但我所在的邮箱却一直没有收到报警邮件。
<!--more-->



### 现象

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/hexo-pic/zabbix1/971D4D6D-5EB0-4ffa-88C7-AAEC05DABC9E.png)
在测试多次后查看zabbix审计日志时发现zabbix均已成功发送邮件，但邮箱中一直没有收到报警邮件。
![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/hexo-pic/zabbix1/DAB4A2B2-5877-4e8c-914D-F45B20A8E328.png)



### 分析

在服务器上尝试调用mailx程序，邮件顺利发送并在邮箱中可以看到报警邮件。
```
Ξ (bochs) ~ → echo "test content" | mailx -s "test title" 17551094687@163.com  
Ξ (bochs) ~ →
```
![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/hexo-pic/zabbix1/Image2.png)
服务器可以顺利的发送邮件，但为什么zabbix发送的邮件迟迟收不到？我转换了思路，不在使用最新版的4.4.5版本，改换成4.0LTS版本，发现问题依然存在，这时就能判断不是zabbix程序的问题。
继续想，会不会是zabbix没有调用到脚本呢，我在脚本中又加入了时间：
```
#!/bin/bash
 
export LANG="zh_CN.utf8"
echo `date "+%Y-%m-%d %H:%M:%S"` 进入脚本>> /tmp/mailx.log
echo $1
echo $2
echo $3

SEND_TO=$1
SEND_SUBJECT=$2
FILE=/tmp/mailtmp.txt
echo "$3" >$FILE
dos2unix -k $FILE

/usr/bin/mailx -s "$SEND_SUBJECT" $SEND_TO < $FILE
echo `date "+%Y-%m-%d %H:%M:%S"` 退出脚本>> /tmp/mailx.log
```
运行测试并查看/tmp/mailx.log，发现zabbix顺利进入了脚本，并且脚本中的参数也正确传递了
```
2020-02-02 09:44:22 进入脚本
************@163.com
故障PROBLEM,服务器:aliyun发生: 无法telnet zabbix_server的10051端口;宏变量函数测试:宏变量测试成功故障!
 事件ID:93PROBLEM:1tzabbix server的10051端口:1宏变量函数测试:宏变量测试成功
2020-02-02 09:44:22 邮件成功发送
```
将脚本中的参数拿到命令行执行参数测试，发现邮件顺利发送。zabbix顺利进入了脚本但是没发送邮件，命令行确是能正常发送。



### 解决

重新梳理了一遍思路后推测，会不会是zabbix无权限调用mailx程序呢？为了验证这个问题，我先将zabbix的默认shell切换为/bin/bash,以使我能顺利的登录到zabbix用户
```
usermod -s /bin/bash zabbix
su - zabbix #切换到zabbix用户
```
调用脚本进行测试发现重要的报错信息
```
-bash-4.2$ echo "11" | mailx -s "22" ############@163.com
-bash-4.2$ Error initializing NSS: Unknown error -8015.
. . . message not sent.
```
这个报错是之前在root账户时从未发生的报错，这时可以断定是权限问题了。查看mailx程序的权限是755，也就是说zabbix可以顺利的调用到mailx程序进行邮件的发送。
后来，终于找到了问题的所在，原来我的mailx配置文件/etc/mail.rc中的秘钥文件位于/root目录下
```
set from="###########@163.com"
set smtp="smtps://smtp.163.com:465"
set smtp-auth-user="###########@163.com"
set smtp-auth-password="############"
set smtp-auth="login"
set ssl-verify="ignore"
set nss-config-dir="/root/.certs"
```
我们知道普通用户是肯定不能进入root用户的家目录的，这就导致了zabbix无法调用到证书，进而无法发送邮件的
将秘钥文件拷贝到其他的目录下，并将这个目录授予777权限，更改mailx的配置文件。
```
cp -R /root/.certs /etc/zabbix && chmod 777 /etc/zabbix/.certs
```
这时测试邮件可以顺利的发送，在尝试用zabbix触发报警邮件，报警邮件顺利发送，邮箱中也有报警邮件了。
![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/hexo-pic/zabbix1/2702EC77-AC9A-432d-86A7-264EF61D6E39.png)
最后，将zabbix的默认shell换回/sbin/nolgin,防止恶意利用
```
usermod -s /sbin/nologin zabbix
```


### 总结

问题终于厘清，邮箱中顺利的接受到了邮件。追根溯源才发现是权限问题，下次有多用户调用的文件时应避免放到/root目录防止普通用户无法调用。
