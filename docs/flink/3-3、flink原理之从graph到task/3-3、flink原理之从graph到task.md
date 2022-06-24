## 1. 开头

## 2. graph
我们知道flink的客户端完成了stream graph -> job graph，然后提交到了dispatch后，会完成job graph -> excution graph -> 物理执行图
![](flink-graph转换图.png)  

### 2.1. flink graph转换
<img src="flink-graph转换图详细.png" style="zoom:150%;" />  

#### 2.1.1. program->stream graph
![](program转换streamgraph.png)    
通过图片可以知道，它是将代码拆分成算子，然后用stream将它们连接起来。   
![](program转换streamgraph2.png)    

* 执行应用的execute方法
* 获取stream graph
  ```
      public JobExecutionResult execute(String jobName) throws Exception {
        Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");
        return this.execute(this.getStreamGraph(jobName));
    }
  ```
* StreamExecutionEnvironment持有Transformation数组
* 我们在使用各种flink的api的时候，实际上那些api会调用这个方法将算子添加到Transformation数组
  ```
  	public void addOperator(Transformation<?> transformation) {
		Preconditions.checkNotNull(transformation, "transformation must not be null.");
		this.transformations.add(transformation);
	}
  ```
  
* StreamGraphGenerator生成graph的时候遍历进行生成graph
  ```
  	Collection<Integer> transformedIds;
		if (transform instanceof OneInputTransformation<?, ?>) {
			transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
		} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
  ```


* 上一步会将算子的信息放入到StreamGraph对象之中，如：
  ```
  	private <T> Collection<Integer> transformPartition(PartitionTransformation<T> partition) {
		Transformation<T> input = partition.getInput();
		List<Integer> resultIds = new ArrayList<>();
  
		Collection<Integer> transformedIds = transform(input);
		for (Integer transformedId: transformedIds) {
			int virtualId = Transformation.getNewNodeId();
			streamGraph.addVirtualPartitionNode(
					transformedId, virtualId, partition.getPartitioner(), partition.getShuffleMode());
			resultIds.add(virtualId);
		}
  
		return resultIds;
	}
  ```

* 最终会生成完整的streamgraph    


具体的streamgraph组成为：  

![](stream-graph组成.png)     

#### 2.1.2. stream graph -> job graph
![](streamgraph转换到jobgraph.png)  

* streamgraph的getjobGraph会将streamgraph生成jobgraph
  ```
        public JobGraph getJobGraph(@Nullable JobID jobID) {
                return StreamingJobGraphGenerator.createJobGraph(this, jobID);
        }
  ```
* 最终是由StreamingJobGraphGenerator来生成的    


jobgraph的核心成员为：    
![](jobgraph的核心成员.png) 


#### 2.1.3. job graph  -> executiongraph      
![](jobgraph到executiongraph.png)    
![](executiongraph组成.png)     


#### 2.1.4. execution graph -> 物理执行图
![](物理执行图转换.png)  

#### 2.1.5. ExecutionGraph调度器  
ExecutionGraph调度器分类：
* DefaultScheduler
  * 默认调度器
  * 外部调度ExecutionGraph
* LegacyScheduler  
  * 基于ExecutionGraph 内部调度
  

![](executiongraph调度器.png)
## 3. task  
### 3.1. task的class结构  
![](task的结构.png)  
```
    public Task(
            JobInformation jobInformation,
            TaskInformation taskInformation,
            ExecutionAttemptID executionAttemptID,
            AllocationID slotAllocationId,
            int subtaskIndex,
            int attemptNumber,
            List<ResultPartitionDeploymentDescriptor> resultPartitionDeploymentDescriptors,
            List<InputGateDeploymentDescriptor> inputGateDeploymentDescriptors,
            MemoryManager memManager,
            IOManager ioManager,
            ShuffleEnvironment<?, ?> shuffleEnvironment,
            KvStateService kvStateService,
            BroadcastVariableManager bcVarManager,
            TaskEventDispatcher taskEventDispatcher,
            ExternalResourceInfoProvider externalResourceInfoProvider,
            TaskStateManager taskStateManager,
            TaskManagerActions taskManagerActions,
            InputSplitProvider inputSplitProvider,
            CheckpointResponder checkpointResponder,
            TaskOperatorEventGateway operatorCoordinatorEventGateway,
            GlobalAggregateManager aggregateManager,
            LibraryCacheManager.ClassLoaderHandle classLoaderHandle,
            FileCache fileCache,
            TaskManagerRuntimeInfo taskManagerConfig,
            @Nonnull TaskMetricGroup metricGroup,
            ResultPartitionConsumableNotifier resultPartitionConsumableNotifier,
            PartitionProducerStateChecker partitionProducerStateChecker,
            Executor executor) {
```
![](task实现的接口.png)  
### 3.2. task触发和执行 
![](task的触发和执行1.png)   
![](task的触发和执行2.png)  
```
          Environment env =
                    new RuntimeEnvironment(
                            jobId,
                            vertexId,
                            executionId,
                            executionConfig,
                            taskInfo,
                            jobConfiguration,
                            taskConfiguration,
                            userCodeClassLoader,
                            memoryManager,
                            ioManager,
                            broadcastVariableManager,
                            taskStateManager,
                            aggregateManager,
                            accumulatorRegistry,
                            kvStateRegistry,
                            inputSplitProvider,
                            distributedCacheEntries,
                            consumableNotifyingPartitionWriters,
                            inputGates,
                            taskEventDispatcher,
                            checkpointResponder,
                            operatorCoordinatorEventGateway,
                            taskManagerConfig,
                            metrics,
                            this,
                            externalResourceInfoProvider);

		...................


       invokable = loadAndInstantiateInvokable(
                                userCodeClassLoader.asClassLoader(), nameOfInvokableClass, env);

		...................    

		try {
			// run the invokable
			invokable.invoke();
		} finally {
			FlinkSecurityManager.unmonitorUserSystemExitForCurrentThread();
		}

```
### 3.3. task的运行状态
![](task的运行状态.png)    

### 3.4. task重启与容错  
&emsp;&emsp;如果Task执行过程失败，Flink需要重启失败的Task以及相关的上下游Task，以恢复Job到正常状态。  
&emsp;&emsp;重启和容错策略分别用于控制Task重启的时间和范围，其中RestartStrategy决定了Task的重启时间，FailureStrategy决定了哪些Task需要重启才能恢复正常。我们已经知道，在整个ExecutionGraph调度过程中，会创建两种SchedulerNg实现类，分别为LegacyScheduler和DefaultScheduler。  
&emsp;&emsp;LegacyScheduler主要是为了兼容之前的版本，现在默认的ExecutionGraph调度器为DefaultScheduler。  
两种调度器对应的Task重启策略的实现方式也有所不同，但对用户来讲，设定重启策略参数都是一样的，仅底层代码实现有所不同。


#### 3.4.1. task重启策略  
&emsp;&emsp;Task重启策略主要分为三种类型：固定延时重启（fixed-delay）、按失败率重启（failure-rate）以及无重启（none）。

#### 3.4.2. 基于LegacyScheduler实现的重启策略  
LegacyScheduler主要是为了兼容之前的版本，现在默认的ExecutionGraph调度器为DefaultScheduler。这里略。
#### 3.4.3. 基于DefaultScheduler容错和重启策略
&emsp;&emsp;Task容错主要通过FailoverStrategy控制：  
FailoverStrategy接口目前支持：
* RestartAllStrategy  
  只要有Task失败就直接重启Job中所有的，Task实例，这样做的代价相对较大，需要对整个Job进行重启，因此不是首选项。
* RestartPipelinedRegionStrategy   
  RestartPipelinedRegionStrategy策略是只启动与异常Task相关联的Task实例。在Pipeline中通过Region定义上下游中产生数据交换的Task集合，当Region中出现失败的Task，直接重启当前Task所在的Region，完成对作业的容错，其他不在Region内的Task实例则不做任务处理，RestartPipelinedRegionStrategy是Flink中默认支持的容错策略。    
  
  FailoverStrategy有三个关键点:   
  * FailoverStrategy  
    用于控制Task的重启范围，包括重启全部Task还是重启PipelinedRegion内的相关Task  
  * RestartBackoffTimeStrategy  
  	用于控制Task是否需要重启以及重启时间和尝试次数。
  * FailoverTopology  
    用于构建Task重启时需要的拓扑、确定异常Task关联的PipelinedRegion，然后重启Region内的Task。  

  

还有更多。。。。。。。。。