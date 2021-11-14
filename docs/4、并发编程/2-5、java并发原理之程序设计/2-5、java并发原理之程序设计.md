## 1. 开头
&emsp;&emsp;我们已经了解了很多关于并发编程的问题，比如我们探讨了并发问题的本质、java线程状态流转、如何使用锁、如何利用多线程提升性能等等。现实编程中，我们如何来进行并发编程呢？通过无数前人的总结，并发程序的设计主要可以关注如下的三个方面。  

## 2. 对共享变量的进行封装  
这是我们要关注的核心问题，即解决多线程同时访问共享变量的问题。经验告诉我们有如下的原则：

* 将共享变量作为对象属性封装在内部，对所有公共方法制定并发访问策略  
  比如很对线程都需要用一个计数器对象，我们将计数的相关操作都封装到counter内部了  
    ```
    public class Counter {
    private long value;
    synchronized long get(){
        return value;
    }
    synchronized long addOne(){
        return ++value;
    }
    }
    ```
* 对于这些不会发生变化的共享变量，建议 final 关键字来修饰


## 3. 注意共享变量间的约束条件
当我们的业务的共享变量存在一种约束条件的话，那么就非常可能存在并发问题，需要特别关注，特别是if关键字的语句。例如这里有个库存的共享变量，需要保证库存的数量有上下限
```
public class SafeWM {
  // 库存上限
  private final AtomicLong upper =
        new AtomicLong(0);
  // 库存下限
  private final AtomicLong lower =
        new AtomicLong(0);
  // 设置库存上限
  void setUpper(long v){
    // 检查参数合法性
    if (v < lower.get()) {
      throw new IllegalArgumentException();
    }
    upper.set(v);
  }
  // 设置库存下限
  void setLower(long v){
    // 检查参数合法性
    if (v > upper.get()) {
      throw new IllegalArgumentException();
    }
    lower.set(v);
  }
  // 省略其他业务代码
}
```
这串代码就存在if中的竞态条件，会有并发安全问题。

**PS：共享变量之间的约束条件，基本上都是if语句，这一点我们需要特别注意**  


## 4. 制定并发访问策略
这是一件很综合的事情，总的来说主要有三大原则：
* 优先使用Java SDK 成熟的工具类
* 迫不得已时才使用低级的同步原语synchronized、Lock、Semaphore
* 避免过早优化，首先保证安全，出现性能瓶颈后再优化  

这些原则在不同的并发场景里如何使用呢，后面会一一探讨。

## 5. 总结
* 我们为什么要多线程？
  硬件的核心矛盾，CPU 与内存、I/O 的速度差异，系统软件，需要充分利用机器资源，则引入了多线程

* 使用多线程有什么问题？
  引入多线程后，引入了可见性、原子性和有序性问题，这三个问题就是很多并发程序的 Bug 之源

* 使用多线程问题解决方案？
  * java使用一些关键字来避免有序性和可见性问题  
  * java提供了互斥锁来解决原子性的问题，锁的资源N:1
  * 管程的通用解决方案

* java的多线程的解决方案有什么注意点？
  * 安全性问题，需要关注数据竞争、静态条件
  * 可能引发活跃性问题，死锁、活锁
  * 性能问题，锁粒度

* java的管程模式有哪些？

* java的状态流转是什么？