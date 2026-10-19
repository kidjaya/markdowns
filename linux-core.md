# Linux · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
> 视角：这是 Java 后端排查线上问题的最后一百米，重点是"SOP 化排查"而非命令大全

## 一、核心心智模型

1. **一切皆文件，进程的一切都在 /proc 下。** `/proc/<pid>` 是内核实时生成的进程视图：cmdline、environ、fd/（打开的文件描述符）、limits（资源限制）、status（内存/线程数）、stack、cgroup、oom_score。**排查问题的第一手材料几乎全在这儿**，比任何第三方工具都准。

2. **普通 ALARM 的三大顺序错觉：先看现象归类再上工具。** CPU 高、内存涨、IO 卡、网络慢，这四类问题的排查路径完全不同。上来就 `top` 然后无目的地翻，是在浪费时间。正确顺序是：**先判断资源类型 → 再定位到进程 → 再定位到线程/代码**。

3. **内存相关的直觉大多是错的。** "free 显示的空闲少了"≠内存不够（Linux 会用空闲内存做 page cache，这是收益不是浪费）；"cached 很大"≠需要清理；真正该看的是 `available` 和是否有 swap in/out（si/so）。而当 swap 开始动的时候，数据库类服务基本已经在崩了。

4. **理解阻塞/非阻塞 与 同步/异步是两个正交维度。** "非阻塞"指调用不等着返回；"同步"指数据就绪后要不要自己拷。**select/poll/epoll 是同步非阻塞**——数据到了还得你自己 read 拷；真正的异步是 io_uring/IOCP，内核拷完通知你。网上大量资料在这里含糊不清，面试也常考。

5. **LOAD 高不代表 CPU 高。** load average 统计的是"运行队列里的任务数 + 不可中断睡眠(D状态)的任务数"。所以 Load 高 CPU 低，通常意味着**卡在 IO 或锁**——这是排查的一个关键分叉点。

## 二、必懂机制

### 2.1 快速判明瓶颈类型（USE 方法）

| 维度 | 看什么 | 指标异常的含义 |
|---|---|---|
| CPU 使用率 | `top` 的 us/sy/ni/id/wa/hi/si/st | us 高=应用；sy 高=内核/系统调用多；wa 高=等 IO；si 高=软中断（网络包多）；**st 高=被宿主机偷走（云上超卖）** |
| CPU 饱和度 | load average、run queue (vmstat 的 r 列) | r > CPU 核数说明排队 |
| 内存 | `free` 的 available、`vmstat` 的 si/so | si/so 不为 0 说明在换页，危险信号 |
| 磁盘 IO | `iostat -x 1` 的 %util、await、r/s、w/s | await 高=排队；%util 接近 100% 说明设备饱和（注意 SSD 上 %util 高不一定代表带宽用满，因为它是"有请求在处理的比例"） |
| 网络 | `sar -n DEV`、`ss -s`、丢包计数 | RX drop/errors 增长说明网卡处理不过来 |

```bash
top              # 按 1 看每核；按 P 按 CPU 排；按 M 按内存排；关注 wa 和 st
vmstat 1         # r=运行队列  b=D状态不可中断  si/so=换页  us/sy/id/wa
mpstat -P ALL 1  # 每核使用分布，看是否有单核打满（常见于某个死循环线程）
iostat -x 1      # r/s w/s await svctm %util
sar -n DEV 1     # 网络吞吐
sar -r 1         # 内存
sar -q 1         # load 与运行队列
pidstat -p <pid> 1 -udr    # 单进程的 CPU/磁盘/内存综合（非常实用）
```

### 2.2 CPU 飙高定位 SOP（必会）

```bash
# 1) 找进程
top -c
# 2) 找线程（注意 -H 是线程视图）
top -Hp <pid>
# 3) 把线程 ID 转成十六进制（jstack 里 nid 是 16 进制）
printf '%x\n' 12345
# 4) 抓栈
jstack <pid> | grep -A 30 'nid=0x3039'
# 5) 反复抓几次对比（一次抓可能是巧合）
for i in {1..3}; do jstack <pid> > /tmp/j$i.txt; sleep 1; done
```
如果是本机 && JDK 8，用 `jstack -F` 或 `-m` 混排栈；JDK 11+ 可以直接 `jcmd <pid> Thread.print`。

常见 CPU 高的代码型原因：
- **死循环**（尤其是 `while(true)` + 没有阻塞的 IO 就绪轮询）
- **正则回溯**（灾难性回溯，CPU 100% 且不出结果）
- **HashMap 被并发 put 导致链表成环**（JDK7，现已基本不会见）
- **GC 线程忙**（这时候要看 `jstat -gcutil`，而不是找业务线程）
- **Netty 的 NIO selector 空轮询 bug** 的变体（注册了 OP_WRITE 但没写完，一直就绪）

```bash
jstat -gcutil <pid> 1000 10     # 看 GC：如果 FGCT 持续涨且 FGC 频繁，是 GC 导致的 CPU 高
```

### 2.3 内存排查 SOP

```bash
free -h                # 重点看 available，不要被 free 那一列吓到
cat /proc/meminfo      # 更全：Slab / PageTables / HugePages / AnonPages
vmstat 1               # si/so 是否 > 0
ps -eo pid,rss,vsz,comm --sort=-rss | head -20    # 谁占得多
pmap -x <pid> | tail -5                            # 进程内存分布
cat /proc/<pid>/status                             # VmRSS / VmSize / Threads
```
关键区分：**OOM 到底谁干的？**
- **JVM 自身的 OOM**（heap/metaspace/direct buffer/GC overhead）→ 日志里会有 `java.lang.OutOfMemoryError`，配合 `-XX:+HeapDumpOnOutOfMemoryError` 留 dump
- **被系统 OOM Killer 杀掉** → `dmesg -T | grep -i 'killed process'`，看 `/proc/<pid>/oom_score_adj`

```bash
dmesg -T | grep -iE "oom|killed|out of memory"
journalctl -k | grep -i oom
```
Java 侧进一步定位：
```bash
jmap -histo:live <pid> | head -30    # 对象直方图（会触发 full GC，生产慎用）
jmap -dump:format=b,file=/tmp/h.bin <pid>   # dump（暂停应用，务必在摘流量后做）
jcmd <pid> VM.native_memory summary         # 堆外内存（NMT），需要启动参数 -XX:NativeMemoryTracking=detail
jstat -gccapacity <pid>                     # 各代容量
```
> **注意**：容器被 OOMKill 时，`java heap dump 通常拿不到`，因为进程直接被 -9 干掉，shutdown hook 不执行。所以要用 HeapDumpOnOutOfMemoryError 针对 JVM OOM，用 cgroup memory limit + 监控 针对容器 OOM。

### 2.4 Load 高 CPU 低：这是 IO 或锁

```bash
vmstat 1          # b 列 > 0 = 有进程卡在不可中断状态（通常是等 IO）
iostat -x 1       # await 高 / %util 高
jstack <pid> | grep -B2 -A5 "BLOCKED\|waiting to lock"    # 找锁竞争
```
一个特别实用的 Diag Path：
- `b > 0` 且 `wa` 高 → 磁盘 IO 瓶颈 → `iotop` / `pidstat -d` 找到具体进程
- `b > 0` 但 `wa` 低 → 可能是网络文件系统（NFS）或 fuse，或者**大量线程卡在同一把锁**
- `r > 核数` 但 CPU 不高 → 也可能在等锁，只是线程还能调度

### 2.5 磁盘与文件

```bash
df -h                     # 空间
df -i                     # inode 使用（inode 满会让"No space left on device"但 df -h 显示还有空间！）
du -sh /path/* | sort -rh | head -20    # 找大目录
iotop                     # 谁在写磁盘
pidstat -d 1
lsof | grep deleted       # 最关键：已被删除但仍被进程占用的文件（空间不释放的真凶）
```
**经典场景**：日志文件被 `rm` 删了，`df` 却没释放 → 进程还持有 fd → 重启进程或 `truncate` 该文件
```bash
> /path/to/logfile    # 正确的"清空日志"姿势（truncate），而不是 rm
```

| 概念 | 说明 |
|---|---|
| inode | 存文件元信息（权限、大小、block 指针、时间戳），**不含文件名**。文件名在目录项里。inode 耗尽时无法新建文件，即使磁盘还有空间 |
| 硬链接 | 多个目录项指向同一 inode。**不能跨文件系统、不能链目录**。只有所有硬链接都被删且无进程占用时数据才真正释放 |
| 软链接(符号链接) | 独立文件，内容是目标路径。可跨文件系统、可链目录，**目标删除后成为悬空链接** |
| VFS | 虚拟文件系统层，屏蔽 ext4/xfs/nfs 差异 |
| ext4 vs xfs | xfs 在大文件与高并发下通常更好，且 xfs 的延迟分配特性可能导致"看起来写入成功但宕机丢失"；ext4 更普及、缩容方便（xfs 不能缩容） |

### 2.6 IO 模型（面试高频，必须分清）

```
阻塞 vs 非阻塞：调用是否立刻返回（针对应用程序等待数据就绪的过程）
同步 vs 异步    ：数据就绪后是自己拷贝还是 OS 拷完通知你
```
| | 阻塞 | 非阻塞 |
|---|---|---|
| **同步** | read/write（BIO） | select/poll/epoll + read/write（NIO，**Java NIO / Netty 属于这类**） |
| **异步** | — | Linux AIO（不完善，Java AIO 在 Linux 上底层仍是 epoll）、io_uring、Windows IOCP |

**epoll vs select/poll**：
- select：fd_set 位图，**默认上限 1024**（FD_SETSIZE），每次调用要把整个集合从用户态拷到内核态，返回后还要 O(n) 遍历找就绪
- poll：改用数组突破了 1024，但拷贝与遍历问题依旧
- epoll：`epoll_ctl` 注册一次，`epoll_wait` 返回**就绪链表**（无需遍历）；底层是红黑树（存注册的 fd）+ 就绪双向链表。**没有 fd 数量硬限制**（受 `/proc/sys/fs/file-max`）

**epoll LT vs ET**：
- LT（水平触发，默认）：只要缓冲区还有数据就一直通知，可以不一次读完。安全但可能重复唤醒
- ET（边缘触发）：只在状态变化时通知一次，**必须一次把数据读完直到 EAGAIN**，否则剩余数据再也不会被通知 —— Netty 用的就是 ET（配合 `#allocateDirect` 与循环读）

**零拷贝两条路**：
```
mmap/write:  磁盘 → page cache →(mmap映射到用户态，省一次拷贝)→ socket buffer → 网卡    （4次上下文，3次拷贝）
sendfile:    磁盘 → page cache → socket buffer → 网卡   （2次上下文，用户态完全不参与，2次拷贝）
（带 DMA gather 的 sendfile 可做到：磁盘→page cache→网卡描述符，真正 1 次拷贝）
```
Kafka/RocketMQ 的高吞吐很大程度依赖 sendfile + mmap。

### 2.7 网络排查

```bash
ss -antp                     # 替代 netstat（更快）
ss -s                        # 总体统计（TIME_WAIT/estab 数量）
ss -antp state time-wait | wc -l
ss -antp | grep ESTAB | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
sar -n DEV 1                 # 吞吐
sar -n TCP,ETCP 1            # 重传、连接失败统计（activeopens/passiveopens/retrans）
sar -n SOCK 1                # socket 使用量
ethtool eth0                 # 网卡速率与双工/协商（半双工是古老但致命的问题）
ip -s link                   # **RX dropped / errors / overruns** —— 网卡丢包第一现场
nstat -z | head -20
cat /proc/net/dev
tcpdump -i any host x.x.x.x and port 8080 -nn -w /tmp/x.pcap
```

conntrack 表满（NAT/LB 场景经典）：
```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
dmesg | grep "nf_conntrack: table full"    # 满了会丢包，表现为随机连接失败
```

句柄限制：
```bash
ulimit -n                       # 当前 shell
cat /proc/<pid>/limits          # 进程真实限制（最准）
ls /proc/<pid>/fd | wc -l       # 已用 fd 数
```
> **systemd 陷阱**：`/etc/security/limits.conf` 对 systemd 启动的服务无效！必须改 service 文件：
```
[Service]
LimitNOFILE=65535
```

### 2.8 systemd 与信号

```ini
[Unit]
Description=my app
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/app
ExecStart=/usr/bin/java -Xms2g -Xmx2g -jar app.jar
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure          # always / on-abnormal
RestartSec=10
LimitNOFILE=65535
Environment="JAVA_OPTS=-Dxxx"

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload && systemctl enable --now app
journalctl -u app -f --since "10 min ago"
```

**信号对 Java 的影响（很重要）**：

| 信号 | Java 行为 |
|---|---|
| SIGTERM (15) | **触发 Shutdown Hook**，优雅停机（Spring 的 ContextClosedEvent、Dubbo 的注销）。`systemctl stop` 默认发这个 |
| SIGINT (2) | 同 TERM |
| SIGQUIT (3) | **JVM 打印所有线程栈到 stdout**（相当于 jstack），然后退出。Nginx 用它 reopen 日志，但对 Java 是 thread dump + 退出 —— 别乱发 |
| SIGHUP (1) | 默认行为也是触发退出（有些应用用它重载配置） |
| SIGKILL (9) | **不执行任何 hook**，直接死。**"kill -9 之后注册中心还有注册信息残留" 就是这么来的** |
| SIGUSR1/SIGUSR2 | JVM 未占用，部分框架（如 Logback）用它触发配置重载 |

```bash
jcmd <pid> VM.version
kill -3 <pid>    # 抓线程栈到 stdout 容器日志里（容器里没 jstack 时的应急手段）
kill -3 <pid> && docker logs <container>    # 组合拳
```

### 2.9 Shell 相关

**环境变量加载顺序**（跨平台最经典的坑）：
```
登录 shell（ssh 登录）：/etc/profile → /etc/profile.d/*.sh → ~/.bash_profile → ~/.bashrc → /etc/bashrc
非登录 shell（bash -c / docker exec bash）：只加载 ~/.bashrc（且交互非登录 shell 才是）
  → 结论：写在 /etc/profile.d/xxx.sh 里最稳妥；或在 systemd/crontab 里显式 source
```
**cron 的经典问题**：环境变量极少（PATH 只有 /usr/bin:/bin），必须写绝对路径；`%` 要转义；输出会发邮件（或直接重定向到 /dev/null）。

```bash
set -euo pipefail        # 写脚本的默认开头：出错即退出、未定义变量报错、管道中任一失败算失败
$(cmd)                   # 优于反引号，可嵌套
awk '{print $1}' file | sort | uniq -c | sort -rn | head     # 统计 Top N
awk -F',' '{sum+=$3} END {print sum/NR}' data.csv            # 求平均
jq '.items[] | select(.status=="ERROR") | .id' app.log       # JSON 日志必备
sed -n '10,20p' file
grep -A5 -B5 "Exception" app.log
```

### 2.10 容器相关的三个特殊点

1. **/proc 未隔离**：容器内 `free`、`nproc`、`top` 看到的是宿主机的。Java 8u191 之前会认到宿主机 CPU 数而启动过多 GC 线程和线程池。
   - 解法：JDK 8u191+ 默认开启 `-XX:+UseContainerSupport`，用 `-XX:MaxRAMPercentage=75.0` 代替 `-Xmx`
2. **资源限制看 cgroup**：`/sys/fs/cgroup/memory/memory.limit_in_bytes`（v1）或 `/sys/fs/cgroup/memory.max`（v2）
3. **Docker 两大基石**：namespace（pid/mnt/net/uts/ipc/user 隔离视图）+ cgroup（cpu/memory/blkio 限制资源）

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| free 显示快没内存了，慌 | page cache 占用，可回收 | 看 `available`；`sar -r` 的 `kbcommit`；不要随便 `echo 3 > drop_caches` |
| swap 使用量上涨 | 物理内存不足或 swappiness 设置不当 | 数据库/Redis 机器建议 `vm.swappiness=1`（0 在较老内核有回收倾向的 bug） |
| 磁盘越来越大但 du 找不到大文件 | 已删除文件仍被进程占用 | `lsof \| grep deleted`；用 truncate 而不是 rm 清日志 |
| inode 满导致无法新建文件 | 海量小文件（Java 的临时文件、日志滚动未清理） | `df -i`；清理；`find /path -xdev -type f \| wc -l` 定位目录 |
| 改了 limits.conf 但服务 fd 限制没变 | systemd 服务不读 limits.conf | service 里设 `LimitNOFILE` |
| 容器里 Java 线程数/GC 线程数异常多 | 看到的是宿主机 CPU 核数 | UseContainerSupport + 显式设 ActiveProcessorCount |
| `too many open files` | fd 耗尽（连接泄漏 / 文件流未关闭 / watcher 未释放） | `ls /proc/pid/fd` 看都是什么；修泄漏；调 ulimit 只是延后爆炸 |
| Load 高但 CPU 空闲 | 卡在 IO 或锁 | `vmstat` 的 b 列 + `jstack` 找 BLOCKED |
| CPU 某核打满其他空闲 | 单线程死循环 / epoll 空轮询 | `mpstat -P ALL`；`top -Hp` + jstack |
| Java 进程 regel死了但 exit code 137 | 容器 OOMKill（128+9） | cgroup memory limit + 监控；`dmesg` 确认 |
| reboot 或 rm 日志后空间没释放 | 同上 persist 占用 | restart 进程或 truncate |
| nf_conntrack table full 引发随机连接失败 | 高并发下 conntrack 表满（尤其 NAT 网关/LB） | 调 `nf_conntrack_max` + `nf_conntrack_buckets`；或用 ipvs/DSR 模式 |
| TIME_WAIT 在服务端大量堆积 | 服务端主动关闭（keepalive 超时） | 看谁先关；调 `tcp_tw_reuse`；扩端口池 |
| 执行命令报 Argument list too long | 参数太多个（rm *.log 文件太多） | `find ... -delete` 或 `xargs -n 100` |
| 字脚本在 cron 下能跑在终端不能跑 | 环境变量差异 | 脚本里显式 export / 绝对路径 |
| Redis/MySQL/Mongo 随机延迟抖动 | THP（透明大页）导致 fork 抖动 | `echo never > /sys/kernel/mm/transparent_hugepage/enabled` 并写入 rc.local/systemd |

## 四、往下钻的入口

- **一键排查脚本思路（值得自己攒一个）**：`top -bn1 | head -20` → `vmstat 1 5` → `mpstat -P ALL 1 3` → `free -m` → `df -h/-i` → `iostat -x 1 3` → `sar -n DEV 1 3` → `ss -s` → `dmesg -T | tail -50`
- **火焰图**：`async-profiler`（`./profiler.sh -d 30 -e cpu <pid>`）+ FlameGraph.pl。这是定位 CPU 热点的最优工具，比 jstack 采样靠谱得多
- ** Arthas 常用**：`dashboard`、`thread -b`（一键查死锁！）、`thread -n 3`、`watch`、`trace`、`ognl`、`jad`/`mc`/`redefine`（热替换）
- **proc 文件系统（第一手信息源）**：`/proc/<pid>/{status,stat,limits,fd/,cmdline,environ,cgroup,oom_score,smaps}`、`/proc/pressure/{cpu,io,memory}`（PSI，Linux 4.20+ 的精确压力指标，比 load 更准）
- **官方文档/书(def)**：Red Hat 的 *Performance Tuning Guide* 章节写得极实，Brendan Gregg 的《Systems Performance》第 2 版（讲 USE 方法与工具链），以及他的 blog/BCC 工具集
- **内核源码瞄点（可选）**：`kernel/sched/core.c`（调度）、`mm/page_alloc.c`、`fs/ext4/`、`net/ipv4/tcp_input.c`（拥塞控制就在 `tcp_cong_avoid`）

## 五、达标自检

1. `free` 里 buff/cache 占用很大，需要清理吗？为什么？什么指标才是判断内存真的不够？
2. Load average 为 20 但 CPU 使用率只有 10%，可能是什么原因？怎么进一步确认？
3. 描述完整的 CPU 飙高排查 SOP，从 top 到具体代码行。
4. jstack 里的 nid 是十进制还是十六进制？如何转换？
5. 什么时候该用 `jmap -dump`？它有什么副作用？
6. 区分 heap OOM、metaspace OOM、direct buffer OOM、failed to create thread 这几种报错各自的成因与解法。
7. 容器里的 Java 进程被杀了，你怎么判断是 JVM OOM 还是容器 OOMKill？两者的取证手段分别是什么？
8. 磁盘显示 no space left on device 但 `df -h` 还有剩余，两条完全不同的排查路径是什么？
9. select / poll / epoll 的三次演进分别解决了什么问题？epoll 的 LT 和 ET 该怎么选，用 ET 漏读会怎样？
10. 为什么说 select/poll/epoll 是"同步非阻塞 IO"而不是异步 IO？真正的异步 IO 在 Linux 上是什么？
11. sendfile 和 mmap 两种零拷贝的区别？为什么 Kafka 快？
12. `net.ipv4.tcp_tw_reuse` 和 `tcp_tw_recycle` 的区别？为什么后者被内核移除？
13. 修改了 `/etc/security/limits.conf` 但服务的 ulimit 没生效，为什么？怎么改？
14. 一个 systemd service 文件通常要写哪些字段？Restart 的几种取值有什么区别？
15. kill -9 和 kill -15 对 Java 进程的影响差异是什么？什么时候必须用 -9？
16. `kill -3` 对 Java 进程意味着什么？在容器里它有什么妙用？
17. 如何统计某个远端 IP 与本机建立了多少连接？写出命令。
18. 网卡丢包从哪里看？`ip -s link` 里 dropped、errors、overruns 各表示什么阶段丢的？
19. nf_conntrack table full 的表现和处理方式是什么？
20. 容器里的 `nproc` 为什么可能不是容器限额？Java 侧有什么影响？
21. 一个 cron 脚本手动执行正常但定时任务失败，列出至少 3 个可能原因。
22. 写一行命令：找出当前目录下大于 100MB 的文件并按大小排序。
23. 怎么用 awk 统计日志里每个状态码的出现次数？
24. THP 应该在什么场景下关闭？它会引起什么现象？
25. 描述一次你该怎么在 5 分钟内判断"服务响应慢"是网络、磁盘还是 CPU 的问题（给出命令顺序）。
