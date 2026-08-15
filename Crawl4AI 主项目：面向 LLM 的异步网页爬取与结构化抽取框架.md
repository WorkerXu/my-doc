# Crawl4AI 主项目：面向 LLM 的异步网页爬取与结构化抽取框架

## 来源与分析基线

- 项目：https://github.com/unclecode/crawl4ai
- 分支：`main`
- 本次核对 commit：`7e801521428ee12509994d39151006f64055ebe3`
- 稳定版本：`0.9.2`
- 重点代码与文档：
  - `crawl4ai/async_webcrawler.py`
  - `crawl4ai/async_configs.py`
  - `crawl4ai/async_dispatcher.py`
  - `crawl4ai/browser_manager.py`
  - `crawl4ai/models.py`
  - `tests/async/test_browser_recycle_v2.py`
  - `docs/blog/release-v0.9.0.md`
  - `docs/blog/release-v0.9.2.md`
  - Docker server 的安全、资源治理与 Artifact 实现/测试

## 1. 结论

Crawl4AI 适合进入博客知识库平台的 **Fetch / Browser Render / Deep Crawl Prefetch Runtime**，但不应成为 Source 生命周期、历史 Coverage、全局 Frontier、URL Identity、全局限流、最终文档版本或 Canonical Markdown 的业务真相。

对 1000+ 技术博客场景，最重要的不是“Crawl4AI 可以直接输出 Markdown”，而是它当前实现暴露出的生产边界：

1. **配置本身就是能力**：网络调用者不能直接获得任意 `CrawlerRunConfig`、浏览器启动参数、Proxy、CDP、持久用户目录、任意 JS 或 Deep Crawl Strategy；
2. **Runtime 的成功不是文章有效**：`CrawlResult.success` 更接近一次抓取/运行成功，不能替代 Soft-404、正文质量、文章类型与 Canonical Version 判定；
3. **多结果必须显式物化**：`CrawlResultContainer` 为兼容单结果用法会把部分属性访问代理到第一个结果，多页 Deep Crawl/批量结果若被 Adapter 当单结果读取，会形成“任务成功但静默漏页”的高风险错误；
4. **`prefetch=True` 是非常适合生产 Discovery Plane 的快速路径**：它可以跳过重型正文清洗和 Markdown，优先提取链接与基础抓取证据，但其输出只能成为 Observation/Edge，不能直接成为文档或 Coverage 完成证据；
5. **停止必须达到 Runtime Quiescence**：v0.9.2 专门修复了流式消费提前关闭后任务、页面、上下文仍运行的问题，平台必须把“停止接单”和“运行时资源已清零”区分开；
6. **长期 Browser Worker 必须有代际回收**：当前 BrowserManager 已具备按页面数滚动 browser generation、旧 generation 引用计数排空和 pending cleanup 的机制。生产平台不能让 Browser 进程无限寿命，也不能把 `max_pages_before_recycle=0` 的关闭状态作为无意识默认；
7. **诊断证据要有预算**：MHTML、network requests、console、screenshot、PDF 等非常适合排障，但体积和敏感性都高，不适合作为所有抓取的默认永久证据；
8. **调用者不能拥有服务端输出路径**：临时截图/PDF/Trace 必须以 opaque artifact id 访问，路径、TTL、配额、授权由平台拥有。

因此推荐架构是：

```text
Platform Truth & Control
PostgreSQL / Object Storage / Source Profile / Coverage / Frontier
                 |
     Safe Runtime Config Compiler
                 |
      Crawl4AI Runtime Adapter
       /                  \
HTTP/Prefetch         Browser/Deep Crawl
       \                  /
       Runtime Result Materializer
                 |
        Fetch Evidence Envelope
                 |
    Raw Artifact / Observation
                 |
 Extraction -> Quality -> Canonical IR -> Markdown
```

---

## 2. `AsyncWebCrawler` 的真实职责

`AsyncWebCrawler` 不是单纯的 Playwright 包装。一次 `arun()` 会综合处理配置、缓存、抓取策略、浏览器/HTTP、重定向、HTML、下载、截图/PDF/MHTML、正文处理、Markdown、结构化抽取以及最终 `CrawlResult`。

从平台角度应把它看成 **一次 Runtime 执行器**：

```text
Platform Fetch Command
 -> CrawlerRunConfig / BrowserConfig
 -> Runtime Cache Context
 -> HTTP or Browser Strategy
 -> Response / Redirect / HTML / Binary
 -> Optional Content Processing
 -> CrawlResult
```

但平台不应直接把 `CrawlResult` 升级为最终业务对象。正确路径应是：

```text
CrawlResult
 -> Runtime Result Materializer
 -> Fetch Evidence Envelope
 -> Immutable Raw Artifact
 -> URL Validity / MIME Route
 -> Extraction Candidate
 -> Quality Gate
 -> Canonical IR
 -> Document Version
```

这样即使 Crawl4AI 的返回对象、缓存、Markdown 算法或内部浏览器策略升级，业务真相仍可重放和迁移。

### 2.1 Runtime `success` 的边界

当前实现中，抓取成功可以由“存在 HTML”或“存在下载文件”等运行条件成立，并不表示：

- URL 是文章；
- 页面不是 Soft-404；
- HTML 含完整正文；
- 页面未被挑战页/登录页替代；
- Markdown 满足知识库质量；
- 二进制内容已经被正确解析。

因此必须坚持：

```text
runtime_success != url_valid != article_valid != extraction_pass
```

平台需要分别记录这几个判定层。

---

## 3. Config Provenance：为什么配置是安全边界

Crawl4AI 0.9.x 已明确把网络请求体视为 UNTRUSTED，而 SDK/进程内调用可视为 TRUSTED。这个变化的核心并不是普通参数校验，而是 **Capability Reduction**。

### 3.1 类型级 Allowlist

不可信网络输入只允许有限配置/策略类型，例如基础 Browser/Crawler 配置、部分非 LLM 抽取/清洗/Markdown 类型。高能力对象故意不开放，包括：

- LLMConfig 与 LLM 策略；
- ProxyConfig / Proxy Rotation；
- Deep Crawl Strategy；
- Dispatcher；
- Domain Mapper / URL Seeder；
- 其它能够执行代码、读取 secret、改变网络路径或递归扩张工作量的类型。

这比“允许所有类型，再列危险 denylist”更安全：未来 Runtime 新增类型时不会自动穿透 Web/API 边界。

### 3.2 字段级 Forbidden

典型高危字段包括：

```text
BrowserConfig:
proxy / proxy_config
extra_args
user_data_dir
cdp_url / debugging_port
cookies / storage_state
headers / init_scripts
browser_context_id / target_id

CrawlerRunConfig:
js_code / js_code_before_wait
c4a_script
deep_crawl_strategy
proxy_config / proxy_rotation_strategy
fallback_fetch_function
base_url
simulate_user / magic
process_in_browser
session_id / shared_data
```

这些字段即使类型合法，也能改变代码执行、网络、浏览器身份、会话或递归范围，所以不能由普通 Web 用户直接提交。

### 3.3 数量级 Clamp

不可信输入还必须约束：

```text
page timeout
wait timeout
scroll steps
viewport
request size
seed count
deep crawl depth/pages
concurrency
wall-clock
artifact bytes
```

对生产平台尤其要避免“0 无意中表示无限”。若某个第三方字段用 0 表示关闭限制，平台应在自己的声明式 Schema 中重新定义安全语义；只有 Operator 发布的显式 unrestricted policy 才可解除上限。

### 3.4 平台应再加一层 Capability Firewall

即使 Crawl4AI 已有 UNTRUSTED 模式，也不应把其网络 API 配置模型原样暴露给博客平台用户。推荐：

```text
Web Source Profile
 -> JSON Schema
 -> Semantic Validation
 -> Capability Allowlist
 -> Numeric Budget Clamp
 -> Server-side Secret Ref Resolution
 -> Operator Policy Injection
 -> Config Compiler
 -> Compilation Report
 -> Immutable Runtime Config Release
 -> TRUSTED in-process Crawl4AI object
```

Web 用户编辑的是业务意图，例如：

```yaml
fetch:
  route: auto
  wait_until: domcontentloaded
  page_timeout_ms: 30000
  screenshot_on_failure: true
```

而不是浏览器内部对象。

---

## 4. `MemoryAdaptiveDispatcher`：只能是 Worker Local Protection

`MemoryAdaptiveDispatcher` 维护本地 PriorityQueue、active tasks、session permit、内存压力、fairness timeout 与本地 RateLimiter。它可以很好地解决：

- 单 Worker 内存压力；
- 单 Worker Browser 并发；
- 局部排队；
- 局部响应式退避。

但它不能成为全局 Scheduler，因为：

- 队列是进程内状态；
- Worker crash 后本地待执行项会丢；
- 本地 RateLimiter 看不到其它 Worker；
- 无法保证同一域名的跨 Worker 总 QPS；
- 不知道 Backfill、Incremental、Repair、Browser Slow Lane 之间的系统级公平；
- 不能承担业务 Lease、Fencing、DLQ 或 Coverage。

正确组合：

```text
PostgreSQL Persistent Frontier
 -> Global Fair Scheduler
 -> Redis Domain Token Bucket / Resource Semaphore
 -> Worker Lease + Fencing
 -> MemoryAdaptiveDispatcher
 -> Crawl4AI Runtime
```

---

## 5. v0.9.2 的流式取消问题：停止不是 generator 返回

v0.9.2 修复了一个对生产抓取非常关键的问题：`run_urls_stream()` 的调用方消费部分结果后关闭/取消 async generator，旧实现中的 per-URL task 仍可能继续运行并持有 browser/context/page/session。

结果可能包括：

- 页面与 context 泄漏；
- 流已经关闭但网站仍收到请求；
- 后续 `arun_many()` 与旧任务重叠；
- `TargetClosedError`；
- 同一 crawler 被前后两批任务串扰；
- Web 显示“停止”，后台仍消耗带宽和 Browser。

当前修复思路是：

```text
finally:
1. cancel 未完成 active task
2. await gather(..., return_exceptions=True)
3. drain queued-but-unstarted URL
4. cancel memory monitor
5. await memory monitor
6. stop monitor
```

回归测试验证 active task 全结束、阻塞任务被取消、本地 queue 为空、session 回到 0。

### 5.1 平台必须有 Runtime Quiescence Barrier

建议状态：

```text
RUNNING
 -> STOP_REQUESTED
 -> QUIESCING
 -> QUIESCED
 -> STOPPED / RETRY / COMMITTED
```

收到 Cancel/Pause/Lease Lost/Recycle/Shutdown 时，必须：

```text
stop local admission
cancel active runtime tasks
await active tasks with grace timeout
drain local queue
close/release page/context/session
cancel helper/monitor tasks
release domain/browser/resource tokens
flush cleanup evidence
verify counters == 0
```

只有：

```text
active_tasks == 0
local_queue == 0
pages == 0
contexts == 0
sessions == 0
held_tokens == 0
```

才能复用 crawler/browser/worker。

若超时：kill 隔离进程/Pod，依靠 fencing 阻止迟到提交，未完成 URL 回到 RETRY。

---

## 6. 新增关键边界：多结果必须显式物化

这是现有方案还需要特别加强的地方。

Crawl4AI 的 `CrawlResultContainer` 可以容纳一个或多个 `CrawlResult`。为了兼容历史上的单结果用法，它对某些属性访问会委托给容器中的第一个结果。这个设计对交互式 SDK 很方便，但对生产 Adapter 有风险：

```text
Deep Crawl / 多结果 Runtime
 -> CrawlResultContainer([r1, r2, r3, ...])
 -> 业务代码直接读取 container.xxx
 -> 实际只看到 first result
 -> Task 看似成功，但 r2/r3/... 静默丢失
```

因此 Adapter 必须规定 **Result Materialization Contract**：

1. 任意 Runtime 返回先转换为 `list[CrawlResult]` 或逐条流式 materialize；
2. 禁止业务代码对未物化的 Container 直接读取正文/URL/Markdown；
3. 每个结果必须生成独立的 `runtime_result_ordinal` 或稳定 runtime result id；
4. 结果关联只依赖 `attempt_id + url_id + requested_url` 等身份，不依赖数组位置；
5. Deep Crawl 一个 root 产生 N 个页面时，N 个结果全部进入 Observation/Artifact 路径；
6. 若预期 cardinality 与 materialize 数不一致，应 FAIL LOUDLY，而不是默认只取第一条；
7. 流式结果可以乱序到达，状态更新必须幂等。

建议 Runtime Adapter 输出统一 Envelope：

```text
runtime_result_envelope
- attempt_id
- root_url_id
- requested_url
- runtime_result_ordinal
- result_url
- final_url
- status_code
- redirected_url
- redirected_status_code
- content_type
- runtime_release_id
- runtime_config_release_id
- result_schema_release_id
- payload/artifact refs
- error_class
```

这样可以避免 Runtime 返回结构升级对业务层产生隐式语义变化。

---

## 7. `prefetch=True`：应明确成为两阶段 Discovery 快路径

当前 `AsyncWebCrawler` 在 `prefetch=True` 时，会绕过重型正文处理/Markdown 路径，优先从 HTML 快速提取 links，并保留基础状态、重定向等结果。这与 1000+ 网站 Backfill 的需求高度匹配。

### Phase A：Discovery Prefetch

```text
URL / Deep Crawl Seed
 -> Browser render if necessary
 -> Crawl4AI prefetch=True
 -> HTML + raw links + status/redirect
 -> Runtime Result Materializer
 -> Persist Observation/Edge
 -> Normalize
 -> Approved Scope
 -> Robots
 -> Graph Depth
 -> Filter/Score
 -> Persistent Frontier
```

这个阶段明确 **不做**：

- 重型 Markdown；
- LLM；
- Canonical Document Version；
- PASS 文章计数；
- Historical Coverage 完成判定。

### Phase B：Full Fetch

```text
Persistent Frontier
 -> Platform Admission
 -> HTTP First / Browser Slow Lane
 -> Raw Artifact
 -> URL Validity
 -> Extraction Portfolio
 -> Quality Gate
 -> Canonical IR
 -> Markdown Projection
```

这一拆分能显著降低 Deep Crawl 的 Browser/CPU 成本，也能保证“Observation Before Selection”。

### 7.1 必须避免的误用

`prefetch` 结果只能证明“Runtime 观察到了页面/链接”，不能证明：

- 链接是历史文章；
- 页面正文抓取完成；
- Markdown 已生成；
- Backfill 已完成。

平台应在类型或状态机上把 `DISCOVERY_OBSERVATION` 与 `FETCHED_ARTIFACT` 明确隔离。

---

## 8. 新增关键边界：Browser Generation 生命周期

长期抓 1000+ 站时，Browser 内存增长、V8 heap、页面/context 残留和浏览器本身的长期碎片化很难完全依靠单页面 close 解决。

Crawl4AI 当前 BrowserManager 已出现针对长期生命周期的机制：

- `memory_saving_mode`：面向高吞吐场景的内存约束；
- `max_pages_before_recycle`：达到页面数阈值后触发浏览器代际切换；
- browser version 进入 context/config signature；
- 旧 generation 进入 pending cleanup；
- 通过 refcount 等待旧 context 的在途请求结束；
- 新 generation 可以继续服务新请求；
- 对 pending browser 数量有安全上限，避免无限堆积。

相关测试验证了：阈值触发 version bump、关闭回收时 generation 不变、并发抓取时旧新代可共存、引用归零后清理。

### 8.1 对平台的落地：Browser Generation Manager

推荐把 Runtime 内部能力提升为平台可观测生命周期：

```text
ACTIVE generation G1
  | pages/age/RSS/error threshold
  v
DRAINING G1 + ACTIVE G2
  | no new admission to G1
  | wait G1 refs -> 0
  v
QUIESCED G1 -> CLOSED
```

平台记录：

```text
browser_generation
- id
- worker_id
- pool_key
- generation_no
- runtime_release_id
- runtime_config_release_id
- isolation_key
- created_at
- pages_served
- peak_rss_bytes
- active_refs
- state               # ACTIVE/DRAINING/QUIESCED/KILLED
- drain_reason         # PAGE_LIMIT/AGE/RSS/ERROR/OPERATOR/SHUTDOWN
- draining_at
- quiesced_at
```

### 8.2 回收阈值必须是平台策略

建议至少同时支持：

```text
max_pages_per_generation
max_generation_age
max_worker_rss
max_consecutive_browser_errors
max_draining_generations
```

不要只依赖第三方默认值。特别是 `max_pages_before_recycle=0` 代表关闭回收时，生产配置必须显式批准，而不能因为默认值而让 Browser 无限寿命。

### 8.3 回收必须与 Quiescence/Fencing 结合

Browser generation 回收不是简单 `close()`：

```text
停止向旧代派新 page
 -> 新代接流量
 -> 旧代只 drain 已存在引用
 -> grace timeout
 -> 正常清零则 QUIESCED
 -> 超时则 kill worker/process
 -> fencing 阻止旧代迟到写入
```

这样既避免全局停机，又避免旧代无限悬挂。

---

## 9. Context/CDP/身份状态必须隔离

BrowserManager 能维护 context、persistent browser 状态、CDP 连接及运行时状态复制能力。这对受信任自动化非常强，但对公共博客知识库意味着更大的隔离要求。

普通公开博客 Lane 建议：

```text
isolation_key = tenant/source-lane/runtime-config-release/browser-generation
```

禁止：

- 不相关 Tenant 共享持久 context；
- 不相关 Source 共享认证 Cookie/localStorage；
- 公开抓取默认使用 user-data-dir；
- Web 用户指定 CDP URL；
- 把 session/profile 跨 Source 复用；
- 因连接缓存而跨安全域共享 Browser 身份。

如果未来抓组织自有的私有站点，应单独建立 `AUTHENTICATED_SOURCE` Lane：独立 Secret、Worker Pool、Network Policy、浏览器 profile、审计和生命周期，不与公共博客混用。

---

## 10. Fetch Evidence Envelope：不要让 Runtime Schema 直接成为业务 Schema

`CrawlResult` 当前可能携带：

- URL / HTML / cleaned HTML；
- media / links；
- downloaded files；
- screenshot / PDF / MHTML；
- extracted content；
- metadata / error；
- response headers / status；
- redirect URL/status；
- SSL certificate；
- network requests / console messages；
- dispatch result 等。

平台应该从中提取稳定的业务证据，而不是直接序列化整个第三方对象作为数据库真相。

推荐：

```text
fetch_evidence
- id
- attempt_id
- runtime_result_ordinal
- requested_url
- result_url
- final_url
- status_code
- redirect_chain_ref
- content_type
- selected_response_headers_ref
- payload_sha256
- runtime_cache_status
- runtime_release_id
- runtime_config_release_id
- runtime_result_schema_release_id
- raw_artifact_id
- captured_at
```

其中 Header 要做 allowlist/redaction，避免把 Cookie、Authorization 等敏感内容带入普通日志。

### 10.1 重型诊断 Artifact 采用按需策略

以下证据成本高或可能包含敏感数据：

```text
screenshot
PDF
MHTML
network requests
console messages
full headers
trace
```

建议按策略捕获：

```text
PASS normal fetch     -> 默认不抓重型诊断
REPAIR/blocked/error  -> 按失败类型抓
canary/debug source   -> 采样抓
operator manual run   -> 显式抓
```

并应用：

- size cap；
- tenant/source quota；
- TTL；
- RBAC；
- redaction；
- 独立 Ephemeral Artifact 生命周期。

Raw HTML/PDF 等用于重放的 durable evidence 与 UI 调试证据必须分开。

---

## 11. Runtime Cache 与增量同步的正确关系

Crawl4AI 的缓存/freshness 能利用 ETag、Last-Modified、head fingerprint 等信息减少重复工作，这一点值得借鉴，但 Runtime Cache 仍只是 Worker 优化。

平台增量同步应拥有自己的证据链：

```text
provider updated/lastmod
+ ETag
+ Last-Modified
+ platform head fingerprint
+ raw payload hash
+ Canonical IR content hash
```

规则：

- RSS/CMS/Sitemap 明确变化 -> 高优复抓；
- HTTP validator -> Conditional GET；
- 304 -> 更新 freshness observation，不产生正文版本；
- 无可靠 validator -> HEAD/GET + fingerprint；
- 最终是否产生 `DocumentVersion` 由平台 Canonical IR/content hash 决定。

Runtime cache hit 不能：

- 推进 Provider Cursor；
- 直接改变 Coverage；
- 取代平台 Raw Artifact 证据；
- 自动判断 Document Version 未变化。

---

## 12. Deep Crawl 的正确边界

Native Deep Crawl 很适合做 URL Gap Provider 和 Browser Prefetch，但不应承担历史完整性真相：

- `max_pages` 是执行预算，不是历史边界；
- visited 是 runtime state；
- local queue crash 可丢；
- runtime depth 不应代替平台持久 URL Edge graph depth；
- score/threshold 是排序/执行策略，不应删除尚未保存的 Observation。

因此：

```text
Deep Crawl Result
 -> Materialize all results
 -> Persist raw links/Observation/Edge first
 -> Platform graph depth
 -> Platform scope/robots
 -> Filter/Score
 -> Persistent Frontier
```

如果需要断点恢复，Runtime checkpoint 也只能是执行优化；业务恢复仍依赖 PostgreSQL Frontier、Lease、Fencing 与持久 Observation。

---

## 13. Markdown 与正文抽取：Runtime 输出只做 Candidate

Crawl4AI 的 Markdown/正文抽取对快速使用非常方便，但长期知识库必须保持：

```text
Immutable Raw Artifact
 -> Extraction Portfolio
 -> Candidate Quality Gate
 -> Canonical IR
 -> Deterministic Markdown Renderer
```

原因：

- extractor 升级可离线 Replay；
- renderer 升级不重新访问网站；
- 可以稳定处理标题、代码块、表格、列表、图片；
- 可以对比多 extractor candidate；
- 可以保留旧 Document Version；
- 能解释“为什么当前 Markdown 变了”。

Crawl4AI runtime Markdown 可以作为 Candidate/诊断对照，但不是最终 Canonical Markdown。

---

## 14. API / Scheduler 双层 Resource Admission

入口层要阻止无限工作进入系统：

```text
request_bytes
source_count
seed_url_count
manual_run_count
active_run_count_per_principal
preview_count
recipe/config size
```

推荐语义：

```text
413 = 请求本身过大
429 = principal/tenant 当前超配额
503 + Retry-After = 平台队列/资源临时满
400/422 = 配置能力或语义非法
```

Scheduler 再硬控真实执行：

```text
pages admitted
HTTP attempts
browser attempts
preview requests
model calls
wall-clock
object bytes
per-domain QPS/concurrency
per-source leases
browser generations/draining generations
```

预算按 attempt/call 统计，不只按成功 URL 统计。

---

## 15. Opaque Artifact Store 与路径安全

Crawl4AI 0.9.0 移除了 Screenshot/PDF API 的 caller-supplied `output_path`，核心思想正确：路径字符串校验很难完整抵御 path traversal、symlink、sibling-prefix 与特权进程任意写入。

平台应区分：

### Durable Evidence Artifact

Raw HTML、原始 PDF、Canonical IR 等写入 S3/MinIO，由服务端生成 object key，按数据政策长期保存。

### Ephemeral UI Artifact

Screenshot、调试 PDF、MHTML、network trace、console dump、Preview 等使用：

```text
opaque artifact_id
server-generated key
owner/tenant
TTL
size cap
quota
RBAC
janitor
```

如果使用本地临时盘，还要有 O_EXCL、O_NOFOLLOW、regular-file check、0600/0700 权限等保护。

---

## 16. 对博客知识库最终方案的优化项

基于当前 Crawl4AI 主项目，本方案应明确加入以下能力。

### 16.1 Result Materialization Contract

新增 Runtime Adapter 硬规则：容器/流必须逐条物化，不允许通过兼容代理静默只消费 first result；Deep Crawl/批量结果必须按身份逐条提交。

### 16.2 Runtime Result Schema Release

新增 `runtime_result_schema_release`，Runtime 升级时对返回字段/cardinality/redirect/binary/stream 行为做 Contract Test。

### 16.3 Prefetch 两阶段状态机

把 Crawl4AI `prefetch=True` 明确映射到 `DISCOVERY_OBSERVATION`，不能生成 PASS Document；只有 Phase B Full Fetch 进入 Raw Artifact -> Extract。

### 16.4 Browser Generation Manager

增加 generation 生命周期、pages/age/RSS/error 阈值、draining 上限、旧代引用排空、Quiescence 与 kill/fencing 收敛。

### 16.5 Browser State Isolation Key

context/session/CDP/persistent state 只能在相同安全隔离键内复用，公开 Source 默认无认证持久身份。

### 16.6 Fetch Evidence Envelope

稳定提取请求 URL、最终 URL、redirect、status、content type、payload hash、选定 header、Runtime release；第三方 `CrawlResult` 不直接作为业务数据库对象。

### 16.7 Diagnostic Evidence Budget

MHTML/network/console/screenshot/trace 默认关闭或按失败/采样启用，有 size/TTL/quota/redaction。

### 16.8 Binary/MIME Route

Runtime 因 downloaded file 成功时，不能误进 HTML 正文抽取。平台根据 MIME/Content-Disposition 分流到 PDF/文档解析或 REJECT/REVIEW。

---

## 17. 必须新增的 Contract / Golden Tests

除现有配置安全、Quiescence、Artifact 测试外，建议加入：

```text
1. CrawlResultContainer 含 2+ 结果 -> 全部被 materialize
2. Adapter 禁止 first-result proxy 读取 -> 静态/单测检测
3. arun_many/stream 乱序 -> attempt_id/url_id 关联正确
4. Deep Crawl root 产生 N 结果 -> N 条 Observation/Envelope，无静默缺失
5. prefetch=True -> 不产生 Canonical Markdown/Document Version
6. prefetch link -> Observation/Edge 先落库再过滤
7. empty HTML + downloaded file success -> MIME-specific pipeline
8. Runtime upgrade -> result schema/cardinality drift 被检测
9. browser recycle threshold -> generation 切换且无 URL 丢失/重复提交
10. old generation drain -> refcount 归零后关闭
11. max draining generations -> 超限触发隔离/退避，不无限增长
12. generation grace timeout -> kill + fencing rejects late commit
13. context/session state -> 不跨 tenant/source isolation key
14. CDP/persistent profile -> 普通 Web 配置不可启用
15. diagnostic artifact -> size/quota/TTL/redaction 生效
16. Runtime cache hit -> 不推进 Provider Cursor/Coverage
17. Runtime success + Soft-404 -> Validity Gate 仍可拒绝
```

---

## 18. 对 1000+ 博客需求的映射

| 需求 | Crawl4AI 可提供 | 平台必须额外拥有 |
|---|---|---|
| 全历史发现 | Deep Crawl、Browser、快速 link prefetch | Provider 并集、Persistent Inventory、Observation/Edge、CC/Wayback、Coverage/Known Gap |
| 批量执行 | async、stream、dispatcher | Result Materialization、attempt identity、Lease/Fencing、全局公平 |
| Markdown | Markdown/正文候选 | Raw Artifact、Quality Gate、Canonical IR、稳定 Renderer |
| 1000+ 站扩展 | 通用 Runtime | Source Profile、Capability Compiler、全局 Scheduler、域名限流 |
| 增量同步 | Runtime cache freshness 可参考 | Provider Cursor、Conditional GET、Version/Tombstone |
| JS-heavy 页面 | Browser/Playwright | Browser Slow Lane、Generation Recycling、Quiescence |
| Browser 内存 | Memory adaptive、recycle 能力 | generation policy、RSS/age/page 阈值、隔离与指标 |
| Web 管理 | Docker API 安全经验可参考 | RBAC、声明式配置、审核、Release、Coverage/Frontier UI |
| 停止/恢复 | dispatcher cleanup | Persistent Frontier、Lease/Fencing、Quiescence Barrier |
| 安全 | UNTRUSTED config、opaque artifact 思路 | 平台 SSRF/Scope/Robots/Network Policy/Secret Manager |
| 可追溯 | CrawlResult 运行证据 | Fetch Evidence Envelope、Raw Artifact、Release/Trace |
| 低成本 | prefetch、async、memory adaptive | HTTP First、两阶段 Discovery、诊断证据预算、跨站公平 |

---

## 19. 最终采用建议

采用 Crawl4AI，但固定为 **可替换、可版本化、受平台能力边界约束的 Runtime Adapter**。

平台负责：

```text
为什么抓
允许抓什么
何时抓
抓取预算
是否抓完整
URL/Document 身份
如何恢复
如何停止并证明静默
如何保存不可变证据
如何形成 Canonical Markdown
```

Crawl4AI 负责：

```text
如何高效执行一次 HTTP/Browser/Deep Crawl
如何在需要时快速 prefetch links
如何提供 Browser 渲染与局部调度能力
如何返回运行时抓取证据
```

最终推荐的关键新增能力是：

**Result Materialization Contract、Prefetch Two-Phase Contract、Browser Generation Manager、Fetch Evidence Envelope、Diagnostic Evidence Budget**。

这几项与已有的 **Capability Firewall、Runtime Quiescence Barrier、双层 Resource Admission、Opaque Artifact Gateway** 组合后，才能把 Crawl4AI 从“好用的爬虫 SDK”安全地转化为“1000+ 博客长期知识库里的可治理执行器”。