# 一次calico-node无法启动的问题记录


集群中两个节点中的`calico-node`启动后一直处于`crashloopbackoff`的分析处理
<!--more-->

### 问题

生产集群中node-4及node-9两个节点的`calico-node`无法启动一直处于`crashloopbackoff`。`calicoctl node status`中的`INFO`列这两个节点显示为`Active Socket: Connection refused`。`kubectl describe`发现Event中有报错：

> 
> Liveness probe failed: Get http://127.0.0.1:9099/liveness: dial tcp 192.168.99.7:9099: getsockopt: connection refused
> 



### 追踪

---

尝试将两个pod重启

```bash
$ kubectl get pods -n kube-system --no-headers --field-selector=status.phase!=Running  | grep "calico-node" | awk '{print $1}' | xargs kubectl delete pod -n kube-system
```

重启完成后pod仍然处于`crashloopbackoff`状态，失败。

---

进入Github搜索相关issue，基本都是因为calico自动获取的ip不是目前在用的ip，但检查日志发现日志中获取的ip就是本地bond0网卡的地址。失败。

---

检查日志发现日志很短,并且两个节点的`calico-node`都是这种情况:执行到Using node name就结束了。与正常节点的`calico-node`启动日志对比，这部分日志并没有什么区别。

```
2020-12-30 12:50:55.337 [INFO][10] startup.go 256: Early log level set to info 
2020-12-30 12:50:55.337 [INFO][10] startup.go 272: Using NODENAME environment for node name 
2020-12-30 12:50:55.337 [INFO][10] startup.go 284: Determined node name: node-4
2020-12-30 12:50:55.436 [INFO][10] startup.go 97: Skipping datastore connection test 
2020-12-30 12:50:55.438 [INFO][10] startup.go 476: Using IPv4 address from environment: IP=172.16.200.103
2020-12-30 12:50:55.440 [INFO][10] startup.go 509: IPv4 address 172.16.200.103 discovered on interface bond0
2020-12-30 12:50:55.440 [INFO][10] startup.go 647: No AS number configured on node resource, using global value
2020-12-30 12:50:55.440 [INFO][10] startup.go 149: Setting NetworkUnavailable to False
2020-12-30 12:50:55.639 [INFO][10] startup.go 530: FELIX_IPV6SUPPORT is false through environment variable
2020-12-30 12:50:55.644 [INFO][10] startup.go 181: Using node name: node-4
```

---

**灵感**：在某个[github issue](https://github.com/projectcalico/calico/issues/4087)中提到虽然就绪探针`readinessProbe`不会导致`crashloopbackoff`,但是`livenessProbe`可能。检查calico-node的配置文件中有关探针的部分

```bash
$ kubectl get daemonsets.apps -n kube-system calico-node -o yaml
...
        livenessProbe:
          httpGet:
            host: 127.0.0.1
            path: /liveness
            port: 9099
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 6
          timeoutSeconds: 1
...
```

- liveness探针自pod启动后5s后开始探测
- 每隔10s进行一次探测
- 检查失败后进入成功状态最少连续探测次数为1次
- 检查成功后进入失败状态最少连续探测次数为6次
- 探测检测超时为1s

如果是配置文件有问题，那么所有的`calico-node`都会有问题，但现在只有node-4及node-9上的`calico-node`有问题。会不会是这两个节点异常导致`calico-node`启动后5s没有pod并没有启动完成，存活探针无法访问9099端口触发pod自动重启，不断反复导致pod处于`crashloopbackoff`？登录node-4及node-9发现两个节点的负载竟然比正常节点高了几十倍。原因就显而易见了：高负载导致pod启动缓慢，存活探针判断失败触发pod不断自动重启引发pod状态异常。



### 解决

驱逐node-4及node-9上的pod

```bash
$ kubectl drain node-4 --delete-local-data --ignore-daemonsets --force
$ kubectl drain node-9 --delete-local-data --ignore-daemonsets --force
```

重启两个节点

```bash
$ reboot
```

重启完成后两个节点上的`calico-node`处于`Running`状态

