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

## 4. 协程模型

## 5. csp模型