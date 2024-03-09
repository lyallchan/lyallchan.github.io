toc: true
title: vscode安装golang插件过程
date: 2018-02-11 20:22:50
tags: [vscode, golang, 被墙]
description:

---

# vscode安装golang插件过程

安装golang插件很简单，在市场中搜索`go`，找到的第一个插件就是。

golang插件需要很多go tools，然后这些go tools都在墙外。

<!--more-->

下面记录了安装这些go tools的方法

# go tools的下载方法

根据vscode的golang插件依赖的go tools[说明页面](https://github.com/Microsoft/vscode-go/wiki/Go-tools-that-the-Go-extension-depends-on)，需要安装的go tools列表如下。

``` bash

go get -u -v github.com/ramya-rao-a/go-outline
go get -u -v github.com/acroca/go-symbols
go get -u -v github.com/nsf/gocode
go get -u -v github.com/rogpeppe/godef
go get -u -v golang.org/x/tools/cmd/godoc
go get -u -v github.com/zmb3/gogetdoc
go get -u -v github.com/golang/lint/golint
go get -u -v github.com/fatih/gomodifytags
go get -u -v github.com/uudashr/gopkgs/cmd/gopkgs
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v sourcegraph.com/sqs/goreturns
go get -u -v github.com/cweill/gotests/...
go get -u -v golang.org/x/tools/cmd/guru
go get -u -v github.com/josharian/impl
go get -u -v github.com/haya14busa/goplay/cmd/goplay
```

除此之外还有一个DEUBG插件，[官方页面](github.com/derekparker)，安装方法

```bash
go get -u -v github.com/derekparker/delve/cmd/dlv
```

# go tools安装在公共位置

go tools默认安装在GOPATH下，GOPATH一般是项目目录。如果你的项目很多，你希望go tools安装在公共目录。

在vscode用户配置中，增加如下配置

```bash
"go.toolsGopath": "c:\\Users\\XXX\\Golang\\Gotools.vscode"
```

# 通过golangtc安装第三方插件

直接安装上述插件，会安装失败。通过[golangtc](https://www.golangtc.com/download/package)。网页上有详细的安装说明，按照说明操作就可以。

如果一切顺利，所有go tools会安装成功。
