---
title: 袋鼠云Flinkx源码解读
categories: Flink
tags:
  - Flink
abbrlink: 783755b3
date: 2021-07-09 10:55:17
---


#袋鼠云Flinx源码解读 

## 架构

袋鼠云通过InputFormat， 并实现InputFormatSourceFunction来统一批流一体的架构，使其既可以继承批处理，又可以实现其流处理。

### KafkaReader

使用线程池，并自定义消费者去拉取消息放入队列，作为队列的生产者，同时InputFormat作为队列的消费者来收集信息并序列化成Row， 随后进入下游算子。 







