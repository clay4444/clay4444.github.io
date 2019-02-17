---
title: Linux常用命令
categories:
  - Linux
abbrlink: d0edc1ed
date: 2017-09-20 08:20:18
---

### sudo

临时借用root的权限执行命令,只在当前命令下有效。命令结束后，还是原来用户。

1.配置当前用户具有sudo的执行权利

​	[/etc/sudoers]

​	...

​	root ALL=(ALL) ALL

​	centos ALL=(ALL) ALL

​	...

$>sudo chown -R centos:centos 

<br/>

### job

放到后台运行的进程.

1.将程序放到后台运行,以&结尾.

​	$>nano b.txt &

2.查看后台运行的jobs数

​	$>jobs

3.切换后台作业到前台来.

​	$>fg %n		//n是job编号.

4.前台正在的进程，放到后台。

​	ctrl + z

> 如果希望其在后台运行，还需要使用bg命令并指定其Ctrl+Z得到的任务号，才可以在后台运行。		
>
> Ctrl + z 只是挂起

5.让后作业运行

​	$>bg %1		//	

6.杀死作业

​	$>kill %1	//

<br/>

### 进程查看,prcess show

$>ps -Af |grep gnome		//-A:所有进程  -f:所有列格式.

$>top					//动态显示进程信息。含有cpu、内存的使用情况.

​						//q,按照q退出。

<br/>

### netcat：  瑞士军刀

- [server]

  nc -lk 8888	//-l : 监听

  ​			//-k : 接受多个连接

- [client]

  nc ip 8888 ;	//客户端指定服务器端

- [windows下]

1.配置环境变量path

2.常用命令

​	cmd>nc -h						//看帮助	

3.启动服务器端

​	cmd>nc -l -p 8888 -s 0.0.0.0	/通配ip

<br/>

### 查看端口

netstat -anop	//显式网络情况

​				//-a : 所有socket

​				//-n : 显式数字地址

​				//-p : pid

​				//-o : timer

查看8004端口占用：

netstat -anop | grep 8004

<br/>

### 免密登陆配置

假如 A  要登陆  B

在A上操作：

首先生成密钥对

**ssh-keygen**   (提示时，直接回车即可)

再将A自己的公钥拷贝并追加到B的授权列表文件authorized_keys中

**ssh-copy-id**   B

{% asset_img ssh免密登陆机制示意图.png %}