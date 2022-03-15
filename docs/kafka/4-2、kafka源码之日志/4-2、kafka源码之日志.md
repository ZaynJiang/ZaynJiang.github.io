## 1. 概览

![](kafka日志结构概览.png) 

### 1.1. 日志结构	

​	一个 Kafka 主题有很多分区，每个分区就对应一个 Log 对象，在物理磁盘上则对应于一个子目录。

​	比如我们可以创建一个双分区的主题 test-topic，那么，Kafka 在磁盘上会创建两个子目录：test-topic-0 和 test-topic-1。每个子目录下存在多组日志段，也就是多组.log、.index、.timeindex 文件组合，只不过文件名不同，因为每个日志段的起始位移不同

### 1.2. 日志段结构

​	Kafka 日志对象由多个日志段对象组成，而每个日志段对象会在磁盘上创建一组文件：

* 消息日志文件（.log）

* 位移索引文件（.index）

* 时间戳索引文件（.timeindex）

* 已中止（Aborted）事务的索引文件（.txnindex）

  没有使用 Kafka 事务，已中止事务的索引文件是不会被创建

因此每个日志段由两个核心组件构成：日志、索引。每个日志段都有一个起始位移值（Base Offset），而该位移值是此日志段所有消息中最小的位移值，同时，该值却又比前面任何日志段中消息的位移值都大

## 2. 日志段分析

日志段源码位于 Kafka 的 core 工程下，具体文件位置是 core/src/main/scala/kafka/log/LogSegment.scala.如下为日志段类声明：

```
@nonthreadsafe
class LogSegment private[log] (val log: FileRecords,
                               val lazyOffsetIndex: LazyIndex[OffsetIndex],
                               val lazyTimeIndex: LazyIndex[TimeIndex],
                               val txnIndex: TransactionIndex,
                               val baseOffset: Long,
                               val indexIntervalBytes: Int,
                               val rollJitterMs: Long,
                               val time: Time) extends Logging
```

* FileRecords

  实际保存 Kafka 消息

* lazyOffsetIndex、lazyTimeIndex、txnIndex

  ​	分别对应于刚才所说的 3 个索引文件。不过，在实现方式上，前两种使用了延迟初始化的原理，降低了初始化时间成本

* baseOffset

  ​	在磁盘上看到的文件名就是 baseOffset 的值。每个 LogSegment 对象实例一旦被创建，它的起始位移就是固定的了，不能再被更改

* indexIntervalBytes

  控制了日志段对象新增索引项的频率，默认情况下，日志段至少新写入 4KB 的消息数据才会新增一条索引项

* rollJitterMs

  rollJitterMs 是日志段对象新增倒计时的“扰动值”。因为目前 Broker 端日志段新增倒计时是全局设置，这就是说，在未来的某个时刻可能同时创建多个日志段对象，这将极大地增加物理磁盘 I/O 压力。有了 rollJitterMs 值的干扰，每个新增日志段在创建时会彼此岔开一小段时间，这样可以缓解物理磁盘的 I/O 负载瓶颈

* time

  用于统计计时的一个实现类,在 Kafka 源码中普遍出现

### 2.1. append 方法

append 方法接收 4 个参数:

* 最大位移值
* 最大时间戳
* 最大时间戳对应消息的位移
* 真正要写入的消息集合

![](kafka添加消息逻辑.png) 

* **第一步：**

  在源码中，首先调用 log.sizeInBytes 方法判断该日志段是否为空，如果是空的话， Kafka 需要记录要写入消息集合的最大时间戳，并将其作为后面新增日志段倒计时的依据。

  ```
  val physicalPosition = log.sizeInBytes()
  if (physicalPosition == 0)
  rollingBasedTimestamp = Some(largestTimestamp)
  ```

* **第二步：**

  代码调用 ensureOffsetInRange 方法确保输入参数最大位移值是合法的

  标准就是看它与日志段起始位移的差值是否在整数范围内，即 largestOffset - baseOffset 的值是不是 介于 [0，Int.MAXVALUE] 之间。

  在极个别的情况下，这个差值可能会越界，这时， append 方法就会抛出异常，阻止后续的消息写入。一旦你碰到这个问题，你需要做的是升级你的 Kafka 版本，因为这是由已知的 Bug 导致的。

* **第三步：**

  append 方法调用 FileRecords 的 append 方法执行真正的写入。

  它的工作是将内存中的消息对象写入到操作系统的页缓存

* **第四步：**

  更新日志段的最大时间戳以及最大时间戳所属消息的位移值属性。

  每个日志段都要保存当前最大时间戳信息和所属消息的位移信息。

  Broker 端提供定期删除日志的功能, 判断的依据就是最大时间戳这个值；

  而最大时间戳对应的消息的位移值则用于时间戳索引项。

  **时间戳索引项保存时间戳与消息位移的对应关系**。

  Kafka 会更新并保存这组对应关系。

* **第五步：**

  更新索引项和写入的字节数了。

  日志段每写入 4KB 数据就要写入一个索引项。

  当已写入字节数超过了 4KB 之后，append 方法会调用索引对象的 append 方法新增索引项

  同时清空已写入字节数，以备下次重新累积计算

````
 def append(largestOffset: Long, largestTimestamp: Long, shallowOffsetOfMaxTimestamp: Long, records: MemoryRecords): Unit = {
    if (records.sizeInBytes > 0) {
      trace(s"Inserting ${records.sizeInBytes} bytes at end offset $largestOffset at position ${log.sizeInBytes} " +
            s"with largest timestamp $largestTimestamp at shallow offset $shallowOffsetOfMaxTimestamp")
      val physicalPosition = log.sizeInBytes()
      if (physicalPosition == 0)
        rollingBasedTimestamp = Some(largestTimestamp)

      ensureOffsetInRange(largestOffset)

      // append the messages
      val appendedBytes = log.append(records)
      trace(s"Appended $appendedBytes to ${log.file} at end offset $largestOffset")
      // Update the in memory max timestamp and corresponding offset.
      if (largestTimestamp > maxTimestampSoFar) {
        maxTimestampSoFar = largestTimestamp
        offsetOfMaxTimestampSoFar = shallowOffsetOfMaxTimestamp
      }
      // append an entry to the index (if needed)
      if (bytesSinceLastIndexEntry > indexIntervalBytes) {
        offsetIndex.append(largestOffset, physicalPosition)
        timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar)
        bytesSinceLastIndexEntry = 0
      }
      bytesSinceLastIndexEntry += records.sizeInBytes
    }
  }
````

### 2.2. read方法

read 方法接收 4 个输入参数：

* startOffset：要读取的第一条消息的位移；

* maxSize：能读取的最大字节数；

* maxPosition ：能读到的最大文件位置；

* minOneMessage：是否允许在消息体过大时至少返回第一条消息

  参数为 true 时，即使出现消息体字节数超过了 maxSize 的情形，read 方法依然能返回至少一条消息。

该方法的逻辑：

* 查找索引确定读取的物理文件的位置

  调用 translateOffset 方法定位要读取的起始文件位置 （startPosition）。输入参数 startOffset 仅仅是位移值，Kafka 需要根据索引信息找到对应的物理文件位置才能开始读取消息。

* 计算读取的总字节数

  确定了读取起始位置，日志段代码需要根据这部分信息以及 maxSize 和 maxPosition 参数共同计算要读取的总字节数。举个例子，假设 maxSize=100，maxPosition=300，startPosition=250，那么 read 方法只能读取 50 字节，因为 maxPosition - startPosition = 50。我们把它和 maxSize 参数相比较，其中的最小值就是最终能够读取的总字节数

* 读取消息

  调用 FileRecords 的 slice 方法，从指定位置读取指定大小的消息集合

```
  def read(startOffset: Long,
           maxSize: Int,
           maxPosition: Long = size,
           minOneMessage: Boolean = false): FetchDataInfo = {
    if (maxSize < 0)
      throw new IllegalArgumentException(s"Invalid max size $maxSize for log read from segment $log")

    val startOffsetAndSize = translateOffset(startOffset)

    // if the start position is already off the end of the log, return null
    if (startOffsetAndSize == null)
      return null

    val startPosition = startOffsetAndSize.position
    val offsetMetadata = LogOffsetMetadata(startOffset, this.baseOffset, startPosition)

    val adjustedMaxSize =
      if (minOneMessage) math.max(maxSize, startOffsetAndSize.size)
      else maxSize

    // return a log segment but with zero size in the case below
    if (adjustedMaxSize == 0)
      return FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY)

    // calculate the length of the message set to read based on whether or not they gave us a maxOffset
    val fetchSize: Int = min((maxPosition - startPosition).toInt, adjustedMaxSize)

    FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize),
      firstEntryIncomplete = adjustedMaxSize < startOffsetAndSize.size)
  }
```

### 2.3. recover 方法

​	Broker 在启动时会从磁盘上加载所有日志段信息到内存中，并创建相应的 LogSegment 对象实例。在这个过程中，它需要执行一系列的操作。总的来说就是，读取日志段文件，然后重建索引文件。当我们的环境上有很多日志段文件，Broker 重启很慢，现在就知道了，这是因为 Kafka 在执行 recover 的过程中需要读取大量的磁盘文件。

![](kafka日志段恢复逻辑.png) 

* recover 开始时，代码依次调用索引对象的 reset 方法清空所有的索引文件

* 开始遍历日志段中的所有消息集合或消息批次（RecordBatch）

  对于读取到的每个消息集合，日志段必须要确保它们是合法的，这主要体现在两个方面：

  * 必须要符合 Kafka 定义的二进制格式；

  * 集合中最后一条消息的位移值不能越界，即它与日志段起始位移的差值必须是一个正整数值。

* 更新遍历过程中观测到的最大时间戳以及所属消息的位移值。这两个数据用于后续构建索引项。

  不断累加当前已读取的消息字节数，并根据该值有条件地写入索引项。

* 最后是更新事务型 Producer 的状态以及 Leader Epoch 缓存。

  这两个并不是理解 Kafka 日志结构所必需的组件，可以忽略它们。

* 遍历执行完成后，Kafka 会将日志段当前总字节数和刚刚累加的已读取字节数进行比较：
  * 如果发现前者比后者大，说明日志段写入了一些非法消息，需要执行截断操作，将日志段大小调整回合法的数值。
  *  Kafka 还必须相应地调整索引文件的大小。
* 日志段恢复的操作也就宣告结束了 

### 2.4. 小结

![](日志段小结.png) 

## 3. 日志

​	日志是日志段的容器，里面定义了很多管理日志段的操作。

![](log相关的类和对象.png) 

