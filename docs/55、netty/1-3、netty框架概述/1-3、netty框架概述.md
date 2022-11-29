## 1. 开头
众所周知，netty是一款优秀的网络通讯框架，netty是由韩国人开发的。这里从几大方面来概述netty框架： 
* 什么是netty框架
* 为什么要使用netty框架
* netty框架未来的发展。

## 2. netty框架是什么？  
### 2.1. netty整体印象
* 类型  
  网络应用程序的框架
* 异步、事件驱动
* 特性  
  高性能，可维护，快速快速开发
* 用途  
  开发高性能的服务器和客户端。  

![](netty整体架构.png)  

### 2.2. netty整体印象

## 3. 为什么要使用netty  
netty已经提供了jdk nio了，为什么还要使用netty呢？  
简单来说，是netty做的比nio更多和更好好。更多的功能表现为：  
* 支持常用的应用层协议  
* 解决了传输存在的问题：粘包、半包问题
* 支持流量整形
* 完善的断连、idle等异常问题  

更好的功能表现为：  
* 规避了jdk nio的bug
  * 经典的epoll bug，异常唤醒空转导致cpu 100%
  * ip_tos参数使用时抛出异常  

* 提供了更加好用的api
* 隔离了变化、屏蔽了细节  


## 4. netty的现状和发展  
* netty 4.1.39.Final(2019.8)
* netty 3.10.6.Final(2016.6)  
### 4.1. netty的使用者  
spark、hadoop、rocketmq、elasticsearch、grpc、dubbo、spring5、zk
  
### 4.2. netty未来趋势  
* 更多流行的趋势
* jdk的更新
* 更易用、人性化的
* ip黑白名单、流量整形
* 应用推广
