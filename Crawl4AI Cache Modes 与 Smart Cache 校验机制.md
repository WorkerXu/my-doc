# Crawl4AI Cache Modes 与 Smart Cache 校验机制

## 1. 调研对象

- 官方文档：https://docs.crawl4ai.com/core/cache-modes/
- 官方项目：https://github.com/unclecode/crawl4ai
- 本次源码核对版本：`7e801521428ee12509994d39151006f64055ebe3`
- 重点源码：
  - `crawl4ai/cache_context.py`
  - `crawl4ai/async_webcrawler.py`
  - `crawl4ai/cache_validator.py`
  - `crawl4ai/async_database.py`
  - `crawl4ai/utils.py`

本调研只关注与“1000+ 技术博客全历史抓取、持续增量同步、浏览器成本控制、结果可证明性”直接相关的缓存与变化检测机制。

---

## 2. 结论先行

Crawl4AI 的 CacheMode 和 Smart Cache 适合作为 **Runtime 执行层的性能优化器**，不适合直接承担知识库的版本真相与增量同步判定。

最重要的结论有五个：

1. `CacheMode` 本质上只是“是否读/写 Crawl4AI 本地缓存”的执行策略，并不是平台级文档版本模型；
2. 当前源码中的 Smart Cache 会用 `ETag`、`Last-Modified` 和 `<head>` 指纹做轻量校验，能够避免一部分不必要的浏览器完整抓取；
3. `<head>` 指纹只覆盖 title、description、OpenGraph 与 modified-time 等有限元数据，**正文发生变化但 head 不变时可能被错误判为 FRESH**；
4. Smart Cache 校验请求异常时，`AsyncWebCrawler.arun()` 会把旧缓存标记为 `hit_fallback` 并继续返回，因此“拿到了内容”不等于“已验证内容仍然新鲜”；
5. 官方文档与具体 release 的缓存语义存在表述差异风险，例如 `BYPASS` 是否写缓存不能只按文字理解，生产系统必须 pin 版本并通过 Contract Test 验证行为。

因此对博客知识库的正确用法是：**平台自己维护 Revalidation Ledger、Raw Artifact 与 Document Version，Crawl4AI cache 只能作为可丢弃的执行缓存；权威增量检测优先使用 HTTP Conditional GET/Provider 变化信号，Smart Cache 只用于浏览器慢路径前的预检。**

---

## 3. CacheMode 的实现原理

### 3.1 五种模式

`cache_context.py` 定义：

```text
ENABLED     正常读写
DISABLED    不读不写
READ_ONLY   只读
WRITE_ONLY  只写
BYPASS      绕过缓存
```

真正决定行为的是 `CacheContext.should_read()` 与 `should_write()`：

```text
should_read  -> ENABLED / READ_ONLY
should_write -> ENABLED / WRITE_ONLY
```

此外：

- `always_bypass=True` 时直接禁止读写；
- `http://`、`https://`、`file://` 被认为可缓存；
- `raw:` 输入不可缓存。

这个实现说明缓存策略与 URL 类型判断集中在 `CacheContext`，优点是调用方不用散落多个 boolean；但同时也说明它是一个 **运行时本地 I/O 决策层**，没有 Source、Document、Version、Provider、Coverage 等业务语义。

### 3.2 旧参数迁移

源码 `_legacy_to_cache_mode()` 把旧参数映射到枚举：

```text
disable_cache                -> DISABLED
bypass_cache                 -> BYPASS
no_cache_read + no_cache_write -> DISABLED
no_cache_read                -> WRITE_ONLY
no_cache_write               -> READ_ONLY
默认                         -> ENABLED
```

这对长期运行系统有两个启示：

1. Runtime 升级时必须做配置兼容测试，不能假定旧 boolean 与新 enum 永久等价；
2. 平台的 Source Profile 不应直接暴露 Crawl4AI 原生参数，而应先编译成平台自己的 `fetch_intent`，再由版本化 Runtime Config Compiler 映射到具体 Crawl4AI 参数。

---

## 4. Smart Cache 的调用链

当前 `AsyncWebCrawler.arun()` 的核心顺序可以概括为：

```text
CrawlerRunConfig
      |
      v
CacheContext
      |
      +-- should_read? -- yes --> async_db_manager.aget_cached_url(url)
      |                              |
      |                              v
      |                    cached CrawlResult
      |                              |
      |                  check_cache_freshness?
      |                              |
      |                              v
      |                        CacheValidator
      |                       /      |      \
      |                    FRESH   STALE   ERROR
      |                      |       |       |
      |                  用缓存   丢缓存   用旧缓存
      |                                  hit_fallback
      |
      +-- 无有效缓存 --> 真正网络/浏览器抓取
```

源码中若 `config.cache_mode is None`，`arun()` 会补成 `ENABLED`；实际默认值仍应以 pin 的 `CrawlerRunConfig` release 为准，不能仅依赖文档摘要。

---

## 5. Smart Cache 校验算法

### 5.1 第一层：HTTP 条件头 + HEAD

`CacheValidator.validate()` 若历史缓存带有：

- `ETag`
- `Last-Modified`

会构造：

```http
If-None-Match: <stored-etag>
If-Modified-Since: <stored-last-modified>
```

然后发送 `HEAD`。

判定：

- 返回 `304`：`FRESH`；
- 返回 `200`：继续结合 `<head>` 指纹；
- 没有可验证元数据：进入下一层。

这比每次拉起 Playwright 浏览器便宜得多，很适合作为“是否需要 Browser Slow Lane”的前置探测。

### 5.2 第二层：只下载 `<head>` 并计算指纹

如果保存过 `head_fingerprint`，Validator 会用 `httpx` 流式 GET 页面：

- `Accept-Encoding: identity`；
- 每次读取约 4KB；
- 读到 `</head>` 即停止；
- 最大读取 64KB。

随后调用 `compute_head_fingerprint()`。

当前指纹只组合这些信号：

```text
<title>
<meta name="description">
<meta name="last-modified">
<meta property="og:title">
<meta property="og:description">
<meta property="og:image">
<meta property="og:updated_time">
<meta property="article:modified_time">
```

最终以 `xxhash.xxh64()` 生成 hash。

这是一个成本很低的“页面元数据是否变化”探针，但不是正文 hash。

### 5.3 第三层：返回 UNKNOWN

如果没有 ETag、Last-Modified、head fingerprint，则返回：

```text
UNKNOWN -> 需要重新抓取
```

在 `AsyncWebCrawler.arun()` 里，`STALE` 与 `UNKNOWN` 都会清掉 `cached_result`，随后进入完整抓取。

### 5.4 ERROR 的特殊行为

如果校验发生 Timeout、RequestError 或其他异常，`CacheValidator` 返回 `ERROR`。

`AsyncWebCrawler.arun()` 对 ERROR 的处理不是强制重新抓取，而是：

```text
cached_result.cache_status = "hit_fallback"
继续使用旧缓存
```

这是一个面向“可用性优先”的 Runtime 设计，但与知识库“新鲜度可证明”的目标存在冲突。

平台必须把：

```text
FRESH_VERIFIED
CACHE_HIT_UNVERIFIED
VALIDATION_ERROR_CACHE_SERVED
STALE_RECRAWLED
UNKNOWN_RECRAWLED
```

分成不同业务状态，绝不能把 `hit_fallback` 统计成增量同步成功。

---

## 6. 本地缓存持久化模型

`async_database.py` 使用 `.crawl4ai/crawl4ai.db` 与内容文件目录保存缓存。

Smart Cache 增加了这些字段：

```text
etag
last_modified
head_fingerprint
cached_at
```

`aget_cache_metadata()` 可以只读取校验元数据，避免先加载全部正文；完整内容则通过内容 hash 关联到本地文件。

这套设计适合单 Runtime/节点本地加速，但对于分布式博客平台存在明显边界：

1. 本地 SQLite 不是跨 Worker 的业务真相；
2. Runtime 重建、容器漂移、缓存清理后可能消失；
3. 同一 URL 的缓存记录没有平台级 Source/Profile/Release/Attempt 身份；
4. 无法承担百万级文档版本链、审计、回滚和多 Release 重放。

所以平台必须把重要结果晋升到 PostgreSQL + Object Storage，Runtime cache 可以随时删除并重建。

---

## 7. 对 1000+ 技术博客增量同步的价值

### 7.1 最大价值：减少 Browser Slow Lane

大量技术博客是静态站、SSR 或服务端生成页面，正文更新频率远低于抓取频率。如果每轮同步都进入 Chromium：

- CPU/内存成本高；
- Browser Pool 压力大；
- 同域名并发更难控制；
- 任务 TTFM 和 Queue Age 增长。

Smart Cache 的思想可以抽象成：

```text
先用便宜信号判断“值得不值得做昂贵抓取”
```

因此应保留这个思路，但把实现升级成平台级 `Revalidation Engine`。

### 7.2 不应把 Crawl4AI 本地缓存直接当增量源

对于增量同步，平台关心的是：

```text
页面是否真的发生业务意义上的变化？
是否生成新的 Raw Artifact？
是否应产生新的 Document Version？
是否需要重跑 Cleaner / Extractor / Markdown？
```

而 Crawl4AI CacheMode 只回答：

```text
这一次 Runtime 是否读/写自己的缓存？
```

两者层级不同。

---

## 8. 关键风险与误判场景

### 8.1 正文变化但 `<head>` 不变

这是最重要的风险。

例如文章修正文中的代码：

```text
<title> 不变
og:title 不变
og:description 不变
article:modified_time 未维护
正文从 v1 改成 v2
```

head fingerprint 完全可能不变，于是 Validator 返回 `FRESH`，旧缓存继续被使用。

因此：

**head fingerprint 只能是 weak evidence，不能单独证明正文未变化。**

### 8.2 HEAD 与 GET 语义不一致

现实网站可能：

- 不支持 HEAD；
- CDN 对 HEAD 与 GET 使用不同缓存策略；
- 返回错误/陈旧的 ETag；
- `Last-Modified` 精度不足；
- 页面个性化导致 Validator 与 Browser 获取不同内容。

所以生产平台的权威轻量校验优先采用标准 Conditional GET，并记录原始响应证据；HEAD 可作为更便宜的前置探测，但不能成为唯一机制。

### 8.3 校验失败时回退旧缓存

对普通爬虫这有利于可用性，对知识库则容易制造“静默陈旧”。

正确做法：

```text
validation error
    -> 可以服务旧 Markdown 给读请求
    -> 但 Source Freshness 状态必须 DEGRADED
    -> Revalidation Task 必须进入重试队列
    -> 不能推进 last_verified_at
```

### 8.4 CacheMode 语义依赖 release

源码 `CacheContext.should_write()` 当前只允许 `ENABLED` / `WRITE_ONLY`，因此 `BYPASS` 在这一层并不会写缓存。

官方不同文档页面对 BYPASS 的文字存在“绕过”与“是否仍写入”的描述差异。这说明平台必须遵守：

```text
Behavior Beats Naming
```

即：

- pin package version / image digest；
- 建 Runtime Contract Test；
- 升级先灰度；
- Web 后台显示运行时实际 release 与契约结果。

---

## 9. 对现有博客知识库方案的优化建议

### 9.1 新增平台级 Revalidation Engine

建议在 Fetch Plane 前增加：

```text
Incremental Scheduler
        |
        v
Revalidation Engine
        |
        +-- Provider changed signal? --------> FRESH FETCH
        |
        +-- Conditional GET 304 ------------> UNCHANGED
        |
        +-- HTTP validator changed ----------> FRESH FETCH
        |
        +-- weak head fingerprint only ------> SAMPLE / POLICY
        |
        +-- validation error ----------------> DEGRADED + RETRY
        |
        +-- unknown --------------------------> FRESH FETCH
```

### 9.2 Revalidation Ledger

建议 PostgreSQL 增加/明确以下字段：

```text
url_id
last_fetch_at
last_verified_at
last_changed_at
etag
last_modified
raw_body_hash
semantic_body_hash
head_fingerprint
validator_strength
validation_method
validation_result
cache_served
runtime_cache_status
consecutive_304
forced_refresh_due_at
validation_error_count
source_validator_trust_score
```

### 9.3 强弱证据分级

建议：

```text
STRONG
- CMS/API version id
- Content revision id
- Conditional GET 304（服务端语义可靠时）
- 新 GET body hash 与旧 hash 相同

MEDIUM
- ETag/Last-Modified 变化
- Sitemap lastmod 变化
- RSS updated 变化

WEAK
- head fingerprint 相同
- 页面长度相近
- link graph 相似
```

只有满足 Source Policy 的强证据才能推进 `last_verified_at`。

### 9.4 强制定期完整抽样

即使连续返回 304/head fingerprint 相同，也应设置：

```text
forced_full_refresh_interval
forced_sample_rate
```

定期绕过弱校验抓取完整正文，计算漏检率。

如果某 Source 出现：

```text
validator says unchanged
但 full sample body hash changed
```

立即降低 `source_validator_trust_score`，后续缩短强制刷新周期。

### 9.5 Cache Policy Compiler

不要让普通 Source 配置直接选择 Crawl4AI CacheMode，改成平台语义：

```yaml
fetch_intent: validate_then_fetch
freshness_class: normal
serve_stale_on_error: true
advance_verified_at_on_stale_cache: false
forced_refresh_interval: 7d
```

Runtime Config Compiler 再根据 pinned release 转换成 Crawl4AI 配置。

建议平台内部定义：

```text
AUTHORITATIVE_FETCH
VALIDATE_THEN_FETCH
REPLAY_FROM_ARTIFACT
RUNTIME_CACHE_WARMUP
DIAGNOSTIC_CACHE_READ
```

比直接暴露 `ENABLED/BYPASS/...` 更稳定。

### 9.6 Runtime Cache 与业务 Artifact 严格分离

Crawl4AI SQLite/cache：

```text
可丢弃、可重建、不可证明业务版本
```

平台 Raw Artifact：

```text
不可变、content-addressed、带抓取时间/headers/release/attempt/provenance
```

任何通过 runtime cache 返回的正文若要进入新的 Document Version，仍必须通过平台自己的 provenance 和 hash 规则。

---

## 10. 推荐的增量状态机

```text
DISCOVERED
   |
   v
DUE_FOR_REVALIDATION
   |
   +-- provider says new/changed --> FETCH_REQUIRED
   |
   +-- conditional GET 304 -------> VERIFIED_UNCHANGED
   |
   +-- validator changed ---------> FETCH_REQUIRED
   |
   +-- weak-evidence unchanged ---> WEAK_UNCHANGED
   |                                  |
   |                                  +-- sample due --> FETCH_REQUIRED
   |
   +-- validation error ----------> VALIDATION_DEGRADED
                                      |
                                      +-- serve previous version allowed
                                      +-- retry scheduled

FETCH_REQUIRED
   |
   v
RAW_ARTIFACT
   |
   +-- body hash unchanged -------> VERIFIED_UNCHANGED
   |
   +-- body hash changed ---------> NEW_DOCUMENT_VERSION
```

其中 `VALIDATION_DEGRADED` 不允许更新 `last_verified_at`。

---

## 11. Web 管理功能建议

新增一个“增量校验”视图，至少展示：

- Source 的 ETag/Last-Modified 可用率；
- 304 命中率；
- Runtime cache hit / hit_validated / hit_fallback；
- `VALIDATION_DEGRADED` 数量；
- 强制完整刷新命中变化率；
- head fingerprint 漏检率；
- Browser Avoided 数量与节省成本；
- Source Validator Trust Score；
- 最近一次强验证、弱验证、完整抓取时间；
- 一键“强制绕过全部 Runtime Cache 重新抓取”。

另外，任何 `hit_fallback` 都应在 Web 上显式标黄/标红，不能显示成普通成功。

---

## 12. 建议的 Contract Tests

每次升级 Crawl4AI 都执行：

1. `ENABLED` 首次抓取后是否写缓存；
2. `ENABLED` 第二次是否读取缓存；
3. `READ_ONLY` miss 时是否真的不写；
4. `WRITE_ONLY` 是否不读取旧缓存；
5. `DISABLED` 是否不读不写；
6. `BYPASS` 的实际读/写行为；
7. `check_cache_freshness=true` + 304 是否返回 `hit_validated`；
8. ETag 改变时是否触发完整抓取；
9. head fingerprint 变化时是否触发完整抓取；
10. Validator timeout 是否返回 `hit_fallback`；
11. 只有正文变化、head 不变时能否被测试捕获；
12. Runtime 重启/缓存丢失后是否不会破坏平台 Document Version 真相。

---

## 13. 最终判断

这篇调研对现有方案不是“把 Crawl4AI Cache 打开即可”的简单功能补充，而是明确了一个重要架构边界：

> **增量同步必须区分 Runtime Cache、Revalidation Evidence、Raw Artifact、Document Version。**

Crawl4AI Smart Cache 的 ETag/Last-Modified/head fingerprint 机制值得用于减少浏览器成本，但它的 `hit_fallback`、head-only fingerprint、release-dependent CacheMode 语义意味着平台必须自己维护更严格的 Revalidation Engine 与证据分级。

建议将这套机制正式并入博客知识库最终技术方案，重点加入：

- Revalidation Engine；
- Revalidation Ledger；
- 强/中/弱变化证据；
- Validator Trust Score；
- Forced Full Refresh Sampling；
- Cache Policy Compiler；
- `hit_fallback` 显式降级；
- Cache Contract Test 与版本灰度。
