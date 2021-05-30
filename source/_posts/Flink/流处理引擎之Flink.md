# 流处理引擎之Flink



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

