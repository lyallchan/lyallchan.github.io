toc: true
title: SPARK standalone HA with Zookeeper in docker
date: 2017-3-2
tags: [spark, zookeeper, docker]

---

安装spark集群，3master，2slaves，master通过zookeeper管理，5个机器全部容器化。

<!--more-->

关于slave还是worker：官方文档叫做worker，但是配置文件却是slave，所以还是用slaves，和配置文件统一

# 制作镜像

## dockerfile

```
# Creates standalone spark cluster

FROM java:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV SPARK="spark-2.1.0-bin-hadoop2.6"
ADD http://127.0.0.1:60060/$SPARK.tgz /
RUN tar -xf /$SPARK.tgz \
        && ln -s /$SPARK /spark \
        && rm /$SPARK.tgz
ENV PATH /spark/sbin:/spark/bin:$PATH

ENV ZOOKEEPER="zookeeper-3.4.9"
ADD http://127.0.0.1:60060/$ZOOKEEPER.tar.gz /
RUN tar -xf /$ZOOKEEPER.tar.gz \
        && ln -s /$ZOOKEEPER /zookeeper \
        && rm /$ZOOKEEPER.tar.gz
RUN mv /zookeeper/conf/zoo_sample.cfg /zookeeper/conf/zoo.cfg
ENV PATH /zookeeper/bin:$PATH

ENV CONDA="Miniconda2-latest-Linux-x86_64.sh"
ADD http://127.0.0.1:60060/$CONDA /
RUN bash /$CONDA -b -p conda
RUN rm /$CONDA
ENV PATH /conda/bin:$PATH
ENV PYSPARK_PYTHON=/conda/bin/python
RUN conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/ \
    && conda config --set show_channel_urls yes
RUN conda install numpy -y

```

`java:on`是自定义容器，安装了`oracle java8`，增加了免密码`ssh`登陆功能

第1段：下载[`spark`](http://spark.apache.org/downloads.html)包，解压缩到根目录，设置PATH变量

第2段：下载[`zookeeper`](http://zookeeper.apache.org/releases.html)包，解压缩到根目录，生成配置文件，设置PATH变量

第3段：下载安装[`conda`](https://conda.io/miniconda.html)，设置PATH变量。设置PYSPARK_PYTHON变量，使得spark能找到python的运行环境。换源并安装numpy包，也可以安装其它需要的包，如pandas，ipython等

为了加快下载速度，在本机上运行了一个文件服务器，从文件服务器上下载所需要的安装包。

文件服务器采用[`caddy`](https://caddyserver.com/docs/cli)，支持http2的web server，也是一个神器，golang编写，单文件运行，一行命令搞定文件服务器。

```
caddy -port 60060 -log stdout browse
```

## 制作镜像

```
docker build --force-rm -t spark . 
```

总结一下，镜像具有以下环境：
1. spark，默认配置文件，在运行时候修改
2. zookeeper，拷贝样例配置文件，其中的配置信息在运行时候修改
3. conda，换源以及安装所需要的包
4. 配置PYSPARK_PYTHON，使得spark能找到python运行环境

# 运行

启动文件`start.sh`

```
#!/bin/sh

DIR=`dirname $0`
DATA=`cygpath -aw $DIR/data`
NETWORK="spark"
CONTAINER="spark"
S_NUM=2 # slaves number 
M_NUM=3 # masters number 

. $DIR/stop.sh  # stop first

if ! docker network ls | grep $NETWORK >>/dev/null; then
    docker network create "$NETWORK"
fi

# start master containers...
echo 
echo starting master containers...
for i in `seq $M_NUM`; do
    docker run \
        --volume "$DATA":/v \
        -p 808$i:8080 \
        -d \
        --hostname ${CONTAINER}$i \
        --name ${CONTAINER}$i \
        --net $NETWORK \
        spark \
        /bin/bash -c "/etc/init.d/ssh start && sleep 99999999"
done

# start spark slave containers...
echo 
echo starting slave containers...
for i in `seq 1 $S_NUM`;do 

    # start si
    docker run \
        --volume "$DATA":/v \
        -p 809$i:8081 \
        -d \
        --hostname ${CONTAINER}_s$i\
        --name ${CONTAINER}_s$i \
        --net $NETWORK \
        spark \
        /bin/bash -c "/etc/init.d/ssh start && sleep 99999999"
done

# start zookeeper in masters
echo
echo starting zookeeper in masters
ZOOKEEPER_DATADIR="/zookeeper/data"
for i in `seq 1 $M_NUM`; do

    #/zoo.cfg specially for server:1=xxxxx
    for j in `seq $M_NUM`; do
        docker exec ${CONTAINER}$i /bin/bash -c "echo server.$j=${CONTAINER}$j:2888:3888 >> /zookeeper/conf/zoo.cfg"
    done
    docker exec ${CONTAINER}$i /bin/bash -c "echo dataDir=$ZOOKEEPER_DATADIR >> /zookeeper/conf/zoo.cfg"
    docker exec ${CONTAINER}$i /bin/bash -c "mkdir -p $ZOOKEEPER_DATADIR"
    docker exec ${CONTAINER}$i /bin/bash -c "echo $i > $ZOOKEEPER_DATADIR/myid"

    docker exec ${CONTAINER}$i /zookeeper/bin/zkServer.sh start
done

# start masters
echo 
echo starting masters...
# spark.depoly.zookeeper.url in /spark/conf/spark-env.sh
ZOOKEEPER_URL=${CONTAINER}1:2181
for i in `seq 2 $M_NUM`; do
    ZOOKEEPER_URL=$ZOOKEEPER_URL",${CONTAINER}$i:2181"
done
SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER"
SPARK_DAEMON_JAVA_OPTS="$SPARK_DAEMON_JAVA_OPTS -Dspark.deploy.zookeeper.url=$ZOOKEEPER_URL"

for i in `seq 1 $M_NUM`; do
    # /spark/conf/slaves
    for j in `seq 1 $S_NUM`; do
        docker exec ${CONTAINER}$i /bin/bash -c "echo ${CONTAINER}_s$j >> /spark/conf/slaves"
    done
    # spark-env.sh with SPARK_DAEMON_JAVA_OPTS
    docker exec ${CONTAINER}$i /bin/bash -c "echo SPARK_DAEMON_JAVA_OPTS='\"$SPARK_DAEMON_JAVA_OPTS\" '>> /spark/conf/spark-env.sh"

    docker exec ${CONTAINER}$i /spark/sbin/start-master.sh
done

# start slaves
echo
echo starting slaves...
# SPARK_SLAVES_SC for start-slave.sh parameter
SPARK_SLAVES_SC="spark://${CONTAINER}1:7077"
for i in `seq 2 $M_NUM`; do
    SPARK_SLAVES_SC=${SPARK_SLAVES_SC}",${CONTAINER}$i:7077"
done
for i in `seq 1 $S_NUM`; do
    docker exec ${CONTAINER}_s$i /spark/sbin/start-slave.sh $SPARK_SLAVES_SC
done

echo
echo done!

```

第1,2,3段准备变量，包括容器映射目录，容器网络，容器名称，slave数量2个，master数量3个。停止已经运行的容器，创建容器网络。

> docker新版本的容器网络功能强大，在同一个网络下的各个容器可以通过hostname互通，不再需要link

第4、5段分别启动`master`、`slave`容器，现阶段只启动`sshd`服务，各主机名和暴露端口如下


主机名 | 端口 | 作用
---|---|---
spark1 | 8081 | master
spark2 | 8082 | master
spark3 | 8083 | master
spark_s1 | 8091 | slave
spark_s2 | 8092 | slave

## 启动zookeeper

第6段，在3台`master`中启动`zookeeper`服务

`conf/zoo.cfg`增加配置

```
server.1=spark1:2888:3888
server.2=spark2:2888:3888
server.3=spark3:2888:3888
dataDir=/zookeeper/data
```

在`dataDir`下增加`myid`文件，即增加`/zookeeper/data/myid`文件，3个容器内容不同，分别为1,2,3，标识`zookeeper`选举顺序

`zkServer.sh start`启动服务

## 启动spark master服务

第7段，在3台`master`中启动`spark master`服务

增加`conf/slaves`文件，指定各`slave`主机名

```
spark_s1
spark_s2
```

增加`conf/spark-env.sh`，设置`spark-master`启动参数，参数指定HA模式为`zookeeper`模式，并指向`zookeeper`服务参数

**这部分内容在官方文档中没有详细描述**，可以参考[网上资料](http://blog.csdn.net/anzhsoft/article/details/33740737)

```
SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=spark1:2181,spark2:2181,spark3:2181"
```

`start-master.sh`启动服务

## 启动spark slave服务

通过命令行参数指定`spark master`集群

```
/spark/sbin/start-slave.sh spark://spark1:7077,spark2:7077,spark3:7077
```

# 参考资料

1. zookeeper模式下，spark master启动参数设置

    http://www.dataguru.cn/thread-333245-1-1.html
    http://blog.csdn.net/anzhsoft/article/details/33740737

2. spark官方文档有关HA配置

    http://spark.apache.org/docs/latest/spark-standalone.html

