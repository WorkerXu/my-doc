# Crawl4AI Self-Hosting：异步任务队列、实时监控与浏览器池

## 1. 调研对象与版本边界

- 官方 Self-Hosting 文档：https://docs.crawl4ai.com/core/self-hosting/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 重点源码：
  - `deploy/docker/work_queue.py`
  - `deploy/docker/job.py`
  - `deploy/docker/api.py`
  - `deploy/docker/artifacts.py`
  - `deploy/docker/MIGRATION.md`
  - `deploy/docker/ARCHITECTURE.md`
  - `deploy/docker/SECURITY-VERIFY.md`
- 关联文档：
  - https://docs.crawl4ai.com/advanced/multi-url-crawling/
  - https://docs.crawl4ai.com/blog/releases/v0.7.7/
  - https://docs.crawl4ai.com/blog/releases/0.7.6/

本次调研只关心 Crawl4AI Self-Hosting 对“1000+ 技术博客全历史回填 + 持续增量同步 + Web 运维”的可复用能力，不把它当成整个知识库平台。

当前官方 Self-Hosting 页面已经以 **v0.9.x** 为标题，并明确提示 0.9.0 是 secure-by-default 的破坏性升级；但页面中仍混有部分旧版示例。因此生产实现不能只照抄文档片段，必须以固定 release/tag 的源码、`MIGRATION.md`、运行时 capability probe 和 Contract Test 共同定义行为契约。

---

## 2. 核心结论

Crawl4AI Self-Hosting 很适合成为知识库平台中的 **Browser Runtime Service / JS Slow Lane**，不适合成为全局调度器、永久任务库、Coverage Truth Store 或最终 Markdown 生成真相。

应该吸收的能力：

1. 异步 `/crawl/job` / `/llm/job` API，把长尾浏览器任务从调用方 HTTP 生命周期中拆开；
2. 有界 `WorkQueue`，限制排队量、worker 数和单 principal 在途数；
3. Browser Pool，复用高成本 Chromium runtime；
4. `MemoryAdaptiveDispatcher`、`SemaphoreDispatcher`、`RateLimiter` 等 worker-local 保护；
5. health、request、browser pool、timeline、WebSocket、Prometheus 等实时运维信号；
6. cleanup、kill browser、restart browser 等运维动作；
7. secure-by-default 的网络/API 边界；
8. Webhook 完成提示；
9. screenshot/PDF 使用 opaque `artifact_id` 而不是 caller-controlled path。

不能直接照搬的部分：

1. `WorkQueue` 是 **单进程内 `asyncio.Queue`**，容器退出或 worker 被取消后，等待中的任务并不是 durable queue；
2. Redis 中的 runtime task status/result 仍只是运行时临时状态，具有 TTL，不能作为平台永久 Attempt/Document 事实；
3. Runtime `task_id` 不能冒充平台 `run_id/task_id/attempt_id/document_id`；
4. webhook 不是 exactly-once 完成协议，只能是 hint；
5. runtime monitor 描述“这个容器现在怎么样”，不能证明“某站历史文章是否抓全”；
6. browser pool 是资源缓存，不是跨 Source 公平调度器；
7. `success=true` 只说明一次运行时调用完成，不代表页面是正文、不是 soft-404、内容完整或历史覆盖完整；
8. Runtime artifact 是 **短期制品**，官方实现有 TTL 和 quota，必须及时晋升到平台对象存储，不能让业务事实引用会过期的 runtime-local id。

因此推荐边界是：

```text
Web Admin / API
      |
PostgreSQL Truth + Durable Task + Fair Scheduler
      |
Global Admission / Lease / Fencing
      |
Runtime Gateway
      |
+-----+------------------------------+
|                                    |
HTTP Fast Lane                Crawl4AI Runtime Pool
                                     |
                           bounded WorkQueue
                           Browser Pool
                           local Dispatcher
                           Monitor / WebSocket / Metrics
                           Webhook / Runtime Redis
                                     |
                         Result Materializer
                                     |
                  Runtime Artifact Promotion Gate
                                     |
Object Storage Raw Artifact -> Extract -> Quality -> IR -> Markdown
```

---

## 3. 异步 Job 的真实实现结构

### 3.1 API、队列和 Redis 是三个不同层次

`job.py` 的 `/crawl/job` 最终调用 `api.py` 中的 handler。当前代码同时存在：

- **HTTP Job API**：接收请求并返回 runtime `task_id`；
- **进程内 WorkQueue**：决定后台 coroutine 什么时候执行；
- **Redis task hash**：保存 runtime task 的 status/result，并设置 TTL。

这三个对象不能混为一个“队列”。更准确的模型是：

```text
POST /crawl/job
      |
      +--> 创建 runtime task id / Redis 状态
      |
      +--> _enqueue_job(factory, principal)
                |
                +--> WorkQueue.submit()
                       |
                  asyncio.Queue
                       |
                    workers
                       |
                 crawl coroutine
                       |
                 Redis result/status
                       |
                    webhook
```

因此“用了 Redis”并不意味着排队本身是 durable 的。Redis 可以保存任务状态，但当前 `work_queue.py` 明确使用进程内 `asyncio.Queue` 承载等待执行的 coroutine factory。

### 3.2 为什么这对平台很重要

如果 Crawl4AI Pod 在排队期间重启：

- 平台不能假设 runtime 队列会恢复；
- runtime Redis 中可能仍留有一段时间的 task 状态；
- webhook 也可能永远不会到达；
- 平台自己的 lease/attempt 必须通过 reconciliation 发现“runtime 丢失/未知”，再决定重试。

所以：

```text
平台 durable task != runtime job
平台 attempt != runtime task id
平台完成态 != runtime Redis completed
```

---

## 4. `WorkQueue`：有界背压的技术原理

`deploy/docker/work_queue.py` 解决了旧实现使用 FastAPI `BackgroundTasks` 无上限接受后台任务的问题。核心接口：

```python
WorkQueue(maxsize, workers, per_principal)
```

其内部是：

```text
asyncio.Queue(maxsize=N)
      |
      +--> worker 1
      +--> worker 2
      +--> ...
      +--> worker M
```

### 4.1 三个限制分别解决什么

- `maxsize`：限制等待队列长度；满时抛 `QueueFull`，上层映射为 HTTP 503，并带 `Retry-After`；
- `workers`：固定真正执行 factory 的后台 worker 数；
- `per_principal`：限制同一调用主体的 queued + running 在途数，超限映射为 HTTP 429。

这里最重要的原理是 **Admission 必须发生在昂贵资源创建之前**。如果请求全部先转成 background task，再让 Chromium 自己“慢慢排”，内存中的 coroutine、Future、request context 和待处理结果早已无界增长。

### 4.2 `0 = unbounded` 是生产风险点

源码明确把：

```text
queue.maxsize = 0       -> unbounded
queue.per_principal = 0 -> unlimited
wall_clock_s = 0        -> no deadline
```

作为兼容旧行为的语义。

对知识库平台不能直接暴露这个语义。外部 Source Profile 中的 `0` 应被 Config Compiler 拒绝或编译成平台安全默认值；只有平台管理员的受审 Release 才能显式申请 unbounded，而且生产环境原则上仍不应使用。

这也是为什么方案需要 **Capability Firewall + Config Compiler**，而不是把 Crawl4AI 原生 YAML/JSON 直接透传给 Web 用户。

### 4.3 `_enqueue_job` 的降级行为也要测试

`api.py` 中 `_enqueue_job` 在 WorkQueue 没有启动时会回退到 FastAPI `BackgroundTasks`。这对测试/无 lifespan 环境有用，但生产启动如果配置或生命周期异常，就可能悄悄退回无边界模式。

因此平台上线必须有启动 Contract Test：

1. runtime readiness 返回 pinned release；
2. WorkQueue 已 started；
3. `maxsize/workers/per_principal` 与发布配置一致；
4. 压满队列时实际返回预期 503；
5. 超 principal 配额时实际返回 429；
6. 任何一项不满足，Runtime Registry 标记 `UNROUTABLE`。

---

## 5. 平台级双层背压

Runtime 自带的队列只能保护一个进程。1000+ Source 还需要平台层解决跨 Pod、公平性和域名级礼貌抓取。

推荐双层：

```text
第一层：Platform Global Admission
- per-source concurrency
- per registrable-domain concurrency/RPS
- per-host token bucket
- global browser slots
- per-runtime slots
- per-run page/request/byte/wall-clock budget
- incremental reserved capacity
- repair reserved capacity
- weighted fair scheduling

第二层：Crawl4AI Local Admission
- WorkQueue maxsize
- workers
- per_principal
- MemoryAdaptiveDispatcher
- RateLimiter
- browser pool memory pressure
```

### 5.1 429、503、504 不能统一当普通失败

建议分类：

- `429 RUNTIME_PRINCIPAL_QUOTA`：该 gateway principal 在途过多；降低该 principal 的提交速率；
- `503 RUNTIME_QUEUE_FULL`：实例排队已饱和；遵守 `Retry-After`，把 task 退回 durable queue，降低 runtime capacity score；
- `504 RUNTIME_WALL_CLOCK`：一次运行超预算；是否重试取决于页面类型和 repair policy；
- connection reset / pod lost：标记 `RUNTIME_LOST`，由 reconciliation + lease 恢复；
- 业务 HTTP 429/503：这是目标网站的响应，必须归入 host rate adaptation，不能和 Runtime 过载混淆。

重试统一采用有限次数、指数退避 + jitter，并保留 error class；不能在 Runtime 和平台两层同时无限重试。

---

## 6. Runtime task、Webhook 与 Reconciliation

### 6.1 Webhook 只能是 hint

官方文档支持 webhook 自定义 header、完成通知以及最佳实践中的幂等处理。平台端应建立 `runtime_event_inbox`：

```text
runtime_event_inbox
- runtime_id
- runtime_task_id
- platform_attempt_id
- event_id / payload_hash
- status
- received_at
- processed_at
- payload_artifact_id
```

处理顺序：

1. 校验来源和 secret；
2. 根据 `(runtime_id, runtime_task_id, event_id/payload_hash)` 幂等去重；
3. 绑定平台 attempt；
4. 拉取/物化所有结果；
5. 晋升 runtime artifact；
6. 只有材料完整后才推进平台 attempt；
7. 写 outbox 触发 Extract/Quality。

### 6.2 一定要有主动对账

周期性 Runtime Reconciler 查询：

- 已提交但长期无 webhook 的 job；
- runtime completed 但平台 materialization 未完成的 attempt；
- runtime endpoint 重启前后的 orphan attempt；
- lease 即将过期但 runtime 仍 active 的 attempt。

Webhook + polling 的正确关系是：

```text
webhook = 快速路径
reconciliation = 正确性兜底
```

而不是二选一。

### 6.3 webhook destination 必须服务端决定

普通 Web 用户不能填写任意 callback URL，否则会制造 SSRF 出口。Runtime Gateway 使用固定 callback endpoint 或服务端 allowlist；自定义 header 也只能由服务器生成，不允许透传 `Authorization`、`Cookie`、`Host` 等敏感字段。

---

## 7. Runtime Artifact Store：本次新增的关键落地点

0.9 hardening 把 screenshot/PDF 从 caller-controlled `output_path` 改为 opaque `artifact_id`。`deploy/docker/artifacts.py` 的实现值得直接吸收其安全思想：

- server 自己生成 32 hex id；
- `O_EXCL | O_NOFOLLOW` 创建文件；
- 文件权限 0600，目录 0700；
- 校验 regular file / 非 symlink；
- 单文件大小限制；
- 总 quota；
- TTL；
- janitor 清理过期文件并在超 quota 时按最老优先删除。

当前源码默认值中：

```text
CRAWL4AI_MAX_ARTIFACT_BYTES   = 50 MiB
CRAWL4AI_ARTIFACT_QUOTA_BYTES = 2 GiB
CRAWL4AI_ARTIFACT_TTL_SECONDS = 3600
```

这意味着 Runtime Artifact **天然不是知识库永久存储**。

### 7.1 必须增加 Artifact Promotion Gate

平台收到 runtime result 后，不允许把 `artifact_id` 直接写进最终 Document Version 并宣称完成。正确流程：

```text
Runtime result
   |
   +--> runtime artifact refs
            |
       FETCH_PENDING
            |
       GET /artifacts/{id}
            |
       stream + byte limit
            |
       sha256 + mime verify
            |
       S3/MinIO immutable object
            |
       DURABLE
            |
       attach platform artifact_id
            |
       attempt MATERIALIZED
```

建议新增：

```text
runtime_artifact_ref
- runtime_id
- runtime_task_id
- platform_attempt_id
- runtime_artifact_id
- declared_mime
- declared_size
- discovered_at
- fetch_started_at
- durable_artifact_id
- sha256
- status: PENDING/FETCHING/DURABLE/MISSING_EXPIRED/TOO_LARGE/CORRUPT
- error_class
```

### 7.2 为什么必须在终态之前晋升

Runtime artifact 会因为：

- TTL 到期；
- quota janitor；
- Pod 重建；
- 人工 cleanup；
- runtime 本地卷异常；

而消失。

因此平台 Attempt 的最终 `SUCCEEDED` 应建立在“所有 required artifact 已经 durable”之上。若 screenshot/MHTML 只是 diagnostic，可按策略标记 optional；但 Raw HTML/PDF 等作为重放输入的材料必须先完成平台持久化。

建议告警：

```text
runtime_artifact_promotion_age_seconds
runtime_artifact_ttl_headroom_seconds
runtime_artifact_missing_expired_total
runtime_artifact_promotion_failure_total
```

当 headroom 低于安全阈值时，应停止向该 runtime 提交新的高制品任务，优先清空晋升 backlog。

---

## 8. Browser Pool：性能缓存而不是业务状态

Self-Hosting 架构把浏览器大体分为 permanent / hot / cold，并由 janitor 根据内存压力清理。

### 8.1 配置签名复用

浏览器启动成本高，按 BrowserConfig signature 复用可以显著降低：

- Chromium 冷启动；
- RSS 峰值；
- renderer 数量抖动；
- JS Runtime 初始化成本。

但 signature 的复用不能跨越安全边界。平台的 isolation key 至少加入：

```text
runtime_profile_release
network_policy_class
credential_scope
session_isolation_class
proxy_policy_class
browser_feature_class
```

公开技术博客默认走 `public-anonymous`，不使用持久 cookie/profile。需要认证的 Source 必须单独审批、独立 credential scope 和池化边界。

### 8.2 Context Generation 与 Process Generation 分离

只关闭 Browser Context 不能证明 Chromium 主进程和 renderer 长期泄漏已消除。7x24 运行时应增加两层代际：

- Context：按页面数、错误、污染风险、session 状态回收；
- Browser Process：按进程 age、累计 page、RSS、FD、renderer crash、错误率主动 drain + replace。

处理流程：

```text
ACTIVE -> DRAINING
  禁止新 admission
  等待 active page 收敛
  超时则 cancel/kill
  校验 active=0
  restart browser/process/pod
  readiness + contract probe
  ACTIVE
```

不能在仍有平台 attempt 绑定时直接点 restart。

---

## 9. Dispatcher 与 Streaming Result

Crawl4AI Multi-URL 能力中的 `MemoryAdaptiveDispatcher`、`SemaphoreDispatcher`、`RateLimiter` 和 streaming mode 很适合 Runtime 内部使用。

关键原则：

1. `MemoryAdaptiveDispatcher` 只知道本实例内存，不知道全局 Browser Slot；
2. Runtime RateLimiter 只做局部保护，目标 host 的长期速率状态应由平台维护；
3. 批量/Deep Crawl 结果优先 streaming 逐条物化；
4. 绝不能假设“输入列表位置 == 输出列表位置”；
5. 每一条结果必须携带/恢复 URL identity、runtime task id、attempt correlation；
6. cardinality mismatch、duplicate result、unknown result 都要显式告警。

对于 1000 Source，较安全的方式不是一次给 Runtime 塞几万个 URL，而是平台生成细粒度 durable task，再由 Runtime Adapter 做受控 micro-batch。

---

## 10. Monitor / WebSocket / Prometheus 的正确定位

官方 monitor 能观察：

- CPU / Memory / Network / Uptime；
- active requests；
- completed request 窗口；
- Browser Pool；
- janitor；
- errors；
- timeline；
- WebSocket 实时更新；
- `/metrics` Prometheus。

这些信号用于 **Runtime Capacity 与运维判断**，不能直接作为业务 Coverage。

平台需要把两类 telemetry 分开：

```text
Runtime Telemetry
- queue saturation
- browser pool reuse
- memory pressure
- runtime 429/503/504
- restart/drain
- artifact quota/promotion

Business Telemetry
- discovered URL count
- fetched article count
- PASS Markdown count
- known gap
- backfill stop reason
- incremental freshness
- extraction quality
```

Runtime 绿灯但 URL Inventory 漏了一半，业务仍然失败。

---

## 11. Monitor Control Action 的平台化

0.9 的 cleanup / kill_browser / restart_browser 是 admin-scope 运维动作。知识库 Web Admin 不应直接把这些 Runtime endpoint 暴露给普通操作员。

平台提供高层动作：

```text
Drain Runtime
Recycle Browser Generation
Force Cold Pool Cleanup
Quarantine Runtime
Resume Runtime
```

每个动作都：

1. 记录 operator / reason / correlation id；
2. 先从 Runtime Registry 摘除新流量；
3. 等待或接管 active attempts；
4. 调用底层 admin endpoint；
5. readiness + capability + contract probe；
6. 再加入路由。

这能避免“为了降内存点一下 restart，把正在抓取的 200 个平台任务变成幽灵任务”。

---

## 12. 0.9 Secure-by-default 的安全含义

`MIGRATION.md` 中最值得平台吸收的是“网络调用只允许低能力、声明式配置”。重要变化包括：

- 默认认证；
- 无 token 时默认 loopback；
- 需要暴露时应放 TLS reverse proxy；
- CORS deny by default；
- TLS verification 默认开启；
- Redis loopback/password，不再公开端口；
- Monitor control 要 admin principal；
- hooks 从任意 Python code 改为固定 declarative actions；
- 禁止通过网络请求传 `js_code`、`proxy`、`user_data_dir`、`cdp_url`、`cookies`、`headers`、`base_url`、`deep_crawl_strategy`、`magic` 等高能力字段；
- LLM endpoint 只按 provider name 选择，key/base URL 服务端持有；
- screenshot/PDF caller 不再拥有路径；
- request body、queue、wall-clock 可设置硬限制；
- webhook header 做敏感字段校验。

### 12.1 对平台的进一步要求

即使 Runtime 已做 hardening，平台仍要再加：

- Source Profile -> Config Compiler -> Runtime Config，而不是原样透传 JSON；
- SSRF/私网/metadata 地址阻断；
- redirect 每跳复验；
- DNS rebinding 防护 / egress pinning；
- secret 只能用引用，不落用户配置；
- Runtime 只部署私网，普通用户永不直接获得 runtime token；
- admin token 只在 Runtime Gateway/Operator Controller；
- Chromium 容器使用 read-only rootfs、tmpfs、cap_drop、no-new-privileges、合理 shm；
- 对 `--no-sandbox` 风险显式登记，条件允许时启用 unprivileged user namespace/seccomp 后打开 Chromium sandbox。

---

## 13. Runtime Registry 与 Capability Contract

Self-Hosting 文档和源码可能跨版本漂移，因此平台必须把“运行时行为”建模为 release：

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
- status
- last_probe_at
```

每次加入流量前 probe：

- auth 是否生效；
- loopback/private exposure 策略；
- `/health` / monitor schema；
- WorkQueue 是否有界且已 started；
- 429/503 行为；
- result schema；
- artifact endpoint；
- hook allowlist；
- forbidden power fields 是否确实被拒绝；
- internal URL / redirect 是否被拒绝；
- monitor admin action 是否需要 admin scope。

行为不匹配就 fail loud，不做“尽量兼容”。

---

## 14. 推荐平台状态机

一次 Browser Attempt 推荐：

```text
LEASED
  -> ADMITTED
  -> RUNTIME_SUBMITTING
  -> RUNTIME_QUEUED
  -> RUNTIME_RUNNING
  -> RUNTIME_TERMINAL
  -> MATERIALIZING_RESULT
  -> PROMOTING_ARTIFACTS
  -> MATERIALIZED
  -> RAW_PERSISTED
  -> SUCCEEDED
```

异常态：

```text
RUNTIME_BACKPRESSURE
RUNTIME_LOST
RESULT_SCHEMA_MISMATCH
RESULT_CARDINALITY_MISMATCH
ARTIFACT_EXPIRED
ARTIFACT_QUOTA
CANCEL_REQUESTED
DRAINING
FAILED_RETRYABLE
FAILED_FINAL
```

其中 **RUNTIME_TERMINAL 不是 SUCCEEDED**。只有 result 已逐条物化、required artifacts 已永久保存、Raw Artifact 已入对象存储后，平台才推进后续 extraction。

---

## 15. 典型故障与处理

| 故障 | 不能怎么做 | 推荐处理 |
|---|---|---|
| Runtime 503 | 紧循环重试 | 遵守 Retry-After，退回 durable queue，降低实例 capacity score |
| Runtime 429 | 加更多并发 | 降 principal 提交速率，检查 per_principal 配置 |
| Pod 重启 | 默认 runtime job 会恢复 | lease + reconciliation，必要时新 attempt |
| webhook 丢失 | 永久等待 | 主动 status reconciliation |
| webhook 重复 | 重复生成文档 | inbox 幂等去重 |
| artifact 404 | 当“无截图”忽略 | 区分 optional/required；required 标记 ARTIFACT_EXPIRED 并 repair |
| artifact quota 满 | 继续提交截图任务 | 暂停高 artifact workload，优先 promotion/cleanup |
| browser RSS 高 | 直接全局 restart | Drain -> quiesce -> recycle -> readiness |
| result 数量不等于输入 | 取相同下标 | 按 URL/correlation identity 对齐，mismatch fail loud |
| monitor 健康 | 推断抓取完整 | Coverage 独立由 URL Inventory/Evidence 计算 |

---

## 16. 必须增加的 Contract / Stress Test

### 16.1 Queue

- `maxsize` 压满后必须 503；
- `Retry-After` 存在；
- `per_principal` 超限必须 429；
- `0` 值不允许从普通 Source Profile 编译出来；
- WorkQueue 未 started 时生产 readiness 失败；
- Pod restart 后平台能恢复 orphan attempt。

### 16.2 Artifact

- opaque id 格式；
- caller path 被拒绝；
- required artifact 在 TTL 内被晋升；
- expired artifact 会触发明确错误；
- quota 满时平台进入 backpressure；
- MIME/size/hash 不匹配时不建立 durable reference。

### 16.3 Security

- 无 auth 不能调用数据 API；
- data principal 不能调用 monitor admin action；
- `js_code/proxy/cookies/cdp_url/user_data_dir/base_url` 等网络 power field 被拒绝；
- internal IP、metadata IP、redirect 到私网都被拒绝；
- webhook arbitrary destination 不可由普通用户指定。

### 16.4 Browser lifecycle

- drain 后不再 admission；
- active request 能收敛；
- 超时强制回收会把平台 attempt 标记为明确可恢复状态；
- recycle 后 browser/process generation 递增；
- readiness 未通过前不重新路由。

### 16.5 Result integrity

- streaming 顺序打乱仍能正确关联；
- duplicate result 幂等；
- missing result / extra result fail loud；
- runtime schema 漂移时隔离实例。

---

## 17. 对博客知识库最终方案的具体优化

本次 Self-Hosting 调研应在总方案中固化以下新增/强化设计：

1. **Runtime Queue 与 Durable Queue 明确分层**：Redis Streams/PostgreSQL 是平台任务真相，Crawl4AI `asyncio.Queue` 只是局部保险丝；
2. **双层 Admission**：平台解决全局公平和硬预算，Runtime 解决单实例保护；
3. **Runtime Registry + Contract Probe**：避免文档/版本漂移导致静默兼容错误；
4. **Webhook Inbox + Reconciler**：webhook 走快速路径，对账保证最终收敛；
5. **Browser 两级代际与 Drain**：context recycle 与 browser process recycle 分离；
6. **Runtime Artifact Promotion Gate**：短 TTL runtime artifact 必须先晋升到 S3/MinIO，再允许平台 Attempt 终态；
7. **Admin Action 平台化**：cleanup/kill/restart 必须先摘流、收敛 active attempt、审计；
8. **0 != 安全默认**：Runtime 中 `0=unbounded` 的配置不能直接暴露给用户；
9. **Runtime Telemetry 与 Coverage Telemetry 分离**；
10. **secure-by-default 再包一层 Capability Firewall**：网络 API 永远只接受声明式低能力 intent。

---

## 18. 最终判断

Crawl4AI Self-Hosting 的最佳位置不是“整个平台”，而是一个 **可替换、受约束、可观测、可隔离的浏览器执行运行时**。

平台必须把长期正确性握在自己手里：

```text
Coverage truth       -> PostgreSQL
Durable queue        -> PostgreSQL + Redis Streams
Raw evidence         -> S3/MinIO
Runtime queue        -> Crawl4AI asyncio.Queue（临时）
Runtime status       -> Crawl4AI Redis（临时）
Browser pool         -> Crawl4AI（性能缓存）
Runtime artifact     -> Crawl4AI（短 TTL，必须晋升）
Final Markdown       -> Canonical IR projection
Incremental truth    -> URL observation + document version
```

在这个边界下，Self-Hosting 能显著减少 Browser Runtime、资源池、监控、安全 hardening 的自研成本，同时不会把历史完整性、任务恢复和知识库可重放性绑定到第三方运行时的内部实现。