# LLM_trader／Semantic Signal LLM：基于新闻与 RAG 的语义交易信号系统

## 1. 调研对象

- 编号：57
- 项目：LLM_trader / Semantic Signal LLM
- 地址：https://github.com/qrak/LLM_trader
- 调研基线：`master` 分支，提交 `8a0e60b2bb2638dbbd4212ce617a34111e35211f`
- 重点：RSS 新闻采集、Crawl4AI 正文补全、异步并发、降级策略、URL/标题去重、正文清洗、缓存更新，以及这些机制对“1000 个技术博客全量历史抓取 + 增量同步 + Web 管理”方案的可迁移价值。

本项目本身是一个交易系统，新闻采集只是其中的 RAG 子系统，因此它不是一个可直接拿来承担 1000 站博客归档的完整爬虫平台。真正有价值的是它在“小成本 Feed 获取 → 仅对正文不足的条目做页面 enrichment → Crawl4AI 失败后回退普通 HTTP → 下游继续工作”这一条链路上的工程实现，以及其中暴露出来的规模化风险。

## 2. 关键源码

本次重点阅读以下文件：

- `src/rag/news_ingestion/rss_primitives.py`
- `src/rag/news_ingestion/rss_provider.py`
- `src/rag/news_ingestion/crawl4ai_enricher.py`
- `src/rag/news_ingestion/schema_mapper.py`
- `src/rag/news_manager.py`
- `config/config.ini.example`
- `tests/test_crawl4ai_enricher.py`
- `tests/test_rss_provider_contract.py`

整体调用链为：

```text
RSS/Feed
  │
  ▼
rss_primitives.fetch_source()
  │ 解析 RSS、标准化 URL、抽取 feed 内正文/摘要
  ▼
RSSCrawl4AINewsProvider._fetch_raw_items()
  │ 多源并发
  ▼
URL 去重 + 标题二次去重 + 时间排序
  │
  ├─ 正文已经足够长 ───────────────────────────────┐
  │                                               │
  └─ 正文过短 → Crawl4AIEnricher.enrich_items()   │
                     │                            │
                     ├─ Crawl4AI/Playwright      │
                     │     │                     │
                     │     └─ 失败/超时           │
                     │                            │
                     └─ aiohttp + HTML extractor │
                                                   │
                                                   ▼
                                      schema_mapper.to_article_schema()
                                                   │
                                                   ▼
                                            NewsManager
                                                   │
                                       本地缓存 / RAG 消费
```

## 3. RSS 入口：先吃低成本结构化数据

`rss_primitives.py` 先把 Feed 当成结构化采集入口，而不是一上来就开浏览器。

默认配置了 CoinDesk、CoinTelegraph、Decrypt、CryptoSlate 四个 RSS；`get_sources()` 支持通过配置替换 URL，也支持只启用部分 source。对每个条目读取：

- `title`
- `link`
- `guid`
- `description`
- `content:encoded`
- `dc:creator`
- `pubDate`
- `category`

其中正文来源的选择顺序是：

```text
content:encoded > description > 空
```

然后立刻把来源写入 `body_source`：

```text
content:encoded
或
description
```

这个字段虽然简单，但对于知识库方案非常重要，因为“正文是谁提供的”必须是事实的一部分。Feed 里带的全文、页面 HTTP 抽取的全文、浏览器渲染后的全文不能混成一个无来源字符串，否则后续无法解释正文为什么变化，也无法在抓取规则升级后重新判定。

### 3.1 URL 标准化

项目的 `normalize_url()` 会：

- 去掉 fragment；
- 去掉尾部 `/`；
- 删除常见追踪参数，例如 `utm_*`、`gclid`、`fbclid`、`mc_cid`、`mc_eid`。

这是一个轻量、实用的第一层归一化，但不应该直接当作 1000 站平台的 canonical identity。它没有完整处理：

- hostname 大小写和 IDN；
- 默认端口；
- 重复 query key；
- query 参数顺序；
- `%xx` 编码等价关系；
- `http -> https` 的站点级规则；
- AMP/打印页/移动版；
- canonical 标签；
- 重定向链；
- 多域名镜像。

因此可迁移的是“尽早规范化并删除明确无业务意义的 tracking 参数”，而不是直接复制这段函数作为平台级 canonical 规则。

### 3.2 Feed 正文抽取

Feed 内的 HTML 不是简单正则全部去标签。实现路径是：

1. 优先使用 BeautifulSoup + lxml；
2. 删除 `script/style/noscript/svg/header/nav/footer/aside/form`；
3. 尝试 `article-body`、`article-content`、`post-content`、`entry-content`、`main` 等 selector；
4. 从 `h1/h2/h3/p/li/blockquote` 拼接段落；
5. BeautifulSoup 不可用时回退自定义 `HTMLParser`；
6. 再失败才做最粗粒度的 strip-html。

这说明作者把正文抽取设计成一个“逐级降级”的过程，而不是单一 extractor。这种理念值得保留，但平台化时应该把每一级产生的结果保存为 `ExtractionCandidate`，由统一质量层选择，而不是只返回最终一个字符串。

## 4. 多源并发：实现简单，但存在“全批次丢结果”的风险

`RSSCrawl4AINewsProvider._fetch_raw_items()` 的核心是：

```python
await asyncio.wait_for(
    asyncio.gather(*(fetch_source(...) for s in sources)),
    timeout=fetch_total_timeout,
)
```

每个 source 自己有 per-source timeout，外层又有 stage timeout。

优点：

- 并发抓多个 Feed；
- 每个 Feed 有超时；
- 整个阶段也有 deadline；
- 某个 `fetch_source()` 内部异常会被包装成 `FetchResult`，不会直接抛出；
- 单个 source 失败后可以记录 source_name、HTTP status 和 error。

但是外层 `asyncio.wait_for(asyncio.gather(...))` 有一个对 1000 站平台非常重要的问题：

**一旦 stage timeout，代码直接返回空列表，已经成功完成的 source 结果也被丢弃。**

测试 `test_fetch_stage_timeout_returns_empty` 甚至把这一行为固化成了 contract。

在四个新闻源里，这种损失可能还能接受；在 1000 个博客中绝对不能这样做。正确做法应是：

- 每个 Provider/Source Task 独立持久化；
- 使用 `as_completed` 或持久任务队列逐个提交结果；
- stage deadline 到达时只取消尚未开始或尚未完成的任务；
- 已成功结果立即落库；
- Run 状态可以是 `PARTIAL_SUCCESS`，并记录具体失败站点；
- 下次只重试失败/过期任务，而不是重新跑全部 1000 站。

因此本次技术方案增加“**部分结果保留（partial-result preservation）**”作为并发调度的硬性要求。

## 5. Crawl4AI enrichment：只为正文不足的条目升级抓取成本

`Crawl4AIEnricher.enrich_items()` 不是对所有链接都开浏览器，它只挑：

```text
URL 安全
AND
现有 body_text 长度 < min_chars
```

默认 `news_min_body_chars = 400`。

这其实是项目里最值得迁移的设计：

> Feed/HTTP 已经能提供可接受正文时，不要无条件启动 Browser；只有低成本候选不足时才升级抓取 route。

对于 1000 站博客知识库，这个思想应扩大为：

```text
Feed/API/Sitemap metadata candidate
        │
        ▼
HTTP snapshot + deterministic extraction
        │
        ├─ quality gate 通过 → 接受
        │
        └─ quality gate 不通过
               │
               ▼
        Browser/Crawl4AI enrichment
               │
               ├─ 通过 → 接受
               └─ 失败 → 站点专用 extractor / 人工检查 / dead-letter
```

但不能简单使用“字符数 >= 400”作为质量标准。平台应该综合：

- 正文字符数和词数；
- 标题与正文相关度；
- 导航/推荐/页脚比例；
- 文本密度；
- DOM 主体节点覆盖率；
- 代码块、表格、图片完整度；
- 是否包含常见错误页文案；
- feed summary 与页面正文相似度；
- 发布时间/作者/canonical 是否合理；
- 页面类型分类结果；
- extractor 历史接受率。

因此本次方案把“长度阈值 enrichment”升级成“**质量门控 + 路由升级**”。

## 6. Crawl4AI 的具体配置和原理

`_enrich_crawl4ai_batch()` 会创建一次 `AsyncWebCrawler`，然后用同一个浏览器实例批量跑所有目标，避免为每篇文章反复启动 Playwright。

关键配置包括：

```text
BrowserConfig:
  headless=True
  verbose=False

CrawlerRunConfig:
  page_timeout = timeout * 1000
  word_count_threshold = 5
  remove_overlay_elements = True
  remove_consent_popups = True
  semaphore_count = worker_count
  stream = False
  magic = True
  simulate_user = True
  override_navigator = True
  wait_until = load
  delay_before_return_html = 1.0
```

Markdown 使用：

```text
DefaultMarkdownGenerator
  └─ PruningContentFilter
       threshold = 0.48
       threshold_type = dynamic
       min_word_threshold = 5
```

并设置忽略链接、忽略图片、跳过内部链接等输出选项。

### 6.1 浏览器复用

这一点值得直接吸收：

- Browser process / context 启动成本高；
- 复用同一 crawler 显著减少 Playwright 启停；
- 在 1000 站场景中，应进一步升级成 Browser Worker Pool；
- browser/context/page 的生命周期和 source/domain 隔离策略必须可配置；
- 对 cookie 污染、内存泄漏、page 泄漏设置最大任务数和最大存活时间后强制 recycle。

### 6.2 并发上限

项目把 `worker_count` 限制为：

```text
max(1, min(configured_concurrency, 6))
```

体现了浏览器并发不能无限放大。我们的平台不能只做一个全局并发值，而要至少同时限制：

- 全局 Browser Worker 数；
- 单域名 QPS；
- 单 registrable domain 并发；
- source 并发；
- 单 worker page 数；
- host 内存/CPU；
- Browser seconds budget；
- 每个 Run 的 deadline 和成本预算。

### 6.3 批次 deadline

项目计算：

```text
wave_count = ceil(url_count / worker_count)
batch_timeout = max(page_timeout * wave_count + 15s, page_timeout)
```

这是一个不错的“波次估算”思路：批次 deadline 不能等于单页 timeout。但平台化后不能把 correctness 依赖在进程内 `wait_for()` 上，应把它变成调度预算：

```text
Task deadline
+ Route attempt deadline
+ Run deadline
+ Budget reservation expiry
```

任何 timeout 后都可以恢复，而不是整个 Python 协程状态消失。

## 7. Crawl4AI 输出选择：fit markdown、raw markdown、cited markdown、HTML 逐级候选

`_extract_crawl4ai_body()` 并不盲信一个字段，而是处理 Crawl4AI 返回对象的不同形态：

1. `markdown` 本身是字符串时直接作为候选；
2. 如果是 `MarkdownGenerationResult`，依次收集：
   - `fit_markdown`
   - `raw_markdown`
   - `markdown_with_citations`
3. 每个 Markdown 都做清洗；
4. 只有长度足够且不含错误页 marker 才接受；
5. Markdown 都不合格时，再从 `cleaned_html` / `html` 做正文抽取。

项目还定义了错误正文 marker，例如：

- article not found
- page not found
- 404 not found
- oops! something went wrong

这体现了一个很重要的原则：

> “Crawler 返回 success”不等于“业务正文成功”。

在我们的知识库方案中，必须把 HTTP 成功、Browser 成功、Extractor 成功、Quality Accepted 分成不同状态，不能用一个 `success=true` 表示整条链成功。

## 8. Markdown 清洗

项目的 `_clean_markdown_text()` 会：

- 统一换行；
- 删除 Markdown reference link 定义；
- 删除图片；
- 把普通链接 `[text](url)` 变成 `text`；
- 删除标题井号；
- 清理多余空白和连续空行。

这适合“把文章压缩成 LLM prompt 文本”，但不适合作为长期知识库最终 Markdown。

对于知识库，我们要保留：

- 标题层级；
- 链接 URL；
- 图片及附件关系；
- code fence 与语言；
- 表格；
- blockquote；
- footnote；
- 原文语义结构。

因此应当区分：

```text
Canonical IR           长期语义真相
Markdown Projection    面向阅读/导出的稳定 Markdown
Prompt Text Projection 面向 LLM 的降噪文本
```

不能把项目的 prompt-friendly cleaner 直接当知识库 Markdown cleaner。

## 9. Crawl4AI 失败后的 aiohttp 回退

如果以下情况发生：

- Crawl4AI import 失败；
- Browser 启动失败；
- 整批超时；
- 单个 CrawlResult 失败；
- Crawl4AI 提取不到足够正文；

实现会回退：

```text
aiohttp GET
  → HTML
  → extract_html_body_text()
```

这个“**抓取引擎不是单点依赖**”的思想必须保留。

但是生产方案不能在 catch 中直接重跑同一批 URL，因为这会产生两个问题：

1. **重复请求**：Crawl4AI 批次超时时，可能已有一部分 URL 成功发出甚至已经完成，但结果没有被消费；随后 aiohttp 会把整个 targets 再抓一遍。
2. **不可恢复**：失败 route 和 fallback route 都只存在于内存协程里，进程重启后无法知道已经尝试到哪一步。

所以本次方案把 fallback 抽象成持久 `FetchRouteAttempt`：

```text
HTTP_STATIC
   ↓ quality fail
BROWSER_CRAWL4AI
   ↓ runtime fail
HTTP_ALT_EXTRACTOR
   ↓
SITE_RECIPE / MANUAL
```

每次 attempt 独立记录开始/结束、请求字节、状态码、Snapshot、错误、成本、是否允许进入下一 route。

## 10. Windows 事件循环兼容

项目专门处理 Windows SelectorEventLoop 无法启动 Playwright 子进程的问题：

- 检测 `sys.platform == win32`；
- 如果当前是 `SelectorEventLoop`；
- `asyncio.to_thread()` 启一个线程；
- 在线程里创建 `ProactorEventLoop`；
- 在独立 loop 里跑 Crawl4AI。

这是典型的“库运行时约束泄漏到业务层”。

对平台方案的启示是：Browser Worker 最好做成独立进程/容器，Linux 为标准生产环境，API/调度器不直接承受 Playwright 事件循环差异。Windows 仅作为开发兼容模式，可采用独立 browser worker process，而不是把平台主任务循环改成特殊事件循环。

## 11. SSRF：有意识但还不够生产级

`_is_safe_external_url()` 会拒绝：

- 非 http/https；
- localhost；
- 字面值是 private/loopback/link-local IP 的 host。

这是值得肯定的安全意识，但对于多租户或 Web 管理端可添加 URL 的平台仍然不够，因为它没有：

- DNS 解析后检查 A/AAAA；
- DNS rebinding 防护；
- 重定向每一跳重新检查；
- metadata endpoint 特殊地址；
- IPv4-mapped IPv6；
- 内部域名/企业 DNS；
- 允许端口策略；
- egress proxy 强制控制。

我们的方案应统一把所有 HTTP、Browser、Feed、Context Provider 请求都放在同一 Egress/SSRF Policy 下，不能每个 crawler 自己写一小段 URL 判断。

## 12. Redirect 关联存在潜在错误

`_enrich_crawl4ai_batch()` 先建立：

```text
url_to_item[normalize_url(original_item_url)] = item
```

之后对 Crawl4AI 结果使用：

```text
resolved_url = result.url or result.redirected_url
item = url_to_item.pop(normalize_url(resolved_url), None)
```

如果 `result.url` 是跳转后的 final URL，而原始 item URL 是另一个地址，字典 key 就可能对不上。这样即使 Crawl4AI 已经成功提取正文，也可能找不到原 item，最后被当成“未匹配目标”进入 aiohttp fallback。

这正好说明平台不能依赖“结果 URL 再反查请求对象”。正确关联方式是：

- task_id / observation_id 作为主关联键；
- requested_url、final_url、redirect_chain 都是该 Observation 的属性；
- final URL 进入 alias/canonical 判断，但不改变请求任务身份。

本次方案明确新增这一约束。

## 13. 去重策略

项目有两层去重：

### 13.1 URL 去重

`dedupe_by_url()`：同 URL 只保留发布时间更新的条目。

### 13.2 标题去重

`dedupe_by_normalized_title()`：标题转小写、去非字母数字、合并空格，然后同标题保留正文最长条目。

标题二次去重对新闻流非常实用，但对历史博客库不能作为 identity，因为：

- 周报经常重复标题；
- “Release Notes”“Weekly Update”“Hello World”可能出现很多次；
- 翻译文章可能同标题但不同正文；
- 站点迁移可能标题相同但版本不同。

因此平台要把标题相似度、正文 SimHash/MinHash、canonical、redirect、publication date、source path 等组合成“Duplicate Hint / Cluster”，由规则决定 merge，而不是直接删除。

## 14. Stable ID：可借鉴，但不能只绑定 URL

`schema_mapper.make_article_id()` 使用：

```text
sha256(url)[:16]
```

优点：

- 确定性；
- 重跑稳定；
- 下游不需要随机 UUID。

但只绑定 URL 仍然不够，因为 canonical 可能变化、站点迁移、URL slug 可能改名。

知识库应区分：

```text
URL Candidate ID      绑定规范化 URL
Document ID           绑定逻辑文章身份
Document Version ID   绑定某次接受的内容版本
Chunk ID              绑定 document version + chunk path/hash
```

并允许多个 URL alias 指向一个 Document。

## 15. 正文清洗：强站点规则的价值和风险

`schema_mapper.py` 维护了不少新闻站专用尾部 marker，例如：

- More For You
- Latest Crypto News
- Top Stories
- Newsletters
- CoinDesk 专用页脚区域

还通过正则检测导航菜单、market ticker 等并裁掉。

这说明生产爬虫最终一定会有站点定制规则；纯通用 extractor 不可能覆盖所有站点。

但是这些规则必须：

- 归属于 `SiteProfileRelease`；
- 有版本；
- 可回滚；
- 可以离线 replay 到旧 Snapshot；
- 有命中率和误删率监控；
- 不直接硬编码在业务 schema mapper 中。

否则规则越积越多，无法判断某篇旧文章当时用了哪一套清洗逻辑。

## 16. “短正文后来补全”机制很值得迁移

`NewsManager.update_news_database()` 对同 URL 的旧记录并不是一律跳过：

```text
如果旧正文 < min_body
且新正文更长
则用新记录更新旧记录
```

这是非常实用的“质量升级”思路。因为首次增量可能只拿到 RSS summary，之后 Crawl4AI 成功拿到全文。

平台化后不能直接原地覆盖，而应该：

1. 保存新的 ExtractionCandidate；
2. 重新计算 Canonical IR；
3. 如果内容语义发生变化，创建新的 Document Version；
4. 如果只是同正文更完整的 extraction 修复，按版本策略标记 `extraction_upgrade`；
5. 原来的 Snapshot、Candidate、Version 全部保留；
6. Markdown/Chunk/Embedding 从新 accepted version 重建。

因此本次方案增加“**Content Candidate Upgrade**”和“body/source provenance”模型。

## 17. 项目不适合直接承担 1000 站历史归档的原因

### 17.1 没有 Coverage 模型

RSS 只返回当前窗口，不证明历史完整。项目默认每个 source 最多 100 items，也没有 sitemap/archive/category pagination 等历史发现逻辑。

### 17.2 没有 durable frontier

任务完全存在 asyncio 协程和内存 list 中，进程重启后没有任务恢复、lease、heartbeat、dead-letter。

### 17.3 没有增量条件请求

没有用 Feed ETag / Last-Modified，也没有保存 cursor；每次都是直接重新 GET。

### 17.4 没有完整 Snapshot/Version

最终只关心当前新闻数据库，没有保存不可变网络快照、Rendered DOM、Extraction Candidate 和历史 Document Version。

### 17.5 聚合阶段超时会清空全部结果

这是小规模 news pipeline 可以接受的 fail-soft 方式，但在平台级同步中会造成明显数据丢失。

### 17.6 正文质量门槛过度依赖字符数

400 字只是启发式，不能区分 500 字导航和 300 字高质量短文。

### 17.7 Feed parser 覆盖范围有限

主要按 RSS 2.0 `/channel/item` 解析，没有完整 Atom、JSON Feed、分页 feed、WebSub 等通用 Provider 能力。

### 17.8 全局 hardcode 太多

Crawl4AI 的 pruning threshold、等待时间、站点尾部 marker 都是进程级代码常量；1000 站需要 Profile Release。

## 18. 本项目对博客知识库方案的直接优化点

本次研究后，将以下设计正式纳入技术方案。

### 18.1 Feed Body Candidate

新增 `FeedItemObservation / ContentCandidate`，保存：

- feed source；
- guid；
- link；
- published/updated；
- description；
- content:encoded；
- body_source；
- candidate_hash；
- quality score。

Feed 全文如果质量足够，可以作为增量同步低成本正文候选；但它不能替代历史 Coverage 证明。

### 18.2 Quality-Gated Enrichment

不再用“正文长度不足”作为唯一 Browser 升级条件，而是 Quality Policy 决定：

```text
ACCEPT
RETRY_HTTP_ALT
ESCALATE_BROWSER
SITE_RECIPE
MANUAL_REVIEW
REJECT_NON_ARTICLE
```

### 18.3 Partial-Result-Preserving Fanout

所有并发 Provider/Source 任务独立落库；Run 超时时保留已完成结果。禁止 `wait_for(gather(...))` 超时后返回空集合覆盖成功事实。

### 18.4 Durable Route Fallback

Crawl4AI → HTTP fallback 不再由内存 exception handler 隐式完成，而是显式 `FetchRouteAttempt`，每次 route 都可观察、可预算、可重试、可恢复。

### 18.5 Browser Pool Reuse

复用 Browser Worker，但设置 recycle；浏览器进程不跟业务 API 同进程。

### 18.6 Requested URL 与 Final URL 分离

任务通过 task/observation ID 关联。Redirect 只写入 redirect chain 和 URL alias，不允许用 final URL 反查原任务。

### 18.7 Duplicate Hint 而不是标题硬去重

Normalized title、SimHash、MinHash、canonical、redirect 都生成 duplicate evidence；只有 identity resolver 才能决定合并。

### 18.8 Content Upgrade

同一文章后续拿到更完整正文时创建新的 Candidate/Version，不覆盖历史事实；触发 Markdown、Chunk、Embedding 增量重建。

### 18.9 SSRF 统一到 Egress 层

所有 Feed/HTTP/Browser 请求统一 DNS 解析、IP 校验、重定向再校验和端口策略。

## 19. 推荐的平台实现方式

将本项目有价值的“小型 pipeline 模式”改造成以下平台级流水线：

```text
[Source]
   │
   ├─ Sitemap/API/Archive/Category → Coverage Candidate
   ├─ RSS/Atom/JSON Feed ─────────→ Coverage Evidence + FeedItemObservation
   │                                  │
   │                                  └─ Feed body candidate
   │
   ▼
[Durable Frontier]
   │
   ▼
[Change Signal]
   │  ETag / Last-Modified / lastmod / GUID / updated / cursor
   ▼
[HTTP Fetch]
   │
   ▼
[Extraction Candidate Ensemble]
   │
   ▼
[Quality Gate]
   ├─ Accept
   ├─ HTTP alternate extractor
   └─ Browser/Crawl4AI
          │
          ▼
      [Quality Gate]
          │
          ▼
[Identity + Version]
   │
   ├─ Snapshot
   ├─ Canonical IR
   ├─ Markdown Projection
   ├─ Chunk Projection
   └─ Search/Embedding Projection
```

关键区别是：每个框都由持久状态连接，而不是一个 asyncio 函数内部的临时控制流。

## 20. 最终结论

LLM_trader 的 RSS + Crawl4AI 新闻子系统并不是完整爬虫平台，但它给出了非常清晰的低成本抓取思想：

1. 先用 RSS 获取结构化条目和已有正文；
2. 只对正文不足的项目升级到 Crawl4AI；
3. 复用 Browser；
4. Browser 失败仍有普通 HTTP fallback；
5. 记录正文来源；
6. 后续拿到更完整正文时允许升级；
7. 通过确定性 schema 让下游不依赖具体抓取引擎。

真正迁移到 1000 站知识库时，必须同时修正它的小系统假设：

- 并发 stage 超时不能丢已完成结果；
- fallback 必须持久化；
- redirect 不能靠 final URL 反查 item；
- SSRF 要做到 DNS/redirect/egress 级；
- 标题只能作为 duplicate hint；
- 字符数门槛必须升级为质量门控；
- Feed 只能作为增量 Change Signal 和正文 Candidate，不能证明历史完整；
- 所有 hardcode extractor/filter 参数要进入版本化 Site Profile；
- 任何正文升级都要形成可追溯 Version，而不是覆盖当前字符串。

因此，本项目最适合被吸收为技术方案中的“**Feed-first + Quality-Gated Enrichment + Durable Route Fallback**”设计，而不是作为平台本体。
