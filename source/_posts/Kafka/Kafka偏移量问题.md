# Kafka偏移量问题



为了测试我们Kafka的偏移量是否会重置，我们做一个测试。

我们向kafka.zhengwu.test topic发送了一条数据，并用kafka.zhengwu.test.group进行了消费，此时消费者组没有积压。 

当前生产者的offset 为 1 

![image-20210603174150422](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603174150422.png)

![image-20210603174225736](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603174225736.png)

我们让这个topic的数据过期，一个很好的设置方式，是修改这个topic的retention.ms设置为1分钟，然后静待其过期。 

过期之后，我们继续通过监控查看topic的情况, 此时可以看到，数据已经过期， current offset仍然是1

![image-20210603174942817](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603174942817.png)



![image-20210603175019303](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603175019303.png)

![image-20210603174810946](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603174810946.png)

这是我们可以看到，消费者组是没有任何变化的，我们继续往生产者发送一条消息，让我们看看会有什么变化。 

从下图我们可以看到， 生产者的偏移量是从 2 (current offset + 1 )开始的，也就是kafka在数据过期之后并不会清空生产者的偏移量

![image-20210603175219542](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603175219542.png)

同理，消费者的偏移量也会从current offset进行维护， 不会从头开始, 这里我们可以知道， kafka又一个`_consumer_offsets` topic， 专门用来存储消费者组对应的偏移量，因此消费者组的偏移量肯定是不会丢失的。  

![image-20210603175421988](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210603175421988.png)

我们在顺带看一下消费者追溯的beginOffset和endOffset。 这里说明消费者针对这个分区4，将从1开始继续消费,  和上面截图保持一致。 

![image-20210604091708446](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210604091708446.png)

那如果是分区3， 当前分区3的偏移量为0， 因此大胆得出结论，分区3应该从0开始

![image-20210604091841583](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210604091841583.png)

消费者因为Kafka内部维护了消费者组的偏移量从而能知道从哪开始， 那为什么生产者没有从0开始消费呢，生产者内部的偏移量是怎么维护的。

实际上，  生产者本身内部并不会维护发送时的偏移量，因为在发送是， 生产者并不知道这是第几条消息，它也不需要知道这是第几条消息，它的任务就是将数据完整的发送到broker中，由broker的LEO维护了每个分区的偏移量，即使数据被删除，偏移量仍然不会变化。 

## 总结

Kafka的每个分区会维护一个LEO（log end offset）：日志末端位移,  即使数据被删除，LEO也不会置为0 

Consumer 通过_consumer_offsets 这个topic维护了每个消费者组的offset位置，即使数据过期，消费者组也不会因此被清空。 