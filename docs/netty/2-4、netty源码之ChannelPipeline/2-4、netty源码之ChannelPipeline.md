## 1. 开头
关于ChannelPipeline，这里有几点重新声明下：
* 一个 Channel 包含了一个 ChannelPipeline
* ChannelPipeline持有一个ChannelHandlerContext组成的双向链表
* ChannelHandlerContext都持有一个ChannelHandler  
如下是一个整体的图
![](channelpipeline整体图.png)  


## 2. channelPipeline的生命周期
### 2.1. channelPipeline创建入口
它是在AbstractChannel构造方法中创建的，即每个 Channel 都有一个 ChannelPipeline
```
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```

### 2.2. channelPipeline关键属性
* DefaultChannelPipeline持有Channel属性，会设置创建它的Channel对象
* 实例化两个ChannelHandlerContext，一个是 HeadContext 实例 head, 另一个是 TailContext 实例 tail，形成双向链表
* TailContext实现了ChannelInboundHandler
* HeadContext实现了ChannelOutboundHandler, ChannelInboundHandler  

**注意：head 和 tail 并没有包含 ChannelHandler, 这是因为 HeadContext 和 TailContext 继承于 AbstractChannelHandlerContext 的同时也实现了 ChannelHandler 接口了, 因此它们有 Context 和 Handler 的双重属性**
```
public DefaultChannelPipeline(AbstractChannel channel) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    this.channel = channel;

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```   

### 2.3. HeadContext、TailContext的构造器  
* HeadContext构造方法inbound = false, outbound = true，都会调用了父类 AbstractChannelHandlerContext 的构造器
* TailContext 的构造器与 HeadContext 的相反， inbound = true, outbound = false，都会调用了父类 AbstractChannelHandlerContext 的构造器
```
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
}
```


### 2.4. ChannelInitializer的添加    
在初始化 Bootstrap, 我们会添加我们自定义的 ChannelHandler到ChannelPipeline
```
Bootstrap b = new Bootstrap();
b.group(group)
 .channel(NioSocketChannel.class)
 .option(ChannelOption.TCP_NODELAY, true)
 .handler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) throws Exception {
         ChannelPipeline p = ch.pipeline();
         p.addLast(new EchoClientHandler());
     }
 });
```
添加的流程如下：
* 在调用handler时, 传入了ChannelInitializer对象
* ChannelInitializer提供了一个 initChannel 方法供我们初始化 ChannelHandler.
* ChannelInitializer 实现了 ChannelHandler
* 最终是由Bootstrap.init 方法中添加到 ChannelPipeline
  ```
    @Override
    @SuppressWarnings("unchecked")
    void init(Channel channel) throws Exception {
        ChannelPipeline p = channel.pipeline();
        p.addLast(handler());
        ...
    }
  ```  
  ![](channelpipeline添加hander.png)

* Bootstrap.init 中会调用 p.addLast() 方法, 将 ChannelInitializer 插入到链表末端
  ```
    @Override
    public ChannelPipeline addLast(EventExecutorGroup group, final String name, ChannelHandler handler) {
        synchronized (this) {
            checkMultiplicity(handler);
            newCtx = newContext(group, filterName(name, handler), handler);
            addLast0(newCtx);
            //......
        }

        return this;
    }
  ```
* 为了添加一个handler到pipeline中, 必须把此handler包装成ChannelHandlerContext
  ```
    private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
        return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
    }
  ```
* 添加到双向链表之中
  ```
    private void addLast0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext prev = tail.prev;
        newCtx.prev = prev;
        newCtx.next = tail;
        prev.next = newCtx;
        tail.prev = newCtx;
    }
  ```

### 2.4. 自定义handler的添加  
&emsp;&emsp;上一节我们分析了一个 ChannelInitializer 如何插入到 Pipeline 。  
&emsp;&emsp;下面我们来分析如下的内容：  
* ChannelInitializer 在哪里被调用？
* ChannelInitializer 的作用是什么呢？
* ChannelHandler 是如何插入到 Pipeline的呢？  

#### 2.4.1. channel 的注册的回顾

* AbstractBootstrap.initAndRegister中, 通过 group().register(channel), 调用 MultithreadEventLoopGroup.register 方法
* MultithreadEventLoopGroup.register 中, 通过 next() 获取一个可用的 SingleThreadEventLoop, 然后调用它的 register
* SingleThreadEventLoop.register 中, 通过 channel.unsafe().register(this, promise) 来获取 channel 的 unsafe() 底层操作对象, 然后调用它的 register.
* AbstractUnsafe.register 方法中, 调用 register0 方法注册 Channel
* AbstractUnsafe.register0 中, 调用 AbstractNioChannel#doRegister 方法
* AbstractNioChannel.doRegister 方法通过 javaChannel().register(eventLoop().selector, 0, this) 将 Channel 对应的 Java NIO SockerChannel 注册到一个 eventLoop 的 Selector 中, 并且将当前 Channel 作为 attachment。