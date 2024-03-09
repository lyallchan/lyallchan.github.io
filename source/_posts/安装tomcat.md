toc: true
title: 安装tomcat
date: 2015-12-04 16:31:06
tags: tomcat
description: 安装tomcat

---
# 下载
 
Tomcat运行需要java和Tomcat。两个都选择最新版本，java是8u65，tomcat是v8。
 
下载到安装目录。安装目录为`~/webapps`
 
```sh
cd
mkdir webapps && cd webapps
wget http://javadl.sun.com/webapps/download/AutoDL?BundleId=111679 -o jre-8u65-linux-i586.tar.gz
wget http://mirrors.hust.edu.cn/apache/tomcat/tomcat-8/v8.0.28/bin/apache-tomcat-8.0.28.tar.gz
```
 
# 安装
 
解压缩到安装目录，完成安装。
 
```sh
cd ~/webapps
tar xf jre-8u65-linux-i586.tar.gz
tar xf apache-tomcat-8.0.28.tar.gz
ln -s jre-8u65-linux-i586 java
ln -s apache-tomcat-8.0.28 tomcat
mkdir download
mv jre-8u65-linux-i586.tar.gz download/
mv apache-tomcat-8.0.28.tar.gz download/
```
 
# 配置
 
设置环境变量：
 
在Linux下配置Tomcat最主要的问题就是配置环境变量。
 
由于是测试环境，采用人工启动和关闭，就在当前用户下设置环境变量。
 
在~/.bashrc增加
 
```sh
# setup for tomcat and java
export JAVA_HOME=~/webapps/java
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:.
export CATALINA_HOME=~/webapps/tomcat
export PATH=$JAVA_HOME/bin:$PATH
```
 
----------
 
另一种方法
 
根据tomcat官方文档`$CATALINA_HOME/RUNNING.txt`，在tomcat目录下的bin目录新建setenv.sh，设置环境变量
 
```sh
JAVA_HOME=~/webapps/java
CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:.
```
 
$CATALINA_HOME不要设置，tomcat的启动脚本会自动寻找启动目录，并将$CATALINA_HOME设置为启动目录
 
$PATH也不需要更改，做个软链接，将startup.sh和shutdown.sh链接到~/bin目录
 
```sh
ln -s ~/webapps/tomcat/bin/startup.sh ~/bin/tomcat.startup
ln -s ~/webapps/tomcat/bin/shutdown.sh ~bin/tomcat.shutdown
```
 
# 绑定地址和端口
 
在`CATALINA_BASE/conf/server.xml`中，更改tomcat绑定的端口和地址。
 
在host一节中，增加address="192.168.56.2"，绑定指定地址，端口默认为`8080`
 
# 启动和关闭
 
启动Tomcat:
$CATALINA_HOME/bin/startup.sh 或者 tomcat.startup.sh
 
关闭Tomcat：
$CATALINA_HOME/bin/shutdown.sh 或者 tomcat.shutdown.sh
 
查看日志：
cd $CATALINA_HOME/logs
tail -f catalina.out
 
查看版本：
$CATALINA_HOME/bin/catalina.sh version
 
# 更改默认页面的上下文
 
打算用apache做反向代理，虚拟路径`/tomcat`指向tomcat。
 
这就要求tomcat的默认页面也在`/tomcat`下。
 
简单把$CATALINA_HOME/webapps下的目录都改名为`tomcat#`的方式即可
 
具体的说明看tomcat config帮助的context一节，其中的(naming section)[/docs/config/context.html#Naming]
 
```sh
cd ~/webapps/tomcat/webapps
mv ROOT tomcat
mv docs tomcat#docs
mv examples tomcat#examples
mv manager tomcat#manager
mv host-manager tomcat#host-manager
~/webapps/tomcat/bin/shutdown.sh
~/webapps/tomcat/bin/startup.sh
```
 
# apache配置反向代理
 
在`~/.apache2/proxy.conf`增加
 
```
# URL from apache
Proxypass /server-info !
Proxypass /server-status !
Proxypass /manual !
 
# reverse proxy for tomcat
Proxypass /tomcat http://192.168.56.2:8080/tomcat
ProxypassReverse /tomcat http://192.168.56.2:8080/tomcat
```
 
# 第一个jsp页面
 
```jsp
mkdir $CATALINA_HOME/webapps/tomcat#t
cd $CATALINA_HOME/webapps/tomcat#t
cat > index.jsp << EOF
 
<%
java.util.Enumeration names = request.getHeaderNames();
while(names.hasMoreElements()){
    String name = (String) names.nextElement();
    out.println(name + ":" + request.getHeader(name) + "<br />");
}
%>
 
EOF
```
