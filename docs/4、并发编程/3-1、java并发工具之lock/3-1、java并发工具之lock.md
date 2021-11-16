## 1. 管程
&emsp;&emsp;在之前我们已经讨论过并发编程中需要解决重要问题主要有两种：互斥和同步；互斥可以使用各种互斥锁。同步也叫做线程协作，需要通过管程模型来实现，synchronized其实已经实现了管程模型。但是java sdk还提供了其它的管程工具。既然synchronized已经满足了，jdk为什么还提供了其它的工具呢？  
* synchronized，在以前性能很长，不过1.6以后已经优化过来了
* 死锁问题的解决方案， synchronized 申请资源的时候，如果申请不到，线程直接进入阻塞状态了，而线程进入阻塞状态，啥都干不了，也释放不了线程已经占有的资源

  
&emsp;&emsp;jdk重新设计一把互斥锁去解决这个问题？jdk设计下的互斥 锁方案主要有如下特点？  
* 能够响应中断，如果果阻塞状态的线程能够响应中断信号，也就是说当我们给阻塞的线程发送中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。这样就破坏了不可抢占条件了
* 支持超时，如果等待时间超出时间限制返回错误
* 非阻塞获取锁，获取锁的时候，立马直到结果，不会阻塞  
* 支持条件变量

&emsp;&emsp;jdk的Lock接口就是弥补synchronized的问题的方案。  
```
// 支持中断的 API
void lockInterruptibly() 
  throws InterruptedException;
// 支持超时的 API
boolean tryLock(long time, TimeUnit unit) 
  throws InterruptedException;
// 支持非阻塞获取锁的 API
boolean tryLock();

```

## 2. lock的实现机制
## 2.1. lock的可见性保障
Java 里多线程的可见性是通过 Happens-Before 规则保证的，而 synchronized 之所以能够保证可见性，也是因为有一条 synchronized 相关的规则：synchronized 的解锁 Happens-Before 于后续对这个锁的加锁