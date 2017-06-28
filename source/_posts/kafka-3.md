---
title: 'Kafka入门之三:Kafka集群安装及演示'
tags:
  - Docker
  - Kafka
id: 1332
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-10-22 23:49:09
---

1\. 简介
在[Kafka入门之一:使用Docker安装Kafka和Zookeeper](http://blog.yaodataking.com/2016/09/kafka-1.html)里我们已经演示使用docker安装单节点的Kafka，也就是一个broker。但实际上kafka是天生支持多broker的。在安装之前，我们先来看几个broker参数。 更多参数参见[官网](http://kafka.apache.org/082/documentation.html#brokerconfigs)
<table>
     <tbody>
         <tr>
             <td width="300" height="">Property</td>
             <td width="200" height="">Default</td>
             <td width="800" height="">Description</td>
         </tr>
         <tr>
             <td>broker.id</td>
             <td>&nbsp;</td>             <td>每个broker都可以用一个唯一的非负整数id进行标识；这个id可以作为broker的&#8220;名字&#8221;，并且它的存在使得broker无须混淆consumers就可以迁移到不同的host/port上。你可以选择任意你喜欢的数字作为id，只要id是唯一的即可。</td>
         </tr>
         <tr>
             <td>log.dirs</td>
             <td>/tmp/kafka-logs</td>             <td>kafka存放数据的路径。这个路径并不是唯一的，可以是多个，路径之间只需要使用逗号分隔即可；每当创建新partition时，都会选择在包含最少partitions的路径下进行。</td>
         </tr>
         <tr>
             <td>port</td>
             <td>9092</td>
             <td>server接受客户端连接的端口</td>
         </tr>
         <tr>
             <td>zookeeper.connect</td>
             <td>null</td>
             <td>ZooKeeper连接字符串的格式为：hostname:port，此处hostname和port分别是ZooKeeper集群中某个节点的host和port；为了当某个host宕掉之后你能通过其他ZooKeeper节点进行连接，你可以按照一下方式制定多个hosts：hostname1:port1, hostname2:port2, hostname3:port3.
ZooKeeper 允许你增加一个&#8220;chroot&#8221;路径，将集群中所有kafka数据存放在特定的路径下。当多个Kafka集群或者其他应用使用相同ZooKeeper集群时，可以使用这个方式设置数据存放路径。这种方式的实现可以通过这样设置连接字符串格式，如下所示：          hostname1：port1，hostname2：port2，hostname3：port3/chroot/path

             这样设置就将所有kafka集群数据存放在/chroot/path路径下。注意，在你启动broker之前，你必须创建这个路径，并且consumers必须使用相同的连接格式。</td>
         </tr>
         <tr>
             <td>message.max.bytes</td>
             <td>1000000</td>
             <td>server可以接收的消息最大尺寸。重要的是，consumer和producer有关这个属性的设置必须同步，否则producer发布的消息对consumer来说太大。</td>
         </tr>
         <tr>
             <td>num.network.threads</td>
             <td>3</td>
             <td>server用来处理网络请求的网络线程数目；一般你不需要更改这个属性。</td>
         </tr>
         <tr>
             <td>num.io.threads</td>
             <td>8</td>
             <td>server用来处理请求的I/O线程的数目；这个线程数目至少要等于硬盘的个数。</td>
         </tr>
         <tr>
             <td>background.threads</td>
             <td>4</td>
             <td>用于后台处理的线程数目，例如文件删除；你不需要更改这个属性。</td>
         </tr>
         <tr>
             <td>queued.max.requests</td>
             <td>500</td>
             <td>在网络线程停止读取新请求之前，可以排队等待I/O线程处理的最大请求个数。</td>
         </tr>
         <tr>
             <td>host.name</td>
             <td>null</td>
             <td>broker的hostname；如果hostname已经设置的话，broker将只会绑定到这个地址上；如果没有设置，它将绑定到所有接口，并发布一份到ZK</td>
         </tr>
         <tr>
             <td>advertised.host.name</td>
             <td>null</td>
             <td>如果设置，则就作为broker 的hostname发往producer、consumers以及其他brokers</td>
         </tr>
         <tr>
             <td>advertised.port</td>
             <td>null</td>
             <td>此端口将给与producers、consumers、以及其他brokers，它会在建立连接时用到； 它仅在实际端口和server需要绑定的端口不一样时才需要设置。</td>
         </tr>
         <tr>
             <td>num.partitions</td>
             <td>1</td>
             <td>如果创建topic时没有给出划分partitions个数，这个数字将是topic下partitions数目的默认数值。</td>
         </tr>
         <tr>
             <td>auto.create.topics.enable</td>
             <td>true</td>
             <td>是否允许自动创建topic。如果是真的，则produce或者fetch 不存在的topic时，会自动创建这个topic。否则需要使用命令行创建topic</td>
         </tr>
         <tr>
             <td>default.replication.factor</td>
             <td>1</td>
             <td>默认备份份数，仅指自动创建的topics</td>
         </tr>
         <tr>
             <td>replica.lag.time.max.ms</td>
             <td>10000</td>
             <td>如果一个follower在这个时间内没有发送fetch请求，leader将从ISR重移除这个follower，并认为这个follower已经挂了</td>
         </tr>
         <tr>
             <td>replica.lag.max.messages</td>
             <td>4000</td>
             <td>如果一个replica没有备份的条数超过这个数值，则leader将移除这个follower，并认为这个follower已经挂了</td>
         </tr>
         <tr>
             <td>replica.socket.timeout.ms</td>
             <td>30*1000</td>
             <td>leader 备份数据时的socket网络请求的超时时间</td>
         </tr>
         <tr>
             <td>replica.socket.receive.buffer.bytes</td>
             <td>64*1024</td>
             <td>备份时向leader发送网络请求时的socket receive buffer</td>
         </tr>
         <tr>
             <td>replica.fetch.max.bytes</td>
             <td>1024*1024</td>
             <td>备份时每次fetch的最大值</td>
         </tr>
         <tr>
             <td>replica.fetch.min.bytes</td>
             <td>500</td>
             <td>leader发出备份请求时，数据到达leader的最长等待时间</td>
         </tr>
         <tr>
             <td>replica.fetch.min.bytes</td>
             <td>1</td>
             <td>备份时每次fetch之后回应的最小尺寸</td>
         </tr>
         <tr>
             <td>num.replica.fetchers</td>
             <td>1</td>
             <td>从leader备份数据的线程数</td>
         </tr>
         <tr>
             <td>replica.high.watermark.checkpoint.interval.ms </td>
             <td>5000</td>
             <td>每个replica检查是否将最高水位进行固化的频率</td>
         </tr>
         <tr>
             <td>zookeeper.session.timeout.ms</td>
             <td>6000</td>
             <td>zookeeper会话超时时间。</td>
         </tr>
         <tr>
             <td>zookeeper.connection.timeout.ms</td>
             <td>6000</td>
             <td>客户端等待和zookeeper建立连接的最大时间</td>
         </tr>
         <tr>
             <td>zookeeper.sync.time.ms</td>
             <td>2000</td>
             <td>zk follower落后于zk leader的最长时间</td>
         </tr>
          <tr>
             <td>auto.leader.rebalance.enable</td>
             <td>true</td>
             <td>如果这是true，控制者将会自动平衡brokers对于partitions的leadership</td>
         </tr>
         <tr>
             <td>leader.imbalance.per.broker.percentage</td>
             <td>10</td>
             <td>每个broker所允许的leader最大不平衡比率</td>
         </tr>
         <tr>
             <td>leader.imbalance.check.interval.seconds</td>
             <td>300</td>
             <td>检查leader不平衡的频率</td>
         </tr>
         <tr>
             <td>offset.metadata.max.bytes</td>
             <td>4096</td>
             <td>允许客户端保存他们offsets的最大个数</td>
         </tr>
         <tr>
             <td>max.connections.per.ip</td>
             <td>Int.MaxValue</td>
             <td>每个ip地址上每个broker可以被连接的最大数目</td>
         </tr>
         <tr>
             <td>max.connections.per.ip.overrides</td>
             <td>&nbsp;</td>
             <td>每个ip或者hostname默认的连接的最大覆盖</td>
         </tr>
         <tr>
             <td>connections.max.idle.ms</td>
             <td>600000</td>
             <td>空连接的超时限制</td>
         </tr>
         <tr>
             <td>log.roll.jitter.{ms,hours}</td>
             <td>0</td>
             <td>从logRollTimeMillis抽离的jitter最大数目</td>
         </tr>
         <tr>
             <td>num.recovery.threads.per.data.dir</td>
             <td>1</td>
             <td>每个数据目录用来日志恢复的线程数目</td>
         </tr>
         <tr>
             <td>unclean.leader.election.enable</td>
             <td>true</td>
             <td>指明了是否能够使不在ISR中replicas设置用来作为leader</td>
         </tr>
         <tr>
             <td>delete.topic.enable</td>
             <td>false</td>
             <td>能够删除topic</td>
         </tr>
      </tbody>
</table>
这些参数在你还没有了解具体用途之前，你都可以默认，今天我们将会使用几个参数。
2\. Kafka集群
2.1 Dockerfile
我们在dockeefile中加入一些broker的参数，即在server.properites文件中设置的参数。

    FROM java:openjdk-8-jre-alpine
    ARG MIRROR=http://mirrors.aliyun.com/
    ARG SCALA_VERSION=2.11
    ARG KAFKA_VERSION=0.8.2.2
    LABEL name="kafka" version=$VERSION
    RUN apk update &amp;&amp; apk add ca-certificates &amp;&amp; \
        apk add tzdata &amp;&amp; \
        ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &amp;&amp; \
        echo "Asia/Shanghai" &gt; /etc/timezone
    RUN apk add --no-cache wget bash \
        &amp;&amp; mkdir /opt \
        &amp;&amp; wget -q -O - $MIRROR/apache/kafka/$KAFKA_VERSION/kafka_$SCALA_VERSION-$KAFKA_VERSION.tgz | tar -xzf - -C /opt \
        &amp;&amp; mv /opt/kafka_$SCALA_VERSION-$KAFKA_VERSION /opt/kafka \
        &amp;&amp; sed -i 's/num.partitions.*$/num.partitions=3/g' /opt/kafka/config/server.properties

    RUN echo "cd /opt/kafka" &gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "sed -i 's%zookeeper.connect=.*$%zookeeper.connect=zookeeper:2181%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "[ ! -z $""BROKER_ID"" ] &amp;&amp; sed -i 's%broker.id=.*$%broker.id='$""BROKER_ID'""%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "[ ! -z $""BROKER_PORT"" ] &amp;&amp; sed -i 's%port=.*$%port='$""BROKER_PORT'""%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "sed -i 's%#advertised.host.name=.*$%advertised.host.name='$""(hostname -i)'""%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "[ ! -z $""ADVERTISED_HOST_NAME"" ] &amp;&amp; sed -i 's%.*advertised.host.name=.*$%advertised.host.name='$""ADVERTISED_HOST_NAME'""%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "sed -i 's%#host.name=.*$%host.name='$""(hostname -i)'""%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "[ ! -z $""HOST_NAME"" ] &amp;&amp; sed -i 's%.*host.name=.*$%host.name='$""HOST_NAME'""%g'  /opt/kafka/config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	echo "delete.topic.enable=true" &gt;&gt; /opt/kafka/config/server.properties &amp;&amp;\
    	echo "bin/kafka-server-start.sh config/server.properties" &gt;&gt; /opt/kafka/start.sh &amp;&amp;\
    	chmod a+x /opt/kafka/start.sh

    EXPOSE 9092
    WORKDIR /opt/kafka
    ENTRYPOINT ["sh", "/opt/kafka/start.sh"]`</pre>
    2.2启动zookeeper
    为简单起见，zookeeper我们暂时使用单个容器。
    <pre>`sudo docker build -f zookeeper.Dockerfile -t alex/zookeeper:3.4.6 .`</pre>
    2.3启动Kafka集群
    使用下面命令启动三个Kafka容器，分别是kafka0，kafka1，kafka2，我们看到启动容器时传入了BROKER_ID，BROKER_PORT等变量。这就可以很方便的启动多个容器。
    <pre>`docker run -itd --name kafka0 -h kafka0 -p9092:9092 -e BROKER_ID=0 -e BROKER_PORT=9092 --link zookeeper alex/kafka_cluster:0.8.2.2
    docker run -itd --name kafka1 -h kafka1 -p9093:9092 -e BROKER_ID=1 -e BROKER_PORT=9092 --link zookeeper alex/kafka_cluster:0.8.2.2
    docker run -itd --name kafka2 -h kafka2 -p9094:9092 -e BROKER_ID=2 -e BROKER_PORT=9092 --link zookeeper alex/kafka_cluster:0.8.2.2`</pre>
    3.验证是否集群正常
    进入kafka0容器内, 创建topic1
    <pre>`docker exec -it kafka0 bash
    bin/kafka-topics.sh --zookeeper zookeeper:2181 --topic topic1 --create --replication-factor 2 --partitions 3
    bin/kafka-topics.sh --zookeeper zookeeper:2181 --topic topic1 --describe `</pre>
    [![2016-10-22_21-17-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_21-17-05.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_21-17-05.jpg)
    启动consumer
    <pre>`bin/kafka-console-consumer.sh --zookeeper zookeeper:2181 --topic topic1`</pre>
    进入kafka2容器内,  启动producer
    <pre>`bin/kafka-console-producer.sh --broker-list kafka2:9092 --topic topic1`</pre>
    [![2016-10-22_21-16-06](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_21-16-06.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_21-16-06.jpg)
    发送一些消息，我看到consumer端正常接收。
    [![2016-10-22_21-16-26](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_21-16-26.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_21-16-26.jpg)
    这就说明集群配置成功。

    4.验证min.insync.replicas和request.required.acks的作用。
    我们创建topic2，并将min.insync.replicas设为2，partitions为1。
    <pre>`bin/kafka-topics.sh --zookeeper zookeeper:2181 --topic topic2 --create --config min.insync.replicas=2 --replication-factor 2 --partitions 1
    bin/kafka-topics.sh --zookeeper zookeeper:2181 --topic topic2 --describe `</pre>
    [![2016-10-22_23-26-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_23-26-17.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_23-26-17.jpg)
    我们看到topic2分配在了kafka2上，并且kafka0是备份，ISR是0和2,现在我们将kafka1和kafka2的broker关闭。我们看到leader已自动转为broker0。然后启动consumer
    <pre>`bin/kafka-console-consumer.sh --zookeeper zookeeper:2181 --topic topic2`</pre>
    在另一个终端上启动producer,并将request-required-acks设置为-1<pre>`bin/kafka-console-producer.sh --broker-list kafka0:9092 --topic topic2 --request-required-acks -1

[![2016-10-22_23-35-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_23-35-05.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_23-35-05.jpg)
我们看到此时发送消息，kafka出现错误。原因是我们在producer发送时设置了检查replicas是否都收到数据的确认，但是因为我们已经关闭broker2, 所以尝试发送3次未果后，producer返回错误。
显然，我们将producer以默认设置发送消息时，consumer马上收到消息了。
[![2016-10-22_22-02-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_22-02-59.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-22_22-02-59.jpg)