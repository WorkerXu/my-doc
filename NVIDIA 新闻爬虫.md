# NVIDIA 新闻爬虫：实现细节与技术原理分析

- 编号：55
- 项目：nvidia-news-crawler
- 地址：https://github.com/kunbona/nvidia-news-crawler
- 调研分支：main
- 调研提交：`d0e364cb7914685fe6b3b540fd298feb34150bc8`
- 主要代码：`main.py`、`crawlers/base_crawler.py`、`crawlers/nvidia_official.py`、`crawlers/google_news.py`、`extractors/content_extractor.py`、`extractors/data_cleaner.py`、`scheduler/task_scheduler.py`、`storage/database.py`、`storage/models.py`、`config/config.yaml`

## 1. 项目定位

`nvidia-news-crawler` 是一个围绕 NVIDIA 主题构建的多来源新闻采集工程。它把 NVIDIA 官方新闻、GTC、投资者关系、Google News、Twitter/X 等来源抽象成多个 crawler，由统一 `CrawlerManager` 初始化、定时运行、清洗、保存到 PostgreSQL，并可导出 JSON/CSV/Excel/Markdown；仓库还包含 AI 分析模块，可接 OpenAI、Claude、DeepSeek 或本地模型做结构化分析、产业解释和报告生成。

它不是一个面向 1000 个技术博客的生产级历史归档系统，但它非常适合作为“Source Adapter 体系、内容表示契约、列表页与详情页边界、增量状态、持久调度、时间语义、去重和 AI 下游解耦”的反面/正面混合样本。其代码量不大，很多架构问题因此暴露得非常直接。

对博客知识库方案最有价值的结论有四个：

1. **来源适配器可以统一接口，但不能把所有来源都压成同一种抓取语义。** NVIDIA 官方站、Google News、投资者报告和社交平台的发现方式、正文获取方式、身份语义和增量游标不同。
2. **Stage 输入输出必须带显式内容表示类型。** 该项目最关键的问题是 `BaseCrawler` 把 Crawl4AI 的 Markdown 结果传给下游，而子类却按 HTML 用 BeautifulSoup 解析，形成“类型正确但语义错误”的静默失败风险。
3. **列表页发现和文章正文抓取必须分层。** 当前多个 crawler 实际只从列表容器里取标题、URL 和列表摘要，却把它直接保存为 article content，无法满足全量正文知识库。
4. **AI 分析天然属于派生下游。** 项目已经把 AI provider 抽成可替换解释器，这个方向正确，但生产系统应进一步让 AI 失败完全不影响抓取、清洗、Markdown 和搜索主链路。

## 2. 实际架构与执行链

主程序 `CrawlerManager` 的执行逻辑可以简化为：

```text
config/config.yaml
  -> 初始化多个 crawler adapter
  -> APScheduler 注册 cron
  -> crawler.async_crawl()
       -> AsyncWebCrawler.arun(url)
       -> result.markdown
       -> adapter.extract_data(...)
  -> DataCleaner.clean_articles()
  -> Database.save_articles()
  -> 可选 FileExporter
  -> CrawlLog
```

AI 模块是另一条逻辑链：

```text
Article
  -> StructuredExtractor
  -> NarrativeInterpreter
  -> AI Interpreter(OpenAI/Claude/DeepSeek/Local)
  -> industry insight / investment thesis / deep research
```

从模块边界看，项目已经意识到 crawler、cleaner、storage、scheduler、AI analysis 应分开；问题在于这些模块仍在一个进程内直接函数调用，缺少 durable task、stage contract、artifact lineage、独立 Worker 和失败隔离，因此不能直接扩大到 1000 站长期运行。

## 3. Source Adapter：方向正确，但抽象层级不完整

### 3.1 `BaseCrawler` 统一了最基本的生命周期

`BaseCrawler` 保存：

- `name`
- `config`
- `url`
- `timeout`
- `retry_times`

并统一提供：

- `async_crawl()`
- 指数退避重试
- `_normalize_date()`
- `_create_article()`
- 抽象 `extract_data()`

这说明“一个站点/来源一个 adapter”的思路是成立的。博客知识库也应该保留这一思路，但接口必须比这里更严格，至少拆成：

```text
probe()
discover()
fetch_detail()
extract()
normalize()
checkpoint()
coverage_evidence()
```

原因是发现列表、正文抓取、抽取规则和增量游标不是同一件事。当前 `BaseCrawler.async_crawl()` 只有一次 `arun(self.url)`，然后直接 `extract_data()`，无法表达分页、sitemap、feed、详情页 fan-out、reconcile、历史覆盖度和增量 checkpoint。

### 3.2 Adapter 不应等于“Python 子类”

当前每增加一种来源就新增一个 Python 类，并在 `_initialize_crawlers()` 里手工增加分支。10 个来源还能接受，1000 个站点会迅速变成部署和版本管理问题。

生产方案应把能力拆成两级：

- **Platform Adapter / Source Family**：WordPress、Ghost、Substack、Medium、自定义文档站等可共享发现与抽取逻辑；
- **Site Profile**：域名、入口、selector、include/exclude、速率、provider、特殊规则等配置化。

只有真正需要代码的特殊站点才实现自定义 Adapter。这样新增站点通常只需新增 Site/Profile，不需要改主程序并重新发布整个爬虫服务。

## 4. 最关键的实现问题：Markdown 被当成 HTML 解析

`BaseCrawler.async_crawl()` 调用 Crawl4AI 后执行：

```python
return await self.extract_data(result.markdown)
```

但 `NvidiaOfficialCrawler.extract_data()` 和 `GoogleNewsCrawler.extract_data()` 的参数名叫 `html_content`，并立即执行：

```python
soup = BeautifulSoup(html_content, "html.parser")
```

这构成一个非常危险的 stage contract 错误：上游传的是 Markdown 表示，下游按 HTML DOM 结构选择 `article`、`h2`、`time`、CSS selector。

即使 Python 类型层面都是字符串，也可能出现：

- 没异常；
- BeautifulSoup 正常创建对象；
- selector 全部匹配不到；
- 返回空数组；
- `async_crawl()` 仍日志打印“completed successfully”；
- 最终把“0 篇文章”误当作正常结果。

这比显式异常更危险，因为它会造成静默数据缺失。

### 4.1 对生产方案的直接改进：Artifact Representation Contract

博客知识库的每个 Artifact 必须明确记录：

```text
representation_type:
  HTTP_BYTES
  HTML_SOURCE
  HTML_RENDERED
  DOM_SERIALIZED
  MARKDOWN_CANDIDATE
  CLEANED_HTML
  JSON_API
  FEED_XML
  PDF_BYTES
  TEXT_EXTRACTED
  CANONICAL_IR

media_type
charset
producer_stage
producer_release
source_snapshot_id
schema_version
content_hash
```

每个 Stage 声明允许的输入类型和输出类型。例如：

```text
CSSSelectorExtractor:
  accepts = [HTML_SOURCE, HTML_RENDERED, CLEANED_HTML]

MarkdownNormalizer:
  accepts = [MARKDOWN_CANDIDATE, CANONICAL_IR]
```

调度器在派发前做 contract validation；Worker 启动后再次校验。这样“Markdown 传给 CSS selector”应在进入 extractor 前直接失败为 `INVALID_REPRESENTATION`，而不是返回空结果。

### 4.2 内容表示不能只靠文件扩展名或变量名

生产系统里常见的混淆还包括：

- rendered DOM 与原始 response HTML 混用；
- Feed summary 当正文；
- JSON API 字段中的 HTML fragment 当纯文本；
- OCR 文本当 PDF 原文；
- Markdown projection 当 Canonical IR；
- gzip bytes 未解压就交给 parser。

因此内容表示必须是数据库和 Artifact Manifest 的一等字段，而不是依赖 `foo.html`、变量名或开发者约定。

## 5. 列表发现与全文抓取没有分层

`NvidiaOfficialCrawler` 在 NVIDIA 新闻入口的每个 article container 中提取标题、URL、日期，然后执行：

```python
content = article_elem.get_text(strip=True)
summary = content[:200]
```

`GoogleNewsCrawler` 同样只从搜索结果卡片读取 headline、source、time、summary。

这意味着当前保存的 `Article.content` 很多时候只是**列表卡片文本**，并不是目标 URL 的文章正文。

对于知识库，这是不可接受的。正确模型应是：

```text
Discovery Page / Feed / Search Result
   -> URL Candidate
   -> Admission
   -> Detail Fetch Route
   -> Source Snapshot
   -> Full-text Extraction
   -> Quality Gate
   -> Accepted Document Version
```

列表页中的标题、发布时间、摘要都只能作为 candidate metadata / evidence，不能直接覆盖正文 canonical 字段。

### 5.1 为什么必须保存 discovery evidence

详情页抓取失败时，列表页信息仍然有价值，因此应保存：

- `discovered_url`
- `discovered_title`
- `discovered_summary`
- `discovered_published_at`
- `discovered_via`
- `provider_id`
- `listing_snapshot_id`
- `first_seen_at`

但只有详情页通过质量验证，才能产生 `Accepted Document Version`。

## 6. Crawl4AI 生命周期和资源模型

`BaseCrawler.async_crawl()` 每次执行都会：

```python
crawler = AsyncWebCrawler()
```

然后调用一次 `arun()`。代码没有表现出长期 Browser Runtime/Context Pool，也没有显式资源关闭策略。

单个主题脚本运行频率低时可能问题不明显；1000 站点会放大：

- Chromium 启动成本；
- 内存峰值；
- browser process 泄漏；
- page/context 数量失控；
- TLS/HTTP 连接复用损失；
- Worker recycle 不可控。

生产方案应该将 Crawl4AI/Playwright 放在 Browser Worker Pool 中：

```text
Browser Worker Process
  -> Browser Runtime（长生命周期，受 generation 控制）
     -> Context Pool（按安全边界/站点隔离）
        -> Page/Tab（短生命周期）
```

并配置：

- 单 Worker 最大 context/page 数；
- RSS/heap watermark；
- 请求数/运行时长达到阈值后优雅 recycle；
- page deadline；
- context reset；
- browser crash 自动重建；
- HTTP-first，只有证据需要时升级 Browser。

## 7. 重试：有指数退避，但缺少持久状态与错误分类

`BaseCrawler` 的 `2 ** attempt` 退避是正确的基本思路，但它把所有异常都视为同一种失败：

```text
exception -> sleep -> retry -> []
```

生产系统至少要区分：

- DNS/连接超时：retryable；
- 429/503：retryable + Retry-After/域级 backoff；
- 404/410：通常 permanent，但需保留观测；
- 401/403/验证码：blocked/manual policy；
- parser contract error：configuration/implementation error；
- selector miss：可能是 drift；
- empty body：可升级 route；
- storage error：不应重新请求源站；
- AI error：不应回滚正文抓取。

更重要的是，当前重试状态只存在于调用栈。进程重启后 attempt 归零，无法做长期 backoff、dead-letter 和可观测统计。生产系统应把 attempt、next_attempt_at、error_class、last_error、lease 写入 PostgreSQL，由 durable scheduler 决定再次派发。

## 8. 时间语义存在严重污染风险

`_create_article()` 中：

```python
"publish_date": publish_date or datetime.now()
```

如果解析不到发布时间，就直接用当前时间作为发布时间。

这会让历史知识库出现系统性错误：一篇 2018 年文章若 selector 漂移导致日期解析失败，可能被记录成 2026 年新文章，进而污染：

- 历史排序；
- 增量窗口；
- 新闻时间线；
- 去重与更新判断；
- RAG 的时间过滤；
- “最近更新”页面。

正确设计必须拆分：

```text
published_at        源站发布时间，未知即 NULL
updated_at_source   源站更新时间，未知即 NULL
first_seen_at       系统首次发现时间
last_seen_at        最近观测时间
fetched_at          网络抓取时间
accepted_at         版本通过质量门时间
```

来源时间还应该有 provenance：HTML meta、JSON-LD、Feed、API、正文、人工 override。任何观测时间都不能伪装成发布时间。

## 9. 去重与增量：当前逻辑只适合单进程单次运行

`DataCleaner` 使用两个进程内集合：

```text
seen_urls
seen_hashes(title + url)
```

这只能防止当前进程生命周期内的重复。重启后集合为空；多副本之间也不共享。

数据库又通过 `Article.url unique=True` 去重，`save_article()` 发现 URL 已存在时直接返回旧记录，不比较内容，也不产生新版本。

这意味着：

- 同 URL 更新正文不会被记录；
- slug 改名会变成新文章；
- redirect/canonical 无法合并身份证据；
- tracking 参数可能造成重复；
- 多语言/镜像/转载关系无法表达；
- 没有历史版本；
- 没有 freshness observation。

博客知识库应使用：

```text
url_identity
url_observation
source_snapshot
extraction_candidate
document
document_version
similarity_evidence
```

URL normalization、redirect、canonical、content hash、标题/作者/发布时间都只是 identity evidence。Accepted Canonical IR hash 未变化时记录 freshness observation；变化时才新增 `document_version` 并触发 Markdown/Search/Embedding 下游。

## 10. 数据库设计：选择 PostgreSQL 是对的，但业务模型过于扁平

项目使用 SQLAlchemy + PostgreSQL，比只用 JSON 文件或内存状态更接近生产系统。不过当前核心只有：

- `Article`
- `FinancialReport`
- `CrawlLog`

`Article.url` 唯一，正文直接存在数据库 Text 字段，状态没有版本化。

对于百万级文档，建议职责拆开：

- PostgreSQL：元数据、状态机、manifest、identity、quality、lineage、lease、checkpoint；
- S3/MinIO：HTML、rendered DOM、PDF、清洗 HTML、候选 Markdown、Canonical IR、最终 Markdown、附件；
- OpenSearch/pgvector：可重建检索投影。

不可变 Snapshot 放对象存储不仅节省数据库膨胀，还使 extraction rule 升级后可以 `REPROCESS`，不用再次访问源站。

### 10.1 `create_all()` 不能代替迁移

`Database._initialize_connection()` 启动时调用 `Base.metadata.create_all(self.engine)`。这能创建缺失表，但不能承担成熟 schema migration。

生产方案应使用 Alembic/等价迁移工具：

- 应用只校验兼容 schema version；
- migration 是独立部署步骤；
- 迁移加全局锁；
- 支持 expand/contract；
- Web 管理端展示 schema/release 版本。

### 10.2 保存批量文章实际上是逐条事务

`save_articles()` 循环调用 `save_article()`，每篇都新建 Session、查询、commit、close。数据量小可工作；大量 backfill 会产生极高事务和连接开销。

生产中应按 batch 做：

- staging/upsert；
- 批量 insert；
- 唯一键冲突处理；
- 事务批大小上限；
- outbox 同事务写入；
- 大对象只写对象存储引用。

## 11. 调度器：Cron 表达清晰，但 APScheduler 进程内状态不能成为业务真相

`TaskScheduler` 使用 `BackgroundScheduler`，`jobs` 还保存在进程内 dict。`CrawlerManager.start()` 通过主线程 `while True: time.sleep(1)` 保持进程不退出。

对单机脚本足够，但 1000 站点会遇到：

- 多副本重复调度；
- 进程重启丢失运行态；
- 无 lease/heartbeat；
- 一个慢站点影响其它站点；
- 没有公平调度和 backpressure；
- 无法对同一站点 FULL_BACKFILL/INCREMENTAL 做互斥；
- 无法可靠取消和恢复。

生产方案应采用 DB-backed Scheduler：

```text
schedule_definition
  -> due schedule instance
  -> create crawl_run/stage_task in PostgreSQL
  -> transactional outbox
  -> Redis Streams
  -> Worker lease
```

HA Scheduler 通过数据库锁/leader election 保证同一 schedule 只物化一次，任务是否完成由 PostgreSQL manifest 和 Finalizer 判断，不依赖 scheduler 进程内存。

## 12. `run_all_crawlers()` 是顺序执行，不具备 1000 站公平并发

`run_all_crawlers()` 对 crawler 字典逐个 await：

```python
for crawler_name in self.crawlers:
    saved = await self.run_crawler(...)
```

虽然函数是 async，但多个 crawler 并没有并发运行。这个例子再次说明：**写了 asyncio 不等于系统具有并发调度。**

1000 站点应采用分层并发：

- global concurrency；
- per-host concurrency；
- per-site token bucket；
- HTTP/Browser/PDF 不同资源池；
- weighted fair queue；
- slow site 隔离；
- queue lag 和 error rate 驱动 backpressure。

不建议简单 `asyncio.gather(1000 jobs)`，因为它仍缺少公平性、租约、动态限流和跨进程持久状态。

## 13. 配置化 selector 值得保留，但需要不可变 Release

`config.yaml` 为不同 crawler 保存 URL、cron、timeout、retry 和 CSS selector，这是正确方向。问题是当前配置只是进程启动时读取的文件，没有：

- 配置版本；
- 发布记录；
- dry-run；
- rollback；
- 变更审计；
- 测试样本；
- 与某次 Document Version 的 lineage 绑定。

生产方案应把 Site Profile / Extraction Profile 作为 immutable release：

```text
profile_id
release_id
version
config_json
created_at
created_by
sample_test_result
status=draft|active|retired
```

每个 Extraction Candidate 必须记录使用的 release_id。规则修改后可对已有 Snapshot 做 shadow replay，质量通过再切流。

## 14. 依赖声明本身也需要发布门禁

`requirements.txt` 中使用：

```text
crawl4ai==0.8.x
```

这不是标准的 PEP 440 通配写法；通常应使用 `==0.8.*` 或固定到经过验证的确切版本。生产系统尤其不能把核心抓取依赖留在模糊/不可复现的状态。

建议：

- lock file / hash pin；
- 镜像固定 digest；
- Adapter Release 记录 crawler library 版本；
- 升级 Crawl4AI/Playwright 时先跑 golden corpus；
- DOM、Markdown、metadata、link discovery 都做回归 diff。

否则一个依赖升级就可能改变 Markdown 生成、链接集合或 Browser 行为，造成全站批量漂移。

## 15. AI 分析模块：Provider 可替换是优点，但必须与主链路彻底解耦

`AIAnalysisManager` 将 OpenAI、Claude、DeepSeek、本地模型封装成 provider，并按“结构化抽取 -> 传统解释 -> AI 三段式解释 -> 行业洞察 -> 投资逻辑 -> 深度研究”执行。这体现了两个可借鉴原则：

1. 模型 Provider 应通过 Gateway/Interpreter 接口替换；
2. AI 结果应建立在标准化文章对象之上，而不是直接耦合抓取器。

但生产知识库需要再进一步：

```text
Accepted Document Version
  -> Derived AI Task
  -> Prompt Release + Model Release
  -> AI Artifact
```

AI 任务必须具有：

- 独立队列；
- 独立预算；
- 幂等键；
- prompt/model/version lineage；
- retry/dead-letter；
- 成本与 token 统计；
- 失败不影响 Markdown/Search readiness。

内容 hash 未变化时，不应重复跑摘要和 embedding。

## 16. 对现有博客知识库技术方案的可落地优化

本项目不会推翻现有的 PostgreSQL + S3/MinIO + Redis Streams + Worker 分层架构，反而验证其必要性。但应明确补强以下能力。

### 16.1 新增 Artifact Representation Registry

所有 stage artifact 强制带 `representation_type`、`media_type`、`schema_version`、`producer_release`、`content_hash`。Worker 声明 accepts/produces，错误类型新增：

```text
INVALID_REPRESENTATION
UNSUPPORTED_MEDIA_TYPE
SCHEMA_MISMATCH
ENCODING_ERROR
```

### 16.2 新增 Adapter Contract Test / Golden Corpus

每个 Platform Adapter / Site Profile 至少保存：

- 典型列表页 snapshot；
- 典型详情页 snapshot；
- 无发布时间样本；
- 代码块/表格/图片样本；
- 重定向/canonical 样本；
- 空页面/挑战页样本。

发布 Profile/Extractor 前自动验证：

- 输入 representation 正确；
- discovery URL 数量在合理区间；
- 正文不是列表摘要；
- title/date/canonical provenance 完整；
- Markdown 结构不发生异常大幅漂移。

### 16.3 Provider -> Candidate -> Detail 的强制二段模型

Feed、Sitemap、Archive、Google News、分类页都只产生 Candidate；任何“列表卡片正文”不能直接进入 Accepted Document Version。只有详情页正文质量满足要求后才 publish。

### 16.4 Unknown time must stay unknown

Quality Policy 增加强规则：

```text
published_at_source == missing
=> published_at = NULL
```

禁止 fallback 到 `fetched_at/first_seen_at`。UI 可以显示“发现于某日”，但不能把它伪装成发布时间。

### 16.5 Scheduler/Worker 与 Adapter 解耦

Adapter 返回纯 Outcome/Artifact/Evidence；不直接决定下一次 cron，不直接持有业务队列，不用进程内集合去重。重试、计划、lease、checkpoint 全由 Control Plane/DB Scheduler 管理。

## 17. 推荐生产接口

```python
class SourceAdapter(Protocol):
    async def probe(self, ctx: ProbeContext) -> ProbeOutcome: ...
    async def discover(self, ctx: DiscoveryContext) -> DiscoveryOutcome: ...
    async def fetch(self, ctx: FetchContext) -> FetchOutcome: ...
    async def extract(self, ctx: ExtractContext) -> ExtractOutcome: ...
    async def checkpoint(self, ctx: CheckpointContext) -> CheckpointOutcome: ...

@dataclass(frozen=True)
class ArtifactRef:
    artifact_id: str
    representation_type: str
    media_type: str | None
    schema_version: str
    content_hash: str
    object_key: str
    producer_release_id: str
```

关键不是 Python 语法，而是把“输入是什么表示、由谁产生、哪个版本规则消费、输出是什么”变成系统可验证的数据契约。

## 18. 结论

`nvidia-news-crawler` 的优点是结构清楚：多来源 crawler、PostgreSQL、Cron、清洗、导出、AI Provider 都已经分出模块；作为个人/小型主题采集项目，这种组织方式容易理解和扩展。

但若目标变成 1000 个技术博客的全量历史归档，它暴露出一组典型生产风险：

- Crawl4AI Markdown 与 HTML parser 的表示契约错位；
- 列表页摘要被当成正文；
- 每次 crawler 只抓单入口，缺少历史 discovery/详情 fan-out；
- publish_date 缺失时错误回填当前时间；
- 去重仅在内存和 URL 唯一键层面；
- 同 URL 更新不会产生版本；
- APScheduler/进程内 job 状态不持久；
- 多 crawler 顺序执行；
- Browser 生命周期和资源池不适合大规模；
- 配置与依赖没有 immutable release；
- AI 派生任务没有独立可靠队列。

因此，这次调研对博客知识库方案最重要的新增不是再加一种 crawler，而是补上 **Artifact Representation Contract + Adapter Contract Test/Golden Corpus**。它能在系统规模扩大前消灭一类极难察觉的“类型没报错、数据却抓空/抓错”的静默质量事故，同时让 Source Adapter 真正成为可独立升级、验证、回放和扩展的生产能力。
