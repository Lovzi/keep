



一般在作业日志会看到这么一个错误

```java
2021-01-04 19:22:17,123 WARN  org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer [] - Encountered error org.apache.kafka.common.errors.ProducerFencedException: Producer attempted an operation with an old epoch. Either there is a newer producer with the same transactionalId, or the producer's transaction has been expired by the broker. while recovering transaction KafkaTransactionState [transactionalId=Source: KafkaSource -> Map -> Filter -> Flat Map -> Sink: KafkaSink-7f86b06891c19f1e76c3f65c90ce752b-2, producerId=24019, epoch=735]. Presumably this transaction has been already committed before
```

我和同事几天的分析终于定位到问题的原因是kafka同一个事务id被不同的epoch使用，当旧的epoch尝试提交，新的epoch也开始提交之后，就容易造成这个问题。
此时，checkpoint会失败， 作业会重启，flink会尝试从前一个checkpoint中恢复作业，也就是更早的epoch去恢复，也就是重复报错，那这个问题的现象就是作业会不间断重启，导致作业不可用。

如何解决？
问题原因是，不同生产者使用了相同的transacationId， 这里不同既可以是不同的producerId也可以是epoch id， 甚至两个毫不相关的作业，使用了一个transacationId， 都有可能造成这个问题。换句话说， 只要当不同生产者，或者相同生产者不同事务(并行事务，已提交的事务不会报错）导致使用了同一个事务id时，问题就容易复现。
所以， 只要我们保证不同作业能够使用不同的事务id， 基本就能够解决这个问题。

但FlinkKafkaProduer的transacationId不能手动配置，代码已经定义transacationId的规则。
事务事务生成器如下

```java
transactionalIdsGenerator = new TransactionalIdsGenerator(
    //这里会构造事务id的前缀， 由getRuntimeContext().getTaskName()  和  getRuntimeContext()).getOperatorUniqueID() 来决定。 
    // 这两个函数分别跟SingleOutputStreamOperator中name()和uid()方法有关。 
    // 修改这两个函数的值，就可以间接的改变事务的id
    getRuntimeContext().getTaskName() + "-" + ((StreamingRuntimeContext) getRuntimeContext()).getOperatorUniqueID(),
    getRuntimeContext().getIndexOfThisSubtask(),
    getRuntimeContext().getNumberOfParallelSubtasks(),
    kafkaProducersPoolSize,
    SAFE_SCALE_DOWN_FACTOR);
```

**SingleOutputStreamOperator中name()和uid()**

```java
public class UIDTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment environment = StreamExecutionEnvironment.getExecutionEnvironment();

        List<String> lst = new ArrayList<>();
        lst.add("1");
        lst.add("1");
        lst.add("1");
        lst.add("1");
        lst.add("1");
        DataStreamSource<String> outDataStreamSource = environment.fromCollection(lst);
        SingleOutputStreamOperator<String> runStream = outDataStreamSource.map(new RichMapFunction<String, String>() {
            @Override
            public String map(String s) throws Exception {
                return "taskName: " + getRuntimeContext().getTaskName() +
                        "operatorId: " + ((StreamingRuntimeContext) getRuntimeContext()).getOperatorUniqueID();
            }
            
        }).name("test").uid("uid2"); // 这里来修改name和uid, 当然这只是案例， 真正要修改的是FlinkKafkaProder这里Sink的。 
        // 懒得去修改代码了，假设这个print Sink是 FlinkKafkaProducer的sink. 修改name和uid 
        
        DataStreamSink<String> print = runStream.print().name("print").uid("printUId");
        System.out.println(outDataStreamSource.getTransformation().getUid());
        System.out.println(runStream.getTransformation().getUid());

        System.out.println(print.getTransformation().getUid());
        environment.execute();
    }
}
```

暂时写到这，待看到源码, 再去uid和OperatorID的关系。