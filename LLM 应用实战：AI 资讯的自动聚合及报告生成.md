# LLM 应用实战：AI 资讯的自动聚合及报告生成——实现细节与技术原理分析

## 1. 调研对象

- 编号：61
- 名称：LLM 应用实战：AI 资讯的自动聚合及报告生成
- 原文：https://www.cnblogs.com/mengrennwpu/p/18529735
- 原文发布时间：2024-11-06
- 主要技术：Crawl4AI、OpenAI Swarm、Python、BeautifulSoup、Textdistance、python-docx
- 与本项目的相关性：高。文章包含“发现文章 URL -> 增量去重 -> 抓取正文 -> AI 处理 -> 排序 -> 报告输出”的完整小型流水线，虽然规模和可靠性远低于 1000 站点知识库要求，但暴露了增量同步、异步抓取、稳定身份、派生内容、批量处理和恢复机制中最典型的问题。

本文不复刻原文代码，而是从生产系统视角拆解其技术原理、边界和可迁移设计。

---

## 2. 原方案的整体数据流

原文实现可以概括为：

```text
AIBase 列表页
 -> CSS Schema 抽取文章链接
 -> URL md5 作为文章 id
 -> 本地 JSON 历史记录去重
 -> Crawl4AI 获取正文 HTML
 -> BeautifulSoup 定向抽标题/日期/正文/图片
 -> Swarm Analysis Agent 编排
    -> Translate Agent
    -> Classifier Agent
    -> Modifier Agent
 -> 按日期保存处理结果
 -> Sorter Agent 排序 Top N
 -> Textdistance 将排序标题重新映射到原记录
 -> python-docx 输出日报
```

它实际上包含两个不同性质的系统：

1. **事实采集链**：URL 发现、抓取、正文解析、历史去重；
2. **派生内容链**：翻译、分类、摘要改写、排序、报告生成。

对于博客知识库，这两个链路必须彻底解耦。采集链负责“源站到底发布了什么”，派生链负责“我们如何消费和理解这些内容”。后者可以重跑、升级、删除，不能污染前者的 Canonical Markdown。

---

## 3. URL 发现机制：CSS Schema 的价值与局限

原文列表页通过 Crawl4AI 的 `JsonCssExtractionStrategy` 抽取链接，核心思路是：

```text
页面 -> CSS baseSelector -> 字段 selector -> href 属性 -> 文章 URL 列表
```

### 3.1 技术原理

这类 Schema Extraction 本质上是把 DOM 解析规则声明成数据，而不是把所有逻辑写死在爬虫函数中。优点是：

- selector 可以配置化；
- 字段结构可验证；
- 同一 Runtime 可以服务多个站点；
- 页面结构变化时，只更新配置，不改调度框架；
- 很适合 Archive、列表页、分页页、目录页等“发现型页面”。

这与当前博客知识库方案中的 `Source Profile + Recipe + Schema` 方向一致。

### 3.2 原方案的关键缺陷

原文只从首页或新闻列表页提取链接。这对于“每日资讯”够用，但对于“全量历史知识库”不够，因为列表页通常只包含最近几十/几百篇文章。

如果把这种方式直接扩展到 1000 个博客，会出现错误的“抓完了”判断：

```text
当前首页没有更多链接 != 历史文章已经覆盖
```

生产系统必须把 URL 发现拆成多个 Provider：

```text
CMS API
Sitemap / Sitemap Index
RSS / Atom / JSON Feed
Archive 年/月页
Category / Pagination
Docs TOC
Common Crawl / URL Seeder 补洞
受限 Deep Crawl 补洞
```

每个 Provider 需要独立 cursor、exhaustion reason、覆盖统计和 gap 证据。

### 3.3 对当前方案的结论

当前方案已经采用 Coverage-first、多 Provider 的架构，不应退化为“抓列表页直到没有新链接”。本次调研进一步确认：**列表页 CSS Schema 应作为 Discovery Provider 的一种实现，而不是 Coverage 的最终证明。**

---

## 4. URL md5 + history：一个有效但不完整的增量同步模型

原文用 URL 的 md5 作为 `id`，再把历史记录读成：

```text
history[id] = item
```

新一轮抓取时，如果 id 已存在则跳过。

### 4.1 这个设计解决了什么

它解决的是最基础的“同一 URL 不重复首次处理”问题：

```text
URL -> deterministic id -> seen set
```

优点：

- 计算便宜；
- 幂等；
- 重启后可恢复；
- 本地文件即可工作；
- 非常适合 Demo 或单站日报。

### 4.2 它没有解决什么

**最大问题：把“URL 已见”误当成“内容永远不再变化”。**

博客文章可能：

- 修改正文；
- 修复代码；
- 更新标题；
- 增加图片；
- 修改发布日期；
- 迁移 URL；
- canonical URL 改变；
- 原 URL 301 到新 URL；
- 被删除后重新恢复。

原方案命中 history 后直接跳过，所以同 URL 的更新永远不会进入后续链路。

对于知识库，正确模型应为：

```text
URL identity
    !=
fetch freshness
    !=
content version identity
```

推荐三层判断：

```text
1. URL / canonical identity
2. HTTP artifact identity: ETag / Last-Modified / raw sha256
3. Canonical content identity: normalized IR / canonical sha256
```

对应增量逻辑：

```text
已知 URL
 -> conditional GET
 -> 304: 只更新检查证据
 -> 200 + raw hash unchanged: 不重抽
 -> 200 + raw changed
 -> extract
 -> canonical hash unchanged: 模板/广告变化，不建版本
 -> canonical hash changed: 建新 DocumentVersion
```

这比单纯 URL 去重要完整得多。

---

## 5. “昨日文章”过滤的时间窗口问题

原文最终只返回 `published_date == yesterday` 的记录，用于日报。

### 5.1 为什么这个逻辑在生产环境会漏数据

发布日期并不是可靠的增量 cursor：

- 网站时区可能不是任务时区；
- 页面发布时间可能只有日期没有时区；
- 文章可能延迟发布；
- 老文章可能当天更新但发布日期不变；
- Feed/CMS 可能晚到；
- 任务失败一天后补跑会直接错过窗口；
- 某些网站会回填旧日期文章。

### 5.2 更稳健的增量模型：watermark + overlap window

如果 Provider 支持时间过滤，应使用：

```text
query_start = durable_watermark - overlap
query_end   = now
```

例如：

```text
watermark = 2026-08-15T08:00:00Z
overlap   = 24h
下一轮仍回看 24h，再依赖 Observation 幂等吸收重复
```

只有在本轮数据和 cursor 都已持久化后，才能推进 watermark。

对于 Feed：

```text
GUID + link + published/updated + rolling overlap
```

对于 Sitemap：

```text
lastmod 只用于候选优先级，不作为绝对可信事实
```

对于 Archive：

```text
持续回扫最近 N 页 / N 天 + 定期 Coverage Verify
```

这个机制可以处理 late arrival、clock skew、任务中断和站点更新。

---

## 6. 抓取 Runtime 生命周期：原文最明显的性能瓶颈

原文的通用 `crawl(url)` 方法在每次抓一个 URL 时都创建一次 `AsyncWebCrawler` context：

```text
for url in urls:
    async with AsyncWebCrawler() as crawler:
        await crawler.arun(url)
```

### 6.1 为什么这会严重浪费资源

Crawl4AI 底层涉及浏览器、Context、Page、网络会话和 DOM 处理。频繁创建/销毁 Runtime 会产生：

- Chromium 启停成本；
- 内存抖动；
- Browser warm-up；
- TLS/连接池复用失败；
- Page 创建成本；
- 低吞吐；
- 高 crash 概率；
- 难以统一限流。

1000 站点、百万 URL 场景下不可接受。

### 6.2 正确的生命周期

应变成：

```text
Browser Process = Worker 级长生命周期
Context         = Source / 安全域隔离
Page            = Task 级短生命周期
```

静态页面更进一步：根本不应进入 Browser，优先 HTTPX/aiohttp。

### 6.3 当前 Crawl4AI 的批处理能力

当前 Crawl4AI 已提供 `CrawlerRunConfig`、`CacheMode`、`arun_many()`、多 URL 配置匹配、prefetch、dispatcher 等能力。原文中的 `bypass_cache=True` 属于旧式调用习惯；当前生产集成应把 Runtime 版本锁定，并通过 Adapter 统一封装当前 API。

尤其重要的是：**不能把 crawler 的并发能力直接等同于平台调度能力。**

平台仍需要：

```text
global permit
 -> domain permit
 -> source permit
 -> bounded lease
 -> arun_many 小批次
 -> per-URL result commit
```

这样才能避免某个大站一次占满所有 Browser slots。

---

## 7. 原文“异步”链路里混入同步 requests 的问题

原文在异步抓取函数中使用同步 `requests.get()` 下载图片。

### 7.1 技术问题

同步网络请求会阻塞 Python event loop：

```text
async task A --requests.get--> 阻塞线程
async task B/C/D 不能继续调度
```

同时原实现没有完整展示：

- connect/read timeout；
- 最大文件尺寸；
- MIME 校验；
- redirect 校验；
- 内容哈希；
- 重复图片去重；
- 对象存储；
- 失败重试；
- SSRF/Egress 约束。

### 7.2 对知识库的正确处理

媒体资源应成为独立 Asset Pipeline：

```text
IR IMAGE block
 -> AssetObservation
 -> Policy Gate
 -> async media fetch
 -> mime/size/hash validate
 -> object storage
 -> document_asset relation
```

建议表：

```text
asset
  id
  source_url
  final_url
  mime_type
  byte_size
  sha256
  object_key
  fetched_at
  status

document_asset
  document_version_id
  asset_id
  role
  original_url
  alt
  caption
```

这样 Markdown 可以选择：

- 保留原站绝对 URL；
- 导出时重写为本地 Asset；
- 对同一图片内容按 hash 去重。

---

## 8. 本地 JSON history 的复杂度和一致性问题

原文将整个站点历史存入 `data/{domain}.json`，新增数据时先更新内存字典，再重写整个文件。

### 8.1 小数据场景为什么可行

几十到几千条：

- 结构简单；
- 容易人工查看；
- 不依赖数据库。

### 8.2 百万记录场景为什么不可行

每次保存都全量重写，复杂度逐渐接近：

```text
单次保存 O(N)
持续新增累计 I/O 约 O(N²)
```

并且：

- 多 Worker 同时写会覆盖；
- 崩溃可能产生半文件；
- 无事务；
- 无唯一约束；
- 无 lease/fencing；
- 无历史版本；
- 无多维查询。

当前方案使用 PostgreSQL durable truth + Object Storage 的方向正确。

---

## 9. BeautifulSoup 定向抽取：站点 Recipe 的典型原型

原文针对 AIBase 手工写：

- `h1` 取标题；
- `span` + 正则找日期；
- 所有 `p` 拼正文；
- 遇到 `Key Points`/“划重点”截断；
- 特定图片规则取图。

这类代码的本质不是“通用爬虫”，而是**站点 Recipe**。

### 9.1 应怎样产品化

把硬编码拆为版本化配置：

```yaml
extract:
  title_selector: h1
  content_selectors: [article p]
  date:
    selectors: [time, span]
    patterns: [...]
  stop_markers: ["Key Points:", "划重点:"]
  remove_selectors: [...]
  image:
    selectors: [article img]
```

然后保留通用 Extractor Runtime。

### 9.2 为什么必须版本化

站点改版时：

```text
Recipe v1 -> Recipe v2
```

旧 Snapshot 可以离线用 v2 重抽，而无需重新访问源站。

这也是 Snapshot-first 架构的重要收益。

---

## 10. Swarm 多 Agent 流水线：逻辑上成立，但不适合作为主链

原文使用：

```text
Analysis Agent
 -> Translate Agent
 -> Classifier Agent
 -> Modifier Agent
 -> finish
```

### 10.1 这里真正需要的是“状态机”，不是“自由 Agent”

翻译、分类、摘要都是预定义步骤，执行顺序固定，实际上可以写成：

```text
TRANSLATE -> CLASSIFY -> SUMMARIZE
```

不需要模型在每一步重新决定“接下来调用哪个 Agent”。

用 Agent 编排会增加：

- 多轮 token；
- tool-call 失败点；
- prompt 漂移；
- 上下文膨胀；
- 难以精确幂等；
- 调试复杂度。

### 10.2 什么时候 Agent 才值得使用

只有步骤本身未知或需要条件规划时：

- 未知网页交互；
- 诊断为何抽取失败；
- 根据页面结构生成 Recipe 草稿；
- Research 工具调用。

批量正文的翻译/分类/摘要应该是普通 Job DAG。

---

## 11. 原文的 process_ids 给出了一个非常重要的生产启发

原文处理阶段会读取已经生成的结果，构造 `process_ids`，已有 id 就跳过，从而支持任务重启。

这是正确方向：**长链路必须逐条提交，而不是等全批完成后一次保存。**

但它只按文章 id 判断，仍然不够。

### 11.1 正确的派生任务幂等键

如果同一文章正文出现了新版本，必须重新处理；如果换了新模型、新提示词或新 Schema，也可能需要重建。

建议：

```text
ENRICH:
(document_version_id,
 enrichment_kind,
 enrichment_release_id)
```

例如：

```text
version v3 + summary + summary_release_7
version v3 + classify + classifier_release_4
version v3 + translate_zh + translator_release_2
```

这样：

- 模型升级可重跑；
- Prompt 升级可重跑；
- 旧结果可保留审计；
- Canonical Markdown 不受影响；
- 同一个 DocumentVersion 不会重复付费。

建议数据模型：

```text
enrichment_release
  id
  kind
  model_release_id
  prompt_release_id
  schema_release_id
  config_json
  status

enrichment_result
  document_version_id
  enrichment_release_id
  status
  result_json
  input_hash
  token_usage
  cost
  error
  created_at
```

---

## 12. LLM 输出解析：字符串切片非常脆弱

原文通过寻找“标题:”和“内容:”字符串位置来拆 LLM 输出。

风险：

- 模型多输出解释文字；
- 模型使用“标题：”全角冒号；
- 内容中再次出现“标题:”；
- 某字段缺失；
- 多语言格式变化。

生产系统应该直接要求结构化输出：

```json
{
  "title_zh": "...",
  "content_zh": "...",
  "category": "technical",
  "summary": "..."
}
```

然后用 Pydantic / JSON Schema 严格校验。

如果校验失败：

```text
INVALID -> retry with repair policy -> QUARANTINE
```

不能用字符串 index 异常让整条链路静默丢失。

---

## 13. 排序结果再用 Levenshtein 对齐，是错误身份模型

原文让 LLM 输出排序后的标题，再通过 Textdistance 的 Levenshtein 相似度把标题映射回原数据。

这是非常典型的“展示字段被误用为主键”。

### 13.1 错配场景

如果有：

```text
OpenAI 发布新模型 A
OpenAI 发布新模型 B
```

LLM 轻微改写或漏字后，模糊匹配可能映射到错误文章。

### 13.2 正确方式

排序输入直接带稳定 id：

```json
[
  {"id":"docv_123", "title":"..."},
  {"id":"docv_456", "title":"..."}
]
```

要求输出：

```json
{
  "ranked_ids": ["docv_456", "docv_123"]
}
```

**稳定 ID 必须贯穿整个流水线。** 标题、URL、slug 都是属性，不是跨阶段关联键。

这个原则也适用于：

- Chunk -> DocumentVersion；
- Embedding -> Chunk；
- Markdown -> Version；
- Asset -> Version；
- Enrichment -> Version；
- Export -> Version。

---

## 14. 摘要/改写不能进入 Canonical Markdown

原文 Modifier Agent 会重写标题并将正文压缩到摘要，用于日报是合理的；对于知识库则必须划清边界。

### 14.1 Canonical 层

必须尽可能忠实源站：

```text
源标题
源正文
代码
表格
引用
图片
链接
metadata
```

### 14.2 Enrichment 层

可以包含：

```text
中文翻译
摘要
主题分类
公司/产品实体
关键词
质量标签
重要性评分
知识图谱关系
```

### 14.3 Projection 层

搜索/RAG 可以按需要把 Canonical + Enrichment 组合成不同视图，但任何 LLM 派生都不能反向覆盖源事实。

---

## 15. 原文串行抓取问题与 1000 站调度的本质差异

作者自己也指出当前下载是串行，希望增加多线程和代理池。

对生产知识库来说，仅把 `for` 改成 `gather()` 仍然不够。

真正的问题是多维公平调度：

```text
全局吞吐
 + 每域名限速
 + 每 Source 并发
 + Browser slot
 + HTTP slot
 + 增量优先级
 + backfill 公平性
 + Retry-After
 + circuit breaker
```

例如 1000 站中某站发现 50 万 URL，不能让它压死其他 999 站的增量。

因此需要：

```text
PostgreSQL durable tasks
 -> Redis Streams transport
 -> bounded leasing
 -> hierarchical permits
 -> small batch execution
 -> per-result commit
```

代理池也不应作为“反爬绕过”的默认能力。429/403/CAPTCHA 应尊重站点策略、退避或进入人工处理，而不是自动换代理突破限制。

---

## 16. 从该文章得到的方案优化点

本次调研建议对现有博客知识库方案增加以下内容。

### 16.1 增加 Watermark + Overlap Incremental

现有方案已有 Provider cursor，但应明确：对于时间型 cursor，不使用单一“上次时间之后”硬切窗口，而使用可配置 overlap。

```text
provider_watermark
  provider_id
  committed_cursor
  committed_at
  overlap_seconds
```

执行：

```text
start = watermark - overlap
fetch/observe
commit observations + next cursor
advance watermark
```

目标是吸收晚到、回填、时区误差和任务中断。

### 16.2 明确“Seen URL 不是 Freshness State”

应把这一点写成增量同步硬约束：

```text
URL 已知只能跳过“首次发现”，不能跳过后续变更检查。
```

### 16.3 增加 Derived Enrichment 子系统

新增：

```text
enrichment_release
enrichment_result
enrichment_task
```

支持翻译、分类、摘要、标签、实体等异步派生任务；全部以 `document_version_id + release` 为幂等键。

### 16.4 增加 Asset Pipeline

图片/附件独立异步下载，做 MIME、大小、hash、SSRF/Egress、对象存储和去重，不允许在正文 async extractor 内用阻塞请求直接下载。

### 16.5 所有跨阶段输出使用 immutable ID

禁止依靠标题模糊匹配恢复原记录。排序、聚类、摘要、导出、RAG Citation 都直接引用 DocumentVersion/Chunk/Asset ID。

### 16.6 固定 AI 工作流优先使用 Typed DAG

批量翻译/分类/摘要不用自由 Agent 编排；Agent 只保留在未知交互、异常诊断、Recipe 生成等非确定场景。

---

## 17. 推荐的增强后处理链

```text
Snapshot / RenderArtifact
 -> Extractor
 -> Canonical IR
 -> DocumentVersion
 -> Deterministic Markdown
 -> outbox: DOCUMENT_VERSION_READY
       |
       +-> Chunk/Search/Embedding
       |
       +-> Asset Fetch
       |
       +-> Optional Enrichment DAG
             -> translate
             -> classify/tag
             -> summarize
             -> entity extraction
             -> ranking/curation view
```

每个分支独立失败、独立重试。

Canonical Version 创建成功不应因为摘要模型欠费而失败。

---

## 18. Web 管理层应增加的观测能力

基于原文处理流水线的经验，管理端建议补充：

### Enrichment

- 每个 DocumentVersion 的派生任务状态；
- 使用的模型/Prompt/Schema Release；
- token/cost；
- validation failure；
- 一键重建某个 Release；
- 新旧结果 Diff。

### Incremental

- Provider watermark；
- overlap window；
- late arrival 数量；
- known URL recheck 数量；
- 304/raw same/canonical same/new version 分布。

### Asset

- 图片/附件下载状态；
- 失败 URL；
- MIME/size policy block；
- hash 去重率；
- storage usage。

这些指标能直接回答系统是否在“看起来正常”但实际漏更新。

---

## 19. 当前 Crawl4AI API 与原文版本差异

原文是 2024 年实现，Crawl4AI API 已持续演进。当前官方资料中，生产调用更倾向：

```text
BrowserConfig
CrawlerRunConfig
CacheMode
AsyncWebCrawler.arun(..., config=...)
AsyncWebCrawler.arun_many(...)
```

并已提供多 URL config matching、prefetch、dispatcher、Deep Crawl state 等能力。

因此本项目不能把原文参数形态直接复制到生产代码，而应：

1. 锁定 Crawl4AI Runtime 版本和镜像 digest；
2. 由 `CrawlerAdapter` 吸收上游 API 变化；
3. Golden Corpus 回归后再升级；
4. 业务数据库不保存 Crawl4AI 私有 DTO；
5. 任何 crawler checkpoint 只是 Runtime state，不替代平台 Coverage/Observation 事实。

参考：

- https://github.com/unclecode/crawl4ai
- https://github.com/unclecode/crawl4ai/releases

---

## 20. 最终判断

这篇文章的价值不在于它本身能支撑 1000 站知识库，而在于它是一个非常清晰的“小规模原型”：

```text
发现 -> 去重 -> 抓取 -> 处理 -> 排序 -> 输出
```

它证明了 Crawl4AI + Schema Extraction 适合快速实现多站内容采集，同时也暴露出从 Demo 走向生产必须解决的关键问题：

- URL seen-set 不能承担版本同步；
- “昨日日期”不能承担可靠 cursor；
- Browser 不能按 URL 反复启动；
- async 中不能混入阻塞媒体请求；
- 本地 JSON 不能承担 durable truth；
- 固定步骤不需要自由 Agent；
- LLM 字符串输出必须改成 Schema；
- 标题不能作为跨阶段身份；
- AI 改写不能成为 Canonical Markdown；
- 派生处理必须有版本化 Release 和独立幂等状态。

对当前技术方案最有价值的新增内容是：**Watermark + Overlap 增量窗口、Derived Enrichment 事实模型、独立 Asset Pipeline、Stable-ID-only 跨阶段关联。** 这些优化可以显著降低漏更新、重复 LLM 成本、错配内容和媒体抓取阻塞风险，同时保持知识库 Canonical 数据的可回放性与长期可维护性。
