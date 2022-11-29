## 1. 开头
&emsp;&emsp;Java并发编程包下提供了Future接口，netty内部用了很多多线程和异步处理，因此Netty对Future进行了封装形成promise。本篇文章主要分析这两大类在netty中的使用。  


## 2. 原生future
以下是原生的future核心api，Future的内部方法可以实现状态检查、取消执行、获取执行结果等操作
```
    // 尝试取消执行
    boolean cancel(boolean mayInterruptIfRunning);
    // 是否已经被取消执行
    boolean isCancelled();
    // 是否已经执行完毕
    boolean isDone();
    // 阻塞获取执行结果
    V get() throws InterruptedException, ExecutionException;
    // 阻塞获取执行结果或超时后返回
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
```

## 3. netty的扩展原生Future
* 增加了更加丰富的状态判断方法
  ```
    // 判断是否执行成功
    boolean isSuccess();
    // 判断是否可以取消执行
    boolean isCancellable();
  ```
* 支持获取导致I/O操作异常
  ```
  Throwable cause();
  ```
* 增加了监听回调有关方法，支持future完成后执行用户指定的回调方法
  ```
    // 增加回调方法
    Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    // 增加多个回调方法
    Future<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners);
    // 删除回调方法
    Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    // 删除多个回调方法
    Future<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners);
  ```
* 增加了更丰富的阻塞等待结果返回的两类方法：
  * sync方法，阻塞等待结果且如果执行失败后向外抛出导致失败的异常。
  * 另一类为await方法，仅阻塞等待结果返回，不向外抛出异常
```
    // 阻塞等待，且如果失败抛出异常
    Future<V> sync() throws InterruptedException;
    // 同上，区别是不可中断阻塞等待过程
    Future<V> syncUninterruptibly();

    // 阻塞等待
    Future<V> await() throws InterruptedException;
    // 同上，区别是不可中断阻塞等待过程
    Future<V> awaitUninterruptibly();
```

## 4. Promise
Promise接口继续继承了Future，并增加若干个设置状态并回调的方法 。所以它常用于传入I/O业务代码中，用于I/O结束后设置成功（或失败）状态，并回调方法
```
    // 设置成功状态并回调
    Promise<V> setSuccess(V result);
    boolean trySuccess(V result);
    // 设置失败状态并回调
    Promise<V> setFailure(Throwable cause);
    boolean tryFailure(Throwable cause);
    // 设置为不可取消状态
    boolean setUncancellable();
```
## 5. netty使用promise

* 这里是我们创建serversocketchannel，并将serversocketchannel进行绑定的地方
```
  final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        //开始register
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
```
* io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)会创建一个DefaultChannelPromise，构造函数传入了当前的channel以及当前所在的线程this
```
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }
```

* 然后接着将promise传递到Unsafe的register方法中。立即返回了以ChannelFuture的形式返回了该promise。  
  **这里是一个异步回调处理：上层的业务可以拿到返回的ChannelFuture阻塞等待结果或者设置回调方法，而继续往下传的Promise可以用于设置执行状态并且回调设置的方法**
    ```
        public ChannelFuture register(final ChannelPromise promise) {
            ObjectUtil.checkNotNull(promise, "promise");
            promise.channel().unsafe().register(this, promise);
            return promise;
        }
    ```
* promise.channel().unsafe().register底层，底层的I/O操作成功与否都可以通过Promise设置状态，并使得外层的ChannelFuture可以感知得到I/O操作的结果。如下代码所示：
```
// io.netty.channel.AbstractChannel.AbstractUnsafe.java
        @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                // 如果已经注册过，则置为失败
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                // 如果线程类型不兼容，则置为失败
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    // 出现异常情况置promise为失败
                    safeSetFailure(promise, t);
                }
            }
        }

        private void register0(ChannelPromise promise) {
            try {
                // 注册之前，先将promise置为不可取消转态
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                doRegister();
                neverRegistered = false;
                registered = true;

                pipeline.invokeHandlerAddedIfNeeded();
                // promise置为成功
                safeSetSuccess(promise);
                pipeline.fireChannelRegistered();
             
                if (isActive()) {
                    if (firstRegistration) {
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                // 出现异常情况置promise为失败
                safeSetFailure(promise, t);
            }
        }
```
* 通过ChannelFuture获取I/O执行结果
  ```
  // io.netty.bootstrap.AbstractBootstrap.java

    final ChannelFuture initAndRegister() {
        //...
        ChannelFuture regFuture = config().group().register(channel);
        // 如果异常不为null，则意味着底层的I/O已经失败，并且promise设置了失败异常
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
  ```

  * ChannelFuture#isDone()方法可以知道底层的注册是否完成，如果完成，则继续进行bind
    ```
     final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }
        //不能肯定register完成，因为register是丢到nio event loop里面执行去了。
        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
    ```

   * 因为注册是个异步操作，如果此时注册可能还没完成，就会新建了一个新的PendingRegistrationPromise，并为原来的ChannelFuture对象添加了一个回调方法，并在回调中更改PendingRegistrationPromise的状态，而且PendingRegistrationPromise会继续被传递到上层。当底层的Promise状态被设置并且回调，就会进入该回调方法。从而将I/O状态继续向外传递。
    ```
    // Registration future is almost always fulfilled already, but just in case it's not.
    final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
    //等着register完成来通知再执行bind
    regFuture.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            Throwable cause = future.cause();
            if (cause != null) {
                // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                // IllegalStateException once we try to access the EventLoop of the Channel.
                promise.setFailure(cause);
            } else {
                // Registration was successful, so set the correct executor to use.
                // See https://github.com/netty/netty/issues/2586
                promise.registered();

                doBind0(regFuture, channel, localAddress, promise);
            }
        }
    });
    return promise;
    ```

 ## 6. DefaultChannelPromise原理分析
 * DefaultChannelPromise继承于父类DefaultPromise，其父类相关字段如下：  
   ```
    // result字段的原子更新器
    @SuppressWarnings("rawtypes")
    private static final AtomicReferenceFieldUpdater<DefaultPromise, Object> RESULT_UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(DefaultPromise.class, Object.class, "result");
    // 缓存执行结果的字段
    private volatile Object result;
    // promise所在的线程
    private final EventExecutor executor;
    // 一个或者多个回调方法
    private Object listeners;
    // 阻塞线程数量计数器  
    private short waiters;
   ```
 ### 6.1. 状态修改   
 这里设置成功状态为例（setSuccess）


    @Override
    public Promise<V> setSuccess(V result) {
    if (setSuccess0(result)) {
            // 调用回调方法
            notifyListeners();
            return this;
    }
    throw new IllegalStateException("complete already: " + this);
    }
    
    private boolean setSuccess0(V result) {
        return setValue0(result == null ? SUCCESS : result);
    }
    
    private boolean setValue0(Object objResult) {
        // 原子修改result字段为objResult
        if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
            RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
            checkNotifyWaiters();
            return true;
        }
        return false;
    }
    
    private synchronized void checkNotifyWaiters() {
        if (waiters > 0) {
            // 如果有其他线程在等待该promise的结果，则唤醒他们
            notifyAll();
        }
    }
    ```
    分析：promise的状态其实就是原子地修改result字段为传入的执行结果。值得注意的是，result字段带有volatile关键字来确保多线程之间的可见性。另外，设置完毕状态后，会尝试唤醒所有在阻塞等待该promise返回结果的线程  


### 6.2. 阻塞线程以等待执行结果  
其他线程会阻塞等待该promise返回结果，具体实现以sync方法
```
    @Override
    public Promise<V> sync() throws InterruptedException {
        // 阻塞等待
        await();
        // 如果有异常则抛出
        rethrowIfFailed();
        return this;
    }

    @Override
    public Promise<V> await() throws InterruptedException {
        if (isDone()) {
            // 如果已经完成，直接返回
            return this;
        }
        // 可以被中断
        if (Thread.interrupted()) {
            throw new InterruptedException(toString());
        }
        //检查死循环
        checkDeadLock();

        synchronized (this) {
            while (!isDone()) {
                // 递增计数器（用于记录有多少个线程在等待该promise返回结果）
                incWaiters();
                try {
                    // 阻塞等待结果
                    wait();
                } finally {
                    // 递减计数器
                    decWaiters();
                }
            }
        }
        return this;
    }
```
所有调用sync方法的线程，都会被阻塞，直到promise被设置为成功或者失败。这也解释了为何Netty客户端或者服务端启动的时候一般都会调用sync方法，本质上都是阻塞当前线程而异步地等待I/O结果返回

### 6.3. 回调机制  
添加回调方法完成之后，会立即检查promise是否已经完成；如果promise已经完成，则马上调用回调方法
```
 @Override
    public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {
        checkNotNull(listener, "listener");

        synchronized (this) {
            // 添加回调方法
            addListener0(listener);
        }

        if (isDone()) {
            // 如果I/O操作已经结束，直接触发回调
            notifyListeners();
        }

        return this;
    }

    private void addListener0(GenericFutureListener<? extends Future<? super V>> listener) {
        if (listeners == null) {
            // 只有一个回调方法直接赋值
            listeners = listener;
        } else if (listeners instanceof DefaultFutureListeners) {
            // 将回调方法添加到DefaultFutureListeners内部维护的listeners数组中
            ((DefaultFutureListeners) listeners).add(listener);
        } else {
            // 如果有多个回调方法，新建一个DefaultFutureListeners以保存更多的回调方法
            listeners = new DefaultFutureListeners((GenericFutureListener<?>) listeners, listener);
        }
    }
```

### 7. 总结  
Netty的Promise和Future机制是基于Java并发包下的Future开发的。其中Future支持阻塞等待、添加回调方法、判断执行状态等，而Promise主要是支持状态设置相关方法。当底层I/O操作通过Promise改变执行状态，我们可以通过同步等待的Future立即得到结果。  