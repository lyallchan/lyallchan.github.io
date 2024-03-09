toc: true
title: 容器化Hbase standalone、Phoenix和Squirrel
date: 2017-2-4
tags: [docker, hbase, squirrel, GUI]
 
---

为了学习和测试Hbase，在Docker中安装Hbase Standalone和Phoenix。另外再安装GUI客户端Squirrel。上面三个组件都容器化，并实现互相访问。

<!--more-->

+ [HBASE](http://hbase.apache.org)：基于列的NO SQL数据库，是HADOOP重要组成部分
+ [Phoenix](http://phoenix.apache.org/index.html)：HBASE插件，提供HBASE的SQL操作语句，并提供JDBC接口
+ [Squirrel](http://squirrel-sql.sourceforge.net/)：是Phoenix主页上推荐的GUI客户端，基于JAVA编写，通过JDBC连接数据库。不仅仅用于Phoenix，也可以用于其它关系型数据库

# 安装Hbase和Phoenix

先下载好Hbase和Phoenix，分别是[hbase 1.2.4](http://mirrors.hust.edu.cn/apache/hbase/stable/hbase-1.2.4-bin.tar.gz)和[Phoenix 4.8.2](http://mirror.bit.edu.cn/apache/phoenix/apache-phoenix-4.8.2-HBase-1.2/bin/apache-phoenix-4.8.2-HBase-1.2-bin.tar.gz)（对应hbase1.2版本）

dockerfile

```
# standalone hbase image with phoenix
# 
# docker build --force-rm -t hbase .
# 

FROM java:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV HBASE_VERSION 1.2.4

RUN apt-get update && apt-get install python -y 

ADD hbase-${HBASE_VERSION}-bin.tar.gz /
RUN ln -s /hbase-${HBASE_VERSION} /hbase
ADD apache-phoenix-4.8.2-HBase-1.2-bin.tar.gz /
RUN ln -s /apache-phoenix-4.8.2-HBase-1.2-bin /phoenix

ADD hbase-site.xml /hbase/conf/
RUN cp /phoenix/*.jar /hbase/lib/
ENV PATH /hbase/bin:/phoenix/bin:$PATH

CMD start-hbase.sh && sleep 99999999

```

java:on是一个java环境

python是因为phoenix的命令行客户端`sqlline.py`是基于python的，如果不需要运行，也可以不用安装

hbase-site.xml参考官网[standalone](http://hbase.apache.org/book.html#quickstart)的配置

hbase-site.xml

```
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>file:///v</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/v</value>
  </property>
</configuration>

```

[安装phoenix](http://phoenix.apache.org/installation.html)比较简单，只要把phoenix/lib目录下所有的jar文件拷贝到hbase/lib目录下，重启hbase服务就可以了。

# 运行Hbase

start_hbase.sh

```

if ! docker network ls | grep hbase; then
    docker network create hbase
fi

docker run \
    --volume v:/v \
    -p 16010:16010\
    -d \
    --hostname hbase_standalone\
    --name hbase_standalone \
    --net hbase \
    hbase

```

新增网络`hbase`，使得容器之间可以互相访问。
    
>docker1.9版本之后的网络功能大大增强，对于用户自定义网络，内置了一个DNS，在同一个网络之下，可以通过hostname相互访问，不再需要container link

映射一个目录，保存hbase数据内容，这个目录对应hbase-site.xml中的hbase.rootdir

暴露端口16010，hbase WEB管理页面默认端口

hbase起来之后，实际上phoenix也起来了，在同一个容器里面可以运行phoenix命令行sqlline

```
docker exec -it hbase /phoenix/bin/sqlline.py localhost
```

为了测试用，也可以使用同一个镜像另起一个容器，运行sqlline

```
docker run \
    --volume sql:/sql \
    -it \
    --rm \
    --hostname phoenix\
    --name phoenix \
    --net hbase \
    hbase \
    sqlline.py hbase_standalone
```

这里的--net hbase和hbase容器的net一致

sqlline.py指向hbase主机名，hbase主机名在运行hbase容器的时候定义

# 安装Squirrel

基于之前的DOCKER中运行GUI程序的镜像，安装Squirrel包，并做必要的配置

先将Phoenix/lib目录下的`phoenix-4.8.2-HBase-1.2-client.jar`拷贝出来，这是phoenix客户端组件，需要加载到squirrel目录中

dockerfile

```
# squirrel: hbase phonix client based on xcfe4:on
# 
# docker build --force-rm -t squirrel .

FROM xcfe4:on
MAINTAINER lyalchan "lyallchan@163.com"

RUN apt-get update && apt-get install -y openjdk-8-jre

ENV VERSION 3.7.1

COPY squirrel.xml /
COPY squirrel-sql-$VERSION-standard.jar /

RUN java -jar /squirrel-sql-$VERSION-standard.jar /squirrel.xml
COPY phoenix-4.8.2-HBase-1.2-client.jar /squirrel/lib/

RUN cp -R root  root.backup
```

xcfe4:on是自制的一个运行GUI应用程序的基础镜像，包括中文输入法

squirrel.xml是安装squirrel时的配置文件

>squirrel通过`java -jar`安装，安装过程中，会和你交互回答很多配置项,这个文件回答了这些配置，实现自动安装

将phoenix客户端组件`phoenix-4.8.2-HBase-1.2-client.jar`放到/squirrel/lib目录

将root用户的配置备份出来，运行时要用到

# 运行Squirrel

start_squirrel.sh

```
#!/bin/sh 

C_NAME="squirrel"
V=`cygpath -aw $(dirname $0)/v`
NETWORK="hbase"

if ! docker network ls | grep $NETWORK; then
    echo start hbase first. quitting...
    exit 127
fi 

if ! ps | grep xinit ; then
    tmux new-session -d -s xwin startxwin
fi

D=10.0.75.1$DISPLAY
xhost + 

docker run \
    -itd \
    --volume $V:/root:rw \
    -e DISPLAY=$D \
    --net $NETWORK \
    --rm \
    --hostname $C_NAME\
    --name $C_NAME \
    $C_NAME \
    bash -c "fcitx; startxfce4"

```

运行squirrel，先要运行hbase和xwin

再设置DISPLAY环境变量传到容器中，10.0.75.1是本机DOCKER的网桥地址

DOCKER RUN的时候将root目录映射出来，很多squirrel的配置都要保存下来，--net要和hbase的net一致，保证squirrel可以访问到hbase，启动输入法fcitx和桌面xcfe4，这两个程序都安装在xcfe4:on镜像中。

拉起桌面以后，先将/root.backup中的配置拷贝回/root，这个动作只要做一次就够了。

```
docker exec squirrel cp -R /root.backup/* /root/
或者
docker cp squirrel:/root.backup/* v
```

启动squirrel

```
docker exec squirrel /squirrel/squirrel-start.sh
```

按照phoenix文档配置squirrel

>1. 新增Driver(Drivers -> New Drvier)
>2. 在"Add Driver"对话框中，设置"Name"为"Phoenix"，"Example URL"为"jdbc:phoenix:localhost"
>3. 在同一个对话框，设置"Class Name"为"org.apache.phoenix.jdbc.PhoenixDriver"
>4. 新增Alias(Aliases -> New Aliases)
>5. 在对话框中，"Name"设置为"Hbase_standalone", "Driver"设置为"Phoenix", "User Name"和"Password"留空
>6. URL设置为"jdbc:phoenix:hbase_standalone"，其中hbase_standalone为hbase的主机名
>7. 测试，通过后按"OK"关闭对话框
>8. 双击新增的alias，连接数据库即可

关闭squerrel容器后，所有配置都保存在当前目录的"./v"目录下，这个目录映射为容器内的"/root"目录

# 参考资料

1. hbase官方网站、下载页面和standalone配置

    http://hbase.apache.org
    
    http://www.apache.org/dyn/closer.cgi/hbase
    
    http://hbase.apache.org/book.html#quickstart
    
2. phoenix官方网站、下载页面和安装指导，安装指导中包括了squirrel配置
    
    http://phoenix.apache.org/index.html
    
    http://www.apache.org/dyn/closer.lua/phoenix/
    
    http://phoenix.apache.org/installation.html
