## 介绍

gRPC 是一个高性能、开源和通用的 RPC 框架，面向服务端和移动端，基于 HTTP/2 设计。

grpc的特点为：

* 语言中立，支持多种语言；
* 基于 IDL 文件定义服务，通过 proto3 工具生成指定语言的数据结构、服务端接口以及客户端 Stub；
* 通信协议基于标准的 HTTP/2 设计，支持双向流、消息头压缩、单 TCP 的多路复用、服务端推送等特性，这些特性使得 gRPC 在移动端设备上更加省电和节省网络流量；
* 序列化支持 PB（Protocol Buffer）和 JSON，PB 是一种语言无关的高性能序列化框架，基于 HTTP/2 + PB, 保障了 RPC 调用的高性能

## grpc示例

### 服务端

编译helloworld.proto

```
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
message HelloRequest {
  string name = 1;
}
message HelloReply {
  string message = 1;
}
```

编写HelloWorldServer 类

```
private void start() throws IOException {
    /* The port on which the server should run */
    int port = 50051;
    server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .build()
        .start();
...
```

服务端接口实现类（GreeterImpl）

```
static class GreeterImpl extends GreeterGrpc.GreeterImplBase {
    @Override
    public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
      HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }
  }
```

### 客户端

## 线程模型

### netty线程回顾

### 服务端线程

aceptor -> worker -> cachethread -> serializing

### 客户端线程

gRPC 客户端的线程主要分为三类：

* 业务调用线程
* 客户端连接和 I/O 读写线程
* 请求消息业务处理和响应回调线程



## 服务端原理

### 服务创建

gRPC 服务端创建采用 Build 模式，对底层服务绑定、transportServer 和 NettyServer 的创建和实例化做了封装和屏蔽。

```
 int port = 50051;
    server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .build()
        .start();
```



## 客户端原理

### rpc功能介绍

gRPC 的通信协议基于标准的 HTTP/2 设计，主要提供了两种 RPC 调用方式：

* 普通 RPC 调用方式，即请求 - 响应模式
* 基于 HTTP/2.0 的 streaming 调用方式

#### 请求响应模式

#### streaming模式

基于 HTTP/2.0,gRPC 提供了三种 streaming 模式：

* 服务端 streaming

  服务端 streaming 模式指客户端 1 个请求，服务端返回 N 个响应，每个响应可以单独的返回。

  onNext回调方法将会被调用多次.

* 客户端 streaming

  客户端发送多个请求，服务端返回一个响应，多用于汇聚和汇总计算场景

* 服务端和客户端双向 streaming

  客户端发送 N 个请求，服务端返回 N 个或者 M 个响应，利用该特性，可以充分利用 HTTP/2.0 的多路复用功能，在某个时刻，HTTP/2.0 链路上可以既有请求也有响应，实现了全双工通信（对比单行道和双向车道）

### 调用流程

## 安全

## 序列化

