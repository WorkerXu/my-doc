# Crawl4AI Markdown Generation：HTML→Markdown、Fit Markdown 与内容过滤

## 1. 调研对象

- 官方文档：https://docs.crawl4ai.com/core/markdown-generation/
- Fit Markdown 文档：https://docs.crawl4ai.com/core/fit-markdown/
- Markdown 生成源码：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/markdown_generation_strategy.py
- 内容过滤源码：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/content_filter_strategy.py
- 调研目标：判断 Crawl4AI 的 Markdown 生成与内容过滤能力，应该如何进入“1000+ 技术博客全历史抓取、增量同步、Markdown 知识库”的生产链路，以及哪些能力只能作为候选/辅助信号、不能直接成为业务真相。

---

## 2. 结论先行

Crawl4AI 的 Markdown Generation 很适合作为网页抓取后的一个**高吞吐 Markdown Candidate Generator**，但不适合把 `result.markdown.fit_markdown` 直接当成知识库最终 Markdown。

最重要的原因有五点：

1. `raw_markdown` 与 `fit_markdown` 的语义不同：前者更接近 HTML→Markdown 的结构转换，后者经过启发式/BM25/LLM 等过滤，是有损结果；
2. `PruningContentFilter` 的阈值、标签权重、文本密度和链接密度会改变保留内容，同一 Raw HTML 在参数或版本变化后可能得到不同结果；
3. `BM25ContentFilter` 天然是 query-conditioned 的，它适合“围绕某个查询挑内容”，不适合作为“保存文章完整正文”的唯一真相；
4. 技术博客中的代码块、表格、引用、脚注、目录、系列导航，有时在统计特征上很像“低密度噪声”，只保留 fit 结果会产生不可逆信息损失；
5. Crawl4AI 源码在异常路径会将部分错误信息写入 MarkdownGenerationResult 字段，因此平台必须独立判断 success/quality，不能只检查字段非空。

因此生产架构应采用：

```text
Immutable Raw HTML
  -> 多路 Extraction / Markdown Candidate
     -> Crawl4AI raw_markdown
     -> Crawl4AI fit_markdown + fit_html
     -> Trafilatura Candidate
     -> Site Recipe Candidate
  -> Candidate Quality + Structure Diff
  -> Canonical IR
  -> Deterministic Markdown Renderer
  -> Final Markdown / Search / RAG
```

Crawl4AI 的价值是“提供另一条高质量候选与诊断证据”，而不是取代 Raw Artifact、Canonical IR、版本化 Renderer 和质量门禁。

---

## 3. DefaultMarkdownGenerator 的实现链路

### 3.1 输入源

`MarkdownGenerationStrategy` 默认 `content_source="cleaned_html"`，源码同时把 `raw_html`、`cleaned_html`、`fit_html` 作为可区分的内容源概念。

对知识库平台，这意味着必须把“输入 HTML 是哪一层”写入 Candidate 元数据。即使 Markdown 内容相同，下面两种结果也不能视为同一生成过程：

```text
raw_html + generator_release=A
cleaned_html + generator_release=A
```

因为上游 HTML 清洗已经改变了可见 DOM，后续 Markdown 差异未必来自 Renderer。

建议 Candidate 至少保存：

```text
input_artifact_id
input_content_source
crawler_release_id
cleaner_release_id
markdown_generator_release_id
content_filter_release_id
html2text_options_hash
output_hash
```

这样后续才能回答“为什么这次 Markdown 和上次不同”。

### 3.2 HTML→Markdown

当前 `DefaultMarkdownGenerator` 使用 Crawl4AI 自己封装的 `CustomHTML2Text` 做转换。默认配置强调保留 Markdown 结构，而不是为了展示而强制折行，例如关闭固定宽度换行，并保留强调、链接、图片及代码标记。

这对技术博客是正确方向：

- 代码块不应因为行宽被二次折行；
- 标题层级、列表、链接、图片语义需要保留；
- 最终用于 Git、RAG 和 diff 的 Markdown 应尽量稳定，避免纯排版变化制造大量版本噪声。

平台层仍然不应直接依赖 Crawl4AI 默认参数，而应建立 `markdown_candidate_release`，显式记录所有转换参数。升级 Crawl4AI 版本时必须用回归语料验证，而不是原地替换镜像。

### 3.3 链接引用转换

源码会扫描生成后的 Markdown 链接，把相对 URL 基于 `base_url` 补全，并对相同 URL 复用同一个引用编号，再生成 references 区域。

这个机制有两个生产价值：

1. 能把正文中的相对链接转换成可追踪的绝对引用；
2. 同一 URL 的去重引用便于压缩正文和保留来源关系。

但最终知识库不应该把引用编号本身作为稳定身份。文章插入一个新链接后，后续编号可能整体移动。Canonical IR 应保存结构化 Link：

```text
LinkRef {
  href_absolute
  anchor_text
  title
  relation
  section_path
  ordinal_in_section
}
```

最终 Markdown Renderer 再按确定性规则重新编号。这样内容版本比较不会被第三方 Renderer 的编号策略绑死。

### 3.4 MarkdownGenerationResult 是“多视图结果”

生成器返回的核心视图包括：

```text
raw_markdown
markdown_with_citations
references_markdown
fit_markdown
fit_html
```

对知识库平台，最佳使用方式不是五选一，而是把它们视为**同一次候选生成的相关证据**：

- `raw_markdown`：高召回的 Markdown 候选；
- `fit_markdown`：降噪候选；
- `fit_html`：解释 fit 结果为什么被保留的结构证据；
- citation/references：链接恢复与 provenance 的辅助证据。

对象存储中可以用一个 Candidate Manifest 关联它们，数据库只保存索引、哈希、质量分数与 Artifact URI，避免把大文本塞进 PostgreSQL。

---

## 4. PruningContentFilter 的技术原理

### 4.1 它不是“正文抽取器真相”，而是 DOM 子树评分器

当前源码先用 BeautifulSoup/lxml 解析 HTML，删除 comment 以及 `nav/footer/header/aside/script/style/form/iframe/noscript` 等显式排除标签，然后从 body 开始递归剪枝。

每个节点计算的核心统计量包括：

```text
text_len      = 节点可见文本长度
tag_len       = 节点内部 HTML 长度
link_text_len = 直接子链接的文本长度
```

随后计算复合分数。当前源码启用的维度包括：

- text density；
- link density；
- tag weight；
- class/id weight；
- text length。

默认权重大致为：

```text
text_density  0.4
link_density  0.2
tag_weight    0.2
class/id      0.1
text_length   0.1
```

可抽象为：

```text
score = weighted_mean(
  text_len / tag_len,
  1 - link_text_len / text_len,
  semantic_tag_weight,
  class_id_signal,
  log(text_len + 1)
)
```

这解释了为什么它对通用网页降噪有效：正文往往文本密度高、链接密度低，并集中在 article/main/section/p 等节点；导航、推荐区和社交分享区域通常链接密度高、class/id 带有负向模式。

### 4.2 固定阈值与动态阈值

`threshold_type="fixed"` 时，低于阈值的节点直接移除。

动态阈值则会根据标签重要性、文本密度和链接比例调整阈值。例如重要语义标签或高文本密度节点会更容易保留，高链接比例节点会被更严格地审查。

这比一个全站固定 CSS selector 更具泛化性，适合 1000 个异构博客；但它仍然是启发式算法，不是“文章正文完整性证明”。

### 4.3 preserve_tags / preserve_classes 很重要

当前实现支持把指定标签或 class 放进 preserve 白名单，命中的节点跳过剪枝。

这对于技术博客尤其关键，可以按 Source Profile/Recipe 保存：

```yaml
markdown_candidate:
  pruning:
    threshold_type: dynamic
    threshold: 0.48
    preserve_tags: [pre, code, table, blockquote]
    preserve_classes:
      - highlight
      - codehilite
      - language-python
      - mermaid
      - math
```

但不能允许普通用户把任意 preserve 配置直接透传 Runtime。应该由平台的 Recipe Compiler 将低风险站点配置编译成受控配置，并设置数量/长度/正则复杂度上限。

### 4.4 典型误删场景

对于技术博客，以下内容可能被误判：

- API 参数表：HTML 标签很多但自然语言文本少；
- 纯代码示例：单词统计与普通 prose 不同；
- 公式/LaTeX：文本密度异常；
- “上一篇/下一篇”系列链接：对知识库关系有价值，但链接密度高；
- 引用与脚注：可能较短；
- 图片型架构图的 figure/figcaption：正文文字较少；
- changelog/release note：大量短列表项；
- 多语言页面：某些词法/分词假设不稳定。

因此 fit 候选必须和 raw/其他抽取器做结构覆盖对比。

---

## 5. BM25ContentFilter 的技术原理

### 5.1 候选块生成

当前实现会把 body 按 DOM 块级结构拆成有顺序的 text chunks，并保留原文位置。若用户没有显式 query，会尝试从 title、h1、meta keywords/description 以及首个较长段落组合出页面 query。

这让 BM25 能在“没有外部问题”的情况下做一个页面自相关过滤，但依然存在偏差：title/description 只描述页面主题，不保证覆盖正文所有技术细节。

### 5.2 BM25 排序

源码使用 `rank_bm25.BM25Okapi`。语义上，BM25 对 chunk 中 query term 的匹配做 TF 饱和，并用文档频率和长度归一化修正，常见形式可表示为：

```text
score(D,Q) = Σ IDF(q) * TF(q,D)*(k1+1)
             / (TF(q,D) + k1*(1-b+b*|D|/avgdl))
```

当前实现还可以先做 Snowball stemming，再清理 tokens；得到 BM25 分数后，会按标签类型加权，例如标题、strong、blockquote、code 等可获得更高权重，最后按 `bm25_threshold` 过滤，并恢复到原始文档顺序输出。

### 5.3 为什么不应作为全量知识库正文过滤器

BM25 的目标是“与 query 相关”，而用户需求是“全量历史文章知识库”。二者目标函数不同。

例如一篇标题为“PostgreSQL 高并发调优”的文章后半段可能包含：

- Linux IO 参数；
- connection pool 配置；
- benchmark 脚本；
- 故障回滚；
- 监控 SQL。

如果 query 主要来自标题，部分低词面相关但非常重要的章节可能被删掉。

所以 BM25ContentFilter 更适合：

1. 搜索结果预览；
2. query-time context reduction；
3. 抓取诊断时快速定位主题块；
4. Candidate 质量比较的一条辅助视图；
5. 后续 RAG 的 query-conditioned projection。

不适合：

- 唯一 Canonical Markdown；
- 决定文章是否“完整抓取”；
- 覆盖率计算；
- 版本真相判定。

---

## 6. LLMContentFilter 的位置

LLM 过滤可以在复杂页面上得到更语义化的输出，但对于 1000+ 站点的持续全量同步，主链不应依赖它：

- 成本随页面数线性增长；
- provider/model/prompt 变化会改变结果；
- 超时、限流、上下文截断会带来不稳定；
- 很难把“没有输出”区分为“页面无正文”还是“模型异常”；
- 可复现性弱于确定性 DOM/规则/统计方法。

合适的使用方式是 REPAIR/REVIEW 慢路径：只有确定性候选都低质量时才触发，并保存 model release、prompt release、token usage、原始输入哈希和输出哈希。

---

## 7. 对现有知识库技术方案的具体优化

### 7.1 新增 Markdown Candidate 层

原方案已有 Raw Artifact -> Extraction Portfolio -> Quality -> Canonical IR -> Renderer，方向正确。本次调研建议把 Crawl4AI Markdown 能力明确建模为 Extraction Portfolio 中的一类 Candidate，而不是 Runtime 的顺手输出。

新增候选类型：

```text
crawl4ai_raw_markdown
crawl4ai_fit_pruning
crawl4ai_fit_bm25       # 默认不进入 canonical 竞争，只作诊断/query projection
site_recipe_dom
trafilatura
jsonld_article_body
llm_repair              # slow lane
```

### 7.2 新增 MarkdownCandidateManifest

```text
MarkdownCandidateManifest
- candidate_id
- source_id
- url_identity_id
- raw_artifact_id
- runtime_release_id
- generator_release_id
- content_filter_release_id
- input_content_source
- options_hash
- raw_markdown_artifact_uri
- fit_markdown_artifact_uri
- fit_html_artifact_uri
- references_artifact_uri
- structure_fingerprint
- text_fingerprint
- generated_at
- quality_status
```

候选文本放对象存储，DB 保存索引和摘要。

### 7.3 增加结构覆盖质量门禁

不能只看字数。至少比较：

```text
heading_recall
paragraph_recall
code_block_recall
table_recall
list_recall
link_recall
image_caption_recall
noise_ratio
main_text_ratio
duplicate_block_ratio
```

对于同一个 Raw Artifact，可用多个候选互相交叉验证：

- raw Markdown 有 12 个 code fence，fit 只有 3 个 -> fit 不得直接晋升；
- DOM recipe 和 Trafilatura 都有 5 个表格，fit 为 0 -> 触发 `STRUCTURE_LOSS_TABLE`；
- fit 比 raw 少 70% 文本但标题/代码/表格覆盖都高 -> 可能是有效降噪；
- 所有确定性候选正文过短 -> 进入 Browser Repair 或人工 Review。

### 7.4 Final Markdown 必须由 Canonical IR 决定性渲染

Canonical IR 应保留结构语义，而不是只存 Markdown 字符串：

```text
DocumentIR
- metadata
- title
- authors[]
- published_at
- updated_at
- canonical_url
- sections[]
  - heading
  - level
  - blocks[]
    - paragraph
    - code(language, text)
    - list
    - table
    - quote
    - image(src, alt, caption)
    - math
    - link
- references[]
```

Renderer 只负责把 IR 投影为稳定 Markdown。这样未来更换 Markdown 语法、引用编号、front matter 格式时，不必重新抓网页。

### 7.5 增加“无需重抓”的再处理能力

Raw Artifact 不变时，升级以下任何组件都应该只产生 Reprocess Task：

```text
cleaner_release
markdown_candidate_release
content_filter_release
extractor_release
quality_policy_release
ir_schema_release
markdown_renderer_release
```

流程：

```text
Raw Artifact
 -> Reprocess old/new Candidate in parallel
 -> Regression Diff
 -> Quality Gate
 -> New Canonical Version（只有语义变化才晋升）
```

这对 1000 个站点非常重要：调参不应触发百万级重新下载。

### 7.6 增加 Candidate Release 回归机制

每次升级 Crawl4AI 或过滤参数前，从历史 Raw Artifact 建立分层回归集：

```text
普通博客
代码密集页
表格密集页
长文
短文
多语言
SPA
Newsletter
文档站
异常 DOM
```

比较 old/new：

- 内容召回；
- 结构召回；
- 噪声率；
- Markdown 稳定性；
- CPU/内存/时延；
- Browser 使用率。

只有达到阈值才发布新的 Release，并支持按 Source Group 灰度。

---

## 8. 推荐的生产执行算法

```python
async def process_fetch(fetch_result):
    raw = promote_immutable_raw(fetch_result)

    candidates = await run_bounded_portfolio([
        site_recipe_candidate(raw),
        trafilatura_candidate(raw),
        crawl4ai_raw_markdown_candidate(raw),
        crawl4ai_pruning_candidate(raw),
    ])

    reports = [quality_check(c, raw) for c in candidates]

    winner = select_candidate_by_policy(candidates, reports)
    if winner is None:
        enqueue_repair(raw.id)
        return

    ir = build_canonical_ir(winner, corroborating_candidates=candidates)
    canonical = validate_ir(ir)
    if not canonical.pass_gate:
        enqueue_review_or_repair(raw.id)
        return

    version = upsert_document_version(
        semantic_hash=canonical.semantic_hash,
        ir_artifact=canonical.ir,
        provenance=canonical.provenance,
    )

    if version.is_new_semantic_version:
        render_markdown(version)
        enqueue_search_projection(version)
```

关键点：

- Crawl4AI 的 fit 结果只是 candidates 中一个；
- Candidate 失败不能污染 Raw；
- 任何过滤器升级都能从 Raw 重放；
- 同一文档的“抓取变化”和“抽取算法变化”分开记录；
- semantic hash 不因引用编号、空行、Renderer 版本小改动而变化。

---

## 9. 增量同步中的 Markdown 判定

增量同步不应简单比较最终 Markdown 字符串，否则 Renderer/过滤器升级会制造“文章更新”。

建议分三层哈希：

```text
raw_content_hash       # 网络取得的规范化原始内容
extraction_hash        # 指定 extractor/filter release 的候选结果
semantic_ir_hash       # Canonical IR 的语义结构
projection_hash        # 某 Renderer 的 Markdown 输出
```

判定：

1. `raw_content_hash` 不变：通常无需重新抽取，除非有新 Release 触发 Reprocess；
2. Raw 变、semantic IR 不变：记录抓取变化但不创建知识文档语义版本；
3. semantic IR 变：创建 Document Version；
4. 只有 projection_hash 变：属于 Renderer 投影变化，不应冒充原文更新。

---

## 10. Web 管理功能补充

建议在现有管理台增加 Markdown/Extraction 诊断页：

### Source 级

- 当前 Candidate Policy / Release；
- Pruning 参数及 preserve 规则；
- 最近 7/30 天候选通过率；
- structure-loss 按类型统计；
- 低质量页面 Top N；
- old/new release 灰度对比。

### URL/Version 级

并排查看：

```text
Raw HTML
Cleaned HTML
Crawl4AI raw_markdown
Crawl4AI fit_markdown
fit_html
Trafilatura
Site Recipe
Canonical IR
Final Markdown
```

同时显示每层的 hash、release、字数、heading/code/table/link 数量、质量原因。

这样运营人员能回答“正文为什么少了一段”，而不是只能看到最终 Markdown。

---

## 11. 风险与边界

### 风险 1：把 fit 当 canonical

后果：算法阈值变动会造成历史知识库大范围漂移，且被剪掉的信息无法从 fit 自身恢复。

对策：Raw 不可变；fit 只作 candidate；Canonical 经过独立质量门禁。

### 风险 2：全站统一阈值

后果：代码站、文档站、Newsletter 的 DOM 统计特征不同，同一个 pruning threshold 不可能在所有站点最优。

对策：默认 Profile + Source Group Profile + 少量站点 override，全部 Release 化和回归测试。

### 风险 3：BM25 误当正文完整性过滤

后果：低词面相关章节被删掉。

对策：BM25 放 query projection/diagnostic，不作为全量正文唯一候选。

### 风险 4：异常字符串被当正常 Markdown

生成器源码存在捕获异常后把错误描述放进结果字段的路径。

对策：Candidate contract 明确 `status/error_code`；错误字段与 artifact 内容分离；质量门禁禁止包含已知 error sentinel 的候选晋升。

### 风险 5：第三方升级无感知

对策：固定 Crawl4AI 版本/镜像 digest；Release + contract test + regression corpus + canary。

---

## 12. 对 1000+ 博客场景的最终建议

Crawl4AI Markdown Generation 应定位为“**可插拔、可重放、可比较的 Markdown 候选生成器**”。

最推荐的组合是：

```text
Discovery：Sitemap/RSS/CMS/Archive/Common Crawl/Wayback/Seeder
Fetch：HTTP First + Browser Slow Lane
Truth：PostgreSQL + Immutable Object Storage
Extraction：Site Recipe + Trafilatura + Crawl4AI raw/pruning Portfolio
Quality：结构覆盖 + 噪声 + 多候选交叉验证
Canonical：Document IR
Projection：Deterministic Markdown + Search/RAG
Incremental：Raw/Extraction/Semantic/Projection 四层哈希
Operations：Release、Replay、Regression、Canary、Web Diagnosis
```

这既利用了 Crawl4AI 在网页清洗和 Markdown 转换方面的工程效率，也避免让第三方启发式过滤器承担“历史正文真相”的职责，符合长期扩展到数千站点、百万/千万文档时所需的可恢复性、可解释性和可演进性。
