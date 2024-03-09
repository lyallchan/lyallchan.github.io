toc: true
title: 我的autohotkey设置
date: 2016-02-06 13:21:04
tags: [autohotkey]
 
---
[TOC]
我的autohotkey启动文件，包含了日常使用的快捷键，拿出来分享
 
<!--more-->
 
# Capslock的处理
 
实现单独按下Capslock为esc，Capslock加上任意键实现CTRL键功能，Capslock+hjkl，Capslock+a，Capslock+e实现编辑状态下的光标键移动，Capslock+q实现alt-F4功能
 
原先Capslock转为shift+Capslock
 
```
$+Capslock::
    SetCapsLockState % getkeystate("Capslock", "t") ? "off" : "on"
    return
$Capslock::
    Gui, 93:+Owner ; prevent display of taskbar button
    Gui, 93:Show, y-99999 NA, capslock-hole
    Send {LCtrl Down}
    KeyWait, Capslock ; wait until the Capslock button is released
    Gui, 93:Cancel
    Send, {LCtrl Up}
    ifinstring, A_PriorKey, Capslock
        Send, {ESC}
Return
#If WinExist("capslock-hole")
;; Capslock-2 : mintty
;; Capslock-1 : TC/Start/Min/Active
    *1::
        Send {LCtrl Up}
        ActiveTC()
        Send {LCtrl Down}
        return
    *2::
        Send {LCtrl Up}
        ActiveMintty()
        Send {LCtrl Down}
        return
#If
#If WinExist("capslock-hole") and Not WinActive("ahk_class mintty")
    *h::Send {Blind}{LCtrl Up}{Left}{LCtrl Down}
    *j::Send {Blind}{LCtrl Up}{Down}{LCtrl Down}
    *k::Send {Blind}{LCtrl Up}{Up}{LCtrl Down}
    *l::Send {Blind}{LCtrl Up}{Right}{LCtrl Down}
    *g::Send {Blind}{LCtrl Up}{Home}{LCtrl Down}
    *a::Send {Blind}{LCtrl Up}{Home}{LCtrl Down}
    *e::Send {Blind}{LCtrl Up}{End}{LCtrl Down}
    *q::Send {Blind}{LCtrl Up}{LAlt Down}{F4}{LAlt Up}{LCtrl Down}
#If ; end context-sensitive block
```
 
其中的关键是Capslock+hljkae的处理，使用了一个隐藏窗口，Capslock按下时，隐藏窗口生效，在隐藏窗口生效的情况下，定义hljkae，相当于产生了一个Capslock的漏洞，这个漏洞中可以自定义个别按键
 
# 窗口的处理
 
实现任意窗口置顶、透明化、最大化、最小化、虚拟桌面切换
 
| 置顶           | 透明化增加      | 透明化减少        | 最大化          | 最小化   | 虚拟桌面切换    |
| -------------- | --------------- | ----------------- | --------------- | -------- | --------------  |
| alt-0          | alt-=           | alt--             | alt-enter       | alt-m    | 双击Fn          |
| win-0          | win-=           | win--             | win-enter       | win-m    | F11             |
| alt-鼠标右键   | 标题栏-鼠标up   | 标题栏-鼠标down   |                 |          | 任务栏-鼠标滚动 |
| 双击shift      |                 |                   |                 |          |                 |
 
 
## 窗口置顶
 
```
!Rbutton::
    MouseGetPos, ox, oy, ow, oc
    WinTopToggle(ow)
    return
<Shift::
    if (A_PriorHotKey = "<Shift" AND A_TimeSincePriorHotKey < 500)
    {
        WinGet ow, id, A
        WinTopToggle(ow)
    }
    return
#0::
!0::
    WinGet ow, id, A
    WinTopToggle(ow)
    return
WinTopToggle(w) {
 
    WinGetTitle, oTitle, ahk_id %w%
    Winset, AlwaysOnTop, Toggle, ahk_id %w%
    WinGet, ExStyle, ExStyle, ahk_id %w%
    if (ExStyle & 0x8)  ; 0x8 为 WS_EX_TOPMOST.在WinGet的帮助中
        oTop = 置顶
    else
        oTop = 取消置顶
    tooltip %oTitle% %oTop%
    SetTimer, RemoveToolTip, 5000
    return
 
    RemoveToolTip:
    SetTimer, RemoveToolTip, Off
    ToolTip
    return
}
 
```
 
win+鼠标右键 : 光标所在窗口置顶
双击shift : 最上面的窗口置顶
win+0 : 当前窗口置顶
shift+alt+0 : 当前窗口置顶
置顶时，有tooltip显示窗口名称以及状态，5s后消失
 
## 窗口透明化
 
```
WheelUp::
    MouseGetPos,ox,oy,ow,oc
    WinGetClass, owc, ahk_id %ow%
    Ifinstring, owc, Shell_TrayWnd
        VDSwitch(1)
    else
    {
        If (oy>7) and (oy<30)
            WinTransplus(ow)
        else
            send, {WheelUp}
        sleep 30
    }
    return
#=::
!=:: ;; alt-=
    WinGet, ow, id, A
    WinTransplus(ow)
    return
WheelDown::
    MouseGetPos, ox, oy, ow, oc
    WinGetClass, owc, ahk_id %ow%
    Ifinstring, owc, Shell_TrayWnd
        VDSwitch(0)
    else
    {
        If (oy>7) and (oy<30)
            WinTransMinus(ow)
        else
            send, {WheelDown}
        sleep 30
    }
    return
#-::
!-:: ;; alt--
    WinGet, ow, id, A
    WinTransMinus(ow)
    return
;; WinTransplus WinTransMinus 对ahk_id为w的窗口
;; 进行透明度增减，每次幅度为10
WinTransplus(w){
 
    WinGet, transparent, Transparent, ahk_id %w%
    if transparent < 255
        transparent := transparent+10
    else
        transparent =
    if transparent
        WinSet, Transparent, %transparent%, ahk_id %w%
    else
        WinSet, Transparent, off, ahk_id %w%
    return
}
WinTransMinus(w){
 
    WinGet, transparent, Transparent, ahk_id %w%
    if transparent
        transparent := transparent-10
    else
        transparent := 240
    WinSet, Transparent, %transparent%, ahk_id %w%
    return
}
```
 
鼠标在窗口标题栏滚动 : 窗口透明化增加或者减弱
win+-\win+= : 当前窗口透明化增加或者减弱
alt+-\alt+= : 当前窗口透明化增加或者减弱
 
如何判断鼠标在标题栏：取得鼠标位置ox，oy，如果oy在7和30之间，判断为标题栏。autohotkey默认的鼠标位置坐标为当前窗口坐标
当窗口透明化大于255（不透明）时，取消窗口透明设置。将窗口透明参数设为255和取消透明设置有一点不同，取消透明设置可以提高窗口性能
 
## 窗口最大化最小化
 
```
!enter:: ;; alt-enter
#enter::
    WinGet,S,MinMax,A
    if S=0
        WinMaximize,A
    else if S=1
        WinRestore,A
    else if S=-1
        WinRestore,A
    return
!m::
#m:: WinMinimize, A
```
 
alt-enter : 最大化/恢复
win-enter : 最大化/恢复
alt+m : 最小化 
win+m : 最小化
 
## 虚拟桌面切换
 
```
WheelUp::
    MouseGetPos,ox,oy,ow,oc
    WinGetClass, owc, ahk_id %ow%
    Ifinstring, owc, Shell_TrayWnd
        VDSwitch(1)
    else
    {
        If (oy>7) and (oy<30)
            WinTransplus(ow)
        else
            send, {WheelUp}
        sleep 30
    }
    return
WheelDown::
    MouseGetPos, ox, oy, ow, oc
    WinGetClass, owc, ahk_id %ow%
    Ifinstring, owc, Shell_TrayWnd
        VDSwitch(0)
    else
    {
        If (oy>7) and (oy<30)
            WinTransMinus(ow)
        else
            send, {WheelDown}
        sleep 30
    }
    return
F11::
    VDSwitch(0)
    return
sc163:: ;; Fn 切换虚拟桌面
    if (A_PriorHotKey="sc163" and A_TimeSincePriorHotkey<250)
        VDSwitch(0)
    return
VDSwitch(c){
    ;; 切换虚拟桌面
 
    cVD := DllCall("VirtualDesktopAccessor\GetCurrentDesktopNumber")
    VDs := DllCall("VirtualDesktopAccessor\GetDesktopCount")
 
    if c=0
        DllCall("VirtualDesktopAccessor\GoToDesktopNumber",int, mod(cVD+1,VDs))
    else
        DllCall("VirtualDesktopAccessor\GoToDesktopNumber",int, mod(VDs-cVD-1,VDs))
}
```
 
任务栏鼠标向上滚动 : 逆序切换桌面
任务栏鼠标向下滚动 : 顺序切换桌面
F11 : 顺序切换桌面
双击Fn : 顺序切换桌面
VDSwitch : 实现切换桌面的函数
+ win10只提供了向前切换（shift+win+右箭头）和向后切换桌面（shift+win+左箭头）的快捷键，切换到第一个或者最后一个桌面后，不会循环切换。
+ github上有一个开源的[dll](https://github.com/Ciantic/VirtualDesktopAccessor)，提供了操纵虚拟桌面的函数，ahk可以用DLLCall调用
+ VDSwitch用了其中两个功能：获取虚拟桌面数量以及goto到第几个虚拟桌面，实现顺序和逆序切换虚拟桌面
 
# 关闭窗口的快捷键
 
```
;; ctrl win alt - q : alt-F4
;; ctrl win alt - w : ctrl-w
!q::send !{F4}
#q::send !{F4}
^q::send !{F4}
!w::send ^w
#w::send ^w
```
 
ctrl+w win+w alt+w : ctrl+w
ctrl+q win+q alt+q : alt+F4
 
左手的三个功能键 
+ w关闭窗口
+ q关闭应用程序
 
 
