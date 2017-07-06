---
title: 'Kafka入门之一:使用Docker安装Kafka和Zookeeper'
tags:
  - Kafka
  - zookeeper
id: 1259
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-09-25 09:41:33
---

## 1. 介绍
Kafka是一个分布式的、可分区的、可复制的提交日志服务。它提供了消息系统功能，但具有自己独特的设计。
几个基本的消息系统术语：
* **Broker：**Kafka集群包含一个或多个服务器，这种服务器被称为broker。
* **Topic:**每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。（物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处）
* **Partition:**Partition是物理上的概念，每个Topic包含一个或多个Partition.
* **Producer:**负责发布消息到Kafka broker
* **Consumer:**消息消费者，向Kafka broker读取消息的客户端。
* **Consumer Group:**每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。
producers通过网络将消息发送到Kafka集群，集群向消费者提供消息，如下图所示：
![producer_consumer](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/producer_consumer.png)

**Topics主题和Logs 日志**
先来看一下Kafka提供的一个抽象概念:topic.
一个topic是对一组消息的归纳。对每个topic，Kafka 对它的日志进行了分区，如下图所示：
![log_anatomy](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/log_anatomy.png)

每个分区都由一系列有序的、不可变的消息组成，这些消息被连续的追加到分区中。分区中的每个消息都有一个连续的序列号叫做offset,用来在分区中唯一的标识这个消息。
在一个可配置的时间段内，Kafka集群保留所有发布的消息，不管这些消息有没有被消费。比如，如果消息的保存策略被设置为2天，那么在一个消息被发布的两天时间内，它都是可以被消费的。之后它将被丢弃以释放空间。Kafka的性能是和数据量大小是无关的，所以保留太多的数据并不是问题。
实际上每个consumer唯一需要维护的数据是消息在日志中的位置，也就是offset.这个offset有consumer来维护：一般情况下随着consumer不断的读取消息，这offset的值不断增加，但其实consumer可以以任意的顺序读取消息，比如它可以将offset设置成为一个旧的值来重读之前的消息。
以上特点的结合，使Kafka consumers非常的轻量级：它们可以在不对集群和其他consumer造成影响的情况下读取消息。你可以使用命令行来"tail"消息而不会对其他正在消费消息的consumer造成影响。
将日志分区可以达到以下目的：首先这使得每个日志的数量不会太大，可以在单个服务上保存。另外每个分区可以单独发布和消费，为并发操作topic提供了一种可能。
**Distribution分布式**
每个分区在Kafka集群的若干服务中都有副本，这样这些持有副本的服务可以共同处理数据和请求，副本数量是可以配置的。副本使Kafka具备了容错能力。
每个分区都由一个服务器作为“leader”，零或若干服务器作为“followers”,leader负责处理消息的读和写，followers则去复制leader.如果leader down了，followers中的一台则会自动成为leader。集群中的每个服务都会同时扮演两个角色：作为它所持有的一部分分区的leader，同时作为其他分区的followers，这样集群就会据有较好的负载均衡。
**Producers生产者**
Producer将消息发布到它指定的topic中,并负责决定发布到哪个分区。通常简单的由负载均衡机制随机选择分区，但也可以通过特定的分区函数选择分区。使用的更多的是第二种。
**Consumers消费者**
发布消息通常有两种模式：队列模式（queuing）和发布-订阅模式(publish-subscribe)。队列模式中，consumers可以同时从服务端读取消息，每个消息只被其中一个consumer读到；发布-订阅模式中消息被广播到所有的consumer中。Consumers可以加入一个consumer 组，共同竞争一个topic，topic中的消息将被分发到组中的一个成员中。同一组中的consumer可以在不同的程序中，也可以在不同的机器上。如果所有的consumer都在一个组中，这就成为了传统的队列模式，在各consumer中实现负载均衡。如果所有的consumer都不在不同的组中，这就成为了发布-订阅模式，所有的消息都被分发到所有的consumer中。更常见的是，每个topic都有若干数量的consumer组，每个组都是一个逻辑上的“订阅者”，为了容错和更好的稳定性，每个组由若干consumer组成。这其实就是一个发布-订阅模式，只不过订阅者是个组而不是单个consumer。
![consumer-groups](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/consumer-groups.png)

由两个机器组成的集群拥有4个分区 (P0-P3) 2个consumer组. A组有两个consumerB组有4个。
相比传统的消息系统，Kafka可以很好的保证有序性。
传统的队列在服务器上保存有序的消息，如果多个consumers同时从这个服务器消费消息，服务器就会以消息存储的顺序向consumer分发消息。虽然服务器按顺序发布消息，但是消息是被异步的分发到各consumer上，所以当消息到达时可能已经失去了原来的顺序，这意味着并发消费将导致顺序错乱。为了避免故障，这样的消息系统通常使用“专用consumer”的概念，其实就是只允许一个消费者消费消息，当然这就意味着失去了并发性。
在这方面Kafka做的更好，通过分区的概念，Kafka可以在多个consumer组并发的情况下提供较好的有序性和负载均衡。将每个分区分只分发给一个consumer组，这样一个分区就只被这个组的一个consumer消费，就可以顺序的消费这个分区的消息。因为有多个分区，依然可以在多个consumer组之间进行负载均衡。注意consumer组的数量不能多于分区的数量，也就是有多少分区就允许多少并发消费。
Kafka只能保证一个分区之内消息的有序性，在不同的分区之间是不可以的，这已经可以满足大部分应用的需求。如果需要topic中所有消息的有序性，那就只能让这个topic只有一个分区，当然也就只有一个consumer组消费它。

## 2. 实例演示
这里我们使用Docker 来分别安装演示Kafka和Zookeeper。
### 2.1 制作dockerfile
#### 2.1.1 Zookeeper
首先制作Zookeeper的dockerfile，这里采用java:openjdk-8-jre-alpine作为源镜像，zookeeper版本3.4.6，使用国内镜像源阿里云。

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

    EXPOSE 2181

    WORKDIR /opt/zookeeper

    VOLUME ["/opt/zookeeper/conf", "/tmp/zookeeper"]

    ENTRYPOINT ["/opt/zookeeper/bin/zkServer.sh"]
    CMD ["start-foreground"]
#### 2.1.2 Kafka
同样Kafka的dockerfile 如下,kafka版本采用0.8.2.2
    
    FROM java:openjdk-8-jre-alpine

    ARG MIRROR=http://mirrors.aliyun.com/
    ARG SCALA_VERSION=2.11
    ARG KAFKA_VERSION=0.8.2.2

    LABEL name="kafka" version=$VERSION

    RUN apk update && apk add ca-certificates && \
        apk add tzdata && \
        ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
        echo "Asia/Shanghai" &gt; /etc/timezone

    RUN apk add --no-cache wget bash \
        && mkdir /opt \
        && wget -q -O - $MIRROR/apache/kafka/$KAFKA_VERSION/kafka_$SCALA_VERSION-$KAFKA_VERSION.tgz | tar -xzf - -C /opt \
        && mv /opt/kafka_$SCALA_VERSION-$KAFKA_VERSION /opt/kafka \
    	&& sed -i 's/num.partitions.*$/num.partitions=3/g' /opt/kafka/config/server.properties \
    	&& sed -i 's/zookeeper.connect=.*$/zookeeper.connect=zookeeper:2181/g'  /opt/kafka/config/server.properties 

    EXPOSE 9092

    ENTRYPOINT ["/opt/kafka/bin/kafka-server-start.sh"]
    CMD ["/opt/kafka/config/server.properties"]
    
### 2.2 Build dockerfile
分别build zookeeper和kafka，build过程略，因为采用了国内镜像源，速度还是比较快的。
    
    sudo docker build -f zookeeper.Dockerfile -t alex/zookeeper:3.4.6 .
    sudo docker build -f kafka.Dockerfile -t alex/kafka:0.8.2.2 .
完成的镜像文件如下，我们看到由于采用了java:openjdk-8-jre-alpine，整个镜像还是比较小的。
![2016-09-25_7-42-04](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_7-42-04.jpg)

### 2.3 启动 zookeeper
    sudo docker run --name zookeeper -itd -p2181:2181 alex/zookeeper:3.4.6
    
### 2.4 启动 kafka
这里要注意link zookeeper容器。
	
	sudo docker run --name kafka -itd -p9092:9092 --link zookeeper alex/kafka:0.8.2.2
检查端口是否都已启动。
![2016-09-25_8-56-19](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_8-56-19.jpg)
如果没有看到，你可能要使用以下命令检查容器启动的logs

    sudo docker logs zookeeper
    sudo docker logs kafka
### 2.5 验证1
进入kafka容器，创建两个topics，分别叫test1，test2
    
    sudo docker exec -it kafka bash
    cd /opt/kafka
    source /root/.bash_profile
    bin/kafka-topics.sh --create --topic test1 --zookeeper zookeeper:2181 --partition 3 --replication-factor 1
    bin/kafka-topics.sh --create --topic test2 --zookeeper zookeeper:2181 --partition 3 --replication-factor 1
    bin/kafka-topics.sh --describe --topic test1 --zookeeper zookeeper:2181
    bin/kafka-topics.sh --describe --topic test2 --zookeeper zookeeper:2181
    bin/kafka-topics.sh --list --zookeeper zookeeper:2181`</pre>
![2016-09-25_7-16-28](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_7-16-28.jpg)
启动consumer,以后我把这里叫做consumer端，以示区分。
	
	 bin/kafka-console-consumer.sh --zookeeper zookeeper:2181 --topic test1
另外开启一个终端(producer端)，进入kafka容器，启动producer，并发送几条消息。
    
    sudo docker exec -it kafka bash
    cd /opt/kafka
    source /root/.bash_profile
    bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test1
![2016-09-25_7-18-07](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_7-18-07.jpg)
我们看到cunsumer端，接收到了这些消息。
![2016-09-25_7-18-34](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_7-18-34.jpg)

### 2.6 验证2
接下来我们将演示消息转发的功能，将我们刚才在topic test1输入的消息转发给topic test2
在cunsumer端，启动接收test2
    
    bin/kafka-console-consumer.sh --zookeeper zookeeper:2181 --topic test2
在producer端，使用kafka-replay-log-producer.sh将test1的消息转发给test2
    
    bin/kafka-replay-log-producer.sh --broker-list localhost:9092 --zookeeper zookeeper:2181 --inputtopic test1 --outputtopic test2

![2016-09-25_7-20-06](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_7-20-06.jpg)

![2016-09-25_7-19-46](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/09/2016-09-25_7-19-46.jpg)

我们看到cunsumer的test2成功接收到了之前test1的消息，而我刚才并没有重发这些消息，这也验证了kafka的消息是持久化的。

参考
[Jason老师的博客](http://www.jasongj.com/tags/Kafka/)
[kafka官网](http://kafka.apache.org/documentation.html)
[Kafka入门经典教程](http://www.aboutyun.com/thread-12882-1-1.html)