# 竞争情报命令行工具（Competitive Intelligence CLI）

- 项目地址：https://github.com/qb-harshit/Competitve-Intelligence-CLI
- 项目类型：Python CLI / Crawl4AI 抓取工具
- 调研编号：49
- 调研日期：2026-08-14

## 1. 项目定位

该项目把竞争情报任务拆成“终端智能体负责分析，CLI 负责采集”。CLI 提供首页、Sitemap、价格、SEO 等抓取命令，采集后的数据保存在本地 JSON，再由 Claude Code、Cursor、Codex 等终端智能体读取并完成交叉分析。

这种定位的关键价值不是竞争情报本身，而是把“网页采集能力”做成可由 Agent 调用的确定性工具。对于博客知识库，可借鉴为：控制面/调度器负责计划，抓取 Adapter 负责执行；抓取器不掌握业务状态真相。

## 2. 包结构与依赖设计

`pyproject.toml` 中基础依赖只有 BeautifulSoup、lxml、requests；Crawl4AI 被放在 `scraping` optional dependency 中，CLI 在真正执行抓取前才检查 Crawl4AI 是否安装。

这说明项目主动把“基础 CLI”和“重型 Browser 抓取依赖”解耦。对长期知识库系统很有价值：

- Sitemap/Feed/HTTP-first 能独立运行；
- Browser/Crawl4AI 作为可选升级能力；
- 重型依赖故障或供应链风险不应让整个采集平台不可用；
- 生产环境应进一步固定 lock、镜像 digest 和 SBOM。

项目 README 也明确提醒 Crawl4AI 会带来较大的传递依赖链，因此供应链隔离是设计的一部分。

但其 `scraping` extra 使用的是 `crawl4ai>=0.8.5`，只给下界而没有锁定最终解析结果。对个人 CLI 可接受，对长期平台不可接受：同一份源代码在不同时间重新构建可能得到不同的 Crawl4AI、浏览器及传递依赖。正确做法是把“可选依赖”进一步升级为“运行时隔离”：HTTP/Discovery Worker 镜像根本不安装 Browser/Crawl4AI，Browser Worker 使用独立 lockfile、独立镜像 digest、独立 SBOM 和独立发布门禁，缩小故障与供应链爆炸半径。

## 3. CLI 调用链

核心入口在 `competitive_intel/cli.py`：

```text
cintel scrape homepage
cintel scrape sitemap
cintel scrape pricing
cintel scrape seo
```

执行前会：

1. 检查 Crawl4AI 是否可用；
2. 调用 `validate_url()`；
3. 创建 `data/companies/{company}_data.json`；
4. 构造具体 Scraper；
5. 使用 `asyncio.run()` 执行一次 CLI 任务；
6. 将结果写回本地 JSON。

对于单机 CLI，这是合理的生命周期；对于 1000 站长期服务端则不能照搬 `asyncio.run()` + 每任务新建执行环境，而应改成长生命周期 Worker event loop。

另一个工程风险是仓库同时保留根目录 `scrapers/`、`utils/`、`main.py` 与真正被 `pyproject.toml` 打包的 `competitive_intel/scrapers/`、`competitive_intel/utils/`、`competitive_intel/cli.py`。两套实现已经出现差异，例如打包版 Sitemap 抓取包含 URL 校验与 50MB 流式下载限制，而根目录版本并不完全一致。生产系统必须明确“可执行代码唯一来源”，构建、测试、审计都针对同一 package tree，避免修复了副本 A、生产实际运行副本 B 的漂移问题。

还存在一个容易被“代码用了 async”掩盖的运行时问题：`cmd_scrape_sitemap()` 的内部 `async run()` 以及 `analyze_and_scrape_sitemap()` 都会直接调用同步的 `fetch_sitemap()`，而该函数内部使用 `requests.get()`。因此在真正进入页面并发抓取前，Sitemap 的 DNS、连接、下载、重试等待全部会阻塞 event loop。单次 CLI 基本可接受，但长生命周期服务端 Worker 中会导致同进程其他协程饿死，放大慢站、超时和 429 的影响。生产实现应优先使用 `httpx.AsyncClient`/`aiohttp`；如果第三方库只能提供同步接口，则应隔离到有上限的 Blocking-I/O 线程池或独立 Worker，并对线程池队列同样施加 backpressure，不能直接在主 event loop 调用同步网络函数。

## 4. Sitemap 抓取实现

`competitive_intel/scrapers/sitemap_analyzer.py` 的核心流程是：

```text
fetch_sitemap
 -> parse_sitemap
 -> filter_urls_by_keywords
 -> categorize_urls
 -> scrape_feature_pages
 -> save_feature_data
```

### 4.1 下载保护

`fetch_sitemap()` 使用 `requests.get(..., stream=True)`，并设定约 50MB 的 `MAX_SITEMAP_BYTES`：

- 先检查 Content-Length；
- 再按 64KB chunk 读取；
- 即使服务器未提供 Content-Length，也能在累计字节超过上限时终止。

这是正确的防内存耗尽思路，但它只是“网络读取流式”，并不是真正的“解析流式”：实现会把所有 chunk 放进列表，再 `b''.join(chunks)` 生成完整 bytes，随后 `ET.fromstring()` 一次性构造 XML DOM。接近 50MB 上限时会同时存在 chunk 列表、拼接后的 bytes 和 XML 对象，峰值内存明显高于 50MB，而且无法在解析中途形成持久 checkpoint。

知识库系统应改为：

```text
HTTP stream
 -> 限制 encoded bytes
 -> 增量 gzip/decompression
 -> 限制 decoded bytes / compression ratio
 -> defusedxml/lxml iterparse
 -> 每解析一个 url/sitemap 条目立即写 durable frontier/sitemap_task
 -> 达到 slice 预算保存 checkpoint 并续跑
```

因此还应增加：

- gzip 解压后大小限制；
- 压缩比限制，防 zip bomb；
- 连接/读取分离超时；
- 安全 streaming XML parser，而不是读完再建 DOM；
- 单次执行预算耗尽后保存 checkpoint，而不是直接放弃剩余内容。

### 4.2 Sitemap Index 递归

解析时同时支持普通 `<urlset>` 和 `<sitemapindex>`。对嵌套 Sitemap：

- 递归抓取；
- 用 `_visited` 做循环检测；
- 用 `MAX_DEPTH = 3` 限制递归深度；
- 用 `MAX_URLS = 5000` 限制结果数量。

这些限制的初衷是 DoS 防护，但对于“全量历史博客”存在本质问题：固定 `MAX_URLS=5000` 会把安全预算变成数据截断条件。一个站点可能有多个 Sitemap，每个 Sitemap 数万 URL，达到上限后当前实现直接返回，剩余历史不会形成持久任务。

正确迁移方式不是取消限制，而是把限制从“全局完成条件”改成“execution slice”：

```text
一次最多解析 N 个条目
 -> 保存 sitemap checkpoint
 -> 剩余条目重新排队
 -> Worker 下次继续
```

同时 `_visited` 不能只存在内存，因为 Worker 重启后会丢失。生产系统应把 `sitemap_url + site_id` 做唯一键持久化。

### 4.3 URL 关键词过滤

项目支持对 Sitemap URL 做关键词匹配，只保留包含 features、pricing、docs 等目标词的页面。

这对竞争情报很合适，因为任务本身就是专题采集；对博客知识库 FULL_BACKFILL 不可直接照搬。关键词和页面类型最多用于：

- 提高文章详情页优先级；
- 选择不同抽取器；
- 降低明显非正文页优先级；
- TOPIC_EXPANSION 中做预算过滤。

历史全量阶段不能因为 URL 不含 `blog/article/docs` 就永久排除，否则会漏掉无语义 permalink、日期路径、数字 slug 等文章。

### 4.4 页面类型分类

项目通过 URL 模式把页面分为 features、products、pricing、customers、faq、api、documentation、other。

这说明一个非常实用的思想：先做廉价的 URL 级分类，再决定抓取优先级和后续处理策略。知识库系统可以扩展为：

```text
ARTICLE
ARCHIVE
CATEGORY
TAG
AUTHOR
PAGINATION
DOCUMENTATION
SEARCH
ASSET
UNKNOWN
```

但分类结果必须带置信度和规则版本，并保持可解释。

### 4.5 同步 I/O 混入异步路径

项目的 Sitemap 获取函数是同步 `requests`，但上层调用链已经进入 `asyncio.run()` 和 async 方法。其技术原理问题不是“同步库不能用”，而是“阻塞调用占用了事件循环线程”：当一个请求慢 20 秒时，同一 loop 中所有原本可继续执行的协程都无法获得执行机会。

生产系统需要把 I/O 类型作为 Adapter contract 的一部分：

```text
ASYNC_NATIVE      可直接运行在 asyncio Worker
BLOCKING_IO       只能进入 bounded blocking pool / 独立 Worker
CPU_BOUND         进入进程池或 CPU Worker
BROWSER           进入 Browser Worker
```

这样调度器可以从运行时层面阻止同步网络调用误入 async Worker，而不是依靠开发者记忆。

## 5. 并发模型

`scrape_feature_pages()` 使用：

```python
semaphore = asyncio.Semaphore(max_concurrent)
```

然后为每个 URL 创建协程，通过 semaphore 把实际并发限制在默认 5 个。这种“有界并发”原则是正确的。

问题在于实现仍会先为全部 URL 创建 task，再 `asyncio.gather(*tasks)`。当 URL 达到百万级时，task 对象本身会占用大量内存。因此生产系统应改成：

- bounded queue；
- worker consumer；
- `arun_many()`/streaming；
- 或每批 100~1000 条创建任务；
- 单条完成后立即持久化，不在内存中等待全部完成。

还需要把并发从“单函数 semaphore”提升为三层约束：全局并发、站点并发、域名并发，并叠加 QPS/token bucket。

## 6. Crawl4AI 页面抓取

`HomepageScraper.scrape_homepage()` 对每个页面执行：

```text
async with AsyncWebCrawler(...)
 -> crawler.arun(url, word_count_threshold=10, bypass_cache=True)
 -> result.cleaned_html
 -> BeautifulSoup clean_content
```

优点：

- Crawl4AI 负责页面抓取和 cleaned HTML；
- 抓取函数本身是 async；
- 结果失败时有 success/error_message；
- 原始长度和清洗后长度会被记录。

但对长期平台有三个明显问题。

### 6.1 每 URL 创建新的 AsyncWebCrawler

`async with AsyncWebCrawler()` 位于单页函数内部，这意味着高频任务可能反复创建/关闭 Browser 或相关运行时。对于 1000 站持续抓取，会增加 Browser 启动成本、内存抖动和故障概率。

生产系统应该让 Browser/Crawl4AI Worker 长期持有 crawler/browser/context pool，并由 task lease 驱动单 URL 或 bounded batch 执行。

### 6.2 `bypass_cache=True` 不适合作为默认增量策略

竞争情报 CLI 强制刷新可以理解，但知识库增量同步应该显式利用 ETag、Last-Modified 和平台 snapshot cache。第三方引擎 cache 只能是执行优化，不能决定业务 freshness。

### 6.3 清洗会破坏技术文章结构

`clean_content()` 使用 BeautifulSoup 后遍历大量 `h1/h2/p/div/span/li/td/...`，最终把文本拼成一段，再按标点拆句、过滤短句。

这种方法对“生成摘要上下文”尚可，对 Markdown 知识库不适合：

- 父 `div` 和子 `p/span` 都参与遍历时可能重复文本；
- 标题层级最终被空格归一化；
- `pre/code` 不在目标结构内，会丢代码块；
- table 只能剩文本，表格结构丢失；
- 链接目标、图片、列表层级丢失；
- 以 `.`、`!`、`?` 切句会破坏域名、版本号、代码和技术符号；
- 过滤短句可能误删 API 名称、命令、错误码等关键信息。

知识库正确方式是 DOM -> Article IR -> Markdown Renderer，并保留 raw HTML/raw Markdown 作为候选和回放依据。

## 7. URL 与路径安全

`utils/sanitize.py` 提供两类保护。

### 7.1 路径安全

公司名只保留字母数字、下划线、连字符，并对 resolve 后路径做目录边界检查。这比直接把用户输入拼文件名可靠得多。

生产系统建议使用 `Path.is_relative_to()` 或等价的路径组件校验，而不是纯字符串前缀比较，并统一由 object key/UUID 生成最终文件路径。

### 7.2 SSRF 防护

`validate_url()`：

- 自动补 https；
- 只允许 http/https；
- 解析 hostname；
- DNS 解析后阻止 loopback、RFC1918、link-local 和常见 IPv6 私网。

这是值得保留的防线，但当前实现仍有生产级缺口：

1. `socket.gethostbyname()` 只得到一个 IPv4 地址，无法覆盖所有 A/AAAA 结果。
2. DNS 解析失败时直接放行给后续抓取器，属于 fail-open；生产平台应记录明确 DNS error 并拒绝执行。
3. 校验发生在“请求之前”，真正连接时客户端会再次解析 DNS，存在 TOCTOU/DNS rebinding 窗口。
4. `requests.get()` 默认自动跟随 redirect，而入口 `validate_url()` 无法约束每个重定向目标。
5. Sitemap 中发现的每个文章 URL 也必须单独校验。

项目打包版 `fetch_sitemap()` 会重新校验嵌套 Sitemap URL，这是正确方向；但 `scrape_feature_pages()` 仍直接把 Sitemap 中解析出的页面 URL 交给 `HomepageScraper`，没有在这一层再次调用 `validate_url()`。恶意或被污染的 Sitemap 理论上可以列出内网地址。

更重要的是，生产系统不能只把 SSRF 防护理解为“Admission 阶段做一次 URL 校验”。Admission 是策略决策，Egress 才是强制执行点。HTTP/Browser/Crawl4AI 的真实网络连接都必须由统一出口执行：解析全部 A/AAAA、拒绝任一不允许地址、连接到已验证目标、禁止客户端自行越过 Gateway 自动跳转，并对每个 redirect 重新解析和授权。这样才能真正抵抗 DNS rebinding 与跳转型 SSRF。

## 8. 本地 JSON 状态模型的适用边界

项目把单公司数据保存在 `data/companies/{name}_data.json`，适合个人 CLI：部署简单、可被终端智能体直接读取。

但在百万文章和多 Worker 场景下会出现：

- 并发写冲突；
- 无 lease；
- 无 durable frontier；
- 无索引和高效查询；
- 单文件越来越大；
- 无细粒度版本和审计。

更具体地看，`save_homepage_data()` 与 `save_feature_data()` 都采用“读取整个 JSON -> 内存修改 -> 直接覆盖原文件”的 read-modify-write 模式，没有文件锁、临时文件 + 原子 rename、事务或 compare-and-swap。两个并发任务即使分别抓取不同 URL，也可能发生 lost update；进程在写入中途崩溃还可能留下截断 JSON。对长期知识库，持久化不能只保证“最终写了文件”，还要保证任务状态与对象引用之间的原子性。

项目还有一个很容易被忽略的身份冲突：`create_feature_name()` 只取 URL path 最后一段作为 key，并把 `-`、`.` 替换为 `_`。例如 `/blog/setup` 与 `/docs/setup` 会得到相同 key `setup`，后写结果可静默覆盖前写结果。技术知识库绝不能把标题、slug 或 URL path 尾段当作内部身份键；应以 `url_id`、`article_id`、规范化 URL hash 和 attempt/snapshot id 作为主键，slug 只用于展示。

此外 JSON 内容没有显式 schema version，Agent/CLI 升级后字段语义变化只能靠调用方猜测。生产 Adapter 输出应携带 `adapter_output_schema_version` 和 runtime/extraction release，保证旧 snapshot 可以用原 schema 解释，也能离线迁移或 replay。

因此知识库平台必须使用 PostgreSQL 管任务/版本/元数据，S3/MinIO 放 raw/DOM/Markdown 大对象，不能沿用单 JSON 文件作为状态真相源。对象写入建议采用“内容寻址临时对象 -> 校验 hash/size -> 数据库事务登记 artifact/snapshot -> 标记 COMPLETE”的两阶段提交语义；重复任务以 hash/idempotency key 复用已有对象，不原地覆盖已完成 snapshot。

## 9. 可直接吸收的设计

1. Crawl4AI 作为可选重型依赖，而不是整个系统的唯一抓取基础。
2. Sitemap 下载必须有 encoded/decoded streaming size limit。
3. Sitemap Index 需要循环检测和嵌套安全校验。
4. URL 页面类型分类可作为低成本的调度/抽取信号。
5. 抓取并发必须使用 semaphore 或等价 bounded concurrency。
6. CLI/Agent 只负责发起动作，采集执行层保持确定性接口。
7. 文件名和 URL 都需要专门的安全校验层。
8. 安装包运行代码必须只有一个 source of truth，并通过构建产物/contract test 验证实际导入路径。
9. 抓取 Adapter 应显式声明运行时类型（async/blocking/CPU/browser），避免阻塞工作混入 event loop。
10. 结果对象应携带稳定内部 ID 和 schema/release 信息，展示 slug 不能承担 identity。

## 10. 不应直接照搬的设计

1. 固定 `MAX_URLS=5000` 作为整个 Sitemap 的终止条件。
2. `_visited` 只存在单进程内存。
3. 为全部 URL 一次性创建 task 并 `gather`。
4. 每 URL 新建一个 AsyncWebCrawler 生命周期。
5. 默认 `bypass_cache=True` 作为长期增量策略。
6. 关键词过滤直接决定“抓或不抓”。
7. 把 cleaned HTML 扁平化为纯文本后再按标点切句。
8. 只在入口 URL 做 SSRF 校验，而不是由 Egress 在真实连接和 redirect 每一跳强制校验。
9. 用本地 JSON 文件承担长期任务和文章状态。
10. `stream=True` 后仍把完整 Sitemap 拼回内存并构造整棵 XML DOM。
11. 仅靠 `>=` 依赖下界重建生产抓取运行时，或让 HTTP Worker 与 Browser 重依赖共用同一镜像。
12. 同一功能保留两套可执行源码并允许实现漂移。
13. 在 async Worker 的事件循环线程里直接执行 `requests` 等同步网络 I/O。
14. 通过 URL 最后一段、标题或 slug 生成内部 map key，允许不同 URL 静默覆盖。
15. read-modify-write 整个 JSON 后原地覆盖，且不提供事务、原子 rename 或 schema version。

## 11. 对博客知识库技术方案的具体优化

基于该项目，技术方案应明确增加以下能力：

### 11.1 持久化 Sitemap Task Tree

每个 Sitemap/子 Sitemap 都变成数据库任务，保存父子关系、深度、etag/last-modified、body hash、checkpoint、lease 和错误状态。

### 11.2 Execution Slice 替代硬截断

`max_bytes/max_entries/max_children/max_runtime` 只控制单次执行资源，达到上限就保存 checkpoint 并续跑，不能把上限当作“已完成”。

### 11.3 页面类型只做评分

URL 规则输出 ARTICLE/ARCHIVE/CATEGORY/TAG/PAGINATION 等类别，并进入 Priority Scorer；FULL_BACKFILL 中不能硬过滤低分 URL。

### 11.4 三层有界并发

全局、站点、域名三层 semaphore/token bucket；百万 URL 使用 bounded queue/streaming，不用全量 gather。

### 11.5 长生命周期 Crawl4AI Worker

Worker 复用 event loop、Browser/context/page pool，单 URL 只是任务，不创建新的运行时生命周期。

### 11.6 结构化清洗

DOM -> Article IR -> Markdown，代码块、表格、列表、链接、图片和标题层级都作为结构节点保存；raw HTML/raw Markdown 永久可 replay。

### 11.7 Admission 与 Egress 分层

Admission 负责 URL 规范化、站点策略、robots、域名/path 等业务准入；Egress 在真实网络连接时强制执行 SSRF 策略。根 URL、子 Sitemap、Sitemap 内文章 URL和 redirect 每一跳都重新做 DNS/Egress/端口校验，DNS 失败必须 fail-closed。

### 11.8 可选依赖升级为运行时隔离

HTTP/Sitemap/Feed Worker 使用最小依赖镜像；Browser/Crawl4AI Worker 使用独立镜像和 lockfile。每种运行时分别固定 dependency lock、SBOM、image digest、Browser revision，并通过 Adapter contract test 后发布。

### 11.9 真正的 Sitemap 流式处理

网络读取、解压、XML 解析和 frontier 写入必须形成一条 streaming pipeline，不把整份 Sitemap 重新聚合到内存。解析到一个 `<url>` 或 `<sitemap>` 就即时做 Admission/持久化，slice 结束只保存可恢复 checkpoint。

### 11.10 构建产物唯一性检查

CI 必须验证实际安装 wheel/container 中只存在预期 package tree，测试导入路径与生产一致；禁止 legacy/root 副本成为第二套未受控实现。runtime bundle 记录最终 wheel/hash，而不仅是源码 commit。

### 11.11 Async Runtime Purity

HTTP/Sitemap/Feed Worker 默认只允许 async-native 网络客户端。Adapter 注册时声明 `runtime_class=ASYNC_NATIVE|BLOCKING_IO|CPU_BOUND|BROWSER`；调度器按类型路由。同步第三方 SDK 必须进入有界 Blocking-I/O pool 或独立 Worker，线程池队列长度、并发和超时都可观测，禁止直接阻塞主 event loop。CI 可通过 lint/contract test 阻止 `requests`、同步 urllib 等出现在 async-native Worker 的网络执行路径。

### 11.12 稳定身份、Schema 与原子 Artifact Commit

所有抓取结果使用 `url_id + attempt_id + snapshot_id` 标识，禁止使用标题、slug、URL path 尾段作为更新 key。Adapter 输出带 `adapter_output_schema_version`、`runtime_bundle_release`、`fetch_profile_release`。

大对象先写内容寻址临时 key 并校验 hash/size，再在 PostgreSQL 事务中写 `fetch_snapshot/artifact_object` 元数据和状态，成功后对象视为不可变 COMPLETE；Worker 崩溃后根据幂等 key 重试或复用对象。这样可避免本地 JSON read-modify-write 的 lost update、半写文件和 silent overwrite，同时保证 replay 能准确解释旧版本数据。

## 12. 结论

该项目不是为百万级博客归档设计的，但它在 Sitemap 安全保护、URL 分类、有界并发、Agent 与抓取工具职责分离、Crawl4AI optional dependency 等方面提供了可复用思路。

对博客知识库最重要的改进，是把项目中的“递归 + 硬上限 + 本地内存状态 + 单机 JSON”升级为“持久化 Sitemap 任务树 + 真流式解析 + execution slice 续跑 + durable frontier + 分布式状态机”；把“入口 URL 预检查”升级为“Admission 决策 + Egress 连接时强制执行”；把“optional extra”升级为“HTTP 与 Browser 运行时物理隔离”。本次进一步补强两条生产级约束：一是 async Worker 必须保持事件循环纯净，阻塞网络 I/O 只能进入受控的 blocking runtime；二是内部身份、结果 schema 与 artifact 写入必须稳定、版本化、原子化，不能用 slug key 和覆盖式 JSON 形成隐式数据丢失。这样既能防止异常站点拖垮系统，也不会因为安全预算导致历史文章被静默截断，并能降低重型抓取依赖、并发写入和长期回放的运行风险。
