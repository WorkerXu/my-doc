# Crawl4AI 官方 URL Seeding：面向大规模抓取的智能 URL 发现

## 1. 调研对象

- 官方文档：https://docs.crawl4ai.com/core/url-seeding/
- 主实现：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_url_seeder.py
- 配置实现：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_configs.py
- 调研目标：判断 `AsyncUrlSeeder` 是否适合承担 1000+ 技术博客的全历史 URL 发现与增量同步，并明确它与平台级 Provider、Coverage、Persistent Frontier、全局限流和 Web 管理之间的边界。

## 2. 结论摘要

`AsyncUrlSeeder` 很适合作为 **URL Discovery Adapter / Provider Accelerator**，尤其适合：

1. 从 Sitemap / Sitemap Index 快速得到大批 URL；
2. 通过 Common Crawl 补充历史 URL 证据；
3. 先做 URL pattern、HEAD/partial-head metadata、BM25 等低成本预处理，再决定哪些页面需要正文抓取；
4. 对 Sitemap Index 并行展开，并通过有界队列对结果流施加局部 backpressure；
5. 使用本地缓存减少部分重复发现/元数据处理成本。

但它不应该直接成为平台的 **Coverage Truth、增量状态机、跨站公平调度器、全局 QPS 限流器或百万 URL 的唯一持久队列**。源码表明，若直接把文档示例放大到 1000+ 站点，会出现若干容易被忽视的语义差异：

- `hits_per_sec` 当前实际通过 `asyncio.Semaphore` 实现，限制的是同时进入校验区的并发，而不是严格“每秒请求数”；
- “有界 queue”并不意味着整体内存有界：`seen`、`results`、`discovered_urls` 以及 sitemap-index 子任务仍会随 URL / 子 sitemap 数增长；
- Sitemap cache 在判断有效性前会先 GET sitemap 内容以读取内部 `<lastmod>`，因此缓存主要减少后续解析/URL 处理，不等价于 HTTP Conditional GET 带来的带宽节省；
- `max_urls` 表示结果预算截断，不能被解释为 Provider 已穷尽；
- Common Crawl “latest index”自身有 7 天本地缓存，且 CC URL 文件按 index/domain/pattern 缓存；生产回填若不显式固定 index，会削弱可重放性；
- `filter_nonsense_urls`、pattern、BM25 都会在进入平台 Inventory 前改变结果集，不适合作为“完整历史”的最终判定；
- `live_check` / head extraction 只是可达性和元数据提示，不能证明 URL 是有效文章；
- `many_urls()` 一次为所有 domain 建任务，且同一个 Seeder 实例存在共享可变状态，不应替代平台跨 1000 Source 的持久调度。

因此最合理的集成方式是：**让 AsyncUrlSeeder 只产生带来源、运行参数、缓存状态和截断原因的 Observation；平台 PostgreSQL/S3 决定 Inventory、Coverage、Frontier、Task 和最终文档版本。**

---

## 3. 官方文档的能力模型

官方文档把 URL Seeding 与 Deep Crawling 区分为两类能力：

- Deep Crawling：边抓边发现，适合实时、动态、目标导向探索；
- URL Seeding：先批量发现，再抓正文，适合大规模覆盖、预过滤和资源控制。

对于博客知识库，全历史回填天然更适合“先 Inventory、后 Fetch”的 Seeding 思路；增量同步则应把 Sitemap/RSS/CMS Provider 的周期 reconcile 与小范围 Deep Crawl/Domain Mapping 补洞结合起来。

`SeedingConfig` 文档公开的关键参数包括：

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

这套配置非常适合“发现执行层”，但其中多个字段会直接改变结果集合或网络行为，生产系统必须把它们版本化，并记录最终编译值。

---

## 4. 源码执行链路

### 4.1 `urls(domain, config)`

主流程可以抽象为：

```text
SeedingConfig
  -> parse source(sitemap / cc)
  -> build async generator
  -> producer discovers URLs
  -> seen de-dup
  -> bounded asyncio.Queue
  -> N workers
  -> optional live/head validation
  -> optional cache
  -> append results list
  -> optional BM25 collective scoring
  -> optional score threshold
  -> return List[Dict]
```

源码中的主队列大小：

```text
queue_size = min(10000, max(1000, concurrency * 100))
```

这确实能防止 producer 无限快地把 URL 塞进 worker queue；`await queue.put()` 在队列满时产生 backpressure。

但是两个集合仍是 O(N)：

```text
seen: set[str]
results: List[Dict]
```

因此它是“队列有界”，不是“整个 discovery job 的内存与结果集有界”。对几十万/上百万 URL 的超大站点，Persistent Inventory 仍应流式写 PostgreSQL，而不是把最终列表留在单进程内存。

### 4.2 `max_urls` 的真实语义

当 `len(results) >= max_urls` 时，worker 会设置 stop event，并清空当前 queue 让任务收敛。

这意味着：

- 它是 **结果预算/提前停止**；
- 并不表示 sitemap/CC 已读完；
- 被清掉或未继续生产的 URL 不等于“不存在”；
- Coverage 必须记录 `RESULT_LIMIT_REACHED` / `BUDGET_EXHAUSTED`，不能记录 `AUTHORITATIVE_PROVIDER_EXHAUSTED`。

平台的 Provider Run 必须显式区分：

```text
EXHAUSTED
RESULT_LIMIT_REACHED
TIME_BUDGET_REACHED
RATE_LIMITED
PROVIDER_ERROR
CANCELLED
```

### 4.3 `many_urls(domains, config)`

源码直接构造：

```text
tasks = [self.urls(domain, config) for domain in domains]
results = await asyncio.gather(*tasks)
```

1000 个站点虽然未必立刻成为问题，但这仍然有三个平台级缺口：

1. 一次性创建 domain 级协程，没有 Source fairness / lease / retry / checkpoint；
2. 一个 Seeder 实例复用 `self._rate_sem`、`self.force`、`self._cache_ttl_hours` 等状态；并发 domain 调用时这些字段是共享的；
3. 任意一个进程退出后，未完成 domain 的进度不会自动成为 durable state。

因此 1000 Source 的生产模式应该是：

```text
Platform Scheduler
  -> lease one Provider Task / small domain batch
  -> invoke Seeder
  -> stream/materialize Observations
  -> checkpoint
  -> ack Task
```

而不是一次 `many_urls(1000_domains)`。

---

## 5. Sitemap 发现原理与关键边界

### 5.1 根 Sitemap 探测

源码会尝试：

```text
https://host/sitemap.xml
https://host/sitemap_index.xml
http://host/sitemap.xml
http://host/sitemap_index.xml
```

先用 HEAD 判断，找到后再 GET 内容；若找不到，退回 `https://host/robots.txt` 并解析 `Sitemap:` 行。

这个探测逻辑适合通用 fallback，但平台不能只依赖它，因为现实站点可能：

- sitemap 路径完全自定义；
- robots 里有多个 sitemap；
- HEAD 返回 403/405，而 GET 正常；
- HTTP/HTTPS 行为不同；
- sitemap 在独立 host/CDN 上。

所以 Source Profile 应允许显式 `provider.url`，自动探测只是 onboarding probe。

### 5.2 Sitemap Index 并行展开

`_iter_sitemap_content()` / `_iter_sitemap()` 会识别 `<sitemap><loc>`，并为每个 sub-sitemap 创建一个 asyncio task，结果写入有界 `result_queue`。

优点：

- sub-sitemap 可并行请求；
- 结果 queue 满时会阻塞 producer；
- URL 可以边到边 yield。

但源码会：

```text
tasks = [asyncio.create_task(process_subsitemap(sm)) for sm in sub_sitemaps]
```

也就是说，**结果队列有界，不等于 sub-sitemap task 数量有界**。如果站点有非常多子 sitemap，仍会一次创建 O(K) task。

另外，普通 sitemap 会先把 URL 收集到 `regular_urls`；顶层 `_from_sitemaps()` 又把发现项加入 `discovered_urls` 以便写缓存。因此超大 sitemap 的整体内存仍可能是 O(N)。

平台优化：

- 小中型站点：可直接使用 Seeder；
- 超大 sitemap index：平台先把 sub-sitemap 作为 durable provider shard 落库，每个 shard 独立 lease；
- 每个 shard 用 bounded streaming parser，把 Observation 分批 upsert；
- `sitemap_shard_id + content_hash` 作为 checkpoint，避免单 job 内存承载全站 inventory。

### 5.3 Sitemap Smart Cache 的真实成本

文档描述 `cache_ttl_hours` + `validate_sitemap_lastmod` 的智能缓存。

源码实际流程是：

1. 找到 sitemap URL；
2. GET sitemap 内容；
3. 解析 sitemap XML 内所有 `<lastmod>`，取最大字符串；
4. 再调用 `_is_cache_valid()` 判断本地 JSON cache 是否可用；
5. 若 cache 有效，则从 cache 返回 URL。

因此一个很重要的结论是：

**当前 smart cache 并没有在“使用 cache”时避免 sitemap 主体 GET。**

它可以减少递归 sitemap 解析、URL 枚举和后续 head metadata 请求，但对于顶层 sitemap 本身，并不是 HTTP revalidation cache。

平台增量同步应该优先自己实现：

```text
If-None-Match: <etag>
If-Modified-Since: <last-modified>
```

并保存：

```text
provider_checkpoint
- provider_id
- request_url
- etag
- last_modified
- response_sha256
- observed_sitemap_lastmod
- fetched_at
- http_status
- provider_release_id
```

304 时直接保留上次 inventory evidence；200 时保存新 Provider Artifact，再做 diff。

### 5.4 `<lastmod>` 比较不是强一致版本号

源码 `_parse_sitemap_lastmod()` 返回所有 `<lastmod>` 的 `max()`；`_is_cache_valid()` 使用：

```text
current_lastmod > cached_lastmod
```

这里比较的是字符串，而不是统一解析到 UTC 的 datetime；不同站点若混用时区、精度或格式，字符串序关系不一定能作为严格时间关系。

同时，文章 `<lastmod>` 是站点提供的 hint，不是资源强一致版本号。

平台应把它记为 `updated_at_hint`，而不是把“未变”当成内容 hash 未变的证明。

---

## 6. Common Crawl 发现原理与可重放性

### 6.1 Latest index

Seeder 会请求：

```text
https://index.commoncrawl.org/collinfo.json
```

取第一个 collection id，并把“latest CC index”缓存在本地；默认 Seeder TTL 常量是 7 天。

因此同一个逻辑任务在不同时间运行，可能选择不同 CC index。

对于历史知识库，生产 Provider Run 必须固化：

```text
cc_index_id
cc_query_pattern
cc_query_url
provider_release_id
```

不要只记录 `source=cc`。

### 6.2 CC URL cache

CC 结果会按：

```text
[index_id]_[domain]_[pattern_hash].jsonl
```

写磁盘；未 `force` 时存在文件就直接读取。

这个 cache 可以作为本地加速，但不是平台真相，原因包括：

- 本地盘可能丢失；
- Runtime/Pod 替换后 cache 消失；
- 它不是 Coverage snapshot；
- cache key 隐含当前 CC index/pattern；
- 文件是否完整取决于上次任务是否正常结束。

平台应把 Common Crawl 视为 Gap Evidence Provider，并把需要重放的原始查询结果或其 checksum/对象存储引用保存下来。

---

## 7. `hits_per_sec`：文档名称与源码语义不完全一致

文档把 `hits_per_sec` 描述为请求速率限制。

当前源码实际上做的是：

```text
self._rate_sem = asyncio.Semaphore(hits_per_sec)

async with self._rate_sem:
    await self._validate(...)
```

Semaphore 只限制同时处于 `_validate()` 中的任务数，并没有基于时间窗口补充 token，也没有记录最近一秒已发送多少请求。

所以：

```text
hits_per_sec = 5
```

不能被生产系统解释成严格 `<= 5 request/s`。

如果单请求 50ms，5 个 permit 理论上可能远高于 5 QPS；如果单请求 3 秒，实际 QPS又会低很多。

平台必须继续使用自己跨 Worker 的：

```text
Redis Lua Token Bucket
+ per-host semaphore
+ Retry-After/shared origin backoff
```

Seeder 的 semaphore 只当本进程局部并发保护。

这也应该进入 Runtime/Provider Contract Test，防止未来版本语义变化。

---

## 8. Live Check / Head Extraction 的实现

### 8.1 Live Check

`_resolve_head()`：

- 发 HEAD；
- 2xx 视为 alive；
- 对 301/302/303/307/308 只验证一个 redirect target；
- 第二个 HEAD 2xx 才返回成功。

因此它只能提供快速可达性提示：

- 某些站不支持 HEAD；
- 可能存在多级 redirect；
- HEAD 200 不代表 GET 正文是文章；
- WAF/CDN 对 HEAD 与 GET 可能有不同规则。

平台不应将 `status=valid` 直接映射为 `VALID_ARTICLE`。

### 8.2 Head Extraction

`_fetch_head()` 使用 GET 流式读取：

- 最多约 64 KiB；
- 发现 `</head>` 就提前关闭；
- 未发现 `</head>` 时只保留前约 10 KiB 做解析；
- 手动跟随 redirect，默认最多 5 次；
- 解析 title/meta/link/JSON-LD/lang。

这是一种很实用的“便宜 metadata probe”，特别适合：

- canonical hint；
- `Article` JSON-LD；
- published/updated hint；
- alternate feed；
- OG type；
- 低成本文章候选排序。

但它仍不应替代正文 Fetch/Raw Artifact，因为：

- 只截取 head；
- HTML 可能 malformed；
- JSON-LD 可能不完整或错误；
- 最终 URL 是 redirect 结果，需要重新做 Source scope/SSRF/robots 校验。

建议平台落：

```text
url_probe_observation
- observed_url
- final_url
- probe_type
- status
- title_hint
- canonical_hint
- content_type_hint
- published_at_hint
- updated_at_hint
- probe_artifact_id
- runtime_release_id
```

---

## 9. Pattern / Nonsense Filter：完整性任务里必须后移

Seeder 会在 discovery 和 validate 阶段应用 pattern / `filter_nonsense_urls`。

默认 nonsense filter 当前会过滤一批工具/管理类路径、隐藏目录、部分典型非正文路径。即使规则看起来合理，也有两个风险：

1. 技术博客可能把真实文章放在非常规路径；
2. 一旦在 Seeder 内丢弃，平台就没有 Observation 可以解释“为什么没抓”。

对于 **全历史 Coverage**，建议：

```text
source = sitemap / sitemap+cc
pattern = *
filter_nonsense_urls = false
```

先保存 Observation，再由版本化 `selection_policy_release` 分类：

```text
ARTICLE_CANDIDATE
NON_ARTICLE
UTILITY
MEDIA
OUT_OF_SCOPE
REVIEW
```

只有探索性搜索或明确主题子集任务，才允许在 Seeder 前置 pattern/BM25。

---

## 10. BM25 的适用范围

当 `extract_head=True` 且 `scoring_method=bm25` 时，Seeder会：

1. 从 title、description、keywords、author、OpenGraph、Twitter、Dublin Core、JSON-LD 等拼接文本；
2. 用简单 whitespace tokenizer；
3. `rank_bm25.BM25Okapi` 计算分数；
4. 对本次候选集合做 min-max 归一化到 0..1；
5. 再按 threshold 过滤和排序。

注意：

- 分数依赖“本次候选集合”，不是稳定的全局绝对分；
- min-max 归一化会随着集合变化而变化；
- whitespace tokenizer 对中文等语言并不理想；
- metadata 缺失时可能退化到 URL 字符串匹配；
- relevance 不能等同于“是不是文章”。

因此 BM25 只适合：

- focused discovery；
- repair/gap prioritization；
- 主题知识库；
- Web 管理端推荐“最可能相关的 URL”。

不适合决定全历史文章是否进入 Inventory。

---

## 11. “有界队列”不等于全链路内存有界

官方文档强调 bounded queue 能支持非常大的 domain。源码确实使用了 bounded queue，但仍有以下 O(N)/O(K) 结构：

```text
seen                  O(unique URLs)
results               O(returned URLs)
discovered_urls       O(sitemap URLs used for cache)
regular_urls           O(one sitemap URL count)
sub-sitemap tasks      O(number of sub-sitemaps)
BM25 corpus            O(scored result count)
```

因此对 1000 博客的常规规模，Seeder足够实用；对少数超大站点/百万 URL source，平台应切换到真正流式的 Provider Adapter：

```text
stream parse
 -> batch insert observation
 -> DB unique constraint dedup
 -> checkpoint shard
 -> release memory
```

不要让一个 Python process 维护全量 `seen/results`。

---

## 12. 安全边界

当前 `async_configs.py` 对网络请求反序列化有显式的 untrusted trust boundary；`SeedingConfig` 被排除在 `UNTRUSTED_ALLOWED_TYPES` 之外。

这与本项目的 Capability Firewall 方向一致：

- Web 用户不应直接提交任意 Seeder/Runtime 原生 config；
- Web 只提交 Source Intent；
- 平台编译并审批 Provider Release；
- `source=cc`、`force`、`concurrency`、`hits_per_sec`、`max_urls`、pattern、filter 策略都要受策略约束；
- 网络访问仍需经过 SSRF/egress/domain admission。

不要因为 Crawl4AI 某个 SDK 配置“只是 discovery”就绕过平台安全编译层。

---

## 13. 对现有博客知识库方案的具体优化

### 13.1 新增 URL Seeder Provider Adapter

```text
provider_type = CRAWL4AI_URL_SEEDER
mode = SITEMAP | COMMON_CRAWL | SITEMAP_PLUS_CC
```

仅负责产生 Observation，不直接写 Document。

### 13.2 新增 Provider Checkpoint / Evidence

```text
provider_run
- provider_run_id
- source_id
- provider_release_id
- runtime_release_id
- started_at
- finished_at
- termination_reason
- result_limit
- result_count
- raw_observation_count
- selected_count
- filtered_count
- cache_hit
- cache_policy
- cc_index_id
- sitemap_root_url
- sitemap_etag
- sitemap_http_last_modified
- sitemap_body_sha256
- sitemap_lastmod_hint
- truncated
```

### 13.3 将本地 Seeder Cache 降级为 Accelerator

业务规则：

```text
Seeder cache hit != provider unchanged
Seeder cache miss != provider changed
Seeder status valid != valid article
CC latest != reproducible backfill
max_urls reached != provider exhausted
```

### 13.4 增量同步优先 Conditional Provider Fetch

每个 Sitemap/Feed Provider 保存 ETag/Last-Modified/Body Hash；有 304 则不重新枚举全量 URL。若站点不支持条件请求，再退到 body hash / sitemap `<lastmod>` / TTL。

### 13.5 超大 Sitemap Index Sharding

当 sub-sitemap 数量或预计 URL 数超过阈值：

```text
root sitemap index
 -> materialize sub-sitemap observations
 -> create SITEMAP_SHARD task per sub-sitemap
 -> fair scheduler
 -> conditional GET
 -> streaming parse
 -> batch upsert URL observations
```

避免一个 Seeder 调用一次性创建所有 sub-sitemap task。

### 13.6 Raw Observation First

Coverage 模式下关闭 Seeder 前置语义过滤，将过滤移动到平台：

```text
raw provider observation
 -> normalize
 -> scope
 -> classify
 -> dedup
 -> frontier
```

### 13.7 固定 Common Crawl Index

历史回填计划生成时解析一次目标 `cc_index_id` 并写入 Run/Release；后续 retry/replay 使用同一 index。需要“补最新”时新建 Provider Run，而不是静默漂移。

### 13.8 Provider Contract Test

新增测试：

1. `hits_per_sec` 是否仍只是 semaphore/concurrency；
2. 100/1000 sub-sitemap 时 task 数和内存曲线；
3. `max_urls` 是否提前停止并正确报告 termination reason；
4. cache valid 时是否仍下载 root sitemap；
5. cache corruption recovery；
6. sitemap `<lastmod>` 不同格式；
7. HEAD 405 / GET 200；
8. 多级 redirect；
9. `filter_nonsense_urls` 对真实 article fixture 的误杀率；
10. CC index 选择和 7 天 cache；
11. `many_urls` 在多 domain 下的共享状态与实际并发；
12. BM25 归一化在候选集变化时是否漂移。

---

## 14. 推荐生产集成架构

```text
                Source Profile / Provider Release
                             |
                       Provider Scheduler
                             |
          +------------------+------------------+
          |                                     |
 Native Sitemap/RSS Adapter          Crawl4AI URL Seeder Adapter
          |                                     |
 Conditional GET                      Sitemap / CC Discovery
          |                                     |
          +---------------+---------------------+
                          |
                    URL Observation
                          |
                  PostgreSQL Inventory
                          |
            Normalize / Scope / Classify
                          |
                    Persistent Frontier
                          |
                    Fetch Scheduler
                          |
               HTTP / Browser Runtime
                          |
                    Raw Artifact
                          |
                 Extract / Quality / IR
                          |
                       Markdown
```

`AsyncUrlSeeder` 在这里是“高性能发现执行器”，而不是控制面。

---

## 15. Web 管理端应新增的诊断

每个 Provider Run 展示：

```text
Provider Type
Seeder Runtime Release
Source Mode (sitemap/cc/both)
CC Index ID
Sitemap Root
HTTP ETag / Last-Modified / 304
Sitemap Body SHA256
Sitemap Lastmod Hint
Seeder Cache Hit / Age
Raw Observations
Filtered Before Platform
Selected After Platform
Duplicate Count
Result Limit
Termination Reason
Sub-sitemap Count
Peak Seeder RSS
Seeder Queue Size
Global Admission Wait
Local Seeder Concurrency
```

尤其要明确区分：

```text
Platform QPS limit
Seeder local semaphore
Provider HTTP cache
Seeder local cache
Coverage complete
Result truncated
```

避免运维人员把“cache hit”“valid”“max_urls 已返回 N 条”误读为历史完整。

---

## 16. 最终判断

这篇官方 URL Seeding 文档补足了现有方案在“先发现、后抓取”上的工程抓手，但源码分析也说明：**不能把 SDK 级便利接口直接等价成平台级大规模同步语义。**

最值得吸收的能力是：

- Sitemap / Common Crawl 快速 bulk discovery；
- Sitemap Index 并行展开；
- partial head metadata probe；
- 局部 backpressure；
- discovery cache；
- 先筛选后正文抓取的成本模型。

最需要平台兜底的边界是：

- 真正的全局 QPS；
- durable checkpoint / lease；
- Coverage stop reason；
- million-URL 内存；
- Common Crawl index pinning；
- Conditional GET；
- Raw Observation before filtering；
- cache/HEAD/BM25 不参与最终文章真相。

因此，本项目应新增 `CRAWL4AI_URL_SEEDER` Provider Adapter，但继续坚持：

**PostgreSQL + Object Storage 才是 Inventory/Coverage/Artifact 真相，Seeder cache、内存 queue、`status=valid`、BM25 score 和 latest CC index 都只是可重建的运行时观察。**
