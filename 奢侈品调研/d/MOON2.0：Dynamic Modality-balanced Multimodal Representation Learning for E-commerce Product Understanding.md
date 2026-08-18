# MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择 **MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding**（Nie et al., CVPR 2026）进行深入分析。

- CVPR 论文页：<https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html>
- arXiv v3（2026-07-29）：<https://arxiv.org/abs/2511.12449>
- arXiv HTML：<https://arxiv.org/html/2511.12449>
- MBE2.0 数据集：<https://huggingface.co/datasets/ZHNie/MBE2.0>
- 当前需求：<https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31>

本轮分析前已先检查 `奢侈品调研/d/`，已存在的分析包括 Ameli、DeepBlocker、Deep Entity Matching、TransClean、GraLMatch、PAM、pyJedAI、选择性预测/置信度控制、LLM 属性抽取等，**MOON2.0 尚未被 d 分析过**，因此满足“不重复分析”的要求。

### 对当前需求最重要的判断

MOON2.0 **不能直接拿来当跨源腕表实体匹配器**。

原因很简单：论文优化的是通用电商的多模态表示与检索能力，而当前 Spec 对“同一个商品”的定义已经非常严格：

> **同一个商品 = 同一个 reference number / 型号。**

并且业务要求：

> **绝对不能误匹配，precision 优先到极致，允许漏匹配。**

因此，如果直接采用 MOON2.0 式 embedding similarity：

```text
sim(item_a, item_b) > threshold => MATCH
```

会在腕表场景遇到一个根本问题：

```text
同品牌 + 同系列 + 同尺寸 + 同材质 + 外观高度相似
≠
同一个 reference
```

例如同系列相邻 reference 可能只差后缀、表盘颜色、材质、机芯或市场版本，但图片和标题 embedding 非常接近。对普通检索来说这不一定是灾难；对当前“不能误合并”的实体解析来说却是致命错误。

### 真正值得迁移的 MOON2.0 思路

MOON2.0 最值得迁移的不是“一个更强的多模态相似度模型”，而是四个架构思想：

1. **Multimodal Joint Learning**：同一套模型同时学 text-only、image-only、multimodal，适应字段缺失；
2. **Modality-driven MoE**：根据输入实际拥有的模态与证据类型动态路由，而不是固定把文本和图片按一个权重融合；
3. **Dual-level Alignment**：同时学习“商品之间的关系”和“单个商品内部图文是否一致”；
4. **Dynamic Sample Filtering**：对噪声训练样本动态降权，而不是把所有弱标签都当真。

对当前需求，我建议把它们改造成一个 **Reference-First Multimodal Evidence Router**：

```text
三源原始商品
   │
   ├─> Source Adapter / 字段统一
   │
   ├─> 品牌规范化
   │
   ├─> reference 候选抽取
   │      ├─ structured field
   │      ├─ title / description
   │      ├─ OCR（表背 / 保卡 / 吊牌 / 盒证）
   │      └─ catalog / reference dictionary
   │
   ├─> 编号角色识别
   │      ├─ official reference
   │      ├─ platform SKU
   │      ├─ seller SKU
   │      ├─ serial number
   │      └─ accessory compatibility reference
   │
   ├─> brand-aware canonicalization
   │
   ├─> Reference Resolver
   │      ├─ provenance-aware evidence
   │      ├─ text / OCR / image consistency
   │      ├─ hard conflict veto
   │      └─ VERIFIED / CONFLICT / ABSTAIN
   │
   ├─> entity key = (canonical_brand, canonical_reference)
   │
   └─> exact join / incremental aggregation
```

最终自动匹配规则仍然应该非常保守：

```text
same canonical_brand
AND item_a.resolved_reference.status == VERIFIED
AND item_b.resolved_reference.status == VERIFIED
AND item_a.canonical_reference == item_b.canonical_reference
AND no trusted conflicting_reference
=> MATCH

trusted verified references are different
=> NO_MATCH

otherwise
=> ABSTAIN / CONFLICT
```

**图片、文本 embedding、MLLM 都不能越权把两个 reference 不同或未知的商品自动合并。**

本报告给出的直接落地方案，是用 MOON2.0 的技术思路提升“reference 能否被可靠解析、跨模态证据能否彼此验证、困难样本能否拒识”的能力，而不是把它变成最终身份裁决器。

---

## 1. 当前 Spec 的任务本质

Notion 需求的关键约束是：

- 数据来源：雷小安、腕表之家、奢当家；
- 数据规模：100 万–1000 万级；
- 需要持续增量；
- 字段高度稀疏；
- reference 有时是独立字段，有时埋在标题；
- 图片可用；
- 允许人工标注几百对；
- precision 极端优先；
- identity definition 已经明确：同 reference 才是同一商品。

这意味着问题表面上叫“跨源商品实体匹配”，实际上最好拆成两个问题：

### 问题 A：Reference Resolution

对每条商品记录回答：

```text
这条记录真正描述的官方 reference 是什么？
```

### 问题 B：Exact Entity Aggregation

如果 A 已经可靠解决，则跨源匹配变成：

```text
(canonical_brand, canonical_reference) exact equality
```

真正困难的是 A，而不是 B。

这也是为什么 MOON2.0 对当前系统有价值：它可以帮助提升**字段稀疏情况下的 reference 证据理解与一致性检查**。

---

## 2. MOON2.0 原论文解决什么问题

MOON2.0 是面向电商商品理解的通用多模态表示学习模型。论文认为传统 dual-flow 架构存在几个问题：

1. 图片和文本分别编码，难建模一个商品对应多图、多文本的 many-to-one 关系；
2. 固定比例混合 image-only / text-only / multimodal 训练会产生 modality imbalance；
3. 大多数训练只建模商品之间关系，没有充分利用一个商品内部图文天然一致性；
4. 电商原始文本和图片噪声很大。

MOON2.0 的主干可以概括为：

```text
                  ┌─────────────────────────┐
raw e-commerce -->│ Image-text Co-augmentation│
                  └────────────┬────────────┘
                               │
                     query / pos / neg
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
      text-only            image-only          multimodal
          │                    │                    │
          └────────────────────┼────────────────────┘
                               │
                    Generative MLLM backbone
                               │
                      Modality-driven MoE
                               │
                     unified representation
                               │
              ┌────────────────┴────────────────┐
              │                                 │
       Inter-product Align               Intra-product Align
              │                                 │
              └────────────────┬────────────────┘
                               │
                    Dynamic Sample Filtering
                               │
                         joint optimization
```

论文进一步构建了 MBE2.0 benchmark：

- 训练样本：5,751,594；
- 测试样本：636,241；
- 总量约 640 万真实电商样本；
- 数据来自淘宝搜索日志；
- 正样本主要来自 query 后购买；
- 低相关曝光构造 negative。

训练设置中，论文报告：

- single-stage supervised finetuning；
- learning rate = `1e-5`；
- cosine scheduler；
- 64 × NVIDIA A100；
- batch size = 4 / GPU；
- 约 18 小时。

这说明原版 MOON2.0 是一个**大规模通用电商表征训练方案**，不是轻量级 entity matching recipe。

---

## 3. MOON2.0 技术实现深入拆解

## 3.1 三种输入形态统一编码

论文对 query / positive / negative 都构造三种输入：

```text
x^t   = text-only
x^i   = image-only
x^mm  = image + text
```

文本包括：

```text
原始 title t
增强 title t_tilde
```

图片包括：

```text
原图 i
增强图 i_tilde_1 ... i_tilde_n
```

视觉编码器先把图片变成 visual tokens，再通过 projector 映射到 LLM hidden dimension；文本 token 和 visual token 一起进入 LLM。

如果：

- 文本 token 长度为 `L`；
- hidden dimension 为 `D`；
- 每张图视觉 token 数为 `V`；
- 有 `n_c` 张增强图；

则最终最后一层 hidden states 约为：

```text
h ∈ R^((2L + (n_c + 1)V) × D)
```

论文直接对最后一层 hidden states 做 mean pooling 得到：

```text
r ∈ R^D
```

作为商品 representation。

### 对当前系统的启发

腕表数据恰好也是“模态不完整”的：

```text
记录 A：有 reference 字段，无好图
记录 B：标题有 reference，无结构字段
记录 C：标题没 reference，但保卡图片有
记录 D：只有商品图和模糊标题
```

因此不应该假设所有记录都能走同一个固定 feature pipeline。

但当前系统并不需要照搬整个 generative MLLM。可以先把“三种输入统一建模”的思想保留，做一个更便宜的 MOON-Lite：

```text
Text Encoder    -> z_text
Image Encoder   -> z_image
OCR Encoder     -> z_ocr
Structured Ref  -> z_ref / symbolic evidence

modality mask + evidence metadata
        │
        v
Evidence Router / Gated Fusion
        │
        v
z_multimodal
```

关键不是模型一定多大，而是**同一语义空间必须显式支持缺图、缺文、缺 reference 等不同输入形态**。

---

## 3.2 Modality-driven Mixture-of-Experts

MOON2.0 把 MoE 放到 LLM 的 FFN 层。

传统 token-level MoE 对 hidden state `h` 计算 gate：

```text
G = softmax(W_g h)
```

每个 expert 是一个 MLP `f_z`，最终输出：

```text
h_hat = Σ_z G_tilde_z * f_z(h)
```

普通 MoE 的问题是：它只看到 token 激活，不知道当前训练目标是：

```text
text -> multimodal
image -> multimodal
multimodal -> multimodal
image -> text
text -> image
```

MOON2.0 因此引入一个可学习的 **Dual-alignment Matrix**：

```text
W* ∈ R^(Z × M)
```

其中：

- `Z` = experts 数量；
- `M` = alignment objectives 数量。

矩阵中的 `W*_(z,m)` 表示 expert z 对 objective m 的偏好。

对每个 expert 在所有目标上的偏好做 softmax：

```text
p(z,m) = exp(W*_(z,m)) / Σ_k exp(W*_(z,k))
```

再把 token routing weight 和 expert objective preference 合起来，得到 objective-specific weight `ω_m`。

此外论文还增加：

```text
L_aux       = expert load balancing
L_sparsity  = expert-objective preference entropy minimization
```

其中 sparsity loss 的目的，是让一个 expert 不要什么都做，而是对少数 alignment objective 形成专长。

### 为什么这个设计适合迁移到三源腕表

当前三源不是简单的“图片 vs 文本”两类信号，而是：

```text
structured_reference
reference-in-title
reference-in-description
OCR-on-caseback
OCR-on-card
brand / collection
visual appearance
source-specific schema
```

这些信号可靠性差异极大。

因此真正应该学习的是：

> **在当前记录的证据组成下，应该相信哪类 expert？**

例如：

```text
记录 1：腕表之家官方 reference 字段存在
=> reference expert 权重最高

记录 2：标题无 reference，保卡 OCR 清晰
=> OCR identifier expert 权重提高

记录 3：标题里出现多个型号，且含“适用/兼容”词
=> role-classification expert 必须优先，视觉不能直接补票

记录 4：只有外观图
=> visual expert 可做候选召回，但不能自动 VERIFIED
```

这比固定：

```text
0.5 * text_similarity + 0.5 * image_similarity
```

更符合真实脏数据。

---

## 3.3 Dual-level Alignment

MOON2.0 的第二个核心是把 alignment 拆成两层。

### 3.3.1 Inter-product Alignment

输入 triplet：

```text
(q, p, n)
```

其中：

- q = query；
- p = positive item；
- n = negative item。

query 可以是：

```text
q_text
q_image
q_multimodal
```

positive / negative 使用 multimodal representation。

它本质是一个 InfoNCE / contrastive objective：

```text
sim(q, p) >> sim(q, n)
```

论文分别对 text→mm、image→mm、mm→mm 建 loss，然后由 modality-driven expert weights 联合。

### 对腕表需求的改造

当前正负例应该严格定义为：

```text
positive:
  same canonical_brand
  same VERIFIED canonical_reference
  preferably cross-source

hard negative:
  same canonical_brand
  same collection / family
  visually similar
  textually similar
  BUT different VERIFIED canonical_reference
```

尤其要大量构造：

```text
Rolex 126610LN vs 126610LV
Omega 同系列相邻 reference
Cartier 同系列不同尺寸/材质 reference
```

这类 hard negative 才真正对应业务风险。

如果训练数据只随机采不同商品作为 negative，模型很快就学会“品牌不同 = negative”，对最危险的 false positive 没帮助。

---

## 3.4 Intra-product Alignment

MOON2.0 还显式要求：

```text
同一个商品的 image representation
应该接近
同一个商品的 text representation
```

并远离其他商品文本。

论文把 positive / negative item 内部的 image-text pairing 也纳入对比学习。

### 对腕表需求的价值更大

当前可以把 intra-product alignment 改成**证据一致性训练**：

```text
title reference context
      ↕
structured reference
      ↕
OCR reference context
      ↕
image-local engraving / dial / card crop
```

目标不是让“整张表的外观”贴近标题，而是让：

> **reference 相关局部视觉证据、OCR token 和 reference 上下文相互支持。**

尤其适合识别：

- 标题写错型号；
- 描述复制了别的商品；
- 图片上传错；
- 标题里出现的是兼容型号而非当前商品；
- 平台货号被错误当作官方 reference。

因此在当前系统中，Intra-product Alignment 最适合成为**冲突发现器**，而不是单纯 embedding 优化器。

---

## 3.5 Image-text Co-augmentation

MOON2.0 使用 MLLM 做两类增强。

### Textual Enrichment

根据：

```text
title T
product description D
image I
extracted entities E
```

生成 enriched title：

```text
T+ = MLLM_text(T, I, E)
```

目的：补足电商标题语义。

### Visual Expansion

论文先：

- 提取主体；
- 去掉无关内容；

再做多粒度增强：

- 背景变化；
- 视角变化；
- logo / fine-grained detail refinement。

最后用 CLIP 做 image-title consistency filter。

### 这部分不能原样用于腕表 reference 解析

这是本轮非常重要的反向结论。

对普通商品 representation，生成式视觉增强可能有效；但对腕表 reference 身份识别，**生成模型修改 logo、表盘细节、刻字、数字、后盖细节属于高风险操作**。

因为当前系统恰恰把这些局部细节作为身份辅助证据。

所以不能让生成模型：

```text
“补清晰”一个模糊的 reference
“修复”一段看不清的后盖刻字
“优化”logo / dial details
```

然后再把生成后的内容当成事实证据。

这会产生 hallucinated identity evidence。

### 当前系统允许的安全增强

只建议使用**非生成式、证据保持型变换**：

```text
crop
resize
rotation correction
perspective correction
contrast adjustment
sharpening with raw image retained
OCR-specific binarization
caseback/card/dial region detection
multi-crop
```

所有增强图都必须保留：

```text
parent_raw_image_id
transform_type
transform_params
```

并且自动决策中，OCR 原始证据仍要能追溯到 raw image。

---

## 3.6 Dynamic Sample Filtering

MOON2.0 注意到电商日志的 triplet 有 pseudo-positive / pseudo-negative 噪声。

论文对每个 triplet 根据当前 embedding margin 计算可靠度：

```text
margin = sim(q, p) - sim(q, n)

phi = sigmoid(kappa * (margin - delta_bar))
```

其中：

- `delta_bar` 随训练衰减；
- reliability threshold 固定为 `0.6`；
- 低可靠 triplet 在训练中降权。

直觉是：

```text
训练早期：先学非常确定的样本
训练后期：逐步吸收更难的样本
```

这对当前几百万到千万级数据非常适合，因为可以用高精度规则构造大量 silver labels，再让模型逐渐学习困难样本。

但必须强调：

> **Dynamic Sample Filtering 只能用于训练数据去噪，不能直接等价为线上 MATCH confidence。**

线上身份裁决仍需 reference 硬证据。

---

## 4. MOON2.0 实验结果真正说明了什么

论文在 MBE2.0 的主要 retrieval R@10 报告为：

```text
text -> multimodal        63.09
image -> multimodal       91.08
multimodal -> multimodal  94.21
text -> image             73.12
image -> text             64.91
```

product classification accuracy：

```text
68.08
```

attribute prediction accuracy：

```text
84.29
```

消融结果很有信息量：

```text
Full MOON2.0
  t->mm R@10      63.09
  i->mm R@10      91.08
  mm->mm R@10     94.21
  t->i R@10       73.12
  i->t R@10       64.91
  class acc       68.08
  attr acc        84.29

w/o MoE
  51.29 / 74.59 / 78.45 / 62.16 / 56.21 / 62.55 / 75.62

w/o Dual-level Alignment
  37.99 / 65.72 / 67.45 / 31.45 / 23.35 / 57.12 / 67.24

w/o Co-augmentation
  59.69 / 78.17 / 80.62 / 64.79 / 58.68 / 66.21 / 77.77

w/o Filtering
  60.63 / 83.40 / 80.00 / 70.40 / 63.21 / 67.99 / 84.04
```

最值得关注的是：

> **Dual-level Alignment 的影响最大。**

这进一步支持当前系统把“跨商品同 reference”与“单商品内部图文/reference 证据一致性”同时建模。

但这些结果也同时说明：

> MOON2.0 是一个强 representation model，不是一个可以保证 zero false merge 的 matcher。

即使 `mm -> mm R@10 = 94.21` 很高，也意味着它仍然会把真正对象排到 top-10 之外；而当前业务要的不是“候选召回不错”，而是“自动合并绝不能错”。

所以它最适合：

```text
candidate generation
hard-negative mining
conflict detection
reference evidence ranking
manual review assistance
```

不适合：

```text
final identity decision by embedding argmax
```

---

## 5. 当前需求的直接落地架构：Reference-First MOON-ER

建议将整个系统拆成 7 层。

## 5.1 Layer 1：Source Adapter

三个平台先统一成一个 normalized product schema。

```text
NormalizedProduct
  product_id
  source
  source_item_id
  source_url
  title_raw
  description_raw
  brand_raw
  model_raw
  reference_raw
  attributes_raw
  images[]
  crawl_time
  source_update_time
  content_hash
  parser_version
```

要求：

- 原始字段永远保留；
- normalized 字段不能覆盖 raw；
- 每次解析带 `parser_version`；
- 增量处理必须 idempotent。

---

## 5.2 Layer 2：Brand Resolution

不要在全库直接做 reference 比较。

先把品牌统一：

```text
Rolex / 劳力士 / ROLEX
=> brand_id = rolex

Omega / 欧米茄 / OMEGA
=> brand_id = omega
```

数据表：

```text
brand_alias
  alias
  brand_id
  source
  confidence
  status
```

如果两个商品的 VERIFIED brand 不同：

```text
=> NO_MATCH
```

这是最便宜、最安全的第一层 blocking。

---

## 5.3 Layer 3：Reference Candidate Extraction

每条记录不要只产出一个 reference，而是产出**候选集合 + provenance**。

```text
ReferenceCandidate
  product_id
  raw_value
  canonical_value
  role
  provenance
  source_field
  span_start
  span_end
  image_id
  bbox
  extractor
  extractor_version
  confidence
  catalog_hit
  created_at
```

### provenance 至少分成

```text
STRUCTURED_FIELD
TITLE
DESCRIPTION
OCR_CASEBACK
OCR_CARD
OCR_TAG
OCR_BOX
CATALOG_MATCH
MLLM_EXTRACTED
```

### role 至少分成

```text
OFFICIAL_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
ACCESSORY_COMPATIBILITY_REFERENCE
UNKNOWN_IDENTIFIER
```

这一步非常重要，因为二奢标题里经常有很多“像型号”的字符串，但它们可能不是当前商品 identity。

---

## 5.4 Layer 4：Brand-aware Reference Canonicalization

不要使用一个全局粗暴 regex 直接：

```text
remove all punctuation
remove all suffixes
```

因为某些品牌 reference 的：

- `/`；
- `.`；
- `-`；
- 字母后缀；
- 材质/盘面后缀；

可能具有身份含义。

建议分两级 canonicalization。

### Level A：Lossless normalization

只做几乎不会改变语义的变换：

```text
Unicode NFKC
uppercase
trim
collapse spaces
normalize full-width punctuation
normalize common OCR confusion only when rule is brand-safe
```

### Level B：Brand-specific canonicalization

每个品牌配置：

```text
separator rules
allowed charset
reference regex families
prefix aliases
suffix semantics
known catalog
OCR confusion map
```

输出：

```text
canonical_reference
canonicalizer_version
```

任何规则升级都能通过版本号重放。

---

## 5.5 Layer 5：Reference Resolver

这是整个系统的核心。

Reference Resolver 不应该输出一个模糊概率，而应该输出带状态的结论：

```text
VERIFIED
CONFLICT
ABSTAIN
INVALID
```

建议引入 evidence ledger：

```text
ReferenceEvidence
  product_id
  canonical_reference
  evidence_type
  evidence_source
  evidence_strength
  raw_pointer
  trusted
  conflict_group
```

### 可以 VERIFIED 的典型条件

示例规则，不建议一开始只依赖 learned score：

#### Rule A：可信结构字段

```text
source-specific authoritative reference field
AND role == OFFICIAL_REFERENCE
AND passes brand grammar/catalog validation
=> VERIFIED
```

#### Rule B：两个独立文本来源一致

```text
TITLE exact candidate
AND DESCRIPTION / structured model field same canonical reference
AND no trusted conflict
=> VERIFIED
```

#### Rule C：文本 + OCR 独立一致

```text
TITLE candidate == OCR_CARD / OCR_CASEBACK candidate
AND OCR confidence high
AND role classifier says official reference
AND no conflict
=> VERIFIED
```

### 必须 CONFLICT 的情况

```text
trusted structured reference = A
trusted OCR/card reference = B
A != B
=> CONFLICT
```

```text
multiple high-trust official references on same product
=> CONFLICT
```

### 必须 ABSTAIN 的情况

```text
only visual similarity
=> ABSTAIN
```

```text
only one weak title candidate
=> ABSTAIN
```

```text
same series but reference missing
=> ABSTAIN
```

```text
MLLM guessed reference without deterministic evidence
=> ABSTAIN
```

这就是 precision-first 的核心。

---

## 5.6 Layer 6：MOON-Lite Multimodal Evidence Router

这里才引入 MOON2.0 的核心思想。

不建议 P0 就做完整 LLM FFN MoE。先做一个更容易落地、容易 debug 的 evidence-aware routing 模型。

### Expert 设计

```text
Expert 1: Reference Context Expert
  input: title / description 中 reference 周边上下文

Expert 2: OCR Identifier Expert
  input: OCR token + bbox + OCR confidence + crop embedding

Expert 3: Product Text Expert
  input: full title + normalized attributes

Expert 4: Visual Product Expert
  input: dial / case / bracelet / caseback image embeddings

Expert 5: Source Schema Expert
  input: source id + field provenance + known source reliability
```

### Gate 输入

```text
has_structured_ref
has_title_ref
has_description_ref
has_ocr_ref
ocr_confidence
num_images
source_id
brand_id
reference_catalog_hit
reference_role
modality_mask
```

Gate 输出：

```text
w_ref_context
w_ocr
w_text
w_visual
w_source
```

但这些权重只能用于：

```text
candidate ranking
conflict score
hard-negative mining
manual-review prioritization
```

不能越过 Reference Resolver 的硬规则直接输出 MATCH。

### 如果后续确实需要升级为 Sparse MoE

可以按 MOON2.0 做：

```text
LLM/VLM FFN
  -> top-k sparse experts
  -> objective preference matrix
```

并让 experts 对这些 objective 专门化：

```text
text_ref -> canonical_ref
ocr_ref -> canonical_ref
image -> text
text -> image
item -> same-reference-item
item -> hard-different-reference-item
```

但这应该是 P2/P3，不是系统第一版依赖项。

---

## 5.7 Layer 7：Deterministic Match Decision

最终 entity key：

```text
entity_key = (brand_id, canonical_reference)
```

推荐决策函数：

```python
def decide_match(a, b):
    if a.brand_status == "VERIFIED" and b.brand_status == "VERIFIED":
        if a.brand_id != b.brand_id:
            return "NO_MATCH"

    if a.ref_status == "CONFLICT" or b.ref_status == "CONFLICT":
        return "CONFLICT"

    if a.ref_status == "VERIFIED" and b.ref_status == "VERIFIED":
        if a.canonical_reference == b.canonical_reference:
            return "MATCH"
        return "NO_MATCH"

    return "ABSTAIN"
```

这个函数看起来“没有 AI 味”，但对于当前业务定义，它恰恰是正确的最后一层。

AI 的价值全部放在前面：

> **尽可能把更多商品从 UNKNOWN/ABSTAIN 推到可靠 VERIFIED，同时保持 reference 解析 precision。**

---

## 6. 为什么要做 Reference Resolver，而不是直接 pairwise matching

假设库里有：

```text
A = 雷小安记录
B = 腕表之家记录
C = 奢当家记录
```

普通 entity matching 会算：

```text
P(A == B)
P(A == C)
P(B == C)
```

但当前 identity definition 已经给了一个更好的 latent key：

```text
reference(A)
reference(B)
reference(C)
```

如果能把 reference 解析可靠，则：

```text
A -> rolex:126610LN
B -> rolex:126610LN
C -> rolex:126610LN
```

不需要再维护三条 pairwise edge；直接：

```text
entity_id = hash("rolex|126610LN")
```

百万到千万级下，这个差别非常大。

### Pairwise 的问题

三源 N 条记录若全比较，理论空间是：

```text
O(N^2)
```

即使 blocking 后，也会持续产生边和 cluster consistency 问题。

### Reference-key 的问题规模

如果每条商品只做一次 reference resolution：

```text
O(N)
```

最终 join：

```text
hash / B-tree lookup
```

增量商品到来时：

```text
resolve reference
-> exact lookup entity_key
-> attach
```

这更符合持续增量。

---

## 7. MOON2.0 Dual-level Alignment 如何改造成腕表训练目标

建议构建一个 `MBE-Watch` 式内部训练集。

## 7.1 高精度 silver positives

不用先人工标几百万对。

可以从严格规则中自动生成高精度正样本：

```text
same verified brand
same verified canonical_reference
cross-source
no internal conflict
```

例如：

```text
雷小安 item 123
腕表之家 item 987
均通过独立高置信证据解析为 126610LN
=> silver positive
```

## 7.2 Hard negatives

负样本必须专门针对误匹配风险：

```text
same brand
same collection/family
high title similarity
high image similarity
different VERIFIED reference
```

也可以构造：

```text
same visible dial style
same material
same size
different suffix/reference
```

这些 hard negatives 比随机负样本重要得多。

## 7.3 Inter-product objective

可以直接复用 MOON2.0 思路：

```text
L_inter_text
L_inter_image
L_inter_multimodal
```

目标：

```text
same-reference cross-source item closer
nearby-reference hard negative farther
```

## 7.4 Intra-product objective

当前应强化：

```text
title reference context <-> OCR crop
structured reference   <-> title context
OCR token              <-> visual region
brand/model text        <-> overall image
```

如果一条商品内部的图文 reference 证据明显对不上，应让 model 输出高 conflict signal。

## 7.5 增加 Reference-aware Margin Loss

可以额外引入：

```text
L_ref_margin = max(0, m + sim(q, n_hard) - sim(q, p))
```

其中 `n_hard` 必须是：

```text
same family, different reference
```

这样模型训练目标直接针对 false merge 风险。

---

## 8. 几百对人工黄金标签应该怎么花

用户允许标注“几百对”，这非常宝贵，不应该随机抽。

建议分层标注。

## 8.1 40%：最危险 hard negatives

重点找：

```text
高图像相似
高标题相似
同系列
reference 相邻
```

这些是评估 precision 的核心。

## 8.2 25%：reference role ambiguity

例如：

```text
标题同时出现官方 reference + 店铺 SKU
“适用 126610” 的配件
盒证单卖
表带/配件引用主表 reference
serial number 被误判为 reference
```

## 8.3 20%：OCR-only / weak-text

验证：

- 表背；
- 保卡；
- 标签；
- 盒贴；

OCR 是否能可靠补 reference。

## 8.4 15%：跨源 schema corner cases

每个平台都要覆盖：

```text
字段语义不一致
脏值
旧字段
新字段
来源自有 ID
```

人工标签的目标不是覆盖平均样本，而是**集中测系统最容易误合并的位置**。

---

## 9. Dynamic Sample Filtering 在当前项目中的最佳用法

如果我们用规则生成 10 万 / 100 万 silver training triplets，规则仍会有少量错误。

可以把 MOON2.0 的动态过滤思路迁移为三层样本权重。

### Tier A：Gold / deterministic

```text
人工黄金标签
可信结构 reference exact
多独立证据一致
```

权重最高。

### Tier B：Silver high-confidence

```text
title + catalog
OCR + title
cross-source strict exact
```

正常权重。

### Tier C：Weak

```text
MLLM-only candidate
fuzzy canonicalization
single low-confidence OCR
```

低权重或仅用于 hard-negative mining。

模型训练中再计算 embedding margin，动态降权可疑 triplet。

但建议加入一条业务约束：

> 如果弱标签和 VERIFIED reference 冲突，必须直接丢弃，而不是让模型“自己学”。

---

## 10. 图片在当前系统中的正确角色

### 图片适合做

1. 找到表背、保卡、盒贴上的 reference；
2. 给 OCR 提供局部 crop；
3. 判断标题/图片是否明显错配；
4. 在 reference 缺失时召回可能的同系列候选；
5. 为人工审核排序；
6. hard-negative mining。

### 图片不适合做

```text
image similarity > 0.95
=> same reference
```

原因：

- 同系列不同 reference 外观极近；
- 同一 reference 不同拍摄条件差异很大；
- 卖家会使用官图；
- 图片可能复用；
- 配件/盒证会引用主表；
- 生成或修图可能改变细节。

### 强制原则

```text
Visual evidence can support.
Visual evidence cannot override a trusted reference conflict.
Visual evidence alone cannot create an automatic MATCH.
```

---

## 11. 推荐的数据表设计

## 11.1 product_record

```sql
CREATE TABLE product_record (
    product_id           BIGINT PRIMARY KEY,
    source               VARCHAR(32) NOT NULL,
    source_item_id       VARCHAR(128) NOT NULL,
    source_url           TEXT,
    title_raw            TEXT,
    description_raw      TEXT,
    brand_raw            TEXT,
    reference_raw        TEXT,
    attributes_json      JSONB,
    content_hash         VARCHAR(64),
    source_update_time   TIMESTAMP,
    crawl_time           TIMESTAMP,
    parser_version       VARCHAR(32),
    UNIQUE(source, source_item_id)
);
```

## 11.2 reference_candidate

```sql
CREATE TABLE reference_candidate (
    candidate_id         BIGSERIAL PRIMARY KEY,
    product_id           BIGINT NOT NULL,
    raw_value            TEXT NOT NULL,
    canonical_value      TEXT,
    role                 VARCHAR(64),
    provenance           VARCHAR(64) NOT NULL,
    source_field         VARCHAR(128),
    image_id             BIGINT,
    bbox_json            JSONB,
    extractor            VARCHAR(64),
    extractor_version    VARCHAR(32),
    confidence           DOUBLE PRECISION,
    catalog_hit          BOOLEAN DEFAULT FALSE,
    created_at           TIMESTAMP DEFAULT NOW()
);
```

## 11.3 reference_resolution

```sql
CREATE TABLE reference_resolution (
    product_id            BIGINT PRIMARY KEY,
    brand_id              VARCHAR(64),
    canonical_reference   VARCHAR(128),
    status                VARCHAR(16) NOT NULL,
    rule_id               VARCHAR(64),
    resolver_version      VARCHAR(32),
    evidence_json         JSONB,
    resolved_at           TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_verified_reference
ON reference_resolution(brand_id, canonical_reference)
WHERE status = 'VERIFIED';
```

## 11.4 match_audit

```sql
CREATE TABLE match_audit (
    audit_id              BIGSERIAL PRIMARY KEY,
    product_a             BIGINT NOT NULL,
    product_b             BIGINT NOT NULL,
    decision              VARCHAR(16) NOT NULL,
    reason_code           VARCHAR(64) NOT NULL,
    resolver_version      VARCHAR(32),
    model_version         VARCHAR(32),
    evidence_snapshot     JSONB,
    created_at            TIMESTAMP DEFAULT NOW()
);
```

审计表非常重要，因为之后如果发现某个 canonicalization rule 错了，需要知道哪些历史 merge 受影响。

---

## 12. 100 万–1000 万级的工程架构

推荐把高成本模型从主路径中隔离。

```text
                  ┌────────────────────────┐
Crawler / Import ->│ Raw Product Storage    │
                  └──────────┬─────────────┘
                             │
                             v
                    Source Normalizer
                             │
                             v
                Brand + Ref Fast Extractor
                             │
                 ┌───────────┴───────────┐
                 │                       │
          high-confidence            unresolved
                 │                       │
                 v                       v
         Reference Resolver      OCR / MLLM / MOON-Lite
                 │                       │
                 └───────────┬───────────┘
                             v
                 VERIFIED / CONFLICT / ABSTAIN
                             │
             ┌───────────────┴────────────────┐
             │                                │
         VERIFIED                        unresolved
             │                                │
             v                                v
  exact entity-key lookup                review queue
             │
             v
     cross-source aggregation
```

### 存储建议

```text
关系型元数据：PostgreSQL
批量分析/审计：ClickHouse 可选
图片：OSS / S3 compatible object storage
文本/模糊 reference 检索：OpenSearch / Elasticsearch 可选
向量候选：Milvus / Qdrant / FAISS service 可选
```

但注意：

> **最终 VERIFIED reference exact join 根本不需要向量数据库。**

向量索引只服务于 unresolved / review / hard-negative mining。

---

## 13. Blocking 与候选召回

千万级时不能全量 pairwise。

建议 blocking hierarchy：

```text
Block 0: canonical brand
Block 1: exact canonical reference
Block 2: reference family / prefix
Block 3: same collection + fuzzy ref
Block 4: dense multimodal retrieval within brand
```

### 自动 MATCH 只允许 Block 1

```text
same brand + exact VERIFIED reference
```

### Block 2–4 的作用

只用于：

```text
找到可能漏抽的 reference
发现 typo
人工复核
构造 hard negatives
```

不能因为 Block 4 图文 embedding 很近就自动合并。

---

## 14. Reference typo / OCR error 怎么处理

precision-first 场景下，不建议直接 fuzzy match 后合并。

例如：

```text
126610LN
1266101N
```

OCR 很可能把 `L` 读成 `1`。

正确流程：

```text
raw OCR token
  -> brand grammar validation
  -> catalog candidate retrieval
  -> edit-distance candidates
  -> contextual / visual / title validation
  -> if exactly one candidate has independent support => VERIFIED
  -> else ABSTAIN
```

而不是：

```text
edit_distance <= 1 => same reference
```

所有 fuzzy correction 必须保存：

```text
raw_value
corrected_value
correction_rule
supporting_evidence
```

---

## 15. “Reference Catalog” 应该成为核心资产

建议系统维护品牌级 reference catalog：

```text
ReferenceCatalogEntry
  brand_id
  canonical_reference
  aliases[]
  collection
  model_name
  size
  material
  movement
  dial
  valid_patterns
  first_seen
  last_seen
  source_support[]
```

初期 catalog 可以从三源中高置信 reference 聚合出来，再持续人工修正。

它的价值是：

1. 过滤看起来像 reference 但实际上不存在的 token；
2. 识别 OCR typo；
3. 生成同系列 hard negatives；
4. 做 brand-specific canonicalization；
5. 解释为什么某个候选被验证/拒绝。

相比训练一个巨大模型，reference catalog 对 precision 的价值往往更直接。

---

## 16. 一个可落地的 Reference Resolver 伪代码

```python
from dataclasses import dataclass
from enum import Enum

class RefStatus(str, Enum):
    VERIFIED = "VERIFIED"
    CONFLICT = "CONFLICT"
    ABSTAIN = "ABSTAIN"
    INVALID = "INVALID"

@dataclass
class Candidate:
    canonical: str
    role: str
    provenance: str
    confidence: float
    trusted: bool
    catalog_hit: bool


def resolve_reference(product, candidates, brand_rules):
    official = [
        c for c in candidates
        if c.role == "OFFICIAL_REFERENCE"
        and brand_rules.valid(c.canonical)
    ]

    trusted = [c for c in official if c.trusted]
    trusted_values = {c.canonical for c in trusted}

    # 多个可信 reference 互相冲突，禁止猜测
    if len(trusted_values) > 1:
        return RefStatus.CONFLICT, None

    # 单个可信 reference，可直接验证
    if len(trusted_values) == 1:
        ref = next(iter(trusted_values))
        return RefStatus.VERIFIED, ref

    # 没有 trusted candidate 时，找独立 provenance 的一致证据
    by_ref = {}
    for c in official:
        by_ref.setdefault(c.canonical, []).append(c)

    verified = []
    for ref, group in by_ref.items():
        provenances = {x.provenance for x in group}
        if len(provenances) >= 2 and any(x.catalog_hit for x in group):
            verified.append(ref)

    if len(verified) == 1:
        return RefStatus.VERIFIED, verified[0]

    if len(verified) > 1:
        return RefStatus.CONFLICT, None

    return RefStatus.ABSTAIN, None
```

第一版可以先规则化，后续再用 MOON-Lite 去提升：

```text
role classification
candidate confidence
OCR candidate ranking
conflict probability
```

而不是一开始就把 resolver 黑盒化。

---

## 17. 增量更新策略

每条商品都计算：

```text
content_hash
parser_version
resolver_version
model_version
```

如果 source item 更新：

```text
if content_hash unchanged:
    skip
else:
    re-extract reference candidates
    re-resolve
    compare old resolution vs new resolution
```

如果状态变化：

```text
ABSTAIN -> VERIFIED
=> attach to entity key

VERIFIED A -> VERIFIED B
=> high-severity alert + re-audit previous joins

VERIFIED -> CONFLICT
=> detach / quarantine + alert
```

任何从 VERIFIED reference 发生变化的记录，都不应该静默更新。

---

## 18. Entity Cluster 不建议由模型维护

如果 identity 就是 reference，则 entity cluster 可以直接定义：

```text
entity_id = hash(brand_id + "|" + canonical_reference)
```

记录表：

```text
entity_member
  entity_id
  product_id
  source
  attached_at
  resolver_version
```

这样不会出现传统图聚类的 transitive contamination：

```text
A~B
B~C
=> 错误地把 A,B,C 全合并
```

只要每个成员都必须 independently VERIFIED 到同一个 reference，cluster 就是天然可解释的。

---

## 19. 如何把 MOON2.0 的 MoE 做成业务可解释的 Evidence Router

建议每次模型输出不仅有 embedding，还记录：

```text
router weights
activated experts
input modality mask
reference candidates attended
OCR regions attended
```

示例：

```json
{
  "product_id": 123,
  "modalities": {
    "structured_ref": false,
    "title_ref": true,
    "ocr_ref": true,
    "image": true
  },
  "router": {
    "ref_context_expert": 0.41,
    "ocr_expert": 0.38,
    "text_expert": 0.08,
    "visual_expert": 0.09,
    "source_expert": 0.04
  },
  "top_reference": "126610LN",
  "conflict_score": 0.03
}
```

这能帮助定位：

- 模型是不是因为图片太像而忽视 OCR；
- 某来源 schema 是否被错误信任；
- 某品牌 reference grammar 是否需要修正。

---

## 20. 训练时如何利用来源差异

MOON2.0 按 modality composition 做专家专门化；当前还应该把 source composition 纳入。

例如 expert / routing feature 可以显式包含：

```text
雷小安
腕表之家
奢当家
```

原因不是让模型“记住平台”，而是不同平台字段质量可能系统性不同：

```text
某平台 model 字段接近官方 reference
某平台 product_no 其实是平台 SKU
某平台标题习惯写多个兼容型号
某平台图片中保卡出现率高
```

这些 source priors 可以作为 evidence reliability，但不能成为 identity 本身。

---

## 21. 一个更适合当前需求的训练 Objective

可以构造：

```text
L_total =
    λ1 * L_reference_role
  + λ2 * L_reference_candidate
  + λ3 * L_inter_same_ref
  + λ4 * L_intra_evidence_alignment
  + λ5 * L_hard_negative_margin
  + λ6 * L_conflict
  + λ7 * L_router_regularization
```

### L_reference_role

分类 token / identifier 是：

```text
official reference
SKU
serial
compatibility reference
unknown
```

### L_reference_candidate

给一条 product，选择正确 canonical reference 候选。

### L_inter_same_ref

同 reference 跨源靠近。

### L_intra_evidence_alignment

同商品图文/OCR/reference evidence 一致。

### L_hard_negative_margin

同系列不同 reference 拉开。

### L_conflict

训练模型识别：

```text
title says A
card OCR says B
```

这类冲突。

### L_router_regularization

对应 MOON2.0 的 expert load balance / specialization。

---

## 22. 为什么不建议第一版就复刻 64×A100 的 MOON2.0

当前业务并不需要赢一个通用 multimodal benchmark。

如果 P0 就复刻：

```text
large generative MLLM
+ MoE FFN
+ synthetic visual augmentation
+ multi-objective contrastive training
```

会有几个问题：

1. 成本高；
2. debug 困难；
3. precision failure 难解释；
4. reference 是结构化身份键，很多问题可以用规则/字典解决；
5. 只有几百人工标签，先把标签花在 schema 和 hard cases 上更划算；
6. 生成式增强可能污染身份细节。

所以推荐：

```text
P0：Reference-first deterministic pipeline
P1：OCR + role classifier + catalog
P2：MOON-Lite evidence router / contrastive verifier
P3：需要时再升级 sparse MoE / MLLM
```

这比“一步到位大模型”更符合 precision-first。

---

## 23. 分阶段可直接实施方案

## Phase 0：数据与 reference 规则基线

实现：

- 三源 schema adapter；
- brand alias map；
- reference candidate regex/parser；
- provenance ledger；
- brand-aware canonicalization；
- VERIFIED / CONFLICT / ABSTAIN；
- exact entity key join。

这一阶段的目标不是高 coverage，而是先建立可解释的高 precision baseline。

## Phase 1：OCR 与编号角色识别

增加：

- image region detection；
- OCR；
- identifier role classifier；
- OCR candidate catalog validation；
- text-OCR conflict detection。

重点提高：

```text
reference 不在字段/标题但出现在图片
```

的 coverage。

## Phase 2：MOON-Lite 双层对齐

训练：

```text
text-only
image-only
OCR-only
multimodal
```

目标：

- same reference retrieval；
- hard-negative discrimination；
- intra-item consistency；
- conflict scoring。

部署位置：

```text
ABSTAIN queue / candidate ranking
```

不直接改动最终 MATCH gate。

## Phase 3：动态路由 / MoE

只有当数据证明不同 modality/source 的静态 fusion 确实出现跷跷板时，再加入：

- sparse experts；
- modality objective preference；
- expert specialization regularization。

## Phase 4：选择性自动化扩容

随着 gold/audit 数据增加：

- 调整哪些 evidence pattern 可以从 ABSTAIN 升级为 VERIFIED；
- 对每个 brand/source pair 单独设置 policy；
- 长尾品牌默认更保守。

---

## 24. 评测指标必须改掉“只看 F1”

当前业务的主指标不应该是普通 F1。

推荐：

### 24.1 Auto-match precision

```text
precision among automatically accepted MATCH
```

这是第一指标。

### 24.2 False merge count

```text
自动合并的错误实体数
```

在 golden set / audit set 里应直接看绝对数量。

### 24.3 Acceptance / Coverage

```text
多少记录进入 VERIFIED
多少跨源 pair 能自动 MATCH
```

coverage 是第二目标。

### 24.4 Abstention rate

```text
ABSTAIN 比例
```

precision-first 下，高 abstention 不是失败，而是明确 trade-off。

### 24.5 Conflict detection recall

人工构造或历史发现的冲突案例中，系统能否抓住：

```text
trusted ref disagreement
```

### 24.6 按 brand/source pair 分层

必须分别评估：

```text
雷小安 x 腕表之家
雷小安 x 奢当家
腕表之家 x 奢当家
```

以及每个大品牌。

全局平均会掩盖某一来源/品牌的系统性 false positive。

---

## 25. “0 个误匹配”不等于已经证明绝对安全

即使测试集里观察到 0 false positive，也只能说明在该样本上没看到错误。

一个常用近似：如果审计 `n` 个自动 MATCH，观察到 0 个错误，则 95% 置信下错误率上界约为：

```text
3 / n
```

例如：

```text
n = 300
=> 上界约 1%

n = 3,000
=> 上界约 0.1%

n = 30,000
=> 上界约 0.01%
```

所以“几百对 gold”足够帮助：

- 找架构问题；
- 训练 hard cases；
- 比较方案；

但不足以统计意义上证明百万级自动 merge 达到极端 precision。

正确路线是：

```text
先低 coverage、高审计
-> 累积更多 accepted-match audit
-> 再逐步扩大自动放行范围
```

---

## 26. Shadow Mode

上线前强烈建议：

```text
model / resolver 产生 MATCH
但暂不真正合并
```

持续保存：

- decision；
- evidence；
- raw reference；
- canonical reference；
- router score；
- conflicts；

然后人工抽查 accepted MATCH。

特别对：

```text
高销量 reference
同系列高相似 reference
新增品牌
新增来源字段
parser version 变化
```

增加采样率。

---

## 27. Hard Veto 规则

为了满足 precision-first，建议以下规则写成代码级不可绕过约束。

### Veto 1：可信 reference 冲突

```text
ref_a != ref_b
=> never MATCH
```

### Veto 2：品牌冲突

```text
trusted brand_a != trusted brand_b
=> never MATCH
```

### Veto 3：商品与配件角色冲突

```text
watch vs strap/box/card/accessory
=> never MATCH as same watch entity
```

### Veto 4：reference role 不确定

```text
candidate may be SKU / serial / compatibility ref
=> cannot VERIFIED
```

### Veto 5：生成内容不能成为硬证据

```text
MLLM-generated text/image
=> cannot independently establish reference
```

### Veto 6：模型 similarity 不能覆盖硬冲突

```text
embedding_sim = 0.999
trusted reference conflict exists
=> NO_MATCH / CONFLICT
```

---

## 28. 为什么 MOON2.0 的 Dynamic Modality Balance 对字段稀疏特别重要

三源记录可能出现：

```text
Source A:
  structured ref ✓
  title ✓
  image △

Source B:
  structured ref ✗
  title ✓
  image ✓
  card image ✓

Source C:
  structured ref ✗
  title △
  image ✓
  OCR △
```

固定 fusion：

```text
score = 0.6 text + 0.4 image
```

必然在不同记录上表现不稳定。

MOON2.0 的核心思想是：

> 不让数据配比隐式决定模型偏向哪一模态，而是显式让模型对不同 modality objective 学专门化。

当前可以进一步升级为：

> 不让一个全局阈值隐式决定“哪类 evidence 可靠”，而是根据 evidence composition 动态路由，同时由 hard policy 控制最终权限。

这是本论文对当前 Spec 最有价值的架构迁移。

---

## 29. MOON2.0 不应该照搬的三点

## 29.1 不照搬“embedding top-1 = entity”

当前 identity 是 reference exact，不是最相似商品。

## 29.2 不照搬生成式视觉细节增强

腕表 reference 相关细节不可生成。

## 29.3 不照搬通用日志 positive definition

MOON2.0 的购买日志 positive 是“相关商品关系”，不等价于“同 reference”。

当前训练 label 必须严格按照：

```text
same verified reference
```

构造。

---

## 30. 推荐的最终服务拆分

```text
services/
  ingestion/
    leixiaoan_adapter
    xxxxx_watch_adapter
    shedangjia_adapter

  normalization/
    brand_resolver
    reference_normalizer
    source_field_mapper

  extraction/
    text_reference_extractor
    identifier_role_classifier
    ocr_pipeline
    visual_region_detector

  catalog/
    reference_catalog
    brand_rules

  resolver/
    reference_resolver
    conflict_detector
    evidence_policy

  multimodal/
    text_encoder
    image_encoder
    ocr_encoder
    evidence_router
    hard_negative_miner

  entity/
    entity_key_service
    entity_membership_service

  review/
    abstain_queue
    conflict_queue
    audit_sampler
```

这样可以保证：

- 大模型不是系统单点；
- parser/规则可独立升级；
- 每个 MATCH 有完整证据链；
- 出现错误时能定位是哪一层。

---

## 31. 推荐日志格式

每次 reference resolution 保存：

```json
{
  "product_id": 123,
  "source": "source_b",
  "brand": {
    "raw": "劳力士",
    "canonical": "rolex",
    "status": "VERIFIED"
  },
  "reference": {
    "status": "VERIFIED",
    "canonical": "126610LN",
    "candidates": [
      {
        "raw": "126610LN",
        "provenance": "TITLE",
        "role": "OFFICIAL_REFERENCE",
        "trusted": false
      },
      {
        "raw": "126610 LN",
        "provenance": "OCR_CARD",
        "role": "OFFICIAL_REFERENCE",
        "trusted": true
      }
    ]
  },
  "conflicts": [],
  "resolver_version": "ref-resolver-0.3.1",
  "model_version": "moon-lite-0.1.0"
}
```

实体匹配日志：

```json
{
  "item_a": 123,
  "item_b": 456,
  "decision": "MATCH",
  "reason": "SAME_VERIFIED_CANONICAL_REFERENCE",
  "brand_id": "rolex",
  "canonical_reference": "126610LN"
}
```

这种日志比只保存一个 `match_score=0.982` 可审计得多。

---

## 32. 直接可执行的最小 MVP

如果现在就开始实现，我会先做下面这一版，而不是先训练 MOON2.0。

### 输入

```text
三源 JSON/DB records
```

### Step 1

统一字段：

```text
title
brand
reference
model
attributes
images
```

### Step 2

brand normalize。

### Step 3

从：

```text
structured reference
model field
title
description
```

抽 reference candidates。

### Step 4

identifier role classifier：

```text
reference / SKU / serial / compatibility / unknown
```

### Step 5

brand-aware canonicalization + catalog validation。

### Step 6

两独立证据或可信字段 => VERIFIED。

### Step 7

同 `(brand, VERIFIED reference)` exact join。

### Step 8

其余进入：

```text
OCR -> MOON-Lite -> review
```

这一版已经能形成完整生产闭环，而且每一步都能独立测 precision。

---

## 33. 后续 MOON-Lite 的数据流水线

```text
VERIFIED records
   │
   ├─> same-ref positives
   │
   ├─> nearby-ref hard negatives
   │
   ├─> title-only views
   │
   ├─> image-only views
   │
   ├─> OCR-only views
   │
   └─> multimodal views
           │
           v
      joint contrastive training
           │
           ├─ inter-product alignment
           ├─ intra-product alignment
           ├─ ref hard-negative loss
           └─ dynamic sample filtering
```

部署：

```text
ABSTAIN product
  -> retrieve candidate references/entities
  -> produce evidence ranking
  -> do NOT auto-match unless resolver policy upgrades to VERIFIED
```

---

## 34. 与论文模块的逐项映射

| MOON2.0 原模块 | 原论文目标 | 当前系统改造 | 是否建议首版使用 |
|---|---|---|---|
| Multimodal Joint Learning | 平衡 image/text/mm retrieval | 支持 title/image/OCR/structured-ref 缺失组合 | 是 |
| Modality-driven MoE | 专家适应不同 modality objective | Evidence Router 根据证据组成动态信任专家 | 简化版先用 |
| Inter-product Alignment | 相关商品检索 | same-reference 跨源对齐 + hard-different-ref 拉开 | 是 |
| Intra-product Alignment | 单商品图文一致 | title/reference/OCR/image 内部证据一致性 | 强烈建议 |
| Text Co-augmentation | 丰富商品标题 | 可用于非身份语义补充，但生成 reference 不可作硬证据 | 谨慎 |
| Visual Co-augmentation | 扩大视觉分布 | 仅保留非生成式 crop/校正；禁用生成身份细节 | 不原样使用 |
| Dynamic Sample Filtering | 降低日志噪声 | silver labels 动态降权 / curriculum | 是 |
| embedding retrieval | 通用商品检索 | 只做 candidate/review，不做最终 MATCH | 是，但限权 |

---

## 35. 与当前需求最匹配的最终架构图

```text
┌────────────────────────────────────────────┐
│  雷小安 / 腕表之家 / 奢当家 原始商品       │
└─────────────────────┬──────────────────────┘
                      │
                      v
┌────────────────────────────────────────────┐
│ Source Adapter + Raw Evidence Preservation │
└─────────────────────┬──────────────────────┘
                      │
                      v
┌────────────────────────────────────────────┐
│ Brand Resolution                           │
└─────────────────────┬──────────────────────┘
                      │
                      v
┌────────────────────────────────────────────┐
│ Reference Candidate Extraction             │
│ structured / title / description / OCR     │
└─────────────────────┬──────────────────────┘
                      │
                      v
┌────────────────────────────────────────────┐
│ Identifier Role Classification             │
│ reference / SKU / serial / compatibility   │
└─────────────────────┬──────────────────────┘
                      │
                      v
┌────────────────────────────────────────────┐
│ Brand-aware Canonicalization + Catalog     │
└─────────────────────┬──────────────────────┘
                      │
                      v
┌────────────────────────────────────────────┐
│ Reference Resolver                         │
│                                            │
│ deterministic evidence policy              │
│ + MOON-Lite multimodal consistency         │
│ + hard conflict veto                       │
└──────────────┬─────────────┬───────────────┘
               │             │
          VERIFIED        CONFLICT/ABSTAIN
               │             │
               v             v
┌─────────────────────┐   ┌───────────────────┐
│ (brand, reference)  │   │ Review / Reprocess│
│ exact entity key    │   └───────────────────┘
└──────────┬──────────┘
           │
           v
┌────────────────────────────────────────────┐
│ Cross-source Entity Aggregation            │
│ no pairwise probabilistic merge required   │
└────────────────────────────────────────────┘
```

---

## 36. 对当前需求的最终建议

MOON2.0 的价值非常明确，但要“取其架构思想，不取其最终决策方式”。

### 建议采纳

1. **三模态/多模态联合训练**：解决字段稀疏和缺失；
2. **动态 evidence routing**：不同来源、不同证据组成使用不同 expert；
3. **Inter + Intra 双层对齐**：既学跨源同 reference，又学单条商品内部图文/OCR一致；
4. **困难负样本**：大量构造同系列不同 reference；
5. **动态训练样本过滤**：安全使用大规模 silver labels；
6. **多模态模型只做辅助层**：召回、OCR 验证、冲突检测、review 排序。

### 明确不建议

1. 直接用 embedding similarity 判 MATCH；
2. 用图片相似度替代 reference；
3. 让 MLLM 自由生成 reference 后直接入库；
4. 用生成式视觉增强修改刻字、logo、表盘细节再当证据；
5. 为追求 recall 强行给每条商品选一个 reference；
6. 用一个全局 confidence threshold 覆盖所有品牌/来源。

### 最终落地原则

当前系统应该被定义成：

> **一个以 reference resolution 为核心、以多模态模型提高证据覆盖、以 deterministic identity key 完成聚合、以 abstention 保证 precision 的跨源实体解析系统。**

最终自动 MATCH 仍然只需要：

```text
same canonical_brand
AND same independently VERIFIED canonical_reference
AND no trusted conflict
```

MOON2.0 的任务，是让更多记录能够安全地到达 `VERIFIED`，而不是替代这条规则。

如果按这个方向实施，当前 100 万–1000 万规模并不需要做昂贵的全量 pairwise entity matching：

```text
resolve once per product
-> exact entity key lookup
-> incremental attach
```

复杂 AI 只服务于最难的 reference extraction / verification 部分，既能利用图片和 MLLM，又不会牺牲“不能误匹配”这个最重要的业务约束。

---

## 37. 论文与需求来源

### 论文

- Nie, Zhanheng et al. **MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding.** CVPR 2026.
- CVPR：<https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html>
- arXiv：<https://arxiv.org/abs/2511.12449>
- arXiv HTML：<https://arxiv.org/html/2511.12449>
- MBE2.0：<https://huggingface.co/datasets/ZHNie/MBE2.0>

### 当前需求

- Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》：<https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31>

### 本轮选择来源

- `WorkerXu/my-doc/奢侈品文章调研.md` 中 MOON2.0 对应条目：其推荐理由指出该工作专门处理电商多模态理解中的模态不平衡、对齐不足和噪声，可用于字段稀疏/脏数据下的辅助验证，但不应让视觉相似度替代 reference 硬规则。
