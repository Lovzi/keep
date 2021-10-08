---
title: Kafka源码解读
categories: Kafka
tags:
  - Kafka
abbrlink: 72017ffd
date: 2021-01-30 16:01:01
---



分析路线：

第一阶段. 了解Kafka的使用和基本原理

第二阶段. 了解Kafka个参数的作用，熟悉Kafka配置调优，根据生产环境的业务进行一个配置调优

第三阶段. 通读源码，对Kafka的原理和特性找到其实现，并不要求每一行都去理解， 画出流程图来帮助理解。

第四阶段. 钻入其中，理解其中的设计模式和架构思想，吸收其优雅的设计理念。

不要认为能把每行代码加上注释就表示自己已经懂了。明白源码的2个标志是：1. 能够自行调试源码；2. 能够在源码上独立编写高阶功能。

## 第一阶段. 了解Kafka的使用和基本原理

见消息引擎之Kafka的来讲解原理。

## 第二阶段. 了解Kafka个参数的作用

##### ProducerConfig的优雅之处

> 1. 通过自定义Map存储对象，类型Object，通过getInt()等对象来转型，比较优雅。
> 2. 通过original(), parse()方法等方式去加载用户定义数据，可以借鉴
> 3. 按照配置的需求定义ConfigDef、ConfigKey等数据结构

1. 

## 第三阶段. 通读源码



### KafkaProducer

在对象构造器中启动线程会造成 this 指针的逃逸。理论上，Sender 线程完全能够观测到一个尚未构造完成的 KafkaProducer 实例。当然，在构造对象时创建线程没有任何问题，但最好是不要同时启动它。

### RecordAccumulator解读



#### 添加批次之append方法梳理

```java
 /**
     * Add a record to the accumulator, return the append result
     * <p>
     * The append result will contain the future metadata, and flag for whether the appended batch is full or a new batch is created
     * <p>
     *
     * @param tp The topic/partition to which this record is being sent
     * @param timestamp The timestamp of the record
     * @param key The key for the record
     * @param value The value for the record
     * @param headers the Headers for the record
     * @param callback The user-supplied callback to execute when the request is complete
     * @param maxTimeToBlock The maximum time in milliseconds to block for buffer memory to be available
     * @param abortOnNewBatch A boolean that indicates returning before a new batch is created and 
     *                        running the the partitioner's onNewBatch method before trying to append again
     */
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Header[] headers,
                                     Callback callback,
                                     long maxTimeToBlock,
                                     boolean abortOnNewBatch) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        ByteBuffer buffer = null;
        if (headers == null) headers = Record.EMPTY_HEADERS;
        try {
            // check if we have an in-progress batch
            /*
             * 步骤一：
             * 先根据分区获取该消息所属的队列中
             *      如果有已经存在的队列，那么我们就使用存在队列
             *      如果队列不存在，那么我们新创建一个队列
             */
            Deque<ProducerBatch> dq = getOrCreateDeque(tp);
            /*
             * 假设我们现在有线程一，线程二，线程三
             *
             */
            synchronized (dq) {
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");
                /*
                 * 步骤二：
                 *      尝试往队列里面的批次里添加数据
                 *
                 *      一开始添加数据肯定是失败的，我们目前只是以后了队列
                 *      数据是需要存储在批次对象里面（这个批次对象是需要分配内存的）
                 *      我们目前还没有分配内存，所以如果按场景驱动的方式，
                 *      代码第一次运行到这儿其实是不成功的。
                 */
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                // 第一次进来的时候appendResult的值就为null
                if (appendResult != null)
                    return appendResult;
            }

            // we don't have an in-progress record batch try to allocate a new batch

            /*
             * 步骤三：计算一个批次的大小
             * 在消息的大小和批次的大小之间取一个最大值，用这个值作为当前这个批次的大小
             *
             * 有可能一个消息比设定好的大小还要打
             * 默认一个批次的大小是16K。
             * 所以我们看到这段代码以后，应该有一个启发
             * 如果我们生产者发送数据的时候，如果我们的消息大小超过了16K, 那么消息就是一条一条的发送出去的，
             * 这样的话批次这个概念的设计也就没有意义了
             *  结论：所有大家一定要根据自己公司的数据大小来设置批次的大小
             */
            if (abortOnNewBatch) {
                // Return a result that will cause another call to append.
                return new RecordAppendResult(null, false, false, true);
            }

            byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
            // 这里去消息和批次的最大值。
            int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            /*
             * 步骤四：
             *  根据批次的大小去分配内存
             *
             *
             *  线程一，线程二，线程三，执行到这儿都会申请内存
             *  假设每个线程 都申请了 16k的内存。
             */
            buffer = free.allocate(size, maxTimeToBlock);
            // Plato: 这里分析真的是太难了，等明天早上再继续。
            synchronized (dq) {
                // 线程1进来了，
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");
                /*
                 * 步骤五：
                 *      再尝试把数据写入到批次里面。
                 *      代码第一次执行到这儿的时候 依然还是失败的（appendResult==null）
                 *      目前虽然已经分配了内存
                 *      但是还没有创建批次，那我们向往批次里面写数据
                 *      还是不能写的, 所以这里appendResult 还是等于null.
                 *
                 *   线程1释放锁，线程二进来执行这段代码的时候，是成功的。
                 */
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null) {
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    return appendResult;
                }
                /*
                 * 步骤六：
                 *  根据内存大小封装批次
                 *
                 *
                 *  线程一到这儿 会根据内存封装出来一个批次。
                 */
                MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
                ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
                //尝试往这个批次里面写数据，到这个时候 我们的代码会执行成功。

                //线程一，就往批次里面写数据，这个时候就写成功了。
                FutureRecordMetadata future = Objects.requireNonNull(batch.tryAppend(timestamp, key, value, headers,
                        callback, time.milliseconds()));

                /*
                 * 步骤七：
                 *  把这个批次放入到这个队列的队尾
                 *
                 *
                 *  线程一把批次添加到队尾
                 */
                dq.addLast(batch);
                incomplete.add(batch);

                // Don't deallocate this buffer in the finally block as it's being used in the record batch
                buffer = null;
                return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true, false);
            } // 释放锁
        } finally {
            if (buffer != null)
                free.deallocate(buffer);
            appendsInProgress.decrementAndGet();
        }
    }
```

不同线程进入此方法可能会涉及三个逻辑

1. 线程1追加记录，没有分配内存，tryApend方法返回null， 为记录分配内存
2. 内存分配完毕后， 线程1尝试添加到批次，此时没有批次，tryApend方法返回null
3. 此时创建批次，继续尝试添加到批次。 添加成功，返回

由于队列中没有批次，第一次添加数据会失败，然后继续执行后面的逻辑，==计算一个批次的大小------->根据批次大小分配内存--------->再次尝试把数据写入到批次中-------->根据内存大小封装批次==，此时此刻，队列有了，队列中的批次批次也构建好了，最后第三次把数据写入到批次中会执行成功。

>这里发现代码中会出现申请内存（free.allocate）和释放内存（free.deallocate）的操作，那么什么时候会去申请内存，什么时候会去释放内存，还有在代码中会看到有些代码中使用了synchronized进行加锁，有些并没有实现synchronized进行加锁。
>
>这里其本质就是用到了分段加锁的设计思想，该加锁的地方就加锁，不该加锁的地方就不加锁，好处就是为了在支持高并发的场景下，既要保证线程的安全，还有尽可能的提升代码的性能。



#### NetworkClient

#### 收获优雅的代码

1. 使用java.util.Objects类（不是Object）,通过requireNonNull函数来assert所有的空对象，nonNull来判断是否空对象

```java
public static <T> T requireNonNull(T var0, String var1) {
        if (var0 == null) {
            throw new NullPointerException(var1);
        } else {
            return var0;
        }
    }
```

1. 线程和业务代码隔离，业务代码继承Runnable, 并实现响应业务功能和run方法，传入到Thread对象中。

```java
   // org.apache.kafka.clients.producer.internals.Sender#Sender
   this.sender = new Sender(client,
                    this.metadata,
                    this.accumulator,
                    config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION) == 1,
                    config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),
                    (short) parseAcks(config.getString(ProducerConfig.ACKS_CONFIG)),
                    config.getInt(ProducerConfig.RETRIES_CONFIG),
                    this.metrics,
                    new SystemTime(),
                    clientId,
                    this.requestTimeoutMs);
            String ioThreadName = "kafka-producer-network-thread" + (clientId.length() > 0 ? " | " + clientId : "");
            //创建了一个线程，然后里面传进去了一个sender对象。
            //把业务的代码和关于线程的代码给隔离开来。

            //关于线程的这种代码设计的方式，其实也值得大家积累的。
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            //启动线程。
            this.ioThread.start();
```

### 问题

为什么通过isr机制，leader宕机，数据不会丢失

[Kafka源码学习](https://app.yinxiang.com/shard/s41/nl/26228147/3a9ad4cd-ef37-4d11-bd6c-7239f11f3a31/)

### 总结

如何看源码？

1. 场景驱动
2. 画图
3. 吸收优秀代码
4. 对理解的代码写好注释

## 第四阶段. 庖丁解牛



