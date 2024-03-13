---
title: Nginx平滑升级
date: 2020-09-19 17:53:44
tags:
 - Nginx
categories:
 - Nginx
toc: true
disableToC: false
disableAutoCollapse: true
---

本篇博客介绍Nginx的平滑升级，重点是理解Nginx的信号。

<!--more-->

### 信号及作用

官方链接：http://nginx.org/en/docs/control.html

|   信号   | 信号值 |                             作用                             |
| :------: | :----: | :----------------------------------------------------------: |
| INT,TERM |  2,15  |                      立即终止Nginx进程                       |
|   QUIT   |   3    |                    等进程结束后关闭Nginx                     |
|   HUP    |   1    |    根据新的配置文件重建worker进程，即执行nginx -s reload     |
|   USR1   |   10   |               重新写入日志，用于日志切割时使用               |
|   USR2   |   12   | 根据新的可执行文件生成新的master进程，同时保持旧的nginx进程不变 |
|  WINCH   |   28   |                        关闭worker进程                        |



### 设置监控

为了真实模拟现场环境，在服务器上设置监控脚本，当状态码不是200或者超时3秒即判定nginx异常。升级完成后检查result文件是否为空，以此判断升级是否平滑。

```shell
[root@bochs ~]# cat nginx_hot_upgrade 
#!/usr/bin/env bash

date=$(date '+%H:%M:%S')

while true;do
        status=$(curl -m 3 -s -o /dev/null -w "%{http_code}" https://www.dmesg.top:456) 
        if [[ "${status}" -ne 200 ]];then 
        echo "${date}" >> result
        echo "WRONG!" >> result
        fi
        sleep 1
done
[root@bochs ~]# bash nginx_hot_upgrade &
```



### 升级！

> 本次升级nginx从`1.18.0`升级至`1.19.2`。



#### 查看现有的Nginx版本

```shell
[root@bochs ~]# nginx -V
nginx version: nginx/1.18.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
built with OpenSSL 1.1.1g  21 Apr 2020
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-openssl=/mnt/nginxinstall/openssl-1.1.1g --with-zlib=/mnt/nginxinstall/zlib-1.2.11 --with-pcre=/mnt/nginxinstall/pcre-8.44
```

#### 编译新版本Nginx

```shell
#下载1.19.2版本nginx
[root@bochs ~]# curl -o /tmp/nginx-1.19.2.tar.gz http://nginx.org/download/nginx-1.19.2.tar.gz

#解压
[root@bochs ~]# cd /tmp && tar -xzvf nginx-1.19.2.tar.gz

#编译
[root@bochs ~]# cd nginx-1.19.2 && ./configure [取旧版本的编译参数，即上面的configure arguments] && make 
注：由于还是指定了旧的nginx编译选项，此步骤不能执行make install，否则会导致配置文件被覆盖。
```



#### 载入新的Nginx可执行文件

```shell
#备份原有的可执行文件nginx
[root@bochs nginx-1.19.2]# mv /usr/sbin/nginx /usr/sbin/nginx-1.18.0

#拷贝新的可执行文件
[root@bochs nginx-1.19.2]# cp objs/nginx /usr/sbin/nginx

[root@bochs nginx-1.19.2]# ls /usr/sbin/nginx*
/usr/sbin/nginx  /usr/sbin/nginx-1.18.0
```



#### 开始升级

```shell
#查看现有的nginx进程
[root@bochs nginx-1.19.2]# ps -ef | grep -v grep |grep nginx
root     22425     1  0 14:26 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22429 22425  0 14:26 ?        00:00:00 nginx: worker process
```

> 这里需要确保nginx的启动方式是以绝对路径方式启动，否则会出现execve() failed while executing new binary process "nginx" (2: No such file or  directory)的报错。



```shell
#向现有的nginx主进程发送USR2信号
[root@bochs nginx-1.19.2]# kill -USR2 22425
[root@bochs nginx-1.19.2]# ps -ef | grep -v grep |grep nginx        
root     22425     1  0 14:26 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22429 22425  0 14:26 ?        00:00:00 nginx: worker process
root     22503 22425  0 14:27 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22508 22503  0 14:27 ?        00:00:00 nginx: worker process
#此时可以发现nginx生成了新的master进程，并且新的master进程是旧master进程的子进程
```

> 从这里可以看出，USR2信号的作用是保持旧nginx进程不变的情况下生成新的nginx活动进程。此时新的Nginx虽然启动但是并没有实际的响应请求。



```shell
#向现有的nginx主进程发送WINCH信号
[root@bochs nginx-1.19.2]# kill -WINCH 22425
[root@bochs nginx-1.19.2]# ps -ef | grep -v grep |grep nginx
root     22425     1  0 14:26 ?        00:00:00 nginx: master process /usr/sbin/nginx
root     22503 22425  0 14:27 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22508 22503  0 14:27 ?        00:00:02 nginx: worker process
#winch信号关闭了旧master的worker进程，目前新的请求已经是由新的nginx在处理了
```

> 从这里可以看出，WINCH信号的作用是关闭由旧的master进程生成的worker进程。实际响应开始由新的master进程生成的worker进程所处理。



#### 结束

```shell
#到此nginx升级就已经结束了，唯一要做的是确认访问正常。然后停止旧nginx的master进程
[root@bochs nginx-1.19.2]# kill -INT 22425 
[root@bochs nginx-1.19.2]# ps -ef |grep nginx
root     22503     1  0 14:27 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22508 22503  0 14:27 ?        00:00:03 nginx: worker process
```

> 由于旧的Nginx其实不再处理请求，所以这里发送的信号TERM、INT及QUIT都是可以的。同时发现，新的nginx的父进程变成了1，也就是旧nginx的父进程。



```shell
#检查result文件，无任何输出。平滑升级成功！
[root@bochs ~]# cat result
```



### 回滚

当升级完成如果发现与后端应用存在不兼容的情形时应当立即回滚至旧版本Nginx，回滚步骤如下



```shell
[root@bochs nginx-1.19.2]# cd /usr/sbin/

#备份新版本的可执行文件
[root@bochs sbin]# mv nginx nginx-1.19.2

#恢复旧版本可执行文件
[root@bochs sbin]# mv nginx-1.18.0 nginx

#向nginx的master进程发送USR2信号
[root@bochs sbin]# ps -ef | grep nginx
root     22503     1  0 14:27 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22835 22503  0 15:43 ?        00:00:00 nginx: worker process
[root@bochs sbin]# kill -USR2 22503
[root@bochs sbin]# ps -ef | grep nginx
root     22503     1  0 14:27 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    22835 22503  0 15:43 ?        00:00:00 nginx: worker process
root     23121 22503  0 17:42 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    23126 23121  0 17:42 ?        00:00:00 nginx: worker process

#向新nginx的master进程发送WINCH信号和QUIT信号
[root@bochs sbin]# kill -WINCH 22503
[root@bochs sbin]# kill -QUIT 22503

#查看目前Nginx状态
[root@bochs sbin]# ps -ef | grep nginx
root     23121     1  0 17:42 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx    23126 23121  0 17:42 ?        00:00:00 nginx: worker process

#查看result文件无任何输出，回滚完成
[root@bochs ~]# cat result
```



### 总结

Nginx的平滑升级主要是根据不同的信号对可执行文件施加不同的效果。总体而言：

- 先发送`USR2`信号给master进程达到新旧版本共存的效果。
- 发送`WINCH`信号关闭旧版本Nginx的worker进程请求由新版本的worker开始响应。
- 发送`QUIT/TERM/INT`关闭旧版本的master进程
