toc: true
title: 我的autohotkey设置（二）
date: 2016-02-09 13:23:01
tags: [autohotkey, wiz]
 
---
这里记录一些基于窗口的，在特定应用中生效的快捷键
[TOC]
 
<!--more-->
 
# wiz的阅读窗口实现hjkl
 
```
;; wiz正文阅读控件 : hjkl为方向键
;; 只在阅读控件中起作用
#If WinActive("ahk_class WizNoteMainFrame")
 
    j::ControlMapKey("WebViewHost", "j", "{down}")
    k::ControlMapKey("WebViewHost", "k", "{up}")
    g::ControlMapKey("WebViewHost", "g", "{home}")
    +g::ControlMapKey("WebViewHost", "+g", "{end}")
 
    return
#if
```
 
# TC的Lister F3关闭
 
```
;; keys tlister from tc
;; f3 : 发送esc，相当于关闭
#If WinActive("ahk_group tc")
 
    F3:: send, {Esc}
 
#If
```
 
# PDFXview实现hjkl
 
```
;; PDFXview : hjkl为方向键
#If WinActive("ahk_class DSUI:PDFXCViewer")
 
    j::send, {down}
    k::send, {up}
    g::send, {home}
    +g::send, {end}
 
    return
#if
```
 
# chrome IE的F3 F4
 
```
;; keys for browsers
;; F3 : 下一个标签
;; F4 : 关闭当前标签
#If WinActive("ahk_group win_browser")
 
    F3:: ^tab
    F4:: ^w
 
#If
```
