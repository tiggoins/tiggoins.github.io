---
title: Jumpserver使用手册
date: 2022-05-10 23:50:47
tags:
 - Linux
categories:
 - Linux
toc: true
---

### 概述

堡垒机4A模型

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810150602504.png)
<!--more-->



### 安装

#### 需求配置

4核/16G,推荐200G以上硬盘。主要用来存储审计录像，审计录像每小时大约为10M，按每天操作4小时并且保留30天计算为127GB。



#### 虚拟机方式

直接安装官网链接:https://docs.jumpserver.org/zh/master/install/setup_by_fast/ -> 在线安装 -> 手动部署



#### k8s方式

官网链接：https://docs.jumpserver.org/zh/master/install/setup_by_fast/ -> 在线安装 -> helm

##### 部署外置MySQL

简化流程，采用单节点部署

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
  namespace: jumpserver
type: Opaque
data:
  rootpassword: "dl50WkRlYmNTQmhrRU5CIw=="
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-for-jumpserver
  namespace: jumpserver
  labels:
    app: mysql
spec:
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
  selector:
    app: mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: jumpserver
  labels:
    app: mysql
spec:
  storageClassName: rbd-immediate
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-for-jumpserver
  namespace: jumpserver
  labels:
    app: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: reg.kolla.org/library/mysql:5.7.39
        name: mysql
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - 'mysqladmin ping -p${MYSQL_ROOT_PASSWORD}'
          initialDelaySeconds: 30
          timeoutSeconds: 1
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - 'mysql -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"'
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 1
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh","-c","rm -rf /var/lib/mysql/lost+found"]
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: rootpassword
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

等Pod处于Ready状态后，进入Pod创建Database`jumpserver`:   `create database jumpserver default charset 'utf8' collate 'utf8_bin';`

##### 部署外置Redis

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-for-jumpserver-config
  namespace: jumpserver
data:
  redis.conf: |
    bind 0.0.0.0
    protected-mode yes
    tcp-backlog 1024
    requirepass "sw@1Wq(jN"
    timeout 0
    tcp-keepalive 300
    databases 16
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    rdb-del-sync-files no
    dir /var/lib/redis
    oom-score-adj no
    oom-score-adj-values 0 200 800
    disable-thp yes
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec
---
apiVersion: v1
kind: Service
metadata:
  name: redis-for-jumpserver
  namespace: jumpserver
  labels:
    app: redis
spec:
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
  selector:
    app: redis
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pv-claim
  namespace: jumpserver
  labels:
    app: redis
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-for-jumpserver
  namespace: jumpserver
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - image: reg.kolla.org/library/redis:7.0.4
        name: redis
        command: ["redis-server"]
        args: 
        - /opt/redis.conf
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: redis-persistent-storage
          mountPath: /var/lib/redis
        - name: config
          mountPath: "/opt"
      volumes:
      - name: redis-persistent-storage
        persistentVolumeClaim:
          claimName: redis-pv-claim
      - name: config
        configMap:
          name: redis-for-jumpserver-config
          items:
          - key: redis.conf
            path: redis.conf
```

##### 安装jumpserver

- 添加helm仓库:`helm repo add jumpserver https://jumpserver.github.io/helm-charts`

- 下载jumpserver的chart文件:`helm pull jumpserver/jumpserver --untar`

- 按需修改`values.yaml`
  1. 推送镜像到内网仓库
  2. 修改`ReadWriteMany`->`ReadWriteOnce`,如果storageclass支持多读多写可不做修改。审计视频存储于
  3. 按需添加资源限制
  4. ingress实现若不是使用的nginx-ingress，则按需修改

在`values.yaml`同级目录执行安装: `helm install jumpserver . -n jumpserver -f value.yaml`



### 使用

#### 创建用户

- 管理员：特权用户。创建各类账户、资产、MFA、授权等等，也可用于审计操作
- 普通用户：日常登陆到jumpserver进行资产访问的用户
- 审计员：对普通用户操作进行审计的专用账户

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810105807605.png)

点击创建,创建两个账户: 普通用户`testuser:testuser123`与审计员`testauditor:testauditor123`

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810110048556.png)



#### 创建特权用户

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810140745053.png)


> **特权用户** 是资产已存在的, 并且拥有 高级权限 的系统用户， 如 root 或 拥有 `NOPASSWD: ALL` sudo 权限的用户。 JumpServer 使用该用户来 `推送系统用户`、`获取资产硬件信息` 等。

管理用户：资产上的root用户或者是具有`NOPASSWD:ALL`权限的普通用户，jumpserver使用该用户创建**系统用户**或者获取硬件信息。

系统用户：在实际运维工作中，给使用人员所分配的登陆资产的系统用户，可由jumpserver自动推送，由jumpserver统一进行密码管理与更新。



#### 创建资产

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810142109547.png)




#### 创建授权

将用户与资产进行权限关联

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810144716275.png)

> 其中节点为一种类似于标签选择器的匹配方式，所有新建的节点均位于`Default`组下





#### 测试登陆

##### webterminal方式

使用新建的普通用户`testuser`登陆到jumpserver中,右上角有一个webterminal选项，点击后跳转到webterminal界面。由于在授权时仅授予了`172.16.200.123`一个节点，故在登陆后仅能看到这一台节点。

##### ssh方式

ssh方式需要将`koko`组件的`2222`端口暴露到公网，采用ingress或者nodeport方式。使用ssh testuser@[ip] -p [port]方式登陆

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810150116342.png)



#### 密码管理

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220810162758407.png)

#### 审计

审计可对日常的操作进行回放审计

登陆管理员或者审计员账户，切换到`审计台`视图进行操作审计

![image](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/jumpserver/image-20220811142708040.png)
