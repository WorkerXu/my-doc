# catalog-forge：把异构商品目录实体解析改造成“Reference First、可拒识、可审计、可增量”的腕表匹配基线

- 分析人：b
- 调研项目：catalog-forge
- 项目地址：https://github.com/alex-hahn/catalog-forge
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么这次选择 catalog-forge

本次执行前先检查了 `奢侈品调研/b` 已有结果，已经分析过 Ameli、AnyMatch、Confidence Classifiers、DeepBlocker、Ditto、GraLMatch、TransClean、pyJedAI、parts-distributor-sku-classifier 等项目/论文，当前目录中没有 `catalog-forge.md`，因此本次选取它继续分析。

当前 Spec 的核心约束非常明确：

1. 三个来源：雷小安、腕表之家、奢当家；
2. 数据规模约 100 万～1000 万，并且持续增量；
3. “同一个商品”被严格定义为 **同一 reference number / 型号**；
4. reference 字段高度稀疏，有时是结构化字段，有时埋在标题，图片也可用；
5. **绝对不能误匹配，precision 优先到极致，允许漏匹配/拒识**；
6. 可以人工标注几百对黄金样本。

catalog-forge 值得分析的原因不是它在公开数据上的 F1 本身，而是它已经把一个真实商品实体解析系统需要的关键工程部件拆得很完整：

```text
ingest
  -> normalize
  -> attribute / identifier extraction
  -> blocking
  -> rule matching
  -> calibrated classifier
  -> uncertainty judge
  -> clustering + cohesion check
  -> golden record
  -> review queue
  -> persistent incremental index
```

而且它把每个阶段的输入输出、provenance、decision reason、冲突、人工 review、增量状态都保留下来，可重跑、可审计、可撤销。

对于当前需求，最重要的结论是：

> **不要原样采用 catalog-forge 的“通用商品模糊匹配”作为最终身份判定，而应该保留它的工程骨架，把最终身份权收紧为 `brand + verified canonical reference` 的确定性规则。**

换句话说，catalog-forge 可以作为“系统骨架”，但腕表同款判定应改造成更保守的 **Reference-First Entity Resolution**。

---

## 2. catalog-forge 的整体架构

项目入口 `src/catalog_forge/pipeline.py` 把整个流程显式拆成阶段，每一阶段都写出独立 Parquet artifact：

```text
listings_raw.parquet      原始 listing
provenance.parquet        字段级来源/血缘
listings.parquet          规范化后的 listing
extractions.parquet       属性/标识符抽取结果
candidates.parquet        Blocking 候选对及候选来源
 decisions.parquet        匹配决策、概率、stage、reason
clusters.parquet          listing -> cluster
merges.parquet            聚类合并日志
hierarchy.parquet         listing -> variant -> product
golden.parquet            每个 cluster 的 golden record
conflicts.parquet         cluster 内字段冲突
review_queue.parquet      人工复核队列
run.json                  参数摘要、耗时、数量、config digest
```

这比“一个模型输入两条记录直接输出 match/no-match”的价值大得多，因为当前需求最大的风险并不是模型平均效果差，而是：

- 某一次 reference 错抽；
- 某个平台 SKU 被当成 manufacturer reference；
- 同系列相邻 reference 被相似度模型误合并；
- 一条错误边通过传递闭包污染整个实体簇；
- 新批次模型或规则变化后无法解释为什么历史结果改变。

把各阶段 materialize 后，可以回答：

```text
这条 listing 原始 reference 是什么？
由哪个 extractor 从哪里抽出的？
规范化前后分别是什么？
为什么进入这组候选？
是哪条规则/哪个模型决定了它？
最终为什么进入这个 entity？
有哪些字段发生冲突？
如果人工否决一条边，哪些实体需要重算？
```

对于“可漏不可错”，这种 **证据链和可回放性** 本身就是安全机制。

---

## 3. Identifier 层：catalog-forge 最值得直接复用的思想

代码：

- `src/catalog_forge/identifiers.py`
- `src/catalog_forge/blocking/exact.py`
- `src/catalog_forge/match/rules.py`

### 3.1 标识符不是普通字符串，而是“有类型、有校验、有语义范围”的值

catalog-forge 对 GTIN 不只是做 string equality，而是：

1. 判断是否满足合法长度；
2. 验证 checksum；
3. 统一到比较 key；
4. 只有合法 GTIN 才能进入强规则。

它对 MPN 也不是直接相信原字段，而是做 normalize，再配合 brand 才认为是强身份信号。

代码中的核心理念可以概括为：

```text
raw string
  -> classify identifier kind
  -> validate
  -> normalize
  -> canonical comparison key
  -> rule decision
```

这正好适合腕表 reference。

### 3.2 缺失和冲突必须是两个不同状态

`identifiers_agree()` 的设计很重要：

```text
1.0  -> 两边都有有效标识且一致
0.0  -> 两边都有有效标识且冲突
None -> 至少一边没有足够证据，无法判断
```

当前系统也应该坚持这个三值逻辑：

```text
UNKNOWN != CONFLICT
```

如果一边没有 reference，不能当作“不一致”；但如果两边都已经得到可信 reference 且不同，就应该是 **硬否决**，任何文本、图片、embedding、LLM 分数都不能覆盖。

### 3.3 MPN 只在品牌范围内有意义

catalog-forge 的 `rule_same_mpn_and_brand` 要求：

```text
normalized MPN 相同
AND brand 相同
```

才判定 MATCH。

这个规则对腕表尤其重要。即使业务口径写的是“同一 reference”，生产上仍建议把身份 key 设为：

```text
identity_key = (canonical_brand_id, canonical_reference)
```

因为不同品牌可能存在相同或相近型号字符串；brand scope 是非常便宜的误匹配保险。

---

## 4. catalog-forge 的 Identifier 方案不能原样照搬到腕表

它的通用 `normalize_mpn()` 会做类似：

```text
去掉非字母数字字符
转大写
长度限制
要求同时含字母和数字
```

这对于一般电商 MPN 是合理启发式，但对腕表不能直接作为最终 canonicalization。

原因是腕表 reference 的“格式字符”有时可能承载型号家族/后缀语义；不同品牌的 reference 规则也差异很大。全局无脑删掉 `/`、`.`、`-`、空格等字符，可能把本应区分的值折叠到一起。

因此建议把 reference 规范化拆成两层：

```text
Layer A: safe normalization
  Unicode NFKC
  trim
  uppercase
  全角 -> 半角
  标准化常见 dash/space 字符

Layer B: brand-aware canonicalization
  Rolex rules
  Omega rules
  Cartier rules
  Patek rules
  ...
```

并且永久保留：

```text
reference_raw
reference_normalized_safe
reference_canonical
canonicalization_rule_id
canonicalization_version
```

如果某个 brand 没有经过验证的规则，宁可停在 safe normalization，不要擅自把两个字符串合并。

---

## 5. 必须新增“编号角色分类”，否则 exact match 也会错

catalog-forge 已经意识到 feed 会把 SKU、电话、内部编码等错误塞进 identifier 字段；当前三源二奢数据更应该显式建一个 identifier role 层。

建议至少定义：

```text
MANUFACTURER_REFERENCE   品牌/制造商 reference，允许参与跨源身份判定
PLATFORM_LISTING_ID      平台商品/listing ID
MERCHANT_SKU             商家/店铺内部 SKU
SERIAL_NUMBER            单只腕表序列号
COMPATIBLE_REFERENCE     配件标题中“适配 XX reference”
MODEL_FAMILY             系列/家族名，不是完整 reference
UNKNOWN                  无法确定角色
```

只有：

```text
role == MANUFACTURER_REFERENCE
```

的值才能升级为 `verified_reference`。

尤其要防止这种典型误匹配：

```text
标题：Rolex 116610LN 适配表带 / 表盒 / 配件

116610LN 出现在标题里
但当前商品并不是 116610LN 腕表本体
```

因此抽出字符串只是第一步，还必须判断：

> **这个 reference 是“当前售卖主体”的型号，还是被描述/适配/比较的其他商品型号？**

这一步比继续提高 fuzzy similarity 更重要。

---

## 6. Blocking：精确标识负责第一路，模糊特征只能负责“找候选”

代码：`src/catalog_forge/blocking/exact.py`

catalog-forge 把 Blocking 与最终 Decision 分开，这是正确设计。

`ExactIdentifierBlocking` 会按 GTIN/MPN/ASIN 建 block；`CompositeKeyBlocking` 还会使用：

```text
brand + model token
brand + quantity
brand + color + size
title prefix
```

README 还特别说明：

> exact identifier 是很强的 candidate signal，但不是无条件 decision，因为 feed 可能复用、污染或错填 identifier。

对当前腕表系统可以进一步简化成两条完全不同的路径。

### 6.1 Verified Reference 路径

已经得到可信 reference：

```text
(brand_id, canonical_reference)
       |
       v
exact hash / B-tree lookup
       |
       v
existing canonical entity
```

这是生产主路径，不需要 ANN、embedding 或 pairwise classifier。

### 6.2 Unresolved 路径

reference 缺失或不确定时才进入候选恢复：

```text
brand block
  + model-shaped token
  + char n-gram / BM25
  + OCR token
  + optional embedding/image retrieval
       |
       v
Top-K candidate references
       |
       v
Reference Verifier
       |
       +-- 找到可验证 canonical reference -> 回主路径
       |
       +-- 无法验证 -> ABSTAIN / REVIEW
```

最关键的权限边界：

> **Blocking 的输出永远不是“同款”；它只是“值得进一步检查”。**

---

## 7. WDC Watches 的结果说明了什么，以及不能说明什么

catalog-forge README 直接报告了 WDC Watches：

```text
records:   3,700
precision: 0.833
recall:    0.880
F1:        0.856
AP:        0.922
threshold: 0.240
```

Blocking 指标：

```text
candidate pairs:    146,425
pair completeness:  0.763
reduction ratio:    0.979
```

同一 README 中，它在 WDC Watches 上相对一些基线明显更好：

```text
title Jaccard  P=0.379 R=0.717 F1=0.496
Splink         P=0.214 R=0.287 F1=0.245
Dedupe         P=0.333 R=0.343 F1=0.338
catalog-forge  P=0.833 R=0.880 F1=0.856
```

这能证明它的商品专用 feature、identifier validation、normalization 和 blocking 是有价值的工程基线。

但 **0.833 precision 对当前 Spec 完全不可接受**。

当前需求不是追求 benchmark F1，而是：

```text
自动合并区：precision 尽可能接近 1
剩余样本：允许大量 ABSTAIN
```

因此我们不应该调一个更高 classifier threshold 后就宣布完成，而应该改变决策权限：

```text
classifier / LLM / image
只能帮助恢复 reference 或排序 review

verified brand + canonical reference exact equality
才拥有自动合并权限
```

---

## 8. Ablation 反而更支持 Reference-First 方案

catalog-forge README 的 ablation 有一组非常有启发性的结果。在一个 identifier 覆盖较低的 harsh fixture（约 19% GTIN、15% MPN）中：

```text
full pipeline
  pair completeness 0.910
  precision 0.518
  recall 0.815
  F1 0.633

identifiers only
  pair completeness 0.061
  precision 0.924
  recall 0.060
  F1 0.114

no classifier
  precision 0.940
  recall 0.052
  F1 0.098
```

对于普通商品匹配论文，`identifiers only` 的 F1 很差；但对于当前需求，它反而揭示了一件关键事情：

> **最严格的 identifier 路径覆盖低，却天然处在更高 precision 区间。**

当前业务允许漏匹配，所以正确策略不是为了提高 recall 把 fuzzy matcher 的权限放大，而是：

1. 先把强标识路径做到极稳；
2. 再提高“reference 恢复率”；
3. 恢复后的 reference 仍回到 exact gate；
4. 恢复不了就不合并。

这与传统优化 F1 的方向不同，但与 Spec 完全一致。

---

## 9. Match Cascade：保留分层思想，但改变最终裁决权

代码：`src/catalog_forge/match/cascade.py`

catalog-forge 原始 cascade 是：

```text
1. Rules
   最便宜、最确定的 pair 先决策

2. Calibrated Classifier
   处理规则无法判断的 pair

3. Judge
   只处理 classifier 不确定区间
```

`MatchDecision` 会记录：

```text
left
right
probability
stage
reason
is_match
```

这个“逐层升级成本”的结构很适合百万/千万级数据，但当前项目必须改为 **不对称的三态决策**：

```text
AUTO_MATCH
AUTO_REJECT
ABSTAIN / REVIEW
```

建议改成：

```text
Stage 0: Identifier Role & Evidence Validation

Stage 1: Hard Rules
  verified ref 不同       -> AUTO_REJECT
  verified brand 不同     -> AUTO_REJECT
  同 brand + 同 verified canonical ref
                        -> AUTO_MATCH
  其他                  -> UNDECIDED

Stage 2: Reference Recovery
  title/OCR/catalog retrieval/NER/LLM
  目标不是直接输出 match，而是输出“候选 reference + evidence”

Stage 3: Verification
  候选是否能被原文 span / OCR box / reference catalog / brand rule 验证？
  能 -> 回 Stage 1
  不能 -> ABSTAIN / REVIEW
```

这样做后，模型永远不会产生这种危险结果：

```text
“标题、价格、图片都很像，所以即使 reference 不同也 match”
```

而是：

```text
“模型认为它很像，因此优先检查 reference；但 reference 冲突就直接 reject。”
```

---

## 10. Rule Ordering：把“冲突否决”放在“相似匹配”之前

`src/catalog_forge/match/rules.py` 有一个很值得直接借鉴的设计：

> Rules 按顺序触发，第一条 decisive rule 获胜；硬冲突规则放在正向 match 规则之前。

catalog-forge 默认先判断：

```text
different valid GTIN -> NON_MATCH
```

然后才判断：

```text
same valid GTIN -> MATCH
same brand + MPN -> MATCH
```

腕表系统应把这一原则进一步强化：

```text
R1 不同 verified manufacturer reference
   => HARD REJECT

R2 brand 冲突
   => HARD REJECT

R3 商品角色是配件/表带/盒证，但 reference 来自 compatible target
   => HARD REJECT / ABSTAIN

R4 同 brand + 同 verified canonical reference
   => AUTO MATCH

R5 只有 fuzzy/text/image 高相似
   => NEVER AUTO MATCH
```

也就是说：

> **negative veto 的权力高于任何模型正向分数。**

---

## 11. Clustering：catalog-forge 对“错误边扩散”的处理非常值得保留

代码：`src/catalog_forge/cluster/components.py`

项目明确指出普通 transitive closure 的风险：

```text
A -- B -- C -- D
         |
         X   <- 一条错误边
         |
E -- F -- G -- H
```

如果 `X` 错了，两个大 component 会整体合并；损害不是“一条边错”，而是整个笛卡尔关系被隐式声明为同实体。

catalog-forge 的处理是：

1. 先按阈值做 connected components；
2. 计算 component cohesion 和 density；
3. 对低 cohesion / 低 density 的 component 用 correlation clustering 重新拆分；
4. 每次 merge 都记录 stage/reason；
5. 人工 reject 某条边后可以 revoke，仅重算受影响部分。

这和此前分析过的 TransClean 方向一致：**pairwise 精度不等于实体簇安全**。

### 11.1 但当前 Spec 可以做得更简单、更强

因为业务已经定义：

```text
same product == same reference
```

所以对于真正 AUTO_MATCH 的记录，不必依赖概率图聚类，直接建立 canonical entity：

```text
entity_key = (brand_id, canonical_reference)
```

实体簇必须满足 invariant：

```text
validated_reference_set(cluster) 的大小 <= 1
```

若两个 cluster 合并后会出现：

```text
{126610LN, 126610LV}
```

则无论哪条 bridge edge 分数多高，都必须拒绝 merge。

因此建议：

```text
Verified 区：确定性 registry / exact grouping
Unresolved 区：候选图 + review，不 materialize 成主实体关系
```

这样能从根上消除“模糊边传递污染”。

---

## 12. Incremental：这是 catalog-forge 对当前 100万～1000万规模最直接的工程参考

代码：

- `src/catalog_forge/incremental/index.py`
- `src/catalog_forge/incremental/resolve.py`

catalog-forge 明确指出：

> 增量解析不是“把 batch pipeline 在一个更小的新批次上重跑”，因为新记录还要和旧 catalog 比；如果每来 1000 条都重建百万级 Blocking 索引，成本仍接近全量。

因此它持久化三类 postings：

```text
exact identifier key -> listing ids
MinHash band          -> listing ids
brand|model token     -> listing ids
```

新批次：

```text
append index
query existing postings
只生成涉及新 listing 的 candidate pairs
```

复杂度主要随 **新批次大小** 增长，而不是每次随全库重新增长。

它还把索引参数和版本一起保存，参数不兼容时拒绝静默混用，这也非常适合线上系统。

### 12.1 腕表场景可以进一步简化主索引

对 verified records，最核心的持久化索引只需要：

```text
(brand_id, canonical_reference) -> entity_id
```

例如 PostgreSQL：

```sql
CREATE UNIQUE INDEX uq_entity_reference
ON canonical_entity (brand_id, reference_canonical)
WHERE status = 'VERIFIED';
```

新批次流程：

```text
新 listing
  -> normalize brand
  -> extract/validate reference
  -> exact lookup identity key
  -> 已存在：attach listing to entity
  -> 不存在：create canonical entity
  -> reference 不确定：进入 unresolved，不创建“猜测 merge”
```

1000 万量级的 B-tree/hash exact lookup 并不需要复杂向量系统。

只有 unresolved tail 才需要：

```text
brand + token inverted index
char n-gram / BM25
OCR token index
optional ANN / image retrieval
```

---

## 13. 建议直接落地的 Reference-First 架构

### 13.1 总体数据流

```text
雷小安 / 腕表之家 / 奢当家
             |
             v
+---------------------------+
| 1. Raw Listing Store      |
| 原始字段/标题/图片/来源    |
+---------------------------+
             |
             v
+---------------------------+
| 2. Brand Normalizer       |
| alias -> canonical brand  |
+---------------------------+
             |
             v
+---------------------------+
| 3. Reference Evidence     |
| structured / title / OCR  |
| role classification       |
+---------------------------+
             |
             v
+---------------------------+
| 4. Brand-aware Canonical  |
| reference normalization   |
+---------------------------+
             |
             v
+---------------------------+
| 5. Deterministic Gate     |
| conflict -> reject        |
| exact verified -> accept  |
| otherwise -> abstain      |
+---------------------------+
       |                 |
       | verified        | unresolved
       v                 v
+----------------+   +----------------------+
| Entity Registry |   | Candidate Recovery   |
| exact key       |   | BM25/ngram/OCR/image |
+----------------+   +----------------------+
       |                 |
       |                 v
       |          +----------------------+
       |          | Human/Model Review   |
       |          | 只帮助恢复 reference |
       |          +----------------------+
       |                 |
       +<--- verified ---+
             |
             v
+---------------------------+
| 6. Audit / Monitoring     |
| provenance / conflicts    |
| rule versions / replay    |
+---------------------------+
```

核心原则可以压缩为一句话：

> **所有智能能力都可以帮助“找到 reference”，但只有确定性 reference gate 可以决定“是不是同一个商品”。**

---

## 14. 建议的数据模型

### 14.1 原始 listing

```sql
listing(
  listing_id,
  source_id,
  source_listing_id,
  title_raw,
  brand_raw,
  reference_raw_field,
  description_raw,
  image_urls,
  captured_at,
  payload_json,
  ingest_version
)
```

原始数据永不覆盖，用于审计和重新抽取。

### 14.2 Reference Evidence

```sql
reference_evidence(
  evidence_id,
  listing_id,
  reference_raw,
  reference_safe_normalized,
  reference_canonical,
  identifier_role,
  evidence_type,          -- structured/title/ocr/catalog/model
  source_field,
  text_span,
  ocr_bbox,
  confidence,
  verifier_status,        -- VERIFIED/REJECTED/UNRESOLVED
  extractor_version,
  canonicalizer_version,
  created_at
)
```

这张表比只在 listing 上放一个 `reference` 字段重要得多，因为同一 listing 可能同时出现：

```text
当前商品 reference
兼容商品 reference
系列号
平台 SKU
序列号
```

必须保留每个候选的来源与角色。

### 14.3 Canonical Entity

```sql
canonical_entity(
  entity_id,
  brand_id,
  reference_canonical,
  status,
  first_seen_at,
  last_seen_at,
  rule_version,
  created_reason
)
```

建议 `entity_id` 用稳定 UUID，而不是直接把当前 canonical string hash 成永久 ID。原因是未来 canonicalization 规则升级时，reference 展示形式可能变化，但业务实体 ID 不应大规模漂移。

### 14.4 Membership

```sql
entity_membership(
  entity_id,
  listing_id,
  link_status,            -- AUTO_VERIFIED / HUMAN_VERIFIED / REVOKED
  evidence_id,
  decision_reason,
  decision_version,
  created_at,
  revoked_at
)
```

### 14.5 Review / Audit

```sql
review_task(
  task_id,
  listing_id,
  candidate_entity_id,
  risk_reason,
  candidate_references,
  priority,
  status,
  reviewer_label
)
```

以及 append-only `decision_audit_log`，记录每次自动/人工状态变化。

---

## 15. Reference 提取：按证据强度做级联，而不是一次大模型调用

建议按以下优先级执行：

### Tier 1：结构化 reference 字段

如果平台字段语义明确，并经过历史数据验证：

```text
structured field
  -> role check
  -> brand-aware canonicalization
  -> reference catalog / format validation
```

这是成本最低、最可靠的来源。

### Tier 2：标题中有明确标签/模板

例如：

```text
型号：126610LN
Ref. 311.30.42.30.01.005
Reference 5711/1A-010
```

用规则/NER 抽取，同时保存原始 span。

### Tier 3：图片 OCR

重点图片不是“主图外观”，而是：

- 表背刻字；
- 保卡；
- 吊牌；
- 盒标；
- 品牌/型号标签。

OCR 输出必须保留：

```text
image_id
bbox
raw OCR text
candidate reference
OCR confidence
normalization trace
```

### Tier 4：受限候选检索 + 模型

只有前几层失败时，才从：

```text
same brand reference catalog
```

召回候选，然后用 title/OCR/属性去验证。

LLM/VLM 最好回答：

```text
“原始证据中支持哪个候选 reference？证据 span/bbox 在哪里？”
```

而不是自由生成：

```text
“你猜这是什么型号？”
```

如果模型给出的 reference 无法对齐到原文/OCR/catalog 证据，就不能升级成 VERIFIED。

---

## 16. 图片应该怎么用

当前 Spec 有图片，但图片不能成为直接同款主键。

### 可以做

```text
1. OCR reference
2. 图片候选召回
3. 同系列相邻 reference hard-negative mining
4. 图文冲突检测
5. 人工 review 排序
```

### 不应该做

```text
image cosine similarity > 0.95
=> AUTO MATCH
```

原因是奢侈腕表同系列相邻 reference 的外观可能极其接近，而业务定义又明确以 reference 为准。

所以图片只能回答：

> “我应该去重点检查哪些 reference？”

不能回答：

> “我可以绕过 reference 直接把两个 listing 合并。”

---

## 17. 最终决策状态机

建议不要只存 boolean `is_match`，而是显式状态：

```text
AUTO_MATCH
AUTO_REJECT
ABSTAIN
HUMAN_REVIEW
HUMAN_MATCH
HUMAN_REJECT
```

核心伪代码：

```python
def resolve_listing(x):
    brand = verify_brand(x)
    refs = collect_reference_evidence(x)

    verified = [r for r in refs if r.role == "MANUFACTURER_REFERENCE" and r.verified]

    # 同一 listing 内就出现多个互相冲突的可信 reference，先隔离
    keys = {(brand.id, r.canonical) for r in verified}
    if len(keys) > 1:
        return ABSTAIN("conflicting verified references inside listing")

    if not brand.verified or len(keys) == 0:
        return ABSTAIN("no verified identity key")

    key = next(iter(keys))
    entity = registry.exact_lookup(key)

    if entity is None:
        entity = registry.create_verified(key)
        return AUTO_MATCH(entity, "new exact reference entity")

    # cluster invariant 再检查一次
    if entity.has_conflicting_verified_reference(key):
        return AUTO_REJECT("entity reference invariant violation")

    return AUTO_MATCH(entity, "same verified brand + canonical reference")
```

注意：这里没有 title similarity threshold，也没有 image threshold。

它们应该出现在 `collect_reference_evidence()` 或 unresolved review 队列，而不是最终身份函数里。

---

## 18. 如何利用几百对黄金标签

几百对标签不应该平均随机抽，而应该围绕 **最可能产生 false positive 的边界**设计。

建议黄金集至少包含：

### A. 正例

```text
跨三个来源、写法不同，但 canonical reference 完全相同
```

用于验证 canonicalization 和 reference extraction。

### B. Hard Negative：相邻 reference

```text
126610LN vs 126610LV
同系列、同品牌、外观近、标题近，只差后缀/字符
```

这是最重要的 false-positive 测试集。

### C. 编号角色负例

```text
manufacturer reference vs 店铺 SKU
reference vs listing id
腕表 reference vs 配件适配 reference
reference vs serial number
```

### D. OCR 扰动

```text
O / 0
I / 1 / l
B / 8
S / 5
缺字符
多字符粘连
连字符/斜杠丢失
```

### E. 来源漂移

分别覆盖：

```text
雷小安
腕表之家
奢当家
```

以及各来源新增模板/字段变化。

---

## 19. 评估指标不要再以 F1 为主

当前业务应该把指标拆成两层。

### 自动放行区

主指标：

```text
Auto-Match Precision
False Positive Count
Hard-Negative False Positive Rate
```

理想发布门槛是黄金集/回放集上 **0 false positive**，并持续 shadow audit。

### 覆盖层

再观察：

```text
Auto-Match Coverage
Reference Extraction Coverage
Abstention Rate
Human Review Rate
```

这两个目标不能混成一个 F1。

当前约束允许 recall 下降，所以应接受：

```text
precision ↑↑↑
coverage ↓
```

而不是为了 F1 把边界候选自动放进来。

### 统计上的一个现实限制

只有几百个黄金样本时，即使一个 false positive 都没有，也不足以严格证明“99.99% precision”这类极端统计保证。

因此安全性要同时依靠：

1. deterministic invariant；
2. hard veto；
3. 0-FP 回放测试；
4. 人工抽检；
5. shadow mode；
6. 规则/模型版本化；
7. 线上冲突监控。

而不是只靠一个置信区间或模型概率。

---

## 20. 推荐的最小可落地技术栈

100 万～1000 万不是必须上一个复杂分布式 AI 平台的规模，尤其主路径是 exact key lookup。

一个务实版本：

```text
Batch / ETL:
  Python + Polars

Raw / Artifact:
  Parquet + 对象存储（或现有数据湖）

Canonical Registry:
  PostgreSQL
  unique B-tree on (brand_id, canonical_reference)

Unresolved Retrieval:
  PostgreSQL pg_trgm / OpenSearch / Elasticsearch
  只存 unresolved tail

Image:
  对象存储 + OCR 结果表

Model Service:
  可选；只处理 reference recovery / review

Audit:
  append-only DB table + Parquet replay artifacts
```

如果现有数据已经在 Spark/ClickHouse/数据湖里，也不必搬家；核心是保持逻辑边界：

```text
raw
-> evidence
-> verified reference
-> exact registry
-> unresolved review
```

---

## 21. 推荐的增量处理流程

每次抓取新批次：

```text
Step 1  source schema validation
Step 2  append raw listing
Step 3  normalize brand
Step 4  collect reference evidence
Step 5  identifier role classification
Step 6  brand-aware canonicalization
Step 7  deterministic gate
Step 8  exact entity lookup / attach
Step 9  unresolved -> candidate/review
Step 10 append audit + metrics
```

只重算：

```text
新 listing
受规则版本变化影响的 listing
人工 reject/纠正所触达的 entity
```

不需要每次全库 pairwise 重跑。

这与 catalog-forge 的 `PersistentIndex + resolve_incremental()` 思路一致，但 verified 主路径更简单、更安全。

---

## 22. Canonicalization 必须版本化

这是实际落地很容易忽视的坑。

假设 v1 规则：

```text
116610-LN -> 116610LN
```

后来发现某品牌的 `/` 或 `.` 不能全删，升级成 v2。

如果系统只保存最终字符串，就无法判断历史 entity 是由哪个规则产生的，也无法安全重算。

因此每个结果至少保存：

```text
canonicalization_version
extractor_version
role_classifier_version
reference_catalog_version
decision_policy_version
```

catalog-forge 在 `run.json` 中保存 config digest、在持久化 index 中检查参数版本，这种做法应直接吸收。

---

## 23. 人工纠错必须是“撤销证据/关系”，不是手工改最终表

catalog-forge 的 `revoke()` / incremental correction 思路很值得保留：

> reviewer 否决的是某条 evidence / merge，而不是直接删除整个 cluster。

当前系统可以这样做：

```text
人工发现 listing A 的 126610LN 是 OCR 错读

不要：
  手工把 entity_id 改掉且不留痕

应该：
  mark evidence E123 = REJECTED
  append audit event
  recompute A 的 verified identity
  只重算受影响 membership
```

这样结果可回放、可追踪，也不会因为一次人工修复破坏整个实体簇。

---

## 24. 需要建立的安全 Invariants

这些 invariant 比模型阈值更重要。

### Invariant 1

```text
一个 VERIFIED entity 只能有一个 canonical brand + reference key
```

### Invariant 2

```text
不同 verified references 不能被任何 similarity score 合并
```

### Invariant 3

```text
UNKNOWN reference 不能自动继承候选 entity 的 reference
```

否则会形成自我强化：

```text
先模糊匹配到 entity
-> 把 entity reference 填回 listing
-> 下次 exact match 看起来“证据一致”
```

这是必须禁止的 provenance 污染。

### Invariant 4

```text
模型生成的 reference 必须能回指原始证据或可信 catalog
```

### Invariant 5

```text
人工 reject 永远高于模型/规则自动 accept，除非人工显式撤销该 reject
```

### Invariant 6

```text
任何规则版本升级都可以对历史结果 replay
```

---

## 25. 建议的测试集和自动化测试

### 25.1 Canonicalization Property Tests

对每个品牌维护：

```text
等价写法 -> 必须同 key
非等价 reference -> 绝不能同 key
```

尤其做一字符 mutation：

```text
原 ref 每一位替换/删除/插入
```

验证不能被过度 normalization 折叠。

### 25.2 Role Classification Tests

固定测试：

```text
“适配 116610LN 表带”
```

不能把 `116610LN` 判为当前商品 manufacturer reference。

### 25.3 Cluster Invariant Tests

任何写入 membership 的事务前后都 assert：

```text
count(distinct verified identity key in entity) <= 1
```

### 25.4 Incremental Replay Tests

同一批数据：

```text
全量处理结果
```

和：

```text
分 100 个 batch 增量处理结果
```

在 verified identity 上必须一致。

### 25.5 Shadow Tests

新 extractor / canonicalizer / model 先只产出 shadow decision，不改变线上 membership；比较：

```text
新增 auto-match
新增 conflict
与旧版本差异
hard-negative FP
```

通过后再切换版本。

---

## 26. 可以直接从 catalog-forge 借什么

### 直接借设计

1. **阶段化 pipeline**：每阶段都有可检查 artifact；
2. **provenance**：字段/证据来源可追踪；
3. **typed identifier + validation**：不要把 ID 当普通字符串；
4. **Blocking 与 Decision 分离**；
5. **hard contradiction rule 优先于 positive rule**；
6. **decision reason**：每次自动决策都能解释；
7. **三态思路**：无法确定时允许停下；
8. **review queue**：只把高价值疑难样本送人工；
9. **persistent incremental index**：新批次只查旧索引；
10. **config/index versioning**；
11. **cluster cohesion / revoke**：防止错误关系传递污染；
12. **稳定 entity/cluster 生命周期**。

### 可以参考代码结构

```text
catalog_forge/identifiers.py
catalog_forge/blocking/exact.py
catalog_forge/match/rules.py
catalog_forge/match/cascade.py
catalog_forge/cluster/components.py
catalog_forge/incremental/index.py
catalog_forge/incremental/resolve.py
catalog_forge/pipeline.py
```

---

## 27. 不应该从 catalog-forge 原样照搬什么

### 27.1 不照搬默认 classifier threshold

公开 WDC Watches 的 precision 0.833 不满足当前要求。

### 27.2 不允许 fuzzy classifier 直接 auto-merge

即使 calibrated probability 很高，只要没有 verified reference，也应 abstain。

### 27.3 不照搬全局 MPN normalization

腕表需要 brand-aware、保守、版本化 canonicalization。

### 27.4 不让 image similarity 取得最终身份权限

图片只用于候选/OCR/冲突辅助。

### 27.5 不依赖普通 transitive closure

verified 区直接按 canonical identity key 归组；unresolved graph 不写入主实体。

### 27.6 不以 F1 作为主要上线目标

上线目标应该是：

```text
Auto-Match Precision / FP
然后才是 Coverage
```

---

## 28. 推荐实施路线

### Phase 0：数据画像

统计三源：

```text
品牌覆盖
结构化 reference 覆盖
标题可抽 reference 覆盖
reference 格式分布
冲突率
平台 SKU/内部 ID 模式
图片可 OCR 比例
```

先知道最安全的 30%～70% 数据能否直接走 exact path。

### Phase 1：只做确定性 MVP

实现：

```text
brand normalization
identifier role
brand-aware reference canonicalization
exact entity registry
audit log
```

先不上 LLM、embedding、复杂 classifier。

这个版本覆盖可能不高，但最符合 precision-first。

### Phase 2：提高 Reference Extraction Coverage

加入：

```text
title regex / NER
reference catalog retrieval
OCR
```

目标指标是：

```text
verified reference coverage ↑
```

不是 pairwise F1。

### Phase 3：Unresolved Candidate & Review

加入：

```text
brand + char n-gram/BM25
model token retrieval
optional image/embedding
review queue
active learning
```

模型只对人工队列排序，或帮助恢复 reference。

### Phase 4：生产增量与漂移监控

加入：

```text
persistent index
source/schema drift alerts
per-brand metrics
shadow replay
version migrations
human correction replay
```

---

## 29. 一个可以直接交给工程实现的 API 契约

### Reference Resolver

```json
{
  "listing_id": "lx_123",
  "brand": {
    "raw": "劳力士",
    "canonical_id": "rolex",
    "status": "VERIFIED"
  },
  "reference_candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "role": "MANUFACTURER_REFERENCE",
      "evidence_type": "TITLE",
      "evidence_location": "title[14:22]",
      "status": "VERIFIED"
    }
  ],
  "identity": {
    "brand_id": "rolex",
    "reference": "126610LN"
  },
  "decision": "AUTO_MATCH",
  "reason": "verified brand + exact canonical reference",
  "policy_version": "ref-first-v1"
}
```

不确定案例：

```json
{
  "listing_id": "sdj_456",
  "reference_candidates": [
    {
      "raw": "116610LN",
      "role": "COMPATIBLE_REFERENCE",
      "status": "REJECTED"
    }
  ],
  "identity": null,
  "decision": "ABSTAIN",
  "reason": "no verified manufacturer reference"
}
```

这种接口比只返回：

```json
{"match": true, "score": 0.97}
```

更符合当前业务，因为它保存了自动决策真正依赖的证据。

---

## 30. 最终方案建议

如果现在就开始实现，我建议把 catalog-forge 的思路收敛成下面这个版本：

```text
                 ┌──────────────────────────────┐
                 │ Raw immutable listings       │
                 └──────────────┬───────────────┘
                                │
                                v
                 ┌──────────────────────────────┐
                 │ Brand + Identifier Role      │
                 │ + Reference Evidence         │
                 └──────────────┬───────────────┘
                                │
                                v
                 ┌──────────────────────────────┐
                 │ Brand-aware Canonicalizer    │
                 └──────────────┬───────────────┘
                                │
                 ┌──────────────┴───────────────┐
                 │                              │
          verified identity               unresolved
                 │                              │
                 v                              v
       ┌──────────────────┐          ┌─────────────────────┐
       │ Exact Registry   │          │ Candidate Recovery  │
       │ brand + ref      │          │ text/OCR/image      │
       └────────┬─────────┘          └──────────┬──────────┘
                │                               │
                │                       verified only
                │                               │
                └──────────────<────────────────┘
                │
                v
       ┌──────────────────┐
       │ Stable Entity ID │
       │ + Membership     │
       └────────┬─────────┘
                │
                v
       ┌──────────────────┐
       │ Audit / Review   │
       │ Replay / Metrics │
       └──────────────────┘
```

这套方案有几个非常实际的优点：

1. **精度边界清楚**：模型没有越权路径；
2. **1000 万规模可做**：绝大多数 verified 记录是 O(log N) exact lookup；
3. **增量成本低**：只处理新批次和受影响实体；
4. **字段稀疏可处理**：缺 reference 的记录进入 extraction/recovery，不强行匹配；
5. **图片有价值但不会制造越权误匹配**：图片用于 OCR/候选/冲突；
6. **人工几百对标签能发挥最大价值**：集中标注 hard negatives 和角色边界；
7. **可解释**：每次 membership 都能回到 raw evidence；
8. **可撤销**：人工 reject 不需要全库重建；
9. **可持续升级**：canonicalizer/extractor/index 都版本化；
10. **真正符合“可漏不可错”**：ABSTAIN 是正常结果，不是失败状态。

---

## 31. 结论

catalog-forge 对当前需求最有价值的并不是“拿来即用的最终 matcher”，而是它把商品实体解析做成了一个 **可分层、可解释、可审计、可增量、可纠错** 的工程系统。

它给当前项目的核心启发可以总结成四条：

```text
1. Identifier 必须先验证、分类、规范化，再参与匹配；
2. Blocking 只负责找候选，不能越权成为同款判定；
3. 硬冲突规则必须先于模型，Unknown 必须允许 Abstain；
4. 增量索引、provenance、review/revoke 与匹配模型同等重要。
```

在此基础上，当前二奢/腕表系统还应比 catalog-forge 更保守一步：

> **把最终自动合并条件固定为：同一 verified brand 作用域内，两个记录都能被可审计证据解析到完全相同的 canonical reference；任何 reference 冲突直接否决，任何证据不足直接拒识。**

这比继续寻找一个更强的通用“同款模型”更贴合 Spec，也更容易在 100 万～1000 万持续增量数据上真正落地。

## 参考

- catalog-forge：https://github.com/alex-hahn/catalog-forge
- README / benchmarks：https://github.com/alex-hahn/catalog-forge/blob/main/README.md
- Pipeline：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/pipeline.py
- Identifier validation：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/identifiers.py
- Exact / composite blocking：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/blocking/exact.py
- Matching rules：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/rules.py
- Matching cascade：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/cascade.py
- Cluster cohesion / correction：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/cluster/components.py
- Persistent incremental index：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/incremental/index.py
- Incremental resolution：https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/incremental/resolve.py
