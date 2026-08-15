# Show HN：一个能从文章、视频和 PDF 中提取观点的 AI 卡片盒系统（Jargon）

## 1. 调研对象与结论

- Hacker News：https://news.ycombinator.com/item?id=46110897
- 当前项目仓库：https://github.com/jargon-io/jargon-v1
- 原发布链接曾指向 `schoblaska/jargon`，GitHub 当前重定向到 `jargon-io/jargon-v1`。
- 仓库当前已归档（archived），最后代码仍适合做架构与实现参考，但不应直接作为目标博客知识库的主系统依赖。
- `LICENSE.md` 明确声明 GPL-3.0；直接复用或修改代码需要评估 GPL-3.0 的传播义务。本方案主要吸收其架构思想和实现经验。

Jargon 不是“1000 个站点全量历史抓取平台”，而是一个 AI 管理的个人研究库。它把文章、论文/PDF、YouTube 等内容摄取进库，生成摘要和 Insight，通过向量相似度建立连接，再从文章/Insight 自动生成研究问题、调用 Web Search 找到更多来源并重新摄取，形成主动研究闭环。

它最值得吸收的不是爬虫能力，而是三点：

1. **把整篇文档进一步拆成可独立检索的观点对象**；
2. **把语义关系用于发现连接，而不是只做文档搜索**；
3. **从已有知识自动提出研究问题并扩展外部来源**。

但源码也暴露了生产级知识库最需要避免的四个问题：

1. 原文事实、AI 摘要、AI Insight 和用户自己的笔记边界不够强；
2. 向量 + LLM 相似判断会触发结构性 `absorb/reparent`，错误合并具有破坏性；
3. 搜索总结大量依赖“摘要的摘要、Insight 的再总结”，存在派生链递归漂移；
4. 模型生成的 `snippet` 并不天然等于原文证据，却可能在后续被当作“quote”继续使用。

因此目标博客知识库应把 Jargon 的能力放在 **Derived Knowledge / Semantic Relation / Research Expansion** 三个独立派生平面中，并增加严格的 **Evidence Binding、Grounding Policy、AI Validation/Repair、Annotation Projection**。

---

## 2. Jargon 的整体执行模型

核心链路可以概括为：

```text
URL / PDF / YouTube
 -> Article
 -> IngestArticleJob
 -> Web/Video/PDF content
 -> LLM content classification
 -> metadata + summary
 -> Article complete
 -> embedding
 -> similarity / absorb
 -> research questions
 -> GenerateInsightsJob
 -> Insight cards
 -> embedding / similarity / research questions
 -> SearchJob
 -> Exa web search
 -> LLM select candidates
 -> new Article
 -> ingest again
```

这是一个“摄取 -> 派生 -> 连接 -> 研究 -> 再摄取”的递归循环。

对个人研究库，这种链路体验很好；对 1000+ 技术博客的长期事实库，必须把它拆成两条互不阻塞的链：

```text
A. Managed Source Sync
Source -> Coverage -> Fetch -> Snapshot -> Canonical IR -> Document Version READY

B. Derived Research
Document Version READY -> Summary/Insight/Relation/Search -> External Evidence
```

B 链任何 LLM、Embedding、Search Provider 故障，都不能让 A 链已经成功保存的正文版本回退为失败。

---

## 3. IngestArticleJob：统一摄取编排器

核心文件：`app/jobs/ingest_article_job.rb`。

### 3.1 URL 类型分流

任务首先判断是否是 YouTube：

```text
YouTube
 -> YoutubeClient.fetch
 -> description + transcript
 -> LLM extract video metadata
 -> Article complete

普通 URL
 -> crawl_content
 -> evaluate_content
 -> full / abstract / partial / video / podcast / paywall / blocked
```

这是一种很实用的“内容类型路由”思路，但目标平台不能把 URL 后缀当作资源真相，应继续使用 MIME、响应头、magic bytes、跳转、HTML 壳和 Site Profile 做 Resource Probe。

### 3.2 Web 抓取是 Exa First，Crawl4AI Fallback

`crawl_content` 的实际逻辑：

```text
Exa crawl(url)
 -> results 非空且 text >= 500 chars
 -> 直接使用 Exa text

否则
 -> Crawl4AI CLI fallback
```

说明 Jargon 中 Crawl4AI 不是统一抓取层，只是内容不足时的 fallback。

`lib/crawl4ai_client.rb` 更薄：使用 `Open3.capture3` 启动类似：

```text
crwl <url> -o markdown-fit
```

成功即直接消费 stdout。

这对个人工具足够，但不适合目标规模，因为缺少：

- Durable Frontier；
- domain/source 级并发和速率预算；
- 长期 Browser Pool；
- immutable Snapshot；
- Crawl Runtime / Adapter Release；
- requested URL / final URL / redirect chain 的稳定平台契约；
- per-URL durable commit 和批任务 partial success；
- Coverage Evidence。

所以目标方案必须坚持 `Snapshot First + Stable Adapter + Bounded Window + Durable Task Store`，不能退化成同步 CLI 封装。

### 3.3 LLM 先判断内容质量/类型

`evaluate_content` 只把最多约 5000 字符交给模型，要求分类：

```text
full
partial
abstract
video
podcast
paywall
blocked
```

若为 abstract，还会让模型寻找 full-text / PDF / DOI 链接。

这个模式有启发：模型可以作为“复杂网页质量分类器”的一个信号。但不能把模型判断作为唯一事实。生产系统应把：HTTP 状态、正文长度、DOM 特征、付费墙 selector、文本重复率、内容类型、站点规则、模型分类共同写入 `quality_evidence`，再由版本化 policy 做最终决策。

### 3.4 PDF 处理

若 full-text URL 被判为 PDF，Jargon 用 HTTPX 下载到临时文件，再调用：

```text
pdftotext -layout
```

提取文本。

目标平台可以保留“原生 parser 优先、OCR fallback”的路线，但必须先持久化 PDF Snapshot，再从 Snapshot 离线解析；否则 parser 升级时无法重放。

### 3.5 元数据和摘要存在截断依赖

元数据抽取大约只使用正文前 10000 字符，摘要同样只使用最多约 10000 字符。

对个人研究足够，但对事实库存在风险：作者、发布日期或结论可能不在截断窗口内。标题/作者/发布时间应优先使用：

```text
CMS/API
JSON-LD
meta tags
HTML semantic fields
Feed metadata
site selectors
```

LLM 只能作为 fallback，并保存 `field_confidence + source_evidence + release_id`。

---

## 4. Article complete 之后：语义层被立即串到事实层

`finalize_article` 的顺序是：

```text
generate_embedding!
find_similar_and_absorb!
generate_searches!
GenerateInsightsJob.perform_later(article)
```

也就是说正文一完成，就立即触发语义聚类、自动研究问题和 Insight。

目标博客知识库必须改成：

```text
Document Version READY
 -> transaction/outbox
 -> async DERIVE
 -> async EMBED
 -> async RELATE
 -> optional RESEARCH
```

`DERIVE/EMBED/RELATE/RESEARCH` 的状态与 Source Sync 状态完全分离。

---

## 5. GenerateInsightsJob：观点卡片的生成机制

核心文件：`app/jobs/generate_insights_job.rb`。

任务把 `article.text` 交给 `StructuredChat`，要求结构化生成多个：

```text
title
body
snippet
```

每个 Insight 创建后又执行：

```text
embedding
 -> find_similar_and_absorb
 -> generate_searches
```

最后再安排 `AddLinksJob` 给 Insight 增加内部链接。

### 5.1 最大价值：把“文档检索”升级成“观点检索”

这值得吸收。用户搜索时可以直接命中一个观点，而不是必须先打开整篇文章。

生产数据模型应明确区分：

```text
SOURCE_FACT
  Snapshot / Canonical IR / Document Version

AI_DERIVATION
  Summary / Insight / FAQ / Entity / Topic / Question

USER_NOTE
  Comment / Counterpoint / Hypothesis / Manual Note
```

HN 讨论中有人指出，传统 Zettelkasten 的重要价值恰恰是用户自己组织、连接和形成思想。如果 AI Insight 与用户笔记共用同一身份模型，最终会无法区分：

- 作者原文说了什么；
- 模型认为作者说了什么；
- 用户自己形成了什么判断。

因此三类对象必须分离，权限、生命周期和审计也应分离。

### 5.2 一个更关键的问题：`snippet` 并不保证是原文引用

源码 Prompt 很重要：

- 普通文章：允许对 source excerpt 用 `...` 收紧；
- partial：要求“infer ONE key insight”；
- video/podcast：明确允许修正文法、标点、删除口头填充词，再作为 snippet。

所以 `snippet` 只是“模型整理后的证据展示文本”，不是天然可验证的 verbatim quote。

这直接决定目标系统不能只保存 `snippet` 字符串，然后把它当引用。

正确模型应是：

```text
knowledge_note
  body = 派生观点

evidence_span[]
  document_version_id
  block_id
  start_offset
  end_offset
  source_text_hash
  evidence_kind
  anchor_status
```

其中：

```text
evidence_kind = VERBATIM | NORMALIZED_TRANSCRIPT | PARAPHRASE
anchor_status = BOUND | AMBIGUOUS | UNBOUND
```

绑定流程：

```text
LLM output claim + evidence hints
 -> exact normalized match first
 -> block/offset resolve
 -> transcript-only normalization alignment when allowed
 -> fuzzy alignment only produces candidate
 -> persist evidence hash
 -> grounding policy decides whether citable
```

规则：

1. 标成 `VERBATIM` 的引用必须能够精确回到 Source Block；
2. 清理口语后的 transcript 只能是 `NORMALIZED_TRANSCRIPT`，不能伪装成逐字引语；
3. `PARAPHRASE` 必须显示为概括，而非引号；
4. `UNBOUND` Insight 可以保留为低可信派生物，但不能进入“可引用证据”通道。

这是从 Jargon 源码得到的最重要增强之一。

---

## 6. StructuredChat：结构化输出的“有限修复循环”

核心文件：`app/models/structured_chat.rb`。

实现包含：

```text
MAX_RETRIES = 2
LLM with schema
 -> validate JSON Schema
 -> validation failed
 -> 把具体 schema error 反馈给模型
 -> 再生成
 -> 最多修复 2 次
 -> 仍失败则抛 ValidationError
```

这是一个简单但正确的生产思想：**Prompt 约束不等于程序约束，模型输出必须通过机器校验；修复必须有上限。**

目标系统应把它升级成四级验证：

```text
Level 1 Schema
  JSON/Pydantic type/required/enum

Level 2 Referential
  document/version/block/target id 必须存在且在允许范围

Level 3 Grounding
  evidence span 必须能绑定原文；quote 必须逐字成立

Level 4 Semantic Invariant
  声称“只加链接/格式”的任务必须证明正文没有变化
```

失败分类建议：

```text
MODEL_SCHEMA_ERROR
INVALID_REFERENCE
UNGROUNDED_EVIDENCE
INVALID_TARGET
INVARIANT_VIOLATION
REPAIR_EXHAUSTED
```

最多做少量 bounded repair；超过阈值进入 quarantine，不能无限重试消耗模型预算。

---

## 7. Embedding、Sameness 与 Parentable：为什么不能直接复制

### 7.1 Embedding 只是候选召回

`Embeddable` 对 Article 使用 summary，对 Insight 使用 body 生成向量。

Article 的 `parent_matching threshold: 0.3` 会先按 cosine distance 找近邻，再调用 `SamenessCheck`。

`SamenessCheck` 继续组合：

```text
embedding candidate
 + normalized title Levenshtein
 + LLM 判断 same underlying work
```

标题相似阈值大致：

- 默认约 0.7；
- embedding 极近时可放宽至约 0.5。

组合召回比单一向量稳健，但仍是非确定性证据。

### 7.2 destructive absorb 的生产风险

`Parentable` 一旦判断相同，会把对象吸收到 synthesized parent 下，还可能通过 `reparent_references` 改写：

- Search 的 source；
- Insight 的 article；
- SearchArticle 的 article。

如果误判，下游引用已经被重写。

对长期事实库，正确做法是：

```text
semantic_relation
- SAME_WORK
- REPUBLISHED
- TRANSLATION_OF
- RELATED
- CONTRASTS
- SUPPORTS
```

并保存独立 evidence：

```text
EXACT_HASH
NEAR_HASH
CANONICAL
REDIRECT
CMS_ID
FEED_GUID
EMBEDDING
TITLE
LLM
```

只有稳定、可验证的 Identity Evidence 才允许影响 Document Identity。Embedding、标题和 LLM 只生成 Relation Candidate。

UI 可以把转载内容“折叠显示”，但这是可逆 Projection，不是物理合并。

---

## 8. AddLinksJob：非常值得吸收的“语义不变量校验”

核心文件：`app/jobs/add_links_job.rb`。

它让 LLM 给 Article summary、Insight body/snippet、Search summary/snippet 增加内部 Insight 链接，但 Prompt 明确要求：

```text
只加链接
不得修改任何原有文字
只能链接到给定 target
```

更关键的是它没有只相信 Prompt，而是做了程序校验：

```text
strip_links(original)
==
strip_links(linked)
```

同时检查所有生成的内部链接目标都在显式 whitelist 中。若不满足，直接放弃这次变换。

这是一个很好的设计模式：

> **当 AI 声称只做“非语义变换”时，必须用确定性 invariant 证明它真的没有改变语义载体。**

目标知识库应进一步优化：不要把“加内部链接”直接写回 `knowledge_note.body`，而是单独保存 Annotation Projection：

```text
knowledge_link_annotation
- note_id
- source_field
- start_offset
- end_offset
- target_note_id
- relation_type
- annotation_release_id
- input_text_hash
- validation_state
```

渲染时再把 annotation 覆盖到原始文本上。这样：

- 添加/删除链接不会改变 Note 的 base content hash；
- Relation Release 升级可以重新建链接；
- 错误链接可整体回滚；
- 不会把展示层 markup 误当作知识事实。

若任何场景必须生成 inline markup，至少验证：

```text
strip_allowed_markup(output) == normalized_input
all_targets in allowed_target_manifest
no offset overlap / broken markup
input_hash unchanged
```

---

## 9. SearchGeneratable 与 SearchJob：主动研究闭环

`SearchGeneratable` 默认 `MAX_SEARCHES = 2`，让模型从 Article/Insight 上下文生成研究问题。

`SearchJob`：

```text
research question
 -> LLM 生成更短 search query
 -> Exa search
 -> 取前 10 个 title/url
 -> LLM 选 1~3 个，Prompt 要求 prefer diversity
 -> Article.find_or_create_by(url)
 -> origin = discovered
 -> Article after_create_commit 自动执行 IngestArticleJob
```

这就形成递归扩展。

### 9.1 研究扩展必须与 Managed Source 分离

目标系统应建立独立：

```text
Research Expansion Run
- max_depth
- max_fanout
- max_searches
- max_new_urls
- max_new_domains
- max_cost
- allowed/disallowed domains
- novelty/diversity policy
- stop_reason
```

搜索发现的 URL 默认只是 `EXTERNAL_RESEARCH`。只有显式 `Promote to Source` 后才进入：

```text
PROBE -> Profile -> FULL_BACKFILL -> INCREMENTAL SLA
```

不能让自动搜索绕过 robots、SSRF、域级限流、Source Profile、版权策略和 Coverage 体系。

### 9.2 只看 title/url 选“最佳文章”太浅

`SearchJob` 的候选筛选阶段只给模型 title + URL，并没有先读取候选正文。

这是一种成本很低的 coarse selection，但不能作为高可信 admission。

目标系统建议四段式：

```text
Stage 0 Search provider rank/title/url/domain
 -> Stage 1 policy/security/dedupe cheap filter
 -> Stage 2 controlled fetch + extract sample/source blocks
 -> Stage 3 relevance + novelty + diversity + authority/evidence score
 -> External Evidence / Reject / Promote Candidate
```

这样可以避免标题党或摘要错误让不相关来源进入后续知识生成。

### 9.3 Job claim 机制不适合平台级可靠任务

`SearchJob#claim_job?` 只是把 `pending/searching` 状态 update 为 `searching`，没有 lease、lease timeout、fencing token，也把已经 `searching` 的记录纳入 claim 范围。

目标平台继续使用 PostgreSQL durable task：

```text
PENDING -> LEASED -> RUNNING -> SUCCEEDED / RETRY_WAIT / FAILED_TERMINAL
```

并包含 `lease_owner / lease_until / fencing_token / attempt / checkpoint / budget_reservation`。

---

## 10. SummarizeSearchJob：最需要防范的“派生套派生”漂移

核心文件：`app/jobs/summarize_search_job.rb`。

它构造回答上下文时，并没有把选中文章的原始正文或 Canonical Block 放进去，而主要拼接：

```text
Article.title
Article.summary          <- 已经是 LLM 派生
Insight.title/body       <- 又是 LLM 派生
Related Article.summary  <- 派生
Related Insight.body     <- 派生
```

然后再让 LLM：

```text
synthesize search results
produce summary
produce a key quote
produce follow-up queries
```

这里形成了典型链：

```text
Source
 -> AI Summary / Insight
 -> AI Search Synthesis
 -> Follow-up Question
 -> Search
 -> New AI Summary / Insight
 -> ...
```

如果没有重新回到源文证据，误差会像“摘要的摘要”一样逐层放大。更危险的是 Prompt 还要求生成“key quote”，但上下文里可能只有 AI summary/Insight，根本没有原始 quote 可以验证。

因此目标方案必须增加原则：

> **Derived Knowledge 可以用于 query planning、candidate expansion、ranking、context compression，但不能成为高可信回答/引文的唯一证据。**

建议增加多输入谱系：

```text
derivation_input
- derivation_run_id
- input_type: SOURCE_BLOCK | AI_NOTE | USER_NOTE | SEARCH_RESULT
- input_id
- input_hash
- role: EVIDENCE | CONTEXT | QUERY_EXPANSION | RANKING_SIGNAL
```

以及：

```text
knowledge_note.grounding_state
  GROUNDED | PARTIAL | UNGROUNDED

knowledge_note.derivation_depth
```

`derivation_depth` 用于发现“摘要的摘要”链是否过深。

高可信/citable 模式应要求：

```text
Evidence Floor = SOURCE_BLOCK_REQUIRED
```

回答生成前执行 Evidence Assembler：

```text
Derived Note / Search hit
 -> resolve source_document_version + source_block_refs
 -> retrieve original Canonical IR blocks
 -> assemble context with explicit provenance
 -> model answer
 -> claim/evidence validator
```

AI Note 仍然可以帮助召回，但最终证据必须回源。

---

## 11. SimilarItemsQuery：语义连接适合做“发现”，不适合做“真相”

`SimilarItemsQuery` 同时从 complete root Article 与 Insight 中按 cosine distance 找近邻，阈值约 0.5，然后把两类结果混合排序。

这种设计对“意外连接”体验很好，但生产系统要保留：

- 原始 distance/score；
- embedding release；
- candidate generation release；
- relation evidence；
- object type；
- source/version provenance。

不能只返回“相关对象”而丢掉为什么相关。

HN 中关于 eventual consistency / strong consistency 是否会被模型错误归为“相同思想”的担忧也说明：相似度只能证明语义接近，不能证明关系方向。目标系统 Relation Type 必须允许 `RELATED`、`CONTRASTS`、`SUPPORTS` 等区别，并允许人工确认。

---

## 12. 对目标 1000+ 技术博客平台的最终融合设计

Jargon 的能力应放置如下：

```text
Managed Source Plane
  Source/Profile/Coverage/Frontier
  -> Fetch/Snapshot
  -> Canonical IR
  -> Document Version READY

Retrieval Plane
  -> Chunk/BM25/Vector

Derived Knowledge Plane
  -> Summary
  -> Insight
  -> User Note
  -> Question
  -> Evidence Binding
  -> Grounding Validation

Semantic Relation Plane
  -> candidate generation
  -> evidence accumulation
  -> reviewable relation

Research Expansion Plane
  -> bounded query generation
  -> search
  -> controlled candidate fetch
  -> relevance/novelty/diversity
  -> External Evidence
  -> explicit Promote to Source
```

### 12.1 推荐核心数据结构

```text
derivation_run
- id
- derivation_type
- knowledge_derivation_release_id
- model_release_id
- prompt_release_id
- validation_release_id
- grounding_policy_release_id
- input_manifest_hash
- output_manifest_hash
- status
- repair_attempts
- token/cost/latency/tool trace


derivation_input
- derivation_run_id
- input_type
- input_id
- input_hash
- role

knowledge_note
- id
- origin: AI | USER
- note_type
- title
- body
- source_document_version_id nullable
- grounding_state
- derivation_depth
- derivation_run_id nullable
- state
- content_hash

knowledge_evidence
- knowledge_note_id
- document_version_id
- block_id
- start_offset
- end_offset
- source_text_hash
- evidence_kind
- anchor_status
- alignment_release_id

knowledge_link_annotation
- note_id
- source_field
- start_offset
- end_offset
- target_note_id
- relation_type
- annotation_release_id
- input_text_hash
- validation_state
```

### 12.2 AI 生成统一验证流程

```text
Generate
 -> JSON/Pydantic Schema Validate
 -> Referential Validate
 -> Evidence Bind
 -> Grounding Validate
 -> Semantic Invariant Validate
 -> bounded repair <= N
 -> READY / PARTIAL / QUARANTINED
```

Schema 正确只是最低门槛，不能等价于“内容可信”。

### 12.3 搜索回答必须回源

推荐：

```text
BM25 + Vector + Relation candidate
 -> candidate notes/chunks
 -> resolve original Source Blocks
 -> Evidence Assembler
 -> answer model
 -> claim-to-evidence validation
 -> response with block-level provenance
```

代码符号、函数名、版本号、RFC/CVE、错误信息必须保留 BM25/field lexical route，不能只依赖 Embedding。

---

## 13. Web 管理功能需要新增的 Jargon 启发项

在现有 Web 管理面基础上增加：

1. **Evidence Inspector**：Insight/回答逐条显示绑定的 Document Version、Block、offset、原文 hash、evidence kind；
2. **Grounding Status**：GROUNDED/PARTIAL/UNGROUNDED，可筛选未绑定派生物；
3. **Derivation Lineage**：显示一条 Note 是直接来自 Source Block，还是来自 Summary/Insight/Search 的多层派生；
4. **AI Validation Quarantine**：查看 schema/reference/grounding/invariant 失败与 repair attempt；
5. **Annotation Review**：内部链接作为独立 Projection，可查看 target whitelist 和 invariant 校验结果；
6. **Relation Review**：展示 exact/canonical/embedding/title/LLM 等多类证据，支持 Confirm/Reject；
7. **Research Run**：显示深度、fanout、候选域多样性、正文复核结果、预算和 stop reason；
8. **Promote to Source**：外部研究来源显式转为 Managed Source 后再走完整 PROBE。

---

## 14. 必须增加的回归测试

### 14.1 Quote / Evidence Grounding

构造：

- 原文逐字引用；
- 用 `...` 压缩的 excerpt；
- 删除“um/uh”等口语后的 transcript；
- 模型完全改写的 paraphrase；
- 一个无法在原文中找到的伪 quote。

验证：

- 只有精确匹配可标 `VERBATIM`；
- transcript 清理标 `NORMALIZED_TRANSCRIPT`；
- paraphrase 不渲染为 quote；
- UNBOUND 不进入 citable 通道。

### 14.2 Summary-of-Summary Drift

只给模型 Article.summary + Insight.body，不给 Source Block，要求产生“quote”。系统必须拒绝把结果标记为 source-grounded。

再通过 Evidence Assembler 加载原始 block 后，才允许形成可引用回答。

### 14.3 AI Transform Invariant

模拟 AddLinks：

- 只增加合法链接 -> 通过；
- 偷改一个单词 -> 拒绝；
- 指向 whitelist 外 target -> 拒绝；
- 链接 offset 重叠/损坏 -> 拒绝。

### 14.4 Structured Output Repair

- 第一次 JSON 不符合 schema；
- 第二次修复成功；
- 连续失败超过 repair 上限；
- schema 通过但 evidence 引用不存在；
- schema 通过但语义 invariant 失败。

验证只有前两类合规结果进入 READY，其余进入 quarantine，且不无限重试。

### 14.5 Research Admission

构造标题高度相关但正文无关的搜索结果。Stage 0 title/url 可以进入候选，但 Stage 2 Fetch + Stage 3 evidence scoring 后必须被拒绝。

### 14.6 Relation Safety

测试同文转载、同标题不同正文、同主题相反观点、多语言翻译。Embedding/title/LLM 只产生 Relation Candidate，不能单独做 destructive merge。

---

## 15. 最终判断

Jargon 不适合作为目标系统的 crawler 主体，但非常值得作为“知识研究体验层”的参考。

应吸收：

1. Article -> Insight 的观点化知识表示；
2. Embedding 驱动的跨对象连接；
3. Research Question -> Web Search -> 新来源的主动研究循环；
4. `StructuredChat` 的 schema 校验 + bounded repair；
5. `AddLinksJob` 的确定性 invariant 校验思想。

必须重构：

1. **AI Insight 必须独立于 Source Fact**；
2. **Snippet 必须升级为可验证 Evidence Binding，不可默认当原文 quote**；
3. **派生知识可以辅助召回，但高可信回答必须回到 Source Block，阻止 summary-of-summary 漂移**；
4. **语义相似只生成可逆 Relation，不执行破坏性 absorb**；
5. **内部链接/高亮作为 Annotation Projection，不修改基础知识内容**；
6. **自动研究必须有预算、深度、fanout、来源多样性、正文复核与显式 Promote**；
7. **结构化输出除 Schema 外还要做引用、证据、语义不变量验证，失败有限修复后隔离**。

最终目标不是复制 Jargon，而是把它优秀的“观点卡片 + 连接 + 主动研究”能力建立在一个可审计、可回放、可扩展的 Source Truth 平台之上。