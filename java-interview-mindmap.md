# Java 后端面试冲刺 49 课时体系

> 这是思维导图的 Markdown 版。**XMind 打开方式**：文件 → 导入 → Markdown → 选本文件。
> 若已拿到 `java-interview-mindmap.xmind`，直接双击即可，无需导入。

## 一 优先级排序（16 份文档）

### S 级 必问核心 · 24 课时
#### ① java-core-foundation · 7 课时
#### ② spring-internals-learning-path · 7 课时
#### ③ mysql-core · 5 课时
#### ④ redis-core · 5 课时
#### 理由：占面试 60%+ 时间，答不上直接判定基础不牢

### A 级 高频分布式 · 13 课时
#### ⑤ dubbo-nacos-zookeeper · 4 课时
#### ⑥ message-queue · 3 课时
#### ⑦ linux-core · 3 课时
#### ⑧ network-protocols · 3 课时
#### 理由：CAP 选型与线上排查是中高级区分度最高的两块

### B 级 方向加分 · 9 课时
#### ⑨ design-patterns · 3 课时
#### ⑩ rabbitmq · 2 课时
#### ⑪ nginx · 2 课时
#### ⑫ mongodb · 1 课时
#### ⑬ object-storage · 1 课时

### C 级 工具机动 · 3 课时
#### ⑭ algorithm · 2 课时（建议挪到早晨或周末）
#### ⑮ git · 1 课时

## 二 每日作息 19:00-23:00

### 第 1 节 19:00-19:45 · 主干精读（最烧脑放这里）
### 休息 19:45-20:00 · 离开工位
### 第 2 节 20:00-20:45 · 动手验证（断点 / 命令行 / 手写）
### 休息 20:45-21:00
### 第 3 节 21:00-21:45 · 踩坑 + 往下钻入口
### 休息 21:45-22:00
### 第 4 节 22:00-22:45 · 达标自检（闭卷自答打标记）
### 收尾 22:45-23:00 · 答不出的抄进明日待补
### 底线：加班则只保第 1 节 + 第 4 节

## 三 49 课时排期 Day1-Day13

### Day 1 · JVM 与 GC
#### 1 运行时数据区 / 栈帧 / 逃逸分析 / 元空间
#### 2 GC Roots 六种 / 三色标记 SATB / 安全点
#### 3 收集器选型 Serial-Parallel-CMS-G1-ZGC
#### 4 自检：JVM+GC 全部题闭卷自答

### Day 2 · JUC 集合收尾 + Spring 开局
#### 5 JMM happens-before / volatile / 锁升级四态
#### 6 AQS / ReentrantLock / 线程池 7 参数 / ThreadLocal 泄漏
#### 7 HashMap 0.75 / CHM / JDK8-21 演进 / 虚拟线程 + Java 自检
#### 8 Spring refresh() 12 步逐个单步断点

### Day 3 · Spring IoC AOP 事务
#### 9 三级缓存循环依赖 / 为何构造器与 prototype 解不了
#### 10 Bean 生命周期 + BeanPostProcessor 全家桶
#### 11 AOP：JDK vs CGLIB 决策 + proceed() 责任链
#### 12 声明式事务 + 8 种失效场景根因

### Day 4 · Spring 收尾 + MySQL 索引
#### 13 MVC doDispatch() + Boot 自动装配 --debug 报告
#### 14 MyBatis 衔接点 + 手写 mini Spring + Spring 自检
#### 15 B+Tree 三优势 / 两千万行阈值 / 聚簇与二级索引
#### 16 索引失效 + 优化器 cost + EXPLAIN 实战

### Day 5 · MySQL 事务架构 + Redis 开局
#### 17 MVCC 版本链 / ReadView 时机 RC vs RR
#### 18 行锁三态 Record-Gap-Next-Key 与退化 / 死锁
#### 19 Buffer Pool + WAL + 两阶段提交 + 主从复制
#### 20 MySQL 自检 + Redis 数据结构与编码切换

### Day 6 · Redis 全量
#### 21 渐进式 rehash / 惰性+定期删除 / 8 种淘汰策略
#### 22 RDB fork COW THP / AOF 重写 / 混合持久化
#### 23 PSYNC2 / Sentinel / Cluster 16384 槽 / big-hot key
#### 24 缓存三兄弟 + 分布式锁 Redlock NPC + Redis 自检

### Day 7 · 消息队列 + ZooKeeper
#### 25 MQ 价值代价 + 不丢不重有序 + 幂等五方案
#### 26 堆积治理 / 死信 / 延迟消息 / RocketMQ 事务 / 选型树
#### 27 RabbitMQ：四种 Exchange / 可靠性三段论 / DLX / prefetch
#### 28 MQ 自检 + ZK：ZNode 四类型 / Watcher / ZAB

### Day 8 · Nacos Dubbo Linux
#### 29 Nacos ephemeral 双模 Distro vs JRaft / 长轮询
#### 30 Dubbo URL 总线 / SPI / Invoker 链 / 容错负载均衡
#### 31 Dubbo 3.x 应用级发现 / 线程模型 + 微服务自检
#### 32 Linux USE 方法 + CPU 内存 磁盘 网络四套命令

### Day 9 · Linux 收尾 + 网络协议
#### 33 五种 IO 模型正交维度 / epoll LT-ET / cgroup / THP OOM
#### 34 五套排障 SOP + Linux 自检
#### 35 TCP 三次握手 / 状态机 CLOSE_WAIT / TIME_WAIT / 半全队列
#### 36 超时重传 拥塞控制 / TLS1.2 vs 1.3 / HTTP1.1-2-3

### Day 10 · 网络收尾 + 设计模式
#### 37 WebSocket / DNS 与 JVM 缓存 / MTU / 抓包 + 网络自检
#### 38 创建型：单例容器级 vs ClassLoader 级 / 工厂 / 建造者
#### 39 结构型：代理 vs 装饰器意图差异 / 适配器（联动 AOP）
#### 40 行为型：模板方法 责任链 观察者 策略 + 自检

### Day 11 · Nginx Mongo 对象存储
#### 41 Nginx 进程模型 / location 优先级 / proxy_pass 斜杠
#### 42 upstream keepalive / 限流 / HTTPS / proxy_cache + 自检
#### 43 MongoDB 聚合体判断 / ESR 规则 / explain / 分片键
#### 44 对象存储 S3 语义 / STS 预签名 / Multipart / 前端直传

### Day 12 · Git 算法 跨文档串联
#### 45 Git 四类对象 / 三区四动词 / reset 三件套 / reflog + 自检
#### 46 算法模板① 双指针 滑动窗口 二分 回溯
#### 47 算法模板② DP 单调栈 图论 前缀和 + 边界 checklist
#### 48 串联：Redis锁 vs ZK锁 / MQ幂等 vs 接口幂等 / Spring事务 vs MySQL事务

### Day 13 · 模考
#### 49 抽 30 题闭卷自答 45 分钟 / 产出最终补漏清单

## 四 S 级核心考点（展开）

### Java 基础 · 7 课时
#### JVM 运行时数据区：栈帧四部分 逃逸分析三件套 直接内存 元空间
#### GC：GC Roots 六种 三色标记 SATB 安全点 G1/ZGC 选型
#### JMM：happens-before 八条 volatile 内存语义 锁升级四态
#### 并发：AQS state+CLH ReentrantLock 线程池 7 参数 ThreadLocal 泄漏
#### 集合：HashMap 扰动函数/扩容/树化/0.75 ConcurrentHashMap
#### 演进：JDK 8→21 虚拟线程
#### 排障：jstat jmap jstack jcmd arthas

### Spring 底层 · 7 课时
#### IoC：AbstractApplicationContext.refresh() 12 步
#### 循环依赖：为什么三级缓存不是两级（提前暴露代理对象）
#### 生命周期：实例化→填充→Aware→BPP-Before→init→BPP-After（AOP 在此）
#### 扩展点：Autowired / AOP / Async / Scheduled 都是 BeanPostProcessor
#### AOP：DefaultAopProxyFactory 决策（5.2+ 默认 CGLIB）
#### 事务：TransactionInterceptor + TransactionSynchronizationManager + 8 种失效
#### MVC+Boot：doDispatch() / AutoConfigurationImportSelector / Condition
#### MyBatis：SqlSessionFactoryBean / MapperProxy / SqlSessionTemplate

### MySQL · 5 课时
#### B+Tree：三优势 16KB 页 1170 分支 两千万行阈值
#### 索引失效：优化器 cost 回表代价 vs 全表扫
#### MVCC：Undo 版本链 ReadView 时机（RC 每次 / RR 首次）
#### 锁：Record Gap Next-Key 三态与退化 死锁
#### 架构：Buffer Pool WAL redo/undo/binlog 主从复制 在线 DDL

### Redis · 5 课时
#### 数据结构：redisObject SDS listpack quicklist skiplist+dict
#### 编码切换阈值：hash-max-listpack-entries=128 等
#### 内存：渐进式 rehash 惰性+定期删除 8 种淘汰策略
#### 持久化：RDB fork/COW/THP AOF 重写 7.0 混合
#### 集群：PSYNC2 Sentinel Cluster 16384 槽 big/hot key
#### 实战：穿透 击穿 雪崩 分布式锁 Redlock NPC 问题

## 五 A/B/C 级考点

### A 级 分布式与排查
#### ZK：ZNode 四类型 Watcher 一次性 ZAB 四字命令 分布式锁
#### Nacos：ephemeral 双模 Distro(AP) vs JRaft(CP) 长轮询 gRPC 端口
#### Dubbo：URL 总线 SPI @Adaptive Invoker 链 容错 3.x 应用级发现
#### MQ：三难题 幂等五方案 堆积治理 死信 延迟消息 事务消息 选型树
#### RabbitMQ：四种 Exchange confirm 只到 Exchange 的陷阱 prefetch DLX
#### Linux：USE 方法 五种 IO 模型 epoll LT/ET 五套排障 SOP
#### 网络：三次握手 CLOSE_WAIT TIME_WAIT 2MSL TLS1.3 HTTP2/3

### B 级 方向加分
#### 设计模式：20 种 框架源码位置 + 最小实现 + 易混区分点
#### Nginx：worker 模型 location 优先级 proxy_pass 斜杠 限流 热升级
#### MongoDB：聚合体判断 ESR 规则 totalDocsExamined 分片键三要素
#### 对象存储：S3 语义 无 rename/append STS 预签名 Multipart 前端直传

### C 级 工具
#### 算法：复杂度速估 双指针 滑动窗口 二分 回溯 DP 边界 checklist
#### Git：四类对象 三区四动词 reset 三件套 reflog merge vs rebase

## 六 使用方法与铁律

### 不要按顺序通读 —— 这是自检清单不是教材
### 流程：先翻达标自检 → 标记答不出 → 回必懂机制补 → 踩坑表反查现网
### 每节只追一个问题，追到具体那一行代码 / 那条命令为止
### 自检不许边看答案边答，答不出就是没掌握
### 学完一模块讲一遍 / 写短文，讲不清的地方就是没懂
### 周末把「记下来的结论」改写成「为什么必须这样」
### 弹性：压缩到 7 天 = 只做 S 级 24 课时 + Day13 模考
### 弹性：拉长到 4 周 = 每天只做第 1 节 + 第 4 节
</content>
