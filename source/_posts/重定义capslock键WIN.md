toc: true
title: 重定义capslock键WIN
date: 2015-12-04 16:31:06
tags: [ahk, capslock]
---
使用ahk在windows重定义capslock
<!--more-->
# 1. 目的
 
CapsLock位于键盘的黄金位置，平常却没有什么用处，修改CapsLock键，使得
 
+ 单独按CapsLock键，映射为`<Esc>`
+ 组合按CapsLock键，如`CapsLock-C`，映射为`<Ctrl-C>`
 
# 2. 所需软件
 
+ AutoHotKey, http://ahkscript.org/
 
# 3. 配置
编辑文件~/bin/caps.ahk，代码为
 
```
#InstallKeybdHook
SetCapsLockState, alwaysoff
Capslock::
    Send {LControl Down}
    KeyWait, CapsLock
    Send {LControl Up}
    if ( A_PriorKey = "CapsLock" )
    {
        Send {Esc}
    }
    return
```
存盘，关联ahk到autohotkey，放入自启动文件夹
