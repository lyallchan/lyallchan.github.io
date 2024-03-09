toc: true
title: 使用vim画表格
date: 2015-12-04 16:31:06
tags: [vim, tabular]
---

使用vim画表格
<!--more -->

# 使用vim画表格
 
在markdown时，可以方便、漂亮的画出表格
 
# 使用的插件
 
+[tabular](https://github.com/godlygeek/tabular)
+[table-mode](https://github.com/vim-scripts/table-mode.git)
 
# 安装
 
使用`pathogen`安装
 
```bash
cd .vim/bundle
git clone https://github.com/godlygeek/tabular
git clone https://github.com/vim-scripts/table-mode.git
```
 
如果有洁癖，把`tabular`和`table-mode`目录下的`.git`和`.gitignore`删除
 
装好以后，`cygwin`下的`vim`使用正常，`gvim`下不生效！！！用`set rtp`查看，发现两个新增目录`after`下插件没有包含在`rtp`中。顺便说一下，可以在[这里](http://www.ajucs.com/archives/478.html)了解一下vim的目录结构。
 
`table-mode`基于`tabular`插件，所以`table-mode`将主要程序放在`after`目录下，以确保`tabular`插件可以先加载。
 
这里要说明一下，为了保证`windows`下和`cygwin`下使用同一套`.vim`文件，我的`$HOME`目录下没有`windows gvim`所需要的`vimfiles`目录，而是使用同一个`.vim`目录。
 
查看`pathogen`源码，里面有一段是这样的
 
```vim
for dir in pathogen#split(&rtp)
  if dir =~# '\<after$'
    let list += reverse(filter(pathogen#expand(dir[0:-6].name.sep.'after'), '!pathogen#is_disabled(v:val[0:-7])')) + [dir]
  else
    let list += [dir] + filter(pathogen#expand(dir.sep.name), '!pathogen#is_disabled(v:val)')
  endif
endfor
```
 
只有在系统默认的`rtp`目录包含`after`目录时，才加载这个目录下的`bundle`的`after`目录。
 
而`windows gvim`默认的`rtp`只包括`$HOME/vimfiles/after`，不包含`$HOME/.vim/after`。
 
找到原因后，解决起来很简单，在`.vimrc`文件中，调用`pathogen`之前，增加
 
```vim
if has("win32")
    set rtp+=$HOME/.vim/after
endif
```
 
# 配置
 
GFW markdown的表格是这样的
 
```
| 表头1 | 表头2 | 表头3 |
| ---   | ---  | ---  |
| 内容1 | 内容2 | 内容3 |
| 内容4 | 内容5 | 内容6 |
```
 
需要配置table-mode的表格转角、分隔符的字符，以符合GFW markdown的要求
 
```vim
let g:table_mode_corner = '|'
let g:table_mode_border=0
let g:table_mode_fillchar=' '
```
 
# 使用
 
有两种使用方式，第一种：任何`|`起始的行，默认激活table-mode，可以一边编辑一边生成表格，第二种：使用`g:table_mode_delimiter`指定的字符做分隔符，先生成表格内容，再用`:Tableize`命令将表格内容格式化成表格
 
由于`|`的键位比较远，用第一种方法试了几次，小指头也抽筋了，所以用第二种方法，使用空格做分隔符，比较方便
 
配置
 
```vim
let g:table_mode_delimiter=' '
```
 
使用演示，先生成表格内容，用空格分隔。
 
```
表头1 表头2 表头2 长表头1 长长表头2
--- --- --- --- ---
内容1 内容2 长内容1 长长内容2 短1
长长内容3 短3 短4 短5 短6
短2 内容5 内容6 内容7 内容8
```
 
然后选中上述内容，再`: Tableize`，漂亮的表格就出来了。由于不是等宽字符的原因，表格线没有对齐，而在`vim`中表格线都对齐，很漂亮。
 
```
| 表头1     | 表头2 | 表头2   | 长表头1   | 长长表头2 |
| ---       | ---   | ---     | ---       | ---       |
| 内容1     | 内容2 | 长内容1 | 长长内容2 | 短1       |
| 长长内容3 | 短3   | 短4     | 短5       | 短6       |
| 短2       | 内容5 | 内容6   | 内容7     | 内容8     |
 
```
 
html的样子
 
| 表头1     | 表头2 | 表头2   | 长表头1   | 长长表头2 |
| ---       | ---   | ---     | ---       | ---       |
| 内容1     | 内容2 | 长内容1 | 长长内容2 | 短1       |
| 长长内容3 | 短3   | 短4     | 短5       | 短6       |
| 短2       | 内容5 | 内容6   | 内容7     | 内容8     |
 
