# Show HN：一个能从文章、视频和 PDF 中提取观点的 AI 卡片盒系统（Jargon）

## 1. 调研对象与结论

- Hacker News：https://news.ycombinator.com/item?id=46110897
- 当前项目仓库：https://github.com/jargon-io/jargon-v1
- 原发布链接曾指向 `schoblaska/jargon`，GitHub 当前重定向到 `jargon-io/jargon-v1`。
- 仓库已归档，适合研究其实现思想，不适合作为目标博客知识库的长期主系统依赖。
- `LICENSE.md` 明确声明 GNU GPL-3.0；若直接复用/修改其代码需要遵守相应开源义务。目标方案主要吸收思想与实现经验，不依赖复制其代码。

Jargon 的定位不是“1000 个站点全量历史抓取平台”，而是一个 AI 管理的个人研究库：文章、论文/PDF、YouTube 被摄取后生成摘要和 Insight 卡片，再利用向量相似度连接相关对象；文章和 Insight 又会自动生成研究问题，调用 Web Search 找到更多来源并重新进入同一摄取链，形成“摄取 → 派生 → 连接 → 研究 → 再摄取”的循环。

对目标博客知识库，最值得吸收的能力有四类：

1. **文档之上再建立可独立检索的 Insight Card**，让知识颗粒度从“文章”下沉到“观点”；
2. **Embedding 不只用于问答检索，也用于关系候选和知识探索**；
3. **从已有知识自动生成研究问题，形成主动研究闭环**；
4. **对模型输出做结构化 Schema 校验，并在失败时做有限次数修复**。

但源码同时暴露出生产知识库必须规避的核心风险：

1. 原文事实、AI 摘要、AI Insight、用户笔记之间的边界不足以支撑严格审计；
2. 向量 + 标题 + LLM 的相似判断会触发结构性 `absorb/reparent`，误判后会改写下游引用；
3. Research Search 的总结主要继续消费 Article Summary / Insight，而不是回到原文，容易产生“摘要的摘要”漂移；
4. `snippet` 在部分场景明确允许删减、清理口语甚至“推断”，并不天然是原文 quote；
5. 自动研究没有生产级 depth/fanout/cost/domain budget，递归扩展存在失控风险；
6. YouTube 字幕被拼成整段文本并丢失时间戳，后续无法做可验证的时间轴证据引用；
7. 抓取、内容判别、元数据、摘要、Embedding、聚类、自动研究被串在一个面向个人应用的工作流中，事实摄取与 AI 派生故障域没有充分隔离。

因此目标系统应保留 Jargon 的知识派生思想，但把它放到 **Derived Knowledge / Semantic Relation / Research Expansion** 三个独立、可重建的派生平面，并补上 **Snapshot First、Evidence Binding、Grounding、非破坏关系、预算控制、多模态定位器、Release/Audit**。

---

## 2. 项目技术栈与架构取向

README 给出的主要技术栈：

- Rails + Hotwire：Web 应用；
- Falcon：异步 Ruby 应用服务器；
- async-job：后台任务；
- RubyLLM：统一 LLM Provider；
- ruby_llm-schema：结构化输出；
- PostgreSQL + pgvector：业务数据与向量检索；
- Exa：神经搜索与网页内容获取；
- Crawl4AI：网页抓取 fallback；
- `pdftotext`：PDF 文本提取；
- 自定义 `YoutubeClient`：视频 metadata + captions。

`CLAUDE.md` 体现了其代码组织原则：ActiveRecord Rich Domain Model，Concern 表达模型 trait，复杂算法用 `Article::SamenessCheck` 等 namespaced class 隔离，Controller 直接调用 Model，不额外引入 service/repository 层。

这对小型一体化个人应用很简洁，但目标平台是 1000+ Source、长期 Backfill/Incremental、多个资源类型与独立 Worker 池，不能照搬其进程/模型边界。目标系统更适合：

```text
API/Control Plane
 -> Durable Task Store
 -> Discovery Worker
 -> HTTP Fetch Worker
 -> Browser Worker
 -> Extract Worker
 -> Projection Worker
 -> Derived Knowledge Worker
 -> Research Worker
```

业务真相留在 PostgreSQL/Object Storage，Crawler、Redis、LLM Job 都只是执行层。

---

## 3. Jargon 的完整执行链

从代码可以还原出主链：

```text
URL
 -> Article.create!
 -> after_create_commit
 -> IngestArticleJob

IngestArticleJob
 -> YouTube ? YoutubeClient : Web/PDF pipeline
 -> content classification
 -> metadata
 -> summary
 -> Article complete
 -> embedding
 -> find_similar_and_absorb
 -> generate research questions
 -> GenerateInsightsJob

GenerateInsightsJob
 -> Insight[]
 -> each Insight embedding
 -> each Insight similarity/absorb
 -> each Insight research questions
 -> AddLinksJob

Search
 -> SearchJob
 -> Exa search
 -> LLM choose candidate URLs
 -> Article.find_or_create_by(url)
 -> candidate Article automatically Ingest
 -> SummarizeSearchJob
 -> summary + snippet + follow-up questions
 -> more Search
```

其本质是递归知识增长系统，而不是采集平台。

对于目标博客知识库，必须拆成两条互不阻塞的链：

```text
A. Managed Source Sync
Source -> Discovery -> Coverage -> Fetch -> Snapshot -> Canonical IR -> Document Version READY

B. Derived Knowledge
Document Version READY -> Summary/Insight -> Embed -> Relation -> Optional Research
```

A 链完成后事实已经安全落地；B 链任何 LLM、Embedding、Search Provider 故障都不能让 Document Version 失败或丢失。

---

## 4. IngestArticleJob：把所有工作集中到一条摄取流水线

核心文件：`app/jobs/ingest_article_job.rb`。

### 4.1 URL 分流

入口先判断是否为 YouTube：

```text
YoutubeClient.youtube_url?
  true  -> process_youtube
  false -> process_web_content
```

普通 Web 流程随后由 LLM 把内容分类：

```text
full
partial
abstract
video
podcast
paywall
blocked
```

若是 `abstract`，模型还被要求找 full-text/PDF/DOI 链接。

这个“统一资源路由 + 质量分类”思路值得保留，但生产实现不能让 LLM 单独决定资源类型。目标平台应把以下信号共同保存为 `resource/quality evidence`：

- HTTP status；
- Content-Type；
- magic bytes；
- URL/redirect；
- DOM 结构；
- 内容长度；
- 站点 Profile；
- paywall/captcha selector；
- parser 质量分；
- 模型分类。

最终由版本化 Policy 判定 `HTML/PDF/TRANSCRIPT/UNSUPPORTED` 和 `READY/PARTIAL/BLOCKED`。

### 4.2 Exa First，Crawl4AI Fallback

`crawl_content` 的实际顺序：

```text
crawl_with_exa(url)
 -> Exa /contents
 -> results 非空且 text >= 500
 -> 使用 result["text"]

否则
 -> Crawl4aiClient.crawl(url)
```

`lib/crawl4ai_client.rb` 很薄，直接：

```text
Open3.capture3("crwl", url, "-o", "markdown-fit")
```

成功就消费 stdout。

这说明 Crawl4AI 在 Jargon 中只是 CLI fallback，而不是具有全局调度、Coverage、Snapshot、Browser Pool 的基础平台。

个人应用这样做成本低，但不能支撑目标规模，因为缺少：

- Durable Frontier；
- domain/source/global 限流；
- Browser 生命周期管理；
- requested/final URL 与 redirect chain；
- immutable Snapshot；
- Adapter Release；
- per-URL commit；
- retry/fencing；
- Coverage Evidence。

目标方案应坚持 `Adapter Boundary + Snapshot First + Bounded Window + Durable Task Store`，Crawl4AI 只能是 Fetch Adapter。

### 4.3 Exa 请求里存在可避免的重复派生

`ExaClient#crawl` 调 `/contents` 时不仅请求 `text: true`，还请求一个 200～300 字符的 summary；但 `IngestArticleJob` 最后又使用自己的 `generate_summary(text)` 重新生成 Article Summary。

也就是说从代码路径看，Exa 返回的 summary 没有成为 Article Summary 的事实来源，却仍可能产生额外远程处理成本。

对目标平台的启发：

- Fetch Adapter 只获取所需网络/正文事实；
- AI Summary 放到统一 Derivation Worker；
- 不让第三方抓取服务的“顺便摘要”静默进入事实层；
- 未使用的远程派生字段不要请求，以减少成本和不可控差异。

---

## 5. Web 内容质量判别：LLM 可以是信号，不能是事实裁判

`evaluate_content` 只把最多约 5000 字符交给模型分类。

优势：

- 对多站点复杂模板有一定泛化；
- 能识别抽象概念，如 abstract、paywall、partial；
- 能顺便寻找 full-text URL。

局限：

1. 前 5000 字符不一定代表整页；
2. Prompt/model 升级会改变分类；
3. 网络错误页和正文可能被模型误判；
4. 无结构化网络证据，很难审计为什么某次是 blocked/partial；
5. 对 1000 站每 URL 都调用 LLM 成本不必要。

生产策略应是：

```text
deterministic rules
 -> HTTP/DOM/content heuristics
 -> site-specific profile
 -> only uncertain cases use classifier model
 -> quality evidence persisted
```

LLM 是 expensive fallback，不是所有网页的第一层 parser。

---

## 6. PDF：实现简单，但缺少 Snapshot 与页级证据

`extract_pdf_text`：

1. HTTPX follow redirects 下载 PDF；
2. 写入 Tempfile；
3. `pdftotext -layout file -`；
4. stdout 作为全文。

该方式对文本型论文足够有效，但生产知识库需要增强：

```text
HTTP stream
 -> immutable PDF Snapshot
 -> MIME/magic/size validation
 -> native parser
 -> per-page blocks
 -> scanned-page detection
 -> OCR fallback only for scanned pages
 -> IR with page/bbox locator
```

为什么必须 Snapshot First：

- parser 升级后可以离线重跑；
- 原 PDF 消失时仍保留来源事实；
- quote 可以定位到页码；
- OCR/原生 parser 的结果可以比较，而不是只有一个最终字符串。

PDF 对本项目不是核心博客入口，但作为 Research External Evidence 很有价值，应支持但不混入 Managed Blog Coverage。

---

## 7. YouTube：最大的缺口是时间轴证据丢失

核心文件：`lib/youtube_client.rb`。

实现：

1. 用正则识别常见 YouTube URL；
2. 规范化为 `youtube.com/watch?v=...`；
3. 调用 Innertube player endpoint；
4. 获取 title/channel/publishDate/description；
5. 取第一个 caption track；
6. 请求 transcript URL；
7. 用 `<text>...</text>` 提取文本；
8. HTML unescape 后把所有 segment 用空格拼成一个大字符串。

优点是无需 Browser 就能获得公开视频字幕。

关键问题是第 8 步把 transcript 扁平化：每段 caption 的时间戳、segment identity、可能的 speaker 信息没有保存。后续模型生成 Insight 时，即使语义正确，也无法告诉用户“这句话发生在 23:17～23:32”。

目标平台应保存：

```text
transcript_segment
  segment_id
  start_ms
  end_ms
  speaker
  raw_text
  raw_text_hash
```

如果需要清理口语，再产生：

```text
normalized_text
normalization_map(raw offsets <-> normalized offsets)
```

Evidence Span 可因此表达：

```text
evidence_kind = NORMALIZED_TRANSCRIPT
start_ms = 1397000
end_ms = 1412000
raw_segment_ids = [...]
```

这比只保存清理后的 snippet 更适合生产级知识引用。

---

## 8. 元数据和摘要：截断意味着需要确定性字段优先

普通 Web 内容中：

- 元数据抽取只输入 `text.truncate(10_000)`；
- Summary 也只输入 `text.truncate(10_000)`。

YouTube metadata 也只取 description 和 transcript excerpt。

这对个人应用速度友好，但目标平台不能让文章标题、作者、发布时间主要依赖 LLM 截断推测。

字段优先级应为：

```text
CMS/API
 -> Feed metadata
 -> JSON-LD
 -> OpenGraph/meta
 -> HTML semantic selector
 -> source profile selector
 -> LLM fallback
```

每个字段保存：

```text
value
source_kind
source_locator
confidence
extractor_release_id
```

LLM fallback 只能补缺，不能覆盖高可信结构化字段而不留证据。

---

## 9. finalize_article：事实层与语义层耦合过紧

Article 完成后立即：

```text
generate_embedding!
find_similar_and_absorb!
generate_searches!
GenerateInsightsJob.perform_later(article)
```

这使“正文摄取完成”紧接着进入多个 AI/Embedding/Research 副作用。

生产系统应改成：

```text
DocumentVersion READY
 -> transactional outbox
    -> MARKDOWN projection
    -> SEARCH projection
    -> EMBED
    -> DERIVE_SUMMARY
    -> DERIVE_INSIGHT
    -> RELATE
    -> optional RESEARCH
```

每一类派生都有独立状态、Release、retry/quarantine，不修改 Version READY。

---

## 10. Embedding：简单、有效，但索引对象和版本必须更严格

`Embeddable` 只是：

```text
text = configured field
LLM.embed(text)
update embedding vector
```

Article 配置 `embeddable :summary`；Insight 配置 `embeddable :body`；Search 也对 summary/search query 做向量。

优点：

- 低复杂度；
- 利用 pgvector 就能完成近邻发现；
- 同一 Concern 可复用。

问题：

- Article 只嵌入 AI Summary，Embedding 的语义会继承 Summary 的遗漏/偏差；
- 没有明确 embedding release/version 字段；
- 模型更换时需要全量重建；
- 技术文本里的 API 名、版本号、错误字符串并不适合只靠语义向量。

目标方案：

```text
Block/Chunk embedding = 原始规范化正文
KnowledgeNote embedding = 派生知识
Document summary embedding = 可选 query planning signal
```

不同对象、模型、chunk recipe 都通过 `embedding_release_id` 版本化；检索采用 BM25 + Vector hybrid。

---

## 11. GenerateInsightsJob：最值得吸收的“观点卡片”设计

任务对全文调用 StructuredChat，生成多个：

```text
title
body
snippet
```

每个 Insight 随后又执行：

```text
embedding
 -> find_similar_and_absorb
 -> generate_searches
```

这把“文章级检索”升级为“观点级检索”，是 Jargon 对目标方案最有价值的启发之一。

目标平台应该正式引入：

```text
knowledge_note
  origin = AI | USER
  note_type = SUMMARY | INSIGHT | FAQ | HYPOTHESIS
  source_version_id
  body
  evidence_span_ids[]
  derivation_release_id
  status
```

并严格分离三层：

```text
SOURCE_FACT
  Snapshot / Canonical IR / Document Version

AI_DERIVATION
  Summary / Insight / FAQ / Entity / Research Question

USER_KNOWLEDGE
  Note / Comment / Counterpoint / Hypothesis
```

如果这三层共用一个“Note”身份，未来无法回答“这是作者说的、模型生成的还是用户自己的判断”。

---

## 12. `snippet` 不是天然引用：必须做 Evidence Binding

GenerateInsights 的 Prompt 直接揭示了风险：

- 普通 Article：source excerpt 可用 `...` 收紧；
- Partial：要求“infer ONE key insight”；
- Video/Podcast：允许修正文法、标点、删除 `um/uh/like/you know` 等 filler。

所以 `snippet` 至少可能有三种语义：

```text
逐字摘录
经过删节的摘录
经过口语清理后的 transcript
模型概括/推断
```

不能只保存一段字符串再统一展示成引号。

正确模型：

```text
evidence_span
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
LLM claim + evidence hint
 -> exact normalized match
 -> block/offset resolve
 -> transcript alignment when allowed
 -> fuzzy match only as candidate
 -> persist source hash + locator
 -> grounding policy
```

展示策略：

- VERBATIM 才使用 quote 样式；
- NORMALIZED_TRANSCRIPT 明示“字幕整理”；
- PARAPHRASE 明示“概括”；
- UNBOUND 不允许成为高可信引用。

---

## 13. StructuredChat：有限修复循环是正确方向

核心文件：`app/models/structured_chat.rb`。

逻辑：

```text
MAX_RETRIES = 2
LLM with schema
 -> ask
 -> JSON Schema validate
 -> validation failed
 -> 把具体错误反馈给模型
 -> retry
 -> 超过次数抛 ValidationError
```

它体现了两个重要原则：

1. Prompt 不等于程序约束，必须机器校验；
2. repair 必须有上限，不能无限烧 token。

目标系统应在此基础上扩展成四级验证：

```text
Level 1 Schema
  JSON/Pydantic type/required/enum

Level 2 Referential
  version/block/target id 存在且在允许范围

Level 3 Grounding
  evidence span 能绑定，verbatim quote 逐字成立

Level 4 Semantic Invariant
  “只加链接/格式”等操作必须证明正文未改变
```

错误分类：

```text
MODEL_SCHEMA_ERROR
INVALID_REFERENCE
UNGROUNDED_EVIDENCE
INVALID_TARGET
INVARIANT_VIOLATION
REPAIR_EXHAUSTED
```

最终失败进入 Quarantine，保留输入、模型、Prompt Release、错误和成本。

---

## 14. AddLinksJob：一个值得保留的“语义不变量校验”范式

`AddLinksJob` 要求 LLM 在已有文本上加内部 Markdown link，并明确“不改变任何词”。

代码没有只相信 Prompt，而是：

1. 生成 linked content；
2. 去掉 link markup；
3. 检查去掉 link 后是否与原文完全相同；
4. 检查所有 link target 是否属于允许集合；
5. 不满足就拒绝结果。

这是非常好的生产思路：**对 AI 任务验证任务不变量，而不只是验证 JSON Schema。**

但 Jargon 最终仍会更新原字段。目标方案进一步改成 Annotation Projection：

```text
base text immutable
 + annotation(span, target)
 -> render-time overlay
```

这样即使内部链接算法改版，基础文本也不需要 mutation。

---

## 15. Parentable + SamenessCheck：组合召回合理，破坏性 absorb 不适合事实库

### 15.1 候选与确认

Article 设置：

```text
parent_matching threshold: 0.3
```

流程：

```text
pgvector cosine nearest neighbor
 -> distance threshold
 -> Article::SamenessCheck
```

`SamenessCheck` 又组合：

1. 标题标准化；
2. Levenshtein 相似度；
3. 默认标题阈值约 0.7；Embedding 极近时放宽到约 0.5；
4. 再让 LLM 判断是否为“同一个 underlying work”，而不是只同主题。

这比“向量低于阈值就认定相同”稳健很多，适合用作 relation candidate pipeline。

### 15.2 风险在于后续结构修改

`Parentable#find_similar_and_absorb!` 会：

```text
become_child_of existing parent
or
create synthesized parent with two records
```

Article 还声明 `reparents`：

- Search.source；
- Insight.article；
- SearchArticle.article。

当 Article 被 absorb 后，下游引用会被迁移到新的 parent。若相似判断误判，影响不仅是一个“聚类标签”，而是结构性改写。

长期事实库必须改成：

```text
candidate relation
 -> evidence
 -> optional LLM classification
 -> semantic_relation row
 -> UI collapse projection
```

关系类型至少：

```text
SAME_WORK
REPUBLISHED
TRANSLATION_OF
RELATED
CONTRASTS
SUPPORTS
```

Relation 可撤销、可重新计算，Document/Version/Source 不被破坏。

---

## 16. Insight 聚类：同样不应把近义观点物理合并

Insight 的 `parent_matching threshold: 0.25`，并且排除同一 Article 的其他 Insight，避免同文内观点互相合并。

这是一个合理的小技巧，但“不同来源出现相近观点”不一定等于同一个观点：

- 结论可能相同但论据不同；
- 一个支持、一个反驳；
- 版本条件不同；
- 同词但语境不同。

目标系统最好建立：

```text
INSIGHT_RELATED
INSIGHT_SUPPORTS
INSIGHT_CONTRASTS
INSIGHT_DUPLICATE_CANDIDATE
```

UI 可以聚类展示，底层 Knowledge Note 保留各自来源与 Evidence Span。

---

## 17. 自动研究：最有想象力，也最需要预算边界

`SearchGeneratable` 默认最多生成 2 个研究问题，Prompt 鼓励：

- first-principles；
- nuanced understanding；
- 跨领域连接；
- 不只向细节钻取。

Article、Insight、Search 都具备继续生成 Search 的能力。

`SearchJob`：

1. 将研究问题改写成 5～10 词搜索 query；
2. Exa Search；
3. 取前 10 个候选；
4. LLM 选 1～3 篇；
5. `Article.find_or_create_by!(url)`；
6. 新 Article 又自动进入 Ingest；
7. 搜索完成后继续 Summary/Follow-up。

这是典型递归研究图。

问题：没有看到平台级：

```text
max_depth
max_fanout
max_urls
max_domains
max_new_documents
max_tokens
max_search_cost
max_wall_time
```

长期运行很容易因为一个 Insight 逐层扩展成大量搜索和新文章。

目标系统必须把 ResearchRun 显式建模，并保存 budget 与 stop_reason。外部搜索发现的 URL 默认只进入 `ExternalEvidence`，不能自动变成 Managed Source，也不能污染博客 Coverage；只有人工/规则 `Promote` 才进入 Source Probe。

---

## 18. SearchJob 的一个实现矛盾：PDF 能摄取，候选选择却主动排除 PDF

Jargon 的 Ingest 支持 abstract → full text/PDF，并有 `pdftotext`；但 `SearchJob` 的候选选择 Prompt 写着“Exclude PDFs and non-article pages”。

这不是严重 bug，更像产品策略选择，但说明“支持某资源类型”和“Research policy 是否允许该资源”必须分层。

目标平台应显式配置：

```text
resource_capability: 可以解析什么
research_policy: 本次研究允许什么
managed_source_policy: 什么能算正式 Source
```

例如技术博客 Coverage 只接受博客文章，但 Research 可以允许 PDF paper 和公开视频字幕作为外部证据。

---

## 19. SummarizeSearchJob：递归派生漂移最明显的地方

`aggregate_content` 构造回答上下文时主要使用：

```text
Article.title + Article.summary
Article.rolled_up_insights(limit 5)
Related Article.summary
Related Insight.body
```

也就是说 Research Summary 并没有强制回到 Article 原文 block。随后模型又生成：

```text
summary
snippet
followup_queries
```

Prompt 还把 snippet 称为“key quote from one of the articles or insights”。但传给模型的 Article 可能只有 summary，Insight 本身也可能是派生文本，因此 quote 的证据语义非常模糊。

这正是“摘要的摘要”漂移问题：

```text
source text
 -> article summary
 -> insight
 -> search synthesis
 -> followup search context
 -> more synthesis
```

每一步都可能缩减限定条件、因果关系和不确定性。

目标 RAG/Research 规则：

1. Summary/Insight 只用于召回、排序、query planning；
2. 要形成最终 Claim 时回到 Source Block；
3. `EvidenceRole` 显式标记 EVIDENCE/CONTEXT/QUERY_EXPANSION/RANKING_SIGNAL；
4. lineage depth 超阈值的派生物不能作为唯一证据；
5. 所有 quote 必须经过 Evidence Binding。

---

## 20. Jargon 对 Web 管理的启发

Jargon 本身是一个可交互的 Web 研究库，Article、Insight、Search 都是用户可浏览对象。这说明对于目标系统，仅有爬虫后台列表不够；Web 管理也应包含知识对象层。

建议目标 Web UI 至少分为：

```text
Source / Coverage
Run / Task / Failure
Document / Version / Snapshot
Canonical Markdown / Diff
Knowledge Note / Evidence
Semantic Relation
Research Run / Candidate / Budget
```

特别是：

- Insight 点击后显示其来源版本和 evidence span；
- Relation 显示“为什么判定 SAME_WORK/RELATED”；
- External candidate 显示从哪个 Research Question 发现；
- Promote 到 Managed Source 是显式管理动作。

---

## 21. 对目标技术方案的具体优化项

综合源码后，目标方案应确保以下设计成为正式能力，而不是后续补丁。

### 21.1 多模态 Source Block Locator

Canonical IR Block 不只保存 text，还保存 locator：

```text
HTML
  dom_path/css_hint + source/normalized offsets

PDF
  page + bbox + text offsets

Transcript
  segment_id + start_ms/end_ms + raw_text_hash
```

这使 Blog、Paper、Video 外部证据可以统一进入 Grounding。

### 21.2 Insight Card 一等公民

Insight 与 Chunk 不同：Chunk 是检索切片，Insight 是“模型理解出的观点”。

```text
Chunk = deterministic/rebuildable text window
Insight = probabilistic derived knowledge with evidence
```

二者必须有不同 schema、Release 和展示方式。

### 21.3 Source Fact / AI Derivation / User Note 三分

不能用一个通用 Note 表混淆来源。至少通过 origin/type/permission/audit 强区分。

### 21.4 Grounding 强制化

每个 Insight 的 evidence hint 都要经过程序绑定。任何无法绑定的内容只能 PARTIAL/UNGROUNDED。

### 21.5 非破坏性关系

Embedding、标题距离、LLM 都只能生成 relation evidence；永不自动删 Document、改 Source identity 或迁移事实引用。

### 21.6 Research 有硬预算

Research Worker 独立 queue、独立成本预算，不能抢占 Managed Source Backfill/Incremental 的资源。

### 21.7 Annotation 代替文本 mutation

内部链接、高亮、相关 Insight 标签都以 Annotation overlay 渲染；需要增强文本时必须执行 stripped-text invariant。

### 21.8 派生任务与事实摄取解耦

`Version READY` 是事实层终点。Embedding/Insight/Research 失败只影响对应 projection，不影响正文同步状态。

### 21.9 Hybrid Retrieval

Article Summary embedding 可以用于高层主题检索，但原始 Block/Chunk 必须建立 BM25 + Vector，保证 API 名、版本号、命令、错误字符串可精确命中。

### 21.10 原始媒体先持久化

PDF、HTML、字幕都先存 Snapshot/Raw artifact，再解析。任何 extractor/model 升级都从已有事实离线重放。

---

## 22. 推荐的生产级派生执行链

```text
DocumentVersion READY
 -> emit VERSION_READY

DERIVE_SUMMARY
 -> structured output
 -> schema validate
 -> evidence bind
 -> persist KnowledgeNote

DERIVE_INSIGHTS
 -> insight candidates
 -> evidence hints
 -> bind to Source Blocks
 -> grounded/partial/quarantined

EMBED
 -> Source Blocks/Chunks
 -> Knowledge Notes independently

RELATE
 -> vector candidate
 -> lexical/title/content evidence
 -> optional LLM classification
 -> semantic_relation

RESEARCH
 -> bounded question generation
 -> external search
 -> candidate ranking
 -> ExternalEvidence ingest
 -> grounding
 -> optional explicit Promote
```

关键点：每一步都是可重跑的 Projection，不修改 Snapshot/Canonical IR。

---

## 23. 推荐的证据对象示例

```json
{
  "knowledge_note_id": "kn_123",
  "note_type": "INSIGHT",
  "claim": "......",
  "source_version_id": "ver_456",
  "evidence": [
    {
      "block_id": "blk_17",
      "start_offset": 83,
      "end_offset": 176,
      "evidence_kind": "VERBATIM",
      "anchor_status": "BOUND",
      "source_text_hash": "sha256:..."
    }
  ],
  "derivation_release_id": "derive_2026_08",
  "model_release_id": "model_x",
  "prompt_release_id": "prompt_y"
}
```

Transcript 则额外带：

```json
{
  "segment_ids": ["s120", "s121"],
  "start_ms": 1397000,
  "end_ms": 1412000,
  "evidence_kind": "NORMALIZED_TRANSCRIPT"
}
```

这样前端可以直接跳回文章段落、PDF 页面或视频时间点。

---

## 24. 最终判断

Jargon 不适合作为 1000 站博客历史抓取系统的基础框架：它没有 Coverage 证明、Durable Frontier、Source Profile、Snapshot/Version 真相层、生产级限流和可恢复调度；其 Crawl4AI 只是 CLI fallback，整体更接近个人研究应用。

但它对目标知识库的“知识层”有非常高的参考价值：

- Article 之上建立 Insight Card；
- Embedding 用来发现潜在连接；
- 自动生成 research question；
- Web Search 找到新外部证据；
- 结构化 LLM 输出 + bounded repair；
- 对“只加链接”任务执行文本不变量校验。

目标方案应吸收这些能力，但必须做四个根本性改造：

```text
Jargon destructive parent/absorb
 -> non-destructive semantic relation

Jargon free-form snippet/quote
 -> evidence span + grounding + typed quote

Jargon recursive summary/insight research
 -> source-block grounding + lineage depth + evidence role

Jargon open-ended research recursion
 -> bounded ResearchRun + ExternalEvidence + explicit Promote
```

再加上 HTML/PDF/Transcript 的媒体定位器后，最终系统可以同时满足两种需求：底层是可证明全量与增量同步的博客事实库，上层则是可审计、可回源、可主动扩展的知识研究系统。