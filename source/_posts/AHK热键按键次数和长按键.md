toc: true
title: AHK热键按键次数和长按键
date: 2022-05-12 15:45 
tags: [AHK, 按键次数, 长按键]
description:

---

# AHK热键按键次数和长按键

写了一个函数，返回热键的按键次数和是否长按键，可以在脚本中调用。

```Autohotkey
KeyCounter(Key, Func, param, Timeout)

返回 ： 热键按键次数，如果是长按键，返回100
入参 ：
    Key : 热键，可以省略，默认为A_ThisHotKey
    Func ：长按键时调用的函数，可以省略，不能为空字符串
    Param ：Func的参数，可以省略
    Timeout ： 超时时间，可以省略，默认500ms
```

<!--more-->

> 说明：这个函数支持单个按键、有修饰键（Ctrl、Alt、Shift、Win）的组合按键。不支持形如`a & b`自定义组合键。
> 有修饰键的组合按键场景，如`CTRL-T`，支持按住`CTRL`键，返回`T`键的按键次数，即按住`CTRL-T`、按住`CTRL`不放，再按`T`键，返回按键次数2，不支持`CTRL-T`，松开`CTRL`，再按一次`CTRL-T`的场景。

使用案例

``` Autohotkey
$F1:: 
    Switch KeyCounter()
    {
        Case 1 : Tooltip, 按键一次
        Case 2 : Tooltip, 按键两次
        Case 3 : Tooltip, 按键三次
        Case 100 : Tooltip, 长按键
    }
    Return
```

> Func参数说明：
> 传入的Func在长按键按下已超时，未弹起时调用
> 如果判断KeyCounter的返回值等于100，再调用函数，则要等到长按键弹起时才能调用
> 推荐使用传入Func方式，这样用户感知更加好

``` Autohotkey
$F1:: 
    Switch KeyCounter(, "Func_long")
    {
        Case 1 : Tooltip, 按键一次
        Case 2 : Tooltip, 按键两次
        Case 3 : Tooltip, 按键三次
    }
    Return

Func_long()
{
    Tooltip, 长按键，按键未弹
}
```

# 代码

``` Autohotkey
KeyCounter(Key:="", Func:="IsFunc", param:="", Timeout:=500)
{
    OldKey := Key ? Key : A_ThisHotKey
    Key := RegExReplace(OldKey, "[\^!#+$~]") ;; 移除热键修饰符

    ;; 等待按键弹起或者按键超时
    ;; 如果按键弹起，认为是短按键
    ;; 如果按键超时，认为是长按键
    Loop 
    {
        sleep, 20
        state:=GetKeyState(Key,"P")
    } Until    state=0  ;; 按键已弹起
            || A_TimeSinceThisHotKey>Timeout ;; 按键已超时

    ;; 分别处理短按键和长按键
    if(state=0) ;; 短按键
    {
        count:=1

        ;; 继续等待按键按下，
        ;; 如果等到按键，则按键次数加1
        ;; 如果没有等到按键，返回按键次数，跳出循环
        Loop
        {
            KeyWait, %Key%, D T0.2 ; 继续等待按键按下,0.2s超时
            if (ErrorLevel=0) ;; 等到按键
            {
                KeyWait, %Key%
                count++
            }
        } Until ErrorLevel = 1 ;; 没有按键按下
        Rtn := count
    }
    else ;; 长按键
    {
        %Func%(param)
        KeyWait, %Key% ;; 等待按键弹起
        Rtn := 100
    }
    Return Rtn
}
```

# 逻辑

```
0. 按下热键
1. 等待按键弹起或者超时还没有弹起
2. 如果按键弹起，认为是短按键，如果按键超时还没有弹起，认为是长按键
    3. 如果是短按键，继续等待按下热键
        4. 如果等到热键按下，则按键次数加1
        5. 如果没有等到热键按下，则返回按键次数
    6. 如果是长按键
        7. 执行长按键函数
        8. 返回长按键代码100
```
