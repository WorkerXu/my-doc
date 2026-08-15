# 使用 Crawl4AI 构建网页抓取 AI 智能体，并自动生成数据看板

## 1. 调研对象

- 编号：41
- 原文标题：Build AI Agents that Scrape the Web and Generate Dashboards with Crawl4AI
- 作者：Raphael Schols
- 原文地址：https://medium.com/data-science-collective/build-ai-agents-that-scrape-the-web-and-generate-dashboards-with-crawl4ai-1f9e5229e428
- 主题：Crawl4AI + AI Agent + 新闻情绪分析 + Streamlit 数据看板

原文公开摘要明确描述的目标是：用 Python、Crawl4AI 与 Streamlit 构建一个由 AI Agent 驱动的情绪分析 Dashboard，持续抓取新闻站点并把抓取结果转化为可视化分析。Medium 正文当前存在访问限制，因此本调研不虚构文章未公开的具体函数、Prompt 或目录结构；实现细节部分基于原文已确认的系统目标、Crawl4AI 官方接口语义以及 Streamlit 官方执行模型进行工程化推导。

## 2. 原文模式的核心价值

这类项目真正值得复用的不是“做一个情绪图表”，而是下面这条数据产品链路：

```text
Web Sources
  -> Crawl
  -> Clean Content
  -> AI Analysis
  -> Structured Facts / Scores
  -> Aggregation
  -> Dashboard
  -> Scheduled Refresh
```

它说明爬虫不应只把 Markdown 写进磁盘。对于长期知识库，抓取后的内容还可以继续派生出主题、情绪、实体、摘要、异常、趋势等分析结果，并在 Web 管理端形成运营视图。

但教程级实现和 1000+ 技术博客的生产系统有本质区别：教程可以把抓取、LLM 分析、DataFrame 与 Streamlit 页面写在一个 Python 进程里；生产平台必须拆分“事实同步”“AI 派生”“分析聚合”“可视化读模型”和“控制命令”。

## 3. Crawl4AI 抓取层的技术原理

Crawl4AI 官方接口以 `AsyncWebCrawler` 为核心。Browser 生命周期与单次抓取参数分离：

```text
BrowserConfig
    |
    v
AsyncWebCrawler
    |
    +-- CrawlerRunConfig(url A)
    +-- CrawlerRunConfig(url B)
    +-- CrawlerRunConfig(url C)
```

这比“每个 URL 新建浏览器”更适合持续任务，因为 Browser/Context 的启动成本可以摊薄，连接、资源和并发更容易治理。

对于多 URL，官方 `arun_many()` 支持批量和 streaming 两种模式，并可由 `MemoryAdaptiveDispatcher` 或 `SemaphoreDispatcher` 控制 Worker 内部并发。生产系统应把它理解为“单 Worker 执行器”，而不是全局调度器：

```text
平台 Scheduler / Durable Task
       |
       v
Worker lease N tasks
       |
       v
Crawl4AI arun_many(stream=True)
       |
       +--> URL A result -> persist -> ack A
       +--> URL B result -> persist -> ack B
       +--> URL C failed -> retry C
```

这样一个 URL 失败不会导致整个批次回滚，也不会因为 Worker 退出而丢失平台级任务状态。

## 4. Markdown 不是 AI 分析输入的唯一真相

Crawl4AI 可以生成 raw Markdown，也可以通过 Pruning/BM25 等过滤器生成更“聚焦”的 fit Markdown。对于文章中的新闻情绪分析场景，fit Markdown 很适合降低 LLM token 成本；但对于知识库，必须保留下面的层次：

```text
Immutable Snapshot
   -> Canonical IR
      -> Canonical Markdown
      -> Filtered/Fit Markdown
      -> AI Analysis Input
```

原因是情绪分析的输入过滤规则可能变化。如果把过滤后的内容直接当成最终知识正文，未来无法判断“模型判断变化”究竟来自模型升级、Prompt 升级、Filter 升级，还是文章真的变化。

因此 AI 分析必须记录：

```text
analysis_input_manifest
- document_version_id
- input_projection_id
- input_hash
- filter_release_id
- model_release_id
- prompt_release_id
- schema_release_id
- analysis_window
- created_at
```

## 5. AI Agent 的正确边界

原文把 Agent 用来把网页内容转化成情绪分析和 Dashboard。生产化时，不应让 Agent 变成“拥有无限抓取权限、可任意改变数据库状态”的超级进程。

推荐分成两个平面：

```text
Control/Data Plane
Source -> Discovery -> Fetch -> Snapshot -> Version

Insight Plane
Document Version -> Analysis Task -> LLM/Model -> Derived Artifact
```

Agent 只消费已经被平台接受的 `Document Version` 或冻结的 Manifest。它可以：

- 做 sentiment / stance / topic / entity；
- 对异常 Source 生成解释建议；
- 根据已有指标生成运营摘要；
- 生成图表说明或自然语言日报。

它不应该：

- 直接决定历史 Coverage 是否 COMPLETE；
- 绕过 Scheduler 自行无限访问站点；
- 覆盖 Canonical Markdown；
- 因模型失败把 Source Sync 标记失败；
- 直接执行暂停、删除、批量重抓等高风险管理操作。

高风险动作必须转换成强类型 Command，走 RBAC、幂等和 Audit。

## 6. 情绪分析如何建模才可重放

教程里最容易出现的实现是：

```python
sentiment = llm(article_text)
```

生产系统不能只保存最终的 `positive/negative`。至少需要：

```text
ai_analysis_record
- id
- document_version_id
- analysis_type             # SENTIMENT / TOPIC / ENTITY / SUMMARY
- analysis_release_id
- input_manifest_id
- output_schema_version
- label
- score
- confidence
- rationale_ref             # 可选，受隐私/成本策略限制
- raw_output_ref            # 可选
- token_usage
- latency_ms
- state
- created_at
```

`analysis_release_id` 固定：

```text
model + model_revision
prompt template
system policy
output schema
sampling parameters
normalization rule
post-processing rule
```

这样模型从 A 升级到 B 时，可以在相同 `Document Version` 上离线重放，比较 label drift，而不必重新抓网页。

## 7. 多篇新闻/博客的聚合不能直接平均

Dashboard 通常会画“正面/负面趋势”。如果直接对所有文章 score 求平均，会产生严重偏差：

- 同一事件被 20 家站点转载，会被重复放大；
- 高频发布 Source 会压过低频但高质量 Source；
- 一篇长文可能和一条短讯权重相同；
- 模型置信度不同；
- 新旧模型输出不可直接混合；
- 某天抓取失败会被误解成情绪下降。

推荐聚合键至少带：

```text
analysis_release_id
source_id / source_group
published_time_bucket
language
dedup_cluster_id
analysis_type
```

聚合前先做 Document Identity / near-duplicate 聚类，再决定 `per_document`、`per_source`、`per_event_cluster` 哪一种权重。

Dashboard 必须展示数据新鲜度和 Coverage，避免把“今天只抓到 20% 数据”显示成真实业务趋势。

## 8. Streamlit 执行模型对架构的影响

Streamlit 官方文档说明，交互会导致脚本重新从上到下执行；`st.cache_data`、`st.cache_resource` 和 `st.session_state` 用于减少重复计算和保存 UI 会话状态。

这意味着一个教程可以写：

```text
button -> crawl -> analyze -> dataframe -> chart
```

但生产上不能让每次页面刷新都触发：

- Crawl4AI 重新抓取；
- Browser 重新启动；
- LLM 再分析全部文章；
- 全量 Embedding；
- 全量数据库扫描。

正确模式：

```text
Backend Workers
  -> durable facts / projections
  -> dashboard read model
  -> Web API
  -> Streamlit/React/Vue render only
```

Streamlit 可以作为 PoC、研究看板、内部实验工具；主生产管理台更适合 React/Vue + FastAPI，并让后端负责状态、权限、长任务和事件流。

## 9. “自持续”系统不应依赖 UI 常驻循环

所谓 self-sustaining，应由 Scheduler 和持久状态实现，而不是：

```python
while True:
    scrape()
    analyze()
    sleep(3600)
```

推荐：

```text
Schedule Policy
 -> create INCREMENTAL Run
 -> Discovery/Change Signal
 -> Fetch changed documents
 -> emit DocumentVersionCreated
 -> enqueue Analysis Task
 -> update Analytics Read Model
 -> Web receives refresh event
```

关键点：

- 调度策略可版本化；
- Worker 随时可以重启；
- 每一步有 idempotency key；
- 重试不会重复产生分析记录；
- AI backlog 不阻塞抓取；
- UI 是否在线不影响后台同步。

## 10. Dashboard 应该消费 Read Model，而不是直接扫事实表

1000+ Source、百万 Document 后，如果每次打开首页都实时 JOIN URL Observation、Fetch Attempt、Document Version、Embedding、Audit，会把管理 UI 变成数据库压力源。

推荐增加运营读模型：

```text
ops_metric_point
- metric_release_id
- metric_name
- time_bucket
- source_id nullable
- dimensions_json
- value
- numerator nullable
- denominator nullable
- watermark
- computed_at
```

和 Source 快照：

```text
source_ops_snapshot
- source_id
- coverage_state
- last_success_at
- last_new_document_at
- freshness_lag_seconds
- queue_backlog
- fetch_success_rate
- accepted_rate
- browser_fallback_rate
- quality_reject_rate
- changed_rate
- current_blocker
- data_watermark
- computed_at
```

Dashboard 首页查聚合 Read Model；点击 Source/Run 后再 drill-down 到事实表。

## 11. Dashboard 的数据新鲜度必须显式表达

每个图表至少显示：

```text
window_start
window_end
data_watermark
computed_at
coverage_ratio
release_scope
```

否则会出现“图表最后更新时间 10:00，但实际只处理到 08:30”的假实时问题。

对于近实时运营，推荐：

```text
Transactional Outbox
 -> event consumer
 -> aggregate update
 -> SSE/WebSocket notification
 -> UI refetch affected query
```

SSE/WebSocket 只负责“告诉前端有变化”，最终数据仍从 API/Read Model 读取，避免把 websocket 消息本身当状态真相。

## 12. 指标基数治理

Prometheus 不适合把 `url_id`、`document_id`、`task_id` 等百万级身份作为 label，否则会导致高基数爆炸。

推荐：

- Prometheus：低基数运行指标，按 service、stage、route、outcome、source_group 等聚合；
- PostgreSQL/ClickHouse 类分析读模型：按 Source、Run、Release、URL drill-down；
- Object Storage：大体积 debug artifact；
- OpenTelemetry Trace：抽样追踪单次链路。

这对 1000 个站点尤为重要。

## 13. 推荐的 Web 运营看板

### 13.1 Fleet Overview

- Source 总数 / active / paused / blocked；
- FULL_BACKFILL completion / known gap；
- freshness SLA；
- queue backlog；
- changed documents；
- accepted / rejected；
- Browser fallback；
- AI analysis backlog；
- Index lag。

### 13.2 Source Health

- Provider yield；
- Feed/Sitemap/Archive 的 unique/overlap；
- 新 URL 率；
- 304/unchanged 比例；
- 429/5xx/timeout；
- Browser fallback；
- DOM drift / selector zero-match；
- 最近成功和 blocker。

### 13.3 Quality

- Markdown 长度/结构质量；
- code/table preservation；
- boilerplate ratio；
- duplicate cluster；
- low-quality quarantine；
- Extractor/Schema Release 对比。

### 13.4 Cost & Capacity

- requests / bytes；
- Browser seconds；
- object storage growth；
- embedding tokens；
- LLM tokens；
- cost per accepted/changed document；
- Worker saturation。

### 13.5 AI Insight

- sentiment/topic/entity 趋势；
- model/prompt release；
- confidence distribution；
- analysis coverage；
- failure/backlog；
- release-to-release drift。

AI Insight 必须明确标注为 Derived Projection，不能和 Source Truth 混为一谈。

## 14. 管理操作与图表必须分离

图表是 Observation；“重抓”“暂停”“重新分析”“切换 Release”是 Command。

推荐：

```text
Dashboard click
 -> POST typed command
 -> idempotency key
 -> RBAC / policy check
 -> transaction + Audit + Outbox
 -> async execution
 -> command status
 -> dashboard refresh
```

不能让前端直接更新 `task.state`、`source.status` 或 Redis key。

## 15. Agent 生成 Dashboard 的安全边界

如果未来允许 Agent 自动生成分析卡片或图表，应限制其能力：

1. Agent 只调用只读 Analytics API；
2. SQL 必须走受限 query service 或预定义 semantic layer；
3. 限制最大扫描窗口、行数、CPU/时间；
4. 禁止读取 Secret、raw credential、任意内部表；
5. 输出保存 `dashboard_artifact_release + query_manifest`；
6. 高风险管理动作仍需显式 Command API；
7. Agent 生成的结论展示数据窗口、样本量、Release 和引用。

“自然语言生成图表”可以是能力层，但不能替代管理台的信息架构和权限模型。

## 16. 失败模型

至少区分：

```text
CRAWL_FAILED
CONTENT_EMPTY
CONTENT_QUALITY_LOW
ANALYSIS_INPUT_NOT_READY
ANALYSIS_TIMEOUT
ANALYSIS_MODEL_ERROR
ANALYSIS_SCHEMA_INVALID
ANALYSIS_LOW_CONFIDENCE
AGGREGATION_STALE
DASHBOARD_READ_MODEL_LAG
DASHBOARD_QUERY_TIMEOUT
COMMAND_REJECTED
COMMAND_CONFLICT
```

`ANALYSIS_*` 失败不得反向把已经 READY 的 Canonical Document 标记为失败。

## 17. 幂等与身份设计

分析任务：

```text
analysis_task_key = hash(
  document_version_id
  + analysis_release_id
  + input_manifest_hash
)
```

聚合任务：

```text
aggregate_key = hash(
  metric_release_id
  + time_bucket
  + dimension_set
  + input_watermark
)
```

这样重试不会生成重复分析结果，Dashboard 也能知道某个时间桶对应哪批输入。

## 18. 对 1000+ 技术博客方案的直接优化

本次调研建议把以下能力正式纳入 `博客知识库技术方案.md`：

1. 新增 **Insight/Analytics Plane**：LLM sentiment/topic/summary 等是可重建派生物，与 Source Sync 解耦；
2. 新增 **Ops Analytics Read Model**：Dashboard 默认读聚合快照，不实时扫描高增长事实表；
3. 新增 **Metric/Dashboard Release**：指标公式、维度、窗口、聚合和 Dashboard schema 版本化；
4. 新增 **Watermark/Freshness**：所有趋势图显示处理水位、数据窗口和 Coverage；
5. 新增 **Dashboard Event Refresh**：Outbox 事件驱动聚合，SSE/WebSocket 仅做失效通知；
6. 新增 **AI Analysis Manifest**：固定 Document Version、输入 Projection、Model/Prompt/Schema Release；
7. 新增 **AI Coverage/Backlog**：AI 派生失败不阻塞抓取，但在 Web 中可观察；
8. 增加 **Prometheus 高基数约束**：URL/Document/Task 级明细进入数据库/分析读模型，不进入 metrics label；
9. 明确 **Dashboard Observation 与 Command 分离**：操作统一通过强类型、幂等、RBAC、Audit API；
10. 明确 **Streamlit 定位**：PoC/实验可以使用，生产管理台不在页面 rerun 中直接执行 Crawl/LLM 长任务。

## 19. 推荐测试

### 19.1 抓取/分析解耦

让 LLM 服务完全不可用，验证 Source Sync、Snapshot、Version、Markdown 仍可正常完成。

### 19.2 Analysis Replay

同一 Document Version：

- Analysis Release A；
- Analysis Release B；
- Filter Release A/B；

必须生成独立且可比较的结果，不重新访问源站。

### 19.3 Dashboard Watermark

人为让 aggregate worker 落后，UI 必须明确显示旧 watermark，不能展示为“实时”。

### 19.4 Dashboard Consistency

随机抽取 Source/Run，把 Read Model 聚合值与事实表离线重算对比。

### 19.5 高基数测试

模拟百万 URL，确认 Prometheus label cardinality 不随 URL 数线性增长。

### 19.6 UI 重跑测试

重复刷新 Streamlit/React 页面不能产生新 Crawl、LLM Task 或重复 Command。

### 19.7 Command Audit

从 Dashboard 触发 retry/pause/reprocess，必须具备 requester、idempotency key、RBAC decision、command state、Audit Log。

## 20. 结论

该文章最有价值的启示，是把“网页抓取”继续向“数据产品”延伸：内容被采集后，可以经过 AI 分析形成持续更新的运营/业务看板。

对于本项目，不能直接照搬“Crawl4AI + Agent + Streamlit 一体化脚本”，而应保留其产品闭环，同时把工程边界升级为：

```text
Durable Crawl Truth
    -> Versioned Content Projection
    -> Versioned AI Insight
    -> Incremental Analytics Read Model
    -> Web Dashboard
    -> Audited Command API
```

这使系统既能满足 1000+ 博客的全量历史回灌和增量同步，也能逐步增加情绪、主题、趋势、质量、成本和异常诊断等智能运营能力，而不会让 LLM 或 Dashboard 变成抓取事实链路的新单点故障。

## 21. 参考资料

- 原文：https://medium.com/data-science-collective/build-ai-agents-that-scrape-the-web-and-generate-dashboards-with-crawl4ai-1f9e5229e428
- Crawl4AI AsyncWebCrawler：https://docs.crawl4ai.com/api/async-webcrawler/
- Crawl4AI arun_many：https://docs.crawl4ai.com/api/arun_many/
- Crawl4AI Multi-URL Crawling：https://docs.crawl4ai.com/advanced/multi-url-crawling/
- Crawl4AI Markdown Generation：https://docs.crawl4ai.com/core/markdown-generation/
- Streamlit Session State：https://docs.streamlit.io/develop/api-reference/caching-and-state/st.session_state
- Streamlit Caching：https://docs.streamlit.io/develop/concepts/architecture/caching
