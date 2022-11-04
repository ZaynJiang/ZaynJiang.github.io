## 基本介绍

`Thrift`是一个**轻量级**、**跨语言**的**远程服务调用**框架，最初由`Facebook`开发，后面进入`Apache`开源项目。它通过自身的`IDL`**中间语言**, 并借助**代码生成引擎**生成各种主流语言的`RPC`**服务端**/**客户端**模板代码

## 使用案例

### 生成代码

### 服务端

```
public class SimpleServer {
    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(ServerConfig.SERVER_PORT);
        TServerSocket serverTransport = new TServerSocket(serverSocket);//设置传输协议
        HelloWorldService.Processor processor =
                new HelloWorldService.Processor<HelloWorldService.Iface>(new HelloWorldServiceImpl());//设置服务
        TBinaryProtocol.Factory protocolFactory = new TBinaryProtocol.Factory();//设置2进制序列化协议
        TSimpleServer.Args tArgs = new TSimpleServer.Args(serverTransport);
        tArgs.processor(processor);
        tArgs.protocolFactory(protocolFactory);

        // 简单的单线程服务模型 一般用于测试
        TServer tServer = new TSimpleServer(tArgs);//和网络模型绑定
        System.out.println("Running Simple Server");
        tServer.serve();//开启服务
    }
}
```

### 客户端

```
public class SimpleClient {
    public static void main(String[] args) {
        TTransport transport = null;
        try {
            transport = new TSocket(ServerConfig.SERVER_IP, ServerConfig.SERVER_PORT, ServerConfig.TIMEOUT);
            TProtocol protocol = new TBinaryProtocol(transport);
            HelloWorldService.Client client = new HelloWorldService.Client(protocol);
            transport.open();

            String result = client.say("Leo");
            System.out.println("Result =: " + result);
        } catch (TException e) {
            e.printStackTrace();
        } finally {
            if (null != transport) {
                transport.close();
            }
        }
    }
}
```

## 网络协议

