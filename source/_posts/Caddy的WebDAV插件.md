toc: true
title: Caddy的WebDAV插件和Total Commander
date: 2018-01-28 13:39:15
tags: [Caddy, WebDAV, Golang，Total Commander]
description:

---

# Caddy是什么

Caddy是用Golang开发的WEB Server，和所有Golang开发的程序一样，编译以后是个只有命令行的单文件程序。

<!--more-->

GitHub地址 https://github.com/mholt/caddy
官网主页 https://caddyserver.com/

原来使用Caddy当作临时文件服务器来用，一行命令搞定

```
Caddy -port 80 browse
```

把当前目录HTTP出来，使用浏览器直接访问`http://localhost`就可以看到当前目录下的文件，方便，绿色。

配合Total commander，可以在Total Commander的usercmd.ini增加一个命令，代码为

```
[em_Caddy]
cmd=%Commander_path%\..\Caddy\Caddy.exe
param=-port 80 browse
menu=HTTP共享当前目录
```
这样，一个按钮就可以在HTTP共享当前目录。



# Caddy的WebDAV插件

这次发现Caddy有了WebDAV插件，那比HTTP共享又方便了一点。

文档 https://caddyserver.com/docs/http.webdav

使用WebDAV要重新下载含有WebDAV的Caddy版本，在下载页面`https://caddyserver.com/download`上可以选择

下载合适的版本以后，发现配置比较麻烦，难道要写一个Caddyfile？不绿色

Caddy命令行支持重定向功能

> Pipe a Caddyfile

> Advanced users may wish to pipe the contents of a Caddyfile into Caddy from programmed environments. If you pipe in the Caddyfile, you must use the -conf flag with a value of stdin - for example:

> $ echo "localhost:1234" | caddy -conf stdin

> Piping the Caddyfile is convenient when starting Caddy using a dynamically-generated Caddyfile from a parent process you have control over.


> 管道化Caddyfile
> 高级用户可以使用管道功能，将Caddyfile动态输入Caddy的stdin。如果使用管道功能，你必须使用`-conf stdin` 参数来表示Caddy从标准输入读取配置文件。举例

> $ echo "localhost:1234" | caddy -conf stdin

> 管道化Caddyfile可以让你动态配置参数。

由于Total Commander自定义命令不支持管道，写一个批命令来做这个事情。

```BAT
@echo off
(
echo :80
echo webdav {
echo    modify false
echo        }
) | %~dp0%caddy.exe -conf stdin
```

很简单，自己看。`%~dp0%`在bat文件的启动目录找到caddy.exe。

在Total Commander中增加一个命令

```
[em_Caddy]
button=%COMMANDER_PATH%\TOTALCMD64.EXE,33
cmd=%COMMANDER_PATH%\..\Caddy\caddy.cmd
menu=WebDAV共享当前目录
```

2018-01-29更新

直接新增命令就行，不需要写bat文件

```
[em_Caddy]
button=%COMMANDER_PATH%\TOTALCMD64.EXE,33
cmd=%COMMANDER_PATH%\..\Caddy\caddy.exe
param=-port 80 "basicauth lyallchan 20002000" webdav
menu=WebDAV共享当前目录
```
增加一个用户认证，这样就不要配置webdav的只读参数

大功告成。

# BUG

要共享的目录层次很深，文件数量很多的时候，出错。

试了几次，Windows下WebDAV启动的时候，似乎要读取共享目录下的所有文件，当文件数量很多的时候，要花很长时间读取目录，资源耗尽的时候会出错。

看了github上，作者说他只是调用了Golang的WEBDAV包，不知道原理。看来以后再研究。






