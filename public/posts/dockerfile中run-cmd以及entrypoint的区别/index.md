# Dockerfile中RUN CMD以及ENTRYPOINT的区别

Dockerfile中的RUN，CMD，ENTRTPOINT三个指令均可以用来指明容器中所运行的指令，但这三者存在的细微的区别。
<!--more-->



简单来说：

### RUN

RUN指令一般用于在容器内安装软件包或者是执行其他的命令，如

```
RUN yum install -y telnet
RUN touch web.xml
```



### CMD

CMD指令主要用来指明生成的Docker镜像在启动时的命令及参数，这个指令可以被docker run后面的命令所取代，比如下面这个Dockerfile文件

```
FROM busybox
CMD echo "hello world"
```

CMD指明了Docker镜像在运行时的输出一个"hello world"

```
[root@bochs Docker]# docker build -t test .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM busybox
 ---> 83aa35aa1c79
Step 2/2 : CMD echo "hello world"
 ---> Running in a1a4d74137d2
Removing intermediate container a1a4d74137d2
 ---> 651b45b58fe9
Successfully built 651b45b58fe9
Successfully tagged test:latest
[root@bochs Docker]# docker run -it test
hello world
```

但是如果在docker run后添加其他指令。那么CMD将直接被替换

```
[root@bochs Docker]# docker run -it test ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
```



### ENTRYPOINT 

ENTRYPOINT与CMD类似，区别在于ENTRYPOINT**一定**会被执行。如果一个Dockerfile中同时存在ENTRYPOINT和

CMD，CMD中的参数会被当做额外参数传给ENTRYPOINT。

```
[root@bochs Docker]# cat Dockerfile 
FROM busybox
ENTRYPOINT ["/bin/echo","hello"]
CMD ["world"]
```

通过docker run 来运行，CMD变成了ENTRYPOINT的参数：

```
[root@bochs Docker]# docker run -it test2 
hello world
```

但是如果指明docker run 的参数china，那么输出就会变为：

```
[root@bochs Docker]# docker run -it test2 china
hello china
```

原本CMD中带的参数world被docker run中的china所替换，但ENTRYPOINT自带的hello依然正常输出



### Shell与Exec格式

 CMD，RUN，ENTRYPOINT可以用两种格式来传递命令和参数，Shell一般表示为指令+命令，如:

```
RUN yum install -y telnet
CMD echo "hello world"
```

第一个大写的单词是Dockerfile的指令。后面跟的就是命令，可以拿到shell中单独执行

Exec格式可以表示为:指令+["命令","命令参数1","命令参数2",...],比如:

```
RUN ["yum","install","telnet"]
ENTRYPOINT ["/bin/bash","-c","echo hello world"]
```

对于这两种格式来说，CMD和ENTRYPOINT最好使用Exec格式，命令和参数分开，层次性较强，而RUN则都可以。


**注意**:ENTRYPOINT的Shell格式和Exec格式差异很大

比如下面这个Shell格式的ENTRYPOINT

```
FROM busybox
ENTRYPOINT echo "hello"
CMD "world"
```

在运行所生成的容器时，仅会输出hello，而CMD带的"world"会被~~忽略~~。同样的docker run带的参数也同样会被~~忽略~~

```
[root@bochs Docker]# docker run -it test
hello
[root@bochs Docker]# docker run -it test china
hello
```

