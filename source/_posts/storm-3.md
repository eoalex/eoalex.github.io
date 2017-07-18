---
title: Storm与kafka集成(上)
tags:
  - Kafka
  - Storm
id: 1619
categories:
  - 大数据
date: 2017-05-15 19:52:57
---

## 1. 简介
我们知道storm的作用主要是进行流式实时计算，对于均匀的数据流storm处理是非常有效的，但是现实生活中大部分场景并不是均匀的数据流，而是时而多时而少的数据流入，这种情况下显然用批量处理是不合适的，如果使用storm做实时计算的话可能因为数据拥堵而导致服务器挂掉，应对这种情况，使用kafka作为消息队列是非常合适的选择，kafka可以将不均匀的数据转换成均匀的消息流，从而和storm比较完善的结合，这样才可以实现稳定的流式计算，storm和kafka结合，实质上无非是把Kafka的数据消费，是由Storm去消费，通过KafkaSpout将数据输送到Storm，然后让Storm安装业务需求对接受的数据做实时处理，最后将处理后的数据输出或者保存到文件、数据库、分布式存储等等。

## 2. 搭建storm和kafka集群
我们要搭建storm和kafka集群,这里使用docker镜像。所以首先使用docker file建立镜像。
### 2.1 storm的docker file

    FROM openjdk:8-jre-alpine

    ARG MIRROR=http://mirrors.aliyun.com
    ARG BIN_VERSION=apache-storm-1.0.3

    # Install required packages
    RUN apk add --no-cache \
        bash \
        python \
        su-exec

    RUN wget -q -O - ${MIRROR}/apache/storm/${BIN_VERSION}/${BIN_VERSION}.tar.gz | tar -xzf - -C /usr/share \
    && mv /usr/share/${BIN_VERSION} /usr/share/storm \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

    WORKDIR /usr/share/storm

    # add startup script
    ADD entrypoint.sh entrypoint.sh
    ADD cluster.xml log4j2/cluster.xml
    ADD worker.xml log4j2/worker.xml
    RUN chmod +x entrypoint.sh

    # supervisor: worker ports
    EXPOSE 6700 6701 6702 6703
    # logviewer
    EXPOSE 8000
    # DRPC and remote deployment
    EXPOSE 6627 3772 3773

    ENTRYPOINT ["/usr/share/storm/entrypoint.sh"]
### 2.2 zookeeper和kafak的docker file
这里略，参见本博客关于[kafka](/2016/10/kafka-4/)的博文。
### 2.3 使用docker-compose 启动集群
    
    version: '2.0'
    services:
      zookeeper0:
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
        image: alex/kafka_cluster:0.8.2.2
        container_name: kafka0
        hostname: kafka0
        ports:
          - "9092:9092"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:2181,zookeeper1:2182,zookeeper2:2183
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
        image: alex/kafka_cluster:0.8.2.2
        container_name: kafka1
        hostname: kafka1
        ports:
          - "9093:9093"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:2181,zookeeper1:2182,zookeeper2:2183
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
        image: alex/kafka_cluster:0.8.2.2
        container_name: kafka2
        hostname: kafka2
        ports:
          - "9094:9094"
        environment:
          ZOOKEEPER_CONNECT: zookeeper0:2181,zookeeper1:2182,zookeeper2:2183
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
      nimbus:
        image: alex/storm:1.0.3
        container_name: nimbus
        command: nimbus -c nimbus.host=nimbus
        environment:
          - STORM_ZOOKEEPER_SERVERS=zookeeper0,zookeeper1,zookeeper2
        hostname: nimbus
        ports:
          - "6627:6627"
      ui:
        image: alex/storm:1.0.3
        container_name: ui
        command: ui -c nimbus.host=nimbus
        environment:
          - STORM_ZOOKEEPER_SERVERS=zookeeper0,zookeeper1,zookeeper2
        hostname: ui
        ports:
          - "8080:8080"
        depends_on:
              - nimbus
      supervisor1:
        image: alex/storm:1.0.3
        container_name: supervisor1
        command: supervisor -c nimbus.host=nimbus -c supervisor.slots.ports=[6700,6701,6702,6703]
        environment:
          - STORM_ZOOKEEPER_SERVERS=zookeeper0,zookeeper1,zookeeper2
        hostname: supervisor1
        ports:
          - "8000:8000"
        depends_on:
              - nimbus