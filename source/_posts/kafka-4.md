---
title: 'Kafka入门之四:Kafka如何使用zookeeper'
tags:
  - Docker
  - docker-compose
  - Kafka
  - zookeeper
id: 1372
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-10-30 23:43:20
---
## 1. 简介
作为解决分布式一致性问题的工具，zookeeper应用广泛，在Hadoop集群部署时，我们已经领略过了，具体可参见博文[Hadoop 2.7.1 实现HA集群部署](/2015/10/hadoop-2-7-1-ha/)。今天我们在kafka之前三节课的基础上演示如何部署zookeeper集群。
## 2. docker-compose安装
因为要使用docker-compose这个工具来部署zookeeper和kafka容器。所以先安装`docker-compose`。

    sudo -i
    curl -L "https://github.com/docker/compose/releases/download/1.8.1/docker-compose-$(uname -s)-$(uname -m)" >    /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
    exit
![2016-10-30_9-07-18](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_9-07-18.jpg)

## 3. zookeeper镜像制作
在之前的基础上增加ZOOK>EEPER_ID，ZOOKEEPER_PORT，ZOOKEEPER_SERVERS参数使之动态配置。
    
    FROM java:openjdk-8-jre-alpine
    ARG MIRROR=http://mirrors.aliyun.com/
    ARG VERSION=3.4.6
    LABEL name="zookeeper" version=$VERSION
    RUN apk update && apk add ca-certificates && \
        apk add tzdata && \
        ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
        echo "Asia/Shanghai" > /etc/timezone

    RUN apk add --no-cache wget bash \
        && mkdir /opt \
        && wget -q -O - $MIRROR/apache/zookeeper/zookeeper-$VERSION/zookeeper-$VERSION.tar.gz | tar -xzf - -C /opt \
        && mv /opt/zookeeper-$VERSION /opt/zookeeper \
        && cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg \
        && mkdir -p /tmp/zookeeper

    RUN echo "[ ! -z $""ZOOKEEPER_PORT"" ] && sed -i 's%clientPort=.*$%clientPort='$""ZOOKEEPER_PORT'""%g'  /opt/zookeeper/conf/zoo.cfg" > /opt/zookeeper/start.sh &&\
    	echo "[ ! -z $""ZOOKEEPER_ID"" ]  && echo $""ZOOKEEPER_ID > /tmp/zookeeper/myid" >> /opt/zookeeper/start.sh &&\
    	echo "[[ ! -z $""ZOOKEEPER_SERVERS"" ]] && for server in $""ZOOKEEPER_SERVERS""; do echo $""server"" >> /opt/zookeeper/conf/zoo.cfg; done" >> /opt/zookeeper/start.sh &&\
    	echo "/opt/zookeeper/bin/zkServer.sh start-foreground" >> /opt/zookeeper/start.sh

    EXPOSE 2181
    WORKDIR /opt/zookeeper
    VOLUME ["/opt/zookeeper/conf", "/tmp/zookeeper"]
    ENTRYPOINT ["bash", "/opt/zookeeper/start.sh"]
    
## 4. kafka镜像制作
在之前的基础上增加BROKER_ID，BROKER_PORT，ADVERTISED_HOST_NAME，HOST_NAME，ZOOKEEPER_CONNECT参数使之动态配置。
    
    FROM java:openjdk-8-jre-alpine
    ARG MIRROR=http://mirrors.aliyun.com/
    ARG SCALA_VERSION=2.11
    ARG KAFKA_VERSION=0.8.2.2
    LABEL name="kafka" version=$VERSION
    RUN apk update && apk add ca-certificates && \
        apk add tzdata && \
        ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
        echo "Asia/Shanghai" > /etc/timezone

    RUN apk add --no-cache wget bash \
        && mkdir /opt \
        && wget -q -O - $MIRROR/apache/kafka/$KAFKA_VERSION/kafka_$SCALA_VERSION-$KAFKA_VERSION.tgz | tar -xzf - -C /opt \
        && mv /opt/kafka_$SCALA_VERSION-$KAFKA_VERSION /opt/kafka \
        && sed -i 's/num.partitions.*$/num.partitions=3/g' /opt/kafka/config/server.properties

    RUN echo "cd /opt/kafka" > /opt/kafka/start.sh &&\
    	echo "[ ! -z $""ZOOKEEPER_CONNECT"" ] && sed -i 's%zookeeper.connect=.*$%zookeeper.connect='$""ZOOKEEPER_CONNECT'""%g'  /opt/kafka/config/server.properties" >> /opt/kafka/start.sh &&\
    	echo "[ ! -z $""BROKER_ID"" ] && sed -i 's%broker.id=.*$%broker.id='$""BROKER_ID'""%g'  /opt/kafka/config/server.properties" >> /opt/kafka/start.sh &&\
    	echo "[ ! -z $""BROKER_PORT"" ] && sed -i 's%port=.*$%port='$""BROKER_PORT'""%g'  /opt/kafka/config/server.properties" >> /opt/kafka/start.sh &&\
    	echo "[ ! -z $""ADVERTISED_HOST_NAME"" ] && sed -i 's%#advertised.host.name=.*$%advertised.host.name='$""ADVERTISED_HOST_NAME'""%g'  /opt/kafka/config/server.properties" >> /opt/kafka/start.sh &&\
    	echo "[ ! -z $""HOST_NAME"" ] && sed -i 's%#host.name=.*$%host.name='$""HOST_NAME'""%g'  /opt/kafka/config/server.properties" >> /opt/kafka/start.sh &&\
    	echo "delete.topic.enable=true" >> /opt/kafka/config/server.properties &&\
    	echo "bin/kafka-server-start.sh config/server.properties" >> /opt/kafka/start.sh

    EXPOSE 9092
    WORKDIR /opt/kafka
    ENTRYPOINT ["bash", "/opt/kafka/start.sh"]
    
## 5. docker-compose文件制作
创建3个zookeeper容器和3个kafka容器。
    
    version: '2.0'
    services:
      zookeeper0:
        build:
          context: .
          dockerfile: myzookeeper.cluster.Dockerfile
        image: alex/zookeeper_cluster:3.4.6
        container_name: zookeeper0
        hostname: zookeeper0
        ports:
          - "2181:2181"
          - "2888:2888"
          - "3888:3888"
        expose:
          - 2181
          - 2888
          - 3888
        environment:
          ZOOKEEPER_PORT: 2181
          ZOOKEEPER_ID: 0
          ZOOKEEPER_SERVERS: server.0=zookeeper0:2888:3888 server.1=zookeeper1:28881:38881 server.2=zookeeper2:28882:38882
      zookeeper1:
        build:
          context: .
          dockerfile: myzookeeper.cluster.Dockerfile
        image: alex/zookeeper_cluster:3.4.6
        container_name: zookeeper1
        hostname: zookeeper1
        ports:
          - "2182:2182"
          - "28881:28881"
          - "38881:38881"
        expose:
          - 2182
          - 2888
          - 3888
        environment:
          ZOOKEEPER_PORT: 2182
          ZOOKEEPER_ID: 1
          ZOOKEEPER_SERVERS: server.0=zookeeper0:2888:3888 server.1=zookeeper1:28881:38881 server.2=zookeeper2:28882:38882
      zookeeper2:
        build:
          context: .
          dockerfile: myzookeeper.cluster.Dockerfile
        image: alex/zookeeper_cluster:3.4.6
        container_name: zookeeper2
        hostname: zookeeper2
        ports:
          - "2183:2183"
          - "28882:28882"
          - "38882:38882"
        expose:
          - 2183
          - 2888
          - 3888
        environment:
          ZOOKEEPER_PORT: 2183
          ZOOKEEPER_ID: 2
          ZOOKEEPER_SERVERS: server.0=zookeeper0:2888:3888 server.1=zookeeper1:28881:38881 server.2=zookeeper2:28882:38882
      kafka0:
        build:
          context: .
          dockerfile: mykafka.cluster.Dockerfile
        image: alex/kafka_cluster:0.8.2.2
        container_name: kafka0
        hostname: kafka0
        ports:
          - "9092:9092"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:2181,zookeeper1:2182,zookeeper2:2183/kafka
          BROKER_ID: 0
          BROKER_PORT: 9092
          ADVERTISED_HOST_NAME: kafka0
          HOST_NAME: kafka0
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        depends_on:
            - zookeeper0
            - zookeeper1
            - zookeeper2
        expose:
          - 9092
      kafka1:
        build:
          context: .
          dockerfile: mykafka.cluster.Dockerfile
        image: alex/kafka_cluster:0.8.2.2
        container_name: kafka1
        hostname: kafka1
        ports:
          - "9093:9093"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:2181,zookeeper1:2182,zookeeper2:2183/kafka
          BROKER_ID: 1
          BROKER_PORT: 9093
          ADVERTISED_HOST_NAME: kafka1
          HOST_NAME: kafka1
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        depends_on:
            - zookeeper0
            - zookeeper1
            - zookeeper2
        expose:
          - 9093
      kafka2:
        build: .
        build:
          context: .
          dockerfile: mykafka.cluster.Dockerfile
        image: alex/kafka_cluster:0.8.2.2
        container_name: kafka2
        hostname: kafka2
        ports:
          - "9094:9094"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:2181,zookeeper1:2182,zookeeper2:2183/kafka
          BROKER_ID: 2
          BROKER_PORT: 9094
          ADVERTISED_HOST_NAME: kafka2
          HOST_NAME: kafka2
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        depends_on:
            - zookeeper0
            - zookeeper1
            - zookeeper2
        expose:
          - 9094
这里要注意的是ZOOKEEPER_CONNECT参数中使用/kafka作为根目录。

## 6. 启动docker-compose
    
    docker-compose -f mykafka-compose.yml up -d
正常情况下，3个kafka容器，3个zookeeper容器启动。
![2016-10-30_23-23-15](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_23-23-15.jpg)

## 7. 验证zookeeper及目录结构
进入一个zookeeper容器使用zkCli.sh客户端命令查看。所有broker信息在kafka目录下
    
    docker exec -it zookeeper0 bash
    bin/zkCli.sh -server zookeeper0:2181
![2016-10-30_22-50-58](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_22-50-58.jpg)

如果不设置/kakfa，一般目录结构如下。
![2016-10-30_22-26-55](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_22-26-55.jpg)

## 8. 验证kafka消息发送
进入一个kafka容器
    
    docker exec -it kafka0 bash
创建一个topic，注意的是，我们之前配置的是/kafka,所以创建topic时一定要带上。
    
    bash-4.3# bin/kafka-topics.sh --zookeeper zookeeper0:2181/kafka --topic topic1 --create --replication-factor 2 --partitions 3 
    Created topic "topic1".
    bash-4.3# 
如果没有带/kafka,我们看到会直接出错。
    
    bash-4.3# bin/kafka-topics.sh --zookeeper zookeeper0:2181 --topic topic1 --create --replication-factor 2 --partitions 3          
    Error while executing topic command org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for /brokers/ids
    org.I0Itec.zkclient.exception.ZkNoNodeException: org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = 
    NoNode for /brokers/ids
    	at org.I0Itec.zkclient.exception.ZkException.create(ZkException.java:47)
    	at org.I0Itec.zkclient.ZkClient.retryUntilConnected(ZkClient.java:685)
    	at org.I0Itec.zkclient.ZkClient.getChildren(ZkClient.java:413)
    	at org.I0Itec.zkclient.ZkClient.getChildren(ZkClient.java:409)
    	at kafka.utils.ZkUtils$.getChildren(ZkUtils.scala:462)
    	at kafka.utils.ZkUtils$.getSortedBrokerList(ZkUtils.scala:78)
    	at kafka.admin.AdminUtils$.createTopic(AdminUtils.scala:170)
    	at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:93)
    	at kafka.admin.TopicCommand$.main(TopicCommand.scala:55)
    	at kafka.admin.TopicCommand.main(TopicCommand.scala)
    Caused by: org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for /brokers/ids
    	at org.apache.zookeeper.KeeperException.create(KeeperException.java:111)
    	at org.apache.zookeeper.KeeperException.create(KeeperException.java:51)
    	at org.apache.zookeeper.ZooKeeper.getChildren(ZooKeeper.java:1472)
    	at org.apache.zookeeper.ZooKeeper.getChildren(ZooKeeper.java:1500)
    	at org.I0Itec.zkclient.ZkConnection.getChildren(ZkConnection.java:99)
    	at org.I0Itec.zkclient.ZkClient$2.call(ZkClient.java:416)
    	at org.I0Itec.zkclient.ZkClient$2.call(ZkClient.java:413)
    	at org.I0Itec.zkclient.ZkClient.retryUntilConnected(ZkClient.java:675)
    	... 8 more
启动consumer也需带/kafka
    
    bin/kafka-console-consumer.sh --zookeeper zookeeper0:2181/kafka --topic topic1 --from-beginning

启动producer，发送消息，consumer端可收到。
    
    bin/kafka-console-producer.sh --broker-list kafka0:9092 --topic topic1

![2016-10-30_22-55-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_22-55-11.jpg)

再看zookeeper的topic目录，看到有新的topic1的目录结构生成。
![2016-10-30_22-56-39](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_22-56-39.jpg)