# 如何用 Python 打造 AI 资讯日报机器人？每天自动推送最新技术动态！

## 1. 调研对象

- 编号：60
- 来源：CSDN / DAMO 开发者矩阵
- 原文：https://damodev.csdn.net/68639263b93e2f4179623d49.html
- 原文发布日期：2025-02-24
- 调研日期：2026-08-16
- 关联技术：Python、Crawl4AI、LLM Structured Extraction、Pydantic、Markdown、企业微信机器人、多源资讯聚合

本文的价值不在于把示例代码直接搬进 1000+ 技术博客知识库，而在于它完整展示了一个“小型日报机器人”的闭环：**列表页抓取 → LLM 结构化筛选 → 时间排序 → Markdown 编排 → 按目标限制分片 → 多机器人推送**。这个闭环正好暴露出从 Demo 走向长期数据平台时最容易被忽略的几类边界：时间窗口、事实与筛选分离、列表页发现与正文事实分离、发布目标能力约束、分片幂等和多目标部分失败。

## 2. 原文实现链路

原文的核心数据流可以归纳为：

```text
AI 新闻列表页
  -> AsyncWebCrawler.arun()
  -> LLMExtractionStrategy(schema=Article)
  -> title/link/summary/release_time
  -> 汇总多个来源
  -> 按 release_time 排序
  -> 截取最新若干条
  -> Markdown section
  -> 按 UTF-8 字节估算切块
  -> 企业微信 Robot Webhook 多目标推送
```

原文使用 Pydantic `Article` 约束结构化字段，借助 Crawl4AI 的 LLM extraction 从网页中一次性提取“某日的高质量 AI 资讯”；然后把所有来源结果合并，按发布时间排序，生成 Markdown，并为了适应企业微信消息大小限制进行分片，最后依次向多个机器人 webhook 发送。

这个方案对于个人机器人很直接，但它把五种不同语义放进了一条同步链：

1. **发现事实**：列表页上实际出现了哪些文章链接；
2. **字段观察**：列表页声称的标题、发布时间、摘要是什么；
3. **AI 选择**：哪些内容属于“高价值资讯”；
4. **Issue 编排**：哪一天、哪个时区、截止到什么时刻应进入本期；
5. **发布执行**：目标平台允许什么 Markdown、最大载荷是多少、如何重试。

在知识库平台里，这五层必须拆开，否则一个 LLM 输出、时间解析错误或某个 webhook 失败，就可能同时污染采集事实、内容选择和发布状态。

## 3. Crawl4AI 实现细节与版本差异

### 3.1 `LLMExtractionStrategy` 的真实技术原理

Crawl4AI 的 LLM extraction 不是“浏览器里内置一个会理解网页的模型”。其本质是：

```text
网页抓取结果
  -> 选择输入表示（markdown/html/fit_markdown 等）
  -> 可选 chunking
  -> 拼接 instruction + schema
  -> 调用 LLM Provider
  -> 聚合各 chunk 输出
  -> 解析为结构化 JSON
```

当前官方文档仍推荐 `extraction_type="schema"` 配合 Pydantic JSON Schema。对于页面很长的情况，`chunk_token_threshold`、`overlap_rate`、`apply_chunking` 会改变调用次数、成本和跨块一致性，因此它们也属于结果语义的一部分，生产系统必须进入版本化 Release，而不是隐藏在代码默认值里。

官方文档：

- https://docs.crawl4ai.com/extraction/llm-strategies/
- https://docs.crawl4ai.com/api/arun_many/
- https://docs.crawl4ai.com/advanced/multi-url-crawling/

### 3.2 原文 API 不能按 2026 当前版本直接照搬

原文示例把 `provider`、`api_token`、`base_url` 等直接传给 `LLMExtractionStrategy`。Crawl4AI 0.5.0 已把这一接口迁移到 `LLMConfig`；当前官方示例使用：

```text
LLMExtractionStrategy(
    llm_config=LLMConfig(...),
    schema=...,
    extraction_type="schema",
    instruction=...
)
```

官方 0.5.0 Release Notes 已明确说明旧的直接 `provider/api_token/base_url` 参数被移除。说明平台不能把博客中的示例 API 视为稳定 Contract，必须通过 `crawler_runtime_release + adapter contract test` 隔离第三方库演进。

参考：https://docs.crawl4ai.com/blog/releases/0.5.0/

### 3.3 多 URL 场景不应自己无界 `gather`

原文强调异步，但 1000+ Source 场景的关键不是“能 async”，而是全局和域名级预算。当前 Crawl4AI 的 `arun_many()` 已提供 Dispatcher、内存自适应和 RateLimiter 等能力；平台仍不能把 Dispatcher 当全局调度真相，因为它只管理某个 crawler runtime 内的执行。正确分层是：

```text
PostgreSQL durable task + fair scheduler + budget
  -> Worker lease
  -> Crawl4AI adapter
  -> arun_many/dispatcher 做本进程资源控制
```

这样 Crawl4AI 的并发优化与平台级 source fairness、per-origin QPS、lease、retry、deadline 不会混在一起。

## 4. 列表页直接 LLM 提取的优点与风险

### 4.1 优点

对于结构变化频繁的资讯列表页，LLM schema extraction 能快速把不同 DOM 形态归一化为统一 `Article` 结构，减少为每个站点维护 CSS selector 的成本。它尤其适合：

- 站点只是少量候选来源；
- 页面结构变化快；
- 只需要做编辑精选；
- 允许一定模型成本；
- 抽取错误不会删除或覆盖事实。

### 4.2 关键风险：LLM 不能创建 URL 事实

原文让模型同时给出 `link`。在生产知识库中，链接是后续 Fetch、Identity、Citation 的安全边界，不能仅凭模型字符串进入正式 URL Inventory。

推荐把列表页处理拆成：

```text
Listing Artifact
  -> deterministic href enumeration
  -> ListingItemObservation(url/title/time hints)
  -> optional LLM classify/rank/field-assist
  -> URL must map back to observed href or approved resolved target
  -> URLObservation
  -> normal Fetch/Artifact/Extract/Quality pipeline
```

LLM 可以回答“这个列表项看起来是不是 AI 技术突破”，但不能凭空生成一个未在 Artifact 中观察到的链接并把它当 Provenance。若模型返回 URL 与页面观测 href 不一致，必须进入 `LLM_URL_UNGROUNDED`/人工复核，而不是自动抓取。

### 4.3 列表摘要不是正文

列表页上的一行摘要通常是站点 teaser，甚至可能由站点二次编辑。它只能作为 discovery metadata。知识库最终正文仍必须抓文章页并进入 Artifact → Extraction Candidate → Quality → Canonical IR → Document Version。这样列表页改版或摘要变化不会生成假的正文版本。

## 5. 日期排序实现存在的结构性问题

原文用字符串 `release_time` 直接排序，并用类似空日期兜底字符串把无时间项放到尾部。这对格式完全一致的 ISO 日期勉强可用，但跨站点时会出现：

- `2026-8-1` 与 `2026-08-01` 字典序不稳定；
- `08/01/2026`、`2026年8月1日` 无法直接比较；
- 来源没有时区；
- 页面展示“3 小时前”；
- 列表页时间与文章页时间冲突；
- `published_at`、`updated_at`、`discovered_at` 被混用；
- 迟到文章在日报已发布后才被发现。

因此应保存时间观察而不是只存一个最终字符串：

```text
publication_time_observation
- document/url/listing_item id
- raw_value
- parsed_at
- parsed_timezone
- normalized_utc
- evidence_type
- evidence_artifact_id
- confidence
- parser_release
```

时间选择策略再根据可信度决定 `effective_published_at`。用于日报的“某一天”必须由 **Issue Window** 定义，而不是由代码运行时的 `date` 字符串隐式决定。

## 6. Issue Window：日报真正缺失的状态模型

日报系统需要先冻结时间语义：

```text
issue_window
- issue_id
- timezone
- window_start
- window_end
- selection_cutoff
- allowed_lateness
- late_arrival_policy
- max_items
- state
```

这里有三个时间不能混：

- **event time**：文章发布时间；
- **ingest time**：平台什么时候发现/接收入库；
- **processing time**：日报任务什么时候实际运行。

例如日报计划每天 09:00 Asia/Shanghai 发布，可以定义“前一日 00:00:00～23:59:59 的 event time”，同时允许 8 小时 late arrival，09:00 到达 cutoff 后冻结 Selection。即使 Scheduler 因故 09:17 才真正执行，Issue 仍然使用同一个窗口，结果不会因为任务延迟漂移。

### 6.1 Late Arrival 不能静默改已发布日报

若文章在 `selection_cutoff` 后才被发现，有三种明确策略：

- `NEXT_ISSUE`：放入下一期；
- `REVISION`：生成本期 revision，重新人工批准；
- `IGNORE_FOR_CURATION_BUT_KEEP_IN_KB`：知识库照常收录，但不修改已发布 Issue。

已发布 Issue 的输入集合必须不可变。不能“每天重跑一次查询”后直接覆盖昨天的消息，否则无法解释用户当时到底看到了什么。

## 7. Selection Manifest：把“最新 30 条”变成可重放决策

原文“合并 → 按日期排序 → 取最新 30 条”隐藏了很多策略。生产实现应冻结：

```text
selection_manifest
- issue_id
- issue_window_id
- selection_policy_release
- candidate_version_ids[]
- included_version_ids[]
- excluded_items[] + reason
- story_cluster_snapshot
- ranking_features
- deterministic_tiebreak
- created_at
- manifest_hash
```

推荐选择流程：

```text
PASS Document Versions in Issue Window
  -> validate effective published time
  -> story-cluster dedupe
  -> source/topic diversity constraints
  -> deterministic base rank
  -> optional LLM semantic score
  -> Manual PIN/EXCLUDE
  -> freeze Selection Manifest
  -> Evidence Package
```

LLM 的“高价值”判断只作为一个版本化 rank/label feature，不能决定 Collection Truth。这样同一个 Issue 可以解释“为什么 A 入选而 B 没入选”，也可以在模型升级后离线对比，而不会改动历史发布事实。

## 8. Markdown 分片真正应该抽象成 Publication Bundle

原文有一个非常实用的细节：按 UTF-8 字节计算消息体大小，而不是简单按 Python 字符数切。中文字符在 UTF-8 中通常占多个字节，所以 `len(text)` 和传输载荷大小并不相等。

但生产系统不能把示例中的某个固定上限写死为全局常量，因为不同目标可能限制：

- bytes 或 characters；
- Markdown / Markdown dialect / plain text；
- 单条消息最大长度；
- 单次最多多少条消息；
- URL/图片/卡片字段长度；
- 每分钟请求数；
- webhook 重试语义。

应版本化：

```text
publication_target_capability_release
- target_id
- message_format
- length_unit: BYTE | CHAR | TOKEN | FIELD
- max_payload
- max_chunks
- supported_markdown_features
- link_policy
- rate_limit_policy
- retry_policy
- capability_source
```

生成前先编译为不可变 Bundle：

```text
publication_bundle
- issue_id
- target_id
- capability_release_id
- source_content_hash
- chunking_release
- chunks[]
- bundle_hash
```

### 8.1 分片算法必须保持语义原子性

正确的顺序不是对最终大字符串 `text[:N]`，而是：

```text
header block
article card 1
article card 2
...
footer block
  -> estimate encoded size per atomic block
  -> pack blocks greedily/deterministically
  -> if one block itself exceeds target limit:
       apply target-specific shortening policy
       or mark PAYLOAD_TOO_LARGE
```

不能从 Markdown link 中间截断，也不能把代码 fence、URL、emoji surrogate/多字节编码切坏。若标题+链接本身超限，必须进入明确的 shortening policy，而不是静默截断事实字段。

## 9. 多机器人推送需要目标级幂等与部分失败恢复

原文对 RobotList 逐个发送。这种方式在 Demo 中够用，但长期运行会出现：

- 前两个群发送成功，第三个超时；
- Worker 重启后整批重发，导致前两个群重复；
- 多分片消息发送到第 3 片失败；
- 某个 webhook 被禁用，不应拖慢其他目标；
- 目标 A 限流时不应占住目标 B 的发送能力。

推荐幂等键：

```text
publish:<issue_id>:<target_id>:<bundle_hash>:<chunk_index>
```

并持久化：

```text
publication_chunk_attempt
- bundle_id
- chunk_index
- target_id
- attempt_no
- idempotency_key
- request_hash
- response_status
- provider_message_id
- outcome
- next_retry_at
```

重试只补失败 chunk；一个 target 的失败不会把 Issue 或其它 target 回滚为失败。最终状态应区分 `DELIVERED`、`PARTIAL`、`FAILED_RETRYABLE`、`FAILED_FINAL`。

## 10. CacheMode.BYPASS 的适用边界

原文为了拿最新资讯使用 `CacheMode.BYPASS`，这符合“每日列表页必须观察当前内容”的直觉，但如果把它扩展到所有正文，会造成不必要请求。

知识库应分层：

- Listing/Feed change signal：可按 freshness policy 低 TTL 或 bypass；
- 已知文章正文：优先 ETag / Last-Modified / conditional GET；
- LLM/Extractor：优先对已经保存的 immutable Artifact 离线重放；
- Issue 重生成：只重放 Selection Manifest/Evidence Package，不重新抓网页。

“缓存绕过”是某个 Fetch Observation 的策略，不是整个系统的全局模式。

## 11. 原文中性能与“反爬”表述不能直接变成容量/合规承诺

原文给出了单机吞吐和“处理验证码/IP 封锁”等亮点描述。生产方案不应把博客中的演示数字当容量基线，也不应把 crawler 的某种挑战处理能力等同于允许绕过访问控制。

1000 Source 的吞吐必须基于真实站点分布做 benchmark，并独立统计 HTTP、Browser、LLM extraction、Embedding、Publication 的资源成本。robots、WAF、CAPTCHA、登录墙仍由 Source Policy 决定；`ROBOTS_BLOCKED/POLICY_BLOCKED/WAF_CHALLENGE` 不应自动升级为规避访问限制的路径。

## 12. 对现有博客知识库技术方案的优化结论

本轮最值得加入现有方案的不是“再增加一个每日定时脚本”，而是补齐以下生产语义：

1. **Listing Discovery Fact Boundary**：列表页先保存 Artifact 并确定性枚举 href；LLM 只做分类/排序/字段辅助，模型 URL 必须能回指页面真实观测或明确 Resolution Evidence。
2. **Issue Window**：显式冻结 timezone、event-time 窗口、selection cutoff、allowed lateness 和 late-arrival policy，避免 Scheduler 延迟和跨时区造成日报漂移。
3. **Selection Manifest**：在 Evidence Package 前冻结候选、入选、排除原因、rank release、cluster snapshot 和 deterministic tie-break，使“最新/高价值 N 条”可解释可重放。
4. **Published Issue Immutability**：迟到内容进入下一期或显式 revision，不静默覆盖历史发布。
5. **Publication Target Capability Release**：长度单位、最大载荷、Markdown 能力、分片数、rate limit、重试策略都由目标能力版本化，禁止硬编码某个渠道的常量成为全局事实。
6. **Deterministic Publication Bundle**：按语义块和真实编码大小切片，保存 bundle/chunk hash，避免对最终字符串粗暴截断。
7. **Chunk-level Delivery Idempotency**：多目标、多分片分别持久化发送状态，部分失败只补失败 chunk，不导致已成功目标重复发送。
8. **Publication Time Evidence**：发布时间保留 raw observation、时区、解析器 Release 与置信度，不用字符串字典序直接决定 Issue 归属。
9. **LLM Extraction Runtime Adapter**：Crawl4AI API 和 LLM 配置必须由 Runtime Release/Adapter 隔离；博客示例的旧参数不能直接固化到平台业务层。
10. **Freshness 与 Replay 分离**：列表入口可以按策略强制新鲜观察，但正文增量优先 Conditional GET，生成/筛选重试基于 Artifact/Manifest 离线重放。

这些补充让现有方案从“可以生成 Newsletter”提升为“同一期输入、时间边界、选择决策、分片结果和每个目标的实际发送都可复现”。对于长期知识库，这一点比单次机器人能否成功推送更重要：Collection Truth 始终独立，日报只是一个有明确 event-time、manifest 和 delivery lineage 的可审计 Projection。

## 13. 推荐的最终执行链路

```text
Source Listing / Feed / Sitemap
  -> Fetch Observation + Immutable Artifact
  -> deterministic URL/link observation
  -> optional LLM classification / metadata assist
  -> validated URLObservation
  -> article Fetch
  -> Extraction Portfolio
  -> Quality Gate
  -> Canonical IR
  -> Document Version
  -> Markdown / Search / Enrichment

Scheduled Daily Issue
  -> create Issue Window(timezone/window/cutoff/lateness)
  -> query eligible PASS Versions
  -> normalize publication time evidence
  -> Story Cluster / diversity / ranking
  -> optional LLM semantic score
  -> Manual PIN/EXCLUDE
  -> freeze Selection Manifest
  -> freeze Evidence Package
  -> compose Digest
  -> Citation/Editorial Gate
  -> target Capability Release
  -> deterministic Publication Bundle
  -> chunk-level idempotent fan-out
  -> Delivery Observation / partial retry
```

这个架构保留了原文“抓取—智能筛选—Markdown—多群推送”的实用闭环，但把所有会影响长期一致性、历史可解释性和扩展性的隐含状态都显式化。
