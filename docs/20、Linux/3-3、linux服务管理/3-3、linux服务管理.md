## 防火墙服务

可分为：

* 硬件防火墙:

  主要是防御DDOS攻击, 流量攻击. 兼顾数据包过滤.

* 软件防火墙

  主要处理数据包的过滤.
  * iptables
  * firewalld

按照过滤层级又可分为

* 包过滤防火墙

* 应用层防火墙

### iptables

CentOS 6 默认防火墙, 主要是包过滤防火墙, 对 ip, 端口, tcp, udp, icmp 协议控制.属于包过滤防火墙

其中核心为规则表，规则表有：

- **filter** 过滤

  ip, 端口, tcp,udp,icmp协议

- **nat** 网络地址转换

- mangle

- raw

其中filter 和 nat，最常用。不同的表有自己的默认规则

例如. `Chain INPUT (policy ACCEPT)` 表示若未匹配任意规则则默认接受。默认 DROP, 配置特定规则 ACCEPT，常见于生产环境

#### 规则链

- **INPUT** (C->S 方向)规则链
- **OUTPUT** (S->C 方向)规则链
- **FORWARD** 转发规则链

- **PREROUTING** 路由前转换(改变目标地址)规则链
- **POSTROUTING** 路由后转换(控制源地址)规则链

#### filter

filter [-t filter] <命令> <规则链> <规则>

默认操作的是 filter 表, 因此一般书写时可以省略.

```
                当前的默认策略是ACCEPT
                    👇
Chain INPUT (policy ACCEPT 501 packets, 33820 bytes)
 pkts bytes target     prot            opt in     out     source               destination
    0     0 ACCEPT     all            --  *      *       10.0.0.1             0.0.0.0/0
  
                👆       👆                👆        👆            👆                    👆
               策略         协议                进入网卡 出去网卡    来源地址              目的地址
                    (tcp,udp,icmp)
    
```

命令示例：

-  iptables -vnL 

  查看filter表详细状态, 可方便查看当前的所有规则

- iptables -t filter -A INPUT -s 10.0.0.1 -j ACCEPT 

  允许来自 10.0.0.1 的包进入
  iptables -t filter -A INPUT -s 10.0.0.2 -j DROP    # 允许来自 10.0.0.1 的包进入

- iptables -D INPUT -s 10.0.0.2 -j DROP

  删除匹配的规则

- iptables -t filter -D INPUT 3

  删除第3条规则(第1条的序号是1)

- iptables -P INPUT DROP

  修改INPUT规则链的默认规则为 DROP

- iptables -t filter -A INPUT -i eth0 -s 10.0.0.2 -p tcp --dport 80 -j ACCEPT

  允许从 eth0 网卡进入, 来自10.0.0.2的包访问本地的80端口.

#### nat

 网络地址转换表，可用于: 端口转发 

iptables -t nat <命令> <规则链> <规则>

规则链
    PREROUTING        

​	路由前转换 - 目的地址转换
​    POSTROUTING  

​     路由后转换 - 控制源地址

动作(target) -j
    DNAT

规则

* --to-destination <ip>[/<mask>]    

​	修改目的地址

* --to-source <ip>[/<mask>]        

​     修改源地址

示例

* iptables -t nat -vnL 

  查看nat表详细状态
  
* iptables -t nat -A PREROUTING -i eth0 -d 114.115.116.117 -p tcp --dport 80 -j DNAT --to-destination 10.0.0.1

  假设 iptables 所在主机ip为 114.115.116.117, 此处配置外网访问该地址(eth0网卡)时将访问tcp协议且是80端口的数据包目的地址转换内网主机

* iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth1 -j SNAT --to-source 114.115.116.117

  另内网访问该地址(eth1网卡)时将数据包源地址转换成iptables所在主机ip 114.115.116.117

#### 配置文件

iptables 的命令是写在内存中, 重启后失效。

iptables 配置文件 `/etc/sysconfig/iptables`

* service iptables save|start|stop|restart

  注意save命令的作用：

  * 将当前内存中的配置保存为配置文件 /etc/sysconfig/iptables
  * 实际执行了 iptables-save > /etc/sysconfig/iptables
  * 下次启动时 iptables-restore < /etc/sysconfig/iptables

需要安装该服务 `yum install iptables-services` 才可以使用该命令

### firewalld

CentOS 7 在 iptables 上又封装了一层, 也就是 firewalld , 使用更简洁(底层使用 netfilter)。属于包过滤防火墙。

* 支持区域 "zone" 概念

  即 iptables 的自建规则链

* firewall-cmd

  控制防火墙

#### 服务管理

firewalld 与 iptables 服务是冲突的.

* systemctl start|stop|enable|disable firewalld.serivce

#### firewall-cmd

命令注释：

```bash
firewall-cmd [选项]

选项
    状态选项
    --state                # 查看firewalld当前状态
    --reload            # 重新加载配置
    
    --zone=public        # 指定区域, 不指定时默认就是 public
        
    Zone选项
    --get-zones            # 查看所有的区(除了public外还有一些iptables默认规则链一样)
    --get-default-zone    # 查看默认的区
    --get-active-zone    # 查看激活的zone    
    
    永久性选项
    --permanent            # 设置选项写入配置, 需要 reload 才能生效 
    
    修改和查询zone的选项
    --list-all            # 查看详细状态(包含以下内容)     
    --list-interfaces    # 查看某一项状态 - 网卡接口
    --list-ports        # 查看某一项状态 - 端口
    --list-services        # 查看某一项状态 - 服务端口
    
    --add-service <服务>        # 添加service端口
    --add-port <端口/协议>       # 添加指定协议指定端口, eg. 81/tcp
    --add-source <源ip[/网段]>    # 添加源地址
    --add-interface <网卡接口>
    
    --remove-service <服务>
    --remove-port <端口/协议>
    --remove-source <源ip[/网段]>
    --remove-interface <网卡接口>
```

命令示例：

* firewall-cmd --permanent --zone=public --add-rich-rule 'rule family=“ipv4” source address=“192.168.0.4/24” port port protocal=“tcp” port=“3306” accept'

命令返回：

```bash
#public区域    激活状态
#  👇          👇
public (active)
  target: default                # 👈
  icmp-block-inversion: no
  interfaces: eth0                # 👈 public区域绑定了 eth0 网卡
  sources:                        # 允许访问的源ip
  services: ssh dhcpv6-client    # 允许访问的服务端口
  ports:                        # 允许访问的端口
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

## telnet服务

### 安装服务

```bash
## telnet 是客户端工具
## telnet-server 是服务端服务
## xinetd 是因为 telnet-server 无法自启动, 因此需要使用它来管理 telnet-server 的启动
yum install telnet telnet-server xinetd -y
```

### 启动服务

```
 systemctl start xinetd
 systemctl start telnet.socket
```

### 服务特点

* telnet 是明文传输, 危险, 因此不能应用于远程管理.
* telnet 不负责端口的监听, 只负责处理传过来的数据。
* 端口监听工作交由 xinetd 负责, xinetd 负责端口监听并将请求转发给 telnet
* root 用户无法用于telnet 远程登录

## ssh服务

客户端配置文件 `/etc/ssh/ssh_config`

服务配置文件 `/etc/ssh/sshd_config`

### 重要配置项

```
Port 22                                            # 服务端口号(默认是22)
PermitRootLogin yes                                # 是否允许root登录
AuthorizedKeysFile  .ssh/authorized_keys        # 密钥验证的公钥存储文件. 默认是放在 ~/.ssh/authorized_keys
```

### 配置项生效

systemctl restart sshd.service

### ssh命令

```bash
ssh 选项 [<user>@]<host>

选项
    常用
    -p <port>        # 指定连接端口
    -4              # 强制 ssh 只使用 IPv4 地址
    -f              # 要求 ssh 在执行命令前退至后台.
    
    端口转发
    -N                      # 不执行远程命令. 用于转发端口.
    -L port:host:hostport    # local, 将本地机(客户机)的某个端口转发到远端指定机器的指定端口
                            # 工作原理是这样的, 本地机器上分配了一个 socket 侦听 port 端口, 一旦这个端口上有了连接, 该连接就经过安全通道转发出去, 同时远程主机和 host 的 hostport 端口建立连接. 可以在配置文件中指定端口的转发.
                             # 只有 root 才能转发特权端口.  IPv6 地址用另一种格式说明: port/host/hostport

    -R port:host:hostport    # remote, 将远程主机(服务器)的某个端口转发到本地端指定机器的指定端口
                            # 工作原理是这样的, 远程主机上分配了一个 socket 侦听 port 端口, 一旦这个端口上有了连接, 该连接就经过安全通道转向出去, 同时本地主机和 host 的 hostport 端口建立连接.
                            # 可以在配置文件中指定端口的转发. 只有用 root 登录远程主机 才能转发特权端口. IPv6 地址用另一种格式说明: port/host/hostport
                            
                            
示例
    ssh -fNL 27117:127.0.0.1:27017 <远程主机地址>        # 建立local隧道, 将来自27117的连接经过<远程主机>转发至127.0.0.1(其实还是<远程主机>)的27017端口
                                                    # 通常适用于<远程主机>未对外开放27017端口
                                                    
    ssh -fNR 2222:127.0.0.1:22 <远程主机地址>            # 建立remote隧道(也叫反向隧道), 将来自<远程主机>2222端口的连接经过本机转发给127.0.0.1(实际上还是本机)
                                                    # 通常适用于外网访问内网服务
```

