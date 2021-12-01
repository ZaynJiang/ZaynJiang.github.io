## 1. 开头  
之前讲过，并发需要解决的的核心问题主要有三个：分工、同步、互斥。而解决分工的问题有一系列的设计模式常用的有：
* Thread-Per-Message 模式
* Worker Thread 模式
* 生产者 - 消费者模式


## 2. Thread-Per-Message模式  
&emsp;&emsp;通常我们处理一个事情，我们自己搞不定，需要委托别人办理，比如写一个server，主线程通常会创建一个子线程，委托子线程去处理请求，这应该能够大大的提升性能。这种委托他人办理的方式，在并发编程领域被总结为一种设计模式，叫做Thread-Per-Message 模式。

### 2.1. Thread实现
我们可以为每个请求都创建一个java线程。
```
final ServerSocketChannel ssc = ServerSocketChannel.open().bind(new InetSocketAddress(8080));
// 处理请求    
try {
  while (true) {
    // 接收请求
    SocketChannel sc = ssc.accept();
    // 每个请求都创建一个线程
    new Thread(()->{
      try {
        // 读 Socket
        ByteBuffer rb = ByteBuffer.allocateDirect(1024);
        sc.read(rb);
        // 模拟处理请求
        Thread.sleep(2000);
        // 写 Socket
        ByteBuffer wb = (ByteBuffer)rb.flip();
        sc.write(wb);
        // 关闭 Socket
        sc.close();
      }catch(Exception e){
        throw new UncheckedIOException(e);
      }
    }).start();
  }
} finally {
  ssc.close();
}   
```
&emsp;&emsp;如上述代码所示，每来一个请求，都创建了一个 Java 线程。这样其实是非常耗费资源的，因为创建线程是非常重量级的操作。
一般来说，可以使用线程池代替频繁创建线程的请求。其实在其它语言中，比如go语言，创建的不是和操作系统一一对应的线程，而是叫做携程的东西，创建成本很低。这种直接使用 Thread-Per-Message 模式就完全没问题了。  
**注意：这种模式存在线程的频繁创建、销毁以及是否可能导致 OOM**

&emsp;&emsp;java如何来创建轻量级的线程呢？可以采用Fiber技术


### 2.2. Fiber实现  
OpenJDK 有个 Loom 项目，就是要解决 Java 语言的轻量级线程问题，提供了fiber请轻量级线程。  
```
final ServerSocketChannel ssc = ServerSocketChannel.open().bind(new InetSocketAddress(8080));
// 处理请求
try{
  while (true) {
    // 接收请求
    final SocketChannel sc = serverSocketChannel.accept();
    Fiber.schedule(()->{
      try {
        // 读 Socket
        ByteBuffer rb = ByteBuffer.allocateDirect(1024);
        sc.read(rb);
        // 模拟处理请求
        LockSupport.parkNanos(2000*1000000);
        // 写 Socket
        ByteBuffer wb = (ByteBuffer)rb.flip()
        sc.write(wb);
        // 关闭 Socket
        sc.close();
      } catch(Exception e){
        throw new UncheckedIOException(e);
      }
    });
  }//while
}finally{
  ssc.close();
}
```
这种通过Fiber.schedule创建的轻量级线程，经过测试后性能还是非常不错的。


### 2.3. 小结  
&emsp;&emsp;Thread-Per-Message 模式在 Java 领域并不是那么知名，根本原因在于 Java 语言里的线程是一个重量级的对象，但是java正在不断的探索轻量级的线程。  



## 3. work thread模式  
&emsp;&emsp;频繁地创建、销毁线程非常影响性能,要想有效避免线程的频繁创建、销毁以及 OOM 问题可以采用Java 领域使用最多的 Worker Thread 模式  

### 3.1. work thread机制
![](work-thread模式.png)  
其实这种工作模式对应到现实世界里，其实指的就是车间里的工人，我们可以在java使用阻塞队列来实现，其实java的线程池就是线程的实现方案  


### 3.2. work thread案例
```
ExecutorService es = Executors.newFixedThreadPool(500);
final ServerSocketChannel ssc = ServerSocketChannel.open().bind(new InetSocketAddress(8080));
// 处理请求    
try {
  while (true) {
    // 接收请求
    SocketChannel sc = ssc.accept();
    // 将请求处理任务提交给线程池
    es.execute(()->{
      try {
        // 读 Socket
        ByteBuffer rb = ByteBuffer.allocateDirect(1024);
        sc.read(rb);
        // 模拟处理请求
        Thread.sleep(2000);
        // 写 Socket
        ByteBuffer wb = 
          (ByteBuffer)rb.flip();
        sc.write(wb);
        // 关闭 Socket
        sc.close();
      }catch(Exception e){
        throw new UncheckedIOException(e);
      }
    });
  }
} finally {
  ssc.close();
  es.shutdown();
}   
```

### 3.3. work thread注意事项  
* 创建线程池需要注意使用有界队列
  ```
  ExecutorService es = new ThreadPoolExecutor(50, 500, 60L, TimeUnit.SECONDS,
  // 注意要创建有界队列
  new LinkedBlockingQueue<Runnable>(2000),
  // 建议根据业务需求实现 ThreadFactory
  r->{
    return new Thread(r, "echo-"+ r.hashCode());
  },
  // 建议根据业务需求实现 RejectedExecutionHandler
  new ThreadPoolExecutor.CallerRunsPolicy());

  ```

  * 注意线程池的线程死锁  
    如果提交到相同线程池的任务不是相互独立的，而是有依赖关系的，那么就有可能导致线程死锁  

反例1：
  ```
    //L1、L2 阶段共用的线程池
    ExecutorService es = Executors.newFixedThreadPool(2);
    //L1 阶段的闭锁    
    CountDownLatch l1=new CountDownLatch(2);
    for (int i=0; i<2; i++){
        System.out.println("L1");
        // 执行 L1 阶段任务
        es.execute(()->{
            //L2 阶段的闭锁 
            CountDownLatch l2=new CountDownLatch(2);
            // 执行 L2 阶段子任务
            for (int j=0; j<2; j++){
                es.execute(()->{
                    System.out.println("L2");
                    l2.countDown();
                });
            }
            // 等待 L2 阶段任务执行完
            l2.await();
            l1.countDown();
        });
    }
    // 等着 L1 阶段任务执行完
    l1.await();
    System.out.println("end");
  ```  
  线程池中的两个线程全部都阻塞在 l2.await(),因为线程池里的线程都阻塞了，没有空闲的线程执行 L2 阶段的任务了。  


反例2：
  ```
    ExecutorService pool = Executors.newSingleThreadExecutor();
    pool.submit(() -> {
    try {
        String qq=pool.submit(()->"QQ").get();
        System.out.println(qq);
    } catch (Exception e) {
    }
    });
  ```
  newSingleThreadExecutor线程池只有单个线程，先将外部线程提交给线程池，外部线程等待内部线程执行完成，但由于线程池只有单线程，导致内部线程一直没有执行的机会，相当于内部线程需要线程池的资源，外部线程需要内部线程的结果，导致死锁

  **注意：提交到相同线程池中的任务一定是相互独立的，否则就一定要慎重**  


## 4. 生产者-消费者模式  
&emsp;&emsp;Java 线程池本质上就是用生产者 - 消费者模式实现的，除了在线程池中的应用，为了提升性能，很多地方也都用到了生产者 - 消费者模式。比如log4j的appender，eventbus等都用到了，而且有时候不知不觉的编程也用到了。对于分布式的场景就可以使用分布式消息系统了，例如rocketmq,这种消息系统更复杂，支持的功能更加丰富。

### 4.1. 机制
  ![](生产消费模式.png)    
  * 核心是一个任务队列
  * 生产者线程生产任务并将任务添加到任务队列中
  * 消费者线程从任务队列中获取任务并执行


### 4.2. 生产者-消费者模式的功能特点  
#### 4.2.1. 功能解耦
&emsp;&emsp;解耦对于大型系统非常重要，在该模式下，生产者和消费者没有任何依赖关系，它们彼此之间的通信只能通过任务队列。此模式天生支持解耦。

#### 4.2.2. 平衡生产者和消费者差异
&emsp;&emsp;因为这种模式生产时异步的，生产者线程只需要将任务添加到任务队列而无需等待任务被消费者线程执行完。假设生产者的速率很慢，而消费者的速率很高，那么我们可以把生产的线程数适当提高，反之一样。

#### 4.2.3. 批量处理任务
&emsp;&emsp;比如我们有一个监控程序，需要将采样数据存储到数据库，一条一条的insert肯定对于性能不太好，所以我们可以，从队列中取出一批数据，然后批量batchinsert即可。
```
// 任务队列
BlockingQueue<Task> bq=new LinkedBlockingQueue<>(2000);
// 启动 5 个消费者线程
// 执行批量任务  
void start() {
  ExecutorService es=xecutors.newFixedThreadPool(5);
  for (int i=0; i<5; i++) {
    es.execute(()->{
      try {
        while (true) {
          // 获取批量任务
          List<Task> ts=pollTasks();
          // 执行批量任务
          execTasks(ts);
        }
      } catch (Exception e) {
        e.printStackTrace();
      }
    });
  }
}

// 从任务队列中获取批量任务
List<Task> pollTasks() throws InterruptedException{
  List<Task> ts=new LinkedList<>();
  // 阻塞式获取一条任务
  Task t = bq.take();
  while (t != null) {
    ts.add(t);
    // 非阻塞式获取一条任务
    t = bq.poll();
  }
  return ts;
}

// 批量执行任务
execTasks(List<Task> ts) {
  // 省略具体代码无数
}
```

#### 4.2.4. 分阶段处理任务
例如我们写的日志，有些数据需要立即执行，有些数据需要批执行。我们都可以采用这种模式执行
* ERROR 级别的日志需要立即刷盘；
* 数据积累到 500 条需要立即刷盘；
* 存在未刷盘数据，且 5 秒钟内未曾刷盘，需要立即刷盘


```
 // 任务队列  
  final BlockingQueue<LogMsg> bq = new BlockingQueue<>();
  //flush 批量  
  static final int batchSize=500;
  // 只需要一个线程写日志
  ExecutorService es = Executors.newFixedThreadPool(1);
  // 启动写日志线程
  void start(){
    File file=File.createTempFile("foo", ".log");
    final FileWriter writer = new FileWriter(file);
    this.es.execute(()->{
      try {
        // 未刷盘日志数量
        int curIdx = 0;
        long preFT=System.currentTimeMillis();
        while (true) {
          LogMsg log = bq.poll(5, TimeUnit.SECONDS);
          // 写日志
          if (log != null) {
            writer.write(log.toString());
            ++curIdx;
          }
          // 如果不存在未刷盘数据，则无需刷盘
          if (curIdx <= 0) {
            continue;
          }
          // 根据规则刷盘
          if (log!=null && log.level==LEVEL.ERROR
           || curIdx == batchSize || System.currentTimeMillis()-preFT>5000){
            writer.flush();
            curIdx = 0;
            preFT=System.currentTimeMillis();
          }
        }
      }catch(Exception e){
        e.printStackTrace();
      } finally {
        try {
          writer.flush();
          writer.close();
        }catch(IOException e){
          e.printStackTrace();
        }
      }
    });  
  }
  // 写 INFO 级别日志
  void info(String msg) {
      bq.put(new LogMsg(LEVEL.INFO, msg));
  }
  // 写 ERROR 级别日志
  void error(String msg) {
    bq.put(new LogMsg(LEVEL.ERROR, msg));
  }
}
// 日志级别
enum LEVEL {
  INFO, ERROR
}
class LogMsg {
  LEVEL level;
  String msg;
  // 省略构造函数实现
  LogMsg(LEVEL lvl, String msg){}
  // 省略 toString() 实现
  String toString(){}
}
```