# Crawl4AI Cache Modes 与 Smart Cache 校验机制

## 1. 调研对象与结论

调研对象：Crawl4AI 官方 `Cache Modes` 文档及当前主仓库中的缓存实现。

主要参考：

- https://docs.crawl4ai.com/core/cache-modes/
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/cache_context.py
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_webcrawler.py
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/cache_validator.py
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_database.py
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_configs.py
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/utils.py
- https://github.com/unclecode/crawl4ai/blob/main/tests/cache_validation/test_head_fingerprint.py

核心结论：**Crawl4AI Runtime Cache 适合作为执行层的性能缓存，不适合作为博客知识库的 Freshness、Version 或 Replay 真相。** 对 1000+ 站点长期同步系统，必须由平台自己维护 Revalidation Ledger、不可变 Raw Artifact、版本化处理链和 Validator 与 Artifact 的绑定关系。尤其要防止文档与源码语义漂移、URL-only 缓存键带来的跨配置污染，以及 Smart Cache 中“弱 head 信号更新强 validator”造成的陈旧正文被长期误判为 fresh。

---

## 2. CacheMode 的实现原理

Crawl4AI 从旧的多个布尔参数迁移到单一 `CacheMode` 枚举：

```text
ENABLED
DISABLED
READ_ONLY
WRITE_ONLY
BYPASS
```

当前 `cache_context.py` 中，`CacheContext` 把缓存决策收敛为两个函数：

```text
should_read()
should_write()
```

当前源码实际决策矩阵为：

| CacheMode | 读 Runtime Cache | 写 Runtime Cache |
|---|---:|---:|
| ENABLED | 是 | 是 |
| DISABLED | 否 | 否 |
| READ_ONLY | 是 | 否 |
| WRITE_ONLY | 否 | 是 |
| BYPASS | 否 | 否 |

`CacheContext` 只把 `http://`、`https://`、`file://` 判断为可缓存；`raw:` 输入本身不可缓存。

### 2.1 旧参数迁移语义

当前源码 `_legacy_to_cache_mode()` 的逻辑是：

```text
disable_cache -> DISABLED
bypass_cache -> BYPASS
no_cache_read + no_cache_write -> DISABLED
no_cache_read -> WRITE_ONLY
no_cache_write -> READ_ONLY
```

这里存在一个很重要的工程风险：**官方文档不同页面与当前源码存在语义漂移。** `core/cache-modes/` 的迁移表把 `no_cache_read` / `no_cache_write` 的映射写成了相反方向，而当前源码和完整 SDK 参考中的语义是 `no_cache_read -> WRITE_ONLY`、`no_cache_write -> READ_ONLY`。此外，部分官方参考对 `BYPASS` 是否会写回缓存的描述也与当前 `CacheContext.should_write()` 行为不一致。

因此生产系统不能根据 enum 名称或某一页文档推断行为，必须：

1. pin Crawl4AI package/image release；
2. 禁止业务配置继续使用旧布尔缓存参数；
3. Cache Policy Compiler 只生成一个明确的 `cache_mode`；
4. 每次 Runtime 升级执行读写、联网、cache hit/miss Contract Test；
5. 把实测语义摘要保存到 Runtime Registry。

---

## 3. Runtime Cache 的存储模型

当前 `async_database.py` 使用本地 SQLite `crawled_data` 表保存缓存索引，并把较大的内容通过 hash 落到文件系统。缓存记录以：

```text
url TEXT PRIMARY KEY
```

为主键，同时保存或引用：

```text
html
cleaned_html
markdown
extracted_content
media
links
metadata
screenshot
response_headers
downloaded_files
etag
last_modified
head_fingerprint
cached_at
```

这意味着 Runtime Cache 并不是单纯的“HTTP 原始响应缓存”，而是一个 **URL -> 完整 CrawlResult** 的缓存，其中还包含已经受某次 `CrawlerRunConfig`、Cleaner、Extractor、Markdown Generator 等配置影响的派生结果。

### 3.1 URL-only key 的重要后果

当前缓存键没有把下面这些维度编码进 key：

- Source/Profile release；
- Browser profile；
- User-Agent；
- locale/timezone；
- Cookie / 登录态；
- Proxy / region；
- session；
- JS 行为；
- CSS selector / target elements；
- extraction strategy；
- markdown generator / content filter；
- 其他会改变页面表示或派生输出的运行参数。

同一个 URL 如果以不同上下文抓取，Runtime Cache 存在跨上下文污染风险。例如：同一 URL 先用英文 locale 抓取并缓存，之后改为中文 locale；或者先用旧 Markdown Generator 产生结果，之后升级 extractor。只要命中 Runtime Cache，返回的可能仍是前一次完整 `CrawlResult`。

因此平台必须引入自己的 `fetch_variant_key` / `representation_key`，并遵循：

```text
同一 URL + 不同表示上下文 != 同一可复用缓存对象
```

生产建议是：

- Authoritative Fetch 默认不读 Runtime Cache；
- 若要使用 Runtime Cache，只允许 `public-default` 等经过证明的稳定表示类；
- 带 Cookie、登录态、session、动态 locale、Proxy region、定制 UA、定制 JS 的任务默认禁用 Runtime Cache；
- 若确实需要 Runtime Cache，按 `source_id + fetch_variant_key + runtime_release` 做独立 cache namespace/Runtime 实例隔离；
- 平台永远不把 Runtime 已缓存的 `cleaned_html/markdown/extracted_content` 当 canonical，派生结果应从平台 Raw Artifact 按 release 重放。

---

## 4. Smart Cache 的执行流程

Smart Cache 不是一个独立 CacheMode，而是 `CrawlerRunConfig` 中的额外 freshness 校验：

```text
check_cache_freshness: bool = False
cache_validation_timeout: float = 10.0
```

默认 `check_cache_freshness=False`。因此若仅使用默认 `CacheMode.ENABLED`，命中缓存后可能直接返回已有结果，不发生 freshness 网络校验。

### 4.1 AsyncWebCrawler 中的主流程

逻辑可概括为：

```text
config.cache_mode 为空
    -> 默认 ENABLED

CacheContext(url, cache_mode)
    |
    +-- should_read() == true
    |       -> 按 URL 读取 cached_result
    |       -> 若 check_cache_freshness=true
    |              -> 读取 etag/last_modified/head_fingerprint
    |              -> CacheValidator.validate()
    |
    +-- FRESH
    |       -> cache_status=hit_validated
    |       -> 直接返回缓存内容
    |
    +-- ERROR
    |       -> cache_status=hit_fallback
    |       -> 继续返回旧缓存
    |
    +-- STALE/UNKNOWN
    |       -> 丢弃 cached_result
    |       -> 真实抓取
    |
    +-- cache miss
            -> 真实抓取
```

真实抓取完成后，Runtime 会从完整 HTML 的 `<head>` 计算 `head_fingerprint`，在 `should_write()` 为 true 时写回 Runtime Cache。

### 4.2 CacheValidator 的多层校验

`cache_validator.py` 的目标是用轻量请求避免启动完整 Browser。

#### 第一层：条件 HEAD

如果已有 ETag 或 Last-Modified，会发送 HEAD，并带：

```text
If-None-Match
If-Modified-Since
```

- 返回 `304`：判定 `FRESH`；
- 返回 `200`：读取新的 ETag / Last-Modified，并进入 head fingerprint 比较；
- 请求异常：返回 `ERROR`。

#### 第二层：head fingerprint

当已有 `stored_head_fingerprint` 时，Validator 会再做一个流式 GET：

- `Accept-Encoding: identity`；
- 每次读取约 4KB；
- 找到 `</head>` 即停止；
- 最多读取 64KB；
- 对 `<head>` 中若干变化信号计算 fingerprint。

当前 fingerprint 重点包含：

- `<title>`；
- `meta description`；
- `last-modified`；
- `og:title`；
- `og:description`；
- `og:image`；
- `og:updated_time`；
- `article:modified_time`。

最终使用 xxhash 做快速摘要。

如果新旧 head fingerprint 相同，Runtime 判 `FRESH`；不同则判 `STALE`。

#### 第三层：无验证数据

如果既没有 ETag/Last-Modified，也没有 head fingerprint，则返回 `UNKNOWN`，调用方会进行完整重新抓取。

---

## 5. `hit_fallback` 的语义

发生 Timeout、RequestError 或 Validator 内部异常时，`CacheValidator` 返回 `ERROR`。`AsyncWebCrawler` 不会因此强制重新抓取，而是：

```text
cache_status = hit_fallback
继续使用旧 cached_result
```

这对“尽量返回内容”的交互式场景很有价值，但对知识库 Freshness 真相危险。

平台必须区分：

```text
服务可用性：可以临时服务 stale 内容
业务验证：不能因此更新 last_verified_at
```

所以 `hit_fallback` 必须：

- 写入 Revalidation Ledger；
- 增加 validation error 计数；
- 不推进 `last_verified_at`；
- 安排后续重试或 fresh fetch；
- 在 Web 后台明确显示“服务了旧版本”；
- 进入监控与告警指标。

---

## 6. Head Fingerprint 只能是弱信号

head fingerprint 的成本很低，但它不覆盖正文。

典型漏检：

```html
<head>
  <title>文章标题不变</title>
  <meta property="article:modified_time" content="站点没有更新">
</head>
<body>
  正文已经发生修改
</body>
```

这时 fingerprint 仍然一致，Smart Cache 会把缓存判为 FRESH。

因此平台证据等级应为：

```text
CMS revision / 与当前 Artifact 绑定的 304 / 完整 body hash -> STRONG
provider update signal / validator changed -> MEDIUM
head fingerprint unchanged -> WEAK
```

WEAK 证据只能用于节省抓取，不得独立关闭业务 freshness 真相。必须配合定期强制完整刷新与抽样核验。

---

## 7. 最重要的风险：Validator Promotion / Stale Validator Poisoning

当前 Smart Cache 实现里存在一个需要平台层主动隔离的风险链路。

假设：

```text
平台缓存正文 A
ETag = E1
head fingerprint = H
```

源站正文更新成 B，但 `<head>` 没变：

```text
正文 B
ETag = E2
head fingerprint = H
```

Smart Cache 可能发生：

1. 条件 HEAD 使用 E1；
2. 服务端返回 200，并给出新 ETag E2；
3. Runtime 拉取 `<head>`，发现 fingerprint 仍为 H；
4. Runtime 判定 FRESH；
5. Runtime 调用 `aupdate_cache_metadata()`，把本地缓存记录的 ETag 从 E1 更新为 E2；
6. **正文缓存仍然是 A，并没有抓取 B**；
7. 下次条件 HEAD 使用 E2；
8. 服务端对 E2 返回 304；
9. Runtime 再次把正文 A 当作经过验证的 fresh 内容。

也就是说，一个原本只有 WEAK 级别的“head 相同”判断，可能把属于新正文 B 的 validator E2 绑定到旧正文 A，随后再通过 304 把旧正文错误升级为“强验证通过”。

### 7.1 平台必须建立 Validator Binding Invariant

平台自己的 ETag/Last-Modified 必须绑定到具体 Raw Artifact：

```text
validator_artifact_id
validator_body_sha256
validator_origin
validator_observed_at
validator_binding_state
```

规则：

1. **只有与完整网络响应一起获得的 validator，才能绑定到该 Raw Artifact。**
2. HEAD/弱 fingerprint 流程发现的新 ETag，只记录为 `UNBOUND_HINT`，不能覆盖当前 authoritative validator。
3. 304 只有在请求使用的 validator 与当前 authoritative Raw Artifact 绑定时，才允许作为 STRONG unchanged 证据。
4. 若 Runtime Smart Cache 在 weak match 后更新了自己的 validator，平台不得把这个新 validator 导入业务 Revalidation Ledger 作为 authoritative validator。
5. Contract Test 必须加入“body 变化、head 不变、ETag 改变”的回归场景。

这一条对长期知识库的正确性比单纯提高 cache hit 率更重要。

---

## 8. `READ_ONLY` 不是离线模式

`READ_ONLY` 的实现只是：

```text
允许读 Runtime Cache
禁止写 Runtime Cache
```

当 URL cache miss 时，`cached_result=None`，主流程仍会继续执行真实抓取。

所以：

```text
READ_ONLY != offline
```

离线 Replay 必须从平台 S3/MinIO Raw Artifact 读取，并通过网络 egress deny 保证“不联网”。不能把 Runtime CacheMode 当安全边界。

---

## 9. 默认值风险

当前主流程在 `cache_mode` 未显式设置时会回退到 `ENABLED`，而 `check_cache_freshness` 默认是 `False`。

对一次性 SDK 示例这很方便，但对长期采集平台不应依赖默认值。否则一个配置字段遗漏就可能变成“无限期直接读旧 Runtime Cache”。

平台 Config Compiler 必须每次显式生成：

```text
cache_mode
authoritative_fetch flag / policy intent
check_cache_freshness
cache_validation_timeout
runtime_cache_namespace or disable reason
```

并把 effective config 保存到 Attempt 审计记录。

---

## 10. 对博客知识库方案的落地建议

### 10.1 业务层不要直接暴露 CacheMode

业务只暴露高层意图：

```text
AUTHORITATIVE_FETCH
VALIDATE_THEN_FETCH
REPLAY_FROM_ARTIFACT
RUNTIME_CACHE_WARMUP
DIAGNOSTIC_CACHE_READ
```

由版本化 Cache Policy Compiler 映射到具体 Runtime 配置。

### 10.2 推荐策略

#### AUTHORITATIVE_FETCH

- 禁止返回旧 Runtime Cache；
- 必须获得真实网络证据；
- 网络响应晋升平台 Raw Artifact；
- validator 与该 Artifact 的 body hash 绑定。

#### VALIDATE_THEN_FETCH

- 优先由平台 Revalidation Engine 使用自己保存的 bound validator；
- 304 必须检查 validator binding；
- weak signal 仅作调度优化；
- 确认变化后再进入 HTTP/Browser full fetch；
- Runtime Smart Cache 最多作为附加优化，不覆盖平台判定。

#### REPLAY_FROM_ARTIFACT

- 直接从 S3/MinIO Raw Artifact 重放；
- egress deny；
- 重新运行 Cleaner/Extractor/IR/Markdown；
- 不依赖 Crawl4AI 本地 cache。

#### RUNTIME_CACHE_WARMUP / DIAGNOSTIC_CACHE_READ

- 可使用 Runtime Cache；
- 不推进 Coverage、Version、Freshness；
- 不把 Runtime 派生结果写成 canonical。

### 10.3 Fetch Variant 隔离

以下任一条件存在时默认禁用共享 Runtime Cache：

- Cookie / Auth；
- session；
- proxy region；
- locale/timezone；
- UA/device profile；
- 自定义 JS；
- 页面交互状态；
- 内容 selector / extraction strategy 变化；
- markdown/filter release 变化。

如确需使用缓存，应按稳定的 `fetch_variant_key` 隔离 Runtime cache namespace。

---

## 11. 必做 Contract Test Matrix

每次升级 Crawl4AI package/image 都必须验证：

| 场景 | 必验行为 |
|---|---|
| ENABLED + hit + freshness=false | 不联网，直接 hit |
| ENABLED + hit + 304 | `hit_validated` |
| ENABLED + hit + HEAD 200 + head 相同 | Runtime 可判 fresh；平台只能记 WEAK |
| ENABLED + hit + head 变化 | 重新抓取 |
| ENABLED + hit + validation timeout | `hit_fallback`，业务验证时间不推进 |
| READ_ONLY + hit | 读 cache，不写 |
| READ_ONLY + miss | 会联网，不写 cache |
| WRITE_ONLY + hit | 不读旧 cache，真实抓取并写 |
| DISABLED | 不读、不写 |
| BYPASS | 实测读/写/联网语义，不按文档名称猜测 |
| legacy no_cache_read/no_cache_write | 验证与 pinned release 的实际映射 |
| 同 URL、不同 locale/cookie/UA | 验证是否存在跨表示污染 |
| 同 URL、不同 extractor/markdown config | 验证旧派生结果是否被错误复用 |
| body 改变、head 不变、ETag 改变 | 验证平台不会把新 validator 绑定到旧 Artifact |
| validator error | 验证 stale serving 与 business verification 分离 |

Contract Test 结果至少记录：

```text
runtime_release
image_digest
cache_mode_effective
should_read
should_write
network_called
request_method
conditional_headers
cache_status
cache_written
returned_artifact_origin
returned_content_hash
validator_used
validator_binding_state
```

---

## 12. 最终判断

Crawl4AI 的 CacheMode 把缓存读写控制从多个布尔开关收敛到一个枚举，工程上明显更清晰；Smart Cache 通过 ETag、Last-Modified、条件 HEAD 和 head fingerprint 能显著减少 Browser 全量抓取成本。

但对于需要长期正确性的博客知识库，以下边界必须坚持：

- Runtime Cache 是性能层，不是业务真相；
- 官方文档与源码可能漂移，生产语义必须由 pinned release + Contract Test 决定；
- Runtime cache 当前以 URL 为核心键，不能天然隔离不同抓取表示或处理配置；
- 缓存中的 Markdown/Extracted Content 不能替代平台可重放处理链；
- head fingerprint 是 WEAK 信号；
- `hit_fallback` 是 availability fallback，不是 freshness proof；
- validator 必须和 Raw Artifact/body hash 绑定；
- 不能允许 weak head match 更新出的新 validator 被提升为旧正文的 authoritative validator；
- `READ_ONLY` 不是离线；
- 平台必须显式编译 cache 配置，不能依赖 Runtime 默认值。

采用这些约束后，Crawl4AI Smart Cache 可以安全地作为“降低网络与 Browser 成本的执行层优化”，而不会破坏知识库的历史完整性、增量同步正确性和可重放性。