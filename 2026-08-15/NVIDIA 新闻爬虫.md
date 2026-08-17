# NVIDIA 新闻爬虫：实现细节与技术原理分析

- 编号：55
- 项目：nvidia-news-crawler
- 地址：https://github.com/kunbona/nvidia-news-crawler
- 调研分支：main
- 调研提交：`d0e364cb7914685fe6b3b540fd298feb34150bc8`
- 主要代码：`main.py`、`crawlers/base_crawler.py`、`crawlers/nvidia_official.py`、`crawlers/google_news.py`、`crawlers/investor_reports.py`、`extractors/content_extractor.py`、`extractors/data_cleaner.py`、`scheduler/task_scheduler.py`、`storage/database.py`、`storage/models.py`、`storage/file_exporter_impl.py`、`config/config.yaml`、`ai_analysis/analysis_manager.py`

## 1. 项目定位与价值

`nvidia-news-crawler` 是一个围绕 NVIDIA 新闻、GTC、投资者关系、Google News、Twitter/X 等来源组织的 Python 采集项目。它使用 Crawl4AI 做网页抓取，BeautifulSoup 做来源级解析，SQLAlchemy + PostgreSQL 做持久化，APScheduler 做定时任务，并提供 JSON/CSV/Excel/Markdown 导出以及 AI 分析模块。

它不是一个可直接承载 1000 个技术博客全量历史归档的生产系统，但很适合用来验证大型博客知识库方案中的关键边界，因为它把常见的小型爬虫设计都集中展示出来：

- crawler 子类抽象；
- YAML selector 配置；
- Crawl4AI；
- 进程内 cron；
- 内存去重；
- URL 唯一键；
- Markdown 导出；
- AI Provider 抽象。

这些设计在 5~10 个来源时看起来合理，放大到 1000 站后会暴露出静默抓空、内容结构损坏、定时任务没有真正执行、版本丢失、指标失真和配置“写了但没生效”等问题。

本次复核后，对博客知识库技术方案最重要的新增结论有六项：

1. **Scheduler 与 async Worker 之间必须有明确执行契约。** 该项目用 `BackgroundScheduler` 直接注册 `async run_crawler`，存在 coroutine 被当普通返回值处理而没有 await 的风险。
2. **清洗必须结构保真。** `DataCleaner.clean_text()` 通过 `" ".join(text.split())` 压平空白和换行，会直接破坏 Markdown、代码块、表格和列表结构。
3. **Markdown Projection 不能截断，也不能把批量导出文件当知识库正文。** 当前 Markdown exporter 会将正文截到 500 字符，并把多篇文章拼在一个带时间戳的文件里。
4. **成功指标必须是语义化 Outcome，而不是一个 `saved_count`。** 已存在 URL 也被 `save_article()` 返回为成功对象，最终会计入保存数量；抓取异常又可能被吞成空列表后记录 success。
5. **配置必须验证“实际生效值”。** `performance.max_workers`、`performance.backoff_factor`、`database.pool_size`、`storage.compression` 等配置存在，但核心执行路径没有真正消费其中多项，容易形成配置幻觉。
6. **可选 AI 模块必须独立健康检查。** 当前提交树中 `ai_analysis/analysis_manager.py` 引用了 `analysis.structured_extractor`、`analysis.narrative_interpreter`，但仓库树中未见对应 `analysis/` 包；这说明派生功能必须与主抓取链路隔离并独立验证可运行性。

## 2. 实际执行链

主程序大致是：

```text
config/config.yaml
  -> CrawlerManager
      -> 初始化 Database / FileExporter / DataCleaner / TaskScheduler
      -> 根据 enabled 手工实例化 crawler 子类
      -> run_crawler()
           -> crawler.async_crawl()
                -> AsyncWebCrawler.arun(url)
                -> result.markdown
                -> extract_data(...)
           -> DataCleaner.clean_articles()
           -> Database.save_articles()
           -> 可选 FileExporter
           -> CrawlLog(status=success/failed)
```

定时链：

```text
config schedule
  -> BackgroundScheduler.add_job(job_func=self.run_crawler)
  -> scheduler thread executor
  -> ? async coroutine boundary
```

AI 链：

```text
Article
  -> StructuredExtractor
  -> NarrativeInterpreter
  -> OpenAI / Claude / DeepSeek / Local Interpreter
  -> industry insight / thesis / deep research
```

项目模块分层意识是好的，但所有模块仍以同进程直接函数调用为主，没有 durable task、stage contract、artifact lineage 和 worker failure isolation。

## 3. Source Adapter 抽象：方向对，粒度不够

`BaseCrawler` 统一了 `name/config/url/timeout/retry_times`，并提供 `async_crawl()`、日期规范化和文章对象构造。这个方向可以保留，但生产接口至少需要拆分：

```text
probe()
discover()
fetch_detail()
extract()
normalize()
checkpoint()
coverage_evidence()
```

原因是：

- 发现列表与抓详情页不是同一阶段；
- Sitemap、Feed、Archive、API 的分页/游标语义不同；
- FULL_BACKFILL 需要历史覆盖证据；
- INCREMENTAL 需要持久 checkpoint；
- extraction 规则升级后应可基于 Snapshot 离线 replay。

另外，`CrawlerManager._initialize_crawlers()` 对每个新来源写一个 `if` 分支，这在 1000 个站点不可维护。生产环境应分成：

- Platform Adapter / Source Family：WordPress、Ghost、Substack、Docusaurus、通用 Feed、通用 Sitemap 等；
- Site Profile：域名、入口、selector、include/exclude、route、速率和特殊规则；
- Custom Adapter：仅用于无法配置化表达的特殊站点。

## 4. 最严重的表示契约错误：Markdown 被当 HTML

`BaseCrawler.async_crawl()`：

```python
result = await crawler.arun(...)
return await self.extract_data(result.markdown)
```

而 `NvidiaOfficialCrawler.extract_data()`、`GoogleNewsCrawler.extract_data()` 等又执行：

```python
soup = BeautifulSoup(html_content, "html.parser")
soup.select("article")
```

上游传的是 Crawl4AI Markdown，下游按 HTML DOM selector 处理。Python 类型都是 `str`，所以不会触发类型错误，最危险的结果是：

```text
没有异常
-> BeautifulSoup 能构造对象
-> selector 匹配 0 项
-> extract_data 返回 []
-> async_crawl 仍打印 completed successfully
-> 主流程继续写 success crawl log
```

这是典型的“语法成功、业务数据失败”。

生产系统必须给 Artifact 加显式表示类型：

```text
HTTP_BYTES
HTML_SOURCE
HTML_RENDERED
DOM_SERIALIZED
CLEANED_HTML
MARKDOWN_CANDIDATE
JSON_API
FEED_XML
PDF_BYTES
OCR_TEXT
CANONICAL_IR
MARKDOWN_PROJECTION
```

每个 Stage 声明 `accepts/produces`。CSS DOM extractor 不接受 Markdown；representation mismatch 必须成为显式 `INVALID_REPRESENTATION`，不能返回空数组。

## 5. 列表发现与正文抓取混为一谈

`NvidiaOfficialCrawler` 从新闻列表的 `article` 元素中取得 title/url/date，然后：

```python
content = article_elem.get_text(strip=True)
summary = content[:200]
```

`InvestorReportsCrawler` 同样保存列表项文本；`GoogleNewsCrawler` 只保存搜索结果摘要和 Google News 链接。

这说明当前 `Article.content` 很多时候不是详情正文，而只是 Discovery Evidence。

知识库必须强制二段：

```text
Listing / Feed / Sitemap / Search
 -> URL Candidate + Discovery Evidence
 -> Admission
 -> Detail Fetch
 -> Snapshot
 -> Full-text Extraction
 -> Quality Gate
 -> Accepted Document Version
```

列表标题、摘要、日期只作为候选证据，不能直接覆盖最终正文。

Google News 这类聚合源还应把“聚合链接”和“最终源站 canonical URL”分开保存，不能让搜索聚合 URL 成为长期 Document Identity。

## 6. APScheduler 与 async coroutine 边界存在功能性风险

`TaskScheduler` 使用：

```python
from apscheduler.schedulers.background import BackgroundScheduler
```

而 `CrawlerManager.schedule_crawlers()` 注册的是：

```python
job_func=self.run_crawler
```

`run_crawler` 本身是 `async def`。

APScheduler 3.x 的 `BackgroundScheduler` 默认使用普通线程执行器。普通 executor 调用 coroutine function 后得到的是 coroutine object，并不会像 `AsyncIOScheduler` 那样自动在事件循环中 await 它。于是可能出现：

```text
cron 到点
-> scheduler 调用 run_crawler(...)
-> 返回 coroutine object
-> job 被调度器认为已执行
-> 实际网络抓取没有开始
```

即便具体运行环境通过额外包装规避了这一点，也说明“调度器到底调用同步函数、协程，还是只投递任务”必须成为可测试契约。

对 1000 站生产系统，正确设计不是让 cron 直接跑业务协程，而是：

```text
DB-backed Scheduler
 -> 物化 CrawlRun / StageTask
 -> Transactional Outbox
 -> Queue
 -> Async Worker 消费并 await 网络任务
```

Scheduler 只负责“生成持久工作”，不直接执行抓取。开发环境若允许单进程模式，也必须用明确的 async adapter/AsyncIOScheduler，并有集成测试验证“到点后确实产生 run/task/result”。

## 7. 重试被吞成空数组，success 语义失真

`BaseCrawler.async_crawl()` 捕获所有异常，最终：

```python
return []
```

主流程只看到 `articles=[]`，后续仍可能：

```text
clean -> save 0 -> save_crawl_log(status="success")
```

因此下面几种情况会被混在一起：

- 源站真的没有新文章；
- selector 漂移导致 0 条；
- Markdown/HTML 表示错位导致 0 条；
- 网络连续失败后被吞掉；
- 正常抓到但全部被质量规则拒绝；
- 所有 URL 都已经存在。

生产 Outcome 至少要区分：

```text
SUCCEEDED_WITH_NEW_VERSION
SUCCEEDED_UNCHANGED
SUCCEEDED_NO_NEW_CANDIDATE
EMPTY_UNEXPECTED
INVALID_REPRESENTATION
SELECTOR_MISS
RETRYABLE_ERROR
PERMANENT_ERROR
BLOCKED
QUARANTINED
```

并保存 error class、attempt、next_attempt_at 和 evidence。

## 8. `saved_count` 不是可靠业务指标

`Database.save_article()` 发现 URL 已存在时直接返回 existing Article；`save_articles()` 只要返回对象就 `saved_count += 1`。

所以“已经存在的旧记录”也会被统计为“saved”。这会污染：

- crawl log；
- 新增文章数；
- 成功率；
- 站点变化率；
- 告警阈值；
- Web Dashboard。

生产系统应按阶段记录正交计数：

```text
discovered_candidates
admitted_urls
fetched_snapshots
fetch_unchanged_304
extracted_candidates
quality_accepted
quality_rejected
new_documents
new_versions
unchanged_versions
existing_identity_hits
duplicates_detected
blocked
failed
```

绝不能用一个 `saved_count` 代表整条链路。

## 9. DataCleaner 会破坏技术博客的结构

`clean_text()`：

```python
text = " ".join(text.split())
text = text.replace("\n", " ").replace("\r", " ")
```

这会把所有连续空白压成一个空格，尤其会损坏：

- fenced code block；
- Python/YAML 缩进；
- Markdown 列表层级；
- 表格换行；
- 引用块；
- pre/code 文本；
- 数学公式和某些空白敏感格式。

对于技术博客知识库，“清洗”不能等于“全文字符串规范化”。正确方式是先构建块级 Canonical IR，再按 block 类型做局部处理：

```text
paragraph -> 允许合并视觉换行
heading   -> trim 边界空白
code      -> 原样保留内部空白和换行
table     -> 保留单元格和行列结构
list      -> 保留层级
pre       -> 原样保留
```

此外，`remove_empty_fields()` 用 `if v` 过滤所有 falsey 值，会把 `0`、`False`、空列表等语义上合法的值与“字段缺失”混在一起。生产 schema 必须区分：missing、NULL、false、0、empty collection。

## 10. Markdown 导出不满足知识库“全量正文”要求

`FileExporter.export_markdown()` 有两个关键问题。

第一，正文超过 500 字符时：

```python
content = content[:500] + "...\n\n[Content truncated]"
```

这与“全量历史文章清洗为 md”直接冲突。知识库 Canonical Markdown 不允许按展示需要截断。

第二，它把一批文章写入一个带时间戳的文件：

```text
nvidia_official_20260815_....md
```

这种导出是报表，不是知识库内容对象。生产方案需要：

```text
Accepted Document Version
 -> deterministic Markdown Projection
 -> one version / one canonical artifact
 -> stable object key + manifest
```

例如：

```text
projections/{document_id}/{version_id}/document.md
```

批量 ZIP/单文件汇总只能是额外 Export Projection，不能替代 canonical Markdown。

## 11. 时间语义污染

`_create_article()`：

```python
"publish_date": publish_date or datetime.now()
```

解析不到源站发布时间就填当前时间，会把历史文章伪装成新文章。应严格拆分：

```text
published_at        源站发布时间，未知即 NULL
updated_at_source   源站更新时间
first_seen_at       首次发现
last_seen_at        最近观测
fetched_at          抓取时间
accepted_at         版本接受时间
```

每个来源时间还需要 provenance。任何系统观测时间都不能回填到 `published_at`。

## 12. 去重和版本模型不足

`DataCleaner` 的 `seen_urls/seen_hashes` 只存在于当前进程；hash 还是 `MD5(title + url)`。数据库则只用 `Article.url unique=True`。

问题包括：

- 重启后内存去重失效；
- 多副本不共享；
- tracking query 可能生成多个 URL；
- redirect/canonical/slug rename 无法表达；
- 同 URL 正文更新直接被忽略；
- 没有历史版本；
- 没有 freshness observation；
- MD5 在这里没有必要，且 title+url 不是内容版本 hash。

生产模型应分为：

```text
url_candidate
url_identity
url_observation
redirect_evidence
canonical_evidence
source_snapshot
extraction_candidate
document
document_version
similarity_evidence
```

Canonical IR hash 未变化时只记录 freshness observation；变化时 append 新 version。

## 13. 数据库：方向正确，但事务与 schema 都太轻

优点是使用 PostgreSQL + SQLAlchemy，而不是只写本地 JSON。

不足：

1. `Article` 把 current 内容、identity、version 混在一张表；
2. `save_articles()` 每篇新建 Session、先查 URL、再 commit，历史回灌会产生大量小事务；
3. 启动时 `Base.metadata.create_all()` 不能代替正式 schema migration；
4. 大正文直接放 `Text`，不适合保存多版本 HTML/DOM/PDF/IR；
5. `database.pool_size` 出现在 YAML，但 `create_engine()` 没有传入该配置；
6. 没有 transactional outbox、lease、checkpoint、manifest。

生产中建议：PostgreSQL 保存状态/索引/lineage；S3/MinIO 保存不可变 Snapshot 和大 Artifact；Alembic 管理迁移；批量回灌采用 staging/bulk insert/upsert 和批事务。

## 14. 配置幻觉：声明了不等于真正生效

`config.yaml` 包含：

```text
performance.max_workers
performance.timeout
performance.retry_delay
performance.backoff_factor
database.pool_size
storage.compression
storage.output_format
```

但当前主要执行链：

- `run_all_crawlers()` 仍顺序 await，没有使用 `max_workers`；
- retry 直接写死 `2 ** attempt`，没有消费 `retry_delay/backoff_factor`；
- `create_engine()` 没传 `pool_size`；
- FileExporter 没有按 `compression: gzip` 进行 canonical archive；
- `storage.output_format` 也不是主流程的强约束。

这种问题在 1000 站尤其危险，因为运营会以为调大参数已经生效。

技术方案应加入 **Resolved Effective Config**：

```text
declared_config
resolved_config
effective_config_hash
consumed_keys
unknown_keys
unused_keys
defaulted_keys
release_id
```

Profile/Worker 发布时做 schema validation 和 unused-key lint；运行详情页显示本次任务真正使用的有效配置，而不只是 YAML 声明值。

## 15. Browser 生命周期与并发模型

`BaseCrawler.async_crawl()` 每次创建：

```python
crawler = AsyncWebCrawler()
```

代码中没有展示长期 Browser Runtime/Context Pool，也没有显式关闭生命周期。1000 站会放大：

- Chromium 启动成本；
- RSS 峰值；
- page/context 泄漏；
- browser crash；
- 连接无法复用。

另外 `run_all_crawlers()` 是逐个 await，async 并不等于并发。

生产应：

```text
HTTP-first
Browser Worker Process
 -> long-lived Browser Runtime
 -> Context Pool
 -> short-lived Page
```

并配 global/per-site/per-host concurrency、weighted fair scheduling、browser generation recycle、memory watermark 和 backpressure。

## 16. 增量同步目前几乎不存在

当前 cron 只是“隔几个小时重新抓入口页”，没有：

- Sitemap `lastmod`；
- Feed GUID/updated；
- ETag/Last-Modified；
- API cursor；
- provider checkpoint；
- body hash；
- overlap window；
- periodic reconcile。

1000 站不应每次全抓详情正文。正确链路：

```text
Change Signal
 -> 可能变化？
    否：freshness observation
    是：conditional/detail fetch
 -> body hash
 -> extraction
 -> Canonical IR hash
 -> unchanged / new version
```

## 17. AI 模块的正确边界与当前风险

Provider 可替换的方向值得保留：OpenAI、Claude、DeepSeek、本地模型被统一成 interpreter。

但按本次调研提交树，`ai_analysis/analysis_manager.py` 引用了：

```python
from analysis.structured_extractor import StructuredExtractor
from analysis.narrative_interpreter import NarrativeInterpreter
```

仓库递归树中没有看到对应 `analysis/` 目录。这意味着可选 AI 功能至少存在依赖完整性/发布完整性风险。

生产系统必须把 AI 做成 Accepted Version 的独立派生 Worker：

```text
Accepted Document Version
 -> AI Task
 -> prompt_release + model_release
 -> AI Artifact
```

AI 服务启动时做 dependency health check；AI unavailable 不影响抓取、Markdown、全文检索 readiness。

## 18. README、测试与发布门禁

本次提交的 `README.md` 是空文件，仓库虽声明 pytest 依赖，但没有在树中看到完整 Golden Corpus/Contract Test 体系。

对于 1000 站，最危险的不是显式崩溃，而是 selector 漂移后“0 条但任务绿色”。因此每个 Adapter/Profile 发布至少需要：

- 典型 listing snapshot；
- 典型 detail snapshot；
- code/table/image 样本；
- missing date；
- redirect/canonical；
- empty/challenge；
- expected candidate count range；
- expected block structure；
- scheduled-run integration test。

## 19. 对博客知识库技术方案的具体优化

本次调研建议在现有方案上新增/强化以下硬约束。

### 19.1 Scheduler 只物化持久任务

生产 Schedule 不得直接执行 crawler coroutine；只能创建 `crawl_run/frontier_task/outbox_event`。Async Worker 才负责 await 网络抓取。单进程开发模式也必须做 coroutine-aware integration test。

### 19.2 Structure-preserving Cleaning

禁止在 Canonical IR 之前对整篇正文做 `split()/join()` 式全局空白压平。Post-Processor 必须 block-aware，并输出 transformation lineage。

### 19.3 Markdown Full Fidelity Contract

Canonical Markdown：

- 永不截断；
- 每个 accepted version 独立 artifact；
- 稳定 object key；
- 确定性 renderer；
- front matter 包含 version/lineage/hash；
- 聚合导出与 canonical projection 分离。

### 19.4 Semantic Run Counters

Dashboard 和 SLO 使用阶段级计数，不再使用 `saved_count` 这种多义指标。

### 19.5 Zero-result Anomaly Detection

当某站历史上每轮通常发现 20~50 个候选，本轮突然为 0，即使无异常也不能自动视为正常。按 site/provider/profile 维护基线：

```text
candidate_count
selector_hit_rate
median_content_length
accepted_rate
published_at_missing_rate
```

异常进入 `EMPTY_UNEXPECTED/DRIFT_SUSPECTED`。

### 19.6 Effective Config Audit

每个 Run 保存 resolved/effective config，校验未使用字段；Web 可直接看“本次真正生效的 timeout、concurrency、route、backoff、selector release”。

### 19.7 Optional Module Health Isolation

AI、Search、Delivery 等可选模块必须有独立启动探针和 readiness，不允许 import/runtime 缺失把 Source Sync 服务拖死。

## 20. 推荐的生产执行模型

```text
Source Intake
 -> Probe
 -> Provider Discovery
 -> Candidate + Coverage Evidence
 -> Admission
 -> Durable Fetch Task
 -> HTTP-first Fetch / Browser fallback
 -> Immutable Snapshot
 -> Representation Validation
 -> Extraction Candidates
 -> Block-aware PostProcess
 -> Canonical IR
 -> Quality Gate + Zero-result/Drift checks
 -> Document Identity Resolution
 -> Append-only Document Version
 -> Full Markdown Projection
 -> Asset / Search / Embedding / AI async projections
```

任务控制：

```text
Schedule Definition
 -> HA Schedule Materializer
 -> CrawlRun / StageTask
 -> Transactional Outbox
 -> Redis Streams
 -> Worker Lease / Heartbeat
 -> Outcome / Manifest
 -> Stage Finalizer
```

## 21. 结论

这个项目最值得借鉴的不是具体 selector，而是它集中暴露了小型爬虫扩成知识库时最容易忽略的边界问题。

已有方案中关于 Representation Contract、Candidate/Detail 分层、append-only version、durable scheduler、Browser Pool、Golden Corpus 的方向是正确的；本次复核进一步补强了四个生产硬约束：

1. **异步执行契约**：普通 BackgroundScheduler 不能模糊地直接调度 coroutine；生产 Scheduler 只生成持久任务。
2. **结构保真清洗**：技术博客中的代码、表格、列表和空白是内容本身，不能用字符串级统一压平。
3. **完整 Markdown Projection**：canonical Markdown 绝不能截到 500 字符，也不能用批量时间戳报表代替每篇文章的稳定版本文件。
4. **可解释成功与有效配置**：必须区分 new/unchanged/existing/empty/error，并验证配置是否真的被执行代码消费。

这些能力可以显著降低 1000 站长期运行时最难发现的一类事故：系统显示绿色、任务也“完成”，但实际上定时协程没跑、正文被压平/截断、selector 抓空，或者运营配置根本没有生效。