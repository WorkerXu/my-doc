# Crawl4AI 官方 AsyncUrlSeeder 博客 URL 发现示例：实现细节与技术原理分析

## 1. 调研对象

- 索引编号：11
- 索引名称：Crawl4AI 官方 AsyncUrlSeeder 博客 URL 发现示例
- 索引地址：https://github.com/unclecode/crawl4ai/blob/main/README-first.md
- 核心实现：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_url_seeder.py
- 当前官方文档：https://docs.crawl4ai.com/core/url-seeding/
- 相关能力：https://docs.crawl4ai.com/core/domain-mapping/
- 调研日期：2026-08-14

本文只分析与“1000 个技术博客站点全量历史文章发现、增量同步、Markdown 知识库”直接相关的实现思想，不把 Crawl4AI 当作整个平台直接照搬。

## 2. 结论先行

AsyncUrlSeeder 最值得吸收的不是某个 API，而是“先发现 URL，再抓正文”的两阶段模型：先通过 Sitemap 与 Common Crawl 快速得到候选 URL，再按 pattern、可达性和 `<head>` 元数据做廉价预处理，最后才进入昂贵的正文抓取。

这个思路非常适合博客知识库，但 Crawl4AI 当前实现更接近单机库级工具，不适合原样承担 1000 站点长期生产调度。主要原因包括：

1. 状态和 cache 以本地文件为主，无法作为多 worker 的 durable 真相源。
2. `many_urls()` 会把多个域直接 `gather`，站点很多时任务数容易被放大。
3. `hits_per_sec` 在实现中主要通过 Semaphore 控制并发，并不是严格意义的“每秒请求数”令牌桶。
4. Sitemap 子文件会并行递归处理，结果队列有界，但子 Sitemap task 数本身仍可能随索引规模增长。
5. Sitemap XML 和 gzip 路径存在整块读入/解压的实现，超大或恶意 Sitemap 不满足生产级硬预算要求。
6. URL Seeder 的 Sitemap 入口发现偏“找到一个可用入口后继续”，而知识库需要 robots 声明、常见路径、历史成功入口等多 root 取并集。
7. 自动 redirect、HEAD 探测、外部历史索引都需要平台级 SSRF、robots、限流和审计约束，不能由库默认行为决定。
8. Common Crawl 很适合补历史 URL，但不能作为“当前页面存在/已更新”的真值。
9. 官方当前又提供了 DomainMapper，能结合 sitemap、Common Crawl、Wayback、crt.sh、probe、robots、feed、homepage 等来源做更广的域映射；该能力适合站点接入 probing，不适合不经审核直接扩大抓取边界。

因此方案中的正确定位应是：

- 抽象 `DiscoveryProvider`，把 AsyncUrlSeeder 作为可替换实现之一；
- 把 URL 发现与正文 Fetch/Capture 完全解耦；
- Sitemap、Feed、Common Crawl、Wayback 等各自持有 durable checkpoint；
- 所有候选 URL 先进入统一 Normalization / Admission / Frontier；
- 平台自己实现分布式 domain limiter、预算、EgressPolicy、outbox、重试和可观测；
- Crawl4AI 继续重点用于 Browser Capture、Markdown/抽取或局部 URL seeding，而不是成为业务状态中心。

## 3. AsyncUrlSeeder 解决的核心问题

传统深爬通常从首页开始逐层跟链接，成本和覆盖率高度依赖站点结构。对于技术博客，很多站点已经通过 Sitemap 或公共历史索引暴露了大量 URL。AsyncUrlSeeder 的思路是先获取“候选 URL 集合”，再决定哪些 URL 值得抓。

典型流程可以抽象为：

```text
Domain
 -> Sitemap / Common Crawl
 -> URL stream
 -> pattern filter
 -> dedupe
 -> optional live check
 -> optional partial head fetch
 -> metadata / relevance
 -> selected URLs
 -> AsyncWebCrawler / downstream crawler
```

对于本项目，应该改成：

```text
Site
 -> multiple DiscoveryProviders
 -> normalized DiscoveredUrl events
 -> Egress / scope / rules admission
 -> PostgreSQL UPSERT Frontier
 -> scheduler
 -> conditional HTTP fetch
 -> Browser fallback
 -> snapshot
 -> extract / Markdown
```

差异在于：生产系统不能把 `List[URL]` 当状态，必须让每一个发现结果成为可恢复、可去重、可审计的 durable record。

## 4. 源码级数据流

`AsyncUrlSeeder.urls(domain, config)` 主要完成以下工作：

1. 解析 `SeedingConfig`，包括 source、pattern、live_check、extract_head、concurrency、hits_per_sec、max_urls、query、cache 参数等。
2. 根据 `source` 组合 Sitemap 和 Common Crawl。
3. 两个来源以 async generator 形式逐个 yield URL。
4. producer 把 URL 写入一个有界 `asyncio.Queue`。
5. 使用进程内 `seen` 集合去重。
6. 多个 worker 从 queue 取 URL，执行 `_validate()`。
7. `_validate()` 可选择：
   - 不请求页面，只返回候选；
   - HEAD live check；
   - 只下载页面前部直到 `</head>` 或达到字节上限，并提取 title/meta/link/JSON-LD/lang。
8. 如果配置 query，可对 head metadata 做 BM25 评分。
9. 最终返回列表。

这里最有价值的实现点是 bounded queue：producer 在 queue 满时会阻塞，形成 backpressure，避免“发现速度远大于验证速度”时无限积累内存。

生产方案需要进一步把这个 bounded queue 扩展为三层：

```text
Provider 内部 bounded stream
 -> batch UPSERT PostgreSQL Frontier
 -> Redis Streams durable delivery
 -> Worker 本地 bounded execution
```

这样进程重启后仍可以继续，而不是依赖当前 event loop 中的 queue。

## 5. Sitemap 发现实现

### 5.1 当前实现思路

AsyncUrlSeeder 会尝试常见 Sitemap 地址，例如 `/sitemap.xml`、`/sitemap_index.xml`；找不到时再读取 robots.txt 中的 `Sitemap:`。它支持 Sitemap index，并递归处理子 Sitemap。

还提供 Sitemap cache：

- cache 带创建时间；
- 可设置 TTL；
- 可基于 Sitemap 内的 lastmod 做简单失效判断；
- `force=True` 可以绕过缓存。

### 5.2 对知识库有价值的点

1. Sitemap 是博客全量 URL 的最高性价比来源之一。
2. Sitemap index 能让超大站点按子文件拆分，天然适合增量 checkpoint。
3. lastmod 可以当优先级提示。
4. robots.txt 的 Sitemap 声明应纳入发现源。
5. Sitemap 与 Common Crawl 的并集能够显著补充覆盖。

### 5.3 不能直接照搬的点

#### 5.3.1 多 root 完整性

生产系统不能“找到一个 sitemap 就结束入口探测”。应把以下来源取并集：

- robots.txt `Sitemap:`；
- 常见路径；
- 站点人工配置；
- 历史成功 root；
- 页面或 Feed 提示；
- DomainMapper/probing 发现。

每个 root 独立维护健康度。一个 root 失败不能让其它 root 失效。

#### 5.3.2 XML 与 gzip 必须流式

源码存在把响应体整体拿到内存、再解压、再构造 XML tree 的路径。对于正常站点很方便，但生产环境必须防：

- 超大 sitemap；
- gzip bomb；
- XML entity/DTD；
- 极深 sitemap index；
- 巨量 `<url>` / `<sitemap>`；
- 循环引用。

生产实现应是：

```text
HTTP stream
 -> compressed bytes limit
 -> bounded decompressor
 -> decoded bytes / ratio limit
 -> secure pull parser
 -> batch emit
```

并对元素数、URL 数、child 数、递归深度、总请求数、wall time 设置硬预算。

#### 5.3.3 child 并发必须有界

AsyncUrlSeeder 对一个 sitemap index 会为子 sitemap 创建并行任务。结果 queue 是有界的，但子 task 数仍可能很多。

生产系统应把 child sitemap 自己也放入 frontier/queue：

```text
root sitemap
 -> emit child sitemap jobs
 -> worker pool concurrency 2~4/domain
 -> each child stream URLs
```

而不是 `create_task(all_children)`。

#### 5.3.4 lastmod 不是变化真值

Sitemap 的 `<lastmod>` 经常不准确。有些站点每次部署都刷新全部 lastmod，有些站点完全不更新。

因此它只能用于：

- fetch priority；
- “可能变化”的 hint；
- reconciliation 排序。

最终正文变化要由：

- HTTP ETag；
- Last-Modified；
- 304；
- body hash；
- article content hash

共同确认。

## 6. Common Crawl 实现

### 6.1 当前实现

AsyncUrlSeeder 会先获取最新 Common Crawl collection id，再向 index API 请求指定 domain/pattern 的记录。返回是 JSON Lines，源码是边流式读取边取出 `url`，同时写入本地 jsonl cache。

这一点非常适合历史 URL discovery，因为不需要真的访问目标博客就能得到过去被公共爬虫见过的地址。

### 6.2 在本项目中的正确作用

Common Crawl 应被定义为“历史候选 URL Provider”，不是页面抓取 Provider。

适合：

- 补 Sitemap 已删除的旧文章 URL；
- 发现历史路径模式；
- 发现站点迁移前后的 URL；
- 对“全量历史”做 coverage reconciliation。

不适合直接判定：

- 页面当前仍存在；
- 当前正文版本；
- canonical；
- 页面发布时间；
- 页面是否应该马上重抓。

### 6.3 增量设计

本地 cache 文件不能作为生产 checkpoint。应该记录：

```text
provider = commoncrawl
collection_id
scope/domain
query pattern/version
last_scan_at
last_success_at
result_count
accepted_count
```

每次出现新 collection 时才做新一轮扫描。老 collection 可低频回扫以补漏，不应该每天全量查询。

对于 Wayback 也采用相同思想，但另外记录 timestamp/capture evidence。

## 7. Pattern 过滤与 URL Admission

AsyncUrlSeeder 的 `pattern` 用 fnmatch 做简单预过滤，速度快，适合在真正抓页面前降低候选数量。

但对于知识库不能仅依赖一个 glob，因为容易误杀：

- `/blog/*` 可能漏掉 `/posts/*`、`/2024/...`；
- query 参数可能表示分页，也可能只是 tracking；
- 同一个 URL 的 host、scheme、trailing slash、percent encoding 可能不同。

生产系统应该把“发现”和“接纳”分开：

```text
raw URL
 -> syntax parse
 -> security normalization
 -> site scope check
 -> canonicalizer_version
 -> include/exclude rule
 -> content-type/path heuristic
 -> accepted/rejected + reason
 -> Frontier UPSERT
```

所有被过滤的 URL 最好保存统计或 rejection reason，避免规则错误时“悄悄漏数据”。

## 8. live_check 与 `<head>` 部分下载

### 8.1 live_check

AsyncUrlSeeder 可以用 HEAD 检查 URL 可达性。这个能力适合在“很旧的历史候选”上做廉价验证，但不能作为最终 Fetch 结果：

- 很多服务器不支持 HEAD；
- HEAD 和 GET 的 ACL 可能不同；
- SPA/软 404 可能一直 200；
- CDN 对 HEAD 的行为可能与正文不同。

生产策略：

```text
HEAD 可选探测
 -> 2xx: 只提升优先级，不视为正文成功
 -> 403/405/异常: GET fallback
 -> 真正抓取仍走 conditional GET
```

### 8.2 partial `<head>` fetch

AsyncUrlSeeder 的 head extraction 实际上是发 GET，但只消费页面头部，遇到 `</head>` 或最大字节数就停止，然后提取：

- title；
- meta；
- OpenGraph；
- link rel；
- JSON-LD；
- html lang。

这是很实用的“轻量分类器”，可以用于：

- 判断页面是否像 article；
- 提前拿 canonical candidate；
- 提前拿发布时间/作者 hint；
- 检测 login/captcha/soft-404 模板；
- probing 阶段生成站点规则。

但本项目目标是全量历史文章，不能用 BM25 或 metadata score 直接丢弃低相关 URL。它们最多用于排序和采样，不用于决定“是否属于历史知识库”的最终真值。

## 9. BM25 评分能力

当 `extract_head=True` 且传入 query 时，AsyncUrlSeeder 可以把 `<head>` 文本上下文作为 BM25 文档计算 relevance score。

这对“搜索后只抓一部分页面”的场景很有价值，但我们的任务是全量博客知识库，默认不应该开启语义裁剪。

可保留三个用途：

1. probing 阶段对几百个候选页做文章概率排序；
2. 运营 Web 中按主题快速找样本；
3. 资源紧张时调整 backfill priority，但不能永久排除。

## 10. Cache 机制与生产改造

AsyncUrlSeeder 有多类本地 cache，包括 Common Crawl、Sitemap、live/head metadata。

优点：

- 开发体验好；
- 重复运行快；
- 单机脚本减少网络请求。

问题：

- 多 worker 不共享；
- pod 重建可能丢；
- 无法做统一审计；
- 规则/version 变化时 cache identity 不够表达全部业务语义；
- 不能替代增量状态。

生产系统应分三层：

```text
L1 worker memory cache     只优化秒级重复
L2 Redis optional cache    只优化热点和限流协作
L3 PostgreSQL/S3 durable   业务真相 + snapshot
```

Sitemap/Feed validator、Common Crawl collection checkpoint、页面 ETag、article version 都进入 PG。

## 11. 并发模型分析

### 11.1 有界 producer-consumer 值得保留

`urls()` 里 queue 大小跟 concurrency 相关，并有上限。这种设计比把所有 URL 一次性 `gather()` 安全。

生产系统继续使用：

- bounded queue；
- micro-batch；
- streaming result；
- backpressure。

### 11.2 `many_urls()` 不应直接用于 1000 站

`many_urls(domains)` 会为所有 domain 建立 coroutine 并 `asyncio.gather()`。如果 1000 个站点，每个 `urls()` 内部再启动 N 个 worker，就会把 task 数放大到非常高。

另外同一个 `AsyncUrlSeeder` 实例中，`self._rate_sem`、部分 config state 会在并行 `urls()` 调用间共享/覆盖，这种对象级可变状态并不适合作为 1000 站统一调度器。

正确方式：

```text
site scheduler
 -> lease 10~50 sites
 -> per-site discovery job
 -> global worker pool
 -> distributed domain limiter
```

每个 worker 一次处理有限 job，不在一个 Python 对象里同时展开 1000 个站。

## 12. `hits_per_sec` 的实现陷阱

源码把 `hits_per_sec` 映射成 `asyncio.Semaphore(hits_per_sec)`，worker 在 `_validate()` 前进入 semaphore，在完成后释放。

这限制的是“同时进行多少次验证”，并没有表达时间维度，所以严格来说不是 token bucket/leaky bucket 意义的 requests per second。

例如一个请求 20ms 完成，即使 semaphore 容量是 5，理论吞吐可以远高于 5 请求/秒。

这也是本项目必须自己实现平台级限流的原因。

建议：Redis + Lua token bucket：

```text
key = domain
capacity
refill_rate
last_refill
current_tokens
concurrency_leases
retry_after_until
```

任何 HTTP、Sitemap、Feed、Browser 顶层、Browser 子请求、媒体请求在真正发出网络包前都要取得 token/lease。

## 13. Redirect、SSRF 与外部输入安全

AsyncUrlSeeder 的部分路径会自动 follow redirects。这对库使用方便，但多租户/管理后台场景风险较大。

所有从远端获得的 URL 都必须重新验证：

- robots 的 Sitemap URL；
- Sitemap child URL；
- Common Crawl URL；
- redirect Location；
- canonical；
- `<base href>`；
- asset URL；
- Browser iframe/fetch/XHR/image。

统一 `EgressPolicy`：

1. 仅 http/https；
2. 拒绝 userinfo、异常端口和混淆 host；
3. DNS 后拒绝 private/loopback/link-local/metadata 等地址；
4. redirect 每 hop 重做校验；
5. 默认不允许跨站点 scope；
6. 网络层再用 egress proxy/namespace 做第二道防线。

## 14. 与 DomainMapper 的关系

截至当前官方文档，Crawl4AI 还提供 DomainMapper。它相比 AsyncUrlSeeder 扫描范围更广，可综合：

- Sitemap；
- Common Crawl；
- Wayback；
- Certificate Transparency；
- common path probe；
- robots；
- Feed；
- homepage links。

它还提供 subdomain discovery 和 soft-404 fingerprint。

这对我们的“新站接入”很有价值，但要注意它会发现 docs、api、app、staging、admin 等 host，而用户需求是“技术博客”，不是“整个组织所有系统”。

因此推荐只在 `probing` 使用 DomainMapper 类能力：

```text
base domain
 -> broad domain mapping with hard budget
 -> candidate hosts/sources
 -> classify host purpose
 -> operator/rule approval
 -> accepted hosts become site scope
```

绝不能：

```text
DomainMapper found host -> 自动加入正式全量抓取
```

特别是 crt/probe/robots-disallow 得到的地址，只能作为“发现证据”，不能绕过 robots 或访问控制。

## 15. 软 404 思路值得加入

DomainMapper 会先访问一个保证不存在的随机路径，记录 title/body fingerprint，再用它识别“所有路径都返回同一个 200 SPA shell”的站点。

本项目可吸收这个思路：

- probing 时建立 soft-404 fingerprint；
- fetch 200 后若正文模板/hash 与 soft-404 高度一致，标记 `soft_404_suspected`；
- 不直接建 article version；
- Web 展示站点 soft-404 模板；
- 定期刷新 fingerprint。

注意 fingerprint 本身也必须在 domain limiter 和 robots/预算约束中执行。

## 16. 推荐的 DiscoveryProvider 生产接口

```python
class DiscoveryProvider(Protocol):
    async def scan(
        self,
        site: Site,
        source: DiscoverySource,
        checkpoint: Checkpoint | None,
        budget: DiscoveryBudget,
    ) -> AsyncIterator[DiscoveredUrl]:
        ...
```

`DiscoveredUrl` 推荐字段：

```text
raw_url
normalized_url
provider
source_id
source_url
evidence
lastmod_hint
pubdate_hint
archive_timestamp
discovered_at
depth
parent_url
confidence
```

Provider 只负责发现，不直接决定抓取。

统一 Admission：

```text
DiscoveredUrl
 -> URL syntax/security normalize
 -> scope/allowed_host
 -> include/exclude
 -> known asset/non-page filter
 -> dedupe
 -> frontier UPSERT
 -> rejected reason/metrics
```

## 17. AsyncUrlSeeder 适配方式

可以提供 `Crawl4AIUrlSeederProvider`，但只使用它的“source adapter”价值：

```text
Crawl4AIUrlSeederProvider
  - source=sitemap
  - source=cc
  - pattern as early prefilter
  - optional head extraction in probing
```

平台外围必须包住：

- distributed domain limiter；
- EgressPolicy；
- durable checkpoint；
- budget；
- retry policy；
- metrics；
- cancellation；
- bounded site concurrency。

对于超大 Sitemap，建议优先使用自研流式 SitemapProvider，而不是直接依赖 AsyncUrlSeeder 当前整块 XML/gzip 处理。

## 18. 全量历史回灌的推荐流程

```text
1. Site probing
2. robots + Feed + Sitemap root union
3. optional DomainMapper-style source discovery
4. approve host scope/rules
5. Feed/Sitemap stream to Frontier
6. Common Crawl / Wayback history candidate scan
7. normalize + dedupe
8. conditional GET when validators exist
9. HTTP body snapshot
10. extraction quality gate
11. Browser fallback if needed
12. Article IR -> Markdown
13. version + provenance
14. low-frequency reconciliation
```

优先级建议：

```text
Feed new item
> Sitemap recently changed/new URL
> known article URL due for refresh
> Common Crawl historical candidate
> Wayback-only deleted candidate
```

这样新内容不会被历史 backfill 淹没。

## 19. 增量同步推荐流程

每个 Provider 有自己的 checkpoint：

### Feed

- ETag/Last-Modified；
- guid/url/pubdate cursor。

### Sitemap

- root/child ETag/Last-Modified；
- last successful scan；
- child tree health；
- lastmod hint。

### Common Crawl

- collection id；
- query/version；
- last completed collection。

### Wayback

- CDX cursor/时间范围；
- last scan watermark。

页面层：

- ETag；
- Last-Modified；
- body hash；
- article content hash。

Provider checkpoint 和页面 checkpoint 分离，避免把“没发现变化”误认为“正文没有变化”。

## 20. 对 Web 管理端的新增要求

从本次调研得到的 Web 功能应包括：

1. Discovery source 列表与每个 source 的状态。
2. Sitemap root/child 树和每个节点 validator。
3. Common Crawl/Wayback collection checkpoint。
4. candidate URL accepted/rejected 数量与原因。
5. Domain mapping 发现的 host 候选，必须人工/规则批准后入 scope。
6. soft-404 fingerprint 与命中统计。
7. 每个 provider 的请求数、字节、错误、budget_exhausted。
8. domain limiter wait 和 Retry-After 状态。
9. head metadata probing 结果和文章样本。
10. 同一 URL 被多个来源发现时的 source attribution。

## 21. 对现有技术方案的具体优化决策

本次调研后，技术方案需要明确加入以下规则：

### 决策 A：Discovery 分成“Source Discovery”和“URL Discovery”

Source Discovery 找 Feed、Sitemap、候选 host；URL Discovery 才产出文章候选 URL。

这样 DomainMapper 类能力可以服务于接入，而不会越过 scope 自动抓取未知子域。

### 决策 B：URL Seeder 是 provider，不是主调度器

不使用 `many_urls(1000 domains)` 作为整个平台入口。

### 决策 C：严格限流由平台实现

不把库里的 `hits_per_sec` 当生产级 QPS 保证。

### 决策 D：超大 Sitemap 使用自研流式 Provider

保留 Crawl4AI Sitemap seeder 作为普通站点/辅助路径；大站或安全边界要求高的路径走 secure streaming parser。

### 决策 E：metadata/head 只用于 probing/priority

全量知识库不以 BM25 relevance 永久删 URL。

### 决策 F：Common Crawl/Wayback 是历史 coverage source

不能直接覆盖当前 canonical/version。

### 决策 G：加入 soft-404 fingerprint

减少 SPA 200 假页面进入知识库。

### 决策 H：多来源 attribution 要进入数据模型

同一 normalized URL 可以同时来自 Feed/Sitemap/CC/Wayback/homepage，不能只保留一个 `discovered_from` 字符串。应至少有 `frontier_url_sources` 关系表或等价结构。

## 22. 最终评价

AsyncUrlSeeder 是非常好的 URL discovery 参考实现，尤其值得采用：

- Sitemap + Common Crawl 双源；
- discovery-first；
- pattern prefilter；
- bounded queue；
- head-only metadata；
- URL 级 cache；
- 可选 relevance scoring。

但它的默认目标仍是“库用户方便地快速得到 URL 列表”，而我们的目标是“1000+ 站点、多年运行、全量历史 + 增量同步 + Web 管理”的服务化系统。

所以最终策略不是把 Crawl4AI 换掉，也不是把整个系统建立在 AsyncUrlSeeder 上，而是：

> 把 Crawl4AI 的 URL seeding、Browser crawling、Markdown/抽取能力拆成可替换 Adapter；把 durable 状态、任务编排、限流、安全、版本、checkpoint、Web 管理牢牢放在平台层。

这既保留了 Crawl4AI 快速演进带来的收益，也避免库内部实现细节变成系统扩展性的上限。
