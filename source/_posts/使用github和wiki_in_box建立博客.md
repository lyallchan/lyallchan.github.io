toc: true
title: 使用github和wiki_in_box建立博客
date: 2015-12-04 16:31:06
tags: [wiki, 静态博客]
description: 使用wiki in a box建立静态博客
---
# 目的
 
1. 建立静态博客，不需要自己维护数据库
2. 可以在不同的机器上编辑，不需要搭建编译环境
3. 支持markdown
 
# 系统选择
 
1. github提供了username.github.io的博客，静态的。username是你的github用户名
2. 可以git clone下来，本地修改好再push上去，git和https的方式都可以。实际上，可以这个博客就是一个普通的repo，名字就是username.github.io
3. 博客系统不使用github推荐的jelly，因为jelly本身需要搭建一套编译环境，不能实现任何机器上都可以编辑的需求
4. 其它静态博客系统如hexo，也都存在类似的问题，需要在本地搭建一套编译环境
5. 有个bash下的编译系统[bashblog](https://github.com/cfenollosa/bashblog)，只需要一个bash文件就可以完成，试用了一下，很是不错，能实现基本的博客增删改功能。有机会可以尝试一下
6. 这次选择的是[ wiki in box ](https://github.com/dmscode/Wiki-in-box)，下文简称box
    + 不需要编译
    + git clone下来就是一套完整的系统，可以本地运行
    + 编辑并git push好markdown文件后，马上就可以使用
7. box的缺点
    + 新增博客后，需要手工修改index.md文件，使得新增的博客在博客主页中能看到...
    + 没有tags系统，也不能搜索
    + 有着其它静态博客通用的缺点，如不能评论等等，box也都有。但网上都有解决方案
 
# 建立过程
 
1. 注册github账号，建立名为`username.github.io`的repo。
    + 我的账号是`lyallchan`
    + 新建repo时，输入`Repository name`为`lyallchan.github.io`，其它不要改变，尤其是不要选择`Initialize this repository with a README`，也不要新增`.gitignore`和`license`
    + 博客主页是`lyallchan.github.io`，repo的地址是`git@github.com:lyallchan/lyallchan.github.io.git`。这时github告诉你这是一个空的repo。
 
2. 针对wiki的一些弱点，做了以下的修改
    + data目录下的md文件不支持中文名，主要是index.html中marked renderer.link只考虑`\w`的情形，漏掉了中文字符。改成只查询`.`，有`.`的表示普通链接，没有`.`表示data下的md文件
    + toc.min.js生成目录时，对全部中文的标题不能准确的生成anchorName，修改index.html的toc调用方法，增加了自定义anchorName功能
    + 大幅修改css，原来css用的是main.css，修改之后使用pandoc.css和github2.css。其中github2.css是wiz里面用的css文件。而pandoc.css修改源于pandoc的模版，但是修改幅度很大，基本和原来没有什么关系了。anyway，还是用的bootstrap。
    + 新建index.gen脚本，自动生成index.md。顺便说一下，针对内部链接，作者说明中是使用`:`分隔，实际上，用`/`也可以。这样在生成index时，直接使用目录分隔符就可以了，不用再做进一步处理
 
3. 将修改好的系统push到新建的repo上
 
假定修改好的系统在`~/wikis`，以后所有数据都放在`~/wiki`。
 
```bash
cd
git clone git@github.com:lyallchan/lyallchan.github.io.git wiki
cd wiki
cp -r ../wikis/. ./
git add .
git commit -m "初始化"
git push
```
# 新增博客
 
```bash
cd ~/wiki/data
vi 新增博客名称.md
./index.gen
cd ..
git add .
git commit -m "新增博客"
git push
```
