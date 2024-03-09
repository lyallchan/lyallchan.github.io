toc: true
title: apache反向代理tomcat，配置session保持
date: 2015-12-04 16:31:07
tags: [apache, tomcat, proxy]
---
配置tomcat集群，关键是要解决session的问题，这篇文章讲述了自己理解的session含义，apache和tomcat如何配置以解决session问题。
<!--more-->

# session是什么
 
http是无状态协议，两次http请求之间没有任何联系。
实际上，有很多情况，后面一次请求，需要看前一次的结果。
最典型的就是用户登录，第一次http请求重定向到登录页面，用户登录后，第二次http请求就不需要再重定向到登录页面。
记录用户登录的信息，就是session，一般使用cookie实现session。
 
# tomcat集群的session
 
如果tomcat使用session，那么单单配置load balance会引起session丢失。
比如系统需要记录用户登录信息，有两台应用服务器，用户在第一台应用服务器上登录，如果第二次http请求定位到第二台应用服务器，第二台应用服务器没有登录信息，session丢失了，会让用户再登录一次，引起错误。
解决方法有两个：
1. session replication，session复制。第一台应用服务器有session信息后，复制到所有应用服务器。
2. session sticky，session保持。用户第一次访问应用服务器后，以后所有访问都访问同一台应用服务器。
两个方法各有优缺点，session复制耗费较多资源，但是基于每次http访问进行负载均衡，更加均衡。session保持比较节约资源，但是基于用户访问进行负责均衡，负载均衡程度取决于用户的访问模式，同时应用服务器切换时，会引起会话丢失，用户需要重新登录。
session复制在tomcat文档中有详细的说明，甚至专门有一篇[HowTo](http://tomcat.apache.org/tomcat-8.0-doc/cluster-howto.html)，下面重点介绍一下session sticky的配置方法。
 
# 配置session sticky的两种模式
 
session sticky的目标是记住用户第一次http请求访问的应用服务器，之后的http请求仍然访问这台应用服务器。
一般使用cookie记录第一次访问的应用服务器标识，有两种方法
1. 在apache中打标识，然后通过apache设置cookie，实现session sticky。
2. 在tomcat中打标识，tomcat设置cookie，apache识别cookie，分配相应的应用服务器，同样实现session sticky。
 
# 基于apache的session sticky
 
1. 在BalancerMember中用route指令指定应用服务器标识。
2. 然后用Header指令在HTTP头中设置Cookie，Cookie中指定本次访问的应用服务器标识。
3. 增加ProxySet，指定依据ROUTEID使用stickysession方式。
上面方法在Apache [proxy_balancer官方文档](http://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html)中有详细说明。
用这种方法只需要在apache中配置，tomcat不需要特别的配置。
 
```
# reverse proxy for tomcat
<Proxy balancer://tomcat>
    BalancerMember http://192.168.56.2:8080 route=1
    BalancerMember http://192.168.56.2:8180 route=2
    ProxySet stickysession=ROUTEID
    Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; secure" env=BALANCER_ROUTE_CHANGED
</Proxy>
 
Proxypass /tomcat balancer://tomcat/tomcat
ProxypassReverse /tomcat balancer://tomcat/tomcat
```
 
# apache和tomcat联合配置实现session stikcy
 
1. 在tomcat配置中，指定应用服务器标识。根据[文档](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html)，在engine中指定。
2. apache配置中，增加ProxySet，指定依据JESSIONID使用stickysession方式。JESSIONID是tomcat默认设置的cookie值。
 
tomcat配置，两个应用服务器分别指定`1`和`2`
```
     <Engine name="Catalina" defaultHost="localhost" jvmRoute="1">
```
 
apache配置
```
# reverse proxy for tomcat
<Proxy balancer://tomcat>
    BalancerMember http://192.168.56.2:8080
    BalancerMember http://192.168.56.2:8180
    ProxySet stickysession=JSESSIONID
</Proxy>
 
Proxypass /tomcat balancer://tomcat/tomcat
ProxypassReverse /tomcat balancer://tomcat/tomcat
```
