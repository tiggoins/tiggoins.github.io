# Kubernetes之RBAC鉴权


RBAC(Role Based Access Control 基于角色的访问控制)。在使用kubeadm部署kubernetes集群时作为默认的鉴权模式开启。二进制部署时需指明api-server的启动参数`--authorization-mode=RBAC`来开启RBAC认证。

<!--more-->

### 资源对象说明

RBAC定义了四种顶级资源来定义权限

- role
- clusterrole
- rolebinding
- clusterrolebinding

role及rolebinding是在名称空间内的，仅能对指定的namespace生效。而clusterrole和clusterrolebinding是集群范围内的，可以对整个集群范围内的资源进行授权。

> 定义的权限都是允许权限，因为k8s默认拒绝所有权限。



#### Role(角色)

Role(角色)是一组权限定义。比如创建一个test-namespace中的pod-reader角色

```yaml
[root@k8s-master ~]# kubectl create role pod-reader --verb=get,list,watch \
--resource=pods,pods/status --namespace=test-namespace \
--dry-run=client -o yaml > pod-reader.yml

[root@k8s-master ~]# cat pod-reader.yml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: test-namespace
rules:
- apiGroups: [""]
  resources: ["pods","pods/status"]
  verbs: ["get","list","watch"]
```

该角色表明test-namespace命名空间中的pod-reader角色拥有对test-namespace命名空间中所有pod的get，list及watch权限。

其中，rules中定义的三项为最重要的内容：

- apiGroups：即api资源组,下表即为当前Kubernetes版本支持的资源组及其下资源名称。

```yaml
[root@k8s-master ~]# kubectl api-resources 
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
pods                              po                                          true         Pod
podtemplates                                                                  true         PodTemplate
replicationcontrollers            rc                                          true         ReplicationController
resourcequotas                    quota                                       true         ResourceQuota
secrets                                                                       true         Secret
serviceaccounts                   sa                                          true         ServiceAccount
services                          svc                                         true         Service
...
deployments                       deploy       apps                           true         Deployment
replicasets                       rs           apps                           true         ReplicaSet
statefulsets                      sts          apps                           true         StatefulSet
tokenreviews                                   authentication.k8s.io          false        TokenReview
...
```

apiGroups是根据resources来确定的，这里是访问pods的权限，所以apiGroup为空值代表核心组。其他的比如对deployments的权限，apiGroup就是apps，以此类推...

- resources：为上表中的NAME字段。也可为其子资源，比如pod中的日志等。
- verbs：定义的是该角色所具有的具体权限，如get、watch、list、patch、delete等。

> 创建role时都需要写明namespace，如果创建的role属于default名称空间时最好也能带上`namespace: default`以养成习惯。



#### ClusterRole(集群角色)

相对于role仅对默认的名称空间有效外，clusterrole是集群范围内的权限,所以无需定义namespace。集群角色不仅能访问集群范围的资源，比如Node。同时也包含非资源型路径，如"/healthz"。按上面的pod-reader的列子类比

```yaml
[root@k8s-master ~]# kubectl create clusterrole pod-cluster-reader \
--verb=get,list,watch --resource=pods,pods/status \
--dry-run=client -o yaml > pod-cluster-reader.yml   

[root@k8s-master ~]# cat pod-cluster-reader.yml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-cluster-reader
rules:
- apiGroups: [""]
  resources: ["pods","pods/status"]
  verbs: ["get","list","watch"]
```

该集权角色拥有全部命名空间下pod的get、list及watch权限。



#### RoleBinding(角色绑定)

Rolebinding(角色绑定)的意思是将用户与角色关联起来，让用户去扮演角色。从而使用户获得角色中定义的权限。如下面的定义表示，将用户jane去扮演pod-reader这个角色，而pod-reader按此前的定义拥有对test-namespace中pod资源的get、list及watch权限。那么在应用了这个角色绑定之后，jane用户就有权限访问test-namespace中的pod资源了。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: test-namespace
subjects:
- kind: User
  name: jane 
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader 
  apiGroup: rbac.authorization.k8s.io
```

> 当创建一个rolebinding之后，其中的roleRef字段中的定义内容是不可变的，如果要改变roleRef中的内容，必须删除该rolebinding并重新创建。




![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/kubernetes/rolebinding.png)

从上图可以看出，rolebinding起到了媒介的作用，让用户和角色关联起来，使用户能够扮演role，从而获得role的权限，进而能够管理pod。



rolebinding可以绑定clusterrole。实现的效果是仅需定义一个集权范围的角色clusterrole，在具体的namespace中用rolebinding去绑定。

这样做的目的是能让角色进行复用，避免在各自的名称空间中重复声明role。

比如这个官方的示例：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

集群角色secret-reader具有读取全部名称空间下secret的权限，但由于rolebinding只在development名称空间下，所以只能读development命名空间内的secret。同样的也能拿到test-namespace命名空间中绑定secret-reader以让用户能读取test-namespace命名空间中的secret。





#### ClusterRoleBinding(集群角色绑定)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-cluster-pods
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-reader 
  apiGroup: rbac.authorization.k8s.io
```

相比于rolebinding可以绑定role及clusterrole，clusterrolebing只能绑定clusterrole，上面的授权表明jane用户可以读取全部集群范围内的所有pod的信息。



#### 常见的rule示例

这些例子都是来自kubernetes官方站点，你可以在这里找到他们：

[Using RBAC Authorization | Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

1、聚合角色

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

该角色拥有pod的get，list，watch权限。同时也拥有batch、extensions两个API组的jobs的get，list等权限。

2、资源名称限定

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

该声明仅允许使用get方法访问名为my-config的configmap，注意要和rolebinding绑定以限制configmap到特定的namespace中去。

3、非资源类型

```yaml
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"] 
  verbs: ["get", "post"]
```

该声明允许使用get和post方法访问/health及其下的子资源，由于是非资源类型，所以必须结合clusterrole和clusterrolebinding使用。



#### 常见的subject示例

1、用户名："alice@example.com"

```yaml
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
```

2、frontend-admins用户组

```yaml
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
```

3、test名称空间下的所有service accounts

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:test
  apiGroup: rbac.authorization.k8s.io
```

4、所有的service accounts账户

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```



### 关于Service Account

在rolebinding或者是clusterrolebing中的subject字段是用来定义用户的。在kubernetes中有两种用户：

- 普通用户，一般在指集群外操作集群的人，用kubeconfig配置文件进行鉴权
- serviceaccount，集群内部定义的用来给pod访问api-server鉴权时所使用的认证账户

区别在与:普通用户不是由kubernetes管理，而serviceaccount属于一种kubernetes资源类型，并且是属于某个namespace的。



一般而言，每个名称空间内都有一个默认的serviceaccount账户。并且如果手动删除这个serviceaccount账户，kubernetes还会自动创建出来。

```yaml
[root@master-1 ~]# kubectl delete sa default  #sa即为serviceaccount的缩写
serviceaccount "default" deleted
[root@master-1 ~]# kubectl get sa           
NAME      SECRETS   AGE
default   1         2s
```

创建service account的语法很简单,比如创建kube-system名称空间下的test账号：

```yaml
kubectl create serviceaccount test --namespace=kube-system
```

一个serviceaccount会包含多个secret文件，以应对特定场景下的加密需求。

```shell
[root@k8s-master ~]# kubectl describe serviceaccount default
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-djqcv
Tokens:              default-token-djqcv
Events:              <none>
```

1）名为Tokens的secret是用于访问API Server的Secret

2）imagePullSecrets是用于下载镜像文件的认证secret

默认情况下，如果pod在创建时没有指明serviceAccount账户名，那么默认的账户default会被挂在到pod内。

```yaml
[root@master-1 ~]# kubectl get pods nginx-deployment-699fc59b58-229wj -o yaml | grep serviceAccount
serviceAccount: default
serviceAccountName: default
```

当然也可以在.spec.serviceAccountName指明要用的serviceaccount

#### 对service accout的授权管理

1、关联my-namespace中的my-sa账户，使其在my-namespace中拥有只读权限

```yaml
kubectl create rolebinding my-sa-view \
  --clusterrole=view \
  --serviceaccount=my-namespace:my-sa \
  --namespace=my-namespace
```

2、给my-namespace下所有的service account授予只读权限

```yaml
kubectl create rolebinding serviceaccounts-view \
  --clusterrole=view \
  --group=system:serviceaccounts:my-namespace \
  --namespace=my-namespace
```

> serviceaccount的规范写法是<namespace>:<serviceaccountname>

3、给所有的service account授予只读权限

```yaml
kubectl create clusterrolebinding serviceaccounts-view \
  --clusterrole=view \
 --group=system:serviceaccounts
```

4、给所有的service account授予超级用户权限（强烈不推荐）

```
kubectl create clusterrolebinding serviceaccounts-cluster-admin \
  --clusterrole=cluster-admin \
  --group=system:serviceaccounts
```



### Dashboard应用授权

> Dashboard 是基于网页的 Kubernetes 用户界面。 你可以使用 Dashboard 将容器应用部署到 Kubernetes 集群中，也可以对容器应用排错，还能管理集群资源。 你可以使用 Dashboard 获取运行在集群中的应用的概览信息，也可以创建或者修改 Kubernetes 资源 （如 Deployment，Job，DaemonSet 等等）。 

官网地址：[GitHub - kubernetes/dashboard: General-purpose web UI for Kubernetes clusters](https://github.com/kubernetes/dashboard)

安装方法很简单：

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

为了使dashboard能在集群外访问，需修改其中的Service定义，将类型改成NodePort

```yaml
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort        #增加类型为NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 32443   #指定nodePort的端口为32443
  selector:
    k8s-app: kubernetes-dashboard
```

重点看下授权部分的配置：

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---
```

从上面可以看出，首先创建了service account账户kubernetes-dashboard，定义了role和clusterrole的权限。最后通过rolebinding及clusterrolebinding将kubernetes-dashboard这个service account账户关联起来。



安装完成后拿到默认的secret中的token去登录dashboard

```shell
[root@k8s-master dashboard]# kubectl get sa -n kubernetes-dashboard
NAME                   SECRETS   AGE
default                1         99s
kubernetes-dashboard   1         99s

[root@k8s-master dashboard]# kubectl describe sa kubernetes-dashboard -n kubernetes-dashboard
Name:                kubernetes-dashboard
Namespace:           kubernetes-dashboard
Labels:              k8s-app=kubernetes-dashboard
Annotations:         Image pull secrets:  <none>
Mountable secrets:   kubernetes-dashboard-token-tl6l9
Tokens:              kubernetes-dashboard-token-tl6l9
Events:              <none>

[root@k8s-master dashboard]# kubectl describe secret kubernetes-dashboard-token-tl6l9 -n kubernetes-dashboard
Name:         kubernetes-dashboard-token-tl6l9
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
              kubernetes.io/service-account.uid: aa1c5a89-263c-47ad-9e96-af159766987a

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImpHMF94T19xZ2dVMWdCVHVIOFVwa0RmNVotWThEWkYxMzItckIya3NwckEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi10bDZsOSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFhMWM1YTg5LTI2M2MtNDdhZC05ZTk2LWFmMTU5NzY2OTg3YSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.GghxWgleay-YuTPfTK8OhFKRAgtZJiCrCmnpsiQvbCx8GQtkmxkyAE5dfrKQZnqdttt2nSx634uMwm-WZIRrTqHD_uctx4eySiY1YfKxlCegoOkGQhZFjOv6lMSC27Qrrw3oy-rMYkMutxT59ThPmwYSHUQs6Jpd12iO6IFFFSIskZTzy1JbMfVDMF7Yf8EKvjqRH_Yj562KWEubOmY_C8dl2LvwVBxKIwBumTPQ56lLDJGvHLohkbBQQeNifLVoan58Xm1KwGOGwvI11fs6j_5SiphAxvRW4gPXfeZwMHOYxT8FHZftsckRKzvckVFVpoUnQsWNjsrfb6lu9X9SDg
```

登录后会发现所有的内容无法显示，查看右上角的提示发现都是无权限，这是因为官网给出的配置文件中只给出了services、configmaps、secrets的权限并且限定了具体的资源名称。这时就需要我们自己定义dashboard的权限以正常访问。

![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/kubernetes/dashboard/dashboard-unauthenticated.png)

dashboard支持token及kubeconfig两种验证方式进行登录。

#### Token登录

为了方便起见，这里直接使用cluster-admin这个clusterrole，这个clusterrole拥有集群下的所有权限。生产环境建议自定义小权限role。

```yaml
[root@k8s-master ~]# kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

1、创建名为dashboard-admin的serviceaccount

```shell
kubectl create serviceaccount dashboard-admin -n default
```

创建完成后，系统会自动创建与此对应的secret：

```shell
[root@k8s-master ~]# kubectl get secret | grep dashboard
NAME                          TYPE                                  DATA   AGE
dashboard-admin-token-c774m   kubernetes.io/service-account-token   3      43s
```



2、创建clusterrolebinding将cluster-admin及dashboard-admin这个serviceaccount进行关联

```shell
kubectl create clusterrolebinding dashboard-admin \
--clusterrole=cluster-admin --serviceaccount=default:dashboard-admin
```



3、最后获取这个serviceaccount对应的secret:dashboard-admin-token-c774m中的token值即可登录进dashboard。该secret拥有全部资源的全部权限。生产环境切勿使用。

```
kubectl describe secret dashboard-admin-token-c774m
```



#### Kubeconfig登录

相比与token每次都要手动获取，kubeconfig文件登录仅需登录时选中就能登录，更加便捷。但是步骤比token登录要复杂许多。

还记得部署kubernetes集群中最后三个步骤么？

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

打开$HOME/.kube./config,其中定义了三个关键的字段：clusters、context、user

```shell
[root@k8s-master ~]# cat ~/.kube/config 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ---
    server: https://192.168.0.106:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: ---
    client-key-data: ---
```

- clusters：定义了集群的访问地址，6443为api-server的安全端口
- context：定义了当前的上下文，表述为users@clusters
- current-context：当前处于的上下文环境
- users：定义了操作集群的用户，这里是kubernetes-admin

有了这个文件，我们才能使用kubectl命令创建访问任何kubernetes资源。Dashboard的Kubeconfig认证也是这种方式。首先是创建用户，然后创建并切换到相关的上下文。

1、在默认名称空间下创建service account:dashboard-reader

```shell
kubectl create serviceaccount dashboard-reader -n default
```



2、使用rolebinding将dashboard-reader与admin这个clusterrole关联起来

```shell
kubectl create rolebinding dahboard-reader --clusterrole=admin --serviceaccount=default:dashboard-reader -n default
```

> 有关admin的权限可以使用`kubectl describe clusterrole admin`查看具体权限，由于这里使用的是rolebinding并且名称空间是default，所以dashboard-reader仅能查看default名称空间下的内容。





3、获取dashboard-reader对应的secret的token并使用base64解码

```shell
KUBE_TOKEN=$(kubectl get secret $(kubectl get secret | awk '/^dashboard-reader/{print $1}') -o jsonpath='{.data.token}' | base64 -d)
```



4、配置集群参数

```shell
kubectl config set-cluster k8s --server=https://192.168.0.106:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --kubeconfig=/root/dashboard-reader.yml
```

- --certificate-authority 为该集群的公钥
- --embed-certs 为 true 表示将 –certificate-authority 证书写入到 kubeconfig 中
- --server 则表示该集群的 apiserver 地址



5、配置客户端认证参数

```shell
kubectl config set-credentials dashboard-reader --token=$KUBE_TOKEN --kubeconfig=/root/dashboard-reader.yml
```



6、设置上下文

```shell
kubectl config set-context dashboard-reader@k8s --cluster=k8s --user=dashboard-reader --kubeconfig=/root/dashboard-reader.yml
```



7、使用该上下文

```shell
kubectl config use-context dashboard-reader@k8s --kubeconfig=/root/dashboard-reader.yml
```



8、将/root/dashboard-reader.yml下载到本地就能顺利使用kubeconfig访问dashboard了

