# 网络协议 · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
> 主线：TCP / TLS / HTTP / WebSocket / DNS，以及它们对应的排障手段

## 一、核心心智模型

1. **TCP 的三个痛点：连接怎么建、数据怎么可靠传、流量怎么控。** 三次握手解决的是"同步初始序列号 + 确认双方收发能力"；可靠传输靠序列号/确认号 + 重传；传输效率靠滑动窗口 + 拥塞控制。**所有 TCP 的复杂行为都能归到这三件事上。**

2. **可靠的代价是延迟。** TCP 保证顺序，代价是**队头阻塞**——前面一个包丢了，后面的包即使到了也不能交给应用层（必须等重传）。HTTP/2 的多路复用把它从应用层移到了传输层，HTTP/3 QUIC 用多 stream 彻底绕开。这是三代 HTTP 演进的唯一主线。

3. **TLS 是"非对称协商密钥 + 对称加密数据"的混合方案。** 非对称只用在握手阶段交换一个共享密钥，之后全是 AES 这类对称加密。为什么？非对称慢一到两个数量级。而 ECDHE 比 RSA 密钥交换强的地方在于**前向安全**：服务器私钥泄露也无法解密历史流量。

4. **TIME_WAIT 是为可靠性付出的"守孝"代价。** 主动关闭方必须等待 2MSL：一是确保最后的 ACK 能到达（丢了重传 FIN 还能回应）；二是让这条连接的残余报文在网络中自然消散，避免污染后续复用了相同四元组的新连接。它不是 bug，是设计。

5. **DNS 是被严重低估的可用性瓶颈。** 它往往是服务不可用的第一个连锁反应点（尤其是 JVM 的 DNS 缓存导致上游 IP 变更后数小时连不上旧 IP）。理解 DNS 的递归/迭代、TTL、多级缓存，能解释很多"诡异地连不上"。

## 二、必懂机制

### 2.1 三次握手

```
C: SYN  seq=x                    →  客户端进入 SYN_SENT
S: SYN  seq=y, ACK=x+1           ←  服务端进入 SYN_RCVD（半连接队列 syn queue）
C:      ACK=y+1                  →  双方 ESTABLISHED（从半连接移到全连接队列 accept queue）
```
为什么三次：**双方各发送 SEQ 并各确认对方的 SEQ，最少需要三次通信**。两次不够，因为服务端无法确认"自己确认客户端的那个 ACK 客户端收到了"——而这决定了一开始发的数据会不会被丢序列号同步。用一句大白话：TCP 是全双工，两个方向都要各自同步一次序列号并确认。

状态机关键节点：

| 状态 | 含义 | 大量出现说明什么 |
|---|---|---|
| SYN_SENT | 已发 SYN 未收到响应 | 目标不通/被防火墙丢包/backlog 满导致服务端不回 SYN+ACK |
| SYN_RCVD | 半连接队列 | SYN Flood 攻击；或 `tcp_max_syn_backlog` 太小 |
| ESTABLISHED | 已建立 | 正常态 |
| FIN_WAIT_1/2 | 主动关闭方 | FIN_WAIT_2 大量且长期存在：对端一直不 close |
| **CLOSE_WAIT** | 被动关闭方，收到对端 FIN 已回 ACK，等待自己调 close | **己方代码 bug**（socket 用完没关 / 连接池没归还）。这是最有诊断价值的一个状态 |
| LAST_ACK | 已发 FIN 等最后 ACK | 少见，大量说明对方来不及 ACK |
| TIME_WAIT | 主动关闭方，2MSL | 高并发短连接场景下正常偏多 |

```bash
ss -ant | awk '{print $1}' | sort | uniq -c | sort -rn    # 统计各状态连接数
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

### 2.2 四次挥手与 TIME_WAIT

```
A: FIN seq=u            → A 进入 FIN_WAIT_1（A 不再发数据，但还能收）
B:      ACK=u+1         ← B 进入 CLOSE_WAIT
B: FIN seq=v, ACK=u+1   ← B 进入 LAST_ACK（B 也要关了）
A:      ACK=v+1         → A 进入 TIME_WAIT，等 2MSL 后 CLOSE
```
为什么四次：TCP 是全双工，**每个方向要单独关闭**。B 收到 FIN 时可能还有数据没发完，所以 ACK（确认收到 FIN）和自己的 FIN（数据发完了）必须分开发——这就是比握手多一次的原因。支持"捎带 ACK + FIN"时可变成三次（对端没数据要发时）。

**TIME_WAIT = 2MSL 的两个理由**：
1. 保证最后一个 ACK 能到达 B（如果丢了对万人也有这说明是因为 ACK 丢了，B 会重发 FIN，A 在 TIME_WAIT 内还能重 ACK）
2. 让旧连接的残余报文在网络中消散，防止被复用同一四元组（源IP:源端口, 目的IP:目的端口）的新连接误收

```
MSL(Maximum Segment Lifetime) Linux 默认 30s，所以 TIME_WAIT 是 60s
cat /proc/sys/net/ipv4/tcp_fin_timeout   # 这个值影响的是 FIN_WAIT_2，不是 TIME_WAIT 本身！
```

TIME_WAIT 过多的治理：
```bash
net.ipv4.tcp_tw_reuse = 1        # 允许将 TIME_WAIT socket 用于新连接（客户端场景有效，需要 tcp_timestamps=1）
# tcp_tw_recycle 在 Linux 4.12 后已删除 —— 它在 NAT 环境下会因时间戳乱序导致大面积连接失败，是著名的坑
net.ipv4.ip_local_port_range = 1024 65535   # 扩大可用端口池（本质是解决四元组上限）
```
> 重要认知：**TIME_WAIT 出现在服务端通常说明服务端主动关闭了连接**（比如 Nginx `keepalive_timeout` 到期、应用层 close），不要只在客户端找原因。

### 2.3 重传、滑动窗口与拥塞控制

**重传两种触发**：
- **超时重传**：RTO 超时。RTO 基于 RTT 动态计算（SRTT + 4×RTTVAR，Karn 算法：重传的包的 RTT 样本不参与计算，因为它无法区分是对原包的 ACK 还是对重传包的 ACK）
- **快速重传**：收到 **3 个重复 ACK** 就立即重传，不等 RTO（因为重复 ACK 说明后续包到了，只是中间那个丢了）

**SACK**（Selective Acknowledgment）：解决"该重传哪些"——接收方告诉发送方哪些区间到了，发送方只补没到的。没有 SACK 时只能重传之后的所有包（回退 N 步）。

**滑动窗口**：`发送窗口 = min(接收窗口 rwnd, 拥塞窗口 cwnd)`。rwnd 由对端通告（受限于对端缓冲区），cwnd 由拥塞控制算。窗口为 0 时发送方会定期发**窗口探测**。

**拥塞控制四阶段**：
```
1. 慢启动     cwnd 从 1 个 MSS 开始，每收到一个 ACK 就 +1 → 指数增长；超过 ssthresh 进入下一阶段
2. 拥塞避免   每 RTT cwnd +1 → 线性增长
3. 快重传     收到 3 个重复 ACK → 立即重传，进入快恢复
4. 快恢复     ssthresh = cwnd/2，cwnd = ssthresh + 3，继续拥塞避免（而非回到慢启动）
```
超时重传（而非快重传）会很严厉：ssthresh = cwnd/2，cwnd 直接回 1。

**BBR**：不再以丢包作为拥塞信号（丢包在现代网络中往往不等于拥塞），而是测量带宽-时延积 BDP = BtlBw × RTprop，把发送速率维持在 BDP 附近。高带宽长肥管道（跨洋）上收益巨大。

**Nagle vs Delayed ACK**（40ms 延迟的经典来源）：
- Nagle：小包要等前面小包的 ACK 才发（为了减少网络上的小包数量）
- Delayed ACK：收到数据不立即 ACK，等 40ms 或等有数据捎带
- **二者叠加**：发送方等 ACK，接收方延迟 ACK → 互相等待到 40ms 超时才动
- 解法：`TCP_NODELAY`（禁用 Nagle）。Nginx、Redis、RPC 框架默认都会开

### 2.4 两种队列：半连接与全连接

```
客户端 SYN → [SYN Queue 半连接队列] → 三次握手完成 → [Accept Queue 全连接队列] → accept() 取走
```
- 半连接队列大小：`max(tcp_max_syn_backlog, 64)`
- **全连接队列大小 = min(listen(fd, backlog), net.core.somaxconn)**。注意 **Linux 2.2 起 backlog 的语义变了**，它指的是全连接队列（以前是半连接）
- 全连接队列满时的行为由 `tcp_abort_on_overflow` 决定：0（默认）丢弃 ACK 让客户端重传；1 则直接回 RST
- 半连接队列满且未开 syncookies → 丢弃 SYN，客户端表现为连接超时

```bash
ss -lnt    # 监听状态下：Recv-Q = 已完成握手等待 accept 的数量，Send-Q = 最大全连接队列长度
           # 注意：已建立连接时 Recv-Q 含义完全不同（未读取字节数），不要混淆！
netstat -s | grep -i "listen"   # 看全连接队列溢出与丢包计数
```
**SYN Flood 与 syncookies**： attacker 发 SYN 后不回 ACK，塞满半连接队列。`net.ipv4.tcp_syncookies=1` 让服务端不分配资源也能回复 SYN+ACK（把状态编码进 seq），是有效的缓解手段。

### 2.5 UDP 与 QUIC

UDP：无连接、不保证到达、不保证顺序、无流量与拥塞控制。头部 8 字节（vs TCP 20+）。

QUIC 在 UDP 之上重建了可靠传输，几个关键改进：
- **0-RTT / 1-RTT 握手**（TLS 1.3 集成在 QUIC 里，不像 TCP+TLS 要分两次握手）
- **多路复用但无队头阻塞**：每个 stream 独立重传，一个包丢只影响它所在的 stream
- **连接迁移**：用 Connection ID 而非四元组标识连接，WiFi 切 4G 不用重连（这是移动端最大的收益）
- **在用户态实现**（不在内核），迭代快

### 2.6 HTTP 演进

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| 传输层 | TCP | TCP | QUIC(UDP) |
| 并发 | 多个 TCP 连接（浏览器通常 6 个） | 单连接多路复用 | 单连接多路复用 |
| 队头阻塞 | 应用层阻塞（请求排队） | 移到传输层（丢包全阻塞） | 解决（stream 独立） |
| 头部 | 纯文本，重复巨大 | HPACK 压缩 + 静态字典 | QPACK |
| 握手 | TCP + TLS 分开 | TCP + TLS 分开 | 合并，0-RTT 可选 |

为什么 pipelining（HTTP/1.1 的管线化）失败了：服务端必须按请求顺序响应，一个慢请求阻塞后面所有；且实现复杂容易出错，浏览器基本都不开。

**首部字段语义（写业务代码真会用到的）**：
- `Content-Length` vs `Transfer-Encoding: chunked`：二选一，同时出现会让行为不可预期
- `Connection: keep-alive`：HTTP/1.1 默认长连接，显式 close 才是关闭
- `Cache-Control: max-age / no-cache / no-store` 与 `ETag` / `If-None-Match`（304 协商缓存）
- `Vary`：决定缓存的 key 要不要带上某个 header（漏配 Vary 会导致 CDN 给用户返回错的内容）
- `Host`：虚拟主机的基础

### 2.7 TLS / HTTPS

**TLS 1.2 完整握手**（RSA 密钥交换版简化）：
```
C: ClientHello      （支持的版本、密码套件列表、随机数 ClientRandom）
S: ServerHello      （选定版本与套件、ServerRandom）+ Certificate（含公钥）+ ServerHelloDone
C: 验证证书链 → 用公钥加密 PreMasterSecret 发给服务端 → ChangeCipherSpec → Finished（已加密）
S: 私钥解出 PreMasterSecret → 双方用 ClientRandom+ServerRandom+PreMasterSecret 算出对称密钥
S: ChangeCipherSpec → Finished
握手完成，后续走对称加密
```
**ECDHE**（现在的主流）：服务端用椭圆曲线临时密钥对，客户端同样，双方用 ECDH 算出共享密钥。服务器的私钥只用于**签名**（证明身份），不用于加密 → 即使私钥将来泄露也解不出历史流量 = **前向安全**。

**TLS 1.3 的变化**：
- 砍掉静态 RSA 密钥交换（只有前向安全的 DHE/ECDHE）
- 砍掉 CBC 模式、RC4、SHA-1、压缩
- 1-RTT 握手，甚至 0-RTT（代价：0-RTT 数据可被重放，因此不适合非幂等请求）
- ClientHello 里就猜着发 key_share，节省一个 RTT

**会话复用**：Session ID（服务端要存状态，集群里要共享存储）vs Session Ticket（服务端无状态，把会话加密后给客户端，**但 ticket key 不轮换会失去前向安全**）。

**证书校验的常见失败原因**：
1. 证书链不完整（缺中间 CA）—— Java 特别严格，浏览器能容忍不代表 Java 能
2. 域名不匹配（SAN 里没有当前域名）
3. 证书过期
4. 系统时间错误
5. Java 需要把根证书导入 cacerts truststore（自签名证书场景）

```bash
openssl s_client -connect example.com:443 -servername example.com   # 看完整证书链
openssl x509 -in cert.pem -noout -text     # 看有效期与 SAN
curl -v -o /dev/null https://example.com   # 看握手过程
```

### 2.8 WebSocket

握手就是一次带 Upgrade 的 HTTP 请求：
```
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

响应 101 Switching Protocols
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
`Sec-WebSocket-Accept` 的计算：`base64(SHA1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`，这个 GUID 是协议规定的魔数。

要点：
- 帧类型 text/binary/**ping/pong**（心跳保活，应用层必须自己做，TCP keepalive 默认 2 小时）
- **代理层必须配 Upgrade 头**，否则 Nginx 会因为不理解 Upgrade 而把它当普通 HTTP 处理
- `proxy_read_timeout` 要调大（否则 60s 被断）

### 2.9 DNS

```
递归查询：本地 DNS 服务器替你查完再返回结果（客户端只用一次请求）
迭代查询：本地 DNS 逐级去问 根 → 顶级(.com) → 权威(example.com)，每次返回下一级的地址
```
记录类型：A（IPv4）、AAAA（IPv6）、CNAME（别名，**不能用于根域名**，且会让 DNS 查询多一跳）、NS（权威服务器）、MX（邮件）、TXT（SPF/域名验证）、SRV（服务发现，K8s/服务注册会用）、PTR（反查）。

**Java 的坑**：JVM 自己缓存 DNS 结果
```java
// 默认：成功结果永久缓存（security manager 存在时）；无 security manager 时由 networkaddress.cache.ttl 控制，默认 -1 = 永不过期
// 现象：DNS 记录的 IP 变了，Java 服务要重启才认
java.security.Security.setProperty("networkaddress.cache.ttl", "60");
System.setProperty("sun.net.inetaddr.ttl", "60");   // 或启动参数 -Dsun.net.inetaddr.ttl=60
```

其他排障点：HTTPDNS（绕过运营商 LocalDNS 防劫持）、容器 `/etc/resolv.conf` 的 `ndots` 配置（影响是否先拼 search 域，会让每次 DNS 查询多几次）。

### 2.10 Java Socket 参数对照

| Socket 参数 | 内核选项 | 含义与建议 |
|---|---|---|
| `backlog` | somaxconn 取 min | ServerSocket 构造时传，**默认只有 50**，高并发服务必须显式调大 |
| `SO_REUSEADDR` | — | 允许绑定 TIME_WAIT 状态的地址。**服务重启"Address already in use"的第一解法** |
| `SO_REUSEPORT` | reuseport | 多进程监听同一端口，内核做负载均衡，解决 Nginx 惊群 |
| `TCP_NODELAY` | tcp_nodelay | 禁用 Nagle。**RPC/Redis/实时消息务必开启** |
| `SO_KEEPALIVE` | tcp_keepalive_* | 默认心跳 7200 秒（2小时），形同虚设。要保活得应用层自己做或调系统参数 |
| `SO_TIMEOUT` | — | 读超时，不等同于连接超时 |

```java
// 正确写法示例
ServerSocket server = new ServerSocket(port, 1024);  // backlog 必须显式给
Socket socket = new Socket();
socket.connect(new InetSocketAddress(host, port), 3000);  // connect timeout
socket.setSoTimeout(5000);   // read timeout
socket.setTcpNoDelay(true);
socket.setKeepAlive(true);
```

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 服务重启报 Address already in use | TIME_WAIT 状态的 socket 占用端口 | `SO_REUSEADDR`（Nginx/Tomcat 默认已开，自写代码要注意） |
| CLOSE_WAIT 几千个越积越多 | 应用没调 close（异常分支漏了、连接池没归还、Netty ByteBuf 没释放） | 查代码 finally/try-with-resources；`lsof -p pid` 看 fd 增长 |
| FIN_WAIT_2 长期大量 | 对端不主动关闭 | 调 `tcp_fin_timeout`；检查对端代码 |
| 客户端连不上，SYN 无响应 | 防火墙丢包 / 全连接队列满 / 半连接队列满（SYN Flood） | `ss -lnt` 看 Recv-Q；`netstat -s` 看溢出计数；开 syncookies |
| 连接很慢但最终能通 | DNS 慢 / 路由黑洞 / MTU 问题 | 用 curl 的分段耗时定位（见下） |
| 小数据交互固定延迟 40ms 左右 | Nagle + Delayed ACK 互相等待 | `TCP_NODELAY` |
| 大文件上传失败但小文件正常 | MTU/PMTUD 黑洞（ICMP 被禁，DF 置位的大包被丢） | 放开 ICMP；调小 MSS（`tcp_mtu_probing=1` 或 iptables clamp-mss） |
| HTTPS 部分 Java 客户端握手失败，浏览器正常 | 证书链缺中间 CA | 补全证书链（fullchain） |
| 容器内 DNS 慢 | ndots 默认 5 导致多次无效查询 | 改 `/etc/resolv.conf` 的 ndots，或用全限定域名带点结尾 |
| 上游 IP 变更后长时间连不通 | JVM DNS 永久缓存 | 设 `networkaddress.cache.ttl` |
| WebSocket 连接 60 秒后断开 | 代理 `proxy_read_timeout` 默认 60s | 调大超时 + 应用层 ping |
| 服务 CPU 全耗在 TLS 握手 | 未启用会话复用；RSA 密钥贵 | `ssl_session_cache`；换 ECDSA 证书 |
| 压测 QPS 上不去，连接数不少但都很闲 | 端口耗尽（TIME_WAIT 占满 ip_local_port_range） | 扩端口池 + tw_reuse + 改短连接为长连接 |
| 抓包看到大量重传 | 网络丢包 / buffer 太小 / 拥塞 | 看 `netstat -s` 重传计数；调 rwnd/tcp_rmem |

## 四、往下钻的入口

- **curl 分段耗时（排障第一手段）**：
```bash
curl -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} \
ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s https://example.com
# dns 慢 → DNS 问题；tcp 慢 → 网络/队列；tls 慢 → 握手（可能没复用）；ttfb 慢 → 服务端处理；total-ttfb 大 → 传输/下载慢
```
- **tcpdump 常用**：
```bash
tcpdump -i any host 10.0.0.1 and port 8080 -nn -s0 -w /tmp/cap.pcap
tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'      # 只看 SYN/RST
tcpdump -i eth0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'  # 有 payload 的包
```
- **Wireshark 显示过滤器（tcp 排障必会）**：`tcp.flags.reset==1`、`tcp.analysis.retransmission`、`tcp.analysis.zero_window`、`tcp.stream eq 5`、`http2`
- **状态/统计**：`ss -antp`、`ss -s`（总统计）、`netstat -s`（各协议错误计数，尤其 retransmitted / listen overflows）、`ip -s link`（看 RX/TX dropped、errors）
- **权威规范**：RFC 793（TCP）、RFC 6298（RTO 计算）、RFC 7323（TCP 扩展 OPTIONS）、RFC 8446（TLS 1.3）、RFC 7540（HTTP/2）、RFC 9000（QUIC）。不需要通读，作为"某个行为到底怎么规定的"的查证依据
- **Java 侧瞄点**：`java.net.Socket` / `SocketOptions`、Netty 的 `ChannelOption`（`SO_BACKLOG`、`TCP_NODELAY`、`SO_REUSEADDR`、`SO_KEEPALIVE`）、`EpollSocketChannel`（Netty 的 epoll 封装）

## 五、达标自检

1. 为什么三次握手必须是三次？两次会有什么问题？
2. 四次挥手为什么要四次？什么情况下能变成三次？
3. TIME_WAIT 出现在服务端意味着什么？它的两个存在理由是什么？
4. `tcp_tw_recycle` 为什么被删除？它在什么环境下有害？
5. `ss -lnt` 里监听态的 Recv-Q/Send-Q 与已建立态的含义有何不同？
6. 全连接队列的大小由哪两个值决定？满了之后内核怎么处理？
7. 什么是 SYN Flood？syncookies 是怎么在没分配资源的情况下工作的？
8. CLOSE_WAIT 大量堆积能说明什么？你如何定位到具体代码？
9. 快速重传为什么是 3 个重复 ACK 而不是 2 个？
10. SACK 解决了什么问题？没有它会怎样？
11. 拥塞控制里"超时重传"和"快重传"导致的降窗有什么区别？
12. Nagle 算法和 Delayed ACK 为什么会互相等待？谁该让步？
13. HTTP/2 的多路复用为什么没能彻底解决队头阻塞？HTTP/3 怎么解决的？
14. TLS 握手为什么是"非对称协商 + 对称加密"，而不是全程非对称？
15. ECDHE 相比 RSA 密钥交换的核心优势是什么？什么是前向安全？
16. TLS 1.3 砍掉了哪些东西？0-RTT 有什么代价？
17. Session Ticket 相对于 Session ID 的取舍是什么？ticket key 不轮换有什么后果？
18. 证书链不完整会导致什么现象？为什么浏览器没事但 Java 报错？
19. WebSocket 的 `Sec-WebSocket-Accept` 是怎么算出来的？为什么要这么设计？
20. JVM 的 DNS 缓存在生产上会引发什么事故？怎么设置和管理它？
21. `ServerSocket(port)` 不传 backlog 会怎样？默认值大概是多少？
22. 一个 Java 服务上线后 DNS 记录变更，为什么它几个月都连到旧 IP？在不重启的前提下怎么缓解？
23. 用 curl 观察到一个请求 dns=0.005 tcp=0.010 tls=0.300 ttfb=1.200 total=1.250，请解读每一段并给出优化建议。
24. 容器内 Java 服务 DNS 解析慢，可能的根因有哪些？（提示：ndots + conntrack + CoreDNS）
