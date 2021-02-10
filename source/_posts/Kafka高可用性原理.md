---
title: Kafka高可用性原理
date: 2018-09-16
tags: Performance
---

分布式系统中，任何机器都可能面临未知的宕机风险，所以很高可用涉及是一个不可避免的话题。但是高可用带来的代价就是一致性问题，这又是一个很大很有趣的话题了。今天我们仅来谈谈kafka的高可用设计。

![这里写图片描述](https://img-blog.csdn.net/2018091621401630?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NfSjMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 高可用设计

实现高可用性的方式一般都是进行replication，对于kafka，如果没有提供High Availablity机制，一旦一个或多个Broker宕机，则宕机期间其上所有Partition都无法继续提供服务。若该Broker永远不能再恢复，那么所有的数据也就将丢失，这是不可容忍的。所以kafka高可用性的设计也是进行Replication。

### Replica如何分布

为了尽量做好负载均衡和容错能力，需要将同一个Partition的Replica尽量分散到不同的机器。如果所有的Replica都在同一个Broker上，那一旦该Broker宕机，该Partition的所有Replica都无法工作，那么这些Replica也就失去的意义。

### Replica如何同步

当没有Replica的时候，producer向broker写入消息非常简单，当有很多Replica的时候是如何处理的呢？

一般来说，对于这种情况有两个处理方法

1. 同步复制，当producer向所有的Replica写入成功消息后才返回。一致性得到保障，但是延迟太高，吞吐率降低。
2. 异步复制，所有的Replica选取一个一个leader，producer向leader写入成功即返回，leader负责将消息同步给其他的所有Replica。但是消息同步一致性得不到保证，但是保证了快速的响应。

而kafka选取了一个折中的方式：ISR（in-sync replicas）。producer每次发送消息，将消息发送给leader，leader将消息同步给他“信任”的“小弟们”就算成功，巧妙的均衡了确保数据不丢失以及吞吐率。具体的，

- 在所有的Replica中，leader会维护一个与其基本保持同步的Replica列表，该列表称为ISR(in-sync Replica)，每个Partition都会有一个ISR，而且是由leader动态维护。
- 如果一个replica落后leader太多，leader会将其剔除。如果另外的replica跟上脚步，leader会将其加入。
- 同步：leader向ISR中的所有replica同步消息，当收到所有ISR中replica的ack之后，leader才commit。
- 异步：收到同步消息的ISR中的replica，异步将消息同步给ISR集合外的replica。

  ![这里写图片描述](https://img-blog.csdn.net/2018091621405549?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NfSjMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 如何Leader Election

了解了Kafka如何做Replication，随之而来的疑问便是如何选取leader？

leader选举可谓是一个经典问题，立马想到了paxos，raft、Zab等算法，然而kafka采用的方法相比就简单很多：

Kafka的Leader选举是通过在zookeeper上创建/controller临时节点来实现leader选举，并在该节点中写入当前broker的元信息。一个节点只能被一个客户端创建成功，创建成功的broker即为leader，即先到先得原则。

当leader和zookeeper失去连接时，临时节点会删除，而其他broker会监听该节点的变化，当节点删除时，其他broker会收到事件通知，重新发起leader选举。

### 如果所有的replica都不工作了？

两种方式

- 等待ISR中任一个Replica恢复，并选取他为leader
  - 等待时间较长，降低了可用性
  - 若ISR中的所有Replica都无法恢复或者数据丢失，则改partition将永不可用
- 选择第一个回复的Replica为新的leader，无论他是否在ISR中
  - 所选leader可能并未包含已被之前leader commit的消息，因此会造成数据丢失
  - 可用性较高