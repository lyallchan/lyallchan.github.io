toc: true
title: junos cli调用shell commands
date: 2016-5-13 11:24:05
tags: [junos, shell, event]
 
---
[TOC]
 
JUNOS是基于FREEBSD修改的操作系统，用户JUNIPER全系列产品。其中的CLI提供了和cisco IOS类似的命令行配置界面，功能强大。
退出cli，可以进入junos的操作系统界面，也就是freebsd的界面，也可以使用一些junos自带的freebsd命令，比如ls、grep等。
但是cli中没有提供直接调用这些命令的方法，一些自动（automation）命令的场景完成比较复杂。
 
比如，PPPOE拨号成功后，发送微信信息进行通知，或者有非法用户试图登陆，发送微信进行告警。
 
PPPOE拨号成功，非法用户试图登陆都可以通过event（事件）获取，发送微信也可以通过命令行发送，参见[这篇文章](http://lyallchan.github.io/2016/03/25/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%E5%88%B0%E5%BE%AE%E4%BF%A1%E4%BC%81%E4%B8%9A%E5%8F%B7/)。
获取event以后，怎样调用命令行脚本呢？这是个难题。
 
 
查阅了很多资料，有几种方式可以完成
 
1. junos的脚本语言，slax可以调用本地命令
2. event有个execute commands功能，可以直接执行cli命令，cli有个ssh命令，可以ssh到另外一台机器执行远程命令
 
下面实现第二种方案，**实现PPPOE拨号成功后，发送微信通知**
 
<!--more-->
 
# junos配置
 
```
[edit event-options]
root@SRX220# show | no-more
policy pppoe-up-alert {
    events snmp_trap_link_up;
    attributes-match {
        snmp_trap_link_up.interface-name matches pp0.0;
    }
    then {
        execute-commands {
            commands {
                "show system uptime";
                "ssh root@192.168.1.210";
            }
            output-filename ssh210.txt;
            destination cfroot;
            output-format text;
        }
    }
}
destinations {
    cfroot {
        archive-sites {
            /cf/root;
        }
    }
}
```
 
定义一个event，捕获snmp_trap_link_up事件，也就是端口up事件，并过滤其中的interface-name为pp0.0，pp0.0为PPPOE up起来的端口。
 
也就是在所有端口up事件中，只关注PPPOE up的事件。
 
成功捕获后，执行cli命令`ssh root@192.168.1.210`，登陆到另外一台机器上去。另外一台机器在登陆文件中部署了发送微信的脚本。
 
# 210的配置
 
```
# juniper srx access
if echo $SSH_CLIENT | grep 192.168.1.252 ; then
    sleep 10
    echo -n `date` ": " >> ~/252.log
    ~/bin/wxsender `wget http://members.3322.org/dyndns/getip -O - -q` >> ~/252.log
    exit
fi
 
```
 
上面的脚本是.bashrc的一部分，先判断是否是junos登陆（junos所在IP为192.168.1.252），如果是，则休眠10s，写日志，并调用微信发送外网地址。
 
`wget http://members.3322.org/dyndns/getip`得到外网地址，`-O -`输出到屏幕，`-q`安静模式，只输出结果，不输出多余的调试信息。
 
`wxsender`具体内容参见[这篇文章](http://lyallchan.github.io/2016/03/25/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%E5%88%B0%E5%BE%AE%E4%BF%A1%E4%BC%81%E4%B8%9A%E5%8F%B7/)。
 
