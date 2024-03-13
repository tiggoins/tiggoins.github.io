---
title: Nginx日志回滚
date: 2020-02-17 16:32:21
tags:
 - Linux
categories:
 - Linux
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/RedBlueWildebeest_ZH-CN1024893552_1920x1080.jpg
toc: true
---

Nginx不像Apache那样自带`rotatelogs`进行日志的回滚，默认配置的日志文件会越来越大造成无法阅读，必须手动为Nginx配置日志回滚的方式。可以使用自定义脚本或是借助Linux自带的`logrotate`命令实现日志回滚。

<!--more-->



### 脚本分割

脚本分割日志的方法比较容易理解，获取昨天的日期并将日志文件命名为带有昨天的日期的名称，重命名结束后向Nginx发送USR1信号,Nginx在收到USR1信号后重新打开日志文件并写入。

```shell
#!/usr/bin/env bash
#anthor ltzhang
#description split_nginx_log

#获取日志文件的路径
LOGS_PATH=/opt/openresty/nginx/logs/

#获取该路径下所有以.log结尾日志
LOGS_LIST=$(ls $LOGS_PATH/*.log)

#获取昨天的日期
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)

#for循环遍历日志列表，将所有日志名称重命名为*.log.$(YESTERDAY)
for logs in ${LOGS_LIST[@]};do
	varlog=$(basename $logs)
	mv -f $LOGS_PATH/$varlog $LOGS_PATH/$varlog.$YESTERDAY
done

#向Nginx的master进程发送USR1信号,Nginx重新打开*.log并写入日志
kill -USR1 $(cat ${LOGS_PATH}/../run/nginx.pid)
```

**优点**：灵活，文件名及路径可以自定义

**缺点**：需要在`crontab`中设置计划任务，较不方便管理阅读



### logrorate分割

`logrotate`是linux自带的日志回滚命令，可以实现日志回滚。`logrotate`借助`crond`实现日志回滚的计划任务,所在的路径为`/etc/cron.daily/logrotate`，即每天执行一次logrotate脚本。

打开脚本，可以发现内容定义

```bash
root 00:13:32  /etc >>> cat /etc/cron.daily/logrotate 
#!/bin/sh

/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```

每天按照`/etc/logrotate.conf`配置文件中的定义执行一次日志回滚操作，并将该日志的状态写入`/var/lib/logrotate/logrotate.status`,如果执行失败，则错误状态码写入`message`日志中。

继续打开主配置文件可以看到许多配置

```bash
root 00:19:05  /etc >>> grep -v '^$' /etc/logrotate.conf | grep -v '^#'
weekly
rotate 4
create
dateext
include /etc/logrotate.d
/var/log/wtmp {
    monthly
    create 0664 root utmp
        minsize 1M
    rotate 1
}
/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
```

`weekly`：每周执行日志回滚

`rotate`：历史日志保留4天

`create`：回滚旧日志后创建新文件。可带权限，改变属主属组。如:`create 0664 nginx root`

`dateext`：以日期作为回滚文件的后缀。默认格式YYYYMMDD，其他的格式可用`dateformat format_string`自行定义。

除此以外还有些常用的参数：

`compress`：压缩回滚文件，默认后缀.gz

`ifempty`：若日志为空也执行回滚操作

` maxsize size`：除按照时间回滚外还支持文件最大大小回滚，当日志超过定义值时执行一次日志回滚操作，适用于每天日志较大的情况，如Nginx日志等

`olddir directory`：指定回滚文件的存放路径，必须与日志文件位于同一文件系统中

`prerotate/endscript`:定义回滚操作结束后执行的动作



配置Nginx回滚只需要按照上面的参数设置自己的配置段即可，例子如下：

```bash
root 01:33:21  /opt/openresty/nginx/logs >>> cat /etc/logrotate.d/openrestry 
/opt/openresty/nginx/logs/*log {
    create 0664 nginx root    
    daily                          
    maxsize 20M
    rotate 14          
    ifempty
    nocompress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /opt/openresty/nginx/run/nginx.pid  2>/dev/null` 2>/dev/null || true
    endscript
}
```

> 如果发现没有正常回滚，可以使用`logrotate -dv /etc/logrotate.conf`观察输出



**优点**：1）可以定义最大大小，实现单日日志分割 2）可以实现指定数量的历史日志（脚本也可配置但较复杂）3）不用单独写入计划任务

**缺点**：无
