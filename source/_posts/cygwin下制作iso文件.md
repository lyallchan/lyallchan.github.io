toc: true
title: Cygwin下制作iso文件
date: 2016-02-17 08:22:01
tags: [cygwin, iso]

---

最近换机器，很多不用的文件需要归档，选用iso格式归档，有两个好处：

1. 文件固定不能编辑了，以防误操作，改变文件
2. win10很好的支持iso文件，双击就加载，认为cdrom，使用时不生成临时文件

<!--more-->

# 工具

找到的工具叫`mkisofs`，实际上调用的是`genisoimage`，反正安装前者就可以

# 使用

由于有长中文名，目录层次很深，最后的iso比较大，需要特定命令参数

```
mkisfofs -J -r -joliet-long -o 归档文件.iso ./
```

-J : 使用joliet格式
-r : Rock Ridge格式(使用Rock Ridge格式，可以保存档案相关的权限)
-joliet-long : 使用长格式
-o : output

# 补充

Totalcommand下有个插件[TotalISO](http://www.ghisler.com/plugins.htm)，可以alt-F5调用mkisofs，很好用。

插件下载下来，TC中双击安装。然后把cygwin下的genisoimage以及需要的cyg*.dll拷贝到TotalISO安装目录，并把genisoimage改名为mkisofs即可。

+ 注意：这里不能到网上随便找一个，早期的mkisofs不支持cp936，utf8编码，不能用。要用新版的
+ cyg*.dll有5个：cygwin1.dll  cygmagic-1.dll  cygiconv-2.dll  cygz.dll  cygbz2-1.dll

# 补充二

对于超大文件，需要增加`-allow-limited-size`参数

如果中文文件名乱码，增加`-jcharset utf8`参数

``` bash
genisoimage -J -r -joleit-long -allow-limited-size -jcharset uft8 -o 归档文件.iso ./
```
