toc: true
title: vim中自定义operator
description: vim中自定义operator
date: 2015-12-04 16:31:06
tags: [vim, operator]
---
# vim的杀手锏之一: {count}operator{motion/text_objects}
比如
 
+ operator = d 删除
+ motion = 2j 向下两行
 
那么
 
+ d2j 删除向下的两行
 
再比如
 
+ operator = y 拷贝
+ motion = 2w 向后两个单词
 
那么
 
+ y2w 拷贝后面的两个单词
 
常用的有
 
+ y$ 拷贝到行尾
+ yy 拷贝整行
+ yiw 拷贝当前单词，光标可以在单词的任意位置
+ d$ 删除到行尾，也可以大写的D
+ dd 删除整行
+ diw 删除当前单词
+ yi( 拷贝()中的内容
+ yit 拷贝html标记之间的内容
 
常用的operator
两个operator对当前行做操作
 
+ d
+ y
+ s
+ g~
+ gU
+ gu
 
常用的motion
 
+ 3j
+ 3k
+ 11G
+ gg
+ G
+ 0
+ ^
+ $
+ w/W
+ b/B
+ f{char}
+ t{char}
+ %
 
常用的text objects
 
+ iw
+ i(/ib
+ i[
+ i"
+ i'
+ it
 
# 用户自定义operator
 
自定义operator要用到`g@`和`operatorfunc`，这是vim预留给用户的operator接口。
`g@`是一个operator，`g@`调用`opfunc`指向的函数，并传入三个参数
 
1. motion的类型：line，char，block
2. '[：motion起点
3. ']：motion终点
 
在opfunc指向的函数中，可以使用这三个参数。
 
`:h g@`的说明
 
> When g@ is invoked, the function defined by the opfunc option is called with
an argument indicating the type of motion ("line", "char" or "block"). In
addition, the '[ and '] marks identify the start and end positions of the
motion.
 
# 自定义加序号的操作符
 
自定义操作符s，使得
 
+ ss = 当前行行首加序号
+ {count}ss = 下面count行，行首加序号
+ s{motion} = 当前行到motion终点行，行首加序号
+ {count}s{motion} = s{count*motion}
 
s原来是删除当前字符，并进入编辑模式。这个功能平时用的不多，比较习惯用`xi`达到同样效果，因此这里把s改成行首加行号的功能
 
## 函数Seqno()
 
函数Seqno()，实现从`'[`到`']`的行首加序号，序号从1开始
 
```vim
function! s:Seqno(type,...)
 
    " 加行号
    " 用户输入{count}operator{motion}的形式
    " 则函数作用的范围是{count}*{motion}，
    " '[ 起始行
    " '] 结束行
    " 比如 2d3j，函数作用范围是2*3=6，6行
    " '[ 表示当前行
    " '] 表示当前行+5
 
    let i = line("'[")
    let j = line("']")
    for l in range(i,j)
        execute l."s/^\\s*/\\0".(l-i+1).'. '
    endfor
 
endfunction
```
 
## 函数ExeSeqno()
 
函数ExeSeqno()，运行Seqno，并完成一下功能
 
1. 将opfunc指向Seqno，由于Seqno是s:范围的，需要先找到脚本的`<SID>`
2. 执行`g@`
 
```vim
function! s:ExeSeqno()
 
    "获取SID
    let s:sid=matchstr(expand('<sfile>'), '<SNR>\d\+_')
 
    "opfunc指向<SID>Seqno
    execute "set operatorfunc=".s:sid."Seqno"
 
    return 'g@'
 
endfunction
```
 
再定义一个函数，处理ss的场景，返回`g@g@`，直接运行，并作用于当前行
 
```vim
function! s:ExeSeqno2()
 
    "获取SID
    let s:sid=matchstr(expand('<sfile>'), '<SNR>\d\+_')
 
    "opfunc指向<SID>Seqno
    execute "set operatorfunc=".s:sid."Seqno"
 
    return 'g@g@'
 
endfunction
```
 
## map
 
```vim
vnoremap s :g/^\s*/ s//\=submatch(0).(line(".")-line("'<")+1).'. '<CR>
nnoremap <expr>s <SID>exeSeqno()
nnoremap <expr>ss <SID>exeSeqno2()
```
