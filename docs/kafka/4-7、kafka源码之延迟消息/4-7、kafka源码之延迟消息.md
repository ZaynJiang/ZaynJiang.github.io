## 1. 开头

延迟（Delayed Operation）

## 2. 时间轮

Kafka 定义了 4 个类来构建整套分层时间轮体系。

* TimerTask 类

  建模 Kafka 延时请求。它是一个 Runnable 类，Kafka 使用一个单独线程异步添加延时请求到时间轮。

* TimerTaskEntry 类

  建模时间轮 Bucket 下延时请求链表的元素类型，封装了 TimerTask 对象和定时任务的过期时间戳信息。

* TimerTaskList 类

  建模时间轮 Bucket 下的延时请求双向循环链表，提供 O(1) 时间复杂度的请求插入和删除。

* TimingWheel 类

  建模时间轮类型，统一管理下辖的所有 Bucket 以及定时任务

### TimerTask 类

### TimerTaskEntry 类

### TimerTaskList 类

### TimingWheel 类

## 3. DelayedOperation

​	分层时间轮代码和 Kafka 概念并无直接的关联，Kakfka通过Timer 和 SystemTimer将主题、分区、副本这样的概念和延迟请求关联起来了

### Timer 

Timer 接口定义了 4 个方法。

add 方法：将给定的定时任务插入到时间轮上，等待后续延迟执行。

advanceClock 方法：向前推进时钟，执行已达过期时间的延迟任务。

size 方法：获取当前总定时任务数。

shutdown 方法：关闭该定时器。

其中，最重要的两个方法是 **add** 和 **advanceClock**，它们是**完成延迟请求处理的关键步骤**。

### SystemTimer

#### 属性

​	SystemTimer 类是 Timer 接口的实现类。它是一个定时器类，封装了分层时间轮对象，为 Purgatory 提供延迟请求管理功能。所谓的 Purgatory，就是保存延迟请求的缓冲区。也就是说，它保存的是因为不满足条件而无法完成，但是又没有超时的请求。下面，我们从定义和方法两个维度来学习 SystemTimer 类。

 3 个比较重要的。这 3 个字段与时间轮都是强相关的。

* **delayQueue 字段**

  它保存了该定时器下管理的所有 Bucket 对象。因为是 DelayQueue，所以只有在 Bucket 过期后，才能从该队列中获取到。SystemTimer 类的 advanceClock 方法正是依靠了这个特性向前驱动时钟。关于这一点，一会儿我们详细说。

* **timingWheel**

  TimingWheel 是实现分层时间轮的类。SystemTimer 类依靠它来操作分层时间轮。

* **taskExecutor**

  它是单线程的线程池，用于异步执行提交的定时任务逻辑

#### 方法

该类总共定义了 6 个方法：

* add

* addTimerTaskEntry

* reinsert

* advanceClock

  advanceClock 方法要做的事情，就是遍历 delayQueue 中的所有 Bucket，并将时间轮的时钟依次推进到它们的过期时间点，令它们过期。然后，再将这些 Bucket 下的所有定时任务全部重新插入回时间轮。

* size 和 shutdow。

SystemTimer 类实现了 Timer 接口的方法，**它封装了底层的分层时间轮，为上层调用方提供了便捷的方法来操作时间轮**

### DelayedOperationPurgatory 

SystemTimer 类实现了 Timer 接口的方法，它的上层调用方是谁呢？答案就是 DelayedOperationPurgatory 类。

 DelayedOperationPurgatory 是一个泛型类，它的类型参数恰恰就是 DelayedOperation。

#### 定义

​	DelayedOperation 类是一个抽象类，它的构造函数中只需要传入一个超时时间即可。这个超时时间通常是**客户端发出请求的超时时间**，也就是客户端参数 **request.timeout.ms** 的值。这个类实现了上节课学到的 TimerTask 接口，因此，作为一个建模延迟操作的类，它自动继承了 TimerTask 接口的 cancel 方法，支持延迟操作的取消，以及 TimerTaskEntry 的 Getter 和 Setter 方法，支持将延迟操作绑定到时间轮相应 Bucket 下的某个链表元素上.

#### 关键方法

DelayedOperation 类额外定义了两个字段：

* **completed** 

* **tryCompletePending**。

关键的方法为：

* forceComplete

强制完成延迟操作，不管它是否满足完成条件。每当操作满足完成条件或已经过期了，就需要调用该方法完成该操作。

* isCompleted

检查延迟操作是否已经完成。源码使用这个方法来决定后续如何处理该操作。比如如果操作已经完成了，那么通常需要取消该操作。

* onExpiration

强制完成之后执行的过期逻辑回调方法。只有真正完成操作的那个线程才有资格调用这个方法。

* onComplete

完成延迟操作所需的处理逻辑。这个方法只会在 forceComplete 方法中被调用。

* tryComplete

尝试完成延迟操作的顶层方法，内部会调用 forceComplete 方法。

* maybeTryComplete

  在以前会有阻塞问题：当多个线程同时检查某个延迟操作是否满足完成条件时，如果其中一个线程持有了锁（也就是上面的 lock 字段），然后执行条件检查，会发现不满足完成条件。而与此同时，另一个线程执行检查时却发现条件满足了，但是这个线程又没有拿到锁，此时，该延迟操作将永远不会有再次被检查的机会，会导致最终超时

  解决方案增加了tryCompletePending字段

  线程安全版本的 tryComplete 方法。这个方法其实是社区后来才加入的，不过已经慢慢地取代了 tryComplete，现在外部代码调用的都是这个方法了。主要目的为规避因多线程访问产生锁争用导致线程阻塞。

```
private[server] def maybeTryComplete(): Boolean = {
  var retry = false  // 是否需要重试
  var done = false   // 延迟操作是否已完成
  do {
    if (lock.tryLock()) {   // 尝试获取锁对象
      try {
        tryCompletePending.set(false)
        done = tryComplete()
      } finally {
        lock.unlock()
      }
      // 运行到这里的线程持有锁，其他线程只能运行else分支的代码
      // 如果其他线程将maybeTryComplete设置为true，那么retry=true
      // 这就相当于其他线程给了本线程重试的机会
      retry = tryCompletePending.get()
    } else {
      // 运行到这里的线程没有拿到锁
      // 设置tryCompletePending=true给持有锁的线程一个重试的机会
      retry = !tryCompletePending.getAndSet(true)
    }
  } while (!isCompleted && retry)
  done
}
```

* run

调用延迟操作超时后的过期逻辑，也就是组合调用 forceComplete + onExpiration

#### DelayedOperationPurgatory 

该类是实现 Purgatory 的地方。从代码结构上看，它是一个 Scala 伴生对象。也就是说，源码文件同时定义了 DelayedOperationPurgatory Object 和 Class。Object 中仅仅定义了 apply 工厂方法和一个名为 Shards 的字段，这个字段是 DelayedOperationPurgatory 监控列表的数组长度信息。因此，我们还是重点学习 DelayedOperationPurgatory Class 的源码。

前面说过，DelayedOperationPurgatory 类是一个泛型类，它的参数类型是 DelayedOperation 的具体子类。因此，通常情况下，每一类延迟请求都对应于一个 DelayedOperationPurgatory 实例。这些实例一般都保存在上层的管理器中。比如，与消费者组相关的心跳请求、加入组请求的 Purgatory 实例，就保存在 GroupCoordinator 组件中，而与生产者相关的 PRODUCE 请求的 Purgatory 实例，被保存在分区对象或副本状态机中。

```
final class DelayedOperationPurgatory[T <: DelayedOperation](
  purgatoryName: String, 
  timeoutTimer: Timer, 
  brokerId: Int = 0, 
  purgeInterval: Int = 1000, 
  reaperEnabled: Boolean = true, 
  timerEnabled: Boolean = true) extends Logging with KafkaMetricsGroup {
  ......
}
```

DelayedOperationPurgatory 还定义了两个内置类，分别是 Watchers 和 WatcherList。

该类的两个重要方法：

* tryCompleteElseWatch 
* checkAndComplete 方法。

前者的作用是**检查操作是否能够完成**，如果不能的话，就把它加入到对应 Key 所在的 WatcherList 中

先尝试完成请求，如果无法完成，则把它加入到 WatcherList 中进行监控。具体来说，tryCompleteElseWatch 调用 tryComplete 方法，尝试完成延迟请求，如果返回结果是 true，就说明执行 tryCompleteElseWatch 方法的线程正常地完成了该延迟请求，也就不需要再添加到 WatcherList 了，直接返回 true 就行了。

否则的话，代码会遍历所有要监控的 Key，再次查看请求的完成状态。如果已经完成，就说明是被其他线程完成的，返回 false；如果依然无法完成，则将该请求加入到 Key 所在的 WatcherList 中，等待后续完成。同时，设置 watchCreated 标记，表明该任务已经被加入到 WatcherList 以及更新 Purgatory 中总请求

待遍历完所有 Key 之后，源码会再次尝试完成该延迟请求，如果完成了，就返回 true，否则就取消该请求，然后将其加入到过期队列，最后返回 false。

总的来看，你要掌握这个方法要做的两个事情：

先尝试完成延迟请求；

如果不行，就加入到 WatcherList，等待后面再试

### 小结

 Timer 接口及其实现类 SystemTimer、DelayedOperation 类以及 DelayedOperationPurgatory 类。你基本上可以认为，它们是逐级被调用的关系，即 **DelayedOperation 调用 SystemTimer 类，DelayedOperationPurgatory 管理 DelayedOperation**。它们共同实现了 Broker 端对于延迟请求的处理，基本思想就是，**能立即完成的请求马上完成，否则就放入到名为 Purgatory 的缓冲区中**。后续，DelayedOperationPurgatory 类的方法会自动地处理这些延迟请求

SystemTimer 类：Kafka 定义的定时器类，封装了底层分层时间轮，实现了时间轮 Bucket 的管理以及时钟向前推进功能。它是实现延迟请求后续被自动处理的基础。

DelayedOperation 类：延迟请求的高阶抽象类，提供了完成请求以及请求完成和过期后的回调逻辑实现。

DelayedOperationPurgatory 类：Purgatory 实现类，该类定义了 WatcherList 对象以及对 WatcherList 的操作方法，而 WatcherList 是实现延迟请求后续自动处理的关键数据结构。