## 1. 采集对象

Prometheus将监控的对象分为三种：

* 在线服务

  大多数后台服务，需要高并发和低延迟，通常指标包括并发数、接口响应时间、错误数和延迟时间等。这些指标的实时性要求很高。

* 离线服务

  通常任务经过多步骤完成。需要关注每个阶段的执行时间，以及执行成功或失败的次数。实时性要求不高

* 批处理服务

  关注每个阶段的执行时间，以及执行成功或失败的次数。实时性要求不高

## 2. 采集规范

Prometheus对监控数据的采集是有一定的规范的。Prometheus为监控对象提供一个标准的HTTP GET接口，调用接口每次都将返回所有的监控数据。监控数据以文本形式组织，这和大多数基于JSON或者Protocol Buffers(下文简称为protobuf)的数据组织形式不同，采用文本形式更加灵活，不必拘束于JSON的层次结构及protobuf 的 proto数据规范。每个指标都占用一行。

## 3. exporter

### 3.1. exporter背景

​	很多监控系统中都已经实现了Prometheus 的监控接口，例如etcd、Kubernetes、CoreDNS等，它们可以直接被Prometheus监控，但是还有大多数监控对象都没办法直接提供监控接口，比如mysql、redis诞生的早，有些本身不支持http，比如操作系统也不支持http。软件作者也不愿意加入prometheus规范的监控代码。因此诞生了exporter。

​	exporter是一个采集监控数据并通过 Prometheus 监控规范对外提供数据的组件。除了官方实现的exporter 如 Node exporter、HAProxy exporter、MySQL serverexporter，还有很多第三方实现如 Redis exporter和 RabbitMQ exporter等。

### 3.2.  工作机制

Prometheus会周期性地调用这些exporter 提供的metrics 数据接口来获取数据

而这些exporter主要通过被监控对象提供的监控相关的接口获取监控数据。这些方式包括：

* http

  比如rabbitmq exporter通过rabbitmq的https获取数据

* tcp

  redis的exporter通过redis提供的命令，mysql通过相关的监控表

* 本地文件

  例如Node exporter通过读取proc文件系统下的文件

* 标准协议

  例如IPMI exporter通过IPMI协议获取硬件相关的信息。这些exporter将不同规范和格式的监控指标进行转化，输出 Prometheus能够识别的监控数据格式，从而极大地扩展Prometheus采集数据的能力

我们可以使用Prometheus提供的客户端包来非常方便的自行开发各个exporter。目前各个语言都有。

### 3.3.  开发示例

主要是基于Prometheus的客户端的方法来生成计数、直方图、仪表盘等指标数据，并开放端口来调用。

我们可以参考exporter-java或者exproter-go的客户端。

### 3.4.  客户端核心机制

比如部分exporter会使用go_client。其中有些细节需要注意。

#### 3.4.1. 并发安全

一般有可能会多协程共同修改某个指标数据，go语言采用atomic原子包的cas不断尝试多线程更新

#### 3.4.2. 分位计算

summary指标需要在客户端计算，需要保存一定时间的监控数据，client维护了两个队列热缓存、冷缓存，stream排序合并、压缩

* 热缓存、冷缓存
* 排序合并
* 压缩

当通过metrics数据接口获取数据时,会调用Summary类型的指标的Write方法,在 Write方法中会通过Stream 的 Query方法获取具体的分位点。在数据没有被压缩之前，计算分位点可以直接通过“分位点×样本数”获取，但被压缩后的样本只能通过数据的宽度和偏移量估算，这是数据压缩带来的不可避免的精度损失。

### 3.5. 常见exporter

Prometheus已经帮我们实现了各种exporter。比如node、redis、mysql

#### 3.5.1. node exporter

用于主机监控，程序入口在node_exporter.go之中。

然后定义了一个采集器集合NodeCollector

并且，NodeCollector实现了Prometheus.Collector 接口，会被 Prometheus的客户端库调用，在Collect中遍历每个Collectors采集器，并执行采集动作，而采集动作就是获取信息，比如内存对于getMemInfo 方法，每个平台都有自己的实现，例如 Linux系统会通过读取/proc/meminfo获取操作系统的各种运行指标。其它的指标就类似了。

#### 3.5.2. redis exporter

Redis exporter ( https://github.com/oliver006/redis_exporter)，支持2.x、3.x、4.x和 5.x版本的Redis。

Redis exporter通过设置环境变量REDIS_ADDR和REDIS_PASSWORD连接到Redis来获取监控数据，如果是多个 Redis节点，则可以通过REDIS_FILE指定一个文件，在该文件中可以定义多个Redis节点的地址。

在Redis exporter启动后，需要在 Prometheus平台配置Redis exporter的地址，然后Prometheus才会去各个exporter去抓取数据。

redis exporter通过Redis原生的INFO ALL命令获取 Redis的所有性能信息，其中包括

* 服务端信息

  启动时间、进程信息

* 客户端信息

  连接个数等

* 内存信息

  使用内存、峰值等

* 持久化

  持久化文件大小、文件修改次数

* 状态

  服务端命令数、主从同步数等

* 副本

  从节点数、备份日志缓冲区大小

* cpu

这些指标被揭晓到后，redis exporter转化格式。

#### 3.5.3. mysql exporter

Prometheus官方提供了MySQL server exporter ( https:/github.com/prometheus/mysqld_exporter)

mysql exporter主要是通过环境变量DATA_SOURCE_NAME指定MySQL的地址，从而读取MySQL的性能数据，并通过默认的9104端口对外提供数据输出。

这里需要注意，MySQL server exporter通过读取MySQL数据库获取监控数据,所以在启动MySQL server exporter之前，需要**先在 MySQL中创建一个exporter用户并对它赋权**，允许查询所有进程列表、所有主从服务器的位置，以及所有库和表的权限。

mysql有两个重要的数据库：分别是存放系统信息的information_schema库和存放性能信息的performance_schema 库。

* information_schema

  放了数据库信息，表信息、列信息、innodb压缩信息、innodb压缩次数、耗时、计数器名称、平均值等等、运行线程id、状态等等

* performance_schema

  该库为监控而设计，包括索引的io、磁盘io、文件打开信息、mysql事件等待时间、sql扫描数、排序总数、锁时间、表维度的io等待等待，具体需要参考相关文档查看

  

## 4. 总结

