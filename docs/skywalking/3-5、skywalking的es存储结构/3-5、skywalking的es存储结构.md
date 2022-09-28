## sw_apm_metrics-*

这类索引按照指标计算函数生成多种，如果计算数量、平均值、百分比等

其中重要的字段

* metric_table

表示该数据指标由oal表达式确定:

如下面的访问数据的平均延时：

```
database_access_resp_time = from(DatabaseAccess.latency).longAvg();
```

* entity_id

  是和这个指标相关的数据，如服务id、数据库地址id、实例id等等，相等于聚合id，这个字段在source对象阶段就已经确定了。oal中目测看无法修改

* service_id

  服务id，可能有这个也可能没有，可以base64解码

### sw_apm_metrics-count-20220915

指标一共有如下几种metric_table

```
event_total = from(Event.*).count();

event_normal_count = from(Event.*).filter(type == "Normal").count();
event_error_count = from(Event.*).filter(type == "Error").count();

event_start_count = from(Event.*).filter(name == "Start").count();
event_shutdown_count = from(Event.*).filter(name == "Shutdown").count();
```

索引文档示例：

```
{
  "_index": "zayn_apm_metrics-count-20220915",
  "_type": "_doc",
  "_id": "event_total_20220915",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "event_total",//
    "time_bucket": 20220915,//时间窗口，这个是日期的还会存储分钟、小时的。
    "value": 1
  }
}
```

这个索引主要聚合事件记录（如索引apm_events-20220915）的指标

```
{
  "_index": "zayn_apm_events-20220915",
  "_type": "_doc",
  "_id": "20c0bf34-9eb1-465c-8b85-06f0b2bfbd83",
  "_version": 1,
  "_score": 0,
  "_source": {
    "start_time": 1663221348038,
    "endpoint": "",
    "service": "monitor-cat-app",
    "name": "Start",
    "end_time": 1663221355195,
    "time_bucket": 202209151355,
    "service_instance": "",
    "type": "Normal",
    "message": "Start Java Application",
    "uuid": "20c0bf34-9eb1-465c-8b85-06f0b2bfbd83",
    "parameters": "{\"OPTS\":\"-DSW_AGENT_COLLECTOR_BACKEND_SERVICES\\u003dlocalhost:11800 -DSW_AGENT_NAME\\u003dmonitor-cat-app -DSW_LOGGING_FILE_NAME\\u003dskywalking-monitor-cat-app.log -Dcom.sun.management.jmxremote -Dfile.encoding\\u003dUTF-8 -Dspring.application.admin.enabled\\u003dtrue -Dspring.jmx.enabled\\u003dtrue -Dspring.liveBeansView.mbeanDomain -Dspring.output.ansi.enabled\\u003dalways -XX:TieredStopAtLevel\\u003d1 -Xverify:none -javaagent:D:\\\\software\\\\IntelliJ IDEA 2019.2.2\\\\lib\\\\idea_rt.jar\\u003d52082:D:\\\\software\\\\IntelliJ IDEA 2019.2.2\\\\bin -javaagent:E:\\\\mycode\\\\myapm\\\\skywalking-java\\\\skywalking-agent\\\\skywalking-agent.jar\"}"
  }
}
```

### sw_apm_metrics-longavg-20220915

保存了一些和平均值相关的指标

#### instance_jvm_memory_heap指标

jvm内存堆的平均值

```
{
  "_index": "zayn_apm_metrics-longavg-20220915",
  "_type": "_doc",
  "_id": "instance_jvm_memory_heap_20220915_bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",
  "_version": 31,
  "_score": 0,
  "_source": {
    "metric_table": "instance_jvm_memory_heap",
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "count": 3118,
    "time_bucket": 20220915,//按天存储，也可以按照小时存储，也可以按照分钟存储
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",
    "value": 629466148,
    "summation": 1962675450232
  }
}
```

entity_id，这个就是实力元数据instance_traffic索引的id，由两部分拼接组成

* bW9uaXRvci1jYXQtYXBw.1

  服务id

* ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx，这个其实和服务id拼接成为实例元数据（instance_traffic）索引的索引id

  见如下的 "_id": "bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",

  而将base64编码，解码后就成了，索引的name字段：

  f445dea66f3948f9b1d47b30aa08de3d@172.16.62.61

  注意f445dea66f3948f9b1d47b30aa08de3d在每次进程重启后都会变

  这个就是实例元信息索引（apm_instance_traffic-20220915）中的name字段

```
{
  "_index": "zayn_apm_instance_traffic-20220915",
  "_type": "_doc",
  "_id": "bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",
  "_version": 8,
  "_score": 0,
  "_source": {
    "last_ping": 202209151404,
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "name": "f445dea66f3948f9b1d47b30aa08de3d@172.16.62.61",
    "time_bucket": 202209151356,
    "properties": 
    xxxxx
  }
}
```

#### database_access_resp_time指标

某个数据库访问平均响应时间

```
{
  "_index": "zayn_apm_metrics-longavg-20220915",
  "_type": "_doc",
  "_id": "database_access_resp_time_2022091513_MTAuMTMwLjM2LjI0NDozMzA2.0",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "database_access_resp_time",
    "count": 2,
    "time_bucket": 2022091513,//按天存储，也可以按照小时存储，也可以按照分钟存储
    "entity_id": "MTAuMTMwLjM2LjI0NDozMzA2.0",
    "value": 2,
    "summation": 4
  }
}
```

entity_id的值为MTAuMTMwLjM2LjI0NDozMzA2.0，代表数据库地址id为sw_service_traffic的索引id

后面的是sw_service_traffic*索引中存储元数据索引id，：

```
{
  "_index": "zayn_apm_service_traffic-20220915",
  "_type": "_doc",
  "_id": "MTAuMTMwLjM2LjI0NDozMzA2.0",
  "_version": 1,
  "_score": 0,
  "_source": {
    "node_type": 1,
    "name": "10.130.36.244:3306",
    "time_bucket": 202209151356
  }
}
```

数据库10.130.36.244:3306，这个node_type为1表示数据库地址信息，它的id为MTAuMTMwLjM2LjI0NDozMzA2.0为entity_id了

#### service_relation_client_resp_time指标

服务的客户端调用响应时间

```
service_instance_relation_client_resp_time = from(ServiceInstanceRelation.latency).filter(detectPoint == DetectPoint.CLIENT).longAvg();
```

```
{
  "_index": "zayn_apm_metrics-longavg-20220915",
  "_type": "_doc",
  "_id": "service_relation_client_resp_time_20220915_bW9uaXRvci1jYXQtYXBw.1-MTAuMTMwLjM2LjIzMzo2Mzc5.0",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "service_relation_client_resp_time",
    "count": 1,
    "time_bucket": 20220915,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1-MTAuMTMwLjM2LjIzMzo2Mzc5.0",
    "value": 125,
    "summation": 125
  }
}
```

entity_id数据bW9uaXRvci1jYXQtYXBw.1-MTAuMTMwLjM2LjIzMzo2Mzc5.0由两部分组成：

* 服务id

* MTAuMTMwLjM2LjIzMzo2Mzc5.0（其实是调用方的服务id，这里调用方没有服务id就用这个代表了）

  调用地址的base64解码为：10.130.36.233:6379



#### endpoint_avg指标

某一次入口请求的耗时指标计算。这个endpoint只会从入口segent中请求获取。

由如下oal生成

```
endpoint_avg = from(Endpoint.latency).longAvg();
```

文档数据如下：

```
{
  "_index": "zayn_apm_metrics-longavg-20220915",
  "_type": "_doc",
  "_id": "endpoint_avg_202209151815_bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "endpoint_avg",
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "count": 3,
    "time_bucket": 202209151815,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
    "value": 46,
    "summation": 140
  },
  "highlight": {
    "metric_table": [
      "@kibana-highlighted-field@endpoint_avg@/kibana-highlighted-field@"
    ]
  }
}
```

其中entity_id由两部分组成

service_id和endponitname的base64编码组成

UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==解码后就是POST:/frame/getOutline

count为执行次数

value为平均耗时

#### endpoint_relation_resp_time指标

服务端被远程调用的耗时情况

```
endpoint_relation_resp_time = from(EndpointRelation.rpcLatency).filter(detectPoint == DetectPoint.SERVER).longAvg();
```

```
{
  "_index": "zayn_apm_metrics-longavg-20220915",
  "_type": "_doc",
  "_id": "endpoint_relation_resp_time_202209151815_VXNlcg==.0-VXNlcg==-bW9uaXRvci1jYXQtYXBw.1-UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "endpoint_relation_resp_time",
    "count": 3,
    "time_bucket": 202209151815,
    "entity_id": "VXNlcg==.0-VXNlcg==-bW9uaXRvci1jYXQtYXBw.1-UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
    "value": 46,
    "summation": 140
  },
  "highlight": {
    "metric_table": [
      "@kibana-highlighted-field@endpoint_relation_resp_time@/kibana-highlighted-field@"
    ]
  }
}
```

entity_id组成由四个部分组成

```
    public String getEntityId() {
        return IDManager.EndpointID.buildRelationId(new IDManager.EndpointID.EndpointRelationDefine(
            serviceId, endpoint, childServiceId, childEndpoint
        ));
    }
```

当前调用方的服务id、调用方的请求名、当前服务的服务id，被调用方的服务名

这个值在进行entry解析时从refs列表中获取

### sw_metrics-cpm-20220915

该索引保存了和cpm相关的指标数据

cpm 全称 call per minutes，是吞吐量(Throughput)指标。

#### endpoint_cpm指标

```
endpoint_cpm = from(Endpoint.*).cpm();
```

```
{
  "_index": "zayn_apm_metrics-cpm-20220915",
  "_type": "_doc",
  "_id": "endpoint_cpm_202209151747_bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "endpoint_cpm",
    "total": 1,
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "time_bucket": 202209151747,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
    "value": 1
  },
  "highlight": {
    "metric_table": [
      "@kibana-highlighted-field@endpoint_cpm@/kibana-highlighted-field@"
    ]
  }
}
```

entitity由两部分组成：service_id和endpointid。

_id由metric_table_time_bucket__entity_id拼接而成

#### service_cpm指标

服务整体的吞吐量，注意service指标的是来自于span的中entry来的

```
service_cpm = from(Service.*).cpm();
```

```
{
  "_index": "zayn_apm_metrics-cpm-20220915",
  "_type": "_doc",
  "_id": "service_cpm_2022091517_bW9uaXRvci1jYXQtYXBw.1",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "service_cpm",
    "total": 1,
    "time_bucket": 2022091517,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1",
    "value": 0
  },
  "highlight": {
    "metric_table": [
      "@kibana-highlighted-field@service_cpm@/kibana-highlighted-field@"
    ]
  }
}
```

entity_id表示服务id和节点类型



### sw_metrics-percent-20220915

百分比指标

#### service_sla指标

```
service_sla = from(Service.*).percent(status == true);
```

实际上计算下来，就是这个服务发生非错误的entryspan和entryspan的总数的比值

SLA 全称 Service-Level Agreement，直译为 “服务等级协议”，用来表示提供服务的水平。在IT中，SLA可以衡量平台的可用性，下面是N个9的计算：

1. 1年 = 365天 = 8760小时
2. 99 = 8760 * 1% => 3.65天
3. 99.9 = 8760 * 0.1% => 8.76小时
4. 99.99 = 8760 * 0.01% => 52.6分钟
5. 99.999 = 8760 * 0.001% => 5.26分钟

因此，全年只要发生一次较大规模宕机事故，4个9肯定没戏，一般平台3个9差不多。但2个9就基本不可用了，相当于全年有87.6小时不可用，每周(一个月按4周算)有1.825小时不可用。下图是服务、实例、接口的SLA，一般看年度、月度即可

```
{
  "_index": "zayn_apm_metrics-percent-20220915",
  "_type": "_doc",
  "_id": "service_sla_202209151747_bW9uaXRvci1jYXQtYXBw.1",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "service_sla",
    "total": 1,
    "percentage": 10000,
    "match": 1,
    "time_bucket": 202209151747,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1"
  },
  "highlight": {
    "metric_table": [
      "@kibana-highlighted-field@service_sla@/kibana-highlighted-field@"
    ]
  }
}
```



entity_id和之前分析的一样，来来自service这个source的entry字段



### sw_metrics-percentile-20220915

99线95线之类的指标，表示采集样本中某些值的占比，Skywalking 有 `p50、p75、p90、p95、p99` 一些列值。其中的 “p99:390” 表示 99% 请求的响应时间在390ms以内。而99%一般用于抛掉一些极端值，表示绝大多数请求

如下几个例子

#### endpoint_percentile指标

```
endpoint_percentile = from(Endpoint.latency).percentile(10); // Multiple values including p50, p75, p90, p95, p99
```

这个percentile(10)，代表的是精确度，10毫秒的精确度

```
    public final void combine(@SourceFrom int value, @Arg int precision) {
        this.isCalculated = false;
        this.precision = precision;

        String index = String.valueOf(value / precision);
        dataset.valueAccumulation(index, 1L);
    }
```

大致意思是将耗时按照桶10毫秒为间隔分割成桶，每个key都有数量。先计算出每一个roofs[i]的实际数量，99线的数量，95线的数量

按照桶key排序，然后从小到大排序，循环累加，放到99线的结果里，95线的结果里，简单点讲，就是遍历key，根据耗时桶从小到大，从50%的请求遍历到99%的请求。

```
  public final void calculate() {
        if (!isCalculated) {
            long total = dataset.sumOfValues();

            int[] roofs = new int[RANKS.length];
            for (int i = 0; i < RANKS.length; i++) {
                roofs[i] = Math.round(total * RANKS[i] * 1.0f / 100);
            }

            int count = 0;
            final List<String> sortedKeys = dataset.sortedKeys(Comparator.comparingInt(Integer::parseInt));

            int loopIndex = 0;
            for (String key : sortedKeys) {
                final Long value = dataset.get(key);

                count += value;
                for (int rankIdx = loopIndex; rankIdx < roofs.length; rankIdx++) {
                    int roof = roofs[rankIdx];

                    if (count >= roof) {
                        percentileValues.put(String.valueOf(rankIdx), Long.parseLong(key) * precision);
                        loopIndex++;
                    } else {
                        break;
                    }
                }
            }
        }
```



以下为endpoint_percentile指标的es文档

```
{
  "_index": "zayn_apm_metrics-percentile-20220915",
  "_type": "_doc",
  "_id": "endpoint_percentile_202209151747_bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "endpoint_percentile",
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "precision": 10,
    "time_bucket": 202209151747,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
    "value": "0,860|1,860|2,860|3,860|4,860",
    "dataset": "86,1"
  },
  "highlight": {
    "metric_table": [
      "@kibana-highlighted-field@endpoint_percentile@/kibana-highlighted-field@"
    ]
  }
}
```

entity_id为serviceId和endpointname的base64值

value的值为p50, p75, p90, p95, p99这几个

### sw_metrics-sum-20220915  

次数累加指标，按分钟、小时、天进行累加的索引

#### instance_jvm_young_gc_count指标

年轻代gc次数聚合sum

```
{
  "_index": "zayn_apm_metrics-sum-20220915",
  "_type": "_doc",
  "_id": "instance_jvm_young_gc_count_202209151355_bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "instance_jvm_young_gc_count",
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "time_bucket": 202209151355,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",
    "value": 14
  }
}
```

entity_id为实例id

也是由服务id和实例信息拼接而成，如前面的例子一样。根据这个entity_id在traffic中能查到更加详细的其它信息。

### sw-metrics-histogram-20220915

Heapmap 可译为热力图、热度图都可以，其中颜色越深，表示请求数越多，这和GitHub Contributions很像，commit越多，颜色越深。横坐标是响应时间，鼠标放上去，可以看到具体的数量。通过热力图，一方面可以直观感受平台的整体流量，另一方面也可以感受整体性能

目前只存储all_heatmap指标

```
all_heatmap = from(All.latency).histogram(100, 20);
```

```
{
  "_index": "zayn_apm_metrics-histogram-20220915",
  "_type": "_doc",
  "_id": "all_heatmap_202209151747",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "all_heatmap",
    "time_bucket": 202209151747,
    "dataset": "1000,0|2000,0|0,0|1800,0|100,0|1700,0|200,0|1600,0|300,0|1500,0|400,0|1400,0|500,0|1300,0|600,0|1200,0|700,0|1100,0|800,1|900,0|1900,0"
  }
}
```

按照分钟小时天存储

### sw-metrics-apdex-*

是一个衡量服务器性能的标准。apdex有三个指标：

- 满意：请求响应时间小于等于T。
- 可容忍：请求响应时间大于T，小于等于4T。
- 失望：请求响应时间大于4T。

T：自定义的一个时间值，比如：500ms。apdex = （满意数 + 可容忍数/2）/ 总数。例如：服务A定义T=200ms，在100个采样中，有20个请求小于200ms，有60个请求在200ms到800ms之间，有20个请求大于800ms。计算apdex = (20 + 60/2)/100 = 0.5。



```
{
  "_index": "zayn_apm_metrics-apdex-20220920",
  "_type": "_doc",
  "_id": "service_apdex_202209201658_bW9uaXRvci1jYXQtYXBw.1",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metric_table": "service_apdex",
    "t_num": 0,
    "total_num": 6,
    "time_bucket": 202209201658,
    "s_num": 6,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1",
    "value": 10000
  }
}
```

#### service_apdex指标

来源于oal表达式

```
service_apdex = from(Service.latency).apdex(name, status);
```

代码逻辑为

```
   @Entrance
    public final void combine(@SourceFrom int value, @Arg String name, @Arg boolean status) {
        int t = DICT.lookup(name).intValue();
        int t4 = t * 4;
        totalNum++;
        if (!status || value > t4) {
            return;
        }
        if (value > t) {
            tNum++;
        } else {
            sNum++;
        }
    }
```



## sw_traffic_*

元数据相关的索引

### sw_service_traffic-20220915

存储服务相关的信息如

```
{
  "_index": "zayn_apm_service_traffic-20220915",
  "_type": "_doc",
  "_id": "MTAuMTMwLjM2LjI0NDozMzA2.0",
  "_version": 1,
  "_score": 0,
  "_source": {
    "node_type": 1,
    "name": "10.130.36.244:3306",
    "time_bucket": 202209151356
  }
}
```

其中node_type由多种：

```
  /**
     * <code>Normal = 0;</code>
     * This node type would be treated as an observed node.
     */
    Normal(0),
    /**
     * <code>Database = 1;</code>
     */
    Database(1),
    /**
     * <code>RPCFramework = 2;</code>
     */
    RPCFramework(2),
    /**
     * <code>Http = 3;</code>
     */
    Http(3),
    /**
     * <code>MQ = 4;</code>
     */
    MQ(4),
    /**
     * <code>Cache = 5;</code>
     */
    Cache(5),
    /**
     * <code>Browser = 6;</code>
     * This node type would be treated as an observed node.
     */
    Browser(6),
    /**
     * <code>User = 10</code>
     */
    User(10),
    /**
     * <code>Unrecognized = 11</code>
     */
    Unrecognized(11);
```

表示一个观测节点类型。

如上的示例的节点类型为数据库，它的地址为10.130.36.244:3306

```
{
  "_index": "zayn_apm_service_traffic-20220915",
  "_type": "_doc",
  "_id": "bW9uaXRvci1jYXQtYXBw.1",
  "_version": 1,
  "_score": 0,
  "_source": {
    "node_type": 0,
    "name": "monitor-cat-app",
    "time_bucket": 202209151357
  }
}
```

这个节点类型为0，代表一个正常服务，名称为monitor-cat-app

### sw_endpoint_traffic-20220915

表示endpoint元信息索引，比如记录了一个segement之中的http调用等信息。

其索引id时service_id和endpointname拼接而成的。

```
{
  "_index": "zayn_apm_endpoint_traffic-20220915",
  "_type": "_doc",
  "_id": "bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==",
  "_version": 1,
  "_score": 0,
  "_source": {
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "name": "POST:/frame/getOutline",
    "time_bucket": 202209151747
  }
}
```

其中UE9TVDovZnJhbWUvZ2V0T3V0bGluZQ==可以由name进行base64编码而成

### sw_instance_traffic-20220915

记录了一个进程实例的元信息，每次启动都会有条记录，因为进程号变了。

```
{
  "_index": "zayn_apm_instance_traffic-20220915",
  "_type": "_doc",
  "_id": "bW9uaXRvci1jYXQtYXBw.1_Yzg0MjA2OWE5ZjJkNGZkOTk5MzIyMjAwMjAyODM1YmVAMTcyLjE2LjYyLjYx",
  "_version": 37,
  "_score": 0,
  "_source": {
    "last_ping": 202209151822,
    "service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "name": "c842069a9f2d4fd999322200202835be@172.16.62.61",
    "time_bucket": 202209151746,
    "properties": "{\"OS Name\":\"Windows 10\",\"hostname\":\"D
    }
}
```

Yzg0MjA2OWE5ZjJkNGZkOTk5MzIyMjAwMjAyODM1YmVAMTcyLjE2LjYyLjYx由name字段base64编码而成

## sw_service_ * _relation _*

这类索引由RPCAnalysisListener解析并生成，在老版本中由MultiScopesAnalysisListener解析

perseEntry会解析很多ServiceRelation、ServiceInstanceRelation

### sw_service_instance_relation_  * _side

记录了实例与实例之间的调用关系，有小时、分钟、天的事件维度

```
public class ServiceInstanceCallRelationDispatcher implements SourceDispatcher<ServiceInstanceRelation> {

    @Override
    public void dispatch(ServiceInstanceRelation source) {
        switch (source.getDetectPoint()) {
            case SERVER:
                serverSide(source);
                break;
            case CLIENT:
                clientSide(source);
                break;
        }
    }

    private void serverSide(ServiceInstanceRelation source) {
        ServiceInstanceRelationServerSideMetrics metrics = new ServiceInstanceRelationServerSideMetrics();
        metrics.setTimeBucket(source.getTimeBucket());
        metrics.setSourceServiceId(source.getSourceServiceId());
        metrics.setSourceServiceInstanceId(source.getSourceServiceInstanceId());
        metrics.setDestServiceId(source.getDestServiceId());
        metrics.setDestServiceInstanceId(source.getDestServiceInstanceId());
        metrics.setComponentId(source.getComponentId());
        metrics.setEntityId(source.getEntityId());
        MetricsStreamProcessor.getInstance().in(metrics);
    }

    private void clientSide(ServiceInstanceRelation source) {
        ServiceInstanceRelationClientSideMetrics metrics = new ServiceInstanceRelationClientSideMetrics();
        metrics.setTimeBucket(source.getTimeBucket());
        metrics.setSourceServiceId(source.getSourceServiceId());
        metrics.setSourceServiceInstanceId(source.getSourceServiceInstanceId());
        metrics.setDestServiceId(source.getDestServiceId());
        metrics.setDestServiceInstanceId(source.getDestServiceInstanceId());
        metrics.setComponentId(source.getComponentId());
        metrics.setEntityId(source.getEntityId());
        MetricsStreamProcessor.getInstance().in(metrics);
    }
}
```

会生成这两个metric类，存入es的索引

entry_id为sourceServiceInstanceId、destServiceInstanceId

示例**service_instance_relation_client_side**索引文档为：

```
{
  "_index": "zayn_apm_service_instance_relation_client_side-20220915",
  "_type": "_doc",
  "_id": "202209151356_bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx-MTAuMTMwLjM2LjIzMzo2Mzc5.0_MTAuMTMwLjM2LjIzMzo2Mzc5",
  "_version": 1,
  "_score": 0,
  "_source": {
    "dest_service_instance_id": "MTAuMTMwLjM2LjIzMzo2Mzc5.0_MTAuMTMwLjM2LjIzMzo2Mzc5",
    "source_service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "component_id": 56,
    "dest_service_id": "MTAuMTMwLjM2LjIzMzo2Mzc5.0",
    "time_bucket": 202209151356,
    "source_service_instance_id": "bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx",
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_ZjQ0NWRlYTY2ZjM5NDhmOWIxZDQ3YjMwYWEwOGRlM2RAMTcyLjE2LjYyLjYx-MTAuMTMwLjM2LjIzMzo2Mzc5.0_MTAuMTMwLjM2LjIzMzo2Mzc5"
  }
}
```

这里会记录组件id，但是entity_id却没有这个信息，这就很奇怪，enttity_id只由源服务id和服务实例与目标服务id和服务实例组成，那么是否存在重复的可能

**注意这里会记录所有的访问外部的调用关系，包括服务调用服务，服务调用redis，服务调用mysql等外部组件**

示例

```
{
  "_index": "zayn_apm_service_instance_relation_server_side-20220919",
  "_type": "_doc",
  "_id": "202209191510_bW9uaXRvci1jYXQtYXBw.1_OTQyYTRhNjJhMzE2NGUzZmIyNTM2ZDA4YTdlNTFiZDRAMTcyLjE2LjYyLjYx-bW9uaXRvci1jYXQtd2Vi.1_ZGU2OWE5ZWJkMDY1NGYwYWFmMDYxMWM4OWUyMzg1ODNAMTcyLjE2LjYyLjYx",
  "_version": 1,
  "_score": 0,
  "_source": {
    "dest_service_instance_id": "bW9uaXRvci1jYXQtd2Vi.1_ZGU2OWE5ZWJkMDY1NGYwYWFmMDYxMWM4OWUyMzg1ODNAMTcyLjE2LjYyLjYx",
    "source_service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "component_id": 14,
    "dest_service_id": "bW9uaXRvci1jYXQtd2Vi.1",
    "time_bucket": 202209191510,
    "source_service_instance_id": "bW9uaXRvci1jYXQtYXBw.1_OTQyYTRhNjJhMzE2NGUzZmIyNTM2ZDA4YTdlNTFiZDRAMTcyLjE2LjYyLjYx",
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1_OTQyYTRhNjJhMzE2NGUzZmIyNTM2ZDA4YTdlNTFiZDRAMTcyLjE2LjYyLjYx-bW9uaXRvci1jYXQtd2Vi.1_ZGU2OWE5ZWJkMDY1NGYwYWFmMDYxMWM4OWUyMzg1ODNAMTcyLjE2LjYyLjYx"
  }
}
```

entity_id由source的service_id和instance_id组成以及目的的service_id和instance_id组成。

component_id为14代表springmvc组件

### sw_endpoint_relation_ * _ side

记录了端点与端点之间的关系，

为zayn_apm_endpoint_relation_server_side-20220919**注意没有zayn_apm_endpoint_relation_client_side-20220919索引**,因为endpoint_relation数据只从entry的span中获取

源码为：

```
public class EndpointCallRelationDispatcher implements SourceDispatcher<EndpointRelation> {

    @Override
    public void dispatch(EndpointRelation source) {
        switch (source.getDetectPoint()) {
            case SERVER:
                serverSide(source);
                break;
        }
    }

    private void serverSide(EndpointRelation source) {
        EndpointRelationServerSideMetrics metrics = new EndpointRelationServerSideMetrics();
        metrics.setTimeBucket(source.getTimeBucket());
        metrics.setSourceEndpoint(
            IDManager.EndpointID.buildId(source.getServiceId(), source.getEndpoint()));
        metrics.setDestEndpoint(
            IDManager.EndpointID.buildId(source.getChildServiceId(), source.getChildEndpoint()));
        metrics.setComponentId(source.getComponentId());
        metrics.setEntityId(source.getEntityId());
        MetricsStreamProcessor.getInstance().in(metrics);
    }
}
```

表示服务侧调用关系

```
{
  "_index": "zayn_apm_endpoint_relation_server_side-20220919",
  "_type": "_doc",
  "_id": "202209191510_bW9uaXRvci1jYXQtYXBw.1-UE9TVDovZnJhbWUvZ2V0TGlzdA==-bW9uaXRvci1jYXQtd2Vi.1-UE9TVDovYXBwSW5mby9nZXRMaXN0",
  "_version": 1,
  "_score": 0,
  "_source": {
    "component_id": 14,
    "source_endpoint": "bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0TGlzdA==",
    "dest_endpoint": "bW9uaXRvci1jYXQtd2Vi.1_UE9TVDovYXBwSW5mby9nZXRMaXN0",
    "time_bucket": 202209191510,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1-UE9TVDovZnJhbWUvZ2V0TGlzdA==-bW9uaXRvci1jYXQtd2Vi.1-UE9TVDovYXBwSW5mby9nZXRMaXN0"
  }
}
```

dest_endpoint由两部分组成，前半部分为service_id, 后半部分表示被调用的endponitname这里base64解码后为POST:/frame/getList

source_endpoint为来源的service_id,和来源endpontiname，这个来源endpontiname为上一个的endponitname，如果没有的话为User

### sw_service_relation_ * _ side

该数据既从entry的span获取，也从exit的span获取

```

        exitSourceBuilders.forEach(exitSourceBuilder -> {
            exitSourceBuilder.prepare();
            sourceReceiver.receive(exitSourceBuilder.toServiceRelation());

            /*
             * Some of the agent can not have the upstream real network address, such as https://github.com/apache/skywalking-nginx-lua.
             */
            final ServiceInstanceRelation serviceInstanceRelation = exitSourceBuilder.toServiceInstanceRelation();
            if (serviceInstanceRelation != null) {
                sourceReceiver.receive(serviceInstanceRelation);
            }
            if (RequestType.DATABASE.equals(exitSourceBuilder.getType())) {
                sourceReceiver.receive(exitSourceBuilder.toServiceMeta());
                sourceReceiver.receive(exitSourceBuilder.toDatabaseAccess());
            }
        });
```

```
public class ServiceCallRelationDispatcher implements SourceDispatcher<ServiceRelation> {

    @Override
    public void dispatch(ServiceRelation source) {
        switch (source.getDetectPoint()) {
            case SERVER:
                serverSide(source);
                break;
            case CLIENT:
                clientSide(source);
                break;
        }
    }

    private void serverSide(ServiceRelation source) {
        ServiceRelationServerSideMetrics metrics = new ServiceRelationServerSideMetrics();
        metrics.setTimeBucket(source.getTimeBucket());
        metrics.setSourceServiceId(source.getSourceServiceId());
        metrics.setDestServiceId(source.getDestServiceId());
        metrics.setComponentId(source.getComponentId());
        metrics.setEntityId(source.getEntityId());
        MetricsStreamProcessor.getInstance().in(metrics);
    }

    private void clientSide(ServiceRelation source) {
        ServiceRelationClientSideMetrics metrics = new ServiceRelationClientSideMetrics();
        metrics.setTimeBucket(source.getTimeBucket());
        metrics.setSourceServiceId(source.getSourceServiceId());
        metrics.setDestServiceId(source.getDestServiceId());
        metrics.setComponentId(source.getComponentId());
        metrics.setEntityId(source.getEntityId());
        MetricsStreamProcessor.getInstance().in(metrics);
    }
}

```

zayn_apm_service_relation_server_side-20220919

表示服务端看应用的调用关系

索引示例为:

```
{
  "_index": "zayn_apm_service_relation_server_side-20220919",
  "_type": "_doc",
  "_id": "202209191510_bW9uaXRvci1jYXQtYXBw.1-bW9uaXRvci1jYXQtd2Vi.1",
  "_version": 1,
  "_score": 0,
  "_source": {
    "source_service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "component_id": 14,
    "dest_service_id": "bW9uaXRvci1jYXQtd2Vi.1",
    "time_bucket": 202209191510,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1-bW9uaXRvci1jYXQtd2Vi.1"
  }
}
```

entity为source_service_id和dest_service_id的组合，如果上级没有，就用user来表示

这里也会记录component_id，14代表的是springmvc类型的组件。

这里还是有个疑问，这个表示索引id为服务之间的绑定关系，如果服务之间可能被Http调用，又可以被rpc调用的关系，那么component_id就会被覆盖了。

zayn_apm_service_relation_client_side-20220919

这个client的类型就比较多了，外部调用的都会被记录

比如数据库、redis、其它服务等等。都会被记录。

注意如果调用下一个服务，会记录endponitname的信息，如

```
{
  "_index": "zayn_apm_service_relation_client_side-20220919",
  "_type": "_doc",
  "_id": "202209191510_bW9uaXRvci1jYXQtYXBw.1-MTI3LjAuMC4xOjg4OTk=.0",
  "_version": 1,
  "_score": 0,
  "_source": {
    "source_service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "component_id": 2,
    "dest_service_id": "MTI3LjAuMC4xOjg4OTk=.0",
    "time_bucket": 202209191510,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1-MTI3LjAuMC4xOjg4OTk=.0"
  }
}
```

dest_service_id解析为127.0.0.1:8899，0表非正常应用。

也会记录下一个service_id，这个service是exit解析的时候，从final NetworkAddressAlias networkAddressAlias = networkAddressAliasCache.get(networkAddress);获取的，这个数据存储在network_address_alias索引之中，后面会介绍

```
{
  "_index": "zayn_apm_service_relation_client_side-20220919",
  "_type": "_doc",
  "_id": "2022091915_bW9uaXRvci1jYXQtYXBw.1-bW9uaXRvci1jYXQtd2Vi.1",
  "_version": 1,
  "_score": 0,
  "_source": {
    "source_service_id": "bW9uaXRvci1jYXQtYXBw.1",
    "component_id": 2,
    "dest_service_id": "bW9uaXRvci1jYXQtd2Vi.1",
    "time_bucket": 2022091915,
    "entity_id": "bW9uaXRvci1jYXQtYXBw.1-bW9uaXRvci1jYXQtd2Vi.1"
  },
  "highlight": {
    "dest_service_id": [
      "@kibana-highlighted-field@bW9uaXRvci1jYXQtd2Vi.1@/kibana-highlighted-field@"
    ]
  }
}
```

## sw_network_address_alias

存储了网络地址和service_id的关系，比如serviceA使用httpclient调用serviceB，解析serviceA的exitspan时，需要知道serviceB的id，可以根据http的网络地址获取到，比如

```
{
  "_index": "zayn_apm_network_address_alias-20220919",
  "_type": "_doc",
  "_id": "MTI3LjAuMC4xOjg4OTk=",
  "_version": 3,
  "_score": 0,
  "_source": {
    "address": "127.0.0.1:8899",
    "last_update_time_bucket": 202209191649,
    "represent_service_instance_id": "bW9uaXRvci1jYXQtd2Vi.1_ZGU2OWE5ZWJkMDY1NGYwYWFmMDYxMWM4OWUyMzg1ODNAMTcyLjE2LjYyLjYx",
    "represent_service_id": "bW9uaXRvci1jYXQtd2Vi.1",
    "time_bucket": 202209191510
  }
}
```

这个数据的id就是127.0.0.1:8899的base64编码。

通过address可以获取服务id

```
    final NetworkAddressAlias networkAddressAlias = networkAddressAliasCache.get(networkAddress);
        if (networkAddressAlias == null) {
            sourceBuilder.setDestServiceName(networkAddress);
            sourceBuilder.setDestServiceInstanceName(networkAddress);
            sourceBuilder.setDestNodeType(NodeType.fromSpanLayerValue(span.getSpanLayer()));
        } else {
            /*
             * If alias exists, mean this network address is representing a real service.
             */
            final IDManager.ServiceID.ServiceIDDefinition serviceIDDefinition = IDManager.ServiceID.analysisId(
                networkAddressAlias.getRepresentServiceId());
            final IDManager.ServiceInstanceID.InstanceIDDefinition instanceIDDefinition = IDManager.ServiceInstanceID
                .analysisId(
                    networkAddressAlias.getRepresentServiceInstanceId());
            sourceBuilder.setDestServiceName(serviceIDDefinition.getName());
            /*
             * Some of the agent can not have the upstream real network address, such as https://github.com/apache/skywalking-nginx-lua.
             * Keeping dest instance name as NULL makes no instance relation generate from this exit span.
             */
            if (!config.shouldIgnorePeerIPDue2Virtual(span.getComponentId())) {
                sourceBuilder.setDestServiceInstanceName(instanceIDDefinition.getName());
            }
            sourceBuilder.setDestNodeType(NodeType.Normal);
        }
```

这个索引的数据由NetworkAddressAliasMappingListener进行解析，从entry的span中获取



```
   public void parseEntry(SpanObject span, SegmentObject segmentObject) {
        if (span.getSkipAnalysis()) {
            return;
        }
        if (log.isDebugEnabled()) {
            log.debug("service instance mapping listener parse reference");
        }
        if (!span.getSpanLayer().equals(SpanLayer.MQ)) {
            span.getRefsList().forEach(segmentReference -> {
                if (RefType.CrossProcess.equals(segmentReference.getRefType())) {
                    final String networkAddressUsedAtPeer = namingControl.formatServiceName(
                        segmentReference.getNetworkAddressUsedAtPeer());
                    if (config.getUninstrumentedGatewaysConfig().isAddressConfiguredAsGateway(
                        networkAddressUsedAtPeer)) {
                        /*
                         * If this network address has been set as an uninstrumented gateway, no alias should be set.
                         */
                        return;
                    }
                    final String serviceName = namingControl.formatServiceName(segmentObject.getService());
                    final String instanceName = namingControl.formatInstanceName(
                        segmentObject.getServiceInstance());

                    final NetworkAddressAliasSetup networkAddressAliasSetup = new NetworkAddressAliasSetup();
                    networkAddressAliasSetup.setAddress(networkAddressUsedAtPeer);
                    networkAddressAliasSetup.setRepresentService(serviceName);
                    networkAddressAliasSetup.setRepresentServiceNodeType(NodeType.Normal);
                    networkAddressAliasSetup.setRepresentServiceInstance(instanceName);
                    networkAddressAliasSetup.setTimeBucket(TimeBucket.getMinuteTimeBucket(span.getStartTime()));

                    sourceReceiver.receive(networkAddressAliasSetup);
                }

            });
        }
    }
```

## sw_events-*

事件相关的记录，它可以被metric进行聚合，上面已经介绍过了。

```
{
  "_index": "zayn_apm_events-20220915",
  "_type": "_doc",
  "_id": "20c0bf34-9eb1-465c-8b85-06f0b2bfbd83",
  "_version": 1,
  "_score": 0,
  "_source": {
    "start_time": 1663221348038,
    "endpoint": "",
    "service": "monitor-cat-app",
    "name": "Start",
    "end_time": 1663221355195,
    "time_bucket": 202209151355,
    "service_instance": "",
    "type": "Normal",
    "message": "Start Java Application",
    "uuid": "20c0bf34-9eb1-465c-8b85-06f0b2bfbd83",
    "parameters": "{\"OPTS\":\"-DSW_AGENT_COLLECTOR_BACKEND_SERVICES\\u003dlocalhost:11800 -DSW_AGENT_NAME\\u003dmonitor-cat-app -DSW_LOGGING_FILE_NAME\\u003dskywalking-monitor-cat-app.log -Dcom.sun.management.jmxremote -Dfile.encoding\\u003dUTF-8 -Dspring.application.admin.enabled\\u003dtrue -Dspring.jmx.enabled\\u003dtrue -Dspring.liveBeansView.mbeanDomain -Dspring.output.ansi.enabled\\u003dalways -XX:TieredStopAtLevel\\u003d1 -Xverify:none -javaagent:D:\\\\software\\\\IntelliJ IDEA 2019.2.2\\\\lib\\\\idea_rt.jar\\u003d52082:D:\\\\software\\\\IntelliJ IDEA 2019.2.2\\\\bin -javaagent:E:\\\\mycode\\\\myapm\\\\skywalking-java\\\\skywalking-agent\\\\skywalking-agent.jar\"}"
  }
}
```

这个数据来源于注册的org.apache.skywalking.oap.server.receiver.event.EventModuleProvider中的事件分析器。

注意告警模块产生的告警记录也会被同步到这里来。

## sw_alarm_record-*

产生的告警记录存储

```
{
  "_index": "zayn_apm_alarm_record-20220920",
  "_type": "_doc",
  "_id": "20220920163619_service_cpm_rule_bW9uaXRvci1jYXQtd2Vi.1_",
  "_version": 1,
  "_score": 0,
  "_source": {
    "start_time": 1663662979263,
    "id0": "bW9uaXRvci1jYXQtd2Vi.1",
    "rule_name": "service_cpm_rule",
    "tags_raw_data": "W3sia2V5IjoibGV2ZWwiLCJ2YWx1ZSI6IldBUk5JTkcifV0=",
    "scope": 1,
    "id1": "",
    "name": "monitor-cat-web",
    "alarm_message": "Alarm caused by Rule service_cpm_rule",
    "time_bucket": 20220920163619,
    "tags": [
      "level=WARNING"
    ]
  }
}
```

* id0表示告警的id

* name

  告警名称，不同的域，这个name组合不一样。

* rule_name表示规则名称

* tags,

  设置的标签数据

  ```
  [{"key":"level","value":"WARNING"}]
  ```

  

又例如endpoint域的告警记录

```
{
  "_index": "zayn_apm_alarm_record-20220920",
  "_type": "_doc",
  "_id": "20220920165916_endpoint_sla_rule_bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0TGlzdA==_",
  "_version": 1,
  "_score": 0,
  "_source": {
    "start_time": 1663664356744,
    "id0": "bW9uaXRvci1jYXQtYXBw.1_UE9TVDovZnJhbWUvZ2V0TGlzdA==",
    "rule_name": "endpoint_sla_rule",
    "tags_raw_data": "W3sia2V5IjoibGV2ZWwiLCJ2YWx1ZSI6IkVSUk9SIn1d",
    "scope": 3,
    "id1": "",
    "name": "POST:/frame/getList in monitor-cat-app",
    "alarm_message": "TEST message",
    "time_bucket": 20220920165916,
    "tags": [
      "level=ERROR"
    ]
  }
}
```

id0为应用名和endpointname的base64拼接值.

## sw_top_n_database_statement*

存储了topN的的数据库语句

```
{
  "_index": "zayn_apm_top_n_database_statement-20220921",
  "_type": "_doc",
  "_id": "878ebb67473348e09307a31b48531b68.170.16637397009500002-6",
  "_version": 1,
  "_score": 0,
  "_source": {
    "trace_id": "878ebb67473348e09307a31b48531b68.170.16637397009500003",
    "latency": 11,
    "service_id": "MTAuMTMwLjM2LjI0NDozMzA2.0",
    "statement": "SELECT count(0) FROM monitor_frame_event WHERE create_datetime BETWEEN ? AND ?",
    "time_bucket": 20220921135501
  }
}
```

service_id表示的是，数据库端点的base64编码。

TraceSegmentId + SpanId 构成

这里需要注意，内存中只保存topN50条，但是每次定时任务会flush到es之中，也就是有可能会重复。