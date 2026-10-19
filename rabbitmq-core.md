# RabbitMQ · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方

## 一、核心心智模型

1. **RabbitMQ 的核心价值是"灵活路由"，不是"高吞吐"。** 它基于 AMQP 的 Exchange + Binding 模型，能表达 Kafka/RocketMQ 表达不了的复杂分发逻辑（按 header 过滤、按 topic 通配、广播、不可路由兜底）。代价是吞吐上限明显低于 Kafka。**选型时先问"我要的是路由灵活性还是吞吐"。**
2. **消息可靠性不是一个开关，是三段责任的拼接。** 生产端不丢（confirm + mandatory）、Broker 不丢（durable + 落盘）、消费端不丢（手动 ack）。**任何一段没做，整条链路就是不可靠的**——而且这三段之间没有任何联动。
3. **confirm 只保证"到达了 Exchange"，不保证"到达了 Queue"。** 这是 RabbitMQ 最大的认知陷阱。不可路由的消息在 `mandatory=false` 时会被**静默丢弃**，而 confirm 依然返回 ack。
4. **prefetch 限制的是 unacked 数量，不是缓冲区大小。** 理解这一点才能解释"为什么 prefetch=1 吞吐上不去"和"为什么 unacked 只增不减"。
5. **RabbitMQ 不提供 exactly-once。** 它只保证 at-least-once（默认）或 at-most-once。**幂等必须由业务侧实现**，这是硬约束而不是可选项。

## 二、必懂机制

### 2.1 AMQP 模型

```
Producer → Exchange --(Binding + RoutingKey)--> Queue → Consumer
```

**Connection vs Channel**：
- Connection 是 TCP 连接，创建成本高（握手、认证）
- Channel 是 Connection 内的**逻辑信道**，多路复用同一条 TCP
- **正确用法：一个应用一个 Connection，每个线程/每个消费者一个 Channel**。常见错误是每条消息建一个 Connection（连接数暴涨）或在多线程间共享一个 Channel（Channel 不是线程安全的）

**Virtual Host**：逻辑隔离单位，相当于 namespace。exchange/queue/binding 都隶属于某个 vhost，不同 vhost 完全隔离（包括权限）。默认 `/`。

**默认交换机**：名字是空字符串 `""`，direct 类型，**每个 queue 会自动绑定到它，routing key 就是 queue 名字**。所以 `basicPublish("", "myQueue", null, body)` 能直接投递到 `myQueue`——这是很多简写例子的原理。

**四种 Exchange 类型**：

| 类型 | 路由规则 | 典型场景 |
|---|---|---|
| **direct** | routing key 完全相等 | 点对点、按业务类型分发 |
| **topic** | 按 `.` 分词的模式匹配，`*` 匹配一个词，`#` 匹配零或多个词 | 多级分类发布订阅（`order.created.cn`） |
| **fanout** | 忽略 routing key，广播到所有绑定队列 | 广播通知、缓存刷新 |
| **headers** | 按消息 header 的 key-value 匹配，忽略 routing key。`x-match: all`（全部匹配）或 `any`（任一匹配） | 多维度条件路由（很少用，性能不如 topic） |

**Alternate Exchange（AE）**：给 exchange 设置 `alternate-exchange` 参数，不可路由的消息会被转发到 AE。**这是 mandatory 的服务端替代方案**，能保证不可路由消息不会静默丢失。

### 2.2 Queue 属性与 arguments

```java
Map<String, Object> args = new HashMap<>();
args.put("x-max-length", 100000);                        // 队列最大消息数
args.put("x-overflow", "drop-head");                     // 满了之后：drop-head（丢队头）/ reject-publish（拒绝生产者，会走 Return）
args.put("x-message-ttl", 60000);                        // 消息 TTL（毫秒）
args.put("x-expires", 1800000);                          // 队列空闲多久后自动删除（毫秒）
args.put("x-max-priority", 10);                          // 优先级 1-255
args.put("x-dead-letter-exchange", "dlx.exchange");      // 死信交换机
args.put("x-dead-letter-routing-key", "dlx.key");        // 死信路由键（不设则用原 routing key）
args.put("x-queue-mode", "lazy");                        // 惰性队列
args.put("x-queue-type", "quorum");                      // quorum 队列（3.8+）
channel.queueDeclare("order.queue", true, false, false, args);
//                       durable  exclusive  autoDelete  arguments
```

三个布尔参数的含义：
- **durable**：队列元数据持久化（Broker 重启后队列还在）。**注意：durable 不等于消息持久化**，消息持久化靠 `deliveryMode=2`
- **exclusive**：独占，只对首次声明它的 Connection 可见，Connection 断开时自动删除
- **auto-delete**：最后一个消费者断开后自动删除

**惰性队列 lazy queue**：消息尽可能存磁盘而非内存，支持百万级堆积。代价是吞吐显著下降（磁盘 IO）。适合"偶尔突发大量消息、允许慢消费"的场景。与 quorum 队列不能同时用。

**三种 Queue 类型**（3.8+）：

| 类型 | 复制机制 | 适用 | 注意 |
|---|---|---|---|
| **classic** | 无（普通集群下不复制消息，只存在于创建它的节点） | 默认，性能最好 | 节点宕机则该队列不可用 |
| **quorum** | **Raft 共识**，多数派复制 | **新版推荐的高可用方案**，替代镜像队列 | 需要奇数节点；不支持 TTL/优先级/排他/惰性；`delivery-limit` 控制重投次数 |
| **stream** | 追加日志 + 多副本 | 需要**重放/多次消费**的场景 | 类似 Kafka 的语义，消息消费后不立即删除 |

### 2.3 可靠性三段论（重点）

#### 第一段：生产端不丢

**publisher confirm 三种模式**：

```java
channel.confirmSelect();   // 开启 confirm

// 1. 普通确认（同步，逐条等待）—— 吞吐最低
if (channel.waitForConfirms()) { /* ok */ }

// 2. 批量确认（同步，批量等待）—— 折中，但失败时不知道是哪条
for (int i = 0; i < batchSize; i++) channel.basicPublish(...);
channel.waitForConfirmsOrDie();   // 有一条失败就抛 IOException

// 3. 异步确认（推荐）—— 吞吐最高
channel.addConfirmListener((deliveryTag, multiple) -> {
        // ack：Broker 已收到且已路由
    }, (deliveryTag, multiple) -> {
        // nack：Broker 处理失败，需重发（deliveryTag 定位是哪条）
    });
```

**关键认知：confirm 只代表 Exchange 收到了消息。**

| 情况 | mandatory=false | mandatory=true |
|---|---|---|
| 消息成功路由到队列 | confirm ack | confirm ack |
| 消息无法路由（无匹配 binding） | **confirm ack，消息静默丢弃** | confirm ack **+ basic.return 回调通知生产者** |

所以**必须同时开启 mandatory + ReturnCallback**：
```java
channel.addReturnListener(returnMessage -> {
    log.error("消息不可路由: exchange={}, routingKey={}, replyCode={}",
        returnMessage.getExchange(), returnMessage.getRoutingKey(), returnMessage.getReplyCode());
    // 落库 + 定时重试
});
channel.basicPublish(exchange, routingKey, true,   // mandatory=true
    MessageProperties.PERSISTENT_TEXT_PLAIN, body);
```

**事务 `txSelect/txCommit`**：能保证可靠但性能下降约 250 倍，**生产环境禁止使用**，用 confirm 替代。

**生产端最终一致性方案**：confirm 异步化 + 失败消息落本地表（记录 exchange/routingKey/body/状态）+ 定时任务扫描重投 + 超过 N 次告警人工介入。这个模式叫**本地消息表**。

#### 第二段：Broker 不丢

三者缺一不可：
1. `exchangeDeclare(..., durable=true)`
2. `queueDeclare(..., durable=true, ...)`
3. `new AMQP.BasicProperties.Builder().deliveryMode(2).build()`（或 `MessageProperties.PERSISTENT_TEXT_PLAIN`）

**但即使三者都开，也不是绝对不丢**：
- 消息写入磁盘有短暂的时间窗口（先写 buffer，由后台线程刷盘）
- 真正需要强保证时用 **quorum 队列**（Raft 多数派确认后才返回）
- 也可以依赖 publisher confirm：Broker 在**消息落盘后**才发 ack（持久化消息的行为），未持久化的消息在写入内存后即 ack

**内存/磁盘水位触发流控**：
- `vm_memory_high_watermark`（默认 0.4，即 40% 内存）→ 触发后 **阻塞所有 publisher**（Connection 进入 blocked 状态）
- `disk_free_limit`（默认 50MB）→ 磁盘剩余低于阈值同样阻塞 publisher
- 阻塞表现：客户端 `Connection` 收到 `connection.blocked` 通知，写入会挂起。客户端要监听 `BlockedListener` 做降级

#### 第三段：消费端不丢

```java
// 错误：autoAck=true，消息一投递出去 Broker 就删了，消费者处理失败消息就没了
channel.basicConsume(queue, true, callback);

// 正确：手动 ack
DeliverCallback callback = (consumerTag, delivery) -> {
    try {
        process(delivery.getBody());
        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);  // multiple=false
    } catch (Exception e) {
        // requeue=true 重新入队（可能无限循环）
        // requeue=false 丢弃或进死信（配了 DLX 的话）
        channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);
    }
};
channel.basicConsume(queue, false, callback);   // autoAck=false
```

`basicNack` 的三个参数：`deliveryTag`、`multiple`（是否批量处理该 tag 之前的全部）、`requeue`。
`basicReject` 相当于 `basicNack` 的 multiple=false 版本。

**unacked 只增不减的常见成因**（排查主线）：
1. 消费端处理慢或卡死（线程池满、下游超时、死锁）
2. `prefetch` 设置过大，一次性投递太多消息给消费者
3. 消费者连接没断开（连接保持，消息就不会被重新投递）—— 这是"看起来消费者挂了但消息一直不重投"的原因
4. 手动 ack 代码路径有分支漏了 `basicAck`

### 2.4 幂等设计

**重复消费的四个来源**：
1. 生产者发送后未收到 confirm（网络抖动），重发
2. 消费者处理完但 ack 前连接断开，Broker 重新投递
3. `requeue=true` 导致消息重新入队
4. 消费失败重试机制（服务端重试队列或应用层重试）

**去重方案**（按可靠程度排序）：
1. **数据库唯一索引**：消息里带业务唯一键（如订单号），插入冲突即视为已处理。最可靠，推荐
2. **Redis SETNX**：`SET processed:<msgId> 1 NX EX 86400`，成功才处理。**注意**：Redis 宕机可能丢，且要处理"SETNX 成功但业务失败"的情况（需删除标记）
3. **去重表 + 事务**：`INSERT INTO consumed_msg(msg_id) VALUES(?)`，靠唯一索引保证，与业务在同一个事务里
4. **乐观锁 version**：更新时 `WHERE version = ?`，适用于状态流转类
5. **状态机前置校验**：处理前先查当前状态，若已处于目标状态则直接返回成功

**幂等表设计要点**：必须有唯一索引（msg_id 或 业务唯一键）、要有过期清理策略（避免无限增长）、去重标记与业务操作要尽量原子（同库同事务最好）。

### 2.5 消费控流

```java
channel.basicQos(1);   // prefetchCount
```

- **prefetchCount 是"该 Channel 上允许的 unacked 消息上限"**，不是缓冲区大小
- `prefetch=1`：严格公平分发，但吞吐低（每处理完一条才能拿下一条，往返延迟无法摊薄）
- 生产推荐：**prefetch = 消费者并发数 × 每批预取量**，如 20~300，需要在吞吐和内存之间权衡
- **批量 ack**：`basicAck(tag, true)` 一次性确认该 tag 及之前的所有消息。必须保证这批都处理完了

**为什么消费端要另起线程池**：`@RabbitListener` 的回调运行在容器的消费线程上。如果在回调里直接做耗时业务，会占满消费线程 → 吞吐上不去、unacked 堆积。正确做法：回调里把消息丢进业务线程池，或者调大 `concurrency`。

**Channel 泄漏**：每个消费者一个 Channel 是合理的，但如果动态创建消费者却没关闭，Channel 会累积。用 Spring 的 `CachingConnectionFactory` 时注意 `channelCacheSize`，超出部分不会被缓存。

### 2.6 死信队列 DLX

成为死信的三种情况：
1. `basicNack` / `basicReject` 且 `requeue=false`
2. 消息 TTL 到期
3. 队列达到 `x-max-length` 上限被淘汰（`x-overflow: drop-head` 时）

```java
// 业务队列
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "dlx.exchange");
args.put("x-dead-letter-routing-key", "dlx.order");   // 不设则用原 routing key
channel.queueDeclare("order.queue", true, false, false, args);

// 死信交换机 + 死信队列
channel.exchangeDeclare("dlx.exchange", "topic", true);
channel.queueDeclare("order.dlq", true, false, false, null);
channel.queueBind("order.dlq", "dlx.exchange", "dlx.order");
```

**死信再投递的循环风险**：如果死信队列又配置回业务队列，会形成无限循环。用消息头 `x-death` 判断重试次数：
```java
List<Map<String, Object>> death = delivery.getProperties().getHeaders().get("x-death");
// x-death 里含 count（重试次数）、queue、reason
// count > N 就不再 requeue，转人工处理
```

### 2.7 延迟消息

**方案一：TTL + DLX**（经典但有致命缺陷）

```java
// 给队列设 x-message-ttl=30000，配 DLX。消息 30 秒后进入死信队列，消费死信队列即为延迟消费
```

**致命缺陷：RabbitMQ 只在队头检查消息是否过期。** 如果一个队列里混着 TTL=10s 和 TTL=60s 的消息，10s 的那条排在 60s 的后面，它要等 60s 那条被消费掉才会被检查——**不同 TTL 混在一个队列会互相阻塞**。
解法：**按延迟时长分队列**（10s 队列、30s 队列、60s 队列...），或者用 `x-message-ttl` 设到消息级别但队列只放单一 TTL。

**方案二：`rabbitmq_delayed_message_exchange` 插件**
```bash
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```
- 声明 `x-delayed-type` 的 exchange，发送消息时加 `x-delay` 头（毫秒）
- 原理：内部用 Mnesia 存储 + 定时器，到期后投递
- **局限**：不适合大规模堆积（内存态）、管理界面可见性差、不支持可靠量化保证、集群下节点宕机可能丢延迟消息

**方案三（生产推荐）**：需要任意精度 + 大规模延迟时，用**外部时间轮**（Redis ZSet 按到期时间排序 + 定时扫描，或 RocketMQ 的延迟级别，或专门的调度服务），RabbitMQ 只负责即时投递。

### 2.8 RPC 模式

```
客户端 → 声明一个临时独占队列（exclusive, auto-delete），把名字放进 reply_to
      → 发消息带 correlation_id（唯一标识本次请求）
      → 服务端处理完 → 用 reply_to 作为 routing key，带上相同的 correlation_id 回发到默认交换机
      → 客户端在自己的临时队列收到响应，按 correlation_id 匹配
```
要点：**必须设超时**（服务端可能挂了）、**必须处理 correlation_id 不匹配的情况**（可能是上一次的残留响应）、临时队列要用 `queueDeclare()` 不传名字让服务端自动生成（`amq.gen-xxx`）。

### 2.9 集群与高可用

**普通集群**：只同步**元数据**（exchange/queue/binding 的定义），**消息不复制**。队列只存在于创建它的那个节点，客户端连到别的节点时由该节点转发。所以**节点宕机 = 该节点上的队列不可用 + 消息丢失**（除非持久化且能重启恢复）。

**镜像队列（ha-policy）**：老的 HA 方案，把队列复制到多个节点。
```bash
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
```
问题：同步开销大、网络分区时行为难预测、性能随副本数线性下降。**官方已不推荐，新版用 quorum 队列替代。**

**Quorum 队列**（推荐）：
```java
args.put("x-queue-type", "quorum");
```
- 基于 **Raft 共识协议**，需要**奇数个节点**（3 或 5）
- 写入需多数派确认，天然的高可用 + 数据安全
- `delivery-limit`：限制消息被重投的次数，超过后进死信（防止毒丸消息无限循环）
- 限制：不支持 TTL、优先级、排他队列、惰性队列

**网络分区处理策略**（`cluster_partition_handling`）：

| 策略 | 行为 | 适用 |
|---|---|---|
| `ignore` | 不处理，分区两侧独立运行，恢复后需人工介入 | 网络极可靠，或允许人工介入 |
| `pause_minority` | **少数派一侧自动暂停**（认为自己被分区了），多数派继续服务 | **推荐**：网络可靠性一般的环境，防止脑裂双写 |
| `autoheal` | 恢复后自动选一个"获胜"分区，其余节点重启同步 | 追求自动恢复，接受部分数据丢失 |

**Federation / Shovel**：跨地域、跨集群的桥接方案。
- Federation：单向订阅上游的消息到本地（exchange 或 queue 级别），适合跨机房汇聚
- Shovel：持续从一个源搬消息到目标，更底层，可跨版本

### 2.10 流控与运维

**水位配置**：
```bash
# rabbitmq.conf
vm_memory_high_watermark.relative = 0.4      # 或 vm_memory_high_watermark.absolute = 2GB
disk_free_limit.absolute = 2GB               # 或 .relative = 2.0（相对内存）
```
触发后 publisher 被阻塞。`credit-based flow control` 是更细粒度的 per-channel 流控。

**常用诊断命令**：
```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers memory
rabbitmqctl list_connections name peer_host channels user state
rabbitmqctl list_channels connection number consumer_count prefetch_count
rabbitmqctl list_consumers
rabbitmqctl list_exchanges / list_bindings
rabbitmq-diagnostics status / check_running / erlang_version
rabbitmq-top                                  # 类似 top 的进程视图
rabbitmq-perf-test --uri amqp://... --producers 10 --consumers 10   # 压测
```

**Policy**：`rabbitmqctl set_policy <name> <pattern> <definition> --priority <n> --apply-to queues`
- pattern 是正则，匹配队列名
- **优先级：policy > 客户端声明的 arguments**（policy 会覆盖 arguments）
- `--apply-to` 可选 `queues` / `exchanges` / `all`

**权限**：用户有 tag（`management`/`policymaker`/`monitoring`/`administrator`），vhost 级权限分 `configure` / `write` / `read` 三个正则。

### 2.11 Spring / Java 客户端要点

**CachingConnectionFactory**：
```java
CachingConnectionFactory cf = new CachingConnectionFactory("localhost");
cf.setChannelCacheSize(25);                        // 缓存的 Channel 数，超出不缓存
cf.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
cf.setPublisherReturns(true);                      // mandatory 的 return 回调需要开启
```

**RabbitTemplate 配置**：
```java
rabbitTemplate.setMandatory(true);
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> { ... });
rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> { ... });
```
`ConfirmType.CORRELATED` 下 `correlationData` 里带自定义 ID，能定位到具体消息；`SIMPLE` 模式下由 template 同步等待。

**`@RabbitListener` 与容器工厂联动**：
```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(CachingConnectionFactory cf) {
    SimpleRabbitListenerContainerFactory f = new SimpleRabbitListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setConcurrentConsumers(5);          // 初始消费者数
    f.setMaxConcurrentConsumers(20);      // 峰值自动扩到（Simple 容器支持动态调整）
    f.setPrefetchCount(50);               // 每个消费者的 unacked 上限 → 总 unacked = 消费者数 × prefetch
    f.setAcknowledgeMode(AcknowledgeMode.MANUAL);   // AUTO / MANUAL / NONE
    f.setDefaultRequeueRejected(false);   // 抛异常时不 requeue（关键！避免无限重试）
    return f;
}
```
- **`setDefaultRequeueRejected(false)` 很重要**：默认 true，抛异常会无限 requeue，造成"毒丸消息"死循环打爆消费者
- 配了 DLX 后 requeue=false 的消息会进死信队列

**序列化坑（必踩）**：
```java
// 默认 SimpleMessageConverter 走 JDK 序列化 → 跨语言不可用、版本不兼容、有安全风险
f.setMessageConverter(new Jackson2JsonMessageConverter());
```
Jackson2JsonMessageConverter 会在 header 里写 `__TypeId__`（全限定类名）。**跨服务时如果包名不同会反序列化失败**，解法：
```java
Jackson2JsonMessageConverter conv = new Jackson2JsonMessageConverter();
DefaultJackson2JavaTypeMapper mapper = new DefaultJackson2JavaTypeMapper();
mapper.setIdClassMapping(Map.of("order.created", OrderCreatedEvent.class));  // 显式映射
conv.setJavaTypeMapper(mapper);
```

**连接恢复**：`ConnectionFactory` 默认开启 `automaticRecovery`（网络恢复后自动重建 Connection 和 Channel）。**但注意：自动恢复会重建 Channel，消费者会自动重新注册，不过未 ack 的消息会被重新投递**（这是幂等必须做的原因之一）。

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 消息发出去了但队列里没有，也没有报错 | 没开 `mandatory`，消息不可路由被**静默丢弃**；confirm 仍返回 ack | 开 `mandatory=true` + `ReturnCallback`；或用 Alternate Exchange |
| unacked 消息只增不减 | 消费者处理慢/卡死；prefetch 过大；连接没断开所以消息不重投 | jstack 看消费者线程；调小 prefetch；处理耗时移到业务线程池；必要时重启消费者让消息重投 |
| 消费端吞吐上不去 | `prefetch=1` 导致每条都要一次网络往返；或在 `@RabbitListener` 回调里做重业务 | prefetch 调到 20~300；回调里只投递到业务线程池 |
| 消息被无限重复消费 | `defaultRequeueRejected` 默认 true，抛异常就无限 requeue | 设 `false` + 配 DLX；或用 `x-death` 头判断重试次数 |
| 消费者假死，消息不再被消费 | 业务线程阻塞导致 heartbeat 超时，连接被 Broker 断掉；或消费者进程还活着但线程卡死 | 监控 unacked 和 consumer count；业务处理加超时；检查 `requestedHeartbeat` |
| 消费端报 `ClassNotFoundException` / 反序列化失败 | JDK 序列化强依赖全限定类名和 `serialVersionUID`，跨服务包名不一致 | 换成 `Jackson2JsonMessageConverter` + 显式 `__TypeId__` 映射 |
| 跨语言消费者读不懂消息体 | 默认 `SimpleMessageConverter` 用 JDK 序列化 | 改用 JSON converter，header 里带 `content-type: application/json` |
| 连接数暴涨 | 每次发消息都新建 Connection，而不是复用 Channel | 一个应用一个 Connection，多线程各用各的 Channel |
| 队列里的消息全没了 | 队列声明为 `durable=false`，Broker 重启后队列消失（元数据没了） | 队列和交换机都声明 durable=true |
| Broker 重启后消息丢了 | 只设了 queue durable，没设消息 `deliveryMode=2` | `exchange durable` + `queue durable` + `deliveryMode=2` 三者齐全 |
| 生产者突然全部卡住 | 内存水位（`vm_memory_high_watermark` 0.4）或磁盘水位触发，publisher 被 block | 监控 `connection.blocked`；扩容；调水位；消费端提速 |
| 集群网络分区后出现双写 | `cluster_partition_handling` 设为 ignore | 改成 `pause_minority`；关键业务用 quorum 队列 |
| 镜像队列性能差、同步卡住 | 镜像队列同步开销大，大队列全量同步会长时间阻塞 | 迁移到 quorum 队列；老版本用 `ha-sync-mode: manual` 避免自动同步风暴 |
| 延迟消息不准时 | TTL + DLX 方案中不同 TTL 混在同一队列，队头阻塞 | 按延迟时长分队列；或改用插件；或外部时间轮 |
| 延迟消息在节点重启后丢失 | `rabbitmq_delayed_message_exchange` 插件的消息存 Mnesia 内存态 | 改用外部调度方案（Redis ZSet / DB 轮询 + 定时） |
| 队列改名后消息全部丢失 | 队列是实体，改名等于新建（旧队列还在但没人消费） | 先建新队列 + 双写 + 迁移消费，最后删旧队列 |
| 管理界面上 Ready=0 但 Unacked 巨大 | 消费者拿了消息但没 ack（处理慢/卡死/漏 ack） | 见"unacked 只增不减"；检查是否有分支漏了 basicAck |
| 消息顺序被打乱 | 多消费者并发消费同一队列；或 requeue 后重新入队位置变了 | 单消费者 + prefetch=1（牺牲吞吐）；或业务侧用版本号/序列号自行排序 |
| 大量消息堆积导致 Broker 内存告警 | 队列是 classic 类型，消息囤在内存 | 改 `x-queue-mode: lazy`；或扩容消费者；或临时转存后重放 |
| `x-max-priority` 设了但优先级没生效 | 优先级只在**队列有积压**时才有意义；且高优先级消息持续到达会导致低优先级消息饥饿 | 谨慎使用；接受饥饿风险或改用多队列 + 加权消费 |

## 四、往下钻的入口

**服务端源码 / 配置**：
- `/etc/rabbitmq/rabbitmq.conf`（新版配置格式）与 `advanced.config`
- Erlang 源码关键模块：`rabbit_channel`（Channel 逻辑）、`rabbit_amqqueue_process`（队列进程）、`rabbit_mirror_queue_mode`、Ra（quorum 的 Raft 实现，独立项目）
- `/var/lib/rabbitmq/mnesia/` —— 元数据存储目录
- `/var/log/rabbitmq/rabbit@<hostname>.log`

**客户端源码入口**：
- `com.rabbitmq.client.impl.ChannelN` —— `basicPublish` / `basicAck` / `basicConsume` 的实现，confirm 的 deliveryTag 维护在这里
- `com.rabbitmq.client.impl.AMQChannel` —— `rpc()` 方法，理解 Channel 如何做同步 RPC
- `org.springframework.amqp.rabbit.connection.CachingConnectionFactory` —— Channel 缓存逻辑
- `org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer` —— 消费者线程模型、动态扩缩容
- `org.springframework.amqp.rabbit.core.RabbitTemplate` —— `doSend` 里 mandatory 和 confirm 的处理

**官方文档具体章节**：
- *RabbitMQ Tutorials* 2（Work Queues）、5（Topics）、6（RPC）—— 官方交互式教程
- *Reliability Guide*（Part 1 生产者侧 / Part 2 消费者侧）—— 必读
- *Quorum Queues* 文档 —— 含与镜像队列的对比矩阵
- *Network Partitions* 文档
- *Memory Use / Alarms* 文档

**必备验证命令**：
```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged
rabbitmqctl list_consumers
rabbitmq-diagnostics status | grep -A5 "Memory"
rabbitmqctl list_connections name state channels
```

## 五、达标自检

1. Exchange 到 Queue 的路由发生在哪一步？binding 记录的是什么？
2. Connection 和 Channel 的关系是什么？为什么不能多线程共享一个 Channel？
3. 默认交换机（名字为空字符串）的 routing key 是什么？为什么 `basicPublish("", queueName, ...)` 能直接发到队列？
4. topic 交换机里 `*` 和 `#` 的匹配规则分别是什么？`order.*.cn` 能匹配 `order.created.cn` 吗？`order.#` 呢？
5. **confirm 返回 ack 了，能说明消息到达队列了吗？为什么？** 不可路由的消息怎么感知？
6. `mandatory=true` 和 Alternate Exchange 有什么区别？各自适用什么场景？
7. 保证 Broker 重启后消息不丢，需要设置哪三个地方？为什么三者缺一不可？
8. 即使三者都设了，为什么说也不是绝对不丢？怎样做到更强保证？
9. `autoAck=true` 会带来什么风险？`basicNack` 的 `requeue` 参数设为 true 和 false 分别会发生什么？
10. `prefetchCount` 限制的是什么？为什么 `prefetch=1` 吞吐低？生产上一般怎么取值？
11. 管理界面上 unacked 只增不减，你的排查步骤是什么？至少说出三个可能原因。
12. 消息成为死信有哪三种情况？如何用 `x-death` 头避免无限重试循环？
13. TTL + DLX 实现延迟消息有什么致命缺陷？为什么？怎么绕过？
14. 官方延迟消息插件的原理是什么？有什么局限？什么场景不该用它？
15. 普通集群、镜像队列、quorum 队列三者的复制机制有什么不同？现在新项目应该选哪个，为什么？
16. 网络分区的三种处理策略分别是什么行为？生产推荐哪个，理由是什么？
17. 内存水位触发后会发生什么？客户端应该怎么感知和应对？
18. RabbitMQ 能保证 exactly-once 吗？幂等有哪些实现方案？各自的前提和失效场景是什么？
19. Spring 的 `defaultRequeueRejected` 默认值是什么？为什么会形成"毒丸消息"死循环？怎么解决？
20. 默认 `SimpleMessageConverter` 有什么问题？换成 Jackson 之后 `__TypeId__` 又有什么坑？
21. 消息顺序在哪些情况下会被破坏？如果业务要求严格顺序，你的方案是什么（代价是什么）？
22. 消费者"假死"（进程活着但不再消费）有哪些可能原因？怎么监控发现？
