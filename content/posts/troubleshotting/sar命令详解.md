---
title: sar命令详解
date: 2020-02-29 22:55:33
tags:
 - Performance
categories:
 - Performance
cover: https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/wallpaper/SiegeofCusco_ZH-CN9108219313_1920x1080.jpg
toc: true
disableToC: false
disableAutoCollapse: true
---
sar是强大的linux系统活动状况收集、报告命令。可以收集CPU，内存，磁盘I/O，网络等诸多数据。对于性能分析是个可靠的利器，本文介绍sar命令的各种用法。

<!--more-->



### 安装 

sar命令是sysstat下的一个工具，所以安装sar需要首先安装sysstat命令，可以考虑yum安装或者使用源码包编译安装等。yum 安装十分便捷，不需要任何复杂的调试就可以使用

```shell
#ubuntu
sudo apt-get install -y sysstat

#centos
sudo yum install -y sysstat
```

yum仓库目前安装的版本为10.1.5，相对于最新版12.3.1要旧很多，所以新的特性可能会无法使用，我推荐[下载](http://sebastien.godard.pagesperso-orange.fr/download.html)最新版本源码包编译。

```shell
#安装gcc等重要环境
sudo yum install -y gcc gcc-c++ #centos
sudo apt-get install -y gcc gcc-c++ #ubuntu

#下载安装包至/tmp目录
wget http://pagesperso-orange.fr/sebastien.godard/sysstat-12.3.1.tar.gz -O /tmp

#进入/tmp目录并解压
cd /tmp && tar -xzvf sysstat-12.3.1.tar.gz

#进入解压目录编译安装
cd sysstat-12.3.1 && ./configure && make && make install

#最后查看任一命令的版本即可得到sysstat版本
Ξ (bochs) /tmp/sysstat-12.3.1 → mpstat -V
sysstat version 12.3.1
(C) Sebastien Godard (sysstat <at> orange.fr)
```

初次使用sar命令会遇到如下报错

```
Ξ (bochs) ~ → sar
Cannot open /var/log/sa/sa29: No such file or directory
Please check if data collecting is enabled
```

这是因为sar找不到记录数据的源文件，这时只需要使用-o参数生成即可`sar -o 2 3`。

### 常用用法 

`sar [command] 2 5 `: 每2秒输出一次sar [command],总计输入五次，省略5表示持续输出

`sar -n DEV 1 -e 22:26:00 >/tmp/123 &`:每秒采样一次网络情况直到22:26并把采样数据输出到/tmp/123

`sar -f /var/log/sa/sa27 -s 23:00:00 -e 00:00:00 -r`:本月27日23点至0点的内存数据，需要通过crontab设置定时任务



### CPU篇

#### -p

**-P {CPU_LIST | ALL}**:用于分析多核CPU的性能状况，可以使用CPU_LIST分析指定核心的CPU状态，可以使用离散值和连续值，也可以使用ALL分析所有CPU核心状态。

```
Ξ (bochs) ~ → sar -P 0 1 3  
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

16:31:33        CPU     %user     %nice   %system   %iowait    %steal     %idle
16:31:34          0      0.00      0.00      0.00      0.00      0.00    100.00
16:31:35          0      0.00      0.00      0.00      0.00      0.00    100.00
16:31:36          0      0.00      0.00      0.00      0.00      0.00    100.00
Average:          0      0.00      0.00      0.00      0.00      0.00    100.00
```

表示每秒采集**<u>0</u>**号CPU状态，总共采样3次。

前两列不必多言，%user指运行非特权用户进程时间百分率

`%nice`是指运行特权用户进程时间百分率

`%system`是指运行内核进程时间，这个时间包括了CPU处理软硬中断的时间

`%iowait`是指等待I/O完成的时间

`%steal`是指运行虚拟机的时间百分率，steal意味着被偷走的时间

`%idle`是指cpu空闲时间百分率，我的机器上并未运行任何程序，所以此列一直为100%



#### -u

**`-u[ALL]`**:报告cpu使用情况，与`-p`不同的是，`-u`只能报告所有cpu。`ALL`选项输出详细信息

```
Ξ (bochs) ~ → sar -u ALL 1 3
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

17:23:29        CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
17:23:30        all      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
17:23:31        all      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
17:23:32        all      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
Average:        all      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
```

<font color=red size=3>这里的%usr和-P的%user的区别在于%usr不包括虚拟机运行的时间</font>

<font color=red size=3>这里的%sys和-P的%system的区别在于%sys不包括各种软硬中断时间</font>

`%irq`是指处理硬中断的cpu时间百分率

`%soft`是指处理软中断的cpu时间百分率

`%guest`和`%gnice`分别指运行普通虚拟程序和特权虚拟程序的时间百分率



#### -q

**`-q`**:用于报告队列长度以及平均负载

```
Ξ (bochs) ~ → sar -q 1 3
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

16:46:47      runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
16:46:48            0       126      0.00      0.00      0.00         0
16:46:49            0       126      0.00      0.00      0.00         0
16:46:50            0       126      0.00      0.00      0.00         0
Average:            0       126      0.00      0.00      0.00         0
```

`runq-sz`:等待cpu调度的任务数

`plist-sz `:处于任务列表的任务总数

`ldavg-1`，`ldavg-5`，`ldavg-15`分别指1分钟，5分钟，15分钟内cpu的负载

`blocked`:表示等待I/O完成而被阻塞的任务总数，不为0则需要留意I/O是否存在性能瓶颈



#### -w

**`-w`**:报告进程上下文切换的次数

```
Ξ (bochs) ~ → sar -w 1 3 
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

17:39:43       proc/s   cswch/s
17:39:44         0.00     83.00
17:39:45         0.00    103.00
17:39:46         0.00     96.00
Average:         0.00     94.00
```

`proc/s`:指每秒创建的进程数

`cswch/s`:指每秒自愿上下文切换的次数，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。

还有一个非自愿的上下文切换次数`nvcswch/s`表示则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。非自愿次数明显升高意味着cpu产生了性能瓶颈。非自愿上下文切换可以使用pidstat加上`-w`选项来输出



### 内存篇

#### -r

**`-r [-h]`**:输出内存使用率统计信息，-h输出更加利于阅读的结果

```
Ξ (bochs) ~ → sar -r -h 1 3
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

21:58:21    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
21:58:22       602.6M      1.6G    165.3M      8.3%    196.8M    878.7M    743.3M     37.3%    830.7M    379.7M      4.0k
21:58:23       602.6M      1.6G    165.3M      8.3%    196.8M    878.7M    743.3M     37.3%    830.7M    379.7M      4.0k
21:58:24       602.6M      1.6G    165.3M      8.3%    196.8M    878.7M    743.3M     37.3%    830.7M    379.7M      4.0k
Average:       602.6M      1.6G    165.3M      8.3%    196.8M    878.7M    743.3M     37.3%    830.7M    379.7M      4.0k
```

`kbmemfree`:剩余内存总量

`kbavail`:可用内存总量，可用内存≈剩余内存+buffer+cache

`kbmemused`:使用的内存总量，使用量=总内存-剩余内存-buffer-cache-slab

`kbbuffers`:被内核用来作为buffer的内存量

`kbcached`:被内核用来作为cache的内存量

`kbcommit`:当前工作负载下，可以保证不出现内存不足的内存量

`%commit`:指`kbcommit`占内存/swap的百分率

`kbactive`:当前活跃内存量。除非万不得已，这部分内存才会被回收

`kbinact`:当前非活跃内存总量，当内存不足时，这部分内存最容易被内核收回

`kbdirty`：脏页大小，脏页指的是暂存于内存还没来得及持久化到硬盘的数据。可以使用`sync`刷入硬盘



#### -B

`-B`:报告系统中分页统计信息

```
λ bochs ~ → sar -B 1 3
Linux 4.4.213-1.el7.elrepo.x86_64 (bochs)       03/02/2020      _x86_64_        (1 CPU)

10:43:36 PM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
10:43:37 PM      0.00     60.00     18.00      0.00      5.00      0.00      0.00      0.00      0.00
10:43:38 PM      0.00     20.00    158.00      0.00    153.00      0.00      0.00      0.00      0.00
10:43:39 PM      0.00     60.00     53.00      0.00     71.00      0.00      0.00      0.00      0.00
Average:         0.00     46.67     76.33      0.00     76.33      0.00      0.00      0.00      0.00
```

`pgpgin/s`:表示每秒从磁盘中换入内存的字节数

`pgpgout/s`:表示每秒从内存换出到磁盘的字节数

`fault/s`:表示系统每秒产生的缺页异常（报告主缺页和次缺页），这不是产生I/O的缺页中断的次数，因为部分缺页中断不需要I/O就能处理

`majflt/s`:表示每秒产生的主缺页异常

`pgfree/s`:每秒被系统放到空闲列表的分页数量

`pgscank/s`:每秒被内核线程[kswapd]扫描的分页数量

`pgscand/s`:每秒被直接扫描的数量

`pgsteal/s`:系统为满足其内存需求声明每秒从cache(分页缓存和swap缓存)回收的页的数量

`%vmeff`:这个指标由pgsteal/(pgscand+pgscank)得到，这是一个衡量页面回收效率的指标



#### -S

`-s [h]`:输出swap空间的使用率统计信息

```
Ξ (bochs) ~ → sar -S 1 1
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

22:20:48    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
22:20:49            0         0      0.00         0      0.00
Average:            0         0      0.00         0      0.00
```

*我的这台机器上未开启swap，所有的统计信息均为0*

`kbswpfree`:未使用的swap量

`kbswpused`：使用中的swap内存量

`%swpused`：使用中的swap内存量占总swap的百分率

`kbswpcad`:被swap缓存的内存量，这部分内存曾经被换出，现在又被换入但仍然位于swap空间

`%swpcad:`kbswpcad占kbswpused的百分率



#### -W

`-W`统计输出swap换入换出信息

```
Ξ (bochs) ~ → sar -W 1 2
Linux 4.15.0-87-generic (test)  02/29/20        _x86_64_        (1 CPU)

22:43:36     pswpin/s pswpout/s
22:43:37         0.00      0.00
22:43:38         0.00      0.00
Average:         0.00      0.00
```

`pswpin/s`:每秒换入swap的内存量

`pswpout/s`:每秒换出swap的内存量

若这两项数值很高，表示内存短缺导致需要频繁换入换出。



### I/O篇



#### -b

`-b`:报告I/O及传输速率统计信息

```
Ξ (bochs) ~ → sar -b 1 3
Linux 4.4.213-1.el7.elrepo.x86_64 (bochs)       03/03/2020      _x86_64_        (1 CPU)

12:52:28 PM       tps      rtps      wtps      dtps   bread/s   bwrtn/s   bdscd/s
12:52:29 PM     10.00      0.00     10.00      0.00      0.00    280.00      0.00
12:52:30 PM     18.00      0.00     18.00      0.00      0.00    256.00      0.00
12:52:31 PM      9.00      0.00      9.00      0.00      0.00    144.00      0.00
Average:        12.33      0.00     12.33      0.00      0.00    226.67      0.00
```

`tps`:每秒钟加到物理设备上的传输总量。一次传输就是加到物理设备的一次I/O请求，由于多次逻辑请求可以合并为单次的I/O请求，所以一次传输的大小是不确定的

` rtps`：每秒加到物理设备的读请求总数

`wtps`：每秒加到物理设备上的写请求总数

`dtps`:每秒丢弃的请求总数

` bread/s`:以块为单位每秒钟从磁盘读取的数据总量，块的大小等同于一个扇区的大小，512字节

`bwrtn/s`：以块为单位每秒钟写入磁盘的数据总量

`bdscd/s`以块为单位丢弃的数据总量



#### -d

`-d -h[--dev=dev_list]`:报告块设备的活动情况

```
Ξ (bochs) ~ → sar -d 1 1 
Linux 4.4.213-1.el7.elrepo.x86_64 (bochs)       03/03/2020      _x86_64_        (1 CPU)

08:06:35 PM       DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
08:06:36 PM       vda      9.00      0.00    100.00      0.00     11.11      0.02      1.89      1.70
```

`tps`:和`-b`的`tps`含义一样，都表示每秒钟加到物理设备上的传输总量

`rkB/s`:每秒从设备读到的字节数

`wkB/s`:每秒写入设备的字节数

`areq-sz`:加到设备上I/O请求平均大小(以字节为大小)

`aqu-sz`:加到设备上请求长度的平均值

`await`:加到设备上I/O请求的平均响应时间，这个时间包括了请求处于等待队列中的时间

`%util`:加到设备上I/O请求所用的时间百分比，对于串行设备，接近100%意味着设备出现了性能瓶颈，但是对于并行设备比如RAID或者SSD，这个值实际上并不能反映出设备的极限



#### -v

`-v`:报告inode，文件以及其他内核表状态

```
Ξ (bochs) ~ → sar -v 1 1
Linux 4.4.213-1.el7.elrepo.x86_64 (bochs)       03/03/2020      _x86_64_        (1 CPU)

08:35:59 PM dentunusd   file-nr  inode-nr    pty-nr
08:36:00 PM      1692      1504      9458         1
Average:         1692      1504      9458         1
```

`dentunusd`:目录缓存中未使用的缓存项数。

`file-nr`:系统使用的文件句柄数，查看目前使用的文件句柄数`lsof|awk '{print $2}'|wc -l`

`inode-nr`:系统使用的inode处理程序数

`pty-nr`:打开的伪终端数，即几个人登陆了系统



### 网络篇

#### -n

`-n DEV [--iface=face_list]`

```
Ξ (bochs) ~ → sar -n DEV 1 1
Linux 4.4.213-1.el7.elrepo.x86_64 (bochs)       03/03/2020      _x86_64_        (1 CPU)

09:04:07 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
09:04:08 PM        lo     20.00     20.00      1.19      1.19      0.00      0.00      0.00      0.00
09:04:08 PM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:04:08 PM br-f3a0301d37ae      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:04:08 PM docker_gwbridge      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:04:08 PM      eth0      9.00      7.00      0.68      0.78      0.00      0.00      0.00      0.00
```

`IFACE`:网络接口

`rxpck/s`:每秒接收的报文数

`txpck/s`:每秒发送的报文数

`rxkB/s`:每秒接收的字节数,``rxkB/s`*1024/`rxpck/s`<60B,意味着收到的是小包

`txkB/s`:每秒发送的字节数

` rxcmp/s`:每秒接收的压缩数据包数

`txcmp/s`:每秒发送的压缩数据包数

` rxmcst/s`:每秒接收的多播数据包数

`%ifutil`:网络接口的利用率百分比，对于半双工接口，利用率使用`rxkB/s`和`txkB/s`之和作为接口速度的百分比计算。对于全双工，这是`rxkB/s`或`txkB/s`中的较大值。
