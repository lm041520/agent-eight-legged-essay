# 04 Elasticsearch：从文本检索到 RAG 检索系统

> 适合对象：会调用搜索 API，但还不清楚分词、倒排索引、刷新、分片、相关性和向量检索如何共同决定结果的人。
>
> 学完目标：你能解释一段文本如何变成可搜索的倒排索引，能正确区分 `text` 与 `keyword`，能设计过滤、排序、分页、更新和删除策略，能把 BM25、向量检索和 RAG 的召回链路联系起来，并能根据延迟、召回率和资源指标排查搜索问题。

## 0. 先看一个知识库搜索请求

用户输入“如何处理 Redis 缓存击穿”，系统希望先过滤当前租户可见的文档，再按相关性返回片段：

```text
请求
  ↓
租户/权限过滤（filter）
  ↓
关键词召回（倒排索引 + BM25）
  + 向量召回（embedding + kNN）
  ↓
合并、去重、重排
  ↓
返回片段、文档标题、权限和分数
```

要得到可信结果，不能只记一个 `match` 示例。必须同时解决：文本怎样分析、哪些字段可过滤、索引何时可见、分页是否稳定、分片是否均衡、更新后旧版本何时回收，以及召回结果是否真的能支持生成答案。

> **核心判断：** Elasticsearch 是面向搜索的分布式文档引擎。它通过倒排索引快速找词，通过分析器把自然语言变成可比较的 token，通过分片和副本扩展容量与可用性；搜索结果的质量取决于字段建模、查询语义、相关性调参和数据生命周期，而不是“把数据库换成 ES”。

## 1. 文档、字段和映射：先定义要回答的问题

### 1.1 文档是搜索视角的对象

一个知识库 chunk 可以建模为：

```json
{
  "chunk_id": "doc-100#3",
  "document_id": 100,
  "tenant_id": "acme",
  "title": "Redis 缓存治理",
  "content": "缓存击穿发生在热点 key 同时过期……",
  "tags": ["redis", "cache"],
  "updated_at": "2026-08-07T10:00:00Z",
  "visibility": "team",
  "embedding": [0.012, -0.084, 0.231]
}
```

搜索字段和事实字段不一定一一对应。`document_id`、`tenant_id`、`visibility` 要精确过滤；`title`、`content` 要全文检索；`updated_at` 要排序和范围过滤；`embedding` 用于向量相似度。

### 1.2 `text` 与 `keyword`

`text` 会经过 analyzer 分词，适合全文检索；`keyword` 作为完整值索引，适合精确匹配、聚合和排序。一个字符串经常使用多字段：

```json
"title": {
  "type": "text",
  "fields": {"raw": {"type": "keyword"}}
}
```

```text
match title       → 查找标题中的词
term title.raw    → 精确匹配完整标题
sort title.raw    → 按完整字符串排序
```

把 `tenant_id` 映射成 `text`，会导致精确过滤和聚合结果不符合预期；把长正文只映射成 `keyword`，则无法按词检索。映射一旦写入数据，改变字段类型通常需要新索引和重建。

### 1.3 数组、对象与 nested

普通对象数组可能把不同元素的字段交叉匹配：

```json
"authors": [
  {"name": "A", "role": "reviewer"},
  {"name": "B", "role": "owner"}
]
```

如果要求同一作者同时满足 `name=A` 和 `role=owner` 不成立，应使用 `nested` 保留数组元素边界。`nested` 查询更准确，但会产生内部文档成本；只有确实需要元素级关联时才使用。

### 1.4 Dynamic mapping、mapping explosion 与 routing

Dynamic mapping 方便快速接入数据，但把未知字段自动加入 mapping 可能让用户自定义 JSON、日志或错误 payload 生成成千上万个字段，最终出现 mapping explosion、集群状态变大和查询变慢。对不受控字段应使用 `dynamic: strict`、`false` 或显式 `flattened` 等策略，并限制对象深度、字段数量和单文档大小；具体选项要按锁定的 Elasticsearch 版本验证。

多租户数据还要决定是否使用 routing。把 `tenant_id` 作为 routing 可以减少某些查询的 fan-out，但会让大租户形成热点，扩容和迁移也更复杂。routing 只是一种分片定位优化，权限 filter 仍然必须存在；不能因为命中了目标分片就认为已经完成了权限校验。

## 2. 一段文本如何变成倒排索引

### 2.1 分析链

写入 `"Cache Misses are Expensive"` 时，分析器可能执行字符过滤、分词、大小写归一、停用词处理和词干化，得到类似：

```text
原文: Cache Misses are Expensive
token: cache, miss, expensive
```

查询文本也要经过兼容的分析链，否则写入和查询使用不同词形会造成召回下降。中文、英文、代码、版本号和专有名词需要不同的 tokenizer；不要因为默认 analyzer 能运行，就认为它适合业务语言。

### 2.2 倒排表和位置

倒排索引把“词”映射到包含它的文档：

```text
cache      → chunk-1, chunk-8, chunk-20
expensive  → chunk-1, chunk-9
```

索引还可保存词频、文档频率和位置信息，用于相关性和短语查询。搜索不用逐篇扫描正文，而是先查词表再合并文档列表，因此对大规模文本更快。

### 2.3 BM25 解决什么问题

BM25 综合词频、逆文档频率和字段长度归一：常见词贡献较小，稀有词贡献较大，长文档不会因包含更多词而无条件占优。它不是语义理解模型，无法稳定理解“缓存击穿”和“热点 key 过期”之间的隐含关系，所以知识库常把关键词召回和向量召回结合。

## 3. 写入、刷新、段合并与删除

### 3.1 近实时而不是立即可搜

写入请求成功表示文档已被接受并记录，但新文档要等 refresh 后才进入可搜索的 segment。默认 refresh 间隔通常带来亚秒级到数秒级的可见延迟；搜索刚上传的文档时，应按业务需要选择等待刷新、查询数据库状态，或向用户明确“正在建立索引”。

频繁强制 refresh 会产生大量小 segment，增加 I/O 和合并压力，不应把 `refresh=true` 当作默认可靠性开关。

### 3.2 segment 是不可变的

倒排索引写成 segment 后通常不原地修改。更新文档会写入新版本，旧版本在合并前仍占空间；删除先记录删除标记，真正回收要等 segment merge。

因此：

- 高频更新字段会造成写放大；
- 大量小批量写入比合理批量更浪费资源；
- 查询结果在版本切换期间需要依赖 `_seq_no`、`_primary_term` 或应用版本控制避免旧写覆盖新写。

### 3.3 translog 与持久化

Elasticsearch 通过 translog 记录尚未安全写入 segment 的变更。刷新解决“可搜索”，fsync/flush 解决“磁盘持久化”层面的不同问题。不要把搜索可见时间、请求成功时间、数据可恢复时间混成同一个指标。

### 3.4 bulk、refresh 与写入背压

批量写入能减少 HTTP 往返，但单批过大可能占用协调节点内存、造成长 GC，并让失败重试变得难以定位。客户端应按文档字节数和条目数设置上限，读取每条 bulk item 的成功/失败结果，把可重试的 429/超时与 mapping、权限等永久错误分开处理。

高写入压力下可以适当放宽 refresh、限制并发和等待队列，优先保护磁盘水位、线程池和 segment merge；不能无限堆积客户端请求。批量回填要记录 checkpoint、失败文档和版本，恢复后只重放未完成批次。写入背压最终要传回任务队列和 API 状态，否则 ES 只是把压力从应用线程转移到集群。

## 4. 查询：相关性、过滤和排序要分开

### 4.1 query 与 filter

影响相关性的全文条件放在 query context；只判断真假、无需打分的租户、权限、状态和时间范围放在 filter context：

```json
{
  "query": {
    "bool": {
      "must": [
        {"multi_match": {"query": "缓存击穿", "fields": ["title^2", "content"]}}
      ],
      "filter": [
        {"term": {"tenant_id": "acme"}},
        {"term": {"visibility": "team"}},
        {"range": {"updated_at": {"gte": "now-180d"}}}
      ]
    }
  }
}
```

过滤条件应尽早缩小候选集，并利用缓存；权限过滤不能在召回之后才做，否则可能把无权文档送进重排或生成模型。

### 4.2 `match`、`term` 和短语

- `match` 适合经过分析的全文字段；
- `term` 适合 keyword、数字、枚举等精确值；
- `match_phrase` 关注词的相邻顺序；
- `prefix`、`wildcard`、正则查询可能扫描大量词项，应限制字段和输入长度。

查询 DSL 的名字不应脱离字段 mapping 记忆。先问“这个字段是全文还是精确值”，再选查询。

### 4.3 排序与 tie-breaker

只按 `_score` 排序时，同分文档的顺序可能变化；只按时间排序又会失去相关性。可以使用 `_score`、`updated_at` 和稳定的 `chunk_id` 组合排序，并在分页时保存一致的搜索上下文。

## 5. 分页：深分页为什么越来越慢

### 5.1 from/size 适合浅页

第 1000 页的 `from=9990&size=10` 要让每个分片先找出前 10000 条，再由协调节点合并，内存和网络成本随页码增加。在线搜索不应允许用户任意跳到极深页。

### 5.2 search_after 与 PIT

`search_after` 使用上一页最后一条的排序值继续向后取，适合稳定的下一页；但如果数据在分页中更新，结果可能漂移。Point in Time（PIT）保存一个一致的搜索视图，再结合 `search_after`，可以减少分页期间段变化带来的不一致。

PIT 不是无限期快照，会占用资源，应设置过期时间并在用户停止翻页后关闭。导出全量数据可以使用 scroll 或专用离线流程，不要把深分页接口当成导出工具。

## 6. 分片、副本和集群资源

### 6.1 分片是数据与计算单元

索引被切成 primary shard，副本 shard 提供冗余和读扩展。一次搜索通常要访问多个分片，再由协调节点合并结果；分片越多，单次请求的 fan-out 越大，协调和网络成本越高。

分片数量应按数据量、单片大小、写入速率和查询并发规划。为了“以后扩容”盲目创建很多小分片，会让集群长期承担额外元数据和查询开销。

### 6.2 副本不是备份

副本与 primary 通常在同一集群，误删、错误更新或整个集群故障时也可能一起失效。快照备份应写到独立对象存储，并定期验证恢复。

### 6.3 写入与搜索的资源竞争

segment merge、向量索引构建、聚合和全文查询都会消耗 CPU、堆内存、磁盘 I/O。写入高峰时可以调大批量、控制 refresh；查询高峰时要限制复杂聚合、选择合理副本和缓存。搜索集群的健康度不能只看节点是否存活。

### 6.4 分片路由与租户热点

默认把文档均匀哈希到分片，适合多数场景；显式 routing 能减少同一租户查询的 fan-out，但会把热点租户集中到少数分片。规划时要用真实租户数据量和查询比例做倾斜压测，监控每个分片的文档数、写入速率、查询耗时和磁盘水位。发现热点后可以拆租户索引、调整 routing 或限制单租户并发，不能只增加副本，因为副本也会复制热点写入。

## 7. 更新、并发和索引生命周期

### 7.1 文档更新不是数据库原地更新

部分更新最终会生成新版本。两个请求同时更新时，后到的请求可能覆盖先到的结果。需要保证顺序时，可以使用外部版本、`if_seq_no`/`if_primary_term` 或在 PostgreSQL 中维护版本后把版本带进文档。

### 7.2 别名与零停机重建

mapping 或 analyzer 需要改变时，通常创建新索引、批量重建、校验文档数和抽样查询，最后原子切换 alias：

```text
knowledge-v1  ← alias knowledge
      ↓ 新建并回填
knowledge-v2  ← alias knowledge
```

应用只访问 alias，不在代码中硬编码版本。切换前还应处理增量写入，可用双写、变更日志或按时间边界补偿。

### 7.3 生命周期管理

知识库、日志和审计数据的保留期不同。按时间或租户拆分索引，配合 ILM/数据流和快照，可以让旧数据迁移到低成本节点或删除。删除前必须确认合规保留要求和事实源是否仍需查询。

## 8. 向量检索与混合检索

### 8.1 向量字段表达语义相似

embedding 把 chunk 映射到高维向量，kNN 根据距离找相似内容。向量召回能找到没有共享关键词的表达，但受模型、切块、维度、距离函数和过滤策略影响。

### 8.2 过滤与向量召回的关系

租户、权限和文档状态必须先成为候选范围。若先取全局 top-k 再过滤，可能导致某个租户没有足够结果，甚至暴露无权文档的分数或元数据。选择 pre-filter、post-filter 或分区索引时，要看引擎能力和召回要求。

### 8.3 BM25 + kNN + rerank

常见 RAG 检索链是：

```text
用户问题
  ├─ BM25：精确术语、错误码、类名
  ├─ kNN：语义相近表达
  └─ 合并去重 → cross-encoder/规则重排 → top-k context
```

关键词与向量分数不在同一尺度，不能简单相加而不做归一或验证。重排模型成本更高，应只处理前几十条候选；最终 context 还要去重、截断、保留来源和权限信息。

### 8.4 切块决定上限

一个 chunk 太长，召回主题不集中、生成上下文浪费；太短，问题所需的定义和条件被拆开。切块时保留标题路径、页码、文档版本、权限和邻居关系，检索返回时可以扩展相邻 chunk，但不能越过权限边界。

### 8.5 召回评估与线上反馈

混合检索的参数要用固定问题集和金标准证据验证。至少分开记录 BM25 Recall@k、向量 Recall@k、融合后的 Recall@k、MRR/nDCG、引用正确率、P95/P99、token 成本和拒答率；不能只看最终答案“像不像”。线上把无结果、低分、用户点选和引用纠错沉淀成可脱敏的 Bad Case，按 embedding、切块、mapping、过滤、重排和生成阶段切片回归，避免一次改动同时掩盖多个原因。

## 9. 索引写入链与知识库状态

推荐把 PostgreSQL 作为文档事实源，Elasticsearch 作为可重建的搜索投影：

```text
上传文件
  ↓
PostgreSQL document/version
  ↓ Outbox/CDC
解析与切块任务
  ↓
embedding 生成
  ↓ bulk 写入 Elasticsearch
  ↓ 刷新可见
index_status = ready
```

`index_status` 要有版本号：只有 Elasticsearch 中的文档版本达到数据库要求，才标记为 ready。删除文档时要发送带版本的删除事件，避免旧事件把已经删除的文档重新写回。

## 10. 常见故障的定位顺序

### 10.1 “写入成功但搜不到”

检查请求返回是否只是接受写入；查看 refresh 间隔、索引是否被关闭、路由是否访问正确 alias；确认 analyzer 是否把查询词分析成预期 token；最后检查租户/状态 filter 是否把结果过滤掉。

### 10.2 “搜索突然变慢”

先看 p95/p99、慢查询和查询类型；区分全文查询、聚合、wildcard、深分页、向量 kNN 哪一类变慢；再看 segment 数、merge、GC、磁盘、线程池拒绝和分片分布。不要先把所有节点规格加大，先找到 fan-out 或单个坏查询。

### 10.3 “召回率下降”

抽样保存原文、分析后的 token、mapping、query DSL 和最终过滤条件；比较切块长度、embedding 版本、top-k、过滤策略和重排阈值。索引重建后要记录版本，不能只看最终答案主观感觉。

### 10.4 “集群磁盘不断增长”

检查更新/删除比例、segment merge、translog、快照和 ILM；确认是否存在重复索引、旧 alias、未清理的历史版本。删除文档不会立即释放空间，强行频繁 force merge 也可能带来更大 I/O 峰值。

## 11. 把搜索结果交给生成模型

RAG 不是“搜到几段文本就拼 prompt”。检索层应输出结构化证据：

```json
{
  "chunk_id": "doc-100#3",
  "title": "Redis 缓存治理",
  "content": "……",
  "score": 0.82,
  "source_url": "/documents/100",
  "document_version": 7,
  "tenant_id": "acme"
}
```

生成层只接收通过权限过滤、版本校验和长度预算的证据；答案引用 `source_url` 和 chunk 位置。若没有足够高质量证据，应返回“不确定”或要求澄清，而不是用低分片段强行生成。

## 12. 本章总结

- mapping 先于查询；`text` 用于全文，`keyword` 用于精确值、排序和聚合。
- 不受控字段会造成 mapping explosion；routing 可能减少 fan-out，也可能制造租户热点，不能替代权限 filter。
- analyzer、倒排索引、BM25 共同决定关键词召回；可见性由 refresh 决定，持久化由 translog/磁盘策略决定。
- filter 处理权限和枚举，query 处理相关性；深分页使用 `search_after`/PIT，而不是无限增大 from。
- 分片决定 fan-out 和并行度，副本提供冗余但不是独立备份；更新、删除和 merge 会产生写放大。
- RAG 通常采用 BM25 + kNN + 重排，权限和版本必须在召回阶段纳入。
- 混合检索要用 Recall、引用正确率、P95/P99 和 token 成本评估，不能只凭答案观感调参。
- 搜索集群要和事实源分离：数据库保存真相，ES 保存可重建投影；mapping 演进用新索引和 alias 切换。

## 13. 官方资料

- [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Text analysis](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html)
- [Term-level queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/term-level-queries.html)
- [Near real-time search](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)
- [Search after](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html#search-after)
- [Point in time](https://www.elastic.co/guide/en/elasticsearch/reference/current/point-in-time-api.html)
- [Index lifecycle management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [kNN search](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
