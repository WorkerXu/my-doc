# catalog-forge：把通用商品 Entity Resolution 改造成 Reference-first 腕表实体匹配系统

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取开源项目 **catalog-forge** 做深入分析。

- 项目：<https://github.com/alex-hahn/catalog-forge>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
- 本次输出：`奢侈品调研/a/catalog-forge.md`

分析前已先检查 `奢侈品调研/a`，当前已有结果包含 `Ameli`、`AnyMatch`、`AutoBlock`、`ComEM`、`DeepBlocker`、`GraLMatch`、`LangExtract`、`LinkTransformer`、`MOON2.0`、`TransClean`、`pyJedAI` 等，**没有 `catalog-forge.md`**，因此本次继续执行。

当前 Spec 的核心不是普通“相似商品匹配”，而是：

1. 雷小安、腕表之家、奢当家三源；
2. 数据量约 100 万～1000 万，并持续增量；
3. “同一个商品”被业务明确为 **同一个 reference number / 型号**；
4. reference 有时在结构化字段，有时埋在标题，图片也可用；
5. **precision 优先到极致，允许漏匹配，但不能误匹配**；
6. 可以人工标注几百对黄金样本。

`catalog-forge` 很适合参考，但**不能原样作为最终 matcher**。项目公开的 WDC Watches 测试结果约为：

```text
precision = 0.833
recall    = 0.880
F1        = 0.856
threshold = 0.240
```

这说明它作为“通用商品 Entity Resolution 工程骨架”很好，但 83.3% precision 与本项目“绝对不能误匹配”的目标差了一个数量级的安全要求，而且它的阈值是按常规 F1 目标调出来的，不能照搬。

本次最重要的结论是：

> **复用 catalog-forge 的工程架构，但重写 identity semantics。**
>
> 不再让模糊相似度、分类器、LLM、图片相似度直接决定“同一商品”；自动合并只允许由：
>
> ```text
> canonical_brand 完全一致
> +
> 已验证 manufacturer reference 完全一致
> +
> 无任何 hard conflict
> ```
>
> 触发。

推荐最终架构：

```text
三源 Listing
   │
   ├─ Ingest + Provenance
   │
   ├─ Brand Normalization
   │
   ├─ Reference Candidate Extraction
   │    ├─ 结构化字段
   │    ├─ 标题/描述规则
   │    ├─ 品牌专用 parser
   │    ├─ 图片 OCR
   │    └─ LLM/VLM（只提候选，不做最终事实）
   │
   ├─ Identifier Role Classification
   │    ├─ MANUFACTURER_REFERENCE
   │    ├─ PLATFORM_SKU
   │    ├─ DEALER_CODE
   │    ├─ LISTING_ID
   │    ├─ SERIAL_NUMBER
   │    └─ UNKNOWN
   │
   ├─ Brand-aware Canonicalization + Validation
   │
   ├─ Reference Evidence Gate
   │    ├─ MATCH：brand + verified canonical reference exact equality
   │    ├─ NON_MATCH：两个可信 reference 冲突
   │    └─ ABSTAIN/REVIEW：缺失、歧义、证据不足
   │
   ├─ Reference Entity
   │    └─ Listing / Offer membership
   │
   └─ Unresolved only
        ├─ Blocking / BM25 / char n-gram / ANN
        ├─ 图片候选召回
        ├─ calibrated classifier / LLM judge
        └─ 仅用于复核排序或补证，不能绕过 Reference Gate
```

如果只做第一期可直接落地的 MVP，我建议甚至先**不上 pairwise ML matcher**：

```text
结构化 reference + 标题 reference 抽取
→ 品牌内严格规范化
→ exact join
→ 冲突/缺失全部拒识
```

这条链路的覆盖率可能不高，但完全符合当前业务“可漏不可错”的优先级，而且能迅速建立高质量的 canonical reference graph。后续 OCR、模型、LLM 都只负责把更多记录安全地送入这条确定性主干。

---

# 1. catalog-forge 到底是什么

`catalog-forge` 是一个面向异构商品目录的 Python Entity Resolution / Attribute Extraction 项目，核心目标是把多个 feed 的 listing 合成统一目录，同时显式建模：

```text
product
  ↓
variant
  ↓
offer
```

项目的关键价值不在某一个模型，而在于它把商品 ER 拆成了可观测、可重跑、可审计的流水线：

```text
ingest
→ normalize
→ taxonomy
→ extract
→ block
→ match
→ cluster
→ hierarchy
→ golden record
→ review queue
```

源码里的 `src/catalog_forge/pipeline.py` 会把中间结果逐阶段落成 Parquet，而不是全部藏在内存中：

```text
listings_raw.parquet
provenance.parquet
listings.parquet
extractions.parquet
candidates.parquet
decisions.parquet
clusters.parquet
merges.parquet
hierarchy.parquet
golden.parquet
conflicts.parquet
review_queue.parquet
run.json
```

这个设计对当前项目非常重要，因为“绝对不能误匹配”不是靠一句 prompt 或一个阈值保证的，必须能回答：

```text
为什么这两条记录会被合并？
原始 reference 是什么？
是谁抽出来的？
用了哪一版规范化规则？
有哪几条独立证据？
有没有冲突证据？
哪个 rule 做的决定？
如果人工判错，能否撤销？
```

`catalog-forge` 的 artifact + provenance + reason log 思路正适合成为这套安全要求的工程骨架。

---

# 2. 项目的真实技术架构

## 2.1 Orchestration：阶段化、可重放的 Pipeline

`src/catalog_forge/pipeline.py` 的核心不是复杂 DAG，而是一个明确的 stage contract：上一阶段产出文件，下一阶段读取并继续加工。每一阶段记录：

- 耗时；
- 行数；
- stage 统计；
- config digest；
- 关键决策与冲突。

这比“一个大函数读 CSV 最后吐 match.csv”更适合生产，因为任何问题都可以在中间层重现。

对当前二奢系统，我建议直接沿用这种思路，并把 artifact 改造成：

```text
00_raw_listings.parquet
01_normalized_listings.parquet
02_reference_candidates.parquet
03_reference_evidence.parquet
04_reference_verified.parquet
05_exact_memberships.parquet
06_unresolved_candidates.parquet
07_review_decisions.parquet
08_reference_entities.parquet
09_conflicts.parquet
10_decision_log.parquet
run.json
```

原则是：

> **最终实体关系必须可以从原始数据 + 版本化规则 + 决策日志完全重放。**

这也是日后改 reference 规范化规则时避免“历史簇不知道为什么这么合”的基础。

---

## 2.2 Extraction Cascade：便宜方法优先，昂贵模型只处理剩余项

`src/catalog_forge/extract/cascade.py` 的默认顺序是：

```text
IdentifierStage
→ PatternStage
→ LearnedStage（可选）
→ LLMStage
```

其设计思想非常值得复用：

> 前面的 stage 已经解决某个属性后，后面的昂贵 stage 不再重复处理；同时统计每个 stage 解决了多少值、花了多少时间。

项目自带样本里给出的量级说明大致是：

```text
identifiers   ~3% values    ~0.02s / 1k listings
patterns     ~78% values    ~0.4s  / 1k listings
learned      ~19% values    ~6s    / 1k listings
llm           disabled by default
```

对腕表 reference 非常适合改成：

```text
Stage 0：结构化字段直取
Stage 1：品牌专用高精度 Regex / Parser
Stage 2：通用标题 reference candidate extractor
Stage 3：图片 OCR reference candidate
Stage 4：小模型 / LLM / VLM 候选抽取
Stage 5：外部 reference dictionary 验证
```

但这里要做一个比 catalog-forge 更严格的改变：

> **前一层抽到的值不是“最终 attribute”，而是 evidence。**

也就是说，不能把“LLM 说 reference=XXX”直接写进 canonical 字段；应记录成：

```json
{
  "candidate": "126610LN",
  "source_kind": "title_llm",
  "confidence": 0.82,
  "raw_evidence": "...劳力士水鬼 126610LN...",
  "span_start": 12,
  "span_end": 20,
  "extractor_version": "ref-llm-v3"
}
```

随后由 Reference Evidence Gate 决定这个候选是否能升级成 `verified_reference`。

---

## 2.3 Identifier Validation：最值得直接借鉴的一层

`src/catalog_forge/identifiers.py` 把 identifier 当作“有语义和校验规则的对象”，而不是普通字符串。

它做了几件很正确的事：

1. GTIN 用 checksum 验证，而不是看到 13 位数字就相信；
2. GTIN 统一到 14 位 canonical key；
3. MPN 做单独规范化；
4. 从自由文本抽 identifier 时保存 span 和 confidence；
5. “缺失”与“冲突”严格区分：

```text
missing identifier     => unknown
valid id A != valid B  => strong contradiction
```

这一点和当前需求高度一致。

但是它的 `normalize_mpn()` **不能直接用于腕表 reference**。该函数要求：

```text
长度 4~40
并且必须同时包含字母和数字
```

这在通用商品 MPN 中合理，但腕表 reference 可能出现全数字形式，也可能包含品牌特定的 `/`、`.`、`-`、后缀等结构。

所以应新增：

```python
normalize_watch_reference(brand_id, raw_reference)
```

而不是复用通用 `normalize_mpn()`。

同时也不能直接复用 `blocking/exact.py` 里的 `model_tokens()`，因为它明确排除了纯数字 token；对腕表来说，纯数字 reference 可能恰恰是最强身份锚点。

这是本次分析中一个非常具体、可以直接避免生产事故的改造点。

---

## 2.4 Blocking：Candidate Generation 与 Decision 必须彻底分离

`catalog-forge` 的 `blocking/exact.py` 很明确地说：即使同一 exact identifier，也只应该先作为 candidate signal，因为 feed 可能：

- 错填 identifier；
- 重用编号；
- 把 bundle 编号放进商品字段；
- 把内部 SKU 填到 identifier 字段。

它提供：

```text
ExactIdentifierBlocking
CompositeKeyBlocking
MinHash
ANN
Sorted Neighborhood
```

而 `CompositeKeyBlocking` 会使用类似：

```text
brand + model_token
brand + quantity
brand + color + size
title_prefix
```

生成候选。

这里最值得当前系统照搬的不是具体算法，而是边界：

> **Blocking 只能回答“值得比较谁”，永远不能回答“谁就是同款”。**

当前项目推荐两条 candidate path：

### Path A：确定性 Reference Key

只要一条记录已经得到 `verified canonical reference`：

```text
(brand_id, canonical_reference)
```

直接走 hash index / exact index，不需要 ANN，也不需要 pairwise 全比较。

### Path B：Unresolved Candidate Retrieval

没有 verified reference 的记录才进入：

```text
brand exact block
→ reference-like token block
→ char n-gram / BM25
→ title embedding ANN
→ image ANN
```

这些候选只送给：

- reference 补证；
- 人工 review；
- hard-negative mining；
- 模型排序。

**不能因为候选 top1 很像就 auto-match。**

---

# 3. Match Cascade：项目最值得复用的“决策成本分层”

`src/catalog_forge/match/cascade.py` 把 pairwise matching 分为三层：

```text
1. Rules
2. Calibrated Classifier
3. Judge（只处理不确定区间）
```

默认 uncertainty band 是：

```text
0.35 <= p < 0.65
```

并且每一层都记录：

- 解决 pair 数；
- 占比；
- 总耗时；
- 单 pair 耗时；
- rule coverage。

这是一种很好的“成本可解释架构”：

```text
便宜且确定的规则解决大多数
模型处理规则解决不了的
最昂贵 judge 只处理少数模糊项
```

对当前项目建议改成：

```text
Stage 1：Reference Hard Rules
  ↓ unresolved only
Stage 2：Evidence Ranker / Calibrated Classifier
  ↓ ambiguous only
Stage 3：LLM/VLM Evidence Judge
  ↓
ABSTAIN / HUMAN REVIEW
```

关键变化是：

> **Stage 2/3 不再拥有 auto-MATCH 权限。**

模型可以输出：

```text
“这条 unresolved listing 很可能是 Rolex 126610LN，建议优先核验”
```

但真正写入 reference entity 前仍必须回到 Stage 1 的确定性校验。

换句话说，模型能提升 coverage，但不能放松 identity rule。

---

# 4. Rule Layer：几乎可以直接映射成腕表 Reference Gate

`src/catalog_forge/match/rules.py` 的设计非常值得照搬。

每条规则返回：

```text
MATCH / NON_MATCH / UNDECIDED
confidence
rule name
human-readable reason
```

并且规则按顺序执行，第一条 decisive rule 生效。

项目默认把“硬冲突规则”放在 match 规则前面，这是正确的安全设计。

例如：

```text
different valid GTIN -> NON_MATCH
same valid GTIN      -> MATCH
same brand + MPN     -> MATCH
```

对当前 Spec 可以直接改造成：

```text
R0 brand_hard_conflict
R1 reference_role_invalid
R2 different_verified_reference
R3 same_verified_reference_and_brand
R4 conflicting_reference_evidence
R5 otherwise_unresolved
```

推荐伪代码：

```python
def decide_identity(a, b):
    # 1. 已确认品牌冲突，直接否决
    if a.brand_verified and b.brand_verified and a.brand_id != b.brand_id:
        return NON_MATCH("different verified brand")

    # 2. 两边都有可信 manufacturer reference
    if a.ref_verified and b.ref_verified:
        if a.reference_role != "MANUFACTURER_REFERENCE":
            return ABSTAIN("left identifier role is not manufacturer reference")
        if b.reference_role != "MANUFACTURER_REFERENCE":
            return ABSTAIN("right identifier role is not manufacturer reference")

        # 3. 同品牌 reference 不同 = 绝对否决
        if a.canonical_reference != b.canonical_reference:
            return NON_MATCH("different verified reference")

        # 4. 有任何独立冲突证据，宁可不合
        if has_hard_conflict(a, b):
            return ABSTAIN("reference agrees but independent evidence conflicts")

        # 5. 唯一自动合并通道
        return MATCH("same verified brand + exact canonical reference")

    # 6. 缺失不是“不相同”，而是“不知道”
    return ABSTAIN("reference evidence insufficient")
```

这里不要给 `MATCH` 一个类似 0.95 的“概率意义”。

自动合并规则更适合表示为：

```text
rule_invariants_satisfied = True / False
```

因为当前业务不是要“95% 大概对”，而是要“只有满足业务身份不变量才能写关系”。

---

# 5. Calibrated Classifier：保留，但改变它的职责

`catalog-forge` 的 `PairClassifier` 使用 HistGradientBoosting + calibration。

项目选择树模型而不是神经网络的理由也很合理：

- 稀疏字段和 missingness indicator 很适合树模型；
- CPU 训练快；
- feature importance 可解释；
- 使用 isotonic / sigmoid 校准后，输出更接近概率，而不是任意 score。

它还把：

```text
Brier score
ECE
Average Precision
feature importance
```

写进训练报告。

这些能力当前仍然有价值，但应换目标。

建议训练：

```text
P(reference candidate is correct | evidence features)
```

或者：

```text
P(pair deserves human review soon | evidence conflict + similarity)
```

而不是训练：

```text
P(pair is same product) -> 直接 merge
```

原因很简单：Spec 已经给出了确定的 same-product 定义——same reference。既然最终业务判据是结构化 identity key，就没有必要让一个统计模型重新学习“什么叫同一商品”。

### 推荐特征

可以从 catalog-forge 的 feature design 中保留：

```text
brand_known
brand_agreement
reference_candidate_known
reference_exact_agreement
reference_edit_distance
reference_pattern_valid
reference_dictionary_hit
identifier_role
source_trust
field_vs_title_agreement
title_vs_ocr_agreement
ocr_confusion_count
reference_evidence_count
model_token_overlap
title_similarity
image_similarity
candidate_rank
```

其中：

```text
reference_exact_agreement
verified_reference_conflict
identifier_role
```

必须由规则层解释，不能只作为普通 feature 被模型权重“平均掉”。

---

# 6. Clustering：catalog-forge 已经意识到“传递闭包很危险”

这是该项目最值得当前系统借鉴的第二个核心点。

`src/catalog_forge/cluster/components.py` 的注释明确指出：

```text
pairwise matching 是局部的；
transitive closure 是全局且无情的；
一条错边把两个密集簇连起来，损失与两个簇大小的乘积有关。
```

项目做法是：

1. 先按 threshold 做 connected components；
2. 计算 component cohesion 与 density；
3. 如果 component 像“两簇被一条桥连接”，则用 correlation clustering 重切；
4. 所有 merge / split 留日志；
5. 支持 `revoke()`，人工拒绝一条边后重算受影响部分。

这与我之前分析 TransClean 时得到的结论高度一致：

> **一条 false-positive bridge edge 的损失远高于一条 pair 错误。**

不过当前项目可以比 catalog-forge 更进一步：

### Primary Identity Cluster 不使用普通 connected component

既然业务实体就是 reference，则：

```text
Reference Entity = (canonical_brand_id, verified_canonical_reference)
```

成员关系是：

```text
listing -> reference_entity
```

不需要：

```text
A~B
B~C
所以 A=C
```

只需要：

```text
A.reference_key = K
B.reference_key = K
C.reference_key = K
```

这从根源上消除了 bridge edge 扩散。

### correlation clustering / cohesion check 放到哪里？

只放在“弱证据候选图”中，用于：

- 发现异常候选簇；
- review 排序；
- 检测可能错抽的 reference；
- 防止人工/模型产生错误 alias。

它不负责产生最终 auto-merge。

---

# 7. Product → Variant → Offer 如何映射到当前腕表业务

`catalog-forge` 的层级设计很有价值，但当前 Spec 的“product”定义与普通电商不完全一样。

推荐映射为：

```text
Model Family / Series（可选，仅导航）
        ↓
Reference Entity（真正的 identity）
        ↓
Source Listing / Offer
```

例如：

```text
某系列
  ├─ reference A
  │    ├─ 雷小安 listing 1
  │    ├─ 腕表之家 listing 2
  │    └─ 奢当家 listing 3
  │
  └─ reference B
       ├─ 雷小安 listing 4
       └─ 腕表之家 listing 5
```

即：

- 系列相同 ≠ 同一商品；
- 外观相似 ≠ 同一商品；
- 标题相似 ≠ 同一商品；
- 图片相似 ≠ 同一商品；
- **reference 相同才是当前业务定义下的同一商品**。

因此 catalog-forge 的 `variant` 层在当前系统里其实更接近真正的 `reference entity`。

---

# 8. Reference 不是一个字符串字段，而是一套 Evidence Model

这是建议直接落地的核心数据模型。

## 8.1 `reference_evidence`

建议每一个候选都独立存，而不是只留最终值：

```text
reference_evidence
------------------
evidence_id
listing_id
source_id
brand_id
candidate_raw
candidate_canonical
identifier_role
source_kind
source_field
raw_fragment
span_start
span_end
image_id
extractor_name
extractor_version
normalizer_version
pattern_id
confidence
validation_status
created_at
```

`source_kind` 例如：

```text
STRUCTURED_REFERENCE_FIELD
TITLE_EXPLICIT_LABEL
TITLE_BRAND_PATTERN
DESCRIPTION_PATTERN
OCR_CASEBACK
OCR_WARRANTY_CARD
OCR_TAG
LLM_TEXT
VLM_IMAGE
EXTERNAL_REFERENCE_CATALOG
MANUAL
```

## 8.2 `listing_identity`

每条 listing 当前安全结论：

```text
listing_identity
----------------
listing_id
canonical_brand_id
verified_reference
reference_role
reference_status
winning_evidence_id
verification_rule
normalizer_version
identity_status
updated_at
```

`reference_status`：

```text
VERIFIED
AMBIGUOUS
CONFLICT
MISSING
INVALID
```

`identity_status`：

```text
AUTO_LINKED
UNRESOLVED
REVIEW
REJECTED
```

## 8.3 `reference_entity`

```text
reference_entity
----------------
reference_entity_id
canonical_brand_id
canonical_reference
official_reference_display
status
created_at
updated_at
```

生产环境不要把 entity id 直接永久绑定成 `hash(reference string)`，因为未来 normalizer 升级时可能导致 ID 漂移。

推荐：

- `reference_entity_id` 使用不可变 UUID / Snowflake ID；
- `(brand_id, canonical_reference)` 是 versioned unique key；
- 通过 alias 表处理后续规范化迁移。

## 8.4 `reference_alias`

```text
reference_alias
---------------
brand_id
alias_value
canonical_reference
normalizer_version
alias_type
status
provenance
```

这样如果之后发现某平台长期把：

```text
“ABC-123”
```

写成：

```text
“ABC 123”
```

可以新增安全 alias，而不是篡改历史实体 ID。

## 8.5 `decision_log`

```text
decision_log
------------
decision_id
left_listing_id
right_listing_id
reference_entity_id
decision
rule_code
reason
input_evidence_ids
rule_version
model_version
reviewer
created_at
```

每个自动 merge 必须能输出人类可理解 reason：

```text
same verified canonical brand Rolex
+ same verified manufacturer reference 126610LN
+ source reference fields passed brand parser
+ no conflicting evidence
```

而不能只输出：

```text
similarity = 0.9821
```

---

# 9. Identifier Role：必须先搞清楚“这个编号是什么”

这是当前系统最容易发生灾难性误匹配的地方。

同一条 listing 中可能出现：

```text
manufacturer reference
平台 SKU
店铺货号
dealer product code
listing id
序列号 / serial
保卡编号
配件编号
机芯编号
GTIN
```

如果把任意“看起来像型号的字母数字串”都当 manufacturer reference，就会出现高置信错误。

建议定义：

```python
class IdentifierRole:
    MANUFACTURER_REFERENCE
    GTIN
    PLATFORM_SKU
    DEALER_CODE
    LISTING_ID
    SERIAL_NUMBER
    MOVEMENT_NUMBER
    ACCESSORY_PART_NUMBER
    UNKNOWN
```

当前 Spec 只有：

```text
MANUFACTURER_REFERENCE
```

可以成为自动同款主键。

注意：

- `SERIAL_NUMBER` 可能识别具体一只表，但 Spec 定义的是“同 reference 商品”，所以 serial 不能替代 reference；
- `GTIN` 可以作为很强的辅助核验，但除非业务正式改变 identity definition，否则不应在 reference 缺失时偷偷替代 reference；
- `PLATFORM_SKU / DEALER_CODE` 绝对不能跨源自动连。

这与 catalog-forge “先验证 identifier 类型再使用”的思想一致，但需要把 role taxonomy 做得更细。

---

# 10. Brand-aware Canonicalization：不能用一个全局正则暴力去符号

通用目录系统很容易做：

```text
upper
remove spaces
remove hyphens
remove dots
```

对普通 MPN 可能够用，但对高价值腕表的 precision-first 匹配不够安全。

推荐 canonicalization 分两层。

## 10.1 Safe Global Normalization

只做没有语义风险的操作：

```text
Unicode NFKC
全角 → 半角
trim
连续空白折叠
英文统一大写
统一常见 Unicode dash 到标准 dash
去零宽字符
```

此层不做模糊替换。

## 10.2 Brand-specific Grammar

每个品牌配置：

```text
允许字符集
长度区间
分段结构
可忽略分隔符
后缀规则
大小写规则
常见展示格式
```

例如（仅示意）：

```text
ROLEX:
  接受连续数字 + 可选字母后缀
  不能把 LN / LV 等后缀删除

OMEGA:
  允许多段点号格式
  可以建立“展示格式”与“比较 key”的双表示
  但任何字段都不能因删除末段而碰撞
```

建议保存两个值：

```text
reference_display_canonical
reference_match_key
```

例如：

```text
raw:     210.30.42.20.01.001
Display: 210.30.42.20.01.001
Key:     21030422001001
```

但只有当 `OMEGA` parser 明确确认这种分隔符等价时才能生成 Key。

### 特别禁止的行为

```text
O ↔ 0
I ↔ 1
S ↔ 5
B ↔ 8
```

这些 OCR confusion **不能直接写进 canonical key**。

正确做法是：

```text
OCR 原始候选：12O610LN
生成纠错候选：120610LN
标记 correction=O_TO_0
进入 validation/review
```

而不是默默把所有 O 变成 0。

---

# 11. Reference Extraction：建议的五级瀑布

## Level 0：结构化字段

优先级最高，但仍做 role + format validation。

```text
source.reference
source.model_no
source.ref_no
```

不能因为字段名叫 `型号` 就 100% 相信，因为平台可能混入商家自定义 SKU。

建议对三源分别维护 adapter：

```python
SOURCE_FIELD_POLICY = {
    "leixiaoan": {...},
    "watchhome": {...},
    "shedangjia": {...},
}
```

记录每个字段历史质量：

```text
precision
null rate
role error rate
brand coverage
```

## Level 1：标题显式标签

例如：

```text
Ref.
Reference
型号
参考号
Ref No
```

标签后的候选比任意裸 token 更可信。

## Level 2：Brand Parser

先识别品牌，再按品牌 grammar 识别 reference-like token。

这是最适合高 precision 的标题抽取层。

## Level 3：图片 OCR

不要对所有图片一视同仁。

先做图片用途分类：

```text
表背
表盘
保卡
吊牌
盒证
商品棚拍
生活图
无关图
```

再优先对：

```text
表背 / 保卡 / 吊牌
```

做 OCR。

OCR 结果仍然只是 `reference_evidence`，需要：

```text
brand grammar
+ reference dictionary
+ 其他来源证据
```

核验。

## Level 4：LLM / VLM

LLM 的职责应是：

```text
“从这段文本中指出可能的 reference，并返回原文 span”
```

而不是：

```text
“根据这只表的描述猜它是什么型号”
```

必须要求 extractive evidence：

```text
candidate 必须能回指原文 token 或 OCR token
```

如果模型输出在原文/图片 OCR 中找不到支持，则不能作为 verified reference。

---

# 12. Reference Evidence Gate：系统真正的安全边界

这是整个落地方案中最关键的组件。

推荐为每条 listing 输出：

```text
VERIFIED_REFERENCE
AMBIGUOUS_REFERENCE
CONFLICTING_REFERENCE
MISSING_REFERENCE
```

## 12.1 可自动 VERIFIED 的典型情况

建议第一期只接受非常保守的组合，例如：

### A. 可信结构化字段 + 品牌 parser 通过

```text
source field trust = HIGH
brand verified
reference role = MANUFACTURER_REFERENCE
brand grammar valid
```

### B. 标题显式标签 + 品牌 parser + reference dictionary 命中

### C. 两个独立证据源一致

例如：

```text
结构化字段 == OCR 保卡
```

或：

```text
标题 parser == OCR 表背
```

## 12.2 必须进入 CONFLICT 的情况

例如：

```text
结构化字段：126610LN
标题显式 Ref：126610LV
```

或者：

```text
标题：126610LN
保卡 OCR：126610LV
```

对于 precision-first 系统，不应该“做个加权投票选高分”，而是：

```text
CONFLICT -> 不自动入实体 -> review
```

## 12.3 MISSING 不是 NON_MATCH

这点必须像 catalog-forge 的 `identifiers_agree()` 一样保留三值逻辑：

```text
same      -> positive evidence
different -> negative evidence
missing   -> unknown
```

绝不能把：

```text
reference missing
```

等价成：

```text
reference different
```

否则会损失大量后续补证机会。

---

# 13. 最终 Auto-Match 真值表

建议把生产规则固化成单元测试和数据库约束。

| Brand A | Ref A | Brand B | Ref B | 结果 |
|---|---|---|---|---|
| verified same | verified R1 | verified same | verified R1 | `AUTO_MATCH` |
| verified same | verified R1 | verified same | verified R2 | `HARD_NON_MATCH` |
| verified B1 | verified R1 | verified B2 | verified R1 | `HARD_NON_MATCH` |
| verified same | verified R1 | verified same | missing | `ABSTAIN` |
| verified same | candidate R1 | verified same | verified R1 | `REVIEW` |
| verified same | OCR-only R1 | verified same | OCR-only R1 | `REVIEW`，第一期不自动合 |
| verified same | verified R1 | verified same | verified R1 + conflict evidence | `REVIEW/QUARANTINE` |
| missing brand | verified R1 | verified brand | verified R1 | `ABSTAIN` |

第一期宁可比这个更保守，也不要更宽松。

---

# 14. 图片到底怎么用：只做“证据补全”和“反证”，不做身份主键

当前有图片，这是优势，但也是最容易把系统带向错误方向的地方。

腕表同系列、相邻 reference 的视觉差异可能极小；同 reference 又可能因为：

- 拍摄角度；
- 光线；
- 表带更换；
- 磨损；
- 后配件；
- 盒证是否出现；

造成巨大视觉差异。

因此不推荐：

```text
image similarity > 0.98 -> same product
```

推荐：

```text
图片 -> Image Utility Classifier
     -> OCR / visual candidate retrieval
     -> reference evidence
     -> Reference Gate
```

另外可以用图片做 **hard contradiction**：

```text
文本声称某 reference
但图片识别到明显不同 reference / 不同品牌
```

这时进入 quarantine，而不是让文本规则强行合并。

视觉 embedding 最适合做：

- unresolved candidate recall；
- hard-negative mining；
- 人工 review 排序；
- 找“外观极像但 reference 不同”的危险样本。

---

# 15. 百万～千万级：不要先做 N² Pair Matching

当前规模 100 万～1000 万，三源笛卡尔积不可接受。

但因为业务身份键非常明确，主链其实不需要 pairwise all-vs-all。

对已验证 reference 的记录：

```text
key = (brand_id, canonical_reference)
```

直接：

```text
hash group / database unique index / sort merge
```

复杂度接近：

```text
O(N)
```

或排序版：

```text
O(N log N)
```

而不是：

```text
O(N²)
```

只有 unresolved 小集合需要 expensive candidate retrieval。

例如假设 1000 万记录中最终 60% 能通过规则/结构化字段安全得到 reference，则：

```text
600 万条完全不需要 pairwise 模型
```

剩余 400 万也应先做品牌分区、token blocking、reference-like retrieval，再进入少量 top-k 复核。

这才是当前需求真正的规模解法。

---

# 16. Persistent Incremental Index：catalog-forge 的增量设计可以直接借

`src/catalog_forge/incremental/index.py` 明确指出：

> 增量处理不是“拿新 batch 再跑一次小型 batch pipeline”。新 listing 必须查询已有 catalog，如果每次都重建百万级 blocking index，就失去了增量意义。

项目会持久化：

```text
exact identifier postings
MinHash band postings
brand/model token postings
known listing ids
```

并且 index 参数有版本，一旦 MinHash 参数不一致就拒绝混用。

这非常适合当前系统改造成：

```text
ReferenceExactIndex:
  brand_id|canonical_reference -> reference_entity_id

ReferenceCandidateIndex:
  brand_id|reference_token -> listing/entity ids

TitleLexicalIndex:
  brand partition -> candidate docs

ImageIndex:
  brand/category partition -> candidate vectors
```

新增 batch：

```text
1. ingest new / changed listings
2. extract evidence
3. verify reference
4. exact key lookup
5. 安全 attach / create entity
6. unresolved 才走 candidate index
7. append index
```

主成本与 **新 batch 大小** 成正比，而不是与全量 catalog 成正比。

---

# 17. 增量合并时必须比 catalog-forge 更保守

`src/catalog_forge/incremental/resolve.py` 会按匹配概率把新记录加入已有 cluster，甚至在新证据连接两个既有 cluster 时自动 merge existing clusters。

在普通 ER 中合理，但当前项目不能直接采用：

```python
if p >= threshold:
    merge(cluster_a, cluster_b)
```

应改成：

```python
if same_verified_reference(a, b) and no_conflict(a, b):
    safe_attach()
else:
    do_not_merge()
```

特别是：

> **新 listing 永远不应该仅凭模糊模型把两个已经存在的 reference entities 合并。**

生产规则可以直接写成：

```text
existing entity merge 默认禁止
```

只有：

```text
人工确认 alias / canonicalization bug
```

才允许进行 entity-level merge，而且必须留下 reversible audit log。

---

# 18. Stable ID 与可回滚是必须项

`catalog-forge` 的 incremental state 会尽量保持 cluster id 稳定；人工 reject 一条 merge 后只重算受影响 cluster，而不是重跑全库。

当前项目建议更强：

```text
reference_entity_id 一旦创建，默认不可变
```

当 normalizer 更新时：

```text
old alias -> entity_id
new canonical key -> same entity_id
```

如果发现历史错误：

```text
revoke membership
rebuild affected entity only
```

而不是重编号整个目录。

这种稳定性会直接影响：

- 下游商品主键；
- 缓存；
- 搜索索引；
- 分析报表；
- 人工 review 历史；
- 外部链接。

---

# 19. 推荐的实际代码目录

如果基于 catalog-forge 思路重新实现，我建议不要直接把所有代码塞进一个 matcher，而是拆成：

```text
reference_resolver/
├── ingest/
│   ├── leixiaoan.py
│   ├── watchhome.py
│   └── shedangjia.py
├── brand/
│   ├── aliases.py
│   └── normalize.py
├── reference/
│   ├── roles.py
│   ├── extract_structured.py
│   ├── extract_text.py
│   ├── extract_ocr.py
│   ├── normalize.py
│   ├── brand_parsers/
│   │   ├── rolex.py
│   │   ├── omega.py
│   │   └── ...
│   └── validate.py
├── evidence/
│   ├── model.py
│   ├── gate.py
│   └── conflict.py
├── decision/
│   ├── rules.py
│   └── reason_codes.py
├── blocking/
│   ├── exact.py
│   ├── lexical.py
│   └── vector.py
├── entity/
│   ├── repository.py
│   ├── membership.py
│   └── correction.py
├── incremental/
│   ├── index.py
│   └── resolve.py
├── review/
│   ├── queue.py
│   └── active_learning.py
├── eval/
│   ├── reference_extraction.py
│   ├── matching.py
│   ├── cluster_safety.py
│   └── regression.py
└── pipeline.py
```

如果希望最快 PoC，也可以直接 fork catalog-forge，然后重点替换：

```text
identifiers.py
blocking/exact.py
match/rules.py
incremental/resolve.py
variants/hierarchy.py
```

同时保留：

```text
pipeline / artifact
provenance
cascade accounting
review queue
cluster correction log
persistent index
benchmark harness
```

---

# 20. 推荐存储技术栈

Spec 说技术栈无限制，但 100 万～1000 万并不需要一开始就堆最重的分布式系统。

## 第一阶段建议

```text
Python 3.12
Polars / PyArrow
Parquet
PostgreSQL
对象存储 S3 / MinIO（图片与 artifact）
```

原因：

- Polars 很适合 1000 万级列式 batch 清洗；
- Parquet 可审计、可重跑；
- PostgreSQL 可以承载 reference entity、unique key、membership、decision log；
- 精确 `(brand_id, canonical_reference)` B-tree / hash 索引足够快。

## Unresolved 规模上来后再加

```text
OpenSearch / Elasticsearch：BM25、char n-gram
pgvector / Qdrant / Milvus：文本和图片候选向量
Kafka：如果抓取更新频率需要事件流
```

不建议第一天就把所有记录 embedding 后进向量库，因为当前核心问题不是“相似搜索能力不够”，而是“身份判据和证据治理必须正确”。

---

# 21. PostgreSQL 可直接落地的核心表

示意 DDL：

```sql
CREATE TABLE reference_entity (
    id                  BIGSERIAL PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    display_reference   TEXT,
    status              TEXT NOT NULL DEFAULT 'ACTIVE',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, canonical_reference)
);

CREATE TABLE reference_membership (
    listing_id          TEXT PRIMARY KEY,
    reference_entity_id BIGINT NOT NULL REFERENCES reference_entity(id),
    decision_id         BIGINT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reference_evidence (
    id                  BIGSERIAL PRIMARY KEY,
    listing_id          TEXT NOT NULL,
    candidate_raw       TEXT,
    candidate_canonical TEXT,
    identifier_role     TEXT NOT NULL,
    source_kind         TEXT NOT NULL,
    raw_fragment        TEXT,
    confidence          DOUBLE PRECISION,
    validation_status   TEXT NOT NULL,
    extractor_version   TEXT NOT NULL,
    normalizer_version  TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE identity_decision (
    id                  BIGSERIAL PRIMARY KEY,
    listing_id          TEXT NOT NULL,
    reference_entity_id BIGINT,
    decision            TEXT NOT NULL,
    rule_code           TEXT NOT NULL,
    reason              TEXT NOT NULL,
    rule_version        TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

真正生产时还应加：

- evidence relation；
- review task；
- alias；
- conflict；
- source snapshot version；
- manual override history。

---

# 22. Golden Labels：几百对应该花在哪里

当前只有几百对人工预算，不能随机抽。

最值得标的是 **hard negatives**，而不是大量明显不同商品。

建议大致分布：

```text
40%：同品牌、同系列、外观近似、reference 仅差 1~2 个字符
25%：真实 positive，跨平台 reference 写法/分隔符不同
15%：reference 埋标题、结构化字段缺失
10%：OCR O/0、I/1、B/8 等混淆
10%：平台 SKU / dealer code / serial 被误识别成 reference
```

人工 UI 不应只问：

```text
“这两个商品是不是同款？”
```

最好同时问：

```text
A 的 manufacturer reference 是什么？
B 的 manufacturer reference 是什么？
该编号属于 manufacturer ref 还是平台 SKU？
哪些证据支持？
```

这样一个标注动作可以同时训练/评测：

- reference extraction；
- identifier role；
- canonicalization；
- pair decision；
- conflict detection。

比只存 `match=1/0` 有价值得多。

---

# 23. 评测指标不能只看 F1

catalog-forge 自己已经展示了一个很重要的事实：删除 cohesion check 时，pairwise F1 几乎不动，但 cluster B³ F1 会明显下降。

因此当前项目至少分四层评测。

## 23.1 Reference Extraction

```text
precision of extracted reference
recall of extracted reference
exact canonical accuracy
role classification precision
```

其中 precision 优先。

## 23.2 Auto-Match

主指标：

```text
Auto-Match Precision
False Positive Count
```

Recall 是次级指标。

## 23.3 Cluster Safety

```text
cluster precision
reference conflict count inside entity
mixed-brand entity count
mixed-verified-reference entity count
bridge-merge incidents
```

其中下面两个必须永远为 0：

```text
mixed-brand auto entity
mixed-verified-reference auto entity
```

## 23.4 Coverage

```text
verified-reference coverage
auto-link coverage
review rate
abstain rate
OCR recovery rate
```

通过这些指标逐步提高覆盖，而不是牺牲 precision。

---

# 24. “绝对不能误匹配”不能靠统计阈值保证

这是需要特别强调的一点。

即使在黄金集上：

```text
0 false positives
```

也不能数学上证明生产数据永远 0 FP。

因此业务目标必须转化成 **系统不变量**：

```text
Invariant 1:
AUTO_MATCH 必须有 verified manufacturer reference。

Invariant 2:
AUTO_MATCH 两侧 canonical brand 必须相同。

Invariant 3:
AUTO_MATCH 两侧 canonical reference 必须字节级一致。

Invariant 4:
任何 verified reference conflict 都有否决权。

Invariant 5:
模型、LLM、图片相似度不能覆盖 hard veto。

Invariant 6:
所有 AUTO_MATCH 必须有可追溯 evidence。

Invariant 7:
所有 merge 都能 revoke。
```

然后用黄金集和线上抽检验证这些 invariant 有没有实现 bug。

这比说：

```text
threshold 调到 0.999
```

安全得多。

---

# 25. Hard Negative Regression Suite

建议把每次线上发现的误判风险沉淀成永久回归测试。

例如：

```text
126610LN != 126610LV
```

类似这种“只差后缀”的 pair 必须永久进 tests。

测试类别：

```text
same series different reference
different suffix
different last digit
spacing-only equivalent
punctuation-only equivalent (brand-approved)
OCR O/0 confusion
platform SKU collision
same ref text but different brand
structured ref vs title conflict
OCR ref vs structured ref conflict
partial ref vs full ref
```

测试不仅测 pair decision，还测：

```text
canonicalizer 不产生碰撞
entity membership 不发生越权合并
incremental update 不把历史两个实体粘起来
```

---

# 26. Review Queue：模型应该优化“谁最值得人工看”

`catalog-forge` 已经有 `review_queue.parquet`，这正是当前几百条人工预算应该放的位置。

review priority 推荐：

```text
P1：两个高可信 reference 冲突
P2：已有 entity 出现新 conflict evidence
P3：同品牌相邻 reference 的高相似 hard negative
P4：高价值热门 reference 的 weak candidate
P5：OCR 与标题不一致
P6：模型极度自信但 hard rule 不允许 merge
```

尤其 P6 很重要：

> 模型最自信、但规则拒绝的样本，往往能暴露新的数据污染模式。

这比只看模型 0.5 附近的不确定样本更符合“防 false positive”的业务目标。

---

# 27. Source Trust 不能是一个全局分数

不要简单定义：

```text
腕表之家 = 0.9
雷小安 = 0.8
奢当家 = 0.85
```

更合理的是：

```text
trust(source, field, brand, extraction_mode)
```

因为同一个平台可能：

- 品牌字段很好；
- reference 独立字段偶尔混 SKU；
- 某些品牌标题很规范；
- 某些品牌 reference 经常缺失。

所以 trust profile 至少按：

```text
source × field × brand
```

维护，并由黄金集持续估计。

这个 trust 只能影响：

```text
是否把 evidence 升级为 VERIFIED
review priority
```

不能覆盖 reference 冲突。

---

# 28. 外部 Reference Dictionary 的作用

如果能获得可信的品牌 reference catalog，它会非常有价值，但仍不能成为“猜测器”。

建议用途：

```text
1. format validation
2. candidate existence check
3. alias normalization
4. title candidate ranking
5. hard-negative generation
```

例如标题抽出：

```text
126610L?
```

字典中有：

```text
126610LN
126610LV
```

这时正确结果是：

```text
AMBIGUOUS
```

而不是强行挑更热门的一个。

字典让系统更知道“不知道”。

---

# 29. 为什么 catalog-forge 的默认 WDC Watches 结果仍然很有价值

它的 WDC Watches precision 约 0.833 不能直接上线，但这个结果仍说明：

1. 商品领域专用 feature 比纯通用 record linkage 有价值；
2. identifier validation、normalization、attribute roles 能显著提升效果；
3. blocking 的 pair completeness 是下游 recall 上限；
4. classifier 与 blocking 是主要贡献项；
5. cluster cohesion 单看 pairwise F1 很容易被忽略，却直接影响实体安全。

尤其项目的 ablation 中：

```text
identifiers-only blocking
precision 很高
但 recall 极低
```

这正好对应当前需求的权衡：

> 第一阶段可以接受这种行为。

因为当前业务明确允许漏匹配。

所以 MVP 反而应该故意从“identifier/reference-only 高 precision 小覆盖”起步，再逐步用 OCR / extraction / candidate ranking 提高 verified-reference coverage。

而不是一开始就追求常规 ER 的最高 F1。

---

# 30. 建议的 MVP：两周量级工程范围（不依赖大模型）

这里的“阶段”表示工程拆分，不承诺固定工期。

## Phase A：三源字段审计

产出：

```text
source_field_map.yaml
brand_aliases.yaml
reference_field_quality.csv
```

统计：

```text
reference 独立字段覆盖率
标题含 reference 比例
品牌覆盖
编号长度/字符模式
疑似 SKU 比例
同字段内部冲突
```

## Phase B：Brand + Reference Deterministic Pipeline

实现：

```text
brand normalizer
reference evidence model
brand parser
canonicalizer
Reference Gate
exact entity join
conflict log
```

这一阶段就能形成第一个可用系统。

## Phase C：Title Recovery

对缺 reference 记录：

```text
explicit label parser
brand regex parser
reference dictionary validation
```

## Phase D：Review UI + Golden Set

支持：

```text
确认 reference
修正 identifier role
拒绝 alias
查看证据
撤销 membership
```

## Phase E：OCR

只处理高价值 unresolved 图片。

## Phase F：Classifier / LLM

只用于：

```text
候选排序
证据补全
review priority
```

不新增 auto-merge 通道。

---

# 31. Production 增量流程

建议线上每个 batch：

```text
[1] 拉取新增/变更 listing
       ↓
[2] source adapter 标准化 schema
       ↓
[3] brand normalization
       ↓
[4] reference evidence extraction
       ↓
[5] role + grammar validation
       ↓
[6] Reference Gate
    ┌───────────────┬────────────────────┐
    │ VERIFIED      │ AMBIGUOUS/CONFLICT │
    ↓               ↓
[7] exact lookup    [8] review/candidate
    ↓
[9] safe attach/create entity
    ↓
[10] append persistent indexes
    ↓
[11] emit decision log + metrics
```

如果一个旧 listing 发生更新：

```text
old verified ref = R1
new verified ref = R2
```

绝不能静默把 entity 改掉。

必须：

```text
membership -> QUARANTINE
emit REF_CHANGED conflict
review / source correction
```

因为这可能是：

- 源站修正；
- 抓取错位；
- 页面复用；
- parser bug；
- 平台数据污染。

---

# 32. 不要把“同 reference”直接等于“同一在售实物”

Spec 已经定义 same product = same reference，因此系统可以把不同来源同 reference 归为同一商品实体。

但底层仍应该保留：

```text
reference entity
vs
listing / offer
```

不要把：

- 价格；
- 成色；
- 附件；
- 年份；
- 单表状态；
- 店铺库存；

覆盖成一个字段。

这些属于 offer / listing 层。

也就是说：

```text
“同 reference”只合 identity，不合交易事实。
```

这恰好是 catalog-forge `product/variant/offer` 层级思想对本项目最实际的价值之一。

---

# 33. 可以直接从 catalog-forge 借什么，必须改什么

## 可以直接借鉴

### 1. Pipeline Artifact Pattern

逐阶段 Parquet + run metadata。

### 2. Provenance

每个字段/证据保留来源。

### 3. Extraction Cascade

规则优先、模型后置、每层成本可测。

### 4. RuleOutcome

```text
MATCH / NON_MATCH / UNDECIDED
+ reason
```

### 5. Cascade Accounting

知道每层解决多少、花多少钱。

### 6. Persistent Index

增量只处理新 batch，不重建全量 blocking。

### 7. Merge/Split Audit

每个关系可解释、可撤销。

### 8. Review Queue

把人工预算集中在真正高风险边界。

### 9. Evaluation Harness

benchmark / ablation / calibration / cluster metrics。

## 必须重写

### 1. Generic MPN Normalization

腕表 reference 不能要求“必须字母+数字混合”。

### 2. Generic Model Token Regex

不能排除纯数字 reference。

### 3. `same_mpn_and_brand` 0.95 Match

改成：

```text
verified manufacturer reference exact equality
```

并且 hard evidence gate 前置。

### 4. F1-tuned threshold

不能拿 WDC 的 0.24 或常规 0.5 上线。

### 5. Classifier Auto-Merge

模型只能辅助 unresolved，不拥有最终 identity 写权限。

### 6. Connected Component Primary Clustering

主实体必须按 verified reference key 建，而不是 pair graph 传递闭包。

### 7. Generic Product/Variant Semantics

当前 `reference entity` 才是业务“same product”的核心层。

---

# 34. 推荐的 Rule Reason Codes

建议从第一天就用稳定 reason code，而不是只写自然语言：

```text
AUTO_MATCH_EXACT_VERIFIED_REFERENCE
NON_MATCH_DIFFERENT_VERIFIED_REFERENCE
NON_MATCH_DIFFERENT_VERIFIED_BRAND
ABSTAIN_REFERENCE_MISSING
ABSTAIN_REFERENCE_AMBIGUOUS
ABSTAIN_IDENTIFIER_ROLE_UNKNOWN
ABSTAIN_REFERENCE_CONFLICT
ABSTAIN_BRAND_UNVERIFIED
REVIEW_OCR_TEXT_CONFLICT
REVIEW_SOURCE_FIELD_CONFLICT
REVIEW_REFERENCE_CHANGED
REVIEW_MODEL_HIGH_SCORE_HARD_VETO
```

好处：

- 可统计；
- 可回归；
- 可按原因抽样；
- 可监控某版 parser 是否突然制造冲突；
- 可做 release gate。

---

# 35. 线上监控指标

每个 source / brand 都应监控：

```text
reference_verified_rate
reference_missing_rate
reference_conflict_rate
identifier_role_unknown_rate
auto_link_rate
review_rate
hard_non_match_rate
new_reference_rate
existing_reference_attach_rate
reference_changed_rate
ocr_recovery_rate
manual_override_rate
```

另外必须有安全报警：

```text
mixed_verified_reference_entity > 0  => P0
mixed_verified_brand_entity > 0      => P0
AUTO_MATCH without evidence > 0      => P0
```

这三个指标理论上应该由数据库/业务不变量保证为 0，而不是“靠监控发现后再处理”。

---

# 36. Normalizer / Parser 必须版本化

`catalog-forge` 会给 model 和 persistent index 做版本检查，这一点应该扩展到当前的 reference pipeline。

至少版本化：

```text
brand_normalizer_version
reference_normalizer_version
brand_parser_version
reference_dictionary_version
ocr_model_version
llm_prompt_version
rule_version
```

每条 evidence 与 decision 都记录这些版本。

当：

```text
reference_normalizer v7 -> v8
```

上线时，不应全库静默重写。

先做：

```text
shadow replay
collision report
changed-key report
new-conflict report
```

只有确认没有把两个旧 reference 错折叠到同一 key，才发布。

---

# 37. Canonicalization Collision Audit

每次改规范化规则必须运行：

```text
raw distinct refs
    ↓ canonicalizer
canonical key
```

找：

```text
N raw references -> 1 canonical key
```

尤其检查：

```text
同品牌下多个历史 reference 被折叠成一个 key
```

这些 collision 全部要人工看。

因为对于当前业务：

> **错误 canonicalization 比 matcher 阈值过低更危险。**

一旦两个不同 reference 被预处理成同一个字符串，后面 exact match 会非常自信地错，而且模型指标未必能发现。

---

# 38. 模型最适合解决的其实是“抽取与排序”，不是“最终匹配”

结合 catalog-forge 的架构，当前系统中 ML/LLM 最划算的几个位置按优先级是：

```text
1. 标题 reference candidate extraction
2. identifier role classification
3. OCR 结果纠错候选生成
4. unresolved candidate ranking
5. review priority
6. source quality anomaly detection
```

而最不应该交给模型的是：

```text
“两个 verified reference 不一样，但图片/标题很像，要不要还是合？”
```

答案必须永远是：

```text
不合。
```

---

# 39. 如果只能做一个改造：先做 Reference Entity，而不是 Pair Matcher

很多 ER 项目第一反应是：

```text
训练 pair classifier
```

但当前 Spec 已经给出比 pairwise label 更强的结构：

```text
reference number 就是 canonical identity key
```

因此数据建模应该从：

```text
pair(a, b) -> match?
```

改成：

```text
listing -> reference_entity?
```

这样：

- 解释性更强；
- 复杂度更低；
- 更适合增量；
- 更容易 hard veto；
- 更容易人工修正；
- 不会因为传递闭包放大一条错边；
- 与业务定义完全一致。

这是本次从 catalog-forge 提炼出的最重要架构变化。

---

# 40. 最终推荐落地方案

## 40.1 P0：Reference-first Deterministic Core

必须先实现：

```text
三源 adapter
brand canonicalization
reference evidence model
brand-aware reference parser
identifier role
Reference Gate
reference entity store
exact membership
conflict quarantine
decision log
```

此时哪怕只有 30%～50% coverage，只要 precision 足够高，就是符合需求的正确起点。

## 40.2 P1：高精度 Coverage Recovery

增加：

```text
标题 parser
reference dictionary
多证据一致性
```

目标是提高 `verified_reference_rate`，不是改 match threshold。

## 40.3 P2：Image OCR

只对 unresolved 高价值记录跑：

```text
图片用途分类
OCR
brand parser
reference dictionary validation
```

OCR 仍是 evidence，不直接 merge。

## 40.4 P3：Active Learning + Calibrated Ranker

几百对黄金样本用于：

```text
hard negative
role confusion
OCR confusion
source conflict
相邻 reference
```

训练 review ranker / extraction verifier。

## 40.5 P4：弱图审计

把 catalog-forge cohesion / TransClean 类图一致性思想用在：

```text
unresolved candidate graph
alias graph
人工 merge history
```

用于找危险 bridge，不作为自动 identity 来源。

---

# 41. 最后给工程实现者的一组不可破坏约束

可以直接写进 README / tests：

```text
1. No verified reference, no auto-match.
2. Different verified references never share an auto-created entity.
3. Different verified brands never share an auto-created entity.
4. Fuzzy similarity can retrieve, rank, explain, but cannot merge.
5. LLM/VLM output is evidence candidate, never canonical truth by itself.
6. OCR correction never silently mutates a reference key.
7. Every auto-match has machine-readable reason + raw evidence.
8. Every membership can be revoked.
9. Existing reference entities are never auto-merged by a weak/new pair.
10. Normalizer changes require collision audit before release.
11. Missing means unknown, not non-match.
12. Conflict means fail closed, not weighted voting.
```

只要这 12 条长期成立，系统即使逐步增加更复杂的 OCR、embedding、LLM 和 cross-encoder，也不会轻易破坏当前最重要的 precision-first 目标。

---

# 42. 总结

`catalog-forge` 对当前 Spec 最有价值的不是其 0.833 的 WDC Watches precision，也不是某个 classifier，而是它已经把产品 Entity Resolution 中几个真实生产难点拆清楚了：

- identifier 要验证；
- extraction 应 cheap-first cascade；
- blocking 与 decision 必须分离；
- rule 决策要带 reason；
- classifier 概率要校准；
- expensive judge 只处理少数难例；
- transitive closure 会放大 false positive；
- cluster 必须可审计、可拆；
- incremental index 必须持久化；
- 中间 artifact、conflict、review queue 都应该落盘。

但是当前腕表业务有一个比通用 ER 更强的先验：

```text
same product = same reference
```

因此最合理的做法不是把 catalog-forge “调高阈值”，而是把整个系统改成：

```text
Reference-first Identity Resolution
```

最终自动合并只接受：

```text
verified canonical brand
+
verified manufacturer reference exact equality
+
no hard conflict
```

然后把 catalog-forge 中的 blocking、classifier、LLM、图片、图聚类等能力全部降级成：

```text
候选生成 / 证据补全 / 冲突发现 / 人工复核排序
```

这条路线既能直接落地，又与“100 万～1000 万、持续增量、字段稀疏、有图片、只能标几百对、绝对不能误匹配”的约束完全对齐。

---

# 参考代码与资料

- catalog-forge：<https://github.com/alex-hahn/catalog-forge>
- Pipeline：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/pipeline.py>
- Identifier validation：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/identifiers.py>
- Exact / composite blocking：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/blocking/exact.py>
- Extraction cascade：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/extract/cascade.py>
- Match cascade：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/cascade.py>
- Rule matcher：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/rules.py>
- Calibrated classifier：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/classifier.py>
- Cluster cohesion / revoke：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/cluster/components.py>
- Incremental persistent index：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/incremental/index.py>
- Incremental resolve：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/incremental/resolve.py>
- 当前需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
