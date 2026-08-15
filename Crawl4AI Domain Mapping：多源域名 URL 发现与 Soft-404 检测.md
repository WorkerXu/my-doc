# Crawl4AI Domain Mapping：多源域名 URL 发现与 Soft-404 检测

## 1. 调研对象与结论

调研对象：Crawl4AI `DomainMapper` 及其配套的多 URL Dispatcher。

- 官方文档：`https://docs.crawl4ai.com/core/domain-mapping/`
- 上游项目：`https://github.com/unclecode/crawl4ai`
- 重点源码：`crawl4ai/domain_mapper.py`、`crawl4ai/async_dispatcher.py`
- 重点版本信息：v0.8.7 引入 DomainMapper，并同时包含 Docker API 安全加固；当前 README 已进入 v0.9.x 系列。

核心结论：**DomainMapper 很适合作为“Source 接入勘探 + Coverage Gap 补洞”的运行时组件，但不能直接作为 1000 站知识库的 Coverage Truth、持久 Frontier、合规边界或全局限流器。**

对现有技术方案最值得吸收的有四点：

1. 把域名发现从单一 Sitemap 扩展为多源证据并集，尤其补入 Common Crawl / Wayback 历史索引；
2. 把 Soft-404 识别做成一等能力，避免 SPA/自定义 404 把大量不存在页面误判为文章；
3. 在 Worker 内采用 Crawl4AI 的 MemoryAdaptiveDispatcher 做资源自保护，同时继续由平台层做跨 Worker 的公平调度和域名限流；
4. 把 Crawl4AI 本地缓存、内存队列、内存 RateLimiter 都降级为运行时优化，PostgreSQL + Object Storage 继续作为业务真相。

另外必须做反向约束：DomainMapper 默认能力面向“域名侦察”，其中证书透明度、子域猜测、`/admin`/`/login` 等通用路径探测并不适合博客知识库生产抓取。知识库系统必须把它收敛到**已批准内容域、已批准路径和 robots/access policy**之内。

---

## 2. DomainMapper 的实现结构

`DomainMapper.scan()` 的整体实现可以抽象为三阶段：

```text
Phase 1 Host Discovery
    -> Phase 2 Per-host Scanning
        -> Phase 3 Normalize / Dedup / Validate / Score
```

### 2.1 Phase 1：Host Discovery

源码中的 `_discover_hosts()` 从多个来源发现 hostname：

- `crt`：查询 crt.sh 证书透明度数据；
- `wayback`：查询 Internet Archive CDX；
- `cc`：复用 `AsyncUrlSeeder` 查询 Common Crawl；
- `dns`：对常见子域前缀做 DNS 猜测；
- 根域本身始终进入候选。

随后 `_validate_hosts()` 会对候选 host 发 HTTP HEAD，确认其可达，并记录重定向目标。

这里有一个对知识库架构非常重要的边界：**“发现了一个子域”不等于“允许主动访问这个子域”。** DomainMapper 原生实现会在发现后主动验证，因此生产系统不能在未批准的 eTLD+1 上直接开启全量子域发现再自动验证。

建议平台拆成两种模式：

```text
Onboarding Metadata Recon
- 只生成 host_candidate
- 不自动进入 Source approved_domains
- 不自动 Fetch 正文

Approved Content Recon
- 只对 approved_domains / approved_hosts 运行 HTTP/Sitemap/Feed/Homepage
- 结果才允许进入 URL Observation
```

如果只是对一个明确批准的博客域做接入，直接固定 `include_subdomains=false` 更安全。

### 2.2 Phase 2：Per-host Scanning

每个 host 先做 Soft-404 指纹，然后并行运行启用的来源：

```text
robots.txt
sitemap
probe
feed
homepage
wayback URL merge
```

各 source 使用独立 timeout，并通过 `asyncio.gather(..., return_exceptions=True)` 并行执行。一个 source 超时或失败不会拖垮整次 map。

这种“Provider 失败隔离”很值得保留到平台层：

```text
Sitemap failure != Source failure
Feed failure != Source failure
Wayback timeout != Backfill failure
```

平台应该分别记录 `provider_run` 状态，在 Coverage Reconcile 阶段再判断整体是否存在 Known Gap。

### 2.3 Phase 3：Post Processing

`_normalize_and_dedup()` 对 URL 做规范化并按规范化结果去重；同一 URL 被多个来源发现时，会把 `source` 字符串合并，例如：

```text
homepage+sitemap+wayback
```

随后可执行：

- nonsense URL 过滤；
- `<head>` 提取；
- BM25 相关性打分；
- `max_urls` 截断。

对通用工具这是合理的，但对需要可审计 Coverage 的知识库平台来说，**不能只保存最终 dedup 结果**。必须把每个来源的 Observation 分开保存：

```text
url_observation
- url_id
- provider_type
- provider_run_id
- raw_url
- normalized_url
- evidence
- observed_at
```

因为“同一 URL 被 Sitemap、Wayback 和 Homepage 同时发现”本身就是 Coverage 证据。

---

## 3. 8 类发现源对知识库的价值

DomainMapper 文档提供八类发现源：

```text
sitemap
cc
wayback
crt
probe
robots
feed
homepage
```

但它们在博客知识库里的角色不同，不能全部等价对待。

### 3.1 sitemap：权威历史入口之一

价值最高。很多博客会提供 sitemap index、按年份拆分的 post sitemap 或 CMS 自动生成 sitemap。

平台中应继续作为 Authoritative Provider，支持：

- sitemap index 递归；
- gzip；
- lastmod；
- 增量 cursor / ETag；
- 站点多 sitemap 合并；
- URL Observation 来源证据。

DomainMapper 可用于接入期快速探测，但生产同步应由独立 Sitemap Provider 执行，便于 cursor、重试、审计和增量调度。

### 3.2 Common Crawl：历史补洞 Provider

Common Crawl 对“站点当前 sitemap 已经不包含旧文章”的情况很有价值。

适合作为：

```text
external_archive_gap / common_crawl_gap
```

而不是权威来源。

原因：

- 公共索引覆盖不是完整历史；
- 索引有时滞；
- 可能包含已经删除、重定向或错误 URL；
- 结果仍需要回源验证。

平台应该保存：

```text
provider = common_crawl
archive_index
first_seen / last_seen if available
original_url
```

并进入正常 URL Identity / Fetch / Tombstone 流程。

### 3.3 Wayback CDX：全历史回填的重要补洞来源

DomainMapper 当前实现对 `*.domain/*` 查询 Wayback CDX，并使用 `collapse=urlkey`、`limit=10000`。

这带来一个直接限制：**单次 DomainMapper 查询不能证明大站历史完整性**。对于 URL 数大于 1 万、存在复杂时间跨度或 CDX 分页的站点，需要平台级 Wayback Provider 自己实现分页/时间窗口切片，例如：

```text
按年份窗口
按 URL prefix
按 CDX page/resumeKey
```

因此方案中应把 Wayback 作为独立 Provider，而不是只调用一次 `DomainMapper.scan()`。

### 3.4 crt：只能用于“候选内容 host”发现

证书透明度能发现 `blog.example.com`、`docs.example.com` 之类内容子域，但也可能发现：

```text
admin
staging
api
internal-like hosts
```

知识库系统不得因为 crt.sh 返回了 hostname 就自动访问。

正确流程：

```text
CT metadata
 -> host_candidate
 -> content-host classifier / human review
 -> approved_host
 -> 才允许网络访问
```

### 3.5 probe：必须替换默认探测路径

DomainMapper 的默认 `DEFAULT_PROBE_PATHS` 面向通用域名侦察，包含：

```text
/docs
/api
/login
/dashboard
/blog
/openapi.json
/swagger.json
/graphql
/status
health
...
```

知识库不应该去探测登录、管理、调试或服务端接口路径。

应提供自己的 `ContentProbeProfile`，只允许低风险内容入口，例如：

```text
/blog
/posts
/articles
/news
/engineering
/tech
/archive
/archives
/feed
/rss
```

而且 Probe 仅在已批准 host 上运行。

### 3.6 robots：应作为访问策略，不作为“可抓路径清单”

DomainMapper 会解析 `robots.txt` 的 `Disallow:` / `Allow:`，并把具体路径加入 probe 候选。

对于知识库平台，这个语义必须改掉：**Disallow 不是邀请，更不是发现后可以访问的路径。**

平台应把 robots 解析结果进入：

```text
access_policy
- allowed
- disallowed
- crawl_delay
- sitemap_directives
- policy_release
```

其中 `Sitemap:` 可以作为发现入口；`Disallow:` 只用于拒绝/限制，不用于主动抓取。

### 3.7 feed：首页增量的重要信号

RSS/Atom 对博客增量同步价值很高，但通常只包含最近 N 篇，所以：

```text
Feed = Incremental first-class provider
Feed != Full-history proof
```

接入时用 DomainMapper 探测 Feed，确认后交给独立 RSS/Atom Provider 持续运行。

### 3.8 homepage：适合浅层补洞和增量探测

源码通过普通 HTTP 获取首页，使用 `quick_extract_links()` 提取内部链接，并额外读取 `<head>` 中的：

```text
alternate
preload
prefetch
next
prev
```

这对发现分页、Feed 和最新文章有价值；默认不需要 Browser。

建议平台保留“HTTP homepage probe”作为低成本增量信号，只有检测到 JS shell 才进入 Browser slow path。

---

## 4. Soft-404 的实现原理与可优化点

### 4.1 当前实现

DomainMapper 会先访问一个随机不存在的 URL：

```text
/c4ai-probe-<random>
```

并记录：

```text
status_code
title
content_length
body_hash = md5(first 2048 bytes)
```

若不存在页面仍返回 HTTP 200，就认为该 host 可能存在 Soft-404。

后续 `_is_soft_404()` 判断：

1. 目标状态必须是 200；
2. 前 2048 bytes hash 与指纹一致，判 Soft-404；
3. 或 `<title>` 与指纹 title 一致，也判 Soft-404。

此外，当 sitemap 返回很多 URL、host 又存在 Soft-404 时，源码会随机抽样最多 5 个 sitemap URL；如果样本全部命中 Soft-404，就跳过整批 sitemap URL。

### 4.2 为什么对知识库很重要

很多 SPA、Next.js fallback、CMS 自定义 404 会出现：

```text
GET /real-article     -> 200 + app shell
GET /does-not-exist   -> 200 + same app shell
```

如果只看 HTTP 200，会造成：

- URL Inventory 膨胀；
- Fetch/Browser 成本暴涨；
- 大量“空文章”进入 REPAIR；
- Coverage 看起来虚高；
- 增量同步重复处理不存在页面。

因此现有方案应该新增显式的：

```text
host_validity_profile
soft_404_signature
url_validity_check
```

### 4.3 平台侧需要比上游更稳健

只比较前 2KB hash/title 对部分动态 404 不够。例如随机 request-id、时间戳、A/B 文案会改变 hash。

建议平台扩展为多特征指纹：

```text
status_code
normalized_title
normalized_text_simhash
content_length_bucket
dom_structure_hash
canonical_target
main_text_length
```

建立 2~3 个随机不存在 URL 的基线，而不是一个样本。

建议判定：

```text
SOFT_404_HIGH_CONFIDENCE
SOFT_404_SUSPECTED
VALID
UNKNOWN
```

“疑似”不能直接永久删除 URL，而应保存 Observation 和 reason_code。

### 4.4 Sitemap 整批跳过需要更保守

上游采用“5 个样本全是 Soft-404 -> 跳过整批”的执行优化，对通用工具很实用，但平台不应因此丢 Coverage 证据。

改为：

```text
保存全部 sitemap Observation
 -> 标记 host/sitemap batch 为 SOFT_404_SUSPECTED
 -> 暂缓 Fetch admission
 -> 增加抽样或人工确认
```

即：**可以延期抓取，不能让已发现 URL 从 Inventory 消失。**

---

## 5. 并发、资源保护与限流实现

### 5.1 MemoryAdaptiveDispatcher

`async_dispatcher.py` 中的 `MemoryAdaptiveDispatcher` 维护：

```text
memory_threshold_percent
critical_threshold_percent
recovery_threshold_percent
max_session_permit
fairness_timeout
memory_wait_timeout
PriorityQueue
```

运行逻辑：

- 内存低于阈值时尽量填满 `max_session_permit`；
- 达到 memory pressure 后暂停领取新任务；
- critical 状态下把任务重新排队；
- 长时间等待的任务通过 fairness score 提高优先级；
- streaming generator 关闭时会 cancel 并 await 活跃任务，防止遗留 Browser page/context。

这个设计非常适合当 **Worker Local Admission Controller**。

### 5.2 为什么不能替代平台 Scheduler

Dispatcher 的队列、memory state、domain delay 都是进程内状态。

Worker 崩溃后它们会丢失，且多 Worker 之间不可见。因此：

```text
PostgreSQL Frontier / Task = 业务任务真相
Redis = 分布式运输与全局 admission
MemoryAdaptiveDispatcher = 单 Worker 资源保护
```

推荐链路：

```text
Fair Scheduler
 -> Domain Token Bucket
 -> Worker Lease Slice (例如 50~200 URL)
 -> Crawl4AI arun_many(stream=true)
 -> MemoryAdaptiveDispatcher
 -> Result-by-result durable commit
```

不要把百万 URL 一次塞给 `arun_many()`。

### 5.3 Crawl4AI RateLimiter 的定位

`RateLimiter` 在内存中按 domain 记录：

```text
last_request_time
current_delay
fail_count
```

遇到 429/503 后做指数退避 + jitter，成功后逐渐降低 delay。

它很适合作为 Worker 内的第二道保护，但无法协调多个 Worker，因此仍需要平台级 Redis Lua Token Bucket / Semaphore。

### 5.4 DomainMapper 的 `hits_per_sec` 不能当严格全局 RPS

DomainMapper 源码里 `hits_per_sec` 被用于创建 `asyncio.Semaphore`，这本质上限制的是**同时在途请求数**，并不严格等价于“每秒请求次数”。

因此知识库方案不能把这个参数当作跨 Worker 的精确速率限制。

应保持：

```text
Global per-domain token bucket = 真正节流
Worker local semaphore = 并发保护
Crawler RateLimiter = 429/503 自适应退避
```

三者职责不同。

---

## 6. Cache 的实现与平台边界

DomainMapper 使用本地文件目录：

```text
~/.crawl4ai/domain_mapper_cache
```

缓存 key 由 domain + 部分 config 生成，并由 `cache_ttl_hours` 控制过期。

这类 cache 有三个问题：

1. 多 Worker 不共享；
2. Worker 被替换后可能丢失；
3. cache key 不是平台发布版本和业务语义的完整表达。

因此只把它当运行时缓存。

平台应自己保存：

```text
provider_run
provider_result_artifact
provider_cursor
url_observation
coverage_snapshot
```

如果需要缓存 Common Crawl / Wayback 查询结果，写 Object Storage，并记录 query、release、hash、captured_at。

---

## 7. 对现有技术方案的具体优化

### 7.1 新增 Provider 类型

在原有 Provider 基础上明确加入：

```text
domain_recon
common_crawl_gap
wayback_gap
soft404_probe
```

其中 `domain_recon` 只用于接入期和周期性结构变化探测，不直接表示文章 Coverage。

### 7.2 新增 Host Candidate / Approved Host 边界

建议模型：

```text
host_candidate
- id
- source_id
- hostname
- discovered_by
- evidence_object_key
- state  # DISCOVERED/APPROVED/REJECTED
- first_seen_at
- reviewed_at

source_host_scope
- source_id
- hostname
- scope_type  # CONTENT/ASSET/EXCLUDED
- enabled
- release_id
```

只有 `APPROVED + CONTENT` host 才能进入主动 Fetch。

### 7.3 新增 Soft-404 Profile

```text
host_validity_profile
- source_id
- hostname
- probe_release_id
- status_behavior
- signature_object_key
- confidence
- captured_at
- expires_at
```

URL 验证结果：

```text
url_validity_observation
- url_id
- artifact_id
- decision
- similarity
- reason_code
- profile_id
```

### 7.4 接入流程优化

原接入流程升级为：

```text
1. Exact-host robots/sitemap/feed probe
2. CMS / archive pattern probe
3. Safe content-path homepage probe
4. Optional metadata-only subdomain discovery
5. Human approve content hosts
6. Common Crawl + Wayback historical gap inventory
7. Soft-404 baseline
8. URL pattern / query policy sampling
9. HTTP vs Browser route sampling
10. Publish Source Profile Release
```

### 7.5 Backfill 优化

推荐新顺序：

```text
Authoritative Providers
    CMS/API + Sitemap + Feed
 -> Historical Gap Providers
    Archive + Common Crawl + Wayback
 -> Safe Domain Recon Gap
 -> Deep Crawl Prefetch Gap
 -> Coverage Reconcile
```

Common Crawl / Wayback 发现的 URL 先进入 Observation/Inventory，再进行 live Fetch 验证。

### 7.6 Worker 执行层优化

浏览器 Worker 采用：

```text
平台 Slice
 -> arun_many(stream=true)
 -> MemoryAdaptiveDispatcher
 -> max_session_permit 按容器内存标定
 -> 每个 result 立即写 Artifact/状态
```

不要等整批结果结束后再持久化。

Worker 退出、stream close、cancel 时必须验证：

- 活跃 task 已 cancel/await；
- Browser page/context 没有泄漏；
- lease 未完成任务回到 RETRY/READY；
- 本地 dispatcher queue 不作为恢复依据。

---

## 8. 安全与合规注意事项

Crawl4AI v0.8.7 的发布记录集中修复了 Docker API 的 RCE、SSRF、认证绕过、任意文件写入、XSS、硬编码 JWT secret 等问题。这说明把 crawler 作为网络服务暴露时，**请求体本身必须被视为不可信输入**。

平台建议：

1. Crawl4AI 版本必须 pin 到经过验证的 immutable runtime release，不使用浮动 `latest`；
2. Docker API 只放内网，优先通过内部 Worker 调用，不对公网开放；
3. Source URL 必须先通过平台 SSRF 校验，再交给 Crawl4AI；
4. 禁止访问 loopback、RFC1918、link-local、云 metadata、Unix/file URL；
5. JS hook、execute-js、下载路径、webhook 等能力默认关闭，确需启用时按 Recipe allowlist；
6. UI 用户不能直接传任意 CrawlerRunConfig 到 Worker；必须引用已发布的配置 release；
7. `robots.txt Disallow` 只做拒绝，不作为 probe 候选；
8. 子域 discovery 只生成候选，未经批准不得主动抓取。

---

## 9. 最终建议

DomainMapper 不应该取代现有的 Source Provider、Persistent Frontier 和 Coverage Ledger，而应该被放在两个位置：

```text
A. Source Onboarding Recon
   帮助发现 Sitemap / Feed / 内容子域 / Soft-404 行为

B. Coverage Gap Provider
   用 Common Crawl / Wayback / Homepage 等补权威 Provider 的盲区
```

真正的生产链仍然是：

```text
DomainMapper / Provider Runtime
 -> 原始发现证据
 -> PostgreSQL URL Observation
 -> URL Identity
 -> Admission / Persistent Frontier
 -> HTTP First / Browser Slow Path
 -> Immutable Artifact
 -> Extraction Portfolio
 -> Quality Gate
 -> Canonical IR
 -> Markdown Projection
```

本次调研最重要的方案增强是：

> **把“域名级多源发现”和“Soft-404 基线”正式纳入 Source 接入与 Coverage Gap 流程；把 Crawl4AI MemoryAdaptiveDispatcher 下沉为 Worker 本地资源控制；同时进一步强化 approved host、robots policy、全局限流和 PostgreSQL 业务真相边界。**
