---
title: Nginx添加Lua实现WAF防火墙
date: 2020-06-20 00:29:00
tags:
 - Nginx
categories:
 - Nginx
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/RedSailboat_ZH-CN2386102503_1920x1080.jpg
toc: true
disableToC: false
disableAutoCollapse: true
---

Web应用防护系统（也称：网站应用级入侵防御系统 。英文：Web Application Firewall，简称： WAF）。利用国际上公认的一种说法：Web应用 防火墙是通过执行一系列针对HTTP/HTTPS的 安全策略来专门为Web应用提供保护的一款产品。

<!--more-->



### nginx+lua安装方法

### 方法一：安装nginx并整合lua模块

#### 安装LuaJIT

LuaJIT的意思是Lua Just-In-Time，是即时的Lua代码解释器。必须去github下载否则运行是会出现报错，项目地址：https://github.com/openresty/luajit2

```shell
git clone https://github.com/openresty/luajit2.git
cd luajit2
make PREFIX=/usr/local/luajit
make install PREFIX=/usr/local/luajit
```

安装完成后将如下环境变量加入到`/etc/profile`中,并执行`source /etc/profile`

```shell
export LUAJIT_LIB=/usr/local/luajit/lib
export LUAJIT_INC=/usr/local/luajit/include/luajit-2.1
```



#### 安装ngx_devel_kit(NDK)

版本地址:https://github.com/vision5/ngx_devel_kit/tags

下载并解压

```shell
cd /mnt
wget https://github.com/vision5/ngx_devel_kit/archive/v0.3.1.tar.gz
tar -xzvf v0.3.1.tar.gz
```



#### 安装最新版的lua-nginx-module

版本地址：https://github.com/openresty/lua-nginx-module/tags

下载最新稳定版并解压

```shell
cd /mnt
wget https://github.com/openresty/lua-nginx-module/archive/v0.10.15.tar.gz
tar -xzvf v0.10.16rc5.tar.gz
```



#### 编译Nginx并加入lua模块

```shell
cd /mnt/nginx-1.18.0
./configure \ 
--prefix=/etc/nginx \
--sbin-path=/usr/sbin/nginx \
--modules-path=/usr/lib64/nginx/modules \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/run/nginx.lock \
--user=nginx \
--group=nginx \
--with-http_gzip_static_module \
--with-http_realip_module \
--with-http_ssl_module \
--with-openssl=/mnt/openssl-1.1.1g \
--with-zlib=/mnt/zlib-1.2.11 \
--with-pcre=/mnt/pcre-8.44 \
--add-module=/mnt/lua-nginx-module-0.10.15 \
--add-module=/mnt/ngx_devel_kit-0.3.1
```

> 其中的openssl,pcre以及zlib需要额外下载并解压到/mnt目录下



启动nginx发现报错

```
[root@k8s-node mnt]# nginx
nginx: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
```

解决办法

```
echo "/usr/local/luajit/lib/" >> /etc/ld.so.conf
ldconfig
```



在默认的location中加入如下指令，访问测试

```nginx
content_by_lua 'ngx.say("hello, lua")';
```

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/lua.png)安装成功



### 方法二 ：直接安装OpenResty



> OpenResty是一个基于Nginx与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。[最新版Openresty](http://openresty.org/en/download.html)



```shell
cd /opt 
tar -xzvf openresty-1.15.8.3.tar.gz
./configure \ 
--prefix=/opt/openresty \
--with-pcre=/opt/pcre-8.44 \
--with-zlib=/opt/zlib-1.2.11 \ 
--with-openssl=/opt/openssl-1.1.1g \
--with-poll_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_addition_module \
--with-stream \
--with-stream_ssl_module \
--with-stream_ssl_preread_module\
--with-http_ssl_module
```

```shell
gmake
gmake install
```

按方法一测试可以访问即可



### WAF模块安装

ngx_lua_waf项目地址：https://github.com/loveshell/ngx_lua_waf.git

这里以Openresty为例，Nginx方法类似

```
cd /opt/openresty/lualib
git clone https://github.com/loveshell/ngx_lua_waf.git waf
```

在openresty的配置文件中添加如下配置项：

```nginx
lua_package_path "/opt/openresty/lualib/waf/?.lua";
lua_shared_dict limit 10m;
init_by_lua_file  /opt/openresty/lualib/waf/init.lua;           
access_by_lua_file /opt/openresty/lualib/waf/waf.lua;
```

整个waf目录结构如下：

```shell
[root@k8s-node lualib]# tree waf
waf
├── config.lua
├── init.lua
├── wafconf
│   ├── args
│   ├── cookie
│   ├── post
│   ├── url
│   ├── user-agent
│   └── whiteurl
└── waf.lua

1 directory, 9 files
```

config.lua中定义了整个waf的配置，详细如下：

```shell
#拦截规则的存放目录
RulePath = "/opt/openresty/lualib/waf/wafconf/"
#是否开启拦截日志记录
attacklog = "on"
#拦截日志的记录目录，nginx的worker进程需要对该目录有权限
logdir = "/opt/openresty/nginx/logs/"
#是否开启url拦截
UrlDeny="on"
#是否开启拦截重定向
Redirect="on"
#是否开启cookie攻击防护
CookieMatch="on"
#是否开启post攻击防护
postMatch="on"
whiteModule="on"
#禁止访问的文件扩展名
black_fileExt={"php","jsp"}
#IP地址白名单
ipWhitelist={"127.0.0.1"}
#IP地址黑名单
ipBlocklist={"1.0.0.1"}
#是否开启CC攻击防护
CCDeny="off"
#定义CC攻击速率，该例为每60秒100次请求
CCrate="100/60"
#定义重定向后的html页面
html=[[...]]
```

拦截测试：

```shell
#出现拦截页面即表示安装成功
curl http://www.example.com/test.php?id=../etc/passwd
```

相关日志：

```
192.168.0.101 [2020-06-20 01:44:01] "GET localhost/index.php?id=/../../../etc/passwd" "-"  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36" "\.\./"
```



