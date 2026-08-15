# Crawl4AI 官方 URL Seeding：面向大规模抓取的智能 URL 发现

## 1. 调研对象与版本锚点

- 官方文档：https://docs.crawl4ai.com/core/url-seeding/
- 主实现：https://github.com/unclecode/crawl4ai/blob/7e801521428ee12509994d39151006f64055ebe3/crawl4ai/async_url_seeder.py
- 配置实现：https://github.com/unclecode/crawl4ai/blob/7e801521428ee12509994d39151006f64055ebe3/crawl4ai/async_configs.py
- Crawl4AI 版本：v0.9.2
- 源码提交：`7e801521428ee12509994d39151006f64055ebe3`
- 调研目标：判断 `AsyncUrlSeeder` 在 1000+ 技术博客全历史回填、增量同步、URL Inventory、Web 管理与可恢复调度中的正确定位，并把文档名称与源码真实语义之间的差异明确下来。

本文以源码行为为准。文档中的字段名称、性能描述和示例只能作为能力提示，生产系统必须使用 pinned release + Contract Test 固化行为。

---

## 2. 结论摘要

`AsyncUrlSeeder` 适合作为 **URL Discovery Accelerator / Observation Producer**，不适合作为平台的 Coverage Truth、增量状态机、全局调度器、严格 QPS 限流器或百万 URL 的持久队列。

它最有价值的能力是：

1. Sitemap / Sitemap Index 的批量 URL 发现；
2. Common Crawl 索引的快速 URL 补洞；
3. partial-head 元数据探测；
4. URL pattern、元数据 BM25 等低成本优先级计算；
5. 有界工作队列带来的局部 backpressure；
6. 本地 Sitemap、CC、HEAD/Live cache 带来的执行层加速。

源码分析得到的关键边界如下：

- `hits_per_sec` 当前通过 `asyncio.Semaphore` 实现，真实语义是局部并发门，不是严格“每秒请求数”；
- 主 worker queue 有界，但 `seen`、`results`、`discovered_urls`、单个 sitemap 的 `regular_urls`、BM25 corpus 和 sitemap-index 子任务仍然可能 O(N)/O(K) 增长；
- Sitemap smart cache 在判断缓存是否可用前就可能 GET 根 sitemap，因此不等价于 HTTP Conditional GET；
- `max_urls` 是提前停止预算，而且共享 `results` 的并发检查不是原子操作，内部请求/结果可能发生少量超额，最终仅通过切片限制返回条数；
- `source=sitemap+cc` 的两个 source 在 generator 中按顺序执行，不是两个独立可审计 Provider Run；
- 更重要的是，producer 会捕获 discovery generator 异常、记录日志后正常结束，因此上游调用可能拿到“部分结果列表”却没有结构化的 Provider failure；
- Common Crawl 的 index 默认取“latest”，而公开 `SeedingConfig` 没有 `cc_index_id` 参数，不能通过公开配置直接固定历史回填 index；
- CC cache 直接写最终 `.jsonl` 文件，中途异常可能留下 partial file；下次 `force=false` 只要文件存在就会当成完整 cache 读取，存在 partial-cache poisoning 风险；
- `many_urls()` 对所有域名 `asyncio.gather()`，并复用同一个 Seeder 的 `_rate_sem`、`force`、cache config 等共享可变状态，不能替代平台级 durable/fair scheduler；
- 文档对 `filter_nonsense_urls` 的描述比 v0.9.2 当前实现更激进：源码中不少 API、媒体、压缩包、源代码扩展名过滤规则实际上被注释掉，说明“行为必须按代码与测试固化”；
- HEAD/live/head-data cache 默认受 Seeder `ttl` 控制，默认常量为 7 天，不能作为 freshness 真相；
- `status=valid`、partial head、BM25、cache hit 都只能是 hint，不能直接生成 `VALID_ARTICLE`、Document 或 PASS Markdown。

因此，对博客知识库最合理的设计是：

**权威 Sitemap/RSS/CMS/Common Crawl index 查询由平台 Provider Adapter 负责持久状态、条件请求、分片与可重放；Crawl4AI URL Seeder 作为 best-effort discovery accelerator 和 metadata probe，仅产生 Observation。**

---

## 3. 官方能力模型与适用场景

官方文档把 URL Seeding 定位为“先发现 URL，再抓正文”，与 Deep Crawling 的“边抓边发现”形成对比。

博客知识库的全历史回填天然适合两阶段：

```text
URL Inventory
  -> Fetch
  -> Raw Artifact
  -> Extract
  -> Quality
  -> Canonical IR
  -> Markdown
```

`SeedingConfig` 当前公开的核心参数包括：

```text
source = sitemap | cc | sitemap+cc
pattern
extract_head
live_check
max_urls
concurrency
hits_per_sec
force
query
scoring_method = bm25
score_threshold
filter_nonsense_urls
cache_ttl_hours
validate_sitemap_lastmod
```

这些字段里有多项会改变结果集合或网络行为，因此在生产平台中都必须经过配置编译、审批和版本化，不能由 Web 用户直接透传。

---

## 4. `urls()` 的执行链路

源码主流程可以抽象为：

```text
SeedingConfig
  -> parse sources
  -> sitemap generator
  -> cc generator
  -> producer
  -> seen de-dup
  -> bounded asyncio.Queue
  -> N workers
  -> optional nonsense filter
  -> optional live/head probe + cache
  -> shared results list
  -> optional BM25 collective scoring
  -> threshold/sort
  -> return List[Dict]
```

主 queue 大小为：

```python
queue_size = min(10000, max(1000, concurrency * 100))
```

`await queue.put()` 会在队列满时阻塞 producer，因此能给 URL discovery 到 validation 之间提供局部 backpressure。

但这不能被理解为“任务整体内存有界”，因为至少还有：

```text
seen                  O(unique URLs)
results               O(returned URLs)
discovered_urls       O(sitemap URLs)
regular_urls           O(one sitemap size)
sub-sitemap tasks      O(number of sub-sitemaps)
BM25 text corpus       O(scored URLs)
```

对常规技术博客足够实用；对几十万到百万 URL 的大站，必须切换到平台级 streaming provider + durable shard。

---

## 5. `max_urls`：返回上限不等于严格执行预算

worker 的核心逻辑是先检查：

```python
if max_urls > 0 and len(res_list) >= max_urls:
    stop_event.set()
    ...
```

然后才调用 `_validate()`，而 `_validate()` 最终会向共享 `res_list` append。

由于多个 worker 并发执行，`len(res_list)` 的检查与后续 append 不是一个原子“领取配额”操作。多个 worker 可能同时看到“还没到上限”，随后都继续请求并 append。

最后 `urls()` 使用：

```python
return results[:max_urls] if max_urls > 0 else results
```

保证的是 **返回条数上限**，不是严格的网络请求数、校验数或内部结果数上限。

因此：

- `max_urls` 只能映射为 `RESULT_LIMIT_REACHED` / budget hint；
- 不能拿它当成本硬配额；
- 不能把“返回了 N 条”解释成 Provider exhausted；
- 平台若需要硬预算，应在 durable scheduler / provider shard 层做 token/lease accounting。

---

## 6. Discovery 异常可能退化为“部分结果正常返回”

这是生产集成中最重要的源码边界之一。

producer 结构大致是：

```python
async def producer():
    try:
        async for u in gen():
            ...
    except Exception as e:
        log_error(...)
    finally:
        producer_done.set()
```

异常被记录后没有重新抛给 `urls()` 调用方。

结果是：

1. Sitemap 已发现 5000 条；
2. 随后 Common Crawl 请求失败；
3. producer 记录 error；
4. worker 把已入队数据处理完；
5. `urls()` 仍可以返回一个普通的 List。

如果平台只看“函数成功返回 + 返回了很多 URL”，很容易把部分结果误判为完整 Provider Run。

因此生产集成必须遵守：

```text
successful function return != provider completed
non-empty result != provider exhausted
```

最佳方案不是从日志猜终止原因，而是：

- Coverage 任务中把 Sitemap 与 Common Crawl 拆成两个独立 Provider Run；
- 每个 Provider 有自己的 typed terminal outcome；
- 权威 Sitemap/CC 使用平台原生 Adapter，能明确返回 HTTP、解析、分页、分片和 completion 状态；
- Seeder 结果只能作为 observation augmentation；
- 若继续使用 Seeder 组合模式，只能标记为 best-effort discovery，不能生成 `AUTHORITATIVE_PROVIDER_EXHAUSTED`。

---

## 7. `source=sitemap+cc` 的真实执行方式

`gen()` 里先：

```text
_from_sitemaps(...)
```

再：

```text
_from_cc(...)
```

因此这两个来源并不是平台语义上的并行 Provider，也没有独立的 source completion 状态、错误类型和 evidence boundary。

对可解释 Coverage，更合理的建模是：

```text
Provider Run A = SITEMAP
Provider Run B = COMMON_CRAWL_INDEX
```

两者可以由平台并行调度，也可以有不同优先级、预算、重试、缓存、证据和停止原因。

`SITEMAP_PLUS_CC` 只适合探索、候选补洞或交互式发现，不适合作为最终 Coverage 的唯一 Provider Run。

---

## 8. Sitemap 根发现机制与局限

Seeder 会尝试：

```text
https://host/sitemap.xml
https://host/sitemap_index.xml
http://host/sitemap.xml
http://host/sitemap_index.xml
```

使用 HEAD 探测；若都没找到，则退到：

```text
https://host/robots.txt
```

解析 `Sitemap:` 行。

局限：

- 自定义 sitemap 路径无法通过 `SeedingConfig` 直接指定；
- HEAD 405/403、GET 200 的站点可能被漏掉；
- robots fallback 固定从 HTTPS 开始；
- sitemap 可能位于独立 host/CDN；
- 平台需要保存明确的 Provider URL 和 HTTP validator。

所以 Source Profile 的显式 sitemap URL 应由 **原生 SitemapProvider** 处理；Seeder 的根探测更适合 onboarding probe 或 best-effort discovery。

---

## 9. Sitemap Index 并行展开：队列有界但 task 数不有界

`_iter_sitemap_content()` / `_iter_sitemap()` 会识别 sitemap index，然后为每个 sub-sitemap 创建 task：

```python
tasks = [asyncio.create_task(process_subsitemap(sm)) for sm in sub_sitemaps]
```

结果通过有界 `result_queue` 返回。

这意味着：

- URL 结果传输具有 backpressure；
- 但有多少 sub-sitemap，就可能一次创建多少 asyncio Task；
- 一个普通 sitemap 又会先建立 `regular_urls` 列表；
- 顶层 `_from_sitemaps()` 为写 cache 还会维护 `discovered_urls`。

所以百万级 sitemap source 应改成：

```text
Root Sitemap Index
 -> save immutable provider artifact
 -> parse sub-sitemap loc
 -> materialize provider_shard
 -> durable fair scheduler
 -> conditional GET one shard
 -> streaming XML parse
 -> batch upsert observations
 -> shard checkpoint
```

这样内存与恢复粒度从“全站”变成“单 shard”。

---

## 10. Sitemap Smart Cache 的真实网络成本

smart cache 逻辑会先尝试找到 sitemap，然后为了得到 sitemap 内部 `<lastmod>`，可能先 GET 根 sitemap 内容，之后才调用 `_is_cache_valid()`。

因此 `cache hit` 不等于“没有访问根 sitemap”。

这类 cache 的价值主要在：

- 少做递归解析；
- 少做 URL 列表重建；
- 少做后续 head/live probe。

但增量同步真正应依赖：

```text
If-None-Match
If-Modified-Since
```

并持久化：

```text
provider_checkpoint
- provider_id
- request_url
- etag
- http_last_modified
- response_sha256
- observed_lastmod_hint
- fetched_at
- http_status
- provider_release_id
```

304 表示本次 representation 未变；200 则保存新 Provider Artifact 后做 inventory diff。

---

## 11. `<lastmod>` 只应当作 Hint

`_parse_sitemap_lastmod()` 会取 XML 中所有 `<lastmod>` 文本的 `max()`，随后 `_is_cache_valid()` 使用字符串：

```python
current_lastmod > cached_lastmod
```

这不是统一解析成 UTC datetime 的强一致比较。

不同站点可能混用：

- 日期；
- datetime；
- 不同时区；
- 不同精度；
- 非严格 ISO 表示。

即使格式完全规范，`<lastmod>` 也只是站点声明，不是内容 hash 或 HTTP validator。

平台只应保存为：

```text
updated_at_hint
```

而不能作为正文没有变化的最终证据。

---

## 12. Common Crawl：latest index 与可重放性问题

Seeder 的 `_latest_index()` 会读取：

```text
https://index.commoncrawl.org/collinfo.json
```

取第一条 collection id，并把 index id 本地缓存，默认 `TTL = 7 days`。

生产回填要求“同一个逻辑任务可重放”，因此必须固定：

```text
cc_index_id
cc_query_pattern
cc_query_url / query parameters
provider_release_id
provider_evidence
```

但 v0.9.2 的公开 `SeedingConfig` 并没有 `cc_index_id` 字段；仓库里也没有公开的 `cc_index_id` 配置入口。

所以现有方案里“固定 Common Crawl index”不能简单写成一个 Seeder 配置字段。推荐实现：

### 12.1 首选：平台原生 CommonCrawlProvider

由平台直接请求：

```text
https://index.commoncrawl.org/<cc_index_id>-index
```

显式传固定 index，支持分页/流式解析/typed failure/evidence/checkpoint。

### 12.2 次选：版本锁定的 Seeder Wrapper

如果为了复用 Seeder 代码，Adapter 可在 pinned release 下显式设置内部 `seeder.index_id`，但这属于内部实现耦合：

- 必须有 Contract Test；
- 升级版本必须重新验证；
- 不应暴露给普通 Web 用户；
- 长期仍优先收敛到原生 Common Crawl Adapter。

因此，Seeder 的 CC 模式定位应是 **gap acceleration**，不是可重放 CC truth provider。

---

## 13. CC Cache 的 partial-file 风险

`_from_cc()` 的行为是：

1. 如果 cache path 存在且 `force=false`，直接逐行读取并返回；
2. 否则对 CC index 发 streaming GET；
3. 使用 `aiofiles.open(path, "w")` 直接打开最终 cache 文件；
4. 每收到一条就写一条；
5. 网络/HTTP/JSON 异常会抛出。

这里没有：

```text
write temp file
 -> fsync/close
 -> completion marker
 -> atomic rename
```

因此如果流式下载中途失败，最终 path 可能已经存在且包含部分 URL。下次 `force=false` 时，逻辑只判断 `path.exists()`，有可能把 partial cache 当成完整 cache。

这对 Coverage 是不可接受的。

平台策略：

- Seeder cache 永远不是 Provider completion evidence；
- Provider attempt 失败后，重试必须 `force=true` 或使用新的 attempt-scoped cache root；
- 更稳妥的 Runtime 包装方式是每 Attempt 使用隔离临时目录，只有成功结束才把 cache 作为 accelerator artifact 晋升；
- 原生 CommonCrawlProvider 应使用对象存储临时 key + completion metadata/atomic promotion；
- Web UI 不允许把 `cache file exists` 显示成“已完整同步”。

---

## 14. `hits_per_sec` 实际是 Semaphore，不是 QPS

源码：

```python
self._rate_sem = asyncio.Semaphore(hits_per_sec)

async with self._rate_sem:
    await self._validate(...)
```

Semaphore 控制的是同时处于 validation 区间的协程数量，不包含基于时间窗口的 token refill。

因此：

```text
hits_per_sec = 5
```

不能推导出：

```text
requests_per_second <= 5
```

请求 50ms 时可能远高于 5 QPS；请求 3 秒时又可能低于 5 QPS。

生产平台继续使用：

```text
Redis Lua token bucket
+ shared per-host semaphore
+ Retry-After
+ origin_backoff_until
```

Seeder semaphore 只做本进程第二道安全网。

---

## 15. `many_urls()`：不应作为 1000 Source 调度器

源码：

```python
tasks = [self.urls(domain, config) for domain in domains]
results = await asyncio.gather(*tasks)
```

问题不仅是一次创建很多 domain 协程，还包括 Seeder 实例有共享可变状态：

```text
self._rate_sem
self.force
self._cache_ttl_hours
self._validate_sitemap_lastmod
self.index_id
```

每个 `urls()` 调用都可能重设 `_rate_sem`。`many_urls()` 中多个域名共享同一个 Seeder，worker 运行时读取的是共享属性，因此不能把 `hits_per_sec` 理解为稳定的 per-domain 或 aggregate rate contract。

1000 Source 应使用：

```text
Persistent Provider Task
 -> lease
 -> Global Admission
 -> one source / small bounded batch
 -> invoke adapter
 -> materialize observations
 -> checkpoint
 -> ack
```

而不是 `many_urls(1000_domains)`。

---

## 16. Live Check 与 Head Extraction

### 16.1 Live Check

`_resolve_head()`：

- HEAD 2xx 视为 alive；
- 对 301/302/303/307/308 验证一个 redirect target；
- 第二个 HEAD 2xx 才返回成功；
- 多级 redirect、HEAD 405、WAF HEAD/GET 差异都可能产生 false negative。

所以 `status=valid/not_valid` 是探测结果，不是文章判定。

### 16.2 Partial Head

`_fetch_head()` 使用 GET stream：

- 手动 redirect，默认最多 5 次；
- 最多读取约 64 KiB；
- 找到 `</head>` 就提前结束；
- 找不到时只拿前约 10 KiB 做解析；
- 解析 title/meta/link/JSON-LD/lang。

它适合产生：

```text
canonical_hint
published_at_hint
updated_at_hint
og_type_hint
language_hint
redirect_hint
```

但不是 Raw Artifact 或正文。

### 16.3 Probe Cache Freshness

HEAD/live cache 的 `_cache_get()` 使用 Seeder constructor 的 `ttl`，默认与 `TTL` 常量一致，即 7 天。

这意味着同一个 cache root 内的 metadata/live probe 可能在数天内直接复用。

所以：

- probe cache 只能是执行加速；
- freshness SLO 不得依赖它；
- Incremental 的正文更新仍依赖 Provider revalidation + GET validator/content hash；
- repair/freshness-sensitive probe 应使用更短 TTL 或 `force`，但由平台策略编译，不让用户直接控制。

---

## 17. `filter_nonsense_urls`：文档与当前实现存在漂移

官方文档把 nonsense filter 描述为会过滤 robots、sitemap、API、媒体、归档、源码、管理路径等大量 URL。

但 v0.9.2 当前源码中，多组规则实际被注释掉，例如：

- feed；
- API/data；
- archive/download；
- media；
- source code/config。

当前真正启用的规则主要包括：

- robots/sitemap；
- 部分 utility files；
- hidden path；
- 常见 admin/login/cart/search 等路径；
- print pattern；
- 部分极短路径。

这说明生产系统不能根据参数名称或文档说明猜过滤行为。

Coverage 模式仍应：

```text
pattern = *
filter_nonsense_urls = false
```

先保存 Observation，再由平台版本化 Selection Policy 分类。

每个 pinned release 都应运行真实 article/non-article fixtures，测 false positive/false negative。

---

## 18. BM25 的实现与边界

当存在 query 且 `extract_head=true` 时，Seeder 会从：

- title；
- description/keywords/author；
- OpenGraph；
- Twitter Card；
- Dublin Core；
- JSON-LD；

拼成文本，用 `rank_bm25.BM25Okapi` 评分。

实现还有几个重要特征：

1. tokenizer 是简单 whitespace split；
2. 分数在当前候选集合内做 min-max normalization；
3. 候选集合改变，归一化分数会漂移；
4. 如果所有 BM25 原始分数一样，代码会给所有项 `0.5`；
5. metadata 缺失时可能退化到 URL 字符串相关度。

因此 BM25 适合：

```text
focused discovery
repair priority
topic subset
admin UI candidate ranking
```

不适合：

```text
article truth
historical completeness
permanent exclusion
```

---

## 19. 对博客知识库技术方案的优化结论

本次源码深挖后，现有方案应进一步收紧 URL Seeder 的职责。

### 19.1 将 Coverage Provider 与 Seeder Accelerator 分层

```text
Authoritative / Replayable Providers
- Native CMS/API
- Native Sitemap
- RSS/Atom
- Archive
- Native CommonCrawlIndex
- Wayback

Best-effort Discovery Accelerators
- Crawl4AI URL Seeder
- Domain Mapping
- Deep/Adaptive Crawl
```

Seeder 只产生 Observation，不产生 Provider complete 结论。

### 19.2 Coverage 模式拆分 Sitemap 与 CC

不要以 `sitemap+cc` 作为一个完整性 Provider Run。

```text
SITEMAP Provider Run
COMMON_CRAWL_INDEX Provider Run
```

分别保存 evidence、count、error、termination reason，再在 Coverage Snapshot 汇总。

### 19.3 增加 Provider Completion Attestation

```text
provider_run
- provider_run_id
- provider_type
- started_at
- terminal_at
- terminal_outcome
- completion_attested
- raw_observation_count
- truncated
- result_limit
- error_class
- evidence_artifact_id
```

只有平台原生 Adapter 或能给出可信 typed completion 的 wrapper 才允许：

```text
completion_attested = true
```

Seeder 普通 List 返回不能自动得到这个值。

### 19.4 Common Crawl 原生化

为可重放回填新增：

```text
provider_type = COMMON_CRAWL_INDEX
- cc_index_id
- url_pattern
- pagination/checkpoint
- query_artifact
```

Seeder CC 保留为快速 gap discovery。

### 19.5 Seeder Cache Attempt Isolation

```text
attempt_cache_root/<attempt_id>/...
```

失败 Attempt 不复用其 cache；成功后 cache 可作为 accelerator artifact 记录，但永不作为 Coverage evidence。

### 19.6 平台硬预算，不依赖 `max_urls`

平台调度器负责：

```text
max_observations
max_requests
max_bytes
max_wall_clock
```

Seeder `max_urls` 仅作为本地软停止手段。

### 19.7 明确两类限流

```text
Platform QPS / concurrency = authoritative
Seeder semaphore            = local safety
```

### 19.8 Raw Observation First

Coverage 模式：

```text
raw observation
 -> normalize
 -> scope
 -> classify
 -> dedup
 -> frontier
```

不在 Seeder 前置 BM25/nonsense filter 永久丢数据。

---

## 20. Web 管理端应展示的 URL Seeder 诊断

每个 discovery run 至少展示：

```text
Provider Type
Authoritative / Accelerator
Provider Release
Crawl4AI Runtime Release
Pinned Commit / Image Digest
Source Component: sitemap | cc
CC Index ID
Completion Attested
Terminal Outcome
Sitemap Root
HTTP ETag / Last-Modified / 304
Provider Evidence SHA256
Seeder Cache Root / Scope
Seeder Cache Hit / Age
Cache Completion Trusted: false
Raw Observations
Selected Observations
Filtered Count
Duplicate Count
Result Limit
Platform Hard Budget
Truncated
Provider Error Class
Sub-sitemap Count
Peak RSS
Local Queue Size
Local Seeder Concurrency
Platform Admission Wait
```

UI 必须明确区分：

```text
Provider complete
Seeder returned list
Cache hit
Result truncated
Platform QPS
Seeder semaphore
```

避免“返回很多 URL”被误读成“历史已抓全”。

---

## 21. Contract Test 清单

每个 Crawl4AI pinned release 至少测试：

1. `hits_per_sec` 的真实请求速率/并发语义；
2. `max_urls` 多 worker 下是否存在内部超额请求/结果；
3. Sitemap 成功、CC 失败时 `urls()` 是否仍返回 partial list；
4. `sitemap+cc` 的执行顺序和 source error 可见性；
5. 根 sitemap HEAD 405 / GET 200；
6. 自定义 sitemap URL 的处理边界；
7. cache hit 时根 sitemap 是否仍 GET；
8. `<lastmod>` 不同格式/时区/精度；
9. 100/1000 sub-sitemap 的 asyncio task 数；
10. `seen/results/discovered_urls` 内存增长曲线；
11. CC index 选择是否仍只能 latest；
12. CC cache 下载中断后是否留下 partial final file；
13. retry `force=false` 是否会读取 partial CC cache；
14. HEAD/live cache TTL；
15. 多级 redirect；
16. partial-head 64 KiB/无 `</head>` 边界；
17. `filter_nonsense_urls` 实际启用规则与文档差异；
18. article fixture 误杀率；
19. BM25 候选集变化时分数漂移；
20. 所有分数相同是否归一化为 0.5；
21. `many_urls` 多 domain 下共享 `_rate_sem` 的真实行为；
22. Runtime crash 后 Seeder local cache/queue 是否会被错误当作持久状态。

Contract Test 的结果必须绑定：

```text
crawl4ai_version
commit_sha
image_digest
url_seeder_release_id
```

---

## 22. 推荐生产架构

```text
                    Source Profile / Provider Release
                                 |
                         Platform Fair Scheduler
                                 |
            +--------------------+---------------------+
            |                    |                     |
     Native Sitemap/RSS    Native CommonCrawl      Seeder Accelerator
            |                    |                     |
   Conditional GET         pinned cc_index_id      sitemap/cc probe
   durable shard           streaming evidence      head/BM25 hint
            |                    |                     |
            +--------------------+---------------------+
                                 |
                           URL Observation
                                 |
                    PostgreSQL Inventory / Coverage
                                 |
                 normalize -> scope -> classify -> dedup
                                 |
                        Persistent Frontier
                                 |
                        Fetch Scheduler
                                 |
                      HTTP / Browser Runtime
                                 |
                        Immutable Raw Artifact
                                 |
                     Extract / Quality / IR
                                 |
                             Markdown
```

其中：

- 原生 Provider 决定可重放的 provider evidence 与 completion；
- Seeder 提升发现速度和元数据预判能力；
- 所有最终身份、Coverage、Task、Document、Version 都在平台真相层闭合。

---

## 23. 最终判断

Crawl4AI URL Seeding 的价值很明确：它把 Sitemap、Common Crawl、并发 head probe、相关度预筛和局部 backpressure 封装成了一个很好用的 discovery SDK。

但源码也说明，SDK 级“好用”不能直接等价为平台级“可审计、可重放、可恢复、严格限流和历史完整”。特别需要防止以下误判：

```text
List returned               != Provider completed
max_urls returned           != Source exhausted
cache file exists           != Cache complete
status=valid                != Valid article
hits_per_sec=N              != <= N requests/s
latest CC index             != Reproducible history
bounded queue               != Bounded job memory
sitemap+cc                  != Two successful providers
BM25 score                  != Article truth
```

因此最终集成原则是：

**Crawl4AI URL Seeder 作为高性能 Observation Accelerator；平台原生 Provider + PostgreSQL + Object Storage 承担 Coverage、Checkpoint、Completion、Evidence、Persistent Frontier 和可重放真相。**