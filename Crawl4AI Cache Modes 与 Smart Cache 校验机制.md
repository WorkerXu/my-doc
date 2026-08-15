# Crawl4AI Cache Modes 与 Smart Cache 校验机制

## 1. 调研对象与结论

- 官方文档：https://docs.crawl4ai.com/core/cache-modes/
- 官方项目：https://github.com/unclecode/crawl4ai
- 本次源码核对版本：`7e801521428ee12509994d39151006f64055ebe3`
- 重点源码：
  - `crawl4ai/cache_context.py`
  - `crawl4ai/async_configs.py`
  - `crawl4ai/async_webcrawler.py`
  - `crawl4ai/cache_validator.py`
  - `crawl4ai/async_database.py`
  - `crawl4ai/utils.py`
  - `tests/cache_validation/*`

本调研只讨论与“1000+ 技术博客全历史回填、持续增量同步、浏览器成本控制、可恢复任务和版本真相”直接相关的缓存行为。

核心结论：**Crawl4AI CacheMode/Smart Cache 应定位为 Runtime 层性能优化，不应成为知识库的 Freshness、Document Version 或 Coverage 真相。** 平台应独立维护 Revalidation Ledger、Raw Artifact、Document Version 和强制刷新策略；Crawl4AI 的缓存只允许作为可丢弃执行缓存。

---

## 2. CacheMode 的真实实现语义

`cache_context.py` 定义五种模式：

```text
ENABLED
DISABLED
READ_ONLY
WRITE_ONLY
BYPASS
```

真正决定本地缓存 I/O 的是：

```python
should_read  -> ENABLED / READ_ONLY
should_write -> ENABLED / WRITE_ONLY
```

同时：

- `always_bypass=True` 时禁止读写；
- `http://`、`https://`、`file://` 可缓存；
- `raw:` 不进入该缓存路径。

因此，在本次核对的源码版本中：

| CacheMode | 读旧缓存 | 写新缓存 | 缓存未命中时是否可能联网 |
|---|---:|---:|---:|
| ENABLED | 是 | 是 | 是 |
| DISABLED | 否 | 否 | 是 |
| READ_ONLY | 是 | 否 | **是** |
| WRITE_ONLY | 否 | 是 | 是 |
| BYPASS | 否 | 否 | 是 |

### 2.1 READ_ONLY 不是“离线只读”

这是容易被名字误导的一个实现细节。

`READ_ONLY` 只表示“不把新结果写回 Crawl4AI cache”。如果本地没有命中，`AsyncWebCrawler.arun()` 后续仍会进入真实抓取。因此：

- `READ_ONLY` 不能作为“禁止联网”的安全边界；
- 真正离线重放应从平台 S3/MinIO Raw Artifact 读取，使用 `raw:`/本地派生处理路径或完全独立的 Replay Worker；
- 平台的 `REPLAY_FROM_ARTIFACT` 不能简单映射成 `CacheMode.READ_ONLY`。

### 2.2 BYPASS 的文档与源码必须以 Contract Test 仲裁

官方不同页面对 `BYPASS` 的描述存在“跳过缓存”与“跳过读取但仍可能写入”的表达差异；而本次核对版本的 `CacheContext.should_write()` 仅对 `ENABLED` 与 `WRITE_ONLY` 返回 True，所以该版本下 `BYPASS` 不读也不写。

这说明生产系统不能根据 enum 名称推导语义，必须：

```text
pin package/image digest
        +
Runtime Contract Test
        +
灰度升级
        +
行为差异审计
```

平台应坚持 **Behavior Beats Naming**。

---

## 3. 默认值：Smart Cache 并不是自动启用

本次源码的 `CrawlerRunConfig.__init__()` 默认值为：

```python
cache_mode = CacheMode.BYPASS
check_cache_freshness = False
cache_validation_timeout = 10.0
```

这有两个重要含义：

1. Smart Cache 是显式能力，不是“开启 Crawl4AI 就自动得到增量校验”；
2. 即便把 `cache_mode` 改成 `ENABLED`，如果 `check_cache_freshness=False`，命中旧缓存时会直接返回 `cache_status="hit"`，不会先验证页面是否改变。

因此平台 Config Compiler 必须同时管理：

```text
runtime_cache_mode_effective
runtime_check_cache_freshness_effective
runtime_cache_validation_timeout_effective
runtime_release
runtime_cache_contract_release
```

不能只保存一个逻辑上的 `cache_enabled=true/false`。

---

## 4. Smart Cache 调用链

Smart Cache 只在以下条件同时成立时进入：

```text
CacheContext.should_read() == true
        AND
本地 cached_result 存在
        AND
config.check_cache_freshness == true
```

核心调用链：

```text
CrawlerRunConfig
      |
      v
CacheContext
      |
      +-- should_read? -- no -------------------------> fresh fetch
      |
      +-- yes --> async_db_manager.aget_cached_url(url)
                      |
                      +-- miss ------------------------> fresh fetch
                      |
                      +-- hit + freshness off --------> cache_status=hit
                      |
                      +-- hit + freshness on
                               |
                               v
                         CacheValidator
                    +----------+----------+----------+
                    |          |          |          |
                  FRESH      STALE     UNKNOWN      ERROR
                    |          |          |          |
             hit_validated   fresh      fresh    hit_fallback
                              fetch      fetch     old cache
```

`ERROR` 路径是“可用性优先”，而不是“新鲜度优先”。这对通用爬虫合理，但对知识库增量同步必须做语义隔离。

---

## 5. CacheValidator 的具体算法

### 5.1 第一层：条件头 + HEAD

若历史缓存元数据包含：

- `ETag`
- `Last-Modified`

Validator 构造：

```http
If-None-Match: <stored-etag>
If-Modified-Since: <stored-last-modified>
```

然后发送 `HEAD`。

处理逻辑：

```text
HEAD 304 -> FRESH
HEAD 200 -> 读取新的 ETag / Last-Modified
             |
             +-- 有历史 head_fingerprint -> 再抓取 <head> 比较
             |
             +-- 无 fingerprint ----------> STALE
```

这能非常便宜地减少 Browser 完整抓取，但 **HEAD 不应成为平台级强验证的唯一依据**：现实中的 CDN、反向代理、应用服务器可能对 HEAD/GET 使用不同缓存或 validator 行为。

平台权威 Revalidation 应优先使用标准 Conditional GET，并保存响应证据。

### 5.2 第二层：流式抓取 `<head>`

如果需要比较 head fingerprint，`_fetch_head()`：

- 使用 `httpx.AsyncClient`；
- `follow_redirects=True`；
- `Accept-Encoding: identity`；
- 每批读取约 4KB；
- 遇到 `</head>` 停止；
- 最多读取 64KB；
- 只保留 head 片段计算 fingerprint。

该设计的目标是用极小带宽判断“页面元数据大概率是否变化”，从而避免昂贵 Browser 渲染。

### 5.3 Head fingerprint 覆盖什么

当前实现主要使用：

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

再生成快速 hash。

它本质是 **metadata fingerprint**，不是正文 fingerprint。

### 5.4 最危险的误判：正文改了，head 没改

例如技术博客只修正正文代码：

```text
title                  不变
description            不变
og:title               不变
article:modified_time   没维护
正文代码                v1 -> v2
```

此时 head fingerprint 仍可能相同。

更关键的是，在有旧 ETag/Last-Modified 时，服务器 HEAD 返回 200 后，Validator 如果发现 head fingerprint 相同，会返回 `FRESH`；也就是说，**即使某些响应头已经变化，匹配的 head fingerprint 仍可能让旧正文被继续使用。**

因此平台必须把：

```text
head fingerprint unchanged
```

定义为 WEAK evidence，绝不能单独推进业务 `last_verified_at`。

### 5.5 UNKNOWN 与 ERROR 的区别

- 没有 ETag、Last-Modified、head fingerprint：`UNKNOWN`；
- 无法证明新鲜：`UNKNOWN` 后完整抓取；
- 顶层校验 Timeout / RequestError / 异常：`ERROR`；
- `ERROR` 在 `arun()` 中不会强制抓新内容，而是返回旧缓存并标记 `hit_fallback`。

因此平台必须至少区分：

```text
VERIFIED_UNCHANGED
WEAK_UNCHANGED
VALIDATION_UNKNOWN
VALIDATION_ERROR_STALE_SERVED
FETCHED_FRESH
CHANGED
```

不能把“本次请求成功返回内容”直接统计成“增量同步成功”。

---

## 6. Runtime cache_status 的业务归一化

Crawl4AI 的返回状态可以帮助诊断，但不能直接做业务状态。

建议平台归一化：

```text
RUNTIME_CACHE_MISS
RUNTIME_CACHE_HIT_UNVERIFIED
RUNTIME_CACHE_HIT_VALIDATED
RUNTIME_CACHE_HIT_FALLBACK
RUNTIME_CACHE_DISABLED
RUNTIME_CACHE_BYPASSED
RUNTIME_CACHE_UNKNOWN
```

并额外记录：

```text
returned_artifact_origin = NETWORK | RUNTIME_CACHE | PLATFORM_ARTIFACT
freshness_evidence_strength = STRONG | MEDIUM | WEAK | NONE
business_verification_advanced = true | false
```

特别规则：

```text
hit_fallback
  -> 可以把旧 Markdown 服务给读请求
  -> Source Freshness 标记 DEGRADED
  -> 不更新 last_verified_at
  -> 不产生“未变化”的业务结论
  -> 创建 retry/revalidation task
```

---

## 7. Runtime 本地缓存的持久化边界

Crawl4AI 本地缓存适合单 Runtime 节点的重复抓取加速，但不适合作为分布式知识库真相。

它保存页面缓存及验证元数据，例如：

```text
etag
last_modified
head_fingerprint
cached_at
```

完整内容通过本地数据库/文件系统组合保存。

对于 1000+ 站点平台，其问题是：

1. Worker/容器迁移后缓存可能不存在；
2. 多 Worker 没有统一业务版本链；
3. Runtime cache key 不等于平台的 Source + URL Identity + Release + Attempt；
4. 无法承担审计、回滚、重放、跨 Release 对比；
5. cache hit 也无法证明 Coverage 或 Freshness。

所以边界必须明确：

```text
PostgreSQL + S3/MinIO = Business Truth
Crawl4AI Local Cache  = Disposable Runtime Optimization
```

---

## 8. 面向博客知识库的正确增量同步架构

### 8.1 平台级 Revalidation Engine

推荐：

```text
Incremental Scheduler
        |
        v
Revalidation Engine
        |
        +-- CMS/API revision changed ---------> FETCH_REQUIRED
        |
        +-- Sitemap/RSS changed signal --------> FETCH_REQUIRED
        |
        +-- Conditional GET 304 ---------------> VERIFIED_UNCHANGED
        |
        +-- validator changed -----------------> FETCH_REQUIRED
        |
        +-- weak head fingerprint only --------> WEAK_UNCHANGED
        |                                         |
        |                                         +-- sample/due -> FETCH_REQUIRED
        |
        +-- validation error ------------------> DEGRADED + RETRY
        |
        +-- no usable evidence ----------------> FETCH_REQUIRED
```

只有进入 `FETCH_REQUIRED` 后，才调用 HTTP Fast Lane / Browser Slow Lane 获取新的 Raw Artifact。

### 8.2 Revalidation Ledger

建议 PostgreSQL 保存：

```text
url_id
source_id
last_fetch_at
last_verified_at
last_changed_at
etag
last_modified
raw_body_sha256
semantic_body_hash
head_fingerprint
validator_strength
validation_method
validation_result
validation_reason
validation_http_status
cache_served
runtime_cache_status
runtime_cache_mode_effective
runtime_check_cache_freshness_effective
runtime_release
runtime_cache_contract_release
consecutive_304
forced_refresh_due_at
validation_error_count
validator_false_negative_count
source_validator_trust_score
```

### 8.3 证据等级

**STRONG**：

- CMS/API 明确 revision/version id 未变化；
- 经站点验证过的 Conditional GET 返回 304；
- 完整 GET 后 Raw Body hash 与历史一致。

**MEDIUM**：

- ETag / Last-Modified 发生变化；
- Sitemap `lastmod` / Feed `updated` 变化；
- 平台确认过但仍可能不可靠的 provider 更新信号。

**WEAK**：

- head fingerprint 相同；
- Content-Length/页面长度近似；
- link graph 相似；
- 页面局部元数据未变化。

只有满足 Source Policy 的 STRONG 证据可以推进 `last_verified_at`。

---

## 9. 强制完整刷新与 Validator Trust Score

长期运行时，任何轻量验证器都会有误判概率，所以必须持续做完整抽样。

配置：

```yaml
incremental:
  forced_full_refresh_interval: 7d
  forced_sample_rate: 0.02
  min_validator_trust_score: 0.98
  serve_stale_on_validation_error: true
  advance_verified_at_on_stale_cache: false
```

抽样流程：

```text
弱/强验证认为未变化
       |
       +-- sample due
              |
              v
         强制 fresh GET
              |
       compare raw body hash
        /             \
      same           changed
       |               |
  trust stable    false negative
                       |
          validator_trust_score 下调
                       |
          缩短强制刷新周期
                       |
     必要时禁用跳过完整抓取策略
```

这把“缓存校验正确性”从假设变成可量化、可自动降级的能力。

---

## 10. Cache Policy Compiler

业务层不要直接让用户选择 Crawl4AI enum，推荐暴露业务意图：

```text
AUTHORITATIVE_FETCH
VALIDATE_THEN_FETCH
REPLAY_FROM_ARTIFACT
RUNTIME_CACHE_WARMUP
DIAGNOSTIC_CACHE_READ
```

映射原则：

### AUTHORITATIVE_FETCH

目标：一定得到当前真实网络响应。

- 禁止从 Runtime cache 返回正文；
- 具体映射使用通过当前 release Contract Test 的配置；
- 不因为官方文档写着 `BYPASS` 就硬编码认为其永远不写/不读；
- 结果必须晋升平台 Raw Artifact。

### VALIDATE_THEN_FETCH

目标：平台先做权威/轻量 Revalidation，确定需要抓取后再 fresh fetch。

- 平台 Revalidation 决策优先；
- Runtime Smart Cache 仅允许做额外性能优化；
- Runtime 的 `hit_validated` 不能覆盖平台 Freshness 判断。

### REPLAY_FROM_ARTIFACT

目标：完全不联网地重跑 Cleaner/Extractor/Renderer。

- 直接读取 S3/MinIO Raw Artifact；
- **不能简单映射到 READ_ONLY**，因为 READ_ONLY cache miss 会联网；
- Replay Worker 应有网络禁用/egress deny 的执行环境更可靠。

### RUNTIME_CACHE_WARMUP / DIAGNOSTIC_CACHE_READ

只服务性能测试、诊断或临时缓存，不推进 Document Version/Freshness/Coverage。

---

## 11. 必须新增的 Runtime Cache Contract Test Matrix

这是本次调研对现有方案最值得补强的地方。

每个 Crawl4AI package/image 升级都执行真实契约测试，至少覆盖：

| 模式 | freshness | 初始缓存 | 模拟服务端 | 期望重点 |
|---|---|---|---|---|
| DISABLED | 任意 | hit/miss | 200 | 不读、不写、真实抓取 |
| BYPASS | 任意 | hit | 200 | 实测是否读/写，禁止按名字推断 |
| ENABLED | false | hit | 任意 | 直接 cache hit，不做校验 |
| ENABLED | true | hit | 304 | `hit_validated`，记录验证方法 |
| ENABLED | true | hit | HEAD 200 + head 同 | Runtime 可判 fresh，但平台只记 WEAK |
| ENABLED | true | hit | head 变 | 重新抓取 |
| ENABLED | true | hit | validation timeout | `hit_fallback`，不推进业务验证时间 |
| READ_ONLY | 任意 | miss | 200 | **仍会联网**，但不写缓存 |
| WRITE_ONLY | 任意 | hit | 200 | 不读旧缓存，真实抓取并写 |

每个 case 记录：

```text
should_read
should_write
network_called
request_method
conditional_headers
cache_status
cache_written
returned_content_hash
returned_artifact_origin
validation_status
```

### 11.1 发布门禁

Runtime Registry 保存：

```text
runtime_release
image_digest
cache_contract_suite_release
contract_passed_at
contract_result_hash
known_behavior_diff
approved_for_authoritative_fetch
approved_for_smart_cache
```

只有 Contract Test 通过的 Runtime release 才能进入生产；出现缓存语义变化时，Config Compiler 必须显式升级映射版本，而不是静默继承旧行为。

---

## 12. 生产指标与告警

建议增加：

```text
runtime_cache_hit_total
runtime_cache_hit_unverified_total
runtime_cache_hit_validated_total
runtime_cache_hit_fallback_total
runtime_cache_write_total
browser_avoided_by_revalidation_total
revalidation_304_total
revalidation_weak_unchanged_total
revalidation_error_total
forced_refresh_total
validator_false_negative_total
validator_trust_score
freshness_age_seconds
stale_served_seconds
runtime_contract_failure_total
```

重点告警：

- `hit_fallback` 突增；
- 某 Source validator false negative 超阈值；
- `last_verified_at` 长时间不推进；
- Runtime 升级后 cache write/read 行为改变；
- Browser Avoided 提升但 false negative 同时上升；
- stale serving 时间超过 Source SLO。

---

## 13. Web 管理后台需要展示什么

Source/URL 诊断页面至少展示：

```text
业务 last_fetch_at / last_verified_at
当前 ETag / Last-Modified
最近验证方法与证据强度
Runtime release / image digest
Runtime cache mode 实际值
check_cache_freshness 实际值
cache_status
是否发生 hit_fallback
是否服务旧版本
下一次 forced refresh
validator trust score
最近 false-negative 记录
```

管理员需要能执行：

- Force Fresh Fetch；
- Force Revalidation；
- Disable Weak Validation for Source；
- Clear Runtime Cache；
- Replay From Platform Artifact；
- 对比某次 Runtime 升级前后的 Contract Test。

---

## 14. 对总体技术方案的具体优化结论

本篇调研应落实为以下方案修改：

1. 保持 **Runtime Cache 不属于业务真相** 的架构边界；
2. 明确 Smart Cache 默认并不启用，Config Compiler 要记录其 effective 配置；
3. 明确 `READ_ONLY != offline/cache-only`，真正重放必须来自平台 Artifact；
4. 强化 `head_fingerprint` 只能作为 WEAK evidence；
5. `hit_fallback` 只能服务旧内容，不得推进 `last_verified_at`；
6. 权威轻量验证优先平台 Conditional GET，不把 Runtime HEAD/head fingerprint 直接提升为强证据；
7. 新增 **Runtime Cache Contract Test Matrix + Runtime Registry 发布门禁**；
8. 将官方文档与源码行为不一致视为正式风险模型，所有 Runtime 升级使用 pinned release + contract + canary；
9. 用强制 fresh sampling 持续测量 validator false-negative，并动态调整 Source Trust Score；
10. Web 后台显式展示业务 Freshness 与 Runtime cache 状态，避免运维把“有返回内容”误认为“已验证新鲜”。

最终建议仍是：**平台自己负责 URL/Version/Freshness/Coverage 真相；Crawl4AI 负责复杂页面执行和可丢弃缓存加速。**