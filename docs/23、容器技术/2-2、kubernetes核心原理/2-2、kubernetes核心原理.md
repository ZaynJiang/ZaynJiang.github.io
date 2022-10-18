

## 1.

pod是k8s操作的最小单元，操作pod的工作都是由控制器controller完成的，对应k8s中的组件就是kube-controller-manager，其下常见的控制器类型有：

- Deployment，无状态容器控制器，通过控制ReplicaSet来控制pod；应用场景web应用；
- StatefulSet，有状态容器控制器，保证pod的有序和唯一，pod的网络标识和存储在pod重建前后一致；应用场景主从架构；
- DeamonSet，守护容器控制器，确保所有节点上有且仅有一个pod；应用场景监控；
- Job，普通任务容器控制器，只会执行一次；应用场景离线数据处理；
- CronJob，定时任务容器控制器，定时执行；应用场景通知、备份；

控制器的yaml文件一般分为两个部分，上半部分是关于控制器的定义，下半部分是关于被控制对象，也就是pod的定义：

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



## 2. pod

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

## 3. API对象

### 3.1. 声明式API概念

### 3.2. 服务网格案例

通过预设sidecar容器配置，应用发布pod时，自动读取sidecar配置生成容器，完成服务网格的初始化

### 3.3. 自定义API对象

Kuberentes的API对象由三部分组成，通常可以归结为： /apis/group/version/resource，例如     apiVersion: Group/Version    kind: Resource APIServer在接收一个请求之后，会按照 /apis/group/version/resource的路径找到对应的Resource类型定义，根据这个类型定义和用户提交的yaml里的字段创建出一个对应的Resource对象 CRD机制： （1）声明一个CRD，让k8s能够认识和处理所有声明了API是"/apis/group/version/resource"的YAML文件了。包括：组（Group）、版本（Version）、资源类型（Resource）等。 （2）编写go代码，让k8s能够识别yaml对象的描述。包括：Spec、Status等 （3）使用k8s代码生成工具自动生成clientset、informer和lister （4） 编写一个自定义控制器，来对所关心对象进行相关操作  （1）（2）步骤之后，就可以顺利通过kubectl apply xxx.yaml 来创建出对应的CRD和CR了。 但实际上，k8s的etcd中有了这样的对象，却不会进行实际的一些后续操作，因为我们还没有编写对应CRD的控制器。控制器需要：感知所关心对象过的变化，这是通过一个Informer来完成的。而Informer所需要的代码，正是上述（3）步骤生成

### 3.4. 自定义控制器

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

### 4.4.  小结

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

## 7. 权限

角色的访问控制（RBAC）

角色（Role），其实就是一组权限规则列表。

而我们分配这些权限的方式，就是通过创建 RoleBinding 对象，将被作用者（subject）和权限列表进行绑定

与之对应的 ClusterRole 和 ClusterRoleBinding，则是 Kubernetes 集群级别的 Role 和 RoleBinding，它们的作用范围不受 Namespace 限制

权限的被作用者可以有很多种（比如，User、Group 等），但在我们平常的使用中，最普遍的用法还是 ServiceAccount。所以，Role + RoleBinding + ServiceAccount 的权限分配方式是你要重点掌握的内容。我们在后面编写和安装各种插件的时候，会经常用到这个组合

## 8. Operator

