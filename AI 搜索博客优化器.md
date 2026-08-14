# AI 搜索博客优化器：实现细节与技术原理分析

## 1. 项目概览

- 项目：AI Search Blog Optimiser
- 地址：https://github.com/mlobo2012/AI-search-blog-optimiser
- 类型：Claude Code / Claude Cowork 插件
- 核心用途：给定企业博客入口，使用 Crawl4AI/Firecrawl 抓取现有文章，再结合站点级品牌语气、Peec AI 搜索数据、竞争对手与证据数据，生成优化建议与文章重写包。
- 与本知识库需求最相关的部分：博客 URL 发现、抓取后端分层、运行级状态、产物持久化、阶段恢复、Dashboard、类型化写入 API、产物对账与质量门禁。

该项目并不是面向“1000 个站点全量历史归档”的通用爬虫平台。它默认面向有限文章批次和内容优化工作流，因此不能直接作为知识库抓取底座。但它在“任务编排与产物一致性”方面有若干值得吸收的实现模式。

## 2. 仓库结构与职责划分

仓库主要分为：

```text
.claude-plugin/   Claude 插件清单与 marketplace 配置
commands/         slash command 入口与总编排约束
skills/           canonical pipeline playbook
agents/           crawl / voice / evidence / recommend / generate 等叶子 Agent
dashboard/        本地 MCP Server、Web Dashboard、质量校验
references/       GEO 规则与参考契约
specs/            设计与缺陷说明
tests/            Dashboard、状态和流水线回归测试
```

关键设计是：**主会话负责 orchestration，子 Agent 只做叶子工作**。`commands/blog-optimiser.md` 明确要求主会话是唯一 orchestrator，不允许再创建一个“总编排子 Agent”。这样避免多个协调者同时创建 run、切换 run_id 或争夺状态所有权。

对知识库系统的启发是：控制面必须明确“谁拥有状态机”。Crawler、Extractor、Indexer 等 Worker 只执行领取到的领域任务，不应该拥有全局 run/schedule 生命周期。

## 3. Run ID 纪律与运行目录

### 3.1 先注册，再执行

新任务必须先调用 `register_run`，拿到具体 `run_id` 和所有运行目录，然后才允许打开 Dashboard 或启动抓取。Crawler 子 Agent 明确禁止自己调用 `register_run`，也禁止切换 run_id。

这相当于把一次处理建立成显式运行上下文：

```text
runs/{run_id}/
  state.json
  run-summary.md
  outputs/
    articles/
    evidence/
    recommendations/
    optimised/
```

站点级可复用资产单独放在：

```text
sites/{host}/
  voice.json
  reviewers.json
```

这种布局把“本次运行产物”和“站点长期资产”分开。它适合迁移到我们的知识库设计：

- `crawl_run`：一次逻辑运行；
- `site_profile_release`：站点长期可复用配置；
- `run_artifact_manifest`：本次运行所有真实落盘/对象存储产物；
- 运行结束时按 manifest 对账，而不是只相信 task 状态。

### 3.2 为什么不能用标题或 slug 当身份

Crawler 规则明确要求：

- 不允许从标题推断 slug；
- 只能使用索引实际出现的 href 或 canonical URL；
- `--article-url` 指定后只处理这些精确 URL，不能用相似文章替代。

这是很重要的身份约束。标题、slug、路径尾段都可能重复或变化，不能作为内部主键。知识库平台应继续使用稳定 `url_id/article_id/snapshot_id`，URL 规范化后的 hash 只作为 URL identity，不以标题决定更新对象。

## 4. URL 发现机制

项目支持两种模式：

### 4.1 精确 URL 模式

若传入一个或多个 `--article-url`：

1. 规范化；
2. 仅去除精确重复；
3. 保留输入顺序；
4. 跳过博客索引发现；
5. 不从最近文章、Sitemap、相关文章补齐；
6. 任何指定 URL 未持久化都应使此次精确任务失败。

这对大型抓取平台非常实用，应增加“Targeted Run / 单 URL 调试”模式，用于：

- 单篇重抓；
- selector 调试；
- extraction replay 验证；
- 用户手工指定疑似漏抓 URL；
- incident 修复。

### 4.2 索引发现模式

Crawl4AI 路径先读取博客入口的 raw markdown；若 markdown 太薄或没有足够链接，再获取 HTML 并解析真实 href。

Firecrawl 路径先执行 map，必要时再 scrape 博客入口。

发现后只接受：

- 同源；
- 实际出现在 map/markdown/html/canonical 中的 URL；
- 符合明显文章路径模式的页面。

并排除：category、tag、author、pagination、feed 等。

这种模式的优点是确定性高，但缺点也明显：它偏向“博客首页最近文章”，不等价于完整历史发现。对于 1000 站全量历史，必须用 Sitemap、API、归档页、分类分页、Feed、HTML 递归等多个 Provider，并为每个 Provider 保存 checkpoint 与 coverage evidence。

## 5. 固定抓取后端与 fallback 顺序

项目强调固定后端顺序，避免 Agent 自由试错导致行为不可复现。

### Crawl4AI

```text
md(raw)
  -> 内容为空/过薄/SPA shell
  -> html
```

### Firecrawl

```text
scrape(markdown + html)
```

此外，Crawl4AI 截图只作为可选产物，缺失不能让整个抓取任务失败。

技术原理是把“成功条件”和“升级条件”结构化，而不是让模型临时决定。我们的平台应继续采用 HTTP-first，并保存 Browser escalation 原因，例如：

```text
EMPTY_ARTICLE_BODY
SPA_SHELL
JS_PAGINATION
LOAD_MORE
INFINITE_SCROLL
DYNAMIC_RENDER_REQUIRED
```

同时固定 adapter release 与 runtime bundle，保证回归可解释。

## 6. Article Record：先结构化，再下游处理

Crawler 把每篇文章整理成统一 JSON，而不是直接把 Markdown 当唯一事实。主要字段包括：

```text
url / fetched_at / crawl_backend
meta: title / description / canonical / OG / Twitter / robots / hreflang
schema: JSON-LD 类型与原始数据
structure: H1 / heading tree / word count / table / list / quote / FAQ / code
media: image / video / iframe / thumbnail
trust: author / published_at / updated_at / credential / entity
links: internal / external / inbound_internal
cta
body_md
raw_html_path
```

该模式本质上是一个轻量 Article IR。它说明最终 Markdown 不应该是抓取系统的唯一中间表示：

1. raw response 用于审计和 replay；
2. structured article record / Article IR 用于身份、diff、质量和转换；
3. Markdown 是一个稳定渲染产物；
4. downstream search / RAG 依赖 article_version，而不是直接依赖 crawler 的临时 markdown。

## 7. 类型化写入：把业务产物和状态一起提交

这是项目最值得吸收的部分之一。

Crawler 不建议在成功时分别执行：

```text
write article.json
update state.json
```

而是使用 `record_crawled_article`。其语义是：**写入文章产物，同时写入对应 crawl-stage 状态**。类似地，后续阶段分别使用：

```text
record_voice_baseline
record_peec_gap
record_competitor_snapshot
record_evidence_pack
record_recommendations
record_draft_package
fail_article_stage
```

这比一个万能的 `update_state` 更安全，因为：

- 参数结构受领域约束；
- 状态迁移和业务产物保持一致；
- 调用者不能随意构造非法状态；
- 可以在服务端做 schema 校验、幂等和原子写入；
- 更容易做事件审计与自动测试。

对我们的知识库方案，应该引入明确的领域写入服务，例如：

```text
record_discovered_url(...)
record_fetch_snapshot(...)
record_extraction_candidate(...)
commit_article_version(...)
record_quality_result(...)
commit_index_result(...)
commit_publish_result(...)
```

这些服务内部统一执行 PostgreSQL transaction + event + outbox，并引用已经完成的 S3/MinIO artifact。通用 `background_task` 只保存运行生命周期，不能成为万能业务写入接口。

## 8. finalize_crawl：状态必须和真实产物对账

Crawler 在所有文章处理完成后必须调用 `finalize_crawl`。主编排器在子 Agent 返回后还会再调用一次，然后用真实 `articles/*.json` 集合决定后续阶段。

规则包括：

- `persisted_count == 0`：整个 crawl 失败；
- 发现数大于真实持久化数：状态为 partial，并只对真实持久化集合继续；
- 精确 URL 模式下，只要任一指定 URL 未持久化即失败；
- 下游阶段使用 `finalize_crawl` 返回的 article slugs，而不是 Agent 自报的成功集合。

它解决的是常见“状态漂移”问题：Worker 可能说成功，但文件没写成；或写了部分文件却把 run 标为全部成功。

大型知识库平台不能照搬“扫描本地文件夹”，但应该保留这个原则，升级为 **Artifact Manifest Reconciliation**：

```text
DB 期望产物
vs
S3/MinIO COMPLETE artifact_object
vs
领域表引用
vs
阶段状态
```

阶段终结器只有在这些集合对齐后才允许将 phase 标为 COMPLETED。否则应标 `PARTIAL/INCONSISTENT` 并生成修复任务。

## 9. state.json、Resume 与阶段状态机

项目流水线分为：

```text
prereqs
crawl
voice
analysis
evidence
recommendations
draft
```

每阶段有状态。Resume 模式读取既有 `state.json`，已完成阶段不重复执行，只恢复未完成阶段。

它的优点是：

- 阶段检查点明确；
- 不会因为重新运行命令而无条件重抓；
- 支持部分失败后继续；
- 每个叶子 Agent 只负责自己的 stage。

但在 1000 站平台中，不能让本地 JSON 成为业务真相源。需要将其概念迁移为 PostgreSQL 中的 `crawl_run + background_task + domain state + event timeline`。可以继续导出 `run-summary.json/md` 作为诊断产物，但它只能是数据库事实的物化视图。

## 10. Dashboard 设计：报告面与控制面分离

项目明确规定：Dashboard 是 report surface，不负责 orchestration，不依赖“continue gate”。

这避免一个常见架构问题：前端页面按钮直接成为任务状态真相源。如果浏览器刷新、断开或多个管理员同时操作，状态容易分裂。

对于我们的 Web 管理端，应采用：

```text
Web UI
  -> Control API（明确命令）
  -> PostgreSQL transaction
  -> run/task/event/outbox
  -> Worker

Dashboard 查询
  <- PostgreSQL / Search / Artifact metadata
```

UI 可以提供 Run Now、Pause、Cancel、Retry、Reprocess，但这些都是明确命令，不能让前端“持有流程锁”。

## 11. 能力发现而不是硬编码服务名

项目对 Peec 和 Firecrawl 使用 capability-based discovery：外部 MCP 前缀可能是动态 UUID，不依赖固定 server name。Crawl4AI 则在当前产品约束下固定为 `c4ai-sse`。

抽象出来的原则是：插件/Adapter 接入应根据能力契约，而不是依赖实例名。

在我们的平台中可定义：

```text
DiscoveryProviderCapability
FetchAdapterCapability
ExtractionAdapterCapability
PublishAdapterCapability
```

例如一个新抓取引擎只要满足：

```text
fetch(url, profile) -> adapter_output_schema
```

就可以注册，而不改核心状态机。

## 12. 项目中的工程限制

该项目适用于内容优化，但不能直接满足本需求，主要限制包括：

### 12.1 发现覆盖有限

默认一次发现最多处理有限文章，并以博客索引实际链接为主。它没有全量历史覆盖台账、Sitemap task tree、Archive pagination checkpoint、durable frontier 等机制。

### 12.2 面向单机/本地运行

`state.json`、本地 run directory 与本地 Dashboard MCP 很适合插件，但不适合多 Worker、多副本、HA 调度。

### 12.3 缺少长期增量调度

项目强调内容优化周期，但不是通用同步 Scheduler。没有完整的 schedule revision、misfire、DST、expected fire、overlap window 和 reconcile 语义。

### 12.4 缺少网络级全局公平调度

没有针对 1000 域名的 weighted fairness、per-domain token bucket、global/site/browser 三层配额和大规模 backpressure。

### 12.5 抓取状态粒度较粗

对百万 URL 平台，需要 URL frontier、attempt、snapshot、artifact、extract candidate、article version 等更细领域模型，而不能只用每篇文章的 stage JSON。

因此正确做法不是采用它作为基础框架，而是吸收它的“运行纪律 + typed writer + artifact reconciliation + report-only dashboard”思想。

## 13. 对博客知识库技术方案的具体优化

### 13.1 增加 Run Workspace / Artifact Manifest 概念

每个 `crawl_run` 应有逻辑 workspace：

```text
crawl_run_id
site_id
mode
expected_fire_at
workspace_manifest_version
```

对象存储中按稳定 ID 组织：

```text
runs/{crawl_run_id}/raw/...
runs/{crawl_run_id}/rendered/...
runs/{crawl_run_id}/extracted/...
runs/{crawl_run_id}/reports/...
```

同时建立 `run_artifact_manifest` / `artifact_object`，阶段结束必须对账 COMPLETE artifacts。

### 13.2 增加 Stage Finalizer

每个长流水线阶段完成时执行服务器端 finalizer：

```text
finalize_discovery(run_id)
finalize_fetch(run_id)
finalize_extraction(run_id)
finalize_index(run_id)
finalize_publish(run_id)
```

Finalizer 只根据数据库和 COMPLETE artifact 判断：

```text
COMPLETED
PARTIAL
FAILED
INCONSISTENT
```

Worker 自报成功不能直接决定阶段完成。

### 13.3 领域写入 API 替代万能 task patch

新增 Typed Domain Writer：

```text
commit_fetch_snapshot
commit_extraction_candidate
commit_article_version
commit_quality_evaluation
commit_index_result
commit_publish_result
```

每个 writer 在单一 DB transaction 中提交：

```text
领域状态
+ run/task event
+ outbox
```

大对象先 staging 上传，完成 hash 校验后再被 writer 引用。

### 13.4 增加 Targeted Run

Web/CLI 支持精确 URL 集合：

```text
TARGETED_FETCH
TARGETED_REPROCESS
```

指定 URL 后不允许系统用相似 URL 替代，结果中必须给出逐 URL 成功/失败证据。它是定位漏抓、修复单篇、验证 selector 和回归测试的重要工具。

### 13.5 增加“发现证据”约束

发现器写入 URL 时同时保存：

```text
provider_id
source_document/sitemap/archive/page
source_parent_url
source_position_or_entry_key
discovered_url
canonical_hint
discovered_at
```

禁止根据标题生成 URL，禁止 silent substitution。这样 Web 可以回答“这个 URL 为什么会进入 frontier”。

### 13.6 Dashboard 查询真实状态，不持有状态

Web Dashboard 的进度、数量和完成状态来自 PostgreSQL + artifact manifest，而不是 Worker 内存或前端 gate。

Run Now / Cancel / Retry / Reprocess 通过 Control API 生成事务性命令与事件；Dashboard 只是可观察、可审计的控制入口。

## 14. 推荐结论

该项目不适合直接承担 1000 个技术博客的全量历史抓取，但它验证了几项非常有价值的工程原则：

1. 一次运行只能有一个明确 `run_id`，Worker 不得私自创建或切换运行上下文；
2. URL 必须来自真实发现证据，不能由标题/slug 猜测；
3. 抓取 fallback 必须固定并结构化记录原因；
4. 文章应先形成结构化 Article Record/IR，再生成 Markdown；
5. 核心业务写入使用 typed writer，让产物与状态一起提交；
6. 阶段结束必须按照真实持久化产物执行 reconciliation；
7. partial 必须是显式状态，不能把部分成功伪装成完成；
8. Resume 只恢复未完成阶段；
9. 站点长期配置和一次运行产物应分开；
10. Dashboard 应是报告/控制入口，但不能成为任务状态真相源。

这些原则已经适合并入博客知识库最终方案，尤其可以提升“Worker 自报成功但对象缺失”“Web 显示完成但真实 Markdown 不完整”“重试后状态和产物不一致”这类长期运行系统最难排查的问题。
