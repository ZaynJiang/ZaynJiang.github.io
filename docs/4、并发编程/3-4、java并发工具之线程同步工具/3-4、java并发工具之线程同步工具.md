## 1. 概述
我们再多线程的场景下，通常有这样的使用场景：多线程从数据库中查询数据，然后结果汇总输出。这种情况如何直到所有的线程已经处理完成并进行下一步的操作呢？ 我们可以采用线程的join的方法，如下示例代码： 
```
  // 查询未对账订单
  Thread T1 = new Thread(()->{
    pos = getPOrders();
  });
  T1.start();
  // 查询派送单
  Thread T2 = new Thread(()->{
    dos = getDOrders();
  });
  T2.start();
  // 等待 T1、T2 结束
  T1.join();
  T2.join();
  // 执行对账操作
  diff = check(pos, dos);
  // 差异写入差异库
  save(diff);
``` 
这种非常原始的方法其实并不是最佳的方案，jdk提供了一些列的线程同步相关的工具类供我们使用。

## 1. CountDownLatch
上面的代码可以解决一定的问题，但是实际生产中一般不会自己创建线程，都是使用线程池的，而线程池的线程不会结束，所以join方法永远也等待不到结束的。这样如何解决等待都完成的场景呢？  
其实有很多种方式可以实现我们的目标，我们可以使用全局变量count计数的方式，也可以使用条件变量的方式，但是jdk已经为我们提供了优雅的解决方案-CountDownLatch 。
## 1.1. CountDownLatch示例1
如下的代码示例：  
```
// 创建 2 个线程的线程池
Executor executor =  Executors.newFixedThreadPool(2);
while(存在未对账订单){
  // 计数器初始化为 2
  CountDownLatch latch = 
    new CountDownLatch(2);
  // 查询未对账订单
  executor.execute(()-> {
    pos = getPOrders();
    latch.countDown();
  });
  // 查询派送单
  executor.execute(()-> {
    dos = getDOrders();
    latch.countDown();
  });
  
  // 等待两个查询操作结束
  latch.await();
  
  // 执行对账操作
  diff = check(pos, dos);
  // 差异写入差异库
  save(diff);
}
```
如上代码CountDownLatch  是一个计数器，每个线程结束都为countdown，在latch的await上，当都countdown后，就可以执行后面的save代码了。  

## 1.2. CountDownLatch示例2
![](CountDownLatch示例2.png)
## 2. CyclicBarrier
上面的代码代码是一个循环操作，查询未对账订单、查询派送单完成后，等着check和save，但是如果check和save耗时很长，那么下一轮的两个查询就一直要等待。是不是可以两个查询查询完了就能够通知对账，然后继续查询呢？ 有点类似于生产者和消费者模型了。  
![](查询与对账.png)    
我们同样可以使用利用一个计数器，如计数器初始化为 2，线程 T1 和 T2 生产完一条数据都将计数器减 1，如果计数器大于 0 则线程 T1 或者 T2 等待。如果计数器等于 0，则通知线程 T3，并唤醒等待的线程 T1 或者 T2，与此同时，将计数器重置为 2，这样线程 T1 和线程 T2 生产下一条数据的时候就可以继续使用这个计数器了。但是jdk依然为这种场景提供了优雅的解决方案了。

## 2.1. CyclicBarrier案例1
```
// 订单队列
Vector<P> pos;
// 派送单队列
Vector<D> dos;
// 执行回调的线程池 
Executor executor = 
  Executors.newFixedThreadPool(1);
final CyclicBarrier barrier =
  new CyclicBarrier(2, ()->{
    executor.execute(()->check());
  });
  
void check(){
  P p = pos.remove(0);
  D d = dos.remove(0);
  // 执行对账操作
  diff = check(p, d);
  // 差异写入差异库
  save(diff);
}
  
void checkAll(){
  // 循环查询订单库
  Thread T1 = new Thread(()->{
    while(存在未对账订单){
      // 查询订单库
      pos.add(getPOrders());
      // 等待
      barrier.await();
    }
  });
  T1.start();  
  // 循环查询运单库
  Thread T2 = new Thread(()->{
    while(存在未对账订单){
      // 查询运单库
      dos.add(getDOrders());
      // 等待
      barrier.await();
    }
  });
  T2.start();
}
```

## 2.2. CyclicBarrier案例2
![](cyclicbarrier案例2.png)  
![](cyclicbarrier案例2结果.png)    
可以在任务中设置一个聚合点，当一定数量的任务都执行到这个点后，可以执行一个回调函数，然后再各个线程继续
## 3. 总结
* CountDownLatch 主要用来解决一个线程等待多个线程的场景；
* CyclicBarrier 是一组线程之间互相等待，而且具备自动重置的功能，一旦计数器减到 0 会自动重置到你设置的初始值，CyclicBarrier 还可以设置回调函数，可以说是功能丰富


## 3. CompletableFuture  
这个和拉姆达表达式很紧密，可以组合多种不同的功能。这里只是简单的阐述一下它大知的一些功能。
1）竞争
CompletableFuture.supplyAsync(()->{}）.applyToEither.(CompletableFuture.supplyAsync(()->{}),,(s)->{return s;}).join();
选取一个执行结果返回，竞争获胜的返回。
2）异常，捕获线程的异常
 CompletableFuture.supplyAsync(()->{ throw new RuntimeException("exception test!")}).exceptionally(e->{System.out.println(e.getMessage());return "Hello world!";}).join();
3）组合，将两个线程的结果组合thenCombine
4）接收，thenAccept
5）变换，thenApplyAsync
