# 我构建了一组自动写博客的 AI 智能体，因此不用再自己写了

## 1. 调研对象

- 原文：I Built a Crew of AI Agents That Write Blogs So I Don’t Have To
- 地址：https://dev.to/kuldeep_paul/i-built-a-crew-of-ai-agents-that-write-blogs-so-i-dont-have-to-heres-how-you-can-too-2i84
- 相关代码仓库：https://github.com/dskuldeep/blog-writing-agent.git
- 主要技术：CrewAI、Tavily Search、Crawl4AI、Gemini、Maxim AI

本文的直接目标是自动生产博客，但真正值得复用到“1000+ 技术博客全量历史抓取 + 增量同步 + Markdown 知识库 + Web 管理”场景的，不是让大模型参与每篇文章的正文生成，而是它体现出的 **阶段化工作流、工具封装、质量审查、修订回路与全链路可观测性**。

## 2. 原项目实现结构

原项目把博客生产拆成多个职责独立的 Agent：

1. Research Agent：搜索资料并抓取网页；
2. Outline Agent：基于研究结果生成文章结构；
3. Writer Agent：生成初稿；
4. Critique Agent：检查初稿质量；
5. Revision Agent：根据审查结果修订；
6. Export Agent：保存最终文件。

任务按顺序执行，前一个阶段的结果成为后一个阶段的输入。CrewAI 负责 Agent、Task 和执行顺序的编排，Gemini 作为统一 LLM，Tavily 负责搜索，Crawl4AI 负责网页抓取，Maxim AI 负责 Trace、Tool Call 和 Event 级观测。

这种结构的关键不是 Agent 数量，而是把复杂工作拆成可观察的阶段，并显式区分“产生候选结果”和“检查候选结果”。这与生产级爬虫系统的 Fetch、Extract、Quality、Repair、Publish 分层高度一致。

## 3. 搜索工具实现原理

文章把 Tavily 封装成 CrewAI 的 Tool。输入包含：

- query；
- max_results；
- search_depth。

工具内部执行搜索，再把结果收敛为 title、content、url 等必要字段，同时为一次 Tool Call 建立追踪记录。

对博客知识库平台的启示是：**外部能力必须通过统一 Adapter/Tool 边界接入，业务层不能依赖供应商返回结构。**

但对于已经明确的 1000 个目标站点，通用搜索引擎不应成为历史 URL Discovery 主路径。更可靠、成本更低的优先级应是：

```text
CMS/API
 -> Sitemap/Sitemap Index
 -> RSS/Atom
 -> Archive/Category/Pagination
 -> 站内有限 Deep Crawl
 -> 搜索引擎/互联网归档补洞
```

搜索适合 Source Onboarding、缺口补洞或异常诊断，而不是已知站点每次增量同步都调用。

## 4. Crawl4AI 工具实现原理

项目中的 WebCrawlerTool 使用 `AsyncWebCrawler`，为每个 URL 创建异步任务，再用 `asyncio.gather` 并发等待结果。单 URL 抓取结果最终被整理成：

```text
content: Markdown
images: 图片 URL
url: 原 URL
title: 页面标题
```

这是一个典型的 I/O 并发模型：网页抓取时间主要消耗在 DNS、TLS、服务器响应和下载等待上，因此用 asyncio 能用较少线程管理大量并发连接。

但是从“演示代码”升级到 1000 站生产系统必须增加以下约束：

- 全局并发上限；
- per-host 并发与 QPS；
- robots/policy；
- 429/5xx/timeout 分类重试；
- 有界 batch，不能对超大 URL 集一次性 `gather`；
- backpressure；
- task lease、checkpoint、幂等键；
- HTTP 与 Browser Worker Pool 隔离；
- Source 级公平调度，避免大站回填饿死其他站点增量任务。

因此生产实现应是：Scheduler 持续投递有限批次任务，Worker 从 Queue 消费；而不是把数十万 URL 一次性构造成同一进程中的协程数组。

## 5. 事件循环实现上的风险

文章为了把异步 Crawl4AI 包装成同步 Agent Tool，采用了 notebook/交互环境常见的 `nest_asyncio` 与同步入口调用异步协程的方式。

这种方式适合 Demo，但不适合作为长期爬虫 Worker 的基础执行模型。生产服务应该让异步 Worker 自己拥有事件循环：

```text
scheduler -> queue -> async http/browser worker -> result store
```

原因包括：

- 嵌套事件循环增加运行时行为复杂度；
- 阻塞式 Agent 调用会削弱异步吞吐；
- Worker 的 timeout、cancel、lease renewal 不容易统一治理；
- Browser 与 HTTP 的资源模型不同，应独立控制。

因此 Crawl4AI 应作为 Fetch Adapter，而不是让 CrewAI Agent 成为抓取运行时。

## 6. Markdown 与图片处理上的问题

原项目直接使用 Crawl4AI 生成的 Markdown，并为了提取图片再次对 Markdown/HTML 做转换和解析。这在内容生成型应用中可接受，但对于可追溯知识库存在两个问题。

第一，第三方库生成的 Markdown 本身已经包含抽取和格式化策略，一旦库升级，同一个原始页面可能得到不同输出。如果直接把它视为最终事实，就难以判断变化来自网站还是 Renderer。

第二，从 Markdown 再转换回 HTML 查找图片是一个有损往返。生产系统更合理的方式是直接保存原始 DOM/Crawl Result 的媒体元数据，对资源 URL 做绝对化和规范化，再决定只保留原链接还是下载到对象存储。

因此建议坚持：

```text
Immutable Raw Artifact
 -> 多 Extractor Candidate
 -> Canonical IR
 -> Quality Gate
 -> Version
 -> Markdown Renderer
```

Crawl4AI Markdown 只能作为 Candidate 或诊断数据，最终 Markdown 从版本化 Canonical IR 确定性生成。

## 7. 多 Agent 流水线映射到知识库平台

可以把原文章的 6 个 Agent 抽象为以下生产阶段：

| 原项目阶段 | 知识库平台映射 | 是否允许 LLM 在热路径参与 |
|---|---|---|
| Research | Discovery + Fetch | 否，确定性 Provider/Fetcher 为主 |
| Outline | Parse Plan / Source Profile | 仅 Probe/规则生成可选 |
| Writer | Extract + Normalize | 否，确定性抽取与 IR 构建 |
| Critique | Quality Gate | 主体是规则/统计，可选异常诊断 |
| Revision | Repair / Alternate Extractor | 规则和替代 Extractor 为主 |
| Export | Version + Markdown + Projection | 否，确定性 Renderer |

最重要的改造是：**不能把“Writer/Revision”理解成让 LLM 改写抓到的第三方文章。** 知识库要求保真，Canonical 内容必须来自源页面事实。LLM 可以提出 selector、识别异常类型、总结诊断原因，但不能默默重写正文后再当源文存储。

## 8. Critique -> Revision 模式值得直接吸收

原项目有 Critique Agent 和 Revision Agent，这提示知识库方案应把质量审查与自动修复做成独立状态机，而不是“一次抽取成功就入库”。

建议状态：

```text
FETCHED
 -> EXTRACTED
 -> QUALITY_CHECK
    -> PASS -> VERSIONED -> RENDERED
    -> FAIL_RECOVERABLE -> REPAIR_PENDING
         -> ALTERNATE_EXTRACTOR
         -> DOM_REPAIR
         -> BROWSER_REFETCH（有恢复证据时）
         -> QUALITY_CHECK
    -> FAIL_UNKNOWN -> QUARANTINE
```

Quality Gate 至少检查：

- title 是否存在；
- 正文长度和段落数量；
- link density；
- boilerplate 比例；
- 登录墙、WAF、错误页特征；
- heading、code、table、list 结构保留；
- language；
- 与其他 Candidate 的一致性；
- 与历史版本相比是否出现异常截断；
- 页面类型与抽取结果是否冲突。

这样才能把“抓到了 HTTP 200”与“得到可用知识文档”区分开。

## 9. Repair 不应是无限 Agent 循环

文章里的修订过程可以由 Agent 根据 Critique 结果继续生成内容，但爬虫系统不能无界循环修复，否则会造成不可控成本和任务长尾。

应把 Repair 编译成有限状态机：

```text
Generic Extractor
 -> Deterministic Recipe
 -> DOM Repair + Generic
 -> Browser Fetch + same Extraction Portfolio
 -> Quarantine
```

每一个升级步骤都要记录：

- trigger reason；
- before/after quality score；
- latency；
- resource cost；
- 是否恢复为 PASS。

如果某 Source 的 Browser 或高成本 Repair 长期低收益，应触发 Source-scoped circuit breaker，而不是无限尝试。

## 10. Maxim AI 可观测性对平台的启示

原项目不仅记录总执行结果，还把 Trace、Tool Call、Event 分层。对于爬虫平台，这一点非常重要。

建议用 OpenTelemetry 形成统一层级：

```text
sync_run_id
 -> source_id
 -> discovery_provider_run
 -> url_observation
 -> classification
 -> route_decision
 -> fetch_task
 -> artifact
 -> extraction_attempt
 -> quality_result
 -> repair/fallback
 -> document_version
 -> markdown_projection
```

并在每个 span/事件上携带：

- source_release_id；
- task_id；
- url_id；
- artifact_id；
- extractor_release；
- quality score；
- retry reason；
- cost；
- trace_id。

这样 Web 管理台才能回答：为什么某篇文章没有入库、为什么用了 Browser、哪一个 Extractor 失败、修复是否有效、某次版本变化来自网站还是 Renderer 升级。

Maxim 本身并非必须依赖。生产方案使用 OpenTelemetry + Prometheus/Grafana + 结构化日志，更容易保持供应商中立。

## 11. 工具调用追踪应扩展为 Artifact Provenance

文章的 Tool Call Trace 解决“Agent 调了什么工具”。知识库还需要进一步回答“结果来自哪个不可变事实”。

建议每次抓取先生成 Artifact：

```text
artifact
- url_id
- fetch_kind
- status_code
- content_type
- etag
- last_modified
- fetched_at
- sha256
- object_key
- runtime_release
```

Extractor 只引用 Artifact ID，Document Version 再引用最终采用的 Artifact 与 Extraction Attempt。这样即使后续升级清洗器，也可以对旧 Artifact 离线 Replay，无须重新访问源站。

## 12. 原项目按 URL 返回结果的局限

Demo 最终按 URL 聚合抓取结果。但生产系统不能只用 URL 作为事实主键，因为 URL 可能发生：

- query 参数变化；
- canonical 变化；
- 301/308 迁移；
- 同 URL 内容更新；
- 同内容多个 URL；
- 路径重构后重发布。

建议拆分：

```text
source_id + normalized_url -> url_identity
url_identity + fetch time -> artifact
stable document identity -> document
document + semantic_hash -> document_version
```

同时保存 canonical/redirect 关系，而不是简单覆盖 URL。

## 13. 对增量同步的具体优化

从文章的阶段化工具模型可以进一步把增量同步拆成独立阶段，而不是整站重复执行抓取：

```text
Discovery Diff
 -> Candidate URL Inventory
 -> Known/New 判定
 -> Conditional Fetch
 -> Extract
 -> Quality
 -> Semantic Diff
 -> Version/Projection
```

建议为每个 Discovery Provider 保存 cursor/watermark/exhaustion evidence。已知 URL 复检时优先使用：

- ETag / If-None-Match；
- Last-Modified / If-Modified-Since；
- Sitemap lastmod；
- RSS updated；
- CMS revision/update time。

HTTP 304 只更新检查时间，不创建正文版本；正文 `semantic_hash` 不变也不应重复向量化。

删除不能因为一次 Sitemap 缺失就立即删除，应进入：

```text
ACTIVE -> SUSPECT_MISSING -> 多证据复检 -> REMOVED
```

## 14. Source Profile 应成为扩展性的核心

原项目通过 Tool 封装让 Agent 可以替换搜索和爬虫工具。对应到 1000 站平台，更重要的是把站点差异做成数据配置，而不是代码分支。

Source Profile 至少包含：

```yaml
scope:
  allowed_hosts: []
  include: []
  exclude: []

discovery:
  providers: [sitemap, rss, archive]

fetch:
  http_first: true
  browser_fallback: true
  concurrency_per_host: 2
  rate_limit: ...

routing:
  article_patterns: []
  index_patterns: []

extraction:
  portfolio: [structured, trafilatura, deterministic]
  recipe: ...

quality:
  thresholds: ...

incremental:
  freshness_class: WARM
  verify_known_urls: true
```

新增网站先 Probe，再生成 Candidate Profile；人工确认后发布 Source Release。站点模板改变必须产生新 Release，通过 Golden Replay/Canary 后切换。

## 15. Agent 最适合放在控制面，而不是数据面

原文证明了 Agent 很适合组织“研究—审查—修订”这类低吞吐、高语义任务。但用户目标是百万级文档的长期同步，数据面必须优先确定性、廉价和可重放。

推荐边界：

### 数据面

完全确定性：

- URL Discovery；
- HTTP Fetch；
- Browser Recipe；
- HTML/PDF 抽取；
- Quality Rule；
- IR；
- Version；
- Markdown；
- Index/Embedding 任务。

### 控制面 Agent

低频使用：

- 新 Source Probe 结果归纳；
- 根据失败样本建议 CSS/XPath/URL Pattern；
- QUARANTINE 聚类与根因分析；
- 对 Drift 生成 Candidate Recipe；
- 帮管理员解释某次失败链路；
- 对 Canary Diff 做语义辅助审查。

Agent 的建议不能直接修改 ACTIVE 配置，必须经过 Candidate Release -> Replay -> Canary -> Approval。

## 16. Web 管理功能应增加“阶段回放”能力

受原项目多 Agent 过程可观察性的启发，Web 后台不能只有“成功/失败列表”，而应按阶段展示一个 URL/Document 的完整执行链：

```text
发现
 -> 分类
 -> 路由
 -> Fetch
 -> Extract Candidate A/B/C
 -> Quality
 -> Repair/Fallback
 -> Canonical IR
 -> Version
 -> Markdown
 -> Search/Vector Projection
```

建议支持：

- Raw HTML / Browser DOM / Candidate / IR / Markdown Diff；
- 一键 Replay 某一个阶段；
- 用 Candidate Source Release 做只读试跑；
- 查看 Repair 前后 quality delta；
- 查看同一文档历史 Version Diff；
- 查看 DLQ/QUARANTINE 根因和批量修复建议；
- Source 级 Pipeline Trace 和成本瀑布图。

## 17. 最终结论

这篇文章不能直接作为 1000 站博客知识库的爬虫架构，因为它以 LLM Agent 为中心、运行规模较小、输出目标是生成内容，而不是长期保存第三方源事实。

但它有三点非常值得吸收：

1. **分阶段流水线**：每个阶段职责单一、输入输出明确，天然适合重试、回放和观测；
2. **Critique -> Revision**：对应 Quality Gate -> Repair/Fallback，使正文质量成为显式流程，而非抓取后的附带检查；
3. **Trace/Tool Observability**：应升级为整个平台的 Artifact Provenance 与阶段 Trace。

因此技术方案应继续坚持 HTTP First、Authoritative Discovery、Immutable Artifact、Extraction Portfolio、Canonical IR 和 Incremental Versioning，同时强化 **Quality-Repair 状态机、阶段级 Trace/Replay、控制面 Agent 与数据面确定性隔离**。这样既能利用 Agent 的语义能力，又不会让其成本、延迟和不确定性成为 1000+ 站点规模化同步的瓶颈。