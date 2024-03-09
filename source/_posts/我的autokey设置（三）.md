toc: true
title: 我的autokey设置（三）
date: 2018-11-26 10:33:14
tags: [autohotkey, 快捷键]
description: 

---

# 我的autokey设置（三）

近期对整个系统的快捷键做了大幅的增改，主要设置在autohotkey中，但也有很多软件自己有快捷键，一并整理。

<!--more-->

# AutoHotKey的设置

## Capslock

    1. Shift+Caps : 切换大小写
    2. 单独Caps ： ESC
    3. Caps+1 ： 激活TC，要求TC放在任务栏第一个
    4. Caps+2 ： 激活ConEMU，要求ConEMU放在任务栏第二个
    5. Caps+（hjklae） ： 任意程序除了VIM、Mintty、ConEMU、EMACS，上下左右，行首行尾，如果标题栏为Powershell或者cmd，则上述快捷键也有效
    6. Caps+（dwu） ： 如果标题栏为Powershell或者cmd，分别为退出、删除前一个单词，删除到行首
    7. Caps+其它键 ： Ctrl+其它键
    8. Caps+BS ： Caps+BS。特别是**ConEMU设置Ctrl-BS删除前一个单词**，则整个在ConEMU中运行的命令行，包括WSL、MINGW64、Cygwin、Powershell、cmd，都可以用Caps+BS删除前一个单词

## 窗口类操作

    1. alt+enter ： 切换最大化
    2. alt+m ： 最小化当前窗口
    3. alt+0 ： 切换窗口置顶
    4. alt++ ： 窗口更加不透明
    5. alt+- ： 窗口更加透明

## 鼠标滚轮滚动事件

    1. 任务栏 ：切换桌面
    2. 屏幕顶端 ： 运行程序Claunch
    3. 标题栏 ：增加或者减少窗口透明度

## hjkl

    1. 在TC中，使用VMATC设置
    2. SumatraPDF，使用AHK设置
    3. Chrome，使用VIM插件

## 其它

    1. ctrl win alt + q : alt-F4， 除了ConEMU
    2. ctrl alt + w : ctrl-w，除了ConEMU
    3. F12 : 切换虚拟桌面
    4. F3 : 下一个窗口，目前支持chrome，IE，edge。Conemu也支持，直接在Conemu中设置的
    5. F2 : 上一个窗口，目前支持chrome，IE，edge。Conemu也支持，直接在Conemu中设置的
    6. Win+o : TC/激活TC，拷贝当前路径到剪贴板
    7. Alt+r ： 运行CLauncher，CLauncher不常驻内存，用完即退出
    8. F3 ： 在TC的LIST中直接关闭，最后效果F3预览，预览完成后F3关闭，手指不移动
    9. ；+d ： 插入当前日期
