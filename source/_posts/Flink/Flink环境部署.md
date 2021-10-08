本文基于Flink1.13.2进行部署， 下载地址： [**https://dlcdn.apache.org/flink/flink-1.13.2/flink-1.13.2-bin-scala_2.11.tgz**](https://dlcdn.apache.org/flink/flink-1.13.2/flink-1.13.2-bin-scala_2.11.tgz)

## 下载&安装

由于考虑后期迭代升级，考虑将配置文件单独进行存放， 故划分目录如下。 

配置:` /data/flink/config/`， 配置文件最终以软连接的方式映射到`/data/flink/flink-1.13.2/conf`下

安装路径: /data/flink/flink-1.13.2

下载完毕后使用下面命令将flink解压缩到/data/flink/flink-1.13.2

```shell
tar -xzvf flink-1.13.2-bin-scala_2.11.tgz -C /data/flink/
```

### 配置结构映射

使用以下命令将可能修改的配置文件进行映射

```shell
# 映射基础flink配置
mv /data/flink/flink-1.13.2/flink-conf.yaml /data/flink/config/flink-conf.yaml
ln -s /data/flink/config/flink-conf.yaml /data/flink/flink-1.13.2/flink-conf.yaml

# 映射masters文件
mv /data/flink/flink-1.13.2/masters /data/flink/config/masters
ln -s /data/flink/config/masters /data/flink/flink-1.13.2/masters

# 映射workers文件
mv /data/flink/flink-1.13.2/workers /data/flink/config/workers
ln -s /data/flink/config/workers /data/flink/flink-1.13.2/workers

```

## 修改Flink全局配置

通过vim打开flink-conf.yaml, 进行修改

```shell
vim /data/flink/flink-1.13.2/flink-conf.yaml
```

### 基础配置（Common）

```yaml
# jobManager 的IP地址
jobmanager.rpc.address: localhost

# JobManager 的端口号
jobmanager.rpc.port: 6123

# 每个 TaskManager 提供的任务 slots 数量大小
taskmanager.numberOfTaskSlots: 1

# 程序默认并行计算的个数
parallelism.default: 1

# 文件系统来源
# fs.default-scheme
```

### 内存配置（Memory）

#### JobManager内存配置

```yaml
# The total process memory size for the JobManager.
#
# Note this accounts for all memory usage within the JobManager process, including JVM metaspace and other overhead.
# JobManager配置总内存
jobmanager.memory.process.size: 512m
# JobManger分配的JVM 堆内存
jobmanager.memory.heap.size: 256m
# JobManger分配的堆外内存（直接内存或本地内存）
jobmanager.memory.off-heap.size: 128m
# JobManger JVM 进程的 Metaspace
jobmanager.memory.jvm-metaspace.size: 64m
# 以下三个值，用于其他JVM开销的本地内存，例如栈空间、垃圾回收空间等。该内存部分为基于进程总内存的受限的等比内存部分。
# JobManager JVM开销的最小值，即当fraction * Total 小于min 时，将使用min值作为JVM开销内存，默认192M
jobmanager.memory.jvm-overhead.min: 16m
# JobManager JVM开销的最大值，即当fraction * Total 大于max 时，将使用max值作为JVM开销内存, 默认是1GB
jobmanager.memory.jvm-overhead.max: 64m
# JobManager JVM开销占总分配内存的比例，默认是0.1
jobmanager.memory.jvm-overhead.fraction: 0.2

```

#### TaskManager内存配置

```yaml

# The total process memory size for the TaskManager.
#
# Note this accounts for all memory usage within the TaskManager process, including JVM metaspace and other overhead.
# TaskManager分配的总内存,如果没有设置，会使用旧的配置taskmanager.heap.size
taskmanager.memory.process.size: 512m
# To exclude JVM metaspace and overhead, please, use total Flink memory size instead of 'taskmanager.memory.process.size'.
# It is not recommended to set both 'taskmanager.memory.process.size' and Flink memory.
#
# TaskManager 使用的Flink程序相关内存，taskmanager.memory.flink.size = taskmanager.memory.process.size - JVM Metaspace and JVM Overhead
# taskmanager.memory.flink.size: 1280m


# 以下三个值为TaskManager JVM开销保留内存, 例如栈空间，垃圾回收空间等。 该内存为基于该进程总内存受限等等比部分
# TaskManager JVM开销占进程总内存的比例，默认0.1. 总进程内存 = taskmanager.memory.process.size
taskmanager.memory.jvm-overhead.fraction: 0.2
# TaskManager JVM开销内存的最小值，当 fraction * Total < min时， 则使用min 作为JVM开销内存
taskmanager.memory.jvm-overhead.min: 16m
# TaskManager JVM开销内存的最大值，当 fraction * Total > max时， 则使用max 作为JVM开销内存
taskmanager.memory.jvm-overhead.max: 64m

# TaskManager JVM 元空间大小
taskmanager.memory.jvm-metaspace.size: 64m

# TaskManager 框架（进程）分配的JVM 堆内存，不会分配给Task
taskmanager.memory.framework.heap.size: 64m
# TaskManager 框架（进程）分配的堆外内存大小，不会分配给Task
taskmanager.memory.framework.off-heap.size: 64m

# 由 Flink 管理的用于排序、哈希表、缓存中间结果及 RocksDB State Backend 的本地内存。内存使用者可以以 MemorySegments 的形式从内存管理器分配内存，或者从内存管理
器保留字节并将其内存使用保持在该边界内。
# taskmanager.memory.managed.size
# 如果未明确指定托管内存大小，则用作托管内存的总 Flink 内存的比例，默认0.4
# taskmanager.memory.managed.fraction: 0.4


# TaskExecutors 的任务堆内存大小。这是为任务保留的 JVM 堆内存大小。如果没有指定，它将被推导出为 Total Flink Memory 减去 Framework Heap Memory、Task Off-Heap Memory、Managed Memory 和 Network Memory
# taskmanager.memory.task.heap.size

# TaskExecutor 的任务堆外内存大小。这是为任务保留的堆外内存（JVM 直接内存和本机内存）的大小。Flink 计算 JVM 最大直接内存大小参数时会完整统计配置的值。默认0
# taskmanager.memory.task.off-heap.size


# The amount of memory going to the network stack. These numbers usually need
# no tuning. Adjusting them may be necessary in case of an "Insufficient number
# of network buffers" error. The default min is 64MB, the default max is 1GB.
# 以下三个值用于设置网络内存,网络内存是为 ShuffleEnvironment 保留的堆外内存（例如，网络缓冲区）
# TaskManager的网络管理内存占总Flink内存的占比，默认为0.1 
# 注意：总Flink内存 = taskmanager.memory.flink.size
taskmanager.memory.network.fraction: 0.1
# TaskManager网络管理内存的最小值，即当 network.fraction * Total Flink < min 时，使用Min值作为网络内存
taskmanager.memory.network.min: 16mb
# TaskManager网络管理内存的最大值，即当 network.fraction * Total Flink > max 时，使用Max值作为网络内存
taskmanager.memory.network.max: 64mb

# 是否应在 TaskManager 启动时预先分配 TaskManager 管理的内存
taskmanager.memory.preallocate: false
```

### 高可用配置（High Availability）

```yaml
#==============================================================================
# High Availability
#==============================================================================

# The high-availability mode. Possible options are 'NONE' or 'zookeeper'.
# 高可用使用的模式，默认是NONE （必须指定）
high-availability: zookeeper

# The path where metadata for master recovery is persisted. While ZooKeeper stores
# the small ground truth for checkpoint and leader election, this location stores
# the larger objects, like persisted dataflow graphs.
#
# Must be a durable file system that is accessible from all nodes
# (like HDFS, S3, Ceph, nfs, ...)
# JobManager 元数据持久化到文件系统 high-availability.storageDir 配置的路径中，storageDir 存储要从 JobManager 失败恢复时所需的所有元数据。
# 在 ZooKeeper 中只能有一个目录指向此位置（必须指定）
high-availability.storageDir: hdfs:///flink/recovery

# The list of ZooKeeper quorum peers that coordinate the high-availability
# setup. This must be a list of the form:
# "host1:clientPort,host2:clientPort,..." (default clientPort: 2181)
# 一个提供分布式协调服务的复制组, 也就是zookeeper 集群中仲裁者的机器 ip 和 port 端口号
high-availability.zookeeper.quorum: 10.39.223.96:2181,10.39.223.97:2181,10.39.223.98:2181

# ZooKeeper 根节点，集群的所有节点都放在该节点下
high-availability.zookeeper.path.root: /flink

## ZooKeeper cluster-id 节点，在该节点下放置集群所需的协调数据
# high-availability.cluster-id: ylink_test

# yarn.application-attempts的设置不应该超过yarn.resourcemanager.am.max-attemps.
yarn.application-attempts: 3

# ACL options are based on https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#sc_BuiltinACLSchemes
# It can be either "creator" (ZOO_CREATE_ALL_ACL) or "open" (ZOO_OPEN_ACL_UNSAFE)
# The default value is "open" and it can be changed to "creator" if ZK security is enabled
#
# high-availability.zookeeper.client.acl: open

# 如果 ZooKeeper 使用 Kerberos 以安全模式运行，必要时可以在 flink-conf.yaml 中覆盖以下配置
# 默认配置为 "zookeeper". 如果 ZooKeeper quorum 配置了不同的服务名称，
# # 那么可以替换到这里。
#
# zookeeper.sasl.service-name: zookeeper
#
# # 默认配置为 "Client". 该值必须为 "security.kerberos.login.contexts" 项中配置的某一个值。
# zookeeper.sasl.login-context-name: Client
```

### 容错和检查点配置（Fault tolerance and checkpointing）

```yaml

#==============================================================================
# Fault tolerance and checkpointing
#==============================================================================

# The backend that will be used to store operator state checkpoints if
# checkpointing is enabled.
#
# Supported backends are 'jobmanager', 'filesystem', 'rocksdb', or the
# <class-name-of-factory>.
# 设置的状态后端组件， 可以是 jobmanager(内存），filesysytem, rocksdb。 还可以自己来实现
state.backend: filesystem

# Directory for checkpoints filesystem, when using any of the default bundled
# state backends.
# 存储检查点的数据文件和元数据的默认目录
state.checkpoints.dir: hdfs:///flink/checkpoints

# Default target directory for savepoints, optional.
# savepoints 的默认目标目录(可选)
state.savepoints.dir: hdfs:///flink/savepoints

# Flag to enable/disable incremental checkpoints for backends that
# support incremental checkpoints (like the RocksDB state backend).
# 用于启用/禁用增量 checkpoints 的标志，默认为禁用
# state.backend.incremental: false

# The failover strategy, i.e., how the job computation recovers from task failures.
# Only restart tasks that may have been affected by the task failure, which typically includes
# downstream tasks and potentially upstream tasks if their produced data is no longer available for consumption.
# 故障恢复策略, full(全图重启，即全部任务重启）， region(局部任务重启）
jobmanager.execution.failover-strategy: region

```

### Web前端配置（ Rest & web frontend）

```yaml
#==============================================================================
# Rest & web frontend
#==============================================================================

# The port to which the REST client connects to. If rest.bind-port has
# not been specified, then the server will bind to this port as well.
#
#rest.port: 8081

# The address to which the REST client will connect to
#
#rest.address: 0.0.0.0

# Port range for the REST and web server to bind to.
#
#rest.bind-port: 8080-8090

# The address that the REST & web server binds to
#
#rest.bind-address: 0.0.0.0

# Flag to specify whether job submission is enabled from the web-based
# runtime monitor. Uncomment to disable.
# 是否能够在WEB端提交作业
#web.submit.enable: false
```

