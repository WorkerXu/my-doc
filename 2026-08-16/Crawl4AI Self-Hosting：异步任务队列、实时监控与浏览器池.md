# Crawl4AI Self-Hosting：异步任务队列、实时监控与浏览器池

## 1. 调研对象与版本边界

- 官方 Self-Hosting 文档：https://docs.crawl4ai.com/core/self-hosting/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 本次重点核对源码提交：`7e801521428ee12509994d39151006f64055ebe3`
- 重点源码：
  - `deploy/docker/work_queue.py`
  - `deploy/docker/job.py`
  - `deploy/docker/api.py`
  - `deploy/docker/server.py`
  - `deploy/docker/crawler_pool.py`
  - `deploy/docker/monitor.py`
  - `deploy/docker/monitor_routes.py`
  - `deploy/docker/governor.py`
  - `deploy/docker/schemas.py`
  - `deploy/docker/artifacts.py`
  - `deploy/docker/config.yml`
  - `deploy/docker/ARCHITECTURE.md`

本次调研只讨论 Crawl4AI Self-Hosting 对“1000+ 技术博客全历史回填、持续增量同步、Markdown 知识库、Web 运维”的可复用能力，不把 Crawl4AI 当成整个知识库平台。

官方 Self-Hosting 页面当前以 v0.9.x 为主，并明确强调 secure-by-default 的行为变化。文档、示例和源码存在跨版本漂移风险，因此生产系统必须使用固定 image digest/release，并用源码、运行时 Capability Probe 和 Contract Test 共同定义真实行为，不能只依据文档标题或接口名称推断语义。

---

## 2. 核心结论

Crawl4AI Self-Hosting 的最佳角色是 **Browser Runtime Service / JS Slow Lane**：负责浏览器执行、局部资源保护、运行时监控和短期结果交付；它不适合作为平台的 Durable Queue、Coverage Truth Store、Document Truth Store、最终 Markdown 真相或跨站公平调度器。

可以直接吸收的能力：

1. `/crawl/job`、`/llm/job` 等异步 Job 接口；
2. 有界 `WorkQueue` 和 429/503 Backpressure；
3. permanent/hot/cold Browser Pool；
4. `MemoryAdaptiveDispatcher`、`RateLimiter` 等实例内保护；
5. Monitor API、WebSocket、Prometheus；
6. secure-by-default 的认证、SSRF/egress、防高能力配置透传思路；
7. opaque artifact ID、TTL、quota、sandboxed artifact store；
8. Webhook 完成提示。

不能直接照搬的部分：

1. `WorkQueue` 是单进程 `asyncio.Queue`，不是持久队列；
2. Redis task hash 有 TTL，只是 Runtime 临时状态；
3. Runtime Job 只有粗粒度状态，不提供可作为平台真相的 queued/running 生命周期；
4. `/crawl/job` 会把完整结果 JSON 存进 Redis task hash，不适合当大批量长期结果总线；
5. Browser Pool 是进程内缓存，且 BrowserConfig 基数过高会造成冷启动、内存和锁竞争；
6. Monitor 中部分浏览器内存是估算值，短窗口数据主要在内存；
7. 部分原生 cleanup/kill/restart 操作可能中断活跃浏览器，不能直接暴露为平台安全运维动作；
8. Runtime artifact 有 TTL/quota，不能作为知识库永久引用。

推荐边界：

```text
Web Admin / API
      |
PostgreSQL Truth + Durable Task + Fair Scheduler
      |
Global Admission / Lease / Fencing
      |
Runtime Gateway + Config Compiler
      |
+----------------------+----------------------+
|                                             |
HTTP Fast Lane                         Crawl4AI Runtime Pool
                                              |
                                    bounded WorkQueue
                                    Browser Pool
                                    local Dispatcher
                                    Monitor / WebSocket
                                    Runtime Redis
                                              |
                              Result Materializer / Reconciler
                                              |
                                  Artifact Promotion Gate
                                              |
S3/MinIO Raw -> Extraction -> Quality -> Canonical IR -> Markdown
```

---

## 3. 异步 Job 的真实实现结构

### 3.1 API、WorkQueue、Redis 是三层不同对象

`/crawl/job` 的真实路径可以抽象为：

```text
POST /crawl/job
   |
   +--> 生成 runtime task_id
   +--> Redis HSET task:{id} status=PROCESSING + TTL
   +--> _enqueue_job(factory, principal)
             |
        WorkQueue.submit()
             |
        asyncio.Queue
             |
        固定数量 workers
             |
        handle_crawl_request()
             |
        Redis result/status
             |
        optional webhook
```

Redis 保存状态不等于 Redis 承担排队。真正“等待被执行”的 Job Factory 位于进程内 `asyncio.Queue`，Pod/进程退出后该等待队列不会自动恢复。

因此必须明确：

```text
平台 durable task != Runtime job
平台 attempt      != Runtime task_id
Runtime Redis     != 平台 Task Truth
Runtime completed != 平台 succeeded
```

### 3.2 Runtime 状态只有粗粒度事实

当前异步 crawl task 在 Redis 中主要经历：

```text
PROCESSING -> COMPLETED
           -> FAILED
```

`WorkQueue` 没有把每个 Job 的“queued/running”阶段作为持久、可查询的 task state 暴露出来。因此平台中的 `RUNTIME_QUEUED`、`RUNTIME_RUNNING` 只能是 **平台根据 submit、poll、monitor、webhook、reconciler 得出的观察态**，而不能伪装成 Runtime 提供的权威状态。

建议平台为每次 Attempt 记录：

```text
runtime_observed_status
runtime_state_source: SUBMIT/POLL/MONITOR/WEBHOOK/RECONCILER
runtime_state_observed_at
runtime_task_id
runtime_instance_id
```

并把未知状态显式表示为 `RUNTIME_UNKNOWN`，而不是根据“没有报错”推断正在运行。

### 3.3 Pod 重启后的恢复责任属于平台

如果 Runtime 在 queued/running 期间重启：

- 进程内 WorkQueue 内容可能消失；
- Redis task hash 可能仍在 TTL 内存在；
- Webhook 可能永远不来；
- Monitor 的短窗口数据会丢失或重置。

所以平台必须依靠 Task Lease、Fencing Token 和 Runtime Reconciler 识别 orphan attempt，再决定继续查询、重试或创建新 Attempt。

---

## 4. `WorkQueue`：有界背压的实现原理

`WorkQueue(maxsize, workers, per_principal)` 的核心数据结构是：

```text
asyncio.Queue(maxsize=N)
      |
      +--> worker 1
      +--> worker 2
      +--> ...
      +--> worker M
```

### 4.1 三个参数的真实语义

- `maxsize`：等待队列上限；满时抛 `QueueFull`，上层映射为 HTTP 503，并带 `Retry-After: 5`；
- `workers`：固定后台 consumer 数；
- `per_principal`：限制同一 principal 的 **queued + running 总在途数**。计数在 submit 成功时增加，在 worker `finally` 中释放；超限映射为 429。

这里的关键技术原则是：**背压必须发生在昂贵 Browser 工作真正创建之前**。否则即使 Chromium 并发有限，调用层的 coroutine、Result、上下文和排队对象仍可能无界增长。

### 4.2 `0 = unbounded` 不能成为平台默认语义

源码/配置中：

```text
queue.maxsize = 0       -> unbounded queue
queue.per_principal = 0 -> unlimited
wall_clock_s = 0        -> no deadline
```

这类兼容语义对公网/多租户平台风险很大。平台 Config Compiler 应拒绝普通 Source/Profile 产生 `0=unbounded`，并把 queue、deadline、page/byte budget 编译成显式安全上限。

### 4.3 WorkQueue 未启动时存在降级路径

`_enqueue_job()` 在 WorkQueue 未 started 时会回退到 FastAPI `BackgroundTasks`。这对测试环境方便，但生产如果 lifecycle 异常，就可能悄悄退回无界后台任务行为。

因此 Runtime readiness/Contract Probe 必须至少验证：

1. WorkQueue 已 started；
2. `maxsize/workers/per_principal` 与发布配置一致；
3. 压满队列时确实返回 503；
4. principal 超限时确实返回 429；
5. `Retry-After` 存在；
6. 任一不符则实例 `UNROUTABLE`。

---

## 5. 双层 Admission 与公平性

Runtime 队列只能保护单实例。1000+ Source 还需要平台层解决站点公平、域名礼貌、增量优先级和跨 Pod 资源总量。

推荐：

```text
Platform Global Admission
- global_http_slots
- global_browser_slots
- per_runtime_slots
- per_source_concurrency
- per_registrable_domain_concurrency
- per_host_rps/token bucket
- per_run page/byte/wall-clock budget
- incremental reserved capacity
- repair reserved capacity
- weighted fair queue / DRR

Crawl4AI Local Admission
- WorkQueue maxsize/workers/per_principal
- process-wide page semaphore
- MemoryAdaptiveDispatcher
- RateLimiter
- browser pool memory pressure
```

错误必须分类：

- Runtime 429 -> `RUNTIME_PRINCIPAL_QUOTA`；
- Runtime 503 -> `RUNTIME_QUEUE_FULL`；
- Runtime 504 -> `RUNTIME_WALL_CLOCK`；
- Runtime lost/reset -> `RUNTIME_LOST`；
- 目标站 429/503 -> `ORIGIN_RATE_LIMIT/ORIGIN_UNAVAILABLE`。

不能把 Runtime 自身过载和目标网站限流混成同一重试策略。

---

## 6. Runtime Result：不要把 Redis 当大型结果总线

### 6.1 `/crawl/job` 会把完整结果 JSON 写进 Redis

异步 crawl 完成后，当前实现把：

```text
json.dumps(result)
```

直接写到 `task:{runtime_task_id}` 的 Redis hash `result` 字段，并设置 task TTL。与此同时，请求 schema 对 URL 列表有上限，但单页 HTML/Markdown/PDF 元数据仍可能很大。

这意味着 Runtime Redis 适合短期 Job 结果交接，不适合承载大规模长期数据面。

### 6.2 平台应限制 micro-batch 的“字节”而非只限制 URL 数量

建议 Runtime Gateway 同时限制：

```text
max_urls_per_runtime_job
max_expected_result_bytes
max_input_body_bytes
max_runtime_wall_clock
max_artifact_bytes
```

高完整性 Browser Repair 默认可以 1 URL/Job；批量场景只做受控 micro-batch。平台为每个预期结果建立：

```text
runtime_task_item
- runtime_task_id
- attempt_id
- input_url_identity_id
- expected_correlation_key
- result_status
- result_sha256
- materialized_at
```

Runtime 一进入 terminal，平台立即拉取、逐条物化并释放 Redis 结果依赖，不等 TTL 临近再处理。

### 6.3 结果关联必须按 Identity，而不是列表位置

`arun_many`、streaming、retry、部分失败都可能让返回顺序和输入位置不再是可靠业务契约。结果应以 normalized URL、platform attempt correlation、runtime task item 做关联。

必须检测：

- duplicate result；
- missing result；
- extra/unknown result；
- cardinality mismatch；
- schema mismatch。

全部 fail loud，不能静默按下标拼接。

---

## 7. Webhook 与 Reconciliation

Webhook 只能是低延迟 hint，不是 exactly-once commit protocol。

平台建立：

```text
runtime_event_inbox
- runtime_id
- runtime_task_id
- platform_attempt_id
- event_id/payload_hash
- received_at
- processed_at
- status
```

处理流程：

1. 校验固定 callback secret/来源；
2. Inbox 幂等去重；
3. 绑定平台 Attempt；
4. 主动查询/拉取 Runtime Result；
5. 逐条 materialize；
6. 晋升 required artifact；
7. Raw Artifact 持久化；
8. Outbox 触发 extraction。

周期性 Reconciler 必须处理：

- 长期无 webhook；
- Runtime completed 但平台未 materialize；
- Pod 重启后的 orphan；
- lease 即将过期；
- late/duplicate webhook。

正确关系：

```text
webhook = fast path
reconciliation = correctness path
```

普通 Web 用户不能指定任意 webhook destination；callback URL 和 header 由 Runtime Gateway 服务器端决定，避免 SSRF 和凭据泄露。

---

## 8. Runtime Artifact Store 与 Promotion Gate

Artifact Store 的安全设计值得吸收：

- server 生成 opaque 32-hex ID；
- `O_EXCL | O_NOFOLLOW`；
- 文件 0600、目录 0700；
- regular-file/symlink 校验；
- 单文件大小上限；
- 总 quota；
- TTL；
- janitor 过期/超配额回收。

当前源码默认值：

```text
MAX_ARTIFACT_BYTES   = 50 MiB
ARTIFACT_QUOTA_BYTES = 2 GiB
ARTIFACT_TTL_SECONDS = 3600
```

因此 Runtime Artifact 天然是短期制品。

平台必须有：

```text
Runtime artifact ref
 -> FETCHING
 -> stream with byte limit
 -> MIME/size verify
 -> SHA-256
 -> immutable S3/MinIO object
 -> DURABLE
 -> platform artifact id
```

建议模型：

```text
runtime_artifact_ref
- runtime_id
- runtime_task_id
- attempt_id
- runtime_artifact_id
- required
- declared_mime
- declared_size
- sha256
- durable_artifact_id
- status: PENDING/FETCHING/DURABLE/MISSING_EXPIRED/TOO_LARGE/CORRUPT
- error_class
```

只有 required artifacts 全部 DURABLE，Attempt 才能进入 `MATERIALIZED/SUCCEEDED`。Screenshot 等诊断材料可按策略 optional；作为重放输入的 Raw HTML/PDF 必须持久化。

核心告警：

```text
runtime_artifact_promotion_age_seconds
runtime_artifact_ttl_headroom_seconds
runtime_artifact_missing_expired_total
runtime_artifact_promotion_failure_total
```

Promotion backlog 接近 TTL 时应优先排空，不再向该 Runtime 提交高 artifact workload。

---

## 9. Browser Pool：性能缓存，不是业务状态

### 9.1 permanent / hot / cold 的真实实现

Pool 按序列化 `BrowserConfig` 的 SHA1 signature 复用：

- `PERMANENT`：默认配置常驻；
- `COLD_POOL`：新/低频配置；
- `HOT_POOL`：使用次数达到阈值后提升；
- janitor 按容器内存压力调整清理周期和 TTL；
- 正常 janitor 会跳过 `active_requests > 0` 的浏览器。

复用能减少 Chromium 冷启动，但它仍只是进程内资源缓存。

### 9.2 BrowserConfig 基数必须受控

`crawler_pool.py` 用整个 BrowserConfig 的序列化结果生成 signature。若 1000 个站点各自产生独特 BrowserConfig，会导致：

- signature 爆炸；
- 大量 cold browser；
- Chromium 冷启动和内存抖动；
- pool reuse 降低；
- janitor churn 增加。

更重要的是，Pool 使用一个全局 `asyncio.Lock`，且新 browser 的 `crawler.start()` 和清理时的 `crawler.close()` 都可能在持锁期间执行。高 BrowserConfig churn 会把慢启动/慢关闭放大为获取路径的 Head-of-Line Blocking。

因此平台 Config Compiler 应把 1000+ Source 映射到少量稳定的 `browser_profile_class`，例如：

```text
public-default
public-js-heavy
public-media-light
isolated-authenticated
```

站点特有 selector、wait 条件、正文规则尽量放在低成本 CrawlerRunConfig/Extraction Recipe，而不是制造新的 BrowserConfig。

建议指标：

```text
unique_browser_signature_count
cold_browser_create_rate
browser_pool_churn_rate
browser_pool_reuse_rate
browser_acquire_latency
browser_pool_lock_wait_seconds
```

### 9.3 安全隔离比复用率优先

复用边界至少包含：

```text
runtime_profile_release
network_policy_class
credential_scope
session_isolation_class
proxy_policy_class
browser_feature_class
```

公开技术博客默认 `public-anonymous`，不使用持久个人 cookie/profile。需要认证的 Source 单独审批并隔离 credential scope。

### 9.4 Context Generation 与 Browser Process Generation 分离

长期运行要分别管理：

- Context：页面数、错误、session 污染、站点状态；
- Browser Process：PID age、累计页面、RSS、FD、renderer crash、错误率。

推荐：

```text
ACTIVE -> DRAINING -> active attempts=0 -> recycle/replace
       -> readiness + contract probe -> ACTIVE
```

---

## 10. Monitor / WebSocket / Prometheus 的可信边界

Runtime Monitor 可用于：

- active/completed request；
- CPU/Memory/Network；
- Browser Pool；
- janitor/errors；
- timeline；
- WebSocket 实时页面；
- Prometheus endpoint。

但这些指标是 Runtime 运维信号，不是 Coverage Truth。

### 10.1 Monitor 数据有精度和耐久性边界

源码中：

- `completed_requests`、`janitor_events`、`errors` 是 maxlen=100 的内存 deque；
- timeline 是约 5 分钟内存窗口；
- endpoint aggregate 才写 Redis，而且有 TTL；
- Browser Pool `memory_mb` 当前是固定估算值，例如 permanent/hot/cold 使用预估常数，并非实际每个 Chromium 进程 RSS。

因此不能直接拿 Dashboard 的 `memory_mb` 做自动扩缩容或容量会计。

更可靠的 Runtime Capacity 输入应包括：

```text
cgroup memory.current / memory.max
memory PSI
OOM/restart count
queue age/saturation
actual process RSS where available
browser slot utilization
browser acquire latency
cold create rate
artifact promotion headroom
```

### 10.2 Runtime Telemetry 与 Business Telemetry 分离

```text
Runtime Telemetry
- queue saturation
- runtime 429/503/504
- cgroup memory pressure
- browser reuse/churn
- acquire latency
- restart/drain
- artifact backlog/headroom

Business Telemetry
- discovered URL count
- fetched article count
- PASS Markdown count
- known gap
- backfill stop reason
- incremental freshness
- extraction quality
```

Runtime 全绿但 URL Inventory 漏掉一半，业务仍然失败。

---

## 11. 原生 Monitor Control Action 不能直接作为平台安全运维原语

源码核对显示，原生 monitor action 更适合作为 Runtime 管理工具，而不是平台保证“零中断”的控制面：

1. `force_cleanup` 会直接关闭 cold pool 中的浏览器，并没有像正常 janitor 一样逐个检查目标 `active_requests`；
2. `kill_browser` 发现有 active request 时主要是 warning，仍可继续关闭目标浏览器；
3. hot/cold `restart_browser` 无法可靠重建原 BrowserConfig，实际语义更接近“删除后等待下次请求惰性重建”；
4. 这些操作和平台 Task Lease/Attempt 并不知道彼此状态。

所以知识库平台不应把底层 cleanup/kill/restart endpoint 直接暴露给普通操作员。

推荐生产运维语义：

```text
Drain Runtime
 -> Registry 停止新 Admission
 -> 等待平台 active attempts 收敛
 -> 超时则明确标记可恢复异常态
 -> replace Runtime Pod / Browser Process Generation
 -> readiness
 -> capability/contract probe
 -> ACTIVE
```

底层 kill/cleanup 仅作为受审 emergency action；执行前平台应确认目标 Runtime/Browser 没有绑定 active attempt，并留下审计记录。

必须新增测试：

- active attempt 存在时平台拒绝普通 recycle；
- emergency kill 后 orphan attempt 可被 Reconciler 恢复；
- replace 后 Runtime generation 递增；
- probe 未通过前禁止重新路由。

---

## 12. secure-by-default 对平台的意义

值得直接继承的安全原则：

- 无凭据且非 loopback 时拒绝启动；
- AuthGate 覆盖 HTTP/WebSocket；
- CORS deny-by-default；
- request body 有硬限制；
- `Provenance.UNTRUSTED` 对网络配置做能力约束；
- egress/SSRF 防护；
- hooks 走声明式 action 而非任意 Python；
- screenshot/PDF caller 不拥有 filesystem path；
- LLM key/base URL 服务端持有；
- monitor destructive actions 需要 admin；
- webhook header 做敏感字段校验。

平台仍需再包一层 Capability Firewall：

```text
Source Profile Intent
 -> Policy Validation
 -> Config Compiler
 -> Approved Runtime Config
```

普通 Web/API 用户不得直接透传：

- JS/Python code；
- proxy/CDP/user_data_dir；
- cookies/secrets/arbitrary headers；
- arbitrary webhook destination；
- arbitrary filesystem path；
- unbounded queue/deadline/budget；
- 高能力 deep-crawl strategy。

Runtime 只在私网，通过 Gateway 访问；Runtime admin token 永不下发前端。

---

## 13. Runtime Registry 与行为契约

建议：

```text
runtime_instance
- runtime_id
- endpoint
- software_release
- image_digest
- result_schema_release
- config_schema_release
- capabilities_hash
- queue_config_hash
- security_profile_release
- browser_generation
- state: ACTIVE/DRAINING/QUARANTINED/UNROUTABLE
- capacity_score
- last_probe_at
```

加入流量前验证：

- auth/security posture；
- WorkQueue started 且有界；
- 429/503/Retry-After；
- Runtime task/result schema；
- artifact API/TTL；
- forbidden power fields；
- internal IP/redirect 拒绝；
- monitor admin permission；
- Browser Pool contract；
- async task status semantics；
- batch/result cardinality contract。

行为不匹配就 `UNROUTABLE`，不做静默兼容。

---

## 14. 推荐平台 Attempt 状态机

```text
LEASED
 -> ADMITTED
 -> RUNTIME_SUBMITTING
 -> RUNTIME_ACCEPTED
 -> RUNTIME_OBSERVED_PROCESSING
 -> RUNTIME_TERMINAL
 -> MATERIALIZING_RESULT
 -> PROMOTING_ARTIFACTS
 -> MATERIALIZED
 -> RAW_PERSISTED
 -> SUCCEEDED
```

可选 UI 可以展示 `RUNTIME_QUEUED/RUNTIME_RUNNING`，但必须标明它们是平台推断态，底层 Runtime 当前主要只给粗粒度 PROCESSING。

异常态：

```text
RUNTIME_BACKPRESSURE
RUNTIME_UNKNOWN
RUNTIME_LOST
RESULT_TOO_LARGE
RESULT_SCHEMA_MISMATCH
RESULT_CARDINALITY_MISMATCH
ARTIFACT_EXPIRED
ARTIFACT_QUOTA
CANCEL_REQUESTED
DRAINING
FAILED_RETRYABLE
FAILED_FINAL
```

`RUNTIME_TERMINAL` 永远不等于 `SUCCEEDED`。只有结果逐条物化、required artifact durable、Raw Artifact 入平台对象存储后，后续 Extraction 才能启动。

---

## 15. 典型故障与推荐处理

| 故障 | 错误做法 | 推荐处理 |
|---|---|---|
| Runtime 503 | 紧循环重试 | 遵守 Retry-After，退回 durable queue，降低 capacity score |
| Runtime 429 | 继续加并发 | 降 principal 提交速率，检查 quota |
| Pod 重启 | 假设 job 自动恢复 | Lease + Reconciler，必要时新 Attempt |
| Runtime 一直 PROCESSING | 永久等待 | deadline + reconciliation + runtime generation 检查 |
| Redis result 过大 | 继续扩大 batch | 按结果字节预算缩 micro-batch，快速 materialize |
| webhook 丢失 | 永久等待 | 主动 poll/reconcile |
| webhook 重复 | 重复生成 Document | Inbox 幂等 |
| artifact 404 | 当可忽略 | required/optional 分类；required repair |
| BrowserConfig 数量暴涨 | 每站独立 browser | 收敛 browser_profile_class，控制 signature 基数 |
| pool acquire 变慢 | 盲目加请求 | 检查 cold create/lock HOL/churn，减少 profile 基数 |
| Dashboard memory 高/低 | 直接按估算扩缩容 | 使用 cgroup/PSI/RSS/queue age 等真实信号 |
| browser RSS 高 | 直接 kill | Drain -> active=0 -> replace -> probe |
| result 数量不等于输入 | 按下标对齐 | runtime_task_item/URL identity 关联，fail loud |
| monitor 绿灯 | 推断抓全 | Coverage 单独从 Inventory/Evidence 计算 |

---

## 16. 必须增加的 Contract / Stress / Chaos Test

### 16.1 Queue / State

- queue 满 -> 503 + Retry-After；
- per_principal 超限 -> 429；
- 普通 Profile 不得编译出 unbounded 值；
- WorkQueue 未 started -> readiness fail；
- Runtime task 粗状态映射正确；
- Pod restart 后 orphan attempt 可收敛。

### 16.2 Result

- 单 Job 结果超过平台 byte budget -> 拒绝/缩 batch；
- duplicate/missing/extra result fail loud；
- streaming 乱序仍按 identity 关联；
- Redis task TTL 前完成 materialization；
- schema 漂移 -> Runtime quarantine。

### 16.3 Browser Pool

- 大量 BrowserConfig signature 时监控 cold create/churn；
- BrowserConfig Compiler 能把站点收敛到有限 profile class；
- pool acquire latency 有 SLO；
- active browser 不被平台普通 recycle 打断；
- Runtime replace 后 generation + contract probe 正常。

### 16.4 Artifact

- opaque id；
- caller path 被拒绝；
- required artifact TTL 内晋升；
- expired/quota/corrupt 均有明确状态；
- MIME/size/hash 不匹配不建立 durable reference。

### 16.5 Security

- 未认证数据 API 不可用；
- data principal 不能调用 admin action；
- power fields 被拒绝；
- internal IP/metadata/redirect 被阻断；
- arbitrary webhook destination 不可由普通用户决定。

### 16.6 Chaos

- Runtime queued/running 时重启；
- webhook 丢失/重复/乱序；
- Redis 暂时不可用；
- S3 写成功 DB 失败；
- DB 成功 Outbox 未发；
- artifact backlog 接近 TTL；
- Browser OOM；
- emergency kill active Runtime。

系统必须最终收敛且不静默丢失业务事实。

---

## 17. 对博客知识库最终方案的具体优化

本次调研应在总方案中固化：

1. **Runtime Queue 与 Durable Queue 分层**：PostgreSQL + Redis Streams 才是平台 Task Truth；
2. **Runtime State 是观察态**：不把 coarse PROCESSING 伪装成精确 queued/running 真相；
3. **双层 Admission**：平台负责公平/总预算，Runtime 负责实例保护；
4. **Runtime Registry + Contract Probe**：固定 release/image digest，以行为契约入流；
5. **Webhook Inbox + Reconciler**：webhook 加速，对账保证正确性；
6. **Runtime Result 立即物化**：Redis task result 是短期交接，不是大型长期数据总线；
7. **按结果字节预算 micro-batch**：不仅按 URL 数量限批；
8. **`runtime_task_item` 逐条关联**：禁止按返回列表位置认领结果；
9. **BrowserConfig 基数控制**：把 1000+ Source 编译到少量 browser profile class；
10. **监控 Browser Pool 锁竞争与 churn**：防止 config signature 爆炸形成 HOL；
11. **Context / Process 两级代际**：长期 Browser 需要主动 Drain/Replace；
12. **Runtime Artifact Promotion Gate**：required artifact durable 后才成功；
13. **Runtime Telemetry 与 Business Telemetry 分离**，Monitor 估算值不能做业务真相；
14. **平台运维优先 Drain/Replace**：底层 kill/restart/cleanup 仅为 emergency action；
15. **Capability Firewall**：平台用户只能表达低能力 intent；
16. **所有 `0=unbounded` 语义显式阻断**。

---

## 18. 最终判断

Crawl4AI Self-Hosting 能显著降低浏览器运行时、局部资源治理、实时监控和安全 hardening 的自研成本，但它应被当作 **可替换、受约束、可观测、可隔离的执行 Runtime**。

平台长期正确性必须掌握在自己的持久化层：

```text
Coverage Truth       -> PostgreSQL
Durable Task         -> PostgreSQL + Redis Streams
URL Frontier         -> PostgreSQL
Raw/IR/Markdown      -> S3/MinIO
Runtime Queue        -> Crawl4AI asyncio.Queue（局部临时）
Runtime Status       -> Crawl4AI Redis（粗粒度、TTL 临时）
Runtime Result       -> Redis 短期交接，立即逐条物化
Browser Pool         -> Crawl4AI（性能缓存，控制 profile 基数）
Runtime Artifact     -> Crawl4AI（短 TTL，必须晋升）
Final Markdown       -> Canonical IR deterministic projection
Incremental Truth    -> URL Observation + Document Version
```

在这个边界下，1000+ 站点的历史完整性、增量正确性、可恢复性、内容耐久性和 Web 运维不会被绑定到单个 Runtime 的进程内队列、浏览器池或短期 Redis 状态。