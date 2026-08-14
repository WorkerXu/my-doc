# Finviz 新闻爬虫：实现细节与技术原理分析

- 编号：52
- 项目：finviz-crawler
- 地址：https://github.com/Ezio0/finviz-crawler
- 调研分支：main
- 调研版本：v3.5
- 调研提交：`f34e1832c06c176c1c8398d3d2a1653a311fd308`
- 主要代码：`scripts/finviz_crawler.py`、`scripts/finviz_query.py`、`scripts/topic_classifier.py`、`scripts/finviz_digest.py`

## 1. 项目定位

`finviz-crawler` 是一个单机、持续运行的金融新闻采集守护进程。它把 Finviz 新闻入口、多个 RSS 源、第三方文章正文抓取、SQLite 状态库、Markdown 文件落盘、主题分类、近似去重、全文检索、失败重试和 Chromium 资源自愈放在同一个 Python 应用中。

它不是面向“1000 个技术博客全量历史归档”的通用框架，但很适合作为长期抓取系统的工程样本。代码把很多真实运行问题直接暴露出来：轻量 HTTP 与 Browser 的成本差异、同步 I/O 混入 asyncio、Browser 内存膨胀、Feed 摘要与全文的语义冲突、标题去重误判、重试调度失效、时间语义混用、索引重建成本、删除策略与知识库保留目标冲突、运行时 schema 演进以及单机 PID 锁在分布式环境中的对应问题。

对博客知识库最有价值的不是复用该项目本身，而是把这些单机实现经验抽象成可横向扩展的控制面、状态机、Worker 和可观测模型。

## 2. 实际架构与主数据流

README 和 `finviz_crawler.py` 的主循环可抽象为：

```text
Finviz 主新闻页 ------------------┐
                                 ├-> headline discovery -> SQLite pending
Finviz ticker 页（代码存在但禁用）-┘
RSS feeds --------------------------> RSS summary -> Markdown + SQLite done

SQLite pending
  -> 按 domain 交错
  -> Crawl4AI / Playwright 抓第三方正文
  -> Markdown 文件
  -> SQLite done / pending / failed

周期任务：
  -> 标题近似去重
  -> FTS5 rebuild
  -> 旧文章过期删除
  -> failed 清理
  -> Chromium watchdog / Browser 重启
```

v3.5 对 Finviz 主页面新增 `curl_cffi` 通道，并保留 Crawl4AI 作为 fallback；第三方正文仍由 Crawl4AI 执行。这个结构已经体现出一个非常重要的生产原则：**发现、轻量 Fetch、重型 Browser Fetch、状态、内容存储和索引是不同成本层级，不应由一个“万能爬虫调用”包办。**

## 3. 传输层：HTTP 优先、Browser 按需升级

### 3.1 `curl_cffi` + Crawl4AI 双通道

`crawl_headlines()` 先通过 `run_in_executor()` 调用 `_fetch_finviz_headlines_sync()`，后者使用 `CurlSession(impersonate="chrome")` 请求 Finviz；没有拿到 headline 时再回退 Crawl4AI。Ticker 页面也实现了同样的双通道，只是主循环中因为 quote 页面仍遭遇 403 而被禁用。

这一设计的正确抽象不是“模拟浏览器指纹”，而是：

```text
低成本 Transport -> 证据驱动升级 -> Browser -> blocked/manual
```

对于 1000 个技术博客，应默认 HTTP-first；只有 HTML 不完整、正文依赖 JS、平台 Profile 明确要求或者 selector 仅在 rendered DOM 命中时，才升级 Browser。

生产系统还应记录升级原因，例如：

- `js_required`
- `empty_http_body`
- `selector_miss_http`
- `content_too_short`
- `rendered_only_metadata`

这样可以统计某个平台的 Browser 升级率，并持续把可降级站点迁回轻量 Fetch。

### 3.2 不应复制反爬绕过语义

v3.5 的提交说明明确把 `curl_cffi` 用于 Cloudflare challenge 绕过。知识库方案不应把“绕过明确访问控制”设计为默认能力。可借鉴的是 Adapter/fallback 架构，而不是挑战绕过。

所有 HTTP、Browser、Feed、Sitemap、Asset 请求都应统一经过 robots、访问政策、SSRF/Egress 校验和域级限流；出现验证码、登录、付费墙或明确拒绝时应进入 `blocked/manual_review`，而不是无限变换指纹。

### 3.3 当前实现的连接池问题

`_fetch_finviz_headlines_sync()` 和 `_fetch_ticker_headlines_sync()` 每次调用都会新建一个 `CurlSession`，没有形成长生命周期连接池。在单个站点、5 分钟周期下问题不大，但在 1000 站点场景会损失 keep-alive、TLS session reuse 和连接复用收益。

生产 HTTP Worker 应维护受控的长生命周期 client pool，并配置：

- per-host connection limit；
- keep-alive；
- DNS cache/TTL；
- connect/read/write/total timeout；
- response bytes 上限；
- redirect 上限；
- circuit breaker；
- worker generation/recycle。

Client 只能在 Worker 生命周期内复用，不能跨不可信租户共享 Cookie 或认证上下文。

## 4. asyncio 主循环中的同步 I/O 问题

项目主进程是 `asyncio` 驱动，但 `fetch_rss_articles()` 直接同步调用 `feedparser.parse(feed_url)`，并且所有 RSS feed 顺序执行。`feedparser.parse()` 对 URL 会执行网络 I/O，因此整个 event loop 可能被一个慢 RSS 源阻塞。

这与 `curl_cffi` 调用被显式放入 `run_in_executor()` 形成鲜明对比。

对 1000 站点的生产方案，必须规定：

1. 网络 I/O 只能通过 async HTTP client 或显式 thread adapter；
2. CPU 较重的 DOM/正文抽取可以进入独立 Process Pool/Extract Worker；
3. 任何第三方同步 SDK 都必须经过 `to_thread`/executor 或独立 Worker，不允许直接阻塞 event loop；
4. 要监控 event-loop lag，并将其作为本地并发收缩信号。

否则即使表面使用了 asyncio，系统仍会被少量同步调用退化为串行。

## 5. SQLite 状态模型与可扩展边界

### 5.1 当前 `articles` 表

主程序使用 SQLite WAL，`articles` 表包含：

- `title_hash`
- `title`
- `url`
- `domain`
- `source`
- `publish_at`
- `article_path`
- `fetched_at`
- `crawled_at`
- `status`
- `retry_count`
- `ticker`
- `topic`
- `last_retry_at`

WAL 对单机守护进程合理，部署成本低，也能支持查询工具并发读取。但它把 URL frontier、文章实体、抓取任务、抓取版本、索引状态和内容生命周期压进一张表，因此无法自然表达多 Worker lease、URL redirect/canonical、不可变 snapshot、多版本正文、抽取 replay 和分布式状态机。

博客知识库应拆分为：

- PostgreSQL：业务状态真相源；
- Redis Streams：短期任务分发；
- S3/MinIO：不可变抓取对象；
- 搜索/向量：可重建派生层。

### 5.2 运行时 ALTER TABLE 暴露 schema 管理问题

`init_db()` 通过“先 SELECT 某列，失败再 ALTER TABLE”演进 schema。这在个人脚本中简单有效，但多副本部署会遇到：

- 多实例并发迁移；
- 半升级版本同时运行；
- 迁移失败后状态不明确；
- 无法审计 schema 版本；
- 回滚困难。

更明显的是 `finviz_query.py` 的 `get_conn()`：函数在第一次 `return conn` 后还有一段创建 `tickers` 表的代码，这段代码永远不可达。与此同时，`add_tickers()` 又直接假设 `tickers` 表存在。这意味着在某些新数据库上，Ticker 管理会直接报 `no such table: tickers`。

这类问题说明生产系统必须使用显式数据库迁移工具，例如 Alembic，并且：

- 服务启动时只校验 schema version，不自行临时 ALTER；
- migration 作为单独部署步骤执行；
- migration 有唯一锁；
- 应用声明可兼容的 schema 版本范围；
- Web 管理端显示当前 schema/release 状态。

## 6. RSS：发现成功不等于正文完整

`insert_rss_article()` 把 RSS summary 写成 Markdown，然后直接把记录标记为 `done`。文件正文还主动写入：

`[RSS summary — full article behind paywall]`

这是一个非常典型的状态建模陷阱：**流程状态和内容完整度被混为一谈。**

对知识库来说：

- `done` 只能表示某个阶段完成；
- RSS summary 只能是 `summary_only`；
- 全文成功才是 `full`；
- 只有 metadata 时是 `metadata_only`。

还必须保留：

- `discovered_via=feed`
- feed entry 原始 metadata；
- summary 原文；
- full-text fetch 是否执行；
- quality state；
- feed 发布时间候选及 provenance。

搜索和 RAG 对 `summary_only` 应显式降权，而不是把它与完整正文等价。

## 7. 时间语义：`publish_at` 被观测时间污染

`insert_headline()` 和 `insert_rss_article()` 都使用 `now_seattle()` 写入 `publish_at`，即便 RSS entry 自身可能包含真实发布时间。Finviz headline 的页面时间字段也没有被解析成 canonical 发布时间。

这会破坏：

- 历史排序；
- 时间过滤；
- 增量同步游标；
- 文章更新时间判断；
- 版本时间线；
- 过期策略。

生产模型至少必须拆分：

- `published_at`
- `updated_at`
- `first_seen_at`
- `last_seen_at`
- `requested_at`
- `responded_at`
- `captured_at`

若无法确认真实发布时间，`published_at` 应为空，并保留候选值及来源，而不是拿抓取时间替代。

## 8. 标题规范化与文章身份

`insert_headline()` 和 `insert_rss_article()` 将标题 lower-case 后存入主 `title` 字段。这提高了匹配便利性，但丢失原始展示形式。

生产系统应保存：

- `title_raw`
- `title_normalized`

并把 normalization 当作版本化组件。作者、标签、canonical URL、发布时间等字段也应遵循 raw candidate -> normalized candidate -> resolved canonical 的三层模型。

更重要的是，标题不能作为文章身份。项目对 title hash 做唯一约束，可能把不同 URL 上的同标题文章直接合并。

文章身份应由 URL/canonical、redirect、正文 hash、近似正文和人工关系共同判断。

## 9. 去重：思路正确，但删除方式不可扩展

项目有两层标题去重：

1. normalized title 的 SHA-256 截断 hash；
2. 最近 48 小时标题词集合 Jaccard，相似度 >= 0.75 直接视为重复。

另外 `purge_duplicates()` 对最近 72 小时文章做两两 Jaccard，复杂度 O(n²)，并直接删除后出现的记录。

可借鉴的是“先廉价精确判断，再近似判断”的分层思路；不能继承的是：

- 标题作为身份；
- 英文 token 化用于多语言；
- O(n²) 全比较；
- 近似重复直接物理删除；
- provenance 丢失。

1000 站点应采用：

1. URL/canonical/redirect identity；
2. 正文规范化 `content_hash` 强重复；
3. SimHash/MinHash + LSH 产生近似候选；
4. 正文相似度确认 duplicate group；
5. 语义向量只做“相似文章”分组，不自动删除。

重复记录必须保留来源 URL 和 discovery evidence。

## 10. 重试状态机存在实质性逻辑缺陷

项目定义：

```text
RETRY_INTERVALS = [0, 1h, 4h, 12h]
```

`mark_retry()` 也会更新 `retry_count` 和 `last_retry_at`。

但是 `get_pending()` 里虽然计算了 delay，判断“是否已经到达下次重试时间”的分支最终只有一个 `pass`，随后记录无条件加入 result。也就是说，代码注释写的是 exponential backoff，真实调度却仍可能在下一轮立即重试。

这是非常重要的工程警示：**重试策略不能只是计算和日志，必须成为数据库可执行条件。**

生产 `fetch_task` 至少需要：

- `attempt_count`
- `last_attempt_at`
- `last_error_class`
- `next_attempt_at`
- `lease_owner`
- `lease_until`
- `cancel_requested_at`

Scheduler 只能领取 `next_attempt_at <= now()` 的任务。

同时应保存不可变 `fetch_attempt` 历史，每一次尝试记录 transport、worker、duration、status、error class、bytes、retry decision，避免只保留“最后一次失败”。

## 11. Browser 资源自愈：项目中最有价值的运行经验之一

项目专门监控 Playwright Chromium 子进程数量和 RSS，超过阈值时 terminate/kill，并且 Browser 每 5 个 cycle 强制重建。README 说明这是为了解决 Chromium 长期运行后的 renderer 泄漏和内存膨胀。

这个经验非常值得保留：Browser 是有代际的重资源，而不是永久可复用的无状态 client。

但当前实现的风险也很明显：

- `--single-process` 降低了隔离能力；
- `--no-sandbox` 不适合作为生产默认值；
- watchdog 在 Browser session 内直接杀子进程，可能打断在途任务；
- 固定 5 轮重启是经验阈值，不是资源自适应；
- 多副本时按主机扫描/杀进程容易误伤其它 Worker。

生产 Browser Worker 应维护 generation：

- `generation_id`
- `started_at`
- `tasks_processed`
- `pages_open`
- `contexts_open`
- `rss_bytes`
- `failure_streak`

达到 `max_tasks / max_rss / max_uptime / max_pages / crash_streak` 后进入 `draining`，停止接新任务，等待在途任务结束，再重建 Browser。容器/cgroup 是最后硬边界。

## 12. 域级公平与限速

`crawl_articles()` 先按 domain 分组，再 round-robin 交错，这个设计能避免同域任务连续占满批次。每个页面后又固定 sleep 3 秒。

它体现了正确原则：跨域可以并行，域内要克制。但固定 sleep 不能扩展到 1000 站点，因为不同站点容量、响应时间、429 策略和 robots 要求不同。

生产方案应采用：

- 全局 worker 并发；
- distributed token bucket；
- per-domain `max_concurrency`；
- `min_interval`；
- 429/503/Retry-After 自适应降速；
- weighted fair queue；
- Full Backfill 低于日常增量的优先级；
- HTTP、Browser、Feed、Sitemap、Asset 共享同一域预算。

如果每类 Worker 各自限流，源站实际承受的请求量会被叠加。

## 13. HTML 正则解析的维护风险

Finviz 主新闻和 ticker 新闻主要使用正则匹配 HTML 结构。对单一页面和快速脚本，这种方式很快；但模板一旦增加 wrapper、调整 class、引入换行或改 DOM 层级，正则容易整体失效。

通用知识库应优先：

1. JSON-LD / OpenGraph / HTML meta；
2. DOM selector；
3. Platform Profile；
4. 通用正文 extractor；
5. 正则只做字段级后处理。

还要对 selector hit rate、正文长度分布、metadata source 命中率做 drift detection，及时发现平台模板漂移。

## 14. Markdown 文件模型与原子写入

`save_article()` 用标题生成文件名，按 ticker/market 分目录，并直接 `open(..., "w")` 覆盖写入。

它适合人工浏览，但存在三类问题：

1. 标题不是稳定 object identity；
2. crash 时可能留下半写文件；
3. 同标题/改标题会导致路径冲突或迁移。

生产对象 key 应使用稳定 ID，例如：

```text
articles/{site_id}/{article_id}/{version_id}/article.md
```

S3/MinIO 写入 immutable object；本地 Markdown Sink 如果需要文件系统导出，应采用：

```text
write temp -> fsync -> atomic rename
```

并使用 manifest/desired-observed reconciliation 保证导出最终一致。

## 15. 自动过期与知识库保留目标冲突

Finviz 是滚动新闻场景，因此 `expire_old_articles()` 会删除超过 N 天的数据库行和 Markdown 文件，`cleanup_failed_articles()` 也会删除旧失败记录。这对短窗口新闻应用合理，但对“全量历史知识库”完全不适用。

知识库应该：

- 404/410：保留 tombstone 和最后成功版本；
- 失败任务：保留 attempt/error 证据；
- 原始抓取：可以迁冷存储，不删除 canonical lineage；
- 用户删除：软删除 + 审计；
- 站点禁用：停止后续同步，不自动删除历史文章。

`finviz_query.py` 的 `remove_tickers()` 更进一步：删除 ticker 配置时，会同时删除该 ticker 的文章文件和数据库行。这个耦合对知识库尤其危险。**配置生命周期、订阅生命周期和内容生命周期必须分离。** 删除一个 Site/Tag/Profile/Subscription 的跟踪关系，不应隐式删除已经归档的 canonical 内容。

## 16. FTS5：本地体验好，但全量 rebuild 不适合大规模

项目创建 SQLite FTS5 virtual table，并周期执行：

```sql
INSERT INTO articles_fts(articles_fts) VALUES('rebuild')
```

对小型个人库简单可靠，但数百万文章下频繁全量 rebuild 代价高，也会产生可见的索引延迟。

生产索引应当是事件驱动派生层：

- 新 `article_version` 发布 -> Index Event；
- Index Worker incremental upsert；
- 索引失败只重试 index；
- 全量重建创建新 Search Generation；
- 校验完成后 alias 原子切换；
- 旧 generation 延迟回收。

搜索故障绝不能触发重新抓源站。

## 17. 主题分类与派生知识

`topic_classifier.py` 采用预编译正则关键词规则，对标题和正文前 500 字符做多标签分类；正文抓到后又会再次分类。这是一个成本低、可解释、无需 LLM 的 enrichment 示例。

但当前规则直接硬编码在 Python 文件中，缺少：

- rule version；
- confidence；
- explain/matched evidence；
- replay；
- canary；
- 多语言 tokenizer；
- canonical/derived 边界。

博客知识库可以借鉴“确定性规则先于 LLM”的原则，但 enrichment 必须是可重建派生层：

- `enrichment_release_id`
- `model/rule version`
- `input_projection_version`
- `result + confidence + evidence`

主题分类失败不能阻塞 canonical Markdown 发布，也不能无痕修改原始正文。

## 18. 查询工具暴露的数据一致性问题

`finviz_query.py` 从 SQLite 读 metadata，再按 `article_path` 去文件系统取正文。这种“双真相源”在单机上勉强可控，但代码已经需要额外检查“DB 有记录但文件是否存在”。

大规模系统中：

- PG 保存 object key 和状态；
- S3/MinIO 是大对象真相源；
- `article_version` 只有在对象持久化成功后才能发布；
- 对象与状态跨系统写入通过 staged write + outbox/reconciliation 保证最终一致；
- Web 预览通过 version/object key 获取，不通过标题路径猜文件。

## 19. 指标语义：尝试、接受、插入不能混为一个数字

主循环中对 Finviz headline 的统计逻辑是：只要 `title_exists()` 为 false，就调用 `insert_headline()` 并立即 `new_count += 1`。但 `insert_headline()` 内部还可能因为 Jaccard 近似重复而直接 return，实际上没有插入数据库。

因此日志里的 `new_count` 可能高于真实新增数。

这个细节对生产系统非常重要：指标必须基于权威状态变化，而不是“调用了某函数”。应明确区分：

- discovered
- admitted
- duplicate_rejected
- inserted
- fetch_succeeded
- extraction_passed
- version_published
- indexed

数据库写操作应返回明确 outcome，业务指标由 outcome 或事件流产生，避免尝试数被误认为成功数。

## 20. PID Lock、优雅退出与分布式对应关系

项目用 PID file 防止重复启动，并通过 SIGINT/SIGTERM 设置 `shutdown_event`。对单机守护进程，这是必要保护。

扩展到 Kubernetes 后，PID file 不再能解决“多个 Scheduler/全局周期任务重复执行”的问题。对应方案应是：

- PostgreSQL advisory lock / leader lease；
- 或所有周期任务都转成数据库可竞争领取的 durable job；
- `SELECT ... FOR UPDATE SKIP LOCKED` / task lease 防重复领取；
- leader 只负责产生任务，不直接持有任务业务状态；
- leader 崩溃后 lease 到期自动接管。

这样多个副本可以 HA，而不是依赖单副本部署。

## 21. 当前项目暴露出的关键代码级缺陷

### 21.1 Retry backoff 未真正生效

`get_pending()` 计算 delay 后只有 `pass`，没有过滤未到期任务。

### 21.2 `finviz_query.get_conn()` 有不可达 schema 初始化代码

第一次 `return conn` 之后的 `CREATE TABLE tickers` 永远不会执行，Ticker 管理依赖隐式旧 schema。

### 21.3 同步 RSS 网络调用阻塞 asyncio

`feedparser.parse(URL)` 在 async 主循环中顺序执行，容易阻塞所有其它协程。

### 21.4 新增统计可能虚高

调用者在 `insert_headline()` 因近似重复跳过后仍可能把 `new_count` 加一。

### 21.5 删除配置会联动删除内容

`remove_tickers()` 删除 ticker 的同时清理文章和文件，内容生命周期与配置生命周期耦合。

### 21.6 `CurlSession` 每请求新建

缺少稳定连接池和连接生命周期管理。

### 21.7 近似去重直接物理删除

`purge_duplicates()` 删除后出现的记录，无法保留 provenance。

这些问题不是在否定该项目；恰恰说明它作为“运行中真实脚本”比理想化 demo 更有调研价值。

## 22. 对博客知识库技术方案的直接优化结论

基于本项目，现有博客知识库方案应明确补充以下能力：

1. **持久化 `fetch_attempt` 历史**，把 retry decision、transport、duration、error class、bytes、worker generation 全部记录下来。
2. **阻塞 I/O 隔离规范**：所有网络 I/O async 化；同步 SDK 只能通过 thread adapter；CPU 重抽取进入独立 Worker。
3. **长生命周期 HTTP client pool**：keep-alive、per-host pool、timeout、bytes budget、circuit breaker。
4. **Scheduler HA/Leader Lease**：用 PG advisory lock/lease 替代单机 PID lock语义。
5. **显式 schema migration**：Alembic + schema version + migration lock，禁止业务代码临时 ALTER。
6. **配置生命周期与内容生命周期分离**：禁用站点/删除订阅/删除标签不隐式删除历史文章。
7. **原子文件/对象发布**：本地 Sink 使用 temp+fsync+rename，对象存储使用 immutable key + manifest reconciliation。
8. **可信指标语义**：数据库 outcome 驱动 discovered/admitted/inserted/published 等指标，不能用调用次数代替成功状态。
9. **Enrichment 独立版本化**：关键词规则、Embedding、LLM 分类都属于可重建派生层，失败不阻塞 canonical Markdown。
10. **请求 provenance 完整化**：snapshot 保存 fetcher/request profile release、实际 request headers hash、redirect chain、transport escalation reason。

## 23. 总结

`finviz-crawler` 的价值主要在运行时工程经验，而不是通用爬虫能力。它证明了几个对 1000 站点知识库非常关键的事实：

- HTTP 与 Browser 必须分层；
- Browser 必须代际回收；
- Feed 是发现和摘要来源，不代表全文；
- retry 必须由 durable `next_attempt_at` 驱动；
- 标题不能作为文章身份；
- 近似重复不能直接物理删除；
- `published_at` 不能用观测时间代替；
- 索引和 enrichment 应是可重建派生层；
- 同步 I/O 会让 asyncio 名存实亡；
- schema、指标、配置删除和文件写入这些“非爬虫核心”问题，最终往往决定长期系统是否可靠。

因此，对博客知识库方案最合理的吸收方式，是保留其“分层 Fetch、资源自愈、域公平、低成本规则分类”的工程思想，同时用 PostgreSQL durable state、不可变对象存储、版本化规则、显式迁移、HA 调度和严格生命周期模型替换单机脚本式实现。