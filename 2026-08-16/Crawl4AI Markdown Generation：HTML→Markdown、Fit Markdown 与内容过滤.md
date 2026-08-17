# Crawl4AI Markdown Generation：HTML→Markdown、Fit Markdown 与内容过滤

## 1. 调研对象与目标

- 官方 Markdown Generation 文档：https://docs.crawl4ai.com/core/markdown-generation/
- 官方 Fit Markdown 文档：https://docs.crawl4ai.com/core/fit-markdown/
- Markdown 生成源码：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/markdown_generation_strategy.py
- 内容过滤源码：https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/content_filter_strategy.py

调研目标不是判断 Crawl4AI “能不能生成 Markdown”，而是判断它在 **1000+ 技术博客全历史回填、持续增量同步、长期 Markdown 知识库** 中应承担什么职责，以及哪些实现语义会影响完整性、可重放性、多语言内容和技术结构保真。

---

## 2. 结论

Crawl4AI Markdown Generation 很适合作为高吞吐、可插拔的 **Markdown Candidate Generator**，但 `raw_markdown`、`fit_markdown`、`fit_html` 都不应该直接成为知识库业务真相。

推荐生产链路：

```text
Immutable Raw HTML
  -> Versioned Cleaner
  -> 多路 Candidate
       - Site Recipe DOM
       - Trafilatura
       - Crawl4AI raw_markdown
       - Crawl4AI Pruning fit_markdown
       - BM25 fit（诊断 / query projection）
       - LLM repair（慢路径）
  -> Structure-aware Quality Gate
  -> Canonical IR
  -> Deterministic Markdown Renderer
  -> Final Markdown / Search / RAG
```

核心理由：

1. `raw_markdown` 更接近 HTML→Markdown 结构转换，`fit_markdown` 是经过过滤后的有损视图，二者目标不同；
2. Pruning 的 DOM 子树评分依赖阈值、标签权重、文本密度、链接密度等启发式，版本或参数变化会改变结果；
3. 当前 Pruning 实现存在几个对生产非常关键的边界：`preserve_*` 不能绝对保护被祖先节点包裹的后代、`min_word_threshold` 使用空格计词不适合 CJK、负 class/id 信号在复合分数中被截断到 0；
4. BM25 是 query-conditioned relevance filter，并且当前 tokenization 主要依赖空格切词，对中文等语言并不适合作为正文完整性过滤；
5. BM25 候选会对完全相同的 chunk 文本去重，可能删除文章中有意重复的代码或提示块；
6. `fit_html` 是过滤器返回块再次用 `<div>` 包装后拼接得到的派生 HTML，不应假设它仍是原 DOM 中连续、原样的子树；
7. Markdown 生成异常路径可能把错误描述写进非空结果字段，因此“字段非空”不能等价于成功；
8. 对百万级历史文章，过滤器或 Renderer 升级应从 Raw Artifact 重放，而不是重新下载网页。

因此 Crawl4AI 的价值是：**高效生成候选、提供过滤视图与诊断证据**；平台负责 Raw 真相、质量判定、Canonical IR、版本语义和最终 Markdown。

---

## 3. DefaultMarkdownGenerator 的实际执行链路

### 3.1 输入内容层必须显式记录

`DefaultMarkdownGenerator` 默认 `content_source="cleaned_html"`，同时支持 raw/cleaned/fit 等内容源语义。

这意味着平台必须把输入层作为 Candidate 身份的一部分：

```text
input_artifact_id
input_content_source
crawler_release_id
cleaner_release_id
markdown_generator_release_id
content_filter_release_id
```

例如：

```text
raw_html + generator=A
cleaned_html + generator=A
```

即使最终文本相同，也不是同一次可复现的生成过程。上游 Cleaner 已经改变 DOM，不能把差异错误归因给 Markdown Renderer。

### 3.2 HTML→Markdown 的参数语义

当前实现使用 Crawl4AI 封装的 `CustomHTML2Text`。默认参数倾向于保留结构并减少展示型折行，例如：

```text
body_width = 0
ignore_emphasis = false
ignore_links = false
ignore_images = false
protect_links = false
single_line_break = true
mark_code = true
escape_snob = false
```

这对技术知识库是合理方向：代码、标题、列表、链接和图片语义不应因为排版宽度被二次改写。

但生产上有一个容易被忽略的实现细节：**参数来源不是简单叠加**。当前调用逻辑按优先级选择：

```text
per-call html2text_options
  else per-call options
  else generator self.options
  else defaults
```

一旦提供更高优先级的参数源，低优先级自定义项不会继续自动合并。

因此平台不能只记录“用户传了哪些 override”，而应由 Config Compiler 先算出 **最终 effective options**，再记录：

```text
effective_options_json
effective_options_hash
option_source
markdown_candidate_release_id
```

这样升级或重放时才能精确得到同一结果，也能避免“以为 constructor option 还生效、实际被 per-call option 覆盖”的隐性漂移。

### 3.3 相对链接与 citation

生成器会把相对 URL 基于 `base_url` 补全，并对相同 URL 复用引用编号，再生成 references。

这适合作为 provenance 辅助，但引用编号不能作为稳定业务身份。正文前部插入一个链接就可能让后续编号变化。

Canonical IR 应保存结构化链接：

```text
LinkRef
- href_absolute
- anchor_text
- title
- relation
- section_path
- ordinal_in_section
```

最终引用编号由平台 Renderer 按确定性规则生成。

### 3.4 MarkdownGenerationResult 应视为“多视图证据”

核心视图包括：

```text
raw_markdown
markdown_with_citations
references_markdown
fit_markdown
fit_html
```

推荐的语义是：

- `raw_markdown`：高召回 Markdown 候选；
- `fit_markdown`：过滤后的降噪候选；
- `fit_html`：解释过滤器保留了哪些块的派生证据；
- citation/references：链接恢复和 provenance 辅助。

对象存储保存大文本，PostgreSQL 只保存 Candidate Manifest、URI、哈希、质量结果和 Release。

### 3.5 `fit_html` 不是原 DOM 快照

当前生成流程会先让 Content Filter 返回若干 HTML block，然后把这些 block 逐个重新包装进 `<div>` 再拼接，并基于该结果继续生成 `fit_markdown`。

因此：

```text
fit_html != 原页面中的一段连续 DOM
```

它可能失去原始祖先上下文、容器属性和部分层级语义。生产上应明确记录：

```text
fit_html_is_synthetic = true
ordered_filter_block_hashes[]
source_dom_artifact_id
filter_release_id
```

若需要精确解释“这个块来自原 DOM 哪里”，平台应另外保存 DOM path / node fingerprint / block ordinal，而不能从 `fit_html` 反推。

### 3.6 异常字符串不能当成功结果

源码在 HTML→Markdown、citation 等异常路径中存在把错误描述写入结果字段的行为。因此：

```text
raw_markdown != ""  ≠  generation success
```

平台 Candidate Contract 必须分离：

```text
status
error_code
error_message
artifact_uri
artifact_hash
```

并在质量门禁中拒绝已知错误 sentinel、异常 traceback、极短错误文本等结果。

---

## 4. PruningContentFilter 的技术原理

### 4.1 本质是 DOM 子树启发式评分

当前实现先解析 HTML，移除 comment，并排除一组明显非正文标签，例如：

```text
nav footer header aside script style form iframe noscript
```

随后从 body 递归处理节点。核心统计量可概括为：

```text
text_len
html/tag_len
link_text_len
semantic_tag_weight
class/id signal
```

默认复合权重大致为：

```text
text_density   0.4
link_density   0.2
tag_weight     0.2
class/id       0.1
text_length    0.1
```

可以抽象成：

```text
score = weighted_mean(
  text_density,
  inverse_link_density,
  semantic_tag_weight,
  class_id_signal,
  normalized_text_length
)
```

它之所以对一般网页有效，是因为正文通常有较高文本密度、较低链接密度，并集中在 article/main/section/p 一类节点；导航和推荐区更容易表现为链接密集或短文本容器。

### 4.2 fixed 与 dynamic threshold

固定阈值模式直接按统一 threshold 剪枝。

动态阈值会根据节点语义、文本比例和链接比例调整要求。当前实现中可以观察到类似：

- 更重要的语义标签降低阈值；
- 高文本密度节点降低阈值；
- 高链接占比节点提高阈值。

这比 1000 个站点全部写 CSS selector 更有泛化性，但仍然只是启发式，不是“正文完整性证明”。

### 4.3 `preserve_tags/classes` 的重要陷阱：不是绝对后代保护

`preserve_tags`、`preserve_classes` 很有价值，但当前递归实现的关键顺序是：

```text
处理当前 node
 -> 如果当前 node 自己命中 preserve，则保留并停止剪枝该节点
 -> 否则先给当前 node 评分
 -> 当前 node 低于阈值则整体 decompose
 -> 只有当前 node 被保留，才继续递归 children
```

因此，下面这种结构存在风险：

```html
<div class="low-density-wrapper">
  <pre class="highlight">...</pre>
</div>
```

即使 `pre` 或 `highlight` 被列入 preserve，只要祖先 `div` 先被判低分并整体删除，递归根本不会走到 `pre`，后代仍会消失。

所以不能把 native preserve 解释成“只要标签在白名单里就绝不会丢”。对于技术博客，平台需要增加更强的 **Protected Structure Policy**：

1. 在运行 Pruning 前标记 code/pre/table/math/mermaid/figure 等受保护节点；
2. 同时保护其必要 ancestor chain，或先独立抽取这些技术 block；
3. Pruning 后按 node fingerprint 校验受保护结构召回率；
4. 召回不达标时禁止 fit candidate 晋升；
5. 对高价值站点可使用平台自定义 filter/fork，而不是依赖 native preserve 的局部语义。

### 4.4 `min_word_threshold` 对中文等 CJK 有明显风险

当前 Pruning 的“word count”近似通过空格数量计算：

```text
word_count ≈ text.count(" ") + 1
```

这对英文 prose 勉强可用，但对中文、日文等没有空格分词的正文非常危险。一整段中文可能被计成极少“词”；如果配置了正的 `min_word_threshold`，节点可能直接得到淘汰分数。

代码块、命令行、短 API 表格也可能受类似影响。

生产建议：

- **默认不要在全局 Profile 设置正的 `min_word_threshold`**；
- Source Language 未知或 CJK 时设为 `None`/禁用该门槛；
- 如确需长度门禁，由平台使用 Unicode-aware token/character/grapheme 规则，而不是空格计词；
- 将 `language_profile_release` 和 tokenizer policy 纳入 Release；
- 回归集必须包含中文、日文、混合中英和代码密集页面。

### 4.5 class/id 负向信号不能被高估

当前实现会根据 class/id 中的 `nav/footer/sidebar/ads/comment/promo/social/share...` 等负向模式计算负分，但进入复合分数时会对 class/id score 做非负截断。

结果是：**负 class/id 更像“得不到这一维的正向贡献”，而不是强烈主动扣分**。在当前实现里，不能假设一个 class 名叫 `sidebar` 就一定会因为 class/id 维度被显著惩罚。

因此生产降噪应更依赖：

- 明确 `exclude_selectors`；
- 已知模板 fingerprint；
- link density / boilerplate ratio；
- Source Recipe；
- Candidate 间交叉验证；
- pinned version + contract test。

不要把源码中的指标名字等同于预期行为。

### 4.6 技术博客典型误删对象

高风险内容包括：

- `pre/code` 代码示例；
- API 参数表、benchmark 表格；
- Mermaid、Math/LaTeX；
- 纯图片架构图的 figure/figcaption；
- 脚注、引用、admonition；
- 短列表 changelog/release note；
- 系列文章“上一篇/下一篇”关系；
- 多语言短段落；
- 复制按钮包裹的代码容器；
- 文档站 tab/panel 中的隐藏技术内容。

因此 fit 候选必须和 raw/Recipe/Trafilatura 做结构覆盖比较。

---

## 5. BM25ContentFilter 的技术原理与边界

### 5.1 它优化的是“相关性”，不是“完整性”

BM25ContentFilter 将页面拆成有顺序的文本 chunk，生成或接受 query，然后使用 `rank_bm25.BM25Okapi` 做相关性打分。

常见 BM25 形式：

```text
score(D,Q) = Σ IDF(q) * TF(q,D)*(k1+1)
             / (TF(q,D) + k1*(1-b+b*|D|/avgdl))
```

当前实现还会根据标签给分数乘权，例如 h1/h2/title/strong/blockquote/code/pre 等有不同权重；通过阈值筛选后，再按原文位置恢复输出顺序。

这非常适合 query-time context reduction，但目标函数和“全量保存文章正文”根本不同。

### 5.2 无显式 query 时的页面自相关 query

没有外部 query 时，会尝试使用：

- title；
- h1；
- meta keywords/description；
- 必要时首个较长段落。

这能得到“和页面主题相关”的内容，但标题和摘要并不覆盖所有技术细节。文章后半段的回滚方案、监控 SQL、边界条件、benchmark script 可能词面相关度较低，却是知识库最重要的内容之一。

### 5.3 CJK tokenization 不适合承担 canonical 过滤

当前 BM25 路径主要依赖 lower + whitespace split，并可配 Snowball stemming。这种 tokenization 对中文、日文等语言不能形成可靠词项。

因此：

- Crawl4AI BM25 fit 不应进入全量正文 canonical 主链；
- 中文 query-time 检索应使用平台自己的语言感知 tokenizer/search analyzer；
- 若把 BM25 fit 用作诊断，质量报告必须记录 language 与 tokenizer compatibility；
- CJK 页面不应因为 BM25 低分被判“无正文”。

### 5.4 完全相同 chunk 的去重可能丢语义

当前 BM25 chunk 流程会对完全相同的 chunk 文本去重，通常保留第一次出现的位置。

对一般网页这能去掉重复模板，但技术文章可能有意重复：

- 相同代码片段在“错误示例”和“修复后上下文”重复出现；
- 同一 warning/admonition 在不同章节反复强调；
- 相同命令分别出现在 Linux/macOS 步骤；
- 教程末尾再次给出完整命令作为总结。

因此 exact chunk dedup 也属于有损行为。BM25 fit 适合诊断/查询投影，不应作为唯一正文版本。

### 5.5 代码边界不能视为稳定结构边界

BM25 的文本 chunk 目标是相关性打分，不是 Canonical IR 构造。代码等元素可能在文本 chunk 过程中和周边内容组合，不能假设 BM25 的 chunk 就等价于最终 section/code block。

Canonical IR 的结构应主要来自 DOM/Recipe/HTML→Markdown 高召回候选与交叉证据。

---

## 6. LLMContentFilter 的生产位置

LLM Filter 可以处理复杂页面，但不适合作为 1000+ 站点持续全量同步主链：

- 成本随页面数增长；
- model/provider/prompt 变化造成结果漂移；
- 超时、限流、上下文截断难与“页面确实无正文”区分；
- 可重现性弱；
- 内容被模型总结/重写后，不能再视为原文证据。

推荐只放在：

```text
REPAIR / REVIEW slow lane
```

并保存：

```text
model_release
prompt_release
input_artifact_hash
output_hash
token_usage
finish_reason
error_code
```

LLM 输出默认不得覆盖 Raw、Canonical IR 的原文结构证据。

---

## 7. 对知识库方案的具体优化

### 7.1 Candidate 类型

```text
site_recipe_dom
trafilatura
jsonld_article_body
crawl4ai_raw_markdown
crawl4ai_fit_pruning
crawl4ai_fit_bm25       # 诊断 / query projection，默认不参与 canonical winner
llm_repair              # slow lane
```

### 7.2 MarkdownCandidateManifest

建议至少：

```text
candidate_id
source_id
url_identity_id
raw_artifact_id
cleaned_artifact_id
runtime_release_id
generator_release_id
content_filter_release_id
input_content_source
effective_options_json
effective_options_hash
raw_markdown_artifact_uri
fit_markdown_artifact_uri
fit_html_artifact_uri
references_artifact_uri
fit_html_is_synthetic
filter_block_manifest_uri
structure_fingerprint
text_fingerprint
language
quality_status
generated_at
```

`filter_block_manifest` 建议保存 block ordinal、hash、可选 DOM path/fingerprint，用于解释 fit 结果来源。

### 7.3 Protected Structure Policy

平台在第三方 Filter 之外增加受保护结构机制：

```text
protected tags/classes/selectors
 -> 标记 node fingerprint
 -> 保护必要 ancestor chain / 独立提取 technical blocks
 -> 执行 pruning
 -> 对比 protected structure recall
 -> 低于阈值则 fit candidate FAIL/WARN
```

对于技术博客，至少关注：

```text
pre code table blockquote figure figcaption
mermaid math katex mathjax
admonition callout tabs
```

站点 override 由 Source Recipe 审批，不让普通 Web 用户透传任意 selector/regex/code。

### 7.4 多语言安全策略

Source Profile 增加平台级意图：

```yaml
language_policy:
  mode: auto
  cjk_safe: true

markdown_candidate:
  pruning:
    min_word_threshold: null
  platform_guardrails:
    protected_structure_policy: ancestor_chain
    unicode_length_gate: true
```

这不是要求 Crawl4AI 原生支持所有字段，而是平台 Config Compiler 的意图层。Compiler 根据 pinned Runtime 能力生成实际安全配置。

### 7.5 结构质量门禁

不能只看字数，至少计算：

```text
heading_recall
paragraph_recall
code_block_recall
table_recall
list_recall
link_recall
image_caption_recall
protected_structure_recall
source_block_order_recall
cjk_text_retention
repeated_block_retention
main_text_ratio
noise_ratio
duplicate_block_ratio
metadata_completeness
```

示例：

- raw 有 12 个 code fence，fit 只有 3 个 -> `STRUCTURE_LOSS_CODE`；
- Recipe/Trafilatura 都识别出 5 个表格，fit 为 0 -> `STRUCTURE_LOSS_TABLE`；
- protected `<pre>` 在低密度祖先下消失 -> `PROTECTED_DESCENDANT_LOSS`；
- 中文正文 5000 字，启用 word threshold 后只剩标题 -> `CJK_WORD_GATE_LOSS`；
- fit 比 raw 少 70% 文本但技术结构和主段落召回高 -> 可判有效降噪；
- 所有确定性 Candidate 都过短 -> Browser Repair/人工 Review。

### 7.6 Canonical IR 与最终 Markdown

Canonical IR 保存结构而不是第三方 Markdown 字符串：

```text
DocumentIR
- metadata
- title
- authors[]
- published_at
- updated_at
- canonical_url
- language
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
- provenance[]
```

最终 Renderer 决定 front matter、引用编号、空行、GFM table、代码 fence 等格式。

---

## 8. 不重抓的 Reprocess 机制

Raw Artifact 不变时，以下 Release 变化只应创建 Reprocess Task：

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
Immutable Raw Artifact
 -> old/new Candidate 重放
 -> Regression Diff
 -> Quality Gate
 -> Canonical IR Diff
 -> 只有 semantic IR 变化才创建新的语义版本
```

对百万文章非常关键：调阈值、修 tokenizer、保护 code/table、升级 Crawl4AI 都不应触发百万次重新下载。

---

## 9. 增量同步的四层哈希

不要只比较最终 Markdown：

```text
raw_content_hash       # 网络取得的规范化原始内容
extraction_hash        # 指定 cleaner/extractor/filter release 的候选
semantic_ir_hash       # Canonical IR 语义结构
projection_hash        # Renderer 输出
```

判定：

1. Raw 不变：除非 Release 变化，否则通常无需重抽取；
2. Raw 变、Semantic IR 不变：记录抓取变化，不制造文章语义版本；
3. Semantic IR 变：创建 Document Version；
4. 只有 Projection 变：属于格式投影升级，不冒充原文更新。

---

## 10. Release 回归与 Contract Test

### 10.1 分层回归语料

```text
普通英文博客
中文技术博客
中英混合页
日文页
代码密集页
表格密集页
Mermaid/Math 页
长文
短文
Newsletter
文档站
SPA
异常 DOM
重复代码/重复提示块
低密度祖先包裹 code/table 的页面
```

### 10.2 每次升级比较

- 正文召回；
- heading/code/table/list/link/image 结构召回；
- protected structure recall；
- CJK retention；
- 重复 block retention；
- 噪声率；
- Markdown 稳定性；
- Candidate winner 变化率；
- CPU/内存/时延。

### 10.3 Crawl4AI Markdown Contract Test 必测项

1. `content_source` 实际输入语义；
2. constructor options 与 per-call `options/html2text_options` 的优先级；
3. raw/citation 失败时是否产生非空错误字符串；
4. `fit_html` 是否仍由过滤 block 二次包装构成；
5. `preserve_tags/classes` 后代位于可剪祖先时的行为；
6. CJK + 正 `min_word_threshold` 行为；
7. negative class/id 模式对最终 composite score 的真实影响；
8. BM25 whitespace tokenization 在中文页的行为；
9. BM25 对完全相同 chunk 的去重行为；
10. BM25 输出是否恢复原始文档顺序；
11. code/table/figure 等结构在 old/new Release 的保留差异；
12. timeout/partial/error 字段与 Candidate status 的映射。

生产语义以 **pinned release + contract test** 为准，不根据参数名、文档描述或历史经验推断。

---

## 11. Web 管理功能补充

### Source 级

- 当前 Candidate/Filter/Language Policy Release；
- effective Markdown options；
- Pruning 参数；
- protected structure 配置；
- CJK-safe 状态；
- 最近 7/30 天 Candidate 通过率；
- structure-loss 分类统计；
- current/previous release 回归与灰度对比。

### URL/Version 级

并排查看：

```text
Raw HTML
Cleaned HTML
Crawl4AI raw_markdown
Crawl4AI fit_markdown
synthetic fit_html
filter block manifest
Trafilatura
Site Recipe
Canonical IR
Final Markdown
```

显示：

```text
release / effective options hash
text/structure fingerprint
heading/code/table/link 数
protected structure recall
language/CJK retention
quality reason
DOM/block provenance
```

这样运营人员能回答“为什么这一段代码或中文正文消失了”，而不是只看到一个最终 Markdown。

---

## 12. 风险与处置

### 风险 1：把 fit 当 canonical

后果：过滤器升级导致历史知识库漂移，被剪内容无法从 fit 恢复。

处置：Raw 不可变；fit 仅 Candidate；Canonical IR 独立质量门禁。

### 风险 2：把 preserve 当绝对保护

后果：受保护 code/table 仍可能随祖先一起被剪掉。

处置：平台 Protected Structure Policy + ancestor chain/独立技术块抽取 + recall gate。

### 风险 3：全局使用 `min_word_threshold`

后果：中文/日文/代码页面被大量误删。

处置：默认禁用；语言感知、Unicode-aware 门禁；CJK 回归语料。

### 风险 4：依赖 class/id 负分假设

后果：以为 sidebar/ads 会被强扣分，实际 composite 行为不一定如此。

处置：explicit exclude recipe + quality gate + pinned source contract tests。

### 风险 5：BM25 作为正文完整性过滤

后果：低词面相关章节、CJK 内容或有意重复技术块被删除。

处置：BM25 仅 query projection/diagnostic，canonical 主链使用高召回确定性候选。

### 风险 6：异常字符串被当 Markdown

处置：Candidate status/error/artifact 分离，禁止“字段非空即成功”。

### 风险 7：第三方升级无感知

处置：固定 Crawl4AI version/image digest；Release、Contract Test、Regression Corpus、Canary、Rollback。

---

## 13. 推荐生产算法

```python
async def process_fetch(fetch_result):
    raw = promote_immutable_raw(fetch_result)
    cleaned = clean_with_release(raw)

    protected = fingerprint_protected_structures(cleaned)

    candidates = await run_bounded_portfolio([
        site_recipe_candidate(cleaned),
        trafilatura_candidate(cleaned),
        crawl4ai_raw_markdown_candidate(cleaned),
        crawl4ai_pruning_candidate(
            cleaned,
            cjk_safe=True,
            protected_structures=protected,
        ),
    ])

    reports = [
        quality_check(
            c,
            raw=raw,
            protected_structures=protected,
        )
        for c in candidates
    ]

    winner = select_candidate_by_policy(candidates, reports)
    if winner is None:
        enqueue_repair_or_review(raw.id)
        return

    ir = build_canonical_ir(
        winner,
        corroborating_candidates=candidates,
        provenance=reports,
    )

    canonical = validate_ir(ir)
    if not canonical.pass_gate:
        enqueue_repair_or_review(raw.id)
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

关键原则：

- Crawl4AI fit 只是候选之一；
- Raw 永远先保存；
- 技术结构与多语言正文有独立保护；
- 第三方实现语义通过 Contract Test 锁定；
- 参数变更优先 Reprocess，不重新访问原站；
- 语义版本与 Markdown 格式版本分离。

---

## 14. 最终建议

对 1000+ 技术博客，Crawl4AI Markdown Generation 应定位为：

> **可插拔、可重放、可比较、受质量门禁约束的 Markdown 候选生成器。**

最推荐组合：

```text
Discovery：Sitemap/RSS/CMS/Archive/Common Crawl/Wayback/Seeder
Fetch：HTTP First + Browser Slow Lane
Truth：PostgreSQL + Immutable Object Storage
Extraction：Recipe + Trafilatura + Crawl4AI raw/pruning Portfolio
Protection：Ancestor-aware technical structure + CJK-safe policy
Quality：结构召回 + 多候选交叉验证 + typed reasons
Canonical：Document IR
Projection：Deterministic Markdown + Search/RAG
Incremental：Raw/Extraction/Semantic/Projection 四层哈希
Operations：Release + Replay + Contract Test + Regression + Canary + Web Diagnosis
```

这样既利用 Crawl4AI 的网页清洗和 Markdown 转换效率，又避免把第三方启发式过滤器、空格分词假设、局部 preserve 语义或错误字符串承担为“历史正文真相”，更适合长期扩展到数千站点和百万/千万级文档。