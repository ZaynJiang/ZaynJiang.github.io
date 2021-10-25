## 1. 开头
&emsp;&emsp;我们在前面已经知道elasticsearch底层的写入是基于lucence依进行doc写入的。elasticsearch作为一款分布式系统，在写入数据时还需要考虑很多重要的事项，比如:可靠性、原子性、一致性、实时性、隔离性、性能等多个指标。elasticsearch是如何做到的呢？下面我们针对elasticsearch的写入进行分析。
<br/>
## 2. lucence写
### 2.1. 增删改
&emsp;&emsp;elasticsearch拿到一个doc后调用lucence的api进行写入的。
```
 public long addDocument();
 public long updateDocuments();
 public long deleteDocuments();
```
&emsp;&emsp;如上面的代码所示，我们使用lucence的上面的接口就可以完成文档的增删改操作。在lucence中有一个核心的类IndexWriter负责数据写入和索引相关的工作。
```
//1. 初始化indexwriter对象
IndexWriter writer = new IndexWriter(new NIOFSDirectory(Paths.get("/index")), new IndexWriterConfig());

//2. 创建文档
Document doc = new Document();
doc.add(new StringField("empName", "王某某", Field.Store.YES));
doc.add(new TextField("content", "操作了某菜单", Field.Store.YES));

//3. 添加文档
writer.addDocument(doc);

//4. 提交
writer.commit();
```
&emsp;&emsp;以上代码演示了最基础的lucence的写入操作，主要涉及到几个关键点：
* 初始化，Directory是负责持久化的，他的具体实现有很多，有本地文件系统、数据库、分布式文件系统等待，elasticsearch默认的实现是本地文件系统。
* Document就是es中的文档，FiledType定义了很多索引类型。这里列举几个常见的类型：
  * stored，字段是否保存
  * tokenized，代表是否做分词
  * indexOptions(NONE、DOCS、DOCS_AND_FREQS、DOCS_AND_FREQS_AND_POSITIONS、DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS)，倒排索引的选项
  * docValuesType，正排索引，建立一个docid到field的的一个列存储。
  * 一些其它的类型。
* IndexWriter在doc进行commit后，才会被持久化并且是可搜索的。
* IndexWriterConfig负责了一些整体的配置参数，并提供了方便使用者进行功能定制的参数：
  * Similarity，这个是搜索的核心参数，实现了这个接口就能够进行自定义算分。lucence默认实现了前面文章提到的TF-IDF、BM25算法。
  * MergePolicy，合并的策略。我们知道elasticsearch会进行合并，从而减少段的数量。
  * IndexerThreadPool，线程池的管理
  * FlushPolicy，flush的策略
  * Analyzer，定制分词器
  * IndexDeletionPolicy，提交管理

### 2.2. 并发模型
&emsp;&emsp;上面我们知道indexwriter负责了elasticsearch索引增删改查。那它具体是如何管理的呢？  
#### 2.2.1. 基本操作
关键点：
* DocumentsWriter处理写请求，并分配具体的线程DocumentsWriterPerThread
* DocumentsWriterPerThread具有独立内存空间，对文档进行处理
* DocumentsWriter触发一些flush的操作。
* DocumentsWriterPerThread中的内存In-memory buffer会被flush成独立的segement文件。
* 对于这种设计，多线程的写入，针对纯新增文档的场景，所有数据都不会有冲突，非常适合隔离的数据写入方式  
  
#### 2.2.2. 更新
&emsp;&emsp;Lucene的update和数据库的update不太一样，Lucene的更新是查询后删除再新增。
* 分配一个操作线程
* 在线程里执行删除
* 在线程里执行新增

#### 2.2.3. 删除
&emsp;&emsp;上面已经说了，在update中会删除，普通的也会删除，lucence维护了一个全局的删除表，每个线程也会维护一个删除表，他们双向同步数据

* update的删除会先在内部记录删除的数据，然后同步到全局表中
* delete的删除会作用在Global级别，后异步同步到线程中。
* Lucene Segment内部，数据实际上其实并不会被真正删除，Segment内部会维持一个文件记录，哪些是docid是删除的，在merge时，相应的doc文档会被真正的删除。

#### 2.2.4. flush和commit
&emsp;&emsp;每一个WriterPerThread线程会根据flush策略将文档形成segment文件，此时segment的文件还是不可见的，需要indexWriter进行commit后才能被搜索。  
这里需要注意：  
elasticsearch的refresh对应于lucene的flush，elasticsearch的flush对应于lucene的commit，elasticsearch在refresh时通过其它方式使得segment变得可读。



#### 2.2.5. merge
&emsp;&emsp;merge是对segment文件合并的动作，这样可以提升查询的效率并且可以真正的删除的文档

<br/>

#### 2.2.6. 小结 
&emsp;&emsp;在这里我们稍微总结一下，一个elasticsearch索引分配对应一个完整的lucene索引, 而一个lucene索引对应多个segment。我们在构建同一个lucene索引的时候, 可能有多个线程在并发构建同一个lucene索引, 这个时候每个线程会对应一个DocumentsWriterPerThread, 而每个 DocumentsWriterPerThread会对应一个index buffer. 在执行了flush以后, 一个 DocumentsWriterPerThread会生成一个segment。  
&emsp;&emsp;对于index buffer和elasticsearch node的关系的话, 那就是在索引期间, 一个es node对应多个index buffer, 至于对应几个index buffer, 那就取决于当前有几个DocumentsWriterPerThread, 也就是说有几个并发线程在写同一个elasticsearch node的lucene索引


## 3. Elasticsearch的写
### 3.1. Elasticsearch整体的流程图
 &emsp;&emsp;在前面的文章已经讨论了写入的流程Elasticsearch  
![](写入流程.png)  
图片来自官网  
当写入文档的时候，根据routing规则，会将文档发送至特定的Shard中建立lucence。  
* 在Primary Shard上执行成功后，再从Primary Shard上将请求同时发送给多个Replica Shard
* 请求在多个Replica Shard上执行成功并返回给Primary Shard后，写入请求执行成功，返回结果给客户端  
  
注意上面的写入延时=主分片延时+max(Replicas Write),即写入性能如果有副本分片在，就至少是写入两个分片的延时延时之和。

### 3.2. Elasticsearch详细流程