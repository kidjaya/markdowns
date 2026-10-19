# Java 基础 · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方

## 一、核心心智模型

1. **JVM 内存结构回答的是"东西放哪"，GC 回答的是"什么时候扔"。** 这两件事相互独立——很多人把方法区和 GC 算法混着背，结果答非所问。
2. **JMM 解决的是一个矛盾：硬件想要重排序和缓存（为了快），程序员想要顺序一致性（为了对）。** happens-before 就是这份契约——它在两者之间划了一条线，线以下随便优化，线以上必须保证。
3. **并发的复杂度全部来自"共享 + 可变"。** synchronized / volatile / CAS / ThreadLocal / 不变对象，五种手段本质上都是在消除这两个条件之一。
4. **锁升级不是优化技巧，是"按竞争激烈程度付费"。** JVM 默认按最乐观的情况给最便宜的锁，只有在真的发生竞争时才逐层加码。JDK 15 撤销偏向锁的原因就是：现实里那笔"乐观税"没人付得起。
5. **集合框架的每个类都在回答一个具体问题。** 背 API 没用，要能说出"为什么 HashMap 负载因子是 0.75""为什么 ConcurrentHashMap 的 get 不用加锁"——答得出这两个，集合就通了。

## 二、必懂机制

### 2.1 JVM 运行时数据区

| 区域 | 存什么 | 是否线程私有 | OOM 场景 |
|---|---|---|---|
| 程序计数器 | 当前字节码指令地址 | 私有 | 唯一不会 OOM 的区域 |
| 虚拟机栈 | 栈帧（局部变量表 / 操作数栈 / 动态连接 / 返回地址） | 私有 | 递归过深 → `StackOverflowError`；线程开太多 → `OutOfMemoryError: unable to create new native thread` |
| 本地方法栈 | Native 方法 | 私有 | 同虚拟机栈（HotSpot 把两者合二为一） |
| 堆 | 对象实例、数组 | 共享 | `OutOfMemoryError: Java heap space` |
| 方法区（元空间） | 类元数据、常量池、静态变量、JIT 编译后代码 | 共享 | `OutOfMemoryError: Metaspace` |

**栈帧结构**四个部分要能对上号：
- 局部变量表：以 slot 为单位，long/double 占 2 个 slot；**第 0 位永远是 `this`**（实例方法）
- 操作数栈：字节码指令的工作区，`iadd` 就是从这里弹两个 int 再压回去
- 动态连接：指向运行时常量池的方法引用，分派靠它
- 方法返回地址：正常返回（PC 计数器）或异常返回（异常处理器表）

**栈上分配三件套**（`-XX:+DoEscapeAnalysis`，JDK 8 默认开启）：
- 逃逸分析：判断对象会不会被方法外引用
- 标量替换：不逃逸的对象直接拆成若干个局部变量，不分配对象头
- 同步消除：锁对象不逃逸时，`synchronized` 直接被 JIT 删掉

**直接内存 / 堆外**：
```java
ByteBuffer.allocateDirect(1024)   // DirectByteBuffer
-XX:MaxDirectMemorySize=256m      // 不设的话默认值 == -Xmx
```
- 不受 GC 管理，靠 `Cleaner`（PhantomReference 的子类）在 DirectByteBuffer 被回收时触发 `Unsafe.freeMemory`
- 典型事故：Netty 堆外泄漏、`MappedByteBuffer` 不释放、NIO 直接缓冲区分配过快 → 表现为 RSS 远超 `-Xmx`，但堆 dump 看不出问题
- 排查入口：`jcmd <pid> VM.native_memory summary`（需启动时加 `-XX:NativeMemoryTracking=detail`）

**元空间替代永久代（JDK 8）的本质原因**：永久代在堆内，大小受 `-XX:MaxPermSize` 限制，且由 GC 管理；元空间改用本地内存，上限默认是机器内存。代价是——**不设 `-XX:MaxMetaspaceSize` 就会无限涨，直到吃光系统内存**。动态类加载场景（Groovy/SpEL/CGLib/反射生成类）必设。

### 2.2 对象存活判定与 GC Roots

引用计数法不用的原因不是"慢"，是**解决不了循环引用**。

可作为 GC Roots 的对象（背这六种）：
1. 虚拟机栈（栈帧局部变量表）中引用的对象
2. 本地方法栈中 JNI 引用的对象
3. 方法区中类静态属性引用的对象
4. 方法区中常量引用的对象（如 String 常量池引用）
5. 被 `synchronized` 持有的对象
6. JVM 内部引用（基本类型对应的 Class 对象、常驻异常对象、系统类加载器）

**四种引用**：

| 类型 | 回收时机 | 典型用途 / 坑 |
|---|---|---|
| 强引用 | 永不回收 | 正常 `new` |
| 软引用 `SoftReference` | 内存不足时（即将 OOM 前）回收 | 缓存。坑：软引用对象生命期不可控，大量软引用会在内存紧张时集中回收，反而引发剧烈 GC 抖动；且 JDK 9 后 `-XX:SoftRefLRUPolicyMSPerMB` 的语义有变化 |
| 弱引用 `WeakReference` | 下次 GC 必回收 | WeakHashMap、ThreadLocalMap 的 key |
| 虚引用 `PhantomReference` | 随时，仅用于收到回收通知 | 堆外内存释放（Cleaner）、资源清理 |

`finalize()` 被弃用（JDK 9 deprecated，18 正式标记 forRemoval）的原因：执行时机不确定、可能复活对象、拖慢 GC、且只执行一次。替代是 `Cleaner` + try-with-resources。

### 2.3 三色标记与并发标记的漏标问题

三色：白（未访问）、灰（已访问但引用未扫完）、黑（已访问且引用全扫完）。

**漏标（对象消失）的两个必要条件**，必须同时成立才会出事：
1. 赋值器插入了一条从黑色对象到白色对象的新引用
2. 赋值器删除了全部从灰色对象到该白色对象的直接/间接引用

破坏任意一个即可。两种解法：

- **增量更新（Incremental Update）— CMS 用**：黑插白时把黑变灰，重新扫描一遍。代价是要重新扫描，产生"浮动垃圾"容忍度低。
- **原始快照 SATB（Snapshot-At-The-Beginning）— G1 / Shenandoah 用**：删除引用时用写屏障把旧引用记下来，当作存活。优点是不用重新扫描，缺点是会产生更多浮动垃圾（本该死的对象被当成活的留到下一轮）。

**跨代引用**问题：老年代引用新生代，Minor GC 时难道要扫整个老年代？解法是 **Remembered Set + Card Table**：
- Card Table 把老年代切成 512 字节的 card
- 写屏障（post-write barrier）在老年代对象字段赋值时把对应 card 标为 dirty
- Minor GC 只扫 dirty card 里的对象作为额外 GC Roots

### 2.4 安全点与 Stop The World

- **Safe Point**：能让线程安全暂停的位置。选在"指令序列可长期执行"的点——方法调用、循环跳转、异常跳转。
- **轮询机制**：JIT 在 safepoint poll 位置插入检查 `polling page` 是否可读的指令；GC 时把该页设为不可读触发 SIGSEGV，线程挂起。
- **坑**：`for (int i = 0; i < max; i++)` 这类**计数循环（counted loop）默认没有 safepoint poll**，循环体里若还有耗时操作，会导致 GC 等待所有线程到达 safepoint 的时间被显著拉长（表现为 GC 日志里 `Spin` 时间长）。`-XX:+UseCountedLoopSafepoints` 可强制插入（有性能代价）。
- **Safe Region**：线程处于 sleep/blocked 状态时无法响应 safepoint，此时标记自己进入安全区；离开安全区前要检查 GC 是否已完成。

### 2.5 垃圾收集器全景

| 收集器 | 算法 | 特点 / 适用 | 关键参数 |
|---|---|---|---|
| Serial / SerialOld | 复制 / 标记-整理 | 单线程，Client 模式、小堆（<100MB） | `-XX:+UseSerialGC` |
| ParNew | 复制 | Serial 的多线程版，唯一能配合 CMS 的新生代 | `-XX:+UseParNewGC` |
| Parallel Scavenge / Old | 复制 / 标记-整理 | 吞吐量优先，JDK 8 **默认** | `-XX:MaxGCPauseMillis` `-XX:GCTimeRatio=99` `-XX:+UseAdaptiveSizePolicy` |
| CMS | 标记-清除 | 低延迟，JDK 9 起废弃、14 移除 | `-XX:CMSInitiatingOccupancyFraction` `-XX:CMSFullGCsBeforeCompaction` |
| G1 | 分区 + 标记-整理 | JDK 9+ **默认**，平衡吞吐与停顿 | `-XX:MaxGCPauseMillis=200` `-XX:InitiatingHeapOccupancyPercent=45` |
| ZGC | 染色指针 + 读屏障 | 亚毫秒停顿，停顿与堆大小无关 | `-XX:+UseZGC` `-XX:+ZGenerational`（JDK 21+） |
| Shenandoah | Brooks Pointer + 读屏障 | 类似 ZGC，RedHat 主导 | `-XX:+UseShenandoahGC` |

**CMS 四阶段**：初始标记(STW) → 并发标记 → 重新标记(STW，处理增量更新) → 并发清除。
**CMS 三大缺点**：① CPU 敏感，并发阶段抢业务线程；② 浮动垃圾 + `Concurrent Mode Failure`（并发期间老年代满了）退化为 SerialOld 全量 STW，停顿极长；③ 标记-清除产生碎片。

**G1 的关键概念**：
- Region 划分（1/2/4/8/16/32MB，由 `-XX:G1HeapRegionSize` 指定或自动），每个 Region 动态扮演 Eden/Survivor/Old
- **Humongous 对象**：大小 ≥ Region 一半，直接进 Old 区的连续 Region，容易引发碎片和过早 Full GC
- Remembered Set 记录跨 Region 引用，占额外 10%~20% 内存
- Young GC（只收 Eden/Survivor）→ Mixed GC（Young + 部分 Old）→ Full GC（退化，JDK 10 前是单线程，必须避免）
- **Evacuation Failure**（to-space exhausted）：没有空 Region 可拷贝，退化为 Full GC
- Pause Prediction Model：根据历史数据选性价比最高的 Region 集合，实现 `MaxGCPauseMillis` 的软目标

**ZGC 染色指针**：把对象指针的 4 个高位 bit 用作 Marked0/Marked1/Remapped/Finalizable 标记，因此**不需要对象头存标记位**，且能并发转移对象。代价是需要虚拟地址映射（指针颜色 ≠ 真实地址），且 32 位平台不支持。

**选型决策表**：

| 场景 | 推荐 |
|---|---|
| 堆 < 4GB，JDK 8 | Parallel / CMS（视停顿要求） |
| 堆 4GB~32GB，JDK 11/17 | G1，通常不需要调参 |
| 堆 > 32GB 或停顿要求 < 10ms | ZGC（JDK 17+ 生产可用，21+ 分代 ZGC 更成熟） |
| JDK 21+ 新项目 | 直接考虑分代 ZGC（`-XX:+UseZGC -XX:+ZGenerational`，JDK 23 起分代为默认） |

### 2.6 类加载机制

七个阶段：加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载。
注意**准备阶段给静态变量赋零值**（`static int a = 1` 此时 a=0），**初始化阶段才执行 `<clinit>()`** 赋 1。

**初始化触发六种情况**（主动引用）：
`new` / 读取或设置非 final 静态字段 / 调用静态方法 / `Class.forName` 反射 / 子类初始化先触发父类 / 启动类（main 方法所在类）

**不触发的典型例子**（被动引用，面试高频）：
```java
class P { static final int V = 1; static { System.out.println("init"); } }
System.out.println(P.V);        // 不触发：常量在编译期进调用方常量池
P[] arr = new P[10];            // 不触发：数组类由 JVM 直接生成
Class.forName("P", false, cl);  // initialize=false，不触发
ClassLoader.loadClass("P");     // 只加载不链接，不触发
```

**双亲委派**：`loadClass()` 先委托父加载器，父找不到才自己加载。保证类的唯一性与安全性（防止伪造 `java.lang.String`）。

**三个破坏场景**（必须能说出为什么必须破坏）：
1. **JDBC / SPI**：`java.sql.DriverManager` 在 Bootstrap ClassLoader 里，却要加载classpath下的 `com.mysql.cj.jdbc.Driver`（父加载器看不到子的类）→ 引入**线程上下文类加载器** `Thread.currentThread().getContextClassLoader()`
2. **Tomcat**：多个 WebApp 各自的类要隔离，且不能让 WebApp 覆盖容器自身的类 → `WebAppClassLoader` 先自己加载再委托父（反序）
3. **模块化 / 热部署**（OSGi、JDK 9 JPMS、Arthas）：每个模块有独立的类加载器，需要平级的委派网而非树

**类卸载三个必要条件**：该类所有实例已回收、加载该类的 ClassLoader 已回收、该类对应的 `Class` 对象无任何引用。所以——**动态生成类的框架（CGLib、Groovy、反射生成）泄漏的根因基本都是 ClassLoader 没被回收**。

`NoClassDefFoundError` vs `ClassNotFoundException`：
- 前者是**运行时**链接阶段找不到类定义（编译时存在、运行时没了 / 静态初始化失败过一次）
- 后者是**编译/加载期** `ClassLoader.loadClass` 找不到类的受检异常

### 2.7 JMM 与并发

**happens-before 八条规则**（重点记这三条 + 传递性）：
- **volatile 规则**：对 volatile 变量的写，happens-before 后续对这个变量的读
- **程序次序规则**：同一线程内，前面的操作 happens-before 后面的
- **传递性**：A hb B，B hb C ⟹ A hb C
其余：管程锁定规则（unlock hb 后续 lock）、线程启动规则（start hb 线程内所有操作）、线程终止规则、线程中断规则、对象终结规则

**volatile 的两层语义**：
1. 可见性：写立即刷新到主存，读从主存取
2. 禁止指令重排序：通过内存屏障实现——写后加 `StoreStore` + `StoreLoad`，读前加 `LoadLoad` + `LoadStore`

**为什么 volatile 不保证原子性**：`i++` 编译成 `getfield / iconst_1 / iadd / putfield` 四条字节码，volatile 只保证每步的可见性和顺序，不保证这四步整体不被打断。

**DCL 单例为什么要 volatile**：`instance = new Singleton()` 分解后，JIT 可能把"把引用赋给 instance"重排到"调用构造函数"之前。另一个线程拿到非 null 的 instance 时对象还没初始化完。

**synchronized 的三个层次**：
- 字节码：`monitorenter` / `monitorexit`（两个 exit，一个正常一个异常路径）
- 对象头 Mark Word：`hashCode(25) | 分代年龄(4) | 偏向锁标志(1) | 锁标志位(2)`，锁标志位：`01` 无锁/偏向、`00` 轻量、`10` 重量、`11` GC 标记
- 锁升级链路：无锁 → 偏向锁（Mark Word 存线程 ID）→ 轻量级锁（线程栈建 Lock Record，CAS 抢）→ 重量级锁（膨胀为 ObjectMonitor，指向互斥量，线程挂起进入内核态）
- **JDK 15 起偏向锁默认关闭（`JEP 374` deprecated，JDK 18+ 移除）**：现代应用大量使用线程池 + 并发容器，这些场景下的同步天然存在多线程访问，偏向锁的"偏向→撤销"反而带来额外成本，收益为负

**AQS（`AbstractQueuedSynchronizer`）**：
- 核心 = `volatile int state` + FIFO 双向队列（CLH 变体）+ 模板方法
- 子类实现 `tryAcquire(int)` / `tryRelease(int)`（独占）或 `tryAcquireShared` / `tryReleaseShared`（共享）
- 抢锁失败 → 封装成 Node 入队 → `acquireQueued` 自旋/挂起 → 前驱是头节点时再次尝试
- `ReentrantLock` 公平与非公平的差异只在：非公平版在 `lock()` 时先插队 CAS 一次，抢不到才入队（吞吐更高，代价是可能饥饿）
- `Condition.await/signal` 对比 `Object.wait/notify`：Condition 可以有多个等待队列（精准唤醒）、可中断、可超时；wait/notify 只有一个等待集

**CAS 与 ABA**：`AtomicStampedReference`（版本号）/ `AtomicMarkableReference`（布尔标记）。`AtomicLong` 高并发下 CAS 争抢严重 → `LongAdder` 用**分段累加**（base + Cell[]），最终 `sum()` 求和，空间换时间。

**ThreadLocal 内存泄漏的真实根因**（必须讲准）：
```java
Thread -> ThreadLocalMap -> Entry(WeakReference<ThreadLocal> key, Object value)
```
- key 是弱引用：ThreadLocal 对象被回收后，key 变 null
- **value 是强引用**：key 为 null 的 Entry 里的 value 仍然可达（只要线程活着）
- 线程池里的线程永不销毁 → value 永久堆积 → 泄漏
- 解法：**用完必须 `remove()`**。ThreadLocalMap 自身也有 `expungeStaleEntry` 兜底清理，但只在特定操作时触发，靠不住
- 父子线程传递用 `InheritableThreadLocal`（创建子线程时拷贝，线程池场景下失效，因为线程已存在）→ 线程池场景用阿里 `TransmittableThreadLocal`（TTL）

**ThreadPoolExecutor 七大参数**：
`corePoolSize` / `maximumPoolSize` / `keepAliveTime` / `unit` / `workQueue` / `threadFactory` / `handler`

提交流程：core 未满 → 建核心线程；core 满 → 入队列；队列满 → 建非核心线程；max 满 → 拒绝策略。

四种拒绝策略：`AbortPolicy`（抛异常，默认）、`CallerRunsPolicy`（调用方自己跑，变相反压）、`DiscardPolicy`（静默丢弃）、`DiscardOldestPolicy`（丢队头）。

**`execute` vs `submit` 的致命差异**：
```java
executor.execute(() -> { throw new RuntimeException("boom"); });  // 打到 UncaughtExceptionHandler，日志可见
executor.submit(() -> { throw new RuntimeException("boom"); });   // 异常被吞进 Future，不 get() 就永远看不见
```
这是生产环境最常见的"任务静默失败"根因。要么用 `execute`，要么 submit 后一定 `future.get()` 并处理 `ExecutionException`。

**为什么不用 `Executors.newFixedThreadPool`**：`LinkedBlockingQueue` 无界（默认 `Integer.MAX_VALUE`），任务堆积会 OOM。`newCachedThreadPool` 的 maximumPoolSize 是 `Integer.MAX_VALUE`，会无限建线程。阿里巴巴 Java 开发手册强制要求手写 `new ThreadPoolExecutor(...)`。

**`shutdown()` vs `shutdownNow()`**：前者不再收新任务但跑完队列里的；后者尝试 `interrupt()` 正在执行的任务并返回队列里未执行的列表（注意：interrupt 只能中断响应中断的阻塞，跑在计算循环里的任务停不下来）。

**CompletableFuture 常用组合**：
`thenApply`（同步转换）/ `thenApplyAsync`（换线程）/ `thenCompose`（扁平化，避免 `CompletableFuture<CompletableFuture<T>>`）/ `thenCombine`（两个结果合并）/ `allOf`（全完成）/ `anyOf`（任一完成）
异常处理：`exceptionally`（只捕获异常）/ `handle`（正常异常都进）/ `whenComplete`（类似 finally，不改变结果）
坑：`thenApply` 默认复用**上一个任务完成的那个线程**，如果上一步是 IO 线程，会把 IO 线程拖住；默认异步用公共 `ForkJoinPool.commonPool()`

### 2.8 集合框架

**HashMap**：数组 + 链表 + 红黑树。
```java
static final int hash(Object key) {   // 高位扰动
    int h; return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
- 为什么 `h ^ (h >>> 16)`：把高 16 位的特征混入低 16 位，因为桶下标是 `(n-1) & hash`，n 小时只有低位参与运算，不扰动会大量碰撞
- **容量为 2 的幂**的两个目的：① `& (n-1)` 替代 `% n` 更快；② 扩容时元素的新位置只取决于 hash 的那个新增高位 bit（要么原位，要么原位 + oldCap），无需重新计算
- 负载因子 0.75：泊松分布下，桶中元素超过 8 个的概率约为千万分之六，是空间与冲突概率的折中
- `TREEIFY_THRESHOLD=8` / `UNTREEIFY_THRESHOLD=6`（留 1 的差值防止反复转换）/ `MIN_TREEIFY_CAPACITY=64`（容量不足 64 时优先扩容而非树化）
- **JDK 7 vs 8 扩容的差异**：7 用头插法，多线程扩容会形成环形链表，`get()` 死循环（CPU 100%）；8 改尾插法解决了死循环，但仍会丢失数据——**HashMap 从头到尾都不是线程安全的**

**ConcurrentHashMap**：
- JDK 7：`Segment` 分段锁，默认 16 段，理论上 16 个线程并发写
- JDK 8：摒弃 Segment，改为**数组 + 链表/红黑树，每个 bin 独立加锁**：空桶用 CAS 插入，非空桶用 `synchronized` 锁住头节点。锁粒度从"段"降到"桶"
- `size()` 返回 int 可能溢出，用 `mappingCount()`（返回 long）
- **为什么 `get()` 不加锁还安全**：① `Node.val` 和 `next` 都是 `volatile`；② 扩容时的 ForwardingNode 标记能指导读线程去新数组查找
- 多线程扩容：`transfer()` 支持多个线程按 stride 分片协同迁移

**其他高频坑**：
- `Arrays.asList(arr)` 返回的是 `Arrays.ArrayList`（内部类），`add/remove` 抛 `UnsupportedOperationException`；基本类型数组会整体当一个元素 → `Arrays.asList(int[])` 返回 `List<int[]>`
- `subList()` 返回的是原 List 的视图，持有原引用 → 原 List 太大时会内存泄漏；且对子列表的 `add` 会反映到原列表
- `CopyOnWriteArrayList`：写时复制整个数组，适合**读极多写极少**（如监听器列表）；写入代价是 O(n) 内存 + CPU
- `PriorityQueue` 是二叉堆，不是 FIFO，遍历不保证有序
- `LinkedHashMap(16, 0.75f, true)` 按访问顺序 → 天然 LRU，重写 `removeEldestEntry` 即可定长缓存

### 2.9 异常与 IO

- **异常的开销主要在 `fillInStackTrace()`**：异常构造时要遍历整个栈生成 `StackTraceElement[]`，栈越深越慢。高频路径（如流控拒绝）可重写 `fillInStackTrace()` 返回 this，或用 `-XX:-StackTraceInThrowable`（这个参数会让所有异常没有栈，慎用）
- try-with-resources 编译后是 `try-finally` + `addSuppressed()`，能保留关闭时的次要异常
- 受检异常的设计意图是"调用方必须处理"；但过度使用会让签名污染（这也是 Kotlin/Scala 全部设为非受检的理由之一）

**IO 模型**（必须分清两组正交概念）：
| | 阻塞 | 非阻塞 |
|---|---|---|
| **同步** | BIO（read 一直等到数据） | NIO 多路复用（select/epoll 等就绪通知，read 本身立即返回） |
| **异步** | — | AIO（注册回调，内核完成数据拷贝后主动通知） |

- **Linux 上 AIO 长期不完善**（glibc 的 `io_submit` 仅支持 O_DIRECT），所以 Netty 在 Linux 上仍是 NIO（epoll），只在 Windows 上用 IOCP
- NIO 三件套：`Buffer`（position/limit/capacity，`flip()` 切换读写，`compact()` 保留未读部分）+ `Channel` + `Selector`
- **Selector 注册时机坑**：`OP_WRITE` 几乎总是就绪的，如果一直注册 `OP_WRITE` 会导致 `select()` 立即返回 → CPU 100%。正确做法是只在有数据要写时才注册，写完立即取消
- 零拷贝：`FileChannel.transferTo()`（对应 `sendfile`）vs `MappedByteBuffer`（mmap）。`MappedByteBuffer` 的释放依赖 GC，文件在 `close()` 后仍可能被占用（Windows 上尤其明显，无法删除文件）
- **JDK 序列化必须换成 JSON/Protobuf/Kryo**：① 性能与体积差（JDK 序列化带完整类元数据）；② 强耦合 `serialVersionUID`；③ **反序列化漏洞（没有白名单就是 RCE）**。`transient` 字段不参与序列化

### 2.10 面向对象与语法细节

**String 不可变的原因**（四层）：
1. `private final char[] value`（JDK 9 起是 `byte[]` + `coder`）
2. 不提供任何修改 value 的方法，且类本身 `final`（防止子类破坏）
3. hash 可缓存（`private int hash`，只需算一次）
4. **安全考量**：String 作为参数传递时（类名、URL、文件路径），不可变保证没有人能在背后改掉它；常量池复用也依赖不可变

**字符串常量池位置变迁**：JDK 6 在永久代 → JDK 7 移到堆 → JDK 8 元空间后仍在堆。
`intern()` 行为差异：JDK 6 是**复制**一份到常量池并返回新引用；JDK 7+ 是**记录**堆中已有对象的引用，返回同一个对象。所以经典面试题 `new String("a") + new String("b")` 再 `intern()` 在 7 和 6 上结果不同。

**JDK 9 Compact Strings**：`value` 从 `char[]`（2 字节）改为 `byte[]` + `coder` 标志位。纯 Latin1 字符只用 1 字节，节省约一半内存。`-XX:-CompactStrings` 可关闭。

**equals 与 hashCode 契约**：equals 相等 ⟹ hashCode 必须相等；hashCode 相等 ⟹ equals 不一定相等。**只重写 equals 不重写 hashCode 会导致 HashMap 里放进去取不出来**。

**Integer 缓存**：`-128~127`，由 `IntegerCache.high` 控制（可用 `-XX:AutoBoxCacheMax=N` 调大）。`Integer a=127, b=127; a==b` 为 true，`128` 则为 false。**注意 `Integer` 与 `int` 用 `==` 比较时会自动拆箱**，此时比较的是值。

**接口默认方法与抽象类的取舍**：需要状态（成员变量）或构造逻辑用抽象类；需要多继承行为或兼容老接口用默认方法。JDK 8 默认方法的"类优先"原则：类的方法优先级高于接口默认方法，冲突时必须显式 `X.super.method()`。

**Lambda 的真相**：不是匿名内部类的语法糖。编译期生成 `invokedynamic` 指令，运行时由 `LambdaMetafactory.metafactory()` 动态生成实现类。证据：
- 匿名内部类编译后生成 `Outer$1.class`，Lambda **不生成** class 文件
- Lambda 里的 `this` 指向外部类（匿名内部类指向自身）
- 栈里的方法名形如 `lambda$main$0`

**Stream 的坑**：
- `parallelStream()` 共用公共 `ForkJoinPool.commonPool()`（默认并行度 = CPU 核数 - 1），一个慢任务会拖垮整个 JVM 里所有 parallelStream。IO 密集必须用自定义 ForkJoinPool 并提交进去
- `peek()` 设计用于 debug，若流没有终止操作（如 `collect`），`peek` 根本不会执行
- `Optional` 误用：`Optional` 不要作为字段或方法参数（无法序列化），只用于返回值

### 2.11 JDK 版本演进必须知道的

| 版本 | 关键变化 |
|---|---|
| 8 | Lambda / Stream / 默认方法 / 元空间取代永久代 / `CompletableFuture` / 时间 API |
| 9 | 模块化 JPMS（**破坏反射访问 JDK 内部类**）/ G1 默认 / Compact Strings / `List.of()` |
| 10 | `var` 局部变量类型推断 |
| 11 (LTS) | **移除 Java EE / CORBA 模块**（JAXB、JAX-WS 从 JDK 剥离 → 老项目升级必遇 `NoClassDefFoundError: javax/xml/bind/JAXBException`，要手动加 `jaxb-api` 依赖）/ ZGC 实验 / HTTP Client 正式 |
| 14 | 移除 CMS / `NullPointerException` 友好提示（`-XX:+ShowCodeDetailsInExceptionMessages`，15 起默认） |
| 15 | 偏向锁默认关闭 / 文本块正式 / ZGC 生产可用 |
| 16 | `record` / `instanceof` 模式匹配 |
| 17 (LTS) | `sealed` 类 / 移除 Applet / 强封装 JDK 内部 API |
| 21 (LTS) | **虚拟线程正式**（JEP 444）/ 分代 ZGC / 记录模式 / switch 模式匹配 |

**容器场景**：JDK 8u191 之前 JVM 认不到 cgroup 限制，会按宿主机核数/内存设置默认堆和 GC 线程数 → 容器里 OOM。
解决：`-XX:+UseContainerSupport`（8u191+ 默认开启），用 `-XX:MaxRAMPercentage=75.0` 替代硬编码 `-Xmx`。

**虚拟线程（Virtual Threads）**：
- 实现：JVM 管理的轻量线程，底层是 `Continuation`，由 `ForkJoinPool`（carrier threads）调度，阻塞时自动挂起并让出 carrier
- 适用：IO 密集（每个请求一个虚拟线程，写法像同步代码但性能接近异步）
- **不适合 CPU 密集**（没有并行加速）
- **不要池化**：创建成本极低，池化反而会限制并发度且带来 ThreadLocal 泄漏
- **Pinning 问题**：虚拟线程在 `synchronized` 块内阻塞时无法卸载，会钉住 carrier thread。JDK 21 的主要来源就是 `synchronized`；JDK 24（JEP 491）改为允许在 `synchronized` 中卸载。当前版本建议把 `synchronized` 换成 `ReentrantLock`

### 2.12 排障工具箱

```bash
jps -l                                    # 找 pid
jstat -gcutil <pid> 1000                  # 每秒打一次 GC，看 O/M 列是否持续上涨、YGC/FGC 频率
jmap -histo:live <pid> | head -30         # 存活对象按类统计（会触发一次 Full GC）
jmap -dump:format=b,file=heap.hprof <pid> # 堆 dump
jstack <pid> > /tmp/stack.txt             # 线程栈，多打几次对比
jcmd <pid> Thread.print                   # 等价 jstack（推荐，jcmd 是官方主推）
jcmd <pid> VM.native_memory summary       # 堆外内存（需 -XX:NativeMemoryTracking=detail）
jcmd <pid> VM.flags                       # 查看生效的 JVM 参数
jinfo -flags <pid>                        # 同上，还能动态改部分 manage 参数
```

**启动参数建议默认加上**：
```
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:NativeMemoryTracking=summary
-Xlog:gc*:file=/data/logs/gc.log:time,uptime:filecount=10,filesize=100m   # JDK 9+
```

**Full GC 频繁的标准排查路径**：
1. `jstat -gcutil` 看老年代占用是否下不来（下不来 = 泄漏，能下来 = 容量不足）
2. 看 GC 日志里 `Pause Full (Allocation Failure)` 还是 `(System.gc())` 还是 `(Metadata GC Threshold)`
3. dump 堆用 MAT 打开 → Histogram 按 Retained Heap 排序 → 看谁持有一大片 → Path to GC Roots
4. 若堆正常但 RSS 高 → 查堆外：NMT、pmap -x、DirectByteBuffer 数量、glibc malloc arena（`MALLOC_ARENA_MAX=4`）

**Arthas 高频命令**：
`dashboard`（总览）/ `thread -n 3`（最忙的 3 个线程）/ `thread -b`（**直接找死锁**）/ `watch com.x.Service method "{params, returnObj}" -x 3`（观测入参返回值）/ `trace`（方法内部耗时）/ `jad`（反编译确认线上代码）/ `ognl`（调用静态方法、查看 Spring 容器里的 Bean）/ `mc` + `redefine`（热替换）

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 线程池任务静默消失 | 用了 `submit()`，异常被封进 `Future`，没 `get()` 永远看不到 | 改用 `execute()`，或统一包装 `try-catch` 打日志；`submit` 后务必 `get()` |
| 容器里 OOM 但 `-Xmx` 设得很小 | JDK 8u191 前认不到 cgroup；或堆外（Metaspace / DirectBuffer / 线程栈 / glibc）占了大头 | 升到 8u191+；用 `MaxRAMPercentage`；加 NMT 查堆外 |
| 接口偶发拿到未初始化完的对象 | DCL 单例没加 `volatile`，指令重排导致引用先于构造赋值 | 单例字段加 `volatile`；或改用静态内部类 / 枚举单例 |
| 用户身份串号 | ThreadLocal 未 `remove()`，线程池复用线程带上了上一个请求的 value | `finally` 里 `remove()`；跨线程池传递用 TTL |
| 异步任务拿不到 traceId / 用户信息 | `InheritableThreadLocal` 只在**创建**子线程时拷贝，线程池线程早就存在了 | 用 `TransmittableThreadLocal` + TTL 的线程池包装 |
| HashMap 在多线程下 CPU 100% | JDK 7 头插法 resize 形成环形链表，`get()` 死循环 | 升级 JDK 8（尾插法解决死循环）；并发场景一律用 `ConcurrentHashMap` |
| Metaspace 持续上涨直至 OOM | 动态类生成（CGLib / Groovy / 反射 / Lambda 大量）导致 ClassLoader 无法卸载 | 设 `-XX:MaxMetaspaceSize`；排查 ClassLoader 泄漏（MAT 看 `ClassLoader` 的 Retained Heap） |
| 堆 dump 看不出问题但 RSS 很高 | 堆外内存泄漏：Netty `ByteBuf` 未释放、`MappedByteBuffer`、JNI、glibc arena | `jcmd VM.native_memory summary`；`pmap -x <pid>` 找大块匿名内存；Netty 开 `-Dio.netty.leakDetection.level=advanced` |
| `parallelStream` 拖垮整个服务 | 共用 `ForkJoinPool.commonPool()`，慢任务占满公共池 | 自定义 `ForkJoinPool` 并提交；或改用 `CompletableFuture` + 业务线程池 |
| 升级 JDK 11 后启动报 `NoClassDefFoundError: javax/xml/bind` | JDK 11 移除了 Java EE 模块（JAXB、JAX-WS、CORBA） | 手动加 `jakarta.xml.bind-api` + `jaxb-runtime` 依赖 |
| GC 停顿莫名很长，日志里有大量 Spin 时间 | 计数循环没有 safepoint poll，线程迟迟无法到达安全点 | `-XX:+UseCountedLoopSafepoints`（有吞吐代价）；或把大循环拆小 |
| 加了 `synchronized` 性能反而更差 | 偏向锁撤销开销 > 同步本身收益（JDK 15 前，线程池场景） | 升级到 JDK 15+；或改用 `ReentrantLock` / 无锁结构 |
| `subList` 之后 OOM | `subList` 是原 List 的视图，持有整个原数组的强引用 | `new ArrayList<>(list.subList(...))` 拷贝一份 |
| `Arrays.asList(arr).add()` 抛异常 | 返回的是 `Arrays` 内部固定大小 List | 包一层 `new ArrayList<>(Arrays.asList(arr))` |
| 递归/深层调用 `StackOverflowError` | 虚拟机栈深度不够（默认 1MB 约对应几千到上万帧） | `-Xss` 加大；但根本上是改掉递归或用显式栈 |
| Netty `OP_WRITE` 一直注册导致 CPU 100% | `OP_WRITE` 在缓冲区可写时几乎总是就绪，`select()` 空转 | 仅在实际有数据待写时才注册 `OP_WRITE`，写完立刻取消 |
| 虚拟线程没有带来吞吐提升 | 任务在 `synchronized` 里阻塞导致 pinning，carrier 被钉住 | 换成 `ReentrantLock`（JDK 24 前）；或升级 JDK 24+（JEP 491） |
| `Integer` 用 `==` 比较结果诡异 | 超过 `IntegerCache` 范围（-128~127）后 `==` 比较的是引用 | 一律用 `equals()` 或 `Objects.equals()` |

## 四、往下钻的入口

**OpenJDK 源码**（按重要性排序）：
- `java.util.concurrent.AbstractQueuedSynchronizer` —— `acquire()` / `acquireQueued()` / `shouldParkAfterFailedAcquire()`，AQS 全貌在这三个方法里
- `java.util.concurrent.ThreadPoolExecutor.execute()` —— 提交流程的 30 行就是七大参数的全部语义
- `java.util.HashMap.resize()` —— JDK 8 尾插 + 高低位拆分的完整实现
- `java.util.concurrent.ConcurrentHashMap.putVal()` / `get()` —— 看 CAS + synchronized per-bin 与 ForwardingNode
- `java.lang.ThreadLocal.ThreadLocalMap` —— `expungeStaleEntry` 的清理逻辑，看完就懂为什么要 `remove()`
- `java.lang.String` / `AbstractStringBuilder` —— Compact Strings 的 `coder` 分支
- `java.lang.invoke.LambdaMetafactory` —— Lambda 的实现机制
- HotSpot：`src/hotspot/share/gc/g1/g1CollectedHeap.cpp`、`src/hotspot/share/oops/markWord.hpp`（Mark Word 位布局）

**官方文档具体章节**：
- JVM 规范 §2.5 Run-Time Data Areas、§6.5 字节码指令集
- JLS §17 Threads and Locks（JMM 的正式定义，happens-before 原文）
- JEP 索引：333(ZGC)、374(弃用偏向锁)、444(虚拟线程)、491(虚拟线程同步改进)
- Oracle 的 *Java Garbage Collection Basics*（G1 的官方入门）

**书籍**：《深入理解 Java 虚拟机》周志明（第 3 版，对应 JDK 11）；《Java 并发编程实战》

## 五、达标自检

1. 对象从创建到被回收，中间在 JVM 里经过了哪些区域？哪些情况下会发生栈上分配？
2. Minor GC 时如何避免扫描整个老年代？Card Table 的写屏障发生在什么时刻？
3. 三色标记为什么会漏标？CMS 和 G1 分别用哪招解决？为什么 G1 的浮动垃圾更多？
4. G1 的 Mixed GC 和 Full GC 分别在什么条件下触发？`to-space exhausted` 意味着什么？
5. ZGC 的染色指针解决了什么具体问题？为什么它能做到停顿与堆大小无关？
6. 说出至少 6 种 GC Roots。
7. 类加载的七个阶段里，"准备"和"初始化"分别在做什么？`static final int A = 1` 在哪个阶段被赋值？
8. 举出三个不触发类初始化的例子，并说明每个为什么不触发。
9. 为什么 JDBC 必须破坏双亲委派？线程上下文类加载器在其中扮演什么角色？
10. 类卸载的三个必要条件是什么？为什么说 CGLib 泄漏本质上是 ClassLoader 泄漏？
11. `volatile` 加在什么位置能禁止指令重排序？DCL 单例不加 volatile 具体会出什么事？
12. 完整描述锁升级的四个阶段，每个阶段对象头里存了什么。JDK 15 为什么关掉偏向锁？
13. AQS 的 `state` 和队列分别解决什么问题？ReentrantLock 的公平与非公平实现差在哪一行？
14. ThreadLocal 内存泄漏中，key 和 value 分别是什么引用？为什么弱引用反而成了泄漏的帮凶？
15. `submit()` 和 `execute()` 在异常处理上的差异是什么？生产上你怎么保证任务失败可感知？
16. 为什么 `HashMap` 容量必须是 2 的幂？扩容时元素新位置怎么算出来的？
17. ConcurrentHashMap 在 JDK 7 和 8 的锁机制有何不同？为什么它的 `get()` 不加锁还线程安全？
18. `h ^ (h >>> 16)` 这行代码解决了什么问题？如果去掉会怎样？
19. 阻塞/非阻塞与同步/异步是两组正交概念吗？epoll 属于哪一种？Linux 上的 AIO 为什么没被 Netty 采用？
20. 虚拟线程在什么情况下会被 pinning？你的项目如果要引入，需要检查哪些代码？
