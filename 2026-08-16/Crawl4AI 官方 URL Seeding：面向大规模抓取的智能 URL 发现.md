# Crawl4AI 官方 URL Seeding：面向大规模抓取的智能 URL 发现

## 1. 调研对象与结论

- 官方文档：https://docs.crawl4ai.com/core/url-seeding/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 本次源码基线：Crawl4AI v0.9.2，commit `7e801521428ee12509994d39151006f64055ebe3`
- 核心实现：`crawl4ai/async_url_seeder.py`
- 相关配置：`crawl4ai/async_configs.py`
- 相关测试：`tests/general/test_async_url_seeder_bm25.py`

结论：`AsyncUrlSeeder` 非常适合做 **低成本 URL 批量发现、候选预筛选和轻量 metadata probe**，但不适合直接承担 1000+ 博客知识库的“历史完整性 Provider”“持久 Frontier”“严格 QPS 控制”“可恢复增量同步”或“Coverage Complete”职责。

生产方案应把它定位为 `DISCOVERY_ACCELERATOR`：其产出首先转成平台 `URL Observation`，之后再由平台做 Normalize、Scope、Robots、Classify、Dedup、Persistent Frontier、Fetch、Quality 和 Coverage。Sitemap、CMS、RSS、Archive、Common Crawl 等真正参与历史 Coverage 的来源应由平台原生 Provider 分开运行并保存 Evidence、Checkpoint 和明确的 terminal outcome。

本次源码核对还发现几处对生产设计很重要的语义：

1. `sitemap+cc` 实现固定先跑 sitemap，再跑 Common Crawl；配合 `max_urls` 时可能在 CC 尚未执行前就停止，因此不能把组合模式理解为“两个来源均完整执行后的最大覆盖”。
2. `max_urls` 在 BM25 评分之前就参与停止发现，因此 `max_urls=10 + query` 不是“全候选中的 top 10”，而是“先拿到至多一批候选，再在这批候选上评分”。
3. 文档宣称 `extract_head=False + query` 可以做 URL 结构评分，但 v0.9.2 源码会直接告警并跳过评分，官方测试也明确断言结果不带 `relevance_score`，说明文档与实现存在漂移。
4. `hits_per_sec` 实际是 `asyncio.Semaphore`，限制的是同时进入 `_validate()` 的局部并发，不是按秒补 token 的严格 QPS。
5. 官方强调 bounded queue，但端到端内存仍包含 `seen`、`results`、Sitemap 的 `discovered_urls`，并且 Sitemap Index 会一次创建全部子 sitemap task，因此不能据此推导“百万 URL 全流程内存有界”。
6. Sitemap Smart TTL Cache 默认仍会先探测并 GET sitemap 以读取 `<lastmod>` 再判断 cache 是否有效，不等价于 HTTP `ETag/If-None-Match` 或 `Last-Modified/If-Modified-Since` 增量同步。
7. Producer、子 sitemap 处理等位置存在“记录错误后继续并返回已有结果”的路径，因此普通 List 返回本身没有“Provider 已完整枚举”的证明力。

---

## 2. AsyncUrlSeeder 的执行模型

`AsyncUrlSeeder.urls(domain, config)` 的核心流程可以概括为：

```text
SeedingConfig
   |
解析 source = sitemap / cc / sitemap+cc
   |
必要时获取 latest Common Crawl index
   |
按固定顺序构建 discovery generator
   |-- sitemap
   `-- cc
   |
bounded asyncio.Queue
   |
producer 去重并写 queue
   |
N 个 worker
   |
_validate()
   |-- no probe -> status=unknown
   |-- live_check -> HEAD
   `-- extract_head -> partial GET + <head> parse
   |
results list
   |
可选 BM25
   |
score_threshold
   |
最终 max_urls slice
```

它的优势是实现简单、异步、对单机批量 URL 发现很友好；局限是多数状态是进程内临时状态，并没有 durable checkpoint、typed completion、分片恢复、跨进程公平性和全局限流语义。

### 2.1 Source 解析不是独立 Provider Run

`source` 会按 `+` 拆成若干 token，但真正 generator 的代码顺序是：

```text
if sitemap in sources:
    yield sitemap URLs
if cc in sources:
    yield CC URLs
```

也就是说无论用户传 `sitemap+cc` 还是 `cc+sitemap`，实际都先 sitemap、后 CC。

这对于探索型工具没有问题，但对于 Coverage 有两个风险：

- 两个来源的错误、缓存、截断和完成状态混在一次 List 返回里，无法独立审计；
- 若前一个来源已触发 `max_urls`，后一个来源可能根本没有运行。

因此生产平台必须拆为独立的 Provider Run：

```text
NATIVE_SITEMAP Provider Run
NATIVE_COMMON_CRAWL_INDEX Provider Run
CRAWL4AI_URL_SEEDER Accelerator Run
```

三者最后在 Coverage Snapshot 中汇总，不把 Seeder 组合模式当作两个权威 Provider 的替代品。

---

## 3. Sitemap 实现细节

### 3.1 自动发现逻辑

Seeder 先尝试：

```text
https://<host>/sitemap.xml
https://<host>/sitemap_index.xml
http://<host>/sitemap.xml
http://<host>/sitemap_index.xml
```

每个候选先走 `_resolve_head()`；如果没有找到默认 sitemap，才尝试 `https://<host>/robots.txt` 并读取其中的 `Sitemap:` 行。

生产含义：

- 自动探测只是 convenience，不应替代 Source Profile 中显式保存的 sitemap URL；
- 站点若不支持 HEAD，但 GET 可用，默认 sitemap 可能被误判不可用；
- 若已经通过 HEAD 找到默认 sitemap，但随后 GET/解析失败，当前路径不会自然退回到 robots 中的其它 sitemap 作为完整枚举策略；
- 生产 Native SitemapProvider 应同时支持显式 URL、robots sitemap、多个 sitemap root、HTTP validator 和 typed error。

### 3.2 Sitemap XML 解析

实现支持：

- 普通 `<urlset>`；
- `<sitemapindex>`；
- `.gz`；
- namespace-agnostic `local-name()`；
- 相对 `<loc>` 通过 `urljoin()` 转绝对 URL；
- LXML 不可用时回退到 `xml.etree.ElementTree`。

这使其兼容性不错，但解析方式是“整份 response 读入内存后解析”，并非真正 streaming XML parser。

### 3.3 Sitemap Index 并发

遇到 Sitemap Index 时，当前实现会：

```text
sub_sitemaps = 全部子 sitemap
for each sub_sitemap:
    asyncio.create_task(process_subsitemap(...))
result_queue = bounded queue
持续从 result_queue yield URL
```

这里 bounded 的只是 **结果队列**，不是子 sitemap task 数。假设一个 sitemap index 有 5 万个子 sitemap，代码会先创建 5 万个 task；`SeedingConfig.concurrency` 也没有直接约束这些子 sitemap fetch task。

因此生产平台的大 sitemap 必须改成 durable shard：

```text
root sitemap index
 -> 保存 root evidence
 -> materialize sub-sitemap identities
 -> provider_shard 表
 -> Fair Scheduler 小批 lease
 -> 每 shard 独立 Conditional GET / parse / checkpoint
```

不能把“result_queue 有界”误解成“整个 Sitemap Index 的并发和内存都严格有界”。

### 3.4 `discovered_urls` 带来的 O(N) 内存

`_from_sitemaps()` 一边 yield URL，一边把所有 URL append 到 `discovered_urls`，最后为了写 sitemap cache 再整体保存。

所以一个超大站至少同时可能存在：

```text
producer seen set          O(N)
worker results list        O(N)
sitemap discovered_urls    O(N)
sub-sitemap task set       O(number_of_sub_sitemaps)
```

bounded queue 只能限制 producer 与 worker 之间的瞬时缓冲，不能保证总体 O(1) 或严格受控内存。

### 3.5 Smart TTL Cache 的真实语义

v0.9.x 增加了：

```text
cache_ttl_hours
validate_sitemap_lastmod
```

cache JSON 保存：

```text
version
created_at
sitemap_lastmod
sitemap_url
url_count
urls[]
```

但代码在判断 cache 是否有效之前，会先 HEAD 探测 sitemap，再 GET sitemap 内容以计算 `_parse_sitemap_lastmod()`。因此即便最终 cache hit，默认路径仍可能发生完整 sitemap 网络下载。

此外 `_parse_sitemap_lastmod()`：

- 抓取 XML 中全部 `<lastmod>`；
- 使用 `max(lastmods)` 取“最大值”；
- cache 校验时用字符串 `current_lastmod > cached_lastmod` 比较；
- 没有把它转换成统一时区、统一精度的时间对象；
- 也没有使用 HTTP ETag / Last-Modified 做标准 Conditional GET。

所以它可以作为本地 accelerator cache，但不能替代平台增量 Provider 的：

```text
If-None-Match
If-Modified-Since
HTTP 304
body sha256
provider checkpoint
```

`<lastmod>` 应被视为 hint，而不是增量同步的唯一真相。

---

## 4. Common Crawl 实现细节

### 4.1 默认使用“latest index”

首次需要 CC 时，Seeder 调 `_latest_index()`：

```text
GET https://index.commoncrawl.org/collinfo.json
取第 1 个 collection id
本地缓存 latest_cc_index.txt
```

Seeder 实例的默认 TTL 为 7 天，因此“同样的配置”在不同时间可能落到不同 Common Crawl index。

对一次探索没有问题，对历史回填 replay 有问题。生产 Native CommonCrawlIndexProvider 必须显式固定：

```text
cc_index_id
url_pattern
query params
page/shard cursor
query evidence
```

同一次 Backfill 的 retry/replay 不允许静默切到“最新 index”。

### 4.2 CC 查询与缓存

`_from_cc()`：

- 生成 Common Crawl index 查询；
- 用 `httpx.AsyncClient.stream()` 流式读 JSONL；
- 每条记录取 `rec["url"]`；
- 同时把 URL 一行一行写入本地 `.jsonl` cache；
- 下次只要 cache 文件存在且 `force=False`，就直接读取该文件。

关键风险在于：cache 没有单独的 `complete=true`、checksum、terminal marker 或事务提交。

若网络在写入中途失败：

1. 文件可能已经存在并包含部分 URL；
2. 当前调用抛出的 CC 错误又可能被上层 producer 捕获并只记录日志；
3. 下一次不 `force` 时，`path.exists()` 即可触发读 cache；
4. 部分文件可能被当成普通 cache 复用。

所以生产方案必须：

- Attempt-scoped cache directory；
- 失败 attempt cache 不复用；
- 成功后再原子 promotion；
- cache 永远不等同 Coverage Evidence；
- 更推荐由 Native CommonCrawlIndexProvider 自己维护 durable cursor/checkpoint。

### 4.3 CC 错误与完成语义

CC 对 503 有有限重试，但没有输出结构化：

```text
EXHAUSTED
RESULT_LIMIT_REACHED
RATE_LIMITED
ERROR_PARTIAL
```

因此普通 Seeder List 无法证明“已经遍历完这个 index 的约定范围”。

---

## 5. Producer / Worker 与 `max_urls`

### 5.1 去重与 backpressure

Seeder 创建：

```text
queue_size = min(10000, max(1000, concurrency * 100))
asyncio.Queue(maxsize=queue_size)
seen = set()
results = []
```

producer：

```text
source generator -> seen dedup -> queue.put()
```

worker：

```text
queue.get() -> _validate() -> results.append()
```

队列满时 producer 会阻塞，这确实提供了局部 backpressure；但 `seen/results` 仍随候选规模增长。

### 5.2 `max_urls` 是软停止，不是严格事务配额

worker 在处理每个 URL 前检查：

```text
if max_urls > 0 and len(results) >= max_urls:
    stop_event.set()
```

多个 worker 共享同一普通 List；多个 worker 可以在相近时刻看到同一个长度并继续执行 `_validate()`。最终函数还会 `results[:max_urls]` 切片，所以“返回条数”可能守住限制，但内部请求数和中间结果数不一定严格等于该限制。

平台如果需要硬预算，应独立维护：

```text
max_requests
max_observations
max_bytes
max_wall_clock
```

并把达限记录成 typed terminal outcome，而不是只依赖 `max_urls`。

### 5.3 `max_urls` 会让 CC 饥饿

因为 generator 固定：

```text
sitemap -> cc
```

若 sitemap 很快提供超过 `max_urls` 个候选，worker 触发 stop 后 producer 会停止，Common Crawl 分支可能完全没有执行。

因此：

```text
source="sitemap+cc", max_urls=100
```

不能解释为“从 sitemap 和 CC 综合得到前 100 条”，更不能解释为“两种来源都完成后得到 100 条”。

### 5.4 `max_urls` 发生在 BM25 之前

BM25 是所有 worker 完成之后才执行。因此：

```text
query + max_urls=10
```

实际流程是：

```text
先发现/验证到约 10 个候选
 -> 停止更多 discovery
 -> 对这批候选算 BM25
 -> 排序
```

而不是：

```text
发现全候选
 -> 全量 BM25
 -> top 10
```

这对 focused discovery 的结果质量影响很大。若要真正 top-K，要么不要在 discovery 前用小 `max_urls`，要么由平台分阶段做候选采样/排序。

---

## 6. `hits_per_sec` 与并发控制

文档把 `hits_per_sec` 描述为 rate limit，但源码实现是：

```python
self._rate_sem = asyncio.Semaphore(hits_per_sec)
...
async with self._rate_sem:
    await self._validate(...)
```

Semaphore 的语义是“同时有多少 coroutine 进入临界区”，不是“每秒允许多少次请求”。如果单次请求耗时 50ms，`Semaphore(5)` 的吞吐可以远大于 5 req/s；如果单次请求耗时 5s，吞吐又会远低于 5 req/s。

另外它主要包围 `_validate()`，并不自然覆盖 Sitemap Index 中所有 sub-sitemap fetch 的全局 QPS 语义。

生产系统必须在 Seeder 外再做跨 Worker 的：

```text
per_host_qps             Token Bucket
per_host_concurrency     Distributed Semaphore
per_domain_concurrency
per_source_concurrency
Retry-After
origin_backoff_until
```

Seeder 的 local semaphore 只能算第二道保护。

---

## 7. Live Check 与 Partial Head

### 7.1 `live_check=True`

`_resolve_head()`：

- 发 HEAD；
- 2xx 返回成功；
- 301/302/303/307/308 只继续验证一层 redirect；
- 目标再 HEAD 2xx 才算成功；
- HEAD 405、网络错误、多级 redirect 等都会得到 `not_valid`。

这不等于文章无效。很多服务器不支持 HEAD，但 GET 正常。

另外纯 `live_check` 结果中会保留原始 URL，redirect target 不会像 extract-head 分支那样单独保留 `original_url/final_url` 关系，因此不应直接用它决定 URL Identity。

### 7.2 `extract_head=True`

它不是 HTTP HEAD，而是 partial GET：

- `Accept-Encoding: identity`；
- 最多跟随 5 层 redirect；
- 每次读取 4096 bytes；
- 找到 `</head>` 或达到约 64 KiB 时停止；
- 用 LXML 或 regex 解析 title/meta/link/JSON-LD/lang。

这是一个很实用的低成本 metadata probe，但 `status=valid` 本质上仍表示“这个 partial GET 成功”，不表示：

```text
这是文章
正文可抽取
MIME 正确
不是 soft-404
不在登录页
最终 URL 在 scope 内
```

所以平台仍需 Fetch + Scope + MIME + Soft-404 + Quality Gate。

### 7.3 Probe Cache

`live/head` 结果按 URL SHA1 缓存在本地 `~/.cache/url_seeder/...`，默认使用 Seeder 实例 TTL。它适合减少重复 probe，但不应成为 Document 或 Provider 真相。

---

## 8. `filter_nonsense_urls` 的真实行为

默认值为 `True`。源码会过滤一批路径，例如：

- robots / sitemap；
- 常见 utility files；
- hidden path；
- `/wp-admin`、`/login`、`/account`、`/search` 等；
- print 相关路径；
- 某些极短 path。

很多更激进的扩展名过滤在当前源码中反而被注释掉。

这说明所谓 “nonsense” 是版本相关 heuristic，而不是稳定协议。对于全历史 Inventory：

```text
pattern = "*"
filter_nonsense_urls = false
query = null
score_threshold = null
```

先保存 Observation，再由平台版本化 Selection Policy 判断是不是文章候选。

对 focused discovery 可以打开过滤，但被过滤数量和 release 都要记录。

---

## 9. BM25 与文档/实现漂移

### 9.1 Metadata BM25

当：

```text
query != null
extract_head = true
scoring_method = bm25
```

Seeder 会把以下 metadata 拼成文本：

- title；
- description / keywords / author / summary；
- Open Graph；
- Twitter Card；
- Dublin Core；
- JSON-LD name/headline/description/keywords；
- `@graph` 中部分字段。

之后：

```text
whitespace tokenize
 -> rank_bm25.BM25Okapi
 -> corpus 内 min-max normalize 到 0..1
 -> score_threshold
 -> descending sort
```

### 9.2 分数不是跨批次稳定值

因为 min-max normalization 依赖当前候选集合：

- 加入/删除候选会改变其它 URL 的归一化分数；
- 不同日期、不同 cache、不同 `max_urls` 会改变 corpus；
- 所有 raw score 相等时源码直接给所有结果 `0.5`。

因此 Seeder BM25 只能用于：

```text
优先级
探索
focused subset
人工候选排序
```

不能用于 Coverage 的永久保留/删除，也不能拿 0.7 当跨批次稳定业务阈值。

### 9.3 官方文档与 v0.9.2 源码不一致

官方 URL Seeding 文档存在“Smart URL-Based Filtering (No Head Extraction)”章节，写明 `extract_head=False + query` 会根据 domain/path/query param/n-gram 做 URL-based scoring。

但 v0.9.2 `urls()` 的实际分支是：

```text
if query and extract_head and scoring_method == bm25:
    _apply_bm25_scoring(...)
elif query and not extract_head:
    warning
```

官方 `test_query_without_extract_head` 也明确断言：

```text
extract_head=False
query=...
=> all results do NOT contain relevance_score
```

虽然源码内部仍保留 `_calculate_url_relevance_score()`，它只在 `_apply_bm25_scoring()` 的 fallback 中使用，而该函数本身没有在 `extract_head=False` 分支被调用。

生产结论：**行为以 pinned release + fixture/contract test 为准，不以文档描述或参数名字为准。**

---

## 10. Error Handling 与 Completion 的根本问题

### 10.1 Producer 会吞掉异常并结束

producer 外层捕获 Exception 后只记录 error，finally 设置 `producer_done`。已有 worker 仍会把 queue 中的候选处理完成，`urls()` 最终可以正常返回一个 List。

因此可能出现：

```text
Sitemap 成功一部分
CC 中途失败
=> 调用方仍得到非空 List
```

若调用方只看“函数没抛异常 / List 非空”，就会错误宣称 Provider 成功。

### 10.2 子 Sitemap 错误同样可局部吞掉

`process_subsitemap()` 捕获异常、记录日志，最后放 sentinel；父级循环会继续处理其它 shard。没有结构化地告诉调用方“第 17 个 sub-sitemap 失败”。

这对于 best-effort discovery 是合理的，但与 Coverage Provider 需要的语义不同。

### 10.3 需要平台 Completion Attestation

平台定义至少这些 terminal outcome：

```text
EXHAUSTED
NOT_MODIFIED
RESULT_LIMIT_REACHED
REQUEST_BUDGET_REACHED
BYTE_BUDGET_REACHED
TIME_BUDGET_REACHED
RATE_LIMITED
ROBOTS_BLOCKED
ACCESS_DENIED
ERROR_PARTIAL
PROVIDER_ERROR
CANCELLED
```

只有 Adapter 能证明约定范围完成时才允许：

```text
completion_attested = true
```

普通 Seeder List 一律默认 `completion_attested=false`。

---

## 11. `many_urls()` 为什么不适合作为 1000 站调度器

`many_urls(domains, config)` 本质上是：

```text
for domain in domains:
    task = self.urls(domain, config)
await asyncio.gather(*tasks)
return dict(zip(domains, results))
```

对几十个域名非常方便，但 1000+ Source 生产调度不能只靠它：

- 一次创建所有 domain coroutine；
- 全部结果最后聚合进一个 dict；
- 没有 durable lease/checkpoint；
- 没有 Source/Tenant fairness；
- 单个大站会与小站共享进程资源；
- 失败恢复以整个进程调用为边界；
- 不能直接表达跨 Worker 的 host/domain 限流。

生产模式应该是：

```text
Persistent Provider Task
 -> Fair Scheduler
 -> lease 1 Source / small bounded batch
 -> Global Admission
 -> invoke Seeder Adapter
 -> batch upsert Observation
 -> checkpoint / terminal outcome
 -> ack
```

---

## 12. 安全边界

v0.9.2 的 `async_configs.py` 把 `SeedingConfig` 排除在 untrusted network body 可构造类型之外，这个设计方向与平台方案一致：Seeder 原生配置不应由普通 Web 用户直接透传。

平台应提供声明式 intent，例如：

```yaml
discovery_acceleration:
  enabled: true
  sources: [sitemap, cc]
  mode: exhaustive_hint
```

Config Compiler 再根据 Capability Policy 生成受控 SeedingConfig，并强制：

- approved hosts；
- 预算上限；
- 禁止任意代理/CDP/本地路径；
- 不允许用户把 0 当 unbounded；
- 不允许用户控制任意 Common Crawl query；
- probe 前再次做 robots/scope/egress 检查。

---

## 13. 对博客知识库技术方案的具体优化

### 13.1 Seeder 保持 Accelerator，不升级为 Coverage Truth

继续保持：

```text
provider_type = CRAWL4AI_URL_SEEDER
role = DISCOVERY_ACCELERATOR
completion_attested = false by default
```

### 13.2 Coverage 必须拆 Provider

```text
Native CMS/API
Native Sitemap
Native RSS/Atom
Native Archive
Native CommonCrawlIndex
Wayback Gap
Seeder Accelerator
```

### 13.3 大 Sitemap Index 必须平台分片

不使用 Seeder 的“一次创建全部 sub-sitemap task”作为持久调度模型。

### 13.4 增量同步不用 Seeder Smart TTL 替代 HTTP Validator

Native SitemapProvider 使用：

```text
ETag
If-None-Match
Last-Modified
If-Modified-Since
304
body hash
```

Seeder cache 只作加速。

### 13.5 记录 Source Stage

Seeder Run 增加：

```text
source_sequence = [sitemap, cc]
source_started
source_completed
source_failed
source_observation_count
stopped_before_source
```

如果 Adapter 无法从原生 API可靠取得这些信息，则该 Run 仍保持 unattested。

### 13.6 `max_urls` 必须显式记录“限制发生在哪个阶段”

至少区分：

```text
DISCOVERY_CANDIDATE_LIMIT
POST_FILTER_LIMIT
POST_SCORE_TOPK
PLATFORM_OBSERVATION_BUDGET
```

避免把 Seeder `max_urls` 当成统一 top-K 或 Coverage 截断语义。

### 13.7 Web UI 需要暴露行为而非只暴露参数

建议展示：

```text
Seeder Release / commit
source sequence
max_urls
observations before stop
stopped before CC: true/false
query/extract_head/scoring actually applied
docs-contract drift status
local queue max/peak
seen count
result count
sub-sitemap count
tasks created
cache path scope/age
partial cache detected
local semaphore value
measured request rate
completion attested: false
```

---

## 14. 建议的 Contract Test

每个 pinned URL Seeder Release 至少验证：

1. `sitemap` 普通 urlset；
2. sitemap index；
3. 100/1000/10000 sub-sitemap task 数和 peak RSS；
4. `concurrency` 是否约束 sub-sitemap fetch；
5. HEAD 405 / GET 200 的 sitemap probe；
6. robots 中多个 sitemap；
7. 默认 sitemap 已发现但 GET 失败时 fallback 行为；
8. cache valid 时实际发生哪些网络请求；
9. `<lastmod>` 非统一格式、时区和精度；
10. ETag/Last-Modified 是否真正参与 Seeder cache；
11. CC latest index 的选择与缓存 TTL；
12. CC stream 中途失败后的 cache 文件状态；
13. 失败 attempt cache 是否会被后续读取；
14. `sitemap+cc + max_urls` 是否在 CC 前停止；
15. `cc+sitemap` 是否仍按 sitemap-first 执行；
16. 多 worker `max_urls` 内部请求/结果是否超额；
17. `query + max_urls` 是否是 pre-score limit；
18. `extract_head=False + query` 是否产生 relevance score；
19. BM25 all-equal 时是否全部为 0.5；
20. BM25 corpus 变化后的分数漂移；
21. `hits_per_sec` 的真实 req/s 曲线；
22. `many_urls(1000 domains)` task 数、内存和失败传播；
23. producer partial error 是否仍返回 List；
24. sub-sitemap partial error 是否对调用方可见；
25. live_check HEAD 405、单级和多级 redirect；
26. extract_head 64KiB 截断、无 `</head>`、非 HTML MIME；
27. `filter_nonsense_urls` article fixture 误杀率；
28. cache force / TTL / corruption recovery；
29. 全历史模式不得把 Seeder List 自动映射 Completion；
30. 所有测试绑定 version、commit SHA 和 image digest。

---

## 15. 推荐生产用法

### 15.1 全历史 Coverage

不要依赖 Seeder 单独完成全历史。优先：

```text
CMS/API -> Native Sitemap -> RSS -> Archive -> Native Common Crawl -> Wayback
```

Seeder 只额外提供 Observation，配置倾向：

```text
pattern = "*"
filter_nonsense_urls = false
query = null
score_threshold = null
extract_head = false
live_check = false
```

若站点巨大，不建议用 `max_urls=-1` 把全域所有 URL 留在单进程 `seen/results` 中；优先切到 Native Provider 的 durable shard/streaming 模式。

### 15.2 Focused Discovery / Repair

可以充分利用：

```text
pattern
extract_head
query + BM25
score_threshold
live_check
```

但这些结果只影响 priority 和候选选择，不修改 Coverage Truth。

### 15.3 多域名

不要把 1000 个 Source 一次交给 `many_urls()`；由平台 Scheduler 给 Seeder 小批次工作。

---

## 16. 最终判断

Crawl4AI URL Seeding 的价值非常明确：它把“先抓页面再发现 URL”转成“先从 Sitemap/Common Crawl 批量拿 URL，再决定是否抓正文”，对降低带宽、浏览器成本和无效抓取非常有效。

但其实现本质仍是 **单进程、best-effort、面向探索和候选发现的异步工具**。它有局部 bounded queue、cache、metadata probe 和 BM25，却没有生产知识库所需的 durable Completion、持久 Shard、跨 Worker admission、严格 QPS、可重放 Provider Evidence 和强一致硬预算。

因此博客知识库最终方案应继续采用：

```text
URL Seeder = Discovery Accelerator
Native Providers = Replayable Inventory / Coverage
PostgreSQL = Frontier / Task / Provider Truth
Object Storage = Evidence / Raw / IR / Markdown Truth
Fair Scheduler + Redis Lua = Global Admission
Pinned Release + Contract Test = 行为真相
```

本次调研新增的最重要约束是：**不能被“maximum coverage”“rate limit”“bounded queue”“smart cache”“max_urls”“URL-based scoring”等文档名称误导，生产语义必须以固定版本源码和自动化 Contract Test 的实际行为为准。**