

## 1. 背景

### 1.1. 控制器

**pod是k8s操作的最小单元，操作pod的工作都是由控制器controller完成的**，对应k8s中的组件就是kube-controller-manager，这个组件，就是一系列控制器的集合，我们可以查看pkg/controller 目录，可以看到多个空控制器目录文件。

常见的控制器类型有：

- Deployment，无状态容器控制器，通过控制ReplicaSet来控制pod；应用场景web应用；
- StatefulSet，有状态容器控制器，保证pod的有序和唯一，pod的网络标识和存储在pod重建前后一致；应用场景主从架构；
- DeamonSet，守护容器控制器，确保所有节点上有且仅有一个pod；应用场景监控；
- Job，普通任务容器控制器，只会执行一次；应用场景离线数据处理；
- CronJob，定时任务容器控制器，定时执行；应用场景通知、备份；

#### 1.1.1. 控制器机制

它们都遵循 Kubernetes 项目中的一个通用编排模式，即：控制循环（control loop）

```
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

* 实际状态

  kubelet 通过心跳汇报的容器状态和节点状态

  监控系统中保存的应用监控数据

  控制器主动收集的它自己感兴趣的信息

* 期望状态

  一般来自于用户提交的 YAML 文件。这些信息往往都保存在 Etcd 中

#### 1.1.2. 控制器配置

控制器配置的yaml文件一般分为两个部分，上半部分是关于控制器的定义，下半部分是关于被控制对象，也就是

例如Deployment控制器配置示例如下：

```
## 上半部分，定义控制器
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  # 管理标签为app=nginx的pod
  selector:
    matchLabels:
      app: nginx
  # 确保被管理的对象数量始终为3
  replicas: 3
  ## 下半部分，定义被控制的对象，也就是pod
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Deployment、ReplicaSet、Pod三者之间是一种层层控制的关系，ReplicaSet通过控制器模式来保证pod的数量永远等于指定的个数，而Deployment同样也通过控制器的模式来操作ReplicaSet。如此，Deployment可以很方便地实现如下两个特性（并不只是Deployment控制器的功能）：水平收缩、滚动更新（https://www.jianshu.com/p/3e56b8c883c5）

StatefulSet主要是为了满足需要维持应用之间拓扑状态、稳定应用各自的网络状态、保持应用各自存储状态的情形而设计的：

* 拓扑状态

  即多个pod之间存在启动依赖关系，比如主从架构中，从节点肯定要依赖并晚于主节点才能启动，且写请求一定只能转发给主节点，这种情况下，Deployment就无法解决。StatefulSet之所以能满足该场景，是因为其给所有pod进行了编号，编号以StatefulSet Name-index命名，比如mystatefulset-0、mystatefulset-1，每个pod按照编号递增，永不重复。这些pod的创建是顺序同步进行的，前面的pod没有进入到running状态，后面的pod就会一直pending。

* 网络状态

  所有pod都进入running之后，各自都会有固定不变的网络身份，即他们的hostname，和它们的pod编号名称一样，固定不变是指无论重新创建多少次，各个pod的网络身份始终一致。这样通过headless service方式访问每个pod的地址就始终固定不变，比如mystatefulset-0.servicename、mystatefulset-1.servicename，如此StatefulSet就保证了pod网络标识的稳定性。

* 存储状态

  每个pod挂载pvc时，也会给pvc命名，命名规则为pvc-StatefulSet Name-index，比如mypvc-mystatefulset-0、mypvc-mystatefulset-1，无论pod重新创建与否，每个pod与其对应的pvc始终都会处于绑定状态。如此StatefulSet就保证了存储状态的稳定性



DaemonSet控制器确保k8s集群中的每个节点有且仅有一个定义的pod运行。那么它是如何实现这个功能的呢？

- 从etcd中获取所有node的信息，循环遍历每个node，没有守护容器的就新建，超过1个的就删除，正好就1个的就表示正常；
- 通过nodeAffinity节点亲和性来保证每个守护pod只会在和其亲和的节点上被调度运行；
- 通过tolerations声明，允许守护pod可以在被标记为污点的Node上调度运行

### 1.2. 声明式api

​	为了使用这些控制器，我们需要编写API 对象，这些api对象有的是用来描述应用，有的则是为应用提供各种各样的服务。但是，无一例外地，为了使用这些 API 对象提供的能力，你都需要编写一个对应的 YAML 文件交给 Kubernetes。

​	声明式API（kubectl apply）执行的是一个PATCH操作，一次能处理多个写操作，并具备merge的能力。 所谓的“声明式”，指的就是我只需要提交一个定义好的API对象来“声明”我所期望的状态是什么样子。“声明式API”允许多个API写端，以Patch的方式对API对象进行修改，而无需关心本地原始YAML文件的内容。正是具有以上两个重要能力，Kubernetes项目才可以基于对API对象的增、删、改、查，在完全无需外界干涉的情况下，完成对“实际状态”和“期望状态”的调谐“reconcile”过程。声明式API才是K8s的核心所在。	

​	一般来说，扩展api server (或者说添加自定义 resource )有两种方式： 

* 通过创建CRDs, 主API server可以处理 CRDs 的 REST 请求（CRUD）和持久性存储。简单，不需要其他的编程。更适用于声明式的API，和kubernetes高度集成统一。
* API Aggregation, 一个独立的API server。主API server委托此独立的API server处理自定义resource。 需要编程，但能够更加灵活的控制API的行为，更加灵活的自定义存储，以及与API不同版本之间的转换。一般更适用于命令模式，或者复用已经存在REST API代码，不直接支持kubectl 和 k8s UI, 不支持scope resource in a cluster/namespace.

​	自定义 resource 可以使你方便的存取结构化的resource 数据。但是只有和controller组合在一起才是声明式API。声明式API允许你定义一个期望的状态。controller通过解读结构化的resource数据，获得期望状态，从而不断的调协期望状态和实际状态。 

​	综上，今天文档中的types.go 应该是给controller来理解CRDs的schema用的。只有掌握了resource的schema，才能解释并得到用户创建的resource API object。 而 kubectl create -f resourcedefinition.yaml 或者 自定义API server， 则定义了RESTful API endpoint. 用于接受 REST 请求，改变 resource 的期望状态

#### 1.2.2. 声明式api不足

声明式api还是有一些本质不足的地方，主要是 

1.通常operator都是用来描述复杂有状态集群的，这个集群本身就已经很复杂了

2.声明式api通过diff的方式来得出集群要做怎么样的状态变迁，这个过程中，常常会有很多状况不能覆盖到，而如果使用者对此没有深刻的认识，就会越用越糊涂。

大致来说，如果一个集群本身的状态有n种，那么operator理论上就要覆盖n*(n-1)种变迁的可能性，而这个体力活几乎是不可能完成的，所有很多operator经常性的不支持某些操作，而不是像使用者想象的那样，我改改yaml，apply一下就完事了

更重要的是，由于情况太多太复杂，甚至无法通过文档的方式，告诉使用者，状态1的集群，按某种方式修改了yaml之后，是不能变成使用者期待的状态2的 

如果是传统的命令式方式，那么就是所有可能有的功能，都会罗列出来，可能n个状态，只有2n+3个操作是合法的，剩下都是做不到的，这样使用者固然受到了限制，但这种限制也让很多时候的操作变得更加清晰明了,而不是每次改yaml还要思考一下，这么改，operator支持吗？ 

当然，声明式api的好处也是明显的，如果这个diff是operator覆盖到的，那么就能极大的减轻使用者的负担，apply下就解脱了

而这个集群状态变迁的问题是本质复杂的，必然没有可以消除复杂度的解法，无非就是这个复杂度在operator的实现者那里，还是在运维者那里，在operator的实现者那里，好处就是固化的常用的变迁路径，使用起来很方便，但如果operator的开发者没有实现某个状态的变迁路径，而这个本身是可以做到的，这个时候，就比不上命令式灵活，个人感觉就是取舍了

## Pod API 对象

POD的直议是豆荚，豆荚中的一个或者多个豆属于同一个家庭，共享一个物理豆荚（可以共享调度、网络、存储，以及安全），每个豆虽然有自己的空间，但是由于之间的缝隙，可以近距离无缝沟通（Linux Namespace相关的属性。

Pod的定义为：

实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序

我们可以把整个虚拟机想象成为一个 Pod，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为 Init Container。然后进行编排

总的来说容器是进程，而一个pod可以包含多个容器，比如我们使用war包和tomcat做一个pod，将war包做成初始化容器，将war包复制到一个目录，而tomcat有也可以做一个容器然后就可以使用这个目录的war包进行web服务了。又比如我们的日志搜集和应用分别放到一个容器里，但是日志收集容器可以从公共目录里读取日志发送。

这里依托于可以这几个容器可以共享目录。这个借助于Infra 容器实现的。Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。

这样，一个 Volume 对应的宿主机目录对于 Pod 来说就只有一个，Pod 里的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume 对应的宿主机目录



**PS：**

* 金丝雀部署

  优先发布一台或少量机器升级，等验证无误后再更新其他机器。优点是用户影响范围小，不足之处是要额外控制如何做自动更新。

* 蓝绿部署

  2组机器，蓝代表当前的V1版本，绿代表已经升级完成的V2版本。通过LB将流量全部导入V2完成升级部署。优点是切换快速，缺点是影响全部用户



### pod生命周期

### configmap

ConfigMap与 Secret 的区别在于，ConfigMap 保存的是不需要加密的、应用所需的配置信息。而 ConfigMap 的用法几乎与 Secret 完全相同：你可以使用 kubectl create configmap 从文件或者目录创建 ConfigMap，也可以直接编写 ConfigMap 对象的 YAML 文件.

一个 Java 应用所需的配置文件（.properties 文件），我们可以通过配置保存到configmap之中

### serviceAccount

### 小结

Kuberentes可以理解为操作系统，那么容器就是进程，而Pod就是进程组or虚拟机（几个进程关联在一起）

Pod的设计之初有两个目的： 

* 为了处理容器之间的调度关系 

* 实现容器设计模式

  Pod会先启动Infra容器设置网络、Volume等namespace（如果Volume要共享的话），其他容器通过加入的方式共享这些Namespace

​	如果对Pod中的容器启动有顺序要求，可以使用Init Contianer。所有Init Container定义的容器，都会比spec.containers定义的用户容器按顺序优先启动。Init Container容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

Pod使用过程中的重要字段： 

* pod自定义/etc/hosts:  spec.hostAliases 

* pod共享PID : spec.shareProcessNamespace 

* 容器启动后/销毁前的钩子： spec.container.lifecycle.postStart/preStop

* pod的状态：spec.status.phase 

* pod特殊的volume（投射数据卷）

  *  密码信息获取

    创建Secrete对象保存加密数据，存放到Etcd中。然后，你就可以通过在Pod的容器里挂载Volume的方式，访问到这些Secret里保存的信息 

  * 配置信息获取

    创建ConfigMap对象保存加密数据，存放到Etcd中。然后，通过挂载Volume的方式，访问到ConfigMap里保存的内容

  * 容器获取Pod中定义的静态信息

    通过挂载DownwardAPI 这个特殊的Volume，访问到Pod中定义的静态信息

  * Pod中要访问K8S的API

    任何运行在Kubernetes集群上的应用，都必须使用这个ServiceAccountToken里保存的授权信息，也就是Token，才可以合法地访问API Server。因此，通过挂载Volume的方式，把对应权限的ServiceAccountToken这个特殊的Secrete挂载到Pod中即可 

* 容器是否健康

  spec.container.livenessProbe。若不健康，则Pod有可能被重启（可配置策略）

* 容器是否可用

  spec.container.readinessProbe。若不健康，则service不会访问到该Pod

## Deployment

Kubernetes 里第一个控制器模式的完整实现即Deployment.

**它实现了 Kubernetes Pod 的水平扩展**。该功能依赖于一个非常重要的概念（API 对象）ReplicaSet对象

滚动更新： 你的游戏角色装备了一套一级铭文，现在有一套二级铭文可以替换。一个个替换，一次替换一个铭文，这就是滚动更新。

## 4. StatefulSet

StatefulSet 其实是一种特殊的 Deployment，只不过这个“Deployment”的每个 Pod 实例的名字里，都携带了一个唯一并且固定的编号。这个编号的顺序，固定了 Pod 的拓扑关系；这个编号对应的 DNS 记录，固定了 Pod 的访问方式；这个编号对应的 PV，绑定了 Pod 与持久化存储的关系。所以，当 Pod 被删除重建时，这些“状态”都会保持不变

pod的状态

Pod 状态是 Ready，实际上不能提供服务的可能原因

* 程序本身有 bug，本来应该返回 200，但因为代码问题，返回的是500
* 程序因为内存问题，已经僵死，但进程还在，但无响应
* Dockerfile 写的不规范，应用程序不是主进程，那么主进程出了什么问题都无法发现
* 程序出现死循环

### 4.1. 拓扑状态

### 4.2. 存储状态

### 4.3. mysql主从案例

PS：一个容器里显然没办法管理两个常驻进程，因为容器是单进程的意思，所以数据复制操作要用sidecar容器来处理，而不用mysql主容器来一并解决。

将数据复制操作和启动mysql服务一并写到同一个sh -c后面，这样算两个进程。

StatefulSet 已经不能解决它的部署问题就可以使用Operator了

## 5. DaemonSet

Kubernetes 集群里的每一个节点会运行一个 Daemon Pod

* 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
* 每个节点上只有一个这样的 Pod 实例；
* 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；
* 而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

### 5.1. 如何实现的

* 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
* 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；
* 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集

### 5.2. 版本控制

DaemonSet 使用 ControllerRevision，来保存和管理自己对应的版本。

## 6. Job与CronJob

### 6.1. Job

重点是这两个参数parallelism 比 completions

### 6.2. CronJob

CronJob 描述的，正是定时任务

## 自定义API对象

### 服务网格案例

通过预设sidecar容器配置，应用发布pod时，自动读取sidecar配置生成容器，完成服务网格的初始化

Kuberentes的API对象由三部分组成，通常可以归结为： /apis/group/version/resource，例如     apiVersion: Group/Version    kind: Resource APIServer在接收一个请求之后，会按照 /apis/group/version/resource的路径找到对应的Resource类型定义，根据这个类型定义和用户提交的yaml里的字段创建出一个对应的Resource对象 CRD机制： 

（1）声明一个CRD，让k8s能够认识和处理所有声明了API是"/apis/group/version/resource"的YAML文件了。包括：组（Group）、版本（Version）、资源类型（Resource）等。 

（2）编写go代码，让k8s能够识别yaml对象的描述。包括：Spec、Status等 

（3）使用k8s代码生成工具自动生成clientset、informer和lister 

（4） 编写一个自定义控制器，来对所关心对象进行相关操作  

（1）（2）步骤之后，就可以顺利通过kubectl apply xxx.yaml 来创建出对应的CRD和CR了。 但实际上，k8s的etcd中有了这样的对象，却不会进行实际的一些后续操作，因为我们还没有编写对应CRD的控制器。控制器需要：感知所关心对象过的变化，这是通过一个Informer来完成的。而Informer所需要的代码，正是上述（3）步骤生成

### 自定义控制器

## 7. 权限

角色的访问控制（RBAC）

角色（Role），其实就是一组权限规则列表。

而我们分配这些权限的方式，就是通过创建 RoleBinding 对象，将被作用者（subject）和权限列表进行绑定

与之对应的 ClusterRole 和 ClusterRoleBinding，则是 Kubernetes 集群级别的 Role 和 RoleBinding，它们的作用范围不受 Namespace 限制

权限的被作用者可以有很多种（比如，User、Group 等），但在我们平常的使用中，最普遍的用法还是 ServiceAccount。所以，Role + RoleBinding + ServiceAccount 的权限分配方式是你要重点掌握的内容。我们在后面编写和安装各种插件的时候，会经常用到这个组合

## 8. Operator

​	在 Kubernetes 中，管理“有状态应用”是一个比较复杂的过程。在 Kubernetes 生态中，还有一个相对更加灵活和编程友好的管理“有状态应用”的解决方案，它就是：Operator。

​	Operator 的工作原理，实际上是利用了 Kubernetes 的自定义 API 资源（CRD），来描述我们想要部署的“有状态应用”；

​	然后在自定义控制器里，根据自定义 API 对象的变化，来完成具体的部署和运维工作



## 9. 总结

声明式api还是有一些本质不足的地方，主要是 

1.通常operator都是用来描述复杂有状态集群的，这个集群本身就已经很复杂了 

2.声明式api通过diff的方式来得出集群要做怎么样的状态变迁，这个过程中，常常会有很多状况不能覆盖到，而如果使用者对此没有深刻的认识，就会越用越糊涂 

大致来说，如果一个集群本身的状态有n种，那么operator理论上就要覆盖n*(n-1)种变迁的可能性，而这个体力活几乎是不可能完成的，所有很多operator经常性的不支持某些操作，而不是像使用者想象的那样，我改改yaml，apply一下就完事了 

更重要的是，由于情况太多太复杂，甚至无法通过文档的方式，告诉使用者，状态1的集群，按某种方式修改了yaml之后，是不能变成使用者期待的状态2的 

如果是传统的命令式方式，那么就是所有可能有的功能，都会罗列出来，可能n个状态，只有2n+3个操作是合法的，剩下都是做不到的，这样使用者固然受到了限制，但这种限制也让很多时候的操作变得更加清晰明了,而不是每次改yaml还要思考一下，这么改，operator支持吗？ 

当然，声明式api的好处也是明显的，如果这个diff是operator覆盖到的，那么就能极大的减轻使用者的负担，apply下就解脱了 

而这个集群状态变迁的问题是本质复杂的，必然没有可以消除复杂度的解法，无非就是这个复杂度在operator的实现者那里，还是在运维者那里，在operator的实现者那里，好处就是固化的常用的变迁路径，使用起来很方便，但如果operator的开发者没有实现某个状态的变迁路径，而这个本身是可以做到的，这个时候，就比不上命令式灵活，个人感觉就是取舍了