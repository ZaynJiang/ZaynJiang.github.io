## 1. java NIO Buffer简单回顾
之前已经了解过java NIO Buffer，它的主要关键点：  
* ByteBuffer底层实现包含四个关键字段，并满足大小关系：mark <= position <= limit <= capacity
* ByteBuffer存在写模式和读模式两种状态，内部方法可以触发状态切换，比如flip方法从写状态切换为读状态
* 不同类型的ByteBuffer支持不同的数据类型，包括ByteBuffer、ShortBuffer、CharBuffer、DoubleBuffer、FloatBuffer、IntBuffer等类型


## 2. Netty的扩展
netty的ByteBuf对ByteBuffer实现了非常多扩展功能，并摒弃了一些不足：  
* 不区分读写状态，不需要切换状态；
* 支持池化，避免频繁的GC回收；
* 支持引用计数；
* 类型兼容（同一个ByteBuf可以承载各种数据类型）；
* 支持Unsafe操作的ByteBuf；
* 支持堆外和堆内两种ByteBuf；
* 支持零拷贝的复合类型CompositeByteBuf；  


## 3. ByteBuf的基本特性
### 3.1. 继承关系  
ByteBuf是一个接口，它有众多的实现。子类的命名非常规整，仅从名字上我们就可以将各个子类划分为以下几类：  
* 池化和非池化的ByteBuf，例如：PooledHeapByteBuf 和 UnpooledHeapByteBuf；
* 含Unsafe操作的ByteBuf，例如：PooledUnsafeHeapByteBuf;
* 分片类型的ByteBuf，例如：PooledSliceByteBuf和PooledDuplicatedByteBuf；
* 组合ByteBuf，例如：CompositeBuf;
* 实现了引用计数的ByteBuf。  


### 3.2. 读写指针  
类似NIO ByteBuffer，ByteBuf底层实现也是字节数组，也同样由读写指针来控制读写位置。在ByteBuf的继承类AbstractByteBuf中定义了以下读写指针字段： 
```
    // 当前读指针
    int readerIndex;
    // 当前写指针
    int writerIndex;
    // 暂存的读指针
    private int markedReaderIndex;
    // 暂存的写指针
    private int markedWriterIndex;
    // 最大容量
    private int maxCapacity;
```  