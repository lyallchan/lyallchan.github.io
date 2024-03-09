toc: true
title: 制作windows7的vagrant box
date: 2016-02-06 13:04:05
tags: [windows, vagrant]
 
---
[TOC]
打算使用windows7的虚拟机来进行一些软件的安装测试等等，测试完成以后，把镜像删除，不留痕迹。这就要求有比较快捷的方式随时生成虚拟机、删除虚拟机。
 
<!--more-->
 
# 技术选择
 
这个需求和docker比较像，可是windows下docker支持的不好，而且好像也不能基于docker生成windows容器
 
## virtualbox
 
virtualbox有链接复制的功能，先安装好一个基本虚拟机，然后备份（系统快照），基于备份再复制一个虚拟机，复制的时候，选择“链接复制”，就可以快速生成一个新的虚拟机。
 
新虚拟机使用基本虚拟机的硬盘（只读），同时也会生成一个新的虚拟硬盘，记录新虚拟机上的操作（读写）。是不是和docker的分层容器比较像！！
 
新虚拟机使用完毕以后，可以随时删除，不影响基本虚拟机。
 
基本的虚拟机可以导出为ova文件，可以进行转移。
 
## vagrant
 
如果喜欢命令行，基于vagrant也可以实现上面的功能，很方便，基本的步骤为
 
1. 安装windows7虚拟机，并安装virtualbox增强工具
2. 对虚拟机进行一些必要的配置，主要是开启winrm，方便vagrant管理
3. 生成vagrant box
4. 按照正常的vagrant使用即可
 
## shadow defender
 
shadow defender(SD)可以保护系统。受SD保护的系统所有磁盘写操作都重定向到一个虚拟的文件系统，系统重启后，恢复到初始状态，好像写操作不存在一样。
 
这种功能对测试软件来说，最有用。上面的新虚拟机安装完成以后，马上安装SD。软件测试完成之后，重启系统，恢复初始状态。
 
SD安装使用比较简单，跟着屏幕上操作就行。
 
# 必要的工具
 
+ [virtualbox以及增强工具](www.virtualbox.org)
+ windows7安装盘
+ [vagrant](www.vagrantup.com)
+ [shadow defender](www.shadowdefender.com)
 
# 安装windows7
 
这没有什么好讲的，按照常规安装，再安装增强工具。安装完成后，虚拟硬盘大约10G
 
虚拟机参数为内存1G，显存18m，CPU1核，硬盘40G，增加模式，这些参数可以根据自己的机器修改，不要让虚机的运行影响本机就可以
 
用户名和密码都为vagrant
 
# 虚拟机配置
 
+ 去除光驱（以后用不到的）
+ 关闭系统还原（虚拟机中用不到）
+ 允许远程连接（可能会用到，这里先开着）
+ 网络位置设为“工作网络”（开启winrm的先决条件）
+ 共享粘贴板设为“双向”
+ 开启winrm，在管理员权限的cmd中开启，脚本为
```
winrm quickconfig -q
winrm set winrm/config/winrs @{MaxMemoryPerShellMB="512"}
winrm set winrm/config @{MaxTimeoutms="1800000"}
winrm set winrm/config/service @{AllowUnencrypted="true"}
winrm set winrm/config/service/auth @{Basic="true"}
sc config WinRM start= auto
```
 
上面的配置结束以后，关闭虚拟机
 
# 生成vagrant box
 
我的机器上有套cygwin环境，下载windows版本的vagrant，默认安装。
 
安装完成以后，在cygwin下生成软链接指向vagrant.exe
 
```
ln -s v /path/to/vagrant.exe
```
 
假定虚拟机的名字为windows7_base，windows的主目录为$WINHOME
 
```
cd $WINHOME/Virtualbox\ VMs/windows7_base
v package --base windows7_base --output win7_pro_x64.box
v box add win7_pro_x64.box --name win7
```
 
这样就生成了一个win7的box名称为win7_pro_x64.box，可以备份以后使用，并且加入了这个box，名称为win7
 
生成的box大小约为3G
 
# 使用
 
新建个目录，生成vagrantfile，启动
 
```
cd
mkdir tmp
cd tmp
v init win7
```
 
在当前目录下，会生成Vagrantfile，将Vagrantfile修改为
 
```
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.box = "win7"
  config.vm.communicator = "winrm"
  config.vm.provider "virtualbox" do |vb|
    vb.linked_clone = true
    vb.gui = true
    vb.name = "plsql"
  end
end
```
 
基于box名称为win7，就是上面v box add --name后面的名字
基于winrm和虚拟机通信，就是上面为什么在虚拟机中要开启winrm的原因
采用链接clone，就是上面说的链接复制的功能
第一次使用的时候，会生成一个win7的基本虚拟机，然后再生成一个“链接复制”的虚机，以后再运行的时候，直接生成“链接复制”的虚机，速度很快
启动gui，可以直接在gui中操作虚拟机
虚拟机名字叫plsql，每生成一个虚机，这个名字要更改的，否则会失败
 
`v up`启动，启动完成后，虚拟机也就生成了，可以按照一般的虚机方式使用。
 
删除虚机时，直接在virtualbox中删除plsql虚拟机。注意其中的基本虚拟机不能删除，否则以后使用会报错。
 
 
# 注册码问题
 
使用vagrant方式，即使基本虚拟机注册好了，新生成的虚拟机也不会注册，需要重新注册激活
 
如果虚机使用时候比较长，建议在控制面板中升级windows defender，升级windows update
 
