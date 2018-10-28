---
title: 基于Docker容器的MyCat高可用方案(1)HAproxy
tags:
  - Docker
  - HAproxy
  - Mycat
  - Mysql
id: 911
categories:
  - 数据库
  - 运维
date: 2016-03-11 22:20:49
---
## 1. 简介
Mycat是一个彻底开源的新颖的数据库中间件产品。它的出现将彻底结束数据库的瓶颈问题，从而使数据库的高可用，高负载成为可能。在
[基于Mycat的MySQL主从读写分离及自动切换的docker实现](/2016/01/17/mycat-mysql-docker-sample1/)一文中，我们已经实现基于Mycat的Mysql高可用，但是Mycat本身也存在稳定性和单点问题，所以本文我们通过HAproxy实现MyCat的高可用。架构图如下：
![2016-03-08_20-27-59](/uploads/2016/03/2016-03-08_20-27-59.jpg)
本文所有组件都采用docker镜像和容器，为简单起见，都运行在一台宿主机上，系统为centos 7
## 2. Mysql配置
创建并启动容器

    docker create --name mysqlsrv101 -v /mysql/data/mysql101:/var/lib/mysql -v /mysql/101:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 mysql:latest
    docker create --name mysqlsrv102 -v /mysql/data/mysql102:/var/lib/mysql -v /mysql/102:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3316:3306 mysql:latest
    docker start mysqlsrv101
    docker start mysqlsrv102
    
mysql配置文件参见[基于Mycat的MySQL主从读写分离及自动切换的docker实现](/2016/01/17/mycat-mysql-docker-sample1/)一文
## 3. Mycat配置
创建并启动容器,两个mycat容器共用同一配置文件。
    
    docker create --name mycat01 -v /mysql/mycatconf:/opt/mycat/conf -p 8066:8066 -p 9066:9066 mycat:1.5
    docker create --name mycat02 -v /mysql/mycatconf:/opt/mycat/conf -p 8067:8066 -p 9067:9066 mycat:1.5
    docker start mycat01
    docker start mycat02
    
mycat的配置文件参见[基于Mycat的MySQL主从读写分离及自动切换的docker实现](/2016/01/17/mycat-mysql-docker-sample1/)一文
## 4. HAproxy配置
### 4.1 下载HAproxy镜像
    [root@localhost ~]# docker pull haproxy
    Using default tag: latest
    latest: Pulling from library/haproxy
    73e8d4f6bf84: Pull complete
    040bf8e08425: Pull complete
    4c4d55db13de: Pull complete
    03fd84c9b2d1: Pull complete
    31209e66fb19: Pull complete
    e89816114fc3: Pull complete
    3de0fe50637a: Pull complete
    7bcd41b9d648: Pull complete
    Digest: sha256:6d1f490cd1e3c95bd38c525ea221d91e8470461602e0a56e9dde58567d99cbb8
    Status: Downloaded newer image for haproxy:latest
    
### 4.2 HAproxy配置
编辑haproxy配置文件`/mysql/haproxy/haproxy.cfg`,由于在docker中运行，需要把daemon注释掉。
    	
    	global
            #daemon  #remark this option while in docker
            maxconn 256

        defaults
            mode http
            timeout connect 5000ms
            timeout client 50000ms
            timeout server 50000ms

        listen  mycat
            bind 0.0.0.0:8068   
            mode tcp           
            balance roundrobin            
            server mycat1 172.17.0.4:8066 weight 1 check  inter 1s rise 2 fall 2 
            server mycat2 172.17.0.5:8066 weight 1 check  inter 1s rise 2 fall 2

        listen stats     #monitor
               mode http
               bind 0.0.0.0:8888
               stats enable
               stats uri /dbs
               stats realm Global\ statistics
               stats auth admin:admin
创建并启动HAproxy容器
    
    docker create --name myhaproxy01 -v /mysql/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro -p 8068:8068 -p 8888:8888 haproxy
    docker start myhaproxy01
    
## 5. 高可用测试
使用mysql客户端连接haproxy的端口8068，我们看到连接正常，并且正常返回查询数据。
    
    [root@localhost ~]# mysql -utest -p -h127.0.0.1 -P8068 -DTESTDB
    Enter password:
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 1612
    Server version: 5.5.8-mycat-1.5-alpha-20160108213035 MyCat Server (OpenCloundDB)

    Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql> select count(*) from person;
    +--------+
    | COUNT0 |
    +--------+
    | 100000 |
    +--------+
    1 row in set (0.04 sec)
    mysql>
    
查看haproxy的监控端口8888，我们看到mycat的两台容器都正常运行。
![2016-03-11_21-43-44](/uploads/2016/03/2016-03-11_21-43-44.jpg)
接下来我们模拟其中一台mycat突然宕机，数据连接情况。
    
    [root@localhost ~]# docker stop mycat02
    mycat02
    [root@localhost ~]# mysql -utest -p -h127.0.0.1 -P8068 -DTESTDB
    Enter password:
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 1616
    Server version: 5.5.8-mycat-1.5-alpha-20160108213035 MyCat Server (OpenCloundDB)

    Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql> select count(*) from person;
    +--------+
    | COUNT0 |
    +--------+
    | 100000 |
    +--------+
    1 row in set (0.04 sec)
    mysql>

我们看到虽然刚才停掉了mycat02,但是mysql还是正常连接，并且返回查询数据。
查看haproxy的监控端口8888，我们看到mycat服务器一台正常，一台宕机。
![2016-03-11_21-46-49](/uploads/2016/03/2016-03-11_21-46-49.jpg)