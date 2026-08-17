# 使用 Crawl4AI 抓取搜狐文章教程：实现细节与技术原理分析

- 编号：2
- 原文：https://juejin.cn/post/7528337041614618676
- 原文标题：使用 Crawl4AI 抓取搜狐文章教程
- 作者：羊八井
- 发布时间：2025-07-19
- Crawl4AI 官方文档：https://docs.crawl4ai.com/
- 调研目标：评估文中“链接发现 -> URL 规则过滤 -> CSS/JSON Schema 抽取 -> HTML 清洗/字段归一化 -> 图片下载 -> `arun_many` 并发”的实现方式，对“约 1000 个技术博客全量历史采集、Markdown 清洗、增量同步、持续扩站和 Web 管理”方案的实际价值与边界。

## 1. 结论

这篇文章最大的价值不是“用 Crawl4AI 抓搜狐”本身，而是展示了一个很适合规模化博客采集平台的工程分层雏形：

```text
列表/入口页
 -> Crawl4AI result.links
 -> 稳定 URL pattern 过滤
 -> 文章 URL 集
 -> JsonCssExtractionStrategy
 -> 结构化字段
 -> HTML/时间/作者/图片后处理
 -> 并发抓取
```

其中有四个思路值得直接进入平台方案：

1. **文章发现和正文 DOM 模板必须解耦。** 文中通过 `result.links['internal']` 获取真实链接，再用 `https://www.sohu.com/a/` 这样的 URL pattern 判断文章候选，不依赖列表卡片的 CSS 层级。列表页改版时，只要真实文章链接仍存在，Discovery 不会因为卡片 selector 失效而完全断流。
2. **稳定站点模板优先使用 CSS/JSON Schema，而不是默认使用 LLM 抽取。** `JsonCssExtractionStrategy` 的 `baseSelector + fields` 是确定性、低成本、可重复的抽取路径，很适合百万级页面。
3. **字段抽取之后还需要一层版本化、确定性的 Post-Processor。** 文中实际已经在做 `clean_html()`、`parse_datetime()`、作者 fallback、图片 URL 处理；这些逻辑不能散落在业务代码里，应成为 Extraction Profile 的正式组成部分。
4. **`arun_many + dispatcher` 只适合 Worker 内并发，不等于生产级任务系统。** Crawl4AI 官方文档确认其 Dispatcher 负责单进程/单 Worker 的资源和并发控制；对于 1000 站点、百万 URL，仍需要 durable frontier、lease、幂等、站点级配额和持久化状态。

同时，文章中的示例代码有几个生产级风险，不能原样复制：

- `json.loads(result.extracted_content)[0]` 默认假设结果一定是合法 JSON、数组非空且只需要第一个元素；模板漂移、错误页或 selector 多匹配时会产生错误或静默丢数据。
- `result.success` 只证明 Crawl4AI 执行层成功，不证明抓到的是文章、更不证明正文和 metadata 合法。
- 图片下载直接根据远端 URL path 生成本地路径，缺少 content hash 去重、SSRF/redirect 检查、MIME/magic bytes 校验、大小限制、持久化重试和引用关系。
- 作者为空时退化到 `original_link` 是业务 fallback，但如果不记录 provenance，后续会把低可信 fallback 当作真实作者。
- 单个 Schema 不能覆盖长期存在的 legacy template、视频/图集误入、AB 测试和局部改版，必须支持多 Profile/fallback 和 drift 监控。

因此，这篇文章对当前总方案不是架构推翻，而是促使方案把 **“链接发现证据、Discovery Fetch Route 与 Article Fetch Route 分离、字段级 Schema Contract、结构化 extracted_content 校验、版本化 Post-Processor、媒体 URL 归一化”** 做得更明确。

## 2. 原文实现链路

原文把搜狐抓取分成两个主要阶段。

### 2.1 阶段一：从入口页发现文章链接

核心代码逻辑是：

```python
async with AsyncWebCrawler(config=browser_config) as crawler:
    result = await crawler.arun(url=url, config=run_config)
    if not result.success:
        ...
    internal_links = result.links.get('internal', [])
    links = [new_link(link, url_clean) for link in internal_links]
    links = [link for link in links if cb_filter(link)]
```

默认过滤器：

```python
def _filter_link(link: Link) -> bool:
    return link.href.startswith('https://www.sohu.com/a/')
```

Crawl4AI 当前官方文档中，`CrawlResult.links` 确实是一个通常包含 `internal` 与 `external` 列表的字典；条目可带 `href`、`text`、`title` 等信息，并且在未关闭链接提取时自动产生。

这个设计的真正意义是：**列表页只负责提供链接图，文章身份候选优先依赖 URL 规则，而不是列表卡片结构。**

传统做法容易写成：

```text
.archive-card:nth-child(...) a.title
```

一旦站点把卡片从 `div` 改成 `article`、增加一层容器或做 AB Test，文章发现可能直接归零。URL path 通常更稳定，例如：

```text
/a/<article_id>
/posts/<slug>
/blog/<yyyy>/<slug>
archives/<id>
```

但 URL pattern 也只能成为 **Admission/Page Type Hint**，不能成为“这是合法文章”的最终证明。搜狐可能存在 `/a/` 下的视频、图集、活动页或异常页，所以后续仍要做 page type 检查与内容质量验证。

### 2.2 阶段二：用 JsonCssExtractionStrategy 抽取结构化文章

原文定义：

```python
SCHEMA = {
    'name': 'Sohu-Article',
    'baseSelector': '#article-container',
    'fields': [
        {'name': 'author', 'selector': '.user-info h4', 'type': 'text'},
        {'name': 'title', 'selector': '.main div.text-title h1', 'type': 'text'},
        {'name': 'published_date', 'selector': '.main .article-info #news-time', 'type': 'text'},
        {'name': 'content', 'selector': '.main article', 'type': 'html'},
        {'name': 'original_link', 'selector': '.main .article-info [data-role="original-link"]', 'type': 'text'},
        {
            'name': 'imgs',
            'selector': '.article img',
            'type': 'list',
            'fields': [
                {'name': 'src', 'type': 'attribute', 'attribute': 'src'},
                {'name': 'alt', 'type': 'attribute', 'attribute': 'alt'},
            ],
        },
    ],
}
```

Crawl4AI 官方的无 LLM 抽取文档同样将 `JsonCssExtractionStrategy` 定义为：

```text
base selector
 + fields
 + text / attribute / html / nested / list 等字段类型
```

其优点是：

- 不需要 LLM 调用，成本低；
- selector 行为确定，可重复；
- 大规模页面吞吐高；
- 输出结构化 JSON，方便进入统一 IR；
- 对同模板的历史页面可以稳定复用。

对于 1000 个站点，应把它定位为 **站点模板明确时的主抽取路径**，而不是一个写死在爬虫函数里的字典。

## 3. `baseSelector` 的技术原理与生产约束

`baseSelector` 决定“一个输出对象对应页面里的哪一个根节点”。字段 selector 通常相对这个根节点执行。

搜狐示例：

```text
#article-container
  -> .user-info h4
  -> .main div.text-title h1
  -> .main article
  -> .article img
```

这比对整页做大量绝对 selector 更容易隔离 header、footer、相关推荐、侧边栏。

但生产中必须给 `baseSelector` 增加 cardinality contract：

```text
expected_min = 1
expected_max = 1
```

原因：

- 0 个：模板改版、错误页、挑战页、页面没加载完；
- 1 个：通常是预期文章；
- >1 个：可能 selector 太宽、页面嵌入多个 article container、AB 结构变化。

原文直接：

```python
item = json.loads(result.extracted_content)[0]
```

相当于把这三种情况全部压成“取第一条”。生产平台应该改成：

```text
parse extracted_content
 -> JSON 类型检查
 -> list cardinality 检查
 -> baseSelector match count 检查
 -> required field contract
 -> field normalization
 -> Candidate Outcome
```

推荐的结构化结果：

```text
ExtractionCandidate
- profile_release_id
- raw_extracted_content_artifact_id
- item_count
- selected_item_index
- field_values
- field_errors
- normalization_trace
- status = VALID | EMPTY | INVALID | AMBIGUOUS
```

这样“Crawl4AI 成功”与“业务抽取合法”就不会混淆。

## 4. 字段级 Schema Contract 应比原文更严格

原文 Schema 只声明 `name/selector/type`，用于教程足够，但生产系统应该扩展为平台自己的 Field Contract：

```yaml
fields:
  - name: title
    selector: ".main div.text-title h1"
    extractor_type: text
    required: true
    cardinality: one
    normalize:
      - strip
      - collapse_whitespace

  - name: published_at
    selector: ".main .article-info #news-time"
    extractor_type: text
    required: false
    cardinality: zero_or_one
    normalize:
      - parse_datetime
    fallback_sources:
      - json_ld.datePublished
      - meta.article:published_time
      - feed.published_at_hint

  - name: content_html
    selector: ".main article"
    extractor_type: html
    required: true
    cardinality: one
    normalize:
      - clean_html
      - normalize_links

  - name: images
    selector: ".article img"
    extractor_type: list
    required: false
    normalize:
      - resolve_media_url
      - dedupe_media
```

这里需要明确区分两个概念：

1. Crawl4AI Schema 是 **执行层 selector 规则**；
2. 平台 Field Contract 是 **业务层字段语义、约束、归一化和 fallback 规则**。

这样未来 Crawl4AI 升级、换 XPath、换 Trafilatura 或使用 API 数据时，不需要改变 Document IR 的字段语义。

## 5. 原文隐藏但非常重要的一层：Post-Processor

文章正文抓取函数中实际做了这些操作：

```python
content = clean_html(item.get('content', ''))
published_time = parse_datetime(item['published_date'])
author = item.get('author', '')
author = author if author else remove_whitespace(item.get('original_link', ''))
images = list(Img.new_imgs(item.get('imgs', []), access_schema))
img_paths = await download_imgs(images, access_schema)
```

这说明 Schema 输出并不是最终知识对象。真正流程是：

```text
Schema Raw Fields
 -> 清洗
 -> 类型转换
 -> fallback
 -> URL 归一化
 -> media 处理
 -> 最终 Article Model
```

当前总方案应显式增加 **Post-Processor Release**，不要把这些逻辑写在站点业务函数中。

推荐：

```text
post_processor_release
- id
- version
- processor_type
- config
- code/runtime_digest
- input_contract
- output_contract
- status
```

常见处理器：

```text
HtmlCleaner
WhitespaceNormalizer
DateParser
AuthorNormalizer
CanonicalUrlResolver
RelativeLinkResolver
MediaUrlResolver
TagNormalizer
LanguageNormalizer
BoilerplateSanitizer
```

一条 Extraction Profile 可以声明：

```text
Schema Candidate
 -> HtmlCleaner@v3
 -> RelativeLinkResolver@v2
 -> DateParser@v4
 -> AuthorFallbackPolicy@v2
 -> IR Builder@v5
```

每一步输出 trace。这样出现错误时间、作者污染或图片链接错误时，可以在 Web 管理端看到“原值 -> 哪个 processor -> 新值”。

更重要的是，Post-Processor 版本升级后可以对已有 Snapshot/Raw Candidate 做 REPROCESS，无需重新访问 1000 个源站。

## 6. 作者 fallback：原文可用，但必须保存 Provenance

原文逻辑：

```python
author = item.get('author', '')
author = author if author else remove_whitespace(item.get('original_link', ''))
```

从“尽量不返回空字符串”的应用代码角度合理，但知识库不能把两个来源视为同等权威。

正确的 IR 应类似：

```text
author.value = "..."
author.source_type = CSS | ORIGINAL_LINK_FALLBACK
author.source_artifact_id = ...
author.extractor_release_id = ...
author.confidence = 0.95 | 0.40
```

否则后续搜索、作者聚合、去重、AI 报告都会把 fallback 数据当成事实。

原则是：

> fallback 可以提高可用性，但不能抹掉不确定性。

这同样适用于发布时间、canonical、标签、摘要和站点名。

## 7. HTML 正文优先于直接保存 Markdown

原文把 `content` 定义为：

```python
{'name': 'content', 'selector': '.main article', 'type': 'html'}
```

这个选择对长期知识库是正确方向。

如果直接把 Crawl4AI Markdown 作为唯一原始事实，未来很难可靠修复：

- 相对链接；
- `srcset` / lazy-load 图片；
- 代码块语言；
- 表格结构；
- figure/figcaption；
- footnote；
- DOM 级 metadata；
- 清洗规则误删。

推荐：

```text
Network/Rendered Snapshot
 -> Candidate HTML
 -> Canonical Document IR
 -> Markdown Projection
```

Markdown 是可重建的发布形式，不是唯一真相源。

## 8. 图片下载代码的原理与生产问题

原文：

```python
parsed_url = urlparse(img_url)
save_path = CONFIG.download_path / (parsed_url.hostname or 'unknown') / remove_suffix(parsed_url.path[1:])
...
if not parsed_url.scheme:
    img_url = f'{access_schema}:{img_url}'
saved_path = await download_img(client, img_url, save_path)
```

它解决了一个常见问题：搜狐可能返回 protocol-relative URL，例如：

```text
//qpic.cn/...
```

通过当前页面 scheme 补成：

```text
https://qpic.cn/...
```

但是面向长期知识库，媒体 URL 归一化至少要覆盖：

```text
//cdn.example.com/a.jpg          protocol-relative
/images/a.jpg                   root-relative
../images/a.jpg                 path-relative
https://.../a.jpg               absolute
srcset="a.jpg 1x, a@2x.jpg 2x"  responsive image
data-src / data-original        lazy load
```

统一算法应以 `final_url` 为 base，而不是仅使用一个 `access_schema`：

```text
raw src
 -> trim/decode
 -> select actual lazy-load/srcset candidate
 -> urljoin(final_url, raw_src)
 -> normalize scheme/host/path
 -> policy check
 -> Asset Task
```

### 8.1 为什么不能直接按远程 path 存本地文件

示例中的：

```text
<hostname>/<remote_path>
```

存在问题：

- 两个不同 query 可能是不同内容；
- 一个 URL 后续内容可能变化；
- path 可极长或包含危险字符；
- 不同 URL 可能指向完全相同图片；
- redirect 后真实来源不同；
- 无法自然做 hash 去重；
- 大规模分布式 Worker 不能依赖本机路径。

生产方案应：

```text
Asset Fetch
 -> status/MIME/magic bytes/size 校验
 -> sha256
 -> S3/MinIO content-addressed object
 -> asset_source 保存 URL 和响应事实
 -> asset_reference 绑定 document_version
```

对象键：

```text
assets/sha256/ab/cd/<sha256>.<ext>
```

这样同一图片只存一次，且 Markdown 可从 `asset_reference` 重建。

## 9. `arun_many` 与 Dispatcher 的真实边界

原文：

```python
async for result in await crawler.arun_many(
    urls=urls,
    config=run_config,
    dispatcher=dispatcher,
):
    ...
```

Crawl4AI 当前官方文档确认：

- `arun_many()` 用于多 URL 并发/批量抓取；
- `stream=True` 时可以按完成顺序流式消费；
- Dispatcher 可使用 `MemoryAdaptiveDispatcher` / `SemaphoreDispatcher`；
- `MemoryAdaptiveDispatcher` 可根据内存阈值控制并发；
- `RateLimiter` 可针对 429/503 做节流和 backoff；
- URL-specific `CrawlerRunConfig` 可以通过 `url_matcher` 选择不同抓取配置。

这非常适合 **Worker 内微批处理**：

```text
Durable Frontier
 -> lease 20~100 个 URL
 -> arun_many(stream=True)
 -> 每个 result 完成后立即写 Snapshot + Outcome
 -> 单 URL ACK
```

但不能把：

```text
urls = 全站百万 URL
await crawler.arun_many(urls)
```

当成平台调度系统。

原因：

- 进程退出后内存任务列表丢失；
- 没有跨 Worker 的业务 lease；
- 无法提供历史全量 Coverage Ledger；
- 单 Worker 的 RateLimiter 不等于全局站点配额；
- 无法保证 URL 幂等；
- API/Browser 重启后 Run 状态无法恢复；
- 重试和 dead-letter 的业务语义不足。

因此职责分工应是：

```text
PostgreSQL Frontier / Scheduler
    负责持久状态、优先级、公平调度、幂等、lease、重试

Crawl4AI Dispatcher
    负责当前 Worker 的浏览器 session、内存、并发、局部 rate limit
```

## 10. Discovery Fetch Route 与 Article Fetch Route 应分开

原文 `base_fetch_links()` 使用专门的 `browser_config = BROWSER_CONFIG.clone(browser_type=CONFIG.browsers.urls_browser)`，这暴露了一个很重要但经常被忽略的事实：

> “发现文章链接需要 Browser”与“文章正文需要 Browser”是两个不同判断。

例如一个站点可能：

```text
归档页：JS 无限滚动，必须 Browser
文章页：服务端完整 HTML，HTTP 就能抓
```

也可能反过来：

```text
RSS/归档：HTTP 可枚举
文章：正文通过 JS hydration，必须 Browser
```

所以 Profile 中应独立定义：

```text
Discovery Provider Fetch Route
Article Fetch Route
Asset Fetch Route
```

而不是用一个 `site.browser_required=true/false` 覆盖整个站点。

这能显著降低 1000 站点规模下的 Browser 成本。

## 11. URL Pattern 的优点和陷阱

文章用：

```python
href.startswith('https://www.sohu.com/a/')
```

优点：

- 快；
- 与 DOM 布局解耦；
- 对列表页改版有韧性；
- 可直接转成 Admission Rule。

但生产系统至少还要处理：

```text
http/https 统一
www/non-www
大小写规则
fragment 去除
tracking query 清理
redirect
canonical
重复 URL
locale 子路径
AMP/mobile 变体
镜像域名
```

因此流程不是简单 `startswith`，而是：

```text
raw href
 -> resolve relative URL
 -> normalize
 -> scope
 -> include/exclude URL rule
 -> crawler-trap policy
 -> page_type_hint
 -> durable frontier
```

并保存 Link Evidence：

```text
source_page_snapshot_id
raw_href
normalized_href
anchor_text
anchor_title
context
internal/external
provider_release_id
admission_decision
admission_rule_release_id
```

Web 管理端可以直接回答：“这个文章 URL 是从哪个归档页的哪个 link 发现的，为什么被接受/拒绝？”

## 12. 页面类型检测必须位于 URL Rule 之后、Acceptance 之前

原文注意事项提到可用 BeautifulSoup 检查视频或图片页面。这是非常重要的补充，因为 URL pattern 经常会把多个页面类型混在一起。

生产模型建议：

```text
URL Rule -> page_type_hint
Fetch -> DOM/metadata evidence
Preflight Classifier -> page_type
Extraction -> Candidate
Quality -> accept/reject
```

页面类型：

```text
ARTICLE
DOC
LISTING
VIDEO
GALLERY
LOGIN
PAYWALL
CHALLENGE
ERROR_PAGE
UNKNOWN
```

不要写成：

```text
URL 符合 /a/ -> 一定是 ARTICLE
```

否则会把视频页、图集页、封禁页或错误页清洗成“知识文章”。

## 13. Schema Fallback 应该是多 Candidate，而不是 catch 后偷偷换 selector

原文建议“捕获提取失败并回退到备用 schema”。方向正确，但生产中 fallback 必须可观测。

推荐：

```text
Profile current-v3
 -> Candidate A: primary CSS schema
 -> invalid
 -> Candidate B: legacy CSS schema
 -> valid
 -> Candidate C: Trafilatura generic extractor
 -> valid, lower score
 -> Quality chooses B
```

并记录：

```text
primary_schema_success_rate
fallback_rate
legacy_schema_hit_rate
field_missing_rate
```

如果某站点过去 fallback 率 1%，突然变成 80%，这通常就是模板漂移事件。

不能只是：

```python
try:
    schema1()
except:
    schema2()
```

然后把结果伪装成“主模板正常”。

## 14. `result.success` 为什么不等于业务成功

原文第一阶段检查：

```python
if not result.success:
    raise ...
```

这是必要的执行层检查，但不充分。

可能出现：

```text
HTTP/Browser 访问成功
Crawl4AI result.success = true
但返回的是登录页 / WAF / challenge / 404 soft page
```

也可能：

```text
result.success = true
extracted_content = []
```

还可能：

```text
extracted_content 有对象
但 title 是空、content 是相关推荐、发布时间解析错误
```

因此平台至少有四层状态：

```text
FETCH_SUCCEEDED
EXTRACTION_VALID
QUALITY_ACCEPTED
DOCUMENT_VERSION_PUBLISHED
```

它们必须分别持久化，不能用一个布尔值代替。

## 15. 对 1000 站点方案的直接优化项

结合这篇文章，当前方案应该补充以下能力。

### 15.1 Discovery Link Evidence

为 `result.links` 等发现来源保存可追溯证据，而不只是 URL：

```text
来源 snapshot
raw/normalized href
anchor text/title/context
provider
规则版本
准入结果
```

### 15.2 Discovery/Article Fetch Route 分离

动态归档页和文章页分别决定 HTTP/Browser route，避免整站被一个 browser flag 绑死。

### 15.3 Field Contract + Cardinality

每个 Schema 字段明确：

```text
required
zero_or_one / one / many
normalizer
fallback source
confidence
```

`extracted_content` 先做 JSON + 数量验证，不允许无条件 `[0]`。

### 15.4 Versioned Post-Processor

把：

```text
clean_html
parse_datetime
author fallback
relative URL normalization
media normalization
```

从散落业务代码升级为可版本化、可重放、可审计的处理链。

### 15.5 Media URL Resolver

统一解决：

```text
protocol-relative
root-relative
path-relative
srcset
lazy-load
```

然后再创建 Asset Task。

### 15.6 Web 调试视图

管理端增加两条诊断链：

```text
Discovery：来源页面 -> result.links -> normalize -> rule -> frontier
Extraction：raw fields -> post-process trace -> Candidate -> Quality -> IR
```

这对 1000 个异构站点定位“为什么漏文章/为什么字段错”非常关键。

## 16. 不应从文章直接照搬的实现

### 16.1 不要把完整 URL 集放内存

教程通常一次处理少量 URL；平台需要 durable frontier + 小批 lease。

### 16.2 不要直接保存到 Worker 本地磁盘

生产系统应使用对象存储和 content-addressed Asset。

### 16.3 不要把 selector 写在 Python 源码中

应变为版本化 `site_extraction_profile_release`，可以在 Web 上审核、测试、回滚。

### 16.4 不要只维护一个搜狐式 Schema

每站点允许多个模板：

```text
current article
legacy article
docs
special feature
```

### 16.5 不要把 `original_link` fallback 当真实作者

必须保存字段来源与 confidence。

### 16.6 不要用异常是否抛出来判断抓取完成

Run/Stage 完成要由 Finalizer 对账 durable task、Artifact、Candidate、Quality 和 Coverage Evidence。

## 17. 推荐的生产实现伪代码

### 17.1 Discovery Worker

```python
async def discover_from_listing(task):
    snapshot = await fetch_provider_page(task.provider_route, task.url)
    result = await crawl4ai_links(snapshot)

    outcomes = []
    for link in result.links.get("internal", []):
        evidence = LinkEvidence.from_crawl_result(
            snapshot_id=snapshot.id,
            href=link.get("href"),
            text=link.get("text"),
            title=link.get("title"),
        )
        normalized = normalize_url(evidence.href, base=snapshot.final_url)
        decision = admission_engine.evaluate(normalized, task.rule_release)
        persist_link_evidence(evidence, normalized, decision)

        if decision.accepted:
            upsert_url_identity(normalized)
            enqueue_frontier_idempotently(normalized, decision.page_type_hint)

    return DiscoveryOutcome(...)
```

### 17.2 Browser Fetch Worker

```python
async def browser_microbatch(tasks):
    configs = build_versioned_crawl4ai_configs(tasks)
    async with runtime_pool.borrow() as crawler:
        async for result in await crawler.arun_many(
            urls=[t.url for t in tasks],
            config=configs,
            dispatcher=bounded_dispatcher,
        ):
            task = map_result_to_task(result)
            snapshot = persist_snapshot(result)
            persist_fetch_outcome(task, result, snapshot)
            ack_task_independently(task)
```

### 17.3 Extraction Worker

```python
async def extract(snapshot, profile_release):
    raw = await run_schema(snapshot, profile_release.schema)

    parsed = validate_json_output(raw)
    if parsed.item_count == 0:
        return ExtractionOutcome.empty(...)
    if parsed.item_count > profile_release.max_items:
        return ExtractionOutcome.invalid("AMBIGUOUS_CARDINALITY")

    fields = validate_field_contract(parsed.items[0], profile_release.fields)
    processed = run_versioned_post_processors(fields, snapshot.final_url)

    candidate = build_candidate(processed)
    quality = evaluate_quality(candidate, profile_release.quality_policy)
    return persist(candidate, quality)
```

### 17.4 Asset Worker

```python
async def fetch_asset(task):
    url = resolve_media_url(task.raw_url, task.document_final_url)
    validate_egress(url)
    response = await bounded_http_get(url)
    validate_status_mime_size_magic(response)
    sha256 = hash_bytes(response.body)
    object_key = put_content_addressed(sha256, response.body)
    upsert_asset_and_reference(task, response, sha256, object_key)
```

## 18. 测试与验收用例

基于这篇文章，至少补充这些 Golden Fixtures / Contract Tests：

1. `result.links` 中同一文章 URL 出现多次，Frontier 只产生一个幂等任务；
2. raw href 是 `/a/123`，能够基于 `final_url` 正确 resolve；
3. raw href 带 fragment/tracking query，规范化后 identity 一致；
4. URL pattern 命中但真实页面是 VIDEO/GALLERY，不能 Accepted；
5. `baseSelector` 匹配 0 个，结果为 `EMPTY/INVALID`；
6. `baseSelector` 意外匹配多个，不能无条件取 `[0]`；
7. `extracted_content` 不是合法 JSON；
8. `extracted_content` 是空数组；
9. required title 缺失但正文存在；
10. published date 解析失败，保持 NULL 并保存原值；
11. author 缺失触发 fallback，provenance/confidence 正确；
12. 图片是 `//cdn/...`；
13. 图片是 `/images/...`；
14. 图片来自 `data-src` / `srcset`；
15. 同一图片多个 URL 最终 hash 相同，只保存一个对象；
16. 主 Schema 失败、legacy Schema 成功，fallback metric 增加；
17. `arun_many` 中一个 URL 失败，其余 task 可以独立 ACK；
18. Worker 在流式结果中途被 kill，未 ACK task 能重新 lease；
19. Discovery 页面需要 Browser、文章页只需 HTTP，路由可以独立配置；
20. 文章页 Browser 抓取成功但返回 challenge DOM，Quality Gate 拒绝。

## 19. 最终评价

这篇文章是一篇很好的“单站点 Crawl4AI 工程实践”，尤其有三个值得长期保留的设计信号：

```text
URL 模式比列表 DOM 更适合做文章发现规则
CSS/JSON Schema 比 LLM 更适合稳定模板的大规模确定性抽取
抽取之后必须继续做清洗、字段归一化和媒体处理
```

对于 1000 个技术博客的平台化场景，需要在其上增加：

```text
Durable Frontier
Provider Coverage
版本化 URL Rule / Extraction Profile
Field Contract / Cardinality
Post-Processor Release
Metadata Provenance
多 Schema Candidate + Quality Gate
Discovery Route / Article Route 分离
独立 Asset Pipeline
可恢复任务状态
Web 诊断与 Drift 监控
```

最终应把文章里的代码看成 **Worker 内局部执行模板**，而不是完整爬虫平台。它最有价值的地方，是验证了当前方案里“Discovery 与 Extraction 解耦、Crawl4AI 作为执行引擎、Schema 抽取作为高性能主路径、媒体和字段后处理必须单独治理”的方向，并进一步暴露出需要补强的字段级契约与 Post-Processor 可重放能力。