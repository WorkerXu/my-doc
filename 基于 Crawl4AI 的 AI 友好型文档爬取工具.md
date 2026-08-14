# 基于 Crawl4AI 的 AI 友好型文档爬取工具：实现细节与技术原理分析

## 1. 调研对象与证据边界

- 编号：44
- 原始文章：https://linux.do/t/topic/428706
- 文章中给出的项目地址：https://github.com/JlonGit/based-on-Crawl4Al
- 上游参考帖：https://linux.do/t/topic/424863
- 上游技术底座：https://github.com/unclecode/crawl4ai
- Crawl4AI 官方文档：https://docs.crawl4ai.com/

原帖可确认：作者在 Crawl4AI 基础上用 Cursor 做了一个更易用的文档抓取工具，支持基本网页和 PDF 抓取、JSON/Markdown 输出、抓取深度和输出目录切换。调研时项目仓库通过 GitHub API 返回 404，因此无法把“原项目实际源码”与文章描述一一核对。

这意味着本文把证据分为三层：

1. **原帖明确事实**：只陈述原帖直接说出的功能；
2. **可核对的上游实现**：分析原帖引用的前置脚本中公开的 Crawl4AI 用法；
3. **生产化推导**：结合 Crawl4AI 当前官方能力和 1000 站长期知识库需求提出架构建议，不把推导写成原项目事实。

## 2. 原帖明确确认的能力

原帖确认的能力只有以下几项：

1. 基于 Crawl4AI 二次封装；
2. 面向文档型站点和普通网页；
3. 增加 PDF 抓取；
4. 支持 JSON 输出；
5. 支持 Markdown 输出，并强调生成内容可读性；
6. 可通过参数切换抓取深度；
7. 可切换输出目录。

因此这个项目的价值不在于重新实现浏览器，而在于把 Crawl4AI 的底层能力包装成“给一个站点就得到 AI 友好文档”的更直接工作流。

## 3. 上游参考脚本可验证的实现方式

原帖来自另一个“抓文档型站点生成 Markdown”的脚本。该上游脚本公开了完整核心代码，可用于理解此类工具最初的实现思路。

### 3.1 URL 发现：从种子页读取 internal links

上游脚本先执行一次 `crawler.arun(seed_url)`，然后读取：

```text
result.links["internal"]
```

把内部链接去重后作为待抓列表。

这个实现的本质是：

```text
Seed URL
 -> 页面渲染/解析
 -> 读取当前页面中的 internal links
 -> 对这些 URL 批量抓取
```

它非常适合“左侧导航包含完整目录”的文档站，因为种子页往往一次就能暴露几十到几百个文档链接。

但它不是严格意义上的全站历史发现器。公开脚本只对种子页提取一次链接，没有把每个子页面中新发现的链接重新写入持久化 frontier。因此如果历史文章只能通过第二层、归档页、分页、标签页或 Sitemap 才可到达，就可能漏抓。

### 3.2 并发：Semaphore 保护固定并发

上游更新后的脚本使用：

```text
asyncio.Semaphore(max_concurrent)
```

每个 URL 在进入抓取前获取信号量，实现简单的进程内并发上限。这比无限 `asyncio.gather` 好得多，也适合百级 URL 的单机工具。

但固定 Semaphore 只回答“这个进程最多同时跑几个任务”，没有回答：

- 不同域名应该各自多少并发；
- 429/503 后是否动态降速；
- Browser、HTTP、PDF/OCR 是否应该共用配额；
- 内存逼近上限时是否停止继续开任务；
- 多个 Worker/Pod 之间怎样保证同一站点的总并发。

Crawl4AI 当前已经提供 `arun_many()`、`MemoryAdaptiveDispatcher`、`SemaphoreDispatcher` 和 `RateLimiter` 等能力，可用于 Adapter 内部的批量执行、内存感知和限速。但对 1000 站生产系统，真正的“站点级总配额”仍必须由系统级调度层统一控制，不能只靠每个 Crawl4AI 进程自己的 Semaphore。

### 3.3 浏览器生命周期：原型存在每 URL 新建 crawler 的高开销

上游脚本在单个 URL 的抓取函数内部使用：

```text
async with AsyncWebCrawler(...) as crawler:
    result = await crawler.arun(...)
```

也就是说每个 URL 都创建和销毁一个 `AsyncWebCrawler` 生命周期。对少量页面可以接受，但对大规模历史回灌会带来明显问题：

- Chromium/context/page 初始化和销毁成本被重复支付；
- TCP/TLS、浏览器缓存和页面上下文无法有效复用；
- 高频启动浏览器更容易形成内存抖动；
- 对数十万 URL 时吞吐与资源消耗会非常差。

Crawl4AI 官方文档支持在一个较长生命周期的 `AsyncWebCrawler` 内重复调用 `arun()`，也支持 `arun_many()` 批处理和 session reuse。因此生产实现应该是：

```text
Browser Worker 启动
 -> 创建长期 crawler/browser runtime
 -> claim 一批任务
 -> arun_many(stream=True) 或受控 arun
 -> 持续复用 browser/context/page pool
 -> 达到 max age/max pages/内存阈值后滚动重启
```

而不是：

```text
每个 URL
 -> 启浏览器
 -> 抓取
 -> 关浏览器
```

### 3.4 页面就绪：`networkidle` + 120 秒超时是粗粒度策略

上游示例对动态页面使用：

```text
wait_until="networkidle"
page_timeout=120000
```

这是一个“尽量等页面安静下来”的通用策略，开发调试方便，但在生产场景会产生两类问题：

1. 有持续埋点、广告、WebSocket、轮询请求的页面可能长时间无法达到理想 network idle；
2. 大量静态文档其实 `domcontentloaded` 后很快就可解析，强制 `networkidle` 会浪费时间。

Crawl4AI 当前支持 `wait_until`、CSS/JS `wait_for`、`delay_before_return_html`、`js_code`、`scan_full_page`、session 等更细粒度控制。生产方案应把“页面已准备好”建模成版本化 Browser Readiness/Action Plan，例如：

```text
默认：domcontentloaded
正文有明确容器：wait_for="css:article"
SPA：wait_for="js:() => window.__DATA_READY__ === true"
无限滚动：显式 scroll/action plan + max rounds
特殊站点：必要时 networkidle
```

因此 `networkidle` 应是某类站点的可选策略，而不是全局默认。

### 3.5 `word_count_threshold=200`：适合去噪，但不能作为完整性门槛

上游示例使用较高的 `word_count_threshold`。这对于减少导航、小块文本有效，但技术文档中大量页面本身就很短，例如：

- 单个 API 方法说明；
- 一个配置项；
- 一个错误码；
- 单个 CLI 参数；
- 一个短代码示例。

如果把固定字数阈值当“是否收录”的业务规则，就会把合法历史文档误删。

因此生产方案可以把 word count 当候选清洗信号，但不能作为 URL admission、历史覆盖或 accepted document 的唯一门槛。是否保留应由页面类型、结构、元数据、链接证据和质量规则共同判断。

### 3.6 输出路径：URL path -> 本地 Markdown 文件

上游脚本把 URL path 拆成目录，并把最后一段作为 `.md` 文件名；如果目标文件已存在则直接返回。

这个做法对一次性导出很直观，但长期增量存在四个问题：

1. `/post?id=1` 和 `/post?id=2` 可能映射到同一个 `post.md`；
2. redirect/canonical 后同一文档可能有多个路径；
3. 页面更新后因为文件已存在而被静默跳过，无法增量覆盖；
4. URL 改 slug 后可能生成第二份文件，无法表达“同一个文档的新地址”。

所以生产系统必须把“文档身份”“URL 身份”和“发布文件路径”分开：

```text
URL -> url_id
canonical/redirect/content evidence -> document_id
accepted document_version -> publish projection
publish profile -> stable output path
```

发布路径可以继续仿站点目录，但必须来自稳定的 `document_id + slug/path mapping`，并由 publish manifest 管理，不允许以“文件存在”作为内容已同步的判断。

### 3.7 去重：`set()` 只是单次内存 URL 去重

上游脚本使用 `set()` 去掉同一次发现中的重复链接。这个手段简单有效，但只能处理同一进程当前内存里的精确字符串重复，不能处理：

- tracking 参数；
- fragment；
- URL 大小写/编码差异；
- trailing slash；
- redirect；
- canonical；
- session 参数；
- 同内容不同 URL；
- Worker 重启后的重复。

生产系统必须使用持久化 URL normalization + 数据库唯一约束 + document identity evidence，Bloom/Set 只能作为前置加速。

## 4. Crawl4AI 当前底层能力与生产映射

### 4.1 批量抓取与 Dispatcher

Crawl4AI 当前 `arun_many()` 支持：

- 多 URL 并发；
- MemoryAdaptiveDispatcher；
- SemaphoreDispatcher；
- RateLimiter；
- streaming results；
- URL-specific configs。

这很适合作为 Browser/Crawl4AI Adapter 内部执行器。生产系统的正确边界是：

```text
系统 Scheduler/Frontier
 -> 决定哪些任务现在允许执行
 -> 给 Crawl4AI Adapter 一小批已授权 URL
 -> Adapter 用 arun_many/dispatcher 高效执行
 -> 每个结果独立提交 attempt/snapshot
```

不要反过来把数十万 URL 一次性交给 `arun_many()`，然后依赖一个进程内 dispatcher 充当业务队列。长任务需要 DB lease、checkpoint、重试、取消和跨进程恢复。

### 4.2 Deep Crawl

Crawl4AI 支持 BFS/DFS/Best-First Deep Crawl，并提供 `max_depth` 等参数。

对小型工具，`max_depth=2/3` 是非常方便的用户参数；但对历史全量知识库：

```text
max_depth = 本次探索预算
而不是 = 历史完整性的证明
```

历史完整性应来自多个 Provider：

```text
Sitemap/Sitemap Index
Feed/API
Archive/Category/Tag/Author
普通站内链接
Browser dynamic listing
周期 Reconcile
```

Deep Crawl 的结果写回 durable frontier；每次 slice 用尽后保存 checkpoint，下一次继续，而不是“到最大深度就宣布完成”。

### 4.3 Markdown 生成

Crawl4AI 的 Markdown 结果可包含：

```text
raw_markdown
markdown_with_citations
references_markdown
fit_markdown
fit_html
```

其中 `fit_markdown` 可以通过 Pruning/BM25 等过滤器压低导航和模板噪声。

对知识库，建议把这些视为“候选产物”而非唯一真相：

```text
Raw Snapshot
 -> cleaned_html
 -> extraction candidates
 -> Canonical Document IR
 -> versioned Markdown Renderer
```

原因是：过滤阈值和算法升级会改变 fit markdown；如果只保存最终 Markdown，就无法可靠重放和比较。保留 raw/cleaned/IR 可以让 renderer 升级时离线重生成，而不重新请求源站。

### 4.4 PDF

Crawl4AI 官方提供 PDFCrawlerStrategy/PDFContentScrapingStrategy 路径，可把 PDF 页级文本、metadata、图片等转成 CrawlResult 风格的结果。

原帖明确宣称加入 PDF 抓取，但因为项目仓库不可读，不能确认它当时具体用了哪一种 PDF parser、是否支持 OCR、是否保留页码。因此生产方案不能把这些推断写成该项目事实。

从需求角度，PDF 必须成为一等 Document：

```text
PDF URL
 -> HTTP download
 -> immutable RAW_PDF artifact
 -> native PDF parser
 -> page text / metadata / images / links
 -> Document IR candidate
 -> quality validator
 -> accepted version
```

扫描件或低文本密度页面再走 OCR：

```text
native parse
 -> image-only / low text density
 -> OCR_REQUIRED
 -> OCR page result + confidence
 -> new candidate
```

原 PDF 永远保留，OCR 不覆盖源文件。

### 4.5 JSON 与 Markdown 双输出

“JSON + Markdown”在单机工具里常表示两个输出文件；生产系统应该把它提升为同一 Canonical IR 的多个 Projection：

```text
Document IR
 -> JSON Projection
 -> Markdown Projection
 -> Search Projection
 -> Vector/External KB Projection
```

这样可以避免“Markdown 和 JSON 分别从不同解析链生成，内容互相矛盾”。

## 5. 对 1000 个技术博客的关键架构修正

### 5.1 增加版本化 Crawl Scope

原型里的“internal link / same domain”在生产上必须被明确化。

建议增加 `crawl_scope_release`，至少定义：

```text
allowed_hosts
allowed_registrable_domains
include_subdomains
path_prefixes
path_excludes
query_policy
follow_redirect_outside_scope
asset_hosts
canonical_cross_host_policy
```

为什么要单独建模：

- 博客正文可能在 `blog.example.com`，图片在 `cdn.example.com`；
- 历史版本可能在 `v1.docs.example.com`、`v2.docs.example.com`；
- `www.example.com` 与 `example.com` 可能互跳；
- 某些外部 canonical 是镜像或迁移证据，不代表允许自动继续抓整个外站。

Scope 必须冻结进 Run Context，避免运行中管理员改 allowlist 导致同一 Run 前后语义不同。

### 5.2 Browser Worker 采用长期 Runtime + 小批量流式执行

推荐：

```text
Worker 生命周期
 -> 初始化 browser/crawler
 -> claim 20~100 个已准入任务
 -> arun_many(stream=True)
 -> 每个 result 立即提交 Snapshot/Attempt
 -> 下游逐个消费
 -> 达到内存/页数/寿命阈值滚动重启
```

不要：

```text
每个 URL 建一个 crawler
或
一次把百万 URL 交给一个 asyncio.gather/arun_many
```

Crawl4AI Dispatcher 是执行层优化，数据库 task/lease 才是恢复层。

### 5.3 页面就绪改为 Readiness Contract

Browser Action Plan 中新增/明确：

```text
wait_until
wait_for
wait_for_timeout
delay_before_return_html
js_code_before_wait
js_code
scan_full_page
scroll_delay
max_scroll_rounds
session_policy
resource_block_policy
```

默认优先 `domcontentloaded`，只有站点证据表明需要时再升到 `networkidle`。正文有稳定 selector 时优先 `wait_for=css:...`，比“等待整个网络安静”更确定。

### 5.4 不允许“文件已存在 = 已同步”

增量同步判断必须基于：

```text
ETag / Last-Modified
Sitemap lastmod
Feed/API cursor
Snapshot body hash
Document IR hash
accepted version
publish_state
```

文件树只是发布视图。发布器必须支持原子 replace/rename、manifest 和版本指针，确保页面更新后可安全替换当前输出，同时历史 `document_version` 仍保留。

### 5.5 word count 只做信号，不做完整性过滤

建议：

- Discovery/Admission 不使用固定正文长度阈值拒绝 URL；
- Extractor 可用 word count 参与候选评分；
- Quality Validator 根据 page type 使用不同阈值；
- API reference/CLI reference/短 FAQ 使用独立质量合同；
- FULL_BACKFILL 中低字数页面也必须有明确结果：accepted / rejected with reason / review，而不是静默消失。

## 6. 推荐生产数据流

```text
Site
 -> Discovery Probe
 -> Crawl Scope Release
 -> Provider Releases
 -> FULL_BACKFILL Run + immutable Run Context

Provider
 -> discovery evidence
 -> URL normalization
 -> scope/admission
 -> durable frontier

HTTP Fetch
 -> immutable RAW_HTTP/RAW_PDF
 -> Content-Type Router

HTML
 -> cheap HTTP extraction
 -> if evidence requires JS: Browser/Crawl4AI
 -> cleaned HTML / raw markdown candidates
 -> Document IR candidate

PDF
 -> native parser
 -> optional OCR
 -> Document IR candidate

Candidate
 -> deterministic Quality Validator
 -> accepted document_version
 -> JSON/Markdown/Search projections
 -> publish manifest
```

## 7. Content-Type Router

原帖同时支持网页和 PDF，这一点对现有方案最大的启发是：不要把“URL = HTML 页面”写死。

建议路由证据组合：

1. response `Content-Type`；
2. `Content-Disposition`；
3. magic bytes；
4. URL/path hint；
5. parser probe。

路由示例：

```text
text/html, application/xhtml+xml -> HTML
application/pdf                 -> PDF
application/json                -> API/JSON
RSS/Atom/XML                    -> Feed/Discovery
image/audio/video               -> Asset/Attachment
unknown/mismatch                -> Sniff/Review
```

记录：

```text
reported_content_type
sniffed_content_type
selected_route
route_reason
route_release_id
```

## 8. 统一 Document IR

HTML 和 PDF 不需要两套知识库模型。建议统一：

```text
document_id
source_type
source_url
canonical_url
metadata
blocks[]
assets[]
links[]
```

Block：

```text
heading
paragraph
list
quote
code_block
table
image
link
embed_placeholder
```

每个 block 可以保存 `source_locator`：

```text
HTML: selector/xpath
PDF: page + bbox
OCR: page + bbox + confidence
```

这样检索命中后可以回溯到原 HTML 或 PDF 第几页，也能对 extractor/renderer 做可靠回归测试。

## 9. Durable Frontier 与发现完整性

生产系统不依赖内存 `set()`，而使用持久化唯一键：

```text
(site_id, sha256(normalized_url))
```

发现来源同时保留 evidence：

```text
source_provider
source_url
href/entry/guid
observed_at
raw_evidence_ref
```

FULL_BACKFILL 完成至少要求：

- 主要 Provider 枚举完成或显式记录 coverage gap；
- durable frontier 无可执行任务；
- retry/dead-letter 已处理；
- Reconcile 不再产生显著新增；
- Stage Finalizer 对真实 Snapshot/Document/Artifact 完成对账。

`max_depth/max_pages/max_runtime` 都只是 slice budget。

## 10. 并发、限流与 Backpressure

至少分四层：

```text
全局配额
 -> site/domain 配额
 -> runtime class 配额
 -> worker-local dispatcher
```

建议 runtime class：

```text
HTTP
BROWSER
PDF
OCR
CPU_EXTRACT
VALIDATOR
```

每域维护：

```text
max_concurrency
token_bucket_rate
burst
min_delay
429_penalty
circuit_state
browser_quota
```

Crawl4AI `MemoryAdaptiveDispatcher`/`RateLimiter` 可作为 Browser Adapter 的局部实现，但不能替代跨 Pod 的站点级配额。

下游积压时优先降低 Discovery 和 Browser expansion，避免“发现比抽取快几个数量级”把 Redis/数据库/对象存储压爆。

## 11. 失败语义与重试

原型里异常通常打印日志后返回空结果；生产系统必须结构化错误：

```text
DNS/TLS/CONNECT_TIMEOUT/READ_TIMEOUT
HTTP_429/HTTP_5XX/HTTP_404_410
ROBOTS_DENIED
CONTENT_TOO_LARGE
CONTENT_TYPE_MISMATCH
BROWSER_TIMEOUT/BROWSER_CRASH
READINESS_TIMEOUT
PARSE_FAILED/EXTRACTION_FAILED
PDF_PARSE_FAILED/PDF_ENCRYPTED
OCR_FAILED
QUALITY_FAILED
LEASE_LOST/CANCELLED
```

可重试错误使用 `Retry-After + jittered exponential backoff`。确定性配置错误不原样无限重试，应进入 probe/review。

## 12. Web 管理应展示的本次新增诊断信息

对单个 URL 建议显示：

```text
discovery evidence
crawl scope decision
normalized URL
frontier state
attempt history
reported/sniffed content type
selected route
HTTP vs Browser escalation reason
browser readiness plan
wait_until / wait_for / timeout
runtime/dispatcher stats
PDF page count / OCR state
raw/cleaned/fit Markdown candidates
Document IR
quality manifest
accepted version
publish path + publish manifest
```

站点页建议增加：

- 当前 crawl scope 及版本；
- Host/Subdomain 覆盖；
- Browser escalation rate；
- 平均 browser page lifetime；
- readiness timeout 分布；
- 429/503 与实际并发变化；
- short-document accepted/rejected 数；
- 输出路径冲突/重命名事件。

## 13. 安全与资源限制

网页和 PDF 混合抓取扩大攻击面：

- 所有 URL 通过 SSRF/Egress Gateway；
- DNS 解析和每次 redirect 后重新检查 IP/端口；
- 浏览器子资源执行网络策略；
- 不绕过验证码、登录、付费墙或明确访问限制；
- PDF parser/OCR 在低权限、受 CPU/内存/临时磁盘限制的隔离 Worker 中执行；
- 限制 PDF 下载字节、页数 slice、图片总像素、解析时长；
- 输出路径不能直接使用未经净化的第三方文件名或 URL path；
- Runtime/Parser/Crawl4AI 版本冻结进 Run Context。

## 14. 验收与压测建议

### 14.1 Browser 生命周期

准备 1000/10000 URL 的动态站测试：

- 比较每 URL 新建 crawler 与长期 runtime 的吞吐、RSS、CPU、失败率；
- 验证 max-pages/max-age 滚动重启不会丢任务；
- Worker 崩溃后 DB lease 可恢复。

### 14.2 Readiness Contract

对持续网络请求站点、SPA、静态站分别测试：

- `domcontentloaded`；
- CSS `wait_for`；
- JS `wait_for`；
- `networkidle`。

目标不是所有站统一一个等待策略，而是自动 Probe 能选出“最便宜且内容完整”的 plan。

### 14.3 短文档完整性

准备 API reference/CLI option/FAQ 等短页面，确保不会因为 `word_count_threshold=200` 静默丢失；任何排除都必须进入 quality/admission reason。

### 14.4 URL/path 冲突

覆盖：

```text
/post?id=1
/post?id=2
/foo
/foo/
redirect old -> new
canonical cross-host
slug rename
```

验证 document identity 和 publish path 不混淆，增量更新能替换 current projection，同时保留历史版本。

### 14.5 PDF

覆盖 native text PDF、扫描 PDF、损坏 PDF、加密 PDF、大 PDF。验证 parser/OCR 降级、页级 checkpoint、来源定位和资源限制。

## 15. 对现有主方案的最终优化结论

现有方案的 Content-Type Router、PDF/OCR、Document IR、durable frontier、Run Context、Artifact、Quality Validator 等方向是正确的。本次复核进一步补强四个实现层问题：

1. **Crawl Scope 版本化**：把“same domain/internal links”从隐含规则提升为可审计的 Run Context 配置；
2. **Browser 长生命周期与批量流式执行**：避免每 URL 重建 crawler，Crawl4AI Dispatcher 只负责 Adapter 内局部并发；
3. **Readiness Contract**：避免全局固定 `networkidle`，优先 `domcontentloaded + 明确 wait_for`，按站点证据升级；
4. **发布路径不承担同步状态**：禁止“文件已存在就跳过”，增量同步由 Snapshot/IR/version/publish_state 决定；
5. **固定字数阈值不参与历史完整性判定**：短 API 文档必须被保留或明确拒绝，不能静默消失。

这些改动不会推翻现有总体架构，而是把原型脚本中最容易在 1000 站规模下暴露的问题变成明确、可版本化、可测试、可观察的生产契约。

## 16. 参考资料

- 原始调研文章：https://linux.do/t/topic/428706
- 上游公开脚本：https://linux.do/t/topic/424863
- Crawl4AI 官方文档：https://docs.crawl4ai.com/
- Crawl4AI Multi-URL Crawling：https://docs.crawl4ai.com/advanced/multi-url-crawling/
- Crawl4AI arun_many：https://docs.crawl4ai.com/api/arun_many/
- Crawl4AI Deep Crawling：https://docs.crawl4ai.com/core/deep-crawling/
- Crawl4AI Browser/Crawler Config：https://docs.crawl4ai.com/core/browser-crawler-config/
- Crawl4AI Page Interaction：https://docs.crawl4ai.com/core/page-interaction/
- Crawl4AI Markdown Generation：https://docs.crawl4ai.com/core/markdown-generation/
- Crawl4AI PDF Parsing：https://docs.crawl4ai.com/advanced/pdf-parsing/
