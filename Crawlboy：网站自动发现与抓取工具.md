# Crawlboy：网站自动发现与抓取工具

## 1. 调研对象

- 项目：https://github.com/aksharahegde/crawlboy
- 调研基线：`main`，tree SHA `dab1da6d157b507ac64e5b64d942b49ffed67bb6`
- 核心文件：`crawlboy/crawler.py`、`crawlboy/meta_extract.py`、`docs/security/*`、`tests/security/test_security_controls.py`、`.github/workflows/security.yml`、`pyproject.toml`
- 项目定位：从站点或 Sitemap 入口发现 URL，使用 Crawl4AI/Playwright 顺序抓取页面，将 Markdown、可选 HTML 和媒体写入本地目录。

Crawlboy 适合分析“单机、安全默认、Sitemap 驱动的 Markdown 导出器”如何实现；它不是面向 1000 站点长期运行的分布式知识库平台，因此本调研重点是提取可复用的安全、发现、路径、媒体和元数据机制，并分析其扩展到生产系统时的缺口。

## 2. 总体执行链路

核心链路位于 `crawlboy/crawler.py::run`：

1. 解析 `--site-url` 或 `--sitemap-url`。
2. `--site-url` 模式先尝试 robots.txt 中的 `Sitemap:`，再按常见路径探测 Sitemap。
3. 递归读取 Sitemap index，展开为页面 URL 列表。
4. URL 去重，并在 site 模式下默认过滤到同 host。
5. 启动一个 `AsyncWebCrawler`，对 URL 列表逐个 `crawler.arun()`。
6. 从 Crawl4AI 结果取 Markdown、HTML、媒体信息。
7. 可选下载图片并改写 Markdown/HTML 中的媒体地址。
8. 可选从 HTML head 抽取 metadata，生成 YAML Front Matter。
9. Markdown/HTML 写入本地目录，错误写入 `errors.jsonl`。

其结构清晰，但整个 URL Frontier、碰撞映射、媒体 cache 和执行统计都保存在单进程内存中，页面抓取主循环也是顺序执行，因此不能直接承担多站点大规模调度、断点续跑和增量同步。

## 3. Sitemap 自动发现与递归展开

### 3.1 入口发现

`discover_sitemap_entry_urls()` 将站点 URL 规范为 origin，然后：

- 请求 `/robots.txt`，解析所有 `Sitemap:` 行；
- 若 robots.txt 没给出 Sitemap，则按 `/sitemap.xml`、`/sitemap_index.xml`、`/sitemap-index.xml`、`/wp-sitemap.xml` 顺序探测；
- `url_looks_like_sitemap()` 通过 XML 根节点是否为 `urlset` / `sitemapindex` 判断候选。

这个实现说明生产系统应该把“入口发现”单独抽象为 Discovery Provider，而不是和页面抓取绑死。robots.txt 中可以同时声明多个 Sitemap，因此 Provider 需要支持一个站点多个 root sitemap，并为每个入口单独记录状态。

### 3.2 递归 Sitemap

`collect_urls_from_sitemap()` 使用 `defusedxml.ElementTree`，根据根节点类型处理：

- `sitemapindex`：递归请求子 Sitemap，并把子结果 `extend` 到父列表；
- `urlset`：提取每个 `<loc>`；
- 达到最大深度或 URL 数时抛错。

优点是有显式深度、URL 数和 payload 体积上限，并使用 `urljoin()` 支持相对 Sitemap 地址。

生产缺口：

1. 每个递归调用返回完整 `list[str]`，父节点再合并，深层或大站点会产生高峰内存。
2. XML 通过 `ET.fromstring()` 一次性解析，无法做到真正流式。
3. URL 上限在子树中分别检查，父层只有子树返回后才能发现全局超额，预算不能及时停止工作。
4. 没有显式 visited-set / cycle detection，恶意 Sitemap index 可以形成循环，最终只依赖 max depth 终止。
5. 没有子 Sitemap 级 checkpoint；进程中断后只能重新展开。

生产系统应采用：HTTP stream -> bounded gzip/deflate stream -> `iterparse/XMLPullParser` -> URL 小批量 UPSERT PostgreSQL。Sitemap root、child sitemap、游标、ETag/Last-Modified、last success、partial reason 都持久化，预算必须按 provider run 全局原子扣减而不是只靠递归函数局部计数。

## 4. 压缩与资源边界

`_decode_sitemap_body()` 发现 `.gz` 或 gzip magic header 时直接 `gzip.decompress(content)`；之后 `fetch_sitemap_bytes()` 再检查解压后的长度。

这里有一个重要的实现差异：代码虽然设置了“解码后最大字节数”，但 `httpx` 已经先把响应完整读入 `response.content`，`gzip.decompress()` 也会先在内存中完成解压，随后才检查长度。因此超大压缩体或压缩炸弹仍可能在检查前制造内存峰值。

生产方案必须同时限定：

- compressed bytes；
- decoded bytes；
- compression ratio；
- XML element/URL 数；
- 单 Sitemap、单 provider run、单 job、单站点/日多个层级的累计预算。

响应体不能先无界进入 RAM。预算耗尽应是 `partial/budget_exhausted`，保留 checkpoint 后可继续，而不能只当作通用网络失败。

## 5. SSRF 与出站网络安全

### 5.1 Crawlboy 的做法

`validate_network_target()`：

- 只允许 HTTP/HTTPS；
- 要求 hostname；
- 直接 IP 与 DNS 解析结果都检查 private、loopback、link-local、multicast、reserved、unspecified；
- 默认拒绝，只有 `--allow-unsafe-network-targets` 才放开。

这套“默认拒绝 + 显式例外”适合 Web 管理型爬虫，因为站点配置、Sitemap 中的 URL、页面媒体 URL 都属于攻击者可控输入。

### 5.2 关键缺口

Sitemap 和媒体请求使用 `follow_redirects=True`。安全检查发生在初始 URL 请求前，并没有对每一个 redirect hop 重新做完整校验。因此公开 URL 可 30x 到内网地址。

此外，应用先 `getaddrinfo()` 校验，然后 HTTP 客户端稍后自行解析和连接，存在 DNS rebinding / TOCTOU 窗口。即使应用层逻辑正确，也不应把它作为唯一防线。

Browser 路径风险更大：顶层 URL 在调用 Crawl4AI 前检查，但 Playwright 页面加载过程中产生的 redirect、iframe、XHR、fetch、图片等子请求不一定经过这个 Python 函数。

生产设计要求：

1. HTTP 层禁用自动 redirect，逐 hop 校验 URL、host、DNS 和 allowed-host policy。
2. DNS 解析与真实连接尽可能由统一 egress proxy 完成，或在网络命名空间/防火墙中硬阻断私网、metadata endpoint。
3. Browser worker 强制 public-only egress，不能只依赖 page-level callback。
4. Sitemap、Feed、archive、页面媒体、canonical 候选、浏览器子资源统一走同一 Egress Policy。
5. 跨 host redirect 默认拒绝，确需放开时由站点规则显式声明并审计。

## 6. 顺序执行、限速与请求预算

Crawlboy 虽然使用异步库，但页面主循环是 `for ... await crawler.arun()`，因此是单实例顺序抓取。`--delay` 只在“成功抓取后”执行；失败、被拒绝或异常路径不会 sleep。

对生产系统的启示：politeness 不能实现为“成功后 sleep”。它必须是域级的请求准入：

- 每一次网络尝试，无论 2xx、4xx、5xx、timeout、redirect 都先消耗 token；
- Redis/Lua 维护 domain concurrency lease + token bucket / next_allowed_at；
- 遇到 429/503/Retry-After 动态退避；
- HTTP、Browser、媒体、Sitemap 共享同域总预算，避免多个 worker 分别“都很克制”，合起来却把源站打爆。

同时需要 backpressure：PG/S3/Extract lag 高时降低 Fetch；Browser 内存高时降 slot；Redis backlog 高时 Scheduler 降速。

## 7. URL 身份、去重与导出路径

### 7.1 Crawlboy 的路径算法

`url_to_relative_md_path()` 把 URL path 拆段后做字符清洗，长度上限 120，根路径映射为 `index.md`。

`relative_md_path_with_collision()` 使用一个进程内 `used` 字典检测冲突；冲突时给后来出现的 URL 追加 SHA-256 短 hash。

`safe_out_path()` 对最终路径 `resolve()`，要求仍位于 `out_dir` 内，能阻挡明显路径穿越和已有 symlink 指向目录外的情况。

### 7.2 生产问题

碰撞分配依赖“本次 URL 出现顺序”：两个不同 URL 规范化到同一路径时，谁先出现谁获得短文件名。换一次 discovery 顺序，导出路径可能变化。这不适合长期知识库 identity。

生产系统应把身份拆成三层：

- `url_identity`：规范化后的抓取 URL；
- `article_key`：`site_id + accepted canonical/identity rule` 的稳定 hash；
- `version_id`：文章版本。

S3 key 和本地导出路径必须是 article identity 的纯函数，发生冲突时双方都按稳定规则带 hash，而不是依赖 crawl 顺序。导出本地目录还需要防 Windows 保留名、超长路径、symlink race，并采用临时文件 + atomic rename；高保证环境可使用 `openat/O_NOFOLLOW`。

## 8. 媒体下载：字节预算之外还要限制请求数量

### 8.1 现有实现

Crawlboy 从 Crawl4AI `media.images` 和 HTML `<img>/<source>` 中收集 URL，逐个请求图片。它设置：

- 单文件最大字节；
- 本次运行媒体总字节；
- 根据 URL hash 生成文件名；
- 进程内 URL cache 避免同 URL 重复下载；
- 下载成功后重写 Markdown/HTML 的 `src/srcset`。

README 称媒体“content-addressed, deduped”，但实现中最终文件名是规范化媒体 URL 的 SHA-256，而不是下载内容的 hash。同一图片经不同 CDN URL 暴露时仍会保存多份，因此严格来说是 URL-addressed cache，不是真正 content-addressed storage。

### 8.2 一个额外的生产级风险

当前上限主要按 bytes 计数，但攻击页面可以生成大量不存在、超时、返回 204 或极小 body 的媒体 URL。即使总字节很小，也可能制造大量 DNS/TCP/TLS/HTTP 请求。`ensure_image_downloaded()` 对失败请求返回 0 bytes，字节预算几乎不下降。

因此生产方案除 `max_asset_file_bytes` / `max_asset_bytes_per_job` 外，还必须增加：

- `max_assets_per_article`；
- `max_asset_requests_per_article`；
- `max_asset_requests_per_job`；
- 单资源 redirect 上限和 wall-time；
- 每域媒体并发；
- 失败请求同样扣“请求次数预算”。

最终存储采用两阶段：先按 URL + validator 做下载缓存，流式下载并算完整 `content_sha256`，然后用 hash 作为 `asset_blob` identity，`article_assets` 记录 source URL、hash、MIME、尺寸、关系，实现跨 URL、跨文章去重。

还要做 MIME + magic sniff。SVG/HTML 等主动内容不能直接在管理 Web 同源预览，应 quarantine 或从独立无 Cookie origin 提供。

## 9. HTML metadata 与 Front Matter

`meta_extract.py` 使用标准库 `HTMLParser` 提取：

- `<title>`；
- `<link rel=canonical>`；
- `meta[name]`；
- `meta[property]`；
- `meta[http-equiv]`。

它对单个字符串值设约 8 KiB 上限，对整个 YAML Front Matter 设约 64 KiB 上限；如果总量仍过大，会逐级缩短字符串，最终只保留 `source_url`。`yaml.safe_dump()` 也避免了危险对象序列化。

这说明 metadata 同样需要预算和确定性。但生产环境还要补三层：

1. **采集期预算**：限制 field count、key length、总 HTML head 扫描量，不能等构建 YAML 时才截断。
2. **provenance**：同一字段可来自 JSON-LD、OpenGraph、meta、selector、heuristic，DB 中保存来源与规则版本。
3. **canonical 不可信**：相对 canonical 需 resolve；跨域 canonical 需 allowed-host/alias policy；危险地址不能直接变成 article identity。

Front Matter 应是导出视图，不是业务真相源。标题、作者、日期、canonical 等先结构化入库，再由版本化 serializer 生成 Markdown Front Matter。

## 10. 日志与异常泄露

`redact_url_for_logs()` 会去除 userinfo、query 和 fragment。显式记录 URL 时这种做法正确。

但 Crawlboy 在异常路径中仍会把 `repr(exc)` / `result.error_message` 作为 error string 写日志与 `errors.jsonl`。第三方库异常消息可能包含完整 URL、重定向地址、header 或 token，因此“显式 URL 字段脱敏”并不等于全链路不泄露。

生产系统需要中央 `LogSanitizer`：日志字段、异常文本、stack trace、dead-letter、OpenTelemetry attributes、Web 错误页都在 sink 前统一清洗；Authorization/Cookie 等 header 采用 allowlist，原始 URL 只有受限权限才能查看。

## 11. 安全控制矩阵与测试

项目的 `docs/security/controls.md` 把网络目标、Sitemap 限额、媒体限额、URL 脱敏、输出路径、依赖审计、Bandit 等映射到测试和 CI gate，这是非常值得直接继承的工程习惯。

`tests/security/test_security_controls.py` 还使用 Hypothesis 检查路径映射，说明安全边界应通过自动化性质测试验证，而不只靠文档。

不过当前测试仍有明显覆盖空白：

- redirect 到 private IP；
- IPv6 / IPv4-mapped IPv6；
- DNS rebinding；
- gzip bomb / 高压缩比；
- Sitemap cycle；
- Browser 子资源 SSRF；
- 媒体请求数量放大；
- 异常文本 token 泄露；
- symlink race；
- active content Web preview。

生产系统应把这些全部列为必须通过的安全回归用例，并把每个安全 requirement ID 映射到测试、指标、告警和 runbook。

## 12. 供应链与依赖升级

`pyproject.toml` 对 Crawl4AI/httpx 等使用下限约束；security workflow 会运行 `pip-audit` 和 Bandit，并为了已知漏洞在安装后强制升级 `lxml`、从 Git 安装新版 `nltk`。

这体现了积极的漏洞响应，但“安装后覆盖传递依赖版本”也有兼容性风险：最终运行组合可能超出上游声明的测试矩阵。

生产方案应采用：

- lockfile / digest 固定依赖；
- SBOM；
- OSV/pip-audit；
- 漏洞例外单（owner、CVE、理由、到期时间）；
- 兼容性回归、golden crawl、canary 后再升级 Browser/Crawl4AI；
- 基础镜像与 Chromium 版本一起追踪，不仅扫描 Python 包。

安全修复不能只通过“临时强制升级”完成后长期遗忘。

## 13. Crawlboy 不具备而生产系统必须具备的能力

1. **持久 Frontier**：Crawlboy URL 队列是内存 list；生产使用 PostgreSQL UPSERT + lease。
2. **断点续跑**：子 Sitemap、Provider、article fetch 都要 checkpoint。
3. **增量同步**：RSS cursor、Sitemap lastmod、ETag/Last-Modified、内容 hash、周期 reconciliation。
4. **水平扩展**：Redis Streams Consumer Group + DB lease + idempotency，而不是单进程顺序循环。
5. **HTTP 快路径**：普通博客页面优先 HTTP；Browser 仅在 CSR/交互依赖时升级。
6. **多站点公平调度**：站点优先级、域级限速、配额、backpressure。
7. **规则版本与回放**：保留 raw snapshot，抽取规则变化后离线 re-extract。
8. **Web 管理**：站点接入、provider coverage、任务、错误、安全事件、预算、规则 diff、文章版本。
9. **对象存储**：Markdown/raw/media 放 S3/MinIO；本地文件树只作为导出功能。
10. **质量治理**：模板页、登录页、软 404、重复正文、日期异常、canonical 异常自动检测。

## 14. 对博客知识库技术方案的最终优化结论

现有总体方向无需被 Crawlboy 替换。Crawlboy 的价值是验证若干“边界实现”并暴露单机方案在生产化后的缺口。本轮方案应明确加入或强化：

1. Sitemap 端到端流式处理、cycle detection、子 Sitemap checkpoint、run 级全局预算。
2. HTTP 手动 redirect 校验 + 网络层 public-only egress；Browser 子资源也必须被网络层约束。
3. 资源预算不仅按 bytes，还必须按**请求次数**计数，尤其是媒体；失败/空响应同样扣预算。
4. Domain limiter 对所有请求尝试生效，不能只在成功后 sleep。
5. Asset 真正采用 `content_sha256` content-addressed blob；URL hash 只做下载缓存键。
6. metadata 在采集阶段限制字段数/key/value/总量，并保留 provenance；canonical 作为不可信输入验证。
7. 导出路径由 article identity 稳定计算，禁止 crawl 顺序影响文件名。
8. 中央异常/日志 sanitizer，覆盖第三方异常、trace、dead-letter、Web 错误展示。
9. Security Control Matrix 扩展为 requirement -> automated test -> metric/alert -> runbook；补 redirect、gzip bomb、DNS rebinding、Browser SSRF、request amplification 等回归。
10. 依赖漏洞处理引入 lockfile、SBOM、waiver 到期和 canary，不依赖长期 ad-hoc 强制覆盖传递依赖。

这些修改已经纳入《博客知识库技术方案.md》的最终方案。