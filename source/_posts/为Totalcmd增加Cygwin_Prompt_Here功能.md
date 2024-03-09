toc: true
title: 为Totalcmd增加Cygwin Prompt Here功能
date: 2016-3-28 15:45:00
tags: [totalcmd, cygwin]
 
---
[TOC]

为Totalcmd增加Cygwin Prompt Here功能，实现启动cygwin，进入TC当前目录。
 
<!--more-->

实现几个功能：

1. TC当前文件是目录的，直接进行当前文件
2. TC当前文件时文件的，进入当前文件所在目录
3. 支持路径以及文件名中含有空格
4. 绿色，不用写注册表

# 整体思路

利用TC的`%P`，`%N`传递当前文件，将当前文件信息写入环境变量，
cygwin启动时，读取环境变量，如果设置了环境变量，则进入环境变量所存的目录，否则进行HOME目录。

# TC中的配置

## 增加新命令em_cygwin

在TC中增加一个新的usercmd，起名`em_cygwin`，具体参数如下：

```
[em_cygwin]
button=%Commander_path%\..\cygwin64\bin\mintty.exe
cmd=%Commander_path%\..\cygwin64\bin\mintty.exe
param=/bin/zsh -lc 'export STARTIN=$(echo $(/bin/cygpath "%P "))%N; exec /bin/zsh'
path=%P
menu=CYGWIN Prompt Here
```

将上述代码写入usercmd.ini，如果没有这个文件，则在totaocmd目录中，新建这个文件。

命令启动mintty，参数为`param=/bin/zsh -lc 'export STARTIN=$(echo $(/bin/cygpath "%P "))%N; exec /bin/zsh'`
 
STARTIN为设置的环境变量，其中路径部分的处理比较复杂，一层一层剥开来解释：

+ %P : 代表TC当前目录，以`\`结束
+ "%P " : 增加引号，防止路径中的反斜杠转义，引号中的空格一定要加，否则%P最后的反斜杠和引号放在一起，会转义成一个引号，引起错误
+ /bin/cygpath "%P " : windows路径格式转换成cygwin路径格式
+ echo $(/bin/cygpath "%P ") : 去掉路径前后的空格

路径处理好以后，加上当前文件名%N。如果%N中含有空格，则TC会自动加上引号。

## 增加alias

增加一个新的alias，做为`em_cygwin`别名。在wincmd.ini的[alias]一节中增加：

```
alias cygwin=em_cygwin
```

## 增加工具栏按钮

在TC工具栏上增加一个按钮，指向em_cygwin。

TC中的配置完成。

## cygwin配置

在.zshrc中增加以下代码处理STARTIN：

```

if [ -z "$STARTIN" ]; then
    cd $HOME
else
    if [ -d "$STARTIN" ]; then
        cd "$STARTIN"
    else
        T=`dirname $STARTIN`
        if [ -d "$T" ]; then
            cd "$T"
        else
            echo "$STARTIN is not a valid path."
            cd $HOME
        fi
    fi
fi

```

比较简单，不多说了。

