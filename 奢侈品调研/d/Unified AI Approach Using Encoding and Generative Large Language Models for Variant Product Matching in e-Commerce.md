# Unified AI Approach Using Encoding and Generative Large Language Models for Variant Product Matching in e-Commerce

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **Unified AI Approach Using Encoding and Generative Large Language Models for Variant Product Matching in e-Commerce**（VARM）进行深入分析。

公开技术材料：

- 2026 Big Data 正式文章：<https://doi.org/10.1177/2167647X261423127>
- 2024 arXiv 技术版本：<https://arxiv.org/abs/2410.02779>
- Amazon Science 页面：<https://www.amazon.science/publications/learning-variant-product-relationship-and-variation-attributes-from-e-commerce-website-structures>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

VARM 研究的不是传统“两个商品是不是同一个实体”，而是一个对腕表尤其重要的相邻问题：

> **两个商品是否属于同一个 variant family，但并不是完全相同的商品；如果是，它们到底在哪些属性上发生了变化。**

这恰好击中了当前需求最大的误匹配来源：

```text
同品牌
  + 同系列
  + 标题高度相似
  + 图片高度相似
  + reference 只差一两个字符 / 后缀
  = 极容易被普通相似度模型误判为同一个商品
```

例如：

```text
Rolex 126610LN  vs  Rolex 126610LV
Omega 310.30.42.50.01.001 vs 310.30.42.50.01.002
Patek 5711/1A-010 vs 5711/1A-011
```

它们对普通文本/图片 matcher 来说往往是“最像的正样本”，但按当前 Spec 的定义，只要 reference 不同，就必须是 **负样本**。

因此，我建议不要把 VARM 原样用于“自动合并”，而是反向利用它的两个最有价值思想：

1. **Informed hard-negative sampling**：专门构造“同品牌、同品类、同系列、近邻 reference”的困难负例，让模型学会区分最危险的相似变体；
2. **Variation attribute discovery**：用生成式 LLM + RAG 离线发现每个品牌/系列常见的变化维度，形成 `variant_rules`，在实时匹配时只做冲突检测。

落地后，建议将整个系统改造成：

```text
原始商品
  -> reference 候选抽取
  -> reference 角色识别
  -> 品牌感知规范化
  -> canonical reference
  -> exact reference lookup
  -> VARM-inspired Variant Guard
  -> 冲突审计
  -> VERIFIED / REVIEW / ABSTAIN
```

最终自动放行原则：

> **只有“品牌确定 + canonical reference 严格一致 + reference 证据可信 + 无强冲突”才能自动合并。**
>
> VARM/LLM/图片/embedding 永远不能单独把两个不同 reference 的商品提升为同一实体；它们只能帮助发现 reference、生成候选、发现冲突或决定拒识。

这是把 VARM 真正迁移到“precision 极端优先、允许漏匹配”的二奢系统时最关键的改造。

---

## 1. 为什么这篇文章特别适合当前 Spec

当前需求有几个决定架构的约束：

- 数据来自雷小安、腕表之家、奢当家三个来源；
- 总量 100 万–1000 万级，持续增量；
- reference 有时是结构化字段，有时藏在标题、描述甚至图片；
- 字段稀疏；
- 有图片；
- 同一商品的业务定义就是 **同一 reference number / 型号**；
- **绝不能误匹配**，precision 优先于 recall；
- 可以人工标注几百对黄金样本。

传统 entity matching 很容易自然地学习：

```text
标题相似 + 品牌相同 + 图片相似 + 价格相近 -> Match
```

但腕表数据里最危险的恰恰是“同系列不同 reference”。

它们通常：

- 标题主体一样；
- 系列一样；
- 机芯、尺寸、年代可能一样；
- 外观几乎一样；
- 图片 embedding 余弦相似度极高；
- 价格区间也可能重叠；
- 唯一真正有区分力的就是 reference 的少数字符或后缀。

VARM 的问题定义正好提醒我们：

> **“高度相关 / 同 variant family”和“同一实体”必须在数据模型和模型标签中明确分开。**

如果系统只有 `MATCH / NON_MATCH` 两类，模型会被迫把“同系列不同 reference”塞进普通负类；这些样本数量通常又远少于随机负样本，于是决策边界会被大量容易样本主导。

我建议直接把业务标签提升成：

```text
SAME_REFERENCE
SAME_FAMILY_DIFFERENT_REFERENCE
UNRELATED
UNKNOWN
```

其中线上自动合并只认可：

```text
SAME_REFERENCE
```

`SAME_FAMILY_DIFFERENT_REFERENCE` 是最高优先级的 **危险负类**。

---

## 2. VARM 原论文的核心技术实现

### 2.1 两段式架构

VARM 将任务拆成两个不同模型负责的部分：

```text
                 Webpage-linked variant groups
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
  正样本 + 负样本构造            Variation group attributes
          │                             │
          ▼                             ▼
 Encoding LLM                   Generative LLM + RAG
 variant matcher                variation/common attributes
          │                             │
          ▼                             ▼
 variant relation               variation dimensions
```

它没有让一个大模型包办所有任务，而是做了一个很工程化的分工：

- **Encoder LLM**：负责高吞吐二分类；
- **Generative LLM**：负责开放式属性理解；
- **RAG**：把相似 product type / brand 的历史 variation attributes 注入 prompt。

这个拆分非常值得保留。

### 2.2 Encoder：DistilBERT pair classification

论文用 DistilBERT 做 variant pair classification。

输入不是两个独立 embedding 后算 cosine，而是把两条商品属性序列拼接成一个序列，做 **early fusion / cross-encoder**：

```text
[product A attributes] [SEP] [product B attributes]
```

输入最长 512 token，并尽量保证两边各占一半 token。

然后：

```text
DistilBERT
  -> pooled representation
  -> linear classification head
  -> cross entropy
  -> variant match / mismatch
```

论文还比较了：

- 通用 DistilBERT；
- 先经过电商领域任务预训练的 DistilBERT；
- 随机负样本；
- informed negative samples；
- zero-shot generative model。

最重要的结论之一是：

> **电商领域预训练 + informed negative sampling 的组合明显优于随机负样本。**

论文进一步观察到，合理构造困难负样本后，可以用少很多训练样本达到随机负采样才接近的效果。

### 2.3 Informed negative sampling

这是本文对当前需求最有价值的技术点。

论文从真实网页中的 variant group 获得正关系，然后按难度构造负样本：

```text
Hard negative:
  同 brand + 同 product type，但不属于同一个 variant group

Medium negative:
  同 product type，不同 brand

Easy negative:
  不同 product type + 不同 brand
```

其本质不是“随机造负样本”，而是：

> **用业务结构制造接近真实决策边界的负样本。**

对于腕表，应做得更极端：

```text
Hard++ negative:
  同 brand
  + 同 collection/series
  + reference 编辑距离很小
  + 标题 token overlap 很高
  + 图片 embedding 很相近
  + 但 canonical reference 不同
```

这种 hard++ 才是当前系统真正要学会拒绝的东西。

### 2.4 Generative LLM + RAG 做 variation attribute discovery

VARM 的第二部分不是训练固定标签分类器，而是给一个 variant group，让生成式模型回答：

```json
{
  "Different": ["..."],
  "Same": ["..."],
  "Reason": ["..."]
}
```

论文使用 Claude 3 Haiku，并把 product type / brand 的历史 variation attributes 作为 RAG context。

例如，不同商品类别的变化维度完全不同：

```text
服装：size / color / length
电视：screen size
键盘：switch / layout / model
珠宝：size / metal / gem type
```

这对腕表也非常成立：

```text
Rolex Submariner:
  bezel color / material / date / bracelet / dial / generation

Omega Speedmaster:
  dial / bracelet / case material / movement / limited edition

Cartier:
  case size / metal / dial / strap / gem setting
```

当前系统不应该把这些全部人工硬编码到统一 schema，而可以借鉴 VARM：

> 离线用 RAG + LLM 从已知 reference family 中总结“这个品牌/系列通常在哪些字段上发生 variant”，再固化成规则与特征。

### 2.5 论文数据规模与结果给我们的现实提醒

论文技术版本使用约：

- 2M product pairs；
- 168K variation groups；
- 70/30 train/eval；
- 同一 variation group 强制放在同一 split，减少信息泄漏；
- 另有 470 对跨站点专家审核 pair；
- variation attribute 也有人工审核集。

其最佳电商预训练 + informed negative 的 encoder 报告 precision 约 **98.39%**、recall 约 **90.68%**。

这个结果对普通商品任务很好，但对当前需求反而说明：

> **哪怕优秀的 variant matcher，也不应该直接拥有自动合并权限。**

1000 万条数据下，哪怕误匹配率只有千分之一，也可能造成大量错误实体。

所以迁移 VARM 时必须从：

```text
model as matcher
```

改成：

```text
model as guard / challenger / abstention signal
```

---

## 3. 对当前业务最关键的架构重构：Reference Entity，而不是 Pair Entity

我建议业务核心实体不要直接是 `ProductCluster`，而是先建立一个稳定的 **Reference Entity Registry**。

### 3.1 核心对象

```text
SourceProduct
ReferenceObservation
CanonicalReference
ReferenceEntity
MatchDecision
ConflictEvidence
```

关系：

```text
SourceProduct
   │ 1:N
   ▼
ReferenceObservation
   │
   ▼
CanonicalReference
   │ N:1
   ▼
ReferenceEntity
   │
   └── 雷小安 record(s)
   └── 腕表之家 record(s)
   └── 奢当家 record(s)
```

注意：

```text
ReferenceEntity key != 只有 reference 字符串
```

至少应为：

```text
(brand_id, canonical_reference)
```

原因是不同品牌可能出现类似甚至完全一样的数字型号。

品牌无法高置信确定时：

```text
不允许自动进入 ReferenceEntity
```

### 3.2 ReferenceObservation 必须保留证据链

不要只保存：

```text
reference = 126610LN
```

而应保存：

```json
{
  "raw_value": "126610 LN",
  "canonical_value": "126610LN",
  "role": "MANUFACTURER_REFERENCE",
  "source_field": "title",
  "extractor": "watch_ref_ner_v3",
  "extractor_score": 0.997,
  "span_start": 18,
  "span_end": 27,
  "evidence_type": "TEXT_SPAN",
  "image_id": null,
  "brand_id": "rolex"
}
```

如果从图片 OCR 得到：

```json
{
  "raw_value": "126610LN",
  "canonical_value": "126610LN",
  "role": "MANUFACTURER_REFERENCE",
  "source_field": "image_ocr",
  "extractor": "ocr_ref_v2",
  "image_id": "...",
  "bbox": [x1, y1, x2, y2]
}
```

这样误匹配发生时能完整回溯。

---

## 4. 直接可落地的整体系统

## 4.1 数据流

```text
              ┌────────────────────────────┐
              │ 雷小安 / 腕表之家 / 奢当家 │
              └─────────────┬──────────────┘
                            ▼
                   Raw Product Ingestion
                            │
                  append-only raw snapshot
                            ▼
                Source Normalization Layer
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       brand            text/title          images
     resolver           normalization        OCR
          └─────────────────┼─────────────────┘
                            ▼
                Reference Candidate Extractor
                            │
                            ▼
                Number Role / Type Classifier
                            │
              platform SKU ? ref ? serial ?
                            ▼
              Brand-aware Canonicalizer
                            │
                            ▼
                    ReferenceObservation
                            │
                 ┌──────────┴───────────┐
                 ▼                      ▼
          exact registry lookup     no safe ref
                 │                      │
                 ▼                      ▼
           Variant Guard             REVIEW /
                 │                   ABSTAIN
                 ▼
          Conflict Gate
                 │
          ┌──────┴──────┐
          ▼             ▼
      VERIFIED       ABSTAIN
          │
          ▼
     ReferenceEntity
```

## 4.2 为什么不建议“先 embedding 全库 ANN，再逐对分类”

当前业务定义已经给了最强主键语义：reference。

如果先对 1000 万商品做 embedding ANN，然后让模型做 pair matching，会有三个问题：

1. 计算量大；
2. 最危险的 near-variant 一定会被高频召回；
3. 模型会有机会把“非常像但 reference 不同”的商品错误放行。

正确设计应该是：

```text
reference exact index 是主路径；
embedding 只服务于 reference 缺失时的候选发现 / 人工复核。
```

因此大多数已有安全 reference 的记录，其复杂度可以近似：

```text
O(1) KV/index lookup
```

而不是 O(N²) pairwise comparison。

---

## 5. Reference 抽取层：先找号码，再判断“它是什么号码”

这是 precision 的第一道闸。

二奢标题里常同时出现：

```text
品牌 reference
平台商品号
店铺 SKU
拍卖 lot
序列号
年份
尺寸
机芯号
配件兼容型号
```

因此 reference pipeline 应拆成两个模型：

```text
Candidate Extraction
    ↓
Role Classification
```

而不是一个正则一把梭。

### 5.1 Candidate Extraction

高 recall 抽取所有可能的字母数字串：

```text
126610LN
116500LN
5711/1A-010
IW371605
Q1368420
310.30.42.50.01.001
```

来源可包括：

- 独立 reference/model 字段；
- title；
- description；
- OCR；
- 结构化参数表。

### 5.2 Role Classification

将每个候选归类为：

```text
MANUFACTURER_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
MOVEMENT_NUMBER
LOT_NUMBER
ACCESSORY_COMPATIBLE_REFERENCE
UNKNOWN_IDENTIFIER
```

只有 `MANUFACTURER_REFERENCE` 可以进入自动匹配主路径。

### 5.3 Evidence tier

建议把 reference 证据分层：

```text
Tier A
  官方/结构化 reference 字段

Tier B
  标题中品牌语法匹配 + 高置信 role classifier

Tier C
  参数表/描述文本抽取

Tier D
  图片 OCR

Tier E
  纯 LLM 推断、没有原文 span
```

自动合并要求至少一侧满足高可信证据，具体规则可以品牌级配置。

`Tier E` 不允许单独自动放行。

---

## 6. Brand-aware Canonicalizer：规范化必须保守

不要做激进 fuzzy normalization。

安全操作：

```text
大小写统一
Unicode 统一
去除明确无语义空格
统一全角/半角
统一品牌规则确认无语义的分隔符
```

危险操作：

```text
自动删除所有 - / .
把 O 全部替换为 0
把 I 全部替换为 1
自动补零
自动截断后缀
只保留数字部分
```

例如：

```text
5711/1A-010
5711/1A-011
```

最后三位差异就是实体边界。

建议 canonicalizer 输出：

```json
{
  "canonical": "5711/1A-010",
  "normalization_ops": [
    "unicode_nfkc",
    "uppercase",
    "trim_spaces"
  ],
  "lossy": false
}
```

任何 `lossy=true` 的规范化结果：

```text
禁止自动合并
```

---

## 7. VARM-inspired Hard Negative Factory

这是本篇最值得直接开发的模块。

### 7.1 黄金正样本

只用高可信 reference 构造正样本：

```text
同 brand
+ canonical reference 完全一致
+ 两侧 reference observation 均达到安全证据等级
```

### 7.2 Hard++ 负样本

按优先级采样：

#### H1：reference 近邻

```text
brand 相同
reference 不同
编辑距离 <= 1/2
```

或：

```text
共享长前缀，只在尾缀/变体码不同
```

#### H2：同 series / collection

```text
Rolex Submariner 不同 reference
AP Royal Oak 不同 reference
Omega Speedmaster 不同 reference
```

#### H3：标题高度相似

```text
Jaccard / BM25 / cross-encoder similarity 高
但 reference 不同
```

#### H4：图片高度相似

```text
CLIP/DINO image similarity 高
但 reference 不同
```

#### H5：真实历史误报

所有人工发现过的 false positive 都应进入：

```text
permanent hard-negative bank
```

### 7.3 Medium / Easy

保留 VARM 原论文思路：

```text
Medium:
  同品类不同品牌

Easy:
  不同品类不同品牌
```

但训练 batch 建议显著提高 H1-H4 占比，防止简单负样本淹没真正边界。

### 7.4 防止 train/test leakage

不能随机按 row 拆 pair。

必须按实体/家族拆：

```text
同 canonical reference 的所有记录 -> 同 split
同一高相关 reference family -> 尽量同 split
```

否则：

```text
A-B 在 train
A-C 在 test
```

会严重高估泛化能力。

---

## 8. Variant Guard：模型只负责“挑战”，不负责“批准”

建议训练一个轻量 cross-encoder，标签为：

```text
SAME_REFERENCE
SAME_FAMILY_DIFFERENT_REFERENCE
UNRELATED
```

输入字段建议：

```text
brand
canonical reference
raw reference
series
title
structured model fields
movement
case size
material
year
OCR snippets
```

但线上使用时需要非常明确的权限设计。

### 8.1 自动合并条件

```python
if not brand_resolved:
    return ABSTAIN

if not safe_reference_a or not safe_reference_b:
    return ABSTAIN

if ref_a != ref_b:
    return NOT_SAME_REFERENCE

if has_reference_role_conflict(a) or has_reference_role_conflict(b):
    return ABSTAIN

if variant_guard.detects_strong_conflict(a, b):
    return ABSTAIN

return VERIFIED_SAME_REFERENCE
```

关键点：

```text
ref_a != ref_b
```

时，无论模型多么相信相似：

```text
都不能 MATCH
```

### 8.2 模型最适合发现什么冲突

例如 exact canonical ref 一样，但其他字段出现极强矛盾：

```text
品牌冲突
明显不同 series
完全不同 category
一个是手表，一个是表带
标题声明“适配 126610LN”，实际售卖的是表带
reference role classifier 一侧判为 COMPATIBLE_REFERENCE
```

这时 Variant Guard 的职责是：

```text
veto / abstain
```

而不是强行找另一个实体。

---

## 9. 把 VARM 的 RAG 属性发现改成“离线 Variant Rule Builder”

不建议实时对每一对商品调用大模型。

更好的方式：

```text
离线任务
  -> 按 brand / collection 收集已知 reference group
  -> 找相邻 reference
  -> 汇总结构化属性差异
  -> LLM + RAG 总结 variation dimensions
  -> 人工审核
  -> 发布成 versioned rules
```

生成结果例如：

```yaml
brand: Rolex
collection: Submariner
version: 7
variation_dimensions:
  - date
  - bezel_color
  - bezel_material
  - dial_color
  - bracelet
  - case_material
  - generation
reference_sensitive_fields:
  - suffix
  - final_2_chars
forbidden_automerge_if:
  - accessory_only == true
  - reference_role != MANUFACTURER_REFERENCE
```

线上只加载确定性 rule，不依赖 LLM 实时输出。

好处：

- 成本低；
- 可版本化；
- 可解释；
- 可回滚；
- 适合 1000 万级持续增量；
- LLM 幻觉不会直接污染实体关系。

---

## 10. 图片应该怎么用

图片是辅助证据，不是同实体主键。

推荐三个用途：

### 10.1 OCR reference

重点 OCR：

```text
表背
保卡
吊牌
证书
盒标
发票/鉴定卡
```

OCR reference 与文本 reference 一致：

```text
提高 observation 可信度
```

OCR 与文本冲突：

```text
ABSTAIN / REVIEW
```

### 10.2 Near-variant hard-negative mining

用 image embedding 找：

```text
图片非常像
+ reference 不同
```

的样本。

这些样本非常适合训练 Variant Guard。

### 10.3 Conflict detector

例如：

```text
标题说腕表
图片主体却是表带/盒子
```

模型可以用来否决。

### 明确禁止

```text
image similarity 高 -> 自动认定 same reference
```

绝对不能这样做。

---

## 11. 数据表设计

### 11.1 source_product

```sql
CREATE TABLE source_product (
    source              TEXT NOT NULL,
    source_product_id   TEXT NOT NULL,
    raw_payload         JSONB NOT NULL,
    title               TEXT,
    brand_raw           TEXT,
    brand_id            TEXT,
    updated_at          TIMESTAMPTZ NOT NULL,
    payload_hash        TEXT NOT NULL,
    PRIMARY KEY (source, source_product_id)
);
```

### 11.2 reference_observation

```sql
CREATE TABLE reference_observation (
    observation_id      BIGSERIAL PRIMARY KEY,
    source              TEXT NOT NULL,
    source_product_id   TEXT NOT NULL,
    raw_value            TEXT NOT NULL,
    canonical_value      TEXT,
    brand_id             TEXT,
    role                  TEXT NOT NULL,
    evidence_type         TEXT NOT NULL,
    source_field          TEXT,
    extractor_version     TEXT NOT NULL,
    extractor_score       DOUBLE PRECISION,
    lossy_normalization   BOOLEAN NOT NULL DEFAULT FALSE,
    evidence_payload      JSONB,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ref_lookup
ON reference_observation (brand_id, canonical_value)
WHERE role = 'MANUFACTURER_REFERENCE'
  AND lossy_normalization = FALSE;
```

### 11.3 reference_entity

```sql
CREATE TABLE reference_entity (
    reference_entity_id  BIGSERIAL PRIMARY KEY,
    brand_id             TEXT NOT NULL,
    canonical_reference  TEXT NOT NULL,
    status                TEXT NOT NULL,
    rule_version          TEXT,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, canonical_reference)
);
```

### 11.4 entity_membership

```sql
CREATE TABLE entity_membership (
    reference_entity_id  BIGINT NOT NULL,
    source               TEXT NOT NULL,
    source_product_id    TEXT NOT NULL,
    decision_status      TEXT NOT NULL,
    decision_version     TEXT NOT NULL,
    confidence_tier      TEXT NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (reference_entity_id, source, source_product_id)
);
```

### 11.5 conflict_evidence

```sql
CREATE TABLE conflict_evidence (
    conflict_id          BIGSERIAL PRIMARY KEY,
    source               TEXT NOT NULL,
    source_product_id    TEXT NOT NULL,
    reference_entity_id  BIGINT,
    conflict_type        TEXT NOT NULL,
    severity             TEXT NOT NULL,
    model_version        TEXT,
    details              JSONB NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 12. 服务拆分

推荐保持模块边界清晰：

```text
ingestion-service
brand-resolver
reference-extractor
identifier-role-classifier
reference-canonicalizer
reference-registry
variant-guard
conflict-gate
review-api
model-training-pipeline
variant-rule-builder
```

### 12.1 reference-registry

这是核心在线服务。

接口：

```http
GET /references/{brand}/{canonical_ref}
POST /reference-observations
POST /resolve-product
```

核心 lookup：

```text
(brand_id, canonical_reference) -> ReferenceEntity
```

1000 万级数据并不需要复杂向量检索才能解决主路径。

### 12.2 variant-guard

部署轻量 encoder：

```text
DistilBERT / MiniLM / small cross-encoder
ONNX / TensorRT / Triton
```

只对：

- reference 抽取存在一定不确定性；
- exact ref 虽相同但有强字段冲突；
- REVIEW 队列；
- hard-negative mining；

进行调用。

不要让所有 exact-safe record 都过模型，以免模型反而成为新故障点。

### 12.3 generative model

只用于：

```text
离线 variant rule discovery
错误案例解释
人工审核辅助
新品牌/新系列 cold start 分析
```

不要处于自动合并同步链路。

---

## 13. 增量更新架构

当前有持续抓取，因此每条商品更新都应是幂等事件。

```text
ProductUpserted
  source
  source_product_id
  payload_hash
  observed_at
```

处理：

```text
1. payload_hash 没变化 -> skip
2. 有变化 -> 重跑 brand/reference extraction
3. observation 未变化 -> membership 保持
4. canonical reference 变化 -> 撤销旧 membership，重新 resolve
5. rule/model version 变化 -> 可异步 re-evaluate 高风险记录
```

尤其要避免：

```text
商品标题后来被卖家修改
但旧 reference membership 永久保留
```

每个自动决策必须存：

```text
extractor_version
canonicalizer_version
rule_version
model_version
```

这样新规则上线后能精确重算。

---

## 14. 100万–1000万规模下的存储/计算建议

不需要一开始就做超重型架构。

建议：

```text
Raw / history:
  Object Storage + Parquet/Iceberg

Online registry / decision:
  PostgreSQL（起步足够）
  按 brand/hash 分区 + 组合索引

Event:
  Kafka / Pulsar

Streaming:
  Flink / Kafka Streams

Offline feature & hard-negative mining:
  Spark / Ray

Search / review:
  OpenSearch

Model serving:
  Triton / ONNX Runtime
```

如果后续在线写入/lookup QPS 极高，`Reference Registry` 可以迁移到 DynamoDB/ScyllaDB/FoundationDB 一类 KV，但逻辑 key 不变：

```text
brand_id + canonical_reference
```

---

## 15. 人工标注几百对应该怎么花

不要随机采几百 pair。

随机样本大部分太容易，对 precision 边界帮助很小。

应该把人工预算集中到：

```text
40% reference 近邻 hard negatives
20% 同系列、图像极近但 reference 不同
15% 配件/表带/盒证标题带兼容 reference
10% OCR/text reference 冲突
10% 新品牌/新格式 reference
5% 随机 sanity sample
```

并单独建立：

```text
Precision Challenge Set
```

其中必须大量包含：

```text
一字符差异
后缀差异
分隔符差异
容易被 aggressive normalization 合并的 pair
```

这些样本比随机准确率更能预测线上 false positive 风险。

---

## 16. 评估指标必须改掉

不要把 F1 当主指标。

主指标建议：

```text
Auto-link Precision
False Positive Count
Coverage / Auto-link Rate
Abstain Rate
Reference Extraction Precision
Reference Role Classification Precision
Conflict Catch Rate
```

核心看：

```text
在系统愿意自动放行的那一小部分样本上，precision 到底是多少？
```

可以画：

```text
coverage - precision curve
```

业务允许漏匹配，所以完全可以主动降低 coverage 换 precision。

### 一个重要统计现实

“几百对黄金标签”不足以从统计意义上证明 `99.99%` 级 precision。

所以需要组合保障：

```text
硬规则 + 高风险挑战集 + 线上抽样审计 + 全链路可回溯 + 默认拒识
```

不能只靠一个模型阈值声称“几乎不会错”。

---

## 17. 线上决策状态机

建议不要只有 `MATCH / NO_MATCH`。

```text
UNRESOLVED
  │
  ├── no brand ----------------------> ABSTAIN
  │
  ├── no manufacturer reference ----> ABSTAIN
  │
  ├── lossy normalization ----------> REVIEW
  │
  ├── exact canonical ref mismatch -> NOT_SAME_REFERENCE
  │
  └── exact canonical ref match
          │
          ├── strong conflict -------> REVIEW
          │
          └── no conflict -----------> VERIFIED
```

`UNKNOWN` 和 `ABSTAIN` 必须是一等公民。

这和当前业务“允许漏匹配”的约束完全一致。

---

## 18. 三个平台的匹配不要直接建 pair 表

如果有三源：

```text
A: 雷小安
B: 腕表之家
C: 奢当家
```

不要长期维护：

```text
A-B pair
A-C pair
B-C pair
```

而应该全部指向：

```text
ReferenceEntity
```

例如：

```text
ReferenceEntity:
  brand = Rolex
  reference = 126610LN

members:
  leixiaoan: 123
  watchhome: 88721
  shedangjia: 5542
```

新增第四来源时无需改 N² pair 逻辑。

复杂度从“来源两两组合”转成：

```text
source record -> reference entity
```

---

## 19. 如何利用 VARM 识别“同系列不同 reference”的危险关系

可以单独维护一个 **Reference Family Graph**，但它绝不能用于 same-entity closure。

```text
126610LN ---- variant-neighbor ---- 126610LV
    │
    └──── previous-generation ---- 116610LN
```

边类型：

```text
SAME_FAMILY
SUCCESSOR
PREDECESSOR
COLOR_VARIANT
MATERIAL_VARIANT
SIZE_VARIANT
UNKNOWN_NEAR_VARIANT
```

这个图有两个用途：

1. 生成 hard negatives；
2. 当两个候选落在 variant-neighbor 上时，提高冲突等级。

但：

```text
Reference Family Graph
```

和：

```text
Reference Entity membership
```

必须完全隔离。

否则一条“同系列关系”很容易被错误传递成“同实体关系”。

---

## 20. 一个直接可执行的 Resolver 伪代码

```python
def resolve_product(product):
    brand = brand_resolver.resolve(product)
    if not brand.is_safe:
        return Decision.abstain("BRAND_UNRESOLVED")

    observations = reference_extractor.extract(product, brand)

    manufacturer_refs = [
        x for x in observations
        if x.role == "MANUFACTURER_REFERENCE"
        and not x.lossy_normalization
    ]

    safe_refs = [x for x in manufacturer_refs if evidence_policy.safe(x)]

    if len(safe_refs) == 0:
        return Decision.abstain("NO_SAFE_REFERENCE")

    canonical_values = {x.canonical_value for x in safe_refs}

    if len(canonical_values) > 1:
        return Decision.review(
            "REFERENCE_CONFLICT_WITHIN_PRODUCT",
            evidence=safe_refs,
        )

    ref = next(iter(canonical_values))

    entity = reference_registry.get(brand.id, ref)

    if entity is None:
        entity = reference_registry.create_pending(brand.id, ref)

    conflicts = deterministic_conflict_rules.check(product, entity)

    if conflicts.has_strong_conflict:
        return Decision.review("DETERMINISTIC_CONFLICT", conflicts)

    if should_call_variant_guard(product, entity):
        guard_result = variant_guard.challenge(product, entity)
        if guard_result.strong_conflict:
            return Decision.review("VARIANT_GUARD_CONFLICT", guard_result)

    return Decision.verified(entity.id)
```

其中：

```text
variant_guard.challenge()
```

没有返回 `MATCH` 的权限，它只返回：

```text
NO_STRONG_CONFLICT
STRONG_CONFLICT
UNCERTAIN
```

这能从 API 级别防止模型越权。

---

## 21. 训练 Variant Guard 的建议特征

尽量让模型看到“容易误合并的差异”，而不是只看到高层语义。

输入示例：

```text
[BRAND] Rolex
[REF_A_RAW] 126610 LN
[REF_A_CANON] 126610LN
[REF_B_RAW] 126610LV
[REF_B_CANON] 126610LV
[TITLE_A] ...
[TITLE_B] ...
[SERIES_A] Submariner
[SERIES_B] Submariner
[CASE_A] 41mm
[CASE_B] 41mm
[BEZEL_A] black
[BEZEL_B] green
```

模型输出：

```text
SAME_FAMILY_DIFFERENT_REFERENCE
```

对安全系统而言，这个标签比泛化的 `NON_MATCH` 有用得多。

---

## 22. Failure Modes 与对应处理

### 22.1 标题里出现兼容型号

```text
“适用 Rolex 126610LN 的表带”
```

危险：抽出 `126610LN` 后错误进入 Rolex 实体。

防护：

```text
role = ACCESSORY_COMPATIBLE_REFERENCE
category = strap
=> 禁止自动匹配
```

### 22.2 OCR 把 `LV` 看成 `LN`

防护：

```text
OCR observation 不能单独自动放行；
必须与文本或结构化 evidence 交叉一致。
```

### 22.3 Aggressive normalization

```text
5711/1A-010 -> 57111A01
5711/1A-011 -> 57111A01
```

这类信息损失是灾难性的。

防护：

```text
lossy normalization 永不参与 auto-link
```

### 22.4 品牌解析错误

同样数字 reference 可能跨品牌碰撞。

防护：

```text
brand unresolved -> abstain
```

### 22.5 模型过度相信图片

同系列外观极像。

防护：

```text
image = auxiliary/veto evidence only
```

### 22.6 已合并实体后来出现冲突

商品详情被更新、新增 OCR、人工纠错后，可能发现原 membership 不安全。

防护：

```text
membership 是 versioned decision
允许 REVOKED
```

---

## 23. 落地优先级

### P0：先构建不依赖模型的安全骨架

- `ReferenceObservation`；
- identifier role schema；
- brand-aware canonicalizer；
- `(brand_id, canonical_reference)` registry；
- `ABSTAIN` 状态；
- evidence lineage；
- 决策版本。

### P1：建立 Hard Negative Factory

- reference 近邻；
- 同系列不同 reference；
- 图片近邻不同 reference；
- 历史 false positive；
- 配件兼容型号。

### P2：训练 Variant Guard

目标不是高 recall，而是：

```text
把最危险的错误合并挑出来
```

### P3：离线 Variant Rule Builder

用 LLM + RAG 发现：

```text
品牌/系列变化维度
reference 语法
高风险后缀
常见冲突字段
```

人工审核后发布规则。

### P4：持续学习

人工 review 的结果进入：

```text
hard-negative bank
reference grammar tests
brand rules
model retraining
```

---

## 24. 与当前 d 目录已有调研的组合方式

这篇 VARM 不应独立存在，而可以与之前已经分析的几个方向组成完整防线：

```text
DeepBlocker / SC-Block
  -> 用于疑难无 reference 记录的候选生成

Using LLMs for Extraction / PAM / multimodal extraction
  -> reference/属性抽取

VARM（本篇）
  -> near-variant hard negative + variation guard

Confidence / Conformal selective prediction
  -> 控制 auto-link coverage，保留 abstain

TransClean
  -> 已形成实体组之后做图一致性审计
```

但主路径必须始终坚持：

```text
Reference hard evidence > learned similarity
```

---

## 25. 最终推荐架构

```text
                        ┌───────────────────────────────┐
                        │  雷小安 / 腕表之家 / 奢当家  │
                        └──────────────┬────────────────┘
                                       ▼
                              Raw / Versioned Records
                                       │
                                       ▼
                                Brand Resolver
                                       │
                                       ▼
                         Reference Candidate Extractor
                          text / field / OCR / image
                                       │
                                       ▼
                         Identifier Role Classifier
                                       │
                                       ▼
                        Brand-aware Canonicalizer
                                       │
                              ReferenceObservation
                                       │
                      ┌────────────────┴────────────────┐
                      ▼                                 ▼
             Safe Manufacturer Ref               Unsafe / missing
                      │                                 │
                      ▼                                 ▼
       (brand, canonical_ref) exact lookup          ABSTAIN
                      │
                      ▼
                 ReferenceEntity
                      │
              Deterministic Conflict Rules
                      │
                      ▼
                VARM Variant Guard
                (challenge only)
                      │
             ┌────────┴────────┐
             ▼                 ▼
        no conflict      conflict / uncertain
             │                 │
             ▼                 ▼
         VERIFIED            REVIEW
             │
             ▼
      Cross-source membership
             │
             ▼
       Transitive/graph audit
```

这个架构最大的优点是：

1. 自动匹配的安全边界是确定性的；
2. VARM 学习到的“相似 variant”不会污染实体主键；
3. 模型失败时通常只会多拒识，不会主动制造错误 merge；
4. reference extraction、模型、规则都可独立升级；
5. 支持持续增量；
6. 新增第四、第五来源不需要增加来源两两 matcher；
7. 所有自动决策都有原始证据和版本可回溯。

---

## 26. 最核心的工程结论

VARM 最值得当前需求借鉴的并不是“Encoder + GenAI 很强”，而是它背后的建模方式：

> **把“非常像但不是同一个”的 variant relationship 单独建模，而不是把它混进普通负样本。**

对腕表实体匹配来说，这可以进一步强化成：

> **reference 不同的同系列商品，是整个训练集里价值最高的负样本。**

因此最终建议是：

```text
1. Reference Entity 作为核心实体层；
2. exact canonical reference 是唯一自动合并主键；
3. reference extraction 必须带 role + evidence lineage；
4. VARM-inspired hard negatives 专门训练 near-variant guard；
5. generative LLM + RAG 只离线发现 variation rules；
6. 图片只做 OCR、hard-negative mining 和冲突证据；
7. 所有不确定情况默认 ABSTAIN；
8. 模型拥有 veto/challenge 权限，但没有越过 reference 的 approve 权限。
```

如果只实施一项来自本文的改造，我会优先做：

> **Hard Negative Factory：持续生成“同品牌、同系列、reference 极近但不同”的 pair，并把所有线上 false positive 永久回灌。**

它能直接针对当前系统最危险、同时也是普通随机训练最容易忽略的错误边界。

---

## 27. 参考资料

1. Herrero-Vidal, P. et al. *Unified AI Approach Using Encoding and Generative Large Language Models for Variant Product Matching in e-Commerce*. Big Data, 2026. <https://doi.org/10.1177/2167647X261423127>
2. Herrero-Vidal, P. et al. *Learning variant product relationship and variation attributes from e-commerce website structures*. arXiv:2410.02779, 2024. <https://arxiv.org/abs/2410.02779>
3. Amazon Science publication page. <https://www.amazon.science/publications/learning-variant-product-relationship-and-variation-attributes-from-e-commerce-website-structures>
