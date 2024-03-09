toc: true
title: Win10下使用cygwinX server运行linux容器桌面，并实现中文输入
date: 2017-2-2 13:17:05
tags: [docker, desktop, 中文输入]
 
---
 
Win10安装Docker for windows，一般在docker中运行命令行程序。如果要在容器内运行桌面程序，那如何配置呢？

<!--more-->

[hub.docker.com](hub.docker.com)上搜索了一下，大多数在容器内安装桌面环境以及VNC Server或者Xpra，然后在Windows中安装VNC客户端或者Xpra，连接到容器，实现容器的桌面应用。

实际上也可以使用X11方案：在windows上运行Xserver，容器内运行Xclient。下面配置使用cygwinX实现windows的Xserver，容器内安装桌面程序，做为Xclient，连接至windows的X server。

# 安装cygwinX

运行cygwin的setup，选择xorg-server和xinit，一路next，完成安装。启动命令是

```
startxwin
```

[cygwinX](https://x.cygwin.com/)在1.7版本之后，因为安全原因，默认不再监听TCP端口，只监听socket端口。也就是只能运行本地X client，或者通过ssh运行远程的X client。

也有[资料](https://linux.cn/article-5579-1.html)说明，mac上安装x server，容器运行时可以映射mac的/tmp/.X11-unix到容器内的/tmp/.X11-unix，实现容器内运行桌面环境，显示在mac的x server上。但是在cgywinX和docker for windows环境下，实验不成功。

在容器内安装ssh server比较麻烦，这里就启用cygwinX的TCP功能，通过TCP连接容器。

```
startxwin -- -listen tcp
xhost +  
```

启动xwin，并关闭xhost的access control，使得任意IP的x client都能连上xwin。

startxwin默认启动xwin multiwindow模式，这样就在windows上启动了一个X server，默认的端口为:0.0

# 容器内运行单个GUI程序

比如运行一个firefox。按照普通方法安装firefox，配置好中文字体。

dockerfile

```
FROM ubuntu:16.04
MAINTAINER lyalchan "lyallchan@163.com"

RUN apt-get update && apt-get -y install firefox

RUN apt-get install -y language-pack-zh-hans
RUN mkdir /usr/share/fonts/win
ADD simsun.ttc /usr/share/fonts/win/
ADD simhei.ttf /usr/share/fonts/win/
RUN mkfontscale && mkfontdir && fc-cache -fv

RUN apt-get autoclean && apt-get autoremove
```

运行的时候，指定DISPLAY环境变量到windows的X server端口。即

```
docker run \
    -e DISPLAY=10.0.75.1:0.0 \
    --rm \
    -it \
    --name firefox \
    --hostname firefox \
    firefox:latest \
    firefox
```

10.0.75.1是windows的一个IP地址，是docker的网桥网关。

这种方法的优点在于不需要在容器内安装桌面，只要安装应用程序，桌面管理由cygwinX完成，生成的镜像小，配置简单。

运行的时候，firefox好像是本地windows的一个应用，有windows的标题栏等等，实际却在容器内运行。

# 运行容器内桌面，实现中文输入

上面的运行方式有个问题，能显示中文，却无法输入中文。查了很多资料也找不到只安装GUI程序和输入法的情况下，正确的输入中文。

那么尝试在容器内安装桌面环境。

## 修改startxwin

xwin有[三种运行模式](https://x.cygwin.com/docs/man1/XWin.1.html)，singlewindow，multiwindow和rootless。

- singlewindow模式：xwin使用远程桌面管理器，管理所有的远程X应用，在本地只显示一个窗口
- multiwindow模式：xwin使用内置的桌面管理器，远程每个X应用都拥有一个独立窗口，好像是本地应用一样。上面的运行方式就是这个方式
- rootless模式：基本是singlewindow，不过在没有远程桌面连接时，隐藏xserver的root窗口，也就是不显示xserver的窗口。

要在容器内运行桌面，需要使xwin运行在singlewindow或者rootless模式，需要修改startxwin。multiwindow也可以运行，但是显示比较奇怪。

startxwin是一个shell程序，很容易找到xwin的serverargs

vi /usr/bin/startxwin

```
# commeted by lyallchan, uncommet to return default behavier
# serverargs="-multiwindow $serverargs"
###
serverargs="-listen tcp -rootless $serverargs"
```

startxwin默认启动xwin的multiwindow模式，修改startxwin，使得xwin监听TCP端口，运行在rootless模式。

## 容器内安装桌面和中文输入法

选来选去，桌面安装xfce4，中文输入法安装fcitx和fcitx-pinyin。

dockerfile

```
FROM ubuntu:16.04
MAINTAINER lyalchan "lyallchan@163.com"

ENV DEBIAN_FRONTEND noninteractive
ENV GTK_IM_MODULE fcitx
ENV QT_IM_MODULE fcitx
ENV XMODIFIERS "@im=fcitx"
ENV LC_CTYPE zh_CN.UTF-8

RUN apt-get update && apt-get -y install firefox

RUN apt-get install -y language-pack-zh-hans
RUN mkdir /usr/share/fonts/win
ADD simsun.ttc /usr/share/fonts/win/
ADD simhei.ttf /usr/share/fonts/win/
RUN mkfontscale && mkfontdir && fc-cache -fv

RUN apt-get install -y xfce4 xfce4-terminal
RUN apt-get install -y fcitx fcitx-pinyin

RUN mkdir -p /root/.config/fcitx
RUN echo [Hotkey] > /root/.config/fcitx/config
RUN echo TriggerKey=CTRL_ALT_SPACE >> /root/.config/fcitx/config
RUN echo SwitchKey=Disabled >> /root/.config/fcitx/config
RUN echo IMSwitchKey=False >> /root/.config/fcitx/config

RUN apt-get autoclean && apt-get autoremove
```

apt-get xfce4的时候，会和用户交互，导致build失败，要设置成noninteractive模式。

安装好输入法之后，要设置3个[环境变量](https://fcitx-im.org/wiki/Input_method_related_environment_variables)，使系统默认启用fcitx输入框架。

配置fcitx的输入法切换键为CTRL_ALT_SPACE，以避免和windows的输入法快捷键冲突。

## 运行容器

运行比较复杂，写个脚本简化操作。

判断xwin是否运行，如果不运行，在tmux中启动xwin，不在后台启动xwin，可以避免屏幕混乱。

关闭xhost的access control，使得任意IP的x client都能连上xwin。

启动容器，并启动fcitx和xfce4。

```
#!/bin/bash 

D=10.0.75.1:0.0

if ! ps | grep xinit ; then
    tmux new-session -d -s xwin startxwin
fi

xhost + 

winpty \
    docker  run \
    -e DISPLAY=$D \
    --rm \
    -it \
    --name=firefox \
    --hostname=firefox \
    firefox:on \
    bash -c "fcitx; startxfce4"

```

# 参考资料

1. hub.docker.com上的桌面镜像

    https://hub.docker.com/r/queeno/ubuntu-desktop/
    https://hub.docker.com/r/rogaha/docker-desktop/
    https://hub.docker.com/r/sevnew/ubuntu-xfce-vnc-desktop-chrome/

2. cygwinX1.7之后，不再监听TCP端口

    https://x.cygwin.com/docs/faq/cygwin-x-faq.html#q-xserver-nolisten-tcp-default
    
3. MAC下运行桌面容器

    https://linux.cn/article-5579-1.html
    
4. ubuntu安装宋体

    http://www.2cto.com/os/201308/239373.html
    
5. xwin的man文档，详细说明了xwin的三种运行模式

    https://x.cygwin.com/docs/man1/XWin.1.html
    
6. fcitx的环境变量设置

    https://fcitx-im.org/wiki/Input_method_related_environment_variables
    https://wiki.archlinux.org/index.php/Fcitx

