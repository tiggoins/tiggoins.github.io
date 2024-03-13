---
title: Nginx-Limit模块
date: 2020-05-05 08:27:44
tags:
 - Nginx
categories:
 - Nginx
toc: true
disableToC: false
disableAutoCollapse: true
---

Nginx的 **limit** 模块主要包括:`ngx_http_limit_req_module`、`ngx_http_limit_conn_module`、`ngx_stream_limit_conn_module` 
<!--more-->

以及`ngx_http_core_module`中limit_rate选项，由于stream主要用来实现四层协议（网络层和传输层）的转发、代理、负载均衡等，并且`ngx_stream_limit_conn_module`和`ngx_http_limit_conn_module`配置基本相同，所以本文不做介绍。



### ngx_http_limit_req_module

该模块用来限制某个特定键的请求处理的速率，这个特定键一般为某个ip。该模块主要用到了"漏桶"算法。

#### 指令

 ```nginx
 limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

 server {
    location /search/ {
        limit_req zone=one burst=5 [nodelay|delay=num];
    }
 ```

首先使用`limit_req_zone`定义一个内存区域(<font color=#FF4500 size=3>定义在http段中</font>)：

**$binary_remote_addr**:特定用于该模块的变量，用来存取客户端ip地址。

**zone=one:10m**：定义一个大小为10m的共享内存区域，名称为*one*。对于ipv4地址占用4个字节，ipv6占用16个字节。存储一个连接状态在32位机器上占用64个字节，在64位机器上占用128个字节。1MB的空间可以存储16k个64字节状态或者是8k个128字节状态。所以10m的空间可以存储大约8w个状态，对于一般的中小型站点已经足够了。

**rate=1r/s**:定义请求速率为每秒仅接受一个请求

在server配置段中引用上述配置(第5行)：

**zone=one**:引用上面`limit_req_zone`定义的内存区域*one*

**burst**:定义请求缓存的长度，意思是如果某ip1秒内发来了10个请求，那么除了正在处理的1个请求，其他的请求会把暂时放置到burst中排队。超过`rate+burst`请求将会被全部丢弃

**nodelay**:一次处理*burst+rate*个请求，其余全部丢弃。如果不希望在请求受到限制时延迟过多的请求应当使用这个参数

**delay=num**:瞬时处理`num+rate`个请求，总共缓存burst个，剩余的`burst-num`按rate定义处理。丢弃多余请求。



#### 其他指令

##### limit_req_dry_run

```nginx
语法：   limit_req_dry_run on | off
默认值： limit_req_dry_run off;
语境：   http server location
```

该指令开启"干跑"模式。开启该配置，请求处理的速率不再被限制。但是在共享内存区域，过量的请求的数值会像之前一样计算。



##### limit_req_log_level

```nginx
语法:	  limit_req_log_level info | notice | warn | error;
默认值: limit_req_log_level error;
语境:	  http server location
```

该指令设置`rate`超过限值或者延时请求处理的日志级别，默认为error级别。



##### limit_req_status

```nginx
语法:	  limit_req_status code;
默认值:  limit_req_status 503;
语境:	   http server location
```

该指令定义拒绝响应请求的http状态码，默认返回*503



#### 测试

1、不开启burst，不开启nodelay

配置如下所示：

```nginx
 http {
    include       mime.types;
    default_type  application/octet-stream;

    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

    log_format  main  '$remote_addr "$request"'
                      '$status'
                      '"$http_user_agent"';

    server {
        listen       80;
        server_name  localhost;

        charset utf-8;

        location / {
            limit_req zone=one;
            root   /usr/share/nginx/html/;
            index  index.html index.htm;
        }

    }

}
```

在另一台虚拟机使用Apahce Benchmark进行压力测试

```
ab -c 10 -n10 http://192.168.0.106/index.html
```

结果如下：

```
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:20:58:41 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
```

严格按照`rate`定义处理请求，除了第一个请求外其余所有的请求全部丢弃并返回503。

2、在18行末尾添加burst=5

```
192.168.0.107 - - [05/May/2020:21:53:35 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:35 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:35 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:35 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:35 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:36 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:37 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:38 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:39 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:53:40 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
```

按照定义，1秒内来了10个请求，burst=5，rate=1。总共缓存5个请求，处理1个请求，其余全部丢弃。这就是21:53:35内丢弃了4个请求。随后缓存的请求按照`rate`定义值处理。

3、在18行末尾继续添加nodelay

```
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:21:58:19 +0800] "GET /index.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
```

瞬间提供`rate+burst`个请求处理能力，丢弃其他请求返回503。

4、将18行的nodelay改成delay=3

```
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:05 +0800] "GET / HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:06 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:42:07 +0800] "GET / HTTP/1.0" 200 70 "-" "ApacheBench/2.3"
```

可以看出nginx瞬间处理了4(rate+delay)个请求，缓存了2个请求，其余请求返回503。缓存下来的请求每秒处理一个。



### ngx_http_limit_conn_module

ngx_http_limit_conn_module与ngx_http_limit_req_module主要区别在于，`conn`限制的连接的数量，`req`限制的是请求的数量。一个连接的生命周期中，会存在一个或者多个请求，这是为了加快效率，避免每次请求都要三次握手建立连接，现在的HTTP/1.1协议都支持这种特性，叫做keepalive。

官方配置示例：

```nginx
 http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    server {
        
        location /download/ {
           limit_conn addr 1;
        }
```

limit_conn_zone：设置共享区域的参数，该区域将保留各种键的状态，与`ngx_http_limit_req_module`不同的是，此模块仅定义了区域了大小。键里可以保存文本，变量或者两者的组合。

limit_conn:设置共享内存区域以及键定义的最大允许的连接数量。超过限值时，服务器将会发送错误给客户端。`limit_conn addr 1`允许一个ip一次最多建立一次连接。如果是`HTTP/2`，那么每个并发的请求都会被视作是一个单独的连接。

**limit_conn_dry_run**,**limit_conn_log_level**,**limit_conn_status**与`ngx_http_limit_req_module`中一致不再说明。

#### 测试

```
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:27 +0800] "GET /test.html HTTP/1.0" 503 197 "-" "ApacheBench/2.3"
192.168.0.107 - - [05/May/2020:23:54:29 +0800] "GET /test.html HTTP/1.0" 200 536870912 "-" "ApacheBench/2.3"
```

test.html为50M，测试1秒内仅能建立一个连接其余全部丢弃。但index.html测试失败，不知道和文件的大小存在什么关联？



### limit_rate

#### 指令

limit_rate是ngx_http_core_module这个核心模块自带的一个配置选项。可以用来限制单个连接的下载速率。对于资源文件下载服务器来说，有必要限制这个值，防止单个ip过高的速率影响他人的正常使用。

```
语法:	   limit_rate rate;
默认值:  limit_rate 0;
语境:	   http, server, location, if in location
```

默认情况下，这个值是*0*，也就是不限制下载速率。官方推荐使用map来灵活定义下载速率。如：

```
map $slow $rate {
    0     40k;
    1     80k;
    default 120k;
}
limit_rate $rate; #http，server，location，if in location
```

当然**$slow**不是nginx自带的变量，所以直接使用会报错。需要根据ngx_http_geo_module模块自带的geo指令来定义$slow变量。

```
geo $remote_addr $slow {
	default 0;
	47.103.215.250 1;
}
```

以上是这样的过程:`$remote_addr`是nginx自带的变量，geo指令拿到这个变量对应的ip地址去和下面的`default`或`47.103.215.250`匹配，如果满足`$remote_addr=47.103.215.250`,则把$slow变量设置为1,否则就设置为0。再拿0或者1去map里匹配得到`$rate`,最后在limit_rate中应用$rate来达到限制速率的要求。

> 注：geo 和 map都是应用在http段中

#### 测试

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/limit_rate测试.png)

​                                           **本地下载测试为40KB/s**

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/20200504085807.png)

​                                     **47.103.215.250上测试为80KB/s**

当然limit_rate只能限制单个ip的一个请求：

> The limit is set per a request, and so if a client simultaneously opens two connections, the overall rate will be twice as much as the specified limit.

如果客户端同时发起了两个链接，这个下载速度会变成限值的2倍。所以对于迅雷这种多线程的下载器，设置这样的限值的无效的。

解决的方法就是配合`ngx_http_limit_conn_module`模块，让服务器每秒只响应一个ip的一个请求，其余请求全部丢弃。

```nginx
    limit_conn_zone $binary_remote_addr zone=one:10m;
    server {
        listen       80;
        server_name  localhost;
        
        location / {
            limit_conn one 1;
            limit_rate 80k;
            root   /usr/share/nginx/html/;
        }

    }
```

在测试端下载test.html

```
[root@k8s-node ~]# wget http://192.168.0.106/test.html
--2020-05-06 00:00:43--  http://192.168.0.106/test.html
Connecting to 192.168.0.106:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 536870912 (512M) [text/html]
Saving to: ‘test.html.2’

test.html.2                        
0%[                                                    ]   1.29M  79.6KB/s    eta 1h 48m 
```

复制终端会话再次请求：

```
Last login: Tue May  5 23:40:24 2020 from 192.168.0.101
[root@k8s-node ~]# wget http://192.168.0.106/test.html
--2020-05-06 00:01:18--  http://192.168.0.106/test.html
Connecting to 192.168.0.106:80... connected.
HTTP request sent, awaiting response... 503 Service Temporarily Unavailable
2020-05-06 00:01:19 ERROR 503: Service Temporarily Unavailable.
```

nginx直接返回了503，这样就达到了限制单个ip的请求速率的目的。
