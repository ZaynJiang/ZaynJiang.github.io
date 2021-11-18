## 1. 开头
&emsp;&emsp;上一篇介绍了elasticsearch的写入流程，了解了elasticsearch写入时的分布式特性的相关原理。elasticsearch作为一款具有强大搜索功能的存储引擎，它的查询是什么样的呢？在使用过程中有哪些需要我们注意的呢？  
&emsp;&emsp;在前面的文章我们已经知道elasticsearch的读取分为两种GET和SEARCH。这两种操作是有一定的差异的，下面我们分别对这两种核心的数据读取方式进行一一分析。

## 2. GET的流程
### 2.1 整体流程
![](get流程总览.png)  
图片来自官网  
以下是从主分片或者副本分片检索文档的步骤顺序：
（ 客户端向 Node 1 发送获取请求。
* 节点使用文档的 _id 来确定文档属于分片 0 。分片 0 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 Node 2 。
* Node 2 将文档返回给 Node 1 ，然后将文档返回给客户端。  
  
&emsp;&emsp;在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡。  
&emsp;&emsp;在文档被检索时，已经被索引的文档可能已经存在于主分片上但是还没有复制到副本分片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。 一旦索引请求成功返回给用户，文档在主分片和副本分片都是可用的

### 2.3 整体流程

### 2.2. GET流程
参考文档：https://zhuanlan.zhihu.com/p/34674517

在协调节点有个http_server_worker线程池。收到读请求后它的具体过程为：  
* 收到请求，先获取集群的状态信息
* 根据路由信息计算id是在哪一个分片上
* 因为一个分片可能有多个副本分片，所以上述的计算结果是一个列表
* 调用transportServer的sendRequest方法向目标发送请求
* 上一步的方法内部会检查是否为本地node，如果是的话就不会发送到网络，否则会异步发送
* 等待数据节点回复，如果成功则返回数据给客户端，否则会重试
* 重试会发送上述列表的下一个。  


在数据节点有个shardTransporthander的messageReceived的入口专门接收协调节点发送的请求。  
```
    private class ShardTransportHandler implements TransportRequestHandler<Request> {

        @Override
        public void messageReceived(final Request request, final TransportChannel channel, Task task) throws Exception {
            if (logger.isTraceEnabled()) {
                logger.trace("executing [{}] on shard [{}]", request, request.internalShardId);
            }
            asyncShardOperation(request, request.internalShardId, new ChannelActionListener<>(channel, transportShardAction, request));
        }
    }
```
最终会调用org.elasticsearch.index.get.ShardGetService#innerGet的方法
```
    private GetResult innerGet(String type, String id, String[] gFields, boolean realtime, long version, VersionType versionType,
                               long ifSeqNo, long ifPrimaryTerm, FetchSourceContext fetchSourceContext) {
        fetchSourceContext = normalizeFetchSourceContent(fetchSourceContext, gFields);
        if (type == null || type.equals("_all")) {
            DocumentMapper mapper = mapperService.documentMapper();
            type = mapper == null ? null : mapper.type();
        }

        Engine.GetResult get = null;
        if (type != null) {
            Term uidTerm = new Term(IdFieldMapper.NAME, Uid.encodeId(id));
            get = indexShard.get(new Engine.Get(realtime, realtime, type, id, uidTerm)
                .version(version).versionType(versionType).setIfSeqNo(ifSeqNo).setIfPrimaryTerm(ifPrimaryTerm));
            assert get.isFromTranslog() == false || realtime : "should only read from translog if realtime enabled";
            if (get.exists() == false) {
                get.close();
            }
        }

        if (get == null || get.exists() == false) {
            return new GetResult(shardId.getIndexName(), type, id, UNASSIGNED_SEQ_NO, UNASSIGNED_PRIMARY_TERM, -1, false, null, null, null);
        }

        try {
            // break between having loaded it from translog (so we only have _source), and having a document to load
            return innerGetLoadFromStoredFields(type, id, gFields, fetchSourceContext, get, mapperService);
        } finally {
            get.close();
        }
    }
```
InternalEngine#get过程会加读锁。处理realtime选项，如果为true，则先判断是否有数据可以刷盘，然后调用Searcher进行读取。Searcher是对IndexSearcher的封装。

从ES 5.x开始不会从translog中读取，只从Lucene中读。realtime的实现机制变成依靠refresh实现

参考文章：https://jiankunking.com/elasticsearch-get-source-code-analysis.html
https://www.cnblogs.com/wangnanhui/articles/13273764.html
它其中的流程主要是：
* shardOperation先检查是否需要refresh，然后调用indexShard.getService().get()读取数据并存储到GetResult中。读取及过滤 在ShardGetService#get()
* GetResult getResult = innerGet(……)；
获取结果。GetResult类用于存储读取的真实数据内容。核心的数据读取实现在ShardGetService#innerGet

InternalEngine#get过程会加读锁。处理realtime选项，如果为true，则先判断是否有数据可以刷盘，然后调用Searcher进行读取。Searcher是对IndexSearcher的封装  
从ES 5.x开始不会从translog中读取，只从Lucene中读。realtime的实现机制变成依靠refresh实现。参考官方链接  


GET是根据Document _id 哈希找到对应的shard的。
根据Document _id查询的实时可见是通过依靠refresh实现的。  




### 2.3. search流程  
对于Search类请求，elasticsearch请求是查询lucence的Segment，前面的写入详情流程也分析了，新增的文档会定时的refresh到磁盘中，所以搜索是属于近实时的。    
elasticsearch的search有两个搜索类型  
* dfs_query_and_fetch，流程复杂一些，但是算分的时候使用了全局的一些指标，这样获取的结果可能更加精确一些。
* query_and_fetch，默认的搜索类型。  

具体的搜索的流程图如下：  


 elasticsearch作为一款分布式搜索引擎，search一般还是会经过两个步骤的。  
![](search流程.png)  
图片来自官网    
查询阶段包含以下三个步骤:
* 客户端发送一个 search 请求到 Node 3 ， Node 3 会创建一个大小为 from + size 的空优先队列。
* Node 3 将查询请求转发到索引的每个主分片或副本分片中。每个分片在本地执行查询并添加结果到大小为 from + size 的本地有序优先队列中。
* 每个分片返回各自优先队列中所有文档的 ID 和排序值给协调节点，也就是 Node 3 ，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。
* 当一个搜索请求被发送到某个节点时，这个节点就变成了协调节点。 这个节点的任务是广播查询请求到所有相关分片并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。



所有的搜索系统一般都是两阶段查询，第一阶段查询到匹配的DocID，第二阶段再查询DocID对应的完整文档，这种在Elasticsearch中称为query_then_fetch，还有一种是一阶段查询的时候就返回完整Doc，在Elasticsearch中称作query_and_fetch，一般第二种适用于只需要查询一个Shard的请求。  
除了一阶段，两阶段外，还有一种三阶段查询的情况。搜索里面有一种算分逻辑是根据TF（Term Frequency）和DF（Document Frequency）计算基础分，但是Elasticsearch中查询的时候，是在每个Shard中独立查询的，每个Shard中的TF和DF也是独立的，虽然在写入的时候通过_routing保证Doc分布均匀，但是没法保证TF和DF均匀，那么就有会导致局部的TF和DF不准的情况出现，这个时候基于TF、DF的算分就不准。为了解决这个问题，Elasticsearch中引入了DFS查询，比如DFS_query_then_fetch，会先收集所有Shard中的TF和DF值，然后将这些值带入请求中，再次执行query_then_fetch，这样算分的时候TF和DF就是准确的，类似的有DFS_query_and_fetch。这种查询的优势是算分更加精准，但是效率会变差。另一种选择是用BM25代替TF/DF模型。
在新版本Elasticsearch中，用户没法指定DFS_query_and_fetch和query_and_fetch，这两种只能被Elasticsearch系统改写。