---
title: 消息引擎之Kafka
date: 2021-01-30 16:01:01
categories: Kafka
tags:
	- Kafka
---

# 消息引擎之Kafka

## Kafka压缩

### 小结

1. 压缩方式及压缩的场景需要权衡，即在带宽、磁盘和CPU之间做一个权衡

### 问题

#### Kafka什么时候会用上零拷贝的特性？

目前来看，Kafka本身只有在消费的时候， 才会用上Linux自带的零拷贝的特性。 其原理就是Kafka本身在讲数据从文件读取到内核页缓存之后，会将其从页缓存直接读取到Socket的缓存中，这个过程目前只发生在Consumer

换句话说，Producer发送数据到Broker乃至Broker写入数据到文件中，目前在2.4版本以前都不会出现零拷贝。 因为从Producer发送数据到Broker本质上就是两台主机的两个进程之间进行交互，是两个进程操作两块完全不同的网卡（单机版测试当我没说），涉及不到零拷贝这块来。 那么从Broker接收到消息将其写入到文件中，这块用不上零拷贝最重要的一个原始，是需要Broker对消息做一个校验，防止因为网络传输丢包的情况导致消息丢失。 这个过程不管怎么样都需要经过用户态的操作（Kafka在用户态）， 内核的那些系统调用可不会管这卵子事情。 所以肯定就用不上零拷贝技术。 

#### Broker 要对压缩消息集合执行解压缩操作，然后逐条对消息进行校验，这个解压缩操作是否能够避免？

按照我的理解，如果单纯只做CRC校验，就完全不必做解压缩操作。 Producer在生产消息之后，将此次消息批次（Kafka以消息批次作为发送的基本单位）进行压缩并对压缩体做CRC校验，同时将CRC和压缩体发送到Broker中。 Broker再次对压缩体进行CRC计算后和生产者计算的CRC进行比对，结果一致则证明没有消息丢失，将压缩体直接写入到文件中。 

那为什么Broker还要做这一次解压缩操作呢，我们的想法大佬肯定想过，据说是Broker不仅仅对消息批次做校验，也会对同一个消息批次内每条消息单独做校验，那这个过程肯定是需要解压缩的。 除非哪天大佬不需要对每条消息都做解压缩了。  据说京东小伙伴已经提出了这个方案并被采纳了https://issues.apache.org/jira/browse/KAFKA-8106

#### Kafka怎么实现零拷贝特性？

Java FileChannel的transferTo方法底层调用sendfile，实现Zero Copy。详细信息你可以查看这个jdk ticket：https://bugs.openjdk.java.net/browse/JDK-4474736

### 参考

[零拷贝的一些理解](https://www.cnblogs.com/lhh-north/p/11031821.html)

## Kafka保证无丢失消息

![img](https://gitee.com/Goook/pictures/raw/master/uPic/3fc09aa33dc1022e867df4930054ce5a-20210131161841315.jpg)

### 小结

#### Kafka如何在保证可用性最大化保证消息不丢失的配置

在保证可用性的前提下，设置ack等于all在某些场景下也会造成数据丢失， 比如当时Kafka的环境非常差劲，导致其ISR集合中只有一个副本。 原始副本假设为4个， 那么即使设置ACK=all，Kafka在将消息写入到ISR的这个副本后，也会返回写入成功，如果最后的ISR集合宕机了，那么消息就是丢失了，因为生产者因为已经返回写入成功而不会在进行重试了。

为了解决这个问题, 需要合理的设置参数`min.insync.replicas`，这个参数的含义是需要发送的副本满足此设置，才视为发送成功。 一般设置为 `replication.factor = min.insync.replicas + 1`。 至于为什么加一， 是因为需要保证可用性。 设置`replication.factor = min.insync.replicas `时，如果有一个副本不可用，则Producer无法生产消息。 

### 问题

#### Kafka 增加主题分区导致消息丢失

当增加主题分区后，在某段“不凑巧”的时间间隔后，Producer 先于 Consumer 感知到新增加的分区，而 Consumer 设置的是“从最新位移处”开始读取消息，因此在 Consumer 感知到新分区前，Producer 发送的这些消息就全部“丢失”了，或者说 Consumer 无法读取到这些消息。严格来说这是 Kafka 设计上的一个小缺陷。这个如何去解决呢？

## Kafka的幂等性

Kafka的幂等性本质上是通过空间换时间的特性，及在Broker端缓存此次消息的标志， 如果发现这个消息之前已经发送过了，则在后台默默丢弃并返回成功的响应给到用户。

#### Kafka幂等性的局限性

Kafka的幂等性有一定的局限性，主要有以下几点： 

* 只能实现分区级的幂等性，比如如果这个Kafka设置的是默认的分区器又没有指定消息的Key，则默认使用轮询的策略进行发送， 那么这个过程每次发送的消息都是不同的分区，这里是无法保证幂等性的。 

* 只能实现单会话级别的幂等性，如果Producer重启，则幂等性保证也就失效了。 

所以单从幂等性无法完全保证一条消息只会成功发送一次， 要做到精确一次保证还需要另一个重要的特性： 事务

## Kafka的事务

Kafka 的事务概念类似于我们熟知的数据库提供的事务。在数据库领域，事务提供的安全性保障是经典的 ACID，即原子性（Atomicity）、一致性 (Consistency)、隔离性 (Isolation) 和持久性 (Durability)。

当然，在实际场景中各家数据库对 ACID 的实现各不相同。特别是 ACID 本身就是一个有歧义的概念，比如对隔离性的理解。大体来看，隔离性非常自然和必要，但是具体到实现细节就显得不那么精确了。

通常来说，隔离性表明并发执行的事务彼此相互隔离，互不影响。经典的数据库教科书把隔离性称为可串行化 (serializability)，即每个事务都假装它是整个数据库中唯一的事务。提到隔离级别，这种歧义或混乱就更加明显了。很多数据库厂商对于隔离级别的实现都有自己不同的理解，比如有的数据库提供 Snapshot 隔离级别，而在另外一些数据库中，它们被称为可重复读（repeatable read）。好在对于已提交读（read committed）隔离级别的提法，各大主流数据库厂商都比较统一。所谓的 read committed，指的是当读取数据库时，你只能看到已提交的数据，即无脏读。同时，当写入数据库时，你也只能覆盖掉已提交的数据，即无脏写。

https://www.jianshu.com/p/f77ade3f41fd

### 注意

Kafka的事务回滚并不像MySQL那样，能够完全的回溯的之前的状态。 他仅仅能够让Broker标记其消息是未提交而已，本质上，只要Producer发送消息到Broker，Broker都会将其写入到文件进行持久化，所以如果消费者设置read_commited为False时，即使事务回滚了，消费者依旧能消费到此次消息，所以消费者和生产者的配合一定要大好，最好每次都设置read_commited为True。

## __consumer_offsets主题

在1.0.9版本之后，Kafka不再通过ZK来管理消费者的位移提交和存储。 该用内部__consumer_offsets来保存消费者组的位移信息。

其消息格式为 <Group.id. Topic, partition>

一旦某个 Consumer Group 下的所有 Consumer 实例都停止了，而且它们的位移数据都已被删除时，Kafka 会向位移主题的对应分区写入 tombstone 消息，表明要彻底删除这个 Group 的信息。

### 几个重要参数

1. `offsets.retention.minutes` 主题偏移量日志文的保留时长(分钟)， 即__consumer_offsets这个topic中消息的保存时长。 

2. `offsets.topic.replication.factor` Kafka位移主题的副本数量， 在新版本中默认为3

   

## Coordinator

所谓协调者，在 Kafka 中对应的术语是 Coordinator，它专门为 Consumer Group 服务，负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理等。

Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移。同样地，当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作。

所有 Broker 都有各自的 Coordinator 组件, 在Broker启动时，都会创建和开启相应的Coordinator 组件。

### Rebalance

#### 如何避免Rebalance

要避免 Rebalance，还是要从 Rebalance 发生的时机入手。我们在前面说过，Rebalance 发生的时机有三个：

* 组成员数量发生变化
* 订阅主题数量发生变化
* 订阅主题的分区数发生变化

后面两个通常都是运维的主动操作，所以它们引发的 Rebalance 大都是不可避免的。接下来，我们主要说说因为组成员数量变化而引发的 Rebalance 该如何避免。

如果Consumer Group组下面的Consumer成员数量发生变化，就一定会引发Rebalance， 这是Rebalance发生的最常见原因。 Consumer成员数量增加很好理解，一般也是业务代码主动增加的操作，处于增加TPS或者是提高伸缩性的需要，这种情况的Rebalance我们无法避免。 

我们需要避免的就是Consumer成员非预期的减少，如果是你自己主动停掉某个Consumer程序，那么这种情况不必多说， 如果是Coordinator错误的认为某些Consumer程序已经停止从而将其提出了Consumer Group，  这种情况我们需要仔细的分析从而避免。 

那什么时候Coordinator会认为一个Consumer程序一定停止了， 主要有以下几点： 

* Consumer程序发送心跳请求超时

  `session.timeout.ms`这个参数用来控制Consumer程序的超时时间。 

* Consumer程序消费能力不足，Coordinator将其踢出了消费组。

  `max.poll.interval.ms` 这个参数限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。 默认值是 5 分钟，表示你的 Consumer 程序如果在 5 分钟之内无法消费完 poll 方法返回的消息，那么 Consumer 会主动发起“离开组”的请求，Coordinator 也会开启新一轮 Rebalance。本质上是Consumer消费能力不足，或者是一次拉去的记录条数过大导致。 



## Consumer提交Offset

Consumer有自动提交和手动提交

自动提交可以理解在poll方法中去拉取消息的同时提交位移信息。这里需要注意一点，**如果处理消息的时长大于自动提交的时间间隔，即使已经满足自动提交的时间，Kafka仍然只会在下次poll方法调用时去提交Offset。





### Kafka 偏移量问题





