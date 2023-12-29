## 指标统计实现

### NodeSelectorSlot

NodeSelectorSlot 负责为资源的首次访问创建 DefaultNode，以及维护 Context.curNode 和调用树。NodeSelectorSlot 被放在 ProcessorSlotChain 链表的第一个位置，这是因为后续的 ProcessorSlot 都需要依赖这个 ProcessorSlot。NodeSelectorSlot 源码如下：

```java
public class NodeSelectorSlot extends AbstractLinkedProcessorSlot<Object> {
    // Context 的 name -> 资源的 DefaultNode
    private volatile Map<String, DefaultNode> map = new HashMap<>(10);
    // 入口方法
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args) throws Throwable {
        // 使用 Context 的名称作为 key 缓存资源的 DefaultNode
        DefaultNode node = map.get(context.getName());
        if (node == null) {
            synchronized (this) {
                node = map.get(context.getName());
                if (node == null) {
                    // 为资源创建 DefaultNode
                    node = new DefaultNode(resourceWrapper, null);
                    // 替换 map
                    HashMap<String, DefaultNode> cacheMap = new HashMap<>(map.size());
                    cacheMap.putAll(map);
                    cacheMap.put(context.getName(), node);
                    map = cacheMap;
                    // 绑定调用树
                    ((DefaultNode) context.getLastNode()).addChild(node);
                }
            }
        }
        // 替换 Context 的 curNode 为当前 DefaultNode
        context.setCurNode(node);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    // 出口方法什么也不做
    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        fireExit(context, resourceWrapper, count, args);
    }
}
```

* 如源码所示，map 字段是一个非静态字段，意味着每个 NodeSelectorSlot 都有一个 map。由于一个资源对应一个 ProcessorSlotChain，而一个 ProcessorSlotChain 只创建一个 NodeSelectorSlot，并且 map 缓存 DefaultNode 使用的 key 并非资源 ID，而是 Context.name，所以 map 的作用是缓存针对同一资源为不同调用链路入口创建的 DefaultNode。
* 在 entry 方法中，首先根据 Context.name 从 map 获取当前调用链路入口的资源 DefaultNode，如果资源第一次被访问，也就是资源的 ProcessorSlotChain 第一次被创建，那么这个 map 是空的，就会加锁为资源创建 DefaultNode，如果资源不是首次被访问，但却首次作为当前调用链路（Context）的入口资源，也需要加锁为资源创建一个 DefaultNode。

**可见，Sentinel 会为同一资源 ID 创建多少个 DefaultNode 取决于有多少个调用链使用其作为入口资源，直白点就是同一资源存在多少个 DefaultNode 取决于 Context.name 有多少种不同取值，这就是为什么说一个资源可能有多个 DefaultNode 的原因**

举个例子，对同一支付接口，我们需要使用 Spring MVC 暴露给前端访问，同时也可能会使用 Dubbo 暴露给其它内部服务调用。Sentinel 的 Web MVC 适配器在调用链路入口创建名为“sentinel_spring_web_context”的 Context，与 Sentinel 的 Dubbo 适配器调用 ContextUtil#enter 方法创建的 Context 名称不同。针对这种情况，我们可以实现只限制 Spring MVC 进来的流量，也就是限制前端发起接口调用的 QPS、并行占用的线程数等。

NodeSelectorSlot#entry 方法最难以理解的就是实现绑定调用树这行代码：

```java
((DefaultNode) context.getLastNode()).addChild(node);
```

这行代码分两种情况分析更容易理解，我们就以 Sentinel 提供的 Demo 为例进行分析

#### 单多次 SphU#entry

Sentinel 的 sentinel-demo 模块下提供了多种使用场景的 Demo，我们选择 sentinel-demo-spring-webmvc 这个 Demo 为例，该 Demo 下有一个 hello 接口，其代码如下

```
@RestController
public class WebMvcTestController {

    @GetMapping("/hello")
    public String apiHello() throws BlockException {
        doBusiness();
        return "Hello!";
    }
}
```

我们不需要添加任何规则，只是为了调试 Sentinel 的源码。将 demo 启动起来后，在浏览器访问“/hello”接口，在 NodeSelectorSlot#entry 方法的绑定调用树这一行代码下断点，观察此时 Context 的字段信息。正常情况下我们可以看到如下图所示的结果

![image-20231228141858164](image-20231228141858164.png) 

从上图中可以看出，此时的 Context.entranceNode 的子节点为空（childList 的大小为 0），并且当前 CtEntry 父、子节点都是 Null（curEntry 字段）。当绑定调用树这一行代码执行完成后，Context 的字段信息如下图所示：

![image-20231228142007957](image-20231228142007957.png) 

从上图可以看出，NodeSelectorSlot 为当前资源创建的 DefaultNode 被添加到了 Context.entranceNode 的子节点。entranceNode 类型为 EntranceNode，在调用 ContextUtil#enter 方法时创建，在第一次创建名为“sentinel_spring_web_context”的 Context 时创建，相同名称的 Context 都使用同一个 EntranceNode。并且该 EntranceNode 在创建时会被添加到 Constant.ROOT。

此时，Constant.ROOT、Context.entranceNode、当前访问资源的 DefaultNode 构造成的调用树如下：

```java
           ROOT (machine-root)
                /
      EntranceNode (context name: sentinel_spring_web_context)
             /
DefaultNode （resource name: GET:/hello）
```

如果我们现在再访问 Demo 的其他接口，例如访问“/err”接口，那么生成的调用树就会变成如下：

```java
                        ROOT (machine-root)
                            /
      EntranceNode (context name: sentinel_spring_web_context)
                    /                                \
DefaultNode （resource name: GET:/hello）     DefaultNode （resource name: GET:/err） 
```

Context.entranceNode 将会存储 Web 项目的所有资源（接口）的 DefaultNode

#### **多次 SphU#entry**

比如我们在一个服务中添加了 Sentinel 的 Web MVC 适配模块的依赖，也添加了 Sentinel 的 OpenFeign 适配模块的依赖，并且我们使用 OpenFeign 调用内部其他服务的接口，那么就会存在一次调用链路上出现多次调用 SphU#entry 方法的情况。

首先 webmvc 适配器在接收客户端请求时会调用一次 SphU#entry，在处理客户端请求时可能需要使用 OpenFeign 调用其它服务的接口，那么在发起接口调用时，Sentinel 的 OpenFeign 适配器也会调用一次 SphU#entry。

现在我们将 Demo 的 hello 接口修改一下，将 hello 接口调用的 doBusiness 方法也作为资源使用 Sentinel 保护起来，改造后的 hello 接口代码如下：

```java
@RestController
public class WebMvcTestController {

    @GetMapping("/hello")
    public String apiHello() throws BlockException {
        ContextUtil.enter("my_context");
        Entry entry = null;
        try {
            entry = SphU.entry("POST:http://wujiuye.com/hello2", EntryType.OUT);
            // ==== 这里是被包装的代码 =====
            doBusiness();
            return "Hello!";
            // ==== end ===============
        } catch (Exception e) {
            if (!(e instanceof BlockException)) {
                Tracer.trace(e);
            }
            throw e;
        } finally {
            if (entry != null) {
                entry.exit(1);
            }
            ContextUtil.exit();
        }
    }
}
```

我们可将 doBusiness 方法看成是远程调用，例如调用第三方的接口，接口名称为“http://wujiuye.com/hello2”，使用 POST 方式调用，那么我们可以使用“POST:http://wujiuye.com/hello2”作为资源名称，并将流量类型设置为 OUT 类型。上下文名称取名为”my_context”

现在启动 demo，使用浏览器访问“/hello”接口。当代码执行到 apiHello 方法时，在 NodeSelectorSlot#entry 方法的绑定调用树这一行代码下断点。当绑定调用树这行代码执行完成后，Context 的字段信息如下图所示：

![image-20231228145210500](image-20231228145210500.png) 

如图所示，Sentinel 并没有创建名称为 my_context 的 Context，还是使用应用接收到请求时创建名为“sentinel_spring_web_context”的 Context，所以处理浏览器发送过来的请求的“GET:/hello”资源是本次调用链路的入口资源，Sentinel 在调用链路入口处创建 Context 之后不再创建新的 Context。

由于之前并没有为名称为“POST:http://wujiuye.com/hello2”的资源创建 ProcessorSlotChain，所以 SphU#entry 会为该资源创建一个 ProcessorSlotChain，也就会为该 ProcessorSlotChain 创建一个 NodeSelectorSlot。在执行到 NodeSelectorSlot#entry 方法时，就会为该资源创建一个 DefaultNode，而将该资源的 DefaultNode 绑定到节点树后，该资源的 DefaultNode 就会成为“GET:/hello”资源的 DefaultNode 的子节点，调用树如下。

```java
                    ROOT (machine-root)
                    /
    EntranceNode (name: sentinel_spring_web_context)
                 /                       \
          DefaultNode （GET:/hello）   .........
               /
         DefaultNode  (POST:/hello2)
```

此时，当前调用链路上也已经存在两个 CtEntry，这两个 CtEntry 构造一个双向链表，如下图所示：

![image-20231228145546934](image-20231228145546934.png) 

虽然存在两个 CtEntry，但此时 Context.curEntry 指向第二个 CtEntry，第二个 CtEntry 在 apiHello 方法中调用 SphU#entry 方法时创建，当执行完 doBusiness 方法后，调用当前 CtEntry#exit 方法，由该 CtEntry 将 Context.curEntry 还原为该 CtEntry 的父 CtEntry。这有点像入栈和出栈操作，例如栈帧在 Java 虚拟机栈的入栈和出栈，调用方法时方法的栈帧入栈，方法执行完成栈帧出栈。

NodeSelectorSlot#entry 方法我们还有一行代码没有分析，就是将当前创建的 DefaultNode 设置为 Context 的当前节点，代码如下：

```java
// 替换 Context.curNode 为当前 DefaultNode
context.setCurNode(node);
```

替换 Context.curNode 为当前资源 DefaultNode 这行代码就是将当前创建的 DefaultNode 赋值给当前 CtEntry.curNode。对着上图理解就是，将资源“GET:/hello”的 DefaultNode 赋值给第一个 CtEntry.curNode，将资源“POST:http://wujiuye.com/hello2”的 DefaultNode 赋值给第二个 CtEntry.curNode。

要理解 Sentinel 构造 CtEntry 双向链表的目的，首先我们需要了解调用 Context#getCurNode 方法获取当前资源的 DefaultNode 可以做什么。

Tracer#tracer 方法用于记录异常。以异常指标数据统计为例，在发生非 Block 异常时，Tracer#tracer 需要从 Context 获取当前资源的 DefaultNode，通知 DefaultNode 记录异常，同时 DefaultNode 也会通知 ClusterNode 记录记录，如下代码所示。

```java
public class DefaultNode extends StatisticNode {
  ......
  @Override
    public void increaseExceptionQps(int count) {
        super.increaseExceptionQps(count);
        this.clusterNode.increaseExceptionQps(count);
    }
}
```

这个例子虽然简单，但也足以说明 Sentinel 构造 CtEntry 双向链表的目的

### ClusterBuilderSlot

#### **ClusterNode 出现的背景**

在一个资源的 ProcessorSlotChain 中，NodeSelectorSlot 负责为资源创建 DefaultNode，这个 DefaultNode 仅限同名的 Context 使用。所以一个资源可能会存在多个 DefaultNode，那么想要获取一个资源的总的 QPS 就必须要遍历这些 DefaultNode。为了性能考虑，Sentinel 会为每个资源创建一个全局唯一的 ClusterNode，用于统计资源的全局并行占用线程数、QPS、异常总数等指标数据。

#### **ClusterBuilderSlot**

与 NodeSelectorSlot 的职责相似，ClusterBuilderSlot 的职责是为资源创建全局唯一的 ClusterNode，仅在资源第一次被访问时创建。ClusterBuilderSlot 还会将 ClusterNode 赋值给 DefaultNode.clusterNode，由 DefaultNode 持有 ClusterNode，负责管理 ClusterNode 的指标数据统计。这点也是 ClusterBuilderSlot 在 ProcessorSlotChain 链表中必须排在 NodeSelectorSlot 之后的原因，即必须先有 DefaultNode，才能将 ClusterNode 交给 DefaultNode 管理。

ClusterBuilderSlot 的源码比较多，本篇只分析其实现 ProcessorSlot 接口的 entry 和 exit 方法。ClusterBuilderSlot 删减后的源码如下。

```java
public class ClusterBuilderSlot extends AbstractLinkedProcessorSlot<DefaultNode> {
    // 资源 -> ClusterNode
    private static volatile Map<ResourceWrapper, ClusterNode> clusterNodeMap = new HashMap<>();
    private static final Object lock = new Object();

    // 非静态，一个资源对应一个 ProcessorSlotChain，所以一个资源共用一个 ClusterNode
    private volatile ClusterNode clusterNode = null;

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args)
            throws Throwable {
        if (clusterNode == null) {
            synchronized (lock) {
                if (clusterNode == null) {
                    // 创建 ClusterNode
                    clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                    // 添加到缓存
                    HashMap<ResourceWrapper, ClusterNode> newMap = new HashMap<>(Math.max(clusterNodeMap.size(), 16));
                    newMap.putAll(clusterNodeMap);
                    newMap.put(node.getId(), clusterNode);
                    clusterNodeMap = newMap;
                }
            }
        }
        // node 为 NodeSelectorSlot 传递过来的 DefaultNode
        node.setClusterNode(clusterNode);
        // 如果 origin 不为空，则为远程创建一个 StatisticNode
        if (!"".equals(context.getOrigin())) {
            Node originNode = node.getClusterNode().getOrCreateOriginNode(context.getOrigin());
            context.getCurEntry().setOriginNode(originNode);
        }
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        fireExit(context, resourceWrapper, count, args);
    }
}
```

ClusterBuilderSlot 使用一个 Map 缓存资源的 ClusterNode，并且用一个非静态的字段维护当前资源的 ClusterNode。因为一个资源只会创建一个 ProcessorSlotChain，意味着 ClusterBuilderSlot 也只会创建一个，那么让 ClusterBuilderSlot 持有该资源的 ClusterNode 就可以省去每次都从 Map 中获取的步骤，这当然也是 Sentinel 为性能做出的努力。

ClusterBuilderSlot#entry 方法的 node 参数由前一个 ProcessorSlot 传递过来，也就是 NodeSelectorSlot 传递过来的 DefaultNode。ClusterBuilderSlot 将 ClusterNode 赋值给 DefaultNode.clusterNode，那么后续的 ProcessorSlot 就能从 node 参数中取得 ClusterNode。DefaultNode 与 ClusterNode 的关系如下图所示。

![image-20231228150157934](image-20231228150157934.png) 

ClusterNode 有一个 Map 类型的字段用来缓存 origin 与 StatisticNode 的映射，代码如下：

```java
public class ClusterNode extends StatisticNode {
    private final String name;
    private final int resourceType;
    private Map<String, StatisticNode> originCountMap = new HashMap<>();
}
```

如果上游服务在调用当前服务的接口传递 origin 字段过来，例如可在 http 请求头添加“S-user”参数，或者 Dubbo rpc 调用在请求参数列表加上“application”参数，那么 ClusterBuilderSlot 就会为 ClusterNode 创建一个 StatisticNode，用来统计当前资源被远程服务调用的指标数据。

例如，当 origin 表示来源应用的名称时，对应的 StatisticNode 统计的就是针对该调用来源的指标数据，可用来查看哪个服务访问这个接口最频繁，由此可实现按调用来源限流。

ClusterNode#getOrCreateOriginNode 方法源码如下：

```java
   public Node getOrCreateOriginNode(String origin) {
        StatisticNode statisticNode = originCountMap.get(origin);
        if (statisticNode == null) {
            try {
                lock.lock();
                statisticNode = originCountMap.get(origin);
                if (statisticNode == null) {
                    statisticNode = new StatisticNode();
                    // 这几行代码在 Sentinel 中随处可见
                    HashMap<String, StatisticNode> newMap = new HashMap<>(originCountMap.size() + 1);
                    newMap.putAll(originCountMap);
                    newMap.put(origin, statisticNode);
                    originCountMap = newMap;
                }
            } finally {
                lock.unlock();
            }
        }
        return statisticNode;
    }
```

为了便于使用，ClusterBuilderSlot 会将调用来源（origin）的 StatisticNode 赋值给 Context.curEntry.originNode，后续的 ProcessorSlot 可调用 Context#getCurEntry#getOriginNode 方法获取该 StatisticNode。这里我们可以得出一个结论，如果我们自定义的 ProcessorSlot 需要用到调用来源的 StatisticNode，那么在构建 ProcessorSlotChain 时，我们必须要将这个自定义 ProcessorSlot 放在 ClusterBuilderSlot 之后。

### StatisticSlot

StatisticSlot 才是实现资源各项指标数据统计的 ProcessorSlot，它与 NodeSelectorSlot、ClusterBuilderSlot 组成了资源指标数据统计流水线，分工明确。

首先 NodeSelectorSlot 为资源创建 DefaultNode，将 DefaultNode 向下传递，ClusterBuilderSlot 负责给资源的 DefaultNode 加工，添加 ClusterNode 这个零部件，再将 DefaultNode 向下传递给 StatisticSlot，如下图所示：

![image-20231228153251098](image-20231228153251098.png) 

StatisticSlot 在统计指标数据之前会先调用后续的 ProcessorSlot，根据后续 ProcessorSlot 判断是否需要拒绝该请求的结果决定记录哪些指标数据，这也是为什么 Sentinel 设计的责任链需要由前一个 ProcessorSlot 在 entry 或者 exit 方法中调用 fireEntry 或者 fireExit 完成调用下一个 ProcessorSlot 的 entry 或 exit 方法，而不是使用 for 循环遍历调用 ProcessorSlot 的原因。每个 ProcessorSlot 都有权决定是先等后续的 ProcessorSlot 执行完成再做自己的事情，还是先完成自己的事情再让后续 ProcessorSlot 执行，与流水线有所区别

StatisticSlot 源码框架如下：

```
public class StatisticSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        try {
            // Do some checking.
            fireEntry(context, resourceWrapper, node, count, prioritized, args);
           // .....
        } catch (PriorityWaitException ex) {
            // .....
        } catch (BlockException e) {
            // ....
            throw e;
        } catch (Throwable e) {
            // .....
            throw e;
        }
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        DefaultNode node = (DefaultNode)context.getCurNode();
        // ....
        fireExit(context, resourceWrapper, count);
    }
}

```

- entry：先调用 fireEntry 方法完成调用后续的 ProcessorSlot#entry 方法，根据后续的 ProcessorSlot 是否抛出 BlockException 决定记录哪些指标数据，并将资源并行占用的线程数加 1。
- exit：若无任何异常，则记录响应成功、请求执行耗时，将资源并行占用的线程数减 1

#### 放行请求

第一种情况：当后续的 ProcessorSlot 未抛出任何异常时，表示不需要拒绝该请求，放行当前请求

当请求可正常通过时，需要将当前资源并行占用的线程数增加 1、当前时间窗口被放行的请求总数加 1，代码如下：

```java
            // Request passed, add thread count and pass count.
            node.increaseThreadNum();
            node.addPassRequest(count);
```

如果调用来源不为空，也将调用来源的 StatisticNode 的当前并行占用线程数加 1、当前时间窗口被放行的请求数加 1，代码如下：

```java
            if (context.getCurEntry().getOriginNode() != null) {
                // Add count for origin node.
                context.getCurEntry().getOriginNode().increaseThreadNum();
                context.getCurEntry().getOriginNode().addPassRequest(count);
            }
```

如果流量类型为 IN，则将资源全局唯一的 ClusterNode 的并行占用线程数、当前时间窗口被放行的请求数都增加 1，代码如下：

```java
           if (resourceWrapper.getEntryType() == EntryType.IN) {
                // Add count for global inbound entry node for global statistics.
                Constants.ENTRY_NODE.increaseThreadNum();
                Constants.ENTRY_NODE.addPassRequest(count);
            }
```

回调所有 ProcessorSlotEntryCallback#onPass 方法，代码如下：

```java
            // Handle pass event with registered entry callback handlers.
            for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
                handler.onPass(context, resourceWrapper, node, count, args);
            }
```

可调用 StatisticSlotCallbackRegistry#addEntryCallback 静态方法注册 ProcessorSlotEntryCallback，ProcessorSlotEntryCallback 接口的定义如下：

```java
public interface ProcessorSlotEntryCallback<T> {
    void onPass(Context context, ResourceWrapper resourceWrapper, T param, int count, Object... args) throws Exception;
    void onBlocked(BlockException ex, Context context, ResourceWrapper resourceWrapper, T param, int count, Object... args);
}
```

- onPass：该方法在请求被放行时被回调执行。
- onBlocked：该方法在请求被拒绝时被回调执行

#### PriorityWaitException异常

这是特殊情况，在需要对请求限流时，只有使用默认流量效果控制器才可能会抛出 PriorityWaitException 异常，这部分内容将在分析 FlowSlot 的实现源码时再作分析。

当捕获到 PriorityWaitException 异常时，说明当前请求已经被休眠了一会了，但请求还是允许通过的，只是不需要为 DefaultNode 记录这个请求的指标数据了，只自增当前资源并行占用的线程数，同时，DefaultNode 也会为 ClusterNode 自增并行占用的线程数。最后也会回调所有 ProcessorSlotEntryCallback#onPass 方法。这部分源码如下

```java
            node.increaseThreadNum();
            if (context.getCurEntry().getOriginNode() != null) {
                // Add count for origin node.
                context.getCurEntry().getOriginNode().increaseThreadNum();
            }
            if (resourceWrapper.getEntryType() == EntryType.IN) {
                // Add count for global inbound entry node for global statistics.
                Constants.ENTRY_NODE.increaseThreadNum();
            }
            // Handle pass event with registered entry callback handlers.
            for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
                handler.onPass(context, resourceWrapper, node, count, args);
            }
```

#### BlockException 异常

捕获到 BlockException 异常，BlockException 异常只在需要拒绝请求时抛出

当捕获到 BlockException 异常时，将异常记录到调用链路上下文的当前 Entry（StatisticSlot 的 exit 方法会用到），然后调用 DefaultNode#increaseBlockQps 方法记录当前请求被拒绝，将当前时间窗口的 block qps 这项指标数据的值加 1。如果调用来源不为空，让调用来源的 StatisticsNode 也记录当前请求被拒绝；如果流量类型为 IN，则让用于统计所有资源指标数据的 ClusterNode 也记录当前请求被拒绝。这部分的源码如下：

```java
            // Blocked, set block exception to current entry.
            context.getCurEntry().setError(e);

            // Add block count.
            node.increaseBlockQps(count);
            if (context.getCurEntry().getOriginNode() != null) {
                context.getCurEntry().getOriginNode().increaseBlockQps(count);
            }

            if (resourceWrapper.getEntryType() == EntryType.IN) {
                // Add count for global inbound entry node for global statistics.
                Constants.ENTRY_NODE.increaseBlockQps(count);
            }

            // Handle block event with registered entry callback handlers.
            for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
                handler.onBlocked(e, context, resourceWrapper, node, count, args);
            }

            throw e;
```

StatisticSlot 捕获 BlockException 异常只是为了收集被拒绝的请求，BlockException 异常还是会往上抛出。抛出异常的目的是为了拦住请求，让入口处能够执行到 catch 代码块完成请求被拒绝后的服务降级处理

#### 其它异常

其它异常并非指业务异常，因为此时业务代码还未执行，而业务代码抛出的异常是通过调用 Tracer#trace 方法记录的。

当捕获到非 BlockException 异常时，除 PriorityWaitException 异常外，其它类型的异常都同样处理。让 DefaultNode 记录当前请求异常，将当前时间窗口的 exception qps 这项指标数据的值加 1。调用来源的 StatisticsNode、用于统计所有资源指标数据的 ClusterNode 也记录下这个异常。这部分源码如下：

```java
           // Unexpected error, set error to current entry.
            context.getCurEntry().setError(e);

            // This should not happen.
            node.increaseExceptionQps(count);
            if (context.getCurEntry().getOriginNode() != null) {
                context.getCurEntry().getOriginNode().increaseExceptionQps(count);
            }

            if (resourceWrapper.getEntryType() == EntryType.IN) {
                Constants.ENTRY_NODE.increaseExceptionQps(count);
            }
            throw e;
```

#### **exit 方法**

exit 方法被调用时，要么请求被拒绝，要么请求被放行并且已经执行完成，所以 exit 方法需要知道当前请求是否正常执行完成，这正是 StatisticSlot 在捕获异常时将异常记录到当前 Entry 的原因，exit 方法中通过 Context 可获取到当前 CtEntry，从当前 CtEntry 可获取 entry 方法中写入的异常。

exit 方法源码如下（有删减）：

```java
@Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        DefaultNode node = (DefaultNode)context.getCurNode();
        if (context.getCurEntry().getError() == null) {
            // 计算耗时
            long rt = TimeUtil.currentTimeMillis() - context.getCurEntry().getCreateTime();
            // 记录执行耗时与成功总数
            node.addRtAndSuccess(rt, count);
            if (context.getCurEntry().getOriginNode() != null) {
                context.getCurEntry().getOriginNode().addRtAndSuccess(rt, count);
            }
            // 自减当前资源占用的线程数
            node.decreaseThreadNum();
            // origin 不为空
            if (context.getCurEntry().getOriginNode() != null) {
                context.getCurEntry().getOriginNode().decreaseThreadNum();
            }
            // 流量类型为 in 时
            if (resourceWrapper.getEntryType() == EntryType.IN) {
                Constants.ENTRY_NODE.addRtAndSuccess(rt, count);
                Constants.ENTRY_NODE.decreaseThreadNum();
            }
        }
        // Handle exit event with registered exit callback handlers.
        Collection<ProcessorSlotExitCallback> exitCallbacks = StatisticSlotCallbackRegistry.getExitCallbacks();
        for (ProcessorSlotExitCallback handler : exitCallbacks) {
            handler.onExit(context, resourceWrapper, count, args);
        }
        fireExit(context, resourceWrapper, count);
    }
```

exit 方法中通过 Context 可获取当前资源的 DefaultNode，如果 entry 方法中未出现异常，那么说明请求是正常完成的，在请求正常完成情况下需要记录请求的执行耗时以及响应是否成功，可将当前时间减去调用链路上当前 Entry 的创建时间作为请求的执行耗时

#### 指标数据添加

ClusterNode 才是一个资源全局的指标数据统计节点，但我们并未在 StatisticSlot#entry 方法与 exit 方法中看到其被使用。因为 ClusterNode 被 ClusterBuilderSlot 交给了 DefaultNode 掌管，在 DefaultNode 的相关指标数据收集方法被调用时，ClusterNode 的对应方法也会被调用，如下代码所示：

```java
public class DefaultNode extends StatisticNode {
   ......
    private ClusterNode clusterNode;

    @Override
    public void addPassRequest(int count) {
        super.addPassRequest(count);
        this.clusterNode.addPassRequest(count);
    }
}
```

记录某项指标数据指的是：针对当前请求，记录当前请求的某项指标数据，例如请求被放行、请求被拒绝、请求的执行耗时等。

假设当前请求被成功处理，StatisticSlot 会调用 DefaultNode#addRtAndSuccess 方法记录请求处理成功、并且记录处理请求的耗时，DefaultNode 先调用父类的 addRtAndSuccess 方法，然后 DefaultNode 会调用 ClusterNode#addRtAndSuccess 方法。ClusterNode 与 DefaultNode 都是 StatisticNode 的子类，StatisticNode#addRtAndSuccess 方法源码如下：

```java
    @Override
    public void addRtAndSuccess(long rt, int successCount) {
        // 秒级滑动窗口
        rollingCounterInSecond.addSuccess(successCount);
        rollingCounterInSecond.addRT(rt);
        // 分钟级的滑动窗口
        rollingCounterInMinute.addSuccess(successCount);
        rollingCounterInMinute.addRT(rt);
    }
```

rollingCounterInSecond 是一个秒级的滑动窗口，rollingCounterInMinute 是一个分钟级的滑动窗口，类型为 ArrayMetric。分钟级的滑动窗口一共有 60 个 MetricBucket，每个 MetricBucket 都被 WindowWrap 包装，每个 MetricBucket 统计一秒钟内的各项指标数据，如下图所示：

![image-20231229100557706](image-20231229100557706.png) 

当调用 rollingCounterInMinute#addSuccess 方法时，由 ArrayMetric 根据当前时间戳获取当前时间窗口的 MetricBucket，再调用 MetricBucket#addSuccess 方法将 success 这项指标的值加上方法参数传递进来的值（一般是 1）。MetricBucket 使用 LongAdder 记录各项指标数据的值。

Sentinel 在 MetricEvent 枚举类中定义了 Sentinel 会收集哪些指标数据，MetricEvent 枚举类的源码如下：

```java
public enum MetricEvent {
    PASS,
    BLOCK,
    EXCEPTION,
    SUCCESS,
    RT,
    OCCUPIED_PASS
}
```

- pass 指标：请求被放行的总数
- block：请求被拒绝的总数
- exception：请求处理异常的总数
- success：请求被处理成功的总数
- rt：被处理成功的请求的总耗时
- occupied_pass：预通过总数（前一个时间窗口使用了当前时间窗口的 passQps）

其它的指标数据都可通过以上这些指标数据计算得出，例如，平均耗时可根据总耗时除以成功总数计算得出

### 总结

- 一个调用链路上只会创建一个 Context，在调用链路的入口创建（一个调用链路上第一个被 Sentinel 保护的资源）。
- 一个 Context 名称只创建一个 EntranceNode，也是在调用链路的入口创建，调用 Context#enter 方法时创建。
- 与方法调用的入栈出栈一样，一个线程上调用多少次 SphU#entry 方法就会创建多少个 CtEntry，前一个 CtEntry 作为当前 CtEntry 的父节点，当前 CtEntry 作为前一个 CtEntry 的子节点，构成一个双向链表。Context.curEntry 保存的是当前的 CtEntry，在调用当前的 CtEntry#exit 方法时，由当前 CtEntry 将 Context.curEntry 还原为当前 CtEntry 的父节点 CtEntry。
- 一个调用链路上，如果多次调用 SphU#entry 方法传入的资源名称都相同，那么只会创建一个 DefaultNode，如果资源名称不同，会为每个资源名称创建一个 DefaultNode，当前 DefaultNode 会作为调用链路上的前一个 DefaultNode 的子节点。
- 一个资源有且只有一个 ProcessorSlotChain，一个资源有且只有一个 ClusterNode。
- 一个 ClusterNode 负责统计一个资源的全局指标数据。
- StatisticSlot 负责记录请求是否被放行、请求是否被拒绝、请求是否处理异常、处理请求的耗时等指标数据，在 StatisticSlot 调用 DefaultNode 用于记录某项指标数据的方法时，DefaultNode 也会调用 ClusterNode 的相对应方法，完成两份指标数据的收集。
- DefaultNode 统计当前资源的各项指标数据的维度是同一个 Context（名称相同），而 ClusterNode 统计当前资源各项指标数据的维度是全局。

## 限流实现

ProcessorSlot 检查实时指标数据是否达到规则所配置的阈值，当达到阈值时，或抛出 Block 异常或采取流量效果控制策略处理超阈值的流量。

Sentinel 实现限流降级、熔断降级、黑白名单限流降级、系统自适应限流降级以及热点参数限流降级都是由 ProcessorSlot、Checker、Rule、RuleManager 组合完成。ProcessorSlot 作为调用链路的切入点，负责调用 Checker 检查当前请求是否可以放行；Checker 则根据资源名称从 RuleManager 中拿到为该资源配置的 Rule（规则），取 ClusterNode 统计的实时指标数据与规则对比，如果达到规则的阈值则抛出 Block 异常，抛出 Block 异常意味着请求被拒绝，也就实现了限流或熔断。

可以总结为以下三个步骤：

- 在 ProcessorSlot#entry 方法中调用 Checker#check 方法，并将 DefaultNode 传递给 Checker。
- Checker 从 DefaultNode 拿到 ClusterNode，并根据资源名称从 RuleManager 获取为该资源配置的规则。
- Checker 从 ClusterNode 中获取当前时间窗口的某项指标数据（QPS、avgRt 等）与规则的阈值对比，如果达到规则的阈值则抛出 Block 异常（也有可能将 check 交给 Rule 去实现）

### 限流规则

Sentinel 在最初的框架设计上，将是否允许请求通过的判断行为交给 Rule 去实现，所以将 Rule 定义成了接口。Rule 接口只定义了一个 passCheck 方法，即判断当前请求是否允许通过。Rule 接口的定义如下：

```java
public interface Rule {
    boolean passCheck(Context context, DefaultNode node, int count, Object... args);
}
```

- context：当前调用链路上下文。
- node：当前资源的 DefaultNode。
- count：一般为 1，用在令牌桶算法中表示需要申请的令牌数，用在 QPS 统计中表示一个请求。
- args：方法调用参数（被 Sentinel 拦截的目标方法），用于实现热点参数限流降级的。

因为规则是围绕资源配置的，一个规则只对某个资源起作用，因此 Sentinel 提供了一个抽象规则配置类 AbstractRule，AbstractRule 的定义如下：

```java
public abstract class AbstractRule implements Rule {
    private String resource;
    private String limitApp;
    // ....
}
```

- resource：资源名称，规则的作用对象。
- limitApp：只对哪个或者哪些调用来源生效，若为 default 则不区分调用来源。

Rule、AbstractRule 与其它实现类的关系如下图所示：

![image-20231229103244763](image-20231229103244763.png) 

FlowRule 是限流规则配置类，FlowRule 继承 AbstractRule 并实现 Rule 接口。FlowRule 源码如下，非完整源码，与实现集群限流相关的字段暂时去掉了

```java
public class FlowRule extends AbstractRule {
    // 限流阈值类型 qps|threads
    private int grade = RuleConstant.FLOW_GRADE_QPS;
    // 限流阈值
    private double count;
    // 基于调用关系的限流策略
    private int strategy = RuleConstant.STRATEGY_DIRECT;
    // 配置 strategy 使用，入口资源名称
    private String refResource;
    // 流量控制效果（直接拒绝、Warm Up、匀速排队）
    private int controlBehavior = RuleConstant.CONTROL_BEHAVIOR_DEFAULT;
    // 冷启动时长（预热时长），单位秒
    private int warmUpPeriodSec = 10;
    // 最大排队时间。
    private int maxQueueingTimeMs = 500;
    // 流量控制器
    private TrafficShapingController controller;
    //.....
    @Override
    public boolean passCheck(Context context, DefaultNode node, int acquireCount, Object... args) {
        return true;
    }
}
```

- FlowRule 的字段解析已在源码中给出注释，现在还不需要急于去理解每个字段的作用。
- FlowRule 实现 Rule 接口方法只是返回 true，因为 passCheck 的逻辑并不由 FlowRule 实现。

Rule 定义的行为应该只是 Sentinel 在最初搭建框架时定义的约定，Sentinel 自己也并没有都遵守这个约定，很多规则并没有将 passCheck 交给 Rule 去实现，Checker 可能是后续引入的，用于替代 Rule 的 passCheck 行为。

Sentinel 中用来管理规则配置的类都以规则类的名称+Manger 命名，除此之外，并没有对规则管理器有什么行为上的约束。

用来加载限流规则配置以及缓存限流规则配置的类为 FlowRuleManager，其部分源码如下：

```java
public class FlowRuleManager {
    // 缓存规则
    private static final Map<String, List<FlowRule>> flowRules = new ConcurrentHashMap<String, List<FlowRule>>();
    // 获取所有规则
    static Map<String, List<FlowRule>> getFlowRuleMap() {
        return flowRules;
    }
    // 更新规则
    public static void loadRules(List<FlowRule> rules) {
        // 更新静态字段 flowRules
    }
}
```

- flowRules 静态字段：用于缓存规则配置，使用 ConcurrentMap 缓存，key 为资源的名称，value 是一个 FlowRule 数组。使用数组是因为 Sentinel 支持针对同一个资源配置多种限流规则，只要其中一个先达到限流的阈值就会触发限流。
- loadRules：提供给使用者加载和更新规则的 API，该方法会将参数传递进来的规则数组转为 Map，然后先清空 flowRules 当前缓存的规则配置，再将新的规则配置写入 flowRules。
- getFlowRuleMap：提供给 FlowSlot 获取配置的私有 API

## 熔断与自适应

## 热点参数及黑名单限流

## 自定义开关降级