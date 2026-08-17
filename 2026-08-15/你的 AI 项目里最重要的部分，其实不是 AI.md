# 你的 AI 项目里最重要的部分，其实不是 AI

## 资料信息

- 原文：The Most Important Part of Your AI Project Isn’t AI
- 作者：Brian Jenney
- 地址：https://brianjenney.medium.com/the-most-important-part-of-your-ai-project-isnt-ai-33938d8a1011
- 发布时间：2026-02-08
- 关联项目：https://github.com/unclecode/crawl4ai
- Crawl4AI 文档：https://docs.crawl4ai.com/

## 一、核心结论

文章最有价值的观点不是某个具体 API，而是把 AI 项目的瓶颈从“模型能力”重新定位到“数据供应链”：数据能否持续被发现、抓取、清洗、结构化、更新和验证，决定了上层 RAG、搜索、摘要与 Agent 的真实上限。

对于“1000 个技术博客全量历史文章 + 持续增量同步 + Markdown 知识库”的需求，这个结论意味着系统不能被设计成一个“批量调用 Crawl4AI 的脚本”，而应该被设计成一个长期运行的数据平台。Crawl4AI、HTTPX、Trafilatura、Playwright、外部托管抓取服务都只是可替换执行器；真正需要长期稳定的是 Source、URL Observation、Snapshot、Document Version、Quality、Coverage、Cost、Freshness 和 Lineage。

文章还给出了一个重要工程信号：作者团队从 Firecrawl 切换到 Crawl4AI 后显著降低了抓取成本，并获得更多基础设施控制权。这个信号不能简单推导为“所有页面都用 Crawl4AI Browser”，而应推导为两条原则：

1. 抓取能力必须可自托管、可观测、可替换，避免核心数据管线绑定单一按量计费服务；
2. 自托管并不等于零成本，必须把 CPU、内存、Browser 秒数、网络、对象存储、代理、LLM Token、失败重试都纳入成本核算，否则只是把 SaaS 账单换成不可见的基础设施账单。

## 二、技术原理分析

### 2.1 数据获取是知识库的事实层，不是模型的前置脚本

一个生产知识库应区分四类事实：

1. **发现事实**：某个 URL 在何时通过 Sitemap、Feed、CMS API、Archive、站内链接或 Common Crawl 被发现；
2. **网络事实**：某次 HTTP/Browser 请求实际返回了什么状态码、Header、Raw HTML、最终 URL、响应哈希；
3. **内容事实**：从某个不可变 Snapshot/Render Artifact 解析出了什么正文、标题、作者、发布时间、Canonical URL；
4. **派生事实**：Markdown、Chunk、Embedding、摘要、分类、标签、实体等由哪个版本的内容、哪个处理器生成。

如果直接把 `result.markdown` 写入磁盘，就丢失了前三层的大量上下文。一旦提取算法升级、页面模板变化、Markdown 规则改变或需要审计，就无法解释“为什么这篇文章现在长这样”。因此正确做法是 Snapshot First，Canonical IR Second，Markdown Projection Last。

### 2.2 “跨站点统一抽取”不等于“一个算法适配所有站点”

文章强调多个新闻站布局差异巨大，抓取工具和 LLM 可以把不同结构转成统一数据。工程上应实现为多级路由，而不是单一 extractor：

```text
L0 Authoritative Data
CMS API / RSS / Atom / JSON-LD / Sitemap metadata
            ↓ 不足
L1 HTTP + Deterministic Extractor
httpx + lxml/trafilatura/readability
            ↓ 质量不达标
L2 Browser Render
Crawl4AI / Playwright deterministic recipe
            ↓ 仍不达标
L3 Schema Extraction
JSON CSS/XPath/固定站点 Recipe
            ↓ 极少数疑难
L4 LLM-assisted Extraction / Agent Browser
严格 Schema、预算、审计、人工复核策略
```

这里的关键不是“越智能越好”，而是**最便宜的确定性路径先尝试，只在质量门禁失败时升级**。这样才能让 1000 个站点的成本稳定可控。

### 2.3 Crawl4AI 应是执行层 Adapter，而不是业务状态机

Crawl4AI 当前提供 `arun_many()`、`MemoryAdaptiveDispatcher`、`SemaphoreDispatcher`、`RateLimiter`、URL-specific config、Deep Crawl、AsyncUrlSeeder、Markdown Generator 等能力，适合做批量执行器。它的 Dispatcher 可以根据内存或并发上限控制当前进程内部任务，并对 429/503 做退避。

但是这些能力不能替代平台状态：

- `arun_many()` 的任务完成不等于平台 Backfill 完成；
- Crawl4AI cache 不等于增量同步版本事实；
- Deep Crawl 的 frontier 不等于历史 Coverage；
- Dispatcher 内部重试不等于平台级 retry/DLQ；
- `fit_markdown` 不等于 Canonical Markdown；
- crawler 进程重启后的内存状态不能承担持久 checkpoint。

因此平台应把一个 Crawl4AI 批次看成一次受控执行：输入是一组已经通过 Scope、Policy、Budget 校验的 FetchPlan，输出是 Snapshot/Render Artifact 和执行指标。

### 2.4 URL Seeding、Prefetch 与 Deep Crawl 的角色不同

Crawl4AI 官方文档对这些能力的定位可以组合成三层发现机制：URL Seeding 适合快速、低成本批量发现；Prefetch 适合在站点结构未知时只获取 HTML 与链接做快速映射；Deep Crawl 适合动态探索和补洞。

对于博客历史回填，优先级应该是：

```text
CMS API
  ↓
Sitemap / Sitemap Index
  ↓
RSS / Atom
  ↓
Archive / Pagination / Tag Index
  ↓
AsyncUrlSeeder(sitemap + 可选 Common Crawl)
  ↓
Prefetch Mapping（仅结构未知或 Coverage 证据不足时）
  ↓
bounded Deep Crawl 补洞
```

不能把 Deep Crawl 设为默认历史发现方法，因为它必须抓页面才能继续发现链接，成本高、Coverage 难证明，并容易被导航页、标签页、日历页制造无限空间。

### 2.5 Markdown 是稳定投影，不应直接作为唯一真相

Crawl4AI 的 `DefaultMarkdownGenerator` 可以输出 raw markdown，并结合 `PruningContentFilter` 等生成 fit markdown。这个能力适合作为候选内容或 extractor 的一部分，但知识库最终 Markdown 应由平台 Canonical IR 确定性渲染。

建议 IR 至少保存：

```yaml
document_id: doc_xxx
version_id: ver_xxx
source_id: src_xxx
canonical_url: https://example.com/post/1
final_url: https://example.com/post/1
published_at: 2026-01-01T10:00:00Z
updated_at: null
author: Jane Doe
language: zh-CN
title: 示例文章
blocks:
  - type: heading
    level: 2
    text: 标题
  - type: paragraph
    text: 正文
  - type: code
    language: python
    text: "print('hello')"
assets: []
```

再由版本化 renderer 输出 Markdown。这样修改链接规则、代码块规则、图片路径、Front Matter 时不需要重新访问原站。

### 2.6 Prefetch 的本质是“先画地图，再决定哪些页面值得做完整处理”

Crawl4AI v0.9.x 的 `CrawlerRunConfig(prefetch=True)` 使用轻量路径，只抓 HTML 并提取链接，跳过 Markdown 生成、正文抓取、媒体抽取和 LLM 抽取。官方文档将其定位为 two-phase crawling：第一阶段快速建立站点链接地图，第二阶段只对通过 Scope/URL Pattern/Article Classifier 的 URL 做完整处理。

对 1000 站点平台，Prefetch 不能替代 Sitemap/URL Seeder，因为它仍然要实际访问页面；它应该只在以下情况触发：

- Sitemap/CMS/Archive 的 Coverage 证据不足；
- 站点归档结构未知，需要发现分页和文章路径模式；
- Deep Crawl 成本过高，希望先用轻量模式估算空间；
- 需要在正式 Backfill 前估算 URL 数量、分支因子和噪声路径。

建议把 Prefetch 输出当作 `DISCOVERY_PREFETCH` 类型的网络事实：至少保存 final URL、状态码、响应哈希、内部链接集合、父子边、发现时间、Runtime Release 和 Scope 判定。若 Raw HTML 已经获取且在可接受 freshness 窗口内，可将其持久化为 Snapshot 并直接进入后续 Extract，避免对被选中的文章二次请求。

重要的是，官方文档给出的性能数字只能作为工具能力说明，不应直接写成平台 SLO。真实吞吐要在自己的 Golden Sites/Benchmark 上测量，并按静态站、JS 站、网络区域分别建立基线。

## 三、针对 1000 站点方案应新增的能力

### 3.1 数据质量门禁 Quality Gate

文章把“cleaning / structuring / keeping updated”放到核心位置，因此方案不能只记录抓取成功率，还要记录内容质量。

新增 `document_quality`：

```text
quality_id
snapshot_id
extractor_release_id
content_length
text_density
boilerplate_ratio
link_density
heading_count
code_block_count
title_confidence
author_confidence
published_at_confidence
content_completeness_score
structure_score
overall_score
gate_result: PASS | QUARANTINE | FAIL
reasons[]
created_at
```

建议规则：

- HTTP 200 但正文少于站点基线的 20%：QUARANTINE；
- 页面标题与正文主标题明显不一致：QUARANTINE；
- 导航/链接文本占比异常高：进入 Browser 或备用 extractor；
- 同 URL 新版本正文突然减少超过 70%：不直接覆盖，进入回归检查；
- 发布时间缺失可以 PASS，但置信度必须记录，不能伪造；
- Canonical Markdown 只有 PASS 版本才能成为默认最新版本。

这能避免“任务成功，但知识库里都是登录页、Cookie Banner、半屏骨架页”的假成功。

### 3.2 每 Source 的 Golden Corpus

为每个站点抽取 3~10 个代表页面：普通文章、长文、代码很多的文章、图片很多的文章、旧模板文章、最新模板文章。

每次升级以下任意组件都跑回归：

- Crawl4AI Runtime；
- Chromium；
- Trafilatura/readability；
- Source Profile；
- Browser Recipe；
- Markdown Renderer；
- LLM Extractor Prompt/Schema。

比较指标不是只看“是否成功”，而是：正文覆盖率、噪声率、标题/作者/时间准确率、代码块完整率、链接保留率、Markdown diff 大小。只有回归通过才能发布新 Release。

### 3.3 成本事实层与 FinOps

新增 `cost_event` 和 `source_budget_policy`。

`cost_event` 建议记录：

```text
run_id / task_id / source_id / document_id
executor_type: HTTP | CRAWL4AI | PLAYWRIGHT | EXTERNAL | LLM
cpu_ms
memory_peak_mb
browser_ms
network_in_bytes
network_out_bytes
proxy_bytes
object_storage_bytes
llm_input_tokens
llm_output_tokens
external_request_cost
estimated_infra_cost
```

Web 管理台至少展示：

- 每 Source 每月成本；
- 每 1000 个有效 Document 的成本；
- Browser fallback 比率；
- 每种 Adapter 的成功率与成本；
- 失败重试浪费成本；
- 304 命中率；
- 每个新版本的平均获取成本。

真正应该优化的是 `cost / accepted_document_version`，而不是“单次请求便宜”。

### 3.4 Acquisition Planner：质量约束下的最小成本路由

新增 Planner，根据 Source Profile 和历史指标生成 FetchPlan：

```python
if authoritative_api_available(url):
    route = "CMS_API"
elif static_http_success_rate > 0.98 and http_quality_p50 > 0.92:
    route = "HTTP"
elif browser_required_ratio > 0.8:
    route = "CRAWL4AI_BROWSER"
else:
    route = "HTTP_WITH_BROWSER_FALLBACK"
```

执行后将结果反哺 Planner，但不能让一个短期异常直接自动修改生产 Profile。自动建议与生产发布分离，Profile 变更要版本化并可回滚。

### 3.5 “自托管优先、托管服务可插拔”的 Provider Strategy

文章的 Firecrawl → Crawl4AI 实践说明自托管可以明显降低规模化抓取成本，但方案不应反向形成 Crawl4AI 锁定。

统一定义：

```python
class FetchAdapter(Protocol):
    async def fetch(self, plan: FetchPlan) -> FetchResult: ...

class RenderAdapter(Protocol):
    async def render(self, plan: RenderPlan) -> RenderResult: ...

class ExtractionAdapter(Protocol):
    async def extract(self, artifact: ArtifactRef, schema: SchemaRef) -> IR: ...
```

默认：HTTPX + Crawl4AI 自托管。

可选：Firecrawl/其他托管 Provider 只作为明确启用的 fallback、灾备或特殊站点 Adapter。这样未来价格、限制、稳定性变化时不需要改业务模型。

### 3.6 在线模板漂移检测：Golden Corpus 只能防“升级回归”，不能防“源站突然改版”

Golden Corpus 主要回答“我们升级 Runtime/Extractor 后有没有变坏”，但技术博客会在没有任何平台 Release 的情况下改主题、CMS、DOM、分页方式或反爬策略。因此生产链还需要在线 Template Drift Detector。

每个 Source 建议维护滚动基线：

```text
layout_fingerprint
article_container_fingerprint
selector_hit_rate
content_length_p50/p95
boilerplate_ratio_p50
link_density_p50
heading_count_p50
code_block_count_p50
http_to_browser_fallback_ratio
quality_pass_ratio
```

每次新样本到达时计算 drift score。以下信号应触发 `Source -> DEGRADED` 或至少告警：

- 稳定 selector 命中率突然大幅下降；
- 同一 Source 的正文长度、结构块数量发生群体性突变；
- HTTP 路径质量骤降、Browser fallback 突然升高；
- 多篇文章同时出现相同 Cookie/Login/Challenge 文本；
- Sitemap/Archive 的 URL pattern 明显改变。

漂移发生后不能让 LLM 自动修改生产 Recipe。正确流程是：生成 Profile/Recipe 候选 -> 在最近样本 + Golden Corpus 上回放 -> 比较 Coverage/Quality/Cost -> 人工或策略审批 -> 发布新 Release。

### 3.7 分布式 Domain Admission 与 Source Fairness

Crawl4AI 的 `RateLimiter` 和 Dispatcher 是执行器内部能力。即使每个 Worker 都很“礼貌”，当几十个 Pod 同时访问同一域名时，整体仍可能超限。因此平台必须在任务被投递给 Fetch Worker 之前做全局 Admission Control。

建议模型：

```text
domain_runtime_state
- registrable_domain
- configured_rps
- burst
- max_concurrency
- inflight
- backoff_until
- circuit_state
- last_429_at
- last_503_at
- adaptive_rps_ceiling
```

实现上可以把 PostgreSQL 作为策略与持久状态事实，把 Redis Lua token bucket/semaphore 作为快速分布式闸门；Redis 丢失只影响短期调度效率，不能造成 Task/Run 丢失。Worker 获取 domain slot 后才可发请求，遇到 `Retry-After`、429、503 时把退避事实回写并降低后续 Admission。

同时只做 P0/P1/P2 优先级还不够。一个拥有百万历史 URL 的大站可能长期占满 Backfill 队列，导致小站饥饿。调度器应在优先级之上按 Source 做 weighted fair queue / deficit round robin：先保证增量任务优先，再在同优先级内给各 Source 公平份额，并允许通过业务权重调整而不是无限占用。

### 3.8 Data Supply SLO：从“请求成功”提升为“可用知识到达速度”

文章的核心是“高质量、持续更新的数据供应链”，因此平台还应增加一组直接反映知识可用性的 SLO：

```text
time_to_usable_data_seconds
= URL 首次被发现 -> DocumentVersion Quality PASS 且索引/Markdown 可用

usable_document_yield
= Quality PASS 的新/变更 DocumentVersion / 实际完整抓取页面数

rejected_fetch_ratio
= QUARANTINE + FAIL / 完整抓取页面数
```

这些指标能识别一种常见假象：HTTP 成功率很高，但大部分抓到的是非文章页、重复页、骨架页或低质量正文。对于知识库真正有意义的是“多少网络访问最终转化成可用、可追溯的知识版本”。

## 四、增量同步的具体实现

### 4.1 两条增量链同时运行

**发现增量**负责“有没有新 URL”：

- RSS/Atom 新 item；
- Sitemap `lastmod` 变化；
- CMS API cursor；
- Archive 首页/分页最近区间；
- 低频 bounded Deep Crawl 补漏。

**内容增量**负责“已知 URL 有没有变化”：

- `If-None-Match` / ETag；
- `If-Modified-Since` / Last-Modified；
- 对不支持条件请求的站点做低成本 HTTP；
- Snapshot byte hash；
- Extracted IR hash；
- Semantic content hash。

Seen URL 绝不能等于“不再抓”。它只阻止重复创建 Document 身份，不阻止 revalidation。

### 4.2 自适应刷新周期

根据站点历史发布/修改频率分层：

```text
HOT:   15~60 分钟发现增量
WARM:  2~6 小时
COOL:  12~24 小时
COLD:  2~7 天
```

文章页自身 revalidation 根据“文章年龄 + 历史修改概率”衰减：新文章频繁检查，超过一定年龄后降低频率，但保留低频校验。

### 4.3 删除不是一次 404

状态建议：

```text
ACTIVE
SUSPECT_MISSING
MISSING_CONFIRMED
MOVED
BLOCKED
```

404/410、Sitemap 移除、Canonical 迁移、Redirect 都保留事实。只有满足连续多次证据或明确 410 才确认删除。Markdown 默认导出可以隐藏已删除文章，但历史版本不能物理抹掉。

## 五、Web 管理功能应围绕数据运营设计

### Source 页面

展示：

- 当前 Profile / Release；
- Discovery Provider 状态；
- 历史 Coverage 与 known gaps；
- 最近增量时间；
- URL 数、Document 数、Version 数；
- 抓取成功率；
- Quality PASS/QUARANTINE/FAIL；
- HTTP/Browser 路由占比；
- 成本趋势；
- Template Drift 分数与最近改版告警；
- Domain Admission 当前 RPS/并发/退避状态；
- `time_to_usable_data` 与 `usable_document_yield`；
- robots / block / 429 / 5xx 状态。

### Document 页面

展示：

- 原 URL、Canonical URL、发现来源；
- Snapshot 历史；
- Render Artifact；
- Extractor/IR/Markdown 版本；
- 版本 diff；
- 质量分数；
- Markdown 预览；
- 手动重新抓取/重新抽取/重新渲染；
- Quarantine 审核与原因。

### Onboarding 页面

新增站点时自动执行 Probe：

1. 检测 robots；
2. 检测 Sitemap/Feed/CMS；
3. 抽样 5~10 个 URL；
4. 比较 HTTP、Crawl4AI Browser、备用 extractor 的质量/耗时；
5. 当权威发现不足时执行受限 Prefetch Mapping，估算 URL 空间与路径模式；
6. 生成 Source Profile 建议；
7. 估算 Backfill 页数、耗时和成本区间；
8. 人工确认后上线。

这使第 1001 个站点主要是“新增配置 + 验收”，不是“新写一个爬虫项目”。

## 六、运行时与安全

Crawl4AI 自托管当前文档强调 secure-by-default：认证、严格请求边界、受限/声明式 hook 等方向。平台仍需再包一层 Policy Gateway，因为 crawler 的安全边界不能代替业务安全边界。

必须做到：

- 起始 URL 和每次 redirect 都做 SSRF 校验；
- 禁止访问 localhost、metadata service、RFC1918 等未授权网段；
- 代理、JS、Hook、Header、Cookie、Browser args 必须来自版本化白名单 Profile；
- 普通 Web 用户不能直接向 crawler 透传任意 JavaScript；
- Runtime 镜像固定 digest，Chromium/依赖固定版本；
- Browser Context 按 Source/安全域隔离；
- Worker 无核心数据库 DDL 权限；
- Secret 通过 Secret Manager 下发，不写入 Profile；
- 全链路 Audit Event 可关联用户操作、Task、Runtime 和 Artifact。

## 七、推荐运行参数与容量原则

1000 个技术博客不应按“1000 个 crawler 常驻”设计，而应按任务池设计。

推荐拆成独立 Worker Pool：

```text
discovery-http     高并发、低内存
fetch-http         高并发、低成本
browser-light      Crawl4AI Browser，限制并发
browser-heavy      Playwright Recipe，严格限额
extract            CPU 型
llm-extract        独立预算
asset              低优先级
index              异步投影
```

Crawl4AI `arun_many()` 在单 Worker 内可使用 `MemoryAdaptiveDispatcher` 做内存保护，并配合 `RateLimiter` 控制 429/503；平台外层仍按 domain/source 维护 token bucket，避免多个 Worker 同时对同一站点施压。平台调度器还应在任务优先级之上提供 Source 级公平份额，避免超大 Backfill Source 饿死其他站点。

浏览器优化：

- Browser 进程复用；
- Context 按站点隔离；
- Page 按 Task 生命周期；
- 默认阻止广告、追踪、字体、视频等非必要资源；
- 只有 Asset Policy 要求时才下载正文图片；
- 不默认截图/PDF；
- 静态文章不要启 Browser。

## 八、最终技术判断

这篇文章不会推翻现有“Coverage First + Snapshot First + Adapter + Incremental”的方向，但它要求进一步把**数据质量、可用知识到达速度和单位有效数据成本**提升为一等公民。

最重要的新增设计是：

1. Quality Gate 与 Quarantine；
2. Source Golden Corpus + 抽取回归；
3. Cost Event + Source Budget + `cost/accepted_document_version`；
4. Acquisition Planner 按质量约束选择最低成本路径；
5. 自托管 Crawl4AI 为默认 Browser/抓取能力，但保持托管服务 Adapter 可替换；
6. Web 管理从“任务管理”升级为“Coverage + Quality + Freshness + Cost”的数据运营控制台；
7. 在 URL Seeder 与 bounded Deep Crawl 之间加入受限 Prefetch Mapping，用两阶段策略减少无效完整抓取；
8. 加入在线 Template Drift Detection，区分“我们升级造成的回归”和“源站改版造成的退化”；
9. 加入分布式 Domain Admission + Source Fair Scheduler，保证横向扩容后仍对源站友好且不会发生队列饥饿；
10. 用 `time_to_usable_data`、`usable_document_yield` 衡量真实数据供应链产出，而不只看请求成功率。

这样系统才能在从 1000 个站点扩展到数千站点后，仍然知道自己抓全了多少、抓得好不好、多久没更新、每类站点花了多少钱、网络请求最终转化出了多少可用知识，以及任意一篇 Markdown 是如何生成的。
