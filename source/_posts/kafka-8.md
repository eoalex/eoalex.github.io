---
title: 'Kafka入门之八:Kafka的新API'
tags:
  - Kafka
id: 1464
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-12-03 13:10:57
---

前面几节我们讲的Kafka都是基于0.8.2.2的版本，截止到今天，kafka实际上已经更新到0.10.1.0，那么API都有哪些变化呢？
**1 Producer API**
在Kafka 0.8.2, Producer已经被重新设计，所以这次变化较少。
1.1增加JAVA接口的发送回调（原来只支持SCALA接口）
异步发送消息到一个主题，然后调用提供的callback，发送确认结果。
<pre lang="java">producer.send(record, new Callback() {
				@Override
				public void onCompletion(RecordMetadata metadata, Exception exception) {
					System.out.printf("Send record partition:%d, offset:%d, keysize:%d, valuesize:%d %n",
							metadata.partition(), metadata.offset(), metadata.serializedKeySize(),
							metadata.serializedValueSize());
				}

			});</pre>

1.2重构Partitioner接口
原来0.8.2.2的接口是这样的，只有两个参数
<pre lang="java">public int partition(Object key, int numPartitions) </pre>
现在0.10.1.0的接口是这样的，有六个参数。
<pre lang="java">public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster)</pre>
**2 Consumer API**
在最新的版本中，Consumer API不再区分High Level还是Low Level。
2.1 重构Consumer包
把kafka.consumer和kafka.javaapi统一到kafka.clients.consumer，使包更加统一。
2.2 新增Subscribe和Assign接口
Subscribe实际实现了原High Level功能，Assign实现了原Low Level功能。
2.2.1 Subscribe
Subscribe通过ConsumerRebalanceListener来监听和动态分配。通过subscribe(List, ConsumerRebalanceListener)来订阅主题列表，或者通过subscribe(Pattern, ConsumerRebalanceListener)来订阅匹配特定模式的主题。 所以，如果一个主题有4个分区，并且一个消费者组有2个进程，那么每个进程将从2个分区来进行消费，如果一个进程故障，分区将重新分派到同组的其他的进程。如果有新的进程加入该组，分区将现有消费者移动到新的进程。具体来说，如果2个进程订阅了一个主题，指定不同的组，他们将获取这个主题所有的消息，如果他们指定相同的组，那么它们将每个获取大约一半的消息。
2.2.2 Assign
如果我们使用Assign接口，那么将不会使用消费者组,也将禁用动态分区分配.下面的代码演示了直接消费parttion 0和1的消息，不管有多少个进程，消费的消息都是一样的。
<pre lang="java">consumer.assign(Arrays.asList(new TopicPartition(topic, 0), new TopicPartition(topic, 1)));
		while (true) {
			ConsumerRecords<String, String> records = consumer.poll(100);
			records.forEach(record -> {
				System.out.printf("client : %s , topic: %s , partition: %d , offset = %d, key = %s, value = %s%n", clientid, record.topic(),
						record.partition(), record.offset(), record.key(), record.value());
			});
		}</pre>
2.3 commit功能
除了保持原来自动commit和手动commit的功能外，kafka增加了两个功能。
1）支持同步和异步的commit并支持commit回调
这是0.8.2.2的手动commit。
<pre lang="java">consumerConnector.commitOffsets();</pre>
在0.10.1.0中手动同步commit。
<pre lang="java">consumer.commitSync();</pre>
在0.10.1.0中手动异步commit并回调。
<pre lang="java">consumer.commitAsync(new OffsetCommitCallback() {
						@Override
						public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {

						}
					});</pre>
2）支持手动commit特定的partition的offset
<pre lang="java">consumer.subscribe(Arrays.asList(topic));
		AtomicLong atomicLong = new AtomicLong();
		while (true) {
			ConsumerRecords<String, String> records = consumer.poll(100);
			records.partitions().forEach(topicPartition -> {
				List<ConsumerRecord<String, String>> partitionRecords = records.records(topicPartition);
				partitionRecords.forEach(record -> {
					System.out.printf("client : %s , topic: %s , partition: %d , offset = %d, key = %s, value = %s%n", clientid, record.topic(),
							record.partition(), record.offset(), record.key(), record.value());
				});
				long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
                consumer.commitSync(Collections.singletonMap(topicPartition, new OffsetAndMetadata(lastOffset + 1)));
			});
		}</pre>
这里需要注意的是，已提交的偏移量应该一直将读取的下一条消息来的偏移量。因此，调用commitSync时，需要添加最后一条消息的偏移量。

2.4 控制消费位置
kafka允许指定位置，通过API指定从任意offset位置开始消费。使用seek(TopicPartition, long)来指定新的位置，也可用seekToBeginning(Collection) 表示从最开始位置和seekToEnd(Collection)表示从最后位置消费。
2.5 消费流程控制
kafka支持动态控制消费流量，分别在poll(long)调用中执行中使用 pause(Collection) 和 resume(Collection) 来暂停消费指定分配的分区，重新开始消费指定暂停的分区。下面的代码将暂停消费partition 3和4.
<pre lang="java">while (true) {
			ConsumerRecords<String, String> records = consumer.poll(100);
			consumer.pause(Arrays.asList(new TopicPartition(topic, 3)));
			consumer.pause(Arrays.asList(new TopicPartition(topic, 4)));
			records.forEach(record -> {
				System.out.printf("client : %s , topic: %s , partition: %d , offset = %d, key = %s, value = %s%n", clientid, record.topic(),
						record.partition(), record.offset(), record.key(), record.value());
			});
		}</pre>

2.6 多线程处理模型
Kafka的Consumer的接口为非线程安全的。多线程共用IO，Consumer线程需要自己做好线程同步。如果想立即终止consumer，唯一办法是用调用接口：wakeup()，使处理线程产生WakeupException。参见[官方文档](http://kafka.apache.org/0100/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#multithreaded)
<pre lang="java">public class KafkaConsumerRunner implements Runnable {
     private final AtomicBoolean closed = new AtomicBoolean(false);
     private final KafkaConsumer consumer;

     public void run() {
         try {
             consumer.subscribe(Arrays.asList("topic"));
             while (!closed.get()) {
                 ConsumerRecords records = consumer.poll(10000);
                 // Handle new records
             }
         } catch (WakeupException e) {
             // Ignore exception if closing
             if (!closed.get()) throw e;
         } finally {
             consumer.close();
         }
     }

     // Shutdown hook which can be called from a separate thread
     public void shutdown() {
         closed.set(true);
         consumer.wakeup();
     }
 }</pre>
如果用以下的方式开启多个线程是禁止的。
<pre lang="java">
		Thread[] consumerThreads = new Thread[2];
		for (int i = 0; i < consumerThreads.length; ++i) {
			consumerThreads[i] = new Thread(runnable);
			consumerThreads[i].start();
		}</pre>
[![2016-12-03_12-50-49](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/12/2016-12-03_12-50-49.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/12/2016-12-03_12-50-49.png)