---
title: 使用Sqoop实现Mysql与HDFS及HIVE互转
tags:
  - hadoop
  - Hive
  - Mysql
  - sqoop
id: 139
categories:
  - 大数据
date: 2015-11-14 13:22:15
---

## 1.简介

Sqoop(发音：skup)是一款开源的工具，主要用于在Hadoop(Hive)与传统的数据库(mysql、postgresql...)间进行数据的传递，可以将一个关系型数据库（例如 ： MySQL ,Oracle ,Postgres等）中的数据导进到Hadoop的HDFS中，也可以将HDFS的数据导进到关系型数据库中。
访问[Sqoop官方网站](http://sqoop.apache.org/)获得更多信息。

## 2.环境

Hadoop 1.2.1 ,Hive 1.2.1,Mysql 5.1.73,配置见下表，Hadoop/Hive/Mysql的安装见其他文章，这里略。

|IP|Name|Function|
|:--|:--|:--|
|192.168.199.11|Hadoop11|Namenode,Hive,Sqoop|
|192.168.199.12|Hadoop12|Datanode|
|192.168.199.13|Hadoop13|Datanode,Mysql Serve|

## 3.安装及启动

### 3.1下载

在Hadoop11上从镜像地址下载Sqoop,并解压
http://mirrors.aliyun.com/apache/sqoop/1.4.6/
![2015-11-14_9-04-40](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_9-04-40.jpg)

	#wget http://mirrors.aliyun.com/apache/sqoop/1.4.6/sqoop-1.4.6.bin__hadoop-1.0.0.tar.gz
	#tar -xzvf sqoop-1.4.6.bin__hadoop-1.0.0.tar.gz

![2015-11-14_10-24-25](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_10-24-25.jpg)

	#mv sqoop-1.4.6.bin__hadoop-1.0.0 sqoop-1.4.6

在Hadoop11上下载mysql java connector
	
	#wget http://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.37.tar.gz
	#tar –xzvf mysql-connector-java-5.1.12.tar.gz

复制mysql connector至sqoop lib目录下
	
	#cp mysql-connector-java-5.1.12-bin.jar /home/grid/sqoop-1.4.6/lib

### 3.2配置

	#vim /etc/profile
		export SQOOP_HOME=/home/hadoop/sqoop-1.4.6
		export PATH=$SQOOP_HOME/bin:$PATH
	#source /etc/profile
	#cd conf
	#cp sqoop-env-template.sh sqoop-env.sh
	#vim sqoop-env.sh

### 3.3启动

在hadoop11上启动hadoop
	
	#start-all.sh

在hadoop13上启动mysql
		
	#service mysqld start
	#mysql

创建演示数据库
	
	mysql>create database sqoop;
	mysql>use sqoop;
创建表并导入sample数据

	mysql>create table tab1 as select table_schema,table_name,table_type from information_schema.TABLES;
	mysql>show tables;
创建空白表用于从hdfs导入数据
	
	mysql>create table tab2 as select * from tab1 limit 0;
	mysql>select * from tab2;

![2015-11-14_12-33-48](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_12-33-48.jpg)

同样创建空白表用于从hive导入数据
	
	mysql>create table tab3 as select * from tab1 limit 0;
	mysql>select * from tab3;

## 4.演示数据互转

### 4.1 mysql与HDFS互转

	#sqoop list-databases --connect jdbc:mysql://hadoop13:3306/ --username hive --password hive@Hadoop
	#sqoop list-tables --connect jdbc:mysql://hadoop13:3306/sqoop --username hive --password hive@Hadoop

导入tab1数据至HDFS
	
	#sqoop import --connect jdbc:mysql://hadoop13:3306/sqoop --username hive --password hive@Hadoop --table tab1 -m 1
	#hadoop fs -ls /user/grid/tab1
	#hadoop fs -cat /user/grid/tab1/part-m-00000
	
![2015-11-14_12-26-56](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_12-26-56.jpg)

导出HDFS数据至mysql tab2
	
	#sqoop export --connect jdbc:mysql://hadoop13:3306/sqoop --table tab2 --username hive --password hive@Hadoop --export-dir hdfs://hadoop11:9000/user/grid/tab1/part-m-00000 -m 1

我们看到mysql tab2表已经有数据了
![2015-11-14_12-37-25](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_12-37-25.jpg)

### 4.2 mysql与hive互转

启动hive
	
	#hive --service metastore	
	#hive --service hiveserver2

导入mysql tab2数据至hive
	
	#sqoop import --connect jdbc:mysql://hadoop13:3306/sqoop --username hive --password hive@Hadoop --table tab2 --hive-table tab2 --hive-import -m 1

进入hive我们看到hive里多了tab2表
![2015-11-14_12-51-15](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_12-51-15.jpg)

进入warehouse我们看到多了一个目录
![2015-11-14_12-55-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_12-55-59.jpg)

导出hive数据至mysql tab3

	#sqoop export --connect jdbc:mysql://hadoop13:3306/sqoop --table tab3 --username hive --password hive@Hadoop --export-dir /user/hive/warehouse/tab2/part-m-00000 --input-fields-terminated-by '\001'

到mysql可以看到tab3表有数据了.
![2015-11-14_13-11-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-11-14_13-11-11.jpg)

至此演示数据结束。