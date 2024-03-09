toc: true
title: spark on mesos in docker
date: 2017-3-17 19:19:49
tags: [spark, mesos, python, zookeeper, jupyter, conda]

---

搭建mesos集群，使用zookeeper管理mesos master，防止单点失效。在mesos集群上运行spark，并搭建jupyter，做为spark client。

<!--more-->

# 搭建zookeeper\mesos集群

3个zookeeper，3个mesos master

## zookeeper dockerfile

```
FROM openjdk8:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV ZOOKEEPER="zookeeper-3.4.9"
RUN busybox wget http://caddy/$ZOOKEEPER.tar.gz -O - | tar zx && ln -s /$ZOOKEEPER /zookeeper
RUN mv /zookeeper/conf/zoo_sample.cfg /zookeeper/conf/zoo.cfg
ENV PATH /zookeeper/bin:$PATH
```

很简单，就是下载解压，默认的配置文件，zookeeper/bin加入路径

## mesos dockerfile

```
FROM openjdk8:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV MESOS="mesos-bin-1.1.0"
RUN busybox wget http://caddy/$MESOS.tar.gz -O - | tar xz && ln -s /$MESOS /mesos 
RUN apt-get update && apt-get install -y \
        libsvn1 \
        libcurl3-nss
ENV PATH /mesos/bin:/mesos/sbin:$PATH

ENV CONDA="Miniconda2-latest-Linux-x86_64.sh"
RUN busybox wget http://caddy/$CONDA -P /
RUN bash /$CONDA -b -p conda
RUN rm /$CONDA
ENV PATH /conda/bin:$PATH
ENV PYTHONPATH=$PYTHONPATH:/mesos/lib/python2.7/site-packages/
RUN conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/ \
    && conda config --set show_channel_urls yes
```

mesos部分也比较简单，下载解压，mesos/bin和mesos/sbin加入路径。

安装了libsvn1和libcurl3-nss包，这是在启动mesos的时候提示缺少文件，再加上去的。

mesos WEB UI需要python运行，这里没有直接安装python，而是使用了conda。

conda是偏重于科学计算的python套件，其中的包管理工具conda比pip好用。

conda的安装包直接是个脚本文件，bash这个文件就可以安装，加上-b参数可以接受默认配置，自动安装

最后的conda config是换国内源

## 启动zookeeper集群

zookeeper start.sh

```
#!/bin/sh

NETWORK="net"
CONTAINER="zoo"
NUM=3 # hosts in cluster 

# start zookeeper containers...
echo 
echo starting zookeeper containers...
for i in `seq $NUM`; do
    docker run \
        -d \
        --hostname ${CONTAINER}$i \
        --name ${CONTAINER}$i \
        --net $NETWORK \
        zookeeper \
        /bin/bash -c "/etc/init.d/ssh start && sleep 99999999"
done

# start zookeeper service
echo
echo starting zookeeper service
ZOOKEEPER_DATADIR="/zookeeper/data"
for i in `seq 1 $NUM`; do

    #/zoo.cfg specially for server:1=xxxxx
    for j in `seq $NUM`; do
        docker exec ${CONTAINER}$i /bin/bash -c "echo server.$j=${CONTAINER}$j:2888:3888 >> /zookeeper/conf/zoo.cfg"
    done
    docker exec ${CONTAINER}$i /bin/bash -c "echo dataDir=$ZOOKEEPER_DATADIR >> /zookeeper/conf/zoo.cfg"
    docker exec ${CONTAINER}$i /bin/bash -c "mkdir -p $ZOOKEEPER_DATADIR"
    docker exec ${CONTAINER}$i /bin/bash -c "echo $i > $ZOOKEEPER_DATADIR/myid"

    docker exec ${CONTAINER}$i /zookeeper/bin/zkServer.sh start
done

echo done!

```

由于启动zookeeper服务器的数量需要动态调整，不必重新生成image，因此采用先启动容器，再动态生成配置文件，启动服务的方案。

NUM是zookeeper服务器的数量，官方推荐奇数，最少3台服务器，这里就定义为3

第二段是启动容器，并同时启动了ssh服务

第三段是动态生成zoo.cfg和myid，并启动zookeeper服务。最后生成的zoo.cfg，除去默认的部分为

```
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
dataDir=/zookeeper/data
```

myid根据服务器顺序分别为1,2,3

zkServer.sh start启动服务。

## 启动mesos master集群

mesos start-master.sh

```
#!/bin/sh

CONTAINER=mesos
NET=net
MASTER=zk://zoo1:2181,zoo3:2181,zoo3:2181/mesos

for i in `seq 3`; do
    docker run \
        -itd \
        --hostname ${CONTAINER}$i \
        --name ${CONTAINER}$i \
        -p 505$i:505$i \
        --network $NET \
        -e MESOS_QUORUM=2 \
        -e MESOS_ZK=$MASTER \
        -e MESOS_WORK_DIR="/" \
        -e MESOS_HOSTNAME=localhost \
        -e MESOS_PORT=505$i \
        mesos \
        mesos-master
done

```

mesos的配置可以有两种方式传递

1. 命令行参数
2. 环境变量

docker环境下，传递环境变量比较方便

MASTER指向zookeeper集群，并在容器启动时，传递给MESOS_ZK

按照顺序分别启动mesos-master容器

+ 暴露5051,5052,5052端口
+ 定义HOSTNAME，这个参数是mesos WEB UI定时刷新页面的时候访问的地址，由于是docker环境，端口暴露后的主机为localhost。
+ 其它参数比较好理解

这样，启动了mesos master集群，并使用zookeeper做HA，防止单点失效。

# 在mesos上运行spark

## spark dockerfile

```
FROM mesos
MAINTAINER lyalchan "lyallchan@163.com"

ENV SPARK="spark-2.1.0-bin-hadoop2.6"
RUN busybox wget -O - http://caddy/$SPARK.tgz | tar zx \
    && ln -s /$SPARK /spark
ENV PATH /spark/sbin:/spark/bin:$PATH
ENV SPARK_HOME=/spark
RUN echo "export MESOS_NATIVE_JAVA_LIBRARY=/mesos/lib/libmesos.so" >> /spark/conf/spark-env.sh
ENV PYSPARK_PYTHON=/conda/bin/python
RUN conda install numpy -y
```

spark在mesos上运行，必须要有mesos环境，因此在mesos镜像基础上建立spark镜像。

官网上说，除了将spark压缩包放在mesos可以访问的地方，也可以在所有mesos slave的相同目录安装spark，配置spark.mesos.excutor.home指向$SPARK_HOME

> Alternatively, you can also install Spark in the same location in all the Mesos slaves, and configure spark.mesos.executor.home (defaults to SPARK_HOME) to point to that location.

安装过程很常规，下载解压。

在spark-env.sh中设置MESOS_NATIVE_JAVA_LIBRARY为libmesos.so

设置PYSPARK_PYTHON为conda的python，使得spark可以找到python

最后装上numpy包

## 启动spark

spark start-mesos-slave.sh
```
#!/bin/sh

DIR=`dirname $0`
DATA=`cygpath -aw $DIR/data`
NET="net"
CONTAINER="spark"
MASTER=zk://zoo1:2181,zoo2:2181,zoo3:2181/mesos
S_NUM=3

# start spark slave containers with mesos
for i in `seq 1 $S_NUM`;do 
    docker run \
        -itd \
        --hostname ${CONTAINER}_s$i \
        --name ${CONTAINER}_s$i \
        --network $NET \
        --privileged \
        --volume "$DATA":/v \
        -e MESOS_MASTER=$MASTER \
        -e MESOS_WORK_DIR="/" \
        -e MESOS_SYSTEMD_ENABLE_SUPPORT=false \
        spark \
        mesos-slave
done
```

与其说启动spark，不如说是启动mesos slave。当客户端提交spark应用后，mesos slave再启动spark。

mesos slave使用容器技术（自带的或者docker）启动framework，framework是运行在mesos上的应用程序。因此启动mesos slave容器比较麻烦，需要特权启动，如果使用docker，还需要绑定宿主机的docker.sock和docker。

S_NUM是启动的spark节点数量，最小1个，这里取3个。

# 搭建jupyter

搭建jupyter做为spark集群客户端。

## jupyter dockerfile

jupyter做为spark集群客户端需要有mesos、spark以及python环境。

上面的spark镜像也有这三个环境，不过其中python是mini conda。

为了使用jupyter，有两个选择，可以在spark镜像基础上再安装jupyter，也可以从头安装mesos和spark，并安装anaconda，anaconda是更加强大的python科学计算包，其中包括了jupyter。

```
# Creates anaconda with spark 

FROM openjdk8:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV MESOS="mesos-bin-1.1.0"
RUN busybox wget http://caddy/$MESOS.tar.gz -O - | tar xz && ln -s /$MESOS /mesos 
RUN apt-get update && apt-get install -y \
        libsvn1 \
        libcurl3-nss
ENV PATH /mesos/bin:/mesos/sbin:$PATH

ENV SPARK="spark-2.1.0-bin-hadoop2.6"
RUN busybox wget -O - http://caddy/$SPARK.tgz | tar zx \
    && ln -s /$SPARK /spark
ENV PATH /spark/sbin:/spark/bin:$PATH
ENV SPARK_HOME=/spark
RUN echo "export MESOS_NATIVE_JAVA_LIBRARY=/mesos/lib/libmesos.so" >> /spark/conf/spark-env.sh

ENV CONDA=Anaconda2-4.3.0-Linux-x86_64.sh
RUN busybox wget http://caddy/$CONDA -P /
RUN bash /$CONDA -b -p conda
RUN rm /$CONDA
ENV PATH /conda/bin:$PATH
ENV PYSPARK_DRIVER_PYTHON jupyter
ENV PYSPARK_DRIVER_PYTHON_OPTS "notebook --ip='*' --NotebookApp.password='' --NotebookApp.token='' --no-browser"
ENV PYSPARK_PYTHON=/conda/bin/python
RUN conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/ \
    && conda config --set show_channel_urls yes

```

mesos\spark常规安装，spark安装好以后，在spark-env.sh设置变量MESOS_NATIVE_JAVA_LIBRARY。

anaconda下载安装好以后，要设置2个环境变量，使得pyspark启动时，调用jupyter notebook，这在pyspark脚本中有详细说明：

> In Spark 2.0, IPYTHON and IPYTHON_OPTS are removed and pyspark fails to launchis set in the user's environment. Instead, users should set PYSPARK_DRIVER_PYTHON to use IPython and set PYSPARK_DRIVER_PYTHON_OPTS to pass options when starting (e.g. PYSPARK_DRIVER_PYTHON_OPTS='notebook').  This supports full customization and executor Python executables.

1. ENV PYSPARK_DRIVER_PYTHON jupyter
2. ENV PYSPARK_DRIVER_PYTHON_OPTS "notebook --ip='*' --NotebookApp.password='' --NotebookApp.token='' --no-browser"

还要设置PYSPARK_PYTHON指向/conda/bin/python，使得spark可以找到python。

## 启动jupyter

jupyter start.sh

```
#!/bin/bash

V=`cygpath -aw $(dirname $0)/v`
CONTAINER="conda"
NETWORK="net"
MASTER=zk://zoo1:2181,zoo2:2181,zoo3:2181/mesos

docker run \
    -itd \
    --hostname ${CONTAINER} \
    --name ${CONTAINER} \
    --network $NETWORK \
    -v "$V":/v:rw \
    -w /v \
    -p 127.0.0.1:8888:8888\
    -p 4040:4040 \
    anaconda \
    /bin/sh -c "pyspark --master mesos://$MASTER"
```

# 测试

系统全部跑起来，需要启动zookeeper、mesos master、mesos slave with spark和jupyter

暴露的端口有

```
5051:5052:5053  3个mesos master服务
8888            jupyter notebook
4040            spark jobs
```

## localhost:5051

访问localhost:5051可以看到

+ 选举出来的server是5051，是第一个mesos容器mesos1
+ 有3个agents，分别是spark的3个容器
+ 有1个framwork，是conda容器的PySparkShell，也就是跑的juypter notebook

停止mesos1，2-3min后，可以看到选举的server变成5052

## localhost:8888

访问localhost:8888，是jupyter notebook界面，可以在其中新建一个notebook，并运行一个spark案例

```
import numpy as np
from pyspark.mllib.stat import Statistics
m = sc.parallelize([np.array([1.0, 20.0, 300.0]), np.array([3.0, 10.0, 200.0]), np.array([2.0, 30.0, 100.0])],2)  
summary = Statistics.colStats(m)

print(summary.mean())  # a dense vector containing the mean value for each column
print(summary.variance())  # column-wise variance
print(summary.numNonzeros())  # number of nonzeros in each column
```

如果有结果，说明安装成功。

## localhost:4040

访问localhost:4040，可以看到刚才跑的案例的分布式计算的过程。
