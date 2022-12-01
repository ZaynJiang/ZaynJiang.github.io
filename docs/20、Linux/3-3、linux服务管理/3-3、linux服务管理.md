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

#### 命令示例：

* firewall-cmd --permanent --zone=public --add-rich-rule 'rule family=“ipv4” source address=“192.168.0.4/24” port port protocal=“tcp” port=“3306” accept'

  命令返回：

  ```
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

### ssh普通命令

#### 语法

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
```

#### 示例

* ssh -fNL 27117:127.0.0.1:27017 <远程主机地址>        

  建立local隧道, 将来自27117的连接经过<远程主机>转发至127.0.0.1(其实还是<远程主机>)的27017端口

  通常适用于<远程主机>未对外开放27017端口

* ssh -fNR 2222:127.0.0.1:22 <远程主机地址> 

  建立remote隧道(也叫反向隧道), 将来自<远程主机>2222端口的连接经过本机转发给127.0.0.1(实际上还是本机)

  通常适用于外网访问内网服务

### 免密ssh

配置访问特定主机时使用特定的私钥文件。

当前主机存在多个私钥文件

访问特定主机时需使用特定私钥文件(非默认)

```
# 假设私钥文件路径: ~/.ssh/xxx_id_rsa
cat > ~/.ssh/config <<'EOF'
Host 192.168.0.143
    PubkeyAuthentication yes
    IdentityFile ~/.ssh/xxx_id_rsa
EOF
# 注意 config 文件和私钥文件的权限都必须是 600
chmod 600 ~/.ssh/config
```

#### 公钥生成

ssh-keygen

```
生成密钥对

ssh-keygen [选项]

选项
    -t <密钥类型=rsa>         # 指定密钥类型: dsa|ecdsa|ed25519|rsa|rsa1
    -C <comment=$USER@$HOSTNAME>            #  注释(追加在公钥文件最后)
```

**生成密钥对一定是在客户端上做, 然后再将公钥传给服务端.**

默认生成的文件目录为：

```
~/.ssh/id_rsa
~/.ssh/id_rsa.pub
```

#### 公钥拷贝

将公钥(通过ssh)拷贝到目标主机

```
ssh-copy-id 选项 [user@]hostname
选项
    -n                                            # dry-run, 不真正执行命令
    -i <identity_file=~/.ssh/id_rsa.pub>        # 手动指定公钥文件
    -p port
    -o ssh_option
```

目标主机的公钥存储文件 `~/.ssh/authorized_keys`, 600 权限

### scp

scp是基于ssh协议在网络之间进行安全传输(文件传输)的命令。远程文件复制

#### 语法

```bash
scp [选项] <src> <dest>

选项
    -C      允许压缩
    -r      递归复制整个目录
    -P         port

src 和 dest 可以是以下任意的
    [<user>@]<host>:/remote/path    # 指定远程文件或目录
    /local/path                        # 指定本地文件或目录
    
示例
    scp a.txt 192.168.0.16:/tmp/    # 拷贝本地的 a.txt 到远程的 /tmp/ 目录下
```

#### 示例

传递大文件更推荐用 rsync(断点续传, 压缩)

### rsync

快速、通用的远程和本地文件传输工具

#### 语法

```bash
    # 本地复制
    rsync [选项] <src> <dest>
    
    # 通过 ssh 远程复制
    Pull: rsync [OPTION...] [USER@]HOST:SRC... [DEST]
    Push: rsync [OPTION...] SRC... [USER@]HOST:DEST
    
    # 通过 rsync 服务
    Pull: rsync [OPTION...] [USER@]HOST::SRC... [DEST]
             rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
    Push: rsync [OPTION...] SRC... [USER@]HOST::DEST
          rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST    
    
选项
    # 常用
    -P                         # 断点续传(保留那些因故没有完全传输的文件，以便加快随后的再次传输), 等同于 --partial --progress
    -z, --compress             # 对备份的文件在传输时进行压缩处理, 加快传输
    -e, --rsh=<COMMAND>     # 以ssh方式进行数据传输, -e "ssh -p2222" 指定连接2222端口, 如果是ssh方式默认使用22端口

    --bwlimit                # 限速, 字节/秒
    -r,--recursive             # 对子目录以递归模式处理

    --progress                 # 显示传输进度
    --partial                # 断点续传
    -c, --checksum             # 打开校验开关，强制对文件传输进行校验。(而不是基于文件的修改时间及大小)
    --delete                 # 删除那些 DEST 中 SRC 没有的文件。
    --delete-before            # rsync 首先读取 src 和 dest 的文件列表, 并删除那些 DEST 中 SRC 没有的文件, 之后再执行同步操作。
                            # 由于分两步骤执行, 因此需要更多的内存消耗以及时间消耗. 因此仅在 dest 的可用空间较小时用这种方式.
    --delete-excluded        # 除了删除 DEST 中 SRC 没有的文件外, 还会一并移除 dest 中 --exclude 或 --exclude-from 指定的文件/目录.
    -u, --update            # 如果 DEST 中的文件比 SRC 新(指修改时间), 就不会对该文件进行同步.    
    --exclude=PATTERN         # 指定排除不需要传输的文件, 比如 --exclude="logs" 会过滤掉文件名包含 "logs" 的文件或目录, 不对齐同步.
    --include=PATTERN         # 指定不排除而需要传输的文件模式。
    -v, --verbose             # 详细模式输出。
    -q, --quiet             # 精简输出模式。
    
    
    -a, --archive             # 归档模式，表示以递归方式传输文件，并保持所有文件属性，等于-rlptgoD
    -t, --list              # list the contents of an archive
    -l, --links                # 保留软链
    -L, --copy-links        # 同步软链时, 以实际的文件来替代
    -p, --perms             # 保持文件的权限属性
    -o, --owner             # 保留文件的属主(super-user only)
    -g, --group             # 保留文件的属组
    -D                        # 保持设备文件信息, 等同于 "--devices --specials"
    -t, --times                # 保持文件的时间属性
```

#### 示例

* rsync -P -z -r root@xx.xx.xx.xx:/data/transfer/archive.zip /data/archive.zip

  当带宽足够大时, 使用 `-z` 反而会拖慢速度.

* rsync -P -e "ssh -p2222" --bwlimit=200 root@xx.xx.xx.xx:/data/transfer/archive.zip /data/archive.zip

## ftp服务

### ftp 协议

文件传输协议。

- 需要同时建立命令链路(21端口, 先建立, 传输命令)和数据链路(传输文件名称, 目录名称, 文件数据)。即两条链路。

- 数据链路

  - 主动模式

    命令链路建立后, 服务端(使用20端口)主动向客户端发起建立数据链路请求(实际可能会被客户端防火墙之类的挡住, 导致无响应)

  - **被动模式**(实际常用)

    命令链路建立后, 服务端会开放大于1024的端口被动等客户端连接

### vsftpd 安装

vsftpd 是服务端、ftp 是客户端

```bash
# 安装必要软件
yum install vsftpd ftp

# 启动 vsftpd 服务
systemctl start vsftpd.service && systemctl enable vsftpd.service
```

- 默认提供匿名账号: `ftp`
- 默认当前系统的账号

### 配置文件

通过 `man 5 vsftpd.conf` 可以查看配置文件详解

#### 配置文件类型

- `/etc/vsftpd/vsftpd.conf` 主配置文件

  ```bash
  anonymous_enable=YES        # 是否允许匿名用户(ftp)
  local_enable=YES            # 是否允许系统本地用户账号登录. 同时会受到 SELinux(ftp_home_dir 项)的影响
  write_enable=YES            # 本地用户是否可写
  connect_from_port_20=YES    # 是否允许开启主动模式(不会影响被动模式)
  userlist_enable=YES            # 是否启用用户黑白名单
  userlist_deny=YES            # 控制用户列表是黑名单还是白名单, YES 表示黑名单. 
                              # 仅在 userlist_enable=YES 时生效
                              
  # 虚拟用户配置相关
  guest_enable=YES            # 允许虚拟用户进行验证
  guest_username=<系统用户名>     # 指定登录身份为某个系统用户
  user_config_dir=/etc/vsftpd/<系统用户名>config        # 指定虚拟用户权限控制文件所在目录
  allow_writeable_chroot=YES        # 虚拟用户是否可写
  pam_service_name=vsftpd.vuser    # 虚拟用户的"可插拔验证模块(pam)"对应文件名称
  ```

  `YES` 和 `NO` 必须是大写的.

  可以使用 `man 5 vsftpd.conf` 来查看该配置文件的帮助

- `/etc/vsftpd/ftpusers` 用户相关

- `/etc/vsftpd/user_list` 用户黑白名单, 比如禁止 root 登录

#### 强制访问控制

强制访问控制对 ftpd 的影响

```bash
# 查看 SELinux 中和 ftpd 相关的布尔配置项
getsebool -a | grep ftpd

# 修改
## -P    同时写入配置文件
## 0     表示 off, 关闭
## 1    表示 on, 打开
setsebool -P <配置项名> 1|0
```

### 账号

#### 账号类型

* 匿名账号
  * 账号: `ftp`
  * 密码: 空
  * 默认目录: `/var/ftp/`

* 普通账号
  - 账号: 系统账号
  - 密码: 系统账号的密码
  - 默认目录: `~`
  - 能访问普通账号的 home 目录

* 虚拟用户

  ```bash
  # 1. 建立一个真实系统账号
  ## 指定 /data/ftp 为该用户的home目录
  ## 指定该用户不可登录到系统
  useradd -d /data/ftp -s /sbin/nologin vuser
  
  # 2. 编写存储虚拟用户账号和密码的临时文件
  ## 该文件格式是: 一行虚拟用户名, 一行对应密码
  cat <<'EOF' > /etc/vsftpd/vuser.tmp
  u1
  123456
  u2
  123456
  u3
  123456
  EOF
  
  # 3. 将上述临时文件转成数据库专用格式
  db_load -T -t hash -f /etc/vsftpd/vuser.temp /etc/vsftpd/vuser.db
  
  # 4. 创建可插拔验证模块配置
  cat <<'EOF' > /etc/pam.d/vsftpd.vuser
  auth sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vuser
  account sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vuser
  EOF
  
  # 5. 修改 /etc/vsftpd/vsftp.conf 确保其中相关配置如下
      guest_enable=YES
      guest_username=vuser
      user_config_dir=/etc/vsftpd/vuserconfig
      allow_writeable_chroot=YES
      pam_service_name=vsftpd.vuser
      # 注释掉以下语句后, 就不再支持匿名和本地用户登录了
      #pam_service_name=vsftpd
      
  # 6. 创建虚拟用户配置所在目录
  mkdir /etc/vsftpd/vuserconfig
  
  # 7. 在虚拟用户配置目录中创建和所要创建虚拟用户名同名的配置文件
  ## 此处创建 u1, u2, u3 的配置文件
  ## 省略 u2 的...
  ## 省略 u3 的...
  cat <<'EOF' > /etc/vsftpd/vuserconfig/u1
  local_root=/data/ftp            # 用户登录后进入的目录
  write_enable=YES                # 可写
  anon_umask=022
  anon_world_readable_only=NO        # 可写?
  anon_upload_enable=YES            # 可上传
  anon_mkdir_write_enable=YES        # 可创建目录?
  anon_other_write_enable=YES        # 可写?
  download_enable=YES                # 可下载
  EOF
  
  # 8. 重启 vsftpd 服务
  systemctl restart vsftpd.service
  ```

#### 虚拟账户示例

* 建立一个真实系统账号

  useradd -d /data/ftp -s /sbin/nologin vuser

  指定 /data/ftp 为该用户的home目录

  指定该用户不可登录到系统

* 编写存储虚拟用户账号和密码的临时文件

  cat <<'EOF' > /etc/vsftpd/vuser.tmp

  ```
  u1
  123456
  u2
  123456
  u3
  123456
  EOF
  ```

  该文件格式是: 一行虚拟用户名, 一行对应密码

* 将上述临时文件转成数据库专用格式

  db_load -T -t hash -f /etc/vsftpd/vuser.temp /etc/vsftpd/vuser.db

* 创建可插拔验证模块配置
  * cat <<'EOF' > /etc/pam.d/vsftpd.vuser
  * auth sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vuser
  * account sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vuser
  * EOF

* 修改 /etc/vsftpd/vsftp.conf 确保其中相关配置如下

  ```
   guest_enable=YES
   guest_username=vuser
   user_config_dir=/etc/vsftpd/vuserconfig
   allow_writeable_chroot=YES
   pam_service_name=vsftpd.vuser
  ```

  注释掉以下语句后, 就不再支持匿名和本地用户登录了

  ```bash
   #pam_service_name=vsftpd
  ```

* 创建虚拟用户配置所在目录

  mkdir /etc/vsftpd/vuserconfig

* 在虚拟用户配置目录中创建和所要创建虚拟用户名同名的配置文件

  此处创建 u1, u2, u3 的配置文件

  省略 u2 的...

  省略 u3 的...

  cat <<'EOF' > /etc/vsftpd/vuserconfig/u1

  ```
  local_root=/data/ftp            # 用户登录后进入的目录
  write_enable=YES                # 可写
  anon_umask=022
  anon_world_readable_only=NO        # 可写?
  anon_upload_enable=YES            # 可上传
  anon_mkdir_write_enable=YES        # 可创建目录?
  anon_other_write_enable=YES        # 可写?
  download_enable=YES                # 可下载
  EOF
  ```

* 重启 vsftpd 服务

  systemctl restart vsftpd.service

### ftp命令

语法

```bash
ftp客户端

ftp <地址>

选项
```

如果提示"没有到主机的路由", 一般是由于被防火墙挡住.

```bash
ls            # 在远程执行 ls
!ls            # 在本地执行 ls

pwd            # 在远程执行 pwd
!pwd        # 在本地执行 pwd

cd            # 切换远程目录
lcd            # 切换本地目录

put    <file>        # 上传文件, 若提示无权限则应检查 "write_enable" 配置项
get <file>        # 下载文件
```

## samba 服务

smb 协议是微软持有的版权, 用于windows之间的共享。而samba则是模拟这种协议, 主要用于共享给windows.

若是 Linux 之间的共享则建议使用 nfs

- 使用 smb 协议
- 使用 cifs 文件系统
- `/etc/samba/smb.conf` 配置文件

### 安装

```bash
# 安装
yum install samba

# 服务
systecmtl start|stop|restart|reload smb.service
```

### 配置文件

配置文件 `/etc/samba/smb.conf` 部分格式说明

```bash
[global]                        # 全局设置
    workgroup = SAMBA
    security = user

    passdb backend = tdbsam

    printing = cups
    printcap name = cups
    load printers = yes
    cups options = raw


[share]                        # 共享名
    comment = my share
    path = /data/share        # 共享路径
    read only = No            # 是否只读, No 表示可写
```

`man 5 smb.conf` 可查看该配置文件的帮助文档

### smbpasswd 命令

```bash
samba 用户的创建和删除

smbpasswd [选项]

选项
    -a         # 添加用户(系统中必须有一个同名的用户, samba 用户访问权限是参考系统同名用户的)
    -x        # 删除用户
    
    -s        # silent, 从标准输入上读取原口令和新口令, 常用于脚本处理smb
```

新创建的用户默认会直接共享出自己的home目录, 也就是 `/home/用户名` 这个目录

### pdbedit 命令

```bash
samba 用户查看

pdbedit [选项]

选项
    -L        # 查看用户
```

### 示例

* 创建系统用户, 此处以 user1 为例

  useradd user1

* 创建同名samba用户

  echo -e "123456\n123456" | smbpasswd -a user1

* 启动samba服务

  systemctl start smb.service

* 使用

  ```
  ## windows 客户端可以通过映射网络驱动器或windows共享来使用
  ## Linux 客户端可以通过磁盘挂载使用(将 127.0.0.1 上的 /home/user1 挂载到了当前的 /mnt 目录)
  ### -t cifs 可省略, 由 mount 自行判断
  ### 输入密码后就挂载成功了
  ### 挂载完毕后可通过 df -hT 或 mount | tail -1 查看挂载信息
  mount -t cifs -o username=user1 //127.0.0.1/user1 /mnt        # 挂载前面在 /etc/samba/smb.conf 里配置的 [share] 共享所指定的目录
  mount -t cifs -o username=user1 //127.0.0.1/share /mnt2        # 此处举例, 挂载在 /mnt2 文件夹上
  ```

* 卸载

  不需要后就卸载掉

  umount /mnt
  umount /mnt2

## nfs服务

主要用于 Linux 之间的共享服务.默认已安装

### 启动步骤

* systemctl start|stop|reload nfs.service

### 配置文件

/etc/exports 主配置文件

* `man 5 exports` 可查看帮助

```bash
<共享目录> <允许来源主机>(权限)...

                    👆 这里不得有空格
                    可指定多个

共享目录
    必须是已存在的目录.

允许来源主机
    *            # 任意主机
    具体ip       # 指定该ip可访问
    
权限(用逗号分隔)
    rw                # 读写权限
    ro                # 只读权限
    sync            # (内存)数据同步写入磁盘, 避免丢失数据
    all_squash        # 使用 nfsnobody 系统用户
```

### 示例

* /data/share *(rw,sync,all_squash)

若权限设置了 `all_squash`, 则会使用 nfsnobody 这个用户来做实际操作, 因此需要将该共享目录的属主和属组设为 nfsnobody

```
chown -R nfsnobody:nfsnobody /data/share/
```

### showmount

显示关于 NFS 服务器文件系统挂载的信息

```
showmount [选项] <host>

选项
    -e, --exports    # 查看所有共享的目录
```

示例：

* mount 主机:/path/dir /local/path/to/mount

* mount localhost:/data/share /mnt

  将localhost上共享的 /data/share 目录挂载到本地的 /mnt 目录

## nginx

- Nginx(engine X) 是一个高性能的 Web 和反向代理服务器.

- Nginx 支持 HTTP, HTTPS 和电子邮件代理协议

  Nginx 模块由于是用c/c++编写的, 要添加新模块还需要重新编译.

- OpenResty 是基于 Nginx 和 Lua 实现的 Web 应用网关,集成了大量的第三方模块.

### 安装与管理

#### 安装

* yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo

  添加 yum 源

* yum install -y openresty

  安装 openresty

#### 管理

systemctl start|reload|stop openresty

### 配置文件

配置文件位于/usr/local/openresty/nginx/conf/nginx.conf

```bash
worker_processes  1;        # 配置多少个worker进程, 最大值建议不大于CPU核心数

error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;

events {
    # use epoll;
    worker_connections  1024;        # 每个worker允许的并发连接, 超过会返回 503 Service Unavailable 错误
}

http {
# 此处的配置会对下面所有 server 生效

    # 访问日志格式定义
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    
    # 访问日志记录文件及采用的格式配置
    access_log  logs/access.log  main;

    # sendfile 和 tcp_nopush 是在内核层面上优化传输链路
    sendfile        on;            # 传输文件时的优化: 直接在内核中将文件数据拷贝给socket.
    tcp_nopush     on;            # 仅sendfile启用时生效, 将http头和实体一同返回, 减少报文段数量

    keepalive_timeout  65;        # HTTP 协议的 keepalive, 建立长连接, 重用TCP连接
    
    gzip  on;                    # 传输时启用 gzip 压缩, 节省带宽, 造成CPU额外消耗.

    server {
        listen       80;            # 监听端口
        server_name  localhost;        # 域名(虚拟主机)
        
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
```

PS：上述配置中的相对路径是基于编译nginx时指定的特定路径。

一般是nginx所在目录, 对应此处是 ``/usr/local/openresty/nginx/`

## LNMP

### mysql

mariadb 是 MySQL 的社区版

* yum install mariadb mariadb-server

  mariadb 是客户端

* 修改配置文件 `/etc/my.cnf`

  ```
  [mysqld]
  character_set_server=utf8
  init_connect='SET NAMES utf8'
  ```

  或者是采用 utf8mb4 编码, 兼容4字节的unicode, 需要存emoji表情的话应使用 utf8mb4

### PHP

* yum install php-fpm php-mysql

  默认源的版本是 5.4, 需要更高的可以用 webtatic 源

### Nginx

```
server {
    location ~ \.php$ {
        root           html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

将.php后缀结尾的转发到127.0.0.1:9000端口

通过 fastcgi 协议将请求转发给 php-fpm

## DNS

DNS 服务介绍

- DNS(Domain Name System) 域名系统
- FQDN(Full Qualified Domain Name) 完全限定域名
- 域分类: 根域、顶级域(TLD)
- 查询方式: 递归、迭代
- 解析方式: 正向解析(主机 -> ip)、反向解析(ip -> 主机)
- DNS 服务器的类型: 缓存域名服务器、主域名服务器(master)、从域名服务器(salve)

```bash
www.baidu.com.
👆        👆      👆
主机名     域名      根域


.com    顶级域
.        根域
```

### BIND安装

bind软件提供 DNS 服务

* yum install bind bind-utils

  bind 提供服务的软件包、bind-utils DNS服务的相关工具

* systemctl start named.service

  服务管理

### BIND配置

主配置文件: `/etc/named.conf`

```
options {
    listen-on port 53 { any; };        // 监听端口及对应网卡
    ...    
    allow-query     { any; };        // any    允许任何人查询
}

// 根域
zone "." IN {
    type hint;
    file "named.ca";                // /var/named/named.ca
};
```

### named-checkconf

```bash
确认配置文件是否正确
named-checkconf
```

## NAS

NAS(Network attached storage)网络附属存储

支持的协议:

- nfs
- cifs
- ftp

一般是通过创建磁盘阵列RAID后, 再通过上述协议共享

### 新增硬盘

- /dev/sde
- /dev/sdf

### 创建共享空间

```bash
# 磁盘分区
fdisk /dev/sde
fdisk /dev/sdf

# 创建 RAID
## 此处创建 RAID1 级别的磁盘阵列
mdadm -C /dev/md0 -a yes -l 1 -n 2 /dev/sd{e,f}1

# 持久化 RAID 配置信息
mdadm --detail --scan --verbose > /etc/mdadm.conf

# 通过逻辑卷的方式以方便后续扩容
## 初始化物理卷
pvcreate /dev/md0
## 创建卷组
vgcreate vg1 /dev/md0
## 创建逻辑卷
### 此处示例, 因此只创建个 200M 的逻辑卷
lvcreate -L 200M -n nas vg1

# 分区格式化
mkfs.xfs /dev/vg1/nas

# 分区挂载
mkdir /share
mount /dev/vg1/nas /share
```

### 协议共享

```bash
# 创建公用用户 shareuser
useradd shareuser -d /share/shareuser
echo 123456 | passwd --stdin shareuser

# 1. 配置ftp共享 - 通过 shareuser 用户登录ftp并访问home目录 (也可以用虚拟用户)
确认 /etc/vsftpd/vsftpd.conf 配置
    pam_service_name=vsftpd
    local_enable=YES
    write_enable=YES

systemctl restart vsftpd.service

# 2. 配置samba服务
echo -e "123456\n123456" | smbpasswd -a shareuser
systemctl restart smb.service

# 3. 配置nfs服务
## 配置为 ro (nfs由于没有用户级别的限制, 因此这种情况下不推荐设置为 rw)
echo '/share/shareuser *(ro)' >> /etc/exports
systemctl restart nfs.service
## 配置为 rw (配合 facl 权限访问控制列表)
echo '/share/shareuser *(rw,sync,all_squash)' >> /etc/exports
setfacl -d -m u:nfsnobody:rwx /share/shareuser
setfacl -m u:nfsnobody:rwx /share/shareuser
```