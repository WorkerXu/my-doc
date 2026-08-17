# 全天候 AI Newsletter 智能体

来源项目：<https://github.com/Aparnap2/newsletter-agent>

调研基线：项目默认分支 `canary`。本文件只记录实现细节、技术原理、风险和对博客知识库方案的可复用结论。

## 1. 项目定位与整体链路

该项目是一条“小而完整”的自动资讯流水线：发现 AI/技术新闻，抓取页面，经过多阶段 LLM Agent 加工成 Newsletter，再通过 Gmail 投递，并提供 FastAPI 管理接口和进程内定时任务。

核心模块：

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

api/fastapi_server.py
  -> 人工触发
  -> 状态/历史
  -> 调度启停
  -> 运行时配置
```

主链路：

```text
schedule / API trigger
  -> 按 topic 搜索候选 URL
  -> Crawl4AI 抓取
  -> HTML/Markdown 规则提取
  -> 关键词相关性过滤
  -> LangGraph: Researcher -> Analyst -> Opinion Writer -> Editor
  -> Gmail API
```

它适合验证“抓取 + AI 加工 + Web 管理 + 投递”的产品闭环，但不具备 1000 个技术博客历史全量抓取所需的覆盖证明、durable frontier、持久化 checkpoint、文档版本、质量审计和多副本一致性。

## 2. 抓取层实现与原理

### 2.1 Browser-first，而不是 HTTP-first

`WebCrawler.crawl_with_crawl4ai()` 对每个 URL 都创建 `BrowserConfig(headless=True)` 和新的 `AsyncWebCrawler`，`CrawlerRunConfig` 使用：

```text
CacheMode.BYPASS
word_count_threshold=10
page_timeout=60000
```

成功后优先处理 `cleaned_html`，否则处理 Markdown。只有 Crawl4AI 抛异常时，才回退到 `httpx.AsyncClient`。

这个顺序适合 demo，但对 1000 站点不经济：

1. 大量技术博客是静态 HTML，浏览器启动和 JS runtime 成本远高于普通 HTTP；
2. 每 URL 新建 crawler，连接池、浏览器进程和 context 都不能充分复用；
3. `CacheMode.BYPASS` 意味着默认绕过缓存，没有 ETag/Last-Modified 条件请求语义；
4. Browser 成功不等于正文有效，SPA 壳页、挑战页、登录页仍可能返回“成功”。

生产方案应改成：

```text
HTTP Fetch
  -> Content Fitness / JS evidence
  -> 静态正文足够：直接进入 Extraction
  -> 动态证据成立：升级 Browser/Crawl4AI
```

Crawl4AI 更适合作为昂贵的渲染/Extraction Candidate 引擎，而不是所有请求的默认网络层。

### 2.2 Discovery 与 Article Fetch 混在一起

`search_news_urls(topic)` 抓 DuckDuckGo HTML，搜索范围硬编码到 TechCrunch、The Verge、VentureBeat、Wired 等站点，然后只取少量 URL。`fetch_live_data()` 随后直接抓这些 URL，并在抓到的页面中继续识别“文章条目”。

这会混淆两个不同问题：

- Discovery：从 Search/RSS/Sitemap/Archive/Listing 得到候选 URL；
- Article Fetch：对候选文章 URL 获取可存证的 origin Snapshot。

知识库必须二阶段化：

```text
Discovery Provider
  -> DiscoveryRecord
  -> URL Admission / Identity
  -> Durable Frontier
  -> Origin Article Fetch
  -> Snapshot
  -> Extraction
```

搜索结果、Feed 摘要、归档卡片只能提供 discovery evidence 和 metadata hint，不能直接成为 accepted 正文。

### 2.3 通用 selector 的“首个命中即成功”会静默抓错

HTML 提取依次尝试：

```text
article
.post
.article
.story
.entry
[data-testid="post"]
...
h2
h3
```

只要某组 selector 抽到内容就停止继续尝试。问题是：

- 推荐区、侧栏、Footer、热门模块也可能命中；
- `h2/h3` 在正文页中常常只是章节标题；
- 一个 origin article 页面可能被错误拆成多个“文章”；
- selector 失效时往往不会报错，而是抽到错误区域；
- 没有模板版本、Golden Sample、DOM 指纹和漂移监控。

所以生产系统应允许多个 Extraction Candidate 并存，用 Content Fitness、正文/链接密度、标题一致性、DOM/template fingerprint、metadata 完整度和站点 Golden Fixture 评分，不能把“第一个 selector 有结果”当成成功证据。

### 2.4 Markdown 只是候选，不是知识真相

项目的 Markdown 路径按 `#` 标题切分，再把标题后的少量文本拼成 summary。这种做法会丢失代码块、表格、引用、图片、数学公式和真实链接结构。

知识库应先构造 Canonical Document IR：

```text
metadata
blocks[]
links[]
images[]
attachments[]
source_snapshot_id
provenance
```

最终 Markdown 只是 IR 的可重建 Projection。未来修改清洗规则时，应从 Snapshot/IR 离线重放，而不是从旧 Markdown 反推结构。

### 2.5 RSS 路径存在可验证的静默退化

`crawl_rss_feeds()` 会调用 `_extract_content_from_url()`，该函数内部使用 `requests.get()`，但 `crawlers/web_crawler.py` 没有导入 `requests`。因此这条路径会触发 `NameError`，随后被函数自己的宽泛 `except Exception` 捕获并返回空字符串。

结果不是任务崩溃，而是“RSS 条目存在、正文为空”的假成功。这比显式失败更危险，因为它会污染下游 AI。

此外，该函数即使成功，也只保留正文前 1000 个字符，不满足全文知识库要求。

这里暴露出生产系统必须具有的**阶段结果契约**：

```text
SUCCEEDED
EMPTY
INVALID
RETRYABLE_ERROR
TERMINAL_ERROR
CANCELLED
```

抓取函数不能通过 `return ""` 把错误伪装成正常结果；Stage Finalizer 必须根据 Artifact、质量结果和任务状态决定是否真正成功。

### 2.6 `asyncio.gather()` 只能覆盖小规模并发

项目把少量搜索 URL 一次性放入 `asyncio.gather(..., return_exceptions=True)`。在当前最多几个 URL 的场景问题不大，但若直接放大到百万 URL，会出现：

- 瞬时创建大量协程；
- 无跨进程/跨 Worker 的 domain quota；
- 无公平调度和 backpressure；
- 进程退出后任务集合丢失；
- 无 lease、heartbeat、retry schedule、dead-letter；
- 大站点可能长期占满资源。

生产系统应使用数据库 durable frontier，Worker 按 lease 领取有限批量，Redis Streams 只做运输；全局、站点/域名、Browser/LLM/OCR route 分别限流。

## 3. 元数据、去重与选文

### 3.1 发布时间被合成为当前时间

HTML/Markdown 提取路径直接把 `published_date` 设为 `datetime.now()`。这会把抓取时间伪装成来源发布时间，对历史知识库属于数据污染。

必须区分：

```text
observed_at       系统首次/最近观察时间
fetched_at        网络请求时间
published_at      来源声明的发布时间，可 NULL
updated_at        来源声明的更新时间，可 NULL
published_at_hint Discovery Provider 提供的候选时间
```

字段应带：

```text
value + source + confidence + extractor_release
```

未知发布时间宁可为 NULL，也不能填当前时间。

### 3.2 相关性过滤不能影响历史覆盖

项目按 topic 关键词给标题/摘要打分，并只保留少量结果。这适合 Newsletter 选文，但不能用于 FULL_BACKFILL。

生产方案应把相关性放到 Derived AI/Selection 层：

```text
Accepted Document Versions
  -> Candidate Set
  -> keyword / embedding / recency / site weight
  -> clustering + MMR diversity
  -> Selection Manifest
```

FULL_BACKFILL 的合法历史 URL 不能因为“相关性低”被静默丢弃。

### 3.3 跨 topic 需要全局 identity

同一 URL 可能被多个 topic 重复搜索到。项目只是简单汇总，没有全局 canonical URL 去重。

生产系统应在 URL Identity 层建立规范化和唯一约束，再在 Document Identity 层处理 redirect、canonical、slug rename、镜像和站点迁移；派生 Newsletter 再以 `document_version_id` 去重。

## 4. LangGraph 多阶段 AI 工作流

`NewsletterAgents` 使用 LangGraph 定义线性 DAG：

```text
researcher
  -> analyst
  -> opinion_writer
  -> editor
  -> END
```

State：

```text
raw_articles
research_summary
key_insights
opinion_analysis
final_newsletter
```

### 4.1 值得保留的技术原理

把一个超长 Prompt 拆成多个职责单一节点是正确方向：

- Researcher：分类、提取重要故事、趋势和事实；
- Analyst：分析战略、市场、技术方向和风险；
- Opinion Writer：生成编辑观点；
- Editor：组织最终 Newsletter。

这种 DAG 天然支持：节点替换、条件分支、人工审核、阶段级重试、A/B 模型和离线 replay。

### 4.2 当前实现仍是进程内、自由文本工作流

不足包括：

- LangGraph state 没有持久化；
- 中间结果没有 Artifact ID；
- 没有 prompt/model/schema release；
- 没有 input document manifest；
- 没有 token/cost/latency；
- 下游主要消费自由文本，citation lineage 容易丢失；
- 失败后只能整体返回原始 state，缺少阶段 checkpoint。

因此应升级为 Versioned AI Workflow：

```text
ai_workflow_release
ai_stage_release
ai_run
ai_stage_execution
selection_manifest
citation_binding
ai_output_artifact
```

每个 Stage 保存明确输入版本、Prompt/Model/Schema 版本、状态、重试、成本和输出 Artifact。

### 4.3 Opinion 必须是可选节点

Newsletter 可有观点，但事实型知识库报告不应强制引入观点。可定义：

```text
FACTUAL_DIGEST:    Research -> Analysis -> Editor
EDITORIAL_DIGEST:  Research -> Analysis -> Opinion -> Editor
TECH_TREND_REPORT: Cluster -> Evidence -> Analysis -> Editor
SITE_CHANGE_REPORT: Diff -> Evidence -> Summary
```

## 5. LLM 客户端实现细节

`OpenRouterClient.generate_completion()` 使用同步 `requests.post()`，并在异常时返回空字符串。虽然另有 `generate_with_retry()`，Agent 节点实际调用的是不带 retry 的 `generate_completion()`。

生产改造要求：

1. LLM 放独立 Worker Pool，不能阻塞抓取 Worker；
2. 按 HTTP 状态和 provider error 分类 retry，支持 Retry-After、指数退避、抖动、熔断；
3. 结构化输出做 schema validation；
4. Prompt/Model/Schema 版本化；
5. 记录 token、cost、latency 和 provider request ID；
6. 输出为空不能视为成功；
7. LLM 故障不能阻塞源站同步。

另一个细节是参数默认值使用 `temperature or self.temperature`。这会让显式传入 `0.0` 时仍回落到默认 temperature。生产代码应使用 `temperature if temperature is not None else default` 这类显式语义，避免配置值被 truthy/falsy 规则改变。

## 6. 调度器与 FastAPI 的运行时问题

### 6.1 进程内 schedule 不具备恢复语义

`NewsletterScheduler` 使用 Python `schedule` 库，`while self.is_running` 中每分钟 `schedule.run_pending()`，并 `time.sleep(60)`。

问题：

- 重启后 schedule 丢失；
- 多副本会重复触发；
- 无 leader election；
- 无 misfire/catch-up；
- 无持久化 next_run/last_run；
- 无稳定的时间窗幂等键。

### 6.2 FastAPI 中还会阻塞 event loop

`/schedule/start` 通过 `asyncio.create_task(run_scheduler())` 启动；`run_scheduler()` 是 async 函数，但内部直接调用同步的 `scheduler.start_scheduler()`，后者进入 `while + time.sleep(60)`。

因此同一 Uvicorn Worker 的 event loop 可能被长期占住，其他 async 请求也会受影响。

生产系统必须把 Scheduler 从 API 进程剥离：

```text
DB-backed schedule
  -> HA Scheduler / leader lease
  -> create durable command/run
  -> transactional outbox
  -> worker queue
```

API 只写命令，不直接跑持久长任务。

### 6.3 Web 状态和配置全部是进程内事实

`api/fastapi_server.py` 使用：

```text
newsletter_history = []
system_status = {...}
global scheduler instance
```

`/config` 也只修改当前进程里的 `config` 对象，重启后丢失；多 Worker 时各进程还能出现不同配置。

因此 Web Control Plane 的 Run、History、Schedule、Config 必须写 PostgreSQL，并采用版本化 Release；页面刷新、滚动发布或 API 多副本不能改变业务事实。

### 6.4 `/health` 只能算 liveness

项目 `/health` 无论 Crawl4AI、OpenRouter、Gmail 是否可用都返回 healthy。生产应拆成：

```text
/health/live
/health/ready
/health/components
```

ready 检查 PG/Redis/S3 等核心依赖；components 展示 Browser、Search、LLM、Email 等可选能力。

## 7. 最关键的新问题：上层会把投递失败计为成功

`NewsletterScheduler.newsletter_run()` 调用：

```text
success = gmail_client.send_newsletter(...)
```

如果 `success=False`，它只写错误日志，并不会抛异常，也不会返回结构化失败结果。

FastAPI 的 `run_newsletter_generation()` 只要 `await scheduler.newsletter_run()` 正常返回，就继续：

```text
system_status["status"] = "idle"
system_status["total_sent"] += 1
newsletter_history.append(status="completed")
```

因此存在一条完整的**假成功链路**：

```text
Gmail send failed
  -> scheduler 仅日志 error
  -> async function 正常 return
  -> API 认为 completed
  -> total_sent + 1
```

这说明生产系统不能用“函数没有抛异常”作为业务成功判据。

必须引入**阶段成功契约 + evidence-based finalization**：

```text
StageResult {
  status,
  produced_artifact_ids,
  counters,
  provider_ack_ids,
  error_code,
  retryable
}
```

例如 Delivery 只有在 provider ACK 已持久化时才可进入 `SENT`：

```text
PENDING -> LEASED -> SENDING -> SENT
                         |-> RETRY_WAIT
                         |-> DEAD_LETTER
```

Newsletter 生成成功与邮件发送成功必须是两个独立事实；`total_sent` 应由 `delivery.status=SENT` 聚合，而不是由任务函数返回推断。

这个原则同样适用于抓取：HTTP 200、Extractor 无异常、Worker 函数 return 都不能自动等价于 Accepted Document。

## 8. Gmail Delivery 实现与生产化

`GmailClient` 使用本地 `credentials.json`、`token.json` 和 `InstalledAppFlow.run_local_server(port=8090)` 完成 OAuth，并将刷新后的 token 写回本地文件。

对单机开发方便，但生产多副本存在问题：

- token 文件不是共享业务状态；
- Pod 重建或无持久卷时 token 丢失；
- 多副本可能并发刷新/覆盖；
- 交互式本地 OAuth callback 不适合无头服务启动；
- Provider message ID 只写日志，没有成为 durable delivery evidence；
- 没有 delivery idempotency key，重试可能重复发送。

生产 Delivery Plane 应：

1. OAuth/SMTP/API 凭据存 Secret Manager/Vault，只在 DB 保存 `secret_ref`；
2. 渠道实现为 `DeliveryAdapter`；
3. delivery 表持久化 channel、audience、window、content artifact、attempt、provider_message_id；
4. 对同一 workflow/audience/window/channel 建唯一幂等键；
5. 失败按 retry policy 重试，超过阈值进 dead-letter；
6. 投递失败不回滚已经 Accepted 的知识文档或已经生成的报告。

## 9. 配置、凭据与版本化

`config.py` 从 `.env` 加载 OpenRouter、Gmail、调度、主题和抓取参数。FastAPI 又允许运行时直接改内存字段。

生产系统需要把“配置值”和“凭据”分开：

```text
site_profile_release / system_config_release
  -> 可审核、可回滚、可追踪

secret_binding
  -> 只保存 secret_ref，不保存明文
```

一个 Run 必须记录它实际使用的配置 Release ID，不能只记录“当前配置”。这样才能解释为什么同一 URL 在不同时间得到不同路由、Extractor 或质量结果。

## 10. 对博客知识库方案的直接优化结论

本项目值得吸收的是产品闭环和分层思想，而不是单机实现。对约 1000 个博客的方案应明确加入：

1. **HTTP-first + Browser escalation**：Crawl4AI 仅在动态证据成立时使用，并复用长期 runtime；
2. **Discovery / Article Fetch 二阶段**：Search/RSS/Sitemap/Archive 只产 URL candidate 和 evidence；
3. **Canonical Document IR**：Markdown 为 Projection，不作为唯一真相；
4. **Durable Frontier**：lease、heartbeat、retry、dead-letter、公平调度、backpressure；
5. **Coverage Ledger**：FULL_BACKFILL 必须说明每个 Provider 是否真正穷举；
6. **Metadata Provenance**：发布时间未知保持 NULL，禁止用抓取时间伪造；
7. **Stage Success Contract**：EMPTY/INVALID/ERROR 不能通过空字符串或正常 return 被吞掉；
8. **Evidence-based Finalizer**：阶段成功由数据库事实、Artifact 和 ACK 对账决定；
9. **Versioned AI Workflow**：Research/Analysis/Optional Opinion/Editor 可重放、可审计；
10. **Persistent Control Plane**：Run/History/Schedule/Config 不依赖 FastAPI 内存；
11. **独立 Delivery Plane**：provider ACK、重试、幂等和 dead-letter 持久化；
12. **Secret/Config 分离**：凭据使用 Secret Manager，Run 绑定配置 Release；
13. **健康检查分层**：liveness、readiness、component capability 分开；
14. **License Gate**：该项目仓库当前未声明开源许可证，代码复用前应确认授权；设计思想可参考，但不能默认把源码直接纳入生产实现。

## 11. 结论

`newsletter-agent` 证明了“抓取 → 多阶段 AI → Web 管理 → 定时投递”的完整产品链路，并且 LangGraph 的 Researcher/Analyst/Opinion/Editor 分工很适合作为知识库下游 Digest/Report 的原型。

但它同时暴露了生产系统最容易被忽略的问题：Browser-first 成本、列表页/正文语义混淆、selector 静默错抓、RSS 错误被空字符串吞掉、抓取时间伪装发布时间、进程内 schedule/config/history、sync I/O 阻塞 async、LLM 中间状态不持久化，以及“邮件实际发送失败但上层仍统计成功”的事实不一致。

因此对博客知识库真正有价值的升级不是再增加一个爬虫库，而是建立**持久化事实模型**：Provider Coverage、Durable Frontier、Immutable Snapshot、Canonical IR、Metadata Provenance、Stage Success Contract、Versioned Workflow 和 Idempotent Delivery。只有这些事实都可恢复、可对账、可重放，1000 个站点长期增量同步才具有可解释的正确性。