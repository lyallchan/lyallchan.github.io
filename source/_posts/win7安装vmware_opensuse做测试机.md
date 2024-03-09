toc: true
title: win7安装vmware_opensuse做测试机
description: win7安装vmware_opensuse做测试机
date: 2015-12-04 16:31:06
tags: [opensuse, vmware]
---
# 目标
 
win7通过虚拟机安装opensuse，做为开发服务器。有几个目标：
 
1. 32位win7，安装vmware workstation 10
2. win7开机启动vmware，并且启动里面的opensuse
3. win7关机，opensuse挂起
4. win7开机登陆特定用户，登陆后，立刻锁定
 
# 几个功能点
 
## vmware开机自启动
 
参考http://blog.vsharing.com/hug_funs/A1010796.html
 
在win7的计划任务中，新建任务
 
+ 名称随便取，这里取`VMWare`
+ 选择`不管用户是否登陆都要运行`
+ 触发器选择`在系统启动时`
+ 操作选择`启动程序`，命令行为`C:\Users\zqqian\bin\login.bat`
+ 其余选项用默认的即可
 
确认后，会让你输入登陆用户的用户名和密码
 
由于权限原因，vmware只能在当初安装的用户下运行，开机启动时，需要指定运行的用户
 
login.bat的内容
 
```bat
@echo off
 
REM startup opensuse in vmware
 
echo Starting OpenSUSE in VMware
 
"C:\Program Files\VMware\VMware Workstation\vmrun.exe" ^
-T ws start c:\users\zqqian\VOS\OpenSUSE\OpenSUSE.vmx nogui
 
echo done.
```  
 
`vmrun`是vmware自带命令，可以后台管理vmware，包括start\stop\suspend，注意最后的`nogui`，不启动图形界面
 
## vmware关机挂起
 
参考http://blog.vsharing.com/hug_funs/A1010796.html
 
在win7的`gpedit.msc`中，设置关机脚本，并配置关机选项
 
+ `计算机配置` - `windows设置` - `脚本` - `关机`
+ 添加关机脚本，脚本为`C:\Users\zqqian\bin\opensuse.stop.bat`
 
```bat
@echo off
 
REM stop opensuse in vmware
 
echo Stopping OpenSUSE in VMware
 
"C:\Program Files\VMware\VMware Workstation\vmrun.exe" ^
-T ws suspend c:\users\zqqian\VOS\OpenSUSE\OpenSUSE.vmx soft
 
echo done.
 
echo sleep 3s
sleep 3s
```
 
+ `计算机配置` - `管理模版` - `系统` - `关机选项`
+ 启用`关闭会阻止或者取消关机的应用程序的自动终止功能`
 
这里使用了`vmrun`的suspend功能，注意最后的`soft`，vmware会调用客户机的suspend脚本，保护客户机操作系统不受损坏。
 
suspend脚本通过vmware-tools安装，在客户机的`/etc/vmware-tools`下， 可以自行修改
 
据网上查阅
>win7在关机时，如果有非系统进程阻塞关机，默认会强制关闭进程。
`关闭会阻止或者取消关机的应用程序的自动终止功能`选项可以取消这种行为，
等待进程结束后，再关机，保证vmrun执行完毕。
 

实际测试的时候，不修改这个选项反而不会强制关闭进程。很奇怪！！
 
由于这台win7装了cygwin，一般也是ssh上去使用的，因此，alias了reboot命令，直接调用这个suspend脚本。
这样，reboot可以保证执行，不会强制关闭。
 
```bash
reboot='~/bin/opensuse.stop.bat;/cygdrive/c/windows/system32/shutdown.exe /r /t 0'
shutdown='~/bin/opensuse.stop.bat;/cygdrive/c/windows/system32/shutdown.exe /s /t 0'
```
 
## 开机自动登陆
 
`Netplwiz`启动用户账户对话框，取消勾选`要使用本机，用户必须输入用户名和密码`，确定后，输入用户名和密码。
 
## 登陆后，自动锁定
 
`rundll32 user32.dll,LockWorkStation`可以锁定用户，将这条命令加到计划任务，触发器选择`登录时`，用户选择之前开机自动登陆的用户。
 
## 客户机suspend脚本安装，实现软关机
 
vmware安装好opensuse，启动并登陆。
在vmware界面上，选择`虚拟机` - `安装vmware-tools`，vmware会把iso文件加载到虚拟机光盘中。
 
在opensuse，打命令安装
 
```bash
sudo mount /dev/cdrom /mnt
cd
mkdir tmp
cd tmp
tar -xf /mnt/VMwareTools-9.6.5-2700074.tar.gz
cd vmware-tools-distrib/
./vmware-install.pl
```
 
跟着提示安装，几个选项全部`yes`。
安装完成后，会提示你运行配置文件，几个选择全部选`no`。
安装和配置过程中，有几个告警，忽略。
 
安装完成后，suspend脚本为`/etc/vmware-tools/suspend-vm-default`，默认的脚本运行报错，用空文件替换。
 
```bash
cd /etc/vmware-tools
for i in `find ./ -name "*default"`; do
sudo mv $i $i.bak
sudo touch $i
sudo chmod 755 $i
done
```
 
