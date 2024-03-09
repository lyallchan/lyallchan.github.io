toc: true
title: epub简繁转换
date: 2015-12-04 16:31:06
tags: [epub, 简繁]
description: epub繁体书转换成简体书
---
# epub繁体书转换成简体书

找到一个项目[opf-cc](http://blog.jjgod.org/2013/01/31/opf-cc/)，python编写的，可以直接转换。

项目代码托管在[github](https://github.com/jjgod/opf-cc)。

# 安装

+ 项目依赖[opencc](https://github.com/BYVoid/OpenCC.git)，作者建议从git源码编译opencc
+ lxml，从pip install安装
+ 而编译opencc时，又要使用cmake和doxygen，这两个都可以从cygwin setup安装

## 安装opf-cc

`opf-cc`本身直接git clone就行。

```bash
cd
cd bin
git clone https://github.com/jjgod/opf-cc.git
```

## 安装opencc

```bash
cd
cd bin
git clone https://github.com/BYVoid/OpenCC.git
cd OpenCC
make
make install
```

## 安装lxml

```bash
pip install lxml
```

## 找不到libxml2
安装lxml的时候，系统报错，提示没有libxml2的头文件，直接从cygwin的setup安装

选择的时候，要选择两个软件`libxml2`和`libxslt`，并且在lib分支下，选择`runtime`和`develop`两个版本

## 找不到pyconfig.h

pyconfig.h包含在python的安装包中，实际已经安装，在`/usr/include/python2.7`目录下，但是gcc编译命令中，-I包括的路径为`/include/python2.7`，

做了个软链接解决
```bash
ln -s /usr/include include
```

## 找不到iconv.h

iconv包含在libiconv-devel中，实际已经安装，不知道什么原因丢失了。cygwin setup重新reinstall即可。

## 找不到cmake或者cmake提示CMAKR_ROOT找不到

cygwin setup安装cmake

找到Makefile文件，将cmake命令指定全路径`/usr/bin/cmake`

## 找不到cygopencc-2.dll

编译opencc时，会提示找不到cygopencc-2.dll，实际文件在`opencc/build`目录下，拷贝一份到`c:\windows\system32`下即可

# 使用

```bash
~/bin/opf-cc/opf-cc.py file.epub
```

或者批量转换
```bash
for i in *.epub; do ~bin/opf-cc/opf-cc.py "$i"; done
```
