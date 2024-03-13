---
title: Docker三剑客之Docker-Machine
date: 2020-05-11 00:04:38
tags:
 - Docker
categories:
 - Docker
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/LacMidi_ZH-CN9682566554_1920x1080.jpg
toc: true
disableToC: false
disableAutoCollapse: true
---
批量化在主机上安装docker可以使用docker-machine
<!--more-->



Docker Machine可以支持在不同的环境下安装配置docker host

1)  常规的Linux操作系统

2）虚拟化平台VirtualBox、Vmware等

3）公有云Amazon Web Services、Microsoft Azure、Google Compute Engine等



### 实验环境描述

|    操作系统版本    |      ip       | 配置  |  角色   |
| :----------------: | :-----------: | :---: | :-----: |
| Ubuntu 18.04.4 LTS | 172.16.89.101 | 4核8G | machine |
| Ubuntu 18.04.4 LTS | 172.16.89.100 | 4核8G |  node1  |
| Ubuntu 18.04.4 LTS | 172.16.89.99  | 4核8G |  node2  |

官方项目地址位于 https://github.com/docker/machine/releases 目前最新版本是v0.16.2

安装的方法也很简单，只需要执行：

```
curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
chmod +x /tmp/docker-machine &&
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```



执行完成执行docker-machine有正常输出即完成安装

```
root@docker-machine:~# docker-machine ls
NAME   ACTIVE   DRIVER   STATE   URL   SWARM   DOCKER   ERRORS
root@docker-machine:~# 
```



由于docker-machine再安装host时需要免密登录到远程主机上，所以在machine节点上创建ssh秘钥，并发送到node1节点和node2节点上

```
root@docker-machine:~# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:xafvCmF4xRwbNLLI8eOnXZECMYWtqkCBCqPE7Zjj7d0 root@docker-machine
The key's randomart image is:
+---[RSA 2048]----+
|...   . =**      |
|+o.. . +.O.= .   |
|=.+.  o +.O +    |
|o+..   o.+ + .   |
|..o   ..S o .    |
| ...  .o = o     |
|  ..... o . .    |
|   ... E . .     |
|          ...    |
+----[SHA256]-----+
```

```
root@docker-machine:~# ssh-copy-id 172.16.89.100
...
root@172.16.89.100's password: 
...

root@docker-machine:~# ssh-copy-id 172.16.89.99
...
root@172.16.89.99's password: 
...

```



### 开始安装

上述操作完成后就可以开始安装了执行docker-machine create子命令在node1上部署docker了

```
root@docker-machine:~# docker-machine create --driver generic --generic-ip-address=172.16.89.100 node1
Running pre-create checks...
Creating machine...
(node1) No SSH key specified. Assuming an existing key at the default location.
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with ubuntu(systemd)...
Installing Docker...
```

同样的可以执行`docker-machine create --driver generic --generic-ip-address=172.16.89.99 node2`在node2上部署docker



在实际执行中发现由于docker-machine默认使用了官方的镜像站下载docker-ce，造成执行过程会一直卡在"Installing Docker..."，如果机器数量较少推荐先在node节点上使用阿里云镜像安装dock-ce，在到machine节点上创建就行了。安装步骤如下

```
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce
```



在node1和node2节点上完成docker-ce安装后即可以继续回到machine节点操作了，这种方法仅适用于机器较少的情况，如果机器多安装慢，使用Ansible安装后再执行docker-machine create。

```
root@docker-machine:~# docker-machine ls
NAME    ACTIVE   DRIVER    STATE     URL                        SWARM   DOCKER     ERRORS
node1   -        generic   Running   tcp://172.16.89.100:2376           v19.03.8   
node2   -        generic   Running   tcp://172.16.89.99:2376            v19.03.8   
```



正常输出即表示docker-machine已经把docker在所有node部署好了

登录到任意的node节点查看docker.service配置

```
vim /etc/systemd/system/docker.service.d/10-machine.conf
```

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock --storage-driver overlay2 --tlsverify --tlscacert /etc/docker/ca.pem --tlscert /etc/docker/server.pem --tlskey /etc/docker/server-key.pem --label provider=generic
Environment=
```

-H tcp://0.0.0.0:2376:使Docker Daemon接收远程连接，非docker-machie安装方式是没有这个选项的

--tls*：对来自远程主机的连接请求启用安全认证和加密措施



#### 更好的体验

为了得到更好的体验，可以安装bash completion script,从而在bash中使用tab补全命令。项目地址位于

https://github.com/docker/machine/tree/master/contrib/completion/bash

将页面上的三个脚本放到machine上/etc/bash_completion.d目录下，同时添加如下代码到$HOME/.bashrc中

```
source /etc/bash_completion.d/docker-machine-prompt.bash 
source /etc/bash_completion.d/docker-machine.bash 
source /etc/bash_completion.d/docker-machine-wrapper.bash
PS1='[\u@\h \W$(__docker_machine_ps1 " [%s]")]\$'
```



docker-machine env子命令可以提示操作node节点所需的环境变量

```
[root@docker-machine ~]# docker-machine env node1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://172.16.89.100:2376"
export DOCKER_CERT_PATH="/root/.docker/machine/machines/node1"
export DOCKER_MACHINE_NAME="node1"
# Run this command to configure your shell: 
# eval $(docker-machine env node1)
```



执行`eval $(docker-machine env node1)`就可以切换到node1上执行指令同时提示符也会变成node1

```
[root@docker-machine ~]# eval $(docker-machine env node1)
[root@docker-machine ~ [node1]]# 
```



在其上执行的所有docker命令搜相当于node1上执行

```
[root@docker-machine ~ [node1]]# docker pull nginx:alpine
alpine: Pulling from library/nginx
cbdbe7a5bc2a: Pull complete 
c554c602ff32: Pull complete 
Digest: sha256:763e7f0188e378fef0c761854552c70bbd817555dc4de029681a2e972e25e30e
Status: Downloaded newer image for nginx:alpine
docker.io/library/nginx:alpine
```

回到node1终端上，发现nginx:alpine镜像已下载，而且machine节点上是没有这个镜像的

```
root@node1:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               alpine              89ec9da68213        2 weeks ago         19.9MB
```

其他的docker命令都可以通过eval $(docker-machine env nodeX)上切换到相应的node节点执行docker子命令



### docker-machine子命令

| 子命令             | 效果                                 |
| ------------------ | ------------------------------------ |
| active             | 输出处于active状态的node节点         |
| config             | 查看machine的damon配置               |
| create             | 创建machine                          |
| env                | 显示node节点的环境变量               |
| ls                 | 显示node节点的状态信息               |
| stop/start/restart | 对node所在的操作系统执行开关重启操作 |
| scp                | 在node节点中复制数据                 |
| upgrade            | 将node节点上的docker升级到最新版本   |

```
docker-machine scp node1:/tmp/a.txt node2:/tmp/
```



### 总结

在多主机环境上使用docker-machine部署docker将大大提高效率。当然由于默认使用的国外的镜像源，安装docker会很慢，介意的话最好先用Ansible批量化安装docker再执行docker-machine将所有机器加入控制主机。


