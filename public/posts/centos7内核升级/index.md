# CentOS7内核升级


CentOS7默认的内核版本是3.10，低版本内核无法使用高版本才适用的软件包，如bcc等。本文将介绍内核升级的两种方法并做出比较。
<!--more-->

### 方法比较
内核升级可以选用基于内核软件包的编译方式也可以选择用yum进行升级，前者运行时间长，并且需要消耗大量的磁盘空间，这一点对于个人学习下使用的云服务器来说很不友好，如果你剩余磁盘已经不够20G，很可能会导致编译终止。第二种方式使用yum安装，安装速度十分快捷，15分钟内就可以得到一个全新内核的linux服务器，所以个人使用更推荐后者。当然在磁盘不紧张的情况下，也可以选用前者练练手，但找kernel-devel包又会让人很头疼。

### 编译方式升级内核
首先去[linux内核官方网站](https://www.kernel.org)下载你想升级的内核版本，这里有三个主要的版本，mainline即为现在开发的主线版本，stable为稳定版本，longterm为长期支持版本。其中longterm是最为稳妥的选择。这里以4.14.170版本为列，并且假定为root用户，如果当前用户不是root用户可以`su -`切换到root用户或者在关键命令前加上`sudo`
安装开发软件包
```
yum groupinstall -y "Development tools"
```
下载安装包至服务器
```
mkdir /kernelupdate && 
wget -O /kernelupgrade/linux4.14.170.tar.xz  https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.14.170.tar.xz 
```
下载完成后解压
```
cd /kernelupdate && tar -xvf linux4.14.170.tar.xz 
```
解压完成后进入目录通过make menuconfig来进行内核配置
![avatar](https://imagesofhexo.oss-cn-shanghai.aliyuncs.com/hexo-pic/kernel-update/makemenu.png)
配置完成选择save然后退出即可，此时会在目录下生成.config文件。或者拷贝当前内核的默认配置到解压目录
```
cp /boot/config-$(uname -r) .config
```
当然没了加快整个的编译过程。可以选择多核心同时编译，使用lscpu或者`cat /proc/cpuinfo|grep "cpu cores"|wc -l`来查看cpu核数
```
make -jn #n换成上一步查询到的cpu核数
```
接下来就是漫长的编译过程，便已完成后需要进一步编译及安装内核模块
```
make modules_install
```
最后就是把编译好的内核安装到系统中
```
make install
```
此步骤完成后请编辑`/boot/grub2/grub.cfg`将多余的内核选项删除，确保不会启动到旧内核
```
reboot
```
最后重启进入新系统就行



### YUM方式升级内核

导入elrepo的证书文件
```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```
导入elrepo文件
```
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```
查询可用的内核升级包
```
yum --disablerepo="*" --enablerepo="elrepo-kernel" list avaliable
```
结果如下：
```
Ξ (bochs) ~ → yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
Loaded plugins: fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
 * elrepo-kernel: mirrors.tuna.tsinghua.edu.cn
Available Packages
kernel-lt-doc.noarch                       4.4.213-1.el7.elrepo         elrepo-kernel
kernel-lt-headers.x86_64                   4.4.213-1.el7.elrepo         elrepo-kernel
kernel-lt-devel.x86_64                     4.4.213-1.el7.elrepo         elrepo-kernel
kernel-lt-tools.x86_64                     4.4.213-1.el7.elrepo         elrepo-kernel
kernel-lt-tools-libs.x86_64                4.4.213-1.el7.elrepo         elrepo-kernel
kernel-lt-tools-libs-devel.x86_64          4.4.213-1.el7.elrepo         elrepo-kernel
kernel-ml.x86_64                           5.5.2-1.el7.elrepo           elrepo-kernel
kernel-ml-devel.x86_64                     5.5.2-1.el7.elrepo           elrepo-kernel
kernel-ml-doc.noarch                       5.5.2-1.el7.elrepo           elrepo-kernel
kernel-ml-headers.x86_64                   5.5.2-1.el7.elrepo           elrepo-kernel
kernel-ml-tools.x86_64                     5.5.2-1.el7.elrepo           elrepo-kernel
kernel-ml-tools-libs.x86_64                5.5.2-1.el7.elrepo           elrepo-kernel
kernel-ml-tools-libs-devel.x86_64          5.5.2-1.el7.elrepo           elrepo-kernel
```
这里的lt即为刚才提到的longterm版,ml为mainline版。这里我还是安装lt版本，顺便安装kernel-devel，美滋滋！
```
yum --disablerepo="*" --enablerepo="elrepo-kernel" install kernel-lt-*
```
安装完成后生成grub配置文件
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
最后编辑`/boot/grub2/grub.cfg`项，将其中的多余内核加载项删除并重启即可
最后是升级好的结果
```
Ξ (bochs) ~ → cat /etc/redhat-release 
CentOS Linux release 7.7.1908 (Core)
Ξ (bochs) ~ → uname -r
4.4.213-1.el7.elrepo.x86_64
```



### 总结

对比上述两种内核升级方法，使用yum安装要简单许多，但yum安装许多内核的模块无法手动配置，这就导致很多的模块可能是用不了的，反正看情况选择合适的方法。最后，内核升级仅推荐在自己的服务器上升级，**谨慎**在生产服务器上操作！


