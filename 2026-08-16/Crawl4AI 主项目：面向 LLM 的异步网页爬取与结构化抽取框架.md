# Crawl4AI 主项目：面向 LLM 的异步网页爬取与结构化抽取框架

## 来源与分析基线

- 项目：https://github.com/unclecode/crawl4ai
- 分支：`main`
- 本次核对 commit：`7e801521428ee12509994d39151006f64055ebe3`
- 稳定版本：`0.9.2`
- 重点代码与测试：
  - `crawl4ai/async_webcrawler.py`
  - `crawl4ai/async_configs.py`
  - `crawl4ai/async_dispatcher.py`
  - `crawl4ai/browser_manager.py`
  - `crawl4ai/models.py`
  - `tests/async/test_browser_recycle_v2.py`
  - `docs/blog/release-v0.9.2.md`

本次不是只依据 README/文档判断能力，而是以源码执行路径和测试实际断言为准。特别是 Browser recycle 的命名与实现语义存在需要平台显式消歧的地方。

## 1. 总体结论

Crawl4AI 适合进入博客知识库平台的 **Fetch / Browser Render / Deep Crawl Prefetch Runtime**，但不应成为 Source 生命周期、历史 Coverage、全局 Frontier、URL Identity、全局限流、最终文档版本或 Canonical Markdown 的业务真相。

对 1000+ 技术博客场景，最重要的生产边界有九个：

1. **配置本身就是能力**：网络调用者不能直接获得任意 `CrawlerRunConfig`、浏览器启动参数、Proxy、CDP、持久用户目录、任意 JS 或 Deep Crawl Strategy；
2. **Runtime success 不是文章有效**：`CrawlResult.success` 更接近抓取/运行成功，不能替代 Soft-404、正文质量、文章类型与 Canonical Version 判定；
3. **多结果必须显式物化**：`CrawlResultContainer.__getattr__` 会把属性访问代理给第一个结果，生产 Adapter 若把多页结果当单结果读，会静默漏页；
4. **`prefetch=True` 应成为 Discovery 快路径**：它在 `aprocess_html()` 中直接快速提取链接并提前返回，适合 Observation/Edge，不适合直接产生 PASS Document；
5. **停止必须达到 Runtime Quiescence**：v0.9.2 专门修复流式 generator 提前关闭后任务、页面继续运行的泄漏；
6. **必须区分 Context recycle 与 Browser process recycle**：当前 `max_pages_before_recycle` 的可验证实现是通过 `_browser_version` 改变 Context signature、创建新 Context、排空并关闭旧 Context；源码和测试没有证明这条阈值路径会重启底层 Browser 进程。因此它只能作为 Runtime Context 生命周期优化，不能替代平台的进程/Pod 代际回收；
7. **Runtime Cache 只是局部优化**：不能推进 Provider Cursor、Coverage 或 Document Version；
8. **诊断证据必须有预算和生命周期**：MHTML、network、console、screenshot、PDF、trace 不应默认永久保存；
9. **调用者不能拥有服务端输出路径**：临时 Artifact 必须通过 opaque id、RBAC、TTL、quota 管理。

推荐边界：

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

## 2. `AsyncWebCrawler` 的真实执行链

`AsyncWebCrawler.arun()` 是复合 Runtime，而不是单一 Playwright 包装。它会处理 Runtime Cache、robots、代理/重试、浏览器或 HTTP 抓取、redirect、HTML、下载文件、可选截图/PDF/MHTML、正文处理、Markdown、结构化抽取与结果封装。

平台应把它视为一次 Runtime Attempt：

```text
Platform Fetch Command
 -> compiled BrowserConfig/CrawlerRunConfig
 -> Runtime Cache Context
 -> HTTP/Browser Strategy
 -> AsyncCrawlResponse
 -> optional aprocess_html
 -> CrawlResultContainer
 -> Result Materializer
```

业务链必须继续：

```text
Runtime Result
 -> Fetch Evidence Envelope
 -> Immutable Raw Artifact
 -> URL Validity / MIME Route
 -> Extraction Candidate
 -> Quality Gate
 -> Canonical IR
 -> Document Version
 -> Markdown Projection
```

这样 Runtime 内部 Markdown、缓存格式、浏览器策略或返回类型发生变化时，业务数据仍然可重放。

### 2.1 `success` 的语义边界

当前 live fetch 路径会把：

```text
bool(html) OR bool(downloaded_files)
```

作为运行成功的重要条件之一，之后才叠加 anti-bot 等判断。因此：

```text
runtime_success
!= URL 实际存在
!= URL 是文章
!= 正文完整
!= extraction_pass
!= canonical_version_accepted
```

特别是二进制下载成功但 HTML 为空时，必须进入 MIME Router，而不是 HTML Quality Gate。

---

## 3. Config Provenance：配置就是 Capability

Crawl4AI 当前代码已经把 SDK/进程内构造与网络不可信输入区分为 `TRUSTED` / `UNTRUSTED`。这不是普通字段校验，而是能力削减。

### 3.1 类型级 Allowlist

不可信网络输入只允许有限类型。高能力类型被排除，例如：

- LLMConfig / LLMExtractionStrategy / LLMContentFilter / LLMTableExtraction；
- ProxyConfig / Proxy Rotation；
- Deep Crawl Strategy；
- Dispatcher；
- Seeding / Domain Mapper；
- 其它可执行代码、读取 secret、改变网络出口或扩大递归工作的类型。

这是正确的安全模型：未来 Runtime 新增类型时，默认不会自动穿过网络边界。

### 3.2 字段级 Forbidden

典型禁止字段包括：

```text
BrowserConfig:
proxy / proxy_config
extra_args
user_data_dir
channel / chrome_channel
cdp_url / debugging_port / host
cookies / storage_state / headers / init_scripts
browser_context_id / target_id

CrawlerRunConfig:
js_code / js_code_before_wait / c4a_script
deep_crawl_strategy
proxy_config / proxy_rotation_strategy
proxy_session_*
fallback_fetch_function
experimental / base_url
simulate_user / override_navigator / magic
process_in_browser
shared_data / session_id
```

这些字段能够改变网络、代码执行、浏览器身份、状态隔离或资源规模，不能作为普通 Web 表单字段。

### 3.3 数值 Clamp

源码还对不可信输入做数量约束，例如 timeout、scroll steps、viewport。一个重要原则是：第三方的 `0` 如果表示“关闭限制”，平台不能直接继承为“无限”。

平台仍应增加自己的 Capability Firewall：

```text
Web Source Profile
 -> API JSON Schema
 -> Semantic Validation
 -> Capability Allowlist
 -> Numeric Budget Clamp
 -> Server-side Secret Ref
 -> Operator Policy Injection
 -> Config Compiler
 -> Compilation Report
 -> Immutable Runtime Config Release
 -> TRUSTED in-process Crawl4AI object
```

Web 用户编辑业务意图，不编辑 Runtime 原生对象。

---

## 4. `MemoryAdaptiveDispatcher`：只适合作为 Worker Local Protection

`MemoryAdaptiveDispatcher` 的价值在单 Worker：

- 本地 PriorityQueue；
- active task/session permit；
- 内存压力模式；
- 本地 RateLimiter；
- 局部退避和并发保护。

它不能代替全局 Scheduler，因为本地队列 crash 可丢、看不到其它 Worker、不能保证跨 Worker 域名 QPS，也不拥有业务 Lease/Fencing/Coverage。

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

## 5. v0.9.2 流式取消：停止不是 generator 返回

v0.9.2 修复了一个关键生产问题：`MemoryAdaptiveDispatcher.run_urls_stream()` 的消费者提前关闭/取消流时，旧实现可能遗留 per-URL crawl task 和 Browser page/context。

当前 `finally` 的核心行为是：

```text
cancel unfinished active tasks
 -> await all active tasks
 -> drain queued-but-unstarted URLs
 -> cancel memory monitor
 -> await memory monitor
 -> stop monitor
```

这说明平台停止语义必须拆成：

```text
RUNNING
 -> STOP_REQUESTED
 -> QUIESCING
 -> QUIESCED
 -> STOPPED / RETRY / COMMITTED
```

Runtime Quiescence 的最低验证集合：

```text
active_tasks == 0
local_queue == 0
pages == 0
contexts == 0
sessions == 0
held_tokens == 0
```

若 grace timeout 到期，必须终止隔离 Worker/Pod，并以 fencing 拒绝迟到提交，未完成 URL 回 RETRY。

---

## 6. `CrawlResultContainer`：First-result 兼容代理是生产风险

`CrawlResultContainer` 内部把单个结果和列表都标准化为 `_results`，支持同步/异步迭代、索引和长度；但它的 `__getattr__` 会把属性读取委托给第一个结果。

交互式代码里这很方便：

```text
container.markdown
container.url
```

但在 Deep Crawl、多结果、批量 Adapter 中会形成：

```text
[r1, r2, r3]
 -> container.xxx
 -> 只读取 r1
 -> Task 可能成功
 -> r2/r3 静默丢失
```

所以必须有 Result Materialization Contract：

1. Runtime 返回先 flatten/iterate；
2. 每条结果生成 `runtime_result_ordinal`；
3. 结果按 attempt/url identity 关联，不按数组位置；
4. Deep Crawl root 产生 N 个结果，必须产生 N 条 Envelope/Observation；
5. expected/materialized cardinality 不一致必须 fail loudly；
6. 流式乱序提交保持幂等；
7. 业务代码禁止直接访问未物化 Container 的正文类属性。

---

## 7. `prefetch=True`：适合两阶段 Discovery

当前 `aprocess_html()` 在 `prefetch=True` 时调用快速 link extraction，然后直接返回只带 HTML、links、基础状态/redirect 等字段的 `CrawlResult`，不会继续执行重型 scraping、Markdown 和 structured extraction。

这非常适合大规模 Backfill 的 Phase A：

```text
Seed / Frontier Candidate
 -> Browser render when needed
 -> Crawl4AI prefetch=True
 -> Materialize Result
 -> raw links/status/redirect
 -> Persist Observation/Edge
 -> Normalize / Scope / Robots
 -> Platform Graph Depth
 -> Filter / Score / Budget
 -> Persistent Frontier
```

Phase A 不产生：

```text
Canonical Markdown
Document Version
PASS article count
Historical Complete
LLM enrichment
```

Phase B 才执行：

```text
Persistent Frontier
 -> HTTP First / Browser Slow Lane
 -> Raw Artifact
 -> Validity / MIME Route
 -> Extraction Portfolio
 -> Quality
 -> Canonical IR
 -> Markdown
```

`prefetch` 是 Observation 优化，不是 Coverage Truth。

---

## 8. Browser recycle 的源码真相：Context Generation != Browser Process Generation

这是本次最重要的修正。

### 8.1 文档命名容易让人误判

`BrowserConfig.max_pages_before_recycle` 的注释把它描述为“达到若干页面后回收 browser process”，并建议高吞吐场景配置 500~1000 页。若只读参数文档，很容易把它直接映射成平台“浏览器进程代际”。

但源码和测试验证的是另一层语义。

### 8.2 实际实现路径

`BrowserManager` 维护：

```text
self.browser
self.contexts_by_config
self._context_refcounts
self._pages_served
self._browser_version
self._pending_cleanup
```

`_make_config_signature()` 把 `_browser_version` 放入 Context signature。

普通非 managed 路径的 `get_page()` 在 signature 不存在时调用：

```text
self.create_browser_context(crawlerRunConfig)
```

而 `create_browser_context()` 的核心是：

```text
self.browser.new_context(...)
```

页面服务计数达到阈值后，`_maybe_bump_browser_version()` 会：

1. 找出当前 signature 的 active/idle ref；
2. 把 active signature 放入 pending cleanup；
3. `_browser_version += 1`；
4. `_pages_served = 0`；
5. idle Context 立即 close；
6. active Context 等 refcount 归零后由 `_maybe_cleanup_old_browser()` close。

关键点：这条阈值路径能明确看到 **Context signature 变更、旧 Context 排空和关闭**，但看不到创建新的底层 Browser 进程，也看不到旧 `self.browser` 因阈值而 `close()` 后重新 `launch()`。

真正关闭 browser 的代码位于更高层 `BrowserManager.close()` 生命周期中。

### 8.3 测试也验证的是 Context 代际

`tests/async/test_browser_recycle_v2.py` 的顶部说明非常直接：

```text
pages_served reaches threshold
 -> bump _browser_version
 -> old signatures pending cleanup
 -> new requests get new contexts
 -> old context refcount reaches 0
 -> cleanup
```

测试断言的是：

- version 是否 bump；
- signature 是否变化；
- pending cleanup 是否收敛；
- context refcount 是否归零；
- 并发下结果是否成功。

它没有断言：

```text
old browser PID exits
new browser PID differs
browser process RSS baseline resets
Playwright/browser process relaunched
```

因此，生产设计不能把字段名、注释里的“browser recycle”直接等价成进程级回收。

### 8.4 正确的两层生命周期

#### Layer A：Runtime Context Generation

由 Crawl4AI 内部管理：

```text
Context Generation C1
 -> page threshold
 -> _browser_version bump
 -> C1 DRAINING + C2 ACTIVE
 -> C1 refcount 0
 -> close old contexts
```

作用：

- 限制 Context 长期积累；
- 释放 Context 内状态；
- 降低页面/Context 级资源泄漏风险。

但不应承诺底层浏览器进程 RSS/V8 完全归零。

#### Layer B：Platform Browser Process Generation

平台独立拥有：

```text
Process Generation P1 (worker/browser PID set)
 -> pages / age / RSS / error threshold
 -> P1 DRAINING + P2 ACTIVE
 -> stop new admission to P1
 -> drain P1 task/page/context/session refs
 -> P1 QUIESCED
 -> crawler.close()/terminate process or Pod
 -> verify old PID gone
 -> P1 CLOSED
```

若 drain timeout：

```text
P1 TIMED_OUT
 -> kill worker/process/Pod
 -> fencing rejects late commit
 -> unfinished URL RETRY
```

这才真正对 Browser 进程、V8、共享缓存和长期碎片化提供硬回收保证。

### 8.5 平台配置应显式分开

建议 Source/Profile 使用：

```yaml
browser:
  memory_saving_mode: true
  runtime_context_recycle_pages: 750
  process_generation_max_pages: 3000
  process_generation_max_age_seconds: 3600
  process_generation_max_rss_bytes: 2147483648
  process_generation_max_consecutive_errors: 20
  max_draining_process_generations: 2
```

`runtime_context_recycle_pages` 可以经 Compiler 映射到 pinned Crawl4AI 的 `max_pages_before_recycle`，但它的能力状态应被标记为：

```text
VERIFIED_EFFECTIVE_AS_CONTEXT_RECYCLE
```

而不是 `VERIFIED_EFFECTIVE_AS_PROCESS_RECYCLE`。

平台 Process Generation 必须独立验证 PID/process identity。

---

## 9. Context/CDP/身份状态隔离

BrowserManager 支持 Context、session、persistent context、CDP、连接缓存等高状态能力。公共博客默认隔离键建议：

```text
isolation_key = tenant + source_lane + runtime_config_release + process_generation
```

禁止：

- 不相关 Tenant/Source 共享认证 Cookie/localStorage；
- 普通 Web 用户指定 CDP；
- 默认 user-data-dir/persistent profile；
- Browser Context ID/Target ID 暴露给外部；
- 未 QUIESCED 的 Context/Process generation 进入公共复用池。

如未来抓取组织自有私有站点，应单独建立 `AUTHENTICATED_SOURCE` Lane，使用独立 Secret、Worker Pool、egress、profile 与审计。

---

## 10. Fetch Evidence Envelope：Runtime Schema 不能直接成为业务 Schema

`CrawlResult` 可携带 HTML、cleaned HTML、links、media、downloaded files、screenshot、PDF、MHTML、metadata、response headers、redirect、SSL、network、console、dispatch result 等。

平台应抽出稳定 Envelope，而不是把第三方对象直接做数据库 schema：

```text
runtime_result_envelope
- attempt_id
- runtime_result_ordinal
- root_url_id
- requested_url
- result_url
- final_url
- status_code
- redirected_url
- redirected_status_code
- content_type
- runtime_release_id
- runtime_config_release_id
- runtime_result_schema_release_id
- raw_artifact_id
- diagnostic_bundle_id
- error_class
```

Header 必须 allowlist/redaction，Cookie、Authorization、query secret 不进入普通日志。

---

## 11. Diagnostic Evidence：按失败/采样保存

高成本/敏感证据：

```text
screenshot
PDF
MHTML
network requests
console messages
full headers
trace
```

推荐：

```text
normal PASS      -> 关闭或极低采样
REPAIR/error     -> 按 failure policy 捕获
canary           -> 受控采样
manual debug     -> 显式开启
```

每类都需要：size cap、tenant/source quota、TTL、RBAC、redaction、janitor。

用于重放的 Durable Raw Artifact 与 UI 调试的 Ephemeral Diagnostic Artifact 必须分层。

---

## 12. Runtime Cache 与增量同步

Crawl4AI Smart Cache 可利用 ETag、Last-Modified、head fingerprint 降低重复工作，但 Runtime Cache 仍是 Worker optimization。

平台增量链应拥有：

```text
Provider updated/lastmod
+ ETag
+ Last-Modified
+ platform head fingerprint
+ raw payload hash
+ Canonical IR content hash
```

Runtime cache hit 不能：

- 推进 Provider Cursor；
- 改变 Coverage；
- 取代 Raw Artifact；
- 自动判定 Document Version 未变化。

最终 Version 由 Canonical IR/content hash 与 identity policy 决定。

---

## 13. Deep Crawl 的正确边界

Native Deep Crawl 适合作为 URL Gap Provider 和 Browser Prefetch，不应成为历史完整性真相：

- `max_pages` 是执行预算；
- visited/local queue 是 runtime state；
- runtime depth 不替代持久 URL Edge graph depth；
- score/threshold 只应影响 priority；
- Runtime checkpoint 只是执行优化。

正确链：

```text
Deep Crawl Result
 -> Materialize all results
 -> Persist Observation/Edge first
 -> Platform graph depth
 -> Scope/Robots
 -> Filter/Score
 -> Persistent Frontier
```

---

## 14. Markdown 与抽取：Runtime 输出只做 Candidate

长期知识库使用：

```text
Immutable Raw Artifact
 -> Extraction Portfolio
 -> Candidate Quality Gate
 -> Canonical IR
 -> Deterministic Markdown Renderer
```

Crawl4AI runtime Markdown 可作为 Candidate/诊断对照，但不是最终 Canonical Markdown。这样 extractor/renderer 升级都可离线 Replay，无需重新访问站点。

---

## 15. API / Scheduler 双层 Resource Admission

入口层控制：

```text
request bytes
source count
seed URLs
manual run count
active runs per principal
preview count
config size
```

推荐状态语义：

```text
413 request too large
429 principal/tenant quota
503 + Retry-After platform temporarily full
400/422 schema/capability invalid
```

Scheduler 再硬控：

```text
pages admitted
HTTP attempts
Browser attempts
process generations/draining generations
preview/model calls
wall-clock
object/diagnostic bytes
per-domain QPS/concurrency
per-source leases
```

预算按 attempt/call/bytes 统计，不只按成功页面统计。

---

## 16. Opaque Artifact 与路径安全

平台区分：

### Durable Evidence

Raw HTML、原始 PDF、Canonical IR 等写 S3/MinIO，由服务端生成 object key。

### Ephemeral Diagnostic Artifact

Screenshot、调试 PDF、MHTML、network trace、console、Preview：

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

调用方没有服务器路径/object key 写权限。本地临时盘还需要 O_EXCL、O_NOFOLLOW、regular-file check、0600/0700 等保护。

---

## 17. 对最终博客知识库方案的具体优化

### 17.1 保留并强化 Result Materialization Contract

Container/Stream 必须逐条物化，Deep Crawl/批量结果按身份提交，禁止 first-result proxy 语义进入业务层。

### 17.2 保留 Runtime Result Schema Release

Runtime 升级对字段、cardinality、redirect、binary、stream 行为做 Contract Test。

### 17.3 保留 Prefetch Two-Phase Contract

`prefetch=True` 只映射为 `DISCOVERY_OBSERVATION`；Phase B Full Fetch 才能产生 Raw Artifact 和正文版本。

### 17.4 把单层 Browser Generation 拆成两层

新增：

```text
Runtime Context Generation
+ Platform Browser Process Generation
```

Crawl4AI `max_pages_before_recycle` 只作为经验证的 Context recycle 优化；进程级 pages/age/RSS/error 回收由平台 Worker/Pod 生命周期管理。

### 17.5 Process Generation 必须记录进程证据

建议模型加入：

```text
browser_process_generation
- id
- worker_id
- generation_no
- isolation_key
- process_instance_id
- browser_pid_set
- state
- pages_served
- peak_rss_bytes
- active_refs
- drain_reason
- created_at
- draining_at
- quiesced_at
- terminated_at
```

只有旧 PID 已退出，才允许把“进程代际已关闭”作为事实。

### 17.6 继续保留 State Isolation、Fetch Evidence、MIME Route、Diagnostic Budget

这些均与 Crawl4AI 当前实现边界一致，不需要削弱。

---

## 18. 必须新增/修正的 Contract 与 Golden Tests

```text
1. CrawlResultContainer 2+ results -> 全部 materialize
2. 业务 Adapter 禁止 first-result proxy 读取
3. arun_many/stream 乱序 -> identity 关联正确
4. Deep Crawl N results -> N 条 Envelope/Observation
5. prefetch=True -> 不产生 Canonical Document
6. prefetch links -> Observation/Edge 先落库再过滤
7. empty HTML + downloaded file -> MIME route
8. Runtime upgrade -> schema/cardinality drift 阻断
9. max_pages_before_recycle threshold -> _browser_version/signature 变化
10. threshold 后旧 Context refcount 归零并关闭
11. 上述 Runtime context recycle 测试不得宣称 Browser PID 已替换
12. Platform process generation threshold -> 新 Browser PID/进程实例与旧代不同
13. Process generation drain -> 旧 PID 在 refs=0 后退出
14. Process generation timeout -> kill + fencing rejects late commit
15. Process recycle 后 RSS 基线重新建立，并有指标证据
16. context/session state 不跨 tenant/source/process-generation isolation key
17. CDP/persistent profile 普通 Web 配置不可启用
18. diagnostic artifact size/quota/TTL/redaction 生效
19. Runtime cache hit 不推进 Provider Cursor/Coverage
20. Runtime success + Soft-404 -> Validity Gate 仍可拒绝
```

其中 9~15 是本轮最关键的修正：**名字叫 recycle 不等于进程真的 recycle，必须用行为/PID 证据证明。**

---

## 19. 对 1000+ 博客需求的映射

| 需求 | Crawl4AI 可提供 | 平台必须额外拥有 |
|---|---|---|
| 全历史发现 | Deep Crawl、Browser、link prefetch | Provider 并集、Persistent Inventory、CC/Wayback、Coverage/Known Gap |
| 批量执行 | async、stream、dispatcher | Result Materialization、Lease/Fencing、全局公平 |
| Markdown | Markdown/正文候选 | Raw Artifact、Quality、Canonical IR、稳定 Renderer |
| 1000+ 站扩展 | 通用 Runtime | Source Profile、Config Compiler、全局 Scheduler、域名限流 |
| 增量同步 | Runtime cache freshness 可辅助 | Provider Cursor、Conditional GET、Version/Tombstone |
| JS-heavy | Browser/Playwright | Browser Slow Lane、Process Generation、Quiescence |
| Context 内存/状态 | `_browser_version`、Context recycle、LRU | Context 只是 Runtime state，不当进程回收保证 |
| Browser 进程长期内存 | memory saving 可辅助 | **平台 Process/Pod Generation、PID/RSS/age/page/error 阈值** |
| Web 管理 | Docker/API 设计可参考 | RBAC、声明式配置、审核、Release、Coverage/Frontier UI |
| 停止/恢复 | dispatcher cleanup | Persistent Frontier、Lease/Fencing、Quiescence Barrier |
| 安全 | UNTRUSTED config 思路 | SSRF/Scope/Robots/Network Policy/Secret Manager |
| 可追溯 | CrawlResult 运行证据 | Stable Envelope、Raw Artifact、Release/Trace |
| 低成本 | prefetch、async、memory adaptive | HTTP First、两阶段 Discovery、诊断预算、跨站公平 |

---

## 20. 最终采用建议

采用 Crawl4AI，但固定为 **可替换、可版本化、受平台 Capability Boundary 约束的 Runtime Adapter**。

平台负责：

```text
为什么抓
允许抓什么
何时抓
抓取预算
历史是否完整
URL/Document 身份
如何恢复
如何证明停止后静默
如何保存不可变证据
如何形成 Canonical Markdown
如何真正回收 Browser 进程/Pod
```

Crawl4AI 负责：

```text
高效执行 HTTP/Browser/Deep Crawl
快速 prefetch links
局部调度/内存自保护
Runtime Context recycle
返回运行时抓取证据
```

最终需要组合的关键能力为：

**Result Materialization Contract、Prefetch Two-Phase Contract、Runtime Context Generation、Platform Browser Process Generation、Runtime Quiescence Barrier、Fetch Evidence Envelope、Diagnostic Evidence Budget、Capability Firewall、Opaque Artifact Gateway。**

核心修正可以概括为一句话：

> **第三方 Runtime 的“browser recycle”名称只能作为提示，生产平台必须通过源码行为和 PID/进程证据确认其真实层级；Crawl4AI 当前可验证的是 Context Generation 回收，Browser 进程的硬生命周期与 RSS 回收必须由平台独立拥有。**