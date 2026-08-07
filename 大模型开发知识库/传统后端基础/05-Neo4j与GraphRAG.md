# 05 Neo4j 与 GraphRAG：从关系建模到可解释检索

> 适合对象：知道知识图谱能表示实体关系，但还不清楚图数据库与关系数据库的边界、Cypher 如何执行、GraphRAG 如何和向量检索协作的人。
>
> 学完目标：你能把业务事实建模成节点、关系和属性，理解 Neo4j 的索引、约束、事务和路径查询，能判断什么时候应该用图数据库，能设计“实体识别—图遍历—文本证据—生成答案”的 GraphRAG 链路，并能处理重复实体、关系版本、权限和大图性能问题。

## 0. 先看一个需要“关系”的问题

在企业知识库里，用户问：“负责 Redis 缓存治理的团队有哪些成员？他们维护的服务和相关事故是什么？”

纯向量检索可以找到“Redis 缓存治理”附近的文本，但不一定能稳定回答成员、团队、服务、事故之间的多跳关系。图模型可以把事实组织成：

```text
(Person)-[:MEMBER_OF]->(Team)
(Team)-[:OWNS]->(Service)
(Service)-[:USES]->(Technology {name:'Redis'})
(Service)-[:AFFECTED_BY]->(Incident)
```

GraphRAG 的一条路径可以是：

```text
问题 → 识别实体/关系 → Cypher 查结构事实
    → 找到关联文档 chunk → 结合向量证据
    → 生成带来源和路径的答案
```

> **核心判断：** 图数据库擅长回答“谁和谁通过什么关系连接、经过几跳、满足哪些路径条件”；向量库擅长回答“语义上哪些文本相似”。GraphRAG 不是把所有文本塞进图，而是让图负责结构导航，让文本和向量负责证据细节。

## 1. 属性图的基本组成

### 1.1 节点、关系和属性

Neo4j 的 property graph 由三类主要元素组成：

- **节点（node）**：实体，如人、团队、服务、文档、技术；
- **关系（relationship）**：有方向、有类型的连接，如 `MEMBER_OF`、`OWNS`、`MENTIONS`；
- **属性（property）**：节点或关系上的键值，如 `name`、`version`、`valid_from`。

```text
(:Person {id:'p-1', name:'Lin'})
  -[:MEMBER_OF {since:'2025-01-01'}]->
(:Team {id:'team-1', name:'Platform'})
```

关系本身不是简单的外键表行，它可以保存时间、来源、置信度、角色和版本。这是图模型表达业务关系的优势，也是数据治理的难点。

### 1.2 标签和类型

节点可以有多个 label，例如 `:Person:Employee`；关系有一个主要 type。标签帮助按实体类型查询和建立索引，但不能把每个业务状态都滥用成 label；经常变化的状态更适合作为属性。

### 1.3 图不是“无结构 JSON”

图查询仍然依赖稳定的 ID、约束、属性类型和关系语义。若同一个人有三种拼写、同一服务有多个 ID，遍历会产生重复和错误路径。建模的第一步不是写 Cypher，而是定义实体唯一性、关系方向、来源和生命周期。

## 2. 图模型怎样从业务事实产生

以文档知识库为例，可以定义：

```text
(:Tenant {tenant_id})
(:Document {document_id, version, title})
(:Chunk {chunk_id, text, embedding_id})
(:Person {person_id, name})
(:Team {team_id, name})
(:Technology {technology_id, name})

(:Document)-[:HAS_CHUNK]->(:Chunk)
(:Chunk)-[:MENTIONS]->(:Technology)
(:Person)-[:MEMBER_OF {valid_from, valid_to}]->(:Team)
(:Team)-[:OWNS]->(:Document)
```

权限边界不能只存在于应用层。每个可见节点或关系都应能沿路径追溯到 tenant，查询必须把租户和文档版本作为条件；否则 GraphRAG 可能把另一个租户的实体路径送给模型。

### 2.1 关系方向要稳定

`(:Person)-[:MEMBER_OF]->(:Team)` 比随意使用无向关系更容易表达“谁属于谁”和反向查询。Cypher 可以在查询时忽略方向，但写入时统一方向有利于唯一约束、可读性和路径推理。

### 2.2 时间有效性不能省略

团队成员会变动，服务也会迁移。只保存当前关系会让历史问题得到错误答案；建议在关系上保存 `valid_from`、`valid_to` 或事件版本，并在查询时明确“当前”还是“某个日期当时”。

## 3. Cypher：先读懂模式，再理解执行

### 3.1 MATCH 是模式匹配

```cypher
MATCH (p:Person)-[:MEMBER_OF]->(t:Team {name: $team_name})
RETURN p.person_id, p.name
ORDER BY p.name;
```

`MATCH` 描述节点和关系的形状，参数 `$team_name` 避免拼接用户输入。查询先定位起点，再沿关系扩展；没有合适索引时可能扫描大量节点。

### 3.2 WHERE 过滤属性或路径

```cypher
MATCH p = (person:Person)-[:MEMBER_OF*1..3]->(team:Team)
WHERE team.tenant_id = $tenant_id
  AND all(n IN nodes(p) WHERE n.active = true)
RETURN p;
```

路径变量可以返回完整路径、节点和关系。`*1..3` 表示长度范围；无上限的可变长度路径在大图上可能产生指数级候选，应限制深度、起点和返回数量。

### 3.3 OPTIONAL MATCH 保留没有关系的实体

```cypher
MATCH (d:Document {document_id: $id})
OPTIONAL MATCH (d)-[:MENTIONS]->(tech:Technology)
RETURN d, collect(tech.name) AS technologies;
```

它类似“左连接”的语义，但多跳 OPTIONAL MATCH 可能产生笛卡尔组合。应先聚合或拆分查询，避免返回大量重复行。

### 3.4 聚合和去重

```cypher
MATCH (t:Team)-[:OWNS]->(s:Service)
RETURN t.name, count(DISTINCT s) AS service_count;
```

`DISTINCT` 要有明确目的。盲目去重可能掩盖模型中重复关系或错误导入，应同时检查数据质量。

## 4. CREATE、MERGE 与实体去重

### 4.1 CREATE 只负责新增

重复执行 `CREATE` 会产生重复节点和关系。导入任务可用它插入已确认的新实体，但必须由上游保证唯一性。

### 4.2 MERGE 是“匹配或创建”

```cypher
MERGE (d:Document {document_id: $document_id})
ON CREATE SET d.created_at = datetime()
SET d.title = $title, d.version = $version;
```

`MERGE` 的匹配键必须对应唯一约束。把不稳定的标题、描述或整段文本放进 MERGE，会造成多个近似实体；应先用外部 ID 或规范化后的 canonical key。

关系也要考虑重复：

```cypher
MATCH (p:Person {person_id: $person_id})
MATCH (t:Team {team_id: $team_id})
MERGE (p)-[r:MEMBER_OF {source_id: $source_id}]->(t)
SET r.valid_from = $valid_from;
```

若同一人和团队允许多段历史关系，关系唯一键应包含时间段或 source_id，而不是简单地把所有历史压成一条。

## 5. 索引、约束和查询计划

### 5.1 约束先保证身份

对 `Document.document_id`、`Person.person_id`、`Tenant.tenant_id` 建立唯一约束，避免并发导入产生重复实体。约束既是数据质量规则，也会帮助定位节点。

### 5.2 索引服务于起点和精确过滤

```cypher
CREATE CONSTRAINT document_id_unique IF NOT EXISTS
FOR (d:Document) REQUIRE d.document_id IS UNIQUE;

CREATE INDEX technology_name IF NOT EXISTS
FOR (t:Technology) ON (t.name);
```

索引应放在经常作为 MATCH 起点或过滤条件的属性上。图遍历的成本主要取决于起点数量、每个节点的度数、路径深度和返回路径数；给所有属性都建索引不会让任意遍历都变快。

### 5.3 用 EXPLAIN/PROFILE 看实际计划

`EXPLAIN` 查看计划而不执行，`PROFILE` 执行并显示行数与数据库命中。重点观察是否出现全标签扫描、意外的笛卡尔积、扩展后才过滤、遍历了远超预期的路径。优化方向通常是先用唯一索引锁定起点，再做有限扩展和早期过滤。

### 5.4 关系幂等与历史唯一性

节点唯一约束不能自动阻止重复关系。导入关系时要先明确业务语义：同一对实体是否只允许一条当前关系，还是允许多个来源、角色或时间段。可以把 `source_id`、`valid_from` 或事件版本纳入关系匹配条件，并让重复导入对同一身份只更新可变属性：

```cypher
MATCH (p:Person {person_id: $person_id})
MATCH (t:Team {team_id: $team_id})
MERGE (p)-[r:MEMBER_OF {source_id: $source_id}]->(t)
ON CREATE SET r.valid_from = $valid_from
SET r.confidence = $confidence, r.event_version = $event_version;
```

这仍不是“任意关系都天然唯一”。并发导入、不同来源 ID 和历史关系要由约束、批次审计和对账共同处理；合并实体前保留旧 ID 映射，才能回滚错误的消歧。

## 6. 事务、锁与并发写入

### 6.1 节点和关系的原子写入

一次事务可以同时创建节点、关系和属性；提交后其他事务才能看到完整结果。跨多个服务或数据库的操作仍然不是 Neo4j 单事务能够自动包住的范围，需要 Outbox、补偿或最终一致性。

### 6.2 并发 MERGE 的冲突

两个导入任务同时 `MERGE` 同一个实体时，唯一约束能阻止重复，但可能出现锁等待或约束冲突。应用应设置事务超时、有限重试和幂等 source_id；不要通过无限重试掩盖模型或热点设计问题。

### 6.3 大事务的代价

一次导入数百万节点会占用事务日志、内存和锁。应按稳定批次提交，并记录批次进度和 source_id；失败后从可重放的批次继续，而不是回滚整个知识图谱。批量导入工具和在线事务的适用场景不同。

## 7. 图查询的性能直觉

### 7.1 起点比边多更危险

```cypher
MATCH (n)-[*1..4]->(m)
RETURN m;
```

如果 `n` 没有限定标签和 ID，这个查询可能从全图开始扩展。应先限定租户、实体类型、起点 ID，再限制路径长度和返回数量。

### 7.2 高度节点会放大结果

一个“通用标签”节点连接百万条关系时，任何经过它的查询都会产生巨大候选集。可按租户、时间或实体类型拆分，或把高频聚合结果预计算成属性/关系，避免在线反复遍历。

### 7.3 不要把文本全文当作图属性查询

用 `CONTAINS` 扫描长文本找关键词通常不如专用全文索引或 Elasticsearch。图数据库负责结构关系；文本相关性、分词和向量检索应由合适的搜索层承担。

## 8. GraphRAG 的完整链路

### 8.1 离线构建

```text
文档解析
  ↓
实体/关系抽取（保留原文 span 和置信度）
  ↓
实体规范化与去重
  ↓
Neo4j 写节点/关系/来源/版本
  + Elasticsearch 写 chunk 文本和向量
```

抽取模型可能把“Redis Cluster”和“Redis 集群”识别成两个实体，因此必须保留 canonical ID、别名和来源 span；低置信度关系可先标记为待审核，而不是直接当作事实。

### 8.2 在线查询

```text
问题
  ↓
实体链接：找到 Redis、缓存治理、团队
  ↓
Cypher：取一到三跳的结构关系和权限边界
  ↓
向量/关键词检索：找相关 chunk 的定义和证据
  ↓
合并、去重、按来源和版本排序
  ↓
LLM 生成答案 + 路径/文档引用
```

Cypher 模板应由服务端维护，实体名称和 ID 使用参数；不要让模型直接拥有任意写权限。返回给模型的结构事实应包含关系类型、时间和来源，否则模型可能把“提到”误说成“拥有”。

### 8.3 何时只用向量检索

如果问题主要是“解释一个概念”“找相似段落”，向量 + 关键词通常更简单。图数据库的收益出现在多跳关系、路径约束、实体聚合、时间有效性和可解释来源明显重要的场景。为了“看起来高级”而把所有 chunk 转成节点，会增加抽取错误、写入成本和查询复杂度。

### 8.4 GraphRAG 的质量闭环

GraphRAG 至少有四个可独立出错的环节：实体链接、关系抽取/去重、图查询路径和最终生成。评估不能只问答案是否流畅，而要保存金标准实体、关系方向、有效时间和来源 span，分别统计实体链接准确率、关系 precision/recall、路径命中率、证据引用正确率以及最终答案的事实一致性。

在线结果应带 `source_span`、抽取模型版本、confidence、document_version 和 tenant。低置信度关系可以降低排序权重、要求第二来源或回退到文本检索；图查询超时则返回已授权的 BM25/kNN 证据并明确“关系证据暂不可用”。这样图层故障或抽取错误不会静默变成模型的确定性幻觉。

## 9. 权限、删除和版本

### 9.1 权限必须进入图查询

查询实体关系时要带 tenant、项目、角色等限制：

```cypher
MATCH (d:Document {document_id: $id})
WHERE d.tenant_id = $tenant_id AND d.deleted_at IS NULL
MATCH (d)-[:HAS_CHUNK]->(c:Chunk)
RETURN c.chunk_id, c.text;
```

若关系路径经过共享实体，必须确认共享实体本身没有把另一个租户的私有关系带出来。权限过滤应在图查询和文本检索两边都执行。

### 9.2 软删除与级联

删除文档时，节点、chunk、实体关系和 Elasticsearch 投影可能由不同任务处理。先写删除事实和版本，再通过事件异步清理图和搜索投影；查询默认过滤 `deleted_at`，清理失败时可重试。不要只删除文档节点却留下可被遍历到的孤儿 chunk。

### 9.3 版本化关系

同一服务的负责人会变更；更新关系前要保留旧关系的 `valid_to` 或事件版本。查询“现在”与查询“2025 年”应使用不同条件，不能让当前状态覆盖历史审计。

## 10. Python 服务中的边界

应用层应把图数据库封装成有类型的查询函数，而不是在业务代码里散落字符串：

```python
async def find_related_documents(
    tenant_id: str,
    technology_id: str,
    max_hops: int = 2,
) -> list[dict]:
    # max_hops 经过白名单校验，Cypher 模板固定，参数只传值
    ...
```

连接池要设置最大连接、事务超时和重试；查询结果要限制数量和字段，避免把整条路径的所有属性返回给 API。图查询失败时，可以降级到 Elasticsearch 的关键词/向量检索，并在结果中标注“关系证据不可用”，不能静默返回不完整的结构答案。

## 11. 典型故障与排查顺序

### 11.1 “同一个人出现多个节点”

检查 canonical ID 规则、唯一约束、MERGE 键和并发导入的 source_id；再做别名合并，不能仅凭名字相同就自动合并，因为同名人员可能是不同实体。

### 11.2 “查询越来越慢”

查看 PROFILE 的起点行数、展开关系数、路径数量和返回行数；检查是否缺索引、无上限路径、重复关系或高度节点；把大范围统计改为离线聚合或按租户分批。

### 11.3 “GraphRAG 答案看似合理但关系错了”

保存实体链接、Cypher、返回路径、关系来源 span 和版本；分别评估实体识别、关系抽取、图查询和生成，不要只看最终文本。低置信度关系应降低权重或要求来源引用。

## 12. 本章总结

- 图数据库的核心是节点、带方向和属性的关系、稳定的实体身份；建模先于查询。
- `MATCH` 描述路径，`MERGE` 需要唯一约束，索引负责快速找到起点，PROFILE 用来验证实际代价。
- 节点唯一不等于关系唯一；关系要按 source、时间和版本设计幂等与历史。
- 路径深度、起点数量和节点度数决定遍历成本；图数据库不是全文搜索引擎。
- GraphRAG 让图负责结构导航和可解释路径，让 Elasticsearch/向量层负责文本证据；两者都必须执行权限和版本过滤。
- 事务、重试、软删除、历史关系和批量导入共同决定数据能否长期可信。
- GraphRAG 要分别评估实体链接、关系、路径、来源和生成，图超时要有授权的文本检索回退。

## 13. 官方资料

- [Neo4j Cypher Manual](https://neo4j.com/docs/cypher-manual/current/)
- [Cypher MERGE](https://neo4j.com/docs/cypher-manual/current/clauses/merge/)
- [Neo4j Indexes](https://neo4j.com/docs/cypher-manual/current/indexes/search-performance-indexes/)
- [Neo4j Constraints](https://neo4j.com/docs/cypher-manual/current/constraints/)
- [Neo4j Transactions](https://neo4j.com/docs/operations-manual/current/database-internals/transactions/)
- [Neo4j Python Driver Manual](https://neo4j.com/docs/python-manual/current/)
- [Neo4j GraphRAG for Python](https://neo4j.com/docs/neo4j-graphrag-python/current/)
