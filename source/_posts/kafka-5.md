---
title: 'Kafka入门之五:Kafka的Leader Election'
tags:
  - Kafka
  - zookeeper
id: 1401
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-11-12 23:21:16
---

## 1 **基于Zookeeper的Leader Election**
### 1.1 **抢注Leader节点——非公平模式**
* 创建Leader父节点，如 /chroot，并将其设置为persist节点
* 各客户端通过在/chroot下创建Leader节点，如/chroot/leader，来竞争Leader。该节点应被设置为ephemeral
* 若某创建Leader节点成功，则该客户端成功竞选为Leader
* 若创建Leader节点失败，则竞选Leader失败，在/chroot/leader节点上注册exist的watch，一旦该节点被删除则获得通知
* Leader可通过删除Leader节点来放弃Leader
* 如果Leader宕机，由于Leader节点被设置为ephemeral，Leader节点会自行删除。而其它节点由于在Leader节点上注册了watch，故可得到通知，参与下一轮竞选，从而保证总有客户端以Leader角色工作。

### 1.2 **先到先得，后者监视前者——公平模式**
* 创建Leader父节点，如 /chroot，并将其设置为persist节点
* 各客户端通过在/chroot下创建Leader节点，如/chroot/leader，来竞争Leader。该节点应被设置为 ephemeral_sequential
* 客户端通过getChildren方法获取/chroot/下所有子节点，如果其注册的节点的id在所有子节点中最小，则当前客户端竞选Leader成功
* 否则，在前面一个节点上注册watch，一旦前者被删除，则它得到通知，返回step3(并不能直接认为自己成为新的Leader，因为可能前面的节点只是宕机了)
* Leader节点可通过自行删除自己创建的节点以放弃Leader

## 1.3 **Leader Election在Curator中的实现**
### 1.3.1 **LeaderLatch**
* 竞选为Leader后，不可自行放弃领导权
* 只能通过close方法放弃领导权
* 强烈建议增加ConnectionStateListener，当连接SUSPENDED或者LOST时视为丢失领导权，可通过await方法等待成功获取领导权，并可加入timeout
* 可通过hasLeadership方法判断是否为Leader
* 可通过getLeader方法获取当前Leader
* 可通过getParticipants方法获取当前竞选Leader的参与方

### 1.3.2 **LeaderSelector**
* 竞选Leader成功后回调takeLeadership方法
* 可在takeLeadership方法中实现业务逻辑
* 一旦takeLeadership方法返回，即视为放弃领导权
* 可通过autoRequeue方法循环获取领导权
* 可通过hasLeadership方法判断是否为Leader
* 可通过getLeader方法获取当前Leader
* 可通过getParticipants方法获取当前竞选Leader的参与方

## 2. **Kafka的Leader Election**
### 2.1 **“各自为政”的Leader Election**
每个Partition的多个Replica同时竞争Leader
**优点：**
实现简单
**缺点：**
* Herd Effect，
* Zookeeper负载过重，
* Latency较大

### 2.2 **基于Controller的Leader Election**
整个集群中选举出一个Broker作为Controller，Controller为所有Topic的所有Partition指定Leader及Follower

**优点：**
* 极大缓解Herd Effect问题，
* 减轻Zookeeper负载
* Controller与Leader及Follower间通过RPC通信，高效且实时

**缺点：**
* 引入Controller增加了复杂度
* 需要考虑Controller的Failover