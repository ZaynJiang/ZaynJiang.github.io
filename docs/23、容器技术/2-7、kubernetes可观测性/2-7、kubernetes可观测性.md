## 2. 监控

### 2.1. Prometheus

Kubernetes项目的监控体系现在已经被Prometheus"一统"，而Prometheus与Kuberentes类似，也是来自Google内部系统的设计理念

Prometheus项目工作的核心：通过pull方式拉取监控对象的metric数据，存储到时序数据库中供后续检索

时序数据库的特点：支持大批量写入、高性能搜索、聚合。

基于这样的核心，Prometheus剩下的组件就是用来配合这套机制运行，

比如 Pushgateway: 允许被监控对象以Push的方式向Prometheus推送数据 

Alertmanager：根据Metrics信息灵活地设置报警 Grafana：活动配置监控数据可视化页面

Kubernetes借助Promethus监控体系，可以提供Custom Metrics的能力，即自定义指标。Custom Metrics借助Aggregator APIServer扩展机制来实现，即对APIServer的请求，会先经过Aggreator来转发，它会根据URL来决定把请求转发给自定义的Custom Metrics APIServer，还是Kubernetes的APIServer。有了这样的体系，就可以方便的实现自定义指标的感知了。比如，现在启动了一个Custom Metrics APIServer，它对应的url是custom.metrics.k8s.io，当我需要获取某一个Pod内的自定义指标（例：http_requests）：

```
https://<apiserver_ip>/apis/custom-metrics.metrics.k8s.io/v1beta1/namespaces/default/pods/sample-metrics-app/http_requests 
```

这个请求就会被Custom Metrics APIServer接收，然后它就会去Promethus里查询名叫sample-metrics-app这个pod的http_requests指标。而Promethus可以通过定期访问Pod的一个API来采集这个指标

### 2.2. kubernetes指标

Metrics数据来源种类有:

第一种是宿主机的监控数据。最主要是借助Node Exporter，以DaemonSet的方式运行在宿主机上，收集包括节点负载（Load）,CPU,内存，磁盘以及网络这些常规信息。

第二种是来自于K8s的API Server,kubelet等组建的/metrics API 。除了常规的 CPU、内存的信息外，这部分信息还主要包括了各个组件的核心监控指标。比如，对于 API Server 来说，它就会在 /metrics API 里，暴露出各个 Controller 的工作队列（Work Queue）的长度、请求的 QPS 和延迟数据等等。这些信息，是检查 Kubernetes 本身工作情况的主要依据。

 第三种 Metrics，是 Kubernetes 相关的监控数据。这部分数据，一般叫作 Kubernetes 核心监控数据（core metrics）。这其中包括了 Pod、Node、容器、Service 等主要 Kubernetes 核心概念的 Metrics。其中，容器相关的 Metrics 主要来自于 kubelet 内置的 cAdvisor 服务。在 kubelet 启动后，cAdvisor 服务也随之启动，而它能够提供的信息，可以细化到每一个容器的 CPU 、文件系统、内存、网络等资源的使用情况

### 2.3. metric server

Metrics Server 在 Kubernetes 社区的定位，其实是用来取代 Heapster 这个项目的。有了 Metrics Server 之后，用户就可以通过标准的 Kubernetes API 来访问到这些监控数据了。Metrics Server 并不是 kube-apiserver 的一部分，而是通过 Aggregator 这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外服务的。kube-aggregator 其实就是一个根据 URL 选择具体的 API 后端的代理服务器

### 2.4. 监控设计原则

USE 原则指的是，按照如下三个维度来规划资源监控指标： 

1. 利用率（Utilization），资源被有效利用起来提供服务的平均时间占比； 

2. 饱和度（Saturation），资源拥挤的程度，比如工作队列的长度； 
3. 错误率（Errors），错误的数量

RED 原则指的是，按照如下三个维度来规划服务监控指标： 

1. 每秒请求数量（Rate）； 
2. 每秒错误数量（Errors）； 

3. 服务响应时间（Duration）；

USE 原则主要关注的是“资源”，比如节点和容器的资源使用情况，而 RED 原则主要关注的是“服务”，比如 kube-apiserver 或者某个应用的工作情况。这两种指标，在我今天为你讲解的 Kubernetes + Prometheus 组成的监控体系中，都是可以完全覆盖到的。

### 2.5. HPA

自动扩展器组件

Custom Metrics 已经成为了 Kubernetes 的一项标准能力。并且，Kubernetes 的自动扩展器组件 Horizontal Pod Autoscaler （HPA）， 也可以直接使用 Custom Metrics 来执行用户指定的扩展策略，这里的整个过程都是非常灵活和可定制的

Kubernetes 里的 Custom Metrics 机制，也是借助 Aggregator APIServer 扩展机制来实现的。这里的具体原理是，当你把 Custom Metrics APIServer 启动之后，Kubernetes 里就会出现一个叫作custom.metrics.k8s.io的 API。而当你访问这个 URL 时，Aggregator 就会把你的请求转发给 Custom Metrics APIServer 。**而 Custom Metrics APIServer 的实现，其实就是一个 Prometheus 项目的 Adaptor**

HPA 通过 HorizontalPodAutoscaler 配置要访问的 Custom Metrics, 来决定如何scale。 Custom Metric APIServer 的实现其实是一个Prometheus 的Adaptor，会去Prometheus中读取某个Pod/Servicce的具体指标值。比如，http request的请求率。 Prometheus 通过 ServiceMonitor object 配置需要监控的pod和endpoints，来确定监控哪些pod的metrics。 应用需要实现/metrics， 来响应Prometheus的数据采集请求

### 2.2. 小结

Prometheus 项目工作的核心，是使用 Pull （抓取）的方式去搜集被监控对象的 Metrics 数据（监控指标数据），然后，再把这些数据保存在一个 TSDB （时间序列数据库，比如 OpenTSDB、InfluxDB 等）当中，以便后续可以按照时间进行检索

很多 PaaS 项目中的Auto Scaling，即自动水平扩展的功能，只能依据某种指定的资源类型执行水平扩展，比如 CPU 或者 Memory 的使用值。凭借强大的 API 扩展机制，Custom Metrics 已经成为了 Kubernetes 的一项标准能力。并且，Kubernetes 的自动扩展器组件 Horizontal Pod Autoscaler （HPA）， 也可以直接使用 Custom Metrics 来执行用户指定的扩展策略，这里的整个过程都是非常灵活和可定制的。Kubernetes 里的 Custom Metrics 机制，也是借助 Aggregator APIServer 扩展机制来实现的。这里的具体原理是，当你把 Custom Metrics APIServer 启动之后，Kubernetes 里就会出现一个叫作custom.metrics.k8s.io的 API。而当你访问这个 URL 时，Aggregator 就会把你的请求转发给 Custom Metrics APIServer 。而 Custom Metrics APIServer 的实现，其实就是一个 Prometheus 项目的 Adaptor



PS：pull和push各有优缺点

prometheus的监控以pull为主.

Pull和Push两种模式的区别非常有意思，Prometheus非常大胆的采用了pull的模式，但是仔细思考后就会觉得非常适合监控的场景。 

Pull模式的特点 

1. 被监控方提供一个server，并负责维护 

2. 监控方控制采集频率 

第一点其实对用户来说要求过高了，但是好处很多，比如pull 不到数据本身就说明了节点存在故障；又比如监控指标自然而言由用户自己维护，使得标准化很简单。 

第二点其实更为重要，那就是监控系统对metric采集的统一和稳定有了可靠的保证，对于数据量大的情况下很重要。  

缺点也很明显，用户不知道你什么时候来pull一下，数据维护多久更新也不好控制，容易造成一些信息的丢失和不准确。 

当把这些优缺点权衡过后就会发现，纯监控的场景确实是适合pull的

## 2. 自定义监控

## 3. 日志

Kubernetes对容器日志的处理方式都叫做cluster-level-logging。容器默认情况下会把日志输出到宿主机上的一个JSON文件，这样，通过kubectl logs命令就可以看到这些容器的日志了。

Kuberentes提供了三种日志收集方案：

* logging agent

  pod会默认日志通过stdout/stderr输出到宿主机的一个目录，宿主机上以DaemonSet启动一个logging-agent，这个logging-agent定期将日志转存到后端。 优势： 1)对Pod无侵入 2)一个node只需要一个agent 3）可以通过kubectl logs查看日志 劣势： 必须将日志输出到stdout/stderr

* sidecar模式1

  pod将日志输出到一个容器目录，同pod内启动一个sidecar读取这些日志输出到stdout/stderr，后续就跟方案1）一样了。 优势：1）sidecar跟主容器是共享Volume的，所以cpu和内存损耗不大。2）可以通过kubectl logs查看日志

* sidercar模式2

  pod将日志输出到一个容器文件，同pod内启动一个sidecar读取这个文件并直接转存到后端存储。 优势：部署简单，宿主机友好 劣势：1） 这个sidecar容器很可能会消耗比较多的资源，甚至拖垮应用容器。2）通过kubectl logs是看不到任何日志输出的

PS:

如果每秒日志量很大时，直接输出到容器的stdout和stderr,很容易就把系统日志配额用满，因为对系统默认日志工具是针对单服务(例如docker)而不是进程进行限额的，最终导致的结果就是日志被吞掉。解决办法一个是增加配额，一个是给容器挂上存储，将日志输出到存储上。

当然日志量大也要考虑写日志时过多的磁盘读写导致整个节点的整体性能下降