# Crawl4AI 新闻正文抓取器：实现细节与技术原理分析

## 1. 项目定位

调研对象：`crawl4ai-news-fetcher`（编号 38）

- 索引名称：Crawl4AI 新闻正文抓取器
- 原始项目地址：https://github.com/Amit506/crawl4ai_news_fetcher
- PyPI：https://pypi.org/project/crawl4ai-news-fetcher/
- 当前可验证发布：0.1.0，2025-10-25
- 许可证：MIT
- Python：>= 3.8

公开发布说明把它定位为一个建立在 Crawl4AI 之上的新闻正文抓取包，能力集中在四点：

1. 对 Google News、短链接等中间链接做智能跳转解析。
2. 用 HTTP、HTML 解析和浏览器自动化多种方法解析最终目标 URL。
3. 抓取目标文章并输出 clean Markdown / HTML。
4. 基于 BM25 按查询对正文做相关性过滤。

当前项目 GitHub 首页无法直接读取，因此本文不把不可验证的内部函数名、类名或调用顺序当作事实。对实现的分析分为两层：项目发布页明确声明的能力，以及其依赖 Crawl4AI 当前公开源码可以验证的底层机制。

## 2. 最值得吸收的设计：把 URL 解析和正文抓取拆成两件事

普通爬虫常把“请求一个 URL，跟随 3xx，拿到最终 HTML”视为一次抓取，但新闻聚合、RSS、Newsletter、搜索结果页中经常出现包装链接：

```text
feed/search URL
  -> tracking URL
  -> short URL
  -> HTML meta refresh
  -> JavaScript redirect
  -> article URL
```

这类场景的关键问题不是正文抽取，而是先回答：**这个候选 URL 最终代表哪一个可抓取资源？**

因此，适合知识库平台的模型不是：

```text
candidate_url -> fetch -> final_url -> document
```

而是：

```text
candidate_url
  -> redirect/link resolution evidence
  -> resolved fetch target
  -> fetch observation
  -> canonical identity evidence
  -> document version
```

这样可以避免把“最终请求地址”“canonical 地址”“来源里看到的包装地址”混为一谈。

## 3. 多策略跳转解析的技术原理

### 3.1 第一层：HTTP Redirect

最低成本路径使用 HTTP 客户端完成：

```text
GET/HEAD A
302 Location: B
301 Location: C
200 C
```

每一跳都应该记录为不可变 Evidence：

```text
from_url
to_url
status_code
location_header
observed_at
method
response_headers_hash
```

不能只保存最终 URL，因为历史上同一个短链可能改变指向；如果只保留 `final_url`，后续无法审计某次同步为何落到某篇文章。

生产实现需要：

- 限制最大跳数，例如 10。
- 检测循环，例如 `A -> B -> A`。
- 每跳重新执行 SSRF / egress / allowed-origin 策略。
- 不默认信任 HEAD；部分站点 HEAD 与 GET 行为不同，必要时降级到小响应 GET。
- 保存 301/308 与 302/303/307 的语义差异，但知识库抓取通常仍以 GET 获取最终正文。

### 3.2 第二层：HTML Redirect / Link Unwrap

HTTP 返回 200 不代表已经到达文章页。聚合器可能返回一个 HTML 页面，通过以下机制再导航：

- `<meta http-equiv="refresh" ...>`
- 中间页中的目标链接
- JavaScript 配置中的目标 URL
- 特定平台的包装参数，例如 `?url=...`

项目发布信息明确提到 HTML parsing，因此平台应把这类能力实现为 `LinkUnwrapAdapter`，而不是把大量站点特例塞进 HTTP Worker。

建议接口：

```text
resolve(html, source_url, profile) ->
  target_candidates[] + evidence + confidence
```

每个 Adapter 可声明：

```text
host_patterns
query_param_rules
css/xpath_rules
meta_refresh_enabled
js_blob_patterns
confidence
```

注意：`<link rel="canonical">` 不应当当作跳转直接跟随。它是文档身份线索，应进入 Identity Evidence；把 canonical 当 redirect 容易误抓跨域转载源、首页或错误 canonical。

### 3.3 第三层：Browser Redirect

如果最终目标依赖 JavaScript 执行，才使用 Playwright / Crawl4AI Browser：

```text
HTTP resolve failed
  -> open browser page
  -> wait for bounded navigation condition
  -> observe navigation chain / current URL
  -> stop after stable window
```

浏览器解析必须是最后手段，因为它比 HTTP 更耗 CPU、内存和 browser-seconds，且并发时更容易出现 page/context 生命周期问题。

对于 1000 个博客的长期同步，Browser 不应该成为默认 URL 解析器，只对以下情况启用：

- HTML 明确存在 JS 跳转迹象；
- 域名 Profile 标记必须执行 JS；
- HTTP/HTML 两层无法得到可信目标；
- 历史成功证据显示 Browser 解析成功率明显更高。

## 4. Crawl4AI 正文与 Markdown 处理机制

项目公开说明中的“clean Markdown / HTML”依赖 Crawl4AI。Crawl4AI 当前公开实现把 Markdown 生成做成独立策略，并支持从 cleaned HTML、raw HTML、fit HTML 等不同内容源生成 Markdown。

对于博客知识库，应该固定两层输出：

```text
Canonical Clean Markdown   = 尽可能忠实的正文投影
Query Focused Markdown     = 面向某个查询/任务的裁剪投影
```

两者不能混在一起。Canonical Markdown 需要长期可重建、可追溯，不能因为某次查询关键词不同就丢掉正文段落。

正文抽取推荐顺序：

1. 站点明确 selector / API 正文。
2. JSON-LD / Article schema 作为元数据与正文候选。
3. Trafilatura / Readability / LXML 确定性抽取。
4. Crawl4AI clean Markdown。
5. Browser rendered DOM 后重复确定性抽取。
6. LLM extraction 只处理极少数结构化异常，不作为默认正文清洗器。

## 5. BM25 查询聚焦过滤的实现原理

项目公开说明明确使用 BM25 做 query-focused filtering。Crawl4AI 当前 `BM25ContentFilter` 的公开源码可验证如下逻辑：

1. 如果调用者提供 `user_query`，直接作为查询。
2. 未提供时，从 title、h1、meta keywords、meta description 获取查询描述；仍不足时取首个较长段落作为回退。
3. 按 DOM 块提取文本 chunk，同时保留块顺序和标签信息。
4. 对 chunk 和 query 分词、可选 stemming，并清理停用词/噪声 token。
5. 使用 BM25Okapi 计算每个 chunk 的相关性。
6. 再按 HTML 标签加权，例如标题标签比普通内容具有更高权重。
7. 过滤低于阈值的 chunk。

BM25 的核心是同时考虑词频、逆文档频率和文档长度归一化：

```text
score(D,Q) = Σ IDF(q) * tf(q,D) * (k1 + 1)
             -------------------------------------
             tf(q,D) + k1 * (1 - b + b*|D|/avgdl)
```

它适合“从一个页面里挑出和查询最相关的块”，原因是：

- 比纯关键词包含关系稳健；
- 不需要 embedding/LLM；
- CPU 成本低，可离线重跑；
- 可解释，能够记录 chunk score 和 threshold。

但它**不适合作为知识库真相层正文清洗策略**。如果查询是“Redis”，文章中与 Redis 无关但仍属于原文的重要背景会被过滤掉。正确用法是把它变成 Projection：

```text
Canonical IR
  -> Clean Markdown
  -> Chunk
  -> BM25 Query View / Search Snippet
```

而不是：

```text
HTML -> BM25 filter -> 唯一 Markdown 真相
```

## 6. 异步并发的适用边界

项目声明 fully asynchronous。对 1000 个站点，异步的真正价值是隐藏网络等待，而不是无限提高并发。

生产并发应分层控制：

```text
global concurrency
  -> per-domain concurrency
  -> per-source rate limit
  -> browser concurrency
  -> task deadline
```

建议 HTTP Worker 大并发、Browser Worker 小并发独立部署。每次 Route Attempt 都要消耗预算：request 数、response bytes、browser seconds，避免某个异常站点拖垮整个系统。

异步任务还必须持久化 checkpoint；只依赖进程内 coroutine 状态，一旦进程退出就无法可靠恢复。

## 7. 对现有博客知识库方案的具体优化

### 7.1 新增 Redirect Resolution Plane

在 Discovery 与 Fetch 之间新增逻辑阶段：

```text
Discovery
  -> URL Resolution
  -> Fetch Route
  -> Extraction
  -> Identity
```

并新增 Worker：

```text
redirect-resolution-worker
```

它负责：

- HTTP redirect chain；
- wrapper 参数解包；
- HTML meta refresh；
- 站点 Profile 中的 LinkUnwrapAdapter；
- 必要时触发 Browser resolution；
- 输出目标候选和证据，不直接决定 Document Identity。

### 7.2 新增数据模型

`redirect_resolution_attempt`

```text
id
task_id
source_id
input_url_id
resolver_release_id
method
state
outcome_code
hop_count
resolved_url_id
confidence
started_at
finished_at
```

`redirect_edge`

```text
id
resolution_attempt_id
seq
from_url_id
to_url_id
evidence_type
http_status
raw_location
snapshot_id
observed_at
```

`evidence_type` 至少包含：

```text
HTTP_LOCATION
META_REFRESH
WRAPPER_PARAM
HTML_TARGET_LINK
BROWSER_NAVIGATION
```

canonical、og:url 等不要进入 redirect_edge，进入独立的 Identity Evidence。

### 7.3 Resolution Cache

对包装链接建立可过期缓存：

```text
(input_url_hash, resolver_release_id)
  -> resolved target + evidence + expires_at
```

缓存必须带 TTL，不能永久缓存，因为聚合链接和短链可能变更。

对 301/308 可使用更长 TTL；对临时跳转、Browser 跳转使用更短 TTL。若 Source Profile 或 Resolver Release 变化，应自然生成新的 cache key。

### 7.4 安全边界

每一跳都必须重新校验：

- scheme 只允许 http/https；
- DNS/IP 是否全球可路由；
- 是否命中私网、link-local、metadata 地址；
- 当前 source 是否允许跨域；
- 最大跳数、最大解析时间、最大 response bytes；
- Browser 导航也执行同样 egress 规则。

重定向链是 SSRF 常见绕过面，因此不能只对起始 URL 做一次安全检查。

### 7.5 Web 管理增强

Source/URL 页面增加“解析链”视图：

```text
feed url
  --302--> tracking
  --meta-refresh--> article
  --canonical evidence--> canonical article
```

管理员可以看到：

- 每一步证据；
- 使用的 resolver release；
- 是否发生跨域；
- 最终抓取 target；
- canonical identity 是否与 fetch target 一致；
- 哪一步失败以及是否触发 Browser fallback。

这对于排查“为什么同一篇文章出现两个文档”“为什么抓到聚合器而不是原文”非常关键。

## 8. 不建议直接照搬的部分

### 8.1 不把编号 38 作为核心生产依赖

目前可验证版本只有 0.1.0，且处于 Alpha，项目 GitHub 地址当前不可直接读取。它适合作为设计样本，不适合承担平台唯一跳转解析层。

生产方案应自行定义稳定的 Resolver Contract，并允许底层实现替换。

### 8.2 不把 BM25 Fit Markdown 当唯一正文

BM25 输出是任务相关视图，不是来源事实。必须保留完整 Canonical IR / Clean Markdown，再把 BM25 作为可重建投影。

### 8.3 不让 Browser 处理所有跳转

浏览器是昂贵 fallback。多数 3xx、wrapper 参数和 meta refresh 都可以在 HTTP/HTML 层完成。

### 8.4 不把 final URL 当 canonical identity

final URL 只证明网络导航终点；canonical identity 需要综合：

- rel=canonical；
- og:url；
- redirect evidence；
- normalized URL；
- platform ID；
- 内容 hash；
- 发布时间/标题等弱特征。

## 9. 推荐的最终处理链

```text
Provider Candidate
      |
      v
URL Normalize + Scope Check
      |
      v
Redirect Resolution
  HTTP -> HTML unwrap -> Browser fallback
      |
      +------> Redirect Evidence Store
      |
      v
Resolved Fetch Target
      |
      v
Conditional HTTP Fetch
      |
      +-- quality fail / JS required --> Browser Fetch
      |
      v
Immutable Snapshot
      |
      v
Multi-candidate Extraction
      |
      v
Quality + Identity
      |
      v
Canonical IR / Document Version
      |
      +--> Clean Markdown Projection
      +--> Chunk Projection
      +--> BM25 Query View
      +--> Full-text Index
      +--> Embedding Index
```

## 10. 结论

编号 38 对当前方案最有价值的不是 BM25 本身，而是提醒平台把“链接解析”提升为一等公民：来源候选 URL、跳转链、实际抓取 URL、canonical identity 是四种不同事实。

把 Redirect Resolution Plane、可审计 redirect evidence、按成本逐级降级的 HTTP/HTML/Browser resolver，以及 BM25 仅作为查询投影引入后，方案会更适合 RSS/聚合器/Newsletter/短链大量存在的 1000 站点场景，同时减少重复文档、错误身份合并和无谓 Browser 成本。

## 参考来源

- 调研索引：https://incorporated-jail-assigned-figure.trycloudflare.com/
- PyPI 发布页：https://pypi.org/project/crawl4ai-news-fetcher/
- Crawl4AI：https://github.com/unclecode/crawl4ai
- Crawl4AI `content_filter_strategy.py`：BM25ContentFilter 当前公开实现
- Crawl4AI `markdown_generation_strategy.py` / Markdown 文档：Markdown Projection 机制
