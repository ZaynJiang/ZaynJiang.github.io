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

## 2. prometheus搭建

这里我们使用k8s搭建，然后还对其自身进行监控

### 2.1. k8s概念

操作pod的工作都是由控制器controller完成的，k8s控制器组件有多种。

- Deployment

  无状态容器控制器，通过控制ReplicaSet来控制pod；应用场景web应用；

- StatefulSet

  有状态容器控制器，保证pod的有序和唯一，pod的网络标识和存储在pod重建前后一致；应用场景主从架构；

- DeamonSet

  守护容器控制器，确保所有节点上有且仅有一个pod；应用场景监控；

- Job

  普通任务容器控制器，只会执行一次；应用场景离线数据处理；

- CronJob

  定时任务容器控制器，定时执行；应用场景通知、备份

### 2.2. 手动安装

我们需要在K8s集群设置一个全局的名称空间

```
kubectl create ns kube-ops
```

#### 2.2.1. 配置configmap

为了配置一个prometheus-cms.yaml，使用数据卷的形式进行configmap热更新。

如果configmap对应的配置文件发生变化后，pod就会热更新

配置如下：

```
TODO
```

#### 2.2.2. 配置deploy

配置deploy控制器，来保证集群，其中重要的配置项为：

* 通过--config.file参数

  指定了配置文件，然后将上面创建的ConfigMap文件以Volume形式挂载到Pod 中。

* 通过--storage.tsdb.path参数

  指定了内置 TSDB文件存储的路径，这里只是为了简单说明，使用了最简单的emptyDir来挂载，如果是在生产环境中使用的，则切记这里应该使用PV/PVC等数据持久化方案。

* 通过--web.enable-lifecycle参数

  开启热更新支持，在使用该参数后，配置文件若发生任何变化，则都可以通过HTTP POST请求localhost:9090/-/reload接口操作使之立即生效。

```
TODO
```

#### 2.2.3. 配置ServiceAccount

为了让集群内的 Pod安全访问Kubernetes接口，Kubernetes提供了ServiceAccount对象。

其主要原理是：**将访问k8s api的token和ca证书挂载到容器内部，那么容器的应用程序就可以访问k8s api了**,prometheus的服务发现机制就是依赖该机制。

因此需要创建ServiceAccount配置

如下

```
TODO
```

#### 2.2.4. 配置RBAC

还需要创建访问权限配置prometheus-rbac.yaml

```
todo
```

该配置中，需要将上一步创建的ServiceAccount绑定到这个rbac的ClusterRole（集群角色)之中。该集群角色声明了Prometheus Pod访问集群所需的权限规则。

#### 2.2.5. 配置Service

在Pod创建完毕后，需要将服务暴露给外部用户进行访问，可以通过一个Ingress对象或者NodePort类型的Service来完成该功能。这里使用NodePort来暴露服务，新建一个名为prometheus-svc.yarm

```
TODO
```

设置端口，目标端口等信息

#### 2.2.6. 创建和启动

以上已经把手动配置的项都完成了。这里我们使用kubectl命令安装

* 创建资源

  ```
  kubectl create -f prometheus-cm.yaml
  kubectl create -f prometheus-rbac.yaml
  kubectl create -f prometheus-deploy.yaml
  kubectl create -f prometheus-svc.yaml
  ```

* 检查

  ```
  get all -n kube-ops
  ```

* 验证dashboard

  访问如下地址就可以查看到了prometheus的dashboard了

  ```
  http://节点:端口
  ```

### 2.2. opearator安装

​	Operator是由CoreOS公司开发的用来扩展Kubernetes API的特定应用程序控制器，用来创建、配置和管理复杂的有状态应用，例如数据库、缓存和监控系统。

​	而prometheus本身有多种组件，比如AlertManager，这些组件服务本身的高可用，虽然完全可以用自定义的方式来实现，但是不够灵活，不具有通用性，手动安装时需要不断更新Prometheus的配置来实现。

​	我们完全可以采用一种更高级方式来实现Kubernetes集群监控，即采用Prometheus Operator项目的安装。Prometheus Operator就是基于Operator框架开发的管理 Prometheus集群的控制器。

![image-20221027194741346](image-20221027194741346.png) 

#### 2.2.1. 资源对象(crd)

Operator是核心部分，是作为一个控制器而存在。Operator会创建Prometheus .ServiceMonitor、AlertManager 及 PrometheusRule这4个CRD资源对象，然后一直监控并维持这4个CRD资源对象的状态。

* Prometheus资源对象

  作为Prometheus Server存在的。

* ServiceMonitor资源对象

  专门提供 metrics数据接口的exporter的抽象

* Alertmanager资源对象

  对应AlertManager组件的抽象。

* PrometheusRule资源对象

  被Prometheus实例使用的告警规则文件的抽象。

#### 2.2.2. ServiceMonitor

ServiceMonitor是通过kubelet的10250端口采集节点数据的。为了保证安全性，将metrics数据迁移到10255这个只读端口，只需将文件中的https-metrics更改成http-metrics 即可

```
TODO
```

#### 2.2.3. 自定义etcd监控项

自定义监控项的通用步骤为：

* 建立一个 ServiceMonitor对象，用于为Prometheus添加监控项

* 将ServiceMonitor对象关联metrics数据接口的一个Service对象
* Service对象可以正确获取metrics数据。

针对etcd集群我们对其进行监控步骤为：

* 配置etcd证书

  ```
  TODO
  ```

* 创建servicemonitor

  ```
  TODO
  ```

* 创建Service

  ```
  TODO
  ```

  这里创建的Service没有采用前面通过标签匹配Pod的做法。因为创建的 etcd 集群大多独立于集群之外，所以在这种情况下需要自定义一个Endpoints。注意metadata区域的内容要和Service 保持一致，并且将Service 的clusterIP设置为None。

#### 2.2.4. 自定义告警

* 配置规则

  在prometheus的config页面查看alertmanager的配置。

  prometheus发现alertmanager。

  * 配置AlertManagers发现

    对AlertManagers 实例的配置，是通过role为 endpoints的 Kubernetes 的服务发现机制获取的。

    匹配的是服务名为alertmanager-main且端口名为web的Service.

  * 配置规则

    创建规则文件

  * 发现规则

    rule 文件已经被注入对应的rulefiles文件夹下了

* 配置告警

  alertmanger可以配置文件配置各种告警接收器。配置信息源码目录contrib/kube-prometheus/manifests来自于创建的alertmanager-secret.yaml文件。

  对AlertManager的配置也可以使用模板（ .tmpl文件)，这些模板可以与alertmanager.yam1 配置文件一起添加到Secret对象中。

  模板会被放置到与配置文件相同的路径下，当然，要使用这些模板文件，则还需要在alertmanager.yaml配置文件中指定

  在创建成功后, Secret对象将会被挂载到AlertManager对象创建的AlertManagerPod 中.

#### 2.2.5. 高级配置

Prometheus Operator为我们提供了自动发现机制，可以通过自定义发现机制动态发现监控对象

* crd配置

  如果需要自动发现集群中的Service，则需要在Service 的 annotation 区域添加prometheus.io/scrape=true声明, crd文件直接保存为prometheus-additional.yaml,然后通过这个文件创建一个对应的Secret对象。

  然后需要在声明Prometheus的资源对象文件（源码目录contrib/kube-prometheus/manifests/下名为 prometheus-prometheus的 YAML文件）中添加这个额外的配置。然后apply配置。

  Prometheus的 Dashboard 中查看可以查看配置

* 持久化

  需要创建storageclass对象。然后关联具体存储引擎

## 3. prometheus监控对象

前面我们配置和部署了prometheus自身和其监控。我们如何监控其它服务呢？

### 3.1. 监控方式

#### 3.1.1. 静态配置

如果服务提供了metric接口，可以直接配置：要在 Prometheus的 Pod中访问其他服务的数据,典型的做法是使用FQDN形式的URL路径,比如在kube-system命名空间下需要监控一个名为traefik的服务，配置的地址就是traefik.kube-system.svc.cluster.local。

如果服务本身没有提供metrics数据接口,就需要借助exporter服务来实现，比如现在需要对一个 Redis服务进行监控，就可以通过redis-exporte获取 Redis 的监控数据。

​	对于这类应用，在一般情况下会以一个**SideCar 容器**的形式将其与主应用部署在同一个Pod 中，创建一个Redis 服务的资源清单文件 prome-redis.yaml 

```
TODO
```

在redis这个Pod中包含了两个容器

一个是redis程序本身

另一个是用于监控redis 的 redis_exporter程序。

**通过Kubernetes Service将这两个程序暴露到不同的端口上。**

验证成功后，在 Prometheus的配置文件的scrape_configs下添加一个抓取任务:

```
- job_name: 'redis'
	static_configs:
	- targets: [ 'redis: 9121']
```

由于这里的 Redis服务和 Prometheus服务同在一个命名空间下，所以直接使用服务名即可访问,用完整的FQDN形式的地址也是可以的。

在更新完成后前往 Prometheus 的 Dashboard 中查看Targets页面的变化

#### 3.1.2. 节点级监控

由于在3个节点上都运行了node-exporter 程序，所以如果通过一个 Service将数据收集到一起，用静态配置的方式配置到 Prometheus中，那么在 Prometheus中就只会显示一条数据，还得自行在指标数据中过滤每个节点的数据，而且数据显然会丢失。

在Kubernetes下， Promethues通过与Kubernetes API集成来完成自动发现，目前主要支持5种服务发现模式:Node、Service、Pod、Endpoints和 Ingress。

* 配置服务发现job

  replace端口

  标签

### 3.2. 监控对象

​	前面我们知道了Prometheus自动发现 Kubernetes集群的节点，用到了Prometheus针对Kubernetes 的服务发现机制 kubernetes_sd_configs。我们这里通过以上的方式对不同的对象进行监控。

#### 3.2.1. 容器监控

**配置方式：**

**指标：**

* cpu
* 内存
* io
* 网络

#### 3.2.2. apiserver监控

apiserver是Kubernetes的核心组件，对它进行监控是非常有必要的，可以直接通过Kubernetes的Service获取监控数据。

**配置方式：**

​	用到role 为endpoints 的kubernetes_sd_configs

```
TODO
```

​	如何需要监控其它组件，比如controller manager、schdule则。需要手动创建单独的Service(因为在默认情况下，这些组件没有对应的Service )，其中kube-sheduler 的指标数据端口为10251，kube-controller-manager对应的端口为10252

**指标**

除了针对容器本身的性能监控，还有针对Kubernetes整个系统的监控。

Kubernetes的每个组件都提供了metrics数据接口,Prometheus可以很方便地获取整个Kubernetes集群的运行状态。可以通过PromQL来计算各种指标。

#### 3.2.3.  service监控

上面的apiserver实际上是一种特殊的Service，pod也一样。现在同样配置一个任务来专门发现普通类型的Service

配置job及其过滤规则。

```
TODO
```

#### 3.2.4.  kube-state-metrics

​	前面配置了自动发现Service ( Pod也是一样的）的监控，但这些监控数据都是应用内部的监控，需要应用本身提供一个metrics数据接口或者对应的exporter来暴露对应的指标数据。

​	在Kubernetes集群上也需要监控 Pod、DaemonSetDeployment、Job、CronJob 等资源对象的状态，这可以反映出使用这些资源部署的应用的状态。虽然通过查看前面从集群中拉取的指标（这些指标主要来自在 apiserver
和kubelet 中集成的 cAdvisor )，并没有具体的各种资源对象的状态指标。

​	对于Prometheus来说，当然需要引人新的exporter来暴露这些指标，Kubernetes提供的名为kube-state-metrics的项目。

​	将kube-state-metrics部署到Kubernetes中后会发现，Kubernetes集群中的Prometheus在 kubernetes-service-endpoints这个Job下自动发现 kube-state-metrics,并开始拉取metrics ，这是因为部署kube-state-metrics 的manifest定义文件kube-state-metrics-service.yaml对 Service 的定义包含prometheus.io/scrape: 'true'这样的 annotation。

​	因此kube-state-metrics的 Endpoint可以被Prometheus自动服务发现。

#### 3.2.5.  主机监控

主机监控是通过node_exporter实现的。

要达到监控所有集群节点的目的，可以通过 DaemonSet控制器来部署node-exporter，这样每个节点就都会自动运行一个Pod。如果从集群中删除或者添加节点，则也会进行自动扩展。创建一个名为prome-node-exporter的YAML资源清单文件，文件内容如下:

```
TODO
```

注意这个pod需要配置一些共享目录。

还有这里使用kubeadm搭建的,所以如果希望Master节点也一起被监控，则需要添加相应的Kubernetes Toleration(容忍）策略，因为使用kubeadm搭建的集群在默认情况下会将Master节点添加上污点。

因为已经指定了hostNetwork=true,所以在每个节点上都会监控一个9100端口，可以尝试通过这个端口去获取监控指标数据。所以不需要创建Service。

或者可以采用heml工具来进行部署。

## 4. AlertManger

### 4.1. 安装

安装alertmanager,并设置告警策略，还可以设置合并、过滤、静默、降噪，通知类型、设置通知人等等

### 4.2. 告警规则

设置如何触发生成一条告警

### 4.3. webhook

​	前面配置的是 AlertManager自带的邮件告警模板，AlertManager支持多种告警接收器，比如 slack、微信等，其中最为灵活的当属 webhook。我们可以定义一个webhook来接收告警信息，然后在webhook里进行处理，并自定义需要发送怎样的告警信息.

## 5. grafana

### 5.1. 安装

### 5.2. 配置数据源

在主页，我们可以根据自己的需求手动新建一个Dashboa，除此之外，在Grafana的官方网站上还有很多公共的 Dashboard可供使用。这里可以使用Kubernetes cluster monitoring这个 Dashboard来展示Kubernetes集群的监控信息。可以在线添加或者手动添加

### 5.3. 插件

比如可以直接配置k8s集群，然后添加

###  5.4. 告警