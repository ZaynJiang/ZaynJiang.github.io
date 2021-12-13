## 1. 开头  


## 2. Actor模型
&emsp;&emsp;Actor 模型本质上是一种计算模型，基本的计算单元称为 Actor，换言之，在 Actor 模型中，所有的计算都是在 Actor 中执行的。在面向对象编程里面，一切都是对象；在 Actor 模型里，一切都是 Actor，并且 Actor 之间是完全隔离的，不会共享任何变量.这个有点类似于redis了，一个actor它的内部操作都是单线程，不用担心并发问题。
&emsp;&emsp;目前能完备地支持 Actor 模型而且比较成熟的类库就是Akka了

### 2.1. 简单上手  
```
// 该 Actor 当收到消息 message 后，
// 会打印 Hello message
static class HelloActor extends UntypedActor {

  @Override
  public void onReceive(Object message) {
    System.out.println("Hello " + message);
  }

}
 
public static void main(String[] args) {

  // 创建 Actor 系统
  ActorSystem system = ActorSystem.create("HelloSystem");

  // 创建 HelloActor
  ActorRef helloActor = system.actorOf(Props.create(HelloActor.class));

  // 发送消息给 HelloActor
  helloActor.tell("Actor", ActorRef.noSender());
}
```

* 创建了一个 ActorSystem（Actor 不能脱离 ActorSystem 存在）
* 之后创建了一个 HelloActor，Akka 中创建 Actor 并不是 new 一个对象出来，而是通过调用 system.actorOf() 方法创建的
* 该方法返回的是 ActorRef，而不是 HelloActor；
* 最后通过调用 ActorRef 的 tell() 方法给 HelloActor 发送了一条消息 “Actor”

### 2.2. 特点
* Actor单元可以处理接收到的消息
* Actor 可以存储自己的内部状态，并且内部状态在不同 Actor 之间是绝对隔离的
* Actor 可以和其他 Actor 之间通信
* 创建更多的 Actor  

### 2.3. 实践  
用 Actor 实现累加器  

```
// 累加器
static class CounterActor extends UntypedActor {
  private int counter = 0;
  @Override
  public void onReceive(Object message){
    // 如果接收到的消息是数字类型，执行累加操作，
    // 否则打印 counter 的值
    if (message instanceof Number) {
      counter += ((Number) message).intValue();
    } else {
      System.out.println(counter);
    }
  }
}
public static void main(String[] args) throws InterruptedException {
  // 创建 Actor 系统
  ActorSystem system = ActorSystem.create("HelloSystem");

  //4 个线程生产消息
  ExecutorService es = Executors.newFixedThreadPool(4);

  // 创建 CounterActor 
  ActorRef counterActor = system.actorOf(Props.create(CounterActor.class));

  // 生产 4*100000 个消息 
  for (int i=0; i<4; i++) {
    es.execute(()->{
      for (int j=0; j<100000; j++) {
        counterActor.tell(1, ActorRef.noSender());
      }
    });
  }

  // 关闭线程池
  es.shutdown();

  // 等待 CounterActor 处理完所有消息
  Thread.sleep(1000);

  // 打印结果
  counterActor.tell("", ActorRef.noSender());

  // 关闭 Actor 系统
  system.shutdown();
}
```
* CounterActor 内部持有累计值 counter
* 当 CounterActor 接收到一个数值型的消息 message 时，就将累计值 counter += message；
* 但如果是其他类型的消息，则打印当前累计值 counter。
* 在 main() 方法中，我们启动了 4 个线程来执行累加操作。
* 整个程序没有锁，也没有 CAS，但是程序是线程安全的


### 2.4. 小结  
Actor 可以创建新的 Actor，这些 Actor 最终会呈现出一个树状结构。Actor 模型和现实世界一样都是异步模型，理论上不保证消息百分百送达，也不保证消息送达的顺序和发送的顺序是一致的，甚至无法保证消息会被百分百处理 

## 3. Stm模型
&emsp;&emsp;我们在学习事务的时候，知道事务总结出了4个特性，原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）。  
&emsp;&emsp;通俗来讲，如果数据库中的SQL 都正常执行，则通过 commit() 方法提交事务；如果 SQL 在执行过程中有异常，则通过 rollback() 方法回滚事务。数据库保证在并发情况下不会有死锁，而且还能保证前面我们说的原子性、一致性、隔离性和持久性，也就是 ACID。
### 3.1. java中的STM  
Java 语言并不支持 STM，不过可以借助第三方的类库来支持，如Multiverse  
```
class Account{
  // 余额
  private TxnLong balance;
  // 构造函数
  public Account(long balance){
    this.balance = StmUtils.newTxnLong(balance);
  }
  // 转账
  public void transfer(Account to, int amt){
    // 原子化操作
    atomic(()->{
      if (this.balance.get() > amt) {
        this.balance.decrement(amt);
        to.balance.increment(amt);
      }
    });
  }
}
```
### 3.2. Multiverse的STM基本原理    
其实和数据库的实现类似，即MVCC（多版本并发控制）:  
* 数据库事务在开启的时候，会给数据库打一个快照,以后所有的读写都是基于这个快照的
* 当提交事务的时候，如果所有读写过的数据在该事务执行期间没有发生过变化，那么就可以提交
* 如果发生了变化，说明该事务和有其他事务读写的数据冲突了，这个时候是不可以提交的

### 3.3. 自己实现java的STM 
```
  // 带版本号的对象引用
  public final class VersionedRef<T> {
    final T value;
    final long version;
    // 构造方法
    public VersionedRef(T value, long version) {
      this.value = value;
      this.version = version;
    }
  }
  // 支持事务的引用
  public class TxnRef<T> {
    // 当前数据，带版本号
    volatile VersionedRef curRef;
    // 构造方法
    public TxnRef(T value) {
      this.curRef = new VersionedRef(value, 0L);
    }
    // 获取当前事务中的数据
    public T getValue(Txn txn) {
      return txn.get(this);
    }
    // 在当前事务中设置数据
    public void setValue(T value, Txn txn) {
      txn.set(this, value);
    }
  }
```  
* VersionedRef 这个类的作用就是将对象 value 包装成带版本号的对象
* 数据的每一次修改都对应着一个唯一的版本号，所以不存在仅仅改变 value 或者 version 的情况，所以这里使用不变性模式
* 对数据的读写操作，一定是在一个事务里面，TxnRef 这个类负责完成事务内的读写操作。
* 读写操作委托给了接口 Txn，Txn 代表的是读写操作所在的当前事务， 内部持有的 curRef 代表的是系统中的最新值

```
// 事务接口
public interface Txn {
  <T> T get(TxnRef<T> ref);
  <T> void set(TxnRef<T> ref, T value);
}
//STM 事务实现类
public final class STMTxn implements Txn {
  // 事务 ID 生成器
  private static AtomicLong txnSeq = new AtomicLong(0);
  
  // 当前事务所有的相关数据
  private Map<TxnRef, VersionedRef> inTxnMap = new HashMap<>();
  // 当前事务所有需要修改的数据
  private Map<TxnRef, Object> writeMap = new HashMap<>();
  // 当前事务 ID
  private long txnId;
  // 构造函数，自动生成当前事务 ID
  STMTxn() {
    txnId = txnSeq.incrementAndGet();
  }
 
  // 获取当前事务中的数据
  @Override
  public <T> T get(TxnRef<T> ref) {
    // 将需要读取的数据，加入 inTxnMap
    if (!inTxnMap.containsKey(ref)) {
      inTxnMap.put(ref, ref.curRef);
    }
    return (T) inTxnMap.get(ref).value;
  }
  // 在当前事务中修改数据
  @Override
  public <T> void set(TxnRef<T> ref, T value) {
    // 将需要修改的数据，加入 inTxnMap
    if (!inTxnMap.containsKey(ref)) {
      inTxnMap.put(ref, ref.curRef);
    }
    writeMap.put(ref, value);
  }
  // 提交事务
  boolean commit() {
    synchronized (STM.commitLock) {
    // 是否校验通过
    boolean isValid = true;
    // 校验所有读过的数据是否发生过变化
    for(Map.Entry<TxnRef, VersionedRef> entry : inTxnMap.entrySet()){
      VersionedRef curRef = entry.getKey().curRef;
      VersionedRef readRef = entry.getValue();
      // 通过版本号来验证数据是否发生过变化
      if (curRef.version != readRef.version) {
        isValid = false;
        break;
      }
    }
    // 如果校验通过，则所有更改生效
    if (isValid) {
      writeMap.forEach((k, v) -> {
        k.curRef = new VersionedRef(v, txnId);
      });
    }
    return isValid;
  }
}
```
* STMTxn 是 Txn 最关键的一个实现类
* STMTxn 内部有两个 Map：inTxnMap、writeMap，用于保存当前事务中所有读写的数据的快照
* get() 方法将要读取数据作为快照放入 inTxnMap,每次读取的数据都是一个版本
* set() 方法会将要写入的数据放入 writeMap，但如果写入的数据没被读取过，也会将其放入 inTxnMap。
*  commit() 方法，我们为了简化实现，使用了互斥锁，所以事务的提交是串行的。commit() 方法的实现很简单，首先检查 inTxnMap 中的数据是否发生过变化，如果没有发生变化，那么就将 writeMap 中的数据写入（这里的写入其实就是 TxnRef 内部持有的 curRef）；如果发生过变化，那么就不能将 writeMap 中的数据写入了

```
@FunctionalInterface
public interface TxnRunnable {
  void run(Txn txn);
}
//STM
public final class STM {
  // 私有化构造方法
  private STM() {
  // 提交数据需要用到的全局锁  
  static final Object commitLock = new Object();
  // 原子化提交方法
  public static void atomic(TxnRunnable action) {
    boolean committed = false;
    // 如果没有提交成功，则一直重试
    while (!committed) {
      // 创建新的事务
      STMTxn txn = new STMTxn();
      // 执行业务逻辑
      action.run(txn);
      // 提交事务
      committed = txn.commit();
    }
  }
}
```
* Multiverse 中的原子化操作 atomic()。
* atomic() 方法中使用了类似于 CAS 的操作
* 如果事务提交失败，那么就重新创建一个新的事务，重新执行

```
class Account {
  // 余额
  private TxnRef<Integer> balance;
  // 构造方法
  public Account(int balance) {
    this.balance = new TxnRef<Integer>(balance);
  }
  // 转账操作
  public void transfer(Account target, int amt){
    STM.atomic((txn)->{
      Integer from = balance.getValue(txn);
      balance.setValue(from-amt, txn);
      Integer to = target.balance.getValue(txn);
      target.balance.setValue(to+amt, txn);
    });
  }
}
```

## 4. 协程模型
线程是个重量级的对象，不能频繁创建、销毁，而且线程切换的成本也很高。为了解决这个问题。出现了池化技术线程池，其实还有一种替代方案，协程技术，Java 语言里目前还没有，协程简单地理解为一种轻量级的线程。线程是在内核态中调度的，而协程是在用户态调度的，所以相对于线程来说，协程切换的成本更低。  
目前支持协程的语言有很多：Golang、Python、Lua、Kotlin。  

### 4.1. golang的协程示例  
go hello("World") 这一行代码就可以创建一个协程。写法非常简单：  
```
import (
	"fmt"
	"time"
)

func hello(msg string) {
	fmt.Println("Hello " + msg)
}

func main() {
  
  // 在新的协程中执行 hello 方法
	go hello("World") 
    
    fmt.Println("Run in main")
    
    // 等待 100 毫秒让协程执行结束
	 time.Sleep(100 * time.Millisecond)

}
```  
为每个成功建立连接的 socket 分配一个协程，相比 Java 线程池的实现方案，Golang 中协程的方案更简单  
```
import (
	"log"
	"net"
)
 
func main() {
    // 监听本地 9090 端口
	socket, err := net.Listen("tcp", "127.0.0.1:9090")
	if err != nil {
		log.Panicln(err)
	}
	defer socket.Close()
	for {
        // 处理连接请求  
		conn, err := socket.Accept()
		if err != nil {
			log.Panicln(err)
		}
        // 处理已经成功建立连接的请求
		go handleRequest(conn)
	}
}
// 处理已经成功建立连接的请求
func handleRequest(conn net.Conn) {
	defer conn.Close()
	for {
		buf := make([]byte, 1024)
        // 读取请求数据
		size, err := conn.Read(buf)
		if err != nil {
			return
		}
        // 回写相应数据  
		conn.Write(buf[:size])
	}
}
```

### 4.2. golang协程实现同步   
 &emsp;&emsp;Java 里使用多线程并发地处理 I/O，基本上用的都是异步非阻塞模型，通常会注册一个回调函数。因为同步意味着等待，这是一种严重的浪费。对于协程来说，等待的成本就没有那么高了，所以基于协程实现同步非阻塞是一个可行的方案。  
 &emsp;&emsp;OpenResty 里实现的 cosocket 就是一种同步非阻塞方案

**注意：阻塞本质上是cpu是否把线程挂起**
```
-- 创建 socket
local sock = ngx.socket.tcp()
-- 设置 socket 超时时间
sock:settimeouts(connect_timeout, send_timeout, read_timeout)
-- 连接到目标地址
local ok, err = sock:connect(host, port)
if not ok then
-  -- 省略异常处理
end
-- 发送请求
local bytes, err = sock:send(request_data)
if not bytes then
  -- 省略异常处理
end
-- 读取响应
local line, err = sock:receive()
if err then
  -- 省略异常处理
end
-- 关闭 socket
sock:close()   
-- 处理读取到的数据 line
handle(line)
```  

* 建立连接、发送请求、读取响应所有的操作都是同步的
* cosocket 本身是非阻塞的，所以这些操作虽然是同步的，但是并不会阻塞

### 4.3. 协程的问题  
&emsp;&emsp;和goto语句类似，代码的书写顺序和执行顺序不一致，协程的使用同样也会存在这个问题。  
&emsp;&emsp;计算机科学家艾兹格·迪科斯彻（Edsger Dijkstra），同时他还提出了**结构化程序设计**。在结构化程序设计中，可以使用三种基本控制结构来代替 goto，这三种基本的控制结构就是今天我们广泛使用的顺序结构、选择结构和循环结构.这三种结构可以组合，组合起来一定是线性的。整体来看，代码的书写顺序和执行顺序也是一致的。Golang 中的 go 语句快速创建协破坏了这种结构。Java 语言的多线程其实也算。  
&emsp;&emsp;开启一个新的线程时，程序会并行地出现两个分支，主线程一个分支，子线程一个分支，这两个分支很多情况下都是天各一方、永不相见。而结构化的程序，可以有分支，但是最终一定要汇聚，不能有多个出口，因为只有这样它们组合起来才是线性的

### 4.4. 小结  


## 5. csp模型
Golang 支持协程，协程可以类比 Java 中的线程，解决并发问题的难点就在于线程（协程）之间的协作。线程中如何通讯呢？一般有两种：  
* 共享内存
* 消息传递（Message-Passing）的方式通信，本质上是要避免共享。   
Golang 比较推荐的方案是后者，它是基于CSP 模型实现的。
### 5.1. csp模型介绍
Golang 实现的 CSP 模型和 Actor 模型看上去非常相似，不以共享内存方式通信，以通信方式共享内存，协程之间通信推荐的是使用 channel，channel 你可以形象地理解为现实世界里的管道
```
import (
	"fmt"
	"time"
)
 
func main() {
    // 变量声明
	var result, i uint64
    // 单个协程执行累加操作
	start := time.Now()

	for i = 1; i <= 10000000000; i++ {
		result += i
	}

	// 统计计算耗时
	elapsed := time.Since(start)

	fmt.Printf(" 执行消耗的时间为:", elapsed)

	fmt.Println(", result:", result)

 
    // 4 个协程共同执行累加操作
	start = time.Now()
	ch1 := calc(1, 2500000000)
	ch2 := calc(2500000001, 5000000000)
	ch3 := calc(5000000001, 7500000000)
	ch4 := calc(7500000001, 10000000000)
    // 汇总 4 个协程的累加结果
	result = <-ch1 + <-ch2 + <-ch3 + <-ch4
	// 统计计算耗时
	elapsed = time.Since(start)
	fmt.Printf(" 执行消耗的时间为:", elapsed)
	fmt.Println(", result:", result)
}
// 在协程中异步执行累加操作，累加结果通过 channel 传递
func calc(from uint64, to uint64) <-chan uint64 {
    // channel 用于协程间的通信
	ch := make(chan uint64)
    // 在协程中执行累加操作
	go func() {
		result := from
		for i := from + 1; i <= to; i++ {
			result += i
		}
        // 将结果写入 channel
		ch <- result
	}()
    // 返回结果是用于通信的 channel
	return ch
}
```  


* calc() 方法的返回值是一个只能接收数据的 channel ch
* 子协程会把计算结果发送到这个 ch 中，而主协程也会将这个计算结果通过 ch 读取出来  

### 5.2. csp与生产者和消费者模型  
channel 可以类比为生产者 - 消费者模式中的阻塞队列，Golang 中 channel 的容量可以是 0，容量为 0 的 channel 在 Golang 中被称为无缓冲的 channel，容量大于 0 的则被称为有缓冲的 channel。无缓冲的 channel 类似于 Java 中提供的 SynchronousQueue，主要用途是在两个协程之间做数据交换。比如上面累加器的示例代码中，calc() 方法内部创建的 channel 就是无缓冲的 channel。而创建一个有缓冲的 channel 也很简单，在下面的示例代码中，我们创建了一个容量为 4 的 channel，同时创建了 4 个协程作为生产者、4 个协程作为消费者  
```
// 创建一个容量为 4 的 channel 
ch := make(chan int, 4)
// 创建 4 个协程，作为生产者
for i := 0; i < 4; i++ {
	go func() {
		ch <- 7
	}()
}
// 创建 4 个协程，作为消费者
for i := 0; i < 4; i++ {
    go func() {
    	o := <-ch
    	fmt.Println("received:", o)
    }()
}
```

### 5.3. CSP 模型与 Actor 模型的区别  
* Actor 模型中没有 channel。  
  虽然 Actor 模型中的 mailbox 和 channel 非常像，看上去都像个 FIFO 队列，但是区别还是很大的。Actor 模型中的 mailbox 对于程序员来说是“透明”的，mailbox 明确归属于一个特定的 Actor，是 Actor 模型中的内部机制；而且 Actor 之间是可以直接通信的，不需要通信中介。但 CSP 模型中的 channel 就不一样了，它对于程序员来说是“可见”的，是通信的中介，传递的消息都是直接发送到 channel 中的。

* Actor 模型中发送消息是非阻塞的，而 CSP 模型中是阻塞的。  
  Golang 实现的 CSP 模型，channel 是一个阻塞队列，当阻塞队列已满的时候，向 channel 中发送数据，会导致发送消息的协程阻塞。

* 我们介绍过 Actor 模型理论上不保证消息百分百送达，而在 Golang 实现的CSP 模型中，是能保证消息百分百送达的。不过这种百分百送达也是有代价的，那就是有可能会导致死锁。  

```
func main() {
    // 创建一个无缓冲的 channel  
    ch := make(chan int)
    // 主协程会阻塞在此处，发生死锁
    <- ch 
}
```  

### 5.4. 小结  
&emsp;&emsp;CSP 模型是托尼·霍尔（Tony Hoare）在 1978 年提出的，不过这个模型这些年一直都在发展，其理论远比 Golang 的实现复杂得多，如果你感兴趣，可以参考霍尔写的Communicating Sequential Processes这本电子书。另外，霍尔在并发领域还有一项重要成就，那就是提出了霍尔管程模型，这个你应该很熟悉了，Java 领域解决并发问题的理论基础就是它  
&emsp;&emsp;Java 领域可以借助第三方的类库JCSP来支持 CSP 模型，相比 Golang 的实现，JCSP 更接近理论模型，如果你感兴趣