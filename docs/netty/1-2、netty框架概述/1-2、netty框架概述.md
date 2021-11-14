## 1. 开头
前面已经介绍了netty所需要的网络基础知识以及一些常见的网络模型，netty对这些网络模型是如何支持的呢？

## 2. netty的reactor模型
Netty对上面的三种reactor模型都是支持,具体的写法如下：  
```
public static void test1() {
    //reactor单线程模型
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup(1);
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(eventLoopGroup);
}
public static void test2() {
    //reactor多线程模型
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(eventLoopGroup);
}
public static void test3(){
    //reactor主从模型
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workGroup = new NioEventLoopGroup();
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(bossGroup, workGroup);
}
```

## 3. Netty运行机制
* 创建workgroup线程组（管理socketchannel的）、创建boss线程组（管理serversocketchannel的）
* Serverbootstrap绑定端口
* 用户请求服务端建立连接
* bossgroup使用线程池acceptor连接，并获取socketchannel注册到workgroup
* workgroup使用handler处理socketchannel  

![](netty的网络模型.png)  


## 4. Netty的优化  

* eventloop阻塞
* 系统参数
* 缓冲区
* 直接内存
* 其它  

![](netty优化.png)  


## 5. 粘包的解决
ByteToMessageDecoder提供一些实现类来解决tcp的粘包问题，FixedLengthFrameDecoder（定长），LineBasedFrameDecoder（行分隔符），DelimiterBasedFrameDecoder（自定义分隔符），LengthFieldBasedFrameDecoder（长度编码，在消息头传输长度），JsonObjectDecoder（json分隔符）    


## 6. 总结