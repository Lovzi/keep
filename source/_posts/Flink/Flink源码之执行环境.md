---
title: Flink源码分析之执行环境
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



### InputFormat

InputFormat是用于生成数据的基本接口

这个InputFormat有以下定义

1. 他描述了是否可以使用并行拆分
2. 他描述了怎样从此InputSplit读取记录
3. 它描述了怎样从基本的InputSplit收集统计信息





### RichInputFormat

相对于InputFormat增加了上下文，默认实现了OpenInputFormat、CloseInputFormat





### InputFormatSourceFunction

#### 问题： Barrier的发送？

