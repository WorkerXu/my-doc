# searxNcrawl：SearXNG 搜索与 Crawl4AI 抓取集成项目

## 1. 项目资料

- 编号：32
- 项目：searxNcrawl
- 地址：https://github.com/DasDigitaleMomentum/searxNcrawl
- 定位：基于 Crawl4AI + Playwright 的网页抓取工具，同时提供 SearXNG 搜索、Python API、CLI 和 MCP Server。
- 本次调研目标：分析其中可用于“1000 个技术博客全量历史抓取 + Markdown 知识库 + 增量同步 + Web 管理”的实现模式，并判断现有《博客知识库技术方案》是否需要增强。

## 2. 总体判断

searxNcrawl 不是一个可直接承担 1000 站长期知识库同步的完整平台。它更像一个“可调用的抓取能力层”：输入一个或多个 URL，或从 seed 做有限深度站点遍历，再把 Crawl4AI 的结果标准化成 Markdown/HTML/Headers/References/Metadata，并通过 Python、CLI、MCP 暴露出去。

它最值得吸收的不是“深度爬虫本身”，而是五个工程化细节：

1. 把 Crawl4AI 的返回形态包在自己的稳定 Document Contract 后面；
2. 对站点遍历显式定义 host/domain scope，避免无边界扩散；
3. 对 Browser crawl 设置调用级 deadline，而不是只依赖底层 page timeout；
4. 对 Markdown 重复块做可观测的去重统计和 guardrail，而不是静默删内容；
5. 把抓取配置集中构造，并让调用侧通过 override 修改最终实际传入 Crawl4AI 的 runtime config。

这些能力适合进入我们的“Engine Adapter / Fetch Route / PostProcess / Effective Config”层，但不能替代 Durable Frontier、Coverage Evidence、Provider Checkpoint、Document Version、Change-Signal Plane 和 Web Control Plane。

## 3. 代码结构与实现路径

### 3.1 对外接口层

仓库同时提供：

- Python API：`crawl_page_async`、`crawl_pages_async`、`crawl_site_async`；
- CLI：单页、多页、site crawl、search；
- MCP Server：把 crawl/search 能力提供给智能体或远程客户端；
- Docker/HTTP transport：作为独立抓取服务运行。

这说明它把“抓取引擎”与“调用入口”分离。对我们的方案而言，抓取 Worker 也应暴露稳定内部协议，Web、调试 CLI、自动化工具、测试工具都通过相同 Control Plane/Task Contract 发起工作，而不是各自直接操作 Crawl4AI。

### 3.2 单页抓取

核心流程可以概括为：

```text
URL
 -> build_markdown_run_config()
 -> AsyncWebCrawler.arun()
 -> asyncio.wait_for(deadline)
 -> normalize Crawl4AI result shape
 -> build_document_from_result()
 -> CrawledDocument
```

`crawl_page_async()` 外层使用 `asyncio.wait_for()` 给整个 `crawler.arun()` 加 deadline。这个边界很重要，因为 Browser 导航、JS wait condition、持续网络请求等问题可能让底层动作长时间不返回。只配置 Playwright/Crawl4AI 的 page timeout 不足以形成完整调用级保护。

项目还记录了这类问题的历史可靠性复盘：早期版本缺少上层 timeout 时，一个挂住的 URL 会让 MCP 调用和批量 gather 一起卡死。后续源码已经加入 wait-for timeout 和 elapsed-time logging。这对生产抓取系统的启示是必须存在多层 deadline，而不是单一 timeout 参数。

建议我们的 Worker 使用：

```text
connect_timeout
read_timeout
page_navigation_timeout
selector_wait_timeout
per_url_deadline
stage_deadline
run_deadline
lease_deadline
```

其中上层 deadline 到期时必须传播 cancellation，并由 Finalizer 把任务写成结构化 Outcome，而不是留下悬挂 coroutine。

### 3.3 多 URL 并发

`crawl_pages_async()` 使用 `asyncio.Semaphore(concurrency)` 控制进程内并发，然后通过 `asyncio.gather()` 汇总结果；单个失败被转为 `CrawledDocument(status="failed")`，避免一个普通异常直接中断所有 URL。

这个模式适合小批量 CLI/MCP 请求，但不适合作为 1000 站生产调度器：

- semaphore 是进程内状态，进程退出即丢失；
- gather 没有 durable lease/checkpoint；
- 无跨 Worker 公平调度；
- 无站点级 token bucket；
- 无任务重试状态机和 dead letter；
- 无 Transactional Outbox。

因此生产系统仍应坚持 PostgreSQL durable task + Queue transport + Worker lease，semaphore 只作为单 Worker 内的最后一道本地资源保护。

### 3.4 Site Crawl 与范围控制

`crawler/site.py` 的核心做法是：

1. 解析 seed URL host；
2. 用 `tldextract` 得到 registrable domain；
3. 根据 `include_subdomains` 生成 allowed host/domain；
4. 用 Crawl4AI `DomainFilter + FilterChain` 限制深度遍历范围；
5. 配置 `max_depth/max_pages`；
6. 聚合每页为 `SiteCrawlResult`。

当前源码实际使用 `DFSDeepCrawlStrategy`。仓库部分文档仍写 BFS，说明文档与实际代码可能出现策略漂移。这一点非常关键：生产系统不能把 README/配置文本当作“本次运行实际执行策略”的证据，必须记录 resolved runtime class、依赖版本和 effective config digest。

另外，项目按 `request_url` 做本轮结果去重。它自己也指出，不同 query 参数但语义相同的 URL 会被视为不同地址。因此我们不能复用这种简单 URL 去重作为 Document Identity，只能把它用在一次 Engine 调用内部的重复结果抑制。

### 3.5 Crawl4AI 返回结果兼容层

项目专门写了 `_extract_first_result()` / `_iterate_results()`，兼容：

- 单个 `CrawlResult`；
- `CrawlResultContainer`；
- list；
- async generator；
- list 中嵌套 container。

这是一个很实用的边界设计：第三方库返回类型变化不能泄漏到整个业务系统。

我们的方案应把它提升为正式的 `CrawlerEngineAdapter`：

```text
Crawl4AI raw return
 -> Engine Result Normalizer
 -> FetchArtifact / CrawlPageOutcome
 -> 后续 Extraction/Quality
```

Adapter 负责兼容依赖版本差异，业务层只接受内部 Representation Contract。

### 3.6 Markdown 生成配置

`build_markdown_run_config()` 集中构造 Crawl4AI 配置，主要包括：

- `target_elements`：`main`、`article`、`.markdown-body`、`.docs-content`、`.prose` 等正文候选；
- `excluded_tags/excluded_selector`：nav、footer、header、aside、sidebar、cookie banner 等；
- `PruningContentFilter`；
- `DefaultMarkdownGenerator`；
- `scan_full_page=True`；
- Browser JS reload + scroll；
- `wait_for` 正文条件；
- CacheMode；
- delay / semaphore 等运行参数。

`RunConfigOverrides` 再把用户或调用方 override 真正写进 `CrawlerRunConfig`。这验证了我们现有方案中“配置必须可证明实际被 Worker 消费”的必要性。

不过这些 selector 只能作为 Generic Profile baseline。1000 个异构博客不能假设都存在 `main`，也不能统一 reload/scroll。我们的方案仍应由 Site/Profile evidence 决定 selector 与 Browser Route。

### 3.7 Document Contract

searxNcrawl 将 Crawl4AI 结果转为 `CrawledDocument`，字段包括：

```text
request_url
final_url
status
markdown
raw_markdown
html
headers
references
metadata
error_message
```

Builder 会：

1. 从 metadata 提取 requested URL；
2. 保留 resolved/final URL；
3. 失败时构造 failure reason；
4. 优先使用 fit markdown，其次 citation markdown，再回退 raw markdown；
5. 缺少 Markdown 时可从 HTML 再生成；
6. 提取 reference links；
7. 附加 dedup stats。

这套 Contract 比直接返回一段 Markdown 好得多，因为它保留了 URL 重定向、原始/清洗内容和失败原因。但对我们的知识库仍不够：我们还必须保存 immutable Snapshot、Canonical IR、field provenance、document_version、quality result、identity evidence 和 release lineage。

## 4. Markdown 精确重复块去重机制

### 4.1 实现原理

`markdown_dedup.py` 的 `exact` 模式大体流程是：

```text
normalize line ending
 -> 按空行拆 section
 -> 再按 Markdown heading boundary 拆分
 -> 每个 section 去除行尾空白
 -> SHA-256 fingerprint
 -> first occurrence wins
 -> 重组 Markdown
```

同时输出：

```text
dedup_sections_total
dedup_sections_removed
dedup_chars_removed
dedup_applied
```

Builder 还设置 guardrail：当 section 数量达到最低值、移除块数达到最低值、且删除比例超过约 45% 时，不静默认为“清洗成功”，而是写 `dedup_guardrail_triggered=true` 并记录 removal rate。

这个思路非常值得吸收：任何有损清洗都必须带统计和风险信号。

### 4.2 不能直接复制的原因

项目是在 Markdown 字符串上按空行/heading 拆块。这对普通文章可工作，但对我们的长期知识库有风险：

- fenced code 内可能有空行；
- 列表、表格、引用块可能被拆出错误边界；
- 合法重复代码示例可能应该保留；
- 两个视觉相同 section 不一定都是 boilerplate；
- 在 Canonical IR 之前删文本会让后续无法恢复。

因此应吸收“fingerprint + metrics + guardrail”而不是直接复制“Markdown split + 删除”。

### 4.3 适配到我们的方案

建议新增 `IntraDocumentRepetition Evidence`：

```text
Extraction Candidate
 -> Canonical IR blocks
 -> block fingerprint
 -> repetition evidence
 -> policy decision
 -> optional non-destructive suppression in Projection
```

每个 block 计算：

```text
block_type
normalized_exact_hash
occurrence_index
first_occurrence_block_id
repeat_count
is_boilerplate_candidate
```

默认规则：

- code/table/math 不自动删除；
- heading + paragraph 组合只有在完全相同且上下文满足 boilerplate 规则时才允许 Projection 级抑制；
- Canonical IR 保留原块或保留 suppression evidence，确保可重建；
- 如果预计删除比例过高，触发 `REPETITION_GUARDRAIL`，进入 warning/quarantine/shadow review；
- Web 文档详情展示 before/after block diff 和 removal rate。

这样既能处理动态网页重复注入、导航模板重复，又不会因为字符串层清洗破坏代码和结构。

## 5. 搜索能力的适用边界

searxNcrawl 集成自托管 SearXNG，并允许 language、time range、category、engine、safe search 等查询参数。对于我们的系统，SearXNG 可作为：

- 某站历史断档时的 gap discovery；
- 新增站点时寻找 RSS/Sitemap/Archive 的辅助线索；
- Reconcile 时发现 canonical host 之外仍被搜索引擎收录的旧文章；
- 人工排查“站点明明有文章但 Provider 没发现”的诊断工具。

但搜索索引绝不能成为历史完整性真相源，因为：

- 索引本身不完整；
- 旧文章可能退索引；
- 搜索排序/数量不可稳定枚举；
- 结果受时间、地区、引擎和策略影响。

所以应新增 `SEARCH_GAP_DISCOVERY` Provider，但产生的只是一类 `DiscoveryEvidence(source=search)`，不能提升为 Coverage Complete。

## 6. 可靠性设计的启示

### 6.1 Timeout Hierarchy

项目早期可靠性复盘证明“只要一个页面挂住就可能拖住整个上游调用”。当前代码已经用 `asyncio.wait_for()` 修复核心路径。

我们的任务系统应进一步明确 timeout hierarchy：

```text
HTTP connect/read/write timeout
Browser navigation timeout
JS/selector wait timeout
per URL deadline
per Stage deadline
per Task lease deadline
per Run deadline
```

错误必须区分：

```text
CONNECT_TIMEOUT
READ_TIMEOUT
NAVIGATION_TIMEOUT
CONTENT_WAIT_TIMEOUT
URL_DEADLINE_EXCEEDED
STAGE_DEADLINE_EXCEEDED
RUN_DEADLINE_EXCEEDED
```

不同错误决定不同 retry/route-upgrade 策略。

### 6.2 Engine Quirk Registry

源码中明确写了 Crawl4AI deep crawl streaming 的兼容处理，并强制某些模式 `stream=False`。再加上仓库文档对 BFS/DFS 的描述出现漂移，说明第三方 crawler 的行为不能完全由类型签名推断。

建议增加：

```text
crawler_engine_release
engine_version
runtime_strategy_class
stream_mode
known_quirk_ids
capability_test_result
resolved_config_hash
```

每次核心依赖升级必须做 capability smoke test，例如：

- single result shape；
- multi result shape；
- deep crawl stream/non-stream；
- cancellation；
- timeout；
- redirect；
- Browser crash；
- Markdown/cleaned_html 字段存在性。

然后将测试结果绑定到 Engine Release。业务代码只使用已通过门禁的 release。

## 7. Crawl Scope Contract

这是本次调研对现有方案最值得新增的一项。

单纯写 `include_subdomains=true/false` 不足以表达生产爬虫边界。建议正式定义：

```text
EXACT_HOST
REGISTRABLE_DOMAIN
HOST_ALLOWLIST
URL_PREFIX_ALLOWLIST
CUSTOM_SCOPE_RULE
```

Scope 在 Site Profile 发布时被解析为 `Resolved Crawl Scope`：

```text
seed_host
registrable_domain
allowed_hosts
allowed_url_prefixes
excluded_hosts
follow_redirect_scope
asset_scope
```

注意：

- `www.example.com`、`docs.example.com`、`blog.example.com` 可能是同一内容域，也可能是完全不同系统；
- redirect 到 CDN/Medium/Substack 时不应默认把新域加入 frontier；
- asset host 与 document crawl host 必须分离；
- 国际化域名需统一 IDNA/host normalization；
- port/scheme 不能用字符串拼接随意判断；
- Scope 越界 URL 保留为 link evidence，但默认不 admission 为文章抓取任务。

Web 管理端应显示“本次运行实际 crawl scope”，避免操作员只看到一个含糊的 include_subdomains 开关。

## 8. 与当前技术方案的差距对比

### 当前方案已经覆盖、无需推翻

- PostgreSQL durable state + Redis Streams + Transactional Outbox；
- HTTP-first / Browser evidence-driven fallback；
- Provider Coverage Evidence；
- Sitemap/Feed/API/Archive 多发现源；
- Canonical IR；
- append-only Document Version；
- Change-Signal Plane；
- Effective Config Audit；
- Golden Corpus / Contract Test；
- Web 管理；
- Search/RAG/AI Projection 隔离；
- 结构保真清洗；
- per-site rate limit、公平调度、backpressure。

这些部分比 searxNcrawl 更适合 1000 站长期运行，应保持。

### 建议新增/强化

1. **Crawler Engine Result Normalizer**：第三方 Crawl4AI 返回任何 shape，都先转内部 Artifact/Outcome Contract。
2. **Resolved Crawl Scope Contract**：把 host/domain/subdomain 边界变成可审计的正式配置。
3. **Intra-document Repetition Evidence + Guardrail**：吸收 exact fingerprint 与 aggressive-removal warning，但在 Canonical IR block 层实现。
4. **Crawler Engine Capability/Quirk Registry**：绑定实际依赖版本、strategy class、stream mode、兼容测试结果。
5. **分层 Deadline + Cancellation Propagation**：对 Browser/URL/Stage/Run 明确不同 timeout 和错误语义。
6. **SearXNG Search Gap Provider**：只用于补充发现和诊断，不作为 Coverage 真相。
7. **Runtime Strategy Attestation**：Run 的 Effective Config 不只保存 YAML/JSON，还保存实际构造出的 crawler strategy/runtime class 与关键字段。

## 9. 建议的数据模型扩展

```text
crawler_engine_release
engine_capability_test
engine_quirk
resolved_crawl_scope
repetition_evidence
search_gap_discovery_evidence
```

关键字段示例：

```text
crawler_engine_release:
  engine_name
  engine_version
  package_lock_digest
  adapter_release_id
  status

resolved_crawl_scope:
  site_id
  profile_release_id
  scope_mode
  seed_host
  registrable_domain
  allowed_hosts_json
  allowed_prefixes_json
  excluded_hosts_json
  effective_hash

repetition_evidence:
  document_version_id / extraction_candidate_id
  block_id
  exact_hash
  first_occurrence_block_id
  repeat_count
  policy_release_id
  action
```

## 10. Web 管理端建议增加

Site Detail / Run Detail 增加：

- Resolved Crawl Scope；
- Crawler Engine Version；
- Runtime Strategy Class；
- Known Quirk / capability test 状态；
- per-URL deadline 和 timeout outcome；
- repetition removal rate / guardrail；
- Search Gap Discovery 证据。

Profile Playground 增加一个“重复块分析”视图：同一 Snapshot 运行旧/新 PostProcessor，显示重复 block、被抑制 block、删除比例和结构 diff。

## 11. 最终结论

searxNcrawl 可以作为我们设计中的“抓取引擎适配与调试服务”参考，但不应作为整体架构。它证明了几个容易被忽略的生产问题：第三方 crawler 返回结构会变化、deep crawl 有运行时 quirks、文档描述可能和实际 strategy 漂移、Browser await 必须有外层 deadline、站点范围必须明确、内容去重必须有 guardrail。

因此本次技术方案优化方向不是把系统改成 SearXNG + Crawl4AI 单体，而是继续保持 Durable Control/Data Plane，同时补强：**Engine Result Normalizer、Resolved Crawl Scope、IR 层重复证据、Engine Capability/Quirk Registry、分层 Deadline、Search Gap Provider 和 Runtime Strategy Attestation**。
