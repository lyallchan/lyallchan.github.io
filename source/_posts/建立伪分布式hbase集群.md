toc: true
title: 建立伪分布式hbase集群
date: 2017-1-17
tags: [docker, hbase, hadoop]
 
---

建立hadoop和hbase的伪分布式集群，hadoop和hbase分别运行在两个容器内，通过hadoop:9000端口访问。
 
<!--more-->

# 建立伪分布式hbase集群

# 目标

1. 建立hadoop伪分布式集群，挂载主机目录做为数据目录，暴露9000端口做为数据访问端口
2. 建立hbase伪分布式集群，使用hadoop:9000做为文件系统
3. 使用容器，并使用docker-compose简化系统启停

# 建立hadoop伪分布式集群

从[官网](http://hadoop.apache.org)下载合适版本。hbase目前版本支持hadoop2.6.1+版本，不支持2.7.0，因此下载最新的2.6.5版本

几个配置文件如下，全部参考[官方文档](http://hadoop.apache.org/docs/r2.6.5/hadoop-project-dist/hadoop-common/SingleCluster.html)

core-site.xml

```
<configuration>
	<property>
		<name>fs.defaultFS</name>
		<value>hdfs://hadoop:9000</value>
	</property>
</configuration>
```

hdfs-site.xml

```
<configuration>
	<property>
		<name>dfs.replication</name>
		<value>1</value>
	</property>
	<property>
		<name>dfs.namenode.name.dir</name>
		<value>file:/v/dfs/name</value>
	</property>
	<property>
		<name>dfs.datanode.data.dir</name>
		<value>file:/v/dfs/data</value>
	</property>
</configuration>
```

这里的`/v/dfs/`是容器映射主机的目录，在docker-compose.xml文件中会有详细定义

yarn-site.xml

```
<configuration>

<!-- Site specific YARN configuration properties -->
	<property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop</value>
    </property>
</configuration>
```

mapred-site.xml

```
<configuration>
	<property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

dockerfile: dockerfile取名hadoop.dockerfile

```
# Creates pseudo distributed hadoop 2.6.5

FROM java:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV HADOOP_VERSION=2.6.5
ADD hadoop-${HADOOP_VERSION}.tar.gz /

ENV PATH /hadoop-${HADOOP_VERSION}/bin:/hadoop-${HADOOP_VERSION}/sbin:$PATH
ENV HADOOP_PREFIX=/hadoop-${HADOOP_VERSION}

ADD core-site.xml ${HADOOP_PREFIX}/etc/hadoop/
ADD hdfs-site.xml ${HADOOP_PREFIX}/etc/hadoop/
ADD mapred-site.xml ${HADOOP_PREFIX}/etc/hadoop/
ADD yarn-site.xml ${HADOOP_PREFIX}/etc/hadoop/

RUN echo export JAVA_HOME=${JAVA_HOME} >> $HADOOP_PREFIX/etc/hadoop/hadoop-env.sh
RUN echo export HADOOP_PREFIX=/hadoop-${HADOOP_VERSION} >> $HADOOP_PREFIX/etc/hadoop/hadoop-env.sh

RUN $HADOOP_PREFIX/bin/hdfs namenode -format -force

ADD hadoop.sh /
RUN chmod +x /hadoop.sh

ENTRYPOINT ["/hadoop.sh"]
```

说明：
1. java:on是自定义的一个java镜像，基于isuper/java-oracle:jdk_8，增加ssh免密码登陆功能，并修正了时区。ssh免密码登陆是hadoop、hbase集群必备条件
2. namenode必须先格式化，这里为了格式化已有的namenode，增加了force参数
3. hadoop.sh启动hdfs和yarn服务，具体为

hadoop.sh

```
#!/bin/bash 

service ssh start
start-dfs.sh
start-yarn.sh
sleep 99999999
```

# 建立hbase伪分布式集群

从[官网](http://hbase.apache.org)下载最新版本，这里是1.2.4版本

配置文件，只有一个hbase-site.xml

```
<configuration>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://hadoop:9000/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>hdfs://hadoop:9000/zookeeper</value>
  </property>
</configuration>
```

使用hadoop:9000端口，hadoop是前面hadoop集群的主机名称，在docker-compose.yml中指定

dockerfile: 取名hbase.dockerfile

```
FROM java:on
MAINTAINER lyalchan "lyallchan@163.com"

ENV HBASE_VERSION 1.2.4

ADD hbase-${HBASE_VERSION}-bin.tar.gz /
ADD hbase-site.xml /hbase-${HBASE_VERSION}/conf/
ENV PATH /hbase-${HBASE_VERSION}/bin:$PATH
RUN echo export JAVA_HOME=$JAVA_HOME >> /hbase-$HBASE_VERSION/conf/hbase-env.sh

ADD hbase.sh /
RUN chmod +x /hbase.sh

ENTRYPOINT ["/hbase.sh"]
```

hbase.sh启动hbase服务，具体为

hbase.sh

```
#!/bin/bash 

service ssh start
start-hbase.sh
sleep 99999999
```

# 建立容器，并使用docker-compose启停

docker-compose.yml

```
version: "2"
services:
    hadoop:
        build: 
            context: .
            dockerfile: hadoop.dockerfile
        image: hadoop_pseudo_distributed:2.6.5
        container_name: hadoop
        hostname: hadoop
        ports:
            - 50070:50070
            - 8088:8088
        volumes:
            - ./data:/v
         
    hbase:
        build: 
            context: .
            dockerfile: hbase.dockerfile
        image: hbase_pseudo_distributed:1.2.4
        container_name: hbase
        depends_on: 
            - hadoop
        hostname: hbase
        ports:
            - 16010:16010
        volumes:
            - ./data:/v
```

定义两个服务，hadoop和hbase

- hadoop使用hadoop.dockerfile做为dockerfile，生成image叫hadoop_pseudo_distributed:2.6.5，启动的容器名为hadoop，主机名为hadoop，暴露两个端口50070和8088，并挂载主机当前目录下的data目录，从hdfs-site.xml可以知道，这个目录被用来当做hadoop的namenode和datanode目录。
- hbase类似

docker1.10以后的网络功能大为增强，在docker daemon中新增了dns功能，所有使用自定义网络的容器都可以通过主机名互通，不再需要使用--link。

把所有文件都放在一个目录下，共12个文件，一个data文件夹，这12个文件为

hadoop文件

- hadoop-2.5.6.tar.gz
- hadoop.dockerfile
- hadoop.sh
- core-site.xml
- hdfs-site.xml
- yarn-site.xml
- mapred-site.xml

hbase文件

- hbase-1.2.4.tar.gz
- hbase.dockerfile
- hbase.sh
- hbase-site.xml

还有一个docker-compose.yml文件

启动系统

```
docker-compose up -d
```

停止系统

```
docker-compose down
```

重建镜像

```
docker-compose build
```

或者

```
docker-compose up -d --build
```

# 使用和验证

## hbase shell

```
docker exec -it hbase /bin/bash
```

进入hbase容器，然后启动hbase shell

```
hbase shell
```

在hbase shell下面创建表`test`，并增加2行

```
create "test", "cf"
put "test", "row1", "cf:a", "value1"
put "test", "row1", "cf:b", "value2"
put "test", "row2", "cf:a", "value3"
put "test", "row2", "cf:c", "value4"
```

新增两行`row1`和`row2`，`row1`有a，b两列，`row2`有a，c两列

```
list "test"
scan "test"
get "test", "row1"
get "test", "row2"

```

## hadoop

```
docker exec -it hadoop /bin/bash
```

进入hadoop容器

```
hadoop fs -ls /hbase
```

查看hdfs的文件目录

## WEB查看

在主机的[localhost:16010](http://localhost:16010)，[localhost:50070](http://localhost:50070)，[localhost:8088](http://localhost:8088)，可以查看hbase和hadoop的运行情况

# 参考资料

1. [hbase官方文档quick start](http://hbase.apache.org/book.html#quickstart)
2. [hadoop官方文档Single Cluster](http://hadoop.apache.org/docs/r2.6.5/hadoop-project-dist/hadoop-common/SingleCluster.html)
3. [sequenceiq的hadoop docker](https://github.com/sequenceiq/hadoop-docker)
4. [基于Docker快速搭建多节点Hadoop集群](http://dockone.io/article/395)
5. [基于docker的hadoop分布式集群](http://blog.csdn.net/kongdefei5000/article/details/51426235)
