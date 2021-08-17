

## Broker端参数

### 存储类参数

* log.dirs:  这是非常重要的参数，指定了 Broker 需要使用的若干个文件目录路径。要知道这个参数是没有默认值的，这说明什么？这说明它必须由你亲自指定。
* log.dir：注意这是 dir，结尾没有 s，说明它只能表示单个路径，它是补充上一个参数用的

在线上生产环境中一定要为log.dirs配置多个路径，具体格式是一个 CSV 格式，也就是用逗号分隔的多个路径，比如/home/kafka1,/home/kafka2,/home/kafka3这样做有两个好处

* 提升读写性能，比起单个磁盘，多个磁盘同时读写能够对数据有更高的吞吐量。 
* 能够实现故障转移， 即 Failover。这是 Kafka 1.1 版本新引入的强大功能。要知道在以前，只要 Kafka Broker 使用的任何一块磁盘挂掉了，整个 Broker 进程都会关闭。但是自 1.1 开始，这种情况被修正了，坏掉的磁盘上的数据会自动地转移到其他正常的磁盘上，而且 Broker 还能正常工作。还记得上一期我们关于 Kafka 是否需要使用 RAID 的讨论吗？这个改进正是我们舍弃 RAID 方案的基础：没有这种 Failover 的话，我们只能依靠 RAID 来提供保障。 

### Zookeeper相关配置

* zookeeper.connect： CSV相关格式参数， 比如zk1:2181,zk2:2181,zk3:2181。 

如果有两套Kafka集群，想要使用同一套zk, 可以使用chroot， 即ZK自带的别名功能， 格式为zk1:2181,zk2:2181,zk3:2181/kafka1和zk1:2181,zk2:2181,zk3:2181/kafka2， 其中chroot只需在最后指定一次。 

### Broker连接相关参数

* listeners：学名叫监听器，其实就是告诉外部连接者要通过什么协议访问指定主机名和端口开放的 Kafka 服务。
* advertised.listeners：和 listeners 相比多了个 advertised。Advertised 的含义表示宣称的、公布的，就是说这组监听器是 Broker 用于对外发布的。
* host.name/port：列出这两个参数就是想说你把它们忘掉吧，压根不要为它们指定值，毕竟都是过期的参数了。



