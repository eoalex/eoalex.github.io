---
title: Postgres-XC集群安装与配置
tags:
  - postgresql
id: 351
categories:
  - 数据库
date: 2015-12-20 13:54:32
---

## 1.Postgres-XC简介

PostgreSQL-XC 是一种提供写可靠性，多主节点数据同步，数据传输的开源集群方案，它包括很多组件，这些 PostgreSQL-XC 组件可以分别安装在多台物理机器或者虚拟机上。
组件：
gtm：利用pg的MVCC全局地控制tuple，一个pgxc中只能有一个gtm。
gtm_standby：gtm的备机。
gtm_proxy：降低gtm压力，用多个代理来进行对coordinator进行操作。
coordinator：协调节点，负责操作所有datanode，本身不存放数据。在coordinator上可以以distribute切片(分布)或者replication复制的方式进行创建表。
datanode：存放数据的节点，如果数据在coordinator上是以切片方式建的表则数据只存放此表的一部分数据，如果是replication方式的表则存放全部数据。
我们来看一下Postgres-XL和postgresql, Postgres-XC的对比 :

|PostgreSQL	|Postgres-XC	|Postgres-XL|
|:-----------|:-----------|:----------|
|Open source SQL database for Enterprises|	Open source SQL database for Enterprises|	Open source SQL database for Enterprises|
|-	|Large coverage of PostgreSQL support|	Large coverage of PostgreSQL support|
|-	|Global MVCC Consistency|	Global MVCC Consistency|
|-	|-	|MPP query support|
|-	|-	|Performance improvements|
|-	|-	|Multi-tenant security|

关于Postgres-XL，参见另一篇博文[Postgres-XL集群安装与配置](/2015/12/20/postgres-xl/)

## 2.安装环境

准备一台Centos 6.5 GNOME 2.28.2
从官网下载源码：http://sourceforge.net/projects/postgres-xc/
安装配置如下图：


|IP|	角色	| 端口	|Nodename	|安装目录|
|:--|:-------|:-----|:--------|:------|
|192.168.199.151|	GTM	|6666|	gtm|	/pgxc_data/gtm|
|192.168.199.151|	Coordinator|	1921	|coord1|	/pgxc_data/coordinator/cd1|
|192.168.199.151|	Coordinator|	1925	|coord2|	/pgxc_data/coordinator/cd2|
|192.168.199.151|	Datanode|	15431	|db1|	/pgxc_data/datanode/dn1|
|192.168.199.151|	Datanode|	15432	|db2|	/pgxc_data/datanode/dn2|

有条件的可以把这个角色分别安装在不同主机上。

## 3.安装配置

### 3.1 创建用户
	
	#groupadd pgxc
	#useradd pgxc -g pgxc
	#passwd pgxc
### 3.2 安装
	
	#tar -xzvf pgxc-v1.0.4.tar.gz
	#cd postgres-xc-1.0.4
	#./configure --prefix=/usr/local/pgsql_xc
	#make
	#make install
![2015-12-20_10-08-23](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_10-08-23.jpg)

### 3.3 创建存放路径
	
	#mkdir -p /pgxc_data/gtm
	#mkdir -p /pgxc_data/coordinator/cd1
	#mkdir -p /pgxc_data/coordinator/cd2
	#mkdir -p /pgxc_data/datanode/dn1
	#mkdir -p /pgxc_data/datanode/dn2
	#chown -R pgxc:pgxc /pgxc_data
![2015-12-20_10-15-22](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_10-15-22.jpg)

### 3.4 配置环境变量
	
	#su - pgxc
	#vi .bash_profile
		export PGHOME=/usr/local/pgsql_xc
		export LANG=en_US.utf8
		export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib
		export PATH=$PGHOME/bin:$PATH:.
		export MANPATH=$PGHOME/share/man:$MANPATH
	#source .bash_profile
### 3.5 初始化节点
	
	#initgtm -Z gtm -D /pgxc_data/gtm
	#initdb -D /pgxc_data/coordinator/cd1 --nodename coord1 
	#initdb -D /pgxc_data/coordinator/cd2 --nodename coord2 
	#initdb -D /pgxc_data/datanode/dn1 --nodename db1
	#initdb -D /pgxc_data/datanode/dn2 --nodename db2 
### 3.6 配置节点
配置gtm节点
	
	#vi /pgxc_data/gtm/gtm.conf
		nodename = 'gtm'
		listen_addresses = '*'
		port = 6666
		startup = ACT

配置 coordinator 节点
	
	#vi /pgxc_data/coordinator/cd1/postgresql.conf
		#------------------------------------
		listen_addresses = '*'
		port = 1921
		max_connections = 100
		# DATA NODES AND CONNECTION POOLING
		#----------------------------------------------
		pooler_port = 6667
		#min_pool_size = 1
		max_pool_size = 100
		
		# GTM CONNECTION
		#-----------------------------
		gtm_host = '192.168.199.151'
		gtm_port = 6666
		pgxc_node_name = 'coord1'

	#vi /pgxc_data/coordinator/cd1/pg_hba.conf
		# IPv4 local connections:
		host all all 127.0.0.1/32 trust
		host all all 192.168.199.151/32 trust
		host all all 0.0.0.0/0 md5

	#vi /pgxc_data/coordinator/cd2/postgresql.conf
		#------------------------------------
		listen_addresses = '*'
		port = 1925
		max_connections = 100
		# DATA NODES AND CONNECTION POOLING
		#----------------------------------------------
		pooler_port = 6668
		#min_pool_size = 1
		max_pool_size = 100
		
		# GTM CONNECTION
		#-----------------------------
		gtm_host = '192.168.199.151'
		gtm_port = 6666
		pgxc_node_name = 'coord2'

	#vi /pgxc_data/coordinator/cd2/pg_hba.conf
		# IPv4 local connections:
		host all all 127.0.0.1/32 trust
		host all all 192.168.199.151/32 trust
		host all all 0.0.0.0/0 md5

配置 datanode 节点
	
	#vi /pgxc_data/datanode/dn1/postgresql.conf
		#------------------------------------
		listen_addresses = '*'
		port = 15431
		max_connections = 100
		
		# GTM CONNECTION
		#-----------------------------
		gtm_host = '192.168.199.151'
		gtm_port = 6666
		pgxc_node_name = 'db1'

	#vi /pgxc_data/datanode/dn1/pg_hba.conf
		# IPv4 local connections:
		host all all 127.0.0.1/32 trust
		host all all 192.168.199.151/32 trust
		host all all 0.0.0.0/0 md5

	#vi /pgxc_data/datanode/dn2/postgresql.conf
		#------------------------------------
		listen_addresses = '*'
		port = 15432
		max_connections = 100
		
		# GTM CONNECTION
		#-----------------------------
		gtm_host = '192.168.199.151'
		gtm_port = 6666
		pgxc_node_name = 'db2'

	#vi /pgxc_data/datanode/dn2/pg_hba.conf
		# IPv4 local connections:
		host all all 127.0.0.1/32 trust
		host all all 192.168.199.151/32 trust
		host all all 0.0.0.0/0 md5

### 3.7 启动节点
启动顺序
gtm + (gtmstandby) + (gtmproxy) + datanode + coordinator
启动gtm
	
	#gtm -D /pgxc_data/gtm &amp;	
	#ps -ef|grep gtm

![2015-12-20_11-47-47](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_11-47-47.jpg)

	#gtm_ctl status -z gtm -D /pgxc_data/gtm

![2015-12-20_11-50-57](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_11-50-57.jpg)

启动数据节点
	
	#postgres -X -D /pgxc_data/datanode/dn1 &amp;
	#postgres -X -D /pgxc_data/datanode/dn2 &amp;
	#ps -ef|grep pgxc
![2015-12-20_11-56-30](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_11-56-30.jpg)

启动coordinator节点
	
	#postgres -C -D /pgxc_data/coordinator/cd1 &amp;
	#postgres -C -D /pgxc_data/coordinator/cd2 &amp;

![2015-12-20_11-58-33](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_11-58-33.jpg)

### 3.8 注册节点
在coord1上注册：
	
	#psql -p 1921 postgres
		select * from pgxc_node;
		create node coord2 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1925);
		create node db1 with(TYPE=datanode,HOST='192.168.199.151',PORT=15431,primary);
		create node db2 with(TYPE=datanode,HOST='192.168.199.151',PORT=15432);
		alter node coord1 with(TYPE='coordinator',HOST='192.168.199.151',PORT=1921);
		select pgxc_pool_reload();
![2015-12-20_14-36-10](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_14-36-10.jpg)

在coord2上注册：
	
	#psql -p 1925 postgres
		select * from pgxc_node;
		create node coord1 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1921);
		create node db1 with(TYPE=datanode,HOST='192.168.199.151',PORT=15431,primary);
		create node db2 with(TYPE=datanode,HOST='192.168.199.151',PORT=15432);
		alter node coord2 with(TYPE='coordinator',HOST='192.168.199.151',PORT=1925);
		select pgxc_pool_reload();

![2015-12-20_14-36-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_14-36-36.jpg)

### 3.9 停止节点
停止顺序
coordinator+datanode+(gtmproxy)+(gtmstandby)+gtm 
	
	#pg_ctl stop -D /pgxc_data/datanode/dn1 -Z datanode -m fast
	#pg_ctl stop -D /pgxc_data/datanode/dn2 -Z datanode -m fast
	#pg_ctl stop -D /pgxc_data/coordinator/cd1 -Z coordinator -m fast
	#pg_ctl stop -D /pgxc_data/coordinator/cd2 -Z coordinator -m fast
	#gtm_ctl stop -D /pgxc_data/gtm -Z gtm -m fast

## 4.验证测试

在coord1上创建一个数据库，并建立一个新表
	
	#psql -p 1921 postgres
	create database pgxc_test;
	\c pgxc_test;
	create table test_xc (id integer,name varchar(32));
	insert into test_xc select generate_series(1,100),'pgxc_test';
![2015-12-20_14-42-14](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_14-42-14.jpg)

在coord2上查询，看是否能够查询到在coord1上新建的数据库和表
![2015-12-20_14-46-53](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_14-46-53.jpg)

在db1上查看
![2015-12-20_14-48-01](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_14-48-01.jpg)

在db2上查看
![2015-12-20_14-48-35](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_14-48-35.jpg)

这说明我们在coordinator上是以distribute切片方式建的表，数据分别放在datanode上。