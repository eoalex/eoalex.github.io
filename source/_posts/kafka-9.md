---
title: 'Kafka入门之九:Kafka Streams'
tags:
  - Kafka
id: 1478
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-12-09 23:12:43
---

## 1. 概述
Kafka Streams是一个客户端程序库，用于处理和分析存储在Kafka中的数据，并将得到的数据写入kafka或发送到外部系统。Kafka Stream中有几个重要的流处理概念：Event time和Process Time、窗口函数、应用状态管理。Kafka Stream的门槛非常低：比如单机进行一些小数据量的功能验证而不需要在其他机器上启动一些服务（比如在Storm运行Topology需要启动Nimbus和Supervisor，当然也支持Local Mode），Kafka Stream的并发模型可以对单应用多实例进行负载均衡。
## 2. 主要特点
* 轻量的嵌入到Java应用中
* 除了Kafka Stream Client lib以外无外部依赖
* 支持本地状态故障转移，以实现非常高效的有状态操作，如join和window函数
* 低延迟消息处理，支持基于event-time的window操作
* 提供必要的流处理原语、hige-level Stream DSL和low-level Processor API

## 3. 开发者指南
### 3.1 核心概念
#### 3.1.1 Stream Processing Topology
* stream是Kafka Stream最重要的抽象，它代表了一个无限持续的数据集。stream是有序的、可重放消息、对不可变数据集支持故障转移
* 一个stream processing application由一到多个processor topologies组成，其中每个processor topology是一张图，由多个streams（edges）连接着多个stream processor（node）
* 一个stream processor是processor topology中的一个节点，它代表一个在stream中的处理步骤：从上游processors接受数据、进行一些处理、最后发送一到多条数据到下游processors

Kafka Stream提供两种开发stream processing topology的API
* high-level  Stream DSL：提供通用的数据操作，如map和fileter
* lower-level Processor API：提供定义和连接自定义processor，同时跟state store（下文会介绍）交互

#### 3.1.2 Time
Time在流处理中时间是一个比较重要的概念，比如说在窗口（windows）处理中，时间就代表两个处理边界
* Event time：一条消息最初产生/创建的时间点
* Processing time：消息准备被应用处理的时间点，如kafka消费某条消息的时间，processing time的单位可以是毫秒、小时或天。Processing time晚于Event time.
* Ingestion Time 消息存入Topic/Partition时的时间

Kafka Stream 使用TimestampExtractor 接口为每个消息分配一个timestamp，具体的实现可以是从消息中的某个时间字段获取timestamp以提供event-time的语义或者返回处理时的时钟时间，从而将processing-time的语义留给开发者的处理程序。开发者甚至可以强制使用其他不同的时间概念来进行定义event-time和processing time。

#### 3.1.3 States
* In-memory State Store
* Persistent State Store

Kafka Stream使用state stores提供基于stream的数据存储和数据查询，Kafka Stream内嵌了多个state store，可以通过API访问到，这些state store的实现可以是持久化的KV存储引擎、内存HashMap或者其他数据结构。Kafka Stream提供了local state store的故障转移和自动发现。

#### 3.1.4 Windows
* Hopping time windows
* Tumbling time windows:Hopping time windows的特例
* Sliding windows:只用于Join操作，可由JoinWindow类指定

### 3.2 KStream vs KTable
#### 3.2.1 概念
Kafka Stream定义了两种基本抽象：KStream 和 KTable，区别来自于key-value对值如何被解释，在一个流中(KStream)，每个key-value是一个独立的信息片断，比如，用户购买流是：alice->黄油，bob->面包，alice->奶酪面包，我们知道alice既买了黄油，又买了奶酪面包。
另一方面，对于一个表table( KTable)，是代表一个变化日志，如果表包含两对同样key的key-value值，后者会覆盖前面的记录，因为key值一样的，比如用户地址表：alice -> 纽约, bob -> 旧金山, alice -> 芝加哥，意味着Alice从纽约迁移到芝加哥，而不是同时居住在两个地方。
这两个概念之间有一个二元性，一个流能被看成表，而一个表也可以看成流。
* KStream:KStream为数据流，每条消息代表一条不可变的新记录.
* KTable:KTable为change log流，每条消息代表一个更新，几条key相同的消息会将该key的值更新为最后一条消息的值.

#### 3.2.2 Example
对于KStream和KTable中插入两条消息 (“key1”, 1), (“key1”, 2) 
对KStream作sum，结果为(“key1”,3)
对KTable作sum，结果为(“key1”,2)

#### 3.2.3 几种Join
1. KStream和KStream的Join，适用于Window Join，结果为KStream。
1. KStream和KTable的Join，KTable的变化只影响KStream中新数据，新结果的输出由KStream驱动，输出为KStream。
1. KTable和KTable的Join，类似于RDBMS的Join，结果为KTable。

## 3.3 Stream API
#### 3.3.1 Low-level processor API 
通过实现Processor接口并实现process和punctuate方法，每条消息都会调用process方法，punctuate方法会周期性的被调用。使用TopologyBuilder拼装processor。

#### 3.3.2 High-level DSL API
使用Stream DSL创建processor topology，开发者可以使用KStreamBuilder类，继承自TopologyBuilder。

## 4. 一个示例
在[PurchaseAnalysis.java](https://github.com/habren/KafkaExample/tree/master/demokafka.0.10.1.0/src/main/java/com/jasongj/kafka/stream)的基础上，算出不同地区（用户地址），不同性别的订单数及商品总数和总金额。输出结果schema如下 
地区（用户地区，如SH），性别，订单总数，商品总数，总金额
示例输出
SH, male, 3, 4, 188888.88
BJ, femail, 5, 8, 288888.88
首先我们要定义OrderAddressGenderSum类，

```java
private String userAddress;
private String gender;
private int orderCount;
private int itemSum;
private double orderAmount;
```		
增加fromOrderUserItem及add方法。

```java
public static OrderAddressGenderSum fromOrderUserItem(OrderUserItem orderUserItem) {
			OrderAddressGenderSum orderAddressGenderSum = new OrderAddressGenderSum();
			if(orderUserItem == null) {
				return orderAddressGenderSum;
			}
			orderAddressGenderSum.userAddress = orderUserItem.userAddress;
			orderAddressGenderSum.gender = orderUserItem.gender;
			orderAddressGenderSum.orderCount = 1;
			orderAddressGenderSum.itemSum = orderUserItem.quantity;
			orderAddressGenderSum.orderAmount = orderUserItem.quantity * orderUserItem.itemPrice;
			return orderAddressGenderSum;
		}

public static OrderAddressGenderSum add(OrderAddressGenderSum v1, OrderAddressGenderSum v2) {
			OrderAddressGenderSum orderAddressGenderSum = new OrderAddressGenderSum();
			orderAddressGenderSum.userAddress = v1.userAddress;
			orderAddressGenderSum.gender = v1.gender;
			orderAddressGenderSum.orderCount = v1.orderCount + v2.orderCount;
			orderAddressGenderSum.itemSum = v1.itemSum + v2.itemSum;
			orderAddressGenderSum.orderAmount = v1.orderAmount + v2.orderAmount;
            return orderAddressGenderSum;
        }
```

定义一个新的streamBuilder，Order使用KStream流，User，Item使用KTable,使用join关联。主要代码如下。

```java
KStreamBuilder streamBuilder = new KStreamBuilder();
		KStream<String, Order> orderStream = streamBuilder.stream(Serdes.String(), SerdesFactory.serdFrom(Order.class), "orders");
		KTable<String, User> userTable = streamBuilder.table(Serdes.String(), SerdesFactory.serdFrom(User.class), "users", "users-state-store");
		KTable<String, Item> itemTable = streamBuilder.table(Serdes.String(), SerdesFactory.serdFrom(Item.class), "items", "items-state-store");
		KTable<String, OrderAddressGenderSum> kTable = orderStream
				.leftJoin(userTable, (Order order, User user) -> OrderUser.fromOrderUser(order, user), Serdes.String(), SerdesFactory.serdFrom(Order.class))
				.filter((String userName, OrderUser orderUser) -> orderUser.userAddress != null)
				.map((String userName, OrderUser orderUser) -> new KeyValue<String, OrderUser>(orderUser.itemName, orderUser))
				.through(Serdes.String(), SerdesFactory.serdFrom(OrderUser.class), (String key, OrderUser orderUser, int numPartitions) -> (orderUser.getItemName().hashCode() & 0x7FFFFFFF) % numPartitions, "orderuser-repartition-by-item")
				.leftJoin(itemTable, (OrderUser orderUser, Item item) ->OrderUserItem.fromOrderUser(orderUser, item), Serdes.String(),SerdesFactory.serdFrom(OrderUser.class))
		        .map((String item, OrderUserItem orderUserItem) -> KeyValue.<String, OrderAddressGenderSum>pair(orderUserItem.userAddress + orderUserItem.gender,OrderAddressGenderSum.fromOrderUserItem(orderUserItem)))
		        .groupByKey(Serdes.String(), SerdesFactory.serdFrom(OrderAddressGenderSum.class))
		        .reduce((OrderAddressGenderSum v1, OrderAddressGenderSum v2) -> OrderAddressGenderSum.add(v1, v2),"gender-amount-state-store");			
		kTable.foreach((key, orderAddressGenderSum) -> System.out.printf("%s\n", orderAddressGenderSum.toString()));
        kTable
        	.toStream()
        	.map((String key, OrderAddressGenderSum orderAddressGenderSum) -> new KeyValue<String, String>(key,orderAddressGenderSum.printSelf()))
        	.to("gender-amount");
		KafkaStreams kafkaStreams = new KafkaStreams(streamBuilder, props);
		kafkaStreams.cleanUp();
		kafkaStreams.start();		
		System.in.read();
		kafkaStreams.close();
		kafkaStreams.cleanUp();
```

最终结果如下。
![2016-12-09_22-43-20](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/12/2016-12-09_22-43-20.png)

>注：此示例基于[harben的GitHub](https://github.com/habren/KafkaExample)上的Purchase Analysis示例修改。