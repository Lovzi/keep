---
title: Flink通读源码之执行环境
categories: Flink
tags:
  - Flink
abbrlink: c8523b50
date: 2021-07-09 10:55:17
---



## Flink执行图

## TypeInformation: 一个算子的描述信息



DataStream数据

StreamOprater， 处理数据的操作类。 





## RuntionContext上下文

RuntionContext定义了非常丰富的方法，不同类型的Operator创建的RuntimeContext也有一定的区别， 因此在Flink中提供了不同的实现类。 



### RichFunction详解





## Watermark

Watermark其本质就是一个时间。 



####Wartmark和LangencyMarker本质上都是流中的元素，即StreamElement

SourceFunction会调用collectWithTimestamp来收集每个元素的timestamp， 会调用emitWatemark来收集Watermark。 

除了这 





