## 1. 开头

在skywaking还有一个重要的service服务，即ServiceAndEndpoint

### 1.1. 基础概念

* service

  一个应用/服务的集群

* serviceInstance

  一个应用/服务的进程

* EndPoint

  类似一次url请求

## 2. ServiceAndEndpointRegisterClient

### 2.1. 主要属性

* registerBlockingStub

  集成了数据序列化，请求发送，响应序列化等功能的client工具，类似Invoker

  用于做注册

*  ServiceInstancePingBlockingStub

  做心跳

* applicationRegisterFuture

  这个定时任务后面再进行分析

### 2.2. 主要方法

* statusChanged

  因为实现了GRPCChannelListener，这个就是如果channel的状态发生了变化将会执行。

  ```
  public void statusChanged(GRPCChannelStatus status) {
      if (GRPCChannelStatus.CONNECTED.equals(status)) {
          Channel channel = ServiceManager.INSTANCE.findService(GRPCChannelManager.class).getChannel();
          registerBlockingStub = RegisterGrpc.newBlockingStub(channel);
          serviceInstancePingStub = ServiceInstancePingGrpc.newBlockingStub(channel);
      } else {
          registerBlockingStub = null;
          serviceInstancePingStub = null;
      }
      this.status = status;
  }
  ```

  * 如果连接成功 
    * 通过ServiceManager.INSTANCE.findService(GRPCChannelManager.class).getChannel()获取一个channel
    * 接着基于channel创建registerBlockingStub和serviceInstancePingStub
  * 如果连接不成功
    * registerBlockingStub、serviceInstancePingStub置为空

* prepare

  ```
  public void prepare() throws Throwable {
      ServiceManager.INSTANCE.findService(GRPCChannelManager.class).addChannelListener(this);
  
      INSTANCE_UUID = StringUtil.isEmpty(Config.Agent.INSTANCE_UUID) ? UUID.randomUUID().toString()
          .replaceAll("-", "") : Config.Agent.INSTANCE_UUID;
  
      SERVICE_INSTANCE_PROPERTIES = new ArrayList<KeyStringValuePair>();
  
      for (String key : Config.Agent.INSTANCE_PROPERTIES.keySet()) {
          SERVICE_INSTANCE_PROPERTIES.add(KeyStringValuePair.newBuilder()
              .setKey(key).setValue(Config.Agent.INSTANCE_PROPERTIES.get(key)).build());
      }
  }
  ```

  * 给GRPCChannelManager添加自身的监听器

  * INSTANCE_UUID

    给自己当前jvm进程创建一个uuid作为实例uu_id

* boot

  ```
  applicationRegisterFuture = Executors
      .newSingleThreadScheduledExecutor(new DefaultNamedThreadFactory("ServiceAndEndpointRegisterClient"))
      .scheduleAtFixedRate(new RunnableWithExceptionProtection(this, new RunnableWithExceptionProtection.CallbackWhenException() {
          @Override
          public void handle(Throwable t) {
              logger.error("unexpected exception.", t);
          }
      }), 0, Config.Collector.AP
  ```

  创建定时任务，传入线程工厂，异常处理打印日志等

* run方法

  比较重要，单独进行分析

### 2.3. run方法

服务boot后会启动定时任务，然后进行run方法，这个方法的核心逻辑是啥呢

* RemoteDownstreamConfig.Agent.SERVICE_ID == DictionaryUtil.nullValue()

  * 申请一个sericeid

    ```
    ServiceRegisterMapping serviceRegisterMapping = registerBlockingStub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS).doServiceRegister(	        Services.newBuilder().addServices(Service.newBuilder().setServiceName(Config.Agent.SERVICE_NAME)).build());
    ```

    这个就是grpc的一些调用了。采用protbuffer协议发送，可以在Register.proto文件中看到定义了很多方法。

    其中KeyStringValuePair是一个键值对，返回是一个ServiceInstanceRegisterMapping

    总的来说就是使用grpc采用probudder协议，传入servicename,请求，返回一个serviceName对应的id即ServiceRegisterMapping

  * 迭代serviceRegisterMapping

    将返回的sericename对应的id保存到内存之中。

    ```
    if (serviceRegisterMapping != null) {
        for (KeyIntValuePair registered : serviceRegisterMapping.getServicesList()) {
            if (Config.Agent.SERVICE_NAME.equals(registered.getKey())) {
                RemoteDownstreamConfig.Agent.SERVICE_ID = registered.getValue();
                shouldTry = true;
            }
        }
    }
    ```

* RemoteDownstreamConfig.Agent.SERVICE_ID != DictionaryUtil.nullValue()

  表示已经存在这个serviceid了。

  ```
  RemoteDownstreamConfig.Agent.SERVICE_INSTANCE_ID == DictionaryUtil.nullValue()
  ```

  ```
  ServiceInstanceRegisterMapping instanceMapping = registerBlockingStub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)
          .doServiceInstanceRegister(ServiceInstances.newBuilder()
      .addInstances(
          ServiceInstance.newBuilder()
              .setServiceId(RemoteDownstreamConfig.Agent.SERVICE_ID)
              .setInstanceUUID(INSTANCE_UUID)
              .setTime(System.currentTimeMillis())
              .addAllProperties(OSUtil.buildOSInfo())
              .addAllProperties(SERVICE_INSTANCE_PROPERTIES)
      ).build());
  ```

  需要先判断当前的实例id是否已经拿到了，没有拿到则根据上述类似的流程传入开始生成的实例uuid调用grpc去获取

  注意这里还会构建一些参数如OSUtil.buildOSInfo()获取一些当前操作系统的信息传入，如ip等等

* 如果都分配好了，则会走心跳相关的请求了

  ```
  } else {
      final Commands commands = serviceInstancePingStub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)
          .doPing(ServiceInstancePingPkg.newBuilder()
          .setServiceInstanceId(RemoteDownstreamConfig.Agent.SERVICE_INSTANCE_ID)
          .setTime(System.currentTimeMillis())
          .setServiceInstanceUUID(INSTANCE_UUID)
          .build());
  
      NetworkAddressDictionary.INSTANCE.syncRemoteDictionary(registerBlockingStub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS));
      EndpointNameDictionary.INSTANCE.syncRemoteDictionary(registerBlockingStub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS));
      ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
  ```

  * 发送心跳给服务端，告诉还活着
  * NetworkAddressDictionary发送字典信息，这些字典信息是什么呢？下一标题进行分析

### 2.4. 字典信息

我们知道一次请求，距离

* 客户端通过http访问,localhost:8080/api到jvm里面
* 该请求访问redis，通过xxxx:6379/get访问了redis
* 接着通过xxxx:3306访问了MySQL的某张表
* 最后将数据返回给客户端

上面提到的ocalhost:8080/api、xxxx:6379/get、xxxx:3306等信息基本变化不到，每次请求不用都把数据发送到服务，因此形成了字典数据，定时发送即可

skywakling做了一个映射关系，这些数据字典形成了一个映射告诉给服务端，每次业务请求数据发送业务id即可了。这样就节省了数据量

发送映射关系就是上面的NetworkAddressDictionary的方法进行传输

```
public enum NetworkAddressDictionary {
    INSTANCE;
    private Map<String, Integer> serviceDictionary = new ConcurrentHashMap<String, Integer>();
    private Set<String> unRegisterServices = new ConcurrentSet<String>();

    public PossibleFound find(String networkAddress) {
        Integer applicationId = serviceDictionary.get(networkAddress);
        if (applicationId != null) {
            return new Found(applicationId);
        } else {
            if (serviceDictionary.size() + unRegisterServices.size() < SERVICE_CODE_BUFFER_SIZE) {
                unRegisterServices.add(networkAddress);
            }
            return new NotFound();
        }
    }

    public void syncRemoteDictionary(
        RegisterGrpc.RegisterBlockingStub networkAddressRegisterServiceBlockingStub) {
        if (unRegisterServices.size() > 0) {
            NetAddressMapping networkAddressMappings = networkAddressRegisterServiceBlockingStub.doNetworkAddressRegister(
                NetAddresses.newBuilder().addAllAddresses(unRegisterServices).build());
            if (networkAddressMappings.getAddressIdsCount() > 0) {
                for (KeyIntValuePair keyWithIntegerValue : networkAddressMappings.getAddressIdsList()) {
                    unRegisterServices.remove(keyWithIntegerValue.getKey());
                    serviceDictionary.put(keyWithIntegerValue.getKey(), keyWithIntegerValue.getValue());
                }
            }
        }
    }

    public void clear() {
        this.serviceDictionary.clear();
    }
}
```

* NetworkAddressDictionary

  * unRegisterServices

    oap列表中还没有注册信息的地址  

  * syncRemoteDictionary

    这方法会拿到这些地址进行注册

  * serviceDictionary保存已注册的字典 

* EndpointNameDictionary

  比如有个/createorder请求

  这个也会建立一个类似数据字典的映射关系，采用EndpointNameDictionary来进行传输的

  ```
  public void syncRemoteDictionary(
      RegisterGrpc.RegisterBlockingStub serviceNameDiscoveryServiceBlockingStub) {
      if (unRegisterEndpoints.size() > 0) {
          Endpoints.Builder builder = Endpoints.newBuilder();
          for (OperationNameKey operationNameKey : unRegisterEndpoints) {
              Endpoint endpoint = Endpoint.newBuilder()
                  .setServiceId(operationNameKey.getServiceId())
                  .setEndpointName(operationNameKey.getEndpointName())
                  .setFrom(DetectPoint.server)
                  .build();
              builder.addEndpoints(endpoint);
          }
          EndpointMapping serviceNameMappingCollection = serviceNameDiscoveryServiceBlockingStub.doEndpointRegister(builder.build());
          if (serviceNameMappingCollection.getElementsCount() > 0) {
              for (EndpointMappingElement element : serviceNameMappingCollection.getElementsList()) {
                  OperationNameKey key = new OperationNameKey(
                      element.getServiceId(),
                      element.getEndpointName());
                  unRegisterEndpoints.remove(key);
                  endpointDictionary.put(key, element.getEndpointId());
              }
          }
      }
  }
  ```

  * OperationNameKey

    封装了serviceid和endpointname

  * unRegisterEndpoints

    没有注册的endpoints

  * find方法

    当没有找到的时候，则需要去注册