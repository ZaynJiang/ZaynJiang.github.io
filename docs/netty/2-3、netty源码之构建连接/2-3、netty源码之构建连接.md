## 1. 开头  

## 2. 创建serversocket
核心为：SocketChannel socketChannel = serverSocketChannel.accept();
### 2.1. NioEventLoop 中的 selector 轮询创建连接事件（OP_ACCEPT）  
![](图片11.png)  
![](图片12.png)    

### 2.2. 如果发现有OP_ACCEPT  事件，轮询任务则会调用到创建socketchannel  
![](图片13.png)     
![](图片14.png)  
![](图片15.png)  

### 2.3. 注册socketchannel
将生成的socketchannel注册到selectkey上  
![](图片16.png)    
![](图片17.png)     
### 2.4.  Eventloop会轮询selectkeys，检查是否有数据  
![](图片18.png)    