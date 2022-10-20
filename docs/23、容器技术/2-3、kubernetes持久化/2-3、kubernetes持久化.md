## 1. 开头

用户提交请求创建pod，Kubernetes发现这个pod声明使用了PVC，那就靠PersistentVolumeController帮它找一个PV配对。

 没有现成的PV，就去找对应的StorageClass，帮它新创建一个PV，然后和PVC完成绑定。

 新创建的PV，还只是一个API 对象，需要经过“两阶段处理”变成宿主机上的“持久化 Volume”才真正有用：

 第一阶段由运行在master上的AttachDetachController负责，为这个PV完成 Attach 操作，为宿主机挂载远程磁盘； 

第二阶段是运行在每个节点上kubelet组件的内部，把第一步attach的远程磁盘 mount 到宿主机目录。

这个控制循环叫VolumeManagerReconciler，运行在独立的Goroutine，不会阻塞kubelet主循环。 

完成这两步，PV对应的“持久化 Volume”就准备好了，POD可以正常启动，将“持久化 Volume”挂载在容器内指定的路径

## 2. 本地持久化案例

PS：当一个pvc创建之后，kubernetes因为dynamic provisioning机制会调用pvc指定的storageclass里的provisioner自动使用local disk的信息去创建pv。而且pv一旦创建，nodeaffinity参数就指定了固定的node。而此时，provisioner并没有pod调度的相关信息。 延迟绑定发生的时机是pod已经进入调度器。此时对应的pv已经创建，绑定了node。并可能与pod的调度信息发生冲突。 如果dynamic provisioning机制能够推迟到pod 调度的阶段，同时考虑pod调度条件和node硬件信息，这样才能实现dynamic provisioning。实现上可以参考延迟绑定，来一个延迟 provision。另外实现一个controller在pod调度阶段创建pv。总结下来就是dynamic provision机制不知道pod需要在哪个node下运行，而提前就创建好了。因此Local Persistent Volume 目前还不能支持 Dynamic Provisioning

## 3. 自定义持久化插件

默认情况下，Kubernetes 里通过存储插件管理容器持久化存储。

k8s一般通过FlexVolume 和 CSI 这两种自定义存储插件。

相比于 FlexVolume，CSI 的设计思想，把插件的职责从“两阶段处理”，扩展成了 Provision、Attach 和 Mount 三个阶段

### 3.2. CSI原理

这里是基于阿里云提供的云盘。容器持久化存储CSI 插件操作过程：

* Register过程

  csi 插件应该作为 daemonSet 部署到每个节点（node）。

  然后插件 container 挂载 hostpath 文件夹，把插件可执行文件放在其中，并启动rpc服务（identity, controller, node）。

  External component Driver Registrar 利用 kubelet plugin watcher 特性watch指定的文件夹路径来自动检测到这个存储插件。

  然后通过调用identity rpc服务，获得driver的信息，并完成注册。  

* Provision过程

  部署External Provisioner。 

  Provisioner 将会 watch apiServer 中 PVC 资源的创建，并且PVC 所指定的 storageClass 的 provisioner是我们上面启动的插件。

  那么，External Provisioner 将会调用 插件的 controller.createVolume() 服务。

  其主要工作应该是通过阿里云的api 创建网络磁盘，并根据磁盘的信息创建相应的pv。 

* Attach过程

  部署External Attacher。

  Attacher 将会监听 apiServer 中 VolumeAttachment 对象的变化

  一旦出现新的VolumeAttachment，Attacher 会调用插件的 controller.ControllerPublish() 服务。

  其主要工作是调用阿里云的api，把相应的磁盘 attach 到声明使用此 PVC/PV 的 pod 所调度到的 node 上。挂载的目录：/var/lib/kubelet/pods/<Pod ID>/volumes/aliyun~netdisk/<name>  

* Mount过程

  mount 不可能在远程的container里完成，所以这个工作需要kubelet来做。

  kubelet 的 VolumeManagerReconciler 控制循环，检测到需要执行 Mount 操作的时候，通过调用 pkg/volume/csi 包，调用 CSI Node 服务，完成 volume 的 Mount 阶段

  具体工作是调用 CRI 启动带有 volume 参数的container，把上阶段准备好的磁盘 mount 到 container指定的目录。

### 3.3. CSI实践

对于一个部署了 CSI 存储插件的 Kubernetes 集群来说：当用户创建了一个 PVC 之后，你前面部署的 StatefulSet 里的 External Provisioner 容器，就会监听到这个 PVC 的诞生，然后调用同一个 Pod 里的 CSI 插件的 CSI Controller 服务的 CreateVolume 方法，为你创建出对应的 PV。这时候，运行在 Kubernetes Master 节点上的 Volume Controller，就会通过 PersistentVolumeController 控制循环，发现这对新创建出来的 PV 和 PVC，并且看到它们声明的是同一个 StorageClass。所以，它会把这一对 PV 和 PVC 绑定起来，使 PVC 进入 Bound 状态。然后，用户创建了一个声明使用上述 PVC 的 Pod，并且这个 Pod 被调度器调度到了宿主机 A 上。这时候，Volume Controller 的 AttachDetachController 控制循环就会发现，上述 PVC 对应的 Volume，需要被 Attach 到宿主机 A 上。所以，AttachDetachController 会创建一个 VolumeAttachment 对象，这个对象携带了宿主机 A 和待处理的 Volume 的名字。这样，StatefulSet 里的 External Attacher 容器，就会监听到这个 VolumeAttachment 对象的诞生。于是，它就会使用这个对象里的宿主机和 Volume 名字，调用同一个 Pod 里的 CSI 插件的 CSI Controller 服务的 ControllerPublishVolume 方法，完成“Attach 阶段”。上述过程完成后，运行在宿主机 A 上的 kubelet，就会通过 VolumeManagerReconciler 控制循环，发现当前宿主机上有一个 Volume 对应的存储设备（比如磁盘）已经被 Attach 到了某个设备目录下。于是 kubelet 就会调用同一台宿主机上的 CSI 插件的 CSI Node 服务的 NodeStageVolume 和 NodePublishVolume 方法，完成这个 Volume 的“Mount 阶段”

## 4. 总结