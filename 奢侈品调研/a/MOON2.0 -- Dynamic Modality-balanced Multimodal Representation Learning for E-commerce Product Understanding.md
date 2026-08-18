# MOON2.0：从动态模态平衡到 Precision-First 腕表 Reference Matching 的落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding** 做深入分析。

- CVPR 2026：<https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html>
- arXiv：<https://arxiv.org/abs/2511.12449>
- MBE2.0：<https://huggingface.co/datasets/ZHNie/MBE2.0>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

在分析前已检查 `奢侈品调研/a`，当前目录不存在本论文对应结果，因此不是重复分析。

MOON2.0 解决的是电商场景下的通用多模态表征学习问题，核心不是“实体匹配”本身，而是：

1. 不同输入模态比例变化时，模型不要被文本或图片某一侧长期主导；
2. 不只学习“商品 A 和商品 B 是否相关”，还显式学习同一商品内部“图像与文本必须语义一致”；
3. 面对真实电商脏数据，用动态样本过滤降低伪正例、伪负例对训练的污染；
4. 通过 MLLM + MoE 让 text-only、image-only、multimodal 输入共用一个表示空间。

这四点与当前三源二奢/腕表场景非常相关，因为雷小安、腕表之家、奢当家的字段覆盖率并不一致：有的记录有 reference，有的只有标题，有的图片信息更完整；如果用固定的文本/图片加权，非常容易在不同来源、不同品牌上发生分布漂移。

但是，**MOON2.0 不能直接拿来当最终 Matcher**。

当前 Spec 的业务定义非常严格：

> 同一个商品 = 同一个 reference number / 型号；绝对不能误匹配，precision 极端优先，允许漏匹配。

而 MOON2.0 的训练和评测目标仍然主要是 retrieval / classification / attribute prediction，它会把“语义非常像、视觉非常像”的商品拉近。在腕表里这恰好是一个危险信号：

```text
同一品牌
  └─ 同一系列
      ├─ Ref A：外观几乎相同
      ├─ Ref B：只差盘面/尺寸/机芯/年份
      └─ Ref C：标题语义也高度重合
```

因此本次建议不是“复现 MOON2.0 后用 cosine threshold 直接判同款”，而是：

> **把 MOON2.0 改造成 Candidate Recall + Multimodal Conflict Detector，最终 AUTO_LINK 必须由 reference 的确定性证据门禁放行。**

推荐落地架构：

```text
                 ┌────────────────────┐
                 │ 三来源增量商品记录 │
                 └─────────┬──────────┘
                           ▼
                Normalization / Parsing
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   结构化 Reference    标题/描述抽取      图片 OCR
          │                │                │
          └──────────┬─────┴───────┬────────┘
                     ▼             ▼
           Reference Evidence Store
                     │
          ┌──────────┴───────────┐
          ▼                      ▼
 Exact/Rule Blocking      MOON2-like Recall
          │                      │
          └──────────┬───────────┘
                     ▼
          Multimodal Verification
      （相似性 + 图文冲突 + hard negative）
                     │
                     ▼
          Precision-First Decision Gate
          ┌──────────┼───────────┐
          ▼          ▼           ▼
      AUTO_LINK   NEED_REVIEW  REJECT
          │
          ▼
 Global Reference Entity
 key = (brand_id, canonical_reference)
```

核心原则只有一句：

> **Embedding 负责找候选和发现冲突，Reference 证据负责自动合并。**

---

## 1. MOON2.0 原论文到底做了什么

### 1.1 原始问题

论文指出，已有电商多模态模型常采用固定比例混合训练，例如把 image-only、text-only、image+text 查询按固定比例送进模型。这样会造成一个问题：训练时的模态比例和真实下游任务的模态比例不同，模型会对某一模态过度依赖。

论文把这个问题称为 **modality imbalance**。

对当前三源数据，它可以直接映射为：

```text
雷小安：标题长、描述相对丰富、图片多
腕表之家：reference/型号字段可能更结构化
奢当家：标题/OCR/图片覆盖又是另一种分布
```

如果我们只训练一个固定的：

```text
score = 0.7 * text_similarity + 0.3 * image_similarity
```

那么这个权重几乎一定会在某些来源、品牌、年份上失效。

MOON2.0 的回答是：不要把模态权重只写死在数据配比里，而是让模型内部的专家路由和多目标训练自己学习“这个输入更应该由哪种能力处理”。

### 1.2 总体 Pipeline

论文把一个训练样本组织成 triplet：

```text
query q
positive p
negative n
```

q / p / n 每个元素又可以生成三种输入：

```text
text-only
image-only
multimodal = image + text
```

同时，positive / negative 还可以带 enriched title 和 augmented images。

输入经过：

```text
text ----------------------┐
                           │
image -> vision encoder -> projector -> visual tokens
                           │
                           ▼
                    Generative MLLM
                           │
                Modality-driven MoE
                           │
                    last hidden states
                           │
                      mean pooling
                           │
                           ▼
                        embedding
```

最后通过两层对齐训练：

- **Inter-product Alignment**：q 应靠近 p、远离 n；
- **Intra-product Alignment**：同一商品自己的 image/text 应互相靠近，错误商品的 image/text 应远离。

此外，训练过程中用 **Dynamic Sample Filtering** 给 triplet 估计可靠性，降低噪声样本的损失权重。

---

## 2. Modality-driven MoE：最值得借鉴的模型结构

### 2.1 为什么普通单塔也不够

MOON2.0 没有给文本、图片分别做完全独立的双塔，而是先把视觉 token 和文本 token 都送入 MLLM，再在 LLM 的 FFN 中使用 Mixture-of-Experts。

普通 MoE 的核心形式是：

```text
h -> gate -> expert_1
          -> expert_2
          -> ...
          -> expert_Z
```

路由权重：

```text
G = softmax(Wg * h)
```

选中的专家输出再加权：

```text
h_hat = Σ G_z * f_z(h)
```

问题在于：只做 token-level routing 时，gate 并不知道当前优化的是：

```text
text query -> multimodal positive
image query -> multimodal positive
multimodal query -> multimodal positive
image -> text intra-product alignment
...
```

因此论文额外增加一个 **Dual-alignment Matrix**：

```text
W* ∈ R^(Z × M)

Z = expert 数量
M = alignment objective 数量
```

其中 `W*[z,m]` 表示 expert z 对 objective m 的偏好。

对每个 expert，在 objective 维度上做 softmax：

```text
p(z,m) = softmax_m(W*[z,m])
```

然后把 token routing 与 objective preference 合起来，得到某个 alignment objective 的动态权重 `omega_m`。

论文还加入熵型 sparsity loss：

```text
L_sparsity
  = mean_z(-Σ_m p(z,m) log p(z,m))
```

最小化熵后，一个 expert 会逐渐专门服务少数几种模态对齐任务，而不是所有 expert 都学习成差不多的平均模型。

### 2.2 对腕表场景怎么改

我们不需要照搬论文里所有 objective，可以定义更贴合 reference matching 的目标：

```text
M1: title_text        -> canonical_product_mm
M2: ocr_text          -> canonical_product_mm
M3: image             -> canonical_product_mm
M4: title+image       -> canonical_product_mm
M5: title             -> same_record_image
M6: ocr_reference     -> same_record_title
M7: back/card image   -> same_record_reference_text
```

还可以把 source 作为输入 metadata，而不是直接作为标签：

```text
source = leixiaoan / xxxxx / shedangjia
```

这样模型可以学习：

- 某来源标题噪声多时更依赖图像或 OCR；
- 某来源 reference 结构化字段可靠时更依赖文本；
- 某品牌表背图比正面图更有 reference 信息时，更重视局部图像证据。

但这里有一个很重要的工程判断：

> **MoE 不是 MVP 必需品。**

论文的完整训练集约 575 万训练样本，训练约 18 小时、64 张 A100、单卡 batch size 4、learning rate 1e-5、cosine scheduler。它是重型训练方案。

当前项目只有几百对人工黄金标签，但有一个非常强的业务监督信号：**高置信 canonical reference 可以自动产生大量正负样本**。

所以更合理的顺序是：

```text
Phase 1: 规则 + reference evidence + off-the-shelf embedding
Phase 2: 利用强 reference 自动构造 triplet，训练普通多模态 encoder
Phase 3: 确认存在明显模态跷跷板后，再引入 MoE
```

不要第一天就复现 64×A100 的 MOON2.0。

---

## 3. Dual-level Alignment：对当前需求价值最高的训练思想

### 3.1 Inter-product Alignment

论文对 q / p / n 做对比学习。

以 text query 为例，核心目标是：

```text
sim(q_text, p_mm) >> sim(q_text, n_mm)
```

并分别为：

```text
q_text  -> p_mm
q_image -> p_mm
q_mm    -> p_mm
```

建立多个 objective。

当前项目可把正负例定义得比论文更严格：

#### Positive

只从满足强 reference 证据的记录中产生：

```text
same brand
AND same canonical_reference
AND no strong conflict
```

例如：

```text
Rolex / 126610LV / 雷小安 record
Rolex / 126610LV / 腕表之家 record
```

才允许作为正例。

#### Negative

不能只随机采样，必须重点生成 **same-family hard negatives**：

```text
Rolex 126610LV vs 126610LN
Omega 210.30...001 vs ...002
同品牌 + 同系列 + 极近 reference
相同外观 + 不同尺寸
相同表壳 + 不同盘面
```

这类 hard negative 对当前需求比随机负例重要得多。

随机负例：

```text
Rolex Submariner vs Cartier Tank
```

模型很快就学会，几乎没有训练价值。

真正决定误匹配率的是：

```text
同品牌 / 同系列 / 同外观 / reference 只差 1~3 个字符
```

### 3.2 Intra-product Alignment

这是 MOON2.0 对当前任务最有启发的部分之一。

很多系统只学习跨商品：

```text
record_A ~= record_B ?
```

但其实每条记录内部也有很多一致性约束：

```text
title
structured attributes
description
OCR from dial
OCR from caseback
OCR from warranty card
image visual semantics
```

如果标题说：

```text
Rolex 126610LV
```

但 OCR 在保卡/表背稳定读到：

```text
126610LN
```

这条记录本身就发生冲突。

对 precision-first 系统来说，这种冲突比“与另一条记录相似度 0.95”更重要。

因此应定义 intra-record consistency：

```text
same record:
  title_ref       <-> structured_ref
  title_ref       <-> ocr_ref
  ocr_ref         <-> image_region
  title_embedding <-> image_embedding
```

一旦强 identifier 证据冲突：

```text
CONFLICT_REJECT
```

而不是让 embedding 去投票。

---

## 4. MLLM-based Image-text Co-augmentation：能借鉴，但 reference 本身绝不能被“生成”

论文的增强分两类。

### 4.1 Textual Enrichment

论文将 title、description、image 和抽取出的 entities 输入 MLLM，产生 enriched title。

对普通商品理解很合理，但腕表 reference 有一个重大风险：

> LLM 可以补全语义，但绝不能创造 identifier。

例如原始标题：

```text
劳力士 潜航者 绿水鬼 41mm
```

如果模型“根据常识”扩写成：

```text
Rolex Submariner 126610LV 41mm
```

即使这个猜测大多数时候正确，也不能把 `126610LV` 当成真实观测证据。

因此需要把字段分成两类：

```text
GENERATIVE_ALLOWED:
  category
  series semantics
  color
  material
  style

GENERATIVE_FORBIDDEN:
  reference number
  serial number
  SKU
  warranty/card number
  seller internal id
```

所有 identifier 必须带 provenance：

```json
{
  "value": "126610LV",
  "normalized": "126610LV",
  "source": "title_span | structured_field | ocr",
  "source_record_id": "...",
  "span_or_bbox": "...",
  "extractor": "regex_v3 | ocr_v2 | llm_extractor_v1",
  "is_generated": false
}
```

如果是模型推测出来而原文不存在：

```text
is_generated = true
```

则永远不能进入 AUTO_LINK 的强证据集合。

### 4.2 Visual Expansion

MOON2.0 会做：

- 主体提取；
- 背景替换；
- 视角扩充；
- logo / 细节增强；
- 最后用 CLIP 做 image-title consistency 过滤。

当前项目中，不建议对生产数据做“生成后再识别 reference”。

更安全的用途是**仅用于训练 representation encoder 的数据增强**：

```text
训练时：允许轻微 crop / background / brightness / viewpoint augmentation
线上证据：只使用原始图片 OCR 与原图 embedding
```

特别是：

```text
表背刻字
保卡型号
吊牌 reference
盘面细字
```

不能被生成模型重绘后再当证据，因为细字符一旦被改写，正好会制造最危险的 false positive。

---

## 5. Dynamic Sample Filtering：非常适合自动生成训练对

### 5.1 原论文公式

论文对 triplet `(q,p,n)` 定义可靠性：

```text
phi = sigmoid(
  kappa * ((sim(q,p) - sim(q,n)) - Delta_bar)
)
```

其中：

- `sim(q,p)`：query 与 positive 的相似度；
- `sim(q,n)`：query 与 negative 的相似度；
- `Delta_bar`：随训练逐渐衰减的 margin；
- `kappa`：sigmoid 的锐度；
- 论文使用固定可靠性阈值 `delta = 0.6`，低可靠 triplet 会被降权。

其本质是：

> 模型刚开始只相信容易、干净的样本；embedding 稳定以后，再逐渐学习困难样本。

### 5.2 对当前项目的直接用法

我们可以用 reference 自动产生百万级训练 pair，但自动规则也不是 100% 干净：

- 标题里可能出现兼容型号；
- 配件商品可能写的是“适用于 Ref XXX”；
- 店铺 SKU 可能被误识别成 reference；
- OCR 可能读错一个字符；
- 同一页面可能同时写现款/上一代 reference；
- 套装、盒证、表带等非主体商品可能混入。

因此训练数据进入 encoder 前做两层过滤：

```text
Layer 1: deterministic evidence filtering
  - reference role == PRODUCT_REFERENCE
  - no strong conflict
  - same brand
  - not accessory / compatibility mention

Layer 2: dynamic embedding filtering
  - margin(q,p,n) 足够可靠才给高 loss weight
```

这就是 MOON2.0 的 Dynamic Sample Filtering 最适合当前项目的地方：

> **它应当用于“净化训练数据”，而不是直接变成线上 AUTO_LINK 阈值。**

论文里的 `0.6` 是训练过滤参数，不是“99.999% precision”的业务阈值，不能混用。

---

## 6. 直接可落地的系统架构

## 6.1 数据层：Global Reference Entity，而不是三源两两匹配

不推荐构造：

```text
雷小安 × 腕表之家
雷小安 × 奢当家
腕表之家 × 奢当家
```

三个独立 matcher。

这样做会出现：

- 规则重复；
- 阈值不一致；
- A=B、B=C、A≠C 的传递冲突；
- 第四来源加入时组合爆炸。

建议先建立全局实体：

```text
GlobalReferenceEntity
primary key = (brand_id, canonical_reference)
```

示例：

```json
{
  "entity_id": "rolex:126610lv",
  "brand_id": "rolex",
  "canonical_reference": "126610LV",
  "series": "Submariner",
  "status": "ACTIVE"
}
```

来源记录只需要链接到它：

```text
source_record -> GlobalReferenceEntity
```

这样系统天然支持 N 个来源。

## 6.2 Reference Evidence Store

每次识别到 reference 都不要只保留最终字符串，而要保留来源证据。

建议表结构：

```sql
reference_evidence(
    evidence_id,
    source_name,
    source_record_id,
    brand_id,
    raw_value,
    canonical_value,
    role,
    evidence_type,
    confidence,
    text_span,
    image_id,
    bbox,
    extractor_version,
    created_at
)
```

`evidence_type`：

```text
STRUCTURED_FIELD
TITLE_EXACT
DESCRIPTION_EXACT
OCR_CASEBACK
OCR_CARD
OCR_TAG
OCR_DIAL
LLM_EXTRACTED_PRESENT_IN_SOURCE
GENERATED_GUESS
```

`role`：

```text
PRODUCT_REFERENCE
COMPATIBLE_REFERENCE
ACCESSORY_FOR_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
UNKNOWN_IDENTIFIER
```

这一步非常关键。

很多误匹配的根源不是 normalization，而是 **identifier role 搞错**：

```text
“适用劳力士 126610LV 的表带”
```

里面确实有 `126610LV`，但这个商品本身并不是 126610LV。

## 6.3 Canonicalization 不能全局一套规则

不建议：

```python
canonical = re.sub(r"[^A-Z0-9]", "", raw.upper())
```

然后所有品牌都直接 exact join。

因为部分品牌的：

- 点号；
- 斜杠；
- 短横线；
- 后缀；
- 大小写；
- 版本位；

可能具有业务含义。

应使用 brand-specific canonicalizer：

```python
canonicalize(brand, raw_reference)
```

返回：

```json
{
  "canonical": "126610LV",
  "format_valid": true,
  "brand_rule_version": "rolex-v4",
  "transformations": ["upper", "trim_spaces"]
}
```

并始终保留 raw value。

---

## 7. Candidate Generation：MOON2.0 只负责 recall，不负责最终合并

### 7.1 第一层：确定性 Blocking

优先级：

```text
1. same brand + exact canonical reference
2. same brand + reference alias table
3. same brand + same series + partial/reference-pattern blocking
4. same brand + multimodal ANN recall
```

只要 record 已经有高置信 reference：

```text
直接进入 exact reference verification
```

根本不需要跑重模型。

### 7.2 第二层：Multimodal ANN

只有 reference 缺失或冲突时才走 embedding。

向量可以先用现成多模态 encoder，不必一开始训练 MOON2.0：

```text
record embedding
  = encoder(title, images)
```

ANN index 必须至少按 brand 分区：

```text
brand_id -> vector index shard
```

最好进一步做：

```text
brand + coarse_series
```

候选数量：

```text
Top-20 / Top-50
```

而不是全库 pairwise。

1000 万数据的全笛卡尔积不可接受，但 ANN recall + brand blocking 很容易控制在工程可运行范围。

### 7.3 Candidate Recall 后必须“反向取证”

假设 record A 没抽到 reference，但 embedding 召回：

```text
candidate entity = Rolex 126610LV
```

不能直接：

```text
cosine > 0.95 => AUTO_LINK
```

而应该触发 targeted extraction：

```text
1. 回看标题/描述是否真实出现 126610LV
2. 对原图重新跑高分辨率 OCR
3. 优先裁剪表背/保卡/吊牌区域
4. 判断出现位置是 PRODUCT_REFERENCE 还是 COMPATIBLE_REFERENCE
5. 与候选 entity reference 做 exact verification
```

如果最终仍然找不到强 reference 证据：

```text
NEED_REVIEW / NO_LINK
```

这就是 precision-first 与普通 retrieval 系统最大的区别。

---

## 8. Precision-First Decision Gate

建议把最终决策写成可审计规则，而不是一个不可解释的 score threshold。

### 8.1 强证据等级

建议先定义：

```text
S3 最高：
  validated structured reference field
  AND brand-consistent

S2：
  original image OCR exact reference
  + reference role confirmed
  + OCR region is caseback/card/tag/dial

S1：
  title/description exact reference span
  + role classifier == PRODUCT_REFERENCE
  + brand format valid

W：
  LLM suggestion / fuzzy match / embedding / visual similarity
```

其中 `W` 永远不能独立触发 AUTO_LINK。

### 8.2 AUTO_LINK 最小规则

第一版可以极端保守：

```python
def decide(record, entity):
    if record.brand_id != entity.brand_id:
        return "REJECT_BRAND_CONFLICT"

    if has_strong_conflicting_reference(record, entity.canonical_reference):
        return "REJECT_REFERENCE_CONFLICT"

    evidences = strong_product_reference_evidence(record)

    if any(e.canonical_value == entity.canonical_reference for e in evidences):
        return "AUTO_LINK"

    return "NEED_REVIEW"
```

即：

> AUTO_LINK 的必要条件不是模型分高，而是至少存在一条可追溯强 reference 证据精确命中，且没有任何强冲突。

### 8.3 双强证据模式

如果后续希望把误匹配概率进一步压低，可以对高风险来源/品牌启用：

```text
AUTO_LINK only if:
  strong reference evidence A
  + independent corroborating evidence B
```

例如：

```text
title exact reference
+ OCR exact reference
```

或：

```text
structured reference
+ title exact reference
```

不同 extractor 之间尽量要求独立来源，避免一个错误传播到两个字段。

---

## 9. 如何用几百对黄金标签

几百对人工标签不应该主要拿去训练一个大模型，而应优先用于 **校准整个决策链的错误边界**。

建议标注集故意偏向困难样本：

```text
40% same brand + same series + different ref
20% title contains compatible/related reference
15% OCR one-character error
10% missing reference but visually very similar
10% cross-source exact same ref
5% weird format / alias / legacy reference
```

需要标的不只 pair label：

```json
{
  "same_reference": false,
  "reference_a": "126610LV",
  "reference_b": "126610LN",
  "identifier_role_a": "PRODUCT_REFERENCE",
  "identifier_role_b": "PRODUCT_REFERENCE",
  "conflict_reason": "DIFFERENT_REFERENCE_SAME_SERIES"
}
```

这样几百条标签可以同时用于：

- identifier role classifier；
- OCR/reference extraction audit；
- hard-negative benchmark；
- decision gate precision estimation；
- source/brand-specific error analysis。

---

## 10. 训练 MOON2-like Encoder 的样本构造方法

等第一版 reference evidence 系统稳定后，可以从生产数据自动生成大量训练 triplet。

### 10.1 Positive

```sql
SELECT a, b
FROM records a
JOIN records b
  ON a.brand_id = b.brand_id
 AND a.canonical_reference = b.canonical_reference
WHERE a.source_name <> b.source_name
  AND a.strong_reference = true
  AND b.strong_reference = true
  AND a.has_conflict = false
  AND b.has_conflict = false;
```

### 10.2 Hard Negative

优先：

```text
same brand
same series
reference !=
embedding similarity high
```

例如：

```python
negative = ann_search(
    query=positive_embedding,
    filter={
        "brand": brand,
        "reference_not": positive_reference
    },
    topk=20
)
```

这里的 ANN 反而可以成为 hard-negative miner。

### 10.3 Dynamic Filtering

对自动生成 triplet：

```text
rule_confidence
× dynamic margin reliability
```

共同决定 loss weight。

例如：

```python
loss_weight = rule_confidence * phi
```

其中 `phi` 采用 MOON2.0 的动态 margin 思路。

### 10.4 训练目标

建议第一版只实现：

```text
L = L_inter + λ * L_intra
```

先不加 MoE。

只有当离线评测发现：

```text
text->mm 变好但 image->mm 明显变差
或
某来源依赖图片、另一来源依赖文本，统一模型产生明显跷跷板
```

再升级为：

```text
L_total
 = L_inter
 + L_intra
 + alpha * L_aux
 + beta * L_sparsity
```

这比先上复杂架构再寻找问题更稳。

---

## 11. 线上增量处理架构

当前规模是 100 万～1000 万，且持续更新，建议拆成离线训练链路和在线/准实时实体解析链路。

### 11.1 Ingestion

```text
crawler output
  -> Kafka / queue
  -> raw object storage
  -> normalized record table
```

幂等 key：

```text
(source_name, source_record_id, source_updated_at)
```

### 11.2 Feature / Evidence Pipeline

```text
record
  ├─ brand normalizer
  ├─ reference regex/parser
  ├─ identifier role classifier
  ├─ OCR worker
  ├─ image embedding worker
  └─ text/mm embedding worker
```

每个 worker 输出都需要：

```text
model_version / rule_version / created_at
```

方便未来重算与审计。

### 11.3 Entity Resolution Service

```text
record_id
   │
   ▼
load evidence
   │
   ├─ exact ref candidate
   └─ ANN candidates if needed
   │
   ▼
verification
   │
   ▼
decision gate
```

输出：

```json
{
  "record_id": "...",
  "decision": "AUTO_LINK",
  "entity_id": "rolex:126610lv",
  "reason_codes": [
    "BRAND_MATCH",
    "TITLE_REFERENCE_EXACT",
    "NO_REFERENCE_CONFLICT"
  ],
  "evidence_ids": ["ev1", "ev2"],
  "pipeline_version": "resolver-2026-08-v1"
}
```

### 11.4 数据组件建议

第一版不必过度设计，可以：

```text
PostgreSQL / MySQL:
  canonical entities
  record state
  evidence metadata

Object Storage:
  images
  OCR artifacts

Vector DB / FAISS service / Milvus / Qdrant:
  multimodal embeddings

Kafka / task queue:
  incremental jobs
```

1000 万量级并不要求所有东西都上复杂分布式系统，真正要重点控制的是：

- 图片/OCR吞吐；
- ANN 索引增量；
- 版本化；
- idempotency；
- reference evidence 可追踪性。

---

## 12. 为什么不建议直接用图像相似度“补齐” reference

MOON2.0 在 image-to-multimodal retrieval 上很强，这很容易让人产生一个危险方案：

```text
reference 缺失
-> 图片搜最相似商品
-> 直接继承那个商品的 reference
```

当前场景绝不能这么做。

原因：

```text
视觉近似是“同系列”的强证据
但不是“同 reference”的充分证据
```

腕表不同 reference 可能只差：

- 表径；
- 盘面颜色；
- 材质；
- 机芯代际；
- 日期窗；
- 表圈；
- 钻刻；
- 区域版本。

所以图像模型的线上权限应该是：

```text
ALLOW:
  candidate retrieval
  OCR ROI ranking
  conflict detection
  hard negative mining
  review prioritization

DENY:
  independently assigning canonical reference
  independently triggering AUTO_LINK
```

---

## 13. 用 MOON2.0 做“冲突探测器”，比做“最终裁判”更有价值

一个非常实用的改造是训练/使用多模态 encoder 计算 **intra-record inconsistency**。

例如：

```text
title embedding <-> image embedding
OCR text embedding <-> product image embedding
```

如果某条记录：

```text
title = Rolex Submariner 126610LV
image strongly resembles a different family
```

系统不应该自动修正为另一个 ref，而是打：

```text
MULTIMODAL_CONFLICT
```

进入人工复核。

这和论文 Dual-level Alignment 完全一致：

- 论文：强化同商品内部图文一致性；
- 我们：把“异常不一致”反过来当作否决/审计信号。

这是非常适合 precision-first 任务的用法。

---

## 14. 评测指标必须从 Recall@K 改成业务风险指标

MOON2.0 主要报告 retrieval Recall@K 等指标。

当前系统离线评测应使用：

```text
1. AUTO_LINK Precision
2. False Positive Count
3. False Positive per 1M records
4. Coverage / Auto-link Rate
5. Abstain Rate
6. Review Precision
7. Candidate Recall@K
8. Reference Extraction Precision
9. Reference Extraction Coverage
10. Conflict Detection Recall
```

最重要的不是总 F1，而是：

```text
在 0 / 极低 false positive 下，能覆盖多少记录？
```

建议画 selective curve：

```text
x-axis: auto-link coverage
 y-axis: measured precision / upper confidence bound of error rate
```

并按：

```text
source
brand
reference family
evidence type
```

分别统计。

### 14.1 发布门槛

第一版建议：

```text
若黄金集出现任何不可解释 false positive：
  不放宽 AUTO_LINK gate
```

系统要通过提高证据质量提升 coverage，而不是通过降低 similarity threshold 提升 coverage。

---

## 15. 实施顺序

### Phase 0：1～2 周可以完成的规则底座

目标：先建立不依赖大模型的正确数据语义。

```text
- brand canonicalization
- raw reference preservation
- brand-specific reference parser
- identifier role schema
- GlobalReferenceEntity
- evidence provenance
- exact reference linker
- conflict veto
```

产物：

```text
高 precision 的 baseline
```

### Phase 1：OCR + Multimodal Recall

```text
- 原图 OCR
- 表背/保卡/吊牌 ROI
- text/image/mm embedding
- brand-sharded ANN
- missing-reference candidate recall
- candidate-triggered targeted extraction
```

注意：

```text
仍然不允许 embedding 独立 AUTO_LINK
```

### Phase 2：用 reference 自动生成训练集

```text
- strong same-ref positives
- same-series different-ref hard negatives
- dynamic sample filtering
- inter-product + intra-product contrastive training
```

### Phase 3：MoE

仅当有离线证据证明固定 encoder 在来源/模态间出现明显跷跷板时：

```text
- insert MoE into FFN
- add objective preference matrix
- add load balancing
- add sparsity entropy loss
```

### Phase 4：主动学习

把人工预算集中到：

```text
- top similarity but different extracted refs
- multiple strong ref conflict
- unknown identifier role
- no ref but candidate gap very小
- new brand/new format
```

人工复核结果回流：

```text
parser rules
role classifier
hard-negative set
calibration set
```

---

## 16. 最小可实现版本的伪代码

```python
from dataclasses import dataclass
from typing import List, Optional


@dataclass
class Evidence:
    canonical_reference: str
    role: str
    strength: str
    provenance: str
    is_generated: bool = False


@dataclass
class MatchDecision:
    action: str
    entity_id: Optional[str]
    reasons: List[str]


def strong_product_refs(evidences):
    return [
        e for e in evidences
        if e.role == "PRODUCT_REFERENCE"
        and e.strength in {"S1", "S2", "S3"}
        and not e.is_generated
    ]


def resolve(record, entity_store, ann_index):
    evidences = extract_reference_evidence(record)
    refs = strong_product_refs(evidences)

    unique_refs = {e.canonical_reference for e in refs}

    # 强证据内部已经冲突，立即拒识
    if len(unique_refs) > 1:
        return MatchDecision(
            action="NEED_REVIEW",
            entity_id=None,
            reasons=["STRONG_REFERENCE_CONFLICT"],
        )

    # 有唯一强 reference：只做精确实体连接
    if len(unique_refs) == 1:
        ref = next(iter(unique_refs))
        entity = entity_store.get(record.brand_id, ref)

        if entity:
            return MatchDecision(
                action="AUTO_LINK",
                entity_id=entity.entity_id,
                reasons=["EXACT_STRONG_REFERENCE"],
            )

        # 合法但首次出现的 reference，可创建 pending entity
        return MatchDecision(
            action="CREATE_OR_REVIEW_NEW_REFERENCE",
            entity_id=None,
            reasons=["NEW_STRONG_REFERENCE"],
        )

    # 无强 reference：MOON2-like embedding 只召回候选
    embedding = encode_multimodal(record)
    candidates = ann_index.search(
        embedding,
        filters={"brand_id": record.brand_id},
        topk=20,
    )

    # 针对候选 reference 做二次、定向取证
    for candidate in candidates:
        targeted = targeted_reference_extraction(
            record,
            expected_reference=candidate.canonical_reference,
        )

        if targeted.has_strong_exact_evidence \
           and not targeted.has_strong_conflict:
            return MatchDecision(
                action="AUTO_LINK",
                entity_id=candidate.entity_id,
                reasons=[
                    "ANN_RECALL_ONLY",
                    "TARGETED_STRONG_REFERENCE_EXACT",
                ],
            )

    return MatchDecision(
        action="NO_LINK",
        entity_id=None,
        reasons=["NO_STRONG_REFERENCE_EVIDENCE"],
    )
```

这段逻辑刻意没有：

```text
if cosine > threshold: AUTO_LINK
```

这是为了符合当前 Spec 的核心约束。

---

## 17. 对论文实验结果的工程解读

MOON2.0 的 ablation 显示：

- 去掉 Modality-driven MoE，多种 retrieval / classification 指标明显下降；
- 去掉 Dual-level Alignment，下降尤其明显；
- 去掉 Co-augmentation 也会下降；
- 去掉 Dynamic Sample Filtering 的平均下降相对较小，但模型对脏数据更敏感。

这说明论文整体设计是有效的，但不能直接按论文指标排序决定当前项目优先级。

对我们的业务：

```text
Dual-level Alignment：高优先级
  因为可以训练 same-ref hard negatives + intra-record consistency

Dynamic Sample Filtering：高优先级
  因为我们的训练对大量来自自动 reference 规则，必须防伪标签污染

Modality-driven MoE：中期优先级
  有明显 modality/source imbalance 再上

Generative Co-augmentation：低优先级且受严格限制
  identifier 禁止生成，原图证据不可被生成图替代
```

也就是说，**最值得落地的不是整个 MOON2.0 模型，而是它的训练思想和模态冲突建模方式。**

---

## 18. 最终建议

### 可以直接采用

1. **Multimodal Joint Learning 思路**
   - 同时支持 text-only / image-only / multimodal；
   - 解决三来源字段覆盖不一致。

2. **Dual-level Alignment**
   - inter-product：同 reference 跨源拉近；
   - intra-product：标题/OCR/图片内部一致；
   - hard negative：同系列不同 reference 强制拉开。

3. **Dynamic Sample Filtering**
   - 自动生成训练 pair 时降低伪正例/伪负例污染。

4. **Embedding 作为 recall 和异常检测层**
   - 不作为最终 AUTO_LINK 证据。

### 需要改造

1. 原论文以 retrieval 为主要目标；当前系统必须加入 `ABSTAIN / NO_LINK / CONFLICT_REJECT`。
2. 原论文增强文本可以补语义；当前系统 identifier 必须 provenance-first，禁止模型生成 reference 后当真值。
3. 原论文视觉增强适合表征学习；当前系统线上 OCR/reference 只能基于原图。
4. 原论文的 similarity / reliability threshold 不能直接当业务 precision threshold。

### 不建议第一阶段做

1. 复现完整 64×A100 的 MOON2.0；
2. 通过 cosine threshold 直接合并商品；
3. 用图片相似结果反向“猜” reference 并写入实体；
4. 把所有品牌 reference 用同一个激进字符串清洗规则规范化；
5. 只用随机负例训练 matcher。

---

## 19. 一句话方案

如果要把 MOON2.0 真正落到当前 Spec，我建议最终系统定义为：

> **Reference-first Evidence Graph + MOON2-style Multimodal Recall/Conflict Model + Deterministic Precision Gate。**

其中：

```text
Reference-first
  保证业务定义不被“相似度”偷换

Evidence Graph
  保留结构化字段、标题、OCR、图片区域的可追溯证据

MOON2-style Model
  解决字段稀疏、图文不平衡、候选召回、hard-negative 与内部冲突

Deterministic Precision Gate
  保证任何没有强 reference 证据的候选都不能自动合并
```

这套方案既保留了 MOON2.0 在多模态电商表征上的优势，又不会破坏当前项目最重要的安全边界：

> **宁可漏掉，也不能把不同 reference 的腕表合并成同一商品。**
