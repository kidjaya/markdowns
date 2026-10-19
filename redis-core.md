# Redis · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
> 版本基线：Redis 6.x / 7.x（标注了 5.x 的差异）

## 一、核心心智模型

1. **单线程是把双刃剑。** Redis 的网络 IO 和命令执行都在一个线程里（6.0 起网络读写可多线程，但**命令执行仍然单线程**）。这带来了原子性、无锁、无上下文切换的红利，也意味着——**任何一个慢命令会阻塞整台实例的所有请求**。所以 Redis 的性能问题 90% 不是吞吐不够，是被某条命令卡住了。

2. **所有的 O(n) 命令都是事故预备队。** `KEYS`、`HGETALL` 一个百万字段的 hash、`SMEMBERS` 一个大 set、`LRANGE 0 -1`——它们的耗时和数据规模成正比，而 Redis 扛不住毫秒级的长命令。判断标准很简单：**操作的复杂度是不是和你无法确定上界的元素数量有关**。

3. **Redis 是内存数据库，但它的瓶颈常常在 fork 和磁盘。** 持久化、主从全量同步都要 fork 子进程。fork 本身虽是写时复制（COW），但父进程的内存页表要复制，且**如果系统开了透明大页 THP，fork 会把 512 倍内存的延迟抖出来**——这是官方文档明确要求关闭 THP 的原因。

4. **过期的 key 不一定真的没了。** Redis 用"惰性删除 + 定期采样"两条腿走路：访问时才检查删除（惰性），外加每秒 10 次（hz）的随机采样清理（定期）。所以过期 key 的内存释放是延迟的，可能出现"已过期但内存没降"。知道这一点，才能理解为什么有些实例 allkeys-lru 也扛不住内存增长。

5. **Redis 不提供银弹。** 它不保证缓存与数据库的一致性，分布式锁也不保证绝对安全（存在 NPC 问题），事务不支持回滚。所有"Redis 解决 XX"的方案，本质上都是把复杂度推给了应用层。

## 二、必懂机制

### 2.1 数据结构与底层编码

Redis 的每个值都是 `redisObject`：`type`（5 大类型）+ `encoding`（底层实现）+ `lru`（24 bit，LRU 时钟或 LFU 计数）+ `refcount`。**同一个 type 在不同数据量下会切换 encoding**，这是理解内存占用的关键。

| 类型 | 小数据量 encoding | 大数据量 encoding | 切换阈值配置项 |
|---|---|---|---|
| String | int / embstr（≤44 字节） | raw（SDS） | 44 字节边界 |
| List | listpack（≤6.x 是 ziplist） | quicklist | `list-max-listpack-size` |
| Hash | listpack | hashtable | `hash-max-listpack-entries=128` / `-value=64` |
| Set | intset（全整数） | hashtable | `set-max-intset-entries=512` |
| ZSet | listpack | skiplist + dict | `zset-max-listpack-entries=128` / `-value=64` |

几个值得记住的细节：
- **SDS**（简单动态字符串）不是 C 字符串：它用 `len` 记录长度（O(1) 取长度、二进制安全），且 SDS 有 alloc 字段做空间预分配。
- **embstr** 是一次分配连续内存（redisObject + SDS 相邻），raw 是两次；边界 44 字节是因为 Redis 的 jemalloc 分配 64 字节一档。
- **ZSet 同时用 dict 和 skiplist**：dict 保证按 member 查 score 是 O(1)，skiplist 保证按 score 范围查是 O(logN)。两个结构共享同一份数据。
- **quicklist = linkedlist + listpack**，是 list 的中庸解法：每个节点是一个 listpack，兼顾了内存和增删效率。

```bash
OBJECT ENCODING mykey        # 看实际 encoding
MEMORY USAGE mykey           # 实际占用字节
DEBUG OBJECT mykey           # 字段数/序列化长度（生产慎用）
```

### 2.2 渐进式 rehash

dict 维护两个哈希表 `ht[0]` 和 `ht[1]`。扩容时不是一次性搬迁，而是：
```
rehashidx = 0（开始搬运）
每次增删改查时：顺带搬运 ht[0] 的一个 bucket
  → 查找跨两张表、新增只写 ht[1]、删除两边都删
全部搬完后 rehashidx = -1，ht[1] 变成 ht[0]
```
**这个设计保证了 1000 万 key 的扩容不会被一次搬运卡死几百毫秒**。副作用也明显：rehash 期间内存里同时有两张表，峰值内存会翻倍；另外 `BGSAVE` 期间会提高扩容触发因子（避免 COW 放大）。

> 注意：redis-cli 里执行 `DEBUG HTSTATS`、`dict` 相关调试只在少量场景有用；更实用的是看 `INFO memory` 的 mem_fragmentation_ratio。

### 2.3 过期删除与内存淘汰

过期删除：
- **惰性删除**：访问 key 时检查 expires 字典，过期就删并返回 nil
- **定期删除**：每秒 `hz`（默认 10）次，随机抽 20 个带过期时间的 key，删掉过期的；如果本轮超过 25% 过期就再来一轮，一轮上限 25ms（ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC）

内存淘汰（8 种 maxmemory-policy）：
- `noeviction`（默认！不淘汰，写请求直接报错）—— **生产环境几乎从不该用，且必须有 maxmemory**
- `allkeys-lru` / `volatile-lru`：近似 LRU。随机采样 `maxmemory-samples`（默认 5）个 key 淘汰最久没访问的那个
- `allkeys-lfu` / `volatile-lfu`：近似 LFU，需要 4.0+
- `allkeys-random` / `volatile-random`
- `volatile-ttl`：淘汰剩余 TTL 最短的

**LRU 是"近似"的**：`redisObject` 里只有 24 bit 存 LRU 时钟（秒级精度），Redis 不维护全局链表。之所以敢近似，是因为采样 5 个再挑一个的效果已经很接近真 LRU，代价却低到可以忽略。

**LFU 的巧思**：同样用 24 bit，高 16 位存分钟级时间戳（ldt，做衰减），低 8 位存对数计数器（logc，最大 255，用概率递增实现）。`lfu-decay-time`（默认 1）控制衰减速度——这就是 LFU 能自动淘汰"曾经很热但现在冷"的 key 的原因。

```bash
INFO stats | grep expired      # 累计删除的过期 key 数
INFO memory                     # used_memory / maxmemory / mem_fragmentation_ratio
CONFIG GET maxmemory-samples    # 提高采样数（如 10）会让 LRU 更精确但 CPU 更贵
```

### 2.4 持久化：RDB / AOF / 混合

**RDB**：
- `SAVE` 阻塞主线程（永远别在生产用）；`BGSAVE` fork 子进程
- fork 后子进程基于当时的内存快照写盘，父进程继续服务；被修改的页触发 COW，父子各一份
- 优势：文件紧凑，恢复快。劣势：丢失最后一次快照之后的所有数据

**AOF**：
- 命令执行完才记录到 `aof_buf`（"写后日志"），因此不会阻塞当前命令，代价是宕机时会丢最后一段
- `appendfsync`：`always`（每条刷，最安全但最慢）、`everysec`（默认，每秒刷，宕机丢 1 秒）、`no`（交给 OS）
- **AOF 重写**：`BGREWRITEAOF`，fork 子进程根据当前数据库状态生成新 AOF（不读旧 AOF）；期间新命令写 `aof_rewrite_buf`，最后追加上
- 关键点：**如果正在重生且 `no-appendfsync-on-rewrite=yes`，主进程这段时间不刷 AOF，会丢数据**

**混合持久化**（`aof-use-rdb-preamble yes`，4.0+ 常用配置）：重写时前半段是 RDB 二进制，后半段是增量 AOF 命令。兼顾了加载速度和增量完整性。

启动加载顺序：**有 AOF 就先加载 AOF**（因为它更完整），否则加载 RDB。

> 最常见的生产事故是 THP 没关：BGSAVE 的 fork 抖动可达秒级。检查：`cat /sys/kernel/mm/transparent_hugepage/enabled` 必须是 `never` 或 `madvise`。

### 2.5 主从复制

旧版用 `SYNC`（全量），2.8 后 `PSYNC`：
- **全量**：从库发 `PSYNC ? -1`，主库 fork 子进程生成 RDB 发给从库；期间新命令记在 replication buffer；从库加载完 RDB 后再接收增量
- **增量**：从库断线重连时带上自己的 `runid` 和 `offset`；主库的 `repl_backlog`（默认 1MB 环形缓冲）里还留着这段就只发增量，否则被迫全量

一个经典死循环：**repl_backlog 太小 + 网络抖动 → 频繁全量 → 主库 fork+RDB 压力大 → 更抖**。调优就要加大 `repl-backlog-size`（比如 100MB）和 `client-output-buffer-limit slave`（防止 RDB 传输慢导致从库连接被杀）。

主从的读写（延迟-lock）：过期 key 由主库负责，主库删除后向从库传播 DEL 命令；从库读到已过期但未删除的 key 会返回它（3.2 后部分修正，但不保证），这是脏读的一个来源。

### 2.6 Sentinel

三个要素：主观下线（sdown，单个哨兵认为挂了，`down-after-milliseconds`）、客观下线（odown，`quorum` 个哨兵都认为挂了）、领导者选举（Raft 变体，**先到先得、半数以上**）。

部署要求：**至少 3 个哨兵且为奇数**，哨兵节点数 = quorum × 2 - 1 左右的换算关系要算清楚。两个哨兵时，quorum=1 会导致网络抖动就误切。

脑裂与数据丢失：主库被误判下线 → 从库被提升 → 老主库还活着时客户端还在写它 → 它恢复后会被清空全量同步。缓解手段：

```conf
min-replicas-to-write 1      # 至少要有 1 个从库正常才接受写
min-replicas-max-lag 10      # 从库延迟超过 10 秒就认为它"不算"，配合上面
```

### 2.7 Cluster

- **16384 个槽**：`slot = CRC16(key) mod 16384`。为什么不是 65536：集群心跳消息里有 bitmap 记录自己负责的槽，65536 需要 8KB 而 16384 只要 2KB，且官方认为 1000 个节点足够了（这个数值是为了让心跳不成为瓶颈）
- **Gossip**：节点每秒随机 ping 几个节点，5 秒或超过半数超时触发节点下线消息散布
- **MOVED**：永久重定向，说明槽已经迁移完。**ASK**：临时重定向，说明正在迁移中，客户端要先发 ASKING
- **hash tag**：`{user1000}.name` 和 `{user1000}.age` 只按 `user1000` 计算槽，因此可以在同一槽，支持跨 key 操作（如 MGET）
- `cluster-require-full-coverage`：默认 yes，任何一个槽无人负责时整个集群停止服务

### 2.8 Big key / Hot key / 遍历

**Big key 的危害**：删除时释放大片内存触发卡646ms以上（要看 jemalloc 是否会立刻归还 OS）、网络传输占满带宽、迁移失败（`MIGRATE` 有超时）、序列化耗时。

```bash
redis-cli --bigkeys                          # 采样统计（不够准确但可用）
redis-cli --memkeys                          # 按内存排序（6.x+）
MEMORY USAGE key
```
安全删除：**永远不要用 `DEL` 删大 key**，用 `UNLINK`（异步释放，`lazyfree-lazy-del` 系列配置），或是分批 `HSCAN`/`ZSCAN` 删除。Hash 的通用做法是：拆成小 hash 分片（`{key}:{hash(field)%100}`）。

**Hot key 的识别**：`redis-cli --hotkeys`（要求 maxmemory-policy 是 LFU 才有意义）；更实际的做法是客户端本地计数打点，或者 `monitor`/自定义采样。**解法**：本地缓存 + 分片复制（`key:{1..N}` 随机或按用户 hash）+ 读写分离。

**遍历必须用 SCAN**：`SCAN cursor MATCH pattern COUNT n` 增量迭代，`COUNT` 只是"建议值"，返回可能重复（要业务侧去重），并且中间有修改不保证快照一致性。

### 2.9 Lua 与事务

**Lua 是 Redis 的原子操作单元**：脚本执行时，整段 Lua 被视为一个命令，不会被打断。这是实现复杂原子逻辑的唯一正解（比如库存扣减 + 记录 + 判断三者一起）。

坑：脚本要短要快，**不能有阻塞操作**；`lua-time-limit`（默认 5000ms）到点后 Redis 只是标记 Busy 让后续请求可以对原:get 'KILL'，脚本并不真的被终止（为防止副作用中途停不下来），要 kill script / shutdown nosave。

**事务 Redis 里的 MULTI/EXEC**：
- 语法错误（入队失败）→ 全部不执行
- 运行错误（运行时类型不对）→ **已执行的不会回滚，Redis 没有回滚**
- 所以 Redis 事务的语义是"原子地批量执行"，不是"ACID 里的 A"
- `WATCH` 提供乐观锁：被 watch 的 key 变了则 EXEC 返回 nil

### 2.10 缓存三大问题

| 问题 | 定义 | 对策 |
|---|---|---|
| 缓存穿透 | 查一个根本不存在的 key，每次都落到 DB | 布隆过滤器前置拦截；**空值缓存**（key 不存在也缓存 null，TTL 要短，如 60s） |
| 缓存击穿 | 一个热点 key 恰好过期，瞬间大量请求打到 DB | 互斥锁重建（Redis SETNX + double check）；**逻辑过期**（永不过期，value 里存过期时间，异步刷新，无阻塞）；热点 key 永不过期 + 主动刷新 |
| 缓存雪崩 | 大量 key 同一时刻集体过期 | 基础 TTL + 随机抖动（`TTL = base + random(0, 300)`）；多级缓存；熔断限流降级 |

一致性：Cache Aside 是事实标准 —— **读：命中返回，未命中读 DB 再回写；写：先更新 DB，再删除缓存**。

为什么是"删"而不是"更新"？因为更新缓存容易出现并发写导致脏数据（两个并发写谁先谁后不确定），而且很多场景缓存是用到才构建的组合结果，直接更新 compute 成本高。

为什么"删"也不彻底？经典的并发场景：
```
1. 缓存 key 恰好过期
2. A 读 DB，得到 v1
3. B 写 DB 改成 v2，然后删缓存
4. A 把自己手里的 v1 写入缓存  ← 脏数据驻留到下次过期
```
这个窗口虽然极小但存在。**延迟双删也不能根治**（只是缩小窗口）。真正要强一致，得靠：重试机制（删失败进 MQ 重试）、订阅 binlog（Canal/Debezium 消费 binlog 异步删缓存，把删缓存的责任从应用移到数据管道）、或串行化。

### 2.11 分布式锁

最小可用版：
```lua
-- 加锁
SET lock_key random_value NX PX 30000
-- 释放：必须 Lua 保证"判断+删除"原子
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else return 0 end
```
四个必知问题：
1. **必须给 value 赋随机值**，否则可能删掉别人加的锁（自己超时后别人刚加上）
2. **必须有过期时间**，否则持锁客户端宕机导致死锁
3. **锁超时 ≠ 业务完成**：业务还没跑完锁过期了，别人进来。Redisson 的看门狗（`lockWatchdogTimeout` 默认 30s，每 10s 续期）就是解决这个问题
4. **主从切换丢锁**：客户端在主库加锁成功，锁还没同步到从库主库就挂了，从库提升后另一个客户端又能加锁。Redlock 试图解决但争议很大（Martin Kleppmann 指出它依赖时钟假设，存在 GC 停顿、IO 停顿的 NPC 问题）

结论：**Redis 锁适合"允许偶发失效、用于减少冲突"的场景（幂等、去重）。做资金、库存这种强一致场景，要配合数据库的乐观锁或唯一约束兜底。**

### 2.12 性能相关

6.0 的多线程：**只多线程化了网络读（read/decode）和网络写（encode/write）**，命令执行仍是单线程。`io-threads`（建议设小于核数，如 4）、`io-threads-do-reads`。

其他：
- **potentially pipeline**：把多条命令打包一次往返，减少 RTT。释放本地端口 sockfd 压力，吞吐常常×5~10
- **MGET 比 pipeline GET 更省**（一条命令），但 cluster 下要注意跨槽，要用 hash tag
- **客户端缓冲区**：`client-output-buffer-limit` 三类 normal/replica/pubsub，慢消费者会撑爆导致连接被杀
- **lazy free 四件套**：`lazyfree-lazy-eviction / -expire / -del / replica-lazy-flush`，把释放内存的活交给后台线程

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 线上突然所有请求超时几十毫秒到几秒 | 有人跑了 `KEYS *` 或按 mode 删大 key；或 BGSAVE fork 抖动 | 禁用 `KEYS`；开启 lazy free；关 THP；慢查询日志 `SLOWLOG GET` 定位 |
| 内存一直涨，明明 key 不多 | 客户端缓冲区堆积、 replica buffer backlog、散列 malloc 碎片 `mem_fragmentation_ratio > 1.5` | `INFO clients`（看 `client_recent_max_input_buffer`）、`MEMORY DOCTOR`、开 activedefrag |
| OOM 被 kill | `maxmemory` 没配，redis 无上限吃内存被 OS OOM killer 干掉 | 必须设 `maxmemory`（给实例预留 20~30% 给 COW 与碎片） |
| 删除一个大 hash 时 redis 卡住几百毫秒 | 同步释放大量内存，jemalloc 归还 OS 耗时 | `UNLINK` 或分批 HSCAN 删；或直接设计上避免大 key |
| cluster 里 MGET 报 CROSSSLOT | key 散落在不同 slot | 用 hash tag 保证同槽，或客户端分组并行 MGET |
| 主从频繁全量同步 | `repl-backlog-size` 太小，断联超过阈值 | 调大 backlog（几十到几百 MB）；排查网络抖动 |
| 从库读到过期数据 | 3.2 前从库自身不删过期 key，依赖主库 DEL 传播 | 升级；读从库的场景要容忍或改只读 key 不过期数据 |
| Lua 脚本执行后 redis 一直 BUSY | 脚本里有慢操作/死循环，`lua-time-limit` 只是标记不终止 | `SCRIPT KILL`（若脚本无写操作）或 kill script / shutdown nosave |
| 事务里一条命令失败但其他还是执行了 | Redis 事务不回滚（设计如此） | 业务侧保证顺序；需要原子就用 Lua |
| 连接池打满，报 connection refused | Lettuce/Jedis 池太小或 `maxmemory-policy=noeviction` 导致写阻塞 | `maxclients`、`ulimit -n`、线程池与连接池匹配；不要把 Redis 连接池调到几千 |
| 数据没丢但 Redis 里读到的和 DB 不一致 | Cache Aside 的并发窗口；或先删缓存后更新 DB 的顺序写反了 | 先更 DB 再删缓存；异步订阅 binlog 兜底；删除失败要重试 |
| 热 key 打爆单分片 CPU | 某个 key 集中访问 | 本地缓存、key 分片打散、读写分离 |

## 四、往下钻的入口

- **命令 / INFO 排障三板斧**：`INFO commandstats`（看命令调用次数分布，找出滥用）、`SLOWLOG GET 20 slowlog-log-slower-than`（抓取慢命令）、`LATENCY DOCTOR` / `LATENCY DOCTOR` 采样延迟分布
- **内存分析**：`MEMORY DOCTOR`、`MEMORY MALLOC-STATS`、`redis-cli --memkeys`；线下的 `redis-rdb-tools` 分析 RDB 找大 key
- **源码入口（7.x）**：`src/t_zset.c`（跳表实现）、`src/dict.c`（渐进式 rehash）、`src/t_hash.c` / `t_list.c`（encoding 切换逻辑）、`src/expire.c`（惰性删除）、`src/evict.c`（LRU/LFU 的 `performEvictions`）、`src/rdb.c` / `aof.c`、`src/replication.c`、`src/cluster.c`、`src/networking.c`（6.0 多线程 IO 的 `handleClientsWithPendingReadsUsingThreads`）
- **官方文档里真正值得读的段落**：Redis docs → *Redis persistence*（很详细，讲清楚了 fork 与 fsync）、*Redis key eviction*（LRU/LFU 近似算法讲得很透）、*Redis replication*、*Distributed locks*（Redlock 论文与其反驳都值得看）

## 五、达标自检

1. 为什么 Redis 用渐进式 rehash？rehash 期间查询、插入、删除分别怎么处理？期间内存会怎样？
2. ZSet 为什么同时用 dict 和 skiplist，不用其中一个行不行？
3. string 的 embstr 和 raw 的边界为什么是 44 字节？
4. `DEL` 一个大集合和 `UNLINK` 的区别是什么？底层参数叫什么？
5. `maxmemory-policy=noeviction` 在写请求进来时会发生什么？生产应该配什么？
6. LFU 是怎么用 24 bit 既存访问频率又支持衰减的？
7. `appendfsync=everysec` 最多丢多少数据？什么情况下会丢超过 1 秒？
8. AOF 重写期间新写入的命令去哪了？为什么重写不是读旧 AOF？
9. BGSAVE 时如果业务还在大量写入，内存会怎样？为什么会涉及 THP？
10. 主从全量同步一定会发生吗？什么情况下可以避免？相关的两个参数是？
11. Cluster 为什么是 16384 个槽而不是 65536？MOVED 和 ASK 有什么区别？
12. 想在 Cluster 里做 `MGET user:{1001}:name user:{1001}:age`，槽会一致吗？依据是什么？
13. Redis 事务能保证原子性吗？能保证回滚吗？运行期错误的那条命令会怎样？
14. Lua 脚本为什么原子？如果脚本里有阻塞操作会怎样？有哪些手段能停掉它？
15. 缓存更新时"先更 DB 还是先删缓存"？反过来会有什么问题？什么时候这两种都不够？
16. 描述一个能把 Cache Aside 打穿的具体并发时序，并给出你的兜底方案。
17. Redis 分布式锁在"主从切换"这个场景下为什么会失效？你怎么应对？
18. Redis 6.0 的多线程到底多线程化了什么？为什么命令执行不用多线程？
19. 一个 Redis 实例响应突然变慢，你会按什么顺序排查？列出你执行的命令和看的指标。
20. 线上要删除一个有 500 万字段的 hash，你怎么删才能不影响业务？
