---
title: ELK完成操作系统日志审计
date: 2017-11-24 11:12:30
tags:
- ELK
- 审计
- 日志分析
---

# 一、filebeat 版本、安装地址等情况
1、filebeat 版本为：5.5.2
2、filebeat 由二进制包安装（filebeat-5.5.2-linux-x86_64.tar.gz，解压后直接可用）
3、filebeat 安装目录：/usr/local/filebeat
4、filebeat 二进制包存放地址：192.168.13.45 中 / home/app/python/python_connection/file_put 下

# 二、安装步骤
 1、在各服务器中的 / etc/bashrc 文件中追加有一下命令

```
##usermonitor----------------------------------------------
##modify by huqing at 2016---------------------------------
STIME=`date -d today +"%Y-%m-%d"`
HISTSIZE=1000
HISTTIMEFORMAT="%Y/%m/%d %T ";
export HISTORY_FILE=/usermonitor/audit172_19_2_49.log.${STIME}.`id -un`
export HISTTIMEFORMAT
export PROMPT_COMMAND='{ thisHistID=`history 1|awk "{print \\$1}"`;
lastCommand=`history 1| awk "{\\$1=\"\" ;print}"`;
user=`id -un`;whoStr=(`who -u am i`);
realUser=${whoStr[0]};
logMonth=${whoStr[2]};
logDay=${whoStr[3]};
logTime=${whoStr[4]};
pid=${whoStr[5]};
ip=${whoStr[6]};
if [ ${thisHistID}x != ${lastHistID}x ];
         then
echo -E "[OPERATE USER:"$user]"[LOGIN USER:"$realUser]"[LOGIN SOURCE IP:"$ip][LOGIN PID:$pid][LOGIN TIME:$logMonth $logDay $logTime] ------"[OPERATE TIME AND COMMAND:"$lastCommand ];lastHistID=$thisHistID;
fi; } >> $HISTORY_FILE'
```

2、修改 / etc/bashrc 文件中追加命令中的 ip 地址：

```
audit172_19_2_49.log.${STIME}.`id -un`   
```

中的 172_19_2_49 修改为本机 ip，该 ip 为日志名字

3、创建 / usermonitor 目录，日志存放在该目录下

4、解压 filebeat 二进制文件, 并移动到 / usr/local/filebeat

5、修改 / usr/local/filebeat/filebeat.yml 文件为以下配置，并把 name 的值改为本机 ip 地址

```
filebeat.prospectors:
- input_type: log
paths:
- /usermonitor/*
name: "172.19.2.49"
output.logstash:
hosts: ["172.19.2.51:6666"]
```

6、执行 `source /etc/bashrc`
7、启动 filebeat
 

```
chmod u+x /usr/local/filebeat/filebeat
nohup /usr/local/filebeat/filebeat -c /usr/local/filebeat/filebeat.yml &
```

 
# 三、python 脚本
1、python 脚本地址
 

```
/home/app/python/python_connection/connection_ssh.py
/home/app/python/python_connection/show.py
```

 
2、配置文件地址及配置格式要求

```
/home/app/python/python_connection/ip.txt
```
格式要求：ip, 端口普通用户名, 普通用户密码, root 用户密码
     注意中间必须用英文符号的逗号隔开，末尾不能有空格
     如：192.168.1.1,22,app,appmima,rootmima

3、运行脚本

```
python show.py
```