# Crawl4AI Domain Mapping：多源域名 URL 发现与 Soft-404 检测

## 1. 调研对象与结论

调研对象：Crawl4AI `DomainMapper`、`DomainMapperConfig`、`AsyncUrlSeeder` 及与多 URL 执行相关的 Dispatcher。

- 官方文档：`https://docs.crawl4ai.com/core/domain-mapping/`
- 上游项目：`https://github.com/unclecode/crawl4ai`
- 重点源码：`crawl4ai/domain_mapper.py`、`crawl4ai/async_configs.py`、`crawl4ai/async_url_seeder.py`、`crawl4ai/async_dispatcher.py`
- 相关测试：`tests/unit/test_domain_mapper_unit.py`、`tests/integration/test_domain_mapper_e2e.py`、`tests/adversarial/test_domain_mapper_adversarial.py`
- 版本背景：DomainMapper 在 v0.8.7 加入；当前官方文档已进入 v0.9.x 系列，生产必须 pin 经过验证的 immutable runtime release。

核心结论：**DomainMapper 很适合作为“Source 接入勘探 + Coverage Gap 辅助发现”的运行时组件，但不能直接作为 1000 站博客知识库的 Coverage Truth、URL Identity、持久 Frontier、当前 URL 有效性真相、合规边界或全局限流器。**

最值得吸收的能力是：

1. 把域名级发现从单一 Sitemap 扩展为 Sitemap、Wayback、CT、Probe、robots、Feed、Homepage 等多源证据；
2. 把 Soft-404 识别提升为一等能力，避免 SPA / 自定义 404 把大量不存在页面误判为文章；
3. 将 Provider 失败隔离、per-source timeout、并行扫描思想用于平台 Discovery Plane；
4. 将 Crawl4AI 的 MemoryAdaptiveDispatcher 用作 Worker 本地资源保护，而不是业务队列；
5. 保持 PostgreSQL + Object Storage 为 URL、任务、Coverage、Artifact 和版本的业务真相。

同时必须增加反向约束：DomainMapper 原生能力更接近“域名侦察工具”，会发现证书子域、猜测常见子域、探测通用路径，并主动对发现 host 发 HTTP 请求。博客知识库必须把它收敛到**已批准内容 host、低风险内容路径、robots/access policy、SSRF allowlist 和平台级限流**内。

本次源码级检查还发现几个对生产设计非常关键的边界：

- `hits_per_sec` 在 DomainMapper 中实际通过 `asyncio.Semaphore` 实现，限制的是在途并发而不是严格 RPS；
- `_normalize_and_dedup()` 的去重 key 会对完整规范化 URL 调用 `.lower()`，可能错误合并大小写敏感 path/query；
- Wayback URL 在并入结果时会被标记为 `status="valid"`，但并没有逐 URL live validation；
- 当前源码里 Common Crawl 在 DomainMapper 中明确参与 host discovery，但没有看到像 Wayback `_wayback_urls` 那样把 CC URL 集合保存后并入 Phase 2 结果的对称路径；因此不能把 DomainMapper 当 Common Crawl 全历史 URL Provider；
- Host validation 对候选 host 发 HEAD，只要请求没有抛异常就会返回该 host，HTTP 403/404 等并不会使 host 被判定为不可达；
- Probe、Soft-404、Feed、Homepage 多处直接以 `https://host` 构造 URL，HTTP-only / 特殊 origin 行为不能只靠 DomainMapper 默认路径处理；
- `extract_head=True` 会对大量结果额外做 metadata fetch，适合小规模勘探，不适合作为百万 URL Inventory 的默认动作；
- `max_urls`、BM25 query、`score_threshold` 都属于选择/排序能力，不能进入全历史 Coverage Truth；
- 当前源码定义了 DomainMapper 本地 cache helper，但在本次检查的 `scan()` 主路径中没有看到它被作为业务 checkpoint 使用；无论具体版本是否启用，都不能依赖该 cache 做增量同步或恢复。

---

## 2. DomainMapper 的三阶段实现

`DomainMapper.scan()` 可以抽象为：

```text
Phase 1 Host Discovery
    -> Phase 2 Per-host Scanning
        -> Phase 3 Normalize / Dedup / Head / Score / Limit
```

### 2.1 Phase 1：Host Discovery

`_discover_hosts()` 从以下途径发现 hostname：

```text
base domain
crt.sh Certificate Transparency
Wayback CDX
Common Crawl via AsyncUrlSeeder
common-subdomain DNS guessing
```

如果 `include_subdomains=false`，则只扫描传入的 exact host。

当启用多个来源时，源码用 `asyncio.wait_for()` 给每个 discovery source 加独立 timeout，再用 `asyncio.gather(..., return_exceptions=True)` 并行执行。某个来源失败不会让其他来源整体失败。

这是一个非常值得平台吸收的设计：

```text
Sitemap Provider timeout != Source failure
Wayback Provider timeout != Backfill failure
Feed Provider failure != URL Inventory rollback
```

平台中应该一一映射为独立 `provider_run`，失败原因和 Coverage Gap 分开记录。

### 2.2 Host validation 不是“内容 host 已确认”

`_validate_hosts()` 对每个候选 host 依次尝试 HTTPS、HTTP HEAD。源码中的关键语义是：

- 请求只要成功返回 Response，就把 host 返回；
- 并未要求必须 2xx；
- 301/302/303/307/308 会记录 redirect target，但原 host 仍被保留；
- `follow_redirects=False`；
- 只有异常才继续尝试下一 scheme。

因此它得到的是“网络层能收到 HTTP 响应”的近似信号，不等价于：

```text
这是内容站
这是允许抓取的 host
首页有效
当前 origin 是 HTTPS
这个 host 属于同一知识库 Source
```

平台应拆出更严格的状态：

```text
DISCOVERED_METADATA_ONLY
REACHABLE
REDIRECTED
BLOCKED
HTTP_ERROR
UNRESOLVED
APPROVED_CONTENT
REJECTED
```

尤其是 CT/DNS 发现的 host，**在人工或规则批准前不应让 DomainMapper 自动做主动 HEAD validation**。如果需要真正的 metadata-only 子域发现，应使用独立 CT/Archive Adapter 先生成 `host_candidate`，而不是直接调用全域 `DomainMapper.scan()`。

### 2.3 Phase 2：Per-host Scanning

每个 host 先建立 Soft-404 fingerprint，然后并行执行部分来源：

```text
robots.txt
sitemap
probe
feed
homepage
wayback URL merge
```

其中：

- `robots.txt` 同时提供 `Sitemap:` 和 path 信息；
- Sitemap 复用 `AsyncUrlSeeder` 解析；
- Probe 默认包含通用产品/API/登录/管理类路径；
- Feed 在一组常见 feed path 上尝试，发现一个可用 Feed 后即停止继续探测；
- Homepage 使用普通 HTTP 客户端抓首页并提取内部链接及 `<head>` link；
- Wayback 在 Phase 1 保存 `_wayback_urls`，Phase 2 再按 host 合并。

对博客知识库而言，运行时 Provider 之间的并行和错误隔离有价值，但来源语义必须重新定义，不能把所有来源都视为“文章 URL 的同等级证明”。

### 2.4 Common Crawl 在当前 DomainMapper 中更像 host discovery signal

当前 `_discover_via_cc()` 会：

1. 通过 `AsyncUrlSeeder` 取得 Common Crawl 结果；
2. 从结果 URL 提取 hostname；
3. 将 hostname 加入 host set。

本次源码检查没有看到 `_cc_urls` 或类似结构，也没有看到 Phase 2 将 Common Crawl URL 集合按 host 并入最终结果的对称逻辑；Wayback 则明确有 `_wayback_urls`。

因此生产架构不能假定：

```text
DomainMapper(source="cc") == Common Crawl 历史 URL 全量 Provider
```

正确做法仍是独立 `common_crawl_gap` Provider，显式遍历索引/时间范围、持久化查询证据、URL Observation 和 cursor。DomainMapper 的 CC 能力最多作为域名结构/host 候选信号以及小规模辅助 discovery。

---

## 3. 八类发现源在博客知识库中的角色

官方文档列出：

```text
sitemap
cc
wayback
crt
probe
robots
feed
homepage
```

它们在知识库里的业务角色不同。

### 3.1 Sitemap：Authoritative Provider

Sitemap 是历史 Inventory 的主要入口之一。生产实现应独立支持：

- sitemap index 递归；
- gzip；
- robots `Sitemap:` 多条声明；
- 默认 sitemap 路径补探测；
- `lastmod`；
- ETag / Last-Modified；
- cursor；
- 多 sitemap 并集；
- Provider Evidence。

DomainMapper 更适合接入期探测，不适合替代长期 Sitemap Provider。

### 3.2 Common Crawl：Historical Gap Provider

Common Crawl 用于发现当前站点已不再链接的旧 URL，但不能证明历史完整，也不能证明 URL 当前有效。

平台保存：

```text
archive_index
query/window
original_url
first_seen/last_seen if available
provider_run_id
evidence_object_key
```

对于全历史需求，不能只依赖最新一个 CC index；应由独立 Provider 明确管理索引范围和恢复点。

### 3.3 Wayback：Historical Gap Provider

DomainMapper 对 `*.domain/*` 查询 Wayback CDX，当前源码使用 `collapse=urlkey` 和单次 `limit=10000`。

这意味着单次 DomainMapper 结果不能证明大站历史完整。平台需要：

```text
year/time window
URL prefix partition
CDX page/resume key
independent provider_run
cursor/checkpoint
```

更关键的是，DomainMapper 将 Wayback URL 加入结果时会填 `status="valid"`，但这只是结果对象的运行时字段，并不是逐 URL 回源验证结论。平台必须忽略这个字段作为业务有效性证据，统一进入自己的 URL Validity Gate。

### 3.4 Certificate Transparency：Host Candidate Signal

CT 很适合发现 `blog.example.com`、`engineering.example.com`、`docs.example.com`，但同样会出现：

```text
admin
staging
api
dev
internal-like
```

正确流程：

```text
CT observation
 -> host_candidate
 -> scope classifier / human review
 -> APPROVED_CONTENT
 -> active HTTP discovery/fetch
```

不能把证书里出现过的域名自动加入 active crawl scope。

### 3.5 Probe：必须替换默认 Path Set

默认 `DEFAULT_PROBE_PATHS` 包含：

```text
/docs
/api
/login
/dashboard
/blog
/openapi.json
/swagger.json
/graphql
/status
/health
...
```

这不适合作为第三方技术博客的默认探测面。

平台应编译自己的 `ContentProbeProfile`，只允许低风险内容路径：

```text
/blog
/posts
/articles
/news
/engineering
/tech
/archive
/archives
/feed
/rss
```

且 Probe 只能在 `APPROVED_CONTENT` host 上运行。

### 3.6 robots：Access Policy，而不是“隐藏 URL 列表”

DomainMapper 会读取 `Disallow:` / `Allow:` 并把 concrete path 加入 probe paths。

对知识库必须改义：

```text
Sitemap:  -> discovery evidence
Allow:    -> access policy input
Disallow: -> deny/restriction input
```

`Disallow` 绝不能变成主动探测入口。

### 3.7 Feed：Incremental First-class Provider

RSS/Atom 非常适合低成本增量同步，但通常只覆盖最近 N 篇，不能承担全历史证明。

DomainMapper 的 Feed discovery 在找到一个可用 Feed 后就停止继续常见路径探测，因此多 Feed / 分类 Feed 站点仍应由独立 Feed Provider 完整管理。

### 3.8 Homepage：浅层结构信号

Homepage source 会提取：

- 普通内部 `<a href>`；
- `<head>` 中 `alternate`、`preload`、`prefetch`、`next`、`prev` 等 link。

适合发现最新文章、分页入口、Feed 和内容导航，但不构成全历史覆盖证明。

---

## 4. Soft-404 的实现原理与平台增强

### 4.1 当前实现

每个 host 先访问随机不存在路径：

```text
/c4ai-probe-<random>
```

记录：

```text
status_code
title
content_length
body_hash = md5(first 2048 bytes)
```

如果 fingerprint 的最终状态不是 200，源码会放弃 Soft-404 fingerprint；如果是 200，则后续 `_is_soft_404()` 主要比较：

1. 目标是否 HTTP 200；
2. 前 2048 bytes MD5 是否完全相同；
3. `<title>` 是否完全相同。

值得注意的是：fingerprint 虽保存 `content_length`，当前 `_is_soft_404()` 并没有使用它参与判定。

### 4.2 Sitemap 批量 Soft-404 优化

当 Sitemap URL 数大于 5 且 host 有 Soft-404 fingerprint 时，源码随机抽样最多 5 个 URL。如果样本全部判 Soft-404，则直接跳过这一批 Sitemap 结果。

对于通用侦察工具这是有效降噪，但对需要审计 Coverage 的平台风险较大：

```text
已发现 URL 证据
!=
允许进入 Fetch Queue
```

正确做法：

```text
Persist all sitemap observations
 -> mark batch SOFT_404_SUSPECTED
 -> lower/defer admission
 -> expand sampling / validate individually
 -> human review if needed
```

**可以延迟抓取，不能删除已发现证据。**

### 4.3 平台 Soft-404 Profile

只比较 2KB hash 和 exact title 无法覆盖动态 request-id、时间戳、A/B 文案、统一站点 title 等情况，也可能因为多个合法页面 title 相同而误报。

建议每个 host 使用 2~3 个随机不存在 URL 建立模板簇，特征：

```text
status_code
redirect_chain
normalized_title
normalized_text_simhash
content_length_bucket
dom_structure_hash
canonical_target
main_text_length
known_error_tokens
```

判定：

```text
VALID
SOFT_404_SUSPECTED
SOFT_404_HIGH_CONFIDENCE
UNKNOWN
```

Validity Profile 应有版本和 TTL，站点模板变化后重新采样。

### 4.4 HTTPS-only 构造带来的边界

当前 `_fingerprint_soft_404()`、`_probe_paths()`、`_discover_feeds()`、`_scan_homepage()` 多处直接使用 `https://{host}` 构造 URL。

因此平台不能把 host 只存为 hostname；至少要维护：

```text
preferred_scheme
canonical_origin
redirect_origin
last_origin_probe_at
```

对于 HTTP-only、HTTPS 配置异常、特殊反代站点，平台应由 Exact-host Probe 先确定 origin，再决定是否调用 DomainMapper 的对应能力。

---

## 5. URL Identity、Dedup 与“valid”语义

### 5.1 DomainMapper 的 dedup 不能作为业务 URL Identity

`_normalize_and_dedup()` 会先 `normalize_url()`，随后使用：

```text
normalized.rstrip("/").lower()
```

作为 dedup key。

问题在于 URL host 大小写不敏感，但 HTTP path 和 query 在很多服务端实现上可以大小写敏感。例如：

```text
/Docs/API
/docs/api
```

理论上可能是两个不同资源。把整条 URL lower-case 后做 Identity 会造成错误合并。

平台规则应是：

```text
scheme: normalize
host: lowercase / IDNA normalize
port: remove default port
fragment: strip
path: preserve case by default
query: preserve case/order before Source Query Policy proves可归一化
```

并保存三层值：

```text
raw_url
runtime_normalized_url
platform_normalized_url
```

DomainMapper 的 dedup 仅作为 runtime convenience。

### 5.2 `status="valid"` 不等于 URL 当前有效

`_scan_host()` 会对 Sitemap、Probe、Feed、Homepage 等返回结果统一创建类似：

```text
status = "valid"
```

Wayback URL 也直接以 `valid` 加入结果。如果 `extract_head=false`，大量 URL 不会发生后续逐 URL head fetch。

因此平台必须把 DomainMapper result status 解释为：

```text
runtime discovery status
```

而不是：

```text
business URL validity
```

所有候选仍进入统一的：

```text
URL Observation
 -> Scope Check
 -> URL Validity Gate
 -> Fetch Admission
```

### 5.3 BM25、score_threshold、max_urls 只能影响优先级

DomainMapper 可以提取 `<head>` 后做 BM25，并使用 `score_threshold` 过滤结果；最后还可以 `max_urls` 截断。

这适合：

- UI 预览；
- focused discovery job；
- 固定预算的人工检索。

不适合全历史 Backfill，因为：

- 低相关性不等于不是文章；
- `max_urls` 是运行时截断，不是可解释 Coverage 边界；
- host 来源是 set，聚合顺序不应被当成稳定业务排序；
- 截断后未返回 URL 不会自动进入平台 Frontier。

全历史模式应固定：

```text
query = None
score_threshold = None
max_urls = -1
```

如果需要打分，先持久化 Observation，再把 score 映射为 scheduler priority。

---

## 6. 并发、资源保护与限流

### 6.1 DomainMapper 的并发结构

源码中有多层 `asyncio.gather()`：

- host discovery sources 并行；
- candidate hosts validation 并行；
- 所有 validated hosts `_scan_host()` 并行；
- probe path 并行；
- DNS prefix 并行；
- `<head>` 提取使用 `config.concurrency` semaphore。

因此 `config.concurrency` 不能被理解为整个 DomainMapper 的唯一全局并发上限。尤其当自定义 `common_subdomains`、`probe_paths` 很大或一个根域发现大量 host 时，平台外层仍必须做 Slice 和 admission。

推荐：

```text
Global Fair Scheduler
 -> Source/Domain Token Bucket
 -> bounded DomainRecon host slice
 -> bounded probe path set
 -> DomainMapper runtime
 -> persist result-by-result/provider-by-provider
```

### 6.2 `hits_per_sec` 不是严格 RPS

DomainMapper 中：

```text
self._rate_sem = asyncio.Semaphore(config.hits_per_sec)
```

它限制的是同时进入部分请求段的并发数，而不是按时间窗口计算的请求次数。

因此：

```text
Redis Token Bucket       = 跨 Worker 真正 RPS / burst
Worker Semaphore         = 本地在途并发保护
Crawler RateLimiter      = 429/503 自适应退避
MemoryAdaptiveDispatcher = Browser Worker 内存保护
```

四者职责必须分开。

### 6.3 `extract_head=True` 的成本

`_extract_heads()` 会为结果集合逐 URL 拉取 head/HTML metadata，文档也明确提示这一选项会增加约 1 次请求/URL。

1000 站全历史发现时，如果先发现百万 URL，再开启 `extract_head=True`，会直接把“低成本 URL inventory”变成第二轮大规模网络抓取。

推荐：

```text
Onboarding preview: extract_head=true on small sample
Bulk backfill inventory: extract_head=false
Focused UI search: extract_head=true + query
Canonical fetch: metadata from Raw Artifact / normal Fetch Plane
```

---

## 7. Cache、配置字段与版本漂移

### 7.1 本地 cache 不能作为增量同步真相

`domain_mapper.py` 定义了：

```text
~/.crawl4ai/domain_mapper_cache
_cache_key()
_read_scan_cache()
_write_scan_cache()
cache_ttl_hours
```

但在本次检查的 `scan()` 主路径中，没有看到这些 helper 被用作业务恢复或 Provider cursor。即使未来版本重新接入，也存在：

- 多 Worker 不共享；
- Worker 替换即丢失；
- cache key 不能表达平台全部 Source/Policy Release；
- TTL 与增量同步业务状态不是同一概念。

所以平台只认：

```text
provider_run
provider_cursor
provider_result_artifact
url_observation
coverage_snapshot
```

### 7.2 配置名不等于运行时能力已经生效

官方配置表包含 `use_browser_for_homepage`。本次源码搜索中，该字段能在文档和 `async_configs.py` 中找到，但在当前 `domain_mapper.py` 的 homepage 执行路径中没有看到对应分支，`_scan_homepage()` 仍直接使用 HTTP client。

这说明集成第三方 crawler 时不能只依赖配置文档；每次 runtime release 都应通过 Golden Test 验证：

```text
config -> actual network behavior
config -> actual filtering behavior
config -> actual browser/http route
```

若版本升级改变行为，必须发布新的 `runtime_release` / `provider_adapter_release`。

### 7.3 source_timeout 是 Recon Budget，不是 Coverage Completion

DomainMapper 给每个来源独立 timeout，这对交互式 scan 很合理。但大 Sitemap、Wayback、慢 Feed 在 timeout 时“跳过来源”不表示 Provider 已完整执行。

平台应把结果记录为：

```text
TIMED_OUT / PARTIAL
```

而不是 `COMPLETE`，并让独立 Provider 继续分页/cursor 恢复。

---

## 8. 对现有博客知识库方案的优化

### 8.1 DomainReconAdapter 的固定职责

DomainMapper 只通过平台 `DomainReconAdapter` 调用，Adapter 负责：

```text
validate_approved_scope()
resolve_canonical_origin()
build_safe_sources()
build_safe_probe_paths()
build_runtime_config()
run_bounded_scan()
expand_source_attribution()
emit_raw_observations()
classify_runtime_filter_reason()
ignore_runtime_valid_as_business_truth()
translate_errors/timeouts()
```

### 8.2 推荐 runtime profile

全历史/结构勘探默认：

```yaml
include_subdomains: false
extract_head: false
query: null
score_threshold: null
max_urls: -1
soft_404_detection: true
```

但 `soft_404_detection` 只作 runtime advisory，最终判定仍由平台 `host_validity_profile` 完成。

`filter_nonsense_urls` 按 Source 内容类型编译：

- 纯 HTML 博客可开启运行时 asset 过滤；
- PDF-heavy / 文档型 Source 应关闭或绕过上游 blanket extension filter，由平台将 URL 分类为 `CONTENT_HTML` / `CONTENT_PDF` / `ASSET`；
- Filter reason 必须可解释。

### 8.3 独立 Authoritative/Historical Provider

DomainMapper 不能替代：

```text
sitemap
rss_atom
common_crawl_gap
wayback_gap
```

原因包括：cursor、分页、历史窗口、Provider Evidence、恢复、CC URL 级行为、Wayback 10k 单查询边界和 DomainMapper timeout。

### 8.4 URL Identity 增强

新增规则：

```text
lowercase only scheme/host
preserve path/query case by default
runtime_normalized_url != platform_normalized_url
query normalization requires Source-specific release
```

并把大小写敏感 URL 对加入 Golden Fixture。

### 8.5 Host Candidate 安全边界

子域 discovery 分两步：

```text
Metadata-only CT/Archive/DNS evidence
 -> host_candidate
 -> approval
 -> active DomainMapper exact-host scan
```

不要在未批准根域上直接用 DomainMapper 自动发现后 active validate 所有子域。

### 8.6 URL Validity 与 runtime status 分离

新增平台约束：

```text
DomainMapper status=valid
!=
url_validity_observation.decision=VALID
```

Wayback、Sitemap、Feed、Homepage 结果统一走平台 Validity Gate。

### 8.7 Worker 执行层

```text
平台 bounded slice
 -> DomainMapper / arun_many
 -> Worker local semaphore / MemoryAdaptiveDispatcher
 -> result-by-result durable commit
```

不把百万 URL 一次交给 `asyncio.gather()` 或 `arun_many()`。

---

## 9. 安全与合规

Crawl4AI v0.8.7 发布记录包含 Docker API 的多项安全加固，后续 v0.9.x 又进一步强化 secure-by-default。对本方案的含义是：Crawler runtime 必须被当作高权限网络执行组件，而不是普通无状态 SDK。

要求：

1. Runtime 版本 pin 到经过测试的 immutable release；
2. Crawler API 只在受控内网使用，不直接暴露公网；
3. Web 用户不能透传任意 Domain、URL、CrawlerRunConfig、JS hook；
4. 所有目标先做 approved scope + SSRF destination validation；
5. 禁止 loopback、RFC1918、link-local、云 metadata、`file://` 等未授权目的地；
6. Redirect 每一跳都重新校验目的地和 approved scope；
7. `robots.txt Disallow` 只作约束，不变成 Probe 候选；
8. CT/DNS/Wayback 发现的 host 未批准前不做主动正文抓取；
9. Probe path 使用内容型 allowlist，不允许管理/登录/调试/健康检查类默认路径；
10. Browser、下载、execute-js、webhook 等高风险能力按 Recipe allowlist 最小化启用。

---

## 10. 必须增加的 Golden / Regression Tests

针对 DomainMapper Adapter 至少覆盖：

```text
1. /Docs/A 与 /docs/a 不被错误合并
2. query 大小写/顺序在未发布 Query Policy 前不被改写
3. Wayback result status=valid 不绕过平台 Validity Gate
4. Common Crawl 独立 Provider 能产生 URL-level Observation
5. 未批准 CT 子域不会触发 active HTTP probe
6. robots Disallow 不进入 Probe
7. Soft-404 动态 request-id 仍能被模板相似度识别
8. 合法页面同 title 不因 title 相同直接判 Soft-404
9. Sitemap Soft-404 批次保留全部 Observation
10. HTTP-only origin 不因 DomainMapper HTTPS 默认路径静默丢失
11. PDF-heavy Source 不被 blanket nonsense filter 丢掉内容文档
12. extract_head=false 时不把 runtime status 当当前有效性
13. max_urls / score_threshold 只允许 focused job，不进入 Backfill Profile
14. hits_per_sec 不作为严格 RPS 验收指标
15. 大量 host/probe path 时外层 Slice 能限制并发
16. source_timeout 产生 PARTIAL/TIMED_OUT，而不是 COMPLETE
17. runtime release 升级后 use_browser_for_homepage 等关键配置做行为回归
```

---

## 11. 最终建议

DomainMapper 在最终架构中放两个位置最合适：

```text
A. Source Onboarding Recon
   exact-host Sitemap / Feed / Homepage / Safe Probe / Soft-404 baseline
   metadata-only host candidate 由独立 Adapter 先完成

B. Coverage Gap Recon
   在 approved scope 内补站点结构盲区
   不能替代独立 Common Crawl / Wayback / Sitemap / Feed Provider
```

生产主链保持：

```text
Provider / DomainRecon Runtime
 -> Raw Discovery Evidence
 -> PostgreSQL URL Observation
 -> Platform URL Normalization / Identity
 -> Approved Scope
 -> URL Validity Gate
 -> Persistent Frontier / Admission
 -> HTTP First / Browser Slow Path
 -> Immutable Artifact
 -> Extraction Portfolio
 -> Quality Gate
 -> Canonical IR
 -> Markdown Projection
```

本次调研对技术方案的核心增强可以概括为：

> **把 Domain Mapping 定位为受控 Recon，而不是 Coverage Truth；把 Soft-404 变成版本化的主机有效性模型；把上游 dedup、status、score、limit、timeout、cache、rate 参数全部降级为运行时提示；URL Identity、当前有效性、跨 Worker 限流、Coverage 和恢复全部由平台持久化模型负责。特别需要防止完整 URL lower-case 去重、Wayback “valid” 误解、Common Crawl URL-level 覆盖假设以及未批准子域的主动探测。**
