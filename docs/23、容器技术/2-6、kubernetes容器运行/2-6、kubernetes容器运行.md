## 1. 开头



## 2. kubelet

kube-apiserver和kubelet是k8s中最核心的两个组件.

其中kubelet是属于SIG-Node的范畴

kubelet的工作核心是一个叫做SyncLoop的控制循环。驱动这个控制循环运行的事件包括四种: 

* Pod更新事件
* Pod生命周期变化
* kubelet本身设置的执行周期 
* 定时的清理事件 

kubelet启动时要做的第一件事就是设置Listeners，注册它所关心的的Informer.这些Informer就是SyncLoop需要处理的数据的来源。kubelet还负责很多子控制循环，比如Volume Manager, Image Manager, Node status Manager.kubelet还会通过Watch机制来监听关心的Pod对象的变化。 CRI是负责kubeblet和底层容器打交道的统一接口

## 3. CRI

调度器将Pod调度到某一个Node上后，Node上的Kubelet就需要负责将这个Pod拉起来。在Kuberentes社区中，Kubelet以及CRI相关的内容，都属于SIG-Node。 

Kubelet也是通过控制循环来完成各种工作的。kubelet调用下层容器运行时通过一组叫作CRI的gRPC接口来间接执行的。通过CRI， kubelet与具体的容器运行时解耦。在目前的kubelet实现里，内置了dockershim这个CRI的实现，但这部分实现未来肯定会被kubelet移除。未来更普遍的方案是在宿主机上安装负责响应的CRI组件（CRI shim），kubelet负责调用CRI shim，CRI shim把具体的请求“翻译”成对后端容器项目的请求或者操作 。 

不同的CRI shim有不同的容器实现方式，例如：创建了一个名叫foo的、包括了A、B两个容器的Pod 

Docker: 创建出一个名叫foo的Infra容器来hold住整个pod，接着启动A，B两个Docker容器。所以最后，宿主机上会出现三个Docker容器组成这一个Pod 

Kata container: 创建出一个轻量级的虚拟机来hold住整个pod，接着创建A、B容器对应的 Mount Namespace。所以，最后在宿主机上，只会有一个叫做foo的轻量级虚拟机在运行

## 4. 安全

Kata Container本质就是一个精简后的轻量级虚拟机，所以它的特点，就是“像虚拟机一样安全，像容器一样敏捷”。2018年，Google发布了gVisor，给容器进程配置了一个用Go语言实现的“极小独立内核”。 

无论是Kata Container还是gVisor，实现安全的方式本质上都是让容器独立一个操作系统内核，避免共享宿主机内核。他们的主要区别是，Kata 通过硬件虚拟化出一台“小虚拟机”，在“小虚拟机”安装裁剪后的Linux内核，而gVisor则是“模拟”出了一个运行在用户态的操作系统内核，通过这个内核来限制容器进程向宿主机发起的调

gVisor虽然现在没有任何优势，但是这种通过在用户态运行一个操作系统内核，来为应用进程提供强隔离的思路，的确是未来安全容器进一步演化的一个非常有前途的方向