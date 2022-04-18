## 1. 开头

每一个节点相关的jvm信息使用采用jvmservice服务进行传输的。

## 2. jvmservice

### 2.1. 核心属性

* JVMMetric

  和jvm相关的信息放在这个对象之中，比如gc，cpu等信息，这个采用了一个LinkedBlockingQueue来存放

* collectMetricFuture

  收集jvm指标的定时任务

* sendMetricFuture

  发送jvm指标的定时任务

* sender

  发送jvm数据的工具

### 2.2. prepare

```
public void prepare() throws Throwable {
    queue = new LinkedBlockingQueue<JVMMetric>(Config.Jvm.BUFFER_SIZE);
    sender = new Sender();
    ServiceManager.INSTANCE.findService(GRPCChannelManager.class).addChannelListener(sender);
}
```

* 创建queue
* 创建sender
* 添加sender作为监听器给grpcchannnelmanager

### 2.3. boot

```
@Override
public void boot() throws Throwable {
    collectMetricFuture = Executors
        .newSingleThreadScheduledExecutor(new DefaultNamedThreadFactory("JVMService-produce"))
        .scheduleAtFixedRate(new RunnableWithExceptionProtection(this, new RunnableWithExceptionProtection.CallbackWhenException() {
            @Override public void handle(Throwable t) {
                logger.error("JVMService produces metrics failure.", t);
            }
        }), 0, 1, TimeUnit.SECONDS);
    sendMetricFuture = Executors
        .newSingleThreadScheduledExecutor(new DefaultNamedThreadFactory("JVMService-consume"))
        .scheduleAtFixedRate(new RunnableWithExceptionProtection(sender, new RunnableWithExceptionProtection.CallbackWhenException() {
            @Override public void handle(Throwable t) {
                logger.error("JVMService consumes and upload failure.", t);
            }
        }
        ), 0, 1, TimeUnit.SECONDS);
}
```

和之前的类似，定一个定时任务并执行

* collectMetricFuture的定时任务传入的是this
* sendMetricFuture的定时任务传入的是sender

### 2.4. run

service的collectMetricFuture定时任务执行的是什么呢，在run方法之中

```
@Override
public void run() {
    if (RemoteDownstreamConfig.Agent.SERVICE_ID != DictionaryUtil.nullValue()
        && RemoteDownstreamConfig.Agent.SERVICE_INSTANCE_ID != DictionaryUtil.nullValue()
    ) {
        long currentTimeMillis = System.currentTimeMillis();
        try {
            JVMMetric.Builder jvmBuilder = JVMMetric.newBuilder();
            jvmBuilder.setTime(currentTimeMillis);
            jvmBuilder.setCpu(CPUProvider.INSTANCE.getCpuMetric());
            jvmBuilder.addAllMemory(MemoryProvider.INSTANCE.getMemoryMetricList());
            jvmBuilder.addAllMemoryPool(MemoryPoolProvider.INSTANCE.getMemoryPoolMetricsList());
            jvmBuilder.addAllGc(GCProvider.INSTANCE.getGCList());

            JVMMetric jvmMetric = jvmBuilder.build();
            if (!queue.offer(jvmMetric)) {
                queue.poll();
                queue.offer(jvmMetric);
            }
        } catch (Exception e) {
            logger.error(e, "Collect JVM info fail.");
        }
    }
}
```

* 判断id有没有
* 构建JVMMetric.Builder jvmBuilder = JVMMetric.newBuilder();，进行收集数据
* 放到队列之中

这些不同的指标数据都是通过CPUProvider、MemoryProvider、GCProvider等provider中过来的

这些provicder是什么呢？

CPUProvider如下：

```
public enum CPUProvider {
    INSTANCE;
    private CPUMetricsAccessor cpuMetricsAccessor;

    CPUProvider() {
        int processorNum = ProcessorUtil.getNumberOfProcessors();
        try {
            this.cpuMetricsAccessor =
                (CPUMetricsAccessor)CPUProvider.class.getClassLoader().loadClass("org.apache.skywalking.apm.agent.core.jvm.cpu.SunCpuAccessor")
                    .getConstructor(int.class).newInstance(processorNum);
        } catch (Exception e) {
            this.cpuMetricsAccessor = new NoSupportedCPUAccessor(processorNum);
            ILog logger = LogManager.getLogger(CPUProvider.class);
            logger.error(e, "Only support accessing CPU metrics in SUN JVM platform.");
        }
    }

    public CPU getCpuMetric() {
        return cpuMetricsAccessor.getCPUMetrics();
    }
}
```

* 其实这个是一个枚举

* 构造方法中会loadClass这个SunCpuAccessor类，然后去创建这个的实例

  这个实例有可能构建失败，因为这是通过OperatingSystemMXBean获取的，而这个OperatingSystemMXBean并不是所有的jdk都有

  ```
  public class SunCpuAccessor extends CPUMetricsAccessor {
      private final OperatingSystemMXBean osMBean;
  
      public SunCpuAccessor(int cpuCoreNum) {
          super(cpuCoreNum);
          this.osMBean = (OperatingSystemMXBean)ManagementFactory.getOperatingSystemMXBean();
          this.init();
      }
  
      @Override
      protected long getCpuTime() {
          return osMBean.getProcessCpuTime();
      }
  }
  ```

* init方法会在他的父类执行

  ```
  public abstract class CPUMetricsAccessor {
      private long lastCPUTimeNs;
      private long lastSampleTimeNs;
      private final int cpuCoreNum;
  
      public CPUMetricsAccessor(int cpuCoreNum) {
          this.cpuCoreNum = cpuCoreNum;
      }
  
      protected void init() {
          lastCPUTimeNs = this.getCpuTime();
          lastSampleTimeNs = System.nanoTime();
      }
  
      protected abstract long getCpuTime();
  
      public CPU getCPUMetrics() {
          long cpuTime = this.getCpuTime();
          long cpuCost = cpuTime - lastCPUTimeNs;
          long now = System.nanoTime();
  
          try {
              CPU.Builder cpuBuilder = CPU.newBuilder();
              return cpuBuilder.setUsagePercent(cpuCost * 1.0d / ((now - lastSampleTimeNs) * cpuCoreNum) * 100).build();
          } finally {
              lastCPUTimeNs = cpuTime;
              lastSampleTimeNs = now;
          }
      }
  }
  ```

  MemoryProvider类似底层采用MemoryMXBean获取

  MemoryPoolProvider内部不同的垃圾回收器实现不一样

  这里需要根据垃圾回收器名字判断是什么垃圾回收器，然后返回不同的垃圾回收名称，因为不同的垃圾回收器的内存区域叫法可能不一样

  ```
  public enum MemoryPoolProvider {
      INSTANCE;
  
      private MemoryPoolMetricsAccessor metricAccessor;
      private List<MemoryPoolMXBean> beans;
  
      MemoryPoolProvider() {
          beans = ManagementFactory.getMemoryPoolMXBeans();
          for (MemoryPoolMXBean bean : beans) {
              String name = bean.getName();
              MemoryPoolMetricsAccessor accessor = findByBeanName(name);
              if (accessor != null) {
                  metricAccessor = accessor;
                  break;
              }
          }
          if (metricAccessor == null) {
              metricAccessor = new UnknownMemoryPool();
          }
      }
  
      public List<MemoryPool> getMemoryPoolMetricsList() {
          return metricAccessor.getMemoryPoolMetricsList();
      }
  
      private MemoryPoolMetricsAccessor findByBeanName(String name) {
          if (name.indexOf("PS") > -1) {
              //Parallel (Old) collector ( -XX:+UseParallelOldGC )
              return new ParallelCollectorModule(beans);
          } else if (name.indexOf("CMS") > -1) {
              // CMS collector ( -XX:+UseConcMarkSweepGC )
              return new CMSCollectorModule(beans);
          } else if (name.indexOf("G1") > -1) {
              // G1 collector ( -XX:+UseG1GC )
              return new G1CollectorModule(beans);
          } else if (name.equals("Survivor Space")) {
              // Serial collector ( -XX:+UseSerialGC )
              return new SerialCollectorModule(beans);
          } else {
              // Unknown
              return null;
          }
      }
  }
  ```

### 2.5. sender

这个类实现了GRPCChannelListener

```
private class Sender implements Runnable, GRPCChannelListener {
    private volatile GRPCChannelStatus status = GRPCChannelStatus.DISCONNECT;
    private volatile JVMMetricReportServiceGrpc.JVMMetricReportServiceBlockingStub stub = null;

    @Override
    public void run() {
        if (RemoteDownstreamConfig.Agent.SERVICE_ID != DictionaryUtil.nullValue()
            && RemoteDownstreamConfig.Agent.SERVICE_INSTANCE_ID != DictionaryUtil.nullValue()
        ) {
            if (status == GRPCChannelStatus.CONNECTED) {
                try {
                    JVMMetricCollection.Builder builder = JVMMetricCollection.newBuilder();
                    LinkedList<JVMMetric> buffer = new LinkedList<JVMMetric>();
                    queue.drainTo(buffer);
                    if (buffer.size() > 0) {
                        builder.addAllMetrics(buffer);
                        builder.setServiceInstanceId(RemoteDownstreamConfig.Agent.SERVICE_INSTANCE_ID);
                        Commands commands = stub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS).collect(builder.build());
                        ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
                    }
                } catch (Throwable t) {
                    logger.error(t, "send JVM metrics to Collector fail.");
                }
            }
        }
    }

    @Override
    public void statusChanged(GRPCChannelStatus status) {
        if (GRPCChannelStatus.CONNECTED.equals(status)) {
            Channel channel = ServiceManager.INSTANCE.findService(GRPCChannelManager.class).getChannel();
            stub = JVMMetricReportServiceGrpc.newBlockingStub(channel);
        }
        this.status = status;
    }
}
```

* run方法启动后，如果SERVICE_ID和service_instance_id初始化了

* 如果GRPCChannelStatus.CONNECTED连接成功

* 创建jvm的builder

  JVMMetricCollection.Builder builder = JVMMetricCollection.newBuilder();

*  queue.drainTo(buffer);

  把queue的数据拿到budder之中

* ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);

  将数据发送走

## 3. 总结

参考文章https://www.bilibili.com/video/BV1fv411z7ks/?spm_id_from=333.788
