toc: true
title: 重定义capslock键MAC
date: 2015-12-04 16:31:06
tags: [mac, capslock]
description: 在mac下重定义capslock

---
# 1. 目的
 
CapsLock位于键盘的黄金位置，平常却没有什么用处，修改CapsLock键，使得
 
+ 单独按CapsLock键，映射为`<Esc>`
+ 组合按CapsLock键，如`CapsLock-C`，映射为`<Ctrl-C>`
 
# 2. 所需软件
 
+ KeyRemap4Macbook：自定义private XML，满足上述需求
+ Seil：CapsLock是外部键，需要使用Seil将Capslock映射为普通键，如keycode80，代表F19键
 
# 3. 配置
 
1. 系统偏好设置 - 键盘 - 修饰键 - CapsLock - 无操作
 
2. Seil：
 
            keycode=80
 
3. KeyRemap4Macbook
 
+ Preference - Misc&Uninstall - Open Private.xml，键入
 
            <?xml version="1.0"?>
            <root>
                <item>
                    <name>F19 to F19</name>
                    <appendix>(F19 to ctrl + F19 Only, send escape)</appendix>
                    <identifier>private.f192f19_escape</identifier>
                     <autogen>
                        --KeyOverlaidModifier--
                        KeyCode::F19,
                        KeyCode::CONTROL_L,
                        KeyCode::ESCAPE
                    </autogen>
                 </item>
            </root>
 
+ 存盘退出后，回到KeyRemap4Macbook - Preference - ChangeKeys - ReloadXML
 
+ 会出现F19 to F19，前面打勾，即可使用。
 
# KeyRemap4Macbook其它设置
 
1. Fn to switch Browsing Mode
2. Option-v to switch Complete vi Mode
 
