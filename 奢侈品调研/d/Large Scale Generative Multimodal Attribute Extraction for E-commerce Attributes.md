# Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择 **Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes**（Khandelwal et al., ACL 2023 Industry Track）进行深入分析。

- 论文页面：<https://aclanthology.org/2023.acl-industry.29/>
- PDF：<https://aclanthology.org/2023.acl-industry.29.pdf>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

论文提出的 MXT（Multimodal eXtraction with T5）把电商属性抽取改造成一个多模态生成式 Question Answering 问题：输入 `attribute-name + product-type + text + image`，用 T5、MAG、ResNet/Xception 多模态融合后生成属性值。论文最值得当前项目借鉴的不是“直接用 T5 生成 reference”，而是以下四点：

1. **一个模型处理大量商品类型 × 属性组合**，而不是每个品牌、每个字段单独训练模型；
2. **文本和图片不是简单拼接，而是做 attribute-aware 的条件融合**，即问“reference number”时只让模型关注与 reference 相关的文本 token 和视觉区域；
3. **用远程监督（distant supervision）把已有结构化商品字段变成训练标签**，大幅减少人工标注；
4. **生产环境按目标 precision 做阈值和规则分层**，并承认 catalog label、值规范化、长尾值都是实际落地中的主要难点。

但当前 Spec 有一个比论文更严格的条件：

> **同一个商品严格定义为同一 reference number，且绝对不能误匹配，precision 优先到极致。**

因此，MXT 原论文的“自由生成未知属性值、支持 value-absent inference”在当前 reference 场景里不能原样使用。对于颜色、材质、表径等属性，自由生成是能力；对于 `126610LN`、`M126610LN-0001`、`IW371604` 这种标识符，自由生成则可能成为不可接受的 hallucination 来源。

本轮建议把 MXT 改造成 **Reference Evidence Extraction & Verification（REEV）**：

```text
三源原始商品
  -> 品牌规范化
  -> 低成本文本 reference 候选抽取
  -> 图片 OCR / 视觉属性抽取（仅 unresolved 数据）
  -> MXT-style attribute-aware 多模态证据融合
  -> reference role 判定（品牌型号 / 平台 SKU / 配件兼容型号 / 内部 ID）
  -> 品牌级 reference canonicalization
  -> 强冲突检查
  -> VERIFIED / CONFLICT / ABSTAIN
  -> exact entity key = (canonical_brand_id, canonical_reference)
  -> Hash Join / Group By 完成跨源聚合
```

最终自动匹配规则仍应保持硬约束：

```text
same canonical_brand_id
AND same VERIFIED canonical_reference
AND no trusted conflicting_reference
=> AUTO_MATCH

otherwise
=> ABSTAIN / MANUAL_REVIEW / NO_MATCH
```

**MXT-style 模型的职责是“帮助拿到可验证的 reference 证据”，而不是直接决定两条商品是不是同一商品。**

这会把当前任务从一个危险的“千万级模糊 pairwise matching”问题，重构为一个更可控的：

> **高精度 reference 抽取 + 规范化 + exact join 问题。**

---

## 1. 为什么选择这篇论文

当前 Spec 的困难不在于“相似商品找不到”，而在于：

- 三个平台字段不统一；
- reference 有时是独立字段，有时埋在标题/描述中；
- 还有图片，但 reference 可能只出现在表背、保卡、吊牌、盒贴上；
- 同系列相邻型号文本和图片极其相似；
- 数据量 100 万–1000 万级，需要持续增量；
- 最重要的是误匹配代价高到无法接受。

`奢侈品文章调研.md` 对这篇论文的推荐点是：它是工业级多模态属性抽取系统，论文报告已经扩展到超过 10K 个 product-type/attribute 组合，并抽取超过 150MM 属性值，同时用 `recall@90P` 这种“固定高 precision 看 recall”的指标做生产评估。

这与当前系统有一个非常直接的对应关系：

```text
论文：attribute = color / sleeve type / neck style / ...
当前：attribute = reference_number / collection / case_size / material / movement / ...
```

其中 `reference_number` 是特殊的“身份属性”：

```text
普通属性错了：可能只是搜索/展示质量下降
reference 错了：可能直接导致实体误合并
```

所以这篇论文适合拿来回答当前系统的一个关键子问题：

> 当 reference 不在结构化字段里时，如何利用标题、描述和图片，把 reference 作为一个高精度属性抽出来，并且让系统可以扩展到很多品牌、很多来源和持续新增的型号？

---

## 2. MXT 原始技术架构

## 2.1 任务建模：把 Attribute Extraction 变成 Q&A

论文不把每个属性做成一个独立分类器，而是统一建模成：

```text
Question = attribute name
Context  = product type + textual description + image
Answer   = attribute value
```

例如：

```text
Question: color
Product type: dress
Text: ...
Image: ...
Answer: red
```

这种 prompt 化设计的重要意义是：输出空间不再跟 `(product-type, attribute)` 数量一起无限膨胀。

传统多任务分类常见形态：

```text
shared encoder
  ├─ color head
  ├─ material head
  ├─ sleeve head
  ├─ neck head
  ├─ ... 每个 PT-attribute 都有独立输出空间
```

当 PT-attribute 上万后，训练、刷新、监控、输出层维护都会变得困难。

MXT 则是：

```text
(attribute_name, product_type, text, image)
                 |
                 v
            shared model
                 |
                 v
          generated value
```

论文在生产环境中将这种方案扩展到 6 个英语市场、超过 10K 个 PT-attribute，并报告抽取了超过 150MM 个属性值。

对当前项目的迁移是：**不要为劳力士、欧米茄、卡地亚分别维护三套 reference 模型，也不要为三个来源分别训练模型；应该共享一个抽取骨干，把 brand/source/attribute 作为条件输入。**

---

## 2.2 第一层：Image-aware Text Encoder

MXT 的文本主干是 `T5 encoder`。

输入文本由三部分构成：

```text
attribute-name + product-type + product description
```

先得到 token embeddings：

```text
T_emb ∈ R^(N × d)
```

然后引入全局图片 embedding。论文使用预训练 ResNet-152 得到每张图片的视觉向量 `V_R`，再通过 MAG（Multimodal Adaptation Gate）把图像信息注入每个文本 token。

核心不是直接：

```text
concat(text_embedding, image_embedding)
```

而是先根据“当前 token + 图片”学习 gate：

```text
g_i = ReLU(W_g [T_emb_i ; V_R] + b_g)
```

然后构造视觉 displacement：

```text
H_i = g_i · (W_H V_R) + b_H
```

再把这个视觉偏移有界地加到文本 embedding 上：

```text
T_hat_i = T_emb_i + alpha * H_i
```

其中 `alpha` 会根据文本向量和视觉偏移的范数做限制，避免视觉信息把原有语言空间完全冲坏。

最后做 LayerNorm + Dropout，再送进 T5 encoder。

### 这个设计对腕表 reference 的意义

如果直接使用图像 global embedding 做“同款相似度”，非常容易把同系列相邻 reference 混在一起。

MAG 值得借鉴的点是：

> 图片不是独立投票，而是用于改变“当前任务下文本 token 的解释”。

例如标题：

```text
劳力士 黑水鬼 41mm 126610 LN 全套
```

当 attribute 问题是：

```text
reference_number
```

模型应该重点关注：

```text
126610 LN
```

当 attribute 问题是：

```text
case_size
```

模型应该关注：

```text
41mm
```

同一条文本，不同 attribute 应该产生不同注意力和证据路径。

---

## 2.3 第二层：Attribute-aware Text-Image Fusion

论文发现仅用一个全局图片 embedding 不够，因为不同属性依赖图片不同区域。

例如服装：

- sleeve type 看袖子；
- neck style 看领口；
- length 看下摆。

所以 MXT 又使用 Xception 的 depthwise separable convolution 学习 region-specific visual features，然后通过 multi-head cross-attention，用已经编码的文本表示去选择与 attribute 相关的视觉区域。

逻辑可以简化为：

```text
T_enc = image-aware text embeddings
V_X   = region-specific visual features

F_A = CrossAttention(query=T_enc, key=V_X, value=V_X)
```

最终得到 attribute-aware fused embedding `F_A`。

### 迁移到腕表场景

对于 reference，不同图片位置的重要性同样差异很大：

```text
正面表盘：
  - 品牌 / 系列 / 外观特征
  - 通常不直接显示完整 reference

表背：
  - 可能出现刻字、编号、型号片段

保卡：
  - reference / serial / purchase info

盒贴 / 吊牌：
  - reference / barcode / SKU

普通场景图：
  - 对 reference 几乎无直接身份价值
```

因此，当前系统也不应该把每张图片都等权做 CLIP 相似度，而应该变成：

```text
Question = reference_number
        |
        v
attribute-aware image region / OCR attention
        |
        v
只挑有机会出现 reference 的视觉区域
```

实际工程上甚至不必一开始完整复刻 Xception + cross-attention；第一阶段可以先用低风险版本：

```text
图片分类器
  -> watch_front / caseback / warranty_card / box_label / tag / other

仅对高价值图片：
  -> OCR
  -> reference candidate extractor
```

这比对所有 1000 万商品图片直接跑大 VLM 更便宜，也更容易做 precision 控制。

---

## 2.4 第三层：T5 Decoder 自回归生成属性值

MXT 把融合表示送到 T5 decoder，自回归生成属性值：

```text
P(y_j | y_<j, x, I)
```

训练目标是标准 token-level negative log likelihood：

```text
L = -Σ log P(y_j | y_<j, x, I)
```

论文采用 greedy decoding。

这种 generative formulation 让它具备两个分类/NER 很难同时做到的能力：

1. **zero-shot value**：训练时没见过的属性值仍有机会生成；
2. **value-absent**：值没明确写在文本里，也可从图片或上下文推断。

这对颜色、服装长度、款式等属性是明显优势。

但对当前 reference 系统，这是最需要改造的地方。

---

## 3. 为什么不能直接让 MXT 生成 reference

## 3.1 Reference 不是普通语义属性，而是身份标识符

例如：

```text
126610LN
126610LV
116610LN
124060
```

这些字符串语义上都很接近，外观也可能接近，但在业务定义下是完全不同的实体键。

一个普通属性模型生成：

```text
black -> dark black
```

可能只是轻微语义偏差。

一个 reference 模型生成：

```text
126610LN -> 126610LV
```

则可能直接造成错误聚类。

因此当前系统必须明确：

```text
Generative Reference != Verified Reference
```

任何“模型补出来的 reference”都不能直接成为自动 MATCH 的硬键。

---

## 3.2 论文自己的 tokenizer limitation 对编号更严重

论文在 limitation 中明确指出，T5 的 open-domain tokenizer 对电商专有词分词不好。

腕表 reference 比普通电商术语更加极端：

```text
M126610LN-0001
IW371604
WSSA0018
210.30.42.20.03.001
Q4148420
RM 11-03
```

如果 tokenizer 把一个 identifier 拆成大量 subword，模型需要逐 token 生成并保持每一位精确一致，错误概率会累积。

因此 reference 应使用 identifier-aware pipeline：

```text
character/token-span extraction
+ regex / syntax constraints
+ candidate dictionary / trie
+ exact canonicalization
```

而不是把主流程建立在自然语言 decoder 的自由生成上。

---

## 3.3 “Value-absent inference”对 reference 是风险，不是优势

论文强调：即使属性值没有出现在文本中，也可以根据图片推断出来。

但 reference 的业务语义不同：

```text
图片看起来像 126610LN
```

不能推出：

```text
reference = 126610LN
```

因为可能是：

- 相邻 reference；
- 改装表；
- 同壳型不同配置；
- 配件/表带；
- 图片复用；
- 假货或错误配图。

所以对 reference 必须采用：

```text
Evidence of identity > Visual similarity
```

图片可以帮助“找证据”，不能代替证据。

---

## 4. 面向当前 Spec 的改造：REEV 架构

下面给出一个可以直接实施的 **Reference Evidence Extraction & Verification** 架构。

## 4.1 总体数据流

```text
                      ┌───────────────────────┐
雷小安 ──────────────>│                       │
腕表之家 ────────────>│   Raw Product Store   │
奢当家 ──────────────>│                       │
                      └──────────┬────────────┘
                                 │
                                 v
                      ┌───────────────────────┐
                      │ Brand Normalization   │
                      └──────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    v                         v
          ┌──────────────────┐      ┌───────────────────────┐
          │ Text Candidate   │      │ High-value Image Gate │
          │ Extraction       │      │ + OCR                 │
          └────────┬─────────┘      └──────────┬────────────┘
                   │                           │
                   └────────────┬──────────────┘
                                v
                  ┌─────────────────────────────┐
                  │ Reference Candidate Pool    │
                  │ value + source + span + box │
                  └──────────────┬──────────────┘
                                 │
                                 v
                  ┌─────────────────────────────┐
                  │ Reference Role Classifier   │
                  │ REF / SKU / SERIAL / COMPAT │
                  └──────────────┬──────────────┘
                                 │
                                 v
                  ┌─────────────────────────────┐
                  │ Brand-specific Canonicalizer│
                  └──────────────┬──────────────┘
                                 │
                                 v
                  ┌─────────────────────────────┐
                  │ Multimodal Evidence Verifier│
                  │ MXT-style attribute aware   │
                  └──────────────┬──────────────┘
                                 │
                                 v
                  ┌─────────────────────────────┐
                  │ Hard Conflict Rules         │
                  └──────────────┬──────────────┘
                                 │
             ┌───────────────────┼───────────────────┐
             v                   v                   v
         VERIFIED             CONFLICT             ABSTAIN
             │
             v
    entity_key=(brand_id, canonical_reference)
             │
             v
       exact join/group-by
             │
             v
     cross-source product entity
```

这个架构保留了 MXT 的多模态属性感知思想，但把最后的“生成属性值”替换成了**候选证据验证 + 强规则收口**。

---

## 4.2 Step A：品牌先规范化

reference 的格式与品牌高度相关，所以先做 brand canonicalization。

建议表：

```sql
brand_dictionary(
  brand_id            bigint primary key,
  canonical_name      text,
  aliases             jsonb,
  reference_rule_set  text,
  status              text
)
```

例如：

```text
ROLEX
Rolex
劳力士
劳
劳服
```

全部映射：

```text
brand_id = rolex
```

### 为什么品牌必须先于 reference

同一串数字在不同品牌可能含义完全不同，甚至是平台 SKU。

因此 canonical key 绝不能只用：

```text
canonical_reference
```

必须是：

```text
(brand_id, canonical_reference)
```

---

## 4.3 Step B：Reference Candidate Extraction

候选抽取不要直接做“预测唯一答案”，而是保留所有可能证据。

### 来源 1：结构化字段

可信度最高：

```text
source_reference_field
official_model_field
platform_attribute.reference
```

保存：

```json
{
  "candidate": "126610LN",
  "evidence_type": "STRUCTURED_FIELD",
  "source_field": "model_no",
  "raw": "126610 LN",
  "confidence": 1.0
}
```

注意：`confidence=1.0` 只表示“成功读到字段”，不代表字段一定真实。后续仍需 conflict check。

### 来源 2：标题 / 描述 span

推荐先做规则 + 字符级模型混合：

```text
1. regex / brand syntax pre-filter
2. identifier token segmentation
3. span scorer
4. role classifier
```

例如：

```text
标题：劳力士潜航者 126610 LN 黑水鬼 41 全套

候选：
126610
126610LN
41
```

通过品牌语法规则：

```text
41 -> SIZE，不是 REF
126610LN -> REF candidate
```

### 来源 3：OCR

图片先分类：

```text
CASEBACK
WARRANTY_CARD
BOX_LABEL
TAG
DIAL
WRIST_SHOT
OTHER
```

只对前四类高价值图片跑高成本 OCR。

OCR 输出必须保存 bbox：

```json
{
  "text": "M126610LN-0001",
  "image_id": "...",
  "bbox": [x1, y1, x2, y2],
  "ocr_confidence": 0.98,
  "image_type": "WARRANTY_CARD"
}
```

这样人工复核可以直接看到“这串 reference 从哪张图哪个区域来的”，而不是只看到模型输出。

---

## 4.4 Step C：Reference Role Classifier

这是当前系统里非常关键但容易被忽略的一层。

标题里出现像型号的字符串，不代表它是“当前出售商品的 reference”。

常见编号角色：

```text
BRAND_REFERENCE      品牌官方型号 / reference
PLATFORM_SKU         平台内部货号
SELLER_SKU           商家 SKU
SERIAL_NUMBER        唯一序列号
COMPATIBLE_REFERENCE 配件适配的手表型号
OTHER_IDENTIFIER     其他编号
```

例如：

```text
适配 Rolex 126610LN 橡胶表带
```

如果只做字符串抽取：

```text
reference = 126610LN
```

将导致灾难性误匹配。

所以候选必须先判角色：

```text
candidate span
+ left/right context
+ category
+ brand
+ source field name
+ image type
    -> role classifier
```

自动进入 VERIFIED 的只能是：

```text
role = BRAND_REFERENCE
```

`COMPATIBLE_REFERENCE` 必须成为强否决证据之一。

---

## 4.5 Step D：Brand-specific Canonicalization

reference canonicalization 需要非常保守。

建议拆成两套值：

```text
raw_reference
canonical_reference
```

并保存规则版本：

```text
canonicalizer_version
```

### 可安全处理的变化

一般可自动统一：

```text
大小写
全角/半角
首尾空格
品牌已知无语义空格
品牌已知无语义分隔符
```

例如某品牌规则明确证明：

```text
126610 LN -> 126610LN
```

### 不能全局删除的字符

不要全局粗暴：

```python
re.sub(r'[^A-Z0-9]', '', ref)
```

因为对某些品牌：

```text
- / . 空格 后缀
```

可能有语义。

正确方式：

```text
canonicalize(brand_id, raw_reference)
```

即每个品牌维护有限、可审计的规则。

建议表：

```sql
reference_normalization_rule(
  brand_id      bigint,
  version       int,
  pattern       text,
  replacement   text,
  rule_type     text,
  reviewed_by   text,
  enabled       boolean
)
```

---

## 4.6 Step E：MXT-style Multimodal Evidence Verifier

这里才是论文核心最适合迁移的位置。

不让模型自由生成 reference，而是给它候选：

```text
Candidate reference = 126610LN
```

让模型回答：

```text
当前商品的多模态证据是否支持“126610LN 是商品自身的 reference”？
```

输出建议不是 `MATCH/NO_MATCH`，而是结构化 evidence score：

```json
{
  "candidate": "126610LN",
  "role_score": 0.9991,
  "text_support": 0.9983,
  "ocr_support": 0.9960,
  "visual_family_support": 0.91,
  "conflict_score": 0.002,
  "evidence_level": "STRONG"
}
```

### 模型输入

```text
attribute = reference_number
brand = ROLEX
category = watch
source = luxiaoan
candidate = 126610LN
text = title + description + structured attrs
image = selected high-value images
ocr = OCR tokens + bbox
```

### 推荐的模型目标

把原论文的 generative loss：

```text
generate(reference)
```

改成多任务：

```text
L = λ1 * candidate_role_loss
  + λ2 * candidate_support_loss
  + λ3 * conflict_loss
  + λ4 * auxiliary_attribute_loss
```

辅助属性可以包括：

```text
collection
case_size
material
bezel_color
movement
watch/accessory category
```

这些辅助任务让视觉 encoder 学到有意义的腕表区域，但最终不能覆盖 reference hard evidence。

---

## 5. 推荐的证据等级，而不是单一概率

当前项目不应该简单地：

```text
score > 0.95 => match
```

因为不同 evidence path 的风险完全不同。

建议定义 `ReferenceEvidenceLevel`：

### L0 — NONE

```text
没有 reference 候选
```

结果：`ABSTAIN`

### L1 — MODEL_INFERRED

```text
只有模型根据标题/图片推断可能 reference
但没有直接字符串证据
```

结果：永不自动 VERIFY，只做候选召回或人工提示。

### L2 — TEXT_SPAN

```text
标题/描述中明确出现 candidate
role classifier 判为 BRAND_REFERENCE
```

可用于候选验证，但是否自动 VERIFY 需要品牌级黄金集校准。

### L3 — OCR_DIRECT

```text
保卡 / 盒贴 / 吊牌 / 表背 OCR 明确出现 candidate
且 role/context 合理
```

通常比纯视觉相似强，但仍要防：

```text
配件标签
复用图片
识别错字符
```

### L4 — STRUCTURED_DIRECT

```text
平台独立 reference/model 字段直接给出
```

高置信，但仍要检查与文本/OCR是否冲突。

### L5 — MULTI_SOURCE_CORROBORATED

同一商品记录内部至少两个独立证据一致：

```text
STRUCTURED + TEXT
STRUCTURED + OCR
TEXT + OCR
```

且无冲突。

这是最适合自动进入 `VERIFIED` 的路径。

---

## 6. Hard Conflict 必须高于模型支持分数

precision-first 系统需要“否决权”。

建议原则：

```text
strong conflict > any positive similarity score
```

例如：

```text
结构化字段：126610LV
标题抽到：126610LN
```

即使图片模型认为 `126610LN` 概率 0.99，也不能自动选择任何一个。

直接：

```text
CONFLICT
```

再如：

```text
标题：适配 126610LN 表带
category：watch strap
```

即使 reference 字符串完全一致：

```text
role = COMPATIBLE_REFERENCE
=> NEVER VERIFY AS WATCH REFERENCE
```

建议 conflict rule：

```python
def verify_reference(evidences):
    trusted = [e for e in evidences if e.level >= TRUSTED_LEVEL]

    canonical_values = set(e.canonical_reference for e in trusted)

    if len(canonical_values) > 1:
        return "CONFLICT"

    if any(e.role in {"PLATFORM_SKU", "SELLER_SKU", "SERIAL_NUMBER", "COMPATIBLE_REFERENCE"}
           and e.is_primary_candidate for e in evidences):
        return "ABSTAIN"

    if has_required_independent_support(evidences) and no_strong_conflict(evidences):
        return "VERIFIED"

    return "ABSTAIN"
```

关键是：**ABSTAIN 是正常输出，不是模型失败。**

---

## 7. 一旦 reference VERIFIED，匹配层应该极简

当记录已经被解析成：

```json
{
  "product_id": "lx_123",
  "brand_id": "rolex",
  "canonical_reference": "126610LN",
  "reference_status": "VERIFIED"
}
```

跨来源匹配不需要再上复杂神经网络。

直接：

```sql
SELECT
    brand_id,
    canonical_reference,
    array_agg(product_id) AS products,
    array_agg(source) AS sources
FROM product_reference
WHERE reference_status = 'VERIFIED'
GROUP BY brand_id, canonical_reference;
```

这在 1000 万级数据上是一个标准 hash/group-by 问题，而不是 `10^14` 级 pairwise 比较问题。

### 数据复杂度变化

错误方案：

```text
N 条商品两两比较
O(N²)
```

10M 规模不可接受。

推荐方案：

```text
每条商品一次 reference extraction
O(N)

然后 exact key join/group
O(N) ~ O(N log N)
```

真正昂贵的多模态模型只跑 unresolved tail：

```text
全部数据
  -> 结构化字段 exact extraction
  -> 文本规则 extraction
  -> unresolved 才跑 OCR / multimodal verifier
```

这样成本可以按漏斗逐层缩小。

---

## 8. 数据库 Schema 建议

## 8.1 raw_product

```sql
CREATE TABLE raw_product (
    source              varchar(32) NOT NULL,
    source_product_id   text NOT NULL,
    crawl_version       bigint NOT NULL,
    title               text,
    description         text,
    raw_attributes      jsonb,
    image_urls          jsonb,
    category_raw        text,
    updated_at          timestamptz,
    PRIMARY KEY (source, source_product_id, crawl_version)
);
```

不要覆盖原始数据，保留每次 crawl version，方便追查 reference 为什么发生变化。

---

## 8.2 reference_evidence

```sql
CREATE TABLE reference_evidence (
    evidence_id             bigserial PRIMARY KEY,
    source                  varchar(32) NOT NULL,
    source_product_id       text NOT NULL,
    raw_candidate           text NOT NULL,
    canonical_candidate     text,
    brand_id                bigint,
    evidence_type           varchar(32) NOT NULL,
    evidence_location       jsonb,
    role                    varchar(32),
    extractor_version       varchar(64) NOT NULL,
    model_score             double precision,
    ocr_score               double precision,
    created_at              timestamptz DEFAULT now()
);
```

`evidence_location` 示例：

```json
{
  "field": "title",
  "char_start": 18,
  "char_end": 26
}
```

或：

```json
{
  "image_id": "img_001",
  "bbox": [120, 80, 460, 160],
  "image_type": "WARRANTY_CARD"
}
```

任何自动匹配都必须可回溯到具体 evidence。

---

## 8.3 product_reference

```sql
CREATE TABLE product_reference (
    source                  varchar(32) NOT NULL,
    source_product_id       text NOT NULL,
    brand_id                bigint,
    canonical_reference     text,
    status                  varchar(16) NOT NULL,
    decision_reason         text,
    verifier_version        varchar(64),
    canonicalizer_version   varchar(64),
    decided_at              timestamptz,
    PRIMARY KEY (source, source_product_id)
);
```

状态：

```text
VERIFIED
CONFLICT
ABSTAIN
NO_REFERENCE
INVALID
```

---

## 8.4 canonical_entity

```sql
CREATE TABLE canonical_entity (
    entity_id               bigserial PRIMARY KEY,
    brand_id                bigint NOT NULL,
    canonical_reference     text NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);
```

映射表：

```sql
CREATE TABLE entity_member (
    entity_id               bigint NOT NULL,
    source                  varchar(32) NOT NULL,
    source_product_id       text NOT NULL,
    joined_at               timestamptz DEFAULT now(),
    PRIMARY KEY (source, source_product_id)
);
```

---

## 9. 如何利用“几百对黄金标签”

Spec 允许人工标注几百对，这个预算不应该平均撒在随机样本上。

建议标签分成三类。

## 9.1 Reference Extraction Gold

标注单条记录：

```text
品牌
真实 reference
reference span
编号角色
是否有冲突
```

优先覆盖：

```text
高频品牌
标题格式复杂来源
reference 相邻型号
配件/表带
结构化字段缺失
OCR 场景
```

---

## 9.2 Hard Negative Pair Gold

专门构造：

```text
same brand
same collection
visually similar
reference differs by 1~3 chars
```

例如：

```text
126610LN vs 126610LV
116610LN vs 126610LN
```

这类样本对 precision 比随机负样本重要得多。

---

## 9.3 Calibration Gold

黄金集最重要的用途之一不是训练，而是：

```text
决定哪些 evidence path 可以自动 VERIFY
```

例如分别测：

```text
STRUCTURED_DIRECT
TEXT_SPAN
OCR_DIRECT
TEXT+OCR
STRUCTURED+TEXT
```

目标不是得到一个总 F1，而是回答：

```text
在这个 evidence path 上，precision 的置信下界是否足够高？
```

如果没有足够证据证明安全，就继续 `ABSTAIN`。

---

## 10. 用论文的 Distant Supervision 低成本构造训练集

论文通过 catalog 中已有属性值做远程监督，从而避免大规模人工标注。

当前系统可以做得更自然。

### 正样本构造

找 reference 独立字段明确的记录：

```text
label = structured_reference
```

训练时把该字段从输入中 mask 掉，只让模型看：

```text
title + description + image/OCR + other attrs
```

任务：

```text
从非 reference 字段恢复 / 验证 reference evidence
```

例如：

```text
raw:
model_no = 126610LN
 title   = 劳力士潜航者 黑水鬼 41mm 126610 LN

train input:
model_no = [MASKED]
title    = ...

label:
126610LN
```

这样可从现有数据自动得到大量弱监督样本。

### Hard Negative 构造

不要随机配负样本，而是：

```text
same brand
same family
edit_distance(reference) small
image/text similarity high
reference != label
```

这种训练集才能真正教模型“不要把相似变体当同 reference”。

---

## 11. MXT 模型如何做工程简化

完整复刻论文的：

```text
T5 + MAG + ResNet-152 + Xception + cross-attention + decoder
```

不是 MVP 的必要条件。

当前系统可以分三阶段。

## Phase 1：规则优先，无大模型

```text
structured extraction
+ brand normalization
+ regex/span candidate extraction
+ brand-specific canonicalization
+ exact join
```

目标：快速覆盖最安全的 50%~80% 数据。

---

## Phase 2：OCR + 轻量 verifier

对 unresolved：

```text
high-value image classifier
+ OCR
+ text/OCR role classifier
+ candidate verifier
```

这阶段通常比直接上 end-to-end VLM 更可控。

---

## Phase 3：MXT-style multimodal model

只处理：

```text
标题极脏
reference 不在显式字段
OCR 多候选冲突
同系列 hard cases
```

模型输出只增加：

```text
candidate ranking
support score
conflict score
auxiliary attributes
```

不增加“自由生成并自动合并”的权限。

---

## 12. 在线 / 增量架构

当前数据持续抓取，不能每次全量重跑。

建议事件化：

```text
product_created / product_updated
       |
       v
normalize_brand
       |
       v
extract_reference_fast
       |
       +-- VERIFIED --> upsert entity_member
       |
       +-- unresolved --> async OCR queue
                           |
                           v
                      multimodal verify
                           |
                           +-- VERIFIED
                           +-- CONFLICT
                           +-- ABSTAIN
```

### 幂等 key

```text
(source, source_product_id, crawl_version, extractor_version)
```

同一记录只有在以下变化时重算：

```text
标题/描述变化
reference 字段变化
图片变化
模型版本变化
canonicalizer 规则变化
```

避免对 1000 万历史数据无差别重复推理。

---

## 13. API 设计建议

## 13.1 Reference extraction API

```http
POST /v1/reference/extract
```

Request：

```json
{
  "source": "wanbiaozhijia",
  "source_product_id": "123",
  "brand": "劳力士",
  "title": "劳力士潜航者 126610 LN 黑水鬼 41mm",
  "description": "...",
  "attributes": {},
  "images": []
}
```

Response：

```json
{
  "brand_id": "rolex",
  "candidates": [
    {
      "raw": "126610 LN",
      "canonical": "126610LN",
      "role": "BRAND_REFERENCE",
      "evidence_type": "TEXT_SPAN",
      "score": 0.999
    }
  ],
  "status": "VERIFIED",
  "canonical_reference": "126610LN",
  "decision_reason": "TEXT_DIRECT+BRAND_RULE+NO_CONFLICT"
}
```

---

## 13.2 Explain API

```http
GET /v1/reference/explain/{source}/{product_id}
```

必须返回完整 evidence chain：

```text
原始字符串
规范化规则
OCR bbox
角色判定
冲突检查
模型版本
最终决策
```

precision-first 系统一定要能回答：

> “为什么这两条商品被合并？”

答案不能只是：

```text
模型分数 0.984
```

而应该是：

```text
两条记录 brand 都规范为 ROLEX；
A 的结构化 model_no = 126610LN；
B 标题直接出现 126610 LN，role=BRAND_REFERENCE；
B 保卡 OCR 再次读到 126610LN；
品牌规则 v12 规范后完全一致；
没有任何 trusted conflicting reference；
因此 entity_key 均为 (ROLEX, 126610LN)。
```

---

## 14. 评估指标必须围绕误匹配风险设计

论文使用 `Recall@90P`，这比只看 F1 更接近当前需求，但 90% precision 对当前业务仍远远不够。

当前建议拆四组指标。

### 14.1 Reference Extraction Precision

```text
P(extracted reference is correct | status=VERIFIED)
```

这是第一优先级。

### 14.2 Auto-Match Precision

```text
P(same reference | AUTO_MATCH)
```

应该是系统最高门槛。

### 14.3 Coverage

```text
VERIFIED records / all records
```

在保证 precision 后才优化 coverage。

### 14.4 Conflict Detection Recall

对已知冲突样本：

```text
有多少被正确拦截，而不是错误进入 VERIFIED
```

---

## 15. 不要用随机切分做最终验收

随机 train/test split 会高估真实效果。

必须专门做以下切片：

```text
unseen_reference
unseen_brand_or_long_tail_brand
same_family_different_reference
reference_one_char_difference
structured_field_missing
OCR_only
accessory_contains_watch_reference
wrong_image
duplicate_image
source_specific_noise
new_crawl_batch
```

特别需要一个：

```text
NEAR_REFERENCE_NEGATIVE
```

集合，专门测试：

```text
文本高度相似
图片高度相似
但 reference 不同
```

如果这一组还有 false positive，系统就不能自动上线。

---

## 16. 论文生产经验对当前系统的直接启发

论文的 deployment 部分比单纯模型结构更值得借鉴。

## 16.1 真实 Catalog 的标签本身有噪声

论文明确提到 distant supervision 的 catalog value 中存在 junk value，需要做 heuristic normalization 和 tail trimming。

当前三个来源同样不能把任一平台字段无条件当 ground truth。

所以：

```text
source field = evidence
source field != truth
```

即使独立 reference 字段也要保留：

```text
field provenance
source reliability
conflict check
```

---

## 16.2 阈值不能只有一个全局值

论文在 >10K PT-attribute 上面临每个组合如何达到目标 precision 的问题。

当前也不应该：

```text
all brands score > 0.95 => VERIFIED
```

不同品牌和来源的 reference 格式差异太大。

建议 threshold key 至少包含：

```text
brand
source
reference_evidence_path
```

例如：

```text
ROLEX + sourceA + STRUCTURED_DIRECT
ROLEX + sourceB + OCR_DIRECT
OMEGA + sourceC + TEXT_SPAN
```

分别校准。

---

## 16.3 一个共享模型，比大量品牌专模更易维护

论文证明 multi-PT 模型在生产维护上优势明显。

当前推荐：

```text
shared encoder
+ brand/source conditioning
+ shared candidate verifier
+ small brand-specific canonicalization rules
```

不要变成：

```text
RolexModel
OmegaModel
CartierModel
IWCModel
...
```

品牌特异性应该优先放在：

```text
reference grammar
normalization rules
threshold/calibration
```

而不是无限复制完整模型。

---

## 17. 计算成本建议

论文训练配置：

```text
20 epochs
batch size = 4
8 × NVIDIA V100 16GB
T5-base
ResNet-152 pretrained image embedding
Adam lr = 5e-5
warmup ratio = 0.1
greedy decoding
```

当前不建议直接照这个配置跑 1000 万商品。

原因：

```text
绝大多数有显式 reference 的记录根本不需要视觉模型
```

推荐成本漏斗：

```text
100% records
  -> cheap structured/text parser

假设剩余 30%
  -> brand-aware span extractor

假设剩余 10%
  -> image type classifier + OCR

假设剩余 2%-5%
  -> multimodal verifier

极少量
  -> manual review
```

最昂贵的模型只跑尾部，这比把 1000 万商品全部送进 VLM 的成本更合理。

---

## 18. 与当前 d 目录其他方案的组合方式

本篇不替代此前的 DeepBlocker、AMELI、TransClean、Conformal 等分析，而是补上一个更明确的“属性抽取生产层”。

可以组合成：

```text
[MXT/REEV]
负责：reference / secondary attributes 从 text+image 中抽取和验证

        |
        v
[Canonical Reference Exact Key]
负责：把 VERIFIED reference 变成实体键

        |
        +------------------+
        |                  |
        v                  v
[DeepBlocker]          [AMELI-style]
只用于 unresolved      候选实体/属性消歧
candidate recall       辅助层

        |
        v
[Conformal / Confidence]
对非硬规则路径做 precision-first calibration / abstain

        |
        v
[TransClean]
对多源实体图做 false-positive 二次审计
```

但要保持一个不变的权限边界：

```text
任何 semantic / image similarity
不能覆盖 trusted reference conflict
```

---

## 19. MVP 可以直接落地的最小方案

如果现在就开始实现，我建议第一版只做下面 8 个组件：

```text
1. BrandNormalizer
2. ReferenceRegexExtractor
3. ReferenceRoleClassifier
4. BrandReferenceCanonicalizer
5. EvidenceStore
6. ConflictDetector
7. ReferenceDecisionEngine
8. EntityExactJoiner
```

先不要上完整 MXT。

### 第一阶段规则

```text
A. 结构化 reference 字段存在
B. brand canonicalization 成功
C. reference 通过品牌语法校验
D. 标题/描述不存在另一个可信冲突 reference
E. 商品不是 accessory

=> VERIFIED
```

第二阶段再加：

```text
OCR + image type classifier
```

第三阶段才加：

```text
MXT-style multimodal verifier
```

这种上线顺序符合“precision 优先，允许漏匹配”。

---

## 20. 一份可直接实现的 Decision Engine 伪代码

```python
from dataclasses import dataclass
from enum import Enum

class Status(str, Enum):
    VERIFIED = "VERIFIED"
    CONFLICT = "CONFLICT"
    ABSTAIN = "ABSTAIN"
    NO_REFERENCE = "NO_REFERENCE"

@dataclass
class Evidence:
    raw: str
    canonical: str | None
    role: str
    kind: str
    direct: bool
    score: float
    source_rank: int

TRUSTED_KINDS = {
    "STRUCTURED_REFERENCE",
    "TEXT_DIRECT_REFERENCE",
    "OCR_CARD_REFERENCE",
    "OCR_BOX_REFERENCE",
}

BLOCKING_ROLES = {
    "PLATFORM_SKU",
    "SELLER_SKU",
    "SERIAL_NUMBER",
    "COMPATIBLE_REFERENCE",
}

def decide_reference(brand_id: str, evidences: list[Evidence]):
    if not evidences:
        return Status.NO_REFERENCE, None, "NO_EVIDENCE"

    ref_evidence = [
        e for e in evidences
        if e.canonical
        and e.role == "BRAND_REFERENCE"
        and e.kind in TRUSTED_KINDS
    ]

    if not ref_evidence:
        return Status.ABSTAIN, None, "NO_TRUSTED_REFERENCE_EVIDENCE"

    values = {e.canonical for e in ref_evidence}

    if len(values) > 1:
        return Status.CONFLICT, None, "MULTIPLE_TRUSTED_REFERENCES"

    candidate = next(iter(values))

    if any(
        e.canonical == candidate and e.role in BLOCKING_ROLES
        for e in evidences
    ):
        return Status.ABSTAIN, None, "REFERENCE_ROLE_CONFLICT"

    independent_kinds = {e.kind for e in ref_evidence if e.canonical == candidate}

    if "STRUCTURED_REFERENCE" in independent_kinds:
        return Status.VERIFIED, candidate, "STRUCTURED_DIRECT_NO_CONFLICT"

    if len(independent_kinds) >= 2:
        return Status.VERIFIED, candidate, "MULTI_EVIDENCE_CORROBORATION"

    return Status.ABSTAIN, None, "INSUFFICIENT_INDEPENDENT_SUPPORT"
```

上线初期宁可让 `TEXT_DIRECT_REFERENCE` 单证据仍然 `ABSTAIN`，等黄金集证明某品牌/来源的 precision 足够高后再逐步放宽。

---

## 21. 最终推荐架构

综合论文与当前 Spec，建议最终架构定为：

```text
                  ┌──────────────────────┐
                  │ Raw Multi-source Data│
                  └──────────┬───────────┘
                             v
                  ┌──────────────────────┐
                  │ Brand Normalization  │
                  └──────────┬───────────┘
                             v
                  ┌──────────────────────┐
                  │ Cheap Ref Extraction │
                  │ field + text spans   │
                  └──────────┬───────────┘
                             v
                  ┌──────────────────────┐
                  │ Is unresolved?       │
                  └──────┬────────┬──────┘
                         │ no     │ yes
                         │        v
                         │  ┌─────────────────┐
                         │  │ Image Gate + OCR│
                         │  └────────┬────────┘
                         │           v
                         │  ┌─────────────────┐
                         │  │ MXT-style Multi-│
                         │  │ modal Verifier  │
                         │  └────────┬────────┘
                         └───────────┤
                                     v
                         ┌─────────────────────┐
                         │ Role + Canonicalize │
                         └──────────┬──────────┘
                                    v
                         ┌─────────────────────┐
                         │ Hard Conflict Check │
                         └──────────┬──────────┘
                                    v
                         ┌─────────────────────┐
                         │ VERIFIED / ABSTAIN  │
                         │ / CONFLICT          │
                         └──────────┬──────────┘
                                    v
                 ┌────────────────────────────────┐
                 │ (brand_id, canonical_reference)│
                 └───────────────┬────────────────┘
                                 v
                         Exact Join / Group
```

这个方案的关键思想可以用一句话概括：

> **用多模态模型扩大“可找到 reference 证据”的覆盖率，用规则、角色识别、规范化和拒识机制保证 reference 证据不会被模型相似度越权。**

---

## 22. 最值得直接采纳的 7 个点

1. **Q&A / prompt 化共享模型**：brand/source/attribute 作为条件，一个模型支持大量组合。
2. **Attribute-aware fusion**：图片不是全局相似度，而是围绕 `reference_number` 这一属性选择视觉证据。
3. **Distant supervision**：用现有独立 reference 字段自动构造训练集，减少人工标注。
4. **多模态只跑 unresolved tail**：先字段/文本，再 OCR，再重模型。
5. **reference 从“生成问题”改成“候选验证问题”**：避免 hallucination 直接进入实体键。
6. **按 evidence path 校准 precision**：不同品牌、来源、字段路径分别设自动放行资格。
7. **一旦 VERIFIED 就 exact join**：不要继续让语义/视觉模型参与最终实体判定。

---

## 23. 最终判断

### 适合直接借鉴

```text
多 PT / 多 attribute 的共享模型思路
MAG / attribute-aware visual fusion 思路
Distant supervision
高 precision 生产阈值意识
分属性/分任务监控
大规模生产部署的模型维护思路
```

### 不应直接照搬

```text
让 T5 自由生成 reference
把 zero-shot reference 当可信身份字段
只靠 greedy decoder 输出做实体键
用普通 tokenizer 直接处理所有品牌 identifier
把图片语义相似度当成 reference 身份证据
只用 recall@90P 作为当前业务验收标准
```

### 建议落地结论

当前跨源二奢/腕表实体匹配系统最稳妥的设计不是“训练一个更强的商品匹配模型”，而是：

```text
Reference Extraction
    +
Reference Role Classification
    +
Brand-aware Canonicalization
    +
Multimodal Evidence Verification
    +
Hard Conflict Veto
    +
Abstention
    +
Exact Entity Key Join
```

MXT 最适合成为这里的 **Multimodal Evidence Extraction/Verification 层**，帮助系统从稀疏文本和图片中找到 reference 与辅助属性；最终自动 MATCH 仍然只由经过验证的 canonical reference 硬证据触发。

这既保留了论文在大规模、多模态、弱监督和长尾属性上的优势，又满足当前 Spec “绝对不能误匹配、允许漏匹配”的核心约束。
