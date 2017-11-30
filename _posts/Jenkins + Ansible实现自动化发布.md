---
title: Jenkins + Ansible实现自动化发布
date: 2016-12-21 11:14:33
categories:
- Linux
tags:
- Operations
- 自动化
- 运维
---
# Jenkins + Ansible实现自动化发布
前期在生产环境中进行应用发布时，采用纯粹手工的方式从测试环境拷贝WAR到生产环境目录进行备份、部署、重启服务等操作。为解决批量部署的效率问题，以及尽可能避免人工操作造成的不可预知的失误，准备采用自动化的方式来实现应用的发布。
> **本文档适用的前提：**
测试环境已经通过CI工具完成自动化编译、打包，并生成了路径、命名固定的WAR包，生产环境只需拷贝该WAR包到指定目录解压即可的场景。

## Jenkins安装
>**部署环境：**
操作系统：ubuntu 14.04
服务器IP：192.168.1.1

**第一种方式**
前往Jenkins官网https://pkg.jenkins.io/debian-stable/，将官方Jenkins key引入。

```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
```
然后将其加入源列表。

```
deb https://pkg.jenkins.io/debian-stable binary/
```
最后，更新本地包索引，并安装稳定版Jenkins

```
sudo apt-get update
sudo apt-get install jenkins
```
<!--more-->
**第二种方式**
前往Jenkins官网https://pkg.jenkins.io/debian-stable/，下载deb包

```
wget https://pkg.jenkins.io/debian-stable/binary/jenkins_2.19.4_all.deb
```

安装deb包

```
dpkg -i jenkins_2.19.4_all.deb
```
按照操作提示步骤进行安装。完成后通过以下命令进行启动

```
/etc/init.d/jenkins start
```

启动后，Jenkins默认将会监听8080端口。

## 安装Ansible
过程略
Ansible安装完成后，需要做以下工作：
### 实现被控服务器免密登陆
在Ansible主控服务器的Jenkins用户下，执行以下命令生产密钥对：

```
ssh-keygen -t rsa
```
将jenkins用户的公钥拷贝到被控服务器的ssh目录下：

```
ssh-copy-id -i .ssh/id_rsa.pub app@192.168.1.2
ssh-copy-id -i .ssh/id_rsa.pub app@172.16.1.2
```
至此，Ansible能够通过jenkins用户免密登陆应用服务器生产、测试环境。
## Jenkins调用Ansible
首先，需要在Jenkins上安装Ansible的插件，名为 **Ansible plugin**，在插件管理界面搜索安装即可。
然后，在Jenkins新建**Freestyle Project**

![Alt text](/images/jenkins_ansible_autodeploy/jenkins_invoke_ansible.png)

调用的Ansible-playbook示例如下：

```
- hosts: 192.168.1.2
  vars:
    testIP: 172.16.1.2
    testhome: /home/app/api-tomcat/webapps
    warname: api.war
    oldhome: /home/app/api-tomcat
    backupwebapps: /home/app/tomcat.bak
    newwar: /home/app/newwar
    zipname: api
  remote_user: app
  tasks:
    - name: 删除/home/app/newwar目录
      file: path={{ newwar }} state=absent
      ignore_errors: yes
    - name: 创建/home/app/newwar目录.改权限
      file: path={{ newwar }} recurse=yes mode=775 owner=app group=app state=directory
    - name: 从测试环境复制war包到/home/app/newwar目录
      shell: scp app@{{ testIP }}:{{ testhome }}/{{ warname }} {{ newwar }}
    - name: 给/home/app/newwar递归改权限
      file: dest={{ newwar}} recurse=yes mode=775 owner=app group=app
    - name: 解压/home/app/newwar目录内的war包在本目录内
      shell: unzip -oq {{ newwar }}/{{ warname }} -d {{ newwar }}/{{ zipname }}
    - name: 再次给/home/app/newwar递归改权限
      file: dest={{ newwar}} recurse=yes mode=775 owner=app group=app
    - name: 创建备份webapps目录/home/app/tomcat.bak并改权限
      file: path={{ backupwebapps }}/{{ zipname }} recurse=yes mode=775 owner=app group=app state=directory
    - name: 备份webapps到目录/home/app/tomcat.bak下并加上时间戳
      shell: cp -a {{ oldhome }}/webapps {{ backupwebapps }}/{{ zipname }}/webapps-`date +%Y%m%d%H%M`
    - name: kill进程方式停止服务.忽略错误返回值
      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
      ignore_errors: yes
    - name: kill进程方式停止服务.忽略错误返回值
      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
      ignore_errors: yes
    - name: 再次kill进程方式停止服务.忽略错误返回值
      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
      ignore_errors: yes
    - name: 查看停止服务的结果.进程是否还在
      shell: ps -ef | grep {{ oldhome }}
    - name: 删除原来的webapps目录下的war包
      file: path={{ oldhome }}/webapps/{{ warname }} state=absent
      ignore_errors: yes
    - name: 删除原来的webapps目录下的程序目录
      file: path={{ oldhome }}/webapps/{{ zipname }} state=absent
      ignore_errors: yes
    - name: 复制/home/app/newwar目录下的解压的新程序目录到webapps目录下
      shell: cp -a {{ newwar }}/{{ zipname }} {{ oldhome }}/webapps/
    - name: 复制/home/app/newwar目录下的war包到webapps目录下
      shell: cp -a {{ newwar }}/{{ warname }} {{ oldhome }}/webapps/
    - name: 启动服务
      shell: "source /etc/profile;nohup {{ oldhome }}/bin/startup.sh &"
    - name: 查看进程中是否存在启动的服务
      shell: ps -ef | grep {{ oldhome }}
```

执行RUN，开始部署操作：
![Alt text](/images/jenkins_ansible_autodeploy/run_project.png)

然后就可以在CONSOLE界面看到执行的结果：
![Alt text](/images/jenkins_ansible_autodeploy/run_project_console_output.png)