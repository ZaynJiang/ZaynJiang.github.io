## 防火墙

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

eg. `Chain INPUT (policy ACCEPT)` 表示若未匹配任意规则则默认接受。默认 DROP, 配置特定规则 ACCEPT，常见于生产环境

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