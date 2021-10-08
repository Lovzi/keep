

## 

## 消费者的位移提交







Marking Coordinator Dead！ 



## Coordinator





## 问题

Kafka每次怎么获取group.id的消费进度？

Kafka并不关注__consumer_offsets的消费情况， 每个消费者的Coordinator会将这个Broker的所有相关分区当前已提交的最新位移缓存起来， 并通过这个缓存来决定消费到了哪个位移