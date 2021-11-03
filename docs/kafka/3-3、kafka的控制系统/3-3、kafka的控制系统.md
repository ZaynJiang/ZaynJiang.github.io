## 1. 什么是控制器  
类似于elasticsearch的master，它的作用是管理和协调整个 Kafka 集群，在运行的过程中只会有一个broker能够成为控制器。它是强依赖于zookeeper的
## 2. ZooKeeper的作用
&emsp;&emsp;ZooKeeper高可靠性的分布式协调服务框架，客户端可以改变zk的节点，如果zk的节点发生了变更，Watch它的客户端将感知到这些变更。  
&emsp;&emsp;依托于这些功能，ZooKeeper 常被用来实现集群成员管理、分布式锁、领导者选举
## 3. 控制器的选举
Broker 在启动时，会尝试去 ZooKeeper 中创建 /controller 节点。Kafka 当前选举控制器的规则是：第一个成功创建 /controller 节点的 Broker 会被指定为控制器。  
## 4. 控制器作用

## 5. 控制器原理
### 5.1. 控制器是什么样的
### 5.2. 控制器核心原理
### 5.3. 故障转移

## 6. 总结