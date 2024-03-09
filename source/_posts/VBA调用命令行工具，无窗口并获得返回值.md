toc: true
title: VBA调用命令行工具，无窗口并获得返回值
date:  2022-03-04 15:01 
tags: [VBA, Powershell, wscript.run]
description:

---

# VBA调用命令行工具，无窗口并获得返回值

wscipt.run 能隐藏窗口调用命令行工具，返回命令行工具执行状态
wscript.exec 能返回命令行工具stdout，但不能隐藏命令行窗口

可以将命令行输出结果重定向到剪贴板，再用vba获取剪贴板内容

<!--more-->

```
    Dim objWsh As Object
    
    Cmdread = "powershell Invoke-Expression 'dir | Set-Clipboard'"

    ' 使用wscript.shell的run方法，可以隐藏powershell的窗口，后面的true表示等待进程结束
    Application.StatusBar = "运行命令...."
    Set objWsh = CreateObject("WSCript.shell")
    objWsh.Run Cmdread, 0, True
    
    ' 获取剪贴板内容，写入对应单元格
    Application.StatusBar = False
    Sheet1.Range("A1").PasteSpecial
```

