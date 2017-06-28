---
title: Postgres-XL集群安装与配置
tags:
  - postgresql
id: 370
categories:
  - 数据库
date: 2015-12-20 19:16:13
---

# 1.Postgres-XL简介

Postgres-XL是从Postgres-XC衍生而来的一款产品, 经过改良, 对MPP这块做了比较大的改进.所以它同样包含了以下组件。
gtm：利用pg的MVCC全局地控制tuple，一个pgxc中只能有一个gtm。
gtm_standby：gtm的备机。
gtm_proxy：降低gtm压力，用多个代理来进行对coordinator进行操作。
coordinator：协调节点，负责操作所有datanode，本身不存放数据。在coordinator上可以以distribute切片(分布)或者replication复制的方式进行创建表。
datanode：存放数据的节点，如果数据在coordinator上是以切片方式建的表则数据只存放此表的一部分数据，如果是replication方式的表则存放全部数据。
我们来看一下Postgres-XL和postgresql, Postgres-XC的对比 :
<table>
<tbody>
<tr>
<td width="120">**PostgreSQL**</td>
<td width="120">**Postgres-XC**</td>
<td width="120">**Postgres-XL**</td>
</tr>
<tr>
<td width="120">Open source SQL database for Enterprises</td>
<td width="120">Open source SQL database for Enterprises</td>
<td width="120">Open source SQL database for Enterprises</td>
</tr>
<tr>
<td style="text-align: center" width="120">-</td>
<td width="120">Large coverage of PostgreSQL support</td>
<td width="120">Large coverage of PostgreSQL support</td>
</tr>
<tr>
<td style="text-align: center" width="120">-</td>
<td width="120">Global MVCC Consistency</td>
<td width="120">Global MVCC Consistency</td>
</tr>
<tr>
<td style="text-align: center" width="120">-</td>
<td style="text-align: center" width="120">-</td>
<td width="120">MPP query support</td>
</tr>
<tr>
<td style="text-align: center" width="120">-</td>
<td style="text-align: center" width="120">-</td>
<td width="120">Performance improvements</td>
</tr>
<tr>
<td style="text-align: center" width="120">-</td>
<td style="text-align: center" width="120">-</td>
<td width="120">Multi-tenant security</td>
</tr>
</tbody>
</table>
关于Postgres-XC，参见另一篇博文[Postgres-XC集群安装与配置](http://blog.yaodataking.com/2015/12/postgres-xc.html)

# 2.安装环境

准备一台Centos 6.5 GNOME 2.28.2
从官网下载源码：http://sourceforge.net/projects/postgres-xl/
官方文档:http://files.postgres-xl.org/documentation/index.html
安装配置如下图：
<table>
<tbody>
<tr>
<td width="100">IP</td>
<td width="102">角色</td>
<td width="92"> 端口</td>
<td width="100">Nodename</td>
<td width="143">安装目录</td>
</tr>
<tr>
<td width="100">192.168.199.151</td>
<td width="102">GTM</td>
<td width="92">6666</td>
<td width="100">gtm</td>
<td width="143">/pgxl_data/gtm</td>
</tr>
<tr>
<td width="100">192.168.199.151</td>
<td width="102">Coordinator</td>
<td width="92">1921</td>
<td width="100">coord1</td>
<td width="143">/pgxl_data/coordinator/cd1</td>
</tr>
<tr>
<td width="100">192.168.199.151</td>
<td width="102">Coordinator</td>
<td width="92">1925</td>
<td width="100">coord2</td>
<td width="143">/pgxl_data/coordinator/cd2</td>
</tr>
<tr>
<td width="100">192.168.199.151</td>
<td width="102">Datanode</td>
<td width="92">15431</td>
<td width="100">db1</td>
<td width="143">/pgxl_data/datanode/dn1</td>
</tr>
<tr>
<td width="100">192.168.199.151</td>
<td width="102">Datanode</td>
<td width="92">15432</td>
<td width="100">db2</td>
<td width="143">/pgxl_data/datanode/dn2</td>
</tr>
</tbody>
</table>
有条件的可以把这个角色分别安装在不同主机上。

# 3.安装配置

1）创建用户
#groupadd pgxl
#useradd pgxl -g pgxl
#passwd pgxl

[![2015-12-20_16-27-50](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-27-50.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-27-50.jpg)
2)安装
安装依赖库
#yum install -y flex bison readline-devel zlib-devel openjade docbook-style-dsssl
#tar -xzvf postgres-xl-v9.2-src.tar.gz
#cd postgres-xl
#./configure --prefix=/usr/local/pgsql_xl
[![2015-12-20_16-11-58](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-11-58.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-11-58.jpg)
#make
#make install
[![2015-12-20_16-26-35](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-26-35.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-26-35.jpg)
3）创建存放路径
#mkdir -p /pgxl_data/gtm
#mkdir -p /pgxl_data/coordinator/cd1
#mkdir -p /pgxl_data/coordinator/cd2
#mkdir -p /pgxl_data/datanode/dn1
#mkdir -p /pgxl_data/datanode/dn2
#chown -R pgxl:pgxl /pgxl_data
[![2015-12-20_16-54-34](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-54-34.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_16-54-34.jpg)

4）配置环境变量
#su - pgxl
#vi .bash_profile
export PGHOME=/usr/local/pgsql_xl
export LANG=en_US.utf8
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib
export PATH=$PGHOME/bin:$PATH:.
export MANPATH=$PGHOME/share/man:$MANPATH
#source .bash_profile
5）初始化节点
#initgtm -Z gtm -D /pgxl_data/gtm
#initdb -D /pgxl_data/coordinator/cd1 --nodename coord1 
#initdb -D /pgxl_data/coordinator/cd2 --nodename coord2 
#initdb -D /pgxl_data/datanode/dn1 --nodename db1
#initdb -D /pgxl_data/datanode/dn2 --nodename db2 
6)配置节点
配置gtm节点
vi /pgxl_data/gtm/gtm.conf
nodename = 'gtm'
listen_addresses = '*'
port = 6666
startup = ACT

配置 coordinator 节点
vi /pgxl_data/coordinator/cd1/postgresql.conf
#------------------------------------
listen_addresses = '*'
port = 1921
max_connections = 100
# DATA NODES AND CONNECTION POOLING
#----------------------------------------------
pooler_port = 6661
max_pool_size = 100

# GTM CONNECTION
#-----------------------------
gtm_host = '192.168.199.151'
gtm_port = 6666
pgxc_node_name = 'coord1'

vi /pgxl_data/coordinator/cd1/pg_hba.conf
# IPv4 local connections:
host all all 127.0.0.1/32 trust
host all all 192.168.199.151/32 trust
host all all 0.0.0.0/0 md5

vi /pgxl_data/coordinator/cd2/postgresql.conf
#------------------------------------
listen_addresses = '*'
port = 1925
max_connections = 100
# DATA NODES AND CONNECTION POOLING
#----------------------------------------------
pooler_port = 6662
max_pool_size = 100

# GTM CONNECTION
#-----------------------------
gtm_host = '192.168.199.151'
gtm_port = 6666
pgxc_node_name = 'coord2'

vi /pgxl_data/coordinator/cd2/pg_hba.conf
# IPv4 local connections:
host all all 127.0.0.1/32 trust
host all all 192.168.199.151/32 trust
host all all 0.0.0.0/0 md5

配置 datanode 节点
vi /pgxl_data/datanode/dn1/postgresql.conf
#------------------------------------
listen_addresses = '*'
port = 15431
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
pgxc_node_name = 'db1'

vi /pgxl_data/datanode/dn1/pg_hba.conf
# IPv4 local connections:
host all all 127.0.0.1/32 trust
host all all 192.168.199.151/32 trust
host all all 0.0.0.0/0 md5

vi /pgxl_data/datanode/dn2/postgresql.conf
#------------------------------------
listen_addresses = '*'
port = 15432
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
pgxc_node_name = 'db2'

vi /pgxl_data/datanode/dn2/pg_hba.conf
# IPv4 local connections:
host all all 127.0.0.1/32 trust
host all all 192.168.199.151/32 trust
host all all 0.0.0.0/0 md5

7）启动节点
启动顺序
gtm + (gtmstandby) + (gtmproxy) + datanode + coordinator
启动gtm
#gtm -D /pgxl_data/gtm &amp;
启动数据节点
#postgres --datanode -D /pgxl_data/datanode/dn1 &amp;
#postgres --datanode -D /pgxl_data/datanode/dn2 &amp;
启动coordinator节点
#postgres --coordinator -D /pgxl_data/coordinator/cd1 &amp;
#postgres --coordinator -D /pgxl_data/coordinator/cd2 &amp;
[![2015-12-20_18-21-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-21-05.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-21-05.jpg)
8）注册节点
在coord1上注册：
#psql -p 1921 postgres
select * from pgxc_node;
create node coord2 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1925);
create node db1 with(TYPE=datanode,HOST='192.168.199.151',PORT=15431,primary);
create node db2 with(TYPE=datanode,HOST='192.168.199.151',PORT=15432);
alter node coord1 with(TYPE='coordinator',HOST='192.168.199.151',PORT=1921);
select pgxc_pool_reload();
在coord2上注册：
#psql -p 1925 postgres
select * from pgxc_node;
create node coord1 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1921);
create node db1 with(TYPE=datanode,HOST='192.168.199.151',PORT=15431,primary);
create node db2 with(TYPE=datanode,HOST='192.168.199.151',PORT=15432);
alter node coord2 with(TYPE='coordinator',HOST='192.168.199.151',PORT=1925);
select pgxc_pool_reload();
在db1上注册：
#psql -p 15431 postgres
select * from pgxc_node;
create node coord1 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1921);
create node coord2 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1925);
create node db2 with(TYPE=datanode,HOST='192.168.199.151',PORT=15432);
alter node db1 with(TYPE='datanode',HOST='192.168.199.151',PORT=15431,primary=true);
select pgxc_pool_reload();
在db2上注册：
#psql -p 15432 postgres
select * from pgxc_node;
create node coord1 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1921);
create node coord2 with(TYPE=coordinator,HOST='192.168.199.151',PORT=1925);
create node db1 with(TYPE=datanode,HOST='192.168.199.151',PORT=15431,primary);
alter node db2 with(TYPE=datanode,HOST='192.168.199.151',PORT=15432,primary=false);
select pgxc_pool_reload();
所有节点都应该如下所示
[![2015-12-20_19-26-22](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_19-26-22.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_19-26-22.jpg)
9）停止节点
停止顺序
coordinator+datanode+(gtmproxy)+(gtmstandby)+gtm  
#pg_ctl stop -D /pgxl_data/coordinator/cd1 -Z coordinator -m fast
#pg_ctl stop -D /pgxl_data/coordinator/cd2 -Z coordinator -m fast
#pg_ctl stop -D /pgxl_data/datanode/dn1 -Z datanode -m fast
#pg_ctl stop -D /pgxl_data/datanode/dn2 -Z datanode -m fast
#gtm_ctl stop -D /pgxl_data/gtm -Z gtm -m fast

# 4.验证测试

在coord1上创建一个数据库，并建立一个新表
#psql -p 1921 postgres
create database pgxl_test;
\c pgxc_test;
create table test_xl (id integer,name varchar(32));
insert into test_xl select generate_series(1,100),'pgxl_test';
在coord2上查询，看是否能够查询到在coord1上新建的数据库和表
[![2015-12-20_18-44-25](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-44-25.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-44-25.jpg)
在db1上查看
[![2015-12-20_18-44-58](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-44-58.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-44-58.jpg)
在db2上查看
[![2015-12-20_18-45-20](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-45-20.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-12-20_18-45-20.jpg)
这说明我们在coordinator上是以distribute切片方式建的表，数据分别放在datanode上。