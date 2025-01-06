---
title: Linux应急响应命令速查
date: 2022-10-10 16:54:50
categories: 渗透测试
tags: 应急响应
---

# Linux应急响应命令速查

## 前言

再处理应急响应事件中，很多命令过长而难以记住，遂将相关应急响应命令做一个整合，方便后续应急事件的处理。所有命令来源于网络及各位大佬的收集整理。

<!--more-->

## 挖矿及勒索远控排查

挖矿一般导致CPU或者GPU占用过高，重点排查进程的相关资源占用及网络连接，同时对计划任务和启动项做检查和清理。

### 0x1 资源占用情况

1.cpu占用

`top -c -o %CPU`

查看cpu占用前5的进程

`ps -eo pid,ppid,%mem,%cpu,cmd --sort=-%cpu | head -n 5`

2.内存占用

`top -c -o %MEM`

`ps -eo pid,ppid,%mem,%cpu,cmd --sort=-%mem | head -n 5`

### 0x2 寻找恶意样本

根据进程名字获取PID

* `pidof "procee_name"`
* `ps -aux | grep "name"`
* `pgrep -f "name"`

根据PID获取进程详细信息

* `lsof -p pid`
* `pwdx pid` //启动恶意程序的文件路径

根据pid查看进程拉起的线程

* `ps H -T -p pid`
* `ps -Lf pid`
* `pstree -agplU`

查看程序运行时间

`ps -eo pid,lstart,etime,cmd | grep pid`

查看文件相关操作时间

`stat filename`

文件查找

```
文件修改
find / -mtime -3 //最近3天修改过的文件
find / -mtime +3 //3天前修改过的文件
find / -mtime -1  //最近24小时修改过的文件
find / -mmin +3  //三分钟前编辑过的文件
find /var/www/html/ -mtime -1 -name *.php
文件访问
find / -atime -3 //最近3天访问过的文件
find / -atime -24 //最近24小时访问的文件

属性修改
find / -cmin -3 //3分钟内修改属性的文件

文件大小
find / -size 10M //寻找10m的文件
find / -size +10M  //大于10m的文件
find / -size -10M //小于10m的文件
find / -size +10MB -20M //介于10m和20m的文件


```



### 0x3 恶意进程处理

查看进程及子进程

`ps ajfx`

杀死进程

`kill -9 pid`

杀死进程及进程组

`kill -9 -1134`//进程pid前面有个**-**

查看文件占用

`lsof file.sh`

文件加锁

* a属性，只能增加内容，无法修改和删除之前的文件
* i属性，无法改变内容及删除文件

`chattr -a`

`chattr -i`

### 0x4 网络连接

`netstat -pantu | grep 8.8.8.8`

`netstat -pantu | grep 80`

根据文件找pid

`lsof | grep install.sh`

`lsof /root/start.sh`

### 0x5 计划任务

列出当前用户计划任务

`crontab -l`

删除计划任务

`crontab -r`

检查如下目录是否存在恶意脚本

```bash
/var/spool/cron/*
/var/spool/anacron/*
/etc/crontab
/etc/cron.d/*
/etc/cron.daily/*
/etc/cron.hourly/*
/etc/cron.monthly/*
/etc/cron.weekly/
/etc/anacrontab
```

开机启动项

* `/etc/init.d`
* /etc/rc.d/*
* /etc/rc.local
* /etc/rc[0-6].d
* /etc/inittab

## 暴力破解



### 0x1  SSH

1.查找特殊权限的账号，默认root

`awk -F: '{if($3==0) print $1}' /etc/passwd`

![image-20221007154207343](http://qn.laohuan.xin/image-20221007154207343.png)

2.查找可以登陆ssh的账号

```bash
s=$( sudo cat /etc/shadow | grep '^[^:]*:[^\*!]' | awk -F: '{print $1}');for i in $s;do cat /etc/passwd | grep -v "/bin/false\|/nologin"| grep $i;done | sort | uniq |awk -F: '{print $1}'
```

3.查看相关登陆session

* `who -a`
* `w`
* `last -p now`
* `sudo netstat -tnpa | grep 'ESTABLISHED.*sshd'`
* `echo $SSH_CONNECTION`
* `ss | grep ssh`

**日志相关**

日志位置

* Ubuntu

  `/var/log/auth.log`

* Centos

  `/var/log/secure`

快捷命令

* `lastb` //登陆失败记录
* `lastlog` //最后一次登陆
* `last` //登陆成功记录

* 查看登陆成功的日志

`cat /var/log/auth.log | grep "Accept"`

![image-20221007160431335](http://qn.laohuan.xin/image-20221007160431335.png)

* 查看登陆失败的日志

`cat /var/log/auth.log | grep "Failed password for" | more`

![image-20221007160653206](http://qn.laohuan.xin/image-20221007160653206.png)

* 统计暴力破解的用户名

```bash
grep "Failed password" /var/log/auth.log|perl -e 'while($_=<>){/for(.*?)from/; print "$1\n";}'|sort|uniq -c|sort -nr
```

![image-20221007160924764](http://qn.laohuan.xin/image-20221007160924764.png)

invaild user说明此用户不存在，但是却出现登陆记录

* 统计用root爆破的IP

  ```bash
  cat /var/log/auth.log | grep "Failed password for" | grep "root" | grep -Po '(1\d{2}|2[0 4]\d|25[0-5]|[1-9]\d|[1-9])(\.(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|\d)){3}' |sort|uniq -c|sort -nr
  ```

  ![image-20221007161242953](http://qn.laohuan.xin/image-20221007161242953.png)

* 查询所有登陆失败的IP

  ```bash
  sudo cat /var/log/auth.log | grep "Failed password for" | cut -d " " -f 9 | sort -nr | uniq|grep -v "invalid"| while read line;do echo [$line];sudo cat /var/log/auth.log | grep "Failed password for" | grep $line | grep -Po '(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|[1-9])(\.(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|\d)){3} '|sort|uniq -c |sort -nr; done
  ```

  ![image-20221007161638799](http://qn.laohuan.xin/image-20221007161638799.png)

* 登陆成功的日期、用户、IP

  ```bash
  grep "Accepted " /var/log/auth.log | awk '{print $1,$2,$3,$9,$11}'
  ```

* 查看拥有sudo权限的账号

  `cat /etc/sudoers | grep -v "^#\|^$" | grep "ALL=(ALL)"`



## Web日志分析

```bash
1、列出当天访问次数最多的IP命令：
cut -d- -f 1 log_file|uniq -c | sort -rn | head -20

2、查看当天有多少个IP访问：
awk '{print $1}' log_file|sort|uniq|wc -l

3、查看某一个页面被访问的次数：
grep "/index.php" log_file | wc -l

4、查看每一个IP访问了多少个页面：
awk '{++S[$1]} END {for (a in S) print a,S[a]}' log_file

5、将每个IP访问的页面数进行从小到大排序：
awk '{++S[$1]} END {for (a in S) print S[a],a}' log_file | sort -n

6、查看某一个IP访问了哪些页面：
grep ^111.111.111.111 log_file| awk '{print $1,$7}'

7、去掉搜索引擎统计当天的页面：
awk '{print $12,$1}' log_file | grep ^\"Mozilla | awk '{print $2}' |sort | uniq | wc -l

8、查看2018年6月21日14时这一个小时内有多少IP访问:
awk '{print $4,$1}' log_file | grep 21/Jun/2018:14 | awk '{print $2}'| sort | uniq | wc -l

9、查看某天IP访问量
grep '23/May/2019' /www/logs/access.log | awk '{print $1}' | awk -F'.' '{print $1"."$2"."$3"."$4}' | sort | uniq -c | sort -r -n | head -n 10

10、统计网段
cat /www/logs/access.log | awk '{print $1}' | awk -F'.' '{print $1"."$2"."$3".0"}' | sort | uniq -c | sort -r -n | head -n 200

11、统计域名
cat  /www/logs/access.log |awk '{print $2}'|sort|uniq -c|sort -rn|more

12、统计http状态码
cat  /www/logs/access.log |awk '{print $9}'|sort|uniq -c|sort -rn|more

13、url统计
cat  /www/logs/access.log |awk '{print $7}'|sort|uniq -c|sort -rn|more

14、url访问量统计
cat /www/logs/access.log | awk '{print $7}' | egrep '\?|&' | sort | uniq -c | sort -rn | more

15、url提取
tail -f /www/logs/access.2019-02-23.log | grep '/index.html' | awk '{print $1" "$7}'	

```





