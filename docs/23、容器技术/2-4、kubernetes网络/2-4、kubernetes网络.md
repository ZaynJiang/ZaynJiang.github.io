## 1. 开头



## 2. 单机网络

单机容器网络的实现原理通过docker0 网桥

容器要想跟外界进行通信，它发出的 IP 包就必须从它的 Network Namespace 里出来，来到宿主机上。

解决这个问题的方法就是：为容器创建一个一端在容器里充当默认网卡、另一端在宿主机上的 Veth Pair 设备

## 3. 跨主机网络

Flannel 项目是 CoreOS 公司主推的容器网络方案。事实上，Flannel 项目本身只是一个框架，真正为我们提供容器网络功能的，是 Flannel 的后端实现。目前，Flannel 支持三种后端实现

* UDP

  最直接、最容易理解、性能最差的实现方式 。 比如Flannel的实现中，flanneld将不同宿主机划分成不同的子网，不同的容器所在的宿主机不同则子网也不同，这些<宿主机,子网>信息存储在etcd中。当容器需要“跨主通信”时， flanneld进程会将发出端的IP包进行UDP封装，然后发送到目的端的宿主机上，目的端的宿主机的flanneld再进行解封装发送给对应的容器来使用

* VXLAN

  Virtual Extensible LAN（虚拟可扩展局域网），通过VXLAN技术在内核态上实现了UDP方式的封装和解封装，提高了性能，即用VTEP实现了flanneld进程的功能

* host-gw



### 3.1. UDP

### 3.2. VXLAN

PS：这里使用UDP，不需要可靠，因为可靠性不是这一层需要考虑的事情，这一层需要考虑的事情就是把这个包发送到目标地址。如果出现了UDP丢包的情况，也完全没有关系，这里UDP包裹的是我们真正的数据包，如果我们上层使用的是TCP协议，而Flannel使用UDP包裹了我们的数据包，当出现丢包了，TCP的确认机制会发现这个丢包而重传，从而保证可靠



这个和一般的网络传输不同，一般的网络传输会直接请求ip地址，根据ip地址，发送ARP请求来获取下一跳设备的mac地址，获取mac地址之后会缓存下来，这是一般路由器或者网桥做的事情。但在容器网络里我们只知道目标容器的ip地址，并不知道宿主机的ip地址，需要将容器的ip地址和宿主机的ip地址关联起来。这个关系由路由规则, fdb存储的数据和ARP记录下来，这些都是在创建node节点的时候，由各个节点的flanneld创建出来。通过路由规则知道了该将数据包发送到flannel.1这个vtep设备处理并且知道了目地vtep的ip地址，通过vtep IP地址ip neigh show dev flannel.1 找到了对端的vtep mac地址，通过对端vtep mac地址查询 bridge fdb list 查到了这个mac地址对应的宿主机地址。 这样封装出包之后按照vxlan逻辑进行正常发送

各个node会将自己的vtep信息上报给apiserver，apiserver将信息同步给各个节点的flanneld，flanneld通过netlink下发，完成信息的同步

### 3.3. CNI

所有容器都可以直接使用 IP 地址与其他容器通信，而无需使用 NAT。所有宿主机都可以直接使用 IP 地址与所有容器通信，而无需使用 NAT。反之亦然。容器自己“看到”的自己的 IP 地址，和别人（宿主机或者容器）看到的地址是完全一样的。

网络模型，其实可以用一个字总结，那就是“通”。容器与容器之间要“通”，容器与宿主机之间也要“通”。并且，Kubernetes 要求这个“通”，还必须是直接基于容器和宿主机的 IP 地址来进行的

总结下来：

CNI插件作用：即配置Infra容器的网络栈，并连到网桥上

整体流程是：kubelet创建Pod->创建Infra容器->调用SetUpPod（）方法，该方法需要为CNI准备参数，然后调用CNI插件（flannel)为Infra配置网络；其中参数来源于1、dockershim设置的一组CNI环境变量；2、dockershim从CNI配置文件里（有flanneld启动后生成，类型为configmap）加载到的、默认插件的配置信息（network configuration)，这里对CNI插件的调用，实际是network configuration进行补充。参数准备好后，调用Flannel CNI->调用CNI bridge（所需参数即为上面：设置的CNI环境变量和补充的network configuation）来执行具体的操作流程

### 3.4. 三层网络

三层和隧道的异同： 相同之处是都实现了跨主机容器的三层互通，而且都是通过对目的 MAC 地址的操作来实现的；不同之处是三层通过配置下一条主机的路由规则来实现互通，隧道则是通过通过在 IP 包外再封装一层 MAC 包头来实现。 三层的优点：少了封包和解包的过程，性能肯定是更高的。 三层的缺点：需要自己想办法维护路由规则。 隧道的优点：简单，原因是大部分工作都是由 Linux 内核的模块实现了，应用层面工作量较少。 隧道的缺点：主要的问题就是性能低

## 4. 网络隔离

Kubernetes 对 Pod 进行“隔离”的手段，即：NetworkPolicy。

NetworkPolicy 实际上只是宿主机上的一系列 iptables 规则。这跟传统 IaaS 里面的安全组（Security Group）其实是非常类似的。

Kubernetes 的网络模型以及大多数容器网络实现，其实既不会保证容器之间二层网络的互通，也不会实现容器之间的二层网络隔离。这跟 IaaS 项目管理虚拟机的方式，是完全不同的。

Kubernetes 从底层的设计和实现上，更倾向于假设你已经有了一套完整的物理基础设施。然后，Kubernetes 负责在此基础上提供一种“弱多租户”

## 5. 负载均衡

Kubernets中通过Service来实现Pod实例之间的负载均衡和固定VIP的场景。 

Service的工作原理是通过kube-proxy来设置宿主机上的iptables规则来实现的。

kube-proxy来观察service的创建，然后通过修改本机的iptables规则，将访问Service VIP的请求转发到真正的Pod上。 

基于iptables规则的Service实现，导致当宿主机上有大量的Pod的时候，成百上千条iptables规则不断刷新占用大量的CPU资源。

因此出现了一种新的模式: IPVS，通过Linux的 IPVS模块将大量iptables规则放到了内核态，降低了维护这些规则的代价（部分辅助性的规则无法放到内核态，依旧是iptable形式）。

 Service的DNS记录： <myservice>.<mynamespace>.svc.cluster.local ，当访问这条记录，返回的是Service的VIP或者是所代理Pod的IP地址的集合 Pod的DNS记录： <pod_hostname>.<subdomain>.<mynamespace>.svc.cluster.local， 注意pod的hostname和subdomain都是在Pod里定义的。 Service的访问依赖宿主机的kube-proxy生成的iptables规则以及kube-dns生成的DNS记录。外部的宿主机没有kube-proxy和kube-dns应该如何访问对应的Service呢？

有以下几种方式： NodePort： 比如外部client访问任意一台宿主机的8080端口，就是访问Service所代理Pod的80端口。由接收外部请求的宿主机做转发。

 即：client --> nodeIP:nodePort --> serviceVIP:port --> podIp:targetPort LoadBalance：

由公有云提供kubernetes服务自带的loadbalancer做负载均衡和外部流量访问的入口 ExternalName：通过ExternalName或ExternalIp给Service挂在一个公有IP的地址或者域名，当访问这个公有IP地址时，就会转发到Service所代理的Pod服务上 Ingress是用来给不同Service做负载均衡服务的，也就是Service的Service

### 5.1. 服务发现

### 5.2. 反向代理

集群外访问到服务，要经过3次代理： 访问请求到达任一宿主机，会根据NodePort生成的iptables规则，跳转到nginx反向代理， 请求再按照nginx的配置跳转到后台service，nginx的配置是根据Ingress对象生成的， 后台service也是iptables规则，最后跳转到真正提供服务的POD

## 6. 总结

