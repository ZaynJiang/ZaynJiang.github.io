## 1. 准备环境

* Oracle Java 8： Oracle 的 JDK 及 Hotspot JVM。

* Gradle 5.0：不支持 Gradle 6.0，因此，需要安装 **Gradle 5.x 版本。**

* Scala 2.12：当前社区编译 Kafka 支持 3 个 Scala 版本，分别是 2.11、2.12 和 2.13。2.11 应该很快就会不支持了，而 2.13 又是刚刚推出的版本，推荐安装 Scala 2.12 版本。

* IDEA + Scala 插件

* Git

* window11， 64

  

## 2. 编译步骤

* 拉取源码

   git clone https://github.com/apache/kafka.git

  切换至2.x分支

* 构建 Kafka 工程

  ```
  
  gradle
  ```

* 恢复生成windows的gradlew执行脚本

  修改wrapper.gradle文件，因为是kafka下的[wrapper](https://so.csdn.net/so/search?q=wrapper&spm=1001.2101.3001.7020).gradle配置中，删除了windows批处理文件gradlew.bat，原因是官方未在windows下进行测试构建。因此删除以下代码（在文件最后）

  ```
  task removeWindowsScript(type: Delete) {
      delete "$rootDir/gradlew.bat"
  }
  wrapper.finalizedBy removeWindowsScript
  ```

* 根目录下执行 gradle wrapper 命令 gradlew.bat文件生成
* 根目录下执行gradlew jar生成 Jar 文件

## 3. 代码结构

![](kafka源码结构.png) 

* **bin 目录**

  保存 Kafka 工具行脚本，我们熟知的 kafka-server-start 和 kafka-console-producer 等脚本都存放在这里。

* **clients 目录**

  保存 Kafka 客户端代码，比如生产者和消费者的代码都在该目录下。

* **config 目录**

  保存 Kafka 的配置文件，其中比较重要的配置文件是 server.properties。

* **connect 目录**

  保存 Connect 组件的源代码。Kafka Connect 组件是用来实现 Kafka 与外部系统之间的实时数据传输的。

* **core 目录**

  保存 Broker 端代码。Kafka 服务器端代码全部保存在该目录下。

* **streams 目录**

  保存 Streams 组件的源代码。Kafka Streams 是实现 Kafka 实时流处理的组件。

## 4. 常见命令

想要测试这 4 个部分的代码，可以分别运行以下 4 条命令：

* ./gradlew core:test

* ./gradlew clients:test

* ./gradlew connect:[submodule]:test

  Connect 组件的测试方法不太一样。这是因为 Connect 工程下细分了多个子模块，比如 api、runtime 等，所以，你需要显式地指定要测试的子模块名才能进行测试

* ./gradlew streams:test

单独对某一个具体的测试用例进行测试：

* gradlew core:test --tests kafka.log.LogTest

构建二进制环境：

* gradlew clean releaseTarGz

  成功运行后，core、clients 和 streams 目录下就会分别生成对应的二进制发布包，它们分别是：

  * kafka-2.12-2.5.0-SNAPSHOT.tgz。它是 Kafka 的 Broker 端发布包，把该文件解压之后就是标准的 Kafka 运行环境。该文件位于 core 路径的 /build/distributions 目录。
  * kafka-clients-2.5.0-SNAPSHOT.jar。该 Jar 包是 Clients 端代码编译打包之后的二进制发布包。该文件位于 clients 目录下的 /build/libs 目录。
  * kafka-streams-2.5.0-SNAPSHOT.jar。该 Jar 包是 Streams 端代码编译打包之后的二进制发布包。该文件位于 streams 目录下的 /build/libs 目录

## 5. Broker源码结构

broker核心源码在core工程下，且用scala写的

![](kafka-core源码结构.png) 

* controller 包

  保存了 Kafka 控制器（Controller）代码，而控制器组件是 Kafka 的核心组件，后面我们会针对这个包的代码进行详细分析。

* coordinator 包

  保存了**消费者端的 GroupCoordinator 代码**和**用于事务的 TransactionCoordinator 代码**。对 coordinator 包进行分析，特别是对消费者端的 GroupCoordinator 代码进行分析，是我们弄明白 Broker 端协调者组件设计原理的关键。

* log 包

  保存了 Kafka 最核心的日志结构代码，包括日志、日志段、索引文件等，专栏后面会有详细介绍。另外，该包下还封装了 Log Compaction 的实现机制，是非常重要的源码包。

* network 包

  封装了 Kafka 服务器端网络层的代码，特别是 SocketServer.scala 这个文件，是 Kafka 实现 Acceptor 模式的具体操作类，非常值得一读。

* server 包

  顾名思义，它是 Kafka 的服务器端主代码，里面的类非常多，很多关键的 Kafka 组件都存放在这里，比如专栏后面要讲到的状态机、Purgatory 延时机制等

kafka的测试用例放在 src/test 之下

## 6. scala语法入门

### 6.1. 变量

val 和 var。val 等同于 Java 中的 final 变量，一旦被初始化，就不能再被重新赋值了。相反地，var 是非 final 变量，可以重复被赋值

```
scala> val msg = "hello, world"
msg: String = hello, world
scala> msg = "another string"
<console>:12: error: reassignment to val
msg = "another string"
scala> var a:Long = 1L
a: Long = 1
scala> a = 2
a: Long = 2
```

### 6.2. 函数

```
def max(x: Int, y: Int): Int = {
    if (x > y) x
    else y
}
```

def 关键字表示这是一个函数。max 是函数名，括号中的 x 和 y 是函数输入参数，
它们都是 Int 类型的值。结尾的“Int =”组合表示 max 函数返回一个整数

```
def deleteIndicesIfExist(
    // 这里参数suffix的默认值是""，即空字符串
    // 函数结尾处的Unit类似于Java中的void关键字，表示该函数不返回任何结果
    baseFile: File, suffix: String = ""): Unit = {
    info(s"Deleting index files with suffix $suffix for baseFile $baseFile")
    val offset = offsetFromFile(baseFile)
    Files.deleteIfExists(Log.offsetIndexFile(dir, offset, suffix).toPath)
    Files.deleteIfExists(Log.timeIndexFile(dir, offset, suffix).toPath)
    Files.deleteIfExists(Log.transactionIndexFile(dir, offset, suffix).toPath)
}
```

### 6.3. 元组

接下来，我们来看下 Scala 中的元组概念。元组是承载数据的容器，一旦被创建，就不能再被更改了。元组中的数据可以是不同数据类型的。定义和访问元组的方法很简单，请看下面的代码

```
scala> val a = (1, 2.3, "hello", List(1,2,3)) // 定义一个由4个元素构成的元组，每个元
a: (Int, Double, String, List[Int]) = (1,2.3,hello,List(1, 2, 3))
scala> a._1 // 访问元组的第一个元素
res0: Int = 1
scala> a._2 // 访问元组的第二个元素
res1: Double = 2.3
```

```
def checkEnoughReplicasReachOffset(requiredOffset: Long): (Boolean, Errors) = {
......
if (minIsr <= curInSyncReplicaIds.size) {
......
(true, Errors.NONE)
} else
(false, Errors.NOT_ENOUGH_REPLICAS_AFTER_APPEND)
}
```

​	checkEnoughReplicasReachOffset 方法返回一个 (Boolean, Errors) 类型的元组，即元组的第一个元素或字段是 Boolean 类型，第二个元素是 Kafka 自定义的 Errors 类型。

​	该方法会判断某分区 ISR 中副本的数量，是否大于等于所需的最小 ISR 副本数，如果是，就返回（true, Errors.NONE）元组，否则返回（false，Errors.NOT_ENOUGH_REPLICAS_AFTER_APPEND）

### 6.4. 循环

下面我们来看下 Scala 中循环的写法。我们常见的循环有两种写法：命令式编程方式和函
数式编程方式

```
scala> val list = List(1, 2, 3, 4, 5)
list: List[Int] = List(1, 2, 3, 4, 5)
scala> for (element <- list) println(element)
```

```
dataPlaneAcceptors.asScala.values.foreach(_.initiateShutdown())
```

​	这一行代码首先调用 asScala 方法，将 Java 的 ConcurrentHashMap 转换成 Scala 语言中的 concurrent.Map 对象；然后获取它保存的所有 Acceptor 线程，通过 foreach 循环，调用每个 Acceptor 对象的 initiateShutdown 方法。如果这个逻辑用命令式编程来实现，至少要几行甚至是十几行才能完成

### 6.5. case 类

Case 类非常适合用来表示不可变数据。同时，它最有用的一个特点是，case 类自动地为所有类字段定
义 Getter 方法

```
case class Point(x:Int, y: Int) // 默认写法。不能修改x和y
case class Point(var x: Int, var y: Int) // 支持修改x和y
```

### 6.6. 模式匹配

```
def describe(x: Any) = x match {
    case 1 => "one"
    case false => "False"
    case "hi" => "hello, world!"
    case Nil => "the empty list"
    case e: IOException => "this is an IOException"
    case s: String if s.length > 10 => "a long string"
    case _ => "something else"
}
```

​	这个函数的 x 是 Any 类型，这相当于 Java 中的 Object 类型，即所有类的父类。注意倒数第二行的“case _”的写法，它是用来兜底的。如果上面的所有 case 分支都不匹配，那就进入到这个分支。另外，它还支持一些复杂的表达式，比如倒数第三行的 case 分支，表示x 是字符串类型，而且 x 的长度超过 10 的话，就进入到这个分支

### 6.7. Option 对象

​	java 也引入了类似的类：Optional。不论是 Scala 中的Option，还是 Java 中的 Optional，都是用来帮助我们更好地规避 NullPointerException异常的

```
scala> val keywords = Map("scala" -> "option", "java" -> "optional") // 创建一个
keywords: scala.collection.immutable.Map[String,String] = Map(scala -> option, scala> keywords.get("java") // 获取key值为java的value值。由于值存在故返回Some(option
res24: Option[String] = Some(optional)
scala> keywords.get("C") // 获取key值为C的value值。由于不存在故返回None
res23: Option[String] = None
```

​	Option 表示一个容器对象，里面可能装了值，也可能没有装任何值。由于是容器，因此一般都是这样的写法：Option[Any]。中括号里面的 Any 就是上面说到的 Any 类型，它能是任何类型。如果值存在的话，就可以使用 Some(x) 来获取值或给值赋值，否则就使用None 来表示

​	Option 对象还经常与模式匹配语法一起使用，以实现不同情况下的处理逻辑。比如，Option 对象有值和没有值时分别执行什么代码

```
def display(game: Option[String]) = game match {
    case Some(s) => s
    case None => "unknown"
}
scala> display(Some("Heroes 3"))
res26: String = Heroes 3
scala> display(Some("StarCraft"))
res27: String = StarCraft
scala> display(None)
res28: String = unknown
```



## 7. 总结