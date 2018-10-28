---
title: Hbase完全分布式安装
tags:
  - Hbase
id: 103
categories:
  - 大数据
date: 2015-11-06 11:17:16
---
## 1. 环境准备
环境：CENTOS6.6, [Hadoop 1.2.1](/2015/10/26/hadoop1-2-1/), zookeeper 3.4.6, jdk1.8.0_51

|IP|Host Name|Function|
|:--|:--|:--|
|192.168.199.11|Hadoop11|namenode|
|192.168.199.12|Hadoop12|datanode|
|192.168.199.13|Hadoop13|datanode|

## 2. 安装配置Habse

### 2.1	下载HBase
从镜像站点下载HBase，Hadoop 1.2.1目前对应的Hbase版本是0.98.15，如图。
![2015-10-30_20-26-35](/uploads/2015/11/2015-10-30_20-26-35.png)

### 2.2	修改/etc/profile

	export HBASE_HOME=/home/grid/hbase-0.98.15-hadoop1
	export PATH=$PATH:$HBASE_HOME/bin

profile生效
	
	#source /etc/profile

复制/etc/profile到hadoop12和hadoop13，并在hadoop12、hadoop13上执行

	#source /etc/profile

### 2.3 配置hbase-env.sh

修改文件`$HBASE_HOME/conf/hbase-env.sh`

	export JAVA_HOME=/usr/java/jdk1.8.0_51

Zookeepr我们用自己安装的，所以改为false

	export HBASE_MANAGES_ZK=false

### 2.4 配置hbase-site.xml

修改文件`$HBASE_HOME/conf/hbase-site.xml`

	<configuration>
		<property>
			<name>hbase.rootdir</name>
			<value>hdfs://hadoop11:9000/hbase</value>
		</property>
		<property>
			<name>hbase.cluster.distributed</name>
			<value>true</value>
		</property>
		<property>
			<name>hbase.zookeeper.quorum</name>
			<value>hadoop11,hadoop12,hadoop13</value>
		</property>
	</configuration>

### 2.5 修改regionservers

![2015-10-30_22-15-02](/uploads/2015/11/2015-10-30_22-15-02.png)

### 2.6 分发Hbase
复制hbase至其他两节点

	#scp -r hbase-0.98.15-hadoop1 grid@hadoop12:/home/grid
	#scp -r hbase-0.98.15-hadoop1 grid@hadoop13:/home/grid
至此HBase集群安装配置完成

## 3. 启动
### 3.1 启动hadoop

首先在hadoop11上启动hadoop

	#start-all.sh

![2015-10-30_22-23-54](/uploads/2015/11/2015-10-30_22-23-54.png)

### 3.2 启动Zookeeper
然后启动zookeeper，分别在三台机器上执行。

	#~/zookeeper-3.4.6/bin/zkServer.sh start

![2015-10-30_22-23-33](/uploads/2015/11/2015-10-30_22-23-33.png)

### 3.3 启动HBase
最后在hadoop11上启动hbase集群

	#start-hbase.sh

![2015-10-30_21-55-48](/uploads/2015/11/2015-10-30_21-55-48.png)

至此Hbase集群启动成功。

## 4. 验证测试hbase

### 4.1 检查jps
运行jps命令，我们看到hbase服务起来了。

![2015-10-30_21-56-49](/uploads/2015/11/2015-10-30_21-56-49.png)

### 4.2 Hbase shell操作

![2015-10-30_22-10-03](/uploads/2015/11/2015-10-30_22-10-03.png)

![2015-10-30_22-10-20](/uploads/2015/11/2015-10-30_22-10-20.png)

### 4.3 web监控

![2015-10-30_22-31-27](/uploads/2015/11/2015-10-30_22-31-27.png)
