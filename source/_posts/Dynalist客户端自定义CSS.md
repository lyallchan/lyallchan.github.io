toc: true
title: Dynalist客户端自定义CSS
date: 2019-04-16 16:48
tags: [electron, css, dynalist]
description: 

------


Dynalist是一个无限列表应用，和workflowy是同类应用，国内的同类应用是幕布。Dynalist支持Markdown和Latex，也支持ToDolist和TAG。

Dynalist有网页端、移动端、桌面客户端等各种应用，桌面客户端用electron打包，可以跨平台支持。

Dynalist收费版本可以自定义CSS，比如自定义Markdown样式，自定义ToDoList样式等，在多端都可以同步。如果你在试用阶段，也想修改CSS，只能自己找出路，目前在网页端和桌面端试验成功。

<!--more-->

## Dynalist网页端

很简单，使用专门针对Dynalist开发的的油猴脚本`dynalist powerpack`，在油猴中安装好，自己摸索一下，功能很丰富。

## Dynalist桌面端

Dynalist桌面端用electron开发，用asar打包，要自定义CSS，涉及两个目录

安装路径为`~/home/appdata/local/dynalist`

配置文件路径为`~/home/appdata/roaming/dynalist`

首先安装asar

```
npm install -g asar
```

在Dynalist的安装目录的`resouces`目录下，有4个.asar文件，实际上是asar的压缩文件，其中`Dynalist.asar`就是整个Dynalist应用，将这个文件使用asar解压

```powershell
cd ~/home/appdata/local/dynalist/resources
asar extract dynalist.asar dyn/
```

Dynalist生产中的CSS为`./resource/dyn/www/assets/css/app.css`，修改这个文件就可以自定义Dynalist的显示界面。

这里举个例子，修改两个样式

1. Markdown的Heading1，字体为红色

2. Checklist字体为蓝色

   在app.css文件最后添加

```css
.is-heading1 { color:red }
.is-checklist { color:blue }
.node-tag[title*="TODO"] { color:blue }
```

存盘退出。

> 如何寻找不同样实对应的class？在Dynalist按`alt`键，会出来菜单栏，选`View-Toggle DevTools`，打开开发者界面，然后细心寻找你需要的目标，这里需要的是耐心

再重新打包，还是在解压缩的目录不变

```powershell
mv dynalist.asar dynalist.asar.backup
asar pack dyn/ dynalist.asar
```

最后一步，Dynalist的配置文件路径中缓存了Dynalist.asar，具体路径为`~\AppData\Roaming\Dynalist\dynalist\packages\dynalist-1.1.19.asar`，删除这个文件，Dynalist会再缓存修改过的`dynalist.asar`，这样修改的内容才可以生效。

