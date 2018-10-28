---
title: 'Kafka入门之十一:Kafka的监控'
tags:
  - JConsole
  - Kafka Manager
id: 1495
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-12-23 22:39:23
---
## 1. 简介
kafka的数据统计是通过一个叫metrics的工具进行收集的，metrics是一个java类库。metrics以JMX的形式提供了对外查看数据的接口，因此我们首先要在kafka启动的时候指定jmx的端口，然后通过可视化工具jconsole或kafka manager查看。下面我们分别介绍一下。
## 2. JMX配置
首先我们看JMX如何配置，在Kafka工具中有个脚本`bin/kafka-run-class.sh`定义了JMX的启动方法。

    # JMX settings
    if [ -z "$KAFKA_JMX_OPTS" ]; then
      KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false  -Dcom.sun.management.jmxremote.ssl=false "
    fi

    # JMX port to use
    if [  $JMX_PORT ]; then
      KAFKA_JMX_OPTS="$KAFKA_JMX_OPTS -Dcom.sun.management.jmxremote.port=$JMX_PORT "
    fi
    
因此，我们只要在docker-compose文件中定义KAFKA_JMX_OPTS和JMX_PORT，那么启动docker同时，JMX自动启动。
    
        hostname: kafka0
        ports:
          - "19092:19092"
          - "29092:29092"
          - "18083:18083"
          - "12345:12345"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:12181,zookeeper1:12182,zookeeper2:12183/kafka
          BROKER_ID: 0
          LISTENERS: PLAINTEXT://kafka0:19092,SSL://kafka0:29092
          ZOOKEEPER_SESSION_TIMEOUT: 3600000
          CONNECT_REST_PORT: 18083
          KAFKA_PROPERTY_AUTO_CREATE_TOPICS_ENABLE: "false"
          KAFKA_PROPERTY_SSL_CLIENT_AUTH: required
          JMX_PORT: 12345
          KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false  -Dcom.sun.management.jmxremote.ssl=false"
    
## 3. JConsole监控
我们知道java 内置了jconsole工具，本博客之前也有介绍过[jconsole](/2016/04/jconsole-remote-mycat/),因此jconsole的使用并不陌生。我们可以通过docker inspect 查看某个broker的IP地址及jmx端口。然后使用命令进入,如下
    
    jconsole 172.19.0.6:12345
![2016-12-23_20-53-38](/uploads/2016/12/2016-12-23_20-53-38.png)

通过查看Mbean标签下的参数，我们可以获取kafka的一些运行参数。
### 4. Kafka Manager监控
Kafka Manager是Yahoo构建的一个开源的基于Web的管理工具，可以简化开发者和运维工程师维护Kafka集群的工作。kakfa manager的[GitHub ](https://github.com/yahoo/kafka-manager)地址。同样的我们创建一个kakfa manager的docker镜像,然后加入`docker-compose`文件。
    
      kafka-manager:
        build:
          context: .
          dockerfile: kafka-manager.Dockerfile
        image: kafka-manager:1.0
        container_name: kafka-manager
        hostname: kafka-manager
        ports:
          - "38080:38080"
        environment:
          ZK_HOSTS: zookeeper0:12181
          PORT: 38080
        expose:
          - 38080

访问38080端口并加入一个Kafka集群。
![2016-12-23_21-11-49](/uploads/2016/12/2016-12-23_21-11-49.png)

检查broker，
![2016-12-23_21-15-55](/uploads/2016/12/2016-12-23_21-15-55.png)

我们还可以管理以下功能：
* 管理几个不同的集群;
* 检查集群的状态(topics, brokers, 副本的分布, 分区的分布);
* 创建topics
* Preferred副本选举
* 重新分配分区

>PS,kakfa监控还有一些工具比如Kafka Web Console，KafkaOffsetMonitor。感兴趣的朋友可以去测试。