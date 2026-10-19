# MySQL · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
> 版本基线：MySQL 5.7 / 8.0 + InnoDB

## 一、核心心智模型

1. **InnoDB 里"索引即数据"。** 聚簇索引的叶子节点存的是整行数据，所以一张 InnoDB 表本质上就是一棵按主键排序的 B+Tree。没有显式主键时会选第一个非空唯一索引，再没有就生成 6 字节的隐藏 `DB_ROW_ID`——这行 rowid 是全局递增的，等于你的插入能力被一个全局计数器串成了临界资源。

2. **所有设计都在对抗"随机写"。** 磁盘随机写比顺序写慢两个数量级。redo log 让事务提交从"随机写数据页"变成"顺序写日志文件"，buffer pool 让随机读变成内存读，change buffer 让非唯一二级索引的写延迟到真正被读取时。理解了这三条，InnoDB 的一半设计就有了答案。

3. **索引是写放大器。** 每个二级索引都是一棵独立的 B+Tree，加一个索引，一次 INSERT 就多一次 B+Tree 插入。更关键的是二级索引叶子存的是主键值，这意味着**主键越大，所有二级索引都跟着膨胀**——用 UUID 做主键，代价是全部二级索引一起变大。

4. **事务隔离 = MVCC 让读不加锁 + 锁让写串行。** 这两套机制各管一半：普通 SELECT 走 MVCC 快照读完全不加锁；写操作和当前读走锁。所谓隔离级别，本质上是 ReadView 生成时机（RC 每次生成、RR 事务首次生成）与间隙锁是否启用（RR 才有 Gap Lock）的组合。

5. **慢的本质永远是"扫描了多少行"。** 索引失效不是玄学，优化器算的是 cost：走索引要回表多少次 vs 全表扫要读多少页。当你看到明明有索引却走全表，八成是优化器估算回表代价高于扫描代价（通常是命中行数超过总行数 20%~30%）。

## 二、必懂机制

### 2.1 索引结构：为什么是 B+Tree

| 候选结构 | 落选原因 |
|---|---|
| 二叉搜索树 / 红黑树 | 树太高。100 万行要 20 层，等于 20 次磁盘 IO |
| 哈希表 | 只支持等值，不支持范围与排序；且哈希冲突退化 |
| B 树 | 每个节点都存数据，非叶子节点塞不下多少 key，导致树变高；且范围查询要中序遍历，来回跳页随机 IO |
| 跳表 | 区间查询效率低于 B+Tree 的叶子链表顺序扫描 |

**B+Tree 的三个决定性优势**：非叶子节点只存 key 不存数据（一个页能放上千个 key，树高通常 2~3 层）；叶子节点用链表相连（范围查询顺着链表扫，顺序 IO）；所有查询都要走到叶子层，复杂度稳定。

一条实用结论：**InnoDB 一个页默认 16KB，主键是 bigint（8 字节）+ 指针（6 字节）约 14 字节，一个非叶子页能放约 1170 个 key。** 树高 3 层可容纳约 1170 × 1170 × (16KB/1KB 行) ≈ 两千万行。这就是为什么单表两千万是个流传甚广的经验阈值——超过它树变高，多一次 IO。

```sql
-- 看一张表的索引结构开销
SELECT index_name, stat_value AS pages
FROM mysql.innodb_index_stats
WHERE database_name='db' AND table_name='t' AND stat_name='size';
```

### 2.2 索引使用与真实失效场景

二级索引叶子存的是主键值，所以通过二级索引查整行必须**回表**（回到聚簇索引再查一次）。避免回表的手段是**覆盖索引**：查询的字段全在索引里，`EXPLAIN` 的 Extra 出现 `Using index`。

```sql
-- 覆盖索引典型写法：只查索引列，Extra = Using index
SELECT id, name FROM user WHERE name = 'x';   -- 有 (name) 索引，无需回表
SELECT * FROM user WHERE name = 'x';           -- 回表
```

**真正会失效的场景**（以及各自的准确性）：

| 场景 | 是否真失效 | 说明 |
|---|---|---|
| 条件列用函数 `where date(create_time)='2024-01-01'` | 失效 | 对列做运算后不再是索引列本身。改写为范围查询即可恢复 |
| 隐式类型转换 `where code = 123`（code 是 varchar） | 失效 | MySQL 会把字符串转数字，等价于对列施 CAST，同上。注意**方向相反时没事**：int 列传 '123' 字符串没问题 |
| `like '%abc'` 前导通配 | 失效 | 'abc%' 可以走，因为前缀有序 |
| `or` 连接非索引列 | 失效 | 有一侧无索引就整体走全表 |
| 两表 join 字段字符集/排序规则不同 | 失效 | utf8mb4_general_ci 与 utf8mb4_0900_ai_ci join 会触发转换 |
| `!=` / `not in` / `not exists` | 通常失效 | 选择性差，优化器倾向全表；但数据分布极端时也可能走索引 |
| 联合索引不满足最左前缀 | 部分失效 | `index(a,b,c)`：where a、a b、a b c 有效；where b、b c 无效 |
| 优化器判断回表代价高 | "主动失效" | 命中行数占比过高时全表更划算，可用 `FORCE INDEX` 验证 |

两个 8.0 的重要优化：
- **索引下推 ICP**（默认开启，5.6+）：联合索引 `index(name, age)` 查 `name like '张%' and age=20`，在索引层就过滤 age，减少回表次数。`Extra = Using index condition`。
- **MRR**（Multi-Range Read）：把回表的主键值在内存排序后再顺序回表，把随机 IO 转成接近顺序 IO。

```sql
SHOW VARIABLES LIKE 'optimizer_switch';  -- 看 index_condition_pushdown / mrr 是否 on
SELECT @@innodb_stats_persistent_sample_pages;  -- 默认 20，统计不准时可加大，改完 ANALYZE TABLE
```

### 2.3 事务与 MVCC

ACID 分别由谁保证：
- **原子性** → undo log（回滚时反向执行）
- **一致性** → 由应用 + 约束 + 其余三者共同保证
- **隔离性** → MVCC + 锁
- **持久性** → redo log（WAL）

**MVCC 的三件套**：每行隐藏列 `DB_TRX_ID`（最后修改的事务 ID）与 `DB_ROLL_PTR`（指向 undo log 的回滚指针）+ 读操作时生成的 **ReadView**。多个版本的行通过 roll_ptr 串成 undo 版本链，顺着链往前找就能读到某个历史版本。

ReadView 的核心是维护一组"活跃事务 ID 列表"，判断某版本的行是否可见：
```
可见 <=> 该版本的 trx_id < 当前最小活跃事务 ID（已提交）
     或  trx_id == 自己这个事务的 ID（自己改的自己可见）
不可见 => 沿 DB_ROLL_PTR 找上一个版本，直到可见或链尾（返回空）
```

**ReadView 生成时机决定隔离级别**：
- READ COMMITTED：每条语句执行前生成新的 ReadView → 同一事务内两次读能看到别人新提交的数据 → 不可重复读
- REPEATABLE READ：事务中第一条快照读时生成，之后复用 → 全程看到同一份数据 → 可重复读

**快照读 vs 当前读**（这是最容易混淆的一处）：

```sql
SELECT * FROM t WHERE id=1;              -- 快照读，走 MVCC，不加锁
SELECT * FROM t WHERE id=1 LOCK IN SHARE MODE;   -- 当前读，加 S 锁
SELECT * FROM t WHERE id=1 FOR UPDATE;   -- 当前读，加 X 锁
UPDATE/DELETE/INSERT                     -- 一定走当前读（要先读到最新版本才能改）
```

**RR 下幻读到底解决了没有？** 分两半回答：快照读靠 MVCC 天然看不到新插入的行；当前读（`select for update`）靠 **Next-Key Lock**（行锁 + 间隙锁）阻止别的事务在区间内插入。所以 InnoDB 的 RR 确实解决了幻读——但前提是你在用当前读。如果你在同一事务里先快照读再当前读，仍会得到"看起来像幻读"的结果（本质是两种读的语义不一致）。

### 2.4 锁

三种行锁形态：
- **Record Lock**：锁单条索引记录
- **Gap Lock**：锁两条记录之间的间隙，不含记录本身，只在 RR 存在。**它的作用就是防止插入，即防幻读**
- **Next-Key Lock** = Record + Gap，锁"左开右闭"区间，是 InnoDB 默认的行锁单位

**退化规则**（必须记）：等值查询且命中唯一索引时，Next-Key 退化为 Record Lock；等值查询未命中记录时，退化为 Gap Lock。范围查询则保持 Next-Key Lock，且会一路锁到第一个不满足条件的记录为止——这就是"明明只查几条却锁了一大片"的来源。

**意向锁 IS/IX**：表级锁，用来快速判断表里有没有行锁，避免逐行检查。**意向锁之间互不冲突**，它只和表级 S/X 锁冲突（这就是为什么 `LOCK TABLES` 会被行锁阻塞，而行锁之间 Isle 不冲突）。

```sql
-- 锁排查三张表（MySQL 8.0）
SELECT * FROM performance_schema.data_locks;       -- 当前持有与等待的锁
SELECT * FROM performance_schema.data_lock_waits;  -- 谁在等谁
SELECT * FROM information_schema.innodb_trx;       -- 事务列表、trx_started、trx_rows_locked
```

死锁：
- 检测默认开启 `innodb_deadlock_detect=on`，用 wait-for graph 检测环，回滚 undo 量小的事务
- 超时兜底 `innodb_lock_wait_timeout` 默认 50 秒，超时只是释放等待，不回滚整个事务
- **关闭死锁检测**（高并发热点更新场景）要配合 `innodb_lock_wait_timeout` 调小，否则会大量请求卡死；这是秒杀场景的经典 trade-off
- `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 段能看到最近一次死锁的两个事务 SQL

### 2.5 Buffer Pool 与 WAL

一条 UPDATE 的真实路径：
```
1. 数据页若在 buffer pool 直接改；不在则从磁盘加载
2. 修改前写 undo log（保证能回滚）
3. 修改 buffer pool 中的页 → 变成脏页
4. 写 redo log buffer
5. 提交时按策略刷 redo → 返回成功（此时脏页还在内存！）
6. 后台线程择机刷脏页到磁盘（checkpoint）
```

InnoDB 的 LRU 链表做了**冷热分离优化**（midpoint insertion）：新页先插入 old 区（默认 37% 处，`innodb_old_blocks_pct`），只有超过 `innodb_old_blocks_time`（默认 1000ms）后再次被访问才晋升到 young 区。这是为了防止一次全表扫描把热数据全部冲掉。

**change buffer**：更新非唯一二级索引时，若目标页不在内存，就把变更缓存起来（存在 ibdata / change buffer），等后续该页被读入再合并。**唯一索引用不了它**——因为唯一性校验必须读索引页判断冲突，这个"必须读"就是它俩性能差异的根源。

**double write buffer**：脏页刷盘时先顺序写入 double write 区（2MB 连续空间），成功后再刷真实位置。解决的是**部分写失效**（16KB 页只写了 4KB 就宕机）——此时 redo 也无能为力，因为它基于完整页做增量。

### 2.6 redo / binlog / 两阶段提交

| | redo log | binlog |
|---|---|---|
| 层次 | InnoDB 引擎层 | MySQL Server 层，所有引擎通用 |
| 内容 | 物理日志（"某页某偏移改成什么"） | 逻辑日志（SQL 语句或行前后镜像） |
| 写入 | 循环写，固定大小，覆盖 | 追加写，文件轮换 |
| 用途 | crash recovery | 主从复制、数据恢复、审计 |

为什么需要两个？因为 redo 是 InnoDB 私有的且循环覆盖，无法长期保留历史；binlog 是 Server 层统一的、可长期归档的逻辑日志。**两阶段提交（2PC）**保证二者一致：
```
prepare 阶段：redo 落盘，标记 PREPARE
写 binlog
commit 阶段：redo 标记 COMMIT
```
崩溃恢复时的判定规则：**binlog 完整则提交，否则回滚**。这就是为什么 `sync_binlog` 必须为 1 才能保证不丢。

```ini
innodb_flush_log_at_trx_commit = 1   # 1=每次提交刷盘(最安全) 2=每秒刷 0=每秒写且每秒刷
sync_binlog = 1                      # 1=每次提交刷盘，配合上面才是真正的双一配置
binlog_format = ROW                  # 强烈建议；STATEMENT 在主从上有 now()/uuid() 不等幂风险
```

### 2.7 复制与主从延迟

三个线程：`dump thread`（主）→ `IO thread`（从，写 relay log）→ `SQL thread`（从，回放）。5.7 后可用 MTS 并行回放 `slave_parallel_workers` + `slave_parallel_type=LOGICAL_CLOCK`（基于组提交的事务可并行）。

延迟的真实成因（不要只看 `Seconds_Behind_Master`，它在大事务执行期间是失真的）：

| 成因 | 现象 | 解法 |
|---|---|---|
| 大事务 | 一个 update 百万行，从库必须串行回放完 | 拆小批量、pt-archiver 归档 |
| 从库单线程回放 | 5.6 前或 parallel_type=DATABASE 的库间并行不生效 | 改 LOGICAL_CLOCK + 开并行 |
| 无主键表 | ROW 格式下每行回表定位，主键缺失导致全表扫 | 强制所有表有主键 |
| 从库读压力大 | 长查询占着资源 | 读写分离时区分次要查询 |
| 主库并发高 | relay log 堆积 | 分库或提升从库配置 |

半同步复制 `rpl_semi_sync_master_wait_point=AFTER_SYNC`（无损半同步，推荐）：主库等至少一个从库返回 ack 才提交。要注意它会退化——超时后自动切异步，`rpl_semi_sync_master_timeout` 默认 10 秒。

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 明明有索引却全表扫描 | 回表代价高 / 统计信息过期 / 隐式转换 | `EXPLAIN` + `ANALYZE TABLE` + 检查字段类型一致性 |
| 加了联合索引还是慢 | 顺序不对，没满足最左前缀；或范围列之后的字段无法用上 | 按 ESR（等值→排序→范围）重排；范围列放最后 |
| count(*) 越来越慢 | 二级索引 / MVCC 导致无法靠缓存计数 | 用计数表或 `SHOW TABLE STATUS` 估算值 |
| 分页 `LIMIT 1000000,20` 巨慢 | 要扫描并丢弃前 100 万行 | 延迟关联：`join (select id from t limit 1000000,20) b using(id)`；或记录上一页最大 id 做游标分页 |
| 死锁频繁但都是同一条 SQL 引发的 | 通常是唯一索引冲突场景下的插入意向锁等待 | 用 `insert ... on duplicate key update` 或先查后插加分布式锁；或对同主键操作做排序保证加锁顺序一致 |
| 主库 OOM / 内存涨到警戒线 | buffer pool 设太大 + 连接数多时每连接的 sort_buffer/join_buffer 累加 | 这些 buffer 是**每连接**分配的，`sort_buffer_size` 别乱调到几十 M |
| 事务长期不提交导致 undo 暴涨 | 长事务导致 purge 线程无法清理它的 undo；history list length 持续上涨 | 监控 `information_schema.innodb_trx` 的 trx_started，杀掉 or 拆分 |
| 批量插入很慢 | 每条 insert 一次 redo 刷盘 | 合并成多值 insert，或 `innodb_flush_log_at_trx_commit=2` 降低刷盘频率批量导入 |
| 从库数据和主库不一致 | STATEMENT 格式下的非确定性函数 | 强制 `binlog_format=ROW` |
| 表跑一段时间后变慢且文件不减 | delete 只标记删除，产生碎片与空洞 | `OPTIMIZE TABLE`（会锁表）或 pt-online-schema-change 重建 |
| Online DDL 卡死 | `ALGORITHM=COPY` 会重建整个表并阻塞 DML | 明确指定 `ALGORITHM=INSTANT`（8.0 支持秒加列）或 `INPLACE`；大表用 gh-ost / pt-osc |
| datetime 在 2038 年后溢出 | 用的是 timestamp（4 字节，范围到 2038-01-19） | 用 datetime（8 字节）或 bigint 存毫秒 |
| utf8mb4 排序导致 join 走不了索引 | 迁移后默认排序规则变更为 utf8mb4_0900_ai_ci | 统一字符集与 collation，尤其注意 `_bin` 与 `_ci` 混用的隐式转换 |

## 四、往下钻的入口

- **执行规划层**：`EXPLAIN FORMAT=JSON`（能看到 cost 估算明细）、`EXPLAIN ANALYZE`（8.0.18+，真实执行统计，这才是真相）
- **优化器决策**：`SET optimizer_trace='enabled=on'` 然后 `SELECT * FROM information_schema.OPTIMIZER_TRACE`，能看到每个访问路径的 cost 计算过程
- **官方典范参考**：MySQL Reference Manual → InnoDB Storage Engine 章节 → 特别是 *InnoDB Multi-Versioning*、*InnoDB Locking and Transaction Model*、*Optimization → Optimization and Indexes* 三节，这三节是原理密度最高的地方，其余大部分是用法手册
- **社区必读**：Percona Blog 的 schema 设计与内核 série；《高性能 MySQL》第 3 版第 4-7 章（第 4 版更新了 8.0 内容，可作为补充）
- **诊断工具**：`pt-query-digest` 分析慢日志、`gh-ost` 在线改表、`pt-online-schema-change`、`mysqlpump`
- **信息 schema 三张必看表**：`information_schema.innodb_trx`、`performance_schema.data_locks`、`sys.schema_unused_indexes`（找没人用的索引）

## 五、达标自检

1. 为什么推荐用自增整型做主键而不是 UUID？把它和"二级索引叶子存主键值"这条性质联系起来解释。
2. 一条 `update t set a=1 where id=1` 从调用到返回成功，磁盘上到底发生了什么变化？按顺序列出 redo/undo/binlog/buffer pool 的写入时机。
3. READ COMMITTED 和 REPEATABLE READ 在 binlog_format=STATEMENT 下有什么兼容性问题？（提示：RC + STATEMENT 会报错）
4. 二级索引的叶子节点存什么？回表一定发生吗？
5. `index(a,b,c)` 在 `where a=1 and c=3` 下用到了几个字段？`where a>1 and b=2` 呢？
6. 什么是 Gap Lock？它在哪个隔离级别下才存在？为什么 RC 下没有死锁却有幻读？
7. 一个 RR 事务里先 `select * from t`（快照读）再 `select * from t for update`（当前读），会不会看到别人新插入的行？为什么？
8. `innodb_flush_log_at_trx_commit=2` 在哪些宕机场景会丢数据？
9. 为什么 change buffer 只对非唯一二级索引生效？
10. 主键 id 已经有序了，为什么 explain 里还出现 `Using filesort`？（至少给出三种可能）
11. binlog 和 redo 都要写，只用其中一个行不行？
12. `LIMIT 100000,10` 慢在哪？延迟关联为什么能解决？它的极限是什么？
13. 主从延迟 30 秒，你怎么在 5 分钟内定位到原因？列出你打的每一条命令。
14. 什么情况下 RR 也阻止不了"看起来像幻读"的现象？
15. 线上要加索引，一张 5000 万行的表，你怎么做才能不影响业务？
16. 一条 SQL 时快时慢，可能有哪些原因？（提示：统计信息、MRR、change buffer 合并、脏页刷盘、自适应哈希冲突、锁等待都要想到）
17. 为什么有时候 `select count(*)` 走二级索引而不是主键？
18. 磁盘还剩 1TB，但写入报 No space left on device，MySQL 侧最可能的原因是什么？
