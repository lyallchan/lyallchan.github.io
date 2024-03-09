toc: true
title: tomcat_apache做负载均衡
date: 2015-12-04 16:31:06
tags: [apache, tomcat, load balance]
description: 建立apache和tomcat的负载均衡
---
# 目标
 
+ 建立apache和tomcat的负载均衡
+ 不考虑session复制
 
# tomcat配置
 
前期tomcat配置完成
 
+ 安装目录`~/webapps/apache-tomcat-8.0.28`
+ 使用192.168.56.2:8080端口
+ 启动命令tomcat.startup/tomcat.shutdown
+ 没有设置环境变量，而是利用启动脚本自己定位$CATALINA_BASE，在bin/setenv.sh中设置classpath和java
 
一摸一样复制一套到另外一个目录，并设置软链接
 
```bash
 
cp -rf ~/webapps/apache-tomcat-8.0.28/ ~/webapps/apache-tomcat-8.0.28-1/
ln -s ~/webapps/apache-tomcat-8.0.28-1 ~/webapps/tomcat-1
ln -s ~/webapps/apache-tomcat-8.0.28-1/bin/startup.sh ~/bin/tomcat-1.startup
ln -s ~/webapps/apache-tomcat-8.0.28-1/bin/shutdown.sh ~/bin/tomcat-1.shutdown
 
```
 
将`~/webapps/tomcat-1/conf/server.xml`的所有`80**`端口都换成`81**`
 
即第二个tomcat为
+ 安装目录`~/webapps/apache-tomcat-8.0.28-1`
+ 使用192.168.56.2:8180端口
+ 启动命令tocmat-1.startup/tomcat-1.shutdown
+ 没有设置环境变量
 
分别启动
 
```bash
tomcat.startup
tomcat-1.startup
```
 
# apache2配置
 
load几个模块，`proxy_balancer`，`slotmem_shm`，`lbmethod_heartbeat`，`lbmethod_bytracffice`，`lbmethod_byrequests`，`lbmethod_bybusyness`
 
在~/.apache2/proxy.conf中，增加balancer配置
 
```conf
 
# reverse proxy for tomcat
<Proxy balancer://tomcat>
    BalancerMember http://192.168.56.2:8080
    BalancerMember http://192.168.56.2:8180
</Proxy>
Proxypass /tomcat balancer://tomcat/tomcat
ProxypassReverse /tomcat balancer://tomcat/tomcat
 
<Location /bm>
    SetHandler balancer-manager
</Location>
Proxypass /bm !
```
 
# 测试
 
apache提供的地址是`https://10.6.86.228`，
测试地址为`https://10.6.86.228/tomcat/t`，显示server的IP:PORT
 
每刷新一次页面，server地址交替显示`192.168.56.2:8080`和`192.168.56.2:8180`
 
 
