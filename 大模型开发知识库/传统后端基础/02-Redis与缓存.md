# 02 Redis 与缓存：从一次读取请求到高可用数据结构

> 适合对象：会调用 Redis 客户端，但还没有把“缓存、持久化、复制、锁”放到同一张图里的人。
>
> 学完目标：你能判断一份数据能不能只放缓存，能设计一套可解释的 key 和过期策略，能说明缓存不一致、击穿、雪崩、热 key、大 key 的成因与处理方式，也能知道 Redis 的持久化、高可用和分布式锁各自保证了什么、没有保证什么。

## 0. 先从一条请求开始

假设知识库首页要显示“最近使用的 20 个文档”。文档的事实记录在 PostgreSQL，读取量远大于修改量。如果每次打开页面都查数据库，数据库连接、磁盘页和排序都会被重复消耗。于是我们在中间加 Redis：

```text
浏览器
  ↓ GET /recent-documents
应用服务
  ↓ 先读 Redis：user:42:recent-documents
  ├─ 命中：直接返回
  └─ 未命中：读 PostgreSQL → 组装结果 → 写入 Redis(TTL) → 返回
```

这里已经出现了 Redis 学习的全部主线：

1. Redis 里存的到底是事实，还是可以丢掉的副本？
2. key 和数据结构怎样设计，才能让一次命令完成主要工作？
3. 数据库更新后，缓存什么时候失效，短暂不一致能不能接受？
4. Redis 重启、网络分区或主节点故障时，应用怎样退化？
5. 多个实例同时处理同一个任务时，Redis 锁能否真的保证正确性？

> **核心判断：** Redis 是一个内存优先的、以数据结构命令为主要接口的数据服务。缓存只是它最常见的用途；当它被用作队列、计数器、排行榜、会话或锁时，必须重新说明数据是否可丢、并发边界在哪里，以及故障后如何恢复。

## 1. Redis 的基本模型：key、value 与一次命令

### 1.1 key 不是数据库表名

Redis 的基本寻址方式是 `key -> value`。它没有关系数据库那样的表、Join 和通用 SQL 优化器；应用通常通过精心设计的 key 直接定位一份数据。

```text
tenant:{tenant_id}:user:{user_id}:profile
tenant:{tenant_id}:conversation:{conversation_id}:messages
tenant:{tenant_id}:document:{document_id}:status
```

好的 key 至少表达四件事：业务域、租户或用户边界、对象类型、对象标识。这样做不是为了好看，而是为了避免跨租户读错数据、便于批量删除和监控，也便于迁移到 Redis Cluster。

不要把未经限制的用户输入直接拼成 key。应限制长度、字符集和层级，避免同一个对象因大小写或格式不同产生多份缓存。

### 1.2 单线程命令为什么看起来“原子”

Redis 的命令处理在一个事件循环中串行执行。像 `INCR`、`HSET`、`SADD` 这样的单条命令不会在执行中途被另一条命令插入，所以常见计数操作不需要在客户端再加一把线程锁。

这并不等于“所有业务操作都是原子的”：

- 客户端先 `GET`，再根据结果 `SET`，中间可能被别的请求改写；
- 一组命令若必须作为一个整体执行，需要事务 `MULTI/EXEC` 或 Lua 脚本；
- Redis 命令的原子性只约束 Redis 内部，不能把 PostgreSQL 更新自动包进同一事务。

例如“余额大于零才扣减”不能写成两个独立命令：

```text
GET quota:user:42        # 应用判断
DECR quota:user:42       # 另一个请求可能已经扣过
```

应该让判断和扣减在 Redis 内一次完成，或者把真正的余额事实放在数据库，用数据库条件更新保证正确性。

### 1.3 时间复杂度不等于实际耗时

`GET`、`HGET`、`SADD` 等常见命令通常是 O(1)，但大 key 的 O(1) 仍可能意味着一次处理几十 MB；`SMEMBERS`、`HGETALL`、`LRANGE 0 -1` 可能按集合大小线性返回数据。网络传输、序列化、内存分配和主线程阻塞往往比公式更重要。

设计命令时要同时问：返回多少字节？是否会遍历整个集合？是否会让 Redis 主线程停顿？能否分页、分片或异步处理？

## 2. 六种常用数据结构，以及它们适合表达什么

### 2.1 String：一个值、计数器或小型序列化对象

String 可以保存文本、整数、二进制或 JSON。它适合缓存单个对象、保存 token、计数器和短状态：

```text
SET tenant:acme:document:100:status "indexed" EX 3600
INCR api:tenant:acme:requests:2026-08-07
```

如果把一个很大的 JSON 全部放在 String 里，更新其中一个字段必须读出、反序列化、整体写回，竞争也更容易发生。对象经常按字段更新时，Hash 更合适。

### 2.2 Hash：对象字段的集合

```text
HSET tenant:acme:user:42 name "Lin" role "admin"
HGET tenant:acme:user:42 role
```

Hash 适合用户资料、任务状态、配置快照等“字段相对稳定”的对象。需要设置过期时间时，过期是整个 Hash 的，不是单个 field 的；若 field 需要不同 TTL，应拆成多个 key 或保存带时间戳的值并在读取时判断。

### 2.3 List：两端队列和有限历史

`LPUSH/RPOP` 可以表达简单队列，`LPUSH/LTRIM` 可以保留最近 N 条记录：

```text
LPUSH tenant:acme:user:42:recent-documents 100
LTRIM tenant:acme:user:42:recent-documents 0 19
```

List 不提供消费确认和重放语义。消费者宕机后，已经 `RPOP` 的消息可能丢失；需要消费组、待处理列表和重新认领时，应使用 Stream 或专门的消息队列。

### 2.4 Set：去重和集合关系

Set 适合标签、权限、关注关系和去重：

```text
SADD tenant:acme:user:42:roles editor reviewer
SISMEMBER tenant:acme:user:42:roles editor
```

`SINTER`、`SUNION` 很方便，但大集合运算会占用主线程。不要在在线请求里对两个百万级集合做交集；可以离线计算结果、限制成员数量或把关系放到更适合查询的数据库。

### 2.5 Sorted Set：按分数排序的索引

Sorted Set 为 member 保存 score，适合排行榜、延迟任务、按时间取数据：

```text
ZADD tenant:acme:documents:recent 1723017600 document:100
ZRANGE tenant:acme:documents:recent 0 19 REV WITHSCORES
```

时间戳是常见 score，但要注意同一秒多个 member 的排序规则；若业务需要严格顺序，可使用单调序列或把时间和 ID 组合。Sorted Set 不是全文搜索，也不是事务调度器，复杂筛选仍应交给数据库或搜索引擎。

### 2.6 Stream：带 ID 的追加日志和消费组

Stream 保存形如 `(id, field, value)` 的消息，`XGROUP` 可以让多个消费者分工，消费者确认后消息会从 Pending Entries List 中移出：

```text
XADD document-events * type indexed document_id 100
XREADGROUP GROUP indexers worker-1 COUNT 20 BLOCK 1000 STREAMS document-events >
XACK document-events indexers 1723017600000-0
```

Stream 能提供比 List 更清晰的消费、确认和重试语义，但它仍需要设置保留长度、处理 Pending 太久的消费者，并为重复消费设计幂等逻辑。对跨服务关键事件，Kafka 或 RabbitMQ 可能更适合；选择依据是吞吐、保留、路由和运维需求，而不是“Redis 也能发消息”。

## 3. 缓存与事实源：先决定谁说了算

### 3.1 三类数据不要混淆

在设计前，把数据分成三类：

| 类型 | 示例 | Redis 丢失后的处理 |
|---|---|---|
| 可重建副本 | 文档列表、模型配置快照 | 从 PostgreSQL/配置中心重新加载 |
| 有时效的派生状态 | 限流计数、短期会话、任务进度 | 重新计算或允许回退 |
| 业务事实 | 账户余额、订单状态、支付结果 | 必须有可靠持久化事实源，不能只依赖缓存 |

“Redis 持久化了”也不能把缓存自动变成事实源：持久化可能有丢失窗口，主从切换可能丢最近写入，误删 key 也无法替代数据库的约束、审计和备份。

### 3.2 Cache Aside：最常用的读写路径

读取时先查缓存，未命中后查事实源，再回填：

```text
读：Redis 命中 → 返回
    Redis 未命中 → PostgreSQL → SET key value EX ttl → 返回
```

写入时先更新事实源，再删除缓存：

```text
写：UPDATE PostgreSQL → DEL cache-key
```

为什么通常是“更新数据库后删除缓存”，而不是先删缓存再更新数据库？先删后写存在这样的时间线：请求 A 删缓存，尚未写库；请求 B 读到旧数据库值并回填旧缓存；A 随后写入新值，缓存反而长期停留在旧值。更新数据库后删除缓存仍可能有极短窗口，但更容易通过延迟双删、版本号或消息失效来降低风险。

### 3.3 一致性不是开关，而是时间窗口

常见策略按强弱和成本排列：

1. **接受短暂不一致**：适合推荐列表、统计数字、搜索结果。
2. **写库后删缓存**：简单可靠，读请求在删除与回填之间可能看到旧值。
3. **延迟双删**：写库、删缓存，等待短时间后再删一次，减小并发读回填旧值的窗口。
4. **消息驱动失效**：数据库提交后写 Outbox，消费者删除对应 key；可重试、可审计，但系统更复杂。
5. **版本化读取**：缓存值带版本，读者只接受不低于要求版本的结果；适合需要读己之写的场景。

不要只说“保证缓存一致性”，要说允许多长时间的旧值、谁是事实源、删除失败如何重试、读请求如何降级。

## 4. 缓存问题为什么会连锁发生

### 4.1 缓存穿透：请求一个确定不存在的对象

攻击者或业务 bug 可能不断查询不存在的 `document_id`。每次都未命中并打到数据库，缓存帮不上忙。

处理方式是组合使用：

- 在参数层拒绝不合法 ID；
- 对“确实不存在”写入很短 TTL 的空值；
- 用 Bloom Filter 做快速拦截，但承认它有误判；
- 对同一来源做限流和监控。

空值缓存不能永久保存，否则对象后来创建时会被旧空值挡住；创建成功时也可以主动删除对应空值 key。

### 4.2 缓存击穿：一个热点 key 在同一时间过期

某个热门知识库首页的 key 过期，数千个请求同时回源数据库。常用做法是给 TTL 加随机抖动，并对单个 key 做互斥重建：

```text
请求 1 获取 rebuild-lock → 回源并回填
请求 2/3 未获取锁 → 等待短时间后重试，或读旧值/降级
```

互斥锁的等待时间必须有限。若锁持有者宕机，依赖 TTL 或可续租机制释放；不能让所有请求无限等待。

### 4.3 缓存雪崩：大量 key 同时失效或 Redis 整体不可用

批量发布、统一 TTL、重启或网络故障都可能造成雪崩。除了 TTL 抖动，还需要：

- 分批预热和限速回源；
- 连接池超时、熔断和舱壁隔离；
- 保留最后一次可接受的旧值；
- 数据库连接数和并发查询上限；
- Redis 集群、哨兵或托管高可用，以及明确的降级页面。

缓存故障处理的目标不是“永远可用”，而是避免缓存故障把事实源也拖垮。

### 4.4 热 key 与大 key

热 key 是访问集中在少数 key；大 key 是单个 key 的成员数或序列化体积过大。两者都会让一个 Redis 节点或一个事件循环承担异常压力。

监控不能只看平均 QPS，还应看命令耗时、单 key 访问分布、value 大小、慢查询日志、网络出入流量和内存碎片。拆分大 key 时要保留可分页的顺序，例如把最近消息按日期分片：

```text
conversation:42:messages:2026-08-07
conversation:42:messages:2026-08-06
```

热点 key 可以复制成多个只读副本，或在应用层加本地短缓存；但写入和失效复杂度也会增加，不能只靠“多复制几份”解决。

## 5. TTL、淘汰与内存：为什么设置过期还不够

### 5.1 TTL 是上限，不是精确删除时间

Redis 会记录过期时间，并在访问或后台主动检查时删除。到达 TTL 的瞬间不一定刚好被删除；因此 TTL 适合表达“最多保留多久”，不能用来安排毫秒级精确任务。

### 5.2 淘汰策略体现业务优先级

达到 `maxmemory` 后，Redis 按配置选择淘汰策略，例如只淘汰带 TTL 的 key、按最近最少使用淘汰，或不淘汰直接返回 OOM。策略要与数据分类匹配：

- 全部是缓存副本时，可以按 LRU/LFU 淘汰；
- 混有会话、锁或 Stream 时，必须明确哪些 key 不能被误淘汰；
- 业务事实不应依赖“永不淘汰”的假设。

给 key 设置 TTL 只是让它最终能被淘汰，不代表内存一定够用。需要估算 value、key、对象编码、复制和碎片开销，并设置内存水位告警。

## 6. RDB、AOF 与恢复：丢多少数据由配置决定

### 6.1 RDB 是时间点快照

RDB 在某个时间点生成二进制快照，文件紧凑，适合备份和快速加载。生成快照时通常会 fork 子进程，父子进程通过写时复制共享页面；写入越多，复制出的内存页越多，后台保存也可能造成内存和磁盘压力。

RDB 的天然问题是两个快照之间的写入可能在故障时丢失。快照频率越高，丢失窗口越小，但 fork、磁盘和 I/O 成本越高。

### 6.2 AOF 是命令追加日志

AOF 记录改变数据的写命令，重启时重放日志。`appendfsync` 决定日志何时刷到磁盘：每秒刷通常允许约一秒窗口，始终刷更安全但写放大和延迟更高。AOF 需要 rewrite 压缩历史命令，rewrite 期间同样要关注磁盘空间和内存。

### 6.3 选择不是二选一

缓存副本可以只使用复制和定期备份；需要较小丢失窗口的会话、Stream 或任务状态，通常会同时启用 RDB 与 AOF，并把备份复制到独立存储。无论选择哪种方式，都要回答：

1. Redis 节点彻底损坏时从哪里恢复？
2. 恢复后允许丢多少秒的数据？
3. 恢复过程中应用是只读、降级还是暂停写入？
4. 备份是否加密、保留多久、是否真的做过恢复演练？

## 7. 复制、哨兵与 Cluster：高可用解决的是哪种故障

### 7.1 主从复制不是同步提交

主节点处理写入，把复制流发送给副本。副本可能落后；客户端读副本时可能读到旧值。发生主从切换时，尚未复制到新主的写入可能丢失。

如果一次写入后马上读取必须看到自己的写入，可以短时间读主、携带版本号，或等待副本达到指定复制偏移。不要把“有副本”直接等同于“强一致”。

### 7.2 Sentinel：监控与故障转移

Sentinel 监控主从、判断主节点不可达并协调提升副本。它解决的是单主故障的自动切换，不是跨机房强一致，也不自动解决应用连接池缓存的旧主地址。客户端需要使用支持 Sentinel 的连接方式，并在切换后重新建立连接。

### 7.3 Cluster：分片和部分高可用

Redis Cluster 把 key 映射到 16384 个 hash slot，再把 slot 分配给不同主节点。带花括号的 hash tag 可以让相关 key 落到同一个 slot：

```text
tenant:{acme}:user:42
tenant:{acme}:conversation:7
```

这样两个 key 才能在一次多 key 操作中一起处理。Cluster 不是关系数据库的分布式事务；跨 slot 的 `MGET`、Lua 和事务会受到限制，跨节点操作应在应用层拆解并处理部分成功。

## 8. 分布式锁：让“占用资源”可验证、可过期

### 8.1 最小安全写法

获取锁要保证只有一个请求成功，并写入随机 token：

```text
SET lock:document:100 token-abc NX PX 10000
```

释放锁不能无条件 `DEL`。请求 A 超时后锁已过期，请求 B 获得新锁；如果 A 恢复并删除 key，就会误删 B 的锁。释放应通过 Lua 原子地比较 token 后删除：

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
end
return 0
```

### 8.2 TTL、续租和 fencing token

锁 TTL 太短，持有者还没完成就过期；TTL 太长，宕机后资源被阻塞。续租只能延长锁，不能阻止进程暂停后拿着旧锁继续写。

对必须防止“旧持有者迟到写入”的系统，应给每次成功加锁分配递增 fencing token，并让数据库或下游资源拒绝更小的 token：

```text
请求 A token=41（暂停）
请求 B token=42（完成并写入）
请求 A 恢复，携带 token=41 → 下游拒绝
```

这说明 Redis 锁本身只是协调信号。最终正确性仍应由数据库条件更新、版本号或下游的 fencing 检查保证。

### 8.3 哪些情况不要用 Redis 锁

如果数据库已经可以用 `UPDATE ... WHERE version = ?`、唯一约束或行锁表达不变量，优先让事实源负责一致性。Redis 锁适合减少重复计算、串行化缓存重建和短期领导者选举；不适合作为支付扣款、库存最终扣减的唯一防线。

## 9. Pipeline、事务、Lua 与 Stream 的边界

### 9.1 Pipeline 只减少网络往返

Pipeline 把多条命令一次发送，能降低 RTT，但不自动提供事务隔离。期间其他客户端仍可能在命令之间执行操作。

### 9.2 MULTI/EXEC 让命令批次连续执行

`MULTI/EXEC` 能让批次中的命令不被其他客户端插入，但它不会像数据库事务一样对所有命令做回滚；某条命令运行时失败，之前成功的命令不会自动撤销。业务需要自己设计补偿。

### 9.3 Lua 适合短小的服务器端不变量

Lua 可以把“检查库存、扣减、记录订单”放在 Redis 内连续执行，并减少网络往返。脚本运行期间会占用主线程，不能在脚本里做大范围遍历或长时间循环；脚本必须幂等、可超时、可审计。

### 9.4 Stream 的可靠消费链

一个可恢复的 Stream 消费链包括：读取、业务处理、确认、失败重试、Pending 超时扫描、死信和幂等记录。确认应放在业务事实成功之后；否则先确认再写库，消费者宕机会造成消息丢失。

## 10. 在 Python 服务中落地

### 10.1 连接池与超时

Redis 客户端连接池应设最大连接数、连接建立超时、读写超时和重试上限。没有超时的请求会在 Redis 网络故障时长期占用 FastAPI worker，最终形成线程/协程耗尽。

应用日志至少记录：命令类别、key 的逻辑类型、命中/未命中、耗时、降级原因。不要把完整 token、用户隐私或超大 value 写进日志。

### 10.2 缓存封装要统一策略

可以把缓存访问封装成三个明确函数：

```python
async def get_document(document_id: int):
    # 1. 读缓存；2. 未命中读数据库；3. 带版本和 TTL 回填
    ...

async def update_document(document_id: int, patch: dict):
    # 1. 数据库事务提交；2. 删除或发布失效事件
    ...

async def rebuild_document_cache(document_id: int):
    # 只允许有限并发；失败时保留旧值或返回降级结果
    ...
```

统一封装能够避免某个业务模块忘记 TTL、使用不一致的序列化格式，或在写库前误写缓存。

### 10.3 序列化与数据版本

缓存值建议包含 schema/version 字段：

```json
{"schema": 2, "updated_at": "2026-08-07T10:00:00Z", "data": {...}}
```

发布新版本时可以先读旧格式并兼容一段时间，再批量失效。直接改变 JSON 字段名而不改 key，容易让旧进程和新进程互相读不懂。

## 11. 从现象到原因：一套排查顺序

### 11.1 “缓存命中率突然下降”

先区分业务流量变化和 key 失效：查看命中率、写入量、过期量、淘汰量和发布记录；再按 key 前缀抽样，检查 TTL 是否统一、序列化版本是否变化；最后看回源数据库的 QPS、连接池和慢查询。

### 11.2 “Redis 延迟突然升高”

先看慢命令和单命令返回大小，再看 CPU、内存、fork、网络和主从同步；搜索 `KEYS`、全量 `SMEMBERS`、超大 `MGET`、大批量 Lua 等阻塞操作；最后确认是否出现热 key 或集群 slot 倾斜。

### 11.3 “锁失效后出现重复处理”

检查锁 TTL 是否小于业务最长耗时、续租是否可靠、释放是否校验 token、处理结果是否有幂等键；如果旧持有者可能迟到写入，补充 fencing token 或数据库版本条件。

## 12. 把整篇内容串起来

一次缓存读取真正经过的是：

```text
业务事实源 → key/数据结构 → 命中与回源策略 → TTL/淘汰
       ↓                         ↓
   持久化与备份             一致性与失效事件
       ↓                         ↓
复制/哨兵/Cluster ← 超时、降级、幂等、观测
```

当你决定“把文档状态放 Redis”时，应沿这条链逐项回答：状态丢了能否从数据库重建？Hash 还是 String？谁负责更新和删除？TTL 是多少、内存满了怎么办？主从切换时能否接受旧值？处理任务要不要 Stream 和确认？这些答案比孤立记住某个命令更接近真实系统设计。

## 13. 本章总结

- Redis 的优势来自内存和原生数据结构，不能用“所有操作都是 O(1)”概括。
- 先划分事实、可重建副本和派生状态，再决定缓存和持久化策略。
- Cache Aside 的关键是明确事实源、失效时机和回源限流；穿透、击穿、雪崩、热 key、大 key 要分别处理。
- RDB 是快照，AOF 是追加日志；复制和 Sentinel/Cluster 解决可用性与扩展，不等于强一致。
- 分布式锁必须有随机 token、原子释放、合理 TTL；强正确性场景仍需要数据库约束、版本号或 fencing。
- Pipeline、事务、Lua、Stream 解决的是不同问题，不能互相替代。
- 生产设计最终要落到超时、降级、监控、容量和恢复，而不是只列命令清单。

## 14. 官方资料

- [Redis 数据类型](https://redis.io/docs/latest/develop/data-types/)
- [Redis 缓存模式](https://redis.io/docs/latest/develop/use/patterns/)
- [Redis Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis 持久化](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis 复制](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
- [Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis 分布式锁](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
