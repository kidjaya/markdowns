# MongoDB · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
> 版本基线：MongoDB 4.4 / 5.0 / 6.0 + WiredTiger

## 一、核心心智模型

1. **文档模型的核心价值是"一次读取拿到完整对象"。** 关系模型里一个业务对象要 join 五张表，MongoDB 里可能一次查询就够。这带来的收益（少 join、少round trip）和代价（冗余、更新要广播、无法跨文档约束）是同一个事物的两面。**选不选用 MongoDB 的判断标准只有一个：你的数据能不能被自然地聚成一个个独立访问的聚合体（Aggregate）。**

2. **MongoDB 是 schemaless，但不是 schemaless-of-thought。** 没有 schema 校验带来了迭代速度，也意味着数据不一致会静默地积累：某个字段有的文档是 string 有的是 int32，有的是数组有的是单值——等到查询时发现"某些数据查不出来"就晚了。生产必须加 **JSON Schema 校验器**（`validator` + `validationLevel`/`validationAction`）。

3. **一切性能问题都要回到"检查了多少文档"。** 和 MySQL 的 rows 类似，MongoDB 的 `explain("executionStats")` 里 `totalDocsExamined` / `nReturned` 的比值是唯一的评判标准：理想应该接近 1，如果几十甚至几万倍，就是 COLLSCAN 或索引没选择性。

4. **分片键的选择是不可逆的架构决策。** 4.4 之前分片键完全不可修改，5.0 起支持 `refineCollectionShardKey`（只能加后缀字段），6.0 支持非分片键字段做相等前pian条件的 fine。但**换分片键基本等于重建集合**。所以它的选择必须在设计阶段一次做对。

5. **MongoDB 的"最终一致"是可以被**（看写关注级别）**调节的。** 默认 `writeConcern: w:1` + `readConcern: local`，意味着主从切换时可能丢失尚未复制的数据。理解这点才能解释"为什么我这里写成功的数据忽然没了"。

## 二、必懂机制

### 2.1 数据模型与 _id

- **BSON** 是 JSON 的二进制超集，多了类型如 ObjectId、Date(毫秒)、Binary、Decimal128、Long、Timestamp。注意 JSON 里所有数字都是 double，**BSON 区分 Int32/Int64/Double/Decimal128**——金额必须用 Decimal128 或 Long 存最小单位，不能用 Double。
- **ObjectId 是 12 字节**：4 字节时间戳（秒） + 5 字节随机值（3 机器+2 进程） + 3 字节递增计数器。因此：
  - 它**天然按创建时间大致有序**，可用于时间范围查询（`_id > ObjectId.fromDate(...)`）
  - 前 4 字节可以取时间，但**只有秒级精度，且不能当作业务创建时间的权威**——因为是驱动生成的，时钟可能偏移
- **限制**：单文档 16MB；嵌套深度 100 层；命名空间（库名.集合名）长度上限约 255 字节；索引 key 不能超过 1024 字节。

### 2.2 索引（和 MySQL 的思路差异很大）

支持的类型：单字段、复合、多键（数组）、文本、地理 2dsphere/2d、哈希、TTL（单字段+秒）、稀疏 sparse、部分 partial（带 filter）、唯一 unique、通配符 Wildcard（4.2+，给文档里的动态字段建索引）。

**复合索引的 ESR 规则**（这是 MongoDB 索引设计最重要的原则）：
```
索引字段顺序应为：Equality（等值匹配）→ Sort（排序字段）→ Range（范围查询）
```
理由：等值字段能精确缩小范围并把后续 field 的排序保持在索引有序性里；排序字段紧跟其后可以避免内存排序；范围字段放最后，因为它之后的字段索引就失效了（范围查询之后的 key 不再有序）。

```javascript
// 查询：status=1 且 create_time 在范围内，按 score 降序取 20 条
db.order.find({status:1, create_time:{$gte:ISODate("2024-01-01"),$lt:ISODate("2024-02-01")}})
        .sort({score:-1}).limit(20)
// ESR 推导：等值是 status → 排序是 score → 范围是 create_time
db.order.createIndex({status:1, score:-1, create_time:1})
```
> 注意：如果 sort 前还有范围查询，MongoDB 通常无法使用索引排序，`explain` 里会出现 `SORT` stage，进而触发 32MB 内存排序上限报错。

**多键索引**（数组字段）：一个文档会产生多个索引项。**复合索引里最多只能有一个数组字段**（因为多字段笛卡尔积会让索引爆炸）。此外 `{a:1, b:1}` 索引里如果 a 和 b 都是数组，是完全不允许的。

**索引前缀规则与 MySQL 的最左前缀一致**，但要注意 MongoDB 是 **`{a:1,b:1}` 与 `{a:-1,b:-1}` 对某些排序是等价的**，而 `{a:1,b:-1}` 是另一个索引。

```javascript
db.coll.getIndexes()
db.coll.aggregate([{$indexStats:{}}])                        // 看索引使用次数，找冷索引
db.coll.aggregate([{$planCacheStats:{}}])                    // 看缓存的执行计划
db.adminCommand({setParameter:1, planCacheEnabled:true})
```

### 2.3 查询诊断

```javascript
db.coll.find({...}).explain("executionStats")
// 或 allPlansExecution 看所有候选计划
```
关键字段：
- `winningPlan.stage`：`COLLSCAN`（全集合扫）/ `IXSCAN`（索引扫）/ `FETCH`（回表）/ `SORT`（内存或磁盘排序）/ `COUNT_SCAN` / `PROJECTION`
- `totalKeysExamined`：扫了多少索引项
- `totalDocsExamined`：检查了多少文档
- **`nReturned`**：返回多少
- `executionTimeMillis`

**判断标准：`totalDocsExamined / nReturned` 越接近 1 越好。** 如果扫了 10 万返回 10 条，索引就有问题。

注意：MongoDB 有**计划缓存**，同一 query shape 第一次跑会比较多个候选计划并择优缓存。这意味着：
- 第一次查询可能选错计划并一直用下去 → 发现慢时要 `planCacheClear()` 或 `hint()` 验证
- 索引增删后要清 plan cache 才会重新评估

慢查询设置：
```javascript
db.getProfilingStatus()                 // {was: 0, slowms: 100}
db.setProfilingLevel(1, {slowms: 100})  // 1=记录慢查询
// 查询存放在 db.system.profile（是个 capped collection，会覆盖）
db.system.profile.find().sort({ts:-1}).limit(10)
```

### 2.4 聚合框架

Pipeline 按顺序对文档做变换，每个阶段有 100MB 内存限制（`allowDiskUse: true` 可落临时文件）。核心原则：**`$match` 和 `$project` 尽量前置**，减少流经管道的文档数和字段大小。

| 阶段 | 要点 |
|---|---|
| `$match` | 能使用索引，但要**放在管道最前面**；`$match` 里用 `$expr`/`$where` 则无法用索引 |
| `$project` | 减少字段能显著减少后续 stage 内存 |
| `$group` | 可能内存爆；`$group` 的 `_id` 是分组 key；无 _id 字段会整组算一个 |
| `$lookup` | 等效 left outer join，**由 mongos/主节点在本地执行嵌套循环**：对被 join 集合的每个匹配都要查一次。目标 key 无索引时等于全集合扫描，数据量大时应改为应用层拼装或做字段冗余 |
| `$unwind` | 展开数组，会撑大文档数，注意 `preserveNullAndEmptyArrays` |
| `$facet` | 多路子管道，一次 scan 算多个结果（适合分页带总数） |
| `$sort` + `$limit` | 相邻时可以用 top-k 优化，sort 不必全排 |
| `$merge` / `$out` | 写回集合，做物化 / ETL |

> 有 `$sort` 但没有 limit 且超过 100MB → `Sort exceeded memory limit of 104857600 bytes`。如果开了 allowDiskUse 会走临时文件（性能暴跌且占 /tmp 空间）。

### 2.5 写关注 / 读关注 / 读偏好（三个维度别混）

- **writeConcern `{w, j, wtimeout}`**：`w:1`（主确认，默认）/ `w:"majority"`（多数派，不丢）/ `j:true`（已写 journal，宕机不丢）；`wtimeout` 防止无限等待但**超时不代表失败**
- **readConcern**：`local`（默认，可能读到回滚数据）/ `majority`（只读已多数提交，保证不读 rollback）/ `linearizable`（线性一致，性能最差）/ `snapshot`（多文档事务 snapshot）
- **readPreference**：`primary`（默认）/ `primaryPreferred` / `secondary` / `secondaryPreferred` / `nearest`（跨机房可用 useful）

三者承诺不同的事：**writeConcern 决定"数据会不会丢"，readConcern 决定"会不会读到未提交/回滚的数据"，readPreference 决定"去哪个节点读"。**

> 常见的正确配置：写 `w:majority` + 读 `readConcern: majority`，这是"不丢且不读脏"的标配。只读本地 latest 数据用 `local` + `primary`。

### 2.6 副本集与选举

成员：primary / secondary / arbiter（仲裁，不存数据只投票，建议不用或少用）/ hidden / delayed（延迟备份）/ priority:0。

- **oplog**：主节点 local.oplog.rs 里按时间顺序的操作日志（幂等的），从节点拉取并重放。默认占磁盘的 5%（64 位），太小会导致从节点脱节做 initial sync
- **选举**：基于 Raft 变体。触发条件（主心跳丢失 >10s、节点新加入、主动 stepDown）。获得**多数票且要有最新 oplog**的节点才能被选为主。**隐藏/延迟节点 priority=0 不能被选主**
- **rollback**：主节点宕机前有一些写入没来得及复制，新主上来后从节点会回滚这部分数据，写入文件 `rollback/` 目录，需要人工介入。**这就是必须配 writeConcern majority 的原因**
- **成员的 votes 上限 7 个投票成员**，其余要设 `votes:0`

```javascript
rs.status()      // members[i].stateStr / optimeDate / lastHeartbeat
rs.conf()        // priority / votes / hidden / slaveDelay
rs.printSecondaryReplicationInfo()   // 延迟秒数
rs.printReplicationInfo()            // oplog 窗口（还够多久）
```

### 2.7 分片

三个角色：`mongos`（路由，无状态）/ `config server`（元数据，必须是副本集）/ `shard`（每个通常也是副本集）。

**分片键选择三要素**：
1. **基数足够大**（cardinality）—— 否则 chunk 数量有限，无法分裂，热点永存
2. **分布均匀**（frequency）—— 某个值占了 30% 的数据，那个 chunk 就是 jumbo chunk
3. **避免单调递增**（monotonicity）—— 时间戳、自增 id 做 ranged 分片会导致所有新写落到最后一个 chunk（因为它最大的 range 永远在最后），形成写热点

| | ranged（范围） | hashed（哈希） |
|---|---|---|
| 范围查询 | 能 targeting，只路由到相关分片 | 必须 scatter-gather 广播全部分片 |
| 写分布 | 单调 key 会热点 | 均匀 |
| 适用 | 有明显范围查询需求 | 只按等值查，要均匀写 |

**命中越多越好**：如果查询条件里不含分片键，mongos 就得广播到所有分片（scatter-gather）再合并，吞吐随分片数下降。所以**分片键必须包含在绝大多数查询条件中**。

运维要点：
- chunk 默认 64MB（可配 1-1024），达到阈值触发分裂，balancer 负责在各 shard 间搬移以保证均衡
- **balancer 迁移会产生大量删除操作**，建议在业务低峰窗口执行（`activeWindow`）
- **jumbo chunk**：超过 chunk 大小且不可分裂（所有文档分片键值相同），会阻止迁移
- 迁移期间**正在写了** 可能会出现重复计数（orphaned documents 孤儿文档），`cleanupOrphaned` 处理

### 2.8 WiredTiger 引擎

- `cacheSizeGB` 默认取 50% RAM（容器里要看 Docker 造成的误判，需要显式设 `wiredTigerCacheSizeGB`）
- **cache 里有读写 ticket**：默认读写 ticket 各 128；如果慢查询堆积，ticket 会被占满，后续请求全部排队 —— 表现为 `db.serverStatus().wiredTiger.concurrentTransactions` 里 available 为 0，这是"看起来 CPU 不高但全卡住"的常见原因
- checkpoint 默认 60 秒或 journal 达 2GB
- journal 保证 durability，writeConcern j:true 才等 journal 落盘
- **压缩**：默认 snappy（集合），索引用前缀压缩；zstd 压缩率更高但 CPU 更贵
- **文档级并发控制**（乐观锁）：写冲突会返回 WriteConflict 由驱动自动重试

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| `Sort operation used more than the maximum 33554432 bytes` | 无索引的 sort，超过 32MB 内存排序上限 | 建符合 ESR 的索引；或加 allowDiskUse 应急（性能暴跌） |
| `$regex` 查询巨慢 | 前导通配（`^.*abc`）或大小写不敏感正则无法用索引，走 COLLSCAN | 用 text search；或改前缀锚定 `/^abc/` 才能用索引 |
| 明明有索引还是慢 | 索引 field 顺序不满足 ESR；或 plan cache 缓存了糟糕的旧计划 | 重建索引顺序；`planCacheClear()` 后重跑；`$indexStats` 看实际使用 |
| `$lookup` 后管道很慢 / 内存爆 | $lookup 在本地逐条去目标集合查，本质 join 嵌套循环 | 目标 key 必须有索引；或冗余字段；或应用层两次查询 + 内存拼装 |
| `$nin` / `$ne` / `$not` 慢 | 这些操作符选择性差，索引定位困难 | 用正向条件改写；或用 `$or` 正向枚举 |
| 写入成功后主从切换数据消失 | writeConcern 默认 w:1，从库还没复制 | 关键数据写 `w:majority` + 读 `readConcern: majority` |
| 数组字段无限增长导致文档膨胀 | 经典 unbounded array 反模式 | 分集合（bucket pattern）或按时间分表水平；数组超过几百个元素就该考虑拆分 |
| 从库延迟持续上涨 | oplog 窗口太小导致从节点脱节需全量重同步；或从库回放速度跟不上写入量 | 加大 oplogSizeMB；`rs.printReplicationInfo()` 看剩余窗口；必要时提升从库资源 |
| 连接池满导致应用 hang | 驱动连接池被打满后新请求全部排队等待 | 调整 maxPoolSize；优化慢查询；避免一个服务开过多 MongoClient |
| 用 string 存 ObjectId 查不到数据 | BSON 类型严格区分，`_id` 是 ObjectId，传 string 不匹配且无法走索引 | 类型一致；迁移时注意 ETL 保留类型 |
| 大批量写入后 CPU 一直高 | WiredTiger cache 未释放/ checkpoint 压力/ ticket 打满 | 观察 `db.serverStatus().wiredTiger.cache` 与 concurrentTransactions |
| count(document) 在分片集群下不准 | 迁移过程中文档可能出现在两个分片（孤儿文档） | 用 `aggregate count` 或直接 `estimatedDocumentCount`（快但不准）/ `countDocuments`（准但慢） |
| 集合越大单条 update 越慢 | 更新导致文档变大要移动位置（padding 不够） | 预估文档大小；避免数组不定增长 |

## 四、往下钻的入口

- **诊断命令三件套**：`db.serverStatus()`（看 connections、opcounters、wiredTiger.cache、concurrentTransactions）、`db.currentOp()`（看正在跑的长操作，配合 `secs_running > 5` 过滤）、`db.coll.explain("allPlansExecution")`
- **慢日志分析**：`system.profile` + `mtools` 工具包（`mloginfo`、`mplotqueries`、`mlogfilter`）
- **官方最值得读的章节**：MongoDB Manual → *Indexes → Indexing Strategies（ESR 一节）*、*Sharding → Shard Keys*、*Replication → Replica Set Election*、*Performance → Explain Results*。这几节是当前bounds中的官方精华，其余多为 API 手册
- **源码瞄点**：WiredTiger 的 `src/session/session_dhandle.c`、mongod 的 `src/mongo/db/exec/`（各个 stage 实现，`SORTStage`、`IndexScan`）、`src/mongo/db/repl/`（选举 Raft）

## 五、达标自检

1. 什么业务形态的数据适合用 MongoDB？什么样的查询要求下应该转回 MySQL / Elasticsearch？
2. ObjectId 里包含哪几部分？它能用于精确的时间范围查询吗？有什么限制？
3. ESR 规则是什么？为什么 Range 字段必须放最后？
4. 复合索引里能不能同时有两个数组字段？为什么？
5. `{a:1,b:1}` 索引能支持 `sort({a:1,b:-1})` 吗？支持则 sort 是怎么做的？
6. `explain` 里要看哪几个指标来判断索引是否有效？理想比值是多少？
7. 什么叫 scatter-gather？什么情况下会发生？如何避免？
8. 单调递增字段做 ranged 分片会出什么问题？用 hashed 分片会失去什么？
9. writeConcern / readConcern / readPreference 分别管什么？画出一个三者组合导致"数据写了但读不到"的具体时序。
10. `$lookup` 在什么情况下性能会崩？你的替代方案是什么？
11. 出现 `Sort exceeded memory limit` 时，你的第一反应是加 allowDiskUse 吗？正确的做法是什么？
12. 为什么 MongoDB 会有 plan cache 导致"加了新索引还是走老计划"？怎么清理？
13. 副本集选举时有哪些节点不能被选为主？为什么？
14. 什么情况下会发生 rollback？如何彻底避免？
15. 一笔重要的订单数据（不能丢），你的 writeConcern 和 readConcern 该怎么配？代价是什么？
16. WiredTiger 的 ticket 是什么？为什么会打满？打满的表现是什么？
17. `$match` 放在 pipeline 后面会发生什么？`$expr` 放在 `$match` 里有什么代价？
18. 集合里有 800 万文档，某个字段的值可能是 string 也可能是 int32，会发生什么事？怎么治理？
19. 分片键选错之后还有什么补救手段？分别在什么版本可用？
20. 什么时候 MongoDB 的多文档事务不该用？（至少给出三个理由）
