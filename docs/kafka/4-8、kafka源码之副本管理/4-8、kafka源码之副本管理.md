## 1. 开头

## 2. AbstractFetcherThread

### 2.1. 定义

```

abstract class AbstractFetcherThread(
  name: String,  // 线程名称
  clientId: String,  // Client Id，用于日志输出
  val sourceBroker: BrokerEndPoint,  // 数据源Broker地址
  failedPartitions: FailedPartitions,  // 处理过程中出现失败的分区
  fetchBackOffMs: Int = 0,  // 获取操作重试间隔
  isInterruptible: Boolean = true,  // 线程是否允许被中断
  val brokerTopicStats: BrokerTopicStats) // Broker端主题监控指标
  extends ShutdownableThread(name, isInterruptible) {
  // 定义FetchData类型表示获取的消息数据
  type FetchData = FetchResponse.PartitionData[Records]
  // 定义EpochData类型表示Leader Epoch数据
  type EpochData = OffsetsForLeaderEpochRequest.PartitionData
  private val partitionStates = new PartitionStates[PartitionFetchState]
  ......
}

```

name：线程名字。

sourceBroker：源 Broker 节点信息。源 Broker 是指此线程要从哪个 Broker 上读取数据。

failedPartitions：线程处理过程报错的分区集合。

fetchBackOffMs：当获取分区数据出错后的等待重试间隔，默认是 Broker 端参数 replica.fetch.backoff.ms 值。

brokerTopicStats：Broker 端主题的各类监控指标，常见的有 MessagesInPerSec、BytesInPerSec 等。

这些字段中比较重要的是 **sourceBroker**，因为它决定 Follower 副本从哪个 Broker 拉取数据，也就是 Leader 副本所在的 Broker 是哪台

#### 2.1.1. PartitionData

```
public static final class PartitionData<T extends BaseRecords> {
    public final Errors error;           // 错误码
    public final long highWatermark;     // 高水位值
    public final long lastStableOffset;  // 最新LSO值 
    public final long logStartOffset;    // 最新Log Start Offset值
    // 期望的Read Replica
    // KAFKA 2.4之后支持部分Follower副本可以对外提供读服务
    public final Optional<Integer> preferredReadReplica;
    // 该分区对应的已终止事务列表
    public final List<AbortedTransaction> abortedTransactions;
    // 消息集合，最重要的字段！
    public final T records;
    // 构造函数......
}
```

PartitionData 这个类定义的字段中，除了我们已经非常熟悉的 highWatermark 和 logStartOffset 等字段外，还有一些属于比较高阶的用法：

preferredReadReplica，用于指定可对外提供读服务的 Follower 副本；

abortedTransactions，用于保存该分区当前已终止事务列表；

lastStableOffset 是最新的 LSO 值，属于 Kafka 事务的概念。

关于这几个字段，你只要了解它们的基本作用就可以了。实际上，在 PartitionData 这个类中，最需要你重点关注的是 **records 字段**。因为它保存实际的消息集合，而这是我们最关心的数据。

说到这里，如果你去查看 EpochData 的定义，能发现它也是 PartitionData 类型。但，你一定要注意的是，EpochData 的 PartitionData 是 OffsetsForLeaderEpochRequest 的 PartitionData 类型。

事实上，**在 Kafka 源码中，有很多名为 PartitionData 的嵌套类**。很多请求类型中的数据都是按分区层级进行分组的，因此源码很自然地在这些请求类中创建了同名的嵌套类。我们在查看源码时，一定要注意区分 PartitionData 嵌套类是定义在哪类请求中的，不同类型请求中的 PartitionData 类字段是完全不同的

#### 2.1.2. **PartitionFetchState**

**PartitionStates[PartitionFetchState]**类型的字段.表征分区读取状态的，保存的是分区的已读取位移值和对应的副本状态

注意这里的状态有两个，一个是分区读取状态，一个是副本读取状态。副本读取状态由 ReplicaState 接口表示，如下所示：

```
sealed trait ReplicaState

// 截断中

case object Truncating extends ReplicaState

// 获取中

case object Fetching extends ReplicaState
```

可见，副本读取状态有截断中和获取中两个：当副本执行截断操作时，副本状态被设置成 Truncating；当副本被读取时，副本状态被设置成 Fetching。

而分区读取状态有 3 个，分别是：

可获取，表明副本获取线程当前能够读取数据。

截断中，表明分区副本正在执行截断操作（比如该副本刚刚成为 Follower 副本）。

被推迟，表明副本获取线程获取数据时出现错误，需要等待一段时间后重试。

值得注意的是，分区读取状态中的可获取、截断中与副本读取状态的获取中、截断中两个状态并非严格对应的。换句话说，副本读取状态处于获取中，并不一定表示分区读取状态就是可获取状态。对于分区而言，它是否能够被获取的条件要比副本严格一些。

接下来，我们就来看看这 3 类分区获取状态的源码定义：

```
case class PartitionFetchState(fetchOffset: Long,

  lag: Option[Long],

  currentLeaderEpoch: Int,

  delay: Option[DelayedItem],

  state: ReplicaState) {

  // 分区可获取的条件是副本处于Fetching且未被推迟执行

  def isReadyForFetch: Boolean = state == Fetching && !isDelayed

  // 副本处于ISR的条件：没有lag

  def isReplicaInSync: Boolean = lag.isDefined && lag.get <= 0

  // 分区处于截断中状态的条件：副本处于Truncating状态且未被推迟执行

  def isTruncating: Boolean = state == Truncating && !isDelayed

  // 分区被推迟获取数据的条件：存在未过期的延迟任务

  def isDelayed: Boolean = 

​    delay.exists(_.getDelay(TimeUnit.MILLISECONDS) > 0)

  ......

}
```

这段源码中有 4 个方法，你只要重点了解 isReadyForFetch 和 isTruncating 这两个方法即可。因为副本获取线程做的事情就是这两件：日志截断和消息获取。

至于 isReplicaInSync，它被用于副本限流，出镜率不高。而 isDelayed，是用于判断是否需要推迟获取对应分区的消息。源码会不断地调整那些不需要推迟的分区的读取顺序，以保证读取的公平性。

这个公平性，其实就是在 partitionStates 字段的类型 PartitionStates 类中实现的。这个类是在 clients 工程中定义的。它本质上会接收一组要读取的主题分区，然后以轮询的方式依次读取这些分区以确保公平性。

鉴于咱们这门儿课聚焦于 Broker 端源码，因此，这里我只是简单和你说下这个类的实现原理。如果你想要深入理解这部分内容，可以翻开 clients 端工程的源码，自行去探索下这部分的源码。

```
public class PartitionStates<S> {
    private final LinkedHashMap<TopicPartition, S> map = new LinkedHashMap<>();
    ......
    public void updateAndMoveToEnd(TopicPartition topicPartition, S state) {
      map.remove(topicPartition);
      map.put(topicPartition, state);
      updateSize();
    }
    ......
}
```

前面说过了，PartitionStates 类用轮询的方式来处理要读取的多个分区。那具体是怎么实现的呢？简单来说，就是依靠 LinkedHashMap 数据结构来保存所有主题分区。LinkedHashMap 中的元素有明确的迭代顺序，通常就是元素被插入的顺序。

假设 Kafka 要读取 5 个分区上的消息：A、B、C、D 和 E。如果插入顺序就是 ABCDE，那么自然首先读取分区 A。一旦 A 被读取之后，为了确保各个分区都有同等机会被读取到，代码需要将 A 插入到分区列表的最后一位，这就是 updateAndMoveToEnd 方法要做的事情。

具体来说，就是把 A 从 map 中移除掉，然后再插回去，这样 A 自然就处于列表的最后一位了。大体上，PartitionStates 类就是做这个用的。

### 2.2. 重要方法

 processPartitionData、truncate、buildFetch 和 doWork

#### 2.2.1. **processPartitionData**

用于处理读取回来的消息集合。它是一个抽象方法，因此需要子类实现它的逻辑。具体到 Follower 副本而言， 是由 ReplicaFetcherThread 类实现的

```
protected def processPartitionData(

  topicPartition: TopicPartition,  // 读取哪个分区的数据

  fetchOffset: Long,               // 读取到的最新位移值

  partitionData: FetchData         // 读取到的分区消息数据

): Option[LogAppendInfo]           // 写入已读取消息数据前的元数据
```

我们需要重点关注的字段是，该方法的返回值 Option[LogAppendInfo]：

* 对于 Follower 副本读消息写入日志而言，你可以忽略这里的 Option，因为它肯定会返回具体的 LogAppendInfo 实例，而不会是 None。

* 至于 LogAppendInfo 类，我们在“日志模块”中已经介绍过了。它封装了很多消息数据被写入到日志前的重要元数据信息，比如首条消息的位移值、最后一条消息位移值、最大时间戳等。

#### 2.2.2. **truncate**

```
protected def truncate(
  topicPartition: TopicPartition, // 要对哪个分区下副本执行截断操作
  truncationState: OffsetTruncationState  // Offset + 截断状态
): Unit
```

这里的 OffsetTruncationState 类封装了一个位移值和一个截断完成与否的布尔值状态。它的主要作用是，告诉 Kafka 要把指定分区下副本截断到哪个位移值

#### 2.2.3. **buildFetch** 

```
protected def buildFetch(
  // 一组要读取的分区列表
  // 分区是否可读取取决于PartitionFetchState中的状态
  partitionMap: Map[TopicPartition, PartitionFetchState]): 
// 封装FetchRequest.Builder对象
ResultWithPartitions[Option[ReplicaFetch]]
```

为指定分区构建对应的 FetchRequest.Builder 对象，而该对象是构建 FetchRequest 的核心组件.

Kafka 中任何类型的消息读取，都是通过给指定 Broker 发送 FetchRequest 请求来完成的.

#### 2.2.4. **doWork** 

**串联前面 3 个方法的主要入口方法，也是 AbstractFetcherThread** 类的核心方法。因此，我们要多花点时间，弄明白这些方法是怎么组合在一起共同工作的。

### 2.3. 小结

* AbstractFetcherThread 类

  拉取线程的抽象基类。它定义了公共方法来处理所有拉取线程都要实现的逻辑，如执行截断操作，获取消息等。

* 拉取线程逻辑

  循环执行截断操作和获取数据操作。

* 分区读取状态

  当前，源码定义了 3 类分区读取状态。拉取线程只能拉取处于可读取状态的分区的数据

## 3. ReplicaFetcherThread

### doWork 

#### maybeTruncate

首先，是对分区状态进行分组。既然是做截断操作的，那么该方法操作的就只能是处于截断中状态的分区。代码会判断这些分区是否存在对应的 Leader Epoch 值，并按照有无 Epoch 值进行分组。这就是 fetchTruncatingPartitions 方法做的事情

#### maybeFetch

### processPartitionData 

### 小结

AbstractFetcherThread 线程的 doWork 方法把上一讲提到的 3 个重要方法全部连接在一起，共同完整了拉取线程要执行的逻辑，即日志截断（truncate）+ 日志获取（buildFetch）+ 日志处理（processPartitionData），而其子类 ReplicaFetcherThread 类是真正实现该 3 个方法的地方。如果用一句话归纳起来，那就是：Follower 副本利用 ReplicaFetcherThread 线程实时地从 Leader 副本拉取消息并写入到本地日志，从而实现了与 Leader 副本之间的同步。以下是一些要点：

doWork 方法：拉取线程工作入口方法，联结所有重要的子功能方法，如执行截断操作，获取 Leader 副本消息以及写入本地日志。

truncate 方法：根据 Leader 副本返回的位移值和 Epoch 值执行本地日志的截断操作。

buildFetch 方法：为一组特定分区构建 FetchRequest 对象所需的数据结构。

processPartitionData 方法：处理从 Leader 副本获取到的消息，主要是写入到本地日志中。

## 4. ReplicaManager

ReplicaManager 类是 Kafka Broker 端管理分区和副本对象的重要组件。每个 Broker 在启动的时候，都会创建 ReplicaManager 实例。该实例一旦被创建，就会开始行使副本管理器的职责，对其下辖的 Leader 副本或 Follower 副本进行管理。

* ReplicaManager 类

  副本管理器的具体实现代码，里面定义了读写副本、删除副本消息的方法，以及其他的一些管理方法。

* allPartitions 字段

  承载了 Broker 上保存的所有分区对象数据。ReplicaManager 类通过它实现对分区下副本的管理。

* replicaFetcherManager 字段

  创建 ReplicaFetcherThread 类实例，该线程类实现 Follower 副本向 Leader 副本实时拉取消息的逻辑。

### 4.1. 代码结构

![image-20220711182807936](image-20220711182807936.png) 

* ReplicaManager 类

  它是副本管理器的具体实现代码，里面定义了读写副本、删除副本消息的方法以及其他管理方法。

* ReplicaManager 对象

  ReplicaManager 类的伴生对象，仅仅定义了 3 个常量。

* HostedPartition 及其实现对象

  表示 Broker 本地保存的分区对象的状态。可能的状态包括：不存在状态（None）、在线状态（Online）和离线状态（Offline）。

* FetchPartitionData

  定义获取到的分区数据以及重要元数据信息，如高水位值、Log Start Offset 值等。

* LogReadResult

  表示副本管理器从副本本地日志中**读取**到的消息数据以及相关元数据信息，如高水位值、Log Start Offset 值等。

* LogDeleteRecordsResult

  表示副本管理器执行副本日志**删除**操作后返回的结果信息。

* LogAppendResult

  表示副本管理器执行副本日志**写入**操作后返回的结果信息

FetchPartitionData 和 LogReadResult 很类似，它们的区别在哪里呢？

其实，它们之间的差别非常小。如果翻开源码的话，你会发现，FetchPartitionData 类总共有 8 个字段，而构建 FetchPartitionData 实例的前 7 个字段都是用 LogReadResult 的字段来赋值的。你大致可以认为两者的作用是类似的。只是，FetchPartitionData 还有个字段标识该分区是不是处于重分配中。如果是的话，需要更新特定的 JXM 监控指标。这是这两个类的主要区别。

### 4.2. 字段

ReplicaManager 类构造函数的字段非常多。这里几个关键的字段比较重要

#### 4.2.1. 基本字段

**1.logManager**

这是日志管理器。它负责创建和管理分区的日志对象，里面定义了很多操作日志对象的方法，如 getOrCreateLog 等。

**2.metadataCache**

这是 Broker 端的元数据缓存，保存集群上分区的 Leader、ISR 等信息。注意，它和我们之前说的 Controller 端元数据缓存是有联系的。每台 Broker 上的元数据缓存，是从 Controller 端的元数据缓存异步同步过来的。

**3.logDirFailureChannel**

这是失效日志路径的处理器类。Kafka 1.1 版本新增了对于 JBOD 的支持。这也就是说，Broker 如果配置了多个日志路径，当某个日志路径不可用之后（比如该路径所在的磁盘已满），Broker 能够继续工作。那么，这就需要一整套机制来保证，在出现磁盘 I/O 故障时，Broker 的正常磁盘下的副本能够正常提供服务。

其中，logDirFailureChannel 是暂存失效日志路径的管理器类。我们不用具体学习这个特性的源码，但你最起码要知道，该功能算是 Kafka 提升服务器端高可用性的一个改进。有了它之后，即使 Broker 上的单块磁盘坏掉了，整个 Broker 的服务也不会中断。

**4. 四个 Purgatory 相关的字段**

这 4 个字段是 delayedProducePurgatory、delayedFetchPurgatory、delayedDeleteRecordsPurgatory 和 delayedElectLeaderPurgatory，它们分别管理 4 类延时请求的。其中，前两类我们应该不陌生，就是处理延时生产者请求和延时消费者请求；后面两类是处理延时消息删除请求和延时 Leader 选举请求，属于比较高阶的用法（可以暂时不用理会）。

在副本管理过程中，状态的变更大多都会引发对延时请求的处理，这时候，这些 Purgatory 字段就派上用场了。

只要掌握了刚刚的这些字段，就可以应对接下来的副本管理操作了。其中，最重要的就是 logManager。它是协助副本管理器操作集群副本对象的关键组件

#### 4.2.2. 自定义字段

​	ReplicaManager 类中那些重要的自定义字段。这样的字段大约有 20 个，我们不用花时间逐一学习它们。像 isrExpandRate、isrShrinkRate 这样的字段，我们只看名字，就能知道，它们是衡量 ISR 变化的监控指标

* controllerEpoch

  这个字段的作用是**隔离过期 Controller 发送的请求**。这就是说，老的 Controller 发送的请求不能再被继续处理了。至于如何区分是老 Controller 发送的请求，还是新 Controller 发送的请求，就是**看请求携带的 controllerEpoch 值，是否等于这个字段的值**

  Broker 上接收的所有请求都是由 Kafka I/O 线程处理的，而 I/O 线程可能有多个，因此，这里的 controllerEpoch 字段被声明为 volatile 型，以保证其内存可见性

* allPartitions

  Kafka 没有所谓的分区管理器，ReplicaManager 类承担了一部分分区管理的工作。这里的 allPartitions，就承载了 Broker 上保存的所有分区对象数据，allPartitions 是分区 Partition 对象实例的容器。这里的 HostedPartition 是代表分区状态的类。allPartitions 会将所有分区对象初始化成 Online 状态

* replicaFetcherManager

replicaFetcherManager。它的主要任务是**创建 ReplicaFetcherThread 类实例**。上节课，我们学习了 ReplicaFetcherThread 类的源码，它的主要职责是**帮助 Follower 副本向 Leader 副本拉取消息，并写入到本地日志中**

### 4.3. 副本读写

无论是读取副本还是写入副本，都是通过底层的 Partition 对象完成的。整个 Kafka 的同步机制，本质上就是副本读取 + 副本写入。

这两个重要方法为 appendRecords 和 fetchMessages。

* appendRecords

  向副本写入消息的方法，主要利用 Log 的 append 方法和 Purgatory 机制，共同实现 Follower 副本向 Leader 副本获取消息后的数据同步工作。

* fetchMessages

  从副本读取消息的方法，为普通 Consumer 和 Follower 副本所使用。当它们向 Broker 发送 FETCH 请求时，Broker 上的副本管理器调用该方法从本地日志中获取指定消息。

#### 4.3.1.副本写入

所谓的副本写入，是指向副本底层日志写入消息。在 ReplicaManager 类中，实现副本写入的方法叫 appendRecords。

放眼整个 Kafka 源码世界，需要副本写入的场景有 4 个。

场景一：生产者向 Leader 副本写入消息；

场景二：Follower 副本拉取消息后写入副本；

场景三：消费者组写入组信息；

场景四：事务管理器写入事务信息（包括事务标记、事务元数据等）。

除了第二个场景是直接调用 Partition 对象的方法实现之外，其他 3 个都是调用 appendRecords 来完成的。

该方法将给定一组分区的消息写入到对应的 Leader 副本中，并且根据 PRODUCE 请求中 acks 设置的不同，有选择地等待其他副本写入完成。然后，调用指定的回调逻辑

```
def appendRecords(
  timeout: Long,  // 请求处理超时时间
  requiredAcks: Short,  // 请求acks设置
  internalTopicsAllowed: Boolean,  // 是否允许写入内部主题
  origin: AppendOrigin,  // 写入方来源
  entriesPerPartition: Map[TopicPartition, MemoryRecords], // 待写入消息
  // 回调逻辑 
  responseCallback: Map[TopicPartition, PartitionResponse] => Unit,
  delayedProduceLock: Option[Lock] = None,
  recordConversionStatsCallback: 
    Map[TopicPartition, RecordConversionStats] => Unit = _ => ())
  : Unit = {
  ......
}
```

![image-20220712150226901](image-20220712150226901.png) 

* **appendToLocalLog**

  该方法主要就是利用 Partition 的 appendRecordsToLeader 方法写入消息集合。最终会appendAsLeader 方法写入本地日志的。总体来说，appendToLocalLog 的逻辑不复杂

* **delayedProduceRequestRequired**

  ```
  private def delayedProduceRequestRequired(
    requiredAcks: Short,
    entriesPerPartition: Map[TopicPartition, MemoryRecords],
    localProduceResults: Map[TopicPartition, LogAppendResult]): Boolean = {
    requiredAcks == -1 && entriesPerPartition.nonEmpty && 
      localProduceResults.values.count(_.exception.isDefined) < entriesPerPartition.size
  }
  ```

  如果所有分区的数据写入都不成功，就表明可能出现了很严重的错误，此时，比较明智的做法是不再等待，而是直接返回错误给发送方。相反地，如果有部分分区成功写入，而部分分区写入失败了，就表明可能是由偶发的瞬时错误导致的。此时，不妨将本次写入请求放入 Purgatory，再给它一个重试的机会

### 4.4. 副本读取

​	在 ReplicaManager 类中，负责读取副本数据的方法是 fetchMessages。不论是 Java 消费者 API，还是 Follower 副本，它们拉取消息的主要途径都是向 Broker 发送 FETCH 请求，Broker 端接收到该请求后，调用 fetchMessages 方法从底层的 Leader 副本取出消息。

​	和 appendRecords 方法类似，fetchMessages 方法也可能会延时处理 FETCH 请求，因为 Broker 端必须要累积足够多的数据之后，才会返回 Response 给请求发送方。

​	可以看一下下面的这张流程图，它展示了 fetchMessages 方法的主要逻辑

![image-20220712151406356](image-20220712151406356.png) 

#### 4.4.1. 方法定义

```
def fetchMessages(timeout: Long,
                  replicaId: Int,
                  fetchMinBytes: Int,
                  fetchMaxBytes: Int,
                  hardMaxBytesLimit: Boolean,
                  fetchInfos: Seq[(TopicPartition, PartitionData)],
                  quota: ReplicaQuota,
                  responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit,
                  isolationLevel: IsolationLevel,
                  clientMetadata: Option[ClientMetadata]): Unit = {
  ......
}
```

**timeout**：请求处理超时时间。对于消费者而言，该值就是 request.timeout.ms 参数值；对于 Follower 副本而言，该值是 Broker 端参数 replica.fetch.wait.max.ms 的值。

**replicaId**：副本 ID。对于消费者而言，该参数值是 -1；对于 Follower 副本而言，该值就是 Follower 副本所在的 Broker ID。

**fetchMinBytes & fetchMaxBytes**：能够获取的最小字节数和最大字节数。对于消费者而言，它们分别对应于 Consumer 端参数 fetch.min.bytes 和 fetch.max.bytes 值；对于 Follower 副本而言，它们分别对应于 Broker 端参数 replica.fetch.min.bytes 和 replica.fetch.max.bytes 值。

**hardMaxBytesLimit**：对能否超过最大字节数做硬限制。如果 hardMaxBytesLimit=True，就表示，读取请求返回的数据字节数绝不允许超过最大字节数。

**fetchInfos**：规定了读取分区的信息，比如要读取哪些分区、从这些分区的哪个位移值开始读、最多可以读多少字节，等等。

**quota**：这是一个配额控制类，主要是为了判断是否需要在读取的过程中做限速控制。

**responseCallback**：Response 回调逻辑函数。当请求被处理完成后，调用该方法执行收尾逻辑。

#### 4.4.2. 核心逻辑

整个方法的代码分成两部分：第一部分是读取本地日志；第二部分是根据读取结果确定 Response。

* 读取本地日志

首先会判断，读取消息的请求方到底是 Follower 副本，还是普通的 Consumer。判断的依据就是看 **replicaId 字段是否大于 0**。Consumer 的 replicaId 是 -1，而 Follower 副本的则是大于 0 的数。一旦确定了请求方，代码就能确定可读取范围。

这里的 fetchIsolation 是读取隔离级别的意思。对于 Follower 副本而言，它能读取到 Leader 副本 LEO 值以下的所有消息；对于普通 Consumer 而言，它只能“看到”Leader 副本高水位值以下的消息。

待确定了可读取范围后，fetchMessages 方法会调用它的内部方法 **readFromLog**，读取本地日志上的消息数据，并将结果赋值给 logReadResults 变量。readFromLog 方法的主要实现是调用 readFromLocalLog 方法，而后者就是在待读取分区上依次调用其日志对象的 read 方法执行实际的消息读取

* 确定 Response

  统计可读取的总字节数，之后，判断此时是否能够立即返回 Reponse

  只要满足以下 4 个条件中的任意一个即可：

  * 请求没有设置超时时间，说明请求方想让请求被处理后立即返回；

  * 未获取到任何数据；

  * 已累积到足够多数据；

  * 读取过程中出错。

如果这 4 个条件一个都不满足，就需要进行延时处理了。具体来说，就是构建 DelayedFetch 对象，然后把该延时对象交由 delayedFetchPurgatory 后续自动处理。

至此，关于副本管理器读写副本的两个方法 appendRecords 和 fetchMessages，我们就学完了。本质上，它们在底层分别调用 Log 的 append 和 read 方法，以实现本地日志的读写操作。当完成读写操作之后，这两个方法还定义了延时处理的条件。一旦发现满足了延时处理的条件，就交给对应的 Purgatory 进行处理。

从这两个方法中，我们已经看到了之前课程中单个组件融合在一起的趋势。就像我在开篇词里面说的，虽然我们学习单个源码文件的顺序是自上而下，但串联 Kafka 主要组件功能的路径却是自下而上

### 4.5. 副本管理器

ReplicaManager 类具有分区和副本管理功能，以及 ISR 管理。

* 分区 / 副本管理

  ReplicaManager 类的核心功能是，应对 Broker 端接收的 LeaderAndIsrRequest 请求，并将请求中的分区信息抽取出来，让所在 Broker 执行相应的动作。

* becomeLeaderOrFollower 方法

  它是应对 LeaderAndIsrRequest 请求的入口方法。它会将请求中的分区划分成两组，分别调用 makeLeaders 和 makeFollowers 方法。

* makeLeaders 方法

  它的作用是让 Broker 成为指定分区 Leader 副本。

* makeFollowers 方法

  它的作用是让 Broker 成为指定分区 Follower 副本的方法。

* ISR 管理

  ReplicaManager 类提供了 ISR 收缩和定期传播 ISR 变更通知的功能

副本管理器是如何管理副本的。这里的副本，涵盖了广义副本对象的方方面面，包括副本和分区对象、副本位移值和 ISR 管理

#### 4.5.1. 分区及副本管理

ReplicaManager 管理它们的方式，是通过字段 allPartitions 来实现的

每个 ReplicaManager 实例都维护了所在 Broker 上保存的所有分区对象，而每个分区对象 Partition 下面又定义了一组副本对象 Replica。通过这样的层级关系，副本管理器实现了对于分区的直接管理和对副本对象的间接管理。应该这样说，**ReplicaManager 通过直接操作分区对象来间接管理下属的副本对象**

对于一个 Broker 而言，它管理下辖的分区和副本对象的主要方式，就是要确定在它保存的这些副本中，哪些是 Leader 副本、哪些是 Follower 副本。

这些划分可不是一成不变的，而是随着时间的推移不断变化的。比如说，这个时刻 Broker 是分区 A 的 Leader 副本、分区 B 的 Follower 副本，但在接下来的某个时刻，Broker 很可能变成分区 A 的 Follower 副本、分区 B 的 Leader 副本。

而这些变更是通过 Controller 给 Broker 发送 LeaderAndIsrRequest 请求来实现的。当 Broker 端收到这类请求后，会调用副本管理器的 becomeLeaderOrFollower 方法来处理，并依次执行“成为 Leader 副本”和“成为 Follower 副本”的逻辑，令当前 Broker 互换分区 A、B 副本的角色。

* becomeLeaderOrFollower 方法

  简单来说，它就是告诉接收该请求的 Broker：在我传给你的这些分区中，哪些分区的 Leader 副本在你这里；哪些分区的 Follower 副本在你这里。becomeLeaderOrFollower 方法，就是具体处理 LeaderAndIsrRequest 请求的地方，同时也是副本管理器添加分区的地方。一共有三 3 个部分向你介绍，分别是处理 Controller Epoch 事宜、执行成为 Leader 和 Follower 的逻辑以及构造 Response。	

  * makeLeaders 

  * makeFollowers 

#### 4.5.2. ISR 管理

## 5. MetadataCache

​	这里的 MetadataCache 是指 Broker 上的元数据缓存，这些数据是 Controller 通过 UpdateMetadataRequest 请求发送给 Broker 的。换句话说，Controller 实现了一个异步更新机制，能够将最新的集群信息广播给所有 Broker

​	Broker 就能够及时**响应客户端发送的元数据请求，也就是处理 Metadata 请求**。Metadata 请求是为数不多的能够被集群任意 Broker 处理的请求类型之一，也就是说，客户端程序能够随意地向任何一个 Broker 发送 Metadata 请求，去获取集群的元数据信息，这完全得益于 MetadataCache 的存在

​	Kafka 的一些重要组件会用到这部分数据。比如副本管理器会使用它来获取 Broker 的节点信息，事务管理器会使用它来获取分区 Leader 副本的信息

​	MetadataCache 是每台 Broker 上都会保存的数据。Kafka 通过异步更新机制来保证所有 Broker 上的元数据缓存实现最终一致性。

​	在实际使用的过程中，你可能会碰到这样一种场景：集群明明新创建了主题，但是消费者端却报错说“找不到主题信息”，这种情况通常只持续很短的时间。不知道你是否思考过这里面的原因，其实说白了，很简单，这就是因为元数据是异步同步的，因此，在某一时刻，某些 Broker 尚未更新元数据，它们保存的数据就是过期的元数据，无法识别最新的主题

### 5.1. MetadataCache 类

MetadataCache 的实例化是在 Kafka Broker 启动时完成的，具体的调用发生在 KafkaServer 类的 startup 方法中

一旦实例被成功创建，就会被 Kafka 的 4 个组件使用。

* KafkaApis

  它是执行 Kafka 各类请求逻辑的地方。该类大量使用 MetadataCache 中的主题分区和 Broker 数据，执行主题相关的判断与比较，以及获取 Broker 信息。

* AdminManager

  这是 Kafka 定义的专门用于管理主题的管理器，里面定义了很多与主题相关的方法。同 KafkaApis 类似，它会用到 MetadataCache 中的主题信息和 Broker 数据，以获取主题和 Broker 列表。

* ReplicaManager

  副本管理器。它需要获取主题分区和 Broker 数据，同时还会更新 MetadataCache。

* TransactionCoordinator

  这是管理 Kafka 事务的协调者组件，它需要用到 MetadataCache 中的主题分区的 Leader 副本所在的 Broker 数据，向指定 Broker 发送事务标记

### 5.2. 类定义及字段

MetadataCache 类构造函数只需要一个参数：**brokerId**，即 Broker 的 ID 序号。除了这个参数，该类还定义了 4 个字段。

partitionMetadataLock 字段是保护它写入的锁对象，logIndent 和 stateChangeLogger 字段仅仅用于日志输出，而 metadataSnapshot 字段保存了实际的元数据信息，它是 MetadataCache 类中最重要的字段，我们要重点关注一下它。

该字段的类型是 MetadataSnapshot 类，该类是 MetadataCache 中定义的一个嵌套类

```

case class MetadataSnapshot(partitionStates: mutable.AnyRefMap
  [String, mutable.LongMap[UpdateMetadataPartitionState]],
  controllerId: Option[Int],
  aliveBrokers: mutable.LongMap[Broker],
  aliveNodes: mutable.LongMap[collection.Map[ListenerName, Node]])
```

从源码可知，它是一个 case 类，相当于 Java 中配齐了 Getter 方法的 POJO 类。同时，它也是一个不可变类（Immutable Class）。正因为它的不可变性，其字段值是不允许修改的，我们只能重新创建一个新的实例，来保存更新后的字段值。

我们看下它的各个字段的含义。

**partitionStates**：这是一个 Map 类型。Key 是主题名称，Value 又是一个 Map 类型，其 Key 是分区号，Value 是一个 UpdateMetadataPartitionState 类型的字段。UpdateMetadataPartitionState 类型是 UpdateMetadataRequest 请求内部所需的数据结构。一会儿我们再说这个类型都有哪些数据。

**controllerId**：Controller 所在 Broker 的 ID。

**aliveBrokers**：当前集群中所有存活着的 Broker 对象列表。

**aliveNodes**：这也是一个 Map 的 Map 类型。其 Key 是 Broker ID 序号，Value 是 Map 类型，其 Key 是 ListenerName，即 Broker 监听器类型，而 Value 是 Broker 节点对象

#### 5.2.1. UpdateMetadataPartitionState 

UpdateMetadataPartitionState 类型。这个类型的源码是由 Kafka 工程自动生成的。UpdateMetadataRequest 请求所需的字段用 JSON 格式表示，由 Kafka 的 generator 工程负责为 JSON 格式自动生成对应的 Java 文件，生成的类是一个 POJO 类，其定义如下：

```
static public class UpdateMetadataPartitionState implements Message {
    private String topicName;     // 主题名称
    private int partitionIndex;   // 分区号
    private int controllerEpoch;  // Controller Epoch值
    private int leader;           // Leader副本所在Broker ID
    private int leaderEpoch;      // Leader Epoch值
    private List<Integer> isr;    // ISR列表
    private int zkVersion;        // ZooKeeper节点Stat统计信息中的版本号
    private List<Integer> replicas;  // 副本列表
    private List<Integer> offlineReplicas;  // 离线副本列表
    private List<RawTaggedField> _unknownTaggedFields; // 未知字段列表
    ......
}
```

UpdateMetadataPartitionState 类的字段信息非常丰富，它包含了一个主题分区非常详尽的数据，从主题名称、分区号、Leader 副本、ISR 列表到 Controller Epoch、ZooKeeper 版本号等信息，一应俱全。从宏观角度来看，Kafka 集群元数据由主题数据和 Broker 数据两部分构成

#### 5.2.2. 重要方法

 MetadataCache 类的方法大致分为三大类：

* 判断类；

  就是判断给定主题或主题分区是否包含在元数据缓存中的方法。MetadataCache 类提供了两个判断类的方法，方法名都是 **contains**，只是输入参数不同

  第一个 contains 方法用于判断给定主题是否包含在元数据缓存中，比较简单，只需要判断 metadataSnapshot 中 partitionStates 的所有 Key 是否包含指定主题就行了。

  第二个 contains 方法相对复杂一点。它首先要从 metadataSnapshot 中获取指定主题分区的分区数据信息，然后根据分区数据是否存在，来判断给定主题分区是否包含在元数据缓存中。

* 获取类；

  MetadataCache 类的 getXXX 方法非常多，其中，比较有代表性的是 getAllTopics 方法、getAllPartitions 方法和 getPartitionReplicaEndpoints，它们分别是获取主题、分区和副本对象的方法

* 更新类

  元数据缓存只有被更新了，才能被读取。从某种程度上说，它是后续所有 getXXX 方法的前提条件

  updateMetadata 方法的主要逻辑，就是**读取 UpdateMetadataRequest 请求中的分区数据，然后更新本地元数据缓存**

### 5.3. 小结

元数据缓存类。该类保存了当前集群上的主题分区详细数据和 Broker 数据。每台 Broker 都维护了一个 MetadataCache 实例。Controller 通过给 Broker 发送 UpdateMetadataRequest 请求的方式，来异步更新这部分缓存数据

* MetadataCache 类

  Broker 元数据缓存类，保存了分区详细数据和 Broker 节点数据。

* 四大调用方

  分别是 ReplicaManager、KafkaApis、TransactionCoordinator 和 AdminManager。

* updateMetadata 方法

  Controller 给 Broker 发送 UpdateMetadataRequest 请求时，触发更新
