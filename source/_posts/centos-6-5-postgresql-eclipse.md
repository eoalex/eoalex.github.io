---
title: Centos 6.5 Postgresql搭建eclipse开发环境
tags:
  - postgresql
id: 249
categories:
  - 数据库
date: 2015-11-28 12:42:12
---

环境Centos 6.5 GNOME 2.28.2

## 1.安装git

	#mkdir /home/src
	#cd src
	#yum install git
![2015-11-23_22-08-45](/uploads/2015/11/2015-11-23_22-08-45.jpg)

![2015-11-23_22-08-13](/uploads/2015/11/2015-11-23_22-08-13.jpg)

## 2.eclipse搭建

	#yum install eclipse
![2015-11-26_20-48-17](/uploads/2015/11/2015-11-26_20-48-17.jpg)
	
	#yum install eclipse-cdt
[![2015-11-26_20-50-09](/uploads/2015/11/2015-11-26_20-50-09.jpg)
	
	#yum install readline-devel
[![2015-11-26_20-54-28](/uploads/2015/11/2015-11-26_20-54-28.jpg)

## 3.安装编译postgresql

	#su - postgres
使用git下载postgresql源代码
	
	#git clone git://git.postgresql.org/git/postgresql.git
![2015-11-24_7-26-40](/uploads/2015/11/2015-11-24_7-26-40.jpg)

切换到一个发布版本
	
	#cd postgresql
	#git checkout REL9_3_4
![2015-11-26_20-35-46](/uploads/2015/11/2015-11-26_20-35-46.jpg)
	
	#cd ..
重新copy一个目录，确保服务器上拉下来的代码不受影响
	
	#cp -rf postgresql/ pgsql
![2015-11-26_20-38-39](/uploads/2015/11/2015-11-26_20-38-39.jpg)

	#yum install bison
	#yum install flex
	#cd pgsql
	#./configure --enable-depend --enable-cassert --enable-debug

系统默认安装至/usr/local/pgsql,我们可以使用下面命令安装至其他目录
	
	#./configure --prefix=/usr/local/pgsqldev --enable-depend --enable-cassert --enable-debug

![2015-11-26_21-26-41](/uploads/2015/11/2015-11-26_21-26-41.jpg)
	#make
	#make all
![2015-11-26_21-53-19](/uploads/2015/11/2015-11-26_21-53-19.jpg)

## 4.使用eclipse调试

### 4.1 import源码包

我们postgres用户名打开eclipse,选择workspace
![2015-11-28_13-27-11](/uploads/2015/11/2015-11-28_13-27-11.jpg)

选择"C/C++ » Existing Code as Makefile Project"
![2015-11-28_13-28-43](/uploads/2015/11/2015-11-28_13-28-43.jpg)

点击Next

![2015-11-28_13-29-33](/uploads/2015/11/2015-11-28_13-29-33.jpg)

选择Linux GCC
![2015-11-28_13-29-59](/uploads/2015/11/2015-11-28_13-29-59.jpg)

这样我们就建立了一个postgres源码的项目。

### 4.2	调试Initdb
建立data目录
		
		#mkdir /home/data
		#chown postgres:postgres -R /home/data
选择initdb右键，debug as，在C/C++ Application下选择新建，Arguments输入-D /home/data
![2015-11-28_13-51-42](/uploads/2015/11/2015-11-28_13-51-42.jpg)

点击Apply按钮，然后点击Debug按钮。在这里我们可以设置断点，启动调试。
![2015-11-28_13-54-59](/uploads/2015/11/2015-11-28_13-54-59.jpg)

### 4.3	调试postgres
首选启动postgres
	
	#/usr/local/pgsql/bin/pg_ctl -D /home/data -l /home/data/logfile start
	#/usr/local/pgsql/bin/psql

![2015-11-28_15-10-28](/uploads/2015/11/2015-11-28_15-10-28.jpg)

在eclipse选择postgre debug as，在Attach下新建debug

![2015-11-28_15-07-06](/uploads/2015/11/2015-11-28_15-07-06.jpg)

我们看到进程5787，5788就是刚才我们运行的psql，双击5788进程，设置断点。
![2015-11-28_15-09-29](/uploads/2015/11/2015-11-28_15-09-29.jpg)

命令行输入SQL语句select current_time;
![2015-11-28_15-33-49](/uploads/2015/11/2015-11-28_15-33-49.jpg)

断点处我们看到刚才输入的SQL语句
![2015-11-28_15-33-22](/uploads/2015/11/2015-11-28_15-33-22.jpg)

### 4.4 postgres测试框架

测试框架包含在postgres源码目录中src/test
![2015-11-28_16-17-03](/uploads/2015/11/2015-11-28_16-17-03.jpg)

主要用例都放在regress目录下,有data、expected、input、output、sql等目录和parallel_schedule、serial_schedule、standby_schedule等设置测试用例并行、串行执行的文件，如下图：

![2015-11-28_16-17-30](/uploads/2015/11/2015-11-28_16-17-30.jpg)
postgres的regress test的流程为：逐个执行sql/目录下的sql脚本，将执行的结果重定向到results/目录下。而expected/目录下则是预期的执行结果。将results目录下的文件逐个与expected中文件进行diff。若文件不一致，则判定为对应的sql脚本执行结果为失败，反之为成功。
参考：[Working_with_Eclipse](https://wiki.postgresql.org/wiki/Working_with_Eclipse)