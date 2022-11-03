## 开头

### 序列化定义

互联网的产生带来了机器间通讯的需求，而互联通讯的双方需要采用约定的协议，序列化和反序列化属于通讯协议的一部分。通讯协议往往采用分层模型，不同模型每层的功能定义以及颗粒度不同，例如：TCP/IP协议是一个四层协议，而OSI模型却是七层协议模型。在OSI七层协议模型中展现层（Presentation Layer）的主要功能是把应用层的对象转换成一段连续的二进制串，或者反过来，把二进制串转换成应用层的对象--这两个功能就是序列化和反序列化。

一般而言，TCP/IP协议的应用层对应与OSI七层协议模型的应用层，展示层和会话层，所以序列化协议属于TCP/IP协议应用层的一部分。

- 序列化： 将数据结构或对象转换成二进制串的过程
- 反序列化：将在序列化过程中所生成的二进制串转换成数据结构或者对象的过程

**另一种说法：**

序列化 (Serialization)将对象的状态信息转换为可以存储或传输的形式的过程。在序列化期间，对象将其当前状态写入到临时或持久性存储区。以后，可以通过从存储区中读取或反序列化对象的状态，重新创建该对象。即

* （序列化）把 应用层的对象 转换成 二进制串
* （反序列化）把 二进制串 转换成 应用层的对象

属于  `TCP/IP`模型的应用层 & `OSI`模型的表示层

### 序列化协议

　　如果序列化后的二进制串为了验证需要写反序列化程序；如果读取方未能成功实现反序列化，不知道是反序列化有问题还是数据有问题。这些问题都会导致调试非常困难。如果序列化后的数据人眼可读，这将大大提高调试效率， XML和JSON就具有人眼可读的优点。

​	系列化的开销分为空间开销、时间开销。序列化需要在原有的数据上加上描述字段增加空间复杂度。复杂的序列化协议会导致较长的解析时间造成时间开销。

当下比较流行的序列化协议，包括XML、JSON、Protobuf、Thrift和Avro

在 **传输数据量较大**的需求场景下，`Protocol Buffer`比`XML、Json` 更小、更快、使用 & 维护更简单。因此适用于传输数据量大和网络环境不稳定 的**数据存储、RPC 数据交换** 的需求场景

## 数据类型

### 关键字

* required

字段只能也必须出现 1 次，多用于必填项，必须赋值的字符

```
required  int32 id = 1 [default = 123]
```

* optional

字段可出现 0 次或多次，可有可无的字段，可以使用[default = xxx]配置默认值

```
optional string name = 1 [default = "张三"]
```

* repeated

字段可出现任意多次（包括 0），多用于 Java List 属性

```
//list String
repeated string strList = 5;
//list 对象
repeated Role roleList = 6;
```

* 其它

```
/使用 proto3 语法 ,未指定则使用proto2
syntax = "proto3";

// proto 文件包名
package com.wxw.notes.protobuf.proto;

//生成 proto 文件所在包路径，一般来说是和文件包名一致就可以
option java_package = "com.wxw.notes.protobuf.proto";

//生成 proto 的文件名
option java_outer_classname="UserProto";

// 引入外部的 proto 对象
import "Role.proto";
```



*// 引入外部的 proto 对象* import "Role.proto";

### 字段编号（标识符）

1 ~ 536870911（除去 19000 到 19999 之间的标识号， Protobuf 协议实现中对这些进行了预留。如果非要在.proto 文件中使用这些预留标识号，编译时就会报警）

在消息定义中，每个字段都有唯一的一个标识符。这些标识符是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改 变。

[1,15]之内的标识号在编码的时候会占用一个字节。

[16,2047]之内的标识号则占用2个字节。所以应该尽可能为那些频繁出现的消息元素保留 [1,15]之内的标识号

### 字段类型

#### 基本类型

| protobuf 类型 | java 类型 |        默认值        |
| ------------- | --------- | :------------------: |
| double        | double    |     数值默认为0      |
| float         | float     |     数值默认为0      |
| int32         | int       |     数值默认为0      |
| int64         | long      |     数值默认为0      |
| bool          | boolean   |   bool默认为false    |
| string        | String    | string默认为空字符串 |

enum默认为第一个元素

#### 复杂类型

* list

  简单list

  ```
  //创建一个 User 对象
  message User{
  	//list Int
  	repeated int32 intList = 1;
  	//list String
  	repeated string strList = 5;
  }
  ```

  复杂list

  ```
  //创建一个 User 对象
  message User{
  	//list 对象
  	repeated Role roleList = 6;
  }
  ```

* ##### Map 

  简单map

  ```
  //创建一个 User 对象
  message User{
  	// 定义简单的 Map string
  	map<string, int32> intMap = 7;
  	// 定义复杂的 Map 对象
  	map<string, string> stringMap = 8;
  }
  ```

  复杂map

  ```
  //创建一个 User 对象
  message User{
  	// 定义复杂的 Map 对象
  	map<string, MapVauleObject> mapObject = 1;
  }
  
  // 定义 Map 的 value 对象
  message MapVauleObject {
  	string code = 1;
  	string name = 2;
  }
  ```

* 对象嵌套

  ```
  //创建一个 User 对象
  message User{
  	// 对象
  	NickName nickName = 1;
  }
  
  // 定义一个新的Name对象
  message NickName {
  	string nickName = 1;
  }
  ```

  

## 序列化示例

### 基本使用

```
<!--  protobuf 支持 Java 核心包-->
<dependency>
	<groupId>com.google.protobuf</groupId>
	<artifactId>protobuf-java</artifactId>
	<version>${protobuf-java-util.version}</version>
</dependency>
```

#### 示例1

skywalking中的jvmmetric对象

```
  long currentTimeMillis = System.currentTimeMillis();

        JVMMetric.Builder jvmBuilder = JVMMetric.newBuilder();
        jvmBuilder.setTime(currentTimeMillis);
        jvmBuilder.setCpu(CPU.newBuilder().setUsagePercent(0.98d).build());
        jvmBuilder.addAllMemory(Lists.newArrayList(Memory.newBuilder().setInit(10).setUsed(100).setIsHeap(false).build()));
        jvmBuilder.addAllMemoryPool(Lists.newArrayList(MemoryPool.newBuilder().build()));
        jvmBuilder.addAllGc(Lists.newArrayList(GC.newBuilder().build()));

        JVMMetricCollection metrics = JVMMetricCollection.newBuilder()
                                                         .setService("service")
                                                         .setServiceInstance("service-instance")
                                                         .addMetrics(jvmBuilder.build())
                                                         .build();

        handler.handle(new ConsumerRecord<>(TOPIC_NAME, 0, 0, "", Bytes.wrap(metrics.toByteArray())));
```

其中获取了JVMMetric.newBuilder()获取对象

jvmBuilder.build()表示对象进行序列化

#### 示例2

```
//序列化
UserProto.User build = user.build();
//转换成字节数组
byte[] s = build.toByteArray();
System.out.println("protobuf数据bytes[]:" + Arrays.toString(s));
System.out.println("protobuf序列化大小: " + s.length);

UserProto.User user1 = null;
String jsonObject = null;
try {
    //反序列化
    user1 = UserProto.User.parseFrom(s);
    System.out.println("反序列化：\n" + user1.toString());
    System.out.println("中文反序列化：\n" + printToUnicodeString(user1));
} catch (InvalidProtocolBufferException e) {
	e.printStackTrace();
}
```

### json互转

引入工具包

```
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java-util</artifactId>
    <version>${protobuf-java-util.version}</version>
</dependency>
```

代码摘自于skywalking

```
public static String toJSON(Message sourceMessage) throws IOException {
    return JsonFormat.printer()
                     .usingTypeRegistry(
                         JsonFormat
                             .TypeRegistry
                             .newBuilder()
                             .add(BytesValue.getDescriptor())
                             .build()
                     )
                     .print(sourceMessage);
}


public static void fromJSON(String json, Message.Builder targetBuilder) throws IOException {
    JsonFormat.parser()
              .usingTypeRegistry(
                  JsonFormat.TypeRegistry.newBuilder()
                                         .add(targetBuilder.getDescriptorForType())
                                         .build())
              .ignoringUnknownFields()
              .merge(json, targetBuilder);
}
```

### 对象互转

这里利用了gson的反序列化json字符串能力.

这里是skywalking框架源码

创建protobuffer空对象，然后将其序列化，然后利用gson反序列化成JsonElement对象，这样就和protobuffer对象一致了

```
gson.fromJson(ProtoBufJsonUtils.toJSON(Commands.newBuilder().build()), JsonElement.class)
```

从protobuffer对象中获取字符串属性字段，然后使用gson进行转换

```
String propertiesString = resultSet.getString(InstanceTraffic.PROPERTIES);
if (!Strings.isNullOrEmpty(propertiesString)) {
	JsonObject properties = GSON.fromJson(propertiesString, JsonObject.class);
```

## 协议原理

### 存储结构

#### 数值类型

#### 字符串类型

#### 嵌套类型

#### list类型

对于同一个 `repeated`字段、多个字段值来说，他们的Tag都是相同的，即数据类型 & 标识号都相同  `repeated`类型可以看成是数组

T - L - V - V - V

### 转换过程

**序列化过程如下：**  1. 判断每个字段是否有设置值，有值才进行编码  2. 根据 字段标识号&数据类型 将 字段值 通过不同的编码方式进行编码

由于：  a. 编码方式简单（只需要简单的数学运算 = 位移等等）  b. 采用 **`Protocol Buffer` 自身的框架代码 和 编译器** 共同完成

所以`Protocol Buffer`的序列化速度非常快。

**反序列化过程如下：**  1. 调用 消息类的 `parseFrom(input)` 解析从输入流读入的二进制字节数据流

>  从上面可知，`Protocol Buffer`解析过程只需要通过简单的解码方式即可完成，无需复杂的词法语法分析，因此 解析速度非常快。    2. 将解析出来的数据 按照指定的格式读取到 `Java`、`C++`、`Phyton` 对应的结构类型中。

由于：  a. 解码方式简单（只需要简单的数学运算 = 位移等等）  b. 采用 **`Protocol Buffer` 自身的框架代码 和 编译器** 共同完成

所以`Protocol Buffer`的反序列化速度非常快。

### 对比XML

XML的反序列化过程如下：  1.  从文件中读取出字符串  2. 将字符串转换为 `XML` 文档对象结构模型  3. 从 `XML` 文档对象结构模型中读取指定节点的字符串  4. 将该字符串转换成指定类型的变量

上述过程非常复杂，其中，将 `XML` 文件转换为文档对象结构模型的过程通常需要完成词法文法分析等大量消耗 CPU 的复杂计算。

因为序列化 & 反序列化过程简单，所以序列化 & 反序列化过程速度非常快，这也是 `Protocol Buffer`效率高的原因

### 存储优化

- **建议1：多用 `optional`或 `repeated`修饰符**  因为若`optional` 或 `repeated` 字段没有被设置字段值，那么该字段在序列化时的数据中是完全不存在的，即不需要进行编码  相应的字段在解码时才会被设置为默认值 
- **建议2：字段标识号（`Field_Number`）尽量只使用 1-15，且不要跳动使用**  因为`Tag`里的`Field_Number`是需要占字节空间的。如果`Field_Number`>16时，`Field_Number`的编码就会占用2个字节，那么`Tag`在编码时也就会占用更多的字节；如果将字段标识号定义为连续递增的数值，将获得更好的编码和解码性能
- **建议3：若需要使用的字段值出现负数，请使用 `sint32 / sint64`，不要使用`int32 / int64`**  因为采用`sint32 / sint64`数据类型表示负数时，会先采用`Zigzag`编码再采用`Varint`编码，从而更加有效压缩数据
- **建议4：对于`repeated`字段，尽量增加`packed=true`修饰**  因为加了`packed=true`修饰`repeated`字段采用连续数据存储方式，即`T - L - V - V -V`方式

### 小结

- `Protocol Buffer`的序列化 & 反序列化简单 & 速度快的原因是：  a.  编码 / 解码 方式简单（只需要简单的数学运算 = 位移等等）  b. 采用 `**Protocol Buffer**` **自身的框架代码 和 编译器** 共同完成
- `Protocol Buffer`的数据压缩效果好（即序列化后的数据量体积小）的原因是：  a. 采用了独特的编码方式，如`Varint`、`Zigzag`编码方式等等  b. 采用`T - L - V` 的数据存储方式：减少了分隔符的使用 & 数据存储得紧凑

## 源码分析