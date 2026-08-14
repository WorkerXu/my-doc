# 使用 Python asyncio 管理 100 多个网站爬虫

## 1. 调研对象

- 索引编号：20
- 文章：Handling 100+ Website Scrapers with Python's asyncio
- 地址：https://dev.to/pradippanjiyar/handling-100-website-scrapers-with-pythons-asyncio-4905
- 相关官方资料：
  - Crawl4AI Multi-URL Crawling：https://docs.crawl4ai.com/advanced/multi-url-crawling/
  - Crawl4AI `arun_many()`：https://docs.crawl4ai.com/api/arun_many/
  - Crawl4AI LLM-Free Extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
  - Crawl4AI Cache Modes：https://docs.crawl4ai.com/core/cache-modes/
  - Python asyncio Task：https://docs.python.org/3/library/asyncio-task.html

## 2. 结论

这篇文章最有价值的不是“把 100 个站点完全并发”，而是三个工程经验：**资源要有边界、站点规则要声明式、失败必须隔离**。这些原则适合博客知识库，但文章中的示例仍是单机、单进程、站点级串行模型，不能直接放大到 1000 个博客的全量历史回灌与长期增量同步。

对当前博客知识库方案的主要优化是：

1. 明确区分“async API”和“真正的并发”。
2. Worker 内采用**有界并发**，跨域并发、域内克制，而不是所有站点串行，也不是一次 `gather()` 全部 URL。
3. Browser Worker 可利用 Crawl4AI `arun_many()` + dispatcher 做本机资源调节，但分布式域限流、任务真相、重试和公平性仍由平台层控制。
4. 站点 CSS/XPath schema 作为版本化规则存储，不写死在爬虫代码里；普通博客优先通用正文抽取，只有结构化列表页/特殊站点再加 schema。
5. 日常增量不能长期使用 `CacheMode.BYPASS` 作为“总是最新”的唯一策略，应由 HTTP Validator、body hash、snapshot 和版本模型决定是否重新抽取。
6. Web/API 不能在请求线程里直接执行长时间抓取；只能创建 durable job，由队列和 worker 执行。

## 3. 文章实际实现链路

文章中的核心结构大致是：

```python
async with AsyncWebCrawler() as crawler:
    for site in urls:
        strategy = JsonCssExtractionStrategy(site["schema"])
        config = CrawlerRunConfig(
            cache_mode=CacheMode.BYPASS,
            extraction_strategy=strategy,
        )
        result = await crawler.arun(site["url"], config=config)
        if not result.success:
            continue
        data = json.loads(result.extracted_content)
        process_and_store(data)
```

它包含几个值得吸收的点：

- 单个 `AsyncWebCrawler` 用 context manager 管理资源，避免每个站点重复初始化 Browser。
- 每个站点通过配置指定 extraction schema，爬虫主流程基本不变。
- 单站失败后 `continue`，不会让整个批次立即退出。
- 抓取与持久化分离出独立处理函数。

但是这个循环本身是**站点级串行**：当前站 `await crawler.arun()` 完成之后才开始下一个站。

## 4. async 不等于并发

这是文章里最容易误读的地方。

`async def` / `await` 的本质是：当协程等待 I/O 时把执行权交回 event loop，使其它已经被调度的 Task 有机会运行。若程序只有一个抓取 Task：

```python
for site in sites:
    await scrape(site)
```

则站点 A 没完成前，站点 B 根本没有被创建为并发 Task。这个写法仍然是顺序执行，只是内部库可以用异步 I/O 实现自己的操作。

真正的并发需要多个 Task 同时处于可运行/等待状态，例如 `asyncio.TaskGroup`、`asyncio.gather()`，或者 Crawl4AI 的 `arun_many()`。

因此文章报告的“4 小时降到约 12 分钟”不能简单归因于“async 让 100 个站点同时跑”。更合理的解释是多因素共同作用：换用了 Crawl4AI 的抓取/抽取路径、复用 Browser 生命周期、页面处理方式变化、目标数据量不同等。文章数据可以作为作者系统的经验值，但不能直接作为 1000 站容量规划公式。

## 5. 为什么不能 `gather()` 1000 个站点或几十万 URL

文章尝试完全并发后遇到封禁、内存飙升和 Browser crash，这一结论方向正确。

问题来自多层资源同时放大：

```text
URL 数
  × 页面并发
  × Browser context/page
  × 子资源请求
  × 页面 DOM/JS 内存
  × 下载字节
  × PG/S3 写入
```

如果一次把上千或数万协程放入 `gather()`：

- 即使 socket 层并未全部建立，也会先创建大量 Task 和关联状态；
- Browser 页面会非常快地耗尽 RAM；
- 同一域可能被瞬间打满；
- 429/503 会形成重试风暴；
- 下游 Extract、S3、PG 可能被短时洪峰压垮；
- cancel 时需要处理大量仍在飞行的任务。

生产系统应使用**有界并发 + 流式领取**，而不是“全串行”和“全并发”二选一。

## 6. 适合 1000 博客的并发拓扑

建议把并发拆成四层：

```text
平台总并发
  -> worker 类型并发
      -> 站点/域并发
          -> 单任务内部子请求并发
```

### 6.1 平台层

Scheduler/Redis 决定不同站点之间的公平调度：

- 增量任务优先于历史 backfill；
- 每站都有并发上限和日预算；
- 大站不能独占全部 worker；
- 失败重试带 backoff，不立即重新入队形成热循环。

### 6.2 HTTP Worker

HTTP 页面成本低，可以同一进程几十个 slot，但每个请求发出前仍必须获得 domain limiter lease。

例如：

- worker 并发 20~50；
- `domain_concurrency=1` 起步；
- 域 token bucket 0.3~1 req/s 起步；
- 不同域之间可并行。

这比“1000 个站依次串行”吞吐高很多，同时又不会对单站形成压力。

### 6.3 Browser Worker

Browser 成本高，应与 HTTP Worker 完全隔离：

- 每进程 4~8 page slot 起步；
- 本机再加内存阈值；
- page wall-time、subrequest count、bytes 都有硬预算；
- worker RSS 过高时停止领取新任务并周期重启。

Crawl4AI 官方 `arun_many()` 支持 dispatcher。`MemoryAdaptiveDispatcher` 可以根据本机内存暂停/调低并发，`SemaphoreDispatcher` 可以固定并发，`RateLimiter` 可以对请求节奏做调节。

但它们只应该作为**Worker 内本地调度器**。平台仍要保留 Redis/DB 的分布式域 limiter，因为多个 Browser Worker/HTTP Worker 可能同时访问同一域，本地 dispatcher 看不到其它进程。

### 6.4 微批而不是全量列表

Worker 每次从 Frontier 领取 20~100 个候选，形成 micro-batch；处理完再领下一批。不要把几十万 URL 一次传给 `arun_many()`。

`stream=True` 时优先按结果完成顺序消费、写 snapshot/状态，避免等整个 batch 完成后一次性持有所有结果。

## 7. TaskGroup、失败边界与取消

Python 官方 `TaskGroup` 提供结构化并发：退出 context 时会等待组内任务；出现未处理异常时能取消同组其它 Task，清理语义比无边界 `create_task()` 更清晰。

但页面抓取属于“单项失败不应取消同批其它页面”的 workload，因此应把**预期页面错误在单 Task 内转换成结果状态**：

```python
async def run_one(item):
    try:
        return await fetch(item)
    except RetryableFetchError as e:
        return ItemResult.retry(item, e)
    except PermanentFetchError as e:
        return ItemResult.failed(item, e)
```

TaskGroup 主要用于捕获 worker 生命周期异常、统一取消和资源清理，而不是让某个普通 404/timeout 取消整个 batch。

取消流程建议：

1. Web 标记 job `cancel_requested`；
2. Scheduler 不再发新 item；
3. worker 在领取、请求前、阶段边界检查 cancel token；
4. 正在飞行的请求按超时/安全边界完成或取消；
5. `try/finally` 释放 limiter lease、Browser page/context、DB lease；
6. 未完成 item 回到可恢复状态。

## 8. 单 Crawler 实例复用：有价值，但边界要更严格

文章使用：

```python
async with AsyncWebCrawler() as crawler:
    ...
```

避免每个页面重复创建/销毁 Browser，这是正确的资源生命周期意识。

生产上进一步建议：

- 一个 Browser Worker 可以长期持有 Browser process；
- page/context 按任务创建和释放；
- 不同不可信站点默认不共享敏感 cookie/storage；
- 无登录博客可复用干净 context 池，但需要清理 service worker、storage、downloads；
- 超过 page count、RSS、运行时长阈值后滚动重启 worker。

HTTP Worker 则复用 `httpx.AsyncClient` / connection pool，但 redirect 必须手动处理并逐 hop 重新做 EgressPolicy。

## 9. 声明式 CSS Schema 的适用位置

文章通过 `JsonCssExtractionStrategy(site["schema"])` 把站点差异放到配置里，这个模式很适合平台扩站。

Crawl4AI 官方文档确认 `JsonCssExtractionStrategy` 支持：

- `baseSelector`；
- 文本字段；
- attribute；
- HTML；
- nested/list；
- 可用一次性 schema generation 辅助生成 selector。

对博客知识库应采用三级策略：

1. **通用正文抽取**：普通博客无需站点 schema。
2. **声明式规则**：CSS/XPath、include/exclude、分页、日期、canonical、wait selector。
3. **Adapter**：只有复杂 API、JS 交互、特殊 identity 才写代码插件。

不要像文章示例那样把 `urls = [{url, schema}, ...]` 长期写在 Python 源码里。生产规则应该存 PostgreSQL，版本化并经过 golden URL 回归测试后 publish。

## 10. Schema 不是 Markdown 正文真相

文章场景是提取“公告/事件列表”，JSON CSS schema 很合适；博客知识库最终要得到稳定 Markdown 正文，不能把所有文章都强行变成逐站 CSS JSON 抽取。

推荐：

```text
HTML/Rendered DOM
 -> 通用正文识别
 -> Article IR
 -> metadata candidates
 -> link/asset resolver
 -> Markdown serializer
```

CSS schema 主要作为：

- 发现归档页上的文章链接；
- 补充作者/日期/标签；
- 对通用正文抽取失败的站点覆盖 body selector。

这样新增站点时大多数不需要写 schema，维护成本才能从 100 扩到 1000+。

## 11. `CacheMode.BYPASS` 不适合作为长期增量策略

文章为了每天抓最新数据使用 `CacheMode.BYPASS`，这对于小规模、只抓首页列表的任务可以接受。

但博客知识库会有数百万历史 URL。如果每轮都 bypass：

- Browser/HTTP 都会重复访问未变化页面；
- 对源站不友好；
- 费用和带宽放大；
- Markdown 重算频繁；
- 依赖 crawler 私有 cache，无法形成平台级可追溯版本。

平台应把缓存和增量拆开：

1. Discovery Source 用 ETag/Last-Modified；
2. 页面用 `If-None-Match` / `If-Modified-Since`；
3. 304 直接结束；
4. 200 后计算 body hash；
5. body 未变不进入 Extract；
6. 变化后保存 snapshot，再生成 article version；
7. Browser 只在 HTTP 内容不完整时升级。

Crawl4AI cache 可以作为 worker 内性能优化，但 PostgreSQL + S3 snapshot 才是业务真相。

## 12. 文章的“失败后 continue”需要升级成 durable state

文章：

```python
if not result.success:
    continue
```

能保证单站失败不终止本次脚本，但会丢失生产级语义：失败原因、尝试次数、下一次重试时间、是否永久失败、是否已经抓过部分数据。

生产系统应记录：

```text
job_item
- stage
- status
- attempt
- last_error_code
- retry_at
- lease_owner
- lease_until
```

并按错误分类：

- 429 -> `Retry-After`；
- 5xx/timeout -> exponential backoff + jitter；
- robots/egress denied -> 不自动热重试；
- 404/410 -> 多轮确认再判断删除；
- extract error -> 可以基于已有 snapshot 离线重抽。

## 13. Flask 请求里 `asyncio.run()` 不适合长任务

文章示例：

```python
@app.route('/api/scrape', methods=['POST'])
def run_scraper():
    asyncio.run(extract_notices_and_events())
```

对于 12 分钟甚至数小时任务，这会把 API 请求生命周期和抓取生命周期绑定：

- HTTP 客户端断开无法表达任务真实状态；
- Web Server worker 被长期占用；
- 不利于 pause/resume/cancel；
- 服务重启时没有 durable recovery；
- 同时多次点击可能重复启动任务。

正确模式：

```text
POST /jobs
 -> PG transaction: job + items + outbox
 -> 立即返回 job_id
 -> outbox relay -> Redis Streams
 -> workers consume
 -> Web 查询/订阅 job progress
```

Cron 也只负责创建增量 job，不直接承担业务状态。

## 14. URL 处理：`urljoin` 只是第一步

文章对相对 URL 用 `urljoin(base_url, extracted_url)`，这个基础动作正确，但平台必须加更多边界：

- 以 `final_url` + 合法 `<base href>` 计算 document base；
- 规范化 host、port、fragment、tracking query；
- 所有发现 URL 都过 allowed-host/EgressPolicy；
- redirect 每 hop 重新校验；
- canonical、图片、srcset、附件复用同一 resolver；
- 不允许 JavaScript URL 通过简单正则变成任意可访问地址后直接请求。

文章中特定站点 `window.open(...)` 的正则兼容可以作为 Adapter 规则，但不能进入通用安全解析器。

## 15. 对现有方案的具体改动

### 15.1 增加 Worker 内结构化并发

明确：

- HTTP worker：async + bounded tasks；
- Browser worker：Crawl4AI `arun_many()`/dispatcher 或等价 slot pool；
- micro-batch 流式处理；
- 不允许 unbounded `gather`；
- TaskGroup 负责生命周期，页面错误转换成 item result。

### 15.2 增加本地并发与分布式并发双层限制

```text
Distributed Domain Limiter (Redis)
       +
Worker Local Limiter (Semaphore / MemoryAdaptiveDispatcher)
```

只有两层同时拿到许可才开始请求。

### 15.3 把 CSS/XPath schema 纳入规则工作台

Web 增加：

- schema 编辑/预览；
- golden page batch test；
- selector 命中率；
- zero-result/字段缺失报警；
- schema version rollback。

### 15.4 明确禁用“API 请求直接跑长任务”

所有手动 Trigger、Cron、Webhook 都只能创建 job。

### 15.5 增加 Browser 资源指标

- local active slots；
- dispatcher wait；
- worker RSS；
- page peak duration；
- result streaming lag；
- Browser recycle count。

## 16. 推荐实现骨架

下面是平台 Worker 的概念模型，不是完整生产代码：

```python
async def browser_worker_loop():
    async with AsyncWebCrawler(browser_config=BROWSER_CONFIG) as crawler:
        while True:
            items = await lease_frontier_batch(limit=50)
            if not items:
                await idle_wait()
                continue

            runnable = []
            for item in items:
                if await job_cancelled(item.job_id):
                    await release_cancelled(item)
                    continue
                if not await distributed_domain_limiter.try_acquire(item):
                    await reschedule(item)
                    continue
                runnable.append(item)

            # 本机并发由硬 slot + 内存阈值继续限制。
            # Crawl4AI arun_many(stream=True) 或等价 bounded TaskGroup。
            async for result in run_many_streaming(crawler, runnable):
                try:
                    await persist_snapshot_and_state(result)
                finally:
                    await distributed_domain_limiter.release(result.item)
```

关键不是具体 API，而是这几个不变量：

1. URL 来自 durable Frontier；
2. 请求前拿分布式域许可；
3. 本机还有独立内存/slot 限制；
4. 结果流式落盘，不攒全批；
5. item 状态幂等写入；
6. 任何退出路径都释放 lease；
7. Worker 重启后未完成 item 可 reclaim。

## 17. 最终评价

这篇文章适合作为“从同步脚本过渡到可维护异步抓取”的案例，但它的规模化结论需要修正：**1000 个博客平台的正确方向不是全串行，也不是无限并发，而是分布式公平调度之上的有界异步并发**。

最值得保留的思想是：

- 复用 crawler 生命周期；
- 声明式 extraction schema；
- 单项失败隔离；
- 不盲目追求并发数；
- 从一开始就保留历史/归档状态。

最需要替换的做法是：

- 站点级全串行；
- `CacheMode.BYPASS` 作为长期更新机制；
- 失败只 `continue`；
- Flask 请求内执行完整抓取；
- Cron 直接跑不可恢复脚本；
- URL/schema 写死在代码中。

这些改动已经足以让“100+ 站点脚本经验”演化成适合 1000+ 博客、数百万 URL、可增量、可观测、可 Web 管理的平台架构。