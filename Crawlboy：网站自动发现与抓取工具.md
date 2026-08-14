# Crawlboy：网站自动发现与抓取工具

## 1. 调研对象与结论

- 项目：https://github.com/aksharahegde/crawlboy
- 调研基线：`main`，commit `dab1da6d157b507ac64e5b64d942b49ffed67bb6`
- 核心代码：`crawlboy/crawler.py`、`crawlboy/meta_extract.py`
- 安全文档：`docs/security/threat-model.md`、`docs/security/controls.md`
- 测试：`tests/security/test_security_controls.py`、`tests/test_meta_frontmatter.py`
- 依赖：`pyproject.toml`

Crawlboy 是一个“以 Sitemap 为入口、用 Crawl4AI/Playwright 顺序抓取、把每页导出为 Markdown”的单机 CLI。它不适合作为 1000 个博客长期同步平台的主体，但它对 Sitemap 递归、安全边界、媒体落盘、路径生成、元数据 Front Matter 等实现非常有参考价值；同时源码也暴露出从单机导出器扩展到生产知识库时必须补齐的关键能力。

本次结论：现有博客知识库总体架构方向正确，不应直接采用 Crawlboy 作为核心调度器；应吸收其安全默认和输出机制，同时进一步强化 robots 执行语义、Sitemap 候选健康度、Browser 快照可重放性、HTML `<base href>` 处理和 AST/DOM 级媒体重写。

## 2. 实际执行链路

`crawlboy/crawler.py::run()` 的主链路如下：

1. 接收 `--site-url` 或 `--sitemap-url`。
2. `--site-url` 模式先拉取 `robots.txt` 中的 `Sitemap:`，没有时探测常见路径。
3. `collect_urls_from_sitemap()` 递归展开 sitemap index。
4. URL 去重；site 模式默认只保留与站点 origin 同 host 的 URL。
5. 创建一个 `AsyncWebCrawler`，对 URL 逐个执行 `await crawler.arun()`。
6. 从 Crawl4AI 结果读取 Markdown、HTML、media。
7. 可选下载图片，并重写 Markdown/HTML 图片地址。
8. 可选从 HTML 中抽取 meta/canonical/title，生成 YAML Front Matter。
9. Markdown/HTML 写本地目录，失败追加到 `errors.jsonl`。

虽然使用 async API，页面主循环本质仍是单 worker 顺序执行；URL frontier、路径冲突表、媒体缓存、统计全部位于进程内存，进程退出后没有可恢复状态。

## 3. Sitemap 发现机制

### 3.1 入口发现

`discover_sitemap_entry_urls()`：

- 先规范化站点 origin；
- 请求 `/robots.txt`；
- 提取全部 `Sitemap:` 行；
- 若存在任何 robots Sitemap，则直接返回这些入口；
- 如果完全没有发现，才依次尝试 `/sitemap.xml`、`/sitemap_index.xml`、`/sitemap-index.xml`、`/wp-sitemap.xml`；
- 候选响应通过 XML 根节点 `urlset/sitemapindex` 判断是否像 Sitemap。

这个实现简单有效，但存在一个生产级完整性问题：**robots 中只要出现一个 Sitemap，就不再探测其它常见入口**。若 robots 声明过时、部分损坏或只覆盖某一内容分区，可能漏掉仍有效的 Sitemap。

生产系统应把 Sitemap 入口当“候选集合”而不是“命中即停止”：

- robots、常见路径、站点配置、历史成功入口都可以产生候选；
- 每个候选独立保存 `source/evidence/etag/last_modified/last_success/status/error/coverage_hint`；
- 周期性低成本验证候选健康度；
- 多个有效 root sitemap 取并集并去重；
- 某个入口失败不能导致其它入口停止工作。

### 3.2 robots 的语义缺口

Crawlboy 使用 robots.txt 的目的主要是**发现 Sitemap**，并没有在页面抓取主循环里按目标 user-agent 执行 `Allow/Disallow` 规则。

这点非常重要：

> “从 robots.txt 找到 Sitemap”不等于“遵守 robots.txt 的抓取权限”。

生产系统必须把两者拆开：

1. `RobotsDiscovery`：解析 Sitemap 声明；
2. `RobotsPolicy`：在 Fetch/DeepCrawl 前按 user-agent、目标 path 和当前 robots 版本做准入判断。

robots 规则需要 TTL + Validator 缓存，失败要区分 404、网络错误、解析错误，并明确 fail-open/fail-closed 策略；策略变化还要可审计。

## 4. Sitemap 递归实现与内存模型

`collect_urls_from_sitemap()` 使用 `defusedxml.ElementTree.fromstring()`：

- `sitemapindex`：逐个递归子 Sitemap，再 `out.extend()`；
- `urlset`：提取 `<loc>` 到 Python list；
- 有最大深度、URL 数、单 Sitemap 解码后字节数限制。

优点：

- 使用 `defusedxml`，避免常见 XML entity 攻击；
- 显式限制递归深度、URL 数和 payload 大小；
- 子 Sitemap 的 `<loc>` 使用 `urljoin()` 解析相对地址。

缺点：

1. HTTP body 先完整进入内存；
2. gzip 使用 `gzip.decompress()` 一次性解压，再检查 decoded 大小；
3. XML 使用 `fromstring()` 一次性建树；
4. 每个子调用返回完整 list，父级不断 extend；
5. 没有 visited-set/cycle detection，只靠 max depth 兜底；
6. 没有 child sitemap checkpoint；
7. `max_urls` 在递归函数局部检查，无法形成统一 provider-run 原子预算；
8. 解析 `<url>` 时只使用 `<loc>`，没有利用 `<lastmod>`。

因此大站或恶意压缩 Sitemap 仍可能在“检查之前”制造内存峰值。

生产实现应采用：

```text
HTTP streaming
 -> compressed-byte limiter
 -> bounded streaming decompressor
 -> decoded-byte + compression-ratio limiter
 -> XMLPullParser/iterparse
 -> 小批量 URL/lastmod UPSERT PostgreSQL
 -> provider checkpoint
```

预算至少包含：compressed bytes、decoded bytes、压缩比、XML element 数、URL 数、child sitemap 数、请求次数、递归深度、wall time。预算耗尽应记录为 `partial/budget_exhausted`，而不是笼统失败。

## 5. 网络安全与 SSRF

`validate_network_target()` 默认：

- 只允许 HTTP/HTTPS；
- 必须有 hostname；
- 检查直接 IP 和 DNS 解析结果；
- 拒绝 private、loopback、link-local、multicast、reserved、unspecified；
- 提供 `--allow-unsafe-network-targets` 显式覆盖。

这是值得继承的安全默认。

但源码仍有几个生产缺口。

### 5.1 redirect hop 没有重新校验

Sitemap 和图片下载均使用 `follow_redirects=True`。初始 URL 校验通过后，30x 可以指向另一个 host，甚至私网地址；代码没有在每个 redirect hop 上重新执行相同安全策略。

生产系统必须禁用自动 redirect，逐 hop 执行：URL 解析、userinfo/port 检查、DNS、allowed-host、egress policy、redirect 次数预算。

### 5.2 DNS rebinding / TOCTOU

代码先 `socket.getaddrinfo()`，随后 httpx/Browser 自己再次解析并连接。校验与连接之间存在时间窗口。

因此应用层校验只能是第一道防线；生产部署还要用统一 egress proxy、网络命名空间或防火墙，让“实际连接”也不能访问私网和 metadata endpoint。

### 5.3 Browser 子资源

主页面 URL 在 `crawler.arun()` 前被校验，但 Playwright 加载页面后可能继续发起 iframe、XHR/fetch、图片、字体等请求。这些请求不一定经过 Python 的 `validate_network_target()`。

Browser worker 必须有 public-only 网络出口和 request budget，不能只依靠 page callback。

### 5.4 URL userinfo

`normalize_site_url()` 会把 `https://user:pass@host/...` 的 userinfo 静默剥离用于站点 origin；但直接从 Sitemap 得到的页面 URL 并没有统一禁止 userinfo。

生产系统不应“悄悄修正”此类输入，应在 URL intake/EgressPolicy 中显式拒绝 userinfo，避免凭据传播、语义混淆和日志泄露。

## 6. 顺序抓取、限速和公平性

主循环：

```python
for url in urls:
    result = await crawler.arun(...)
```

所以单实例是串行抓取。`--delay` 只在**成功抓取之后**执行；被安全策略拒绝、异常、`result.success == False` 都直接进入下一 URL。

生产系统的 politeness 不能依赖“成功后 sleep”，而要在每次请求尝试之前取得 domain lease/token：

- HTTP、Browser、Sitemap、Feed、媒体共享域总限流；
- 失败、timeout、404、redirect 同样消耗请求次数；
- 429/503/Retry-After 动态降速；
- 每域 concurrency + token bucket；
- 单个大站不能占满全部 worker；
- PG/S3/Extract lag 增长时触发 backpressure。

## 7. HTTP 快路径与 Browser 成本

Crawlboy 对页面统一使用 Crawl4AI/Browser，并显式 `CacheMode.BYPASS`。这适合简单 CLI 的“每次都抓最新页面”，但不适合 1000 站点长期运行：

- 每个页面启动/驱动浏览器网络栈成本高；
- 没有 If-None-Match / If-Modified-Since 的页面级增量快路径；
- 没有 304；
- 同正文没变也会重新执行 Browser 和 Markdown 生成。

生产系统应：

1. HTTP 优先获取状态、validator、content-type 和 body；
2. 304 直接结束；
3. 200 且 body hash 未变不创建文章版本；
4. 只有 CSR shell、JS 正文、质量失败时升级 Browser；
5. Browser 也受请求数、字节、wall time、内存、子资源预算。

## 8. Browser 快照必须可重放

Crawlboy 在 Browser 成功后直接使用 `result.html` 和 `result.markdown` 写文件，但对生产知识库而言，“raw snapshot”不能只抽象成一个模糊的 HTML 文件。

对于 Browser 页面，至少应区分：

- `raw_http_body`：服务端原始响应；
- `rendered_dom`：等待条件完成后的 DOM snapshot；
- `capture_profile`：等待 selector、滚动/点击脚本、viewport、locale、timezone、JS 开关等配置；
- `browser_engine/browser_version/crawl4ai_version`；
- `captured_at` 和 final URL。

原因：JS 页面内容可能取决于运行时、时间、外部 API 和浏览器版本。若只保存服务端 HTML，后续离线 re-extract 无法重现当时正文；若只保存 Markdown，又无法升级抽取器。

推荐数据模型增加：

```text
fetch_snapshots
- object_key_http_raw
- object_key_rendered_dom
- capture_kind(http/browser/archive)
- capture_profile_version
- browser_engine_version
- fetcher_version
```

Extract 读取“标准化 capture artifact”，而不是隐式读取某个 Browser 对象。

## 9. URL 身份与本地路径

Crawlboy 的 `url_to_relative_md_path()`：

- 只根据 URL path 生成文件目录；
- segment 只保留字母、数字、`_`、`-`；
- 每段最大 120；
- `/` 映射 `index.md`。

`relative_md_path_with_collision()` 用进程内字典检测碰撞；后来者追加 URL SHA-256 的短 hash。

问题是碰撞结果依赖 discovery 顺序：A、B 两个 URL 映射到同一路径，谁先出现谁获得短路径。下次顺序变化后导出路径可能互换。

生产系统要把：

- fetch URL identity；
- accepted article identity；
- export path

完全分离。`article_key` 由 `site_id + stable identity` 计算，导出 path 必须是 article identity 的纯函数；碰撞双方都使用稳定 hash 后缀，不依赖本次抓取顺序。

## 10. 媒体发现和存储

Crawlboy 图片来源有两部分：

1. Crawl4AI `result.media.images`；
2. 对 HTML `<img>/<source>` 用正则再扫 `src/srcset`。

下载时：

- 有单文件最大字节；
- 有本次运行总媒体字节；
- 进程内 URL cache 去重；
- 文件名使用规范化图片 URL 的 SHA-256 短 hash；
- 根据 Content-Type 或 URL 后缀选择扩展名。

README 称其为 content-addressed，但当前实现实际是 **URL-addressed**：相同二进制经两个不同 CDN URL 会落两份文件。

生产系统应先流式下载并计算完整内容 hash，再按 `content_sha256` 存 blob；source URL 只作为缓存/来源记录。

此外仅限制 bytes 不够：攻击页面可生成大量 404、timeout、204 或极小文件 URL，使请求数爆炸而字节预算几乎不变。必须增加：

- `max_assets_per_article`；
- `max_asset_requests_per_article`；
- `max_asset_requests_per_job`；
- redirect/request wall-time；
- 每域 asset concurrency；
- 失败请求也扣 request budget。

MIME 不能只信 header/扩展名；应进行 magic sniff。SVG/HTML 等 active content 应 quarantine 或从隔离 origin 提供。

## 11. `<base href>` 与相对资源解析

Crawlboy 在资源重写时用 `base_for_assets = redirected_url/result.url/original_url`，再 `urljoin(base_for_assets, relative)`。

但 HTML 规范允许 `<base href>` 改变相对 URL 的解析基准。当前额外的正则 HTML 扫描并没有显式解析 `<base href>`。在部分文档站、镜像站或反向代理环境中，这会导致图片和链接解析错误。

生产抽取层应生成 `document_base_url`：

1. 默认 final URL；
2. 若存在 `<base href>`，先按 final URL resolve；
3. 再通过 EgressPolicy/allowed-host 验证；
4. 危险或异常 base 忽略并记录质量事件；
5. 所有相对链接、图片、srcset、canonical 的 resolve 都复用同一版本化解析器。

## 12. 不应使用正则直接重写 Markdown/HTML

`rewrite_markdown_images()` 用正则匹配 `![alt](...)`；HTML 也用正则匹配 `<img>/<source>` 属性。

这对常见简单页面可用，但不是完整语法解析器：

- Markdown URL 可包含转义、括号、title；
- reference-style image 不匹配；
- 重建语法时可能丢失 image title；
- HTML 属性和实体存在大量合法边界情况；
- `srcset` 也有专门语法。

生产系统应避免“先生成 Markdown，再用正则修链接”。更稳妥的链路是：

```text
HTML/rendered DOM
 -> DOM/Article IR
 -> 解析并标准化链接/媒体引用
 -> Asset relation resolution
 -> Markdown AST / serializer
 -> 最终 Markdown
```

即媒体重写发生在 DOM/IR/AST 层，最终 Markdown 由版本化 serializer 一次生成。这样规则升级后可以离线重放，也能保证链接规范化和输出确定性。

## 13. Metadata / Front Matter

`meta_extract.py` 使用 `HTMLParser` 抽取：

- `<title>`；
- `<link rel=canonical>`；
- `meta[name]`；
- `meta[property]`；
- `meta[http-equiv]`。

单字符串值约 8 KiB，上层 YAML 约 64 KiB；过大时逐级截断，最终只保留 source_url；使用 `yaml.safe_dump()`。

值得保留的是“metadata 也要有预算”和“确定性序列化”。但生产系统要进一步：

- 限制 field count、key length、head 扫描量；
- 标题/作者/日期/canonical 优先结构化入库；
- 保存字段 provenance；
- canonical 视为不可信输入并验证 allowed host；
- JSON-LD/OpenGraph/meta/selector/heuristic 冲突时记录候选和选择依据；
- Front Matter 只是导出视图，不是真相源。

## 14. 错误模型与日志

Crawlboy 将失败追加到 `errors.jsonl`。显式 URL 会通过 `redact_url_for_logs()` 去掉 userinfo、query、fragment，这是正确做法。

但异常文本仍可能直接来自 `repr(exc)` 或第三方 `result.error_message`。第三方异常可能包含完整 URL、header 或 token，因此仅清洗 URL 字段不够。

生产系统需要中央 `LogSanitizer`，覆盖：

- structured log；
- exception message/stack；
- dead-letter；
- OTel span attribute；
- Web 错误页；
- audit/security event。

另外 append-only `errors.jsonl` 不区分“历史失败”和“当前仍失败”。生产系统应以 `job_item + attempt + error_event` 建模，Web 默认展示当前状态，同时允许追溯历史尝试。

## 15. 删除、缺失与增量同步

Crawlboy 每次都从 Sitemap 列表开始抓，没有长期状态，因此没有：

- Feed cursor；
- Sitemap Validator；
- 页面 Validator；
- disappearance reconciliation；
- 删除确认；
- 内容版本；
- rule replay。

生产系统应把“发现”本身做增量：

- Feed：guid/url/pubdate cursor；
- root/child Sitemap：ETag、Last-Modified、lastmod hint；
- page：ETag/Last-Modified/body hash；
- 周期 reconciliation 判断长期未见 URL；
- 404/410 多次确认后才 source_deleted；
- raw/rendered snapshot 保留后，抽取规则变化只 re-extract，不 refetch。

## 16. 安全工程方法值得继承

项目额外提供 threat model、security control 文档和安全测试。`tests/security/test_security_controls.py` 覆盖了：

- URL 日志脱敏；
- output path escape；
- private IP；
- DNS 解析到私网；
- Sitemap URL/depth 限额；
- 媒体文件大小；
- Hypothesis 路径性质测试。

这说明安全要求应落成自动化回归，而不是只写在架构文档里。

生产系统的 Control Matrix 应继续扩展：

```text
Requirement
 -> automated test
 -> runtime metric/alert
 -> runbook/owner
```

还需覆盖：redirect SSRF、DNS rebinding、IPv4-mapped IPv6、Browser 子资源 SSRF、gzip bomb、Sitemap cycle、request amplification、symlink race、active content preview、异常文本 secret 泄露。

## 17. 供应链

`pyproject.toml` 采用 `crawl4ai>=0.8.6`、`httpx>=0.27.0` 等下限约束；dev 依赖包含 pip-audit、Bandit、Hypothesis 等。

对生产平台而言仅使用下限约束会使环境随时间漂移，Browser/Chromium 与 Crawl4AI 的组合尤其需要可重复。

建议：

- lockfile + image digest；
- SBOM；
- OSV/pip-audit；
- Chromium/Playwright/Crawl4AI 版本一起追踪；
- 漏洞 waiver 有 owner 和过期时间；
- 升级先跑 golden crawl + canary。

## 18. 对最终技术方案的优化项

基于源码复核，最终方案应明确包含以下点：

1. Crawlboy 只能作为参考实现/诊断工具，不作为生产调度核心。
2. robots 的“发现 Sitemap”和“抓取准入”必须分离，Fetch/DeepCrawl 真正执行 user-agent 规则。
3. Sitemap 入口采用多来源候选并集和独立健康状态，不能“robots 有一个入口就停止其它发现”。
4. Sitemap 端到端 streaming，加入 compressed/decoded/ratio/request/element 多维预算、cycle detection、child checkpoint。
5. 所有请求逐 hop Egress 校验，并由网络层阻断私网；Browser 子资源同样受控。
6. Domain limiter 对失败请求同样计费；预算同时按 bytes、request count、wall-time 控制。
7. 页面增量 HTTP 优先，Browser 只作升级路径。
8. Browser 抓取同时保存 raw HTTP 与 rendered DOM，并记录 capture profile/browser 版本，以支持可重放 re-extract。
9. `<base href>` 作为不可信输入验证后生成统一 `document_base_url`。
10. 媒体重写在 DOM/IR/Markdown AST 层完成，不使用正则作为生产级最终重写器。
11. Asset 以真实 `content_sha256` 做 content-addressed blob，URL 仅作为 source/cache key。
12. Markdown/export path 由稳定 article identity 决定，不受 crawl 顺序影响。
13. Metadata 结构化保存 provenance；Front Matter 只是版本化导出视图。
14. 错误采用 job item/attempt/event 状态模型，并通过中央 LogSanitizer 防泄露。
15. 安全要求必须映射到 automated test、metric/alert 和 runbook。

这些结论已用于完善《博客知识库技术方案.md》。