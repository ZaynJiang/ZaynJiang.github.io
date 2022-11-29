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

## 3. Java NIO Buffer  
&emsp;&emsp;一个 Buffer 其实就是一块内存区域, 我们可以在这个内存区域中进行数据的读写。   
&emsp;&emsp;NIO Channel 进行交互时，需要使用到 NIO Buffer, 即数据从 Buffer读取到 Channel 中, 并且从 Channel 中写入到 Buffer 中。常见的nio的buffer封装有：  
* ByteBuffer
* CharBuffer
* DoubleBuffer
* FloatBuffer
* IntBuffer
* LongBuffer
* ShortBuffer  
### 3.1. Buffer的核心概念  
#### 3.1.1. Buffer基本使用  
* 将数据写入到 Buffer 中.
* 调用 Buffer.flip()方法, 将 NIO Buffer 转换为读模式.
* 从 Buffer 中读取数据
* 调用 Buffer.clear() 或 Buffer.compact()方法, 将 Buffer 转换为写模式.  

```
    public static void main(String[] args) {
        IntBuffer intBuffer = IntBuffer.allocate(2);
        intBuffer.put(12345678);
        intBuffer.put(2);
        intBuffer.flip();
        System.err.println(intBuffer.get());
        System.err.println(intBuffer.get());
    }
```
如上的代码：
* 我们分配两个单位大小的 IntBuffer, 因此它可以写入两个 int 值.
* 我们使用 put 方法将 int 值写入, 然后使用 flip 方法将 buffer 转换为读模式, 然后连续使
* 用 get 方法从 buffer 中获取这两个 int 值  
**注意：调用一次 get 方法读取数据时, buffer 的读指针都会向前移动一个单位长度(在这里是一个 int 长度)Buffer 属性**
#### 3.1.2. buffer的主要属性
* Capacity  
  最多写入capacity 个单位的数据到 Buffer 中
* Position    
  表示操作的指针
  * 当从一个 Buffer 中写入数据时, 我们是从 Buffer 的一个确定的位置(position)开始写入的，当我们从 Buffer 中读取数据时, 我们也是从某个特定的位置开始读取的.  
  * 当我们调用了 filp()方法将 Buffer 从写模式转换到读模式时, position 的值会自动被设置为0, 每当我们读取一个单位的数据, position 的值递增1.
* limit  
  limit - position 表示此时还可以写入/读取多少单位的数据，如果此时 limit 是10, position 是2, 则表示已经写入了2个单位的数据, 还可以写入 10 - 2 = 8 个单位的数据

#### 3.1.2. Buffer的分配  
每个类型的 Buffer 都有一个 allocate()方法, 我们可以通过这个方法分配 Buffer ，如：
```
  ByteBuffer buf = ByteBuffer.allocate(48);
```  
对于buffer的内存来源有两种，有Direct Buffer 和 Non-Direct Buffer 。具体的区别有：  
* Direct Buffer
  * 所分配的内存不在 JVM 堆上, 不受 GC 的管理.(但是 Direct Buffer 的 Java 对象是由 GC 管理的, 因此当发生 GC, 对象被回收时, Direct Buffer 也会被释放)
  * 因为 Direct Buffer 不在 JVM 堆上分配, 因此 Direct Buffer 对应用程序的内存占用的影响就不那么明显(实际上还是占用了这么多内存, 但是 JVM 不好统计到非 JVM 管理的内存.)
  * 申请和释放 Direct Buffer 的开销比较大. 因此正确的使用 Direct Buffer 的方式是在初始化时申请一个 Buffer, 然后不断复用此 buffer, 在程序结束后才释放此 buffer.
  * 使用 Direct Buffer 时, 当进行一些底层的系统 IO 操作时, 效率会比较高, 因为此时 JVM 不需要拷贝 buffer 中的内存到中间临时缓冲区中
* Non-Direct Buffer
  * 直接在 JVM 堆上进行内存的分配, 本质上是 byte[] 数组的封装.
  * 数据会进行从内核态拷贝到用户态  
    因为 Non-Direct Buffer 在 JVM 堆中, 因此当进行操作系统底层 IO 操作中时, 会将此 buffer 的内存复制到中间临时缓冲区中. 因此 Non-Direct Buffer 的效率就较低.

### 3.2. 一些buffer主要方法  
* 重置 position  
  Buffer.rewind()方法可以重置 position 的值为0， 因此我们可以重新读取/写入 Buffer 了。如果是读模式, 则重置的是读模式的 position, 如果是写模式, 则重置的是写模式的 position
* mark()和 reset()   
  我们可以通过调用 Buffer.mark()将当前的 position 的值保存起来, 随后可以通过调用 Buffer.reset()方法将 position 的值回复回来，用来多次读取数据。
* flip
  当从写模式变为读模式时, 原先的 写 position 就变成了读模式的 limit
* rewind  
  这个方法仅仅是将 position 置为0
* clear    
  clear 将 positin 设置为0, 将 limit 设置为 capacity.  
  主要的使用场景为：  
  * 在一个已经写满数据的 buffer 中, 调用 clear, 可以从头读取 buffer 的数据.
  * 为了将一个 buffer 填充满数据, 可以调用 clear, 然后一直写入, 直到达到 limit

### 3.3. buffer的compare  
* 两个 Buffer 是相同类型的
* 两个 Buffer 的剩余的数据个数是相同的
* 两个 Buffer 的剩余的数据都是相同的.     


## 4. Java Nio Selector  
&emsp;&emsp;Selector是用来实现单一的线程来操作多个 Channel的方法。  
&emsp;&emsp;为了使用 Selector, 我们首先需要将 Channel 注册到 Selector 中, 随后调用 Selector 的 select()方法, 这个方法会阻塞, 直到注册在 Selector 中的 Channel 发送可读写事件. 当这个方法返回后, 当前的这个线程就可以处理 Channel 的事件了。  

### 4.1. selector核心概念  
* 创建选择器  
  ```
  Selector selector = Selector.open();
  ```
* Channel 注册到选择器
  ```
  channel.configureBlocking(false);
  SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
  ```  
**注意：如果一个 Channel 要注册到 Selector 中, 那么这个 Channel 必须是非阻塞的, 即channel.configureBlocking(false)，FileChannel 是不能够使用选择器因为它只能是阻塞的**   

* 事件类型  
  一个 Channel发出一个事件也可以称为 对于某个事件，我们注册的时候指定了我们对 Channel 的什么类型的事件感兴趣
  * Connect  
    即连接事件(TCP 连接), 对应于SelectionKey.OP_CONNECT
  * Accept  
    即确认事件, 对应于SelectionKey.OP_ACCEPT
  * Read  
    即读事件, 对应于SelectionKey.OP_READ, 表示 buffer 可读.
  * Write   
    即写事件, 对应于SelectionKey.OP_WRITE, 表示 buffer 可写.  


* SelectionKey        
  使用 register 注册一个 Channel 时, 会返回一个 SelectionKey 对象
  * interest set  
    即我们感兴趣的事件集, 即在调用 register 注册 channel 时所设置的 interest set。可以通过如下方式获取我们注册的这些信息：  
    ```
    int interestSet = selectionKey.interestOps();
    boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
    boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
    boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
    boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;
    ```
  * ready set  
    表示我们的channel哪些操作准备好了。可以采用如上的方式来判断，也可以通过如下的方式来判断
    ```
    int readySet = selectionKey.readyOps();
    selectionKey.isAcceptable();
    selectionKey.isConnectable();
    selectionKey.isReadable();
    selectionKey.isWritable();
    ```  
  * Channel 和 Selector  
    一个selectorkey一定会对应一个channel的，我们可以通过 SelectionKey 获取相对应的 Channel 和 Selector  
    ```
      Channel  channel  = selectionKey.channel();
      Selector selector = selectionKey.selector();
    ```
  * Attaching Object
    我们可以在selectionKey中附加一个对象,也可以获取它。


### 4.2. selector操作channel
&emsp;&emsp;如果我们在注册 Channel 时,会说明对什么事件感兴趣，而当 select()返回时, 我们就可以获取有这种事件的Channel 了。  
&emsp;&emsp;select()方法返回的值表示有多少个 Channel 可操作。即：Selector.select()方法获取对某件事件准备好了的 Channel。
示例代码：
```
Set<SelectionKey> selectedKeys = selector.selectedKeys();

Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

while(keyIterator.hasNext()) {
    
    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
}
```

**注意, 在每次迭代时, 我们都调用 "keyIterator.remove()" 将这个 key 从迭代器中删除, 因为 select() 方法仅仅是简单地将就绪的 IO 操作放到 selectedKeys 集合中, 因此如果我们从 selectedKeys 获取到一个 key, 但是没有将它删除, 那么下一次 select 时, 这个 key 所对应的 IO 事件还在 selectedKeys。 我们可以动态更改 SekectedKeys 中的 key 的 interest set. 例如在 OP_ACCEPT 中, 我们可以将 interest set 更新为 OP_READ, 这样 Selector 就会将这个 Channel 的 读 IO 就绪事件包含进来了.**  


### 4.3. selector整体使用流程
* 通过 Selector.open() 打开一个 Selector.
* 将 Channel 注册到 Selector 中, 并设置需要监听的事件(interest set)
* 循环判断:
  * 调用 select() 方法
  * 调用 selector.selectedKeys() 获取 selected keys
  * 迭代每个 selected key:
      * 从 selected key 中获取 对应的 Channel 和附加信息(如果有的话)
      * 判断是哪些 IO 事件已经就绪了, 然后处理它们.   
 **注意：如果是 OP_ACCEPT 事件, 则调用 "SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept()" 获取 SocketChannel, 并将它设置为 非阻塞的, 然后将这个 Channel 注册到 Selector 中**
      * 根据需要更改 selected key 的监听事件.
      * 将已经处理过的 key 从 selected keys 集合中删  

* 关闭selector  
   Selector.close()方法时, 我们其实是关闭了 Selector 本身并且将所有的 SelectionKey 失效, 但是并不会关闭 Channel  

### 4.4. slector的使用示例 
```
public class NioEchoServer {
    private static final int BUF_SIZE = 256;
    private static final int TIMEOUT = 3000;

    public static void main(String args[]) throws Exception {
        // 打开服务端 Socket
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 打开 Selector
        Selector selector = Selector.open();

        // 服务端 Socket 监听8080端口, 并配置为非阻塞模式
        serverSocketChannel.socket().bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);

        // 将 channel 注册到 selector 中.
        // 通常我们都是先注册一个 OP_ACCEPT 事件, 然后在 OP_ACCEPT 到来时, 再将这个 Channel 的 OP_READ
        // 注册到 Selector 中.
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            // 通过调用 select 方法, 阻塞地等待 channel I/O 可操作
            if (selector.select(TIMEOUT) == 0) {
                System.out.print(".");
                continue;
            }

            // 获取 I/O 操作就绪的 SelectionKey, 通过 SelectionKey 可以知道哪些 Channel 的哪类 I/O 操作已经就绪.
            Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();

            while (keyIterator.hasNext()) {

                SelectionKey key = keyIterator.next();

                // 当获取一个 SelectionKey 后, 就要将它删除, 表示我们已经对这个 IO 事件进行了处理.
                keyIterator.remove();

                if (key.isAcceptable()) {
                    // 当 OP_ACCEPT 事件到来时, 我们就有从 ServerSocketChannel 中获取一个 SocketChannel,
                    // 代表客户端的连接
                    // 注意, 在 OP_ACCEPT 事件中, 从 key.channel() 返回的 Channel 是 ServerSocketChannel.
                    // 而在 OP_WRITE 和 OP_READ 中, 从 key.channel() 返回的是 SocketChannel.
                    SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept();
                    clientChannel.configureBlocking(false);
                    //在 OP_ACCEPT 到来时, 再将这个 Channel 的 OP_READ 注册到 Selector 中.
                    // 注意, 这里我们如果没有设置 OP_READ 的话, 即 interest set 仍然是 OP_CONNECT 的话, 那么 select 方法会一直直接返回.
                    clientChannel.register(key.selector(), OP_READ, ByteBuffer.allocate(BUF_SIZE));
                }

                if (key.isReadable()) {
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    ByteBuffer buf = (ByteBuffer) key.attachment();
                    long bytesRead = clientChannel.read(buf);
                    if (bytesRead == -1) {
                        clientChannel.close();
                    } else if (bytesRead > 0) {
                        key.interestOps(OP_READ | SelectionKey.OP_WRITE);
                        System.out.println("Get data length: " + bytesRead);
                    }
                }

                if (key.isValid() && key.isWritable()) {
                    ByteBuffer buf = (ByteBuffer) key.attachment();
                    buf.flip();
                    SocketChannel clientChannel = (SocketChannel) key.channel();

                    clientChannel.write(buf);

                    if (!buf.hasRemaining()) {
                        key.interestOps(OP_READ);
                    }
                    buf.compact();
                }
            }
        }
    }
}
```
