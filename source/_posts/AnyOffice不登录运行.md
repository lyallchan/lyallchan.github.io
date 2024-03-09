toc: true
title: Anyoffice不登录运行
date: 2018-01-05 11:30:06
tags: [Anyoffice, 登录]
description: 

---

# AnyOffice不登录运行

工作需要，开机后不要人工操作，就运行Anyoffice。而Anyoffice的启动文件WinGUI.exe又必须在用户登陆后再运行。

使用计划任务的“不管用户是否登录都要运行”的功能，可以实现开机，Anyoffice就以特定用户运行的目的。

<!--more-->

# 操作

计划任务 -> 创建任务 -> 名称随便输入，选择“不管用户是否登录都要运行” -> 触发器选择“启动时” -> 操作选择“启动程序”并选择WinGUI.exe程序，确定，即可。

WINGUI.exe运行时，要设置自动登录。
