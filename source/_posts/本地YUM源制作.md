---
title: 本地YUM源制作
tags:
  - 环境部署问题系列
categories:
  - big-data
abbrlink: 78696dcf
date: 2017-10-07 09:22:00
---

### 1. YUM相关概念

#### 1.1. 什么是YUM

YUM（全称为 Yellow dog Updater, Modified）是一个在Fedora和RedHat以及CentOS中的Shell前端软件包管理器。基于RPM包管理，能够从指定的服务器自动下载RPM包并且安装，可以自动处理依赖性关系，并且一次安装所有依赖的软件包，无须繁琐地一次次下载、安装。

#### 1.1. YUM的作用

在Linux上使用源码的方式安装软件非常满分，使用yum可以简化安装的过程

<br>

### 2.  YUM的常用命令

安装httpd并确认安装

yum instll -y httpd

 <br>

列出所有可用的package和package组

yum list

 <br>

清除所有缓冲数据

yum clean all

 <br>

列出一个包所有依赖的包

yum deplist httpd

 <br>

删除httpd

yum remove httpd

<br>

### 3.1.  制作本地YUM源

#### 3.1 为什么要制作本地YUM源

​	YUM源虽然可以简化我们在Linux上安装软件的过程，但是生成环境通常无法上网，不能连接外网的YUM源，说以接就无法使用yum命令安装软件了。为了在内网中也可以使用yum安装相关的软件，就要配置yum源。

#### 3.2YUM源的原理

YUM源其实就是一个保存了多个RPM包的服务器，可以通过http的方式来检索、下载并安装相关的RPM包

{% asset_img lld.png %}

#### 3.3. 制作本地YUM源

1.准备一台Linux服务器，用最简单的版本CentOS-6.7-x86_64-minimal.iso

2.配置好这台服务器的IP地址

3.上传CentOS-6.7-x86_64-bin-DVD1.iso到服务器

4.将CentOS-6.7-x86_64-bin-DVD1.iso镜像挂载到某个目录

mkdir /var/iso

mount -o loop CentOS-6.7-x86_64-bin-DVD1.iso /var/iso

5.修改本机上的YUM源配置文件，将源指向自己

备份原有的YUM源的配置文件

cd /etc/yum.repos.d/

rename .repo .repo.bak *

vi CentOS-Local.repo

~~~shell
[base]
name=CentOS-Local
baseurl=file:///var/iso
gpgcheck=1
enabled=1   #很重要，1才启用
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
~~~

添加上面内容保存退出

6.清除YUM缓冲

yum clean all

7.列出可用的YUM源

yum repolist

8.安装相应的软件

yum install -y httpd

9.开启httpd使用浏览器访问<http://192.168.0.100:80>（如果访问不通，检查防火墙是否开启了80端口或关闭防火墙）

service httpd start

10.将YUM源配置到httpd（ApacheServer）中，其他的服务器即可通过网络访问这个内网中的YUM源了

cp -r /var/iso/ /var/www/html/CentOS-6.7

11.取消先前挂载的镜像

umount /var/iso

12.在浏览器中访问http://192.168.0.100/CentOS-6.7/

{% asset_img lld1.png %}

13.让其他需要安装RPM包的服务器指向这个YUM源，准备一台新的服务器，备份或删除原有的YUM源配置文件

cd /etc/yum.repos.d/

rename .repo .repo.bak *

vi CentOS-Local.repo

~~~shell
[base]
name=CentOS-Local
baseurl=http://192.168.0.100/CentOS-6.7
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
~~~

添加上面内容保存退出

14.在这台新的服务器上执行YUM的命令

yum clean all

yum repolist

15.安装相应的软件

yum install -y gcc

16、加入依赖包到私有yum的repository

进入到repo目录

执行命令：  createrepo .

<br>

### 补充：本地yum仓库的安装配置

两种方式： 

- **a** 、每一台机器都配一个本地文件系统上的yum仓库 file:///packege/path/
- **b** 、在局域网内部配置一台节点(server-base)的本地文件系统yum仓库，然后将其发布到web服务器中，其他节点就可以通过http://server-base/pagekege/path/

制作流程：  先挑选一台机器mini4，挂载一个系统光盘到本地目录/mnt/cdrom，然后启动一个httpd服务器，将/mnt/cdrom 软连接到httpd服务器的/var/www/html目录中 (cd /var/www/html; ln -s /mnt/cdrom ./centos )

然后通过网页访问测试一下：  <http://mini4/centos>   会看到光盘的目录内容

至此：网络版yum私有仓库已经建立完毕  

剩下就是去各台yum的客户端配置这个http地址到repo配置文件中

<br>

无论哪种配置，都需要先将光盘挂在到本地文件目录中

mount -t iso9660 /dev/cdrom   /mnt/cdrom

为了避免每次重启后都要手动mount，可以在/etc/fstab中加入一行挂载配置，即可自动挂载

vi  /etc/fstab

~~~
/dev/cdrom              /mnt/cdrom              iso9660 defaults        0 0	
~~~
