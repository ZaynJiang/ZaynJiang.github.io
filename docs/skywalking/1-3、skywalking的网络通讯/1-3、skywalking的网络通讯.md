## 1. 开头

前面的premain方法，我们已经知道了字节码如何利用插件对字节码进行增强的。在premain方法之中有个

```
ServiceManager.INSTANCE.boot();
```

会将所有的服务启动起来，其中会包括一个叫做GRPCChannelManager的service，它是负责和skywaking的服务端进行网络通讯的。

## 2. GRPCChannelManager

该类的作用就是创建和维护grpc的连接

```
@DefaultImplementor
public class GRPCChannelManager implements BootService, Runnable {
    private static final ILog logger = LogManager.getLogger(GRPCChannelManager.class);

    private volatile GRPCChannel managedChannel = null;
    private volatile ScheduledFuture<?> connectCheckFuture;
    private volatile boolean reconnect = true;
    private Random random = new Random();
    private List<GRPCChannelListener> listeners = Collections.synchronizedList(new LinkedList<GRPCChannelListener>());
    private volatile List<String> grpcServers;
    private volatile int selectedIdx = -1;
    private volatile int reconnectCount = 0;

    @Override
    public void prepare() throws Throwable {

    }

    @Override
    public void boot() throws Throwable {
    
    }
```

它有几个核心的字段，下面一步步进行分析

### 2.1. GRPCChannel

skywaking基于grpc提供的managerchannel进行封装的管理类。

```
public class GRPCChannel {
    /**
     * origin channel
     */
    private final ManagedChannel originChannel;
    private final Channel channelWithDecorators;
    
}
```

* managerchannel

  原生的managerchannel，是基于grpc封装的管理channel

* channel

  原生的channel，不过对于这个它增加了一些装饰器

#### 2.1.1. 构造方法

```
private GRPCChannel(String host, int port, List<ChannelBuilder> channelBuilders,
    List<ChannelDecorator> decorators) throws Exception {
    ManagedChannelBuilder channelBuilder = NettyChannelBuilder.forAddress(host, port);

    for (ChannelBuilder builder : channelBuilders) {
        channelBuilder = builder.build(channelBuilder);
    }

    this.originChannel = channelBuilder.build();

    Channel channel = originChannel;
    for (ChannelDecorator decorator : decorators) {
        channel = decorator.build(channel);
    }

    channelWithDecorators = channel;
}
```

* 实例化grpcchannel的时候，需要传入ip地址和端口

* ChannelBuilder

  NettyChannelBuilder.forAddress(host, port)通过创建出来channelBuilder，然后迭代进行构建丰富它的特性

  ChannelBuilder这个是sky继承于ManagedChannelBuilder的一个泛型接口，我们实现这个接口可以让grpc设置一些传输参数，比如数据量，连接数之类的。因为可能有多个builder，所以通过这么多的builder可以完整的创建一个builder。

  skywaking有两个实现：

  * StandardChannelBuilder

    ```
    @Override public ManagedChannelBuilder build(ManagedChannelBuilder managedChannelBuilder) throws Exception {
    	return managedChannelBuilder.nameResolverFactory(new DnsNameResolverProvider()).maxInboundMessageSize(MAX_INBOUND_MESSAGE_SIZE).usePlaintext(USE_PLAIN_TEXT);
    }
    ```

    这主要设置了下最大的入口消息量和USE_PLAIN_TEXT表示使用明文传输

  * TLSChannelBuilder

    这个就是需要加入证书ssl

    ```
        @Override public NettyChannelBuilder build(
            NettyChannelBuilder managedChannelBuilder) throws AgentPackageNotFoundException, SSLException {
            File caFile = new File(AgentPackagePath.getPath(), CA_FILE_NAME);
            if (caFile.exists() && caFile.isFile()) {
                SslContextBuilder builder = GrpcSslContexts.forClient();
                builder.trustManager(caFile);
                managedChannelBuilder = managedChannelBuilder.negotiationType(NegotiationType.TLS)
                    .sslContext(builder.build());
            }
            return managedChannelBuilder;
        }
    ```

  channelBuilder = builder.build(channelBuilder)每次去创建channelBuilder

* ChannelDecorator

  ```
  for (ChannelDecorator decorator : decorators) {
      channel = decorator.build(channel);
  }
  ```

  先通过this.originChannel = channelBuilder.build();创建出一个原生的channel,在通过迭代装饰器来装饰这个channel

  这个就是channel的装饰器附着了一些功能，在skywaking中有。

  * AuthenticationDecorator

    比如这个可以给给grpc每次请求添加一个token，这个token来自于Config.Agent.AUTHENTICATION文件

    ```
    public Channel build(Channel channel) {
        if (StringUtil.isEmpty(Config.Agent.AUTHENTICATION)) {
            return channel;
        }
    
        return ClientInterceptors.intercept(channel, new ClientInterceptor() {
            @Override
            public <REQ, RESP> ClientCall<REQ, RESP> interceptCall(MethodDescriptor<REQ, RESP> method,
                CallOptions options, Channel channel) {
                return new ForwardingClientCall.SimpleForwardingClientCall<REQ, RESP>(channel.newCall(method, options)) {
                    @Override
                    public void start(Listener<RESP> responseListener, Metadata headers) {
                        headers.put(AUTH_HEAD_HEADER_NAME, Config.Agent.AUTHENTICATION);
    
                        super.start(responseListener, headers);
                    }
                };
            }
        });
    }
    ```

  * AgentIDDecorator

#### 2.1.2. builder

简单的一个构造器模式，来帮助创建Builder、GRPCChannel等原生类

```
public static class Builder {

}
```

### 2.2. 重要属性

* ScheduledFuture

  定时检测grpcchannel是否存活

* reconnect

  要不要去重试

* List<GRPCChannelListener>

  监听器，当channel的状态发生变化时，就会通知到所有注册的监听器

  即触发statusChanged的方法

  ```
  public interface GRPCChannelListener {
      void statusChanged(GRPCChannelStatus status);
  }
  ```

* grpcServers、selectedIdx、reconnectCount

  用来确定来连接哪一个oap实例

  * grpcServers

    可以配置多个oap地址

  * selectedIdx

    上次选中的oap下标是多少

    

### 2.3. boot启动

```
public void boot() throws Throwable {
    if (Config.Collector.BACKEND_SERVICE.trim().length() == 0) {
        logger.error("Collector server addresses are not set.");
        logger.error("Agent will not uplink any data.");
        return;
    }
    grpcServers = Arrays.asList(Config.Collector.BACKEND_SERVICE.split(","));
    connectCheckFuture = Executors
        .newSingleThreadScheduledExecutor(new DefaultNamedThreadFactory("GRPCChannelManager"))
        .scheduleAtFixedRate(new RunnableWithExceptionProtection(this, new RunnableWithExceptionProtection.CallbackWhenException() {
            @Override
            public void handle(Throwable t) {
                logger.error("unexpected exception.", t);
            }
        }), 0, Config.Collector.GRPC_CHANNEL_CHECK_INTERVAL, TimeUnit.SECONDS);
}
```

* 拿到配置的server列表

* 开启GRPCChannelManager定时任务来检测grpcchannel是否存活

  * DefaultNamedThreadFactory

    自定义的线程工程，可以命名序列号加1

  * RunnableWithExceptionProtection

    自定义runable，可以增加一些异常处理

### 2.4. run方法

```
@Override
public void run() {
   
    if (reconnect) {
        if (grpcServers.size() > 0) {
            String server = "";
            try {
                int index = Math.abs(random.nextInt()) % grpcServers.size();
                if (index != selectedIdx) {
                    selectedIdx = index;

                    server = grpcServers.get(index);
                    String[] ipAndPort = server.split(":");

                    if (managedChannel != null) {
                        managedChannel.shutdownNow();
                    }

                    managedChannel = GRPCChannel.newBuilder(ipAndPort[0], Integer.parseInt(ipAndPort[1]))
                        .addManagedChannelBuilder(new StandardChannelBuilder())
                        .addManagedChannelBuilder(new TLSChannelBuilder())
                        .addChannelDecorator(new AgentIDDecorator())
                        .addChannelDecorator(new AuthenticationDecorator())
                        .build();
                    notify(GRPCChannelStatus.CONNECTED);
                    reconnectCount = 0;
                    reconnect = false;
                } else if (managedChannel.isConnected(++reconnectCount > Config.Agent.FORCE_RECONNECTION_PERIOD)) {
                    // Reconnect to the same server is automatically done by GRPC,
                    // therefore we are responsible to check the connectivity and
                    // set the state and notify listeners
                    reconnectCount = 0;
                    notify(GRPCChannelStatus.CONNECTED);
                    reconnect = false;
                }

                return;
            } catch (Throwable t) {
                logger.error(t, "Create channel to {} fail.", server);
            }
        }

    }
}
```

* 判断是否需要重连

* 获取oap地址

* 选取一个地址，不选择上一次的下标即可

* 构建managedChanne

  ```
   managedChannel = GRPCChannel.newBuilder(ipAndPort[0], Integer.parseInt(ipAndPort[1]))
                          .addManagedChannelBuilder(new StandardChannelBuilder())
                          .addManagedChannelBuilder(new TLSChannelBuilder())
                          .addChannelDecorator(new AgentIDDecorator())
                          .addChannelDecorator(new AuthenticationDecorator())
                          .build();
  ```

  使用GRPCChannel的构造方法创建manager

  增加一些StandardChannelBuilder、TLSChannelBuilder，增加一些通讯配置

  增加一些chaneel装饰器,比如在请求加入token

* notify

  通知监听器

### 2.5. shutdown

## 3.  总结

