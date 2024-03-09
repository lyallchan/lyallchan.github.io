toc: true
title: Keepass使用记录
date: 2017-2-19 21:13:02
tags: [keepass, autohotkey, 命令行]

---

最近打算更换密码管理工具，比较了几个密码管理软件，keepass真心不错。

<!--more-->

# keepass

记录密码的神器。

提供auto-type方法和命令行工具，以方便使用账号和密码。

比如，打开网易163邮箱，按下keepass热键，自动找到相应的账号，填写用户名和密码，回车完成登陆。

再比如，编写脚本，使用keepass中存储的账号，自动进行VPN连接。

# 下载和安装

http://keepass.info

有classic和professional两个版本，实际上是版本1和版本2。

版本1不需要任何运行库，版本2基于.net编写。相对来说，版本2的功能更加多一点，基于版本2的插件数量也多。

两个版本都有便携版本下载，下载好以后，解压缩就可以运行。也可以下载[中文语言包](http://keepass.info/translations.html)，解压缩后放在keepass所在目录。

官方文档表示便携版本不会写任何数据到安装目录以外的地方。

使用很简单，看着界面就会做。下面重点讲讲如何自动化运行

# 全局auto-type

http://keepass.info/help/base/autotype.html#autoglobal

针对一个账号，定义一个按键序列。keepass可以模拟键盘自动输入到当面界面，比如登陆界面或者一个WEB页面等等。

默认的按键序列是{USERNAME}{TAB}{PASSWORD}{ENTER}，也就是输入用户名-TAB键-输入密码-回车，适用于绝大部分WEB登陆页面。

如果遇到特殊场景，也可以自定义按键（编辑记录页面->自动输入）。

定义好按键序列之后，就可以启动全局auto-type。默认启动热键是ctrl-alt-a。

在WEB登陆页面，比如[网易邮箱](mail.163.com)，输入焦点放在登录名字段中，按下ctrl-alt-a，keepass自动输入用户名和密码，然后登陆。很快捷。

背后的原理为

1. keepass为auto-type注册一个全局热键ctrl-alt-a
2. 根据当前窗口的标题，寻找对应记录的标题字段，如果记录的标题字段包含在当前窗口的标题中，就根据按键序列模拟键盘输入。
3. {USERNAME}和{PASSWORD}叫做[placeholder](http://keepass.info/help/base/placeholders.html)，用来替换相应记录的用户名和密码字段。其它常用的placeholder还有{TITLE}和{URL}。
4. keepass2中也可以选择URL字段匹配当前窗口标题，在工具-选项-高级-自动输入中更改。
5. 如果记录Title字段还不够复杂，还可以在编辑记录-自动输入页面中自定义匹配规则。

keepass还提供了一个命令行选项`-auto-type`，用于其它已经打开的keepass进程运行触发全局auto-type

```
keepass.exe -auto-type
```

# 命令行自动化

keepass提供命令行选项，总结一下包括3类：

1. 启动keepass，并打开相应的数据库
2. 全局auto-type，打开指定记录URL，这个URL不光能打开页面，也能运行一个程序，后面会细说
3. 退出keepass

## 启动keepass

```
keepass.exe 数据库文件 -keyfile:keyfile -pw:数据库主密码 -minimize
```

## 自动执行

keepass打开之后，可以使用命令自动执行

```
keepass.exe -auto-type
keepass.exe -entry-url-open -uuid:CA5F4E61B4E87648917B7C5B4AC6B5C7
```

第一个命令执行自动全局auto-type

第二个命令根据uuid（指定记录），打开该记录的URL

URL说明[在此](http://keepass.info/help/base/autourl.html)

URL最普通的形式为`http://`，`ftp://`，`mailto://`等，如果你的系统定义了ssh，也可以使用`ssh://`形式。

URL第二种形式为`cmd://`，可以运行一个命令，举例

```

cmd://C:\windows\notepad.exe c:\test\testfile.txt
cmd://c:\windows\explorer.exe %TEMP%
cmd://c:\windows\telnet.exe 192.168.1.1 {USERNAME} {PASSWORD}
```

`cmd://`可以使用环境变量，也可以使用placehoder，[placeholder](http://keepass.info/help/base/placeholders.html)说明也很值得一看。

## 退出keepass

```
keepass.exe -close-all
```

# 应用

## autohotkey自动启动keepass，执行auto-type后退出

```
;; keepass
^!o::
    KEEPASS_DIR := "xxxxx\keepass2"
    WinGet, a_id, ID, A
    run %KEEPASS_DIR%\keepass.exe xxxxx.kdbx -keyfile:xxxxx -minimize
    sleep 1000
    WinActivate, ahk_id %a_id%
    run %KEEPASS_DIR%\keepass.exe -auto-type
    sleep 1000
    run %KEEPASS_DIR%\keepass.exe -exit-all
    return

```
定义热键ctrl-alt-o，取得当前窗口id，启动keepass，暂停1秒，激活刚才的窗口（keepass启动时，有可能会切换当前窗口），执行auto-type，退出keepass

## 自动拨VPN

```
#!/bin/bash

KEEPASS_DIR="xxxxxx/keepass2"

KEEPASS=$KEEPASS_DIR/keepass.exe
KEEFILE=$(cygpath -aw xxxxxx)
KEYFILE=$(cygpath -aw xxxxxx)

cygstart -d $KEEPASS_DIR $KEEPASS $KEEFILE -keyfile:$KEYFILE -minimize
sleep 1
$KEEPASS -entry-url-open -uuid:CA5F4E61B4E87648917B7C5B4AC6B5C7
$KEEPASS -exit-all
```

最后四句，启动keepass，暂停1秒，根据uuid打开URL，最后退出

URL定义为

```
cmd://rasdial eni {USERNAME} {PASSWORD}
```
其中eni是在网络中定义好的VPN条目。
