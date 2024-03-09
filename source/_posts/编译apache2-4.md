title: 编译apache2.4
description: 编译apache2.4
toc: true
tags: [apache, 编译]
date: 2015-12-04 15:34:29
---
# apache2.4的特殊之处
 
Apache 2.4的源文件中，删除了必要的apr和apr-util等程序包，直接编译2.4的时候，会提示各种出错。
 
2.4的[官方文档](http://httpd.apache.org/docs/2.4/install.html)有详细的描述
 
> **APR and APR-Util**
Make sure you have APR and APR-Util already installed on your system. If you don't, or prefer to not use the system-provided versions, download the latest versions of both APR and APR-Util from Apache APR, unpack them into /httpd_source_tree_root/srclib/apr and /httpd_source_tree_root/srclib/apr-util (be sure the directory names do not have version numbers; for example, the APR distribution must be under /httpd_source_tree_root/srclib/apr/) and use ./configure's --with-included-apr option. On some platforms, you may have to install the corresponding -dev packages to allow httpd to build against your installed copy of APR and APR-Util.
**Perl-Compatible Regular Expressions Library (PCRE)**
This library is required but not longer bundled with httpd. Download the source code from http://www.pcre.org, or install a Port or Package. If your build system can't find the pcre-config script installed by the PCRE build, point to it using the --with-pcre parameter. On some platforms, you may have to install the corresponding -dev package to allow httpd to build against your installed copy of PCRE.
 
 
大致意思为
 
> 编译Apache 2.4前，确保APR和APR-Util已经安装，如果没有安装，或者不想使用系统提供的APR版本，从[APR网站](http://apr.apache.rog)下载最新版本的APR和APR-Util，将它们解压缩到apache2.4源代码的srclib目录，不能带版本号，即`/httpd-source-tree-root/srblic/apr/`。
> apache用到PCRE，2.4版本没有包括PCRE模块。从[PCRE](http://www.pcre.org)上下载源代码，编译安装。
> apache ./configure的时候，使用--with-included-apr包含apr源码，使用--with-pcre指定编译安装后的PCRE位置。
 
 
下面简单记录ubuntu下编译安装apache 2.4的过程
 
# 环境准备
 
```bash
apt-get update
apt-get -y install build-essential
 
cd /
mkdir apache2 && cd apache2
 
wget http://apache.fayea.com//httpd/httpd-2.4.17.tar.gz
wget http://apache.fayea.com//apr/apr-1.5.2.tar.gz
wget http://apache.fayea.com//apr/apr-util-1.5.4.tar.gz
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.37.tar.gz
```
 
pcre下载有点麻烦，http下载方式被墙，ftp方式还可以用。找到下载链接后，直接wget。
 
# 编译pcre到/apache2/pcre
 
```bash
cd /apache2
tar xf pcre-8.37.tar.gz
cd pcre-8.37
./configure --prefix=/apache2/pcre
make && make install
```
 
# 编译apache2到/apache2/httpd
 
```bash
 
# 解压缩apache源码
cd /apache2
tar xf httpd-2.4.17.tar.gz
 
# 解压缩apr和apr-util到apache源码目录的srclib目录，不能有版本号
cd httpd-2.4.17/srclib
tar xf ../../apr-1.5.2.tar.gz
tar xf ../../apr-util-1.5.4.tar.gz
mv apr-1.5.2 apr
mv apr-util-1.5.4 apr-util
 
# 编译apache到/apache2/httpd
# 同时编译proxy相关的模块
cd /apache2/httpd-2.4.17
./configure \
--prefix=/apache2/httpd \                       # 指定安装位置
--with-included-apr \                           # 指定apr源文件位置
--with-pcre=../pcre \                           # 指定prce位置
--enable-proxy=shared \                         # 下面几行都是编译proxy模块
--enable-proxy_ajp=shared \
--enable-proxy_connect=shared \
--enable-proxy_http=shared \
--enable-proxy_balancer=shared \
--enable-slotmem_shm=shared \
--enable-lbmethod_heartbeat=shared \            # loadbalancer的4种方式
--enable-lbmethod_byrequests=shared \
--enable-lbmethod_bybusyness=shared \
--enable-lbmethod_bytraffic=shared
make && make install
```
 
