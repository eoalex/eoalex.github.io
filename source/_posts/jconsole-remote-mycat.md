---
title: JConsole远程监控Mycat示列
tags:
  - JConsole
  - Mycat
id: 1025
categories:
  - 数据库
  - 运维
date: 2016-04-03 15:24:40
---

## 一、简介
JConsole是一个基于JMX的GUI工具，用于连接正在运行的JVM。JConsole可以以三种方式连接正在运行的JVM：
1.Local：使用JConsole连接一个正在本地系统运行的JVM，并且执行程序的和运行JConsole的需要是同一个用户。JConsole使用文件系统的授权通过RMI连接器连接到平台的MBean服务器上。这种从本地连接的监控能力只有Sun的JDK具有。
2.Remote：使用下面的URL通过RMI连接器连接到一个JMX代理：hoostname:port或service:jmx:protocol:sap。JConsole为建立连接，需要在环境变量中设置jmxremote.password来指定用户名和密码从而进行授权。
3.Advanced:使用一个特殊的URL连接JMX代理。一般情况使用自己定制的连接器而不是RMI提供的连接器来连接JMX代理，或者是一个使用JDK实现了JMX和JMX Rmote的应用。本文我们就第二种remote方式来监控Mycat的运行情况。本文实验环境还是在docker中运行。

## 二、配置
在使用JConsole监控前，我们必须对mycat的容器做一些设置，增加jmxremote.password文件至容器中，另外增加expose 1984端口。
具体builder文件如下：

    from centos:7
    MAINTAINER Alex Wu

    #install java8
    ADD jdk-8u51-linux-x64.gz /usr/local
    RUN ln -s /usr/local/jdk1.8.0_51 /usr/local/java
    ENV JAVA_HOME /usr/local/java
    ENV PATH $PATH:$JAVA_HOME/bin
    COPY jmxremote.password /usr/local/java/jre/lib/management/

    #install mycat
    VOLUME /opt/mycat/conf
    ADD Mycat-server-1.5.1-RELEASE-20160328130228-linux.tar.gz /opt

    EXPOSE 8066 9066 1984

    CMD ["/opt/mycat/bin/mycat","console"]

jmxremote.password文件配置如下，第一列是用户名，第二列是密码：
    
    monitorRole monitor
    controlRole monitor
    
我们还需要在Wrapper.conf文件里配置jmx端口，并指定docker宿主机的IP地址。详细配置如下：
   	 
	# Java Additional Parameters
    #wrapper.java.additional.1=
    wrapper.java.additional.1=-DMYCAT_HOME=.
    wrapper.java.additional.2=-server
    wrapper.java.additional.3=-XX:MaxPermSize=64M
    wrapper.java.additional.4=-XX:+AggressiveOpts
    wrapper.java.additional.5=-XX:MaxDirectMemorySize=2G
    wrapper.java.additional.6=-Dcom.sun.management.jmxremote
    wrapper.java.additional.7=-Dcom.sun.management.jmxremote.port=1984
    wrapper.java.additional.8=-Dcom.sun.management.jmxremote.authenticate=false
    wrapper.java.additional.9=-Dcom.sun.management.jmxremote.ssl=false
    wrapper.java.additional.10=-Dcom.sun.management.jmxremote.rmi.port=1984
    wrapper.java.additional.11=-Djava.rmi.server.hostname=192.168.199.168
    wrapper.java.additional.12=-Xmx4G
    wrapper.java.additional.13=-Xms1G
    
配置完成,启动mycat容器。（事先我们已启动mysql容器，可参见本博客以前文章，这里不再详细介绍）
    
    docker create --name mycat101 -v /mysql/mycatconf:/opt/mycat/conf -p 8066:8066 -p 9066:9066 -p 1984:1984 mycat
    docker start mycat101
    
我们现在可以使用JConsole来监控Mycat了，(JConsole我们这里使用Ubuntu平台),下面是连接画面。
![2016-04-03_14-51-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-03_14-51-13.jpg)

下面显示已连上
![2016-04-02_10-30-02](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-02_10-30-02.jpg)

## 三、测试
接下来我们用mycat的测试工具来模拟插入200万条记录。
    
    ./test_stand_insert_perf.sh jdbc:mysql://localhost:8066/TESTDB test test 100 "0-100M,100M1-200M"
    
测试数据导入过程中。
![2016-04-02_10-30-31](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-02_10-30-31.jpg)

导入完成，我们看到tps 4818左右。

![2016-04-02_10-45-01](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-02_10-45-01.jpg)

查看Mycat 9066端口
    
    mysql> show @@threadpool;
    +------------------+-----------+--------------+-----------------+----------------+------------+
    | NAME             | POOL_SIZE | ACTIVE_COUNT | TASK_QUEUE_SIZE | COMPLETED_TASK | TOTAL_TASK |
    +------------------+-----------+--------------+-----------------+----------------+------------+
    | Timer            |         2 |            0 |               0 |           3137 |       3137 |
    | BusinessExecutor |         4 |            0 |               0 |        2001010 |    2001010 |
    +------------------+-----------+--------------+-----------------+----------------+------------+
    2 rows in set (0.00 sec)

    mysql> show @@heartbeat;
    +---------+-------+------------+------+---------+-------+--------+---------+--------------+---------------------+-------+
    | NAME    | TYPE  | HOST       | PORT | RS_CODE | RETRY | STATUS | TIMEOUT | EXECUTE_TIME | LAST_ACTIVE_TIME    | STOP  |
    +---------+-------+------------+------+---------+-------+--------+---------+--------------+---------------------+-------+
    | host101 | mysql | 172.17.0.2 | 3306 |       1 |     0 | idle   |       0 | 6,10,10      | 2016-04-01 14:41:16 | false |
    | host102 | mysql | 172.17.0.3 | 3306 |      -1 |     0 | idle   |       0 | 0,0,0        | 2016-04-01 14:41:16 | false |
    +---------+-------+------------+------+---------+-------+--------+---------+--------------+---------------------+-------+
    2 rows in set (0.00 sec)

    mysql> show @@datanode;
    +------+----------------+-------+-------+--------+------+------+---------+------------+----------+---------+---------------+
    | NAME | DATHOST        | INDEX | TYPE  | ACTIVE | IDLE | SIZE | EXECUTE | TOTAL_TIME | MAX_TIME | MAX_SQL | RECOVERY_TIME |
    +------+----------------+-------+-------+--------+------+------+---------+------------+----------+---------+---------------+
    | dn1  | localhost1/db1 |     0 | mysql |      0 |    9 | 1000 |   18037 |          0 |        0 |       0 |            -1 |
    | dn2  | localhost1/db2 |     0 | mysql |      0 |    0 | 1000 |      35 |          0 |        0 |       0 |            -1 |
    | dn3  | localhost1/db3 |     0 | mysql |      0 |    1 | 1000 |     205 |          0 |        0 |       0 |            -1 |
    +------+----------------+-------+-------+--------+------+------+---------+------------+----------+---------+---------------+
    3 rows in set (0.00 sec)

    mysql> show @@processor;
    +------------+-----------+-----------+-------------+---------+---------+-------------+--------------+------------+----------+----------+----------+
    | NAME       | NET_IN    | NET_OUT   | REACT_COUNT | R_QUEUE | W_QUEUE | FREE_BUFFER | TOTAL_BUFFER | BU_PERCENT | BU_WARNS | FC_COUNT | BC_COUNT |
    +------------+-----------+-----------+-------------+---------+---------+-------------+--------------+------------+----------+----------+----------+
    | Processor0 | 257904920 | 257967902 |           0 |       0 |       0 |         884 |         1000 |         11 |      334 |        1 |       10 |
    +------------+-----------+-----------+-------------+---------+---------+-------------+--------------+------------+----------+----------+----------+
    1 row in set (0.00 sec)

    mysql> show @@datasource;
    +----------+---------+-------+------------+------+------+--------+------+------+---------+
    | DATANODE | NAME    | TYPE  | HOST       | PORT | W/R  | ACTIVE | IDLE | SIZE | EXECUTE |
    +----------+---------+-------+------------+------+------+--------+------+------+---------+
    | dn1      | host101 | mysql | 172.17.0.2 | 3306 | W    |      0 |   10 | 1000 |   18284 |
    | dn1      | host102 | mysql | 172.17.0.3 | 3306 | R    |      0 |    0 | 1000 |       0 |
    | dn3      | host101 | mysql | 172.17.0.2 | 3306 | W    |      0 |   10 | 1000 |   18284 |
    | dn3      | host102 | mysql | 172.17.0.3 | 3306 | R    |      0 |    0 | 1000 |       0 |
    | dn2      | host101 | mysql | 172.17.0.2 | 3306 | W    |      0 |   10 | 1000 |   18284 |
    | dn2      | host102 | mysql | 172.17.0.3 | 3306 | R    |      0 |    0 | 1000 |       0 |
    +----------+---------+-------+------------+------+------+--------+------+------+---------+
    6 rows in set (0.01 sec)

    mysql> show @@cache;
    +-------------------------------------+-------+------+--------+------+------+---------------+---------------+
    | CACHE                               | MAX   | CUR  | ACCESS | HIT  | PUT  | LAST_ACCESS   | LAST_PUT      |
    +-------------------------------------+-------+------+--------+------+------+---------------+---------------+
    | ER_SQL2PARENTID                     |  1000 |    0 |      0 |    0 |    0 |             0 |             0 |
    | SQLRouteCache                       | 10000 |    1 |      1 |    0 |    1 | 1459522244686 | 1459522244749 |
    | TableID2DataNodeCache.TESTDB_ORDERS | 50000 |    0 |      0 |    0 |    0 |             0 |             0 |
    +-------------------------------------+-------+------+--------+------+------+---------------+---------------+
    3 rows in set (0.02 sec)

我们再看看JConsole会记录什么?
![2016-04-02_10-43-39](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-02_10-43-39.jpg)

我们看到CPU基本稳定在30%左右，内存使用量基本在50M至250M波动，线程和类基本固定在一个数值。
所以，我们看到使用JConsole可以使我们简单快捷的进行JAVA进程的监控。