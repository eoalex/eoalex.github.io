---
title: Mycat环境下的mysql数据迁移实验
tags:
  - Mycat
  - Mysql
id: 972
categories:
  - 数据库
  - 运维
date: 2016-03-20 00:36:49
---

我们知道在生产环境下，数据库服务器如果出现磁盘IO、内存、CPU等性能的瓶颈，除了做一些性能优化外，选择一些数据迁移至新服务器也是不错的解决方案。特别在以Mycat为中间件的环境中，有些场景的数据迁移可能会非常的简单。
我们今天就来做这样一个场景，travelrecord表原来定义为10个分片，现在由于原服务器性能的原因将这10个分片中的2个分片转移到第二台MySQL上。大概思路是这样的，先导出第一台服务器2个分片的数据，然后导入到第二台服务器，同时配置这两台主从复制，主要同步这2个分片的数据，然后修改mycat配置，把这2个分片配置在第二台服务器，重新reload配置，确定写往2个分片的数据都写在了第二台服务器，就可以停止主从复制，数据迁移完成。这种方法应该说是一种比较快的数据迁移做法，基本上业务中断在数据的导出和导入。下面我们就具体通过docker容器方式来实现。
1.准备工作
1.1启动第一台mysql容器

    docker create --name mysqlsrv103 -v /mysql/data/mysql103:/var/lib/mysql -v /mysql/103:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3336:3306 mysql:latest
    docker start mysqlsrv103
    `</pre>
    1.2创建数据库
    <pre>`docker exec -it mysqlsrv103 /bin/bash
    mysql -uroot -p
    mysql&gt;create database db1;
    mysql&gt;create database db2;
    mysql&gt;create database db3;
    mysql&gt;create database db4;
    mysql&gt;create database db5;
    mysql&gt;create database db6;
    mysql&gt;create database db7;
    mysql&gt;create database db8;
    mysql&gt;create database db9;
    mysql&gt;create database db10;
    `</pre>
    1.3 配置mycat
    schema.xml配置
    [![2016-03-19_23-40-56](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-19_23-40-56.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-19_23-40-56.jpg)
    rule.xml配置
    [![2016-03-19_23-44-49](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-19_23-44-49.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-19_23-44-49.jpg)
    autopartition-long.txt文件设置
    <pre>`# range start-end ,data node index
    # K=1000,M=10000.
    0-100M=0
    100M1-200M=1
    200M1-300M=2
    300M1-400M=3
    400M1-500M=4
    500M1-600M=5
    600M1-700M=6
    700M1-800M=7
    800M1-900M=8
    900M1-1000M=9
    1000M1-2000M=10
    `</pre>
    启动mycat容器
    <pre>`docker create --name mycat03 -v /mysql/conf03:/opt/mycat/conf -p 8066:8066 -p 9066:9066 mycat:1.5
    docker start mycat03
    docker exec -it mycat03 /bin/bash
    `</pre>
    进入mycat8066端口，创建travelrecord表
    <pre>`mysql -utest -p -h127.0.0.1 -P8066 -DTESTDB
    create table travelrecord (id bigint not null primary key,user_id varchar(100),traveldate DATE, fee decimal,days int);
    `</pre>
    我们可以使用以下命令生成一些测试数据
    ./test_stand_insert_perf.sh jdbc:mysql://localhost:8066/TESTDB test test 100 "0-2000M"
    好，准备工作完成。接下来我们将启动和配置第二台服务器。

    2.配置第二台mysql容器
    <pre>`docker create --name mysqlsrv104 -v /mysql/data/mysql104:/var/lib/mysql -v /mysql/104:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3346:3306 mysql:latest
    docker start mysqlsrv104
    `</pre>
    导出第一台mysql容器db9,db10数据库
    mysqldump -uroot -p -h127.0.0.1 -P3336 db9 &gt; /root/db9.sql
    mysqldump -uroot -p -h127.0.0.1 -P3336 db10 &gt; /root/db9.sql
    导入到第二台mysql容器
    create database db9;
    create database db10;
    source /root/db9.sql;
    source /root/db10.sql;

    3.主从配置
    修改第一台mysql参数,加入需要同步的两个分片数据库db9和db10
    <pre>`
    [mysqld]
    log-bin=mysql-bin
    server-id=103
    lower_case_table_names=1
    binlog-do-db=db9
    binlog-do-db=db10
    `</pre>
    修改第二台mysql参数，加入需要同步的两个分片数据库db9和db10
    <pre>`
    [mysqld]
    log-bin=mysql-bin
    server-id=104
    lower_case_table_names=1
    replicate-do-db=db9
    replicate-do-db=db10
    `</pre>
    重启容器使配置生效。
    进入第一台mysql，show master status;
    <pre>`mysql&gt; show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000005 |      154 | db9,db10     |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)
    `</pre>
    进入第二台mysql,设置master参数，启动slave。
    <pre>`
    change master to master_host='172.17.0.2',master_user='root',master_password='123456',master_log_file='mysql-bin.000005',master_log_pos=154;
    start slave;
    `</pre>
    查看状态,如果Slave_IO_Running和Slave_SQL_Running都显示Yes,说明主从配置成功。
    <pre>`
    mysql&gt; show slave status\G;
    *************************** 1\. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 172.17.0.2
                      Master_User: root
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.000005
              Read_Master_Log_Pos: 154
                   Relay_Log_File: a26491a4abb2-relay-bin.000002
                    Relay_Log_Pos: 320
            Relay_Master_Log_File: mysql-bin.000005
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB: db9,db10
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
                 Master_Server_Id: 103
                      Master_UUID: 5a5e5ab5-ed6c-11e5-938d-0242ac110002
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
    `</pre>

    4\. 修改mycat配置
    把db9,db10更改至第二台mysql容器。
    [![2016-03-20_0-05-38](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-20_0-05-38.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-20_0-05-38.jpg)
    <pre>`
    mysql&gt; reload @@config_all;
    Query OK, 1 row affected (0.15 sec)
    Reload config success
    `</pre>

    进入mycat 8066端口，手工插入一条数据
    <pre>`
    mysql -utest -p -h127.0.0.0 -P8066 -DTESTDB
    insert into travelrecord (id,user_id,traveldate,fee,days) values(9800001,'huang','2014-06-05',720.5,3);  

[![2016-03-20_0-30-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-20_0-30-59.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/03/2016-03-20_0-30-59.jpg)
查看log，我们看到数据已经写往第二台mysql容器。至此我们确认数据迁移成功，此时可以停止主从复制了，第一台mysql容器的二个分片数据库也可以放心的删除了。