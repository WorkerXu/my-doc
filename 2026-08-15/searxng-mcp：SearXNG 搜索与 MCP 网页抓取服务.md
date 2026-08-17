# searxng-mcp：SearXNG 搜索与 MCP 网页抓取服务

- 编号：33
- 项目地址：https://github.com/TadMSTR/searxng-mcp
- 调研基线：`main` 分支，代码树 `07d5102093feb7a7dd62886f29aae8c276dbc345`
- 重点代码：`src/fetch.ts`、`src/routing.ts`、`src/domain-db.ts`、`src/cache.ts`、`src/crawl.ts`、`src/fetch-utils.ts`、`src/http-transport.ts`、`src/ssrf-guard.ts`、`src/extractors/*`

## 1. 项目定位

searxng-mcp 不是单纯的搜索 MCP 封装，而是把“搜索、结果重排、全文抓取、站点爬取、正文抽取、缓存、域名能力学习、可观测性”组合成一条可降级的研究型数据获取链路。它围绕自托管 SearXNG 提供搜索入口，抓取侧把 Firecrawl、Crawl4AI、Raw HTTP 等实现组织为分层路由，并针对 GitHub、`llms-full.txt`、Kiwix、本地浏览历史等来源提供 fast path。

对“1000 个技术博客全量历史文章 + 后续增量同步”的价值不在于直接采用这个 MCP Server，而在于它展示了一个重要工程方向：**抓取策略不应永久写死为同一套固定 cascade，而应持续记录站点/域名的实际能力，通过历史成功率、内容质量和运维覆盖来决定后续路由，同时保留降级路径和人工覆盖。**

## 2. 总体调用链

项目的搜索与抓取大致分成两条链：

```text
搜索：
query
  -> SearXNG
  -> 本地 reranker
  -> Top-N
  -> 可选全文抓取
  -> 可选 Ollama 摘要

单页抓取：
URL
  -> URL/域名规则检查
  -> Fetch Cache
  -> 平台/内容 fast path
  -> robots.txt gate
  -> 域名能力路由决策
  -> Firecrawl
  -> Crawl4AI
  -> Raw HTTP + Readability
  -> 可选 Wayback
  -> Post Extract
  -> Cache
```

`fetchPage()` 是抓取主控制器。它并不把 Firecrawl/Crawl4AI 当作互斥产品，而是把它们当成具有不同成本、动态渲染能力和失败特征的 route。抓取成功后还会再经过统一的后处理，使上层拿到较稳定的 `{title, url, text}` 结果。

这一结构的工程意义是“**路由与抽取解耦**”：Fetcher 负责拿到可解释的页面表示，Post Extract 再负责标题、JSON-LD、Readability 等内容解释。对知识库系统还应进一步拆开为 Fetch Snapshot、Extraction Candidate、Canonical IR 和 Markdown Projection，避免任何第三方抓取器返回的 Markdown 直接成为唯一真相。

## 3. `fetchPage` 的实现细节

### 3.1 Cache 先行但 fail-soft

`src/cache.ts` 使用 Valkey/Redis 兼容后端。关键点不是缓存本身，而是它明确给连接和命令设置 `commandTimeout`、`connectTimeout`、`maxRetriesPerRequest`；缓存后端卡死时，`cacheGet()` 会退化成 cache miss，而不是拖死整个搜索/抓取请求。`cacheSet()` 也是 best-effort，不让缓存故障影响主业务结果。

v3.16 进一步给缓存错误增加节流日志，连接失败日志会对 URL 凭证做脱敏，`cachePing()` 也复用同一套短超时，因此健康检查本身不会因为缓存卡死而无限挂住。

这解决了常见的“缓存从性能组件变成可用性单点”问题。但对于长期知识库不能照搬其“缓存内容即主要可复用内容”的边界：Redis 应只作为性能层，真正不可变的抓取证据必须落在 S3/MinIO，业务状态与索引状态落 PostgreSQL。

### 3.2 Fast Path

项目在通用 cascade 之前有多种特殊路径：

- GitHub URL 直接走 GitHub/raw/API 路径；
- 白名单文档站优先尝试 `llms-full.txt`；
- 已有离线 ZIM 的站点可从 Kiwix 返回；
- 可选 Hister 读取浏览历史中已经渲染过的页面；
- YouTube/Reddit 有针对内容形态的专用实现；
- PDF 跳过不合适的 tier，直接进入更适合的提取路线。

其底层思想是：**先识别“是否存在更高保真、更低成本、更稳定的表示提供者”，再进入通用网页抓取。**

知识库里可以把它抽象成 `RepresentationProvider`：Platform API、Feed Full Content、Sitemap Metadata、llms-full、Static HTTP、Browser DOM、PDF Extractor 等都只是不同表示来源。每个 Provider 都必须保存 provenance、freshness、映射关系和可信度，不能因为 fast path 更便宜就无条件当 Canonical Document。

### 3.3 robots 与 SSRF 之前/之后的边界

普通网页路径会经过 robots.txt gate。项目还单独实现了 SSRF 防护：

1. 字符串级 URL 检查阻止明显非法/私网地址；
2. 自己发出的 HTTP 请求使用自定义 Undici dispatcher，在 connect-time DNS lookup 阶段检查实际要连接的 IP；
3. redirect 每次连接仍经过 DNS guard，因此可缩小 DNS rebinding 和“预解析后再连接”的 TOCTOU 问题；
4. 对 Firecrawl/Crawl4AI 这种由外部服务自己重新解析 URL 的抓取器，只能先做 `assertResolvedPublic()`，源码明确指出外部服务再次解析时仍然存在 TOCTOU 窗口。

对于有 Web 管理端、允许用户输入 Target URL 的系统，这一点非常重要。生产方案还需加网络层防线：Browser/Firecrawl/Crawl4AI Worker 独立网段、默认拒绝访问 RFC1918/link-local/metadata 地址、统一 egress proxy、Kubernetes NetworkPolicy/云防火墙、DNS policy，并限制 redirect、下载大小、解压膨胀比和 Content-Type。应用层校验不能替代网络隔离。

### 3.4 Tier Cascade

普通页面的主 cascade 是 Firecrawl -> Crawl4AI -> Raw HTTP，最后可选 Wayback。每个 tier 经 `runTier()` 包装：

- 记录 span 和延迟；
- 区分 hit/miss/error；
- 产生 metric/event；
- 更新域名能力统计；
- 单 tier 异常转换成该 tier 失败，让后续 tier 继续尝试。

所以 fallback 不是异常控制流的偶然效果，而是明确的业务状态机。

目标系统应进一步区分两类成功：

```text
Transport Success: 请求本身成功，如 HTTP 200 / Browser loaded
Content Success: 得到符合文章质量门槛的正文
```

如果浏览器返回一个“Enable JavaScript”“Access denied”或 200 状态的空壳页面，只能算 transport hit，不能算 content hit。自适应路由统计应该主要使用“可接受正文”的成功率，而不是单纯 HTTP 成功率。

### 3.5 Metadata side-channel 的隐藏成本

`fetch.ts` 在进入 tier cascade 后，会并行启动 `fetchRawHtmlForMetadata(url)`，主 tier 成功后再等待这份 HTML，用于 JSON-LD、OpenGraph 和 Readability 的后处理。这能隐藏额外请求的延迟，但没有消除额外网络访问本身。

对研究型 MCP 工具，这种做法可以接受；对 1000 站点长期同步则要特别治理：如果主 route 本来已经拿到 Raw HTML/Rendered DOM，再额外请求一次只是为了 metadata，会放大站点流量、限流风险、成本和内容版本不一致风险。更合理的生产规则是：

1. 主 route 能提供 HTML/DOM 时，metadata、链接和正文抽取必须共享同一 Snapshot；
2. 主 route 只提供 Markdown/API 正文、确实没有源表示时，才允许独立 `METADATA_PROBE`；
3. 独立 Probe 应采样、限频、缓存并有单独预算，不应每篇文章无条件执行；
4. Probe 的运行事实要按独立 `operation_type` 记录，不能污染 `ARTICLE_FETCH` 的 route 成功率；
5. 主内容与 Probe 来自不同时间点时，字段 provenance 必须明确，不得假设二者天然属于同一版本。

## 4. 域名能力数据库与自适应路由

这是本项目最值得吸收的部分。

### 4.1 Domain Capability DB

`src/domain-db.ts` 以 `domain:<hostname>` 保存每个域名的能力记录，包含：

- 各抓取 tier 在 30 天窗口内的 attempts / ok / fail / last_fail_reason；
- robots.txt 是否存在、是否允许；
- `llms-full.txt` 是否存在；
- JSON-LD Article 与 OpenGraph title 的采样情况；
- metadata side-channel 是否可访问；
- 域名是否在搜索结果中出现；
- preferred strategy 等。

记录有 schema version；schema 变化时旧结构不会被错误沿用。写入是 best-effort，不允许“学习能力失败”反向导致正文抓取失败。

### 4.2 并发写的一致性

多个请求可能并发更新同一域名。项目没有只靠单进程锁，而是使用 Valkey `WATCH/MULTI/EXEC` 做乐观并发控制。大意是：

```text
WATCH domain:key
old = GET domain:key
new = mutate(old)
MULTI
SET domain:key new EX ttl
EXEC
```

若 EXEC 因并发修改返回冲突则有限次数重试。这比进程内 Promise queue 更适合多进程/多副本运行。

对于生产知识库，能力观察不应该只存在 Redis。更稳妥的设计是：**PostgreSQL 保存 append-only Observation 真相，异步聚合出 Route Capability Projection；Redis 只缓存 Projection。** 这样可审计某个路由为何被选择，也不会因 TTL、flush 或缓存重建而永久丢失历史学习。

### 4.3 当前路由算法

`src/routing.ts` 的规则非常清晰：

- operator `tier_skip` 覆盖优先；
- 冷启动不足 10 次仍保留默认 cascade；
- 某 tier 至少 10 次尝试且成功率低于 30% 时，后续可以跳过该 tier；
- 决策会记录 skipped 原因。

优点是易解释、易运营，而且能快速避开某个域名对特定抓取引擎“长期必失败”的浪费。

但不能直接作为 1000 站点长期调度算法，原因有四个：

1. **固定阈值容易固化旧事实。** 某站点一周 WAF 异常可能把一个 route 判死，站点恢复后仍长期被跳过。
2. **只看成功率没有成本/质量维度。** Browser 可能 95% 成功但成本是静态 HTTP 的几十倍；Raw HTTP 可能 90% 成功却偶尔正文缺失。
3. **域名粒度过粗。** 同域名 `/blog/`、`/docs/`、`/api/` 可能完全不同，需要 host + path scope，甚至 provider scope。
4. **没有显式探索。** 如果最差 route 永久不再试，就无法发现环境变化。

目标方案应采用带置信度的自适应路由：EWMA/Beta-Binomial 都可以；同时保存 success、quality、latency、cost，并设计 exploration floor、cooldown、定期 canary re-probe。人工 override 要有原因、创建人、TTL 与自动到期，不能成为永久不可见配置。

## 5. 抓取能力学习如何转化为生产数据模型

建议至少保留三类对象。

### 5.1 Capability Observation

每次 route 实际尝试都追加一条不可变事实：

```text
site_capability_observation
- site_id
- host_key
- path_scope
- operation_type        # DISCOVERY_FETCH / ARTICLE_FETCH / METADATA_PROBE / ASSET_FETCH
- route_id
- route_release_id
- attempted_at
- transport_outcome
- content_outcome
- quality_score
- http_status
- blocker_type
- error_class
- latency_ms
- bytes
- cost_units
- render_required
- snapshot_id
- trace_id
```

Observation 不做覆盖更新，保留原始事实。`METADATA_PROBE` 必须和正文抓取分开统计，因为“拿到一份可解析 HTML metadata”与“得到可接受正文”是不同能力。

### 5.2 Route Capability Projection

聚合最近窗口与衰减历史：

```text
site_route_capability_projection
- site_id
- path_scope
- operation_type
- route_id
- sample_count
- confidence
- ewma_success
- ewma_quality
- p50_latency
- p95_latency
- cost_estimate
- last_success_at
- last_failure_at
- cooldown_until
- next_probe_at
- projection_version
```

### 5.3 Route Decision

每个 Task 保存当时为什么选择这一条路：

```text
route_decision
- decision_id
- task_id
- candidate_routes
- selected_route
- skipped_routes
- reasons
- capability_projection_version
- policy_release_id
- operator_override_id
- decided_at
```

这样未来审计“为什么某次没有用 Browser”“为什么这个站点两个月都只走静态 HTTP”时有完整证据，而不是看当前配置猜历史行为。

## 6. 站点抓取 `crawl_site` 的实现与边界

项目的站点抓取是三阶段 fallback：

1. Firecrawl `/crawl`：启动异步 job，轮询状态，拿到页面列表和 Markdown；
2. Sitemap：从 robots 的 `Sitemap:` 以及常见 sitemap 路径发现 XML，递归 Sitemap Index；
3. BFS：可选，限制深度、页数和同域范围。

页面内容在 crawl 时也写入 fetch cache，因此后续 `fetch_url` 可以复用。

这适合研究工具，但用于“全量历史知识库”需要调整优先级：

- Sitemap/Feed/API/Archive/分类分页等是 **Coverage Provider**，应先用于证明“理论上有哪些历史文章”；
- Firecrawl/BFS 是 Discovery Engine 或 gap finder，不能用“爬完 max_pages”证明历史完整；
- Search 也只能发现缺口，Top-N 更不能成为完整性证据。

另一个值得注意的实现点是 BFS：它先 `fetchPage()` 获取正文，随后为了抽取链接又发一次 raw HTTP 获取 HTML。研究工具可以接受，但 1000 站点会造成明显重复流量。目标方案应保存同一次网络访问得到的 Raw/Rendered Snapshot，由 Discovery Parser 和 Extractor 共同消费；同一页面不应为了“找链接”和“抽正文”各抓一次。

## 7. 正文质量策略

项目的 Post Extract 做了几种实用增强：

- 从 JSON-LD 的 Article/NewsArticle/BlogPosting/TechArticle 读取标题/正文线索；
- 标题按 OpenGraph、Twitter、`<title>`、`<h1>` 等候选组合；
- Crawl4AI 返回 Markdown 时，还可把 raw HTML 交给 Readability，再比较两者正文结果；
- metadata HTML 作为 side-channel 与主 cascade 并行获取，避免完全串行增加延迟。

可借鉴的是 **Candidate Competition**：不要永远只信一个 Extractor。一个 Snapshot 可以产生多个 Extraction Candidate，再由 Quality Policy 选择 Accepted Candidate。

生产系统可比较：

- 正文字数与 DOM text ratio；
- 标题一致性；
- 代码块/表格/图片保留率；
- boilerplate 比例；
- 重复块比例；
- 发布时间/作者字段 provenance；
- 与 Golden URL 的结构特征偏移。

但比较策略要确定性、版本化，并保存“落选 Candidate”，这样规则升级能离线 replay，而不是重复访问源站。

## 8. 可观测性与事件模型

项目可选 OpenTelemetry 和 NATS。它给 search/fetch/tier/cache/error 等阶段建立 span、counter、histogram，并让事件带 request_id/trace_id。

对知识库系统，应把这类技术指标和业务语义计数分开：

```text
技术：latency、timeout、connection error、browser crash、queue lag
业务：new_version、unchanged、existing_identity、unexpected_empty、rejected、blocked、coverage_gap
路由：route_selected、route_skipped、route_escalated、probe_success、probe_failure
```

只看“请求成功率”很容易掩盖抽取空正文、重复入库、站点漏抓等业务失败。

## 9. 对 1000 站点方案最值得吸收的能力

### 9.1 从静态 Profile 升级为“Profile + Learned Capability”

Site Profile 仍是人工声明的稳定策略，但运行事实单独形成 Capability Observation。两者结合：

```text
Resolved Route Policy
= Global Policy
+ Platform Adapter Policy
+ Site Profile Release
+ Operator Override
+ Learned Capability Projection
+ Task-specific constraints
```

这样新增网站可以先用保守通用策略，随着观测积累自动降低 Browser 使用率，同时允许管理员覆盖。

### 9.2 Fast Path 必须成为一等 Provider

对于技术博客尤其有价值：

- GitHub/GitLab 仓库内容走 API/raw；
- 文档站发现 `llms-full.txt` 可作为替代表示；
- RSS/Atom 若含 full content 可直接产 Candidate；
- WordPress/Ghost/Dev.to 等平台可有 API Adapter；
- PDF 走专门解析器；
- 静态博客走 HTTP + DOM Extractor。

Fast Path 的目标不是“绕过主链路”，而是更快地产生符合相同 Artifact Contract 的 Snapshot/Candidate。后续 Quality、Identity、Versioning 必须完全一致。

### 9.3 自适应 Browser Escalation

不要配置“某站点永远 Browser”。每次先根据能力投影选择最便宜、足够可靠的路线；遇到 challenge、正文过短、JS shell、selector miss 等明确证据再升级 Browser。Browser 成功后也应定期重新探测 HTTP，防止长期高成本锁定。

### 9.4 路由学习与 Coverage 必须彻底分离

“某域名 Firecrawl 成功率高”只说明 Fetch Capability，不能说明 Sitemap/Feed 是否覆盖全部历史文章。Capability Evidence 与 Coverage Evidence 是两套不同证据，Web 管理端也应分开展示。

## 10. 不应直接照搬的设计

1. **Valkey 不能作为长期能力真相源。** TTL/flush 会丢学习；快照恢复仍弱于 append-only DB 事实表。
2. **10 次 + 30% 是演示级阈值。** 生产需要置信度、时间衰减、质量/成本和探索机制。
3. **Firecrawl-first 不能代表全量历史发现。** 先用权威 Provider 做 Coverage，再用 Crawl/Search 做 gap discovery。
4. **BFS 不应双抓同一 URL。** Discovery 与正文抽取消费同一 Snapshot/DOM。
5. **Fetch cache 不是知识库历史。** 原始响应、Rendered DOM、IR 必须不可变归档。
6. **Fast Path 结果不能默认 canonical。** 需要 provenance、freshness、representation contract 和质量验收。
7. **外部 Fetcher 的 SSRF 预解析不够。** 必须补 egress 网络隔离和目的地址策略。
8. **固定长度返回适合 MCP，不适合归档。** 知识库存全量内容；截断只属于查询/API Projection。
9. **域名级 route stats 太粗。** 至少支持 host/path/operation scope。
10. **只记录成功/失败不足以驱动优化。** 要记录正文质量、成本、延迟、blocker、render-required 等语义。
11. **Metadata side-channel 不应默认每次都独立联网。** 能复用 Snapshot 就必须复用，不能用“并行请求不增加主链路延迟”掩盖源站流量和一致性成本。
12. **资源边界不能只限制公网 HTML。** 内部 Firecrawl/Crawl4AI/SearXNG/Reranker/LLM 的 JSON/文本响应同样必须设置字节上限、deadline 和取消传播。

## 11. 推荐路由算法

目标系统可采用如下可解释流程：

```text
1. Hard Gate
   - scheme / robots / scope / SSRF / egress / content-type / policy

2. Representation Fast Path
   - platform API / full feed / llms-full / static artifact
   - 通过 provenance + freshness + quality gate 才接受

3. Build Route Candidates
   - HTTP_STATIC
   - HTTP_STRUCTURED
   - CRAWL4AI
   - PLAYWRIGHT_PROFILE
   - PDF_EXTRACT
   - 其他 Adapter

4. Apply Site/Profile/Operator constraints

5. Read Capability Projection

6. Rank
   score = expected_quality
           - λ * expected_latency
           - μ * expected_cost
           - ν * failure_risk

7. Exploration
   - 冷启动保留所有合法路线
   - 对被降权路线保留最小 canary 探测率
   - cooldown 到期重新探测

8. Execute selected route

9. Quality Gate
   - transport success != content success

10. On classified failure
    - 写 Observation
    - 若预算允许，升级下一 route

11. Persist RouteDecision + Runtime Attestation
```

路由算法必须是 Release，参数变化后可以用历史 Observation 离线回放，对比 Browser 比例、抓取成功率、质量和成本，再决定发布。

## 12. 对 Web 管理端的直接启发

建议新增“站点能力与路由”页面，至少显示：

- 每个站点各 route 的 7/30/90 天成功率；
- content acceptance rate，而不只是 HTTP success；
- p50/p95 延迟和估算成本；
- 当前 preferred route 与原因；
- 最近 blocker/error 分类；
- 是否处于 cold-start、cooldown、exploration；
- Fast Path 能力：Feed Full、API、llms-full、JSON-LD、Browser Required；
- route 决策时间线；
- 手工 override（带原因和失效时间）；
- “立即 Probe”按钮，生成标准 `PROBE` Run，而不是在 Web 请求线程直接抓取。

这样新增第 1001 个网站时，管理员不必猜“应该用 Crawl4AI 还是 Playwright”，系统可先探测再逐步学习。

## 13. v3.16 可靠性与资源边界治理

### 13.1 有界读取不是单个抓取器的实现细节

`fetch-utils.ts` 把 Raw HTML 最大读取量限制为 2 MiB，达到上限后主动取消剩余 body，避免超大响应进入 JSDOM 后产生内存放大。v3.15.1 还把 Firecrawl/Crawl4AI 单页 tier 的 JSON 返回从无界 `res.json()` 改为“有界文本读取 + JSON.parse”。

这个原则应提升为平台级 Contract：**所有网络和 IPC 边界都必须是有界的。** 不仅公网 HTML，包括内部 Firecrawl/Crawl4AI、SearXNG、Reranker、Embedding、LLM、对象存储签名下载、Provider API 都要有：

- connect/read/total deadline；
- request body 与 response body 最大字节；
- Content-Type/解压膨胀限制；
- cancellation propagation；
- 超限后的明确错误分类，而不是 OOM 或任务假死。

`crawl.ts` 中 Firecrawl crawl job 的启动/轮询控制响应仍有直接 `res.json()` 路径，这恰好说明“修过某几个 HTTP 调用”不能视为完成治理；应该由统一 HTTP Adapter/SDK Contract 强制，而不是依赖调用方自觉。

### 13.2 长生命周期资源必须有 hard cap、idle eviction 与 in-flight 保护

`http-transport.ts` 对 MCP session map 做了三层保护：

1. 定期清理长时间无活动 session；
2. 超过最大 session 数时回收最久未使用且非 busy 的 session；
3. 维护 `inFlight` 计数，长任务执行期间不允许被 idle sweep 或容量回收误杀，完成后再刷新 activity 时间。

这不是 MCP 专属技巧，对博客知识库的 Browser Context Pool、HTTP session、DNS resolver cache、模型 runtime handle、临时下载对象、WebSocket/SSE session 都适用。只做“空闲超时”会误杀长任务，只做“最大数量”又可能回收正在工作中的对象；必须同时存在容量上限、空闲回收、busy pin 和最终释放。

### 13.3 健康检查要区分活着、可接单和依赖降级

项目 `/health` 通过有界 `cachePing()` 返回 cache up/degraded 和当前 session 数，避免健康检查自己被卡死。长期知识库应进一步把语义拆开：

- **liveness**：进程/Event Loop/Worker 是否还活着；
- **readiness**：是否能安全接收本类型新任务；
- **dependency degraded**：缓存、搜索、reranker、AI 等非关键依赖是否降级；
- **critical dependency unavailable**：PostgreSQL/S3 等当前任务必须依赖的真相源是否不可用。

例如 Valkey 故障若系统设计为 fail-soft，不应直接把 HTTP 抓取 Worker 判死并触发重启风暴；应该降低 readiness 或显示 degraded，由业务按依赖性质决定是否继续。

### 13.4 日志本身也属于安全边界

v3.16 在缓存连接失败日志里对带密码的 Valkey URL 做凭证脱敏，并对重复错误日志节流。目标系统的 URL、Authorization、Cookie、代理凭证、Secret Manager reference、对象存储签名 URL 同样要做结构化 redaction；高频站点故障需要按 site/error fingerprint 节流，否则日志洪泛会反过来成为稳定性问题。

## 14. 对最终技术方案的优化结论

searxng-mcp 最有价值的不是 SearXNG 或 MCP 接口本身，而是四组可迁移的工程思想：

1. **多路 Fetch Adapter + fast path + fallback cascade**，让不同站点类型用最合适的抓取方式；
2. **Domain Capability Observation + adaptive routing**，把真实运行数据变成后续路由依据；
3. **fail-soft、可观测、SSRF/robots 防护和人工 override**，让抓取能力可以长期运维；
4. **有界网络读取、资源池容量治理和 side-channel 预算化**，避免系统在 1000 站点规模下因为隐藏的第二次请求、卡死依赖或无界进程内状态失稳。

对于博客知识库，应该把这些能力升级为更耐久的架构：PostgreSQL 保存 append-only Observation 和 RouteDecision，Redis 只缓存投影；自适应路由同时考虑内容质量、成本、延迟与探索；Coverage 与 Fetch Capability 分离；所有 route 最终产出统一不可变 Snapshot，Discovery、Metadata、Extraction、Quality 与 Markdown Projection 尽量基于同一证据链执行；确需独立 side-channel 时显式建模、限频和审计。这样才适合 1000 站点、百万级文档和多年增量同步，而不是把研究型抓取 cascade 直接放大运行。
