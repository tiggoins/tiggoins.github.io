---
title: Nginx的location匹配优先级
date: 2020-09-27 00:18:44
tags:
 - Nginx
categories:
 - Nginx
toc: true
disableToC: false
disableAutoCollapse: true
---

一直以来觉得自己掌握了nginx，直到昨天面试发现自己说的location匹配优先级都不正确。主要原因也是之前没有做过多location的配置，没有去想过这个点，造成面试不理想。故写此博客理解location匹配的优先级。

<!--more-->



### <font color="red">前言</font>  

location是由`ngx_http_core_module`模块提供的指令

官网链接：http://nginx.org/en/docs/http/ngx_http_core_module.html#location

语法及优先级：

|  location  |          含义          | 优先级 |
| :--------: | :--------------------: | :----: |
|   =/uri    |        精确匹配        |   1    |
|    /uri    |  无前缀非完整路径匹配  |   4    |
|  ^~ /uri   |        路径匹配        |   2    |
| ~ pattern  |  区分大小写的正则匹配  |   3    |
| ~* pattern | 不区分大小写的正则匹配 |   3    |

**注**:数字越大代表优先级越低



### <font color="red">测试</font>  

首先根据6种匹配优先级设定测试文件，并分别配置对照组测试优先级

#### `=/uri` 和 `/uri`

例1：

```nginx
location =/1/2/1.txt {
	return 111;
}
  
location /1/2/1.txt {
    return 222;
}
```

使用curl请求测试

```shell
[root@bochs nginx]# curl -m 1 -s -w %{http_code} -o /dev/null http://www.dmesg.top:82/1/2/1.txt
111
```

**结论**：`=/uri` 的优先级高于 `/uri`



#### `/uri` 和 `^~`

例2：

```nginx
location /1/2/1.txt {
	return 111;
}

location ^~ /1/2/ {
	return 222;
}
```

> 这里如果路径匹配也是`/1/2/1.txt`,会提示重复。意味着如果都是完整路径，路径匹配与无前缀匹配一致。

curl请求测试

```shell
[root@bochs nginx]# curl -m 1 -s -w %{http_code} -o /dev/null http://www.dmesg.top:82/1/2/1.txt
111
```



例3：

```nginx
location /1/2/ {
	return 111;
}

location ^~ /1/2 {
    return 222;
}
```

curl请求测试

```shell
[root@bochs nginx]# curl -m 1 -s -w %{http_code} -o /dev/null http://www.dmesg.top:82/1/2/1.txt
222
```

**结论**：`^~`比`/uri`在非全路径时具有更高的优先级，但为全路径时`/uri`的优先级等于`^~`



#### `^~` 和 `~`

例4：

```nginx
location ~ \/1\/2/1\.txt$ {
	return 111;
}

location ^~ /1/2/1.txt {
	return 222;
}
```

curl请求测试

```shell
[root@bochs nginx]# curl -m 1 -s -w %{http_code} -o /dev/null http://www.dmesg.top:82/1/2/1.txt
222
```


例5：

```nginx
location ^~ /1 {
	return 222;
}
 
location ~ \.txt$ {
	return 111;
}
```

curl请求测试：

```shell
[root@bochs nginx]# curl -m 1 -s -w %{http_code} -o /dev/null http://www.dmesg.top:82/1/2/1.txt
222
```

**结论**：路径匹配`^~`的优先级高于正则表达式匹配`~`



#### `/uri` 和  `~`

例6：

```
location /1/2 {
	return 222;
}
 
location ~* \.txt$ {
	return 111;
}
```

curl请求测试

```shell
[root@bochs nginx]# curl -m 1 -s -w %{http_code} -o /dev/null http://www.dmesg.top:82/1/2/1.txt
111
```



例7：

```nginx
location /1/2/1.txt {
        return 222;
}
 
location ~* \.txt$ {
        return 111;
}  
```

curl请求测试

```shell
[root@bochs nginx]# curl -m1 -s -w %{http_code} -o /dev/null www.dmesg.top:82/1/2/1.txt
111
```

**结论**:无论`/uri`是否是完整路径，正则匹配的优先级都是高于无前缀匹配`/uri`的



### <font color="red">总结</font>

从上面的几个实验可以发现优先级关系。

精确匹配`=` > 路径匹配`^~` >  正则匹配`~` > 无前缀匹配`/uri` 

<strong style="background:yellow">特殊的：</strong>当无前缀匹配中的uri为全路径时，优先级等于路径匹配。
