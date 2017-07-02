---
title: Storm on Yarn平台搭建
tags:
  - Storm
  - Yarn
id: 290
categories:
  - 大数据
date: 2015-11-29 14:12:24
---

## 1. 简介

1)Storm：一个实时计算框架，与Hadoop离线计算框架互补，分别用于解决不同场景下的问题。Storm的官方网站是 http://storm.apache.org，关于Storm，可以参见淘宝搜索的文章：[Storm简介](http://www.searchtb.com/2012/09/introduction-to-storm.html)。
2)YARN：YARN是Hadoop 2.0中新引入的资源管理系统，所有应用程序和框架，比如MapReduce、Storm和Spark等，均可运行在YARN之上。
3)Storm On YARN：尝试将Storm运行在YARN上，这将带来众多好处。Storm On YARN最有名是Yahoo！的开源实现。

## 2. 环境准备

|IP|Host Name|Function|
|:--|:--|:--|
|192.168.199.21|Hadoop21|namenodezookeeper,ResourceManager|
|192.168.199.22|Hadoop22|datanode,zookeeper,NodeManager|
|192.168.199.23|Hadoop23|datanode,zookeeper,NodeManager|

>hadoop2.7.1安装参见[hadoop-2-7-1-完全分布式安装部署](/2015/10/27/hadoop2-7-1/),
zookeeper安装参见[hadoop-2-7-1-实现ha集群部署](/2015/10/27/hadoop-2-7-1-ha/)中Zookeeper的安装。

## 3. Storm on Yarn安装配置

### 3.1 下载Storm on Yarn

我们用grid安装
	
	#su grid
	#mkdir storm
	#cd storm
从GitHub上下载Storm on Yarn
	
	#wget https://github.com/yahoo/storm-yarn/archive/master.zip
![2015-11-28_22-31-20](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-28_22-31-20.jpg)

	unzip master.zip
修改hadoop版本
	
	vim storm-yarn-master/pom.xml
修改Hadoop的版本号为2.7.1

### 3.2 编译Storm on Yarn

下载maven并解压
	
	#wget http://mirrors.aliyun.com/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
	#cd storm/storm-yarn-master
使用maven编译
	
	#/home/grid/apache-maven-3.3.9/bin/mvn package -DskipTests
![2015-11-29_8-44-16](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_8-44-16.jpg)

解压storm.zip至/home/grid/storm下
	
	#unzip /home/grid/storm/storm-yarn-master/lib/storm.zip /home/grid/storm

### 3.3 配置Storm on Yarn

	#vim ./bashrc
![2015-11-29_13-41-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_13-41-05.jpg)
	
	#source ./bashrc
将编译好后的storm-yarn-master/lib/storm.zip 添加进hdfs
	
	hadoop fs -mkdir -p /lib/storm/0.9.0-wip21
	hadoop fs -moveFromLocal storm.zip /lib/storm/0.9.0-wip21

## 4. Storm on Yarn启动测试

### 4.1 Storm on Yarn启动

启动hadoop
	
	#start-dfs.sh
	#start-yarn.sh
启动zookeeper
	
	#~/zookeeper-3.4.6/bin/zkServer.sh start
启动storm on yarn
	
	#storm-yarn launch $STORM_HOME/storm-0.9.0-wip21/conf/storm.yaml
访问http://192.168.199.21:8088/cluster，我们看到storm on yarn已启动。
注：因为storm是作为一个yarn程序运行在集群上的，所以会有一个AppId，如下图所示
![2015-11-29_11-34-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_11-34-13.jpg)

通过以下命令我们可以获取Nimbus host
	
	#storm-yarn getStormConfig -appId application_1448767639808_0001 -output ~/storm/storm.yaml
	#cat ~/storm/storm.yaml | grep nimbus.host
访问Nimbus host http://192.168.199.22:7070/可以看到Storm UI

![2015-11-29_12-49-24](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_12-49-24.jpg)

### 4.2 Storm on Yarn测试

1)提交Topology
	
	storm jar /home/grid/storm/storm-yarn-master/lib/storm-starter-0.0.1-SNAPSHOT.jar storm.starter.WordCountTopology wordCountTopology -c nimbus.host=192.168.199.22

![2015-11-29_12-56-08](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_12-56-08.jpg)

2)监控Topology

![2015-11-29_12-56-24](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_12-56-24.jpg)

3)关闭Topology
	
	storm kill wordCountTopology
![2015-11-29_14-06-03](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_14-06-03.jpg)

4)关闭Storm on yarn
	
	storm-yarn shutdown –appId [applicationId]
![2015-11-29_14-11-51](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-29_14-11-51.jpg)