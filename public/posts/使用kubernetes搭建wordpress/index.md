# 使用Kubernetes搭建wordpress


本篇博客使用kubernetes搭建wordpress，旨在理解kubernetes各组件以及协作关系。
<!--more-->

### 创建数据库

```yaml
[root@k8s-master wordpress]# cat wordpress-database.yaml 
apiVersion: v1
kind: Service
metadata:
  name: wpdb
  labels:
    app: wpdb
spec:
  type: ClusterIP
  selector:
    app: wpdb
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
---    
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wpdb
  labels:
    app: wpdb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wpdb
  template:
    metadata:
      labels:
        app: wpdb
    spec:
      containers:
      - image: mysql:5.7.31
        imagePullPolicy: IfNotPresent
        name: wpdb
        env:
        - name: MYSQL_DATABASE
          value: wpdb
        - name: MYSQL_USER
          value: wpuser
        - name: MYSQL_PASSWORD
          value: poipoi@098
```

> 使用Deployment控制器创建pod资源，使用mysql:5.7.31镜像。并且传入了MYSQL_DATABASE、MYSQL_USER及MYSQL_PASSWORD三个变量。创建**Service**对象，将容器内的3306映射到**ClusterIP**的3306端口以供wordpress主程序访问。

### 创建wordpress

```yaml
[root@k8s-master wordpress]# cat wordpress.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: wordpress
  name: wordpress
spec:
  selector:
    app: wordpress
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - image: wordpress:5-php7.2
        name: wordpress
        env:
        - name: WORDPRESS_DB_NAME
          value: wpdb
        - name: WORDPRESS_DB_USER
          value: wpuser
        - name: WORDPRESS_DB_PASSWORD
          value: poipoi@098
        - name: WORDPRESS_DB_HOST
          value: wpdb.default.svc.cluster.local
```

> 依然采用Deployment控制器创建pod资源类型，使用wordpress:5-php7.2作为基础镜像。将连接数据库的变量传入。需要注意的是**wpdb.default.svc.cluster.local**为长格式域名，由于创建wordpress及数据库时未指明namespace，所以两个pod均在默认的namespace下创建，所以这里的域名可以直接用**wpdb**短格式域名。



上述deployment资源及service资源创建查看是否正常。

```shell
[root@k8s-master wordpress]# kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
wordpress-cd68468cb-x8sx6   1/1     Running   0          116m
wpdb-9c65c8bdc-chm6s        1/1     Running   0          127m
[root@k8s-master wordpress]# kubectl get services
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
wordpress    ClusterIP   10.110.1.27      <none>        80/TCP     126m
wpdb         ClusterIP   10.102.111.102   <none>        3306/TCP   127m
```

这部分结束，那么wordpress就已经创建完成了，可以在服务器上直接curl+ClusterIP:[port]进行访问了。

```shell
[root@k8s-master wordpress]# curl 10.110.1.27:80
<!DOCTYPE html>

<html class="no-js" lang="zh-CN">

        <head>

                <meta charset="UTF-8">
                <meta name="viewport" content="width=device-width, initial-scale=1.0" >

                <link rel="profile" href="https://gmpg.org/xfn/11">

                <title>雷探长博客 &#8211; Just another WordPress site</title>
<link rel='dns-prefetch' href='//www.test.com' />
<link rel='dns-prefetch' href='//s.w.org' />
```

### 创建Ingress Controller及默认的backend服务

上面创建的wordpress还只能在服务器内部访问，要想让外部用户访问必须创建ingress实现HTTP7层路由。使用Ingress创建负载分发时，ingress controller会基于ingress规则将客户端的请求直接转发到Service，跳过了kube-proxy组件的转发功能。

在定义ingress策略前，需要首先需要创建ingress controller及默认的backend服务。Ingress Controller为后端Service都提供了一个统一的入口。同时为了顺利启动Ingress Controller还需要配置默认的backend，用于客户端请求不存在的地址时，返回404应答。

#### 创建Ingress Controller

```yaml
[root@k8s-master wordpress]# cat ingress-daemonset.yaml 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-lb
  labels:
    name: nginx-ingress-lb
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nginx-ingress-lb
  template:
    metadata:
      labels:
        name: nginx-ingress-lb
    spec:
      containers:
      - image: registry.aliyuncs.com/google_containers/nginx-ingress-controller:0.9.0-beta.2
        name: nginx-ingress-lb
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        args:
          - /nginx-ingress-controller
          - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
```

> 这里为Nginx容器设置了hostPort，将容器的80和443分别映射到宿主机的80和443端口，这样客户端就可以通过访问http://物理机:80或https://物理机:443来访问该Ingress Controller。

#### 创建backend服务

```yaml
[root@k8s-master wordpress]# cat default-http-backend.yaml 
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kube-system
  labels: 
    k8s-app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
disableToC: false
disableAutoCollapse: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: default-http-backend
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      containers:
      - name: default-http-backend
        image: registry.aliyuncs.com/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
```

使用kubectl apply命令创建上述资源并查看是否正确运行

```
[root@k8s-master wordpress]# kubectl apply -f default-http-backend.yaml
deployment.apps/default-http-backend created
service/default-http-backend created
[root@k8s-master wordpress]# kubectl apply -f ingress-daemonset.yaml
daemonset.apps/nginx-ingress-lb created
```

> 创建上述资源时指定了名称空间，查看容器时需要带上`-n kube-system`参数

```shell
[root@k8s-master wordpress]# kubectl get pods -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
...
default-http-backend-797b95869d-4bl4k   1/1     Running   0          164m
nginx-ingress-lb-kcfws                  1/1     Running   0          163m
...
```

> 这里的backend服务采用了Deployment构建，并且声明了数量为1。Ingress Controller采用了DaemonSet构建，每个工作节点运行一个pod，我这里的环境是1+1，所以是1个pod。



部署完Ingress Controller及backend服务后，就可以访问任一工作节点的80端口访问，得到404说明部署成功。

```shell
[root@k8s-master wordpress]# curl k8s-node
default backend - 404
```

### 定义Ingress策略

这里使用www.test.com 来设置Ingress策略，定义对**/**的访问请求转发到后端的wordpress的规则。

```yaml
[root@k8s-master wordpress]# cat ingress-wordpress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wordpress-ingress
spec:
  rules:
  - host: www.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: wordpress
          servicePort: 80

```

这里的ServiceName和ServicePort是之前创建的wordpress service对象的参数。

> 需要注意的是这里的80端口和Service对象的80一样是虚拟的，本机并不会监听80端口。

```
[root@k8s-master wordpress]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
...
wordpress    ClusterIP   10.110.1.27      <none>        80/TCP     171m
...
```

创建ingress策略

```shell
[root@k8s-master wordpress]# kubectl apply -f ingress-wordpress.yaml 
ingress.extensions/wordpress-ingress configured
```

到这里创建wordpress就结束了，要访问wordpress需要在你的电脑的host文件上做域名ip关联。将www.test.com关联到任一工作节点的ip,使用浏览器访问www.test.com即可。

```
C:\Windows\System32\drivers\etc>more hosts
192.168.0.107 www.test.com
```

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/k8s-wordpress.png)

安装完成


