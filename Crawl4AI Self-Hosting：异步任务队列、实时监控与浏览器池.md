# Crawl4AI Self-Hosting：异步任务队列、实时监控与浏览器池

## 1. 调研对象

- 官方文档：https://docs.crawl4ai.com/core/self-hosting/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 重点源码：
  - `deploy/docker/work_queue.py`
  - `deploy/docker/job.py`
  - `deploy/docker/ARCHITECTURE.md`
  - `deploy/docker/MIGRATION.md`
- 关联文档：
  - https://docs.crawl4ai.com/advanced/multi-url-crawling/
  - https://docs.crawl4ai.com/blog/releases/v0.7.7/
  - https://docs.crawl4ai.com/blog/releases/0.7.6/

本次调研关注的不是“怎样用 Crawl4AI 抓一个 URL”，而是它在自托管模式下如何解决 **长任务异步化、浏览器资源复用、运行时限流、实时监控、安全边界和运维控制**，以及这些机制应该怎样纳入 1000+ 技术博客全历史抓取平台。

> 版本注意：当前 Self-Hosting 页面顶部已经说明 0.9.x 为 secure-by-default，但页面部分旧示例仍保留 0.8.x 写法。生产设计应以 `main` 分支的 `deploy/docker/MIGRATION.md`、实际发布版本源码和 Contract Test 为准，不能把文档中的旧示例直接当作安全契约。

---

## 2. 核心结论

Crawl4AI Self-Hosting 很适合充当本方案中的 **Browser Runtime Service / Slow Lane**，但不适合作为 1000 个站点的全局业务调度器和 Coverage Truth Store。

应该吸收的能力：

1. **异步 Job API**：长抓取任务提交后返回 `task_id`，状态与结果异步获取；
2. **有界 Work Queue**：固定 worker 数、队列容量、调用方并发配额，避免无限 BackgroundTask；
3. **浏览器池**：按配置签名复用 permanent/hot/cold browser，并以 janitor 做内存自适应清理；
4. **运行时并发保护**：MemoryAdaptiveDispatcher / SemaphoreDispatcher / RateLimiter；
5. **实时运维面**：health、request、browser pool、error、timeline、WebSocket、Prometheus；
6. **人工控制动作**：cleanup、kill browser、restart browser；
7. **安全加固**：默认认证、声明式 hooks、危险字段禁止从网络传入、artifact id 代替 caller-controlled path、Redis 隔离、队列/请求体/时间预算；
8. **Webhook**：任务完成事件可推送到上游，适合事件驱动流水线。

不能直接照搬的部分：

1. Docker server 的队列本质上是 **进程内 `asyncio.Queue`**，不是跨 Pod 的 durable queue；
2. Monitor 主要描述 **运行时近期状态**，不能代替文章、URL、Task、Coverage、Version 的长期业务事实；
3. Browser pool 的 hot/cold/permanent 是资源复用策略，不等于平台级公平调度、域名限速或 lease；
4. `/crawl/job` 的 `task_id` 是 runtime task identity，不能直接作为平台 Article/Attempt/Task 的唯一业务主键；
5. Webhook 可能重复、延迟、丢失或重放，必须由平台幂等消费，并保留 reconciliation；
6. 单个 Crawl4AI 容器的内存阈值只能保护本容器，不能解决集群全局浏览器容量；
7. 运行时返回 `success=true` 只代表执行成功，不代表抓到了正确文章、完整历史或高质量 Markdown。

因此最终架构应采用：

```text
平台控制面/业务真相
    |
    |  durable Task + lease + admission + idempotency
    v
Runtime Gateway
    |
    +---- HTTP Worker
    |
    +---- Crawl4AI Self-Hosted Runtime Pool
             |
             +-- bounded work queue
             +-- browser pool
             +-- memory adaptive dispatcher
             +-- runtime monitor / websocket / prometheus
             +-- webhook callback
    |
    v
Result Materializer
    |
    v
Raw Artifact -> Extract -> Quality -> Canonical IR -> Markdown
```

---

## 3. 异步 Job Queue 的实现原理

### 3.1 为什么要异步 Job

Self-Hosting 文档区分两类调用：

- 同步 `/crawl`：客户端一直等待抓取结束；
- 异步 `/crawl/job`、`/llm/job`：立即返回 `task_id`，后台执行，之后查询状态或由 webhook 通知。

这个模型解决了浏览器抓取天然存在的长尾：页面可能需要 JS、等待选择器、滚动、重试、下载资源，HTTP 请求生命周期不应该承担完整作业生命周期。

对于知识库平台，这个原则应推广为：

```text
API Request != Crawl Task != Crawl Attempt != Runtime Request
```

Web 管理端点击“回填”只创建一个 `crawl_run`；`crawl_run` 再生成大量持久化 task；task 被 worker lease 后，才发起一次 runtime request。任何一层都不能互相冒充。

### 3.2 `work_queue.py` 的关键变化

官方源码明确记录了一个重要演进：旧版 `/crawl/job` 和 `/llm/job` 使用 FastAPI `BackgroundTasks`，没有边界，调用方可以无限入队并耗尽内存和 browser slot；新实现引入：

```python
WorkQueue(maxsize, workers, per_principal)
```

核心结构是：

```text
asyncio.Queue(maxsize=N)
      |
      +--> worker 1
      +--> worker 2
      +--> ...
      +--> worker M
```

它提供三层保护：

- `maxsize`：等待队列上限，满时返回 QueueFull，对应 HTTP 503；
- `workers`：固定后台 worker 数，控制实际并行量；
- `per_principal`：按 principal 统计并发/在途 job 数，超限返回 429。

这背后的技术原理是 **Admission Control 必须发生在创建昂贵资源之前**。如果先接受无限任务，再依赖浏览器层排队，内存、Future、闭包、请求上下文就已经被放大。

### 3.3 对知识库平台的改造

不能把该 `asyncio.Queue` 直接作为平台主队列，因为容器退出会丢队列状态，也不能跨副本协调。

平台应使用：

- PostgreSQL：Task truth、状态机、lease、attempt、deadline；
- Redis Streams（或后续 Kafka/NATS JetStream）：任务运输；
- Runtime 内部 WorkQueue：**第二道局部保险丝**。

即形成双层背压：

```text
Global Admission
  - tenant/source/domain/browser quotas
  - durable task + lease
  - fair scheduling
        |
        v
Runtime Admission
  - container queue maxsize
  - workers
  - per-principal
  - memory threshold
```

当 Runtime 返回 429/503 时，上游不是立即无限重试，而应记录 `RUNTIME_BACKPRESSURE`，带抖动地重新排队，并降低该 runtime endpoint 的可用容量估计。

---

## 4. Webhook 的实现原则

Crawl4AI 的 Job API支持在完成时发送 webhook，支持自定义 header、成功/失败 payload，并有重试机制。文档也明确要求 webhook handler 幂等、快速返回并异步处理。

知识库平台应把 webhook 定义成 **提示事件（hint）**，而不是唯一完成凭证。

推荐事件模型：

```json
{
  "runtime_id": "c4ai-pool-a-03",
  "runtime_task_id": "crawl_xxx",
  "platform_attempt_id": "att_xxx",
  "event_type": "runtime.completed",
  "event_id": "...",
  "status": "completed",
  "emitted_at": "...",
  "result_ref": "..."
}
```

消费逻辑：

1. 先认证 callback；
2. 以 `(runtime_id, runtime_task_id, event_id)` 去重；
3. 查询平台 attempt 当前状态；
4. 若 attempt 已终态，则只做审计，不重复生成文档；
5. 拉取/接收 runtime result；
6. Result Materializer 逐条物化；
7. 原子更新 attempt + outbox；
8. 超时未收到 webhook 的 job 由 reconciliation 定时查询 runtime status。

必须有 reconciliation 的原因：Webhook 是网络消息，不具备 exactly-once 语义。可靠系统依赖 **幂等 + 至少一次 + 对账**，而不是假设“回调一定只来一次且永不丢”。

安全上还要额外限制：

- runtime 的 webhook destination 必须由服务端配置或 allowlist 选择，不能让普通 Web 用户填写任意 URL，否则会形成 SSRF；
- callback header 不允许透传 `Authorization`、`Cookie` 等敏感 hop-by-hop/header；
- webhook payload 大结果优先只传 artifact/result reference，避免放大请求体和敏感数据泄露。

---

## 5. Browser Pool 的技术原理

官方架构文档描述了 3 级浏览器池：

- **Permanent**：默认配置常驻；
- **Hot Pool**：高频配置；
- **Cold Pool**：低频或首次使用配置；
- **Janitor**：根据内存压力改变清理周期和 TTL。

### 5.1 为什么按配置签名复用

浏览器进程启动成本高，Chromium 的进程树、共享内存、JS runtime 都比普通 HTTP 请求昂贵。对相同 BrowserConfig 反复创建 Browser 会造成：

- 冷启动延迟；
- RSS/共享内存尖峰；
- 并发时进程爆炸；
- 稳定性下降。

Crawl4AI 把 BrowserConfig 序列化后计算签名，将相同配置映射到同一池对象。其本质是 **按资源兼容性 key 做复用**。

### 5.2 热冷分层

Hot/Cold 的意义不是业务优先级，而是缓存价值估计：

```text
高复用配置 -> 更长 TTL -> 减少重新启动成本
低复用配置 -> 更短 TTL -> 优先释放内存
```

这类似缓存的 admission + eviction，只是缓存对象变成了重量级 browser runtime。

### 5.3 Memory Pressure + Janitor

官方实现会读取容器 cgroup 内存，并在不同压力档位调整清理策略。它解决的是“浏览器不是请求结束就释放，怎样避免池无限膨胀”。

本方案应进一步做 **两级生命周期**：

1. **Context Generation**：按页面数、错误率、污染风险、session key 回收 context；
2. **Browser Process Generation**：按 PID age、累计 page 数、RSS、crash/renderer error、FD 数、总任务数，主动 drain 并替换整个 browser process/Pod。

原因是“回收 context”不能证明 Chromium 主进程、renderer、GPU/utility 子进程的长期泄漏已被清除。对 7×24 小时知识库平台，必须把进程代际作为独立治理对象。

### 5.4 配置签名不能包含跨安全域 Secret

如果 browser pool 只按 viewport、UA、headless 等配置签名复用，却忽略 tenant、source auth scope、cookie/profile，就可能把安全状态跨任务串用。

因此平台的 pool/isolation key 至少要纳入：

```text
runtime_profile_release
network_policy_class
credential_scope
session_isolation_class
proxy_policy_class
browser_feature_class
```

公开技术博客默认使用无认证、无持久 profile 的 public isolation class；需要登录的来源必须单独审批并独立池化。

---

## 6. Dispatcher、限流与资源保护

Crawl4AI 的 Multi-URL Crawling 提供：

- `MemoryAdaptiveDispatcher`：内存超过阈值时暂停新任务；
- `SemaphoreDispatcher`：固定并发；
- `RateLimiter`：对 429/503 等进行退避；
- `CrawlerMonitor`：记录运行状态；
- streaming mode：结果完成一个就消费一个，避免整个批次堆在内存。

这些是非常好的 **worker-local guardrail**，但在 1000 站点场景必须再加平台全局约束：

```text
global_browser_slots
per_runtime_browser_slots
per_source_concurrency
per_registrable_domain_concurrency
per_host_rps
per_source_daily_budget
per_run_page_budget
repair_budget
```

推荐采用“全局 token + 本地 dispatcher”组合：

```text
Redis/DB distributed admission token
          +
Crawl4AI local MemoryAdaptiveDispatcher
```

前者解决跨 Worker/Pod 的公平和硬上限，后者解决单机瞬时内存压力。

另外，批量抓取优先使用 streaming 结果逐条进入 Result Materializer。平台绝不能假设 runtime 返回 list 的位置与输入 URL 一一对应；应按 URL / runtime task id / correlation id 显式关联。

---

## 7. 实时监控系统的实现原理

Self-Hosting 的 monitor 维护：

- active requests；
- completed requests 的有限窗口；
- endpoint 聚合统计；
- browser pool；
- janitor events；
- errors；
- memory/request/browser timeline；
- WebSocket 实时推送；
- Prometheus `/metrics`；
- `/health`；
- 手工 cleanup/kill/restart 操作。

其价值在于把原来“只能翻日志”的浏览器运行时，变成可观察、可干预的服务。

### 7.1 两类监控必须分开

对知识库平台，应在 Web 管理端同时展示两套但不同语义的数据：

**业务监控（平台真相）**

- Source coverage；
- Backfill progress；
- frontier depth；
- queue age；
- discovered/fetched/pass/repair/dlq；
- freshness；
- article version；
- extractor quality；
- gap/stop reason。

**运行时监控（Crawl4AI telemetry）**

- runtime health；
- active request；
- browser pool count / reuse；
- RSS/memory pressure；
- request latency/error；
- janitor events；
- browser restart/kill。

运行时监控只能回答“爬虫服务现在健康吗”，不能回答“这个博客历史是否抓全”。

### 7.2 WebSocket 的定位

WebSocket 适合 Web 管理端的实时视图，但不能作为历史指标存储。建议：

```text
Crawl4AI /monitor/ws
        |
Runtime Telemetry Collector
        +--> Prometheus / TSDB
        +--> Web Admin live channel
```

Web UI断线后应从聚合 API/Prometheus 恢复状态，而不是依赖浏览器页面一直在线。

### 7.3 控制动作要经过平台审计

Crawl4AI 支持 cleanup、kill browser、restart browser。最终 Web Admin 不应把这些 endpoint 直接暴露给用户，而应：

```text
Admin Action
 -> RBAC
 -> reason / ticket
 -> audit log
 -> Runtime Control Gateway
 -> Crawl4AI admin endpoint
 -> action result + correlation id
```

同时提供“一键 Drain Runtime”：先停止新 admission，等待 active request 收敛，再重启 Pod。它比直接 kill 单个 browser 更适合生产滚动维护。

---

## 8. 0.9.x 安全加固带来的架构启示

官方 `MIGRATION.md` 的变化非常值得直接吸收。

### 8.1 网络 API 只能接受声明式低能力配置

0.9.x 禁止通过远程请求直接传入大量高风险字段，包括任意 JS、proxy、CDP URL、持久 user data dir、deep crawl strategy、magic/simulate_user 等；hooks 也从 Python 代码改为固定的声明式 action。

这验证了本方案的 **Capability Firewall + Config Compiler** 方向：

```text
Web 表单/REST JSON
 -> Source Profile DSL
 -> Policy Validation
 -> Capability Check
 -> Runtime Config Compiler
 -> pinned Runtime Config
```

普通用户只能配置“需要滚动”“阻止图片”“等待选择器”等经过审核的 intent，而不是直接注入代码。

### 8.2 caller 不拥有服务器文件路径

新版 screenshot/pdf 返回 `artifact_id`，而不是让调用者提供 `output_path`。这是典型的 **Caller Never Owns Paths** 原则，可避免路径穿越、覆盖任意文件和宿主路径泄露。

本方案所有 raw HTML、MHTML、screenshot、PDF、Markdown 都应由 Artifact Service 分配 object key，客户端只拿 opaque artifact id。

### 8.3 Runtime 默认必须是私网服务

官方新版默认 loopback + token，Redis 不再公开，CORS deny by default，monitor 控制动作需要 admin principal。

生产部署建议：

```text
Internet/Web Admin
   |
API Gateway
   |
Platform Control Plane
   |
private mTLS network
   |
Runtime Gateway
   |
Crawl4AI Pods
```

Crawl4AI runtime 不直接面向公网，不接受终端用户 token，不暴露 Redis/monitor control。

### 8.4 `0 = unbounded` 是危险语义

官方配置中多个限制以 `0` 表示无限。平台 Config Compiler 必须禁止“默认 0 被误当作关闭功能”，所有生产预算应显式非零，并在编译报告中展示：

```text
resolved_max_body_bytes
resolved_wall_clock_s
resolved_queue_maxsize
resolved_runtime_workers
resolved_per_principal
```

任何 unbounded 值都需要 admin override 和审计。

---

## 9. 对现有博客知识库方案的具体优化

基于本次调研，应在最终方案中增加/强化以下设计。

### 9.1 新增 Runtime Service Plane

原本 Fetch Plane 中直接使用 Crawl4AI 的描述还不够，应显式增加：

- Runtime Registry；
- Runtime Gateway；
- Runtime Admission；
- Runtime Result Materializer；
- Runtime Telemetry Collector；
- Runtime Control Gateway；
- Runtime Reconciliation。

这样平台不会把第三方 crawler SDK/server 的内部状态与业务状态混为一谈。

### 9.2 双层队列与双层限流

- 平台层：durable queue、fair scheduler、lease、domain token、browser token；
- Crawl4AI 层：bounded work queue、workers、per-principal、memory adaptive dispatcher。

目标是既防止全局争抢，也防止单个 runtime 因突发流量被击穿。

### 9.3 增加 Runtime Callback Inbox

Webhook 不能直接修改文档状态。新增 inbox：

```text
runtime_event_inbox
- event_key unique
- runtime_id
- runtime_task_id
- attempt_id
- event_type
- payload_hash
- received_at
- processed_at
- process_status
```

通过 inbox + outbox 实现幂等事件衔接。

### 9.4 增加 Runtime Reconciler

周期扫描：

- platform attempt = RUNNING 但 runtime 无 active request；
- webhook 超时；
- runtime task completed 但平台未 materialize；
- runtime 重启导致 job 丢失；
- attempt lease 过期。

根据 fencing token 决定重试或归档，避免 ghost running task。

### 9.5 Web 管理端增加 Runtime 运维页

至少展示：

- 每个 Runtime Pod/实例健康；
- active request、queue saturation；
- browser permanent/hot/cold 数；
- memory pressure；
- pool reuse；
- error rate/latency；
- runtime version/release；
- drain/restart/cleanup 操作；
- 操作审计。

同时与业务 Source/Coverage 页面分离，避免用户把 runtime success rate 当 coverage。

### 9.6 明确 Browser Process Generation

监控数据已经能暴露 browser age、hits、memory，因此平台可以建立主动进程代际策略：

- max process age；
- max pages；
- max RSS；
- crash/error threshold；
- drain timeout；
- replacement readiness；
- force-kill fallback。

“永久浏览器”只意味着池策略上的 permanent，不应解释为生产进程真的永不重启。

### 9.7 安全配置采用编译而不是透传

禁止 Web 用户直接填写：

- arbitrary JS/Python hook；
- proxy URL；
- CDP URL；
- local path；
- browser profile path；
- LLM base URL；
- deep crawl object；
- monitor admin action endpoint。

只允许受控 DSL，并在 server-side 编译到 pinned release 支持的字段。

---

## 10. 推荐运行流程

### 10.1 历史回填

```text
Source Approved
 -> Discovery Providers 构建 URL Inventory
 -> Persistent Frontier
 -> Fair Scheduler
 -> HTTP Fast Lane
 -> 若动态渲染/质量失败，则 Browser Slow Lane
 -> Global Browser Token
 -> Runtime Gateway
 -> Crawl4AI /crawl/job
 -> runtime task id 与 platform attempt 绑定
 -> webhook/poll reconciliation
 -> Result Materializer
 -> Raw Artifact
 -> Extraction Portfolio
 -> Quality Gate
 -> Canonical IR
 -> Markdown + Version
 -> Coverage Ledger
```

### 10.2 增量同步

优先不用 browser：

```text
CMS/RSS/Sitemap/Listing Delta
 -> URL/updated_at/etag observation
 -> conditional fetch
 -> content hash
 -> unchanged: touch freshness evidence
 -> changed/new: extraction/version pipeline
```

只有当 HTTP 无法得到正文、需要 JS、质量门禁失败或站点 Profile 明确要求时才进入 Crawl4AI browser runtime。

---

## 11. 生产验收指标

新增 Runtime 相关指标：

- `runtime_queue_saturation_ratio`
- `runtime_429_total`
- `runtime_503_total`
- `runtime_active_requests`
- `runtime_memory_pressure`
- `runtime_browser_pool_size`
- `runtime_browser_reuse_ratio`
- `runtime_janitor_events_total`
- `runtime_browser_restart_total`
- `runtime_callback_duplicate_total`
- `runtime_callback_lag_seconds`
- `runtime_reconciliation_recovered_total`
- `runtime_attempt_orphan_total`
- `browser_process_generation_age_seconds`
- `browser_process_generation_pages`
- `browser_process_drain_seconds`

建议告警关注“趋势 + SLO”，不要机械复制官方示例阈值。内存阈值、复用率、成功率需在实际 workload 压测后确定。

---

## 12. 最终判断

Crawl4AI Self-Hosting 的最大价值不是“多了一个 Docker 包装”，而是把浏览器抓取从单次 SDK 调用提升成了一个具有 **异步任务、资源池、局部背压、实时监控、运维控制和安全边界** 的运行时服务。

对于 1000+ 博客知识库，最合理的组合不是让它接管整个爬虫平台，而是：

- 平台掌握 Source、Coverage、Frontier、Task、Version、Artifact 和公平调度；
- Crawl4AI 负责受控的 Browser Runtime；
- 两者之间通过 Runtime Gateway、持久 Attempt、幂等 Callback Inbox、Result Materializer 和 Reconciler 解耦；
- Web 管理端同时呈现业务面和运行时面；
- 所有高能力配置均经过 Capability Firewall；
- 单机自适应保护与集群全局硬预算同时存在。

这套分层能够保留 Crawl4AI 自托管的工程收益，又避免把其进程内队列、近期监控和 browser cache 错当成长期知识库的业务真相，是当前方案最值得吸收的优化方向。
