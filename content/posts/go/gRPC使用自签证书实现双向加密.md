---
title: gRPC使用自签证书实现双向加密
date: 2021-04-28 10:50:47    
tags:    
 - Go    
categories:    
 - Go   
toc: true
disableToC: false
disableAutoCollapse: false
---

### 前言

最近在学习gRPC，在生成服务端证书与客户端实现双向加密中遇到了证书错误，特此记录下过程及解决方法。
<!--more-->



### 错误

transport: authentication handshake failed: x509: certificate relies on legacy Common Name field, use S
ANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0。从错误来看是因为证书中依赖传统的CN(Common Name)字段，而Go自1.15版本废弃了common name方式认证，需要使用SANS证书进行认证，或者设置Go环境变量**GODEBUG=x509ignoreCN=0**。



### 解决

#### 生成CA证书

- 生成CA私钥，长度为4096位

```bash
$ openssl genrsa -out ca.key 4096
```

- 生成CA公钥

```bash
$ openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/O=example/CN=test.com" \
 -key ca.key \
 -out ca.crt
```

#### 生成服务端证书

- 生成服务端私钥

```bash
$ openssl genrsa -out server.key 4096
```

- 生成服务端证书认证请求

```bash
$ openssl req -sha512 -new \
    -subj "/O=example/CN=test.com" \
    -key server.key \
    -out server.csr
```

- 生成服务端x509 v3扩展文件

```bash
$ cat > v3-server.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=test.com
DNS.2=www.test.com
EOF
```

- 生成服务端公钥

```bash
$ openssl x509 -req -sha512 -days 3650 \
    -extfile v3-server.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in server.csr \
    -out server.crt
```



#### 生成客户端证书
- 生成客户端私钥

```bash
$ openssl genrsa -out client.key 4096
```

- 生成客户端证书认证请求

```bash
$ openssl req -sha512 -new \
    -subj "/O=example/CN=test.com" \
    -key client.key \
    -out client.csr
```

- 生成客户端x509 v3扩展文件

```bash
$ cat > v3-client.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=test.com
DNS.2=www.test.com
EOF
```

> 该文件与客户端证书唯一不同就在于extendedKeyUsage，服务端是serverAuth,客户端为clientAuth。

- 生成客户端公钥

```bash
$ openssl x509 -req -sha512 -days 3650 \
    -extfile v3-client.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in client.csr \
    -out client.crt
```

### 应用

拷贝上述生成的ca.crt,server.crt,server.key,client.crt,client.key文件到项目的certs目录中。服务端及客户端代码示例：

- 服务端

```go
cert,err:=tls.LoadX509KeyPair("certs/server.crt","certs/server.key")
	if err!=nil{
		log.Fatal(err)
	}
	certPool:=x509.NewCertPool()
	ca,err:=ioutil.ReadFile("certs/ca.crt")
	if err!=nil{
		log.Fatal(err)
	}
	certPool.AppendCertsFromPEM(ca)

	cred:=credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		ClientAuth: tls.RequireAndVerifyClientCert,
		ClientCAs: certPool,
	})
```

- 客户端

```go
cert,_:=tls.LoadX509KeyPair("certs/client.crt","certs/client.key")
	certPool:=x509.NewCertPool()
	ca,_:=ioutil.ReadFile("certs/ca.crt")
	certPool.AppendCertsFromPEM(ca)

	cred:=credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		ServerName: "test.com", //alt_names文件中的任意域名一致
		RootCAs: certPool,
	})
```



