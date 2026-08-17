# AI 搜索博客优化器：实现细节与技术原理分析

## 1. 调研对象与结论

- 项目：AI Search Blog Optimiser
- 地址：https://github.com/mlobo2012/AI-search-blog-optimiser
- 主语言：Python
- 形态：Claude Code / Claude Cowork 插件 + 本地 MCP Dashboard Server
- 调研基线：仓库主分支 README、CHANGELOG、`commands/blog-optimiser.md`、`skills/blog-optimiser-pipeline/SKILL.md`、`agents/blog-crawler.md`、`dashboard/server.py`、`dashboard/quality_gate.py`、specs/tests。
- 仓库 README/CHANGELOG 当前公开版本线为 0.7.x，CHANGELOG 最新记录到 0.7.1。

项目目标不是“1000 个技术博客全量历史归档”，而是抓取有限批次现有博客文章，结合 Peec 数据、竞争对手与证据，生成 GEO 优化建议及重写文章包。因此它不适合直接作为知识库抓取底座，但其**运行所有权、精确目标语义、能力发现、Typed Writer、真实产物对账、确定性质量门禁、只读 Dashboard、契约回归测试**非常值得吸收。

最终判断：**借鉴运行协议与一致性机制，不直接复用其单机状态模型和有限抓取模型。**

## 2. 核心流水线

主流程由 `skills/blog-optimiser-pipeline/SKILL.md` 定义：

```text
prereqs
 -> register run
 -> crawl
 -> voice
 -> analysis
 -> evidence
 -> recommendations
 -> draft
 -> deterministic validation
 -> final report
```

其关键设计不是 Agent 数量，而是职责边界：

```text
Main Session = Orchestrator / Global State Owner
Leaf Agents   = Crawl / Evidence / Recommend / Generate Worker
Dashboard MCP = Host-side Artifact + State Service
Dashboard Web = Report Surface
```

主编排负责阶段顺序和全局状态，叶子 Agent 不能创建新的 Run，也不能切换 Run ID。这个原则对分布式知识库同样成立：Control Plane / Scheduler 才能创建 `crawl_run`，Worker 只能消费既定 `crawl_run_id + run_context_id + task_id`。

## 3. Run 所有权：提示词规则必须升级成服务端权限

项目明确规定 Crawler：

- 必须从 Orchestrator 收到 `run_id`；
- 缺少 `run_id` 立即失败；
- 不得调用 `register_run`；
- 不得创建或切换其他 Run；
- 所有写入都必须使用收到的 Run。

这解决了典型分裂：

```text
Orchestrator: run A
Worker:       自行创建 run B
Artifact:     写入 B
后续阶段:     继续读取 A
Dashboard:    A 显示空产物或错误状态
```

大型平台不能只依靠 Agent 指令约束，而要在服务端强制：

```text
create_run 权限只给 Scheduler / Control Plane
Worker Credential 不具备 create_run
Domain Writer 校验 crawl_run_id
Domain Writer 校验 task_id + lease_epoch
Domain Writer 校验 run_context_id/context_hash
Worker 不得在“找不到 Run”时自动创建 fallback Run
```

## 4. 仅有 Run ID 还不够：必须冻结 Run Context

该项目用 `register_run` 返回固定 `run_id`、run directory 和各种 absolute path，已经解决了一部分运行归属问题，但大规模长期系统还会遇到规则版本漂移。

例如运行期间可能发生：

- Extraction Rule 发布新版本；
- URL Filter 改动；
- Fetch Profile 改 Browser fallback；
- Crawl4AI / Firecrawl Adapter 升级；
- Markdown Renderer 改版；
- Quality Contract 改阈值；
- Secret Version 轮换。

如果 Worker 每次读取“最新配置”，同一 Run 会混入多种语义。因此方案应冻结不可变 `run_context_manifest`：

```text
run_context_id
crawl_run_id
site_profile_release_id
site_source_release_ids
provider_release_ids
url_filter_release_id
url_scorer_release_id
fetch_profile_release_id
browser_action_plan_release_id
adapter_binding_release_id
adapter_schema_release_id
extraction_pipeline_release_id
markdown_renderer_release_id
quality_contract_release_id
runtime_bundle_release_ids
credential_profile_id
secret_version_refs
policy_release_id
context_hash
created_at
```

Worker payload 只引用 `run_context_id`，不能隐式读取 latest release。

## 5. Capability-based Discovery：不硬编码服务名

项目在 Cowork 环境里不能假定 Peec/Firecrawl 的 MCP 前缀固定，因此采用 capability discovery：先搜索工具能力，再选择实际连接实例。Crawl4AI 是主测试路径，Firecrawl 是支持的替代路径。

这是比 `if crawler == crawl4ai` 更可扩展的模式。知识库平台应定义 `adapter_capability_release`：

```text
adapter_id
adapter_release
capabilities
input_schema_version
output_schema_version
runtime_class
cost_class
health_probe
limits
created_at
```

能力例如：

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

Control Plane 在创建 Run 前做**小而确定的能力探针**：

1. 校验要求的 capability 是否存在；
2. 执行低成本 health probe；
3. 根据站点需求、健康度、成本和策略选择 Adapter；
4. 生成 `adapter_binding_release`；
5. 将 Binding 冻结到 Run Context；
6. Run 运行中不得静默切换另一实现。

如果确实要 fallback，应创建显式的 fallback attempt，并保存 `fallback_reason`，而不是偷偷换后端。

## 6. URL 发现：精确目标语义非常值得保留

项目支持重复 `--article-url`。只要该参数存在：

1. 跳过博客索引发现；
2. 只处理输入 URL；
3. 保留输入顺序；
4. 只删除真正重复项；
5. 不用最近文章、相关页、Sitemap 或推测 URL 补位；
6. 任意目标没有真实持久化即视为失败。

这应直接映射为：

```text
TARGETED_FETCH
TARGETED_REPROCESS
```

适合漏抓修复、Selector 调试、Browser Action 调试、Golden Case、事故恢复。

核心语义是：**目标失败就是失败，不能用“相似成功”掩盖精确目标缺失。**

## 7. 索引发现模式的价值与规模边界

Crawler 在发现模式下：

- Firecrawl：优先 `map`，结果太薄再 scrape 入口页；
- Crawl4AI：先 `md(raw)`，链接不足再拿 HTML；
- 只接受真实出现在 map/Markdown link/HTML href/canonical 的 URL；
- 不根据标题猜 slug；
- 排除 category/tag/author/pagination/feed；
- 最后按 `max_articles` 截取。

这是一种**确定性有限发现**，适合内容优化批处理，不等于全量历史覆盖。

面向 1000 站必须扩展成多 Provider：

```text
Sitemap / Sitemap Index
RSS / Atom / JSON Feed
站点公开 API
年/月 Archive
Category / Tag / Author Pagination
普通 HTML 同域链接
Browser 动态列表
外部公开索引补漏
```

每个 Provider 需要独立 checkpoint、continuity、coverage evidence 和完成条件。任何 `max_articles/max_pages/max_entries` 都只能是 execution slice 预算，不能作为 FULL_BACKFILL 完成判据。

## 8. 固定 fallback：减少 Agent 临场自由度

项目的 Crawler 使用固定抓取顺序：

```text
Firecrawl: scrape(markdown + html)

Crawl4AI:
md(raw)
 -> empty/thin/anti-bot/script-heavy shell
 -> html
```

它还明确禁止为了补 metadata 随意调用 JS 或 LLM 推断字段。该原则适合长期系统：**fallback 是版本化产品契约，不是 Worker 自由试错。**

推荐 `fetch_profile_release`：

```text
HTTP_FETCH
 -> HTTP_STRUCTURED_EXTRACT
 -> RENDER_JS
 -> SITE_API_ADAPTER
 -> MANUAL_DIAGNOSIS
```

Browser escalation 记录原因：

```text
EMPTY_ARTICLE_BODY
SPA_SHELL
JS_PAGINATION
LOAD_MORE
INFINITE_SCROLL
DYNAMIC_RENDER_REQUIRED
EXTRACTION_READINESS_FAILED
```

## 9. Article Record：Markdown 不能成为唯一事实

项目为文章构造结构化记录，包括：

```text
url/fetched_at/crawl_backend
meta: title/description/canonical/OG/Twitter/robots/hreflang
schema: JSON-LD types/raw_ldjson
structure: h1/heading tree/word count/table/list/quote/FAQ/code
media: images/videos/iframes/thumbnail
trust: author/role/published_at/updated_at/entities
links: internal/external/inbound_internal
body_md
raw_html_path
```

它已经接近轻量 Article IR。知识库平台应进一步拆成：

```text
Raw Network Snapshot
 -> Rendered/Cleaned DOM
 -> Extraction Candidates
 -> Article IR
 -> Quality Selection
 -> Markdown Render
 -> Article Version
 -> Index/Publish
```

这样 Extraction Rule 或 Renderer 升级时可离线 replay，不必再次访问源站。

## 10. 本地原子写的真正语义与局限

`dashboard/server.py` 的 `_atomic_write` 使用：

```text
write .tmp
 -> rename/replace
```

并用进程级 `RLock` 串行保护状态写入。这能防止单个 JSON 文件出现半写入，适合单机插件。

但 `record_crawled_article` 的业务操作本质上是：

```text
写 articles/{slug}.json
 -> merge state.json
 -> 写 state.json
```

这两个文件并不是一个跨文件 ACID 事务。如果进程恰好在二者之间退出，就可能出现“文章文件已存在、state 还没更新”。项目用 `finalize_crawl` 再扫描真实产物来修正状态，这是合理的补偿式一致性。

大型平台不应照搬本地文件事务，而应升级为：

```text
S3/MinIO staging upload
 -> server-side artifact commit
 -> PostgreSQL Domain Writer transaction
 -> run_artifact_manifest
 -> Stage Finalizer reconcile
```

对象存储和 PostgreSQL 无法组成普通单库事务，所以必须依赖 Reservation/Commit + Manifest + Finalizer，而不是假装跨系统“完全原子”。

## 11. Typed Writer：把状态迁移和产物提交绑定

项目优先使用：

```text
record_crawled_article
record_voice_baseline
record_peec_gap
record_competitor_snapshot
record_evidence_pack
record_recommendations
record_draft_package
fail_article_stage
finalize_crawl
finalize_run_report
```

而不是让 Agent 随意 `write_json + update_state`。

这种做法的价值是把约束下沉到服务端。知识库平台应定义：

```text
record_discovered_url(...)
commit_fetch_snapshot(...)
commit_extraction_candidate(...)
commit_article_version_candidate(...)
commit_quality_manifest(...)
accept_article_version(...)
commit_link_edges(...)
commit_index_result(...)
commit_publish_result(...)
```

Writer 统一负责 schema、Run Context、lease、cancel、幂等、Artifact 引用、领域更新、event 和 outbox，并在单 PostgreSQL 事务提交。

## 12. Finalizer：真实 persisted set 才是下游输入

项目 `finalize_crawl` 会重新查看真实 `articles/*.json`，并对比发现数和精确目标集合：

- persisted = 0：FAILED；
- persisted < discovered：PARTIAL；
- exact URL 缺失：FAILED；
- exact URL 出现未请求产物：FAILED；
- 下游只消费 Finalizer 返回的 `article_slugs`。

这个原则非常重要：**Worker self-report 不是阶段完成事实。**

大型平台的 Finalizer 应比较：

```text
数据库期望集合
vs COMPLETE artifact_object
vs run_artifact_manifest
vs 领域表引用
vs quality manifest
vs retry/dead-letter 集合
```

输出：

```text
COMPLETED
PARTIAL
FAILED
INCONSISTENT
```

Resume Planner 也必须基于 Finalizer 的 canonical persisted/accepted set 计算剩余工作，而不是根据客户端或 Worker 的 stage flag 猜测。

## 13. 确定性 Quality Gate：生产者不能证明自己正确

`dashboard/quality_gate.py` 是独立确定性验证器。它读取真实文章、建议、证据、HTML/schema 等产物，计算结构化 Manifest，并检查来源数量、内部/外部链接、schema、FAQ、标题结构、scope drift、trust/reviewer、recommendation implementation map 等。

最有价值的原则不是 GEO 规则，而是：

```text
Producer -> Candidate Artifact
Validator -> Deterministic Quality Manifest
Finalizer -> Accepted Set
Indexer/Publisher -> only accepted set
```

知识库平台的 `quality_contract_release` 应定义：

```text
required_checks
optional_checks
thresholds
site_overrides
schema_version
validator_runtime_release
```

机器校验事实与人工 Override 分开存储，人工不能覆盖原始 Validator 结果。

## 14. 真实 Bug 进一步暴露两个大型系统常见问题

### 14.1 字段所有权冲突：后写者覆盖正确结果

CHANGELOG 0.6.x 记录了一个典型事故：Validator 已得到正确 trust/author 结果，随后 `dashboard/server.py` 的另一个 fallback writer 又把结果覆盖，导致正确状态被 clobber。后续通过 early-return 才修复。

这个问题在多人、多 Worker 平台更危险。应新增**Field Ownership Registry**：

```text
entity_type
field_path
owner_service
allowed_transition
immutable_after_stage
merge_policy
schema_version
```

例如：

```text
quality_manifest.*         只允许 Quality Validator 写
article_version.current    只允许 Acceptance Service 推进
fetch_snapshot.*           只允许 Fetch Commit Writer 创建
crawl_run.final_status     只允许 Stage/Run Finalizer 计算
manual_override.*          只允许人工审核服务写
```

其他服务只能写 companion evidence，不得重写权威字段。这样可以从架构上消除“第二个 writer 把正确结果覆盖”的问题。

### 14.2 契约形状漂移：生产端和校验端字段不一致

项目多轮 Bug 包括 generator 输出 `rec_implementation_map` 的字段形状与 Validator 预期不同、非适用 sentinel 形状缺失、recommendation count/prompt-id 约束漂移等。

根因是契约同时散落在 Agent prompt、Python Validator、Dashboard、测试和文档中。

平台应建立**Canonical Contract Registry + 代码生成/校验**：

```text
JSON Schema / Pydantic schema
 -> Worker SDK types
 -> API schema
 -> Validator input schema
 -> Web form/view model
 -> Contract tests
```

发布门禁要求：新增 stage/artifact/field 必须同时通过 producer-consumer compatibility、golden artifact、replay 和 downgrade/rollback 测试。

## 15. Dashboard 的正确边界

项目明确规定 Dashboard 是 report surface，不拥有 orchestration，也没有前端 continue gate。这个设计应保留：

```text
Web Command
 -> Control API
 -> PostgreSQL transaction
 -> event/outbox
 -> Worker

Web Query
 <- PostgreSQL/Search/Artifact metadata
```

浏览器刷新、断线、多个管理员同时打开页面，都不能改变后台状态机。

项目为了 Cowork 生命周期把 HTTP Dashboard 作为 detached process 独立存活，这对单机插件合理；生产平台应改成无状态 Web/API 多副本，所有事实回到 PostgreSQL/Object Storage，而不是依赖本地 daemon lock/PID 文件。

## 16. Link Graph：小批量二次重写需升级成增量图模型

项目抓完文章后再读取所有文章 JSON，计算 `inbound_internal` 并重写。这在几十篇文章时可行，在百万文章规模会产生 O(N) 重扫和大量对象重写。

大型方案应维护一等模型：

```text
article_link_edge_id
source_article_id
source_article_version_id
target_url_id
target_article_id
href
anchor_text
rel
first_seen_at
last_seen_at
status
```

Link Resolver 增量解析 target，支持：

- inbound/outbound；
- broken link；
- orphan article；
- redirect/canonical 重解析；
- 覆盖补漏；
- 检索和内容图谱增强。

新增 URL 仍必须来自真实 href evidence，不能从语义猜测。

## 17. 不应直接照搬的部分

### 17.1 单机 `state.json`

适合个人插件，不适合多副本、多 Worker、HA Scheduler。生产状态真相源应是 PostgreSQL。

### 17.2 有限 `max_articles`

适合内容优化批次，但不能代表 FULL_BACKFILL 完成条件。

### 17.3 顺序抓取与 Agent 批处理

适合数篇文章。1000 站需要 durable frontier、分布式 lease、公平调度、域级限流和 backpressure。

### 17.4 依赖本地目录作为 Artifact Namespace

生产应使用对象存储 + Artifact Registry + content hash + retention policy。

### 17.5 GEO/Peec 业务规则

与知识库抓取目标无直接关系，只借鉴其 evidence/validator/contract 架构。

## 18. 对最终博客知识库方案的具体优化

本次调研建议把以下内容正式纳入最终方案：

1. **运行前 Capability Probe**：低成本探针确认 Adapter 可用性，选择 Binding 后冻结进 Run Context。
2. **Adapter Binding 不可变**：同一 Run 不静默切换 crawler，实现变化必须形成显式 attempt/provenance。
3. **Field Ownership Registry**：关键领域字段只能由唯一权威服务写，防止 downstream clobber。
4. **Canonical Contract Registry**：Artifact/Domain Schema 成为单一事实源，API、SDK、Validator、Web 和测试共享。
5. **Golden Contract Test**：精确 URL、Artifact shape、Validator、Finalizer、Replay 都要进入发布门禁。
6. **Canonical Persisted Set**：所有 Resume/下游阶段只读取 Finalizer 输出的真实 persisted/accepted set。
7. **Artifact Reservation + Commit**：把项目的 `.tmp + rename` 单机原子性升级为对象存储 staging/commit 协议。
8. **Quality Manifest 独立计算**：生产 Worker 不允许自行将结果标记为通过。
9. **Exact Target Contract**：Targeted Fetch/Reprocess 任一目标缺失不得静默补位。
10. **Link Graph 增量化**：不再通过全量重写文章 JSON 维护 inbound link。

## 19. 许可证与代码复用风险

仓库 LICENSE 顶部包含 `Commons Clause` 条件，并附带 Apache 2.0 文本，同时 `Software:` 字段写的是 `AI Heroes Travel Agent`，与当前仓库名称不一致。GitHub 元数据也不能给出标准 SPDX 许可证标识。

因此本项目应作为**架构调研参考**使用；若计划直接复制或改造其代码，需要先做许可证和 Software 标识的法律确认。知识库方案本身只吸收通用设计思想，建议采用 clean-room 实现，不直接搬运受许可不确定性影响的代码。

## 20. 最终评价

AI Search Blog Optimiser 的价值不在于“能抓多少页面”，而在于它用真实工程演进证明了几条长期平台原则：

1. 全局 Run 必须有唯一 Owner；
2. Worker 不能自行创建或切换 Run；
3. Run Context 必须冻结配置语义；
4. URL 必须来自真实发现证据或精确输入；
5. fallback 必须固定、版本化并记录原因；
6. Artifact 和状态写入必须通过 Typed Writer；
7. Finalizer 必须以真实产物集合重新计算完成状态；
8. 质量由独立确定性 Validator 证明；
9. 关键字段要有单一写入所有权，避免后处理覆盖正确事实；
10. Producer/Validator/Web 的数据结构必须来自同一 Canonical Contract；
11. Capability-based Adapter Binding 比硬编码 crawler 名称更适合长期扩展；
12. Dashboard 是事实展示与命令入口，不是流程真相源；
13. 本地 `state.json + atomic rename` 适合插件，但生产必须升级为 PostgreSQL + Object Storage + Manifest Reconciliation。

这些增量会直接降低三类最难排查的事故：**同一 Run 混用不同语义、状态显示成功但真实 Artifact 不完整、多个 Writer/契约版本互相覆盖或漂移**。