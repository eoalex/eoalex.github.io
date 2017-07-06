---
title: 基于Mycat的MySQL主从读写分离及自动切换的docker实现(1)
tags:
  - Docker
  - Mycat
  - Mysql
id: 520
categories:
  - 数据库
  - 运维
date: 2016-01-17 16:32:21
---
## 1. 简介
Mycat是一个彻底开源的新颖的数据库中间件产品。它的出现将彻底结束数据库的瓶颈问题，从而使数据库的高可用，高负载成为可能。本文将用Docker实现MySQL在主从配置（1主1从）情况下的读写分离及自动切换。
![2016-01-17_18-54-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-17_18-54-27.jpg)
## 2. MySQL主从配置
### 2.1 下拉Image
从Docker官方下拉MySQL的image
	
	#docker pull mysql:latest
### 2.2 设置目录
为了使MySql的数据保持在宿主机上，我们先建立几个目录。
	
	#mkdir -pv /mysql/data
建立主服务器的配置目录
	
	#mkdir -pv /mysql/101
建立从服务器的配置目录
	
	#mkdir -pv /mysql/102
![2016-01-16_22-55-07](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_22-55-07.jpg)
### 2.3 设置主从服务器配置
编辑101.cnf	
	
	#vi /mysql/101/101.cnf
	    [mysqld]
	    log-bin=mysql-bin
	    server-id=101
编辑102.cnf
	
	#vi /mysql/102/102.cnf
	    [mysqld]
	    log-bin=mysql-bin
	    server-id=102

### 2.4 创建主从服务器容器

    #docker create --name mysqlsrv101 -v /mysql/data/mysql101:/var/lib/mysql -v /mysql/101:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 mysql:latest
    #docker create --name mysqlsrv102 -v /mysql/data/mysql102:/var/lib/mysql -v /mysql/102:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3316:3306 mysql:latest
  
启动容器
    
    #docker start mysqlsrv101
    #docker start mysqlsrv102
![2016-01-17_19-07-02](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-17_19-07-02.jpg)

### 2.5 主服务器状态
登录主服务器的mysql，查询master的状态

    mysql> show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000003 |      154 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)

### 2.6 从服务器配置
登录从服务器的mysql，设置与主服务器相关的配置参数，change master to 

```
master_host='172.17.0.2',master_user='root',master_password='123456',master_log_file='mysql-
    bin.000003',master_log_pos=154;
    start slave;
```
![2016-01-16_23-08-54](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_23-08-54.jpg)
检查从服务器复制功能状态。

    mysql> show slave status\G
    *************************** 1\. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 172.17.0.2
                      Master_User: root
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.000003
              Read_Master_Log_Pos: 154
                   Relay_Log_File: 140ef1385e2b-relay-bin.000002
                    Relay_Log_Pos: 320
            Relay_Master_Log_File: mysql-bin.000003
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB:
              Replicate_Ignore_DB:
               Replicate_Do_Table:
           Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
                       Last_Errno: 0
                       Last_Error:
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 154
                  Relay_Log_Space: 534
                  Until_Condition: None
                   Until_Log_File:
                    Until_Log_Pos: 0
               Master_SSL_Allowed: No
               Master_SSL_CA_File:
               Master_SSL_CA_Path:
                  Master_SSL_Cert:
                Master_SSL_Cipher:
                   Master_SSL_Key:
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error:
                   Last_SQL_Errno: 0
                   Last_SQL_Error:
      Replicate_Ignore_Server_Ids:
                 Master_Server_Id: 101
                      Master_UUID: 6c303202-bbfd-11e5-8dd5-0242ac110002
                 Master_Info_File: /var/lib/mysql/master.info
                        SQL_Delay: 0
              SQL_Remaining_Delay: NULL
          Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
               Master_Retry_Count: 86400
                      Master_Bind:
          Last_IO_Error_Timestamp:
         Last_SQL_Error_Timestamp:
                   Master_SSL_Crl:
               Master_SSL_Crlpath:
               Retrieved_Gtid_Set:
                Executed_Gtid_Set:
                    Auto_Position: 0
             Replicate_Rewrite_DB:
                     Channel_Name:
               Master_TLS_Version:
    1 row in set (0.00 sec)
    mysql>

### 2.7 MySQL主从服务器测试
    进入主服务器(容器mysqlsrv101),创建几个数据库
	    
	    create database db1;
	    create database db2;
	    create database db3;
![2016-01-16_23-14-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_23-14-36.jpg)
进入从服务器(容器mysqlsrv102),可查看到刚才主服务器创建的数据库。
![2016-01-16_23-14-08](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_23-14-08.jpg)

至此MySQL主从服务器配置成功。
## 3.安装MyCat
### 3.1 Dockerfile文件
我们使用下面的Dockerfile自己build mycat镜像。

```
from centos:7
MAINTAINER Alex Wu

#install java8
ADD jdk-8u51-linux-x64.gz /usr/local
RUN ln -s /usr/local/jdk1.8.0_51 /usr/local/java
ENV JAVA_HOME /usr/local/java
ENV PATH $PATH:$JAVA_HOME/bin

#install mycat
VOLUME /opt/mycat/conf
ADD Mycat-server-1.5-alpha-20160108213035-linux.tar.gz /opt

EXPOSE 8066 9066
CMD ["/opt/mycat/bin/mycat","console"]
```

### 3.2 启动Mycat容器
复制mycat配置文件至`/mysql/mycatconf`
	
	#docker create --name mycat01 -v /mysql/mycatconf:/opt/mycat/conf -p 8066:8066 -p 9066:9066 mycat:1.5
    #docker start mycat01

## 4. 测试
### 4.1 测试读写分离
负载均衡类型，目前的取值有4种：
```
1. balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上。
2. balance="1"，全部的readHost与stand by writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。
3. balance="2"，所有读操作都随机的在writeHost、readhost上分发。
4. balance="3"，所有读请求随机的分发到wiriterHost对应的readhost执行，writerHost不负担读压力.
```
writeType属性,负载均衡类型，目前的取值有3种：
```
1. writeType="0", 所有写操作发送到配置的第一个writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后已切换后的为准，切换记录在配置文件中:dnindex.properties .
2. writeType="1"，所有写操作都随机的发送到配置的writeHost。
3. writeType="2"，没实现。
```
switchType属性
```
    - 1 表示不自动切换,默认值,心跳语句为select user()
    - 2 基于MySQL主从同步的状态决定是否切换,心跳语句为show slave status
    - 3 基于MySQL galary cluster的切换机制,心跳语句为show status like ‘wsrep%’
```
按如下修改schema.xml的balance,writeType,switchType的值。
![2016-01-17_16-18-45](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-17_16-18-45.jpg)

打开log4j.xml的debug选项，重启mycat容器，进入8066端口
    
    mysql -utest -p -h127.0.0.1 -P8066 -DTESTDB
插入三条记录

    mysql> insert into travelrecord (id,user_id,traveldate,fee,days) values(1,'wang','2014-01-05',510.5,3);
    Query OK, 1 row affected, 1 warning (0.10 sec)

    mysql> insert into travelrecord (id,user_id,traveldate,fee,days) values(5000001,'zhang','2015-01-05',1510.5,5);
    Query OK, 1 row affected, 1 warning (0.01 sec)

    mysql> insert into travelrecord (id,user_id,traveldate,fee,days) values(10000001,'huang','2014-06-05',720.5,3);
    Query OK, 1 row affected, 1 warning (0.00 sec)

我们看mycat.log文件，看到数据写入了主服务器。
![2016-01-17_16-04-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-17_16-04-27.jpg)
    我们再使用查询语句查询。

    mysql> select * from travelrecord;
    +----------+---------+------------+------+------+
    | id       | user_id | traveldate | fee  | days |
    +----------+---------+------------+------+------+
    |  5000001 | zhang   | 2015-01-05 | 1511 |    5 |
    |        1 | wang    | 2014-01-05 |  511 |    3 |
    | 10000001 | huang   | 2014-06-05 |  721 |    3 |
    +----------+---------+------------+------+------+
    3 rows in set (0.18 sec)

![2016-01-17_16-17-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-17_16-17-42.jpg)
我们看到数据是从服务器读取的，这证明了读写分离的配置是成功的。
### 4.2 测试自动切换
在MySQL主服务器突然宕机时，通过配置，Mycat会自动将连接转到MySQL从服务器上，在从服务器上读写数据。（此时MySQL主从服务器最好配置成主从互备)
按如下修改schema.xml的balance,writeType,switchType的值。
![2016-01-18_21-21-03](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-18_21-21-03.jpg)
重启mycat容器，进入9066管理端口。
    
    mysql -utest -p -h127.0.0.1 -P9066 ,我们看到心跳检测正常。
![2016-01-18_20-56-41](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-18_20-56-41.jpg)

此时，我们模拟宕机，停止MySQL主服务器的容器。再从9066端口查看心跳，主服务器出现异常。
![2016-01-18_21-09-14](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-18_21-09-14.jpg)
进入8066端口。
    
    mysql -utest -p -h127.0.0.1 -P8066 -DTESTDB,我们先清空数据，然后再插入三条记录。

    mysql> insert into travelrecord (id,user_id,traveldate,fee,days) values(1,'wang','2014-01-05',510.5,3);
    Query OK, 1 row affected, 1 warning (0.10 sec)

    mysql> insert into travelrecord (id,user_id,traveldate,fee,days) values(5000001,'zhang','2015-01-05',1510.5,5);
    Query OK, 1 row affected, 1 warning (0.01 sec)

    mysql> insert into travelrecord (id,user_id,traveldate,fee,days) values(10000001,'huang','2014-06-05',720.5,3);
    Query OK, 1 row affected, 1 warning (0.00 sec)

我们看mycat.log文件，看到数据写入了从服务器。
![2016-01-18_21-08-08](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-18_21-08-08.jpg)
我们再使用查询语句查询。看到查询也是从服务器读取的。
![2016-01-18_21-11-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-18_21-11-11.jpg)
查看dnindex.properties,看到localhost1值为1，也就是从服务器。

    [root@8872a66b639d conf]# cat dnindex.properties
    #update
    #Mon Jan 18 00:57:29 UTC 2016
    localhost1=1
    [root@8872a66b639d conf]#

这证明了自动切换的配置是成功的。