toc: true
title: 升级到Hexo5
date: 2020-12-13 15:08
tags: [Hexo5, Nodejs]
description: 

------

# 背景

升级Nodejs，结果Hexo报错，上网查了一下，原来是Nodejs版本太高。

因为懒，Hexo一直没有升过级，还是用的5年前的版本。到Hexo[官网](http://hexo.io)看看，最新的已经是5.1版本了，决定升级Hexo，并将过程记录如下。

<!-- more -->

升级需求很简单，保留Next主题，保留Git发布，保留[markdown图片自动生成功能](http://lyallchan.github.io/2018/02/07/%E4%BD%BF%E7%94%A8VSCode%E5%9C%A8Hexo%E4%B8%AD%E4%BD%BF%E7%94%A8%E5%9B%BE%E7%89%87/)。

# 安装

按照Hexo官网

```
mkdir Hexo5
cd Hexo5
npx hexo init
```

系统下载好hexo，初始化hexo目录，完成很顺利。

# 插件

```
cd Hexo5
npm install hexo-deployer-git --save
npm install hexo-theme-next
npm install hexo-simple-image --save
```

这三个插件在hexo官网上都能找到。

第一个是git发布。

第二个是[next主题](https://theme-next.js.org)，老版本Hexo的主题文件都在`theme`目录下，现在采用插件方式。

第三个是markdown图片插件，替代了原来的[hexo-asset-image](http://lyallchan.github.io/2018/02/07/%E4%BD%BF%E7%94%A8VSCode%E5%9C%A8Hexo%E4%B8%AD%E4%BD%BF%E7%94%A8%E5%9B%BE%E7%89%87/)。

使用hexo-simple-image插件，vscode的配置稍有不同，如下

```
"pasteImage.insertPattern": "${imageSyntaxPrefix}./${imageFilePath}${imageSyntaxSuffix}"
```

增加了`./`

# 配置

主配置文件还是`Hexo5\_config.yml`，配置基本同原来配置，修改了一个地方

```
language: zh-CN
```

Next配置和原来不一样，原来Next是在theme目录下，改为插件以后，在node_modules目录下。按照[官网](https://theme-next.js.org)配置方法，将`hexo5\node_modules\hexo-theme-next\_config.yml`文件拷贝到`hexo5\_config.next.yml`，即

```
cd Hexo5
cp hexo5\node_modules\hexo-theme-next\_config.yml hexo5\_config.next.yml
```

修改`_config.netx.yml`，具体的配置方法可以参见https://www.jianshu.com/p/9f0e90cc32c2

# 便利

总是使用`npx hexo`不方便，写了一个powershell脚本，起名`hexo.ps1`供调用

```
Push-Location
Set-Location ...\hexo5
npx hexo $args
Pop-Location

write-host "Press any key to continue..."
$null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown')
```

