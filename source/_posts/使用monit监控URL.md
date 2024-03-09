toc: true
title: 使用monit监控URL
date: 2016-02-17 08:32:01
tags: [monit]
 
---
监控一个主机的WEB服务是否存活，最有效的方法是直接监控提供WEB服务的进程，如apache、nginx或者tomcat、was之类的，如果进程消失，肯定有故障。实际过程中，经常会有进程还在，但是不能对外提供服务了，有必要再增加直接监控的WEB端口的手段，WEB端口不能访问了，哪怕WEB服务进程还在，也需要告警。两种方式互为补充。
<!-- more -->
 
# monit
 
[monit](http://mmonit.com/monit/)是一个轻量的监控工具，大小不到1M，提供的功能很丰富，能监控本机的CPU、文件、目录、进程等基本信息，也能监控本机的网卡，包括网卡UP/Down、网络流量告警，还能监控其它主机的网络可达性，包括第4层协议、第7层协议，HTTP做为第7层协议自然也支持。发现告警之后，能定义各种告警动作，包括发送邮件，重启进程，运行特定脚本/程序。
 
集中式的监控系统，如nagios，zabbix，cacti，在一台主机上监控所有其它主机，存在单点故障。 monit一般部署在每台需要监控的主机上，可以认为是分布式系统，可以作为集中式监控的补充。
 
monit只有一个配置文件，在monit启动命令行指定。
下面记录monit安装和监控主机IP:PORT的过程。
 
# 安装
 
monit只有类unix版本，没有windows版本。ubuntu下安装很简单
 
```sh
sudo apt-get install monit
```
 
安装好以后，可执行文件为`/usr/bin/monit`，配置文件为`/etc/monit/monitrc`，daemon文件为`/etc/init.d/monit`，还有一个upstart的启动文件，若干man，example文件，总共1M不到。
 
# 配置
 
`/etc/monit/monitrc`本身就是一个详细的配置帮助文件，根据我们的目标，选择了如下的一些配置
 
## 基本的daemon配置
 
```
set daemon 3600
set logfile /var/log/monit.log
set idfile /var/lib/monit/id
set statefile /var/lib/monit/state
```
第1条命令指定monit运行在daemon模式，3600s也就是1个小时轮询一次。deamon模式是monit主要运行模式。如果不使用daemon模式，monit只运行一次，然后退出。monit运行在daemon模式，如果在命令行再次运行monit，则会发送wake-up信号给monit后台进程，然后马上做一次轮询。`monit quit`会kill掉monit后台进程。
其余3条命令指定log、id和state文件。
 
## email告警配置
 
```
set mailserver smtp.163.com
  username xxx@xxx.com
  password xxxx
  using sslv3
 
set mail-format {
    from: xxx@xxx.com
    reply-to: xxx@xxx.com
    subject: monit alert $SERVICE $EVENT
    message: Monit alert at $DATE
             $HOST $ACTION $SERVICE
}
set alert xxx@xxx.com
set eventqueue basedir /var/lib/monit/events slots 100
```
第一组命令指定smtp服务器以及相应的账号和ssl版本，smtp服务器用来发送告警邮件。
第二组命令指定告警邮件的格式，可用的变量为
+ $EVENT: 事件
+ $SERVICE: 服务名称。monit将每个监控对象定义成服务，实际上就是监控对象
+ $DATE: 告警日期和时间
+ $HOST: monit运行的主机，这在多台主机都运行monit的时候有用
+ $ACTION: 告警发生后，monit的动作。这个动作在定义监控服务的时候指定。
+ $DESCRIPTION: 告警描述
第三条命令指定告警邮件接收人，可以有多个，使用`,`分隔
第四条命令指定当smtp服务器有故障的时候，本地暂存告警邮件的位置以及数量，这里暂存100条
 
上面是全局的邮件告警配置，在后面配置单个服务的时候，还可以指定针对这个服务的告警配置，当两者有冲突时，以单个服务的配置为准。
 
## HTTPD配置
 
```
set httpd port 2812
  allow 127.0.0.1
  allow 10.6.86.226
  allow 10.6.86.228
  allow 10.6.86.1
  allow 172.17.42.1
  allow root:admin read-only
```
 
monit内置了一个http server，开启后，可以在浏览器上查看监控状态。monit强烈建议开启http server。monit的命令行命令（如monit status/quit）通过http接口和后台进程交互，如果不开启http server，相应的命令行命令不能使用。
monit还支持ssl方式，即可以用https访问，相对来说安全一点。https需要证书，具体配置可以看[官方文档](https://www.mmonit.com/monit/documentation/monit.html)。
 
前面5个allow，指定可以访问monit httpd的客户IP地址，不要忘了127.0.0.1，否则monit命令不能执行访问http接口。
最后一个allow指定web访问的账号，并且规定只读权限。
monit支持简单认证和PAM认证。上述方式是简单认证。简单认证发送的是明文账号，存在不安全性。monit建议在开启https功能后，使用简单认证。这里的使用环境是内网环境，相对安全，因此在没有开启https的情况下，也使用了简单认证。
PAM认证就是使用monit所在主机的系统账号，并且只支持账号组的方式，相应的配置格式为`allow @group`，在group组内的账号都可以web访问。
 
## 定义服务
 
定义服务，就是定义监控对象。monit文档都叫服务，我觉得这里叫做监控比较好。
 
### 服务类型
 
monit可以定义9种类型，包括process/file/fifo/filesystem/directory/system/program/network/host。
 
+ process: 监控指定进程是否存活
+ file/fifo/filesystem/directory: 监控指定的文件或者目录是否存在
+ system: 监控系统资源，包括CPU、内存和负载
+ program: 监控指定程序的退出代码或者运行时间的长短
+ network: 监控系统指定interface，包括up/down，带宽，误码率，流量，ping
+ host: 监控其它主机，可以定义端口、tcp/udp、protocol（http/mysql/websocket）。protocol为http的时候，还可以进一步定义url和返回内容
 
我们的目标是监控主机的web端口，因此用host类型，相应的配置语法为
 
```
check host <service> address <host ip address>
```
 
### 服务测试
 
服务测试提供了测试监控对象状态的一种方法，根据状态不同，可以指定相应的动作
 
通用语法为
 
```
if <test> then <aciton1> else if succeeded then <action2>
```
 
test有几种类型
1. 状态失效，如网络不可达，进程丢失如果失败
2. 参数变化，如文件权限，文件md5，文件大小，日期等
3. 参数达到门限，如CPU>80%, 磁盘空间大于85%
 
当test为真时，则执行action1
当从失效状态转到正常状态，则执行action2，这一句是可选项
 
action有5种类型
1. alert: 发送告警邮件
2. restart/start/stop: 重启/启动/停止service。在监控进程时特别有用，先定义start、stop、restart的命令，当进程丢失时，启动相应service，恢复进程
3. exec: 执行用户定义的命令或者脚本
 
### 监控web端口
 
```
check host csa35 with address 10.7.83.35
        if failed port 80 protocol http then alert
        if failed port 9080 protocol http then alert
        if failed port 9081 protocol http then alert
```
 
这个基本上是自然语言了。相应的业务逻辑为:
 
检查主机（check host），服务名称（$SERVICE）为csa35，ip地址为10.7.83.35，如果http 80/9080/9081 失败，则告警
 
这里不指定url，默认的url是`/`
 
### 其它的典型例子
 
从官网文档上找了一个典型例子，监控apache进程，比较全面展现了monit的能力
 
```
check process apache with pidfile /var/run/httpd.pid
       start program = "/etc/init.d/httpd start"
       stop program  = "/etc/init.d/httpd stop"
       if cpu > 40% for 2 cycles then alert
       if total cpu > 60% for 2 cycles then alert
       if total cpu > 80% for 5 cycles then restart
       if mem > 100 MB for 5 cycles then stop
       if loadavg(5min) greater than 10.0 for 8 cycles then stop
```
 
使用pidfile监控进程，服务名为apache
start命令
stop命令
如果2次轮询，进程cpu都大于40%，则告警
如果2次轮询，系统cpu都大于60%，则告警
如果5次轮询，系统cpu都大于80%，则重启（先stop，再start）
如果5次轮询，内存mem都大于100MB，则stop
如果8个次轮询，5分钟平均负载都大于10，则stop
 
# 启动
 
```bash
sudo /etc/init.d/monit start
```
 
# 查看
 
tail /var/log/monit.log
或者
web界面2812端口查看
