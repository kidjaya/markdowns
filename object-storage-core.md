# 对象存储（OBS / OSS / S3）· 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
>
> 说明：本文的 OBS 指**对象存储服务**（Object Storage Service，华为云语义，兼容 AWS S3 API，与阿里云 OSS / 腾讯云 COS 同类）。若你指的是推流录制软件 OBS Studio，请另行说明。

## 一、核心心智模型

1. **对象存储本质是一个"无限容量的 HTTP KV 服务"，不是文件系统。** 扁平命名空间、通过 HTTP 读写、**不能就地修改只能整体覆盖**。所有"像操作文件一样操作它"的直觉都会踩坑——没有 rename、没有 append、没有目录（`/` 只是 key 里的普通字符）、没有文件锁。
2. **它解决的是"海量非结构化数据的成本与运维"问题，不是"性能"问题。** 靠 EC（纠删码）/多副本保证持久性（11 个 9）、靠 HTTP 提供无限扩展、靠分层存储压低成本。代价是延迟（几十到几百 ms）和语义贫乏。
3. **一致性与"你以为的一致性"是两回事。** 存储侧现在普遍强一致，但**你在前面加的 CDN、本地缓存、应用缓存会引入不一致**——排查不一致问题时，先分清是哪一层。
4. **权限的默认答案永远是"私有"。** 公共读 = 任何人可以刷你的流量账单。所有授权都应该走临时凭证（STS）+ 预签名 URL，永久 AK/SK 只允许出现在服务端。
5. **成本的全部秘密在三个地方：存储量 × 存储类型、请求次数、外网流量。** 海量小文件、频繁 LIST、走公网 Endpoint，是三个最常见的账单杀手。

## 二、必懂机制

### 2.1 什么时候用，什么时候不用

| 该用 | 不该用 |
|---|---|
| 图片/视频/附件/日志归档 | 需要随机写（改文件中间某几个字节） |
| 静态资源 + CDN 回源 | 需要 POSIX 语义（文件锁、硬链接、权限位） |
| 数据备份、冷归档 | 需要强一致的跨对象事务 |
| 大数据分析的数据湖 | 需要低延迟（<10ms）随机读 |
| 前端直传（跳过应用服务器） | 需要频繁的小文件随机读写（用本地盘/EBS） |

**与块存储/文件系统的根本差异**：

| | 块存储（EBS/云硬盘） | 文件系统（NFS） | 对象存储（S3） |
|---|---|---|---|
| 抽象单位 | 裸块 | 文件 + 目录树 | 对象（key + data + metadata） |
| 访问方式 | 挂载为设备 | 挂载为目录 | HTTP RESTful |
| 修改语义 | 任意字节随机写 | 随机写 | **整体覆盖（immutable）** |
| 元数据 | 无 | 有限（文件名/时间/权限） | **自定义 key-value，随对象存** |
| 扩展性 | 单机/单盘 | 受限于单套元数据 | 近乎无限 |

### 2.2 术语对照表

| AWS S3 | 阿里云 OSS | 华为云 OBS | 腾讯云 COS | 含义 |
|---|---|---|---|---|
| Bucket | Bucket | 桶 | Bucket | 命名空间 + 计费/权限单元 |
| Object | Object | 对象 | Object | 一个文件 |
| Key | ObjectName | 对象名 | Key | 对象在桶内的唯一标识（可含 `/`） |
| Region | Region | Region | Region | 地域 |
| Endpoint | Endpoint | Endpoint | Endpoint | 访问域名（区分内网/公网） |
| AccessKeyId/SecretAccessKey | AccessKeyId/Secret | AK/SK | SecretId/SecretKey | 永久密钥 |
| STS | STS | 临时访问密钥/委托 | STS | 临时凭证服务 |
| Multipart Upload | 分片上传 | 分段上传 | 分块上传 | 大文件分片 |
| Presigned URL | 签名 URL | URL 签名 | 预签名 URL | 临时访问链接 |
| Lifecycle | 生命周期 | 生命周期 | 生命周期 | 自动转换/删除规则 |
| Storage Class | 存储类型 | 存储类别 | 存储类型 | 标准/低频/归档 |

### 2.3 权限与凭证

**永久 AK/SK 的风险**：泄露即完全失控（可删桶、可刷流量）。规则：**AK/SK 只存在于服务端配置文件或 KMS，绝不进客户端、绝不进代码仓库、绝不进前端。**

**STS 临时凭证（正确姿势）**：
```
服务端 → AssumeRole(指定 Policy) → 拿到 {AccessKeyId, SecretAccessKey, SessionToken, Expiration}
      → 下发给客户端
      → 客户端用临时凭证直传对象存储
```
Policy 要限定到最小范围：
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/uploads/${userId}/*",
    "Condition": {
      "StringEquals": {"s3:x-amz-content-type": "image/jpeg"},
      "NumericLessThanEquals": {"s3:content-length-range": [0, 10485760]}
    }
  }]
}
```
- `Resource` 限定 key 前缀（防止用户改路径）
- `content-length-range` 限定大小（防止传超大文件）
- 有效期一般设 15~60 分钟，客户端要处理过期续期

**预签名 URL**：
```java
// AWS SDK v2
S3Presigner presigner = S3Presigner.builder().region(REGION).credentialsProvider(creds).build();
GetObjectPresignRequest req = GetObjectPresignRequest.builder()
    .signatureDuration(Duration.ofMinutes(15))
    .getObjectRequest(r -> r.bucket("b").key("k")).build();
PresignedGetObjectRequest url = presigner.presignGetObject(req);
```
- GET / PUT **都能签**，HEAD/PUT 签名可用于上传
- 有效期上限各家不同（S3 最长 7 天，受凭证有效期限制）
- **签名绑定具体操作和 key**，改一个字符就 `SignatureDoesNotMatch`
- 适合：临时下载授权、前端直传（后端下发签名 PUT URL）

**三层权限模型与优先级**：
1. **IAM/RAM Policy**（用户/角色级）—— 定义"谁能在哪些资源上做什么"
2. **Bucket Policy**（桶级资源策略）—— 定义"这个桶允许谁访问"，可跨账号授权
3. **ACL**（对象/桶级）—— 老式细粒度授权，**官方已不推荐**，默认关闭

判定原则：**默认全部拒绝 → 有显式 Allow 且无显式 Deny 才允许 → 显式 Deny 永远优先**。

**防盗链 Referer 白名单**：只能防君子（Referer 可伪造），**必须与签名 URL / CDN 鉴权配合使用**。

**加密**：
- **SSE-S3 / SSE-OBS**：服务端托管密钥，AES-256，最简单
- **SSE-KMS**：KMS 托管密钥，支持密钥轮换和审计，合规场景用
- **客户端加密**：数据在客户端加密后再传，服务端存密文。适合极高敏感数据（代价是失去服务端的所有处理能力）

### 2.4 一致性模型

- **AWS S3 自 2020 年 12 月起对 PUT / DELETE / LIST 全部强一致**：新对象写入后立即可读、覆盖写后立即返回新内容、LIST 立即反映变更。此前是 "read-after-write 一致 + list 最终一致"
- 各家云跟进情况不同，阿里云 OSS / 华为云 OBS 也提供强一致（覆盖写立即可读）
- **覆盖写没有中间态**：不会读到"写了一半"的对象
- **但 CDN / 缓存层引入的不一致不在存储侧**——你看到旧内容，先查 CDN 缓存 TTL、回源配置、浏览器缓存

**ETag 的坑（做完整性校验必踩）**：
- **单次 PUT 上传**：ETag = 文件内容的 MD5（32 位十六进制）
- **分片上传**：ETag = `各分片 MD5 拼接后的 MD5` + `-` + 分片数，形如 `"a1b2...-8"`
- **服务端加密（SSE-KMS）后**：ETag 不再是 MD5
- 所以**不能用 ETag 做分片上传文件的 MD5 校验**。正确做法：客户端自己算 MD5/CRC64，上传时通过 `Content-MD5` 头或自定义元数据带上；或用新版 SDK 的 Checksum（CRC32/CRC32C/SHA1/SHA256）功能

### 2.5 上传

**单次 PUT 上限 5GB**，超过必须分片。

**Multipart Upload 三步**：
```java
// 1. Initiate —— 拿到 uploadId
CreateMultipartUploadResponse init = s3.createMultipartUpload(r -> r.bucket(b).key(k));
String uploadId = init.uploadId();

// 2. UploadPart —— 分片编号 1~10000，每片 5MiB~5GB（最后一片可小于 5MiB）
//    并行上传，每片返回 ETag
List<CompletedPart> parts = ...;   // partNumber + ETag

// 3. Complete —— 服务端按 partNumber 顺序拼接
s3.completeMultipartUpload(r -> r.bucket(b).key(k).uploadId(uploadId)
     .multipartUpload(m -> m.parts(parts)));
```
关键约束：
- **分片编号 1~10000**，所以单个文件最大 = 10000 × 5GB = 50TB
- **除最后一片外每片至少 5MiB**（否则服务端拒绝）
- **Complete 是原子的**：要么成功（对象完整可见），要么失败（对象不存在）。不会有中间态
- **未完成的 multipart 会持续占用存储并计费，但在 LIST 里不可见** —— 这是隐形成本来源

**清理未完成分片**（必做）：
```java
s3.listMultipartUploads(r -> r.bucket(b))            // 列出未完成的
  .uploads().forEach(u -> s3.abortMultipartUpload(r -> r.bucket(b).key(u.key()).uploadId(u.uploadId())));
```
或配置生命周期规则自动清理：
```xml
<AbortIncompleteMultipartUpload><DaysAfterInitiation>7</DaysAfterInitiation></AbortIncompleteMultipartUpload>
```

**断点续传**：记录 `uploadId` + 已完成分片的 `{partNumber, ETag}` 列表（存 DB 或本地文件），重试时跳过已完成的分片。各家 SDK（S3 TransferManager、华为云 `ObsClient.uploadFile`）内置了断点续传和并发分片。

### 2.6 下载

```java
// Range 断点续传 —— 单位是字节，闭区间
GetObjectRequest req = GetObjectRequest.builder().bucket(b).key(k).range("bytes=0-1048575").build();

// 流式读取 —— 必须关闭！
try (ResponseInputStream<GetObjectResponse> is = s3.getObject(req)) {
    // 处理流
}   // 不 close 会耗尽连接池
```

**Content-Type 必须显式设置**：不设的话默认是 `application/octet-stream`，浏览器会**下载而不是预览**（图片打不开就是这个原因）。
**Content-Disposition**：`inline`（浏览器内预览）或 `attachment; filename="xxx"`（强制下载）。

**CDN 回源**：对象存储作为 CDN 源站。
- 回源鉴权：配置签名 URL / Cookie / Header 鉴权，防止绕过 CDN 直刷源站
- 缓存刷新：对象更新后要主动刷新 CDN 缓存（或等 TTL 过期）
- 回源流量也是钱，注意缓存命中率

**大文件流式下载**：绝不能 `readAllBytes()` 全读进内存。用流 + 缓冲区边读边写（`response.getOutputStream()`）。

### 2.7 生命周期与成本

**存储类型分层**：

| 类型 | 特点 | 最小存储天数 | 取回 |
|---|---|---|---|
| 标准 Standard | 高频访问，毫秒级 | 无 | 直接读 |
| 低频 IA | 单价约为标准 60%，有**数据取回费** | 通常 30 天 | 直接读，收流量费 |
| 归档 Archive | 单价约为标准 20%，**不可直接读** | 通常 60~90 天 | 需先 `Restore`，分钟级~小时级 |
| 深度归档 Deep Archive | 最便宜 | 通常 180 天 | `Restore`，数小时~12 小时 |

**提前删除仍按最小存储天数计费** —— 把文件转归档后发现不用了立刻删，照样收 90 天的钱。这是最常见的成本坑之一。

**生命周期规则三类**：
- **Transition**：N 天后转换存储类型（`Days: 30 → IA`，`Days: 90 → Archive`）
- **Expiration**：N 天后删除（日志、临时文件）
- **AbortIncompleteMultipartUpload**：N 天后清理未完成分片（**强烈建议配**）

**成本杀手清单**：
1. **外网下行流量**：通常是最贵的一项。内网/VPC Endpoint 走内网免流量费
2. **请求次数**：海量小文件（百万级）场景下 GET/PUT 请求费可能超过存储费。解法：合并小文件打包、缓存 LIST 结果、减少轮询
3. **跨区域复制 CRR**：双份存储 + 跨区流量
4. **未完成的分片上传**
5. **归档数据提前删除 / 频繁读取**
6. **版本控制开着又不清理旧版本**：每个覆盖写都留一份历史，容量翻倍增长
7. **走了公网 Endpoint 而不是内网 Endpoint**：既有流量费又有 NAT 网关瓶颈

### 2.8 高级特性

**版本控制 Versioning**：
- 开启后每次覆盖写产生新版本，`DELETE` 只是写一个 **Delete Marker**（不是真删）
- 好处：误删可恢复、有完整审计轨迹
- 代价：容量持续增长（必须配合生命周期规则清理非当前版本 NoncurrentVersionExpiration）
- **MFA Delete**：删除需要 MFA，防止误操作

**Object Lock（WORM）**：合规保留，对象在指定时间内不可删除不可修改。金融/医疗合规场景必需。

**Event Notification**：对象创建/删除时触发通知到 MQ / 函数计算 / HTTP 回调。
- **注意是"至少一次投递"，可能重复**，消费端必须幂等
- 事件可能乱序
- 与 LIST 相比，事件通知是更高效的感知方式（避免轮询）

**CORS**：前端直传必配。
```xml
<CORSConfiguration><CORSRule>
  <AllowedOrigin>https://example.com</AllowedOrigin>
  <AllowedMethod>PUT</AllowedMethod><AllowedMethod>GET</AllowedMethod>
  <AllowedHeader>*</AllowedHeader>
  <ExposeHeader>ETag</ExposeHeader>   <!-- 关键：前端要读 ETag 必须暴露 -->
  <MaxAgeSeconds>3000</MaxAgeSeconds>
</CORSRule></CORSConfiguration>
```

**其他**：静态网站托管（Index/Error 文档）、S3 Select（服务端按 SQL 过滤，省流量）、Inventory（每日对象清单，用于审计和大批量核对）、S3 Batch Operations。

### 2.9 前端直传架构（推荐）

```
1. 前端请求上传凭证
      ↓
2. 后端校验用户身份 → 调用 STS AssumeRole（Policy 限定 key 前缀、大小、content-type）
      → 返回 {临时 AK/SK/Token, bucket, key, endpoint, 有效期}  或直接返回预签名 PUT URL
      ↓
3. 前端用凭证直接 PUT 到对象存储（不过应用服务器）
      ↓
4. 前端上传成功 → 回调后端 /upload/callback?key=xxx
      ↓
5. 后端验签/查对象元数据确认存在 → 写库记录 → 返回业务 ID
```

**为什么要直传**：文件过应用服务器会占用双倍带宽（进+出）、占内存（多用户并发上传可能 OOM）、拖垮应用节点。

**安全要点**：
- Policy 限定 key 前缀到 `uploads/{userId}/`，防止覆盖他人文件
- 限定 `content-length-range` 防超大文件
- 限定 `content-type` 防上传可执行脚本（XSS 风险）
- 后端回调时必须**校验对象真实存在且大小符合**（用 HeadObject），不能只信前端汇报
- 凭证有效期尽量短

### 2.10 Java SDK 要点

**AWS SDK v2（兼容 S3 的厂商都能用）**：
```java
S3Client s3 = S3Client.builder()
    .region(Region.of("cn-north-4"))
    .endpointOverride(URI.create("https://obs.cn-north-4.myhuaweicloud.com"))  // 兼容厂商
    .credentialsProvider(StsClientProvider.create())   // 临时凭证，自动刷新
    .httpClientBuilder(ApacheHttpClient.builder()
        .maxConcurrency(200)                            // 默认 50，高并发必调
        .connectionAcquisitionTimeout(Duration.ofSeconds(10))
        .connectionTimeout(Duration.ofSeconds(5))
        .socketTimeout(Duration.ofSeconds(30)))
    .overrideConfiguration(c -> c
        .apiCallTimeout(Duration.ofSeconds(60))
        .retryStrategy(RetryStrategy.builder().maxAttempts(3).build()))  // 必须显式设，否则可能无限重试
    .build();
```

**华为云 OBS SDK**：
```java
ObsConfiguration conf = new ObsConfiguration();
conf.setEndPoint("https://obs.cn-north-4.myhuaweicloud.com");
conf.setSocketTimeout(30000);
conf.setConnectionTimeout(10000);
conf.setMaxConnections(1000);          // 默认 1000
conf.setIdleConnectionTime(30000);
conf.setMaxErrorRetry(3);
ObsClient obsClient = new ObsClient(ak, sk, conf);
```

**连接池耗尽是最常见的生产事故**：
- 现象：上传/下载全部超时，但服务端监控正常，本机 CPU 空闲
- 根因：拿到 `InputStream` 没有 `close()`，连接不归还池中，池被耗尽
- 解法：**必须 try-with-resources**；调大 `maxConnections`；监控池使用率

**其他要点**：
- **重试和超时必须显式配置**：默认可能是 3 次重试 + 无超时，遇到网络故障会长时间挂起线程
- 分片大小建议 5~10MB（兼顾并发度与重试成本）
- `TransferManager` / `ObsClient.uploadFile` 内置并发分片 + 断点续传，优先用
- 客户端要计算 MD5/CRC64 并与服务端比对（防静默损坏）

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| `SignatureDoesNotMatch` | ① 本机时间偏移超过 15 分钟（签名含时间戳）；② 中文/特殊字符 key 未 URL 编码；③ 请求头参与了签名但被代理改写；④ Host 头被改（走了错误的 Endpoint） | 开 NTP 时间同步；key 做 URL 编码；检查 Endpoint 与 Region 是否匹配；代理不要改写参与签名的头 |
| 有权限却返回 404 而不是 403 | 没有 `ListBucket` 权限时，`HeadObject` 对不存在和"无权限"都返回 404（**故意的**，避免探测桶内容） | 先确认有 `s3:GetObject`；用 `ListObjectsV2` 验证；排权限时不要用 HEAD 判断存在性 |
| 图片在浏览器里是下载而不是预览 | 上传时没设 `Content-Type`，默认 `application/octet-stream` | 上传时显式设置；或批量修改对象元数据 |
| 上传/下载全部超时，服务端正常 | **连接池耗尽**——`InputStream` 没有 close，连接不归还 | 全部改 try-with-resources；调大 maxConnections；监控池使用率 |
| 前端直传报 CORS 错误 | 桶未配 CORS，或 CORS 里没暴露 `ETag` | 配 `AllowedOrigin/Method/Header`；`ExposeHeader: ETag` |
| 前端拿不到 ETag | CORS 的 `ExposeHeader` 没配 | 加上 `<ExposeHeader>ETag</ExposeHeader>` |
| 分片上传后 ETag 与文件 MD5 对不上 | 分片上传的 ETag 是 `md5-of-md5s-N`，不是文件 MD5 | 客户端自算 MD5 存元数据；或用 Checksum 功能 |
| 账单莫名暴增 | ① 桶被设为公共读，被爬虫刷流量；② 大量 LIST 请求；③ 走了公网 Endpoint；④ 未完成分片未清理 | 桶设私有；走 CDN；用内网 Endpoint；配生命周期清理分片；开流量告警 |
| 用了内网 Endpoint 却连通失败 | 内网 Endpoint 只在同 Region 的 VPC 内可达，ECS 与桶不在同 Region 或没配 VPC Endpoint | 确认 Region 一致；配置网关型 VPC Endpoint；或改用公网 |
| NAT 网关带宽打满 | 大量流量走了公网 Endpoint | 改内网 Endpoint / VPC Endpoint；或走 CDN |
| 归档文件读取报 `InvalidObjectState` | 归档类型必须先 `Restore` 才能读 | 先发 `RestoreObject`，等待（分钟到小时级）后再读；加急档位要额外付费 |
| 删了文件但容量没降 | ① 开了版本控制，删除只写 Delete Marker；② 生命周期规则还没触发；③ 未完成分片未清理 | 清理非当前版本；`AbortMultipartUpload`；配 NoncurrentVersionExpiration |
| 覆盖写后读到的还是旧内容 | **不是存储侧问题**——CDN 缓存 / 浏览器缓存 / 应用本地缓存 | 刷新 CDN；URL 加版本号；检查 Cache-Control |
| 想 rename 对象 | 对象存储**没有 rename 原语** | 必须 `CopyObject` + `DeleteObject`。大对象复制耗时（跨区域更慢），且非原子（中途失败会留下副本） |
| 事件通知重复触发业务逻辑 | 事件通知是**至少一次**投递，可能重复也可能乱序 | 消费端幂等（用对象 key + versionId 做去重键）；不要依赖事件顺序 |
| 跨区域复制后两边不一致 | CRR 是异步的，有延迟；且复制失败的对象需要人工重试 | 监控 `ReplicationLatency`；用 RTC（复制时间控制）或定期用 Inventory 对账 |
| 密钥泄露 | AK/SK 硬编码进代码/配置文件并推到仓库 | 立即吊销密钥（**先吊销再清理历史**）；改用 IAM 角色 / 实例角色 / STS；仓库加密钥扫描 |
| 大批量上传触发限流 | 单前缀请求速率上限（S3 是 3500 PUT/s + 5500 GET/s per partitioned prefix） | key 前缀做散列打散（如用 hash 前缀而非时间戳）；新版自适应能力已缓解但大批量时仍要注意 |
| 百万级小文件请求费用超过存储费 | 请求次数单独计费 | 小文件合并打包；用 CDN；减少 LIST 轮询（改事件通知） |
| SDK 调用长时间挂起不返回 | 未配置超时/重试策略，默认行为在网络故障下会无限等待 | 显式设 `apiCallTimeout` / `socketTimeout` / `maxAttempts` |

## 四、往下钻的入口

**官方 API 名（排障时直接对应）**：
- `PutObject` / `GetObject` / `HeadObject` / `DeleteObject` / `CopyObject` / `ListObjectsV2`
- `CreateMultipartUpload` / `UploadPart` / `CompleteMultipartUpload` / `AbortMultipartUpload` / `ListMultipartUploads` / `ListParts`
- `PutBucketLifecycleConfiguration` / `PutBucketCors` / `PutBucketNotificationConfiguration` / `PutBucketVersioning`
- `RestoreObject`（归档取回）/ `SelectObjectContent`（S3 Select）/ `GetBucketInventoryConfiguration`

**SDK 源码入口**：
- `software.amazon.awssdk.services.s3.S3Client`（AWS v2）
- `software.amazon.awssdk.services.s3.presigner.S3Presigner`
- `software.amazon.awssdk.services.sts.StsClient` / `StsAssumeRoleCredentialsProvider`
- `software.amazon.awssdk.transfer.s3.S3TransferManager`（并发分片 + 断点续传）
- 华为云：`com.obs.services.ObsClient` / `ObsConfiguration` / `com.obs.services.model.UploadFileRequest`
- 连接池：`software.amazon.awssdk.http.apache.ApacheHttpClient.Builder`

**命令行工具（排障/批量操作）**：
```bash
aws s3 ls s3://bucket/prefix/ --recursive --summarize
aws s3 cp local.txt s3://bucket/key --storage-class GLACIER
aws s3api head-object --bucket b --key k
aws s3api list-multipart-uploads --bucket b          # 查未完成的上传
aws s3api abort-multipart-upload --bucket b --key k --upload-id xxx
aws s3api put-bucket-lifecycle-configuration --bucket b --lifecycle-configuration file://lc.json
aws s3 presign s3://bucket/key --expires-in 3600
# 华为云：obsutil；阿里云：ossutil；腾讯云：coscli
```

**监控与审计**：
- CloudTrail / 操作审计（记录所有 API 调用，安全事件必备）
- CloudWatch / 云监控指标：`BucketSizeBytes`、`NumberOfObjects`、`AllRequests`、`GetRequests`、`BytesDownloaded`
- 访问日志（Access Logging）：记录每个请求，用于分析访问模式和排查异常流量
- **必须配置**：账单告警、流量突增告警、请求量突增告警

**官方文档具体章节**：
- AWS S3 Developer Guide → *Uploading and Downloading Objects*、*Multipart Upload Overview*、*Lifecycle Configuration Elements*、*Storage Classes*、*Presigned URLs*
- AWS S3 API Reference → *Common Errors*（错误码速查）
- 华为云 OBS 文档 → *服务端加密*、*生命周期管理*、*临时授权访问*

## 五、达标自检

1. 对象存储和文件系统的根本差异是什么？为什么没有 rename 操作？要改名怎么做？
2. 什么场景下不应该用对象存储？举出至少 4 种，并说明该用什么替代。
3. 永久 AK/SK 和 STS 临时凭证，各自的适用场景和风险是什么？生产环境应该怎么用？
4. 前端直传时，STS Policy 至少要限定哪几项？分别防的是什么？
5. Bucket Policy、IAM Policy、ACL 三者的优先级关系是什么？显式 Deny 的地位如何？
6. 预签名 URL 的签名依赖哪些要素？为什么改一个字符就会失败？它和 STS 凭证该怎么选？
7. **分片上传的 ETag 和单次上传的 ETag 有什么不同？为什么不能用 ETag 校验文件完整性？**
8. Multipart Upload 的分片编号范围、单片大小限制分别是什么？算出单个文件的理论最大值。
9. **未完成的 multipart 上传有什么后果？怎么发现和清理？**
10. 覆盖写是原子的吗？会不会读到"写了一半"的对象？
11. 存储侧已是强一致，但用户还是读到了旧内容——可能的原因有哪些？按什么顺序排查？
12. Content-Type 不设会带来什么具体问题？Content-Disposition 的两个取值分别有什么效果？
13. 四种存储类型的单价、最小存储天数、取回方式分别是什么？归档文件直接读会报什么错？
14. 生命周期规则有哪三类？为什么强烈建议配 `AbortIncompleteMultipartUpload`？
15. 列出至少 5 个对象存储的成本杀手，并给出对应的优化手段。
16. 版本控制开启后，执行一次 `DELETE` 实际发生了什么？容量会怎样变化？如何控制成本？
17. 事件通知是精确一次投递吗？消费端要做什么防护？
18. 大文件下载时为什么要流式处理？`InputStream` 不 close 会导致什么后果？
19. `SignatureDoesNotMatch` 的常见根因有哪些？至少说出 4 个，并给出排查顺序。
20. **为什么无权限时 `HeadObject` 返回 404 而不是 403？这对权限排障有什么影响？**
21. 走公网 Endpoint 和内网 Endpoint 在成本和架构上有什么差异？什么情况下内网 Endpoint 会连不通？
22. 前端直传后，后端回调时应该做什么校验？为什么不能只信前端汇报的结果？
23. 海量小文件场景下，请求费用可能超过存储费用，有哪些优化手段？
24. 你的服务调用对象存储 SDK 时，必须显式配置哪些参数？不配置会有什么风险？
