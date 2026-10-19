# Nginx · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
> 版本基线：Nginx 1.20+ / OpenResty

## 一、核心心智模型

1. **多进程单线程，而不是多线程。** master 负责管理（读配置、起 worker、热升级），worker 负责干活。**一个 worker 只有一个事件循环线程**，靠 epoll 事件驱动处理成千上万连接。这个设计回避了多线程的锁竞争和调试地狱，代价是每个请求都不能有阻塞操作——一旦卡住（比如磁盘慢 IO），被它占住的那个 worker 上的所有连接一起遭殃。

2. **Nginx 的配置是"声明 + 继承"，不是脚本。** 它不会从上往下执行，配置指令有自己的作用域（http/server/location/if）和合并规则。**`if` 尤其危险**（官方有名一篇文章叫 *If is Evil*），因为它不在常规上下文归并流程里——所以宁可用 `location` 匹配或 `map`，也不要在 location 里写复杂 if。

3. **反向代理的坑 90% 在 header 和 URI 拼接。** Nginx 转发时默认丢掉原始 Host，不配 `X-Forwarded-For` 后端就拿不到真实 IP；`proxy_pass` 结尾有没有斜杠决定了 URI 是"替换"还是"拼接"——这是最高频的 404 来源。

4. **超时配置是分布式系统的最后一道防线。** `proxy_read_timeout` 太长会导致上游故障时连接堆积拖垮 Nginx；太短会误杀正常长任务。合理的值是比后端 p99 略高，而不是随便设 60s。

5. **Nginx 不只是 Web 服务器，它是流量治理的总入口。** 路由、限流、鉴权、灰度、TLS 终止、缓存、压缩、访问控制都能在这一层做完，而且往往比在应用层做更早生效、更低代价。但**业务逻辑不应该写进 Nginx 配置里**——那超出了它的能力边界（除非用 OpenResty 的 Lua）。

## 二、必懂机制

### 2.1 进程模型与连接数

```nginx
worker_processes auto;              # 通常等于 CPU 核数
worker_cpu_affinity auto;           # CPU 亲和绑定，减少缓存失效
worker_rlimit_nofile 65535;         # 必须 >= worker_connections * 2
events {
    use epoll;
    worker_connections 10240;       # 单个 worker 的最大连接数
    multi_accept on;
    accept_mutex on;                # 防止惊群（reuseport 场景可关）
}
```

**最大并发连接数 = worker_processes × worker_connections**，但因为反向代理时一个客户端连接会消耗两个连接（对客户端 + 对上游），实际承载的客户数是它的一半。同时它受限于系统的 `ulimit -n`（systemd 服务不受 `/etc/security/limits.conf` 限制，要改 service 里的 `LimitNOFILE`）。

热加载原理：`nginx -s reload` → master 收到 HUP → **用新配置起一批新 worker**，老 worker 不再接受新连接、处理完现有请求才退出 → 全程不断连接。这也是为什么改配置后旧连接可能还用旧行为。

### 2.2 location 匹配规则（必须背下来）

匹配优先级从高到低：
```
1. =  精确匹配 /login          → 命中即停止
2. ^~ 前缀匹配，且命中后不再检查正则
3. ~  区分大小写的正则
4. ~* 不区分大小写的正则
5.    普通前缀匹配（取最长的那个）
```
关键差异：**普通前缀匹配是"记下最长的"，然后还会继续检查正则**；而 `^~` 和 `=` 一旦命中就立即停止，不再看正则。所以：

```nginx
location ^~ /static/ { root /var/www; }   # 优先级最高，且不会被下面的正则覆盖
location ~ \.php$ { ... }                 # 正则
location / { proxy_pass http://app; }     # 兜底
```

`root` 与 `alias` 的区别（`alias` 是导致 404 的高频点）：
```nginx
location /img/ { root /data; }        # 请求 /img/a.jpg → /data/img/a.jpg（拼接完整 URI）
location /img/ { alias /data/pic/; }  # 请求 /img/a.jpg → /data/pic/a.jpg（用 alias 替换 location 前缀）
```

### 2.3 proxy_pass 的 URI 替换规则

**结尾带 `/`** 时，location 匹配到的部分会被**替换掉**：
```nginx
location /api/ {
    proxy_pass http://backend/;     # /api/user/1 → http://backend/user/1  （/api/ 被替换成 /）
}
```
**结尾不带 `/`** 时，完整 URI 会**原样拼上去**：
```nginx
location /api/ {
    proxy_pass http://backend;      # /api/user/1 → http://backend/api/user/1
}
```
**带 URI 路径**时：
```nginx
location /api/ {
    proxy_pass http://backend/app/; # /api/user/1 → http://backend/app/user/1
}
```

### 2.4 反向代理与真实 IP

```nginx
proxy_set_header Host $host;                    # 或 $http_host；不设则默认是 proxy_pass 的 host
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

proxy_http_version 1.1;                         # 长连接必需
proxy_set_header Connection "";                 # 清掉 close，配合 upstream keepalive
```
后端（Java）取真实 IP 的正确姿势：
```nginx
set_real_ip_from 10.0.0.0/8;      # 可信代理的 CIDR
set_real_ip_from 172.16.0.0/12;
real_ip_header X-Forwarded-For;    # 或 X-Real-IP
real_ip_recursive on;              # 递归剥离可信代理，拿到第一个不可信 IP
```
> **安全警告**：不配 `set_real_ip_from` 就信任 `X-Forwarded-For` 等于允许任何人伪造 IP（伪造太容易了）。多层代理时务必逐个放行可信网段。

### 2.5 upstream 与负载均衡

```nginx
upstream backend {
    least_conn;                      # 轮询(默认) / ip_hash / hash $key consistent / random
    server 10.0.0.1:8080 weight=3 max_fails=2 fail_timeout=30s;
    server 10.0.0.2:8080 backup;     # 备用机，主全挂才启用
    keepalive 64;                    # 每个 worker 到上游的空闲长连接数
    keepalive_requests 1000;
    keepalive_timeout 60s;
}
```
- **轮询**（默认，可加 weight）、**least_conn**（少连接优先，适合长耗时请求）、**ip_hash**（会话保持，注意它会让一部分机器流量倾斜，且不支持 weight 与 backup）、**hash $request_uri consistent**（一致性哈希，缓存场景）
- **被动健康检查**：`max_fails` 次失败后在 `fail_timeout` 内标记为不可用。开源版没有主动健康检查（商业版 Nginx Plus 有，开源替代可用 nginx_upstream_check_module 或改用 APISIX/OpenResty）
- **`proxy_next_upstream`**：失败时换机器重试。默认是 `error timeout`，**千万不要无脑加 `http_500`**——那会让非幂等的 POST 请求被重试到另一台机器，造成重复下单。要加也只能针对幂等接口

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;   # 幂等场景
proxy_next_upstream_tries 2;
proxy_next_upstream_timeout 10s;
```

### 2.6 超时三兄弟

| 指令 | 含义 | 注意 |
|---|---|---|
| `proxy_connect_timeout` | 与上游**建立 TCP 连接**的超时，默认 60s | 内网通常给 5s 就够 |
| `proxy_send_timeout` | 向上游**两次写操作之间**的间隔超时，默认 60s | **不是整个请求发送的总时间**，流式上传要留意 |
| `proxy_read_timeout` | 等待上游**两次读之间**的间隔超时，默认 60s | 这是通常说的"后端响应慢"的那个超时 |

另外客户端侧还有：`client_header_timeout`、`client_body_timeout`、`send_timeout`，以及超时纠正三件套 `client_max_body_size`（默认 1m！上传超限报 **413**）。

### 2.7 限流与防护

```nginx
limit_req_zone $binary_remote_addr zone=ip_limit:10m rate=10r/s;   # 漏桶
server {
    location /api/ {
        limit_req zone=ip_limit burst=20 nodelay;   # burst 缓冲队列，nodelay 立即处理不排队
        limit_req_status 429;
        limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
        limit_conn conn_limit 20;                    # 单 IP 并发连接数
    }
}
```
- `limit_req` 是**漏桶**（恒定速率出水，burst 是桶容量），`rate=10r/s` 表示平均速率
- `nodelay`：burst 内的请求立即处理（相当于允许突发），不加则排队等待（延迟大但绝对匀速）
- `$binary_remote_addr`（4/16 字节）比 `$remote_addr`（字符串）省内存，10MB zone 能存约 16 万条
- 限流 key 可以按业务定制：`$http_authorization`、`$cookie_sessionid`、`$server_name`

### 2.8 静态资源与缓存

```nginx
sendfile on;              # 零拷贝，静态文件必备
tcp_nopush on;            # 配合 sendfile，攒满一个 MSS 再发
tcp_nodelay on;           # keepalive 连接上禁用 Nagle，降低小包延迟
open_file_cache max=10000 inactive=30s;   # 缓存文件描述符，高并发静态服务很有用

location ~* \.(js|css|png|jpg)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
    gzip_static on;       # 优先返回预压缩的 .gz 文件
}
```
代理缓存（减少对后端的压力）：
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m use_temp_path=off;
location / {
    proxy_cache my_cache;
    proxy_cache_key $scheme$proxy_host$request_uri;
    proxy_cache_valid 200 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;  # 后端挂了返回过期缓存，非常实用
    proxy_cache_lock on;                  # 缓存未命中时只放行一个请求去回源
    proxy_cache_background_update on;     # 后台异步更新过期缓存
    add_header X-Cache-Status $upstream_cache_status;  # HIT/MISS/EXPIRED/BYPASS，排障必备
}
```

### 2.9 HTTPS / WebSocket / HTTP2

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;   # 必须含中间证书，否则部分客户端（尤其 Java/Android）校验失败
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;      # 会话复用，大幅降低握手 CPU
    ssl_session_timeout 1d;
    ssl_session_tickets off;               # 或定期轮换 ticket key
    ssl_stapling on;                       # OCSP 封套，减少客户端验证延迟

    location /ws/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;      # WebSocket 必配
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;                    # 长连接必须调大，否则 60s 就被断开
    }
}
```

### 2.10 日志与变量

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'rt=$request_time urt="$upstream_response_time" upstream=$upstream_addr';
```
- `$request_time`：从收到客户端第一个字节到写完响应的总时间
- `$upstream_response_time`：后端耗时（多个 upstream 用逗号分隔，含重试的每一次）
- **二者差值大 = Nginx 侧有问题**（通常是客户端慢或网络问题），二者接近 = 后端慢
- `$upstream_addr`：实际打到哪台机器，定位某台机器慢的利器

```nginx
access_log /var/log/nginx/access.log main buffer=32k flush=5s gzip;  # 缓冲写能显著降低 IO
error_log /var/log/nginx/error.log warn;
```

### 2.11 运维与热升级

```bash
nginx -t                    # 检查配置语法（改完必做）
nginx -T                    # 打印完整生效配置（排查继承问题时必备）
nginx -s reload             # 平滑重载
nginx -s reopen             # 重新打开日志文件（配合 logrotate）
```
**二进制热升级**（不中断服务换 Nginx 版本）：
```
kill -USR2 <old_master_pid>   # 新 master 启动（新老 worker 共存）
kill -WINCH <old_master_pid>  # 优雅关闭老 worker
kill -QUIT <old_master_pid>   # 确认没问题后退出老 master
```

```nginx
location = /nginx_status { stub_status; allow 127.0.0.1; deny all; }  # 简单监控
```

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| `proxy_pass` 后路径多了一层或少了一层 | 结尾有无 `/` 导致 URI 是拼接还是替换 | 按 2.3 的规则改写；不确定就用 `proxy_pass http://backend$request_uri` |
| 后端拿到的 Host 是 upstream 名字 | 未设置 `proxy_set_header Host` | 显式设置 `$host` 或 `$http_host` |
| 后端记录的 IP 全是 Nginx 内网 IP，或可被伪造 | 缺 `X-Forwarded-For` / 未配 `set_real_ip_from` | 三者齐全配置（见 2.4） |
| 上传大文件报 413 | `client_max_body_size` 默认 1m | 在 `http` 或 `server` 或 `location` 级调大；**注意三者是嵌套继承关系** |
| 大请求头报 400/502 | `large_client_header_buffers` 太小（Cookie 超长常见） | 调大到 `4 16k` |
| 偶发 504 但后端没超时 | 通常是后端慢到超过 `proxy_read_timeout`（也可能是 DNS 或 LB 层） | 对比 `$request_time` 与 `$upstream_response_time` |
| worker_connections 改大没生效 | 受 `ulimit -n` 限制，且 systemd 服务不吃 limits.conf | systemd unit 里写 `LimitNOFILE=65535` |
| upstream 长连接没生效 | 缺 `proxy_http_version 1.1` + `Connection ""` 组合 | 二者必须同时配 |
| 配置改了 reload 后行为没变 | 老 worker 还在处理旧连接；或存在多处配置互相覆盖 | 等老 worker 退出；用 `nginx -T` 看最终生效值 |
| location 里用了 if 导致行为诡异 | if 不在常规上下文归并流程 | 改用 `location` / `map` / `try_files`；少用甚至别用 if |
| WebSocket 连上 60 秒就断 | `proxy_read_timeout` 默认 60s | 调到 3600s 以上，并用应用层 ping/pong 保活 |
| HTTPS 部分客户端（Java/Android）报证书错误 | 证书链不全，缺中间 CA | `ssl_certificate` 里合并 leaf + intermediate |
| CPU 高在 TLS 握手 | 没开会话复用；RSA 2048 握手昂贵 | `ssl_session_cache` + 换 ECDSA 证书 |
| 静态文件慢 | 未开 sendfile；或关闭了 open_file_cache 导致大量 stat 系统调用 | sendfile on + open_file_cache |
| error_log 报 `an upstream response is buffered to a temporary file` | 响应超过 proxy_buffers，写临时文件（IO 增加） | 调大 `proxy_buffers`/`proxy_buffer_size`；或用 `proxy_buffering off` 流式转发（会影响 upstream keepalive） |
| DNS 变更后 upstream 不生效 | Nginx 在启动/重载时解析域名并永久缓存 | 用变量 + `resolver`（如 `set $upstream "http://x.com"; proxy_pass $upstream;`），或商业版/nginx_upstream_check |

## 四、往下钻的入口

- **调试命令**：`nginx -T`（看最终配置）、`nginx -V`（看编译参数与模块）、`stub_status`、`ss -antp | grep nginx`
- **扩展延伸**：OpenResty（Lua 编程 Nginx）、APISIX / Kong（基于 Nginx/OpenResty 的 API 网关，含主动健康检查与服务发现）
- **源码瞄点**：`src/core/ngx_cycle.c`（配置解析与继承）、`src/http/ngx_http_upstream.c`（负载均衡轮询 `ngx_http_upstream_get_peer`）、`src/event/modules/ngx_epoll_module.c`、`src/core/ngx_connection.c`（`worker_connections` 限制就在 `ngx_event_process_init` 里）
- **官方文档真正值得读的段落**：Nginx Admin Guide → *Load Balancing*（比较实用）、`Module ngx_http_proxy_module` 的指令全表（排障时查这一页就够了）、Nginx Blog 的 *Inside NGINX: How We Designed for Performance*

## 五、达标自检

1. `worker_connections = 10240` 且 4 个 worker，最多能承载多少客户端？什么时候要除以 2？
2. `nginx -s reload` 为什么不断连接？老 worker 什么时候退出？
3. `location = /a`、`location ^~ /a/`、`location ~ ^/a`、`location /a` 同时存在时，请求 `/a/1` 和 `/a` 分别命中谁？
4. `root` 和 `alias` 在处理 `/img/a.jpg` 时路径分别是什么？
5. `proxy_pass http://b` 和 `proxy_pass http://b/` 在 location `/api/` 下的区别？再写成 `http://b/app/` 呢？
6. 不配 `proxy_set_header Host` 时后端 Host 是什么？
7. 怎么让后端拿到真实的客户端 IP？为什么不能直接信任 XFF？
8. `ip_hash` 有什么副作用？什么场景下不它而该用 Redis 存会话？
9. `proxy_next_upstream` 加 `http_500` 在什么情况下会出事？
10. `proxy_read_timeout` 与 `proxy_send_timeout` 的准确定义分别是什么？举个例子说明二者不同。
11. 一次请求耗时 3 秒，`$request_time=3.0`，`$upstream_response_time=0.1`，问题大概在哪？
12. `limit_req` 的 burst 加不加 `nodelay` 有什么区别？分别适合什么场景？
13. 上传 10MB 文件报 413，你要改哪个配置？如果改了还报可能是什么原因？
14. upstream 长连接要配哪几个指令？少配一个会怎样？
15. WebSocket 转发必须配什么？为什么还要调 timeout？
16. `ssl_certificate` 应该放什么内容？放错了什么表现？
17. 为什么要开 `ssl_session_cache`？它对性能的影响路径是什么？
18. 502 和 504 分别意味着什么？各自的排查方向是什么？
19. Nginx 二进制热升级的信号顺序是什么？
20. 如果 Nginx 后面挂的应用发布重启期间出现大量 502，你会怎么配置来缓解？
