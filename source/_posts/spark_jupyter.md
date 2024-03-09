toc: true
title: 使用jupyter notebook做为spark客户端
date: 2017-3-3
tags: [spark, jupyter, anaconda]

---

安装好[spark集群](http://lyallchan.github.io/2017/03/02/spark_zookeeper/)，使用jupyter notebook做为spark的客户端，或者官方叫做`driver node`

<!--more-->

# 镜像

dockerfile

```
# Creates anaconda with spark 

FROM java:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV SPARK="spark-2.1.0-bin-hadoop2.6"
ADD http://127.0.0.1:60060/$SPARK.tgz /
RUN tar -xf /$SPARK.tgz \
        && ln -s /$SPARK /spark \
        && rm /$SPARK.tgz
ENV PATH /spark/sbin:/spark/bin:$PATH

ENV ANACONDA=Anaconda2-4.3.0-Linux-x86_64.sh
ADD http://127.0.0.1:60060/$ANACONDA /
RUN bash /$ANACONDA -b -p anaconda2
RUN rm /$ANACONDA
ENV PATH /anaconda2/bin:$PATH
RUN conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/ \
    && conda config --set show_channel_urls yes

ENV PYSPARK_DRIVER_PYTHON=jupyter
ENV PYSPARK_DRIVER_PYTHON_OPTS="notebook --ip=* --NotebookApp.password='' --NotebookApp.token='' --no-browser"

```

1. 和[spark镜像](http://lyallchan.github.io/2017/03/02/spark_zookeeper/)相比，这个镜像没有安装zookeeper，使用了功能更加全面的[anaconda](https://docs.continuum.io/anaconda/install)，其中包括了ipython，jupyter notebook
2. 首先安装spark，python需要spark环境启动sc(SparkContext)
3. 然后安装anaconda，并换国内源，加快下载速度
4. 最后设置环境变量，使用pyspark启动jupyter notebook。pyspark在spark的/bin目录下，用来启动spark集群下的python环境，是个shell脚本，其中有这两个变量环境设置的详细说明
> In Spark 2.0, IPYTHON and IPYTHONOPTS are removed and pyspark fails to launchis set in the user's environment. Instead, users should set PYSPARKDRIVERPYTHON to use IPython and set PYSPARK_DRIVER_PYTHON_OPTS to pass options when starting (e.g. PYSPARK_DRIVER_PYTHON_OPTS='notebook').  This supports full customization and executor Python executables.

5. jupyter notebook的参数为绑定任意IP地址，不需要密码，不需要token，不启动浏览器，详细说明可以参加[官网说明](http://nbviewer.jupyter.org/github/ipython/ipython/blob/3.x/examples/Notebook/Configuring%20the%20Notebook%20and%20Server.ipynb)

# 启动

start.sh

```
#!/bin/bash

CONTAINER="conda"
NETWORK="spark"

if docker ps -a | grep $CONTAINER > /dev/null; then
    docker rm -f $CONTAINER > /dev/null
fi

V=`cygpath -aw $(dirname $0)/v`

docker run \
    -itd \
    --hostname $CONTAINER \
    --name $CONTAINER \
    -v "$V":/v:rw \
    -w /v \
    -p 127.0.0.1:8888:8888\
    --net $NETWORK \
    anaconda \
    /bin/sh -c "pyspark --master spark://spark1:7077,spark2:7077,spark3:7077"

```

1. 使用spark容器的network，以找到spark集群
2. 最后一行容器的启动命令一定要使用`/bin/sh -c "pyspark ****"`的形式，如果直接启动pyspark，会造成juypter notebook一直重启，使用/bin/bash也会有同样现象，很奇怪。详情可以参考[github的讨论](https://github.com/ipython/ipython/issues/8367)
3. pyspark指向的master为先前启动的spark standalone集群，url和启动spark slave服务时的url一致。

# 使用

容器启动完成后，打开本机浏览器，指向localhost:8888就可以访问jupyter notebook

+ 在这个环境中已经启动好了sc，可以通过sc使用spark
+ 使用`docker logs -f conda`查看jupyter notebook输出

跑一个官网例子

```
import numpy as np
from pyspark.mllib.stat import Statistics

mat = sc.parallelize(
    [np.array([1.0, 20.0, 300.0]), np.array([3.0, 10.0, 200.0]), np.array([2.0, 30.0, 100.0])],10
)  # an RDD of Vectors

# Compute column summary statistics.
summary = Statistics.colStats(mat)
print(summary.mean())  # a dense vector containing the mean value for each column
print(summary.variance())  # column-wise variance
print(summary.numNonzeros())  # number of nonzeros in each column
```
