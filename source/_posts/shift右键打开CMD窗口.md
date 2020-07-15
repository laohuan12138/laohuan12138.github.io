---
title: shift右键打开CMD窗口
date: 2020-07-15 21:49:45
categories: 渗透测试
tags: 渗透测试技巧
---

在windows更新到Win10 1703后，<kbd>Ctrl</kbd>+<kbd>右键</kbd>变成了“在此处打开 Powershell 窗口”,然而在平时的工作中，并不需要强大的powershell，CMD就能够满足我们的大部分工作且启动速度较快。

<!--more-->

修改注册表使<kbd>ctrl</kbd>+<kbd>右键</kbd>能打开CMD

1.<kbd>windows</kbd>+<kbd>R</kbd> 输入**regedit**打开注册表编辑器，找到如下键值

`计算机\HKEY_CLASSES_ROOT\Directory\Background\shell\cmd`

2.将默认项`HideBaseOnVelocityId`，修改为`ShowBaseOnVelocityId`

![](http://cdn.laohuan.art/2020-07-15_22-09-38.png)

3.若提示''重命名值时产生错误''

选中CMD项，右键 → 选择权限 → 高级 → 更改所有者 → 将所有者修改成你的当前用户

4.此时Ctrl+右键就可以打开CMD了

![](http://cdn.laohuan.art/2020-07-15_22-15-02.png)



