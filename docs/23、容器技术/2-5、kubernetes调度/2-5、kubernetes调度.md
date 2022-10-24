## 1. 开头

## 2. 资源模型

​	Pod 是最小的原子调度单位，所有跟调度和资源管理相关的属性，都是 Pod 对象属性的字段。其中最重要的是 Pod 和 CPU 配置。其中，CPU 属于可压缩资源，内存属于不可压缩资源。当可压缩资源不足时，Pod 会饥饿；当不可压缩资源不足时，Pod 就会因为 OOM 被内核杀掉

​	Pod. 即容器组，由多个容器组成，其 CPU 和内存配额在 Container 上定义，其 Container 配置的累加值即为 Pod 的配额。

​	limits 和 requests - requests

​	kube-scheduler 只会按照 requests 的值进行计算

​    limits：kubelet 则会按照 limits 的值来进行设置 Cgroups 限制.

* QoS 模型

  * Guaranteed： 同时设置 requests 和 limits，并且 requests 和 limit 值相等。

  优势一是在资源不足 Eviction 发生时，最后被删除；并且删除的是 Pod 资源用量超过 limits 时才会被删除；优势二是该模型与 docker cpuset 的方式绑定 CPU 核，避免频繁的上下午文切换，性能会得到大幅提升。 

  * Burstable：不满足 Guaranteed 条件， 但至少有一个 Container 设置了 requests 
  * BestEffort：没有设置 requests 和 limits。

1.当集群中的计算资源不很充足, 如果集群中的pod负载突然加大, 就会使某个node的资源严重不足, 为了避免系统挂掉, 该node会选择清理某些pod来释放资源, 此时每个pod都可能成为牺牲品
2.kubernetes保障机制：

限制pod进行资源限额
允许集群资源被超额分配, 以提高集群的资源利用率
为pod划分等级, 确保不同的pod有不同的服务质量qos, 资源不足时， 低等级的pod会被清理, 确保高等级的pod正常运行

3.kubernetes会根据Request的值去查找有足够资源的node来调度此pod
limit对应资源量的上限, 既最多允许使用这个上限的资源量, 由于cpu是可压缩的, 进程是无法突破上限的, 而memory是不可压缩资源, 当进程试图请求超过limit限制时的memory, 此进程就会被kubernetes杀掉
对于cpu和内存而言, pod的request和limit是指该pod中所有容器的 Requests或Limits的总和,
例如： 某个节点cpu资源充足, 而内存为4G，其中3GB可以运行pod, 而某个pod的memory request为1GB, limit为2GB， 那么这个节点上最多可以运行3个这样的pod
待调度pod的request值总和超过该节点提供的空闲资源, 不会调度到该节点node上;

request和limit区别是：在调度的时候，kube-scheduler 只会按照 requests 的值进行计算。而在真正设置 Cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置。 当宿主机资源紧张的时间kubelet就会对Pod进行Eviction（资源回收），这时候会运用到QoS划分： 第一个会删除的是BestEffort类Pod，也就是Pod 既没有设置 requests，也没有设置 limits。 接下来第二个会删除的是Burstable类Pod。这是至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别 最后一类会删除的Pod是Guaranteed，也就是设置了request和limit，或者只设置limit的Pod。 Eviction 在 Kubernetes 里分为 Soft 和 Hard 两种模式。Soft Eviction 允许你为 Eviction 过程设置一段“优雅时间”，比如文章里的例子里的 imagefs.available=2m，就意味着当 imagefs 不足的阈值达到 2 分钟之后，kubelet 才会开始 Eviction 的过程。而 Hard Eviction 模式下，Eviction 过程就会在阈值达到之后立刻开始。 Kubernetes 计算 Eviction 阈值的数据来源，主要依赖于从 Cgroups 读取到的值，以及使用 cAdvisor 监控到的数据。 当宿主机的 Eviction 阈值达到后，就会进入 MemoryPressure 或者 DiskPressure 状态，从而避免新的 Pod 被调度到这台宿主机上。它使用taint把node taint，就无法调度Pod到其node之上了。 cpuset 方式是生产环境里部署在线应用类型的 Pod 时，非常常用的一种方式。设置 cpuset 会把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力，这样能大大减少CPU之间上下文切换次数，从而提高容器性能。设置cpuset的方式是，首先Pod 必须是 Guaranteed 的 QoS 类型；然后，你只需要将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值即可

## 3. 调度原理

messos二级调度是资源调度和业务调度分开；

优点：插件化调度框架（用户可以自定义自己调度器然后注册到messos资源调度框架即可），灵活可扩展性高

缺点：

资源和业务调度分开无法获取资源使用情况，进而无法做更细粒度的调度.

k8s调度是统一调度也就是业务和资源调度进行统一调度，可以进行更细粒度的调度；

缺点其调度器扩展性差

### 3.2. 调度算法

Predicates 和 Priorities 这两个调度策略

### 3.3. 调度失败

kubernetes使用优先级和抢占机制，解决调度失败的问题。

调度器的作用就是为Pod寻找一个合适的Node。

 调度过程：待调度Pod被提交到apiServer -> 更新到etcd -> 调度器Watch etcd感知到有需要调度的pod（Informer） -> 取出待调度Pod的信息 ->Predicates： 挑选出可以运行该Pod的所有Node  ->  Priority：给所有Node打分 -> 将Pod绑定到得分最高的Node上 -> 将Pod信息更新回Etcd -> node的kubelet感知到etcd中有自己node需要拉起的pod -> 取出该Pod信息，做基本的二次检测（端口，资源等） -> 在node 上拉起该pod 。

 Predicates阶段会有很多过滤规则：比如volume相关，node相关，pod相关 Priorities阶段会为Node打分，Pod调度到得分最高的Node上，打分规则比如： 空余资源、实际物理剩余、镜像大小、Pod亲和性等 

Kuberentes中可以为Pod设置优先级，高优先级的Pod可以： 

1、在调度队列中先出队进行调度 

2、调度失败时，触发抢占，调度器为其抢占低优先级Pod的资源。

Kuberentes默认调度器有两个调度队列： activeQ：凡事在该队列里的Pod，都是下一个调度周期需要调度的 unschedulableQ:  存放调度失败的Pod，当里面的Pod更新后就会重新回到activeQ，进行“重新调度” 

默认调度器的抢占过程： 确定要发生抢占 -> 调度器将所有节点信息复制一份，开始模拟抢占 ->  检查副本里的每一个节点，然后从该节点上逐个删除低优先级Pod，直到满足抢占者能运行 -> 找到一个能运行抢占者Pod的node -> 记录下这个Node名字和被删除Pod的列表 -> 模拟抢占结束 -> 开始真正抢占 -> 删除被抢占者的Pod，将抢占者调度到Node上 

## 4. 硬件管理

Kuberentes通过Extended Resource来支持自定义资源，比如GPU。为了让调度器知道这种自定义资源在各Node上的数量，需要的Node里添加自定义资源的数量。实际上，这些信息并不需要人工去维护，所有的硬件加速设备的管理都通过Device Plugin插件来支持，也包括对该硬件的Extended Resource进行汇报的逻辑。 

Device Plugin 、kubelet、调度器如何协同工作： 

汇报资源： Device Plugin通过gRPC与本机kubelet连接 ->  Device Plugin定期向kubelet汇报设备信息，比如GPU的数量 -> kubelet 向APIServer发送的心跳中，以Extended Reousrce的方式加上这些设备信息，比如GPU的数量  

调度： Pod申明需要一个GPU -> 调度器找到GPU数量满足条件的node -> Pod绑定到对应的Node上 -> kubelet发现需要拉起一个Pod，且该Pod需要GPU -> kubelet向 Device Plugin 发起 Allocate()请求 -> Device Plugin根据kubelet传递过来的需求，找到这些设备对应的设备路径和驱动目录，并返回给kubelet -> kubelet将这些信息追加在创建Pod所对应的CRI请求中 -> 容器创建完成之后，就会出现这个GPU设备（设备路径+驱动目录）-> 调度完成

## 5. 总结

