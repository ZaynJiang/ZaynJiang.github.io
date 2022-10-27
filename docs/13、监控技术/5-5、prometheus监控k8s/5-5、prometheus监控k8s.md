## 1. k8s监控体系

### 1.1. k8s介绍

​	Kubernetes是Google基于 Borg开源的容器编排调度引擎。Kubernetes 的目标不止是成为一个编排系统，更是提供一种规范，该规范可以让我们描述集群的架构并定义服务的最终状态，让系统自动达到和维持这种状态。

​	因此Kubernetes相当于一个云操作系统了。

### 1.2. k8s监控指标

对于k8s的监控指标，主要分为三类。

* 节点

  节点的CPU、 Load、Disk、Memory等指标。

* 内部系统组件的状态

  比如kube-scheduler 、 kube-controller-manager 、Kube-DNS/CoreDNS等组件的详细运行状态。

* 编排级的metrics

  比如 Deployment、Pod、Daemonset、StatefulSet等资源的状态、资源请求、调度和API延迟等数据指标。

### 1.3. heapster

早期（Kubernetes 1.10.x之前）k8s提供了heapster，并InfluxDB和Grafana 的组合来对系统进行监控。**Heapster 用于对集群进行监控和数据聚合，以 Pod 的形式运行在集群中**，在正确部署后可以在 Kubernetes提供的dashboard插件页面查看采集的数据

同时也可以使用kubectl top命令查看集群中 Node和 Pod 的资( CPU、Memory 、 Storage )使用情况

### 1.4.  cAdivsor

​	kubelet包含一个cAdivsor组件。它是 Google开源的容器资源监控和性能分析工具，它为容器而生，不需要单独安装cAdvisor。cAdvisor作为kubelet内置的一部分程序，可以直接使用。kubelet服务在启动时会自动启动cAdvisor服务，**cAdvisor会实时采集所在节点及在节点上运行的容器的性能指标数据**。

​	**而Heapster（pod）会通过kubelet 提供的API获取各个Node节点相关的监控指标数据**，并将其汇总后发送给后台支持的数据库，比如OpenTSDB、Kafka、Elasticsearch ,InfluxDB等。在一般情况下，我们直接将InfluxDB作为数据存储后端，并通过Grafana展示InfluxDB采集的数据。通过Grafana展示 InfluxDB存储数据的界面。

### 1.5.  kube-state-metric

heapster除了获取cAdivsor的数据，还会添加其它数据，比如kube-state-metric。

kube-state-metric通过监听api server生成的资源对象的状态指标。

这个数据包括：

* Deployment调度了多少个Pod副本

* 现在可用的有几个
* 有多少个Pod是 running.
* stopped 或terminated状态?Pod重启了多少次?

kube-state-metric相当于一个工具，使用heapster会进行采集这些指标数据

### 1.6. metric server

metric server是一个针对Kubernetes监控的数据聚合工具，是 Heapster的替代品。

Kubernetes的HPA和 kubectl top命令都依赖heapster或者metrics-server，要使用这两个功能，至少要安装其中一个。

#### 1.6.1. 侧重点

* kube-state-metrics

  主要关注集群资源相关的一些元数据，比如Deployment,Pod、副本状态和资源限额等静态指标。

* metrics-server

  主要关注资源度量API的实现，比如CPU、文件描述符、内存、请求延时等实时指标。

### 1.7. Prometheus 

Prometheus 对Kubernetes也有很好的支持。

Prometheus 默认的 Pull 策略的支持，而且是在代码层面的原生支持，可以**通过Prometheus提供的Kubernetes服务发现机制来实现集群的自动化监控**。

例如，若我们在集群中部署了新的应用，Prometheus就可以自动发现 Pod、Service、Ingress等资源的监控信息，而不需要我们做任何手工配置。当我们删除一个资源时，Prometheus也会自动移除对这个应用相关资源的监控。

