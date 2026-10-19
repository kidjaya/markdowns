# 技术栈核心知识清单 · 总览

> 面向 5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 目录：`D:\makrdowns`，共 15 份独立文档，6178 行

## 交付清单

| # | 文件 | 覆盖主干 |
|---|---|---|
| 1 | `mysql-core.md` | B+Tree 索引与失效场景、MVCC/ReadView、行锁三态与退化、Buffer Pool + WAL、主从复制与延迟、执行计划、表设计、读写分离与在线 DDL |
| 2 | `redis-core.md` | 底层数据结构与编码切换、渐进式 rehash、8 种淘汰策略、RDB/AOF 混合持久化、主从 PSYNC2、Sentinel、Cluster 16384 槽、big/hot key、缓存三兄弟、分布式锁 |
| 3 | `mongodb-core.md` | 文档模型与选型边界、ESR 索引规则、explain 诊断、聚合管道与 `$lookup` 陷阱、writeConcern/readConcern、副本集选举与 oplog、分片键三要素、WiredTiger |
| 4 | `nginx-core.md` | master/worker + epoll、location 匹配优先级、proxy_pass URI 替换、upstream 策略与 keepalive、限流、HTTPS/HTTP2/WebSocket、proxy_cache、热升级 |
| 5 | `network-protocols-core.md` | TCP 状态机与 TIME_WAIT、超时重传与拥塞控制、半/全连接队列、HTTP 1.1→2→3、TLS 1.2/1.3 握手、WebSocket、DNS、MTU/PMTU、抓包排障 |
| 6 | `linux-core.md` | CPU/内存/磁盘/网络排查命令体系、进程与 cgroup、THP/OOM、五种 IO 模型与 epoll LT/ET、inode 与权限、信号与 systemd、五套完整排障 SOP |
| 7 | `java-core-foundation.md` | 运行时数据区、三色标记与 SATB、安全点、GC 收集器全景与选型、类加载与双亲委派三破坏、JMM/happens-before、锁升级、AQS、ThreadLocal 泄漏、线程池、HashMap/CHM、JDK 8→21 演进、虚拟线程、排障工具箱 |
| 8 | `git-core.md` | 四类对象与快照模型、撤销三件套精确差异、reflog 救援、merge vs rebase、stash/clean、双点三点 diff、bisect/blame、CRLF 与 .gitattributes、submodule/subtree、filter-repo |
| 9 | `rabbitmq-core.md` | AMQP 模型、四种 Exchange、queue arguments、可靠性三段论、幂等五方案、prefetch 语义、DLX、延迟消息三种方案、quorum 队列、网络分区策略、Spring 客户端配置 |
| 10 | `object-storage-core.md` | 与块/文件系统的差异、术语对照、STS 与预签名、ETag 陷阱、Multipart Upload、生命周期与成本杀手、版本控制、事件通知、前端直传架构、SDK 连接池 |
| 11 | `message-queue-core.md` | MQ 价值与代价、不丢/不重/有序三难题、Pull vs Push、堆积治理、重试与死信、延迟消息、RocketMQ 事务消息、四大产品对比与选型决策树、消息设计规范 |
| 12 | `dubbo-nacos-zookeeper-core.md` | ZK 数据模型/Watcher/ZAB/四字命令、Nacos 服务模型/ephemeral 双模/gRPC 端口/配置长轮询、Dubbo URL 总线/SPI/调用链/容错/负载均衡/超时重试/线程模型/3.x 应用级发现 |
| 13 | `design-patterns-core.md` | 创建型/结构型/行为型共 20 种模式，每种给出框架中的具体源码位置、手写最小实现、与其他易混模式的区分点 |
| 14 | `algorithm-core.md` | 简单/中等难度的算法套路：识别特征 + Java 模板 + 分层题号清单 + 复杂度速查 + 边界 checklist |
| — | `spring-internals-learning-path.md` | Spring / Spring Boot 底层原理自学手册，六模块源码入口与断点清单、四阶段路线 |

**篇幅速览**：共 6178 行。

| 文件 | 行数 |
|---|---|
| `dubbo-nacos-zookeeper-core.md` | 657 |
| `design-patterns-core.md` | 942 |
| `algorithm-core.md` | 616 |
| `rabbitmq-core.md` | 444 |
| `message-queue-core.md` | 433 |
| `git-core.md` | 419 |
| `java-core-foundation.md` | 417 |
| `object-storage-core.md` | 392 |
| `linux-core.md` | 321 |
| `network-protocols-core.md` | 303 |
| `nginx-core.md` | 296 |
| `redis-core.md` | 253 |
| `mysql-core.md` | 232 |
| `mongodb-core.md` | 207 |
| `spring-internals-learning-path.md` | 181 |
| `overview.md` | 63 |

## 统一文档结构

每份文档严格遵循同一模板，便于对照使用：

```
一、核心心智模型    3-5 条，讲清"本质上是什么、解决了什么矛盾"
二、必懂机制        覆盖全部主干，每条落到具体名字（参数名/类名/命令/配置）
三、高频踩坑        | 现象 | 根因 | 解法 | 表格，根因挖到实现层面
四、往下钻的入口    源码类/方法名、配置路径、排障命令、官方文档具体章节
五、达标自检        12-24 个问题，只列问题不给答案
```

## 建议的使用方式

**不要按顺序通读。** 这是自检清单，不是教材：

1. 挑一份，直接翻到「五、达标自检」
2. 逐题自答，答不上来的做标记
3. 回到「二、必懂机制」补对应条目
4. 「三、高频踩坑」用来对照现网问题，反查自己系统有没有同类隐患

**优先级建议**（如果时间有限）：
- 第一梯队：`java-core-foundation.md` → `mysql-core.md` → `redis-core.md`（面试与日常最密集）
- 第二梯队：`message-queue-core.md` → `rabbitmq-core.md` → `linux-core.md`（线上排查最常用）
- 第三梯队：其余按当前工作涉及的技术挑

## 关键决策说明

- **风格取向**：所有条目强制落到具体名词（参数名、类名、方法名、命令、文件路径），刻意避免"需要注意性能""要合理配置"这类没有落点的表述
- **版本号明确标注**：JDK 8/11/17/21 的差异、Spring Cloud Alibaba 版本矩阵、RabbitMQ 3.8+ 的 quorum 队列、Dubbo 2.x/3.x 差异均已标明
- **踩坑表的根因深度**：要求挖到实现层面（如"fork 时 THP 导致 copy-on-write 放大"而非"配置不当"）
- **自检题不给答案**：保证它作为检查工具的有效性，而非阅读材料

## 已知限制

- `mongodb-core.md` 篇幅最短，分片与聚合部分相对其他文档略薄，后续可按需要补充
- 部分内容为业界主流最佳实践总结，具体参数值会随版本演进，落地前请以对应版本的官方文档复核
- 各文档独立成篇，跨文档的关联（如"Redis 分布式锁 vs ZK 分布式锁"）未做交叉引用
