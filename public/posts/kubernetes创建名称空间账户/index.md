# Kubernetes创建名称空间账户


在一套k8s集群中会存在多个名称空间，各个名称空间由其项目组进行维护。为达到名称空间隔离，防止其他用户操作本空间下的资源，需要创建特定的config文件。使用对应的config仅能对相应名称空间下的资源进行管理。本例以k8s-api用户为例，该用户仅能操作k8s-api名称空间下的部分资源。

### 创建证书

1. 创建私钥

```bash
  openssl genrsa -out k8s-api.key 2048
```

2. 创建证书签署请求CSR

```bash
  openssl req -new -key k8s-api.key -out k8s-api.csr -subj "/CN=k8s-api"
```

3. 使用k8s集群的ca证书及私钥签发用户证书

```bash
  openssl x509 -req -in k8s-api.csr  \
  -CA /etc/kubernetes/ssl/ca.crt  \
  -CAkey /etc/kubernetes/ssl/ca.key  \
  -CAcreateserial  \ 
  -out k8s-api.crt  \
  -days 3560
```

**注**：签发周期为10年，实际环境取决于需求。

### 生成k8s-api用户要用到的config文件
config文件存放着kubectl用来操作k8s集群用的到证书、集群信息等，位于$HOME/.kube/目录下。
1. 生成默认的config(此步骤结束会生成k8s-api.conf)

```bash
  kubectl config set-cluster k8s-api  \
  --certificate-authority=/etc/kubernetes/ssl/ca.crt  \
  --embed-certs=true  \
  --server=https://172.16.200.101:6443  \
  --kubeconfig=k8s-api.conf
```
**注**：1. \-\-embed-certs=true: 将证书文件内嵌到config中
        2. \-\-server=https://172.16.200.101:6443:指定apiserver的地址

2. 将创建的用户公钥及私钥加入到config文件中

```bash
  kubectl config set-credentials k8s-api  \
  --client-certificate=k8s-api.crt  \
  --client-key=k8s-api.key  \
  --embed-certs=true  \
  --kubeconfig=k8s-api.conf 
```

3. 配置上下文参数

```bash
  kubectl config set-context k8s-api  \
  --cluster=k8s-api  \
  --user=k8s-api  \
  --kubeconfig=k8s-api.conf
```

4. 查看生成好的config文件

```yaml
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority-data: ignore... (ca.crt | base64)
      server: https://172.16.200.101:6443
    name: k8s-api
  contexts:
  - context:
      cluster: k8s-api
      namespace: k8s-api
      user: k8s-api
    name: k8s-api
  current-context: ""  (进入相应用户后设置该字段)
  kind: Config
  preferences: {}
  users:
  - name: k8s-api
    user:
      client-certificate-data: ignore... (cat k8s-api.crt | base64)
      client-key-data: ignore...  (cat k8s-api.key | base64)
```

### 创建名称空间

```bash
  kubectl create ns k8s-api
```

### 创建RBAC规则

#### 创建role定义k8s-api用户拥有的权限

  **注**：该角色配置仅作为参考，实际环境按需调整

  ```bash
  cat k8s-api-rolebinding.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: k8s-api-role
  namespace: k8s-api
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - create
  - get
  - list
  - watch
  - update
- apiGroups:
  - apps
  resources:
    - deployments
  verbs:
  - create
  - get
  - delete
  ```

```
$ kubectl create -f k8s-api-role.yaml 
```

  #### 创建rolebinding

```bash
  cat k8s-api-rolebinding.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: k8s-api-rolebinding
  namespace: k8s-api
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: k8s-api-role
subjects:
- kind: User
  name: k8s-api
```

```bash
  kubectl create -f k8s-api-rolebinding.yaml 
```

### 创建用户并测试权限

#### 创建系统用户k8s-api并设置密码

  ```bash
  $ useradd k8s-api
  $ passwd k8s-api
  $ mkdir /home/k8s-api/.kube
  $ cp ./k8s-api.conf /home/k8s-api/.kube/config
  $ chown -Rf k8s-api.k8s-api /home/k8s-api/
  ```

#### 切换到k8s-api用户并测试相关权限

  ```bash
  $ su - k8s-api
  
  $ kubectl config use-context k8s-api
  
  $ kubectl get ns
  Error from server (Forbidden): namespaces is forbidden: User "k8s-api" cannot list resource "namespaces" in API group "" at the cluster scope
  
  $ kubectl get pods -n kube-system
  Error from server (Forbidden): pods is forbidden: User "k8s-api" cannot list resource "pods" in API group "" in the namespace "kube-system"
  
  $ kubectl get pods -n k8s-api (可不加名称空间,默认就是k8s-api)
  NAME                                  READY   STATUS    RESTARTS   AGE
  busybox-deployment-5ddc5f454f-76c7d   1/1     Running   0          19h
  busybox-deployment-5ddc5f454f-gqbdb   1/1     Running   0          19h
  nginx-deployment-2-5957d54f6d-5cj4r   1/1     Running   0          19h
  nginx-deployment-2-5957d54f6d-8bdfx   1/1     Running   0          19h
  
  $ kubectl get deployments -n k8s-api (可不加名称空间,默认就是k8s-api)
  Error from server (Forbidden): deployments.apps is forbidden: User "k8s-api" cannot list resource "deployments" in API group "apps" in the namespace "k8s-api"
  
  $ kubectl create deployment k8s-api-rbac-test --image=reg.kolla.org/library/nginx:1.19 --namespace=k8s-api --replicas=2   
  deployment.apps/k8s-api-rbac-test created
  ```

  可以看出k8s-api用户仅拥有对k8s-api名称空间下pod及deployment的相应权限，保证了隔离性。
