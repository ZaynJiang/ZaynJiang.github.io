## 1. 开头    


## 2. applicationMaster的启动
* 当我们使用yarn-session启动集群时会向resourceManagement（yarn的资源管理器）发起请求。  
  会调用appMaster:org.apache.flink.yarn.YarnClusterDescriptor#startAppMaster方法启动  
  该方法也会用于提交jobgraph（即per-job模式，会同时启动集群和提交job）
  
  ```
      private ApplicationReport startAppMaster(
            Configuration configuration,
            String applicationName,
            String yarnClusterEntrypoint,
            JobGraph jobGraph,
            YarnClient yarnClient,
            YarnClientApplication yarnApplication,
            ClusterSpecification clusterSpecification)
            throws Exception {
  ```
  
* hadoop会创建容器启动org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint#main的方法
* 上述的方法会执行ClusterEntrypoint.runClusterEntrypoint(yarnJobClusterEntrypoint);
* 接着调用org.apache.flink.runtime.entrypoint.ClusterEntrypoint#startCluster启动集群
* runCLuster会启动各种服务组件
  ```
     private void runCluster(Configuration configuration, PluginManager pluginManager)
            throws Exception {
        synchronized (lock) {
            initializeServices(configuration, pluginManager);
  
            // write host information into configuration
            configuration.setString(JobManagerOptions.ADDRESS, commonRpcService.getAddress());
            configuration.setInteger(JobManagerOptions.PORT, commonRpcService.getPort());
  
            final DispatcherResourceManagerComponentFactory
                    dispatcherResourceManagerComponentFactory =
                            createDispatcherResourceManagerComponentFactory(configuration);
  
            clusterComponent =
                    dispatcherResourceManagerComponentFactory.create(
                            configuration,
                            ioExecutor,
                            commonRpcService,
                            haServices,
                            blobServer,
                            heartbeatServices,
                            metricRegistry,
                            executionGraphInfoStore,
                            new RpcMetricQueryServiceRetriever(
                                    metricRegistry.getMetricQueryServiceRpcService()),
                            this);
  
            clusterComponent
                    .getShutDownFuture()
                    .whenComplete(
                            (ApplicationStatus applicationStatus, Throwable throwable) -> {
                                if (throwable != null) {
                                    shutDownAsync(
                                            ApplicationStatus.UNKNOWN,
                                            ShutdownBehaviour.STOP_APPLICATION,
                                            ExceptionUtils.stringifyException(throwable),
                                            false);
                                } else {
                                 .......................
                                }
                            });
        }
    }
  ```



## 3. dispatcher组件  
&emsp;&emsp;上述applicationmaster启动时会创建主要的组件，其中dispatcher组件非常重要。

### 3.1. 启动过程
* 创建DispatcherResourceManagerComponent  
  上面的启动会进行创建
```
            clusterComponent =
                    dispatcherResourceManagerComponentFactory.create(
                            configuration,
                            ioExecutor,
                            commonRpcService,
                            haServices,
                            blobServer,
                            heartbeatServices,
                            metricRegistry,
                            executionGraphInfoStore,
                            new RpcMetricQueryServiceRetriever(
                                    metricRegistry.getMetricQueryServiceRpcService()),
                            this);
```
dispatcher是集群的主要组件，它的主要功能为：
* 主要负责JobGraph的接收
* 根据JobGraph启动JobManager
* RpcEndpoint服务
* 通过DispatcherGateway Rpc对外提供服务
* 从ZK中恢复JobGraph (HA模式)保存整个集群的Job运行状态    


* 创建dispatcherRunner
  ```
       dispatcherRunner =
                    dispatcherRunnerFactory.createDispatcherRunner(
                            highAvailabilityServices.getDispatcherLeaderElectionService(),
                            fatalErrorHandler,
                            new HaServicesJobGraphStoreFactory(highAvailabilityServices),
                            ioExecutor,
                            rpcService,
                            partialDispatcherServices);
  ```


* dispatcherRunner启动DispatcherLeaderProcess
```
    private DispatcherRunnerLeaderElectionLifecycleManager(
            T dispatcherRunner, LeaderElectionService leaderElectionService) throws Exception {
        this.dispatcherRunner = dispatcherRunner;
        this.leaderElectionService = leaderElectionService;

        leaderElectionService.start(dispatcherRunner);
    }
```

```
    public void grantLeadership(UUID leaderSessionID) {
        runActionIfRunning(() -> startNewDispatcherLeaderProcess(leaderSessionID));
    }

    private void startNewDispatcherLeaderProcess(UUID leaderSessionID) {
        stopDispatcherLeaderProcess();

        dispatcherLeaderProcess = createNewDispatcherLeaderProcess(leaderSessionID);

        final DispatcherLeaderProcess newDispatcherLeaderProcess = dispatcherLeaderProcess;
        FutureUtils.assertNoException(
                previousDispatcherLeaderProcessTerminationFuture.thenRun(
                        newDispatcherLeaderProcess::start));
    }
```
如下图为dispatcher的启动流程  
![](dispatcher启动流程.png)  


### 3.2. 接收job
* 接这个上面的dispatcherleaderprocess。会启动dispatchgateway。  、
  ```
      protected void onStart() {
        final DispatcherGatewayService dispatcherService =
                dispatcherGatewayServiceFactory.create(
                        DispatcherId.fromUuid(getLeaderSessionId()),
                        Collections.singleton(jobGraph),
                        ThrowingJobGraphWriter.INSTANCE);
  
        completeDispatcherSetup(dispatcherService);
    }
  ```
* dispatcher的submitjob会接收提交的任务 
  ```
     public CompletableFuture<Acknowledge> submitJob(JobGraph jobGraph, Time timeout) {
        log.info("Received JobGraph submission {} ({}).", jobGraph.getJobID(), jobGraph.getName());
        try {
            if (isDuplicateJob(jobGraph.getJobID())) {
                final DuplicateJobSubmissionException exception =
                        isInGloballyTerminalState(jobGraph.getJobID())
                                ? DuplicateJobSubmissionException.ofGloballyTerminated(
                                        jobGraph.getJobID())
                                : DuplicateJobSubmissionException.of(jobGraph.getJobID());
                return FutureUtils.completedExceptionally(exception);
            } else if (isPartialResourceConfigured(jobGraph)) {
                return FutureUtils.completedExceptionally(
                        new JobSubmissionException(
                                jobGraph.getJobID(),
                                "Currently jobs is not supported if parts of the vertices have "
                                        + "resources configured. The limitation will be removed in future versions."));
            } else {
                return internalSubmitJob(jobGraph);
            }
        } catch (FlinkException e) {
            return FutureUtils.completedExceptionally(e);
        }
    }
  ```
* 持久化和运行job
  ```
  
    private void persistAndRunJob(JobGraph jobGraph) throws Exception {
        jobGraphWriter.putJobGraph(jobGraph); //持久化job
        runJob(jobGraph, ExecutionType.SUBMISSION);//运行
    }
  ```
* 创建job manager并启动
  ```
      private void runJob(JobGraph jobGraph, ExecutionType executionType) throws Exception {
        Preconditions.checkState(!runningJobs.containsKey(jobGraph.getJobID()));
        long initializationTimestamp = System.currentTimeMillis();
        JobManagerRunner jobManagerRunner =
                createJobManagerRunner(jobGraph, initializationTimestamp);
  ```
  ```
      JobManagerRunner createJobManagerRunner(JobGraph jobGraph, long initializationTimestamp)
            throws Exception {
        final RpcService rpcService = getRpcService();
  
        JobManagerRunner runner =
                jobManagerRunnerFactory.createJobManagerRunner(
                        jobGraph,
                        configuration,
                        rpcService,
                        highAvailabilityServices,
                        heartbeatServices,
                        jobManagerSharedServices,
                        new DefaultJobManagerJobMetricGroupFactory(jobManagerMetricGroup),
                        fatalErrorHandler,
                        initializationTimestamp);
        runner.start(); //启动job manager
        return runner;
    }
  ```
  具体的流程如：  
  ![](dispatcher接受job.png)  

### 3.3. dispatcher核心成员
![](dispatcher核心组件.png)  



## 4. resourcemanager组件   

resoucemanager负责集群资源的管理 

![](resourcemanager流程图.png)    
### 4.1. resourcemanager启动
* 和dispatcher的启动入口类似
```
private void runCluster(Configuration configuration, PluginManager pluginManager)
        throws Exception {
synchronized (lock) {
        initializeServices(configuration, pluginManager);

        // write host information into configuration
        configuration.setString(JobManagerOptions.ADDRESS, commonRpcService.getAddress());
        configuration.setInteger(JobManagerOptions.PORT, commonRpcService.getPort());

        final DispatcherResourceManagerComponentFactory
                dispatcherResourceManagerComponentFactory =
                        createDispatcherResourceManagerComponentFactory(configuration);
}
```

* 开启resourcemanager
  ```
      private void startNewLeaderResourceManager(UUID newLeaderSessionID) throws Exception {
        stopLeaderResourceManager();
  
        this.leaderSessionID = newLeaderSessionID;
        this.leaderResourceManager =
                resourceManagerFactory.createResourceManager(
                        rmProcessContext, newLeaderSessionID, ResourceID.generate());
  
        final ResourceManager<?> newLeaderResourceManager = this.leaderResourceManager;
  
        previousResourceManagerTerminationFuture
                .thenComposeAsync(
                        (ignore) -> {
                            synchronized (lock) {
                                return startResourceManagerIfIsLeader(newLeaderResourceManager);
                            }
                        },
                        handleLeaderEventExecutor)
                .thenAcceptAsync(
                        (isStillLeader) -> {
                            if (isStillLeader) {
                                leaderElectionService.confirmLeadership(
                                        newLeaderSessionID, newLeaderResourceManager.getAddress());
                            }
                        },
                        ioExecutor);
    }
  ```


### 4.2. resourcemanager资源分配
#### 4.2.1. resourcemanager资源 
![](task-manager的slot资源.png)  
* TM有固定数量的Slot资源
* Slot数量由配置决定
* Slot资源由TM资源及Slot数量决定
* 同一TM上的Slot之间无差别  

#### 4.2.2. taskMananger资源类型
resourcemanager管理的资源主要类型有：  
* 内存
* cpu
* 其它拓展资源（gpu）  

分配的时候主要就分配这些资源。  
内存资源的类型如下：   
![](taskmanager的内存资源.png)  

#### 4.2.3. resourcemanager申请资源流程
其流程为：
* activemanager接收newwork请求
* 资源信息封装到WorkerResourceSpec  
  ```
  
    private final CPUResource cpuCores;
  
    private final MemorySize taskHeapSize;
  
    private final MemorySize taskOffHeapSize;
  
    private final MemorySize networkMemSize;
  
    private final MemorySize managedMemSize;
  
    private final int numSlots;
  
    private final Map<String, ExternalResource> extendedResources;
  ```
* processSpecFromWorkerResourceSpec来封装TaskExecutorProcessSpec  
  ```
        * <pre>
        *               ┌ ─ ─ Total Process Memory  ─ ─ ┐
        *                ┌ ─ ─ Total Flink Memory  ─ ─ ┐
        *               │ ┌───────────────────────────┐ │
        *                ││   Framework Heap Memory   ││  ─┐
        *               │ └───────────────────────────┘ │  │
        *               │ ┌───────────────────────────┐ │  │
        *            ┌─  ││ Framework Off-Heap Memory ││   ├─ On-Heap
        *            │  │ └───────────────────────────┘ │  │
        *            │   │┌───────────────────────────┐│   │
        *            │  │ │     Task Heap Memory      │ │ ─┘
        *            │   │└───────────────────────────┘│
        *            │  │ ┌───────────────────────────┐ │
        *            ├─  ││   Task Off-Heap Memory    ││
        *            │  │ └───────────────────────────┘ │
        *            │   │┌───────────────────────────┐│
        *            ├─ │ │      Network Memory       │ │
        *            │   │└───────────────────────────┘│
        *            │  │ ┌───────────────────────────┐ │
        *  Off-Heap ─┼─   │      Managed Memory       │
        *            │  ││└───────────────────────────┘││
        *            │   └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
        *            │  │┌─────────────────────────────┐│
        *            ├─  │        JVM Metaspace        │
        *            │  │└─────────────────────────────┘│
        *            │   ┌─────────────────────────────┐
        *            └─ ││        JVM Overhead         ││
        *                └─────────────────────────────┘
        *               └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
        * </pre>
        */
        public class TaskExecutorProcessSpec extends CommonProcessMemorySpec<TaskExecutorFlinkMemory> {
  ```
* YarnResourceManagerDriver#requestResource来申请资源


### 4.3. resourcemanager资源分配
在上面的资源申请流程中，activemanager会充当taskmanager的管理工作。 其实taskmanager的管理有两大类：  
* Standalone部署模式
* ActiveResourceManager部署模式
  * Kubernetes,Yarn,Mesos
  * Slot数量按需分配，根据Slot Request请求数量启动TaskManager.
  * TaskManager空闲一段时间后，超时释放
  * On-Yarn部署模式不再支持固定数量的TaskManager   

### 4.4. taskmanager的task调度图
![](taskmanager调度整体流程图.png)  
* SlotRequest
  * task + slot -> allocate taskslot
* slot sharing
  * slot sharing group任务共享slot计算资源
  * 单个slots中相同任务只能有一个。    
  

![](task在slot资源分布.png)

