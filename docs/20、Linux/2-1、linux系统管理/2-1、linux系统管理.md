## 说明

管理Linux主要是要求我们如何设置我们的linux

## 网络管理

### 网络状态查看

有两套工具包

net-tools和iproute（centos7以后主推的版本）
* net-tools
  ifconfig
  route
  netstat

* iproute2
  ip
  ss

#### 网卡信息

网络接口配置

直接输入ifconfig就是该命令了

```
ifconfig
```

显示含义：

```
eth0 第⼀块⽹网卡（⽹网络接⼝）
```

你的第⼀个⽹网络接⼝口可能叫做下⾯的名字

* eno1 板载网卡
* ens33 PCI-E网卡
* enp0s3 无法获取物理理信息的 PCI-E ⽹网卡
* CentOS 7 使⽤用了了⼀致性⽹网络设备命名，以上都不不匹配则使⽤用 eth0

PS:

* 网卡名称固定之后，方便编写多主机批量控制脚本
* 板载网卡有可能不被linux识别，因为非服务器主板驱动程序和linux的兼容性并不好

**修改网卡名称**

⽹网卡命名规则受 biosdevname 和 net.ifnames 两个参数影响

|       | biosdevname | net.ifnames | 网卡名 |
| ----- | ----------- | ----------- | ------ |
| 默认  | 0           | 1           | ens33  |
| 组合1 | 1           | 0           | em1    |
| 组合2 | 0           | 0           | eth0   |

* 编辑 /etc/default/grub ⽂文件，增加 biosdevname=0 net.ifnames=0   

  编辑系统内核引导工具，修改启动参数

```
vim /etc/default/grub
```

在grub_cmdline_linux的值后面增加如下的参数

```
biosdevname=0 net.ifnames=0
```

* 执行修改启动参数命令

```
grub2-mkconfig -o /boot/grub2/grub.cfg
```

* 重启

  ```
  reboot
  ```

#### 网卡物理连接

这里查看eth0网卡的连接

```
mii-tool eth0
```

虚拟机可能不支持

#### 网络网关路由

如果需要连接其它的网络设备，查看下一个网络网哪里走，则需要看路由/网关

```
route -n
```

-n不解析域名，可以使用route --help来查看.如果要解析成域名，会很慢

![](20200601091136350.png) 

* 第一行的意思就是去往所有目标地址数据包由网关10.12.236.1 通过网卡ens33来转发
* 0.0.0.0代表的是匹配所有目标地址
* 实际上前两条是自动生成的，因为是直连的网段，在每块网卡上每配置一个IP就会生成一条记录（一个网卡上可以配置多个IP）。表示去往这两个网段的数据包，直接由网卡接口ens33发送出去

目的地址， 网关位置

default 

### 网络配置

有两套命令

#### 网卡配置

* 修改ip地址（可以同时修改掩码）

  ```
  ifconfig <接⼝口> <IP地址> [netmask ⼦子⽹网掩码 ]
  ```

* 启动网卡

  ```
  ifup <接⼝口>
  ```

  还原默认配置后启动

* 关掉网卡

  还原默认配置

  ```
  ifdown <接⼝口>
  ```

#### 网关配置

修改默认网关

需要先删除，再添加

* 删除默认网关

  ```
  route del default gw <⽹网关ip>
  ```

* 添加默认网关

  ```
  route add default gw <⽹网关ip>
  ```

如果需要修改明细路由（自定义目的的网关），需要按照如下配置

* 添加指定目的的网关

  ```
  route add -host <指定ip> gw <⽹网关ip>
  ```

* 添加指定网段的网关和掩码

  ```
  route add -net <指定⽹网段> netmask <⼦子⽹网掩码> gw <⽹网关ip>
  ```

如果需要永久生效需要写入/etc/rc.local

#### ip命令

刚刚说了有两套，这里还有一套命令

* ip addr ls
  * ifconfig
* ip link set dev eth0 up
  * ifup eth0
* ip addr add 10.0.0.1/24 dev eth1
  * ifconfig eth1 10.0.0.1 netmask 255.255.255.0
* ip route add 10.0.0/24 via 192.168.0.1
  * route add -net 10.0.0.0 netmask 255.255.255.0 gw 192.168.0.1

### 网络故障排除

* ping

  看网络是否通

* traceroute

  追踪路由，追踪网络中的每一跳，会显示经历过多少个路由过来。可以

  traceroute -w 1 表示等待1秒

* mtr

  检查网络中间是否数据包丢失了	

* nslookup

  查看域名对应的ip是什么，是由那个服务器解析的dns

* telnet

  主机能通，端口不一定通

* tcpdump

  抓包

  tcpdump-i any -n port 80 and host 10.xx.xx.xx -w /cxc/tcpfile

  所有的网卡，发往80端口和某个ip的数据包保存到tcpfile文件之中。-n表示以ip的形式显示

* netstat

  监听提供的服务服务状态

  netstat -ntpl

  n表示显示ip，t表示监听tcp的，p表示显示进程，l表示Listen的状态

  注意监听地址，如果是127.0.0.0表示只能对本机进行服务

  如果对所有的机器提供的话，需要改称0.0.0.0

* ss

  监听提供的服务允许服务的ip范围

### 网络配置文件

前面都是一些临时的网络配置信息。比如重启后，可能数据会重置了。如何将这些配置固化下来呢。需要使用网络配置管理的命令。

#### 管理程序

网络服务管理理程序分为两种，分别为SysV和systemd

* service network start|stop|restart | status

  status表示网络状态，restart会重启网络服务，之前的配置会失效。

* chkconfig -list network

  查看配置的网络

* systemctl list-unit-files NetworkManager.service

* systemctl start|stop|restart NetworkManger

* systemctl enable|disable NetworkManger

**PS：NetworkManage服务，network服务是centos6的网络默认管理工具， centos7重写了一遍就是NetworkManage服务，因为network只能支持service来管理， 而centos7默认的服务管理工具换成了systemctl，就有了大家要学两套网络管理工具的麻烦的事情 ifconfig 和ip 是同样的情况，他们都可以查询网络状态，都可以设置ip，但是设置了之后只能保存在内存中，重启之后配置就没有了。要想重启之后还需要保持配置需要写入配置文件， 通过service network restart 重新加载配置文件来让网络配置生效**

#### 配置文件

网络配置⽂文件主要有：

* /etc/sysconfig/network-scripts/ifcfg-eth0

  这个和网卡的名称对应

  可以设置网卡的ip获取方式，**静态ip还是动态ip**，编辑完成后需要，重新加载配置文件来生效

  service network restart或者systemctl  restart NetworkManager.service

* hostname xxx

  修改主机名称，这个是临时设置

* hostnamectl set-host xxxx

  修改主机名称，这个是永久设置

* /etc/hosts

  hosts相关，需要注意，很多进程依赖于主机名，所有主机名称修改后，需要将新的主机名和127.0.0.1对应起来。

## 软件包管理

包管理理器器是⽅方便便软件安装、卸载，解决软件依赖关系的重要⼯工具

* CentOS、RedHat 使⽤用 yum 包管理理器器，软件安装包格式为 rpm
* Debian、Ubuntu 使⽤用 apt 包管理理器器，软件安装包格式为 deb

### rpm命令

### yum包管理器

### 源代码编译

### 内核升级

### grub配置



## 内核管理

## 进程管理

## 内存管理

## 磁盘管理