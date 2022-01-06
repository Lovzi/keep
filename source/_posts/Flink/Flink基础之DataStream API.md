---
title: Flink基础之DataStream API
categories: Flink
tags:
  - Flink
abbrlink: 397d80d1
date: 2022-01-05 14:48:13
---


## 控制延迟和吞吐量

默认，Flink中的元素不会一对一的立刻发送到下游，这样会导致不必要的网络传输，我们会存储到一个Buffer中。这个Buffer的大小实际是每次网络传输的单元，可以在Flink客户端的默认配置文件中来设置这个Buffer的大小。也可以在作业的执行环境甚至是算子中来设置。 通过设置Buffer的大小、等待的时间来控制数据的延迟。 

默认等待时间为100ms，在作业中可以用以下几种方式来设置等待时间，如果设置为-1来最大化，即`setBufferTimeout(-1)`则代表不设置超时等待时间，只会在Buffer存满时才会向下发送

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
```

默认Buffer大小为

## 执行模式：批/流的各种区别

### 批执行模式

在批执行模式里，一个作业中的任务可以分隔成若干个Stage，并且按照先后执行。 之所以可以这样做是因为输入是有界的，并且Flink可以在移动到下个节点之前可以将全量数据进行按照一个Stage的逻辑进行完整计算。 在早先的例子中，作业将会有三个Stage。 相当于被Shuffle Barriers分隔成三个任务。

如之前所述，相对于流处理模式来说，批处理不会立即发送记录到下游任务中，在Stage的处理过程中，需要Flink去实现将任务计算的中间结果存储到一个非临时存储中，并允许在上游任务已经计算完成并关闭时，下游任务能继续读取到上游的结果。这会增加处理过程的延迟, 但是同样，这也会允许Flink任务失败的时候回溯到上一个可用的结果中重新计算，而不是重新计算整个作业。另一个更有效的作用是，批作业可以使用更少的资源，因为系统可以在计算完成一部分之后再继续计算另一部分，直至完成所有数据的计算。 

任务管理器将一直保留中间结果直到下游任务不在消耗它们。（从技术上讲，它们会一直保存，直到下游区域产生它们的输出）在那之后，只要空间允许，他们将会保留，以便在作业失败时，回溯到我们之前提到的更早的结果中去。 

### 状态后端/状态

在流处理模式中，Flink使用StateBackend来控制怎样去存储状态以及怎样去给工作打快照。 

在批处理模式中，Flink配置的状态后端将被忽略，取而代之的是数据会按照Key进行分组（通过排序），我们会依次处理一个Key的所有记录，并允许在同一个时间只保留一个Key的状态。 当移动到下个Key的数据操作时，给定Key的状态将被丢弃。 

### 处理顺序





### 事件事件/水位线

### 重要提示：

批处理和流处理在某些情况会有不同的表现：

* 在批处理模式，reduce和sum等Rolling 算子只会在计算完成后，产出一个结果。 在流处理模式中，每个新纪录到达时，都会产生一个结果。 

批处理不支持的语义：

* [Checkpointing](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/concepts/stateful-stream-processing/#stateful-stream-processing)  以及任何依赖checkpoint的操作都不会工作
* [Iterations](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/datastream/operators/overview/#iterate) 即数据递归，目前在批处理也无法实现

实现自定义算子应该谨慎，否则可能产生不正确的表现，具体细节在下面会有附加解释。 

### Checkpointing

正如上面介绍的那样，相对于批处理程序，失败恢复不会使用checkpointing . 

因为没有checkpointing, 所以我们要记住，某些特性，例如 [CheckpointListener](https://nightlies.apache.org/flink/flink-docs-release-1.13/api/java//org/apache/flink/api/common/state/CheckpointListener.html) 以及Kafka的[EXACTLY_ONCE](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/datastream/kafka/#kafka-producers-and-fault-tolerance) 语义、`StreamingFileSink`的[OnCheckpointRollingPolicy](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/datastream/streamfile_sink/#rolling-policy) 特性都不会工作，如果需要一个事务性的Sink去工作，需要确保使用在[FLIP-143](https://cwiki.apache.org/confluence/x/KEJ4CQ)被提出的Unified Sink API。 

### 谨慎使用自定义算子

在批处理模式使用自定义算子的时候要记住对执行模式的假设，否则，在流处理中工作的很好的运算符，在批处理中很有可能产生错误的结果。 算子永远不会限定于特定的Key。 

首先，不应该在运算符中缓存最后看到的水印，在批处理中我们是逐个Key的去处理记录。其结果就是水印在每个Key之间会从Max_Value切换到Min_Value，同样，定时器首先按照Key的顺序出发，随后按照每个Key内的时间戳顺序触发。 此外，不支持手动修改一个Key. 









