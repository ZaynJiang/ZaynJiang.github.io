## 历年版本

| **版本号** | **发布日期**   | **多少个次要版本迭代** | **历时** |
| ---------- | -------------- | ---------------------- | -------- |
| 8.0        | 2022年2月11日  | 最新8.14.1             | 至今     |
| 7.0        | 2019年4月11日  | 17个次要版本           | 48个月   |
| 6.0        | 2017年11月15日 | 8个次要版本            | 17个月   |
| 5.0        | 2016年10月27日 | 6个次要版本            | 13个月   |

## 8.x功能特性新增

### 向量搜索

* 在 8.0版本 中

  提供新的kNN 搜索 API。这是一个里程碑的版本，在此之前，ES虽然支持向量检索，但是是以脚本的方式提供的。虽然此方法可以保证准确的结果，但它通常会导致搜索速度缓慢，并且无法很好地适应大型数据集。它的性能是极低的。在8.0中引入了 kNN search API | Elasticsearch Guide [8.0] | Elastic的技术预览版。使用dense_vector字段，k-nearest neighbor (kNN) search | Elasticsearch Guide [8.0] | Elastic可找到与查询向量最接近的k个向量（通过相似性度量来衡量）。kNN 通常用于为推荐引擎提供支持，并根据自然语言处理 (NLP) 算法对相关性进行排名。以前，Elasticsearch 仅支持使用script_score带有向量函数的查询进行精确的 kNN 搜索。虽然此方法可以保证准确的结果，但它通常会导致搜索速度缓慢，并且无法很好地适应大型数据集。作为索引速度较慢和准确性不完美的代价，新的 kNN 搜索 API 允许您以更快的速度对更大的数据集运行近似 kNN 搜索。

* 在 8.4版本中

  把knn搜索加入到search API中，以支持 ANN 搜索。它由与旧 _knn_search端点相同的 Lucene ANN 功能提供支持。该knn选项可以与其他搜索功能（例如查询和聚合）结合使用。

* 在 8.7版本中

  允许多个 KNN 搜索子句。某些向量搜索场景需要使用几个 kNN 子句进行相关性排名，例如，当基于多个字段进行排名时，每个字段都有自己的向量，或者当文档包含图像向量和文本向量时。用户可能希望获得基于所有这些 kNN 子句的组合的相关性排名。

* 在 8.8版本 中

  将 Introducing Elastic Learned Sparse Encoder: Elastic’s AI model for semantic search — Elastic Search Labs 模型引入到我们的机器学习模型库中，您可以开箱即用。ELSER 通过启用语义搜索来提高搜索结果的相关性。这种搜索方法考虑单词的含义，而不是仅仅依赖字面术语。ELSER 是一种预训练的域外稀疏向量模型，无需对特定源数据进行微调。它从一开始就为您提供相关的搜索结果。并且在此版本中，还提升了KNN检索语法，加入了similarity参数，允许过滤给定相似性之外的最近邻结果。

  试用案例参考：https://blog.csdn.net/UbuntuTouch/article/details/131183508

> Elastic Learned Sparse EncodeR（或 ELSER）是一种由 Elastic 训练的 NLP 模型，使你能够使用稀疏向量表示来执行语义搜索。 语义搜索不是根据搜索词进行字面匹配，而是根据搜索查询的意图和上下文含义检索结果

* 在8.9版本中。

  text_embedding query_vector_builderkNN 搜索的扩展普遍可用。

* 在8.10版本中。

  启用跨段并行 knn 搜索，使得 knn 查询对由多个段组成的分片更快，来优化knn搜索的性能
  

> 参考链接：https://www.elastic.co/guide/en/elasticsearch/reference/8.9/knn-search.html#exact-knn
>
> 在过去的两年间，官方一直致力于ES矢量搜索的优化。在 Elasticsearch 7.0版中，就针对高维矢量引入了字段类型。在 ES 7.3和 7.4 版本中，又引入了对矢量相似函数的支持。
>
> 在 Elasticsearch 8.0 版中，自然语言处理 (NLP) 模型原生支持直接引入了 Elasticsearch，让矢量搜索功能更容易实现。此外，Elasticsearch 8.0 版还包含了对近似最近邻搜索的原生支持，因此可以快速且大规模地比较基于矢量的查询与基于矢量的文档语料库
>
> https://mp.weixin.qq.com/s/ptjciKeNGA4ohQPzSKZcNg
>

#### 核心机制

对于存储嵌入，dense_vector 字段有两种主要的索引类型可供选择：

* flat 类型（包括 flat 和 int8_flat）：存储原始向量，不添加 HNSW 数据结构。使用 flat 索引类型的 dense_vector 将始终使用精确的 kNN，kNN 查询将执行精确查询而不是近似查询。

* HNSW 类型（包括 hnsw 和 int8_hnsw）：创建 HNSW 数据结构，允许使用近似 kNN 搜索。

  使用近似值而不是考虑所有文档。 为了提供 kNN 的有效近似，Elasticsearch 和 Lucene 使用分层导航小世界 HNSW (Hierachical Navigation Small Worlds)。

  HNSW 是一种图数据结构，它维护不同层中靠近的元素之间的链接。 每层都包含相互连接的元素，并且还与其下方层的元素相连接。 每层包含更多元素，底层包含所有元素

  ![image-20240703201144177](image-20240703201144177.png) 

  HNSW 的效果取决于多个因素：

  它是如何构建的。HNSW 的构建过程会考虑一定数量的候选节点，作为某一特定节点的近邻。增加考虑的候选节点数量会使结构更精确，但会在索引时花费更多时间。dense vector index_options 中的 ef_construction 参数用于此目的。

  搜索时考虑的候选节点数量。在寻找更近结果时，过程会跟踪一定数量的候选节点。这个数量越大，结果越精确，但搜索速度会变慢。kNN 参数中的 num_candidates 控制这种行为。

  我们搜索的分段数量。每个分段都有一个需要搜索的 HNSW 图，其结果需要与其他分段图的结果结合。分段越少，搜索的图就越少（因此速度更快），但结果集的多样性会减少（因此精度较低）。

  总的来说，HNSW 在性能和召回率之间提供了良好的权衡，并允许在索引和查询两方面进行微调。

  使用 HNSW 进行搜索可以在大多数情况下通过 kNN 搜索部分完成。对于更高级的用例，也可以使用 kNN 查询，例如：

  将 kNN 与其他查询结合（作为布尔查询或固定查询的一部分）
  使用 function_score 微调评分
  提高聚合和字段折叠（field collapse）的多样性

### 支持NLP能力

* 在8.0版本中

  可以上传在 Elasticsearch 之外训练的PyTorch模型，并在摄取时使用它们进行推理。第三方模型支持将现代自然语言处理 (NLP)和搜索用例引入 Elastic Stack，例如：

  * Fill-mask
  * Named entity recognition (NER)
  * Text classification
  * Text embedding
  * Zero-shot classification

* 在8.2版本中

  可以统计数据 NLP 推理速度。有三项新统计数据是：

  每分钟峰值吞吐量
  最后一分钟吞吐量
  平均推理时间毫秒最后分钟
  目的是表明推理当前是否满足要求，或者集群是否需要扩展以满足需求。最后一分钟的统计数据可以快速反馈以显示机器学习节点扩展的效果

### 时间序列

在 8.8 版本中，Elasticsearch 提供对时间序列数据流 (TSDS) 索引的支持。TSDS 索引是包含时间序列指标数据作为数据流一部分的索引。Elasticsearch 将传入文档路由到 TSDS 索引中，以便特定时间序列的所有文档都位于同一分片上，然后按时间序列和时间戳对分片进行排序。这种结构有几个优点：

* 同一时间序列的文档在分片上彼此相邻，因此在磁盘上彼此相邻存储，因此操作系统页面更加同质并且压缩效果更好，从而大幅降低 TCO。
* 时间序列的分析通常涉及比较每两个连续的文档（样本）、检查给定时间窗口中的最后一个文档等，当下一个文档可能位于任何分片上并且实际上位于任何索引上时，这是相当复杂的。按时间序列和时间戳排序可以改进分析，无论是在性能方面还是在我们添加新聚合的能力方面。
* 最后，作为指标数据时间序列索引生命周期管理的一部分，Elasticsearch 启用下采样操作。当对索引进行下采样时，Elasticsearch 会保留一个文档，其中包含时间序列中每个时间段的统计摘要。然后，支持的聚合可以在数据流上运行，并包括下采样索引和原始数据索引，而用户无需意识到这一点。还支持对下采样索引进行下采样，以达到更粗略的时间分辨率。

Time series data stream (TSDS) | Elasticsearch Guide [8.7] | Elastic是一项用于优化时间序列数据的 Elasticsearch 索引的功能。这涉及对索引进行排序以实现更好的压缩并使用合成 _source 来减小索引大小。因此，TSDS 指数明显小于包含相同数据的非时间序列指数。TSDS 对于管理大量时间序列数据特别有用。并且提供了采样优化算法，可减少 Elasticsearch 时间序列索引中存储的文档数量，从而缩小索引并改善查询延迟。这种优化是通过预先聚合时间序列索引来实现的，使用 time_series 索引模式来识别时间序列。下采样被配置为 ILM 中的一项操作，使其成为管理 Elasticsearch 中大量时间序列数据的有用工具

### 存储及性能优化

#### 底层存储优化

* keyword、match_only_text、 和text字段的存储节省
  在8.0版本中更新了倒排索引（一种内部数据结构），以使用更节省空间的编码。这种变化将有利于keyword各个领域、 match_only_text各个领域，以及在较小程度上的text各个领域。在我们使用应用程序日志的基准测试中，这意味着字段索引大小message（映射为match_only_text）减少了 14.4%，磁盘占用空间总体减少了 3.5%。

* 更快地索引geo_point、geo_shape和范围字段
  在8.0版本中优化了多维点的索引速度，多维点是用于geo_point、geo_shape和范围字段的内部数据结构。Lucene 级基准测试报告称，这些字段类型的索引速度提高了 10-15%。主要由这些字段组成的 Elasticsearch 索引和数据流可能会显着提高索引速度。

* 管道处理性能提升
  在8.3版本中通过避免（深度）递归，改进了具有同步处理器的管道的管道执行逻辑。在我们模拟日志记录用例的夜间基准测试中，这导致摄取管道上花费的 CPU 时间减少了 10%，整体摄取速度提高了 3%。

* filters/range/date_histogram aggs 性能提升
  在8.4版本中，在没有子聚合时，对聚合进行提速。这非常常见，例如，Kibana 的发现选项卡顶部的直方图没有date_histogram任何子聚合。在我们的集会测试中，该特定聚合速度加快了约 85%，从 250 毫秒降至 30 毫秒。

* 使用显式“_id”改进了 get、mget 和索引的性能
  在8.7中，将布隆过滤器的误报率_id从约 10% 降低至约 1%，从而减少了段中不存在术语时的 I/O 负载。这可以提高通过 检索文档时的性能_id，这种情况发生在执行 get 或 mget 请求时，或者发出_bulk提供显式 _id 的请求时。

* Encode using 40, 48 and 56 bits per value
  在8.8版本中，做了编码的提升。

  使用编码如下： * 对于每个值采用 [33, 40] 位的值，使用每个值 40 位进行编码 * 对于每个值采用 [41, 48] 位的值，使用每个值 48 位进行编码 * 对于采用 [ 49, 56] 每个值位，使用每个值 56 位进行编码

  这是对 ForUtils 使用的编码的改进，ForUtils 不对每个值超过 32 位的值应用任何压缩。

  请注意，每个值的 40、48 和 56 位表示字节的精确倍数（每个值 5、6 和 7 个字节）。因此，我们总是使用比长值所需的 8 个字节少 3、2 或 1 个字节的值来写入值。

  看看存储字节的节省，对于 128 个（长）值的块，我们通常会存储 128 x 8 字节 = 1024 字节，而现在我们有以下内容： * 每个值 40 位：写入 645 字节而不是 1024 字节，节省379 字节 (37%) * 每个值 48 位：写入 772 字节而不是 1024，节省 252 字节 (24%) * 每个值 56 位：写入 897 字节而不是 1024，节省 127 字节 (12%)

  还将压缩应用于规格指标，假设每个值超过 32 位的压缩值对于浮点值效果很好，因为浮点值的表示方式（IEEE 754 格式）。

* 并发索引和搜索下更好的索引和搜索性能
  在8.9版本中，提升并发索引和搜索的性能。当像匹配短语查询或术语查询这样的查询针对常量关键字字段时，我们可以跳过分片上的查询执行，其中查询被重写以不匹配任何文档。我们利用包括常量关键字字段的索引映射，并以这样的方式重写查询：如果常量关键字字段与索引映射中定义的值不匹配，我们将重写查询以匹配任何文档。这将导致分片级别请求在数据节点上执行查询之前立即返回，从而完全跳过分片。在这里，我们利用尽可能跳过分片的功能来避免不必要的分片刷新并改善查询延迟（通过不执行任何与搜索相关的 I/O）。避免这种不必要的分片刷新可以改善查询延迟，因为搜索线程不再需要等待不必要的分片刷新。不匹配查询条件的分片将保持搜索空闲状态，索引吞吐量不会受到刷新的负面影响。在引入此更改之前，查询命中多个分片，包括那些没有与搜索条件匹配的文档的分片（考虑使用索引模式或具有许多支持索引的数据流），可能会导致“分片刷新风暴”，从而增加查询延迟搜索线程等待所有分片刷新完成后才能启动和执行搜索操作。引入此更改后，搜索线程只需等待包括相关数据的分片刷新完成。请注意，分片预过滤器的执行以及重写发生的相应“可以匹配”阶段取决于所涉及的分片总数以及是否至少有一个分片返回非空结果（请参阅 pre_filter_shard_size设置为了解如何控制这种行为）。Elasticsearch 在所谓的“可以匹配”阶段对数据节点进行重写操作，利用这样一个事实：此时我们可以访问索引映射并提取有关常量关键字字段及其值的信息。这意味着我们仍然将搜索查询从协调器节点“扇出”到相关数据节点。在协调器节点上不可能基于索引映射重写查询，因为协调器节点缺少索引映射信息。


> 在过去的三年间，ES一直致力于：减少内存使用（从而让每个节点托管更多的数据），降低查询开销（在大规模部署中成效尤为明显），引入一些全新的功能来提高相关性。
>
> 例如，在 7.x 系列版本中，官方提高了日期直方图和搜索聚合的速度，增强了页面缓存的性能，并创建了一个新的“pre-filter”搜索短语。此外，官方还通过减少内存堆完全支持 ARM 架构，引入全新的方式以使用更少的存储空间，以及让用户能够借助新的 Frozen Phase 和 Searchable Snapshot 轻松将计算与存储分离开来，从而降低了资源需求（即：降低了客户的总体拥有成本）

#### 更高效的快照

8.5版本中，快照操作的开销已显着减少。快照现在以更高效的顺序运行，并且需要的网络流量比以前少得多。

### 安全

从 ES 7.x 开始，官方提供免费的安全功能，不过仅限于索引级别的安全设置，而字段和文档级别的需要付费。

从 8.0 开始，ES 简化了安全功能。自管理集群默认启用 Elastic Stack 安全性，配置工作几乎为零（其实8.x 的安全配置更麻烦了，知识默认启用了安全功能而已）。

8.x 默认启用了以下功能：

- 用户认证
- 具有基于角色的访问控制的用户授权
- Kibana Spaces 多租户
- 使用 TLS 的加密节点到节点通信
- Elasticsearch 和 Kibana 之间使用 TLS 进行加密通信
- 使用 HTTPS 与 Elasticsearch API 进行加密通信

第一次运行 Elasticsearch 时，Elasticsearch 将自动执行以下任务：

- **TLS 设置**：它将生成证书颁发机构、传输和 HTTP 层证书，以及 TLS 舞蹈所需的所有相关“好东西”——当然包括私钥、证书指纹等。它还将存储密码安全。
- **设置安全设置**：这还包括用于传输和 HTTP 加密的设置：xpack.security.ssl.http.enabled 和 xpack.security。ssl.transport.enabled
- **准备传入连接**：它将生成一种特殊类型的令牌，即*注册令牌*，它封装了 Kibana 连接到 Elasticsearch 所需的所有信息，并启用其注册 API，该 API 会监听新连接。
- 为ES 内置超级用户生成密码。

### 地图功能

地图功能更加丰富

## 应用场景

### 以图搜图

也可以通过向量嵌入进行分析。 在此类向量嵌入上训练的算法可以对图像进行分类、识别对象并在其他图像中检测它们、搜索相似图像以及将所有类型的图像（以及视频）分类为不同的类别。 Google Lens 使用的图像识别技术是一种常用的图像分析工具。

https://mp.weixin.qq.com/s/fM-xuOibkYPwC4mVijBR2g

### 搜索引擎(语义检索)

使用向量嵌入来检索信息并帮助识别语义关系。 向量嵌入帮助搜索引擎接受用户查询并返回相关的主题网页、推荐文章、更正查询中拼写错误的单词以及建议用户可能认为有帮助的类似相关查询。 该应用程序通常用于支持语义搜索。

参考：https://blog.csdn.net/star1210644725/article/details/134748429

#### 区别

语义搜索与关键字搜索
        语义搜索和关键字搜索之间的区别在于，关键字搜索会返回单词与单词、单词与同义词或单词与相似单词的匹配结果。语义搜索旨在匹配查询中单词的含义。在某些情况下，语义搜索可能不会生成直接的单词匹配结果，但会符合用户的意图。

关键字搜索引擎使用的是查询扩展或简化工具，如同义词或省略词。它们还会使用自然语言处理和理解工具，如允许拼写错误、词汇切分和规范化。另一方面，语义搜索能够通过使用矢量搜索返回与含义相匹配的查询结果。

以“chocolate milk”（巧克力牛奶）为例。 语义搜索引擎会区分“chocolate milk”（巧克力牛奶）和“milk chocolate”（牛奶巧克力）。 尽管查询中的关键字相同，但它们的书写顺序会影响含义。作为人类，我们知道“milk chocolate”（牛奶巧克力）是指各种巧克力，而“chocolate milk”（巧克力牛奶）是巧克力口味的牛奶

#### 语义搜索方案

![image-20240703171833992](image-20240703171833992.png) 

 第一步，先是依靠深度学习大模型，将文本内容（这里并不局限于文本，还可以是不同模态的数据，例如图片和声音）到更多维度的空间。通常512维度，甚至是1024维度的空间中。映射的基本原则是，越相近的内容，空间距离越近。这里举个例子，苹果和橘子的空间距离是1，苹果和猫咪的空间距离应该是大于1的，而猫咪和cat的空间距离是小于1的。利用维度空间的距离，来算事物的相似性，或者说问题和答案的相似度。

第二步，将映射后的数据存储在向量数据库（数据在多维度的空间位置，使用多位浮点类型的数组维护的，把它称为向量）。   

第三步，将问题使用相同的模型，也映射成向量。

第四步，计算问题和答案的空间距离。这里比较普遍的做法是计算cosin值。即向量检索

#### 语义搜索实践

https://cloud.tencent.com/developer/article/2379632

https://www.elastic.co/cn/blog/text-similarity-search-with-vectors-in-elasticsearch

#### 注意事项

* 使用eland，它可以根据我们指定的模型id，去hugging face上拉取模型

* es所谓的机器学习能力，仅支持文本类操作的模型。官方一直在说拥有跨模态的能力。实际上es并不支持，将图片转向量的模型导入到es中（例如常用的CLIP多模态模型，其实它是两部分，双塔模型，一个是将图片做embedding，转成向量。另一个模型是将我们的文本内容做embedding转为向量。其中图片转向量的模型，在es中是不支持上传的，文本转向量的模型是可以上传的）。如下所示，上传clip 将图片转为向量的模型。会报错

  ```
  docker run -it   -v /u01/isi/.cache/huggingface/hub/sentence-transformers/clip-vit-base-patch32:/eland/sentence-transformers/clip-vit-base-patch32   --rm elastic/eland \
  eland_import_hub_model \
  --url http://elastic:123123@10.99.100.49:9200 \
  --hub-model-id sentence-transformers/clip-vit-base-patch32 \
  --task-type text_embedding \
  --start
  ```

  

### 自然语言处理 (NLP)

广泛使用向量嵌入来执行情感分析、命名实体识别、文本分类、机器翻译、问答和文档相似性等任务。 通过使用嵌入，算法可以更有效地理解和处理文本相关数据

* **文本情感分析确定** (text sentiment analysis) 一段文本中表达的情感或情感，通常将其分类为积极、消极或中性。 它用于分析产品评论、社交媒体帖子和客户反馈。

  与文本情感分析相关的**毒性检测** (toxicity detection) 可识别在线攻击性或有害语言。 它帮助在线社区的版主在在线讨论、评论或社交媒体帖子中维护一个相互尊重的数字环境。

* **意图识别** (intent recoginition) 是文本情感分析的另一个子集，用于理解用户文本输入背后的目的（或意图）。 聊天机器人和虚拟助理通常使用意图识别来响应用户查询。

* **二元分类** (biary classification) 将文本分为两个类或类别之一。 一个常见的例子是垃圾邮件检测，它将文本（例如电子邮件或消息）分类为垃圾邮件或合法类别，以自动过滤掉未经请求的和可能有害的内容。

* **多类分类** (multiclass classification) 将文本分为三个或更多不同的类或类别。 这使得从新闻文章、博客文章或研究论文等内容中组织和检索信息变得更加容易。

* **主题分类** (topic categorization) 与多类分类相关，将文档或文章分组为预定义的主题或主题。 例如，新闻文章可以分为政治、体育和娱乐等主题。

* **语言识别** (language identification) 确定一段文本的书写语言。 这在多语言环境和基于语言的应用程序中非常有用。

* **命名实体识别** (named entity recognition) 侧重于对文本中的命名实体进行识别和分类，例如人名、组织、位置和日期。

* **问题分类涉** (question classifcation) 及根据预期答案类型对问题进行分类，这对于搜索引擎和问答系统非常有用。

### 个性化推荐系统

利用向量嵌入来捕获用户偏好和项目特征。 它们根据用户与向量中的项目之间的密切匹配，帮助将用户个人资料与用户可能喜欢的项目（例如产品、电影、歌曲或新闻文章）进行匹配。 一个熟悉的例子是 Netflix 的推荐系统。 有没有想过它是如何选择符合你口味的电影的？ 它通过使用项目相似性度量来建议与用户通常观看的内容相似的内容。

### 异常检测

算法使用向量嵌入来识别各种数据类型中的异常模式或异常值。 该算法对代表正常行为的嵌入进行训练，以便它可以学习发现与规范的偏差，这些偏差可以根据嵌入之间的距离或相异性度量来检测。 这在网络安全应用程序中特别方便

在Elasticsearch中直接实现异常检测逻辑可能不是最直接的方法，但可以通过以下几种方式结合Elasticsearch进行异常检测：

- 基于相似度的异常检测

  - 将已知的正常样本存储为向量。
  - 当新数据进入时，计算其向量表示。
  - 使用Elasticsearch的向量搜索功能找到与新数据最相似的正常样本。
  - 如果新数据与任何正常样本的相似度都很低（即，在相似度阈值以下），则可以认为该数据是异常的。

- 基于聚类的异常检测

  （间接使用Elasticsearch）：

  - 在Elasticsearch外部使用聚类算法（如K-means、DBSCAN）对正常数据进行聚类。
  - 将聚类中心或聚类标签作为元数据与数据一起存储在Elasticsearch中。
  - 当新数据进入时，首先使用聚类算法判断其属于哪个聚类（或不属于任何聚类），然后根据聚类结果判断是否为异常。

- 结合时间序列分析

  - 如果数据有时间序列特性，可以结合时间序列分析技术（如ARIMA、STL分解）来识别异常模式。
  - 将这些分析的结果与Elasticsearch中的向量搜索结合，以更全面地检测异常

### **图分析**

使用图嵌入，其中图是由线（称为边）连接的点（称为节点）的集合。 每个节点代表一个实体，例如人、网页或产品，每条边代表这些实体之间的关系或连接。 这些向量嵌入可以做很多事情，从在社交网络中推荐朋友到检测网络安全异常（如上所述）。

### **音频和音乐**

也可以被处理和嵌入。 向量嵌入捕获音频特征，使算法能够有效地分析音频数据。 这可用于各种应用，例如音乐推荐、流派分类、音频相似性搜索、语音识别和说话人验证

### **抄袭检测**

根据文档与数据库中文档的匹配程度来检测抄袭

## 硬件选择

https://cloud.tencent.com/developer/article/2296488

## 版本升级

### 整体思路

如果是 8.1.3 之前的 8.X 版本，直接升级就可以。

如果是 7.X 版本，需要先升级至7.X 最新版：7.17.5（下图蓝色部分），然后再由 7.17.5 升级到我们期望的 8.X 版本（下图红色部分）。

根据官方文档https://www.elastic.co/guide/en/elasticsearch/reference/7.17/rolling-upgrades.html

![image-20240702165747571](image-20240702165747571.png) 

![image-20240702165820110](image-20240702165820110.png) 

### 升级风险

升级势必会有风险，且不可回退。

所以，建议提前做好集群的[**快照**](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484568&idx=1&sn=f22d369879359e9308a36dbc1b4964f4&chksm=eaa82cb0dddfa5a68e40962960363d69840221cd619b9d65f3676f47dfdc5bad9e086932d37a&scene=21#wechat_redirect)，以备不时之需。

这是大前提，大家一定要做好万全的准备！

### 步骤

4.1 7.13.0 升级到 7.17.5 实战

略