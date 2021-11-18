## 1. 开头  
&emsp;&emsp;前面我们已经说过，锁的范围越大性能越差，在不同的场景下，需要使用不同粒度的锁从而提升下性能。在一些读多写少的
场景下，我们就可以使用java sdk实现的读写锁ReadWriteLock。

## 2. 读写锁的定义  
* 允许多个线程同时读共享变量；
* 只允许一个线程写共享变量；
* 写的时候，竞争其他线程读；

## 3. 只从内存中读写的缓存  
```
class Cache<K,V> {
  final Map<K, V> m =
    new HashMap<>();
  final ReadWriteLock rwl =
    new ReentrantReadWriteLock();
  // 读锁
  final Lock r = rwl.readLock();
  // 写锁
  final Lock w = rwl.writeLock();
  
  //读缓存
  V get(K key) {
    r.lock();
    try { 
        return m.get(key); 
    } finally { 
        r.unlock(); 
    }
  }
  
  //写缓存
  V put(String key, Data v) {
    w.lock();
    try { 
        return m.put(key, v); 
    } finally { 
        w.unlock(); 
    }
  }
}
```

## 4. 按需加载缓存写法范式  

```
class Cache<K,V> {
  final Map<K, V> m =
    new HashMap<>();
  final ReadWriteLock rwl = 
    new ReentrantReadWriteLock();
  final Lock r = rwl.readLock();
  final Lock w = rwl.writeLock();
 
  V get(K key) {
    V v = null;
    // 读缓存
    r.lock();         ①
    try {
      v = m.get(key); ②
    } finally{
      r.unlock();     ③
    }
    // 缓存中存在，返回
    if(v != null) {   ④
      return v;
    }  
    // 缓存中不存在，查询数据库
    w.lock();         ⑤
    try {
      // 再次验证
      // 其他线程可能已经查询过数据库
      v = m.get(key); ⑥
      if(v == null){  ⑦
        // 查询数据库
        v= 省略代码无数
        m.put(key, v);
      }
    } finally{
      w.unlock();
    }
    return v; 
  }
}
```
**注意：在写入逻辑中需要m.get(key)重新查询一缓存，因为有可能多个线程同时进入到写入这里，这样会导致都要查询数据库，如果数据量特别大的话，会导致缓存穿透，极端会导致数据库崩溃。**  

**PS：一般设计缓存的时候，需要考虑同步机制，可以采用超时机制，也可以采用定时同步和双写机制，需要具体场景具体分析了.**

## 5. 锁的升级和降级  
* 锁的升级不被允许，ReadWriteLock不允许持有读锁时再获取写锁, 如下的代码w.lock会一直阻塞，这时很危险的。
```
// 读缓存
r.lock();         ①
try {
  v = m.get(key); ②
  if (v == null) {
    w.lock();
    try {
      // 再次验证并更新缓存
      // 省略详细代码
    } finally{
      w.unlock();
    }
  }
} finally{
  r.unlock();     ③
}
```

* 锁可以降级,如下代码， 线程持有w.lock()的时候，然后再调用 r.lock()， 相当于降级为读锁了。即本线程在释放写锁之前，获取读锁一定是可以立刻获取到的，不存在其他线程持有读锁或者写锁（读写锁互斥），所以java允许锁降级，此刻别的线程是可以读的，但是降级之前是不能读的。
```
 // 获取读锁
    r.lock();
    if (!cacheValid) {
      // 释放读锁，因为不允许读锁的升级
      r.unlock();
      // 获取写锁
      w.lock();
      try {
        // 再次检查状态  
        if (!cacheValid) {
          data = ...
          cacheValid = true;
        }
        // 释放写锁前，降级为读锁
        // 降级是可以的
        r.lock(); ①
      } finally {
        // 释放写锁
        w.unlock(); 
      }
    }
    // 此处仍然持有读锁
    try {
        use(data);
    } finally {
        r.unlock();
    }
```

PS: 排查程序cpu利用低且有假死现象的步骤：  
* ps -ef | grep java查看pid
* top -p查看java中的线程
* 使用jstack将其堆栈信息保存下来，查看是否是锁升级导致的阻塞问题
* 查看代码是否有读锁里面获取写锁的情况

## 6. 总结
&emsp;&emsp;ReadWriteLock提供了读写锁来帮助开发者控制锁的粒度从而提升程序的性能，其中缓存的实现就是例子，需要注意缓存的写法范式。  
&emsp;&emsp;读写锁可以进行降级，即再释放写锁之前，可以调用读锁的lock降级为读锁。要非常注意不可以升级，如果在读锁中获取写锁，将会出现阻塞。  
可以总结读写锁的特性为：
* 获取写锁的前提是读锁和写锁均未被占用
* 获取读锁的前提是没有其他线程占用写锁
* 申请写锁时不中断其他线程申请读锁
* 公平锁如果过有写申请，能禁止读锁