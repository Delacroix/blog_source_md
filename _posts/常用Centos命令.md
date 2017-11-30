---
title: 常用Centos命令
date: 2016-11-04 10:56:52
categories:
- Linux
tags: 
- 基础
---

#### 安装方式：
**• yum安装[需联网]**
`yum install vim` [yum方式安装vim软件及相关依赖包，如有更新包，则提示是否更新]
`yum search vim` [yum方式搜索vim软件是否存在]
`yum list |grep vim` [yum方式列出vim安装状态、版本及包全名]

<!--more-->

**• rpm安装**
`rpm -ivh vim-minimal.x86_64.rpm` [rpm方式安装vim软件，如差依赖包，会有提示]
`rpm -qa |grep vim` [查询系统是否安装vim软件，有则显示包全名及版本号]
`rpm -ql yum` [列出yum软件所安装的文件]
**• 源码安装[常规安装方式]**
`./configure` 配置源码包相关安装 –help 打印帮助文档
`make` 编译代码
`make install` 安装软件及输出安装信息
>注：Centos6及以下系统系统使用的pyton2.6，Centos7使用的python2.7，yum命令会使用python命令，系统版本对应python版本，请勿随意升级
 
#### 查看端口：
**• netstat[Centos6及以下]**
`netstat -nltp` [打印TCP协议，状态为LISTEN的端口信息，并打印PID号]
`netstat -nultp` [打印TCP、UDP协议，状态为LISTEN的端口信息，并打印PID号]
`netstat -anp` [打印TCP、UDP、SOCKET协议所有状态，并打印PID号]
`netstat -rn` [打印路由表信息]
`netstat -anp |awk ‘$1==”tcp”{print $6}’ |sort -n |uniq -c |sort -n` [打印TCP连接状态]
>注：UDP无LISTEN状态、LISTEN状态仅适用于TCP

**• ss[Centos7]**
ss命令与netstat命令基本类似，参考以上命令
 
#### 查看网络地址：
**• ifconfig[Centos6及以下]**
`ifconfig -a` [打印所有网络设备信息]
`ifconfig eth0` [打印eth0网卡信息]
`ifconfig eth0 up/down` [启用或停用eth0网卡]
`ifconfig eth0 mtu 1316` [更改eth0网卡MTU值为1316]
>注：建立VPN服务器，如果客户机不能上网，需确认MTU值是否正确

**• ip[Centos7]**
ip命令与ifconfig命令基本类似，参考以上命令
**• route**
`route -n` [打印路由表信息]
>注：添加/删除路由表需使用route命令
 
#### 查看进程：
**• ps**
`ps aux |grep bash`[查看bash进程信息，打印启动进程用户，PID，启动命令等]
`ps -ef |grep bash` [查看bash进程信息，打印UID，PID，PPID，启动命令等]
**• pstree**
`pstree -a` [打印进程之间子父关系]
**• top**
`top M` [以兆字节显示系统硬件相关信息]
 
#### 查看文件占用：
**• lsof**
`lsof -Pi :10086` [查看10086端口连接情况，打印启动进程用户，PID，协议，命令等]
`lsof -p 19523` [查看19523进程号信息，打印启动进程用户，PID，命令，类型，调用文件或目录情况等]
`lsof redis.sh` [查看redis.sh文件被调用情况，打印命令，PID，启动进程用户等]
`lsof +d /mnt` [查看/mnt目录被调用情况，打印命令，PID，启动进程用户，启动命令等]
`lsof +D /mnt` [查看/mnt目录被调用情况，打印命令，PID，启动进程用户，启动命令等]
>注：小写d不包含子目录，大写D包含子目录下文件

**• fuser**
`fuser -nv tcp 53` [打印TCP协议端口为53的相关信息]
`fuser -v 53/tcp` [同上]
**• ldd**
`ldd -v /usr/bin/vim` [打印vim命令共享库调用情况]
>注：太多命令会调用GLIBC库，请勿随意升级或覆盖安装
 
#### 查找文件：
**• find**
`find . -type f -name “*tw*”` [查找当前目录及子目录下文件名含有tw的文件]
`find / -maxdepth 3 -type f -name “*.log”` [查找/目录下文件名为log结尾的文件，深度只查找到第3层]
`find . -type d -name “*log*”` [查找当前目录及子目录文件夹名含有log的文件夹]
 
#### 查找文件内容：
**• grep**
`grep “dir” *` [查找当前目录所有含有dir字符的文件，不扫描子目录]
**• find+grep**
`find . |xargs grep “path”` [查找当前目录及子目录所有文件含有path字符的文件]
>注：请写明具体路径，不要直接填写为/，这会导致IO和CPU占用过高

