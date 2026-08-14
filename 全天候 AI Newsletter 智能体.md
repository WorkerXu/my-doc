# 全天候 AI Newsletter 智能体

来源项目：<https://github.com/Aparnap2/newsletter-agent>

## 1. 项目定位

该项目实现的是一条“持续发现技术资讯 → 抓取 → 多阶段 AI 加工 → 邮件投递 → Web 管理”的轻量流水线。它不是面向百万级历史文档的生产级爬虫平台，但组件边界与数据流非常适合作为博客知识库系统的反例和参考：一方面能看到 Crawl4AI、RSS、搜索、LangGraph、调度器、FastAPI、邮件投递如何组合成完整闭环；另一方面也能明确哪些单机、进程内做法在 1000 个技术博客、长期增量同步场景下必须替换为持久化、可恢复的实现。

项目主要模块：

```text
scheduler/newsletter_scheduler.py
        |
        v
crawlers/web_crawler.py
        |
        v
agents/newsletter_agents.py -> agents/llm_client.py
        |
        v
email_service/gmail_client.py

api/fastapi_server.py 负责人工触发、状态、历史、调度开关和运行时配置
```

核心链路是：

```text
定时触发
  -> 按 topic 搜索候选 URL
  -> Crawl4AI 抓列表页/页面
  -> HTML/Markdown 规则提取文章卡片
  -> 关键词相关性排序
  -> LangGraph: Researcher -> Analyst -> Opinion Writer -> Editor
  -> Gmail 投递
```

## 2. 抓取层实现细节

### 2.1 Crawl4AI 是主路径，httpx 是异常回退

`WebCrawler.crawl_with_crawl4ai()` 每处理一个 URL 都创建一次 `BrowserConfig(headless=True)` 和 `AsyncWebCrawler`，并使用 `CrawlerRunConfig` 抓取页面。配置里显式使用 `CacheMode.BYPASS`，设置低字数阈值和页面超时。成功后优先读取 `cleaned_html`，否则读取 Markdown，再用自定义规则提取条目。

如果 Crawl4AI 抛异常，代码才回退到 `httpx.AsyncClient` 请求原始 HTML，并继续走 BeautifulSoup 抽取。

这个顺序对资讯 demo 容易理解，但对大规模博客知识库并不合适：

1. Browser/Crawl4AI 初始化成本远高于普通 HTTP；大量静态技术博客会浪费 CPU、内存和浏览器进程。
2. 每 URL 新建 crawler，无法充分复用浏览器进程、连接池、DNS/TLS 会话和 context。
3. `CacheMode.BYPASS` 使每次都重新抓，无法天然利用缓存，更没有 ETag / Last-Modified 条件请求语义。
4. HTTP 仅在 Browser 路径异常时才使用，实际上应该反过来：HTTP-first，检测到 JS 渲染证据、正文缺失或 SPA 列表后再升级 Browser。

因此，知识库平台应把 Crawl4AI 定位为“浏览器渲染和 Markdown candidate 引擎”，而不是默认网络请求层。

### 2.2 搜索发现与内容抓取被混在同一层

项目的 `search_news_urls(topic)` 直接抓 DuckDuckGo HTML，并把搜索范围硬编码到 TechCrunch、The Verge、VentureBeat、Wired 等域名。搜索结果只保留少量 URL，然后 `fetch_live_data()` 并发抓这些页面。

这说明一个重要架构问题：**URL Discovery 与 Article Fetch 必须分离**。

搜索页、RSS、Sitemap、归档页、分类页、站内列表页都应该只产生“候选 URL + 发现证据”，不能直接当最终文章正文。生产系统应形成两段式：

```text
Discovery Provider
  -> DiscoveryRecord(url, title_hint, published_hint, evidence)
  -> URL Admission / Identity
  -> durable frontier
  -> Article Fetch
  -> Snapshot
  -> Extraction
```

这样才能做到同一个文章 URL 无论来自 RSS、Sitemap、Archive、HTML Frontier 还是搜索聚合源，都落到同一个 URL Identity / Document Identity 上，同时保留各 Provider 的覆盖证据。

### 2.3 HTML 提取策略是“通用 selector 首个命中”

项目依次尝试 `article`、`.post`、`.article`、`.story`、`.entry`、若干常见 class，最后甚至尝试 `h2`、`h3`。一旦某组 selector 抽到内容就停止继续尝试。条目内再找标题、链接和摘要。

这种策略适合“新闻列表卡片快速抽取”，但存在几个生产风险：

- 页面里的推荐、热门、侧栏、Footer 也可能匹配 `.article` / `h2`；
- 首个 selector 有结果不代表结果质量最好；
- 列表页条目通常只有摘要，不是最终文章；
- selector 失效后不一定报错，可能静默抽到错误区域；
- 没有模板版本、黄金样本和漂移检测。

生产系统应把 selector 抽取结果当 candidate，并通过 Content Fitness、字段置信度、正文长度分布、链接密度、模板指纹和 Golden Sample 做质量验证。如果列表页只产生文章链接，应继续进行二次 Article Fetch，而不是把卡片摘要作为最终知识正文。

### 2.4 Markdown 提取同样偏向“列表页识别”

Markdown 解析通过标题行识别条目，把标题下面的若干文本拼成 summary。这里并没有保留 Markdown 原始结构，也没有把代码块、表格、图片、引用、真实链接关系构造成统一 IR。

面向知识库时，应把 Crawl4AI Markdown 视为 extraction candidate，而不是最终真相。最终需要 Canonical Document IR，再由 IR 渲染 Markdown。这样能避免未来更换清洗规则时从 Markdown 反推结构。

### 2.5 RSS 路径暴露了同步 I/O 与可靠性问题

RSS 使用 `feedparser.parse()` 遍历条目，并对每条链接调用 `_extract_content_from_url()`。该函数内部使用同步 `requests.get()` 和 BeautifulSoup，并且只保留前 1000 字符。

当前文件没有导入 `requests`，因此调用该函数时会触发 `NameError`，再被函数自身的 `except` 捕获并返回空字符串。这个问题不会导致整个任务崩溃，而会形成“RSS 条目存在，但正文默默为空”的静默退化。

这类问题说明：

1. 关键字段不能依赖“异常后返回空字符串”继续运行；
2. 每个阶段要显式区分 `SUCCESS / EMPTY / INVALID / RETRYABLE_ERROR / TERMINAL_ERROR`；
3. 质量门禁必须独立于抓取函数本身；
4. 同步网络调用不能混入大量 async worker 热路径；
5. 全文知识库不能把正文截成固定 1000 字符。

### 2.6 并发方式只能用于小规模 demo

项目用 `asyncio.gather(*crawl_tasks, return_exceptions=True)` 并发抓取搜索结果。当前搜索结果上限很小，所以风险有限；但将同样模式扩大到 1000 个站点会产生典型问题：

- 瞬间创建过多协程；
- 无跨 Worker 的域名级并发限制；
- 无公平调度；
- 无 backpressure；
- 进程退出后任务集合丢失；
- 无 lease / heartbeat / dead-letter；
- 无持久化重试计数。

因此，大规模系统应使用数据库 durable frontier + Redis Streams/Kafka 类传输层，Worker 只租约式领取有限任务，并配合 global/domain/site 三层额度。

## 3. 文章过滤与元数据问题

### 3.1 相关性排序

项目把 topic 分词后，对标题命中加 2 分、摘要命中加 1 分；如果完全没有命中，再用一组 AI/科技关键词做兜底。最后按分数排序取前 N。

优点是成本低、解释简单，适合 Newsletter 选文；缺点是不能用于 FULL_BACKFILL 的覆盖裁剪。对于知识库：

- FULL_BACKFILL 不能因为“不相关”就丢历史文章；
- 相关性只能用于抓取优先级或派生 Digest 的候选排序；
- Newsletter/报告可进一步使用 embedding、主题簇、时间衰减、MMR/max-min diversity；
- 选文必须生成 Selection Manifest，记录候选集、打分、过滤原因和最终选择。

### 3.2 发布时间被错误合成

HTML/Markdown 抽取路径把 `published_date` 直接设为当前日期。这会把“抓取时间”伪装成“文章发布时间”，对于历史知识库是严重数据污染。

正确设计必须区分：

```text
observed_at       本系统看到它的时间
fetched_at        网络抓取时间
published_at      来源声明的发布时间，允许未知
updated_at        来源声明的更新时间，允许未知
published_hint    Discovery Provider 提供的候选时间
```

并为每个元数据字段保留 `value + source + confidence + extractor_release`。无法确定发布时间时宁可为 NULL，也不能填 `now()`。

### 3.3 跨 topic 重复

同一文章可能同时被多个 topic 搜到，项目会把全部结果直接汇总，没有全局 canonical URL 去重。生产系统必须在 URL Identity 层做规范化和唯一约束，派生 AI 选文时再做 document_version 级去重。

## 4. LangGraph 多阶段 AI 流程

项目最值得保留的设计是 `NewsletterAgents` 使用 LangGraph 定义线性状态图：

```text
researcher
  -> analyst
  -> opinion_writer
  -> editor
  -> END
```

状态包含：

```text
raw_articles
research_summary
key_insights
opinion_analysis
final_newsletter
```

### 4.1 技术原理

这实际上是把原本一个超长 Prompt 拆成多个职责单一的状态节点：

- Researcher：分类、选重要故事、找趋势、提取事实；
- Analyst：根据研究摘要分析战略影响、市场影响、技术趋势、机会风险；
- Opinion Writer：在事实与分析之上生成编辑观点；
- Editor：把前面阶段的内容整合为最终 Newsletter。

图式工作流的优势是：

1. 每个节点职责清楚，可替换模型；
2. 以后可以增加分支、人工审核、条件节点；
3. 中间结果天然适合持久化；
4. 可针对单阶段重试，不必整个报告重跑；
5. 同一 accepted document 集可以重放不同 Prompt/模型版本。

### 4.2 当前实现的可靠性不足

项目虽然用了 LangGraph，但仍然把整个图当进程内同步函数执行：

- 中间 state 没有落数据库或对象存储；
- 任何阶段失败后只返回空文本或原始 state；
- Agent 调用的是 `generate_completion()`，并没有使用已经实现的 `generate_with_retry()`；
- LLM 返回自由文本，没有结构化 schema；
- 没有保存模型版本、Prompt hash、token、cost、输入文档清单；
- 下游只看上一阶段生成的自由文本，容易丢失 source citation；
- 无 stage checkpoint，进程崩溃后只能整体重跑。

知识库方案应该保留“DAG/Graph”思想，但把它生产化为 Versioned AI Workflow：

```text
ai_workflow_release
ai_stage_release
ai_run
ai_stage_execution
ai_input_manifest
ai_output_artifact
citation_binding
```

每个 Stage 必须保存：

```text
workflow_release_id
stage_release_id
input_document_version_ids
input_artifact_ids
prompt_release_id
model_release_id
structured_schema_release_id
started_at/finished_at
status
retry_count
token_usage/cost
output_artifact_id
error_code
```

这样才能实现失败续跑、阶段重试、模型 A/B、离线 replay 和完整 lineage。

### 4.3 Opinion 阶段应该可配置

“观点”适合 Newsletter，但不适合所有知识库报告。生产系统中应把 `OPINION` 定义成可选节点：

- `FACTUAL_DIGEST`：Research -> Analysis -> Editor；
- `EDITORIAL_DIGEST`：Research -> Analysis -> Opinion -> Editor；
- `TECH_TREND_REPORT`：Cluster -> Evidence -> Analysis -> Editor；
- `SITE_CHANGE_REPORT`：Diff -> Evidence -> Summary。

同一底层文档集合可以由不同 workflow release 生成不同类型的派生产物。

## 5. 调度器实现与生产问题

`NewsletterScheduler` 使用 Python `schedule` 库，在进程中登记每天的时间点，然后 `while self.is_running` 循环，每分钟 `schedule.run_pending()`，并 `time.sleep(60)`。

这种实现适合单进程 demo，但不是持久化调度：

- 进程重启后计划全部丢失；
- 多副本会重复调度；
- 没有 leader election；
- 没有 misfire/recovery 语义；
- 没有数据库里的 next_run / last_run / schedule_version；
- 无法对“同一时间窗是否已生成/已发送”做幂等判断。

更重要的是，FastAPI 中 `/schedule/start` 通过 `asyncio.create_task(run_scheduler())` 启动任务，而 `run_scheduler()` 随即调用同步的 `scheduler.start_scheduler()`。后者内部是 `while + time.sleep(60)`，会占住当前 event loop 所在线程，可能把同一 Uvicorn Worker 的其他异步请求一起阻塞。

生产方案应该使用 HA Scheduler：

1. schedule 作为数据库实体持久化；
2. Scheduler 只负责生成 due command；
3. 通过 PostgreSQL advisory lock / leader lease 避免多副本重复；
4. 创建 `run` 后经 Transactional Outbox 投递任务；
5. 每个 run 有唯一幂等键；
6. missed schedule 可配置 `SKIP / CATCH_UP_ONCE / CATCH_UP_ALL`；
7. Web API 只写 command，不直接在请求进程里跑长任务。

## 6. FastAPI Web 管理实现

项目提供了 `/status`、`/generate`、`/history`、`/schedule/start`、`/schedule/stop`、`/config`、`/health` 等接口，并用一个简单 HTML 首页展示状态。

这个接口集合非常适合作为知识库 Web 管理面的最小功能参考，但当前状态完全依赖内存：

```text
newsletter_history = []
system_status = {...}
global scheduler instance
```

FastAPI `BackgroundTasks` 也只是进程内后台执行。如果进程崩溃、滚动发布、Pod 被驱逐，任务和状态都可能丢失。多 Uvicorn/Gunicorn Worker 时，每个 Worker 还会拥有不同的内存状态。

`/config` 修改的也是进程内对象，并明确说明重启后不保留。多副本下不同 Worker 看到的配置也可能不一致。

`/health` 无论数据库、爬虫、LLM、邮件服务是否可用都固定返回 healthy，这只能做进程存活探针，不能作为 readiness。

知识库 Web Control Plane 应设计为：

```text
POST /runs
  -> DB transaction: create command/run + outbox
  -> 立即返回 run_id

GET /runs/{id}
  -> 从 PostgreSQL 读取真实状态

POST /schedules
  -> 写 versioned schedule

GET /health/live
  -> 仅进程存活

GET /health/ready
  -> 检查 PG/Redis/S3 等核心依赖

GET /health/components
  -> Browser/Crawl4AI/LLM/Search/Email 等能力健康矩阵
```

Web 页面刷新、API 进程重启都不应改变后台任务事实。

## 7. LLM 客户端实现问题

OpenRouter 客户端使用同步 `requests.post()` 调用 chat completions。它有一个 `generate_with_retry()`，固定等待 2 秒重试，但 Agent 实际没有调用这个带重试版本。

生产上需要进一步升级：

- LLM 调用放到独立 Worker Pool，不能阻塞抓取 Worker；
- async client 或受控线程池；
- Retry-After、指数退避、抖动、限流和熔断；
- provider/model fallback 由 policy 决定；
- Prompt、Model、Schema 版本化；
- 输入必须引用 accepted document_version；
- 结构化输出做 schema validation；
- 所有观点/摘要保留 source citation；
- cost/token/latency 记录到 stage execution；
- LLM 故障不能把抓取 Run 改成失败。

## 8. 对博客知识库方案的可复用部分

这个项目值得直接吸收的不是它的单机调度代码，而是以下产品与架构思想：

### 8.1 “抓取事实”和“内容消费”解耦

Newsletter 是知识库的下游消费方式之一。知识库同步成功后，可以继续生成：

- 每日技术 Digest；
- 每周趋势；
- 站点更新摘要；
- 主题 Newsletter；
- 研究报告；
- 邮件/Slack/飞书推送。

但这些都不能成为源站抓取主链路的同步依赖。

### 8.2 Versioned AI Workflow

保留 LangGraph/DAG 的分阶段思路，将其升级成可持久化、可回放、可审计的 `AI Workflow Release`。Research/Analysis/Opinion/Editor 每阶段都形成独立 Artifact 和 lineage。

### 8.3 Web 管理面应覆盖“人工触发 + 调度 + 历史 + 配置”

项目提供的端点说明运营人员确实需要直接查看状态和触发任务。大规模系统应把这些功能拓展成真正的 Control Plane，同时禁止使用 FastAPI BackgroundTasks、全局内存列表或进程内 scheduler 充当业务状态真相。

### 8.4 Discovery 与 Fetch 必须二阶段化

项目把搜索结果页、列表页抽取和文章内容混在一个类里。知识库应明确：RSS/Sitemap/Search/Archive/HTML Listing 只发现 URL；最终正文必须重新抓 origin article URL，并保存 Snapshot。

### 8.5 元数据必须有来源语义

抓取时间不能伪装发布时间；发现 hint、原站 metadata、JSON-LD、OpenGraph、HTML time、Feed pubDate 都应作为 evidence 合并，并保留解析来源和置信度。

## 9. 针对 1000 站点场景需要补齐的能力

该项目没有覆盖但博客知识库必须具备：

1. FULL_BACKFILL 的历史覆盖证明；
2. Sitemap/RSS/API/Archive/HTML Frontier 等多 Provider 对账；
3. ETag、Last-Modified、Feed GUID、cursor、lastmod 和 body/IR hash 增量；
4. durable frontier、lease、heartbeat、dead-letter；
5. 跨 Worker 的 domain quota、公平调度和 backpressure；
6. Snapshot/IR/Markdown 的不可变版本与 lineage；
7. canonical、redirect、slug rename、站点迁移后的 Document Identity；
8. 抽取模板漂移检测；
9. 数据库持久化的 Web 状态和操作审计；
10. 版本化 Adapter/Provider/Extractor/AI Workflow；
11. 搜索、版本 diff、离线 reprocess；
12. 依赖感知健康检查与告警；
13. Notification 的幂等投递、重试和 dead-letter；
14. LLM 派生结果的结构化输出、引用和成本治理。

## 10. 结论

该项目适合作为“小而完整”的 AI 内容流水线参考。它验证了 Crawl4AI + 多来源发现 + LangGraph 多阶段加工 + FastAPI 管理 + 定时投递这一产品闭环，但其 Browser-first、每 URL 新建 crawler、CacheMode.BYPASS、列表页直接充当文章、合成发布时间、进程内 schedule、FastAPI BackgroundTasks、内存 history/config、阻塞式 scheduler、自由文本 Agent state 和缺少持久化 checkpoint 等实现，不能直接放大到 1000 站点生产环境。

对博客知识库最有价值的优化是：在既有抓取平台之外补齐一个**持久化、版本化的 Derived AI Workflow Plane**，把 Research -> Analysis -> Optional Opinion -> Editor 变成独立可重放 DAG；同时进一步明确 Discovery 与 Article Fetch 二阶段模型、元数据 provenance、Control Plane 长任务命令化、依赖感知健康检查以及 Digest/Delivery 的时间窗幂等。这样既保留项目的产品闭环价值，又不会把单机 demo 的运行时状态误当作生产事实。