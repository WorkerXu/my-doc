# 基于 Crawl4AI 的 AI 友好型文档爬取工具

## 1. 调研对象与可验证范围

调研条目：

- 文章：Based on Crawl4AI - AI友好的文档爬取工具
- 地址：https://linux.do/t/topic/428706
- 文章中给出的项目地址：https://github.com/JlonGit/based-on-Crawl4Al

文章描述的工具由 Crawl4AI 二次封装而来，目标是更方便地把文档型网站转成知识库素材，明确宣称具备：

1. 基本网页抓取；
2. PDF 抓取；
3. JSON 与 Markdown 输出；
4. Markdown 可读性优化；
5. 通过命令参数切换抓取深度和输出目录。

当前检查时，文章给出的 GitHub 仓库地址已返回 404，因此无法把仓库源码中的具体实现细节当作已验证事实。文章同时明确引用了其思路来源“一个爬取文档型站点的小脚本，方便生成知识库”，该公开文章包含完整 Python 源码：https://linux.do/t/topic/424863 。下面的源码级分析以这个公开脚本作为可验证实现参照，同时结合 Crawl4AI 官方文档分析技术原理；不会把来源脚本的每一行代码等同于已经失联的二次封装仓库。

Crawl4AI 官方参考：

- Quick Start / Markdown：https://docs.crawl4ai.com/core/quickstart/
- Browser/Crawler Config：https://docs.crawl4ai.com/core/browser-crawler-config/
- Multi-URL Crawling：https://docs.crawl4ai.com/advanced/multi-url-crawling/
- Deep Crawling：https://docs.crawl4ai.com/core/deep-crawling/
- PDF Parsing：https://docs.crawl4ai.com/advanced/pdf-parsing/

## 2. 公开脚本的真实执行流程

公开源码的主流程可以还原为：

```text
CLI(seed_url, output_dir)
  -> extract_links(seed_url)
      -> 创建 AsyncWebCrawler
      -> arun(seed_url)
      -> result.links["internal"]
      -> set 去重
  -> crawl_concurrently(urls)
      -> asyncio.Semaphore(max_concurrent)
      -> 每个 URL 创建独立 AsyncWebCrawler
      -> CrawlerRunConfig(
           word_count_threshold=200,
           wait_until="networkidle",
           page_timeout=120000
         )
      -> arun(url)
      -> done callback 中异步 save_markdown()
  -> URL path 映射本地目录和 .md 文件
```

这段实现非常适合作为“几十到几百页文档站的个人脚本”，但不能直接扩展成 1000 站长期运行的知识库系统。

## 3. 链接发现原理：实际是一跳发现，不是可证明的全站递归

源码中的 `extract_links()` 只对种子 URL 调用一次 `crawler.arun(url)`，读取：

```python
links = result.links.get("internal", [])
```

然后直接把这批链接交给并发抓取。后续抓每个页面时没有再把新发现的链接写回 frontier，因此从可验证代码看，它是：

```text
seed page -> seed page 上的一批 internal links -> 抓正文
```

而不是：

```text
seed -> children -> grandchildren -> ... -> 直到 frontier 穷尽
```

这点很关键。文档站常见的侧边栏确实可能一次列出几乎全站页面，所以“一跳”在 MkDocs、Docusaurus、VitePress 等站点上经常看起来像“抓全了”；但这只是站点模板结构带来的偶然覆盖，不是算法保证。

典型漏页场景：

- 侧边栏只显示当前章节，其他章节点击后才出现；
- 折叠菜单默认不渲染全部 href；
- CSR/虚拟列表只渲染可见项；
- 老版本文档藏在版本切换器里；
- 多语言文档藏在 locale selector 中；
- API reference 由动态路由生成，不在首页导航；
- PDF/附件只从正文二级页面链接；
- orphan page 只在 Sitemap/API 中存在。

因此方案中新增 `DOC_NAVIGATION` Provider 是有价值的，但它必须作为一等“发现证据”，而不能把 `result.links["internal"]` 本身当完整性证明。

## 4. 为什么要把文档导航树单独建模

文档站的侧边栏和目录与普通正文链接不同，它具有结构语义：

```text
Getting Started
  Installation
  Quick Start
API
  Client
    create()
    delete()
```

如果只保存 URL 集合，会丢掉：

- tree path/breadcrumb；
- 层级；
- 顺序；
- 章节父子关系；
- version/locale 上下文；
- 某个导航快照中是否缺页的覆盖证据。

因此更合理的持久化模型是：

```text
navigation_snapshot
  source_url
  render_mode
  container_signature
  navigation_hash
  version_selector_state
  locale_selector_state
  captured_at

navigation_entry
  href
  anchor_text
  tree_path
  level
  order
  version_hint
  locale_hint
  source_locator
```

`navigation_hash` 可用于检测文档目录变化，是一个很便宜的增量信号；但目录未变不代表页面正文未变，所以它不能替代 ETag、Last-Modified、body hash、IR hash 和周期 Reconcile。

## 5. `set()` 去重的隐性问题

源码使用类似：

```python
unique_links = list(set(...))
```

对于个人抓取脚本足够简单，但生产系统有两个问题：

1. `set` 不是持久化去重，进程重启后状态丢失；
2. 直接集合化会丢掉导航顺序和重复出现位置。

导航顺序本身对技术文档有用：可以恢复 breadcrumb、章节顺序、默认发布目录，也可以发现同一 URL 是否同时出现在多个导航节点。

生产方案应：

- 用 PostgreSQL 唯一约束控制 URL 幂等；
- 用 `discovery_evidence` 保存每次来源；
- 用 `navigation_entry.order/tree_path` 保存结构；
- 内存 Set/Bloom 只做局部加速。

## 6. 并发模型：Semaphore 有价值，但作用域太小

源码后续更新加入：

```python
semaphore = asyncio.Semaphore(args.max_concurrent)
```

这比无限 `asyncio.gather()` 好，说明作者已经意识到并发控制问题。原理是每个抓取协程在进入实际网络/浏览器工作前获取 permit，结束后自动释放，从而限制单进程并发数。

问题是：对 1000 站生产系统，单个 Python 进程里的 Semaphore 不是全局限流。

例如有 20 个 Worker，每个 `max_concurrent=10`，同一域名理论上可能瞬间出现 200 个并发请求。因此要分层：

```text
全局容量
 -> site/domain quota
 -> runtime class quota
 -> worker-local dispatcher/semaphore
```

Crawl4AI 当前提供 `MemoryAdaptiveDispatcher`、`SemaphoreDispatcher`、`RateLimiter` 和 `arun_many(stream=True)`，非常适合作为 Worker 内部调度器；跨 Worker 的 domain quota、任务 lease、幂等和公平性仍应由系统状态层管理。

## 7. 每 URL 创建 AsyncWebCrawler 的成本

源码在每个 `limited_crawl(url)` 内部：

```python
async with AsyncWebCrawler(...) as crawler:
    result = await crawler.arun(...)
```

这意味着并发抓 100 个 URL 时，生命周期上会反复创建和关闭 crawler/browser runtime。对于 Crawl4AI/Playwright 路径，这会带来：

- Chromium 启动成本；
- context/page 初始化；
- 内存峰值；
- 字体、JS runtime、网络栈反复热身；
- 浏览器 crash 概率上升；
- 整体吞吐下降。

生产上应改成长期 runtime：

```text
Worker 启动
 -> 初始化 AsyncWebCrawler
 -> claim 一小批 URL
 -> arun_many(stream=True) 或受控 arun
 -> 每个结果及时提交
 -> 继续复用 runtime
 -> 到 max age/max pages/RSS 阈值滚动重启
```

这也是现有主方案中“Browser/Crawl4AI 长生命周期 Runtime”的技术依据之一。

## 8. 一个容易忽略的异步完成性问题：done callback 没有被 gather

源码中的模式是：

```python
task = asyncio.create_task(limited_crawl(url))
task.add_done_callback(
    lambda t, url=url: asyncio.create_task(
        _handle_crawl_result(t, url, args.output)
    )
)
tasks.append(task)

await asyncio.gather(*tasks)
```

`gather(*tasks)` 只等待 `limited_crawl` 本身完成，不等待 done callback 新创建的 `_handle_crawl_result` 任务。

也就是说存在这个时序：

```text
抓取任务全部完成
 -> gather 返回
 -> main 打印 All done
 -> 部分 save_markdown 仍在事件循环里运行
 -> main 返回
 -> asyncio.run 开始关闭 loop
```

如果写盘任务还没完成，理论上会出现“抓取协程完成，但 Markdown 没有完全落盘”的窗口。

个人脚本通常因为写文件很快而不容易触发，但在大文件、慢磁盘、NFS、对象存储或高并发时会变成真实一致性问题。

生产系统不能把“协程完成”当业务完成，应该：

- 使用 `TaskGroup`/明确 gather 把派生任务纳入父阶段；或
- 更进一步，让抓取、Artifact commit、领域写入都有持久化 task state；
- 最后由 Stage Finalizer 对数据库期望集合和 COMPLETE Artifact 对账。

这也是本次方案新增“任何派生异步保存都必须处于可追踪完成范围”的原因。

## 9. `networkidle` 作为默认等待条件的问题

公开脚本配置：

```python
wait_until="networkidle"
page_timeout=120000
```

`networkidle` 的直觉是“页面网络请求安静后再提取”，对于部分 SPA 很方便，但不适合作为所有页面的默认值。技术博客可能长期存在：

- analytics beacon；
- WebSocket；
- long polling；
- 广告请求；
- 延迟加载资源；
- 无限滚动预取。

这些会导致页面迟迟不进入网络空闲状态，形成长尾等待。当前 Crawl4AI 官方 `CrawlerRunConfig` 的默认 `wait_until` 已是 `domcontentloaded`，也支持 `wait_for`、`delay_before_return_html`、JS 等更明确的 readiness 控制。

生产策略应优先：

```text
普通静态页 -> domcontentloaded
稳定正文/导航容器 -> domcontentloaded + wait_for(css)
SPA -> wait_for(css/js)
Load More -> 显式 action plan + 最大轮数
没有更可靠信号时 -> networkidle
```

Readiness 需要版本化并记录证据，否则某次站点改版后空 Markdown 很难解释。

## 10. `word_count_threshold=200` 为什么会误伤技术文档

Crawl4AI 官方说明中 `word_count_threshold` 的语义是过滤低于阈值的文本块，它适合去除短噪声，但不是“页面是否值得抓”的业务规则。

技术文档中大量高价值页面天然很短：

- 一个 API method；
- CLI option；
- 错误码说明；
- 配置字段；
- migration note；
- 一小段代码示例；
- FAQ。

固定 200 字阈值可能造成“URL 已发现并抓取成功，但核心正文块被过滤”，最后生成极短或空 Markdown。

因此：

- URL admission 不能受这个阈值控制；
- extraction candidate 可以保留 raw/fit 两种结果；
- Quality Validator 按 page type 使用不同阈值；
- 短文档必须有明确 accepted/rejected-with-reason，而不是静默消失。

## 11. Markdown 输出原理与为什么不能直接把 `result.markdown` 当最终真相

Crawl4AI 本身能从抓取结果产生 Markdown，并区分 raw Markdown 与应用 content filter 后的 fit Markdown。作为个人知识库，这非常方便。

但长期知识库若直接把 Crawl4AI Markdown 当唯一 canonical 数据，会产生几个问题：

1. 升级 Crawl4AI/Markdown generator 后，同一页面输出可能变化；
2. 想更换 Renderer 时需要重新访问源站；
3. JSON、Markdown、向量索引若各自独立抽取，字段可能不一致；
4. 很难精确保存 HTML selector、PDF page 等 provenance。

更稳的链路是：

```text
Raw Snapshot
 -> Extraction Candidate
 -> Canonical Document IR
 -> JSON Projection
 -> Markdown Projection
 -> Search/Vector Projection
```

Crawl4AI `raw_markdown/fit_markdown` 仍应保存，作为候选、调试、回归比较和降级结果；最终 accepted Markdown 由版本化 Renderer 从 Document IR 生成。

## 12. 路径映射与文件写入的几个数据一致性问题

源码使用 URL path 构建本地文件，例如：

```text
/docs/a/b -> output/docs/a/b.md
```

这个做法直观，适合手工浏览，但不能承担 identity。

### 12.1 Query 丢失导致冲突

源码明确忽略 query 和 fragment。如果两个合法页面是：

```text
/post?id=1
/post?id=2
```

它们可能映射到同一个 `post.md`。

### 12.2 “文件已存在就 return”破坏增量同步

源码：

```python
if filepath.exists():
    return
```

这会把文件存在误当成内容已同步。页面更新、抽取规则升级、Markdown renderer 改版后都不会重写。

长期系统应以：

```text
URL identity
Document identity
Document version
Publish path
```

四层分离。内容是否变化看 Snapshot/IR hash；发布器对 current projection 原子 replace，同时保留旧 document_version。

### 12.3 `sanitize_filename()` 定义但没有实际使用

公开源码定义了文件名清理函数，但在保存路径构造中没有调用。跨 Windows/Linux、特殊字符、URL 编码和超长路径时会产生兼容性问题。

生产方案不允许 Worker 直接根据不可信 URL 拼本地路径；统一由 Publish Profile 生成确定性、安全的输出路径。

## 13. PDF 支持应通过 Content-Type 路由，而不是把扩展名当真相

待调研文章的二次封装项目明确宣称支持 PDF。当前 Crawl4AI 官方也提供 `PDFCrawlerStrategy` 与 `PDFContentScrapingStrategy`，可抽取页级文本、metadata、图片，并支持 batch 处理。

生产系统仍不应该只通过 `.pdf` 后缀判断，因为真实站点可能：

- `/download?id=123` 返回 PDF；
- `.pdf` URL 实际返回登录 HTML；
- Content-Type 配错；
- Content-Disposition 才提供文件类型。

所以路由要综合：

```text
Content-Type + Content-Disposition + magic bytes + URL hint + parser probe
```

PDF 应保存原始二进制 Artifact，然后 native parser；文本覆盖低时再进入 OCR。这样 parser/OCR 升级可以离线重处理，不必重复下载。

## 14. 两阶段抓取比“每个 URL 都完整 Crawl4AI”更适合 1000 站

文章/脚本最值得吸收的思想，是“先从入口拿 URL，再并发抓内容”。生产化后应把它明确升级为两个 profile：

### Discovery Fetch Profile

目标：尽可能便宜地枚举 URL 和发现证据。

- HTTP 优先；
- 只解析链接、Sitemap、导航、分页；
- 默认不做 LLM 抽取；
- 默认不生成最终 Markdown；
- 不下载大媒体；
- CSR 导航看不到时才升级 Browser；
- 结果写 durable frontier。

### Content Fetch Profile

目标：把已经 admitted 的 URL 做高质量内容处理。

- 条件请求；
- Content-Type Router；
- HTML extraction；
- 必要时 Browser/Crawl4AI；
- PDF/OCR；
- Document IR；
- Quality；
- Publish。

这个分层可以显著减少 Browser 使用量，同时使发现完整性与正文质量解耦。

## 15. 抓取深度参数应该是预算，不应该定义“抓完了”

文章二次封装项目的卖点之一是能方便切换抓取深度。这对个人工具很有用，但生产系统要区分两个概念：

```text
execution slice budget != historical completeness
```

`max_depth=3`、`max_pages=500`、`max_runtime=10min` 都只是这一轮 Worker 的执行预算。达到预算时应保存 checkpoint 并重新排队，而不是把 Run 标记为历史全量完成。

真正的 FULL_BACKFILL 完成应由 Provider Coverage + durable frontier + retry/dead-letter + Reconcile 共同证明。

## 16. 对 1000 站方案的具体优化结论

本次调研对既有方案不是架构方向上的推翻，而是补上了一个此前不够显式的能力：**文档站官方导航结构本身是一类高价值 Discovery Provider**。

已纳入主方案的优化：

1. 新增 `DOC_NAVIGATION` Provider；
2. 新增 `navigation_snapshot/navigation_entry`；
3. 保存 tree path、level、order、version/locale hint；
4. 新增 navigation hash 作为增量信号；
5. Probe 显式识别侧边栏、章节树、版本/语言 selector；
6. Discovery Fetch Profile 与 Content Fetch Profile 分离；
7. HTTP 看不到导航时，允许因 `NAVIGATION_NOT_VISIBLE` 升级 Browser Discovery；
8. 文档站版本/语言作为明确维度保存，避免 v1/v2 或中文/英文被错误互相覆盖；
9. 异步派生保存任务必须可追踪，禁止用未 gather 的 callback 证明完成；
10. Web 增加导航树预览、目录条目数、已抓数、缺口和版本/语言诊断；
11. Golden Navigation 纳入运行时发布测试；
12. 验收中加入静态/CSR/折叠菜单/局部导航/版本/多语言文档站。

## 17. 哪些文章思路保留，哪些只适合个人脚本

### 应保留

- Crawl4AI 作为网页/动态站点抓取能力；
- 从种子入口自动发现文档链接；
- 并发控制；
- Markdown 输出；
- PDF 作为知识库内容；
- CLI/配置化抓取范围和执行预算；
- 文档型站点优先利用官方导航结构。

### 不直接沿用

- seed 页面一跳 internal links = 全站；
- 每个 URL 新建一个 AsyncWebCrawler；
- 所有页面默认 `networkidle`；
- 固定 `word_count_threshold=200` 作为通用策略；
- `asyncio.gather` 只等待抓取、不等待派生写盘；
- `set()` 作为最终去重；
- URL path 直接作为唯一文件 identity；
- 文件存在就跳过；
- 本地 CLI 的 Semaphore 当全局限流；
- 抓取深度达到上限就认为覆盖完成。

## 18. 最终判断

这篇文章的价值主要不是提供一套可直接部署到 1000 站的系统，而是验证了一个非常实用的产品形态：**给一个文档入口，自动利用站内导航发现页面，再用 Crawl4AI 把网页/PDF 转为 AI 友好的 Markdown/JSON**。

对于个人或小规模一次性知识库，这个方向简单有效；对于长期生产知识库，必须把其中隐含的“导航发现、并发、深度、输出目录”全部提升为持久化、版本化、可审计的领域能力。

本次最终方案因此采用：

```text
DOC_NAVIGATION Provider
+ Navigation Snapshot
+ Discovery/Content 两阶段抓取
+ durable frontier
+ HTTP-first / Browser escalation
+ 长生命周期 Crawl4AI runtime
+ Content-Type/PDF 路由
+ Canonical Document IR
+ Quality Validator
+ Stage Finalizer
+ Publish Manifest
```

这样既保留了文章方案“文档站低成本接入”的优点，又避免把个人脚本中的一跳发现、进程内状态和直接文件写盘放大到 1000 站后形成不可恢复、不可证明完整、无法增量同步的问题。