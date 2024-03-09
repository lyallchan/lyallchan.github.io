title: 安装和配置nvm、nodejs和hexo
description: 安装和配置nvm、nodejs和hexo
toc: true
tags: [nodejs, hexo, 静态博客]
date: 2015-12-10 17:47:13
---

打算使用hexo建立自己的静态博客。
hexo是nodejs应用，下面记录了从新的系统安装，使用hexo的全过程
 
# 安装nodejs
 
nodejs版本更新很快，nvm是比较好的node版本管理工具。
先安装nvm，然后使用nvm安装node
 
## linux
 
```sh
# git and source nvm
git clone https://github.com/creationix/nvm ~/.nvm
source ~/.nvm/nvm.sh
 
# install latest nodejs version, and
# set default node version to be used in new shell
nvm install node
nvm use node
nvm alias default node
 
# automatically source nvm on login
echo '[ -e ~/.nvm/nvm.sh ] && source ~/.nvm/nvm.sh' >> ~/.bashrc
```
 
## windows
 
从下面的地址下载nvm for windows安装包，然后一路回车安装完成即可。
 
https://github.com/coreybutler/nvm-windows
https://github.com/coreybutler/nvm-windows/releases
 
安装完成以后，nvm安装目录为`c:\Users\username\AppData\Roaming\nvm\ `

安装node
 
```sh
nvm install latest 32
nvm use latest 32
```
 
# 安装cnpm
 
淘宝有个npm[镜像](http://npm.taobao.org/)，速度很快
淘宝npm镜像推荐使用[cnpm](https://github.com/cnpm/cnpm)，那么先安装cnpm
 
```sh
npm install cnpm -g --registry=https://registry.npm.taobao.org
```
 
# 安装[hexo](https://hexo.io)
 
假定博客目录为~/hexo
 
```sh
cnpm install hexo-cli -g
cd
mkdir hexo
hexo init hexo
```
修改配置文件`~/hexo/_config.yml`，主要修改site、url和deploy三个地方，其余使用默认值就可以
 
修改site配置，包含了静态博客站点的基本信息

```
title: 折腾手记
subtitle: 运维！！运维！！
description:
author: lyallchan
language: zh-CN
timezone:
```
 
修改URL配置，指向真实的博客地址

```
url: http://lyallchan.github.io
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:
```
 
修改deploy配置，能使用hexo直接部署博客
这里使用的是github.io的空间

```
deploy:
  type: git
  repo: git@github.com:lyallchan/lyallchan.github.io.git
```
 
# 安装[maupassant](https://www.haomwei.com/technology/maupassant-hexo.htm)主题
 
```sh
cd ~/hexo
git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
cnpm install hexo-renderer-sass --save
cnpm install hexo-renderer-jade --save
```
 
maupassant的配置文件为`theme/maupassant/_config.yml`，根据实际情况修改，比较简单。
 
修改`~/hexo/_config.yml`的theme配置，使得maupassant主题生效
 
```
theme: maupassant
```
# 文章生成和部署
 
用hexo new生成新文章，用hexo deploy直接可以部署，如果要本地查看，hexo server在localhost:4000生成本地的博客，可在浏览器中查看
 
新文章
```sh
cd ~/hexo
hexo new first_blog
vi source/_posts/first_blog.md
```
 
部署
```sh
cd ~/hexo
hexo deploy -g
```
 
本地查看
```sh
cd ~/hexo
hexo g
hexo server
```
 
# 整个博客环境的备份和迁移
 
github上建个新的[工程](https://github.com/lyallchan/blog_src)，将整个`~/hexo`目录git管理起来即可。
换台机器，先安装好了node和cnpm，将博客git clone下来，进入目录后，`cnpm install`即可建立起博客环境。

假定新的环境目录为~/blog
 
```sh
cnpm install hexo-cli -g 
cd
git clone https://github.com/lyallchan/blog_src blog
cd blog
cnpm install
```
 
有几个目录不需要git，相应的.gitignore的内容为
 
```
.DS_Store
Thumbs.db
db.json
*.log
public/
.deploy*/
node_modules/
```
