## 1. oal介绍

​	在流模式(Streaming mode)下，SkyWalking 提供了 观测分析语言(Observability Analysis Language，OAL) 来分析流入的数据。OAL 聚焦于服务，服务实例以及端点的度量指标，因此 OAL 非常易于学习和使用。

​	OAL脚本仍然是编译语言，OAL运行时动态生成Java代码。您可以在系统环境上设置`SW_OAL_ENGINE_DEBUG=Y`，查看生成了哪些类。

### 1.1. 语法

```text
// 声明一个指标
METRICS_NAME = from(SCOPE.(* | [FIELD][,FIELD ...])) // 从某一个SCOPE中获取数据
[.filter(FIELD OP [INT | STRING])] // 可以过滤掉部分数据
.FUNCTION([PARAM][, PARAM ...]) // 使用某个聚合函数将数据聚合

// 禁用一个指标
disable(METRICS_NAME);
```

参考：https://skywalking.apache.org/docs/main/v8.4.0/en/concepts-and-designs/oal/

### 1.2. 示例

```text
// 从ServiceInstanceJVMMemory的used获取数据，只需要 heapStatus 为 true的数据，并取long型的平均值
instance_jvm_memory_heap = from(ServiceInstanceJVMMemory.used).filter(heapStatus == true).longAvg();
```

org.apache.skywalking.oap.server.core.source.ServiceInstanceJVMMemory如下：

```text
@ScopeDeclaration(id = SERVICE_INSTANCE_JVM_MEMORY, name = "ServiceInstanceJVMMemory", catalog = SERVICE_INSTANCE_CATALOG_NAME)
@ScopeDefaultColumn.VirtualColumnDefinition(fieldName = "entityId", columnName = "entity_id", isID = true, type = String.class)
public class ServiceInstanceJVMMemory extends Source {
    @Override
    public int scope() {
        return DefaultScopeDefine.SERVICE_INSTANCE_JVM_MEMORY;
    }

    @Override
    public String getEntityId() {
        return String.valueOf(id);
    }

    @Getter @Setter
    private String id;
    @Getter @Setter @ScopeDefaultColumn.DefinedByField(columnName = "name", requireDynamicActive = true)
    private String name;
    @Getter @Setter @ScopeDefaultColumn.DefinedByField(columnName = "service_name", requireDynamicActive = true)
    private String serviceName;
    @Getter @Setter @ScopeDefaultColumn.DefinedByField(columnName = "service_id")
    private String serviceId;
    @Getter @Setter
    private boolean heapStatus;
    @Getter @Setter
    private long init;
    @Getter @Setter
    private long max;
    @Getter @Setter
    private long used;
    @Getter @Setter
    private long committed;
}
```

### 1.2. 完整案例

## 类加载信息监控