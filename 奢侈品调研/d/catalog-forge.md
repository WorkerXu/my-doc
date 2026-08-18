# catalog-forge：面向跨源二奢/腕表实体匹配的 Reference-First 改造方案

> 分析人：d  
> 日期：2026-08-18  
> 对应需求：Notion「调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）」  
> 调研对象：[`alex-hahn/catalog-forge`](https://github.com/alex-hahn/catalog-forge)  
> 调研清单来源：`my-doc/奢侈品文章调研.md`

## 1. 结论先行

`catalog-forge` 是这批调研对象里非常适合做**工程骨架**的项目，但不适合原样作为当前需求的最终 matcher。

它最有价值的不是某个模型，而是把产品实体解析拆成一组可审计、可重跑、可增量、可撤销的阶段：

```text
ingest
  -> normalize
  -> taxonomy / attribute extraction
  -> multi-strategy blocking
  -> rules
  -> calibrated classifier
  -> uncertainty judge
  -> clustering + cohesion repair
  -> product/variant/offer hierarchy
  -> golden record
  -> review queue
```

当前 Spec 的定义比通用商品 ER 更特殊：**“同一个商品”就是同一个 manufacturer reference number / 型号**，且“绝对不能误匹配，允许漏匹配”。因此最合理的落地不是复制 catalog-forge 的 `probability >= threshold => match`，而是保留它的工程架构，并把决策核心改成：

```text
brand-scoped canonical reference exact equality
                 +
reference role / provenance / conflict hard gate
                 +
ABSTAIN（证据不足即拒识）
```

推荐最终原则：

1. **Reference 是身份键，不是一个普通相似度特征。**
2. **模型可以帮助“找到/解释 reference”，不能越过 reference 硬规则直接创建 identity edge。**
3. **图片只用于 OCR、reference 取证、冲突否决，不用视觉相似直接判同款。**
4. **任何 reference 冲突都优先产生 NON_MATCH / QUARANTINE，而不是交给模型“综合判断”。**
5. **三源聚类必须在 cluster 内保持 `brand_id + canonical_reference` 单值不变量。**
6. **增量数据只查询持久化 reference index，不需要每次全量重跑。**
7. **每次自动合并都必须保留证据、来源、规则版本和可撤销边。**

如果只做一期，建议先不上 LLM matcher，优先上线“reference evidence extraction + canonicalization + exact gate + review queue”。这个版本最符合 precision-first，而且非常容易解释和回滚。

---

## 2. 为什么选 catalog-forge

本次先检查了 `奢侈品调研/d` 已有分析，已覆盖 ALMSER-GB、Ameli、AnyMatch、ComEM、DeepBlocker、pyJedAI、TransClean、WDC Products、Confidence Classifier、Conformal Selective Prediction 等对象；`catalog-forge` 尚未分析，因此符合“每次先排除已分析文章/项目”的要求。

调研清单对 `catalog-forge` 的推荐理由是：它面向异构商品目录，显式建模 `product -> variant -> offer`，有 identifier validation、blocking、概率校准，并直接给出了 WDC Watches 基准结果。这与本需求的字段稀疏、跨来源、百万到千万级、持续增量、腕表 reference 主导的特点高度接近。

项目 README 给出的 WDC Watches 测试集结果为：

| 指标 | catalog-forge |
|---|---:|
| records | 3,700 |
| precision | 0.833 |
| recall | 0.880 |
| F1 | 0.856 |
| AP | 0.922 |
| F1-optimal threshold | 0.240 |

Blocking 在 WDC Watches 上：

| 指标 | 值 |
|---|---:|
| candidate pairs | 146,425 |
| pair completeness | 0.763 |
| reduction ratio | 0.979 |

这组数字说明两件事：

- 它确实是一个有效的通用商品 ER 工程基线；
- **0.833 precision 完全达不到当前“不可误匹配”的业务约束，因此其 classifier/threshold 不能原样作为自动合并标准。**

反而是项目中“valid identifier、rule first、calibrated probability、abstention 思想、cohesion check、incremental index、audit artifacts”更值得直接复用。

---

## 3. catalog-forge 的技术架构拆解

### 3.1 Pipeline 是“阶段产物驱动”，不是纯内存黑盒

核心文件：

- `src/catalog_forge/pipeline.py`
- `src/catalog_forge/identifiers.py`
- `src/catalog_forge/blocking/*`
- `src/catalog_forge/match/*`
- `src/catalog_forge/cluster/*`
- `src/catalog_forge/incremental/*`

`pipeline.py` 明确把每阶段输出保存成 Parquet：

```text
listings_raw.parquet     原始 ingest 结果
provenance.parquet       字段级 lineage
listings.parquet         标准化记录
candidates.parquet       blocking 候选 + 候选来源策略
extractions.parquet      属性抽取结果
decisions.parquet        pair 决策、概率、stage、reason
clusters.parquet         listing -> cluster
merges.parquet           实际形成 cluster 的边及原因
hierarchy.parquet        listing -> variant -> product
golden.parquet           聚合后的 canonical record
conflicts.parquet        cluster 内字段冲突
review_queue.parquet     人工复核队列
run.json                 配置摘要、阶段耗时、count
```

这个设计非常适合当前项目。对“不能误匹配”的系统来说，最重要的不是在线返回一个 score，而是可以回答：

- 这两个记录为什么合并？
- reference 是从哪个字段/标题/哪张图片抽出来的？
- 哪条规则、哪个版本做的 normalization？
- 当时有没有冲突证据？
- 谁/哪次模型运行把它放行？
- 错了以后能不能只撤销这条边并重算受影响 cluster？

建议直接继承“每阶段物化 + provenance”思想。

---

### 3.2 Identifier validation：先证明“它是什么编号”，再比较字符串

`src/catalog_forge/identifiers.py` 是项目里对当前需求最有价值的代码之一。

它没有把 identifier 当普通字符串，而是建模为：

```python
Identifier(
    kind=GTIN / ISBN / ASIN / MPN,
    value=raw_value,
    key=normalized_comparison_key,
    confidence=...,
    span=...
)
```

关键点：

1. GTIN 必须先过 check digit，合法才允许参与强规则。
2. GTIN-8/12/13/14 统一到 14 位 comparison key。
3. MPN 去掉 separator、统一大小写，但会拒绝太短、纯数字、纯字母等明显异常值。
4. 从自由文本抽 identifier 时，带 `EAN:` / `GTIN:` / `MPN:` 等标签的证据置信度高于裸数字。
5. 缺失与冲突严格区分：
   - `None`：不知道；
   - `0.0`：两边都有合法 identifier，但明确不一致。

这恰好对应腕表系统应该做的事情，只是要把通用 MPN 扩成 `watch_reference`，并且额外加入**编号角色分类**。

二奢数据里最危险的错误不是字符串没归一，而是把这些东西混成一种“型号”：

```text
manufacturer reference / model number
serial number
case number
platform SKU
merchant SKU
listing id
inventory code
certificate number
movement/caliber number
compatible-with model（配件标题里的适配型号）
```

因此我们需要：

```text
raw candidate
 -> role classifier / rule parser
 -> brand-scoped validator
 -> canonicalizer
 -> evidence record
```

而不是：

```text
regex 抽一个像型号的 token -> fuzzy match
```

---

### 3.3 Blocking：多路召回取并集，而不是单一规则

`catalog-forge/blocking` 有：

- exact identifier blocking
- composite key blocking
- MinHash
- Sorted Neighborhood
- ANN

`exact.py` 中，identifier blocking 只负责**生成 candidate**，并明确提醒“exact identifier block 是强候选信号，不等于最终决策”。

对没有 identifier 的记录，项目会使用：

```text
brand + model token
brand + quantity
brand + color + size
title prefix
```

生成候选。

这个“多路 blocking 取 union”的工程思路可以保留，但本需求要把职责进一步拆开：

### 自动合并候选

只需要：

```text
brand_id + canonical_reference
```

这是 O(N) 建索引后近似 O(1) 的 exact lookup，不需要全量 pairwise。

### reference 恢复候选

只有当 reference 缺失时才需要：

```text
brand + reference-like token prefix
brand + series
brand + title model tokens
OCR token
text/image embedding top-k
```

这些 candidate **只能用于帮助恢复 reference / 进入 review**，不能直接形成 match edge。

这样可以从架构上阻断“召回模型越权做 identity decision”。

---

### 3.4 MatchCascade：rules -> calibrated classifier -> judge

`src/catalog_forge/match/cascade.py` 把 matcher 分为三层：

```text
1. Rules
2. Classifier
3. Judge（只处理 uncertainty band）
```

它的价值在于“贵模型只处理前层无法确定的少量 pair”，而且每个 decision 包含：

```text
left_id
right_id
probability
stage
reason
is_match
```

默认 rule layer 位于 `match/rules.py`。典型规则：

```text
different valid GTIN -> NON_MATCH
same valid GTIN -> MATCH
same brand + normalized MPN -> MATCH
product-defining attribute conflict -> NON_MATCH
quantity conflict -> NON_MATCH
```

并且**规则顺序本身是语义的一部分**：hard contradiction 放在 positive rule 前面。

这非常适合改成腕表 reference 规则：

```text
R0: brand 明确冲突                         => NON_MATCH
R1: reference role 不是 manufacturer_ref    => ABSTAIN
R2: 双方 canonical reference 明确冲突        => NON_MATCH
R3: 一方存在多个互相冲突的高可信 reference   => QUARANTINE
R4: brand 一致 + canonical reference 严格相等 => MATCH
R5: 其余                                  => ABSTAIN
```

注意：**这里不应该再有“classifier score 高于阈值 => MATCH”。**

Classifier/Judge 的最佳用途改成：

- 判断某个 title token 是否真的是 reference；
- 对 OCR 多候选做排序；
- 对 brand/series/reference 候选做消歧；
- review queue 排序；
- 预测 `needs_review`；
- 产生解释性辅助信号；
- 但不能绕过 R2/R4。

---

### 3.5 Missingness-aware features：非常适合字段稀疏

`src/catalog_forge/match/features.py` 的一个正确设计是：每个可缺失证据除了 value feature，还带 `*_known`。

例如：

```text
gtin_agreement + gtin_known
mpn_agreement  + mpn_known
brand_agreement + brand_known
model_token_overlap + model_token_known
price_ratio + price_known
```

因为：

```text
没有 reference != reference 不一致
```

当前三源数据高度稀疏，这个编码必须保留。

建议抽取层使用类似结构：

```text
reference_present
reference_role_known
reference_format_valid
reference_in_brand_dictionary
reference_from_structured_field
reference_from_title
reference_from_ocr
reference_multi_source_agreement
reference_conflict
brand_known
series_known
image_ocr_available
```

这些特征可以训练一个 **ReferenceEvidenceScorer**，但最终输出应是：

```text
VERIFIED_REFERENCE
AMBIGUOUS_REFERENCE
NO_REFERENCE
CONFLICTING_REFERENCE
```

而不是 match probability。

---

### 3.6 Calibrated probability：值得保留，但用于“选择性自动化”

`src/catalog_forge/match/classifier.py` 使用 `HistGradientBoostingClassifier`，并用 held-out validation 做 isotonic calibration；样本太少时退化为 sigmoid calibration。

这比直接使用未校准相似度合理，因为概率阈值才有业务意义。

当前需求可以把 calibration 用在两个地方：

#### A. reference extraction confidence

例如：

```text
P(candidate token is manufacturer reference | evidence)
```

只有达到高阈值且没有任何冲突时，才把 candidate 写成 `verified_reference`。

#### B. review prioritization

越靠近决策边界、越可能影响大 cluster、来源越重要的记录优先人工看。

不要把它用于：

```text
P(two listings are same) > 0.999 => auto merge
```

因为当前业务定义已经给出了更强的可验证身份键：reference。既然有硬定义，就没有必要让黑盒模型重定义 identity。

---

### 3.7 Clustering：transitive closure 后做 cohesion repair

`src/catalog_forge/cluster/components.py` 对“错误边传播”处理得很好。

它明确指出：pairwise matcher 一条错误边，如果直接做 connected component/transitive closure，会把两个大 cluster 整体焊死。

项目先做 connected components，再计算：

```text
cohesion = component 内已有边平均权重
edge density = 已有内部边 / 所有可能内部 pair
```

低 cohesion / 低 density 的 component 会用 pivot correlation clustering 重切；并保存 SplitEvent。

对当前 reference-first 系统，还可以更简单、更强：

### Cluster invariant

任一 cluster 必须满足：

```text
unique(brand_id) <= 1
unique(canonical_reference) <= 1
```

只要某条新边导致：

```text
{refA} U {refB}, refA != refB
```

直接拒绝 merge，而不是先合并再修复。

cohesion repair 仍可保留，作为二级安全网；尤其用于“reference 误抽取导致两个错误相同 canonical ref”的极端情况。

---

### 3.8 Incremental：持久索引使成本随 batch 增长，而不是随全库平方增长

`src/catalog_forge/incremental/resolve.py` 的设计非常适合 100 万–1000 万级持续增量。

核心思想：

```text
new batch
 -> persistent index 查询 candidates
 -> 只拼接涉及到的历史 rows
 -> matcher
 -> 只更新受影响 cluster
 -> 新记录加入 index
```

而不是：

```text
每天把全库重新 blocking + pairwise match
```

项目还支持 reviewer reject 一条 edge 后，只重算受影响 cluster，并尽量保持 cluster id 稳定。

当前需求应直接实现为两个持久索引：

```text
ExactReferenceIndex:
  key = brand_id + canonical_reference
  value = entity_id / listing_ids

ReferenceCandidateIndex:
  key = brand_id + token/prefix/series
  value = candidate canonical references
```

第一套服务自动匹配；第二套只服务抽取/人工复核。

---

## 4. catalog-forge 哪些地方不能直接照搬

### 4.1 不能用普通概率阈值定义 identity

项目最终仍允许：

```python
is_match = probability >= threshold
```

这适合一般 ER，不适合“reference 相同才是同一个商品”的本需求。

即使 classifier 校准得很好，`0.999` 也不是“零误匹配”。而且训练分布变化、新品牌、新来源、卖家标题污染都会让概率失真。

**改法：match probability 降级为 review evidence；auto-match 只接受 hard gate。**

---

### 4.2 `normalize_mpn` 太通用，不够懂腕表 reference

项目通用 MPN normalization：

- 去 separator；
- 大写；
- 长度限制；
- 要求字母数字混合。

腕表不能简单套这个逻辑，因为真实 reference 可能：

- 纯数字；
- 带点、斜杠、连字符；
- 尾缀代表材质/表带/盘面；
- 不同品牌格式完全不同；
- 某些 separator 是语义结构，不一定应该全部删除。

因此需要 **brand-scoped canonicalizer**：

```python
canonicalize_reference(brand_id, raw_ref) -> CanonicalReference | Invalid | Ambiguous
```

不同品牌使用不同 parser/version。

---

### 4.3 图片不能只作为 generic multimodal similarity

本需求图片的最大价值是“取证”，不是“看起来像”。

优先级应为：

```text
表背 / 表耳 / 保卡 / 吊牌 / 证书中的 OCR reference
    >
盘面上可读型号信息
    >
视觉外观 embedding
```

外观高度相似恰恰是同系列相邻 reference 的最大 false-positive 风险。

所以图片模块输出应是：

```text
OCR token + bounding box + image_id + confidence + role candidate
```

而不是一个简单 cosine similarity。

---

### 4.4 “同 brand + 同 reference”也要有异常保护

理论上 Spec 把同 reference 定义为同商品，因此 exact equality 就足够。

生产上仍建议加入数据质量保护：

- 同 reference 但 brand 强冲突 -> QUARANTINE；
- reference 是平台 SKU/serial -> 不参与 identity；
- 同 listing 同时出现多个互斥 reference -> QUARANTINE；
- title 中 reference 只出现在“适配/兼容/for xxx”语境 -> 不放行；
- reference 来自 OCR 且低置信/字符易混（O/0、I/1、S/5） -> 需要第二证据；
- canonicalizer 发生可能有损的字符删除 -> 记录 transformation，不直接强合并。

---

## 5. 建议直接落地的目标架构

## 5.1 数据模型

### Listing

```text
listing_id
source_id                    # 雷小安 / 腕表之家 / 奢当家
source_listing_id
brand_raw
brand_id                     # canonical brand
brand_confidence
title
description
structured_reference_raw
images[]
ingested_at
source_updated_at
```

### ReferenceEvidence

```text
evidence_id
listing_id
raw_value
canonical_value
brand_id
role                         # manufacturer_reference / serial / sku / case_no / unknown
role_confidence
source_type                  # structured / title / description / image_ocr / external_dict
source_pointer               # field name / image_id + bbox
extractor                    # regex-v3 / llm-v2 / ocr-v4 ...
extractor_version
format_valid
in_reference_dictionary
confidence
created_at
```

### ListingReferenceState

```text
listing_id
status                       # VERIFIED / AMBIGUOUS / MISSING / CONFLICT
canonical_reference
verification_reason
reference_evidence_ids[]
rule_version
```

### MatchDecision

```text
left_listing_id
right_listing_id
brand_id
left_reference
right_reference
verdict                      # MATCH / NON_MATCH / ABSTAIN / QUARANTINE
reason_code
reason_text
evidence_ids[]
decision_rule_version
created_at
```

### Entity

```text
entity_id
brand_id
canonical_reference
created_at
updated_at
```

这里 entity key 逻辑上就是：

```text
UNIQUE(brand_id, canonical_reference)
```

如果后续业务确认某些品牌 reference 全局唯一，也仍建议保留 brand 维度，避免数据污染跨品牌扩散。

---

## 5.2 Reference 提取流水线

建议 cascade：

```text
Stage A 结构化字段
  -> 最便宜，优先级最高

Stage B 标题/描述 deterministic parser
  -> 品牌词典 + 品牌 regex/parser + label context

Stage C OCR
  -> 只对 reference 缺失/冲突记录处理图片

Stage D restricted candidate retrieval
  -> 在 brand reference KB 中找 top-k 候选

Stage E LLM/VLM judge（可选）
  -> 只回答：哪个 token 是当前售卖商品本身的 manufacturer reference？
  -> 必须允许 NONE / AMBIGUOUS

Stage F cross-evidence verifier
  -> structured / title / OCR / external KB 多证据一致才升 VERIFIED
```

典型输出规则：

```text
结构化 ref 合法 + brand parser 合法
    => VERIFIED（高）

标题 ref 合法 + reference KB 命中 + 无冲突
    => VERIFIED（高）

OCR ref + 标题 ref 一致
    => VERIFIED（高）

单个低置信 OCR ref
    => AMBIGUOUS

标题出现多个不同合法 ref
    => CONFLICT

“适配 Rolex 126610”但商品是表带
    => role = COMPATIBLE_MODEL，不是 manufacturer_reference
```

---

## 5.3 Brand-scoped canonicalization

接口建议：

```python
@dataclass
class CanonicalizationResult:
    brand_id: str
    raw: str
    canonical: str | None
    status: Literal["VALID", "INVALID", "AMBIGUOUS"]
    transformations: list[str]
    parser_version: str
    confidence: float

class ReferenceCanonicalizer(Protocol):
    def canonicalize(self, brand_id: str, raw: str) -> CanonicalizationResult: ...
```

不要一开始做一个“所有品牌万能 regex”。

建议：

```text
common normalization
 -> unicode normalization
 -> trim
 -> case normalization
 -> conservative whitespace normalization

brand parser
 -> RolexParser
 -> OmegaParser
 -> CartierParser
 -> PatekParser
 -> APParser
 -> BreitlingParser
 -> GenericFallbackParser
```

其中 GenericFallback 只允许生成 `AMBIGUOUS` 或低等级 VERIFIED，不能像成熟品牌 parser 一样自动放行。

所有 transformation 都要保留，例如：

```text
raw:  "126610 LN"
canonical: "126610LN"
transformations:
  - trim
  - uppercase
  - remove_semantically_safe_space
```

如果某种变换可能损失语义，不执行自动 equality。

---

## 5.4 Match Gate：核心只需要 tri-state / four-state rules

推荐伪代码：

```python
def decide(a, b):
    if known(a.brand_id) and known(b.brand_id) and a.brand_id != b.brand_id:
        return NON_MATCH("brand_conflict")

    if a.ref_state == CONFLICT or b.ref_state == CONFLICT:
        return QUARANTINE("internal_reference_conflict")

    if a.ref_role != MANUFACTURER_REFERENCE or b.ref_role != MANUFACTURER_REFERENCE:
        return ABSTAIN("reference_role_unverified")

    if not a.canonical_reference or not b.canonical_reference:
        return ABSTAIN("reference_missing")

    if a.canonical_reference != b.canonical_reference:
        return NON_MATCH("reference_conflict")

    if not a.reference_verified or not b.reference_verified:
        return ABSTAIN("reference_not_verified")

    return MATCH("same_verified_brand_scoped_canonical_reference")
```

这段逻辑比一个复杂 matcher 更符合 Spec。

最关键的一点：

```text
ABSTAIN 不是失败，而是系统的正常输出。
```

precision-first 的系统必须把“我不知道”作为一等状态。

---

## 5.5 Candidate/Entity Index

### ExactReferenceIndex

可以从 PostgreSQL 起步：

```sql
CREATE UNIQUE INDEX uq_entity_brand_ref
ON entity (brand_id, canonical_reference);

CREATE INDEX idx_listing_ref_lookup
ON listing_reference_state (canonical_reference)
WHERE status = 'VERIFIED';
```

如果全量 1000 万记录仍然只是这种 exact key lookup，PostgreSQL / MySQL 都够用；没必要为了“千万级”直接上向量数据库。

批量离线可用：

- Polars / DuckDB 做 ETL；
- Parquet 做阶段产物；
- PostgreSQL 做在线 entity/reference index 与审计；
- 对 reference KB / prefix retrieval 再加 Elasticsearch/OpenSearch 或本地 FST/Trie。

### ReferenceCandidateIndex

只用于 reference 恢复：

```text
brand_id
canonical_reference
aliases
series
model_name
format_tokens
```

支持：

```text
prefix
edit distance（只召回）
character ngram
BM25
embedding（可选）
```

任何 fuzzy top-1 结果都不能直接创建 entity match。

---

## 5.6 增量处理

推荐每批：

```text
1. ingest 新/更新 listing
2. 只对变化字段重新生成 reference evidence
3. 计算 ListingReferenceState
4. VERIFIED:
     lookup (brand_id, canonical_reference)
       found -> attach entity
       not found -> create entity
5. AMBIGUOUS / CONFLICT / MISSING:
     enqueue review / enrichment
6. 更新 persistent index
7. 写 audit log
```

复杂度大致是：

```text
O(batch_size * extraction_cost)
+
O(batch_size * index_lookup)
```

而不是 O(total_records²)。

这也是 catalog-forge incremental design 最值得直接借的地方。

---

## 6. 图片/OCR 怎么接入，而不破坏 precision

建议图片只做 `ReferenceEvidenceProvider`。

### 图片选择优先级

```text
保卡 / 证书 / 吊牌
> 表背 / 表壳 / 表耳刻字
> 盒标
> 普通商品主图
```

### OCR 后处理

不要直接把 OCR string 交给 fuzzy matcher。

流程：

```text
OCR token
 -> confusion-aware normalization（O/0, I/1 等只生成候选，不直接替换）
 -> brand parser
 -> reference KB candidate retrieval
 -> 多图/文本交叉验证
```

例如 OCR 得到：

```text
12661OLN
```

不能直接 edit-distance 成 `126610LN` 后自动匹配。

只能：

```text
candidate = 126610LN
reason = OCR one-char ambiguity
status = AMBIGUOUS
```

如果标题也明确出现 `126610LN`，才可升级成 VERIFIED。

---

## 7. 人工标注的最佳用法

Spec 允许人工标注几百对。不要随机标 300 对；随机样本大部分是简单 negative，价值很低。

建议标三类：

### A. Reference role hard cases

```text
manufacturer ref vs serial
manufacturer ref vs merchant SKU
商品本体型号 vs compatible-with 型号
机芯号 vs 表款 reference
```

### B. Canonicalization hard cases

```text
separator 差异
suffix 差异
OCR 字符混淆
品牌特殊格式
同系列只差 1 位字符
```

### C. Cross-source conflict cases

```text
两个来源标题一致但 reference 不同
图片几乎一致但 reference 不同
结构化字段和标题 reference 冲突
结构化字段和 OCR 冲突
```

标注数据优先训练：

```text
ReferenceRoleClassifier
ReferenceEvidenceScorer
ReviewPriorityModel
```

而不是训练一个端到端 `same/not-same` 模型替换硬规则。

---

## 8. 验收指标：不要只看 F1

当前需求下 F1 不是主 KPI。

建议指标分层：

### Reference extraction

```text
verified_reference_precision
verified_reference_coverage
reference_conflict_rate
role_classification_precision
```

### Auto-match

```text
auto_match_precision                # 第一 KPI
auto_match_coverage
auto_match_false_positive_count
hard_negative_false_positive_count
```

### Abstention / Review

```text
abstain_rate
review_yield
review_queue_size
median_review_latency
```

### Cluster safety

```text
cluster_reference_invariant_violations = 0
cluster_brand_invariant_violations = 0
cross-reference merge count = 0
largest cluster by reference
```

### 增量稳定性

```text
entity_id_stability
records_reprocessed_per_batch
rollback_count
```

特别要注意：几百条黄金标签只能帮助校准和发现边界，**不能数学上证明“零误匹配”**。如果要声称 99.9%+ precision，需要专门构造大量 hard-negative 验收集，并持续在线抽检。

上线 gate 建议至少要求：

```text
- gold/hard-negative 集上 0 false positive
- 所有 brand/ref invariant violation = 0
- 新品牌 parser 未通过验收前默认不 auto-match
- 新数据源先 shadow mode
```

---

## 9. 推荐的服务拆分

不建议一期就拆很多微服务。可以先做一个 repo 内模块化 monolith：

```text
src/
  ingest/
  brand/
  reference/
    evidence.py
    role_classifier.py
    canonicalize/
      base.py
      rolex.py
      omega.py
      cartier.py
      ...
    validators.py
    kb.py
  image/
    ocr.py
  matching/
    gate.py
    index.py
    decisions.py
  clustering/
    invariant.py
    repair.py
  review/
    queue.py
  audit/
    provenance.py
```

等吞吐或组织边界真的需要时再拆成服务。

数据层：

```text
Object storage / Parquet  -> raw + stage artifacts
PostgreSQL                -> listings / entities / evidence / decisions / review
Optional OpenSearch       -> reference candidate retrieval
Optional Redis            -> hot exact reference cache
```

---

## 10. 一期最小可落地版本（建议）

### Phase 0：数据 profiling

输出：

```text
三源 reference 字段覆盖率
reference 在 title 中的覆盖率
brand 覆盖率
同 listing 多 reference 比例
疑似 SKU/serial 混入比例
每品牌 reference 形态分布
```

### Phase 1：Reference-first deterministic baseline

实现：

```text
canonical brand
structured/title reference extractor
brand-scoped canonicalizer v1
role rules
exact entity index
MATCH / NON_MATCH / ABSTAIN / QUARANTINE
provenance + audit
```

这一步不需要 LLM。

### Phase 2：OCR enrichment

只处理 Phase 1 的 MISSING/AMBIGUOUS。

加入：

```text
图片优先级
OCR
brand reference KB
cross-evidence verifier
```

### Phase 3：少量标注 + calibrated evidence model

训练：

```text
reference role
reference evidence confidence
review priority
```

模型只影响 VERIFIED/AMBIGUOUS 的证据判断，不改变 exact identity 定义。

### Phase 4：增量与回滚

实现 catalog-forge 风格：

```text
persistent index
stable entity id
edge/decision audit
review reject -> local recompute
shadow mode for new source/brand
```

---

## 11. 可以直接从 catalog-forge 借的实现思想

| catalog-forge 设计 | 当前项目怎么用 |
|---|---|
| identifier checksum/validation before rules | reference 先做 role + brand format validation |
| identifier normalization key | brand-scoped canonical reference |
| exact identifier blocking | `brand_id + canonical_reference` exact index |
| multi-strategy blocking union | 只用于 reference 恢复和 review candidate |
| missingness-aware features | reference evidence scorer |
| rule-first cascade | hard contradiction / exact reference gate |
| calibrated classifier | extraction confidence / review priority |
| uncertainty band | AMBIGUOUS -> 人工，而不是强判 |
| human-readable reason | 每次 merge 保存 reason/evidence |
| Parquet stage artifacts | 全链路可重跑、审计 |
| cohesion check | cluster reference invariant + secondary repair |
| revocable edges | 人工拒绝后局部重算 |
| persistent incremental index | 新 batch 只查历史 reference/entity index |
| stable cluster ids | entity id 稳定，不因重跑全量变更 |

---

## 12. 不建议做的方案

### 不建议 1：直接 title embedding / image embedding top-1 判同款

原因：同系列相邻 reference 是最危险 hard negative，视觉和语义往往极像。

### 不建议 2：让 LLM 直接回答“是不是同一商品”并自动合并

原因：LLM 很难保证极端 precision，且 reference 已经提供了可验证定义。

### 不建议 3：全库 pairwise

1000 万记录平方级比较没有必要；exact reference index 才是主通路。

### 不建议 4：统一字符串 normalize 后直接 equality

如果没有先做 identifier role，平台 SKU / serial 也会被“规范化得很漂亮”，随后制造灾难性误合并。

### 不建议 5：只看 pairwise F1

一条错边通过传递闭包可能污染整个 cluster。必须监控 cluster invariant 与 merge audit。

---

## 13. 最终推荐决策

如果让我直接给这个 Spec 落一版，我会以 catalog-forge 为工程参考，但把核心 matcher 改写为下面这条链路：

```text
三源 raw listing
  -> canonical brand
  -> reference evidence extraction
       structured
       title/description
       OCR（按需）
  -> identifier role classification
  -> brand-scoped reference validation/canonicalization
  -> ListingReferenceState
       VERIFIED / AMBIGUOUS / MISSING / CONFLICT
  -> ExactReferenceIndex lookup
  -> hard MatchGate
       MATCH only when verified brand + canonical reference exact equal
       reference conflict => NON_MATCH
       insufficient evidence => ABSTAIN
       internal conflict => QUARANTINE
  -> entity assignment
  -> cluster invariant check
  -> audit + review queue
  -> persistent incremental index
```

一句话概括：

> **借 catalog-forge 的工程体系，不借它的“概率阈值决定 identity”；把 reference 从 feature 提升为受验证、可追溯、品牌作用域的唯一身份键，让模型只帮助恢复 reference，并把拒识设计成正常路径。**

这是目前最符合“100万–1000万级、持续增量、字段稀疏、有图片、可标少量黄金数据、precision 优先到极致”约束的实现方向。

---

## 14. 主要代码/资料入口

- 项目：<https://github.com/alex-hahn/catalog-forge>
- README / benchmarks：<https://github.com/alex-hahn/catalog-forge/blob/main/README.md>
- Pipeline：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/pipeline.py>
- Identifier validation：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/identifiers.py>
- Exact blocking：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/blocking/exact.py>
- Match cascade：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/cascade.py>
- Rules：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/rules.py>
- Features：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/features.py>
- Calibrated classifier：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/match/classifier.py>
- Clustering/cohesion repair：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/cluster/components.py>
- Incremental resolve：<https://github.com/alex-hahn/catalog-forge/blob/main/src/catalog_forge/incremental/resolve.py>
