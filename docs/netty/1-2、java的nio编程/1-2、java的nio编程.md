## 1. 开头  
Java NIO 是由 Java 1.4 引进的异步 IO。它主要有三大核心概念组成。
* Channel
* Buffer
* Selector  

### 1.1. NIO和IO的对比  
* IO 基于流(Stream oriented), 而 NIO 基于 Buffer (Buffer oriented)
  * Stream  
    &emsp;&emsp;以流式的方式顺序地从一个 Stream 中读取一个或多个字节， 因此我们也就不能随意改变读取指针的位置
  * Buffer   
    &emsp;&emsp;从Channel 中读取数据到 Buffer 中, 当 Buffer 中有数据后, 我们就可以对这些数据进行操作了. 不像 IO 那样是顺序操作, NIO 中我们可以随意地读取任意位置的数据
* IO 操作是阻塞的, 而 NIO 操作是非阻塞的  
  * 阻塞  
    &emsp;&emsp;调用一个 read 方法读取一个文件的内容, 那么调用 read 的线程会被阻塞住, 直到 read 操作完成
  * 非阻塞  
    &emsp;&emsp;非阻塞地进行 IO 操作. 例如我们需要从网络中读取数据, 在 NIO 的非阻塞模式中, 当我们调用 read 方法时, 如果此时有数据, 则 read 读取并返回; 如果此时没有数据, 则 read 直接返回, 而不会阻塞当前线程
* IO 没有 selector 概念, 而 NIO 有 selector 概念 
  * selector  
     Selector线程可以监听多个 Channel 的 IO 事件, 当我们向一个Selector中注册了Channel后, Selector内部的机制就可以自动地为我们不断地查询(select) 这些注册的Channel是否有已就绪的 IO 事件(例如可读, 可写, 网络连接完成等)。通过这样的Selector机制, 我们就可以很简单地使用一个线程高效地管理多个Channel了  


## 2. Channel    

### 2.1. Channel和Stream对比  
&emsp;&emsp;NIO 的 I/O 操作都是从 Channel 开始的. 一个 channel 类似于一个 stream。这里我们对Chanel和stream进行对比：  
java Stream 和 NIO Channel   
* 同一个 Channel 中执行读和写操作, 同一个 Stream 仅仅支持读或写.
* Channel 可以非阻塞地读写, 而 Stream 是阻塞的同步读写.
* Channel 总是从 Buffer 中读取数据, 或将数据写入到 Buffer 中
### 2.2. java Channel
&emsp;&emsp;java定义了多种channel类型，这些通道涵盖了 UDP 和 TCP网络 IO以及文件 IO。
* FileChannel, 文件操作
* DatagramChannel, UDP 操作
* SocketChannel, TCP 操作
* ServerSocketChannel, TCP 操作, 使用在服务器端.  


#### 2.2.1. FileChannel
&emsp;&emsp;FileChannel 是操作文件的Channel, 我们可以通过 FileChannel 从一个文件中读取数据, 也可以将数据写入到文件中  
**注意： FileChannel 不能设置为非阻塞模式**
```
public static void main( String[] args ) throws Exception {
    RandomAccessFile aFile = new RandomAccessFile("myfile.xml", "rw");
    FileChannel inChannel = aFile.getChannel();
    ByteBuffer buf = ByteBuffer.allocate(48);
    int bytesRead = inChannel.read(buf);
    while (bytesRead != -1) {
        buf.flip();
        while(buf.hasRemaining()){
            System.out.print((char) buf.get());
        }
        buf.clear();
        bytesRead = inChannel.read(buf);
    }
    aFile.close();
}
```
操作一个FileChannel的步骤如下：  
* 获取FileChannel  
  RandomAccessFile aFile = new RandomAccessFile("test.txt", "rw");
  FileChannel inChannel = aFile.getChannel();  

* FileChannel 中读取数据   
  ByteBuffer buf = ByteBuffer.allocate(48);
  int bytesRead = inChannel.read(buf); 

* 写入数据  
  ```
    ByteBuffer buf = ByteBuffer.allocate(48);
    buf.clear();
    buf.put(newData.getBytes());
    buf.flip();
    while(buf.hasRemaining()) {
        channel.write(buf);
    }
  ```
* 关闭  
  channel.close();
* 设置 position  
  long pos channel.position();
  channel.position(pos +123);  
* channel.size()  
  返回channel中的文件大小
* channel.truncate(1024)  
  情况数据库  
* channel.force(true);  
  我们可以强制将缓存的未写入的数据写入到文件中
    

#### 2.2.2. SocketChannel  
&emsp;&emsp;SocketChannel 是一个客户端用来进行 TCP 连接的 Channel    
创建一个 SocketChannel 的方法有两种:
* 打开一个 SocketChannel, 然后将其连接到某个服务器中
* 当一个 ServerSocketChannel 接受到连接请求时, 会返回一个 SocketChannel 对象  

使用关键点：  
* 打开 SocketChannel  
```
    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.connect(new InetSocketAddress("http://example.com", 80));
```
* 关闭  
  socketChannel.close(); 

* 读取数据  
```
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = socketChannel.read(buf);
```  
**注意：如果 read()返回 -1, 那么表示连接中断了.**
* 写入数据  
```
String newData = "New String to write to file..." + System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());
buf.flip();
while(buf.hasRemaining()) {
    channel.write(buf);
}
```  
* 非阻塞模式     
   connect, read, write 都是非阻塞的，不用等待结果就返回了。
  ```
    socketChannel.configureBlocking(false);
    socketChannel.connect(new InetSocketAddress("http://example.com", 80));

    while(! socketChannel.finishConnect() ){
        //wait, or do something else...    
    }
  ```   
&emsp;&emsp;在非阻塞模式中, 或许连接还没有建立, connect 方法就返回了, 因此我们需要检查当前是否是连接到了主机, 因此通过一个 while 循环来判断。   


#### 2.2.3. ServerSocketChannel  
&emsp;&emsp;用在服务器为端的, 可以监听客户端的 TCP 连接  
关键如下：
* 开启  
  ```
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.socket().bind(new InetSocketAddress(9999));
    while(true){
        SocketChannel socketChannel = serverSocketChannel.accept();
        //do something with socketChannel...
    }
  ```  

* 关闭  
  ```
    serverSocketChannel.close();
  ```     


* 监听连接  
  ```
  while(true){
     SocketChannel socketChannel = serverSocketChannel.accept(); 
    //do something with socketChannel...
  }
  ```   
  * accept()方法来监听客户端的 TCP 连接请求
  * accept()方法会阻塞, 直到有连接到来
  * 当有连接时, 这个方法会返回一个 SocketChannel 对象

* 非阻塞模式  
  ```
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.socket().bind(new InetSocketAddress(9999));
    serverSocketChannel.configureBlocking(false);
    while(true){
        SocketChannel socketChannel = serverSocketChannel.accept();
        if(socketChannel != null){
            //do something with socketChannel...
        }
    }
  ```  
  accept()是非阻塞的, 因此如果此时没有连接到来, 那么 accept()方法会返回null,需要循环判断  


  #### 2.2.4. DatagramChannel
  用来处理 UDP 连接的  
  核心点为：
  * 打开
    ```
    DatagramChannel channel = DatagramChannel.open();
    channel.socket().bind(new InetSocketAddress(9999));
    ```  
  * 读取数据
    ```
    ByteBuffer buf = ByteBuffer.allocate(48);
    buf.clear(); 
    channel.receive(buf);
    ```
   * 连接某地址
    ```
     channel.connect(new InetSocketAddress("example.com", 80));
    ```   
**注意： connect 并不是向 TCP 一样真正意义上的连接，因为它是UDP的。我们仅仅可以从指定的地址中读取或写入数据**

  * 发送数据
    ```
    String newData = "New String to write to file..." + System.currentTimeMillis();
    ByteBuffer buf = ByteBuffer.allocate(48);
    buf.clear();
    buf.put(newData.getBytes());
    buf.flip();
    int bytesSent = channel.send(buf, new InetSocketAddress("example.com", 80));
    ```    

