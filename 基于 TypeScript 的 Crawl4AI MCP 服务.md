# 基于 TypeScript 的 Crawl4AI MCP 服务

## 1. 调研对象与结论

- 项目：`omgwtfwow/mcp-crawl4ai-ts`
- 地址：https://github.com/omgwtfwow/mcp-crawl4ai-ts
- 本次分析基于项目 `main` 分支当前源码树，tree SHA：`da1435444654f6de4498cbc7932ceec1d7664638`。
- 语言：TypeScript。
- 定位：把 Crawl4AI HTTP 服务包装成 MCP Server，对外暴露 Markdown、HTML、批量抓取、智能抓取、递归抓取、Sitemap、浏览器自动化、会话和 LLM 抽取等工具。

它不适合直接作为“1000+ 技术博客全量历史 + 长期增量”的生产调度平台，因为其 frontier、session、批处理关联、Sitemap 解析、状态持久化和安全边界都偏工具型；但它非常值得借鉴的不是 MCP 本身，而是以下工程思想：

1. 把底层抓取引擎能力封装成稳定的 Adapter/Tool Contract；
2. 支持同一批 URL 使用不同 crawler config，适配异构站点；
3. 用有限递归、内容类型探测和 session 作为接入诊断能力；
4. 把执行耗时、内存等服务侧指标暴露出来；
5. 对底层 API 版本差异做兼容层，而不是让业务层直接依赖 Crawl4AI 参数细节。

本项目进一步暴露出生产方案必须补齐的几个关键点：**Adapter 能力协商、批任务结果关联、统一错误分类、浏览器 session fencing、JS 动作安全、SSRF/Egress、durable frontier**。

## 2. 代码结构与调用链

源码主要由以下层组成：

- `src/server.ts`：MCP Server 与工具注册、请求入口；
- `src/handlers/*`：各类 MCP tool 的业务 Handler；
- `src/crawl4ai-service.ts`：对 Crawl4AI HTTP endpoint 的客户端封装；
- `src/schemas/*` 与 `src/types.ts`：参数类型、校验与 API contract；
- `src/__tests__/*`：单元和集成测试，覆盖 batch crawl、recursive crawl、smart crawl、session、Sitemap 等。

核心调用模式是：

```text
MCP Tool
  -> 参数 Schema / Type
  -> Handler
  -> Axios / Crawl4AI HTTP API
  -> 结果格式化为 MCP content
```

对知识库平台而言，这一层次结构最值得吸收的是“**抓取引擎 Adapter 与业务状态机分离**”。未来可以替换 Crawl4AI、Playwright 或其它抓取器，而不改变站点、Provider、frontier、文章版本和 Web 管理的核心模型。

## 3. `batchCrawl`：异构配置能力及其隐藏问题

`CrawlHandlers.batchCrawl()` 支持两种请求模式。

### 3.1 单一共享配置

旧模式将 `options.urls` 一次传给 `/crawl`，并构造共享 `crawler_config`：

- `remove_images` 转换成 `exclude_tags`；
- `bypass_cache` 转成 `cache_mode=BYPASS`；
- `max_concurrent` 直接传给 Crawl4AI。

### 3.2 每 URL 独立配置

新模式允许 `options.configs` 中的每一项带独立 URL/config，Handler 会：

1. 从 `configs` 提取 URL 列表；
2. 同时发送 `urls` 与 `configs`；
3. 保留 `max_concurrent`。

这非常贴合 1000 个技术博客的现实：不同站点甚至同一站点不同路径，可能需要不同 UA、Browser、wait、JS、selector、cache 和 proxy 配置。因此生产系统不应把“批任务”设计成“一批共用一份抓取规则”，而应让每个 fetch task 固化自己的执行配置版本。

### 3.3 结果关联不能依赖数组位置

源码在 `configs` 模式下虽然重新计算了 `urls`，但最终输出状态时仍使用 `options.urls[i]`。若调用者只提供 `configs`，这会导致结果展示 URL 与实际任务不一致甚至出现 `undefined`。

这暴露出生产系统一个重要原则：**异步批处理绝不能用返回数组位置作为任务身份。**

知识库平台应要求：

- 每个 fetch task 有不可变 `task_id`；
- Worker 向 Adapter 发送 `task_id + url + execution_bundle`；
- Adapter 结果必须回传相同 `task_id`；
- 即使服务端乱序、部分失败、重试或拆批，也能准确关联；
- 数据库以 `task_id / attempt_id` 关联，不以 batch index 关联。

### 3.4 服务端资源指标

`batchCrawl` 会读取：

- `server_processing_time_s`；
- `server_memory_delta_mb`；
- `server_peak_memory_mb`。

这说明 Adapter 应保留“执行资源遥测”能力。生产系统可利用这些指标做：

- HTTP/Browser worker 分池；
- 按站点估算成本；
- 动态 max_concurrent；
- Browser 高内存站点限流；
- 发现异常页面或内存泄漏；
- 容量规划和 Autoscaling。

## 4. `smartCrawl`：能力探测而不是生产全量发现

`smartCrawl()` 先尝试对目标 URL 发 HEAD，然后综合 URL 特征判断内容类型：

- URL 包含 `sitemap` 或 `.xml` -> sitemap；
- URL 包含 `rss` / `feed` -> RSS；
- Content-Type 为 `text/plain` -> text；
- XML -> xml；
- JSON -> json；
- 默认 HTML。

随后统一调用 Crawl4AI `/crawl`。若类型为 sitemap/rss/xml 且允许 follow links，会从返回内容中用正则解析：

- `<loc>...</loc>`；
- `<link>...</link>`；
- `href="..."`。

然后最多只跟进前 10 个 URL，并发 3。

### 4.1 可借鉴点

它说明新站点录入时不必让管理员先手工填写“这是 Sitemap / Feed / 动态站点”。可以先做 Probe，再生成候选配置。

因此知识库方案应明确设置 `Discovery Probe`：

1. 根 URL GET；
2. robots Sitemap；
3. 常见 Sitemap；
4. HTML `rel=alternate` Feed；
5. Feed/API/Archive 候选；
6. Browser sample；
7. 样本详情页；
8. 自动生成 Provider/Profile/Contract/Action Plan 草稿；
9. 管理员审核后才发布生产配置。

### 4.2 不能直接照搬点

`smartCrawl` 只是交互式能力探测，不是全量历史发现：

- HEAD 可能被禁用或与 GET 行为不一致；
- CDN 可能返回错误 Content-Type；
- URL 命名不能作为强证据；
- regex 不能可靠解析 XML namespace、CDATA、Atom 和 Sitemap Index；
- 最多 follow 10 个链接不代表覆盖历史；
- 没有 Provider checkpoint、coverage、continuity gap；
- 没有 durable frontier。

生产 Probe 应把 HEAD 只当弱信号，并增加最终 URL、body sniffing、XML root、HTML alternate、robots、公开 API、页面结构等多证据判定。

## 5. `crawlRecursive`：典型 BFS，但只能用于小规模探测

`crawlRecursive()` 使用：

- `visited: Set<string>` 去重；
- `toVisit` 数组作为 FIFO queue；
- `depth` 维护层级；
- `max_depth` 默认 3；
- `max_pages` 默认 50；
- include/exclude regex；
- 只跟进与起始 URL hostname 相同的 internal link；
- 单页失败记录错误但不中断全局。

算法本质是有界 BFS：

```text
enqueue(start, depth=0)
while queue not empty and crawled < max_pages:
    dequeue
    if visited / over depth -> skip
    crawl page
    collect internal links
    normalize relative links
    same-host links -> enqueue(depth+1)
```

### 5.1 值得保留的能力

这种递归非常适合：

- 新站点接入结构探测；
- URL pattern 发现；
- 小规模规则回归测试；
- 找归档、分类、详情页候选。

### 5.2 生产化必须替换内存 frontier

1000+ 站点长期运行必须改成 durable frontier：

- URL 和 visited 落 PostgreSQL；
- claim 使用 lease；
- 失败写 `attempt_count / next_attempt_at`；
- 任务有幂等键；
- domain limiter 统一管理跨 worker 并发；
- robots、allow/deny、canonical、redirect identity 都在准入层处理；
- Redis Streams 只负责投递，不是真相源；
- 进程重启后可继续；
- 大站点不会因为一个 Worker 崩溃丢掉 frontier。

同 hostname 也不是充分的访问边界：同域可能有搜索、登录、日历、无限参数等 crawler trap，因此仍需路径规则、query 规则、最大组合数和预算。

## 6. Sitemap 实现：正则解析的生产风险

项目 `parseSitemap()` 直接用 Axios GET，再用：

```text
/<loc>(.*?)<\/loc>/g
```

解析 URL，最后只在 MCP 文本中显示前 100 个。

此方式不适合全量生产：

- Sitemap Index 需要递归；
- `.xml.gz` 需要流式解压；
- namespace / entity / encoding 需要 XML parser；
- 超大 Sitemap 不应一次载入内存；
- 需要读取 `lastmod`；
- Sitemap Index 与 URL Set 的语义不同；
- 需要对每个 child sitemap 建 checkpoint/coverage；
- 还需防 XML bomb、超大文档和恶意实体。

方案应使用 `defusedxml/lxml` 等安全 parser，配合 streaming pull parsing、大小预算、递归深度上限和 gzip 限额。

## 7. Browser 配置与 Action Plan 契约化

`crawl()` 会把大量输入转换为 `browser_config` 与 `crawler_config`：

- browser type；
- viewport；
- UA / headers / cookies；
- proxy；
- word count threshold；
- excluded tags；
- overlay removal；
- `js_code`；
- `wait_for` / timeout；
- scroll delay；
- iframe；
- external link；
- screenshot / pdf；
- session；
- cache mode。

它证明了“站点抓取规则”不应散落成 Worker if/else，而应形成版本化配置契约。

推荐拆为：

1. **Fetch Profile**：HTTP/Browser、UA、headers、cookie 策略、proxy、timeout、缓存、资源预算；
2. **Browser Action Plan**：wait、scroll、click、load-more、pagination、有限动作；
3. **Extraction Contract**：正文、metadata、link selector、JSON 映射、readiness；
4. **Execution Bundle**：一次 fetch task 固化上述 release id 与 Adapter release。

这样规则升级后可 canary、rollback、snapshot replay，也能解释“某篇文章当时为什么这样抓”。

## 8. JS 校验不能被当成安全沙箱

`crawl4ai-service.ts` 中的 `validateJavaScriptCode()` 主要用字符串规则拒绝：

- HTML entity；
- 明显 HTML tag；
- 一些字面量 `\n` 误用。

这类校验只能提高工具易用性，不能构成安全隔离。它无法证明任意 JS 不会：

- 发起额外网络请求；
- 读取敏感页面状态；
- 死循环；
- 大量分配内存；
- 访问意外 DOM/Storage；
- 执行与抓取无关的行为。

生产方案应默认不用任意 JS，而是优先提供受控 Action DSL：

- `wait_selector`；
- `click_selector`；
- `scroll`；
- `next_page`；
- `load_more`；
- `extract_links`。

确需 JS 时应：

- 单独权限；
- 版本化；
- 静态规则/AST 检查；
- 最大执行时间；
- Browser Context 网络 egress 限制；
- CPU/内存预算；
- 审计日志；
- 禁止在普通管理员无感情况下自动生成并直接发布。

## 9. Session 管理：工具型内存状态与生产 lease 的差别

`SessionHandlers` 使用进程内 `Map` 保存：

- session id；
- created_at；
- last_used；
- initial_url；
- browser_type。

创建 session 时，如果传入 `initial_url`，会先调用一次 `/crawl` 预热，并把 `session_id` 放入 crawler config。`clear` 只删除本地 Map；源码注释明确说明真实 Crawl4AI browser session 会依靠服务端 inactivity 或重启清理。

这有两个生产风险：

1. 控制面记录的“session 已删除”并不等于远端浏览器状态已释放；
2. Worker 崩溃、网络分区和重复调度时可能出现旧 session 被错误复用。

生产系统应引入 `browser_session_lease`：

- `session_lease_id`；
- `worker_id`；
- `adapter_session_id`；
- `generation/fencing_token`；
- `lease_until`；
- `last_heartbeat_at`；
- `max_requests`；
- `max_lifetime`；
- `cleanup_state`。

每次 Browser 请求都携带 fencing token，旧 lease 即使“复活”也不能继续写生产结果。会话释放应显式调用 Adapter cleanup；显式失败时进入 orphan cleanup job，而不是只依赖超时。

敏感 cookie/localStorage 默认不持久化，确有业务需求时应独立加密、最小权限和审计。

## 10. Crawl4AI Service Adapter：版本兼容与能力协商

`crawl4ai-service.ts` 把 `/md`、`/html`、`/crawl`、`/screenshot`、`/pdf`、`/execute_js`、`/llm/...` 等 endpoint 包装为 TypeScript 方法。

源码还明显存在对 Crawl4AI 版本差异的兼容：

- 新版 `proxy`；
- 旧版 `proxy_config`；
- batch per-URL `configs` 注释为新版本能力；
- 某些 endpoint 只允许有限字段。

因此生产系统不能假设“Adapter 永远支持同一套参数”。应新增 **Adapter Capability Manifest**。

建议每个 Adapter release 在启动时注册：

```text
adapter_name
adapter_release_id
engine_name
engine_version
api_version
supports_http
supports_browser
supports_per_url_config
supports_session
supports_execute_js
supports_screenshot
supports_pdf
supports_markdown
supports_raw_html
supports_metrics
max_batch_size
max_concurrency
supported_browser_types
supported_cache_modes
```

Control Plane 发布 Fetch Profile/Action Plan 前先做 capability validation；Worker claim task 时固定 `adapter_release_id`，避免运行中升级引擎后同一 job 的语义发生漂移。

若 Adapter 不支持某能力，应在调度前 fail-fast，而不是到了 Worker 才报错。

## 11. 错误处理：应升级为统一错误分类和重试矩阵

`crawl4ai-service.ts` 对 Axios 错误做了有价值的归一化：

- timeout；
- DNS `ENOTFOUND`；
- connection refused/reset；
- network unreachable；
- HTTP status；
- LLM 401/504 等。

目前它最终还是以文本 Error 返回，生产平台应进一步结构化为：

```text
error_class
error_code
retryable
retry_after
http_status
domain_scope
adapter_scope
root_cause
```

建议错误类别：

- `dns_failure`；
- `connect_timeout`；
- `read_timeout`；
- `connection_reset`；
- `tls_failure`；
- `http_401_403`；
- `http_404_410`；
- `http_429`；
- `http_5xx`；
- `robots_denied`；
- `egress_denied`；
- `content_too_large`；
- `browser_crash`；
- `browser_timeout`；
- `extraction_failed`；
- `adapter_contract_error`。

不同错误使用不同策略，例如：

- 429 尊重 Retry-After 并降低 domain bucket；
- 404/410 通常不快速重试；
- DNS/5xx 指数退避；
- 403 进入策略审核而非无限重试；
- Adapter 参数不兼容直接失败并阻止同 release 后续任务。

## 12. SSRF / Egress 是必须补强的边界

项目中多处会直接对外部 URL 使用 Axios，例如：

- `smartCrawl` 的 HEAD；
- `parseSitemap` 的 GET；
- service 的 `detectContentType` / `parseSitemap`；
- 其它 Crawl4AI endpoint 最终也会访问用户输入 URL。

部分方法只做 `new URL()` 语法校验，这并不能防 SSRF。

若 MCP Server 部署在公司内网、云 VPC 或可访问 metadata endpoint 的环境中，任意 URL 抓取必须经过统一 Egress Policy：

- 只允许 http/https；
- 禁止 localhost、RFC1918、link-local、loopback、metadata IP；
- DNS resolve 前后都检查；
- redirect 每跳重新检查；
- 端口 allowlist；
- DNS rebinding 防护；
- Browser 子请求也应用同一策略；
- 下载图片、附件、Sitemap 不能绕过统一 egress gateway。

第三方 Adapter 不允许直接自行发外部请求绕开平台 Egress 层。

## 13. MCP 的合理定位

MCP 很适合：

- 单 URL 调试；
- 站点 Probe；
- 查询站点状态；
- 搜索知识库；
- 获取 Markdown；
- 受控 dry-run；
- 运维诊断。

MCP 不应成为：

- frontier 真相源；
- checkpoint 真相源；
- Browser session 真相源；
- article version 真相源；
- 长周期 backfill scheduler。

原因是交互式 tool call 更适合“调用能力”，不适合承载可恢复、可审计、跨天运行的生产工作流。

## 14. 对博客知识库技术方案的具体优化

基于本项目，本次建议在已有方案上新增或强化以下内容。

### 14.1 Adapter Capability Manifest

抓取 Adapter 必须注册 engine/API 版本及能力矩阵，Profile/Action Plan 发布前做兼容校验；Task 固化 Adapter release。

### 14.2 Execution Bundle 与逐 URL 配置

每个 fetch task 固化：

- `profile_release_id`；
- `contract_release_id`；
- `action_plan_release_id`；
- `adapter_release_id`；
- `canonicalizer_release_id`；
- 资源预算。

同一 batch 只是一种传输优化，不意味着共享配置。

### 14.3 Batch Result Correlation

所有批结果必须用 `task_id/attempt_id` 关联，不依赖返回顺序。部分失败、乱序、拆批、重试均可正确归档。

### 14.4 结构化错误分类与 Retry Policy

将 Adapter 文本错误转换成平台统一错误 taxonomy，让 Scheduler、Domain Limiter、Web 管理和告警共用同一语义。

### 14.5 Browser Session Fencing 与显式清理

在已有 session lease 基础上加入 generation/fencing token、heartbeat、显式 cleanup 和 orphan reaper。

### 14.6 Browser Action 安全

默认使用白名单 Action DSL，任意 JS 仅作为受控例外；即使做字符串/AST 校验也不能把它当成安全沙箱。

### 14.7 Adapter 遥测

记录处理耗时、峰值内存、Browser 秒数、网络字节、retry、queue wait，支持成本与容量分析。

### 14.8 Discovery Probe 继续保持“探测与生产分离”

保留 smart crawl 的自动识别思想，但正式 parser、durable provider 和 coverage 才负责生产全量。

## 15. 不建议直接采用的实现

以下能力可以用于开发工具，但不应直接进入生产核心：

- regex 解析 Sitemap/RSS；
- HEAD/URL 名称决定内容类型；
- follow 前 10 个 URL 作为历史发现；
- 单进程 `Set + Array` 做长期 frontier；
- 本地 Map 做 Browser session 真相源；
- 只删除本地 session 记录而不显式远端清理；
- 批结果用数组 index 关联；
- 任意 JS 只靠字符串检查；
- Adapter 直接访问外部 URL 绕过统一 Egress Policy；
- 把 MCP call 当持久任务调度器。

## 16. 最终结论

`mcp-crawl4ai-ts` 对本知识库项目最有价值的不是“使用 MCP 抓网页”，而是展示了一个清晰的抓取能力封装层，以及异构 URL 配置、递归发现、会话、浏览器动作和服务侧遥测如何被统一工具化。

生产方案应吸收：

- Adapter/Tool Contract；
- per-URL config；
- Discovery Probe；
- Browser session；
- 递归发现；
- 执行指标；
- API 版本兼容思路。

同时必须生产化为：

- `Adapter Capability Manifest`；
- immutable `Execution Bundle`；
- `task_id/attempt_id` 批结果关联；
- durable frontier；
- 正式 XML/Feed parser；
- 结构化错误 taxonomy；
- Browser session fencing + cleanup；
- 白名单 Browser Action DSL；
- 统一 SSRF/Egress；
- MCP 仅作为查询、Probe、dry-run 和运维入口。

这些补强能让底层抓取引擎在未来版本升级或替换时保持业务状态稳定，也能避免工具型实现中的内存状态、位置关联和安全边界问题扩散到 1000+ 站点的长期生产系统。