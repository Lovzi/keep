### Kafka高水位和Leader Epoch机制

## 什么是高水位？



水位多来自于流式处理领域，现在的大多数流式框架基本都有这个特性，比如耳熟能详的 Flink和Spark Stream。 常理来讲，水位是介于有序数据流和无序数据流的一个边界。

 教科书中关于水位的经典定义通常是这样的：

>在时刻 T，任意创建时间（Event Time）为 T’，且 T’≤T 的所有事件都已经到达或被观测到，那么 T 就被定义为水位。

“Streaming System”一书则是这样表述水位的：

> 水位是一个单调增加且表征最早未完成工作（oldest work not yet completed）的时间戳。

借鉴大佬的一张图用以理解水位：

![img](https://gitee.com/Goook/pictures/raw/master/uPic/8426888d04e1e9917619829b7e3de877.png)

图中标注“Completed”的蓝色部分代表已完成的工作，标注“In-Flight”的红色部分代表正在进行中的工作，两者的边界就是水位线。

但是在Kafka中，水位的概念确稍有不同。 Kafka的水位不是时间出，而是基于Offset。它定义了分区中已提交和未提交消息的边界。 另外KaFka使用的表述是高水位（High Watermark），简称是HW。 

