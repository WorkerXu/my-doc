# AI 搜索博客优化器：实现细节与技术原理分析

## 1. 项目定位

- 项目：AI Search Blog Optimiser
- 地址：https://github.com/mlobo2012/AI-search-blog-optimiser
- 类型：Claude Code / Claude Cowork 插件
- 主要语言：Python
- 核心用途：给定企业博客入口，通过 Crawl4AI 或 Firecrawl 抓取已有文章，再结合站点级品牌语气、Peec AI 搜索数据、竞争对手页面与证据数据，生成 GEO 优化建议和文章重写包。

它不是为“1000 个网站全量历史归档 + 长期增量同步”设计的通用爬虫平台。其默认工作量是有限文章批次，重点是内容优化流水线，而不是全站覆盖、持续同步和多租户调度。

但这个项目对博客知识库方案仍然非常有参考价值，尤其是以下工程问题：

1. Run 的所有权与运行上下文纪律；
2. 精确 URL 任务的确定性语义；
3. 抓取后端固定 fallback；
4. 结构化 Article Record；
5. Typed Writer 将业务产物与状态绑定提交；
6. Finalizer 根据真实产物纠正 Worker 自报状态；
7. Dashboard 只做报告和控制入口；
8. 服务能力按 capability 发现，而不是硬编码实例名；
9. 确定性质量验证器生成 Manifest，而不是依赖生成 Agent 自评分；
10. 从真实缺陷中可以看到“运行上下文漂移、Artifact Contract 漏配、状态与产物分裂”会如何发生。

因此正确的借鉴方式不是直接采用这个项目作为爬虫底座，而是吸收其运行协议和一致性思想，再升级为 PostgreSQL + S3/MinIO + 分布式 Worker 的长期平台架构。

## 2. 仓库结构与职责边界

仓库主要结构如下：

```text
.claude-plugin/   插件与 marketplace 描述
commands/         Slash Command，主编排入口
skills/           canonical pipeline playbook
agents/           crawl / voice / gap / evidence / recommend / generate 等叶子 Agent
config/           配置
references/       GEO 规则与契约
specs/            设计、缺陷与修复说明
dashboard/        MCP Server、Web Dashboard、确定性质量校验
tests/            回归与端到端测试
dist/             分发产物
```

项目的关键边界是：**主会话是唯一 orchestration owner，子 Agent 只做叶子工作。**

`commands/blog-optimiser.md` 明确要求流水线直接在主会话编排，不能再创建一个总编排 Agent。Crawler 也被明确禁止自行 `register_run`。

这个约束背后的原理是“全局状态机必须有单一拥有者”。如果任意 Worker 都能创建 Run、切换 Run ID、改变全局阶段，就会出现：

```text
Orchestrator 认为自己在 run A
Crawler 自己创建 run B
Artifact 写入 B
后续阶段继续读取 A
最终 Web 状态与真实产物完全分裂
```

这不是理论风险。项目自己的 Bug Spec 就记录了这一事故：Crawler 曾在 Orchestrator 已创建 Run 后又创建新 Run，导致文章落到错误目录，原始 Run 查不到任何文章，同时 Peec 项目关联丢失。

对知识库平台的直接启发是：

- `crawl_run` 只能由 Control Plane / Scheduler 创建；
- Worker 只能领取属于既定 `crawl_run_id` 的 work unit；
- Worker 无权创建、替换、切换 run；
- 所有领域写入必须同时校验 `crawl_run_id + task_id + lease_version`；
- 如果运行上下文缺失，Worker 应 fail-fast，而不是自行创建 fallback run。

## 3. Run ID 纪律还不够：需要不可变 Run Context

项目通过 `register_run` 返回：

```text
run_id
state.json
articles_dir
media_dir
raw_dir
...
```

Crawler 必须对所有 Dashboard MCP 调用使用同一个 `run_id`。这解决了“写到错误 Run”的问题，但对大规模平台还不够。

因为长期平台中配置可能在运行期间变化，例如：

- 管理员发布新的 Extraction Rule；
- Fetch Profile 改了 Browser fallback；
- URL Filter 更新；
- Quality Contract 更新；
- Crawl4AI Adapter 升级；
- Markdown Renderer 发布新版本；
- Credential Profile 切换 Secret Version。

如果 Worker 每次都查询“当前最新配置”，同一个 `crawl_run_id` 内就可能混合多个规则版本：上午抓取使用 release A，下午重试使用 release B。此时即使 Run ID 没变，结果也不可复现。

因此应进一步引入 **Run Context Manifest**，在 Run 创建时一次性冻结：

```text
run_context_id
crawl_run_id
site_profile_release_id
site_source_release_ids
provider_release_ids
url_filter_release_id
url_scorer_release_id
fetch_profile_release_id
cache_policy_release_id
browser_action_plan_release_id
adapter_binding_release_id
adapter_schema_release_id
extraction_pipeline_release_id
quality_contract_release_id
markdown_renderer_release_id
runtime_bundle_release_id
policy_release_id
credential_profile_id
credential_secret_version_refs
context_hash
created_at
```

Worker payload 只引用 `run_context_id`，不能在执行中隐式读取 latest release。

这样才能回答：“为什么这篇文章在这次 Run 中得到这个结果？”并能够用完全相同的配置做 replay。

## 4. URL 发现机制

### 4.1 精确 URL 模式

项目支持重复传入 `--article-url`。Crawler 的规则是：

1. 对 URL 做规范化；
2. 只去除精确重复；
3. 保留用户输入顺序；
4. 完全跳过博客索引发现；
5. 不允许用最近文章、相关文章、Sitemap URL 或猜测 URL 补位；
6. 任意指定 URL 没有真实持久化，都必须明确报告失败。

这个语义非常适合知识库平台，应保留为：

```text
TARGETED_FETCH
TARGETED_REPROCESS
```

用途包括：

- 单篇漏抓修复；
- 单 URL 抓取诊断；
- Selector 调试；
- Browser Action 验证；
- Extraction Replay；
- 事故修复；
- Golden Case 回归。

这里最重要的不是“支持单 URL”，而是**精确目标不可被静默替代**。如果目标失败，系统必须失败，而不是返回一个“看起来类似”的文章冒充成功。

### 4.2 索引发现模式

Firecrawl 路径优先 `map`，必要时再 scrape 博客入口；Crawl4AI 路径先读取 raw Markdown，内容太薄或链接不足时再拿 HTML。

只接受真实出现在：

```text
map result
markdown link
HTML href
canonical
```

中的同源 URL，并过滤 category、tag、author、pagination、feed 等非文章路径。

这保证确定性，但它只适合有限博客索引抓取，不能代表完整历史覆盖。项目本身还有默认文章数上限，README 也明确说明这是 run-size cap。

对于 1000 站全量历史平台，必须扩展成多 Discovery Provider：

```text
Sitemap / Sitemap Index
平台 API
RSS / Atom / JSON Feed
年/月 Archive
Category / Tag / Author Pagination
HTML 同域递归
Browser 动态列表
外部公开索引补漏
```

并为每个 Provider 保存 checkpoint、continuity、coverage evidence 和完成条件。

## 5. URL 身份：真实证据优先于标题和 slug

Crawler 明确规定：

- 不允许根据标题推导 slug；
- 不允许根据文章名拼 URL；
- URL 必须真实出现在链接或 canonical 中；
- exact target 失败时不能找相似内容替代。

这是一个重要的 identity 原则。

标题、slug、URL path 尾段都会变化，也可能重复。大型知识库内部必须使用稳定 ID：

```text
url_id
article_id
article_version_id
attempt_id
snapshot_id
artifact_object_id
```

`normalized_url_hash` 可以作为 URL identity 唯一键的一部分，但不能让标题承担主键职责。

每条新 URL 同时保存 `discovery_evidence`，使 Web 能回答：

```text
谁发现了这个 URL？
从哪个 Sitemap / Archive / Feed / HTML 页面发现？
原始 entry/href 在哪里？
什么时候发现？
为什么被 Admission 接受或拒绝？
```

## 6. 固定抓取后端与 fallback

项目的抓取策略不是让 Agent 自由试错，而是固定顺序。

Firecrawl：

```text
scrape(markdown + html)
```

Crawl4AI：

```text
md(raw)
  -> empty / thin / anti-bot / script-heavy shell
  -> html
```

Crawler 不允许为了填 metadata 随意升级到 `execute_js`，也不允许用 LLM 总结器“猜”结构字段。

原理是：**fallback 是产品契约，不是 Agent 临场判断。**

知识库系统应把 fallback 固化到版本化 `fetch_profile_release`：

```text
HTTP_FETCH
 -> HTTP_STRUCTURED_EXTRACT
 -> BROWSER_RENDER
 -> SITE_API_ADAPTER
 -> MANUAL_DIAGNOSIS
```

每次升级必须记录结构化原因：

```text
EMPTY_ARTICLE_BODY
SPA_SHELL
JS_PAGINATION
LOAD_MORE
INFINITE_SCROLL
DYNAMIC_RENDER_REQUIRED
EXTRACTION_READINESS_FAILED
```

并把 Adapter、Browser Revision、Runtime Bundle 一起记录，保证结果可解释、可复现。

## 7. Article Record：Markdown 不能是唯一事实

项目会把抓取结果先整理成结构化 Article Record，大致包括：

```text
url / fetched_at / crawl_backend
meta:
  title / description / canonical / OG / Twitter / robots / hreflang
schema:
  JSON-LD 类型 / raw ldjson
structure:
  H1 / heading tree / word count / table / list / quote / FAQ / code
media:
  images / videos / iframes / thumbnail
trust:
  author / role / published_at / updated_at / credentials / entities
links:
  internal / external / inbound_internal
cta
body_md
raw_html_path
```

这实际上已经是一个轻量 Article IR。

对知识库的正确数据链应是：

```text
Raw Network Snapshot
 -> Cleaned/Rendered DOM
 -> Extraction Candidates
 -> Article IR
 -> Quality Selection
 -> Markdown Render
 -> Article Version
 -> Index / Publish
```

Markdown 是最终稳定输出格式之一，但不是唯一中间事实。这样规则升级时可以基于 snapshot / IR replay，而无需重新访问源站。

## 8. Typed Writer：产物与状态必须一起提交

Crawler 对成功文章不建议：

```text
write article.json
update state.json
```

而是调用 `record_crawled_article`，其业务语义是“提交文章产物，同时提交对应 crawl-stage 状态”。后续阶段也存在类似 typed tools：

```text
record_voice_baseline
record_peec_gap
record_competitor_snapshot
record_evidence_pack
record_recommendations
record_draft_package
fail_article_stage
```

这种做法优于万能 `update_state`，因为服务端可以统一完成：

- schema 校验；
- 幂等检查；
- 合法状态迁移；
- Artifact 引用校验；
- 状态更新；
- Event；
- Outbox；
- 事务提交。

迁移到知识库平台后，应使用：

```text
record_discovered_url(...)
commit_fetch_snapshot(...)
commit_extraction_candidate(...)
commit_article_version(...)
commit_quality_evaluation(...)
commit_index_result(...)
commit_publish_result(...)
```

`background_task` 只保存运行生命周期和领域对象引用，不能变成万能业务表。

## 9. finalize_crawl：Worker 自报成功不是完成证据

项目在所有文章抓取结束后调用 `finalize_crawl`，并用真实 `articles/*.json` 集合重新计算阶段结果。

关键规则包括：

- 真实持久化文章数为 0：失败；
- 发现数 > 真实持久化数：partial；
- exact URL 模式下任何目标缺失：失败；
- 下游阶段只处理 Finalizer 返回的真实 persisted set。

它解决的是典型状态漂移：

```text
Worker 日志：成功
状态文件：completed
真实文件：缺 7 篇
Dashboard：绿色
```

大型平台需要把本地目录扫描升级成 Artifact Manifest Reconciliation：

```text
数据库期望集合
vs COMPLETE artifact_object
vs run_artifact_manifest
vs 领域表引用
vs retry/dead-letter 集合
```

Stage Finalizer 输出：

```text
COMPLETED
PARTIAL
FAILED
INCONSISTENT
```

Finalizer 必须是幂等、可重复执行的服务器端操作。Worker 崩溃后重新调用 Finalizer，结果应一致。

## 10. 确定性 Quality Gate：不能相信生成器自评分

这个项目最值得进一步吸收的设计来自 `dashboard/quality_gate.py`。

质量验证器不是让生成 Agent 说“我觉得通过”，而是服务器端重新读取真实产物：

```text
source article JSON
recommendations JSON
evidence JSON
rendered HTML
schema JSON
existing manifest
site reviewer data
```

然后做确定性验证，包括：

- author/reviewer 是否真实可见；
- role/credential 是否符合要求；
- scope drift；
- 内外部 evidence 数量；
- inline evidence 数量；
- 必须引用的 claim 是否出现；
- 内部链接数量；
- Schema 文件是否具备要求类型；
- Rendered HTML 是否真正嵌入同样 Schema；
- FAQ 可见问题与 FAQ Schema 是否一致；
- H2 是否满足问句结构要求；
- 段落长度和 atomic paragraph 比例；
- TL;DR、trust block、semantic HTML、完整 section 等模块；
- 推荐项是否真实实现；
- 可见 meta-language 是否泄漏；
- visible entity 是否有来源支撑；
- off-page 建议是否被错误写入正文。

最终输出 `manifest.json`，包含：

```text
quality_gate.status
blocking_issues
missing_required_modules
module_checks
score_breakdown
author_validation
scope_drift
source_grounding
schema_checks
evidence counts
reader artifact
```

程序退出码也由 Manifest 的真实校验结果决定。

### 对知识库平台的迁移

我们的知识库虽然不是 GEO 重写系统，但同样需要**服务端确定性质量 Manifest**。建议增加：

```text
quality_contract_release
quality_manifest
validator_release
```

对最终 Markdown / Article Version 检查：

```text
源 URL / canonical / provenance 完整
front matter schema 合法
正文非空且无截断
heading 结构合法
Article IR -> Markdown code/table/list/image/link 保留率
图片与附件引用可解析
内部链接可解析
canonical/final URL 冲突
boilerplate 比率
内容 hash 与 renderer release
COMPLETE artifact 引用
site-specific required checks
```

输出状态：

```text
PASSED
BLOCKED
REVIEW_REQUIRED
```

只有 PASSED 的候选版本才能自动成为 `article.current_version_id` 并进入 Index/Publish。质量失败的新版本保留审计，但不能覆盖上一版高质量内容。

人工 override 也不能把 validator 的结果篡改成 passed，而是单独记录：

```text
override_actor
override_reason
override_at
override_scope
```

这样机器事实与人工决策始终分离。

## 11. 项目真实 Bug 对架构的进一步启发

### 11.1 Bug：Crawler 创建重复 Run

真实事故说明“写文档说不要创建 Run”还不够。平台必须在服务器端强制：

- Worker token / role 没有 create run 权限；
- Domain Writer 校验 Run ownership；
- `crawl_run_id + task_id + lease_version` 是提交条件；
- 不存在“找不到 Run 就自动创建”的 fallback。

### 11.2 Bug：新增 evidence 阶段但 Artifact Namespace 漏配

项目曾新增 evidence stage，但 `write_json_artifact`、`read_json_artifact`、`list_artifacts` 的 namespace enum 没同时加入 `evidence`，导致 playbook 和真实 API contract 不一致。

根因是 Artifact Kind 定义散落在多个工具/枚举中。

大型平台应建立单一 **Artifact Kind Registry**：

```text
artifact_kind
schema_version
namespace
allowed_media_types
producer_stage
allowed_consumers
retention_policy
required_for_finalize
validator
```

所有 write/read/list/cleanup/UI/Finalizer 都从同一 Registry 或生成代码读取，避免每新增一个 Artifact 都手工修改 5 个 enum。

更进一步，Worker 不应自由拼对象 key，而由服务端：

```text
reserve_artifact(attempt_id, kind)
 -> staging key

commit_artifact(reservation_id, sha256, size, schema_version)
 -> COMPLETE artifact_object + manifest entry
```

这样 Artifact 的 namespace、schema 和生命周期全部由平台控制。

### 11.3 Bug：缺少 validate_article，生成器自评分

项目曾出现 playbook 要求 `validate_article`，但 Dashboard 实际没有这个工具，Generator 只能自行 re-audit 并直接推状态。

这正说明：**质量状态不能由生产同一产物的 Worker 自己证明。**

因此知识库 Quality Validator 必须是独立、确定性、可 replay 的服务器端组件，且：

- 读取真实已提交 Artifact；
- 不能使用 Worker 的 self-report 作为通过依据；
- Manifest 是质量事实；
- `finalize_quality(run_id)` 根据 Manifest 对账；
- Index/Publish 只消费 Finalizer 返回的 accepted version set。

## 12. Artifact Contract 应成为一等公民

结合 Typed Writer、Manifest 和 Bug 3，可以抽象出更完整的 Artifact 协议。

建议所有长阶段统一：

```text
1. reserve artifact
2. 上传 staging object
3. 计算 hash/size/media type
4. server validate kind/schema
5. Domain Writer 引用 COMPLETE artifact
6. 写领域状态 + event + outbox
7. Finalizer 对账 expected vs actual
```

同时对每个新阶段要求 contract test：

```text
Artifact Kind 已注册
Domain Writer 已实现
Read/List API 可见
Finalizer 已声明 required/optional artifact
Web 能展示
Cleanup 能处理 staging
Schema validator 已注册
端到端测试通过
```

这能把“新增流程节点但漏配某个 namespace”的问题在发布前拦截。

## 13. Capability-based Adapter Discovery

项目对 Firecrawl 和 Peec 不依赖固定 MCP Server 前缀，而通过 capability 动态发现。这个思想适合更大规模的抓取引擎插件化。

建议定义：

```text
adapter_capability_release
adapter_binding_release
```

能力可以包括：

```text
DISCOVER_MAP
DISCOVER_SITEMAP
FETCH_HTTP
FETCH_MARKDOWN
RENDER_JS
SCREENSHOT
EXTRACT_STRUCTURED
DOWNLOAD_MEDIA
```

站点 Probe 根据需求、成本、健康度和策略选择 Adapter，并在 Run Context 中冻结具体 Binding。

因此核心状态机依赖的是能力契约：

```text
FetchAdapterCapability.fetch(request) -> AdapterOutputSchema
```

而不是：

```text
if engine == "crawl4ai": ...
else if engine == "firecrawl": ...
```

这样以后增加自研 HTTP Fetcher、Playwright、Browserless、Firecrawl 或 Crawl4AI 都不需要改业务状态机。

## 14. 内部链接二次解析值得升级为一等 Link Graph

Crawler 在所有文章 JSON 写完后会再次读取文章集合，计算 `links.inbound_internal`，然后重写文章记录。

这个做法对小批量有用，但在百万文章平台中不能通过“全部读一遍再重写 JSON”实现。

应升级为：

```text
article_link_edge
```

字段至少包括：

```text
source_article_version_id
target_url_id
target_article_id nullable
raw_href
normalized_href
anchor_text
rel
link_type
source_block_path
provenance
created_at
```

Resolver 异步把目标 URL 映射到 article identity，并物化：

- outbound links；
- inbound links；
- orphan article；
- broken internal link；
- 未抓取内部目标；
- 站点内容图谱。

它既能帮助知识库检索，也能帮助发现覆盖检查。但新目标 URL 仍必须以真实 href 作为 `discovery_evidence`，不能由文本推断。

## 15. Resume 与阶段状态

项目使用本地 `state.json` 保存：

```text
prereqs
crawl
voice
analysis
evidence
recommendations
draft
```

Resume 时跳过已完成阶段，只继续未完成阶段。

这个思路正确，但本地 JSON 不适合作为分布式平台真相源。应迁移为：

```text
crawl_run
background_task
domain state
task_event/run_event
outbox_event
run_artifact_manifest
```

`run-summary.json/md` 可以继续生成，但只能是 PostgreSQL 已提交事实的诊断视图，而不能反过来驱动业务状态。

## 16. Dashboard：报告面与控制面分离

项目规定 Dashboard 是 report surface，不拥有 orchestration，也不存在前端“continue gate”。

知识库 Web 应沿用：

```text
Web UI
  -> Control API command
  -> PostgreSQL transaction
  -> event + outbox
  -> Worker

Web query
  <- PostgreSQL + Search + Artifact Metadata
```

Run Now、Pause、Cancel、Retry、Reprocess 都是显式命令。浏览器断线、刷新或多个管理员同时打开页面，都不应该改变后台状态机。

质量页面也应读取 `quality_manifest`，而不是显示 Worker 的“成功”字符串。

## 17. 项目不适合直接作为知识库底座的原因

### 17.1 历史覆盖有限

默认偏向博客首页/索引和有限文章批次，没有：

- Provider Coverage Ledger；
- Sitemap Task Tree；
- Archive Pagination Checkpoint；
- Durable Frontier；
- 完整历史完成判定。

### 17.2 面向单机插件

本地 `state.json` 和 Run Directory 很适合个人插件，但不适合多 Worker、多副本、HA Scheduler。

### 17.3 没有通用长期增量调度

缺少完整的：

```text
schedule_revision
expected_fire_at
misfire_policy
DST policy
overlap window
bounded catch-up
reconcile
```

### 17.4 没有 1000 域名级公平调度

缺少全局/site/domain/Browser 多层配额、weighted fairness、token bucket、backpressure。

### 17.5 状态粒度不足

百万 URL 平台需要独立建模：

```text
frontier
attempt
snapshot
artifact
candidate
quality manifest
article version
index/publish state
```

而不是只保存文章级 stage JSON。

## 18. 对博客知识库最终方案的具体优化

基于源码和真实 Bug，建议在现有方案上增加以下能力。

### 18.1 不可变 Run Context Manifest

Run 创建时冻结所有规则和运行时 release。Worker 不读取 latest 配置，所有任务只引用 `run_context_id`。

### 18.2 Capability Registry + Adapter Binding

Provider/Fetcher/Renderer/Extractor 通过 capability contract 注册。Run 创建时选择具体 Binding 并冻结，避免服务名硬编码，也避免运行中切换实现。

### 18.3 Artifact Kind Registry

Artifact 类型、schema、namespace、生产者、消费者、retention 和 Finalizer 要求只有一个 canonical registry，其他 API/SDK/UI 由它生成或读取。

### 18.4 Server-side Artifact Reservation

对象 key 由服务器分配，Worker 只上传到 reservation 返回的 staging key；commit 时校验 kind/schema/hash，杜绝任意路径写入。

### 18.5 Deterministic Quality Manifest

对最终 Markdown/Article Version 做独立确定性验证；质量事实与 Worker self-report 分离。只有通过 Manifest 的版本才能自动成为 current/indexed/published。

### 18.6 finalize_quality

在 extraction 与 index 之间新增：

```text
finalize_quality(run_id)
```

对账 candidate article versions、quality manifest、COMPLETE Markdown artifact、blocking/review 状态，返回 canonical accepted set。

### 18.7 Article Link Graph

将 inbound/outbound internal link 从文章 JSON 二次重写升级为增量 `article_link_edge` 图模型，服务发现覆盖、断链诊断和检索。

### 18.8 新阶段发布门禁

任何新 stage / artifact kind 必须同时具备 Registry、Writer、Finalizer、Web、Cleanup、Contract Test，否则不能发布。

## 19. 推荐结论

AI Search Blog Optimiser 不应直接承担 1000 个技术博客的全量历史知识库抓取，但它提供了很有价值的工程验证，特别是：

1. Run 必须有唯一 owner，Worker 不得创建或切换运行；
2. Run ID 之外还需要冻结不可变 Run Context，才能真正复现；
3. URL 只能来自真实发现证据或管理员精确输入；
4. 精确目标任务绝不允许静默替代；
5. 抓取 fallback 应固定、版本化、可解释；
6. Article Record / IR 应先于 Markdown；
7. 核心业务写入必须通过 Typed Domain Writer；
8. Stage Finalizer 必须以真实 COMPLETE Artifact 对账；
9. Artifact 类型应该由单一 Registry 管理，避免多个 namespace 枚举漂移；
10. 产物的生产者不能自行证明质量，最终 Article Version 应由独立确定性 Quality Manifest 决定；
11. Capability-based Adapter Binding 比硬编码 crawler 实例更适合可扩展平台；
12. 内部链接应形成增量 Link Graph；
13. Dashboard 读取已提交事实并发出显式命令，不能成为状态真相源。

这些补强项能直接降低长期系统最难排查的三类事故：**同一 Run 混用不同配置、状态显示成功但真实产物不完整、规则/阶段扩展后 Artifact Contract 漏配**。它们适合并入博客知识库最终技术方案。