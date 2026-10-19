# 面试冲刺 · 优先级排序与 49 课时表

> 面向：5 年经验 Java 后端，目标是面试 + 补底层原理
> 素材：`D:\makrdowns` 下 16 份技术文档
> 时间窗：**每晚 19:00–23:00**，45 分钟课时 + 15 分钟休息，每天 4 节
> 总量：**49 课时 = 13 天**（以 2026-10-19 为 Day 1，可顺延）

---

## 一、面试优先级排序

排序依据：**面试考察密度 × 你的短板（Spring 底层）**。不是按文档篇幅排。

### S 级 · 必问核心（24 课时，占面试 60%+ 时间）

| # | 文档 | 课时 | 为什么排这个位置 |
|---|---|---|---|
| 1 | `java-core-foundation.md` | 7 | JVM / JUC / 集合每场必问，答不上直接判定"基础不牢"，没有回旋余地 |
| 2 | `spring-internals-learning-path.md` | 7 | 你的核心诉求。IoC / AOP / 事务 / MVC / 自动装配是 5 年经验的分水岭，也是简历深挖的入口 |
| 3 | `mysql-core.md` | 5 | 索引 + 事务 + 锁，后端三大件，几乎场场问到，且追问很深 |
| 4 | `redis-core.md` | 5 | 缓存三兄弟 + 分布式锁是高频场景题，容易从"用过"被问穿到"为什么" |

### A 级 · 高频分布式（13 课时）

| # | 文档 | 课时 | 为什么排这个位置 |
|---|---|---|---|
| 5 | `dubbo-nacos-zookeeper-core.md` | 4 | 微服务方向必问。CAP 选型、注册中心对比是经典追问链，能拉开档次 |
| 6 | `message-queue-core.md` | 3 | 不丢 / 不重 / 有序 + 幂等五方案，中间件必考，且常与业务场景结合 |
| 7 | `linux-core.md` | 3 | 线上排查题。中高级面试区分度最高的一块，"你怎么定位 CPU 飙高"是标准开场 |
| 8 | `network-protocols-core.md` | 3 | TCP 状态机 / TLS 握手 / HTTP 演进，考察底层功底，常被用来判断深度 |

### B 级 · 方向加分（9 课时）

| # | 文档 | 课时 | 为什么排这个位置 |
|---|---|---|---|
| 9 | `design-patterns-core.md` | 3 | 结合 Spring 源码回答是加分项；与 S 级联动，紧跟着 Spring 学性价比最高 |
| 10 | `rabbitmq-core.md` | 2 | JD 明确提 RabbitMQ 时上升为必考；否则只需能讲清选型理由 |
| 11 | `nginx-core.md` | 2 | 网关 / 反向代理，运维向或偏架构的岗位常问 |
| 12 | `mongodb-core.md` | 1 | 主要考"选型判断"——什么时候不该用 MySQL |
| 13 | `object-storage-core.md` | 1 | 有文件上传 / 归档业务才会问，低频 |

### C 级 · 工具与机动（3 课时）

| # | 文档 | 课时 | 为什么排这个位置 |
|---|---|---|---|
| 14 | `algorithm-core.md` | 2 | 建议挪到早晨或周末，不与晚间主时段抢时间。这里只背模板 |
| 15 | `git-core.md` | 1 | 工具熟练度，低频但容易翻车（撤销三件套、reflog） |

**合计 49 课时。**

---

## 二、每日作息模板（19:00–23:00）

| 时段 | 时长 | 干什么 |
|---|---|---|
| **第 1 节** 19:00–19:45 | 45 min | **主干精读**。最烧脑的放这里，读「二、必懂机制」 |
| 休息 19:45–20:00 | 15 min | 离开工位，别刷手机（刷了就回不来） |
| **第 2 节** 20:00–20:45 | 45 min | **动手验证**。源码下断点 / 命令行实操 / 手写最小实现 |
| 休息 20:45–21:00 | 15 min | |
| **第 3 节** 21:00–21:45 | 45 min | **踩坑 + 钻入口**。看「三、高频踩坑」「四、往下钻的入口」 |
| 休息 21:45–22:00 | 15 min | |
| **第 4 节** 22:00–22:45 | 45 min | **达标自检**。翻到「五、达标自检」闭卷自答，答不出的打标记 |
| 收尾 22:45–23:00 | 15 min | 把答不出的题抄进「明日待补」，第二天第 1 节先补 |

**三条硬规矩：**
1. 每节只追一个问题，追到具体那一行代码 / 那一条命令为止，不泛读
2. 自检题**不许边看答案边答**，答不出就是没掌握，标记后次日补
3. 加班到 21 点后，只保第 1 节 + 第 4 节（精读 + 自检），中间两节顺延

---

## 三、49 课时排期表

### Day 1 — JVM 与 GC

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 1 | JVM 运行时数据区（§2.1）：栈帧四部分、逃逸分析三件套、直接内存、元空间替代永久代 | 跑 `jcmd <pid> VM.native_memory summary` |
| 2 | GC 基础（§2.2）：GC Roots 六种、四种引用、三色标记与 SATB、安全点 | 画出三色标记过程 |
| 3 | GC 收集器全景与选型：Serial/Parallel/CMS/G1/ZGC 适用场景与关键参数 | 写出你项目的 JVM 参数并说明每项理由 |
| 4 | **Java①自检**：JVM + GC 全部题闭卷自答，标记答不出的 | 产出「明日待补」清单 |

### Day 2 — JUC / 集合 / 收尾 + Spring 开局

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 5 | JMM 与 volatile：happens-before 八条、volatile 的内存语义、锁升级四态 | 解释 DCL 单例为什么必须 volatile |
| 6 | 并发工具：AQS（state + CLH 队列）、ReentrantLock 公平/非公平、线程池 7 参数与拒绝策略、ThreadLocal 泄漏根因 | 手写一个简易 AQS 互斥锁 |
| 7 | 集合：HashMap（扰动函数、扩容、树化阈值、0.75 的由来）、ConcurrentHashMap、JDK 8→21 演进、虚拟线程 | **Java 整份自检 + 补漏** |
| 8 | Spring IoC 主线：`AbstractApplicationContext.refresh()` 12 步逐个单步 | 起一个最小工程（只含 spring-context），下断点走完 12 步 |

### Day 3 — Spring IoC / AOP / 事务

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 9 | 循环依赖与三级缓存：为什么三级不是两级、构造器与 prototype 为何解不了 | 断点 `DefaultSingletonBeanRegistry.getSingleton()` |
| 10 | Bean 生命周期与扩展点：完整顺序 + BeanPostProcessor 全家桶（Autowired / AOP / Async / Scheduled） | 列出生殖顺序并说明 AOP 代理在哪一步生成 |
| 11 | AOP：`DefaultAopProxyFactory.createAopProxy()` 的 JDK/CGLIB 决策、`ReflectiveMethodInvocation.proceed()` 责任链 | 断点验证 Spring 5.2+ 默认走 CGLIB |
| 12 | 声明式事务：`TransactionInterceptor`、`TransactionSynchronizationManager`、传播行为 | **逐个解释 8 种失效场景根因** |

### Day 4 — Spring 收尾 + MySQL 索引

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 13 | MVC 链路 + Boot 自动装配：`doDispatch()`、`AutoConfigurationImportSelector`、`OnClassCondition` | `--debug` 启动读 ConditionEvaluationReport |
| 14 | MyBatis 衔接点：`SqlSessionFactoryBean` / `MapperProxy` / `SqlSessionTemplate`；**手写 mini Spring 收尾** | **Spring 整份自检 + 补漏** |
| 15 | MySQL 索引结构：B+Tree 三个优势、两千万行阈值怎么算出来的、聚簇与二级索引的耦合 | 用 `innodb_index_stats` 查自己表的索引页数 |
| 16 | 索引失效场景 + 优化器 cost 估算 + 执行计划解读 | 拿一条线上慢 SQL 跑 `EXPLAIN` 并对着改 |

### Day 5 — MySQL 事务与架构 + Redis 开局

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 17 | MVCC 与隔离级别：Undo Log 版本链、ReadView 生成时机（RC vs RR）、快照读 vs 当前读 | 画 RR 下 ReadView 的时序图 |
| 18 | 锁：Record / Gap / Next-Key 三态与退化条件、死锁成因与排查 | 复现一个间隙锁导致的死锁 |
| 19 | Buffer Pool + WAL：change buffer、redo/undo/binlog 两阶段提交、主从复制与延迟、读写分离、在线 DDL | 说明 binlog 与 redo 的两阶段提交 |
| 20 | **MySQL 自检**；Redis 数据结构与编码切换：redisObject、SDS、listpack/quicklist、skiplist+dict | `OBJECT ENCODING` 逐个类型验证切换阈值 |

### Day 6 — Redis 全量

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 21 | 渐进式 rehash、惰性+定期删除、8 种淘汰策略（allkeys-lru vs volatile-ttl 的差异） | 说明"过期了内存没降"的根因 |
| 22 | 持久化：RDB fork / COW / THP、AOF 重写、7.0 混合持久化 | 解释为什么官方要求关闭 THP |
| 23 | 主从 PSYNC2、Sentinel 判定流程、Cluster 16384 槽、big key / hot key 治理 | 用 `redis-cli --bigkeys` 扫一遍 |
| 24 | 缓存穿透/击穿/雪崩 + 分布式锁（SET NX PX、Redlock、NPC 问题）+ **Redis 自检** | 写出你项目的缓存更新策略并挑自己的毛病 |

### Day 7 — 消息队列 + ZooKeeper

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 25 | MQ 价值与代价、不丢/不重/有序三难题、幂等五方案 | 说明为什么 exactly-once 跨系统做不到 |
| 26 | 堆积治理、重试与死信、延迟消息三种方案、RocketMQ 事务消息、四大产品选型决策树 | 给你项目选一个 MQ 并写理由 |
| 27 | RabbitMQ：AMQP 模型、四种 Exchange、可靠性三段论（confirm 只到 Exchange 的陷阱）、prefetch 语义、DLX | 说明 mandatory 与 Alternate Exchange 的取舍 |
| 28 | **MQ 自检**；ZooKeeper：ZNode 四类型、Watcher 一次性的坑、ZAB、四字命令 | 用临时顺序节点写一个分布式锁 |

### Day 8 — Nacos / Dubbo / Linux

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 29 | Nacos：`ephemeral` 双模（Distro vs JRaft）、配置长轮询、gRPC 端口、与 Eureka 的 CAP 差异 | 说明网络抖动时 AP 与 CP 实例的行为差异 |
| 30 | Dubbo：URL 总线、SPI（`@SPI` / `@Adaptive`）、调用链 Invoker、容错与负载均衡策略 | 断点看一次 Invoker 链的组装 |
| 31 | Dubbo 3.x 应用级发现、线程模型、超时重试；**微服务自检** | 说清 2.x 与 3.x 注册数据量的量级差 |
| 32 | Linux 排查体系：USE 方法、CPU/内存/磁盘/网络四套指标与命令 | 对本机跑一遍 `vmstat 1` / `iostat -x 1` / `sar -n DEV 1` |

### Day 9 — Linux 收尾 + 网络协议

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 33 | 五种 IO 模型的正交维度、epoll LT/ET 差异、进程与 cgroup、THP 与 OOM | 说清"同步非阻塞"到底同步在哪 |
| 34 | 五套排障 SOP 完整走一遍；**Linux 自检** | 写一份你自己用的排查 checklist |
| 35 | TCP：三次握手为什么三次、状态机（CLOSE_WAIT 的诊断价值）、TIME_WAIT 2MSL 的两个理由、半/全连接队列 | `ss -s` 看本机连接状态分布 |
| 36 | 超时重传与拥塞控制、TLS 1.2 vs 1.3（ECDHE 前向安全）、HTTP 1.1→2→3 的队头阻塞主线 | 用 `curl -v` 看一次 TLS 握手 |

### Day 10 — 网络收尾 + 设计模式

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 37 | WebSocket、DNS 递归/迭代与 JVM DNS 缓存、MTU/PMTU、抓包排障；**网络自检** | wireshark/tcpdump 抓一次握手 |
| 38 | 创建型：单例（Spring 容器级 vs GOF ClassLoader 级）、工厂、建造者 —— 对照 Spring 源码位置 | 写出 5 种单例写法各自的坑 |
| 39 | 结构型：代理 vs 装饰器的意图差异、适配器、门面 —— 与 Spring AOP 联动 | 说明 Spring AOP 用的是哪种代理模式 |
| 40 | 行为型：模板方法、责任链（对应 `ReflectiveMethodInvocation`）、观察者、策略；**设计模式自检** | 说出 MVC 里前端控制器/适配器/策略分别对应谁 |

### Day 11 — Nginx / MongoDB / 对象存储

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 41 | Nginx 进程模型、`location` 匹配优先级、`proxy_pass` 带不带斜杠的 URI 替换、header 透传 | 写一个 location 配置并预判匹配结果 |
| 42 | upstream 策略与 keepalive、限流、HTTPS/HTTP2/WebSocket、proxy_cache、热升级；**Nginx 自检** | 说明 `worker_connections` 与 `ulimit -n` 的关系 |
| 43 | MongoDB：文档模型与聚合体判断、ESR 索引规则、`explain` 的 `totalDocsExamined/nReturned`、副本集与分片键三要素 | 对比同一需求在 MySQL 与 Mongo 的建模差异 |
| 44 | 对象存储：S3 语义（无 rename/append/目录）、STS 与预签名、Multipart Upload、生命周期成本、前端直传架构 | 画一张前端直传的时序图 |

### Day 12 — Git / 算法 / 跨文档串联

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 45 | Git：四类对象、三区四动词、reset 三件套精确差异、reflog 救援、merge vs rebase；**Git 自检** | `git reflog` 救回一次误操作 |
| 46 | 算法模板①：双指针、滑动窗口、二分、回溯 —— 背模板 + 识别特征 | 每类默写 1 遍模板 |
| 47 | 算法模板②：DP、单调栈/队列、图论、前缀和 —— 背模板 + 边界 checklist | 默写边界 checklist |
| 48 | **跨文档串联**（文档没写，自己补）：Redis 锁 vs ZK 锁 / MQ 幂等 vs 接口幂等 / Spring 事务 vs MySQL 事务 / 缓存一致性 vs 主从延迟 | 产出一张对比表 |

### Day 13 — 模考

| 节 | 内容 | 动手 / 产出 |
|---|---|---|
| 49 | **全真模考**：从 16 份文档的「五、达标自检」随机抽 30 题，闭卷自答，计时 45 分钟 | 答不出的拉成最终补漏清单，进入面试前最后一天 |

---

## 四、弹性方案

| 情况 | 怎么砍 |
|---|---|
| 加班到 21:00 后才到家 | 只保第 1 节（19:00 改到到家后）+ 第 4 节自检；中间两节顺延到周末 |
| 周末 | 上午加 2 节（09:00–09:45 / 10:00–10:45）用来追进度，晚上按常 |
| 想压缩到 7 天 | 只做 S 级（Day 1–6，24 课时）+ Day 13 模考，A/B/C 级改为面试前速览 |
| 想拉长到 4 周 | 每天只做 2 节（第 1 节精读 + 第 4 节自检），中间两节改为隔天交替 |

**优先级底线**：时间不够时，保 `java-core-foundation` → `spring-internals-learning-path` → `mysql-core` → `redis-core`，这四份拿下就有 60% 的面试覆盖率。

---

## 五、使用说明

1. **不要按顺序通读。** 这些文档是自检清单，不是教材
2. 每节课先翻「五、达标自检」，答不上来的做标记，再回「二、必懂机制」补对应条目
3. 「三、高频踩坑」用来对照现网问题，反查自己系统有没有同类隐患
4. 每学完一个模块，写一篇短文或讲一遍 —— 讲不清楚的地方就是还没懂的地方
5. 周末回头看笔记，把"记下来的结论"改写成"为什么必须这样"

---

## 配套文件

- `java-interview-mindmap.xmind` —— 同目录，XMind 直接打开，含优先级梯队 + 课时表 + 核心考点三层结构
</content>
