# Dubbo / Nacos / Zookeeper · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方

## 一、核心心智模型

1. **注册中心的本质是一个"带健康检测的分布式配置中心"，存的是"谁在哪"这张表。** 三者（ZK / Nacos / Eureka）的全部差异，归结到一句话：**这张表在 CAP 里选择牺牲哪一边**。ZK 选 CP（分区时宁可不可用也不给错数据），Eureka 选 AP（宁可给过期数据也要可用），Nacos 两个都支持、按实例类型切换。
2. **Zookeeper 的价值不在存储，在"临时节点 + Watcher"这个原语组合。** 临时节点让"服务下线"变成自动的（session 断开即删除），Watcher 让"变更感知"变成推送的。ZK 做注册中心是这两个原语的自然结果，但它本意并不是为注册中心设计的——这也是它在大规模服务发现场景下被 Nacos 替代的原因。
3. **Nacos 的关键分水岭是 `ephemeral` 这个布尔值。** `true` 走 Distro（AP，客户端心跳）、`false` 走 JRaft（CP，服务端探测）。网络抖动时两者行为完全相反。不知道这个参数，就等于没用过 Nacos。
4. **Dubbo 的灵魂是 URL。** 所有配置（协议、地址、超时、重试、序列化、版本、分组）都被编码进一个 URL 字符串，在 Invoker 链路上层层传递。**理解 Dubbo = 理解 URL 怎么在 Invoker 链路上流转和被改写。**
5. **Dubbo 3.x 最大的变化是服务发现粒度从"接口级"降到"应用级"。** 注册数据量从 O(接口数 × 实例数) 降到 O(实例数)，这是解决大规模集群注册中心压力的根本手段。不知道这个，就分不清 2.x 和 3.x。

## 二、必懂机制（一）：Zookeeper

### 2.1 数据模型

树形结构，节点叫 **ZNode**，用路径标识（`/services/order/provider1`）。

**四种节点类型**：
| 类型 | 说明 |
|---|---|
| PERSISTENT | 持久节点，显式删除才消失 |
| PERSISTENT_SEQUENTIAL | 持久顺序节点，自动追加单调递增序号（`-0000000001`） |
| EPHEMERAL | 临时节点，**创建它的 session 断开即自动删除**；不能有子节点 |
| EPHEMERAL_SEQUENTIAL | 临时顺序节点（分布式锁的基础） |

**Stat 结构字段**（`get /path` 能看到）：
```
czxid      创建该节点的事务 ID
mzxid      最后修改该节点的事务 ID
pzxid      最后修改子节点列表的事务 ID
ctime/mtime 创建/修改时间（毫秒）
version     数据版本号（每次 setData 递增）
cversion    子节点版本号
aversion    ACL 版本号
ephemeralOwner  临时节点所属 session ID（持久节点为 0）
dataLength  数据长度
numChildren 子节点数
```

**版本号用于乐观锁 CAS**：`setData(path, data, version)` 传入 expected version，不匹配则抛 `BadVersionException`。这是实现"检查再设置"原子性的唯一手段。

**ZXID**：64 位全局递增事务 ID，高 32 位是 epoch（leader 任期），低 32 位是计数器。Leader 变更时 epoch 递增，用于识别"过期 Leader"发出的提议。

### 2.2 Watcher 机制

三类注册点与事件：
| 注册方式 | 监听内容 | 触发事件 |
|---|---|---|
| `getData(path, watch)` | 节点数据 | NodeDataChanged / NodeDeleted |
| `exists(path, watch)` | 节点是否存在 | NodeCreated / NodeDeleted / NodeDataChanged |
| `getChildren(path, watch)` | 子节点列表 | NodeChildrenChanged / NodeDeleted |

**三个必须知道的特性**：
1. **一次性（one-shot）**：Watcher 触发一次后即失效，必须重新注册。这是 ZK 最难受的设计，客户端要自己在回调里重新 watch
2. **通知不含变更内容**：只告诉你"变了"，不告诉你"变成什么"。收到通知后要自己去 get 最新数据
3. **异步无序**：通知可能乱序、可能丢失（在重新注册成功之前的变更不会被感知）

**这个"一次性 + 无内容"的组合导致的问题**：在收到通知到重新注册之间的窗口期内，如果节点又变了，你会漏掉这次变更。**生产上要么用 Curator 的 `NodeCache` / `PathChildrenCache`（自动重注册），要么收到通知后强制拉取一次全量做兜底。**

### 2.3 Session

- 客户端与 ZK 建立 TCP 长连接即创建 session，靠**心跳（ping）**维系
- **sessionTimeout 协商**：客户端提议的值会被服务端强制落在 `[2 × tickTime, 20 × tickTime]` 区间内。所以设 1000ms 在 tickTime=2000 的配置下会被改成 4000ms
- **临时节点随 session 过期而删除**。session 过期由**服务端**判定（客户端以为自己还活着，服务端已判死）
- **连接断开 ≠ session 过期**：断连后只要在 sessionTimeout 内重连（且连到有相同 session 数据的节点），临时节点还在
- **Curator 的重试策略**：`ExponentialBackoffRetry` / `RetryNTimes` / `RetryOneTime`。网络抖动时 Curator 会自动重试命令，但**重试期间会话可能已过期** → 抛出 `SessionExpiredException`，此时必须重建 `CuratorFramework` 客户端
- **窗口期问题**：客户端 session 过期后，临时节点要等服务端清理完成才真正消失。这个窗口内注册中心里可能残留"已死"的实例

### 2.4 ZAB 协议

两种模式：

**崩溃恢复模式**（选主 + 数据同步）：
1. Leader 选举：节点进入 LOOKING 状态，互相投票（先投自己，比较 `(zxid, myid)`，zxid 大者优先，zxid 相同则 myid 大者胜）
2. 获得**过半（quorum）**票数的节点成为 Leader
3. 数据同步：Leader 与 Follower 对齐，Follower 补齐缺失或回滚多余的事务

**消息广播模式**（类似二阶段提交）：
1. Leader 收到写请求 → 生成事务提议（带 zxid）→ 发给所有 Follower
2. Follower 写本地事务日志 → 返回 ACK
3. Leader 收到**过半 ACK** 即提交（不需要等所有人）→ 广播 COMMIT
4. Follower 收到 COMMIT 应用变更

**角色**：
| 角色 | 职责 | 参与投票 |
|---|---|---|
| Leader | 处理所有写请求，发起提议 | 是 |
| Follower | 处理读请求，参与写投票和选举 | 是 |
| Observer | 处理读请求，**不参与投票** | 否（用于提升读扩展性，不影响 quorum） |

**为什么 ZK 是 CP**：**选举期间（LOOKING 状态）整个集群不可写、也可能不可读**。这个特性放到注册中心上的后果是——网络分区时，整套服务发现可能整体不可用，而此时服务本身其实是健康的。这就是 Eureka/Nacos AP 方案存在的理由。

**节点数与容灾**：
- quorum = `n/2 + 1`
- 3 节点容忍 1 台宕机，4 节点也只容忍 1 台（4 的 quorum 是 3）→ **所以 4 台和 3 台容灾能力相同，部署永远是奇数台**
- 5 节点容忍 2 台

### 2.5 存储与运维

**两类文件必须分盘**：
```
dataDir=/data/zookeeper/data        # 内存数据库快照 snapshot
dataLogDir=/data/zookeeper/datalog  # 事务日志（顺序写，IO 密集）
```
分到不同磁盘能显著提升性能（事务日志的顺序写不会被快照的随机写打断）。

**自动清理（不配会撑爆磁盘，经典事故）**：
```
autopurge.purgeInterval=24      # 每 24 小时清理一次（默认 0 = 关闭！）
autopurge.snapRetainCount=10    # 保留最近 10 个快照
```
也可以手动 `zkCleanup.sh -n 10`。**配置为 0 意味着永不清理，磁盘必然写满。**

**四字命令**（`nc localhost 2181 <cmd>` 或 `telnet`）：
| 命令 | 含义 |
|---|---|
| `conf` | 打印服务配置 |
| `cons` | 列出所有客户端连接详情 |
| `crst` | 重置连接统计 |
| `dump` | 列出 session 和临时节点 |
| `envi` | 环境信息 |
| `ruok` | 健康检查，返回 `imok` |
| `srst` | 重置服务器统计 |
| `srvr` | 服务器完整信息 |
| `stat` | 简略状态 + 客户端列表 |
| `wchs` | Watcher 概要 |
| `wchc` | 按连接列出 Watcher |
| `mntr` | **监控用**，输出集群健康、延迟、节点数等（Zabbix/Prometheus 采集这个） |

注意：3.5+ 版本默认只允许 `srvr` 等白名单命令，需要在配置里加 `4lw.commands.whitelist=*`。

### 2.6 典型应用

**分布式锁（临时顺序节点方案）**：
```
1. 在 /lock 下创建临时顺序节点 /lock/lock-000000001
2. getChildren("/lock") 拿到所有子节点
3. 如果自己是序号最小的 → 获得锁
4. 否则对"序号刚好比自己小的前一个节点"注册 Watcher，等待
5. 前一个节点被删除（锁释放或持有者宕机）→ 收到通知 → 回到第 2 步
```
**为什么监听"前一个"而不是"父节点"**：避免**羊群效应**——父节点变更会唤醒所有等待者，而其实只有一个能拿到锁。

**ZK 分布式锁的固有缺陷**：
1. **session 超时导致误释放**：持有者因 GC 停顿/网络抖动导致心跳失败 → session 过期 → 临时节点被删 → 锁被别人拿到 → **两个持有者同时操作共享资源**
2. **没有 fencing token**：拿到锁的客户端无法识别"我的锁已经过期了"

**所以生产建议**：ZK 锁必须配合**业务侧的 fencing token（单调递增版本号）**，资源侧校验 token 单调性才能彻底安全。或者直接用 Redisson（带看门狗 + 业务超时评估）。

**其他典型应用**：Leader 选举（最小序号者为主）、配置中心（**不推荐**，Watcher 一次性 + 无历史版本 + 无灰度，远不如 Nacos/Apollo）、命名服务、集群成员管理。

## 二、必懂机制（二）：Nacos

### 3.1 定位

服务发现 + 配置管理二合一。

| 产品 | 服务发现 | 配置管理 | 一致性 |
|---|---|---|---|
| **Nacos** | ✅ | ✅ | **双模（AP/CP 可切）** |
| Eureka | ✅ | ❌ | AP |
| Consul | ✅ | ✅ | CP |
| Apollo / Spring Cloud Config | ❌ | ✅ | — |
| Zookeeper | 可用但有包袱 | 可用但不推荐 | CP |

### 3.2 服务模型

三层结构：**Service → Cluster → Instance**

- **Service**：服务名（如 `order-service`）
- **Cluster**：集群（如 `BJ` / `SH`，用于同城多机房优先调用）
- **Instance**：实例（IP + Port + weight + healthy + enabled + metadata + ephemeral + clusterName）

**三级隔离**：`namespace` → `group` → `serviceName`
- **namespace**：环境隔离（dev / test / prod），**最常用**，默认 `public`
- **group**：组，默认 `DEFAULT_GROUP`
- **注意：一个服务由 `namespace + group + serviceName` 三者共同定位**，只带 serviceName 不唯一

### 3.3 健康检查与一致性协议（核心）

**这是 Nacos 最重要的知识点。**

| | 临时实例 `ephemeral=true`（默认） | 持久实例 `ephemeral=false` |
|---|---|---|
| 一致性协议 | **Distro（AP）** | **JRaft（CP）** |
| 健康检测方式 | **客户端主动上报心跳** | **服务端主动探测**（TCP / HTTP / MySQL） |
| 不健康时 | 超过 15s 标记为不健康，超过 30s 自动剔除 | 探测失败标记为不健康，**不会自动剔除** |
| 网络抖动时 | 客户端心跳断 → 被剔除（可能误删健康实例） | 服务端探测失败 → 只是标记，不会删 |
| 适用 | 业务服务（Spring Boot 应用） | 基础设施（数据库、Redis、第三方服务） |
| 注册方式 | Nacos Client SDK | 需要手动注册或用 API |

**健康保护阈值 `ProtectThreshold`**（默认 0.75）：
```
当健康实例数 / 总实例数 < 阈值 时，
即使实例被判定为不健康，也仍然返回给消费者（不全部摘除）
```
**为什么需要**：防止雪崩——如果大量实例因网络抖动同时被判定不健康而全部剔除，剩下的少数健康实例会被全部流量打垮，导致彻底雪崩。保护阈值宁可返回部分不健康实例，也不能让流量全部压到少数实例上。

### 3.4 服务发现的推拉机制

**1.x（UDP + HTTP 长轮询）**：
- 客户端启动时拉一次全量，之后发起 30 秒长轮询
- 服务端有变更时通过 **UDP** 推送（UDP 不可靠，只是加速感知）
- 本地磁盘缓存（fail-fast 保护：服务端全挂时仍能启动）

**2.x（gRPC 长连接）—— 大变化**：
- 改用 gRPC 双向长连接推送
- **端口偏移规则**：
  | 端口 | 用途 | 计算 |
  |---|---|---|
  | 8848 | 主端口（HTTP API / 控制台） | 配置 `server.port` |
  | **9848** | gRPC 客户端端口 | 主端口 + 1000 |
  | **9849** | gRPC 服务端端口（集群间同步） | 主端口 + 1001 |
  | 7848 | JRaft 端口 | 主端口 - 1000 |
- **必须放行的端口**：8848、9848、9849、7848。**最常见的生产问题就是只放了 8848，导致"注册成功但订阅不到"**（客户端能注册，但收不到推送）

**2.x 连接数问题**：gRPC 连接是**按服务粒度**建立的（不是按实例），服务数 × 客户端数 = 连接数。大规模集群下连接数会暴涨，需要调整 `nacos.remote.server.grpc.executor.thread.count` 等参数。

**客户端本地缓存**：`ServiceInfoHolder` 把服务信息缓存到 `~/nacos/naming/<namespace>` 目录。服务端不可用时用缓存启动。

### 3.5 配置管理

**定位三要素**：`dataId` + `group` + `namespace`

**长轮询机制**（1.x 经典实现，2.x 改为长连接但思想相同）：
```
ClientWorker.LongPollingRunnable
1. 客户端发起 HTTP 请求到 /nacos/v1/cs/configs/listener
2. 携带本地所有配置的 MD5
3. 服务端比对 MD5，无变更则挂起请求（默认 30 秒超时）
4. 配置变更时服务端触发 LocalDataChangeEvent，立即返回变更的配置 key
5. 客户端拿到变更 key，重新拉取配置内容
```

**Spring Cloud 动态刷新**：
- `@Value` **不会自动刷新**
- 需要 `@RefreshScope` + `@Value`，或 `@ConfigurationProperties`（自动刷新，推荐）
- **`@RefreshScope` 的原理**：被标注的 Bean 是代理（`@ScopedProxyMode`），配置变更时该 scope 的 Bean 缓存被清空，下次访问时重新创建 → 拿到新配置
- **`@RefreshScope` 的坑**：① Bean 重建后原来的状态丢失；② 与某些框架（如 Feign、定时任务）冲突；③ 大量 Bean 加 `@RefreshScope` 会导致刷新时性能抖动

**其他功能**：
- **灰度发布（Beta）**：`betaIps` 指定先推给哪些 IP
- 历史版本与回滚（默认保留 30 天）
- 配置导入/导出、监听查询接口 `/nacos/v1/cs/configs/listener`
- 一个实例监听的配置过多时会有批次限制，注意分批

**鉴权（必须开）**：
```properties
nacos.core.auth.enabled=true
nacos.core.auth.plugin.nacos.token.secret.key=<自定义 32 位以上 Base64 字符串>
```
**默认密钥是 `SecretKey012345678901234567890123456789012345678901234567890123456789`**（且 2.2.0 之前 `user-agent` 绕过漏洞 CVE-2021-29441）。开着鉴权却用默认密钥 = 等于没鉴权。这是最常见的未授权访问事故来源。

### 3.6 部署要点

- 集群最少 **3 节点**
- **必须外置 MySQL**（Derby 仅限单机模式，集群下各节点数据不共享）
- `cluster.conf` 里必须写**真实 IP**，不能写 `127.0.0.1` 或 `localhost`（否则节点间无法通信）
- 通过 VIP / 域名 / SLB 暴露给客户端（客户端只需配一个地址）
- 数据一致性：服务数据靠 Distro/JRaft，配置数据靠 **JRaft log + MySQL 持久化**（配置是强一致的）
- **版本升级注意 gRPC 端口**（1.x → 2.x 必须放行 9848/9849）

**Spring Cloud Alibaba 版本矩阵**（配错会各种诡异报错，必查）：

| Spring Cloud Alibaba | Spring Cloud | Spring Boot | Nacos |
|---|---|---|---|
| 2021.0.5.0 | 2021.0.5 | 2.6.x | 2.x |
| 2022.0.0.0 | 2022.0.0 | 3.0.x | 2.2.x+ |
| 2023.0.1.0 | 2023.0.1 | 3.2.x | 2.3.x+ |

## 二、必懂机制（三）：Dubbo

### 4.1 架构与设计哲学

五个角色：**Provider / Consumer / Registry / Monitor / Container**

**核心设计：URL 总线 + SPI**
```
dubbo://192.168.1.10:20880/com.example.OrderService?
  version=1.0.0&group=pay&timeout=3000&retries=0&serialization=hessian2
  &side=provider&methods=create,query&timestamp=...
```
**为什么 URL 是 Dubbo 的灵魂**：
- 所有配置（协议、地址、接口名、超时、重试、序列化、版本、分组、方法级参数）都被编码进 URL
- 服务导出时，URL 被注册到注册中心
- 服务引用时，从注册中心拿到 URL，逐层加工（Cluster → Router → LoadBalance）后生成 Invoker
- **所有扩展点都通过 URL 的参数来决策行为**（如 URL 里的 `loadbalance` 参数决定用哪个 LoadBalance 实现）

理解这一点的直接收益：排查配置不生效问题时，直接把 URL 打出来看（` Arthas` 里用 `ognl` 调用 `getUrl()`），比看配置文件准得多。

### 4.2 SPI 机制

**为什么不用 JDK SPI，要自制 ExtensionLoader**：
| 能力 | JDK SPI | Dubbo SPI |
|---|---|---|
| 加载方式 | 一次性实例化所有实现类 | **懒加载，按需实例化** |
| IOC | 无 | **支持 setter 注入其他扩展** |
| AOP | 无 | **支持 Wrapper 包装类** |
| 配置格式 | 无 key，只能遍历 | `key=value` 按名字取 |
| 默认实现 | 无 | `@SPI("dubbo")` 指定默认值 |

**三件套**：
```java
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);

// 1. 按名字获取具体扩展
Protocol p = loader.getExtension("dubbo");

// 2. 获取自适应扩展（运行时根据 URL 参数决定用哪个）
Protocol adaptive = loader.getAdaptiveExtension();   // @Adaptive 注解
// 生成的代码类似：String name = url.getParameter("protocol", "dubbo");
//                return loader.getExtension(name);

// 3. 获取自动激活的扩展（@Activate，按 group/value 条件过滤）
List<Filter> filters = loader.getActivateExtension(url, "provider");
```

**@Adaptive 的实现**：Dubbo 用 **javassist 动态生成**一个 `Protocol$Adaptive` 类（不是反射），里面从 URL 取参数再委派。可以在 `ExtensionLoader.createAdaptiveExtensionClassCode()` 里看到拼字符串的源码，debug 时可以打出来看。

**Wrapper 实现 AOP**：如果扩展实现类有一个**以该接口为参数的构造函数**，Dubbo 认为它是包装类，会自动包在外层。
```java
class ProtocolFilterWrapper implements Protocol {
    public ProtocolFilterWrapper(Protocol protocol) { this.protocol = protocol; }
    // 在 refer/export 前后织入 Filter 链
}
```
`ProtocolListenerWrapper` / `ProtocolFilterWrapper` / `QosProtocolWrapper` 都是这么实现的。

**配置文件位置**：`META-INF/dubbo/`、`META-INF/dubbo/internal/`、`META-INF/services/`

### 4.3 服务导出与引用

**导出（Provider）**：
```
ServiceConfig.export()
  → 收集配置生成 URL
  → Protocol.export(invoker)   [被 FilterWrapper / ListenerWrapper 包装]
    → DubboProtocol.export() → 启动 Netty Server → 监听端口
    → RegistryProtocol.export() → 把 URL 注册到注册中心 + 订阅配置
  → 返回 Exporter
```

**引用（Consumer）**：
```
ReferenceConfig.get()
  → 从注册中心订阅，拿到 provider URL 列表
  → RegistryProtocol.refer() → 构建 RegistryDirectory
  → Cluster.join(directory) → 生成 ClusterInvoker（带容错能力）
  → Protocol.refer() → DubboProtocol 建立 Netty 客户端连接
  → ProxyFactory.getProxy(invoker) → 生成代理对象注入到 @Reference 字段
```

### 4.4 调用链（核心，必须逐层说清）

```
Proxy（JavassistProxyFactory / JdkProxyFactory）
  ↓
Cluster（容错：Failover / Failfast / Failsafe / Failback / Forking / Broadcast）
  ↓
Directory（RegistryDirectory：持有 invokers 列表，订阅注册中心变更）
  ↓
Router（路由：ConditionRouter / TagRouter / ScriptRouter）
  ↓
LoadBalance（负载均衡：Random / RoundRobin / LeastActive / ShortestResponse / ConsistentHash）
  ↓
Filter 链（ContextFilter / ExceptionFilter / MonitorFilter / TimeoutFilter / TPSLimiterFilter ...）
  ↓
Protocol → Exchange → Transport（Netty NIO 客户端）
  ↓
Serialization（Hessian2 默认）
```

**Cluster 容错策略**：
| 策略 | 行为 | 适用 |
|---|---|---|
| **Failover**（默认） | 失败自动切换其他节点，重试 `retries` 次 | 幂等的读操作。**非幂等写操作必须设 `retries=0`** |
| Failfast | 失败立即报错，只调一次 | 非幂等写操作 |
| Failsafe | 失败忽略，返回空 | 日志审计等非核心操作 |
| Failback | 失败记录，定时重发 | 消息通知 |
| Forking | 并行调多个，一个成功即返回 | 实时性要求高，牺牲资源 |
| Broadcast | 逐个调用所有，任一失败即失败 | 通知所有节点刷新缓存 |

**Router（灰度/机房优先的实现手段）**：
- **ConditionRouter 条件路由**：`consumer.host = 192.168.1.* => provider.host != 192.168.1.*`
- **TagRouter 标签路由**：给实例打 tag，请求带 tag 优先调同 tag。这是**灰度发布的标准实现**
- **ScriptRouter**：用脚本定义路由规则

**LoadBalance**：
| 策略 | 特点 |
|---|---|
| **Random**（默认） | 加权随机，实现简单，分布均匀 |
| RoundRobin | 加权轮询。**缺点：慢节点会累积请求**（请求按序分配，不会因为节点慢而少分） |
| **LeastActive** | **优先调活跃数（正在处理的请求数）最小的节点**。慢节点累积的活跃数高，自然少分到请求 → **最适合节点性能不均的场景** |
| ShortestResponse | 按历史响应时间的滑动窗口加权，响应时间短的权重高 |
| ConsistentHash | 同一参数的请求固定落到同一节点，用于**会话保持 / 有状态服务** |

**Filter**：内置的有 `ContextFilter`（RpcContext 传递）、`ExceptionFilter`（异常包装）、`TimeoutFilter`、`MonitorFilter`（监控埋点）、`TokenFilter`（令牌验证）、`TPSFilter`（限流）。
自定义 Filter：实现 `Filter` 接口 + 加 `@Activate(group = {CONSUMER, PROVIDER})` + 在 `META-INF/dubbo/` 注册。

**序列化**：默认 Hessian2。
- Kryo / FST：性能更好，需要注册类
- Protobuf：跨语言，需要 IDL
- Java：原生，体积大、性能差、有安全风险，**不要用**
- **兼容性坑**：Hessian 对 DTO 加字段是兼容的，但**改字段名、改类型、删字段会反序列化失败**；枚举序列化要小心（Hessian 按 name 序列化，按 ordinal 反序列化的老版本有问题）

### 4.5 超时与重试（最容易出事的地方）

```yaml
dubbo:
  consumer:
    timeout: 3000      # 默认 1000ms
    retries: 0         # **默认是 2，意味着最多调用 3 次！**
```

**`retries` 默认 2 的危险性**：任何一次抖动（网络、GC、慢查询）都会导致**同一笔业务被调用 3 次**。如果接口是创建订单/扣款/发券，就是资损事故。

**规则：**
1. **所有非幂等写接口必须显式设 `retries=0`**
2. 或者全局设 `retries=0`，只对明确幂等的读接口单独开启
3. 服务端也要配 `timeout` —— 作为兜底保护自己（防止被慢客户端拖死）

**配置优先级**（从高到低）：
```
方法级 > 接口级 > 全局（consumer/provider 级）
Consumer 侧 > Provider 侧（同一级别时）
```
**回退规则**：如果 Consumer 侧没配某个参数，会用 Provider 侧的（Provider 的 URL 会带出自己的配置）。这个"Provider 配置可以下发到 Consumer"的特性很实用，但也意味着你在 Provider 上配了 `retries=2`，Consumer 没配就会继承。

### 4.6 线程模型（重点）

**两类线程**：
1. **IO 线程（Netty EventLoop）**：负责网络读写、编解码。**绝不能阻塞**
2. **业务线程（Dubbo ThreadPool）**：负责执行实际的业务方法

**Dispatcher 五种策略**（决定"请求处理交给谁"）：

| 策略 | 语义 |
|---|---|
| **all**（默认） | 除 IO 读写和心跳外，**全部**（包括解码后的业务调用、响应、连接事件、异常）都派发到业务线程池 |
| direct | **全部都在 IO 线程执行**，不派发 |
| message | 只有"请求/响应消息"派发到线程池，其他（连接、心跳、异常）在 IO 线程 |
| execution | 只有"请求消息"派发，响应和其他事件在 IO 线程 |
| connection | 连接/断开事件派发到线程池（**这个 pool 是有队列的**），其余在 IO 线程 |

**为什么业务不能跑在 IO 线程**：一个 Netty EventLoop 线程负责**多条连接**。如果业务阻塞在这个线程上，它负责的所有连接都不可用——一个慢请求会拖垮一批连接。所以 `all` 是默认且安全的选择。

**ThreadPool 四种策略**：
| 策略 | 行为 |
|---|---|
| **fixed**（默认） | 固定大小（`threads` 默认 200），队列默认 `queues=0`（SynchronousQueue，无缓冲） |
| cached | 空闲 1 分钟回收，无限增长 |
| limited | 可增长但有上限 |
| eager | 先创建线程到 `core` 再到 `max`，最后才进队列（优先创建线程，保证低延迟） |

**`Thread pool is EXHAUSTED!` 错误的排查主线**：
```
RejectedExecutionException: Thread pool is EXHAUSTED!
  → 业务方法耗时过长（慢 SQL、外部接口慢、锁竞争）
  → 或 threads 太小 / 流量突增
  → 或下游服务不可用导致大量请求堆积在等待
```
排查步骤：
1. `jstack` 看业务线程（`DubboServerHandler-xxx` 前缀）在干什么，是否有大量 BLOCKED 或 TIMED_WAITING
2. 如果是慢 SQL / 外部接口 → 优化耗时或加超时熔断
3. 如果是流量 → 扩容 + 调大 `threads`（注意：`threads` 调太大也会有上下文切换开销，一般 200~500）
4. **设置合理的 `timeout`**：下游超时不设，请求会一直占着线程

**注意 `queues` 参数**：默认 0（直接拒绝，快速失败）。设成非 0 会缓冲请求，但**会让超时控制失效**（请求在队列里排队的时间不计入 timeout）→ 用户感知的超时远超配置值。

### 4.7 服务治理

**本地存根 Stub / Mock**：
- **Stub**：Consumer 端先执行本地逻辑（参数校验、缓存、容错）再调远程
- **Mock**：`MockClusterInvoker`，服务不可用时返回预设值。`mock="return null"` / `mock="force:return null"` / 自定义 Mock 类。用于**服务降级和容灾演练**

**泛化调用 `GenericService`**：不需要依赖服务方的 API jar，用 `Map` 描述参数即可调用。用于**网关、测试平台、服务治理平台**。
```java
GenericService service = (GenericService) context.getBean("orderService");
Object result = service.$invoke("create", new String[]{"com.example.OrderDTO"},
                                new Object[]{paramMap});
```

**优雅停机**：
```
1. 收到 SIGTERM
2. Dubbo ShutdownHook 触发
3. 先从注册中心注销（Provider 摘流）
4. 等待正在处理的请求完成（默认 10 秒，dubbo.service.shutdown.wait）
5. 关闭协议端口、线程池
```
- **QOS 端口**（默认 22222）：`dubbo.application.qos-enable=true`，可通过 `telnet localhost 22222` 执行 `offline`（手动摘流）、`ls`、`invoke` 等运维命令。`qos-accept-foreign-ip=false` 防外网访问
- **注意**：`kill -9` 不会执行 shutdown hook，Provider 不会主动摘流，Consumer 要等注册中心心跳超时才发现 → 会有几秒的调用失败。所以**必须用 `kill <pid>`（SIGTERM），不能用 -9**

**主机地址绑定（多网卡坑）**：
```yaml
dubbo:
  protocol:
    host: 192.168.1.10    # 显式指定，多网卡机器必配
```
不配的话 Dubbo 会自动选一个网卡地址，**在有多网卡（内网/外网/ docker0）的机器上很容易选错**，导致 Consumer 拿到不可达的地址。

### 4.8 Dubbo 3.x 新特性（区分懂不懂的分水岭）

**1. 应用级服务发现**（最大的变化）

| | 接口级（2.x） | 应用级（3.x） |
|---|---|---|
| 注册单位 | 每个**接口**一条记录 | 每个**应用实例**一条记录 |
| 数据量 | O(接口数 × 实例数) | **O(实例数)** |
| 1000 实例 × 50 接口 | 50000 条 | 1000 条 |
| 注册中心压力 | 大，推送风暴 | **小** |

接口到应用的映射关系通过**元数据中心**查询，不再依赖注册中心。

**三阶段迁移模型**（`dubbo.application.service-discovery.migration`）：
```
FORCE_INTERFACE     → 强制接口级（迁移起点）
APPLICATION_FIRST   → 双订阅，优先应用级（平滑过渡，推荐）
FORCE_APPLICATION   → 强制应用级（迁移终点）
```
迁移时必须所有 Provider 和 Consumer 都升级到 3.x，且分阶段推进。

**2. Triple 协议**
- 基于 **HTTP/2**，兼容 gRPC，支持 streaming
- 解决了 Dubbo 协议对网关、多语言、穿透性不友好的问题
- **是官方推荐的未来默认协议**

**3. 其他**：Proxyless Mesh（无 sidecar 的服务网格）、统一路由规则 v3（更强大的流量治理）、Native Image 支持。

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| Consumer 报 `No provider available` | ① 接口全限定名/版本/分组不匹配；② Provider 未注册成功；③ 注册中心地址不一致；④ 路由规则过滤掉了 | 控制台查服务列表；核对 interface/version/group；核对两边注册中心地址；查路由规则 |
| 接口报 `NoSuchMethod` / 找不到方法 | **方法签名（参数类型）不一致**，DTO 类结构不同 | 保证 API jar 版本一致；方法签名不能改；用版本号隔离 |
| 重复下单 / 重复扣款 | **`retries` 默认 2**，一次抖动就调 3 次 | 非幂等接口设 `retries=0`；业务侧做幂等（唯一键） |
| 超时设得比下游还短，雪崩放大 | 上游 timeout 小于数据库连接池等待时间，导致大量重试叠加 | timeout 要大于下游 P99；配合熔断（Sentinel）；retries=0 |
| `Thread pool is EXHAUSTED` | 业务耗时过长 / threads 太小 / 下游卡死 | jstack 看线程；优化慢点；调 `threads`；设合理 timeout；加熔断 |
| 序列化失败 / DTO 字段读不到 | DTO 改了字段名/类型，两边 API jar 版本不一致 | 只向后兼容地加字段；统一 API jar 版本；枚举序列化要谨慎 |
| Provider 和 Consumer 互相看不到 | 两边配的注册中心地址/namespace/group 不一致 | 核对 `registry.address`、`namespace`、`group` |
| Nacos 注册成功但订阅不到服务 | **2.x 的 gRPC 端口 9848/9849 未放行**（只开了 8848） | 放行 8848、9848、9849、7848 |
| Nacos 集群起不来 | `cluster.conf` 写了 `127.0.0.1`；或用 Derby 而非外置 MySQL | cluster.conf 写真实 IP；配外置 MySQL |
| Nacos 未授权访问 | 用了默认 `SecretKey0123456789...` 且开了鉴权 | 修改自定义密钥；升级到修复版；限制 8848 端口访问来源 |
| 配置改了不生效 | `@Value` 不自动刷新；或 `@RefreshScope` 未加；或 dataId/group/namespace 不匹配 | 用 `@ConfigurationProperties`（推荐）或加 `@RefreshScope`；核对三要素 |
| `@RefreshScope` 加了但 Bean 状态丢了 | `@RefreshScope` 的原理是**销毁并重建 Bean** | 无状态 Bean 才用它；有状态的改用事件监听手动更新 |
| 网络抖动导致实例被大量摘除 | 临时实例走 AP + 心跳，心跳断即剔除 | 调大心跳超时；开启健康保护阈值；关键服务改用持久实例（CP） |
| ZK 集群选举期间服务全部不可用 | ZK 是 CP，选举期间不可写 | 大规模服务发现场景改用 Nacos（AP）；ZK 集群要 3/5 节点且网络质量要好 |
| ZK 磁盘写满 | `autopurge.purgeInterval` 默认 0（永不清理） | 配 `autopurge.purgeInterval=24` + `snapRetainCount=10`；dataLogDir 与 dataDir 分盘 |
| ZK session 过期导致锁失效 | GC 停顿/网络抖动导致心跳超时，临时节点被删，锁被他人获得 | 业务侧加 fencing token；或改用 Redisson；调大 sessionTimeout |
| ZK Watcher 漏掉变更 | Watcher 一次性，重新注册前的变更感知不到 | 用 Curator 的 NodeCache/PathChildrenCache；收到通知后强制拉全量兜底 |
| 多网卡机器注册了错误 IP | Dubbo 自动选网卡选错（选到 docker0 或外网网卡） | 显式配 `dubbo.protocol.host`；或配 `dubbo.protocol.name` 指定网卡名 |
| 直连地址没关，连到了测试环境 | `dubbo.reference.url` 里写了直连地址且忘了删 | 上线前检查配置；用 URL 不等于注册中心发现时优先直连 |
| `kill -9` 导致调用失败几秒 | 没走 shutdown hook，Provider 未主动摘流，Consumer 等心跳超时才发现 | 用 `kill <pid>`（SIGTERM）；或用 QOS 的 `offline` 先摘流再停 |
| Dubbo 3.x 迁移后部分服务找不到 | 部分应用还是接口级，部分已经是应用级 | 用 `APPLICATION_FIRST` 双订阅过渡；确保所有节点都升级 |
| 接口级服务发现下注册中心压力大 | 数据量 = 接口数 × 实例数 | 迁移到 Dubbo 3.x 应用级服务发现 |
| 消费端报 `SerializationException` | Provider 用 Hessian2、Consumer 配了别的序列化；或类路径不一致 | 显式配置 `serialization`；统一 API jar |

## 四、往下钻的入口

**Zookeeper 源码 / 配置**：
- `zoo.cfg`：`tickTime` / `initLimit` / `syncLimit` / `dataDir` / `dataLogDir` / `clientPort` / `maxClientCnxns` / `autopurge.*`
- ZK 源码：`org.apache.zookeeper.server.quorum.LeaderElection`（选主）、`FastLeaderElection`、`ZooKeeperServer`、`DataTree`（内存数据库）
- Curator：`org.apache.curator.framework.recipes.locks.InterProcessMutex`、`CuratorFramework` 的 `ConnectionStateListener`
- 目录：`/data/zookeeper/data/version-2/`（snapshot）、`/data/zookeeper/datalog/version-2/`（事务日志）

**Nacos 源码 / 配置**：
- `conf/application.properties`：`nacos.core.auth.enabled`、`db.url.0`、`server.port`、`nacos.naming.cleanup.*`
- `conf/cluster.conf`
- 源码入口：
  - `com.alibaba.nacos.client.config.impl.ClientWorker.LongPollingRunnable`（配置长轮询）
  - `com.alibaba.nacos.client.naming.NacosNamingService`（服务发现客户端）
  - `com.alibaba.nacos.naming.core.Service` / `Instance`
  - `com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl`（AP）
  - `com.alibaba.nacos.core.distributed.raft.JRaftServer`（CP）
  - `com.alibaba.nacos.core.remote.grpc.GrpcServer`（2.x 长连接）
- Open API：`/nacos/v1/ns/instance/list`、`/nacos/v1/cs/configs`、`/nacos/v1/cs/configs/listener`

**Dubbo 源码入口**（按调用顺序）：
- `org.apache.dubbo.config.ServiceConfig.export()`
- `org.apache.dubbo.config.ReferenceConfig.get()` / `init()`
- `org.apache.dubbo.common.extension.ExtensionLoader.getAdaptiveExtension()`（SPI 核心）
- `org.apache.dubbo.registry.integration.RegistryProtocol.export()` / `refer()`
- `org.apache.dubbo.rpc.cluster.Directory` / `RegistryDirectory.notify()`
- `org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke()`
- `org.apache.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance.doSelect()`
- `org.apache.dubbo.remoting.transport.dispatcher.all.AllChannelHandler`（Dispatcher）
- `org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol`
- 3.x：`org.apache.dubbo.registry.client.migration.MigrationRuleListener`（应用级发现迁移）

**排查命令与工具**：
```bash
# ZK
echo mntr | nc localhost 2181
echo srvr | nc localhost 2181
zkServer.sh status
# Nacos
curl 'http://localhost:8848/nacos/v1/ns/instance/list?serviceName=order-service'
curl 'http://localhost:8848/nacos/v1/cs/configs?dataId=x&group=y'
ss -antp | grep 9848        # 看 gRPC 连接数
# Dubbo（QOS）
telnet localhost 22222
> ls              # 列出服务
> offline         # 摘流
> invoke com.x.Service.method(args)
```

**官方文档具体章节**：
- ZK: *ZooKeeper Programmer's Guide*（Data Model、Watches、Consistency Guarantees）与 *Administrator's Guide*（Maintenance）
- Nacos: *Nacos 架构与原理* 中的一致性协议章节、*2.x 版本升级指南*
- Dubbo: *Dubbo 官网 → 原理 → 框架设计*、*SPI 扩展实现*、*线程模型*、*3.x 应用级服务发现迁移指南*

## 五、达标自检

**Zookeeper**
1. ZNode 有哪四种类型？临时节点在什么条件下被删除？临时节点能有子节点吗？
2. Watcher 的三个特性是什么？为什么说"收到通知到重新注册"之间存在风险窗口？
3. Stat 结构里 version / cversion / aversion 分别是什么？怎么用它们做 CAS？
4. sessionTimeout 的配置为什么可能被服务端改掉？改成什么范围？
5. 连接断开和 session 过期是一回事吗？重连成功的条件是什么？
6. ZAB 的两种模式分别做什么？为什么过半 ACK 就可以提交，不需要等所有节点？
7. Follower 和 Observer 的区别是什么？Observer 存在的意义是什么？
8. 为什么 ZK 是 CP？这个特性作为注册中心有什么问题？
9. 3 节点和 4 节点的容灾能力一样吗？为什么？
10. `dataDir` 和 `dataLogDir` 为什么要分盘？不配 autopurge 会发生什么？
11. 用临时顺序节点实现分布式锁的完整流程是什么？为什么要监听"前一个"而不是父节点？
12. **ZK 分布式锁在什么情况下会失效？怎么彻底解决？**

**Nacos**
13. Nacos 和 Eureka / Consul / Apollo 的定位差异是什么？
14. `namespace` / `group` / `serviceName` 三者的关系是什么？只带 serviceName 能唯一定位服务吗？
15. **临时实例和持久实例在一致性协议、健康检测方式、网络抖动时的行为上有什么区别？**
16. 健康保护阈值解决的是什么问题？默认是多少？
17. Nacos 2.x 的 9848 / 9849 / 7848 端口分别是什么？只放 8848 会出现什么现象？
18. 配置长轮询的原理是什么？为什么服务端要挂起 30 秒？
19. 为什么 `@Value` 配置改了不刷新？`@RefreshScope` 的原理是什么？它有什么副作用？
20. Nacos 鉴权的默认密钥问题是什么？正确配置是什么？
21. 部署 Nacos 集群时，`cluster.conf` 最容易犯什么错？为什么必须用外置 MySQL？

**Dubbo**
22. 为什么说 URL 是 Dubbo 的灵魂？请举例说明 URL 在调用链上被如何使用。
23. Dubbo 为什么要自制 SPI 而不是用 JDK SPI？至少说出 3 个理由。
24. `@Adaptive` 的实现原理是什么？`getExtension` / `getAdaptiveExtension` / `getActivateExtension` 三者的区别？
25. Wrapper 类是怎么实现 AOP 的？写一个 Wrapper 需要满足什么条件？
26. **完整说出 Dubbo 的调用链（Proxy → ... → Transport），每一层负责什么。**
27. Failover / Failfast / Failsafe / Failback / Forking / Broadcast 各自的语义和适用场景？
28. LeastActive 为什么适合节点性能不均的场景？ConsistentHash 适用什么场景？
29. **`retries` 的默认值是多少？这会带来什么风险？正确的做法是什么？**
30. Dubbo 配置优先级是怎样的？Consumer 侧没配的参数会怎样？
31. **Dispatcher 五种策略的区别是什么？为什么业务不能跑在 IO 线程？**
32. `Thread pool is EXHAUSTED` 的排查步骤是什么？`queues` 设为非 0 会有什么副作用？
33. 优雅停机的流程是什么？为什么不能用 `kill -9`？
34. 多网卡机器会遇到什么问题？怎么解决？
35. **Dubbo 3.x 应用级服务发现解决了什么问题？数据量的变化是什么？三阶段迁移模型是什么？**
36. Triple 协议基于什么？相比 Dubbo 协议有什么优势？
37. 你的服务要上线 Dubbo 3.x，从 2.x 迁移的完整步骤是什么？需要注意什么？
