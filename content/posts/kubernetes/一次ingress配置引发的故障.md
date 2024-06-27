---
title: 一次Ingress配置引发的故障
date: 2024-06-27 07:09:08
tags:
 - Kubernetes
 - TroubleShooting
categories:
 - Kubernetes
 - TroubleShooting
toc: true
---

本文介绍了一次由ingress配置引发数个业务无法访问的问题，虽发生在测试环境但也起到了很好的提醒作用。

<!--more-->

版本信息：

- Ingress-Nginx：v0.34.0

- Nginx: 1.19.1

  

### 背景

6月26日接客户反馈，测试环境部分Ingress资源在变更后无法访问，错误代码502，未更新的Ingress访问正常。

#### 架构概述

集群中Ingress架构为三个keepalived+ingress，分布在node-1、node-11及node-21节点，外部访问的流量主要从keepalived主节点的`ingress-controller`实例接入到集群中。问题出现时keepalived主节点位于node-1节点上。



### 排查

#### 初步排查

1. 检查node-1节点内核日志及系统日志未发现明显异常；
2. 检查`kube-system`命名空间下pod工作状态，发现除了node-4节点上的kube-proxy为pending外无明显异常；
3. 查看node-1节点上的ingress实例日志，发现大量类似`Connect(): Connection Refused`的错误；
4. 在node-1节点上使用curl直接访问日志中某个报错service，访问正常。

为进一步确认，手动将node-21节点keepalived的权重调高将虚拟地址绑定到node-21节点上，发现问题依然存在。至此，问题基本可以确定是ingress本身的故障导致的外部无法访问，



#### 日志分析

继续分析ingress-controller的日志，发现频繁出现如下的报错：

```nginx
Error: exit status 1
2024/06/26 16:03:59 [emerge] 31831#31831: "proxy_max_temp_file_size" must be equal to zero to disable temporary files usage or must be equal to or greater than maximum of the value of "proxy_buffer_size" and one of the "proxy_buffers" in /tmp/nginx-cfg272696236:128257
nginx: [emerge] "proxy_max_temp_file_size" must be equal to zero to disable  temporary files usage or must be equal to or greater than maximum of the value of "proxy_buffer_size" and one of the "proxy_buffers" in /tmp/nginx-cfg272696236:128257
nginx: configuration file /tmp/nginx-cfg272696236:128257 test failed
```

由此可以判断是`proxy_max_temp_file_size`的值非0，或者是这个值小于了`proxy_buffer_size`或者`proxy_buffers`的值。而`proxy_max_temp_file_size`的大小默认为1024m。

```go
func NewDefault() Configuration {
	...
	cfg := Configuration{
		...
			ProxyMaxTempFileSize:        "1024m",
		...
```

分析到此，可以通过将`proxy_max_temp_file_size`增大或者直接设置为0取消对临时文件的缓冲。但在查看社区文档时发现这个值不在可配置的控制器[configmap](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#configmaps)范围内，而只在各个ingress的[annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#proxy-max-temp-file-size)配置中，考虑到集群中ingress有1000+个，一个个配置ingress显然不合适。遂打算将`NewDefault`中的`ProxyMaxTempFileSize`改为0后重新构建镜像后替换上线。**后面发现这个值虽不在可配置的列表内，但是配置上去确实有效的，这点着实让人没想到😂**。



### 解决

解决的方法比较简单，修改代码中的默认`ProxyMaxTempFileSize`值为0然后重新构建镜像替换上线，这里列出主要的步骤：

```bash
# git clone https://github.com/kubernetes/ingress-nginx 
# git checkout controller-v0.34.0
# sed -i 's/ProxyMaxTempFileSize:        "1024m"/ProxyMaxTempFileSize:        "0"/g'  \
	./internal/ingress/controller/config/config.go
# make build
# make image
```

这里的`make build`由于网络原因拉取镜像缓慢导致构建多次失败，无奈只能修改为本地编译

```bash
# export PKG=k8s.io/ingress-nginx
# export ARCH=amd64
# export GIT_COMMIT=git-6c73d66ae
# export REPO_INFO=https://github.com/kubernetes/ingress-nginx
# export TAG=v0.34.0
# ./build/build.sh
```

将镜像导出替换线上环境后各个业务的访问均恢复正常。



### 深入排查

将ingress-controller中出错的临时`nginx.conf`拷贝出来，搜索`proxy_max_temp_file_size`关键字发现均为1024m，而搜索`proxy_buffer_size`时发现有三个localtion都是2000m，并且同属一个ingress资源，并且其`proxy_buffer`的配置为`4k  2000m`。这一步应该一开始就能排查出来，只不过将正常加载的、正在使用的`nginx.conf`作为了排查依据，并且在排查ingress资源时只排查了`proxy_max_temp_file_size`：`kubectl get ingress -o yaml | grep "proxy_max_temp_file_size"`。😑

以上，就可以判断是由于该ingress中自行配置的`proxy_buffer_size`及`proxy_buffer`值大于了ingress默认的的`proxy_max_temp_file_size`，导致nginx在热加载新的conf时频繁的崩溃，后续所有更新的ingress都受此影响无法访问。为了验证这点，在集群中找到了这个ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    meta.helm.sh/release-name: xxxxxxxxxxxxxxxxxxx
    meta.helm.sh/release-namespace: xxxxxxxx
    nginx.ingress.kubernetes.io/affinity: cookie
    nginx.ingress.kubernetes.io/proxy-body-size: 2000M
    nginx.ingress.kubernetes.io/proxy-buffer-size: 2000M
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "6000"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "6000"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "6000"
    nginx.ingress.kubernetes.io/session-cookie-hash: sha1
    nginx.ingress.kubernetes.io/session-cookie-name: INGRESSCOOKIE
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/upstream-hash-by: sticky
  creationTimestamp: "2023-12-26T04:15:44Z"
  generation: 4
  labels:
    app.kubernetes.io/managed-by: Helm
  managedFields:
  - apiVersion: extensions/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:meta.helm.sh/release-name: {}
          f:meta.helm.sh/release-namespace: {}
          f:nginx.ingress.kubernetes.io/affinity: {}
          f:nginx.ingress.kubernetes.io/proxy-body-size: {}
          f:nginx.ingress.kubernetes.io/proxy-buffer-size: {}
          f:nginx.ingress.kubernetes.io/proxy-connect-timeout: {}
          f:nginx.ingress.kubernetes.io/proxy-read-timeout: {}
          f:nginx.ingress.kubernetes.io/proxy-send-timeout: {}
          f:nginx.ingress.kubernetes.io/session-cookie-hash: {}
          f:nginx.ingress.kubernetes.io/session-cookie-name: {}
          f:nginx.ingress.kubernetes.io/ssl-redirect: {}
          f:nginx.ingress.kubernetes.io/upstream-hash-by: {}
        f:labels:
          .: {}
          f:app.kubernetes.io/managed-by: {}
      f:spec:
        f:rules: {}
    manager: helm
    operation: Update
    time: "2024-06-25T08:15:53Z"
```

该ingress创建于`2023-12-26`并且在`2024-06-25`使用helm更新了`nginx.ingress.kubernetes.io/proxy-buffer-size`的值为2000M。

nginx错误产生的具体位置如下，省去了其他部分：

```c
// src/http/modules/ngx_http_proxy_module.c -- ngx_http_proxy_merge_loc_conf

size = conf->upstream.buffer_size;     // size = proxy_buffer_size
if (size < conf->upstream.bufs.size) { // 若size < proxy_buffers.size;则size = proxy_buffers.size
	size = conf->upstream.bufs.size;
}
...
if (conf->upstream.max_temp_file_size_conf == NGX_CONF_UNSET_SIZE) {
	conf->upstream.max_temp_file_size = 1024 * 1024 * 1024; // proxy_max_temp_file_size 未显式设置时为1G
} else {
	conf->upstream.max_temp_file_size = conf->upstream.max_temp_file_size_conf;
}

if (conf->upstream.max_temp_file_size != 0
        && conf->upstream.max_temp_file_size < size) {  
    // (proxy_max_temp_file_size < proxy_buffer_size || proxy_max_temp_file_size < proxy_buffers.size)nginx崩溃
	ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
	             "\"proxy_max_temp_file_size\" must be equal to zero to disable "
	             "temporary files usage or must be equal to or greater than "
	             "the maximum of the value of \"proxy_buffer_size\" and "
	             "one of the \"proxy_buffers\"");
	return NGX_CONF_ERROR;
}
```



### 监控与预警

好在故障只是发生在测试环境，但也惊出一身冷汗：如果业务人员创建或更新的ingress资源的`Annotation`与控制器的默认配置存在冲突引发ingress重载配置失败，那么后续更新的所有ingress都将无法访问，必须借助监控手段及时发现问题，经查阅文档，发现有指标可以监控到这个错误：

```
# HELP nginx_ingress_controller_config_last_reload_successful Whether the last configuration reload attempt was successful
# TYPE nginx_ingress_controller_config_last_reload_successful gauge
```

告警规则交给ChatGPT：

```yaml
groups:
- name: nginx_ingress_alerts
  rules:
  - alert: NginxIngressControllerReloadFailure
    expr: nginx_ingress_controller_config_last_reload_successful == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Nginx Ingress Controller Reload Failure"
      description: "The Nginx Ingress Controller has failed to reload its configuration. (instance {{ $labels.instance }})"
```
完！
