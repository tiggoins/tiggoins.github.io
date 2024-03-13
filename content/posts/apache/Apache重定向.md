---
title: Apache重定向 
date: 2020-02-17 16:30:47
tags:
 - Apache
categories:
 - Apache
toc: true
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/HuntsMesa_ZH-CN7400133267_1920x1080.jpg
disableToC: false
disableAutoCollapse: true
---

apache的重定向主要是用RewriteCond及RewriteRule来完成，前者用于判断匹配条件，条件符合的将会转到下一条的Rewrite进行处理。

<!--more-->

先看一个例子

```
#打开重写模块
RewriteEngine On

#定义重写的基准路径
RewriteBase /

#定义判断，若"%{http_host}"=="dmesg.top",则继续执行下面的RewriteRule
RewriteCond %{http_host} ^dmesg.top$ [NC]

#定义重写规则，将所有访问dmesg.top中的请求重定向到www.dmesg.top的相同路径
RewriteRule ^(.*)$ www.dmesg.top/$1 [R=301]
```



### RewriteCond


语法

> RewriteCond String Pattern [flags]

定义一个条件，当String所规定的内容与Pattern规则匹配时，才会执行RewriteRule规定重写。



**string**

（1）$N：RewriteRule后向引用。$N引用紧跟在RewriteCond之后的RewriteRule中Pattern的小括号中的规则在当前URL中匹配的内容。N是0 <= N <= 9之间的整数。

（2）%N：RewriteCond后向引用 。%N引用最后一个RewriteCond的Pattern中的小括号中的规则在当前URL中匹配的内容。N是0 <= N <= 9之间的整数。



**pattern**

```t
(1) !：表示TestString不与当前正则匹配；格式是!CondPattern。
(2) >: 将condPattern作为普通字符串与String比较，String大于Pattern为真；格式是>Pattern。
(3) =：将condPattern作为普通字符串与String比较，String与Pattern相同时为真；格式是=Pattern。
(4) -d：将String当作一个目录名，检查它是否存在以及是否是一个目录；格式是-d。
(5) -f：将String当作一个文件名，检查它是否存在以及是否是一个文件；格式是-f。
(6) -s：将String当作一个文件名，检查它是否存在以及是否是一个长度大于0的文件；格式是-s。
(7) -l： 将String当作一个文件名，检查它是否存在以及是否是一个 symbolic link；格式是-l。
(8) -F：检查String是否是一个合法的文件，而且通过服务器范围内的当前设置的访问控制进行访问。检查通过一个内部subrequest完成的, 因此需要谨慎使用，以防止降低服务器的性能。
(9) -U：检查String是否是一个合法的URL，而且通过服务器范围内的当前设置的访问控制进行访问。检查通过一个内部subrequest完成的, 因此需要谨慎使用，以防止降低服务器的性能。
```



**[flags]**

```
（1）NC：表示不区分大小写。
（2）OR：默认的情况下，二个RewriteCond之间是AND的关系，用这个标志将关系改为OR,即满足任一的RewriteCond即执行RewriteRule。
RewriteEngine On
RewriteCond  [regex1] [OR]
RewrireCond  [regex2] 
RewriteRule  [rule]
```



### RewriteRule

> RewriteRule Pattern Substitution [flags]

**pattern**

作用于当前URL的正则表达式；此url不包含协议、域名和查询字符串部分。



**Substitution**

当RewriteCond满足时，用来替换原始URL指定内容的字符串，还可以包括以下扩展：

（1）$N：RewriteRule后向引用。$N引用紧跟在RewriteCond之后的RewriteRule中Pattern的小括号中的规则在当前URL中匹配的内容。N是0 <= N <= 9之间的整数。

（2）%N：RewriteCond后向引用 。%N引用最后一个RewriteCond的Pattern中的小括号中的规则在当前URL中匹配的内容。N是0 <= N <= 9之间的整数。



**[flags]**

```
(1) R：表示重定性，[R=301]表示301重定向，默认是302重定向。
(2) F：强制当前URL为被禁止的，即，立即反馈一个HTTP响应代码403(被禁止的)。
(3) G：强制当前URL为已废弃的，即，立即反馈一个HTTP响应代码410(已废弃的)。
(4) L：立即停止重写操作，并不再应用其他重写规则。 
(5) N：重新执行重写操作(从第一个规则重新开始). 这时再次进行处理的URL已经不是原始的URL了，而是最后一个重写规则处理的URL。
(6) C：此标记使当前规则与下一个(其本身又可以与其后继规则相链接的， 并可以如此反复的)规则相链接。 它产生这样一个效果: 如果一个规则被匹配，通常会继续处理其后继规则， 即，这个标记不起作用；如果规则不能被匹配，则其后继的链接的规则会被忽略。
(7) NC：忽略大小写。
(8) QSA：此标记强制重写引擎在已有的替换串中追加一个请求串，而不是简单的替换。
```



### 示例

1、强制https跳转

```
#开启Rewrite重写功能
RewriteEngine on

#判断规则为端口非443结尾
RewriteCond %{SERVER_PORT} !^443$

#强制访问到https的相应路径，响应状态码是301永久重定向
RewriteRule ^(.*)?$ https://%{SERVER_NAME}%{REQUEST_URI} [L,R]
```



2、404页面

```
#开启Rewrite重写功能
RewriteEngine on

#请求的文件名为一个常规文件但不存在，默认是AND，所以此处的flags不用设置
RewriteCond %{REQUEST_FILENAME} !-f

#请求的文件名为一个目录但不存在
RewriteCond %{REQUEST_FILENAME} !-d

#如果既不是常规文件，也不是常规目录，那么就跳转到404页面。
RewriteRule (.*) /404.htm
```



3、基于User-Agent的访问控制

```
#开启Rewrite重写功能
RewriteEngine on

#判断客户端user-agent信息是否为IE浏览器
RewriteCond %{HTTP_USER_AGENT} ^.*MSIE.* [NC]

#满足是IE浏览器，跳转到404页面，。表示访问任何URL
RewriteRule . /404.html [L]
```



4、QSA

```
RewriteEngine On
RewriteRule /pages/(.+) /page.php?page=$1
```

假如现在访问/pages/88?count=1页面，只会映射到：/page.php?page=88。

因为默认条件下，不会获取到查询字符串部分，(.+)只能匹配到88。



```
RewriteEngine On
RewriteRule /pages/(.+) /page.php?page=$1 [QSA]
```

访问/pages/88?count=1页面，将会映射到：/page.php?page=88&count=1

（1）[QSA]和正则表达式的子表达式配合使用。

（2）88?count=1中的问号被更换为$1



5、伪静态

```
RewriteRule ^article-([0-9]+)-([0-9]+)\.html$ portal.php?mod=view&aid=$1&page=$2&%1
```

输入article-8342-1.html实质访问portal.php?mod=view&aid=8342&page=1



6、访问路径添加/结尾

```
#开启Rewrite重写功能
RewriteEngine on
#如果当前路径不是完整的文件路径，则不生效
RewriteCond ${REQUEST_FILENAME} ! -f
#如果请求路径是以/结尾，则不生效
RewriteCond ${REQUEST_URI} !(.*)/$
#在路径后加上/,返回状态码301
RewriteRule ^(.*)$ $1/ [L,R=301]
```



7、防盗链

```Apache
RewriteEngine on
RewriteCond %{HTTP_REFERER} !^http://DomainName/.*$ [NC]
RewriteCond %{HTTP_REFERER} !^http://DomainName$ [NC]
RewriteCond %{HTTP_REFERER} !^DomainName/.*$ [NC]
```

