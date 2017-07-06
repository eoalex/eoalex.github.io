---
title: 'Kafka入门之六:Kafka的Consumer'
tags:
  - Kafka
id: 1406
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-11-13 09:46:44
---

## **1. 几个概念**
### **1.1 High Level Consumer**
　　很多时候，客户程序只是希望从Kafka读取数据，不太关心消息offset的处理。同时也希望提供一些语义，例如同一条消息只被某一个Consumer消费（单播）或被所有Consumer消费（广播）。因此，Kafka Hight Level Consumer提供了一个从Kafka消费数据的高层抽象，从而屏蔽掉其中的细节并提供丰富的语义。
### **1.2 Consumer Group**
　　High Level Consumer将从某个Partition读取的最后一条消息的offset存于Zookeeper中(Kafka从0.8.2版本开始同时支持将offset存于Zookeeper中与将offset存于专用的Kafka Topic中)。这个offset基于客户程序提供给Kafka的名字来保存，这个名字被称为Consumer Group。Consumer Group是整个Kafka集群全局的，而非某个Topic的。每一个High Level Consumer实例都属于一个Consumer Group，若不指定则属于默认的Group。
### **1.3 Consumer Rebalance(基于Hight Level Consumer API)**
　　Kafka保证同一Consumer Group中只有一个Consumer会消费某条消息，实际上，Kafka保证的是稳定状态下每一个Consumer实例只会消费某一个或多个特定Partition的数据，而某个Partition的数据只会被某一个特定的Consumer实例所消费。也就是说Kafka对消息的分配是以Partition为单位分配的，而非以每一条消息作为分配单元。这样设计的劣势是无法保证同一个Consumer Group里的Consumer均匀消费数据，优势是每个Consumer不用都跟大量的Broker通信，减少通信开销，同时也降低了分配难度，实现也更简单。另外，因为同一个Partition里的数据是有序的，这种设计可以保证每个Partition里的数据可以被有序消费。
　　如果某Consumer Group中Consumer（每个Consumer只创建1个MessageStream）数量少于Partition数量，则至少有一个Consumer会消费多个Partition的数据，如果Consumer的数量与Partition数量相同，则正好一个Consumer消费一个Partition的数据。而如果Consumer的数量多于Partition的数量时，会有部分Consumer无法消费该Topic下任何一条消息。
## **2. 特点**
### **2.1 Kafka Hight Level Consumer API的特点：**
1) 如果Consumer Group中的consumer线程数量比partition多，那么有的线程将永远不会收到消息。因为kafka的设计是在一个partition上是不允许并发的，所以consumer数不要大于partition数。
2) 如果Consumer Group中的consumer线程数量比partition少，那么有的线程将会收到多个消息。并且不保证数据间的顺序性，kafka只保证在一个partition上数据是有序的。
3) 增减consumer，broker，partition会导致rebalance，所以rebalance后consumer对应的partition会发生变化。
4) High-level接口中获取不到数据的时候是会block的。
### **2.2 关于Consumer Group（基于Hight Level Consumer API）的特点：**
1) 以Consumer Group为单位订阅topic，每个consumer一起去消费一个topic；
2) Consumer Group 通过zookeeper来消费kafka集群中的消息(这个过程由zookeeper进行管理)
相对于Low Level API自己管理offset，Hight Level API把offset的管理交给了zookeeper，但是Hight Level API并不是消费一次就在zookeeper中更新一次，而是每间隔一个(默认1000ms)时间更新一次offset，可能在重启消费者时拿到重复的消息。此外，当分区leader发生变更时也可能拿到重复的消息。因此在关闭消费者时最好等待一定时间（10s）然后再shutdown。
3) Consumer Group设计的目的之一也是为了应用多线程同时去消费一个topic中的数据。
## 3. 实验
### 3.1 Kafka Consumer Group的特性
创建一个Topic (名为topic1)，再创建一个属于group2的Consumer实例，并创建三个属于group1的Consumer实例，然后通过Producer向topic1发送Key分别为6，7，8，9，10，11六条消息。结果发现属于group2的Consumer收到了所有的这三条消息，同时group1中的3个Consumer分别收到了这六条消息，如下图所示。
![2016-11-12_22-53-28](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-12_22-53-28.png)

### 3.2 Consumer Rebalance实验
1) 如果topic1有0，1，2共三个Partition，当group1只有一个Consumer(名为Consumer1)时，该 Consumer可消费这3个Partition的所有数据。
![2016-11-12_23-59-46](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-12_23-59-46.png)

2) 增加一个Consumer(Consumer2)后，其中一个Consumer（Consumer1）可消费2个Partition的数据（Partition 0和Partition 1），另外一个Consumer(Consumer2)可消费另外一个Partition（Partition 2）的数据。
![2016-11-13_00-09-55](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-13_00-09-55.png)

3) 再增加一个Consumer(Consumer3)后，每个Consumer可消费一个Partition的数据。Consumer1消费partition0，Consumer2消费partition1，Consumer3消费partition2。
![2016-11-13_00-16-30](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-13_00-16-30.png)

4) 再增加一个Consumer(Consumer4)后，其中3个Consumer可分别消费一个Partition的数据，另外一个Consumer(Consumer4)不能消费topic1的任何数据。
![2016-11-13_00-25-16](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-13_00-25-16.png)

5) 此时关闭Consumer1，其余3个Consumer可分别消费一个Partition的数据。
![2016-11-13_00-35-24](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-13_00-35-24.png)

6) 接着关闭Consumer2，Consumer3可消费2个Partition，Consumer4可消费1个Partition。
![2016-11-13_00-42-07](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-13_00-42-07.png)

7) 再关闭Consumer3，仅存的Consumer4可同时消费topic1的3个Partition。
![2016-11-13_00-47-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-13_00-47-11.png)

小结，Consumer Rebalance的算法如下：
* 将目标Topic下的所有Partirtion排序，存于$P_T$
* 对某Consumer Group下所有Consumer排序，存$于C_G$，第$i$个Consumer记为$C_i$
* $N=size(P_T)/size(C_G)$，向上取整
* 解除$C_i$对原来分配的Partition的消费权（i从0开始）
* 将第$i*N$到$（i+1）*N-1$个Partition分配给$C_i$