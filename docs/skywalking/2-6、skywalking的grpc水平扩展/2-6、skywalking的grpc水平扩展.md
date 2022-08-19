## 1. nginx

   因此，如果是虚机或者一般[容器](https://cloud.tencent.com/product/tke?from=10680)（非Kubernetes平台）部署的时候，为了实现 Skywalking OAP 负载均衡，需要自己做一层反向代理。在网上查阅资料之后，发现 Nginx 已经支持 gRPC 代理。在 2018年3月17日，NGINIX官方宣布在nginx 1.13.10中将会支持gRPC，这一宣告表示了NGINX已完成对gRPC的原生支持。众所周知，gRPC 已经是新一代微服务的事实标准 RPC 框架。对于实现来说，可以用服务框架等手段来做到负载均衡，但业界其实还没有非常成熟的针对 gRPC 的反向代理软件。 NGINIX 作为老牌负载均衡软件对 gRPC 进行了支持，之前已经可以代理 gRPC 的 TCP 连接，新版本之后，还可以终止、检查和跟踪 gRPC 的方法调用：

- 发布 gRPC 服务，然后使用 NGINX 应用 HTTP/2 TLS 加密、速率限制、基于 IP 的访问控制列表和日志记录；
- 通过单个端点发布多个 gRPC 服务，使用 NGINX 检查并跟踪每个内部服务的调用；
- 使用 Round Robin, Least Connections 或其他方法在集群分配调用，对 gRPC 服务集群进行负载均衡