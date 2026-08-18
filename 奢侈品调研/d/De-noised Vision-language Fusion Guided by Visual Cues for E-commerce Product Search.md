# De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本轮从 `奢侈品文章调研.md` 中选择 **De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search**（Hu et al., CVPR 2024 Workshop）进行深入分析。

- CVPR Open Access：<https://openaccess.thecvf.com/content/CVPR2024W/MULA/html/Hu_De-noised_Vision-language_Fusion_Guided_by_Visual_Cues_for_E-commerce_Product_CVPRW_2024_paper.html>
- 作者 PDF：<https://kevin-hu.com/publication/mula_2024/mula_2024.pdf>
- Amazon Science：<https://www.amazon.science/publications/de-noised-vision-language-fusion-guided-by-visual-cues-for-e-commerce-product-search>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

在分析前已检查 `奢侈品调研/d`，此前 d 已经分析过 Ameli、AnyMatch、Confidence Classifier、Conformal Selective Prediction、Ditto、DeepBlocker、Zalando 多模态商品匹配、GraLMatch、MOON2.0、TransClean、pyJedAI、属性抽取/规范化等条目；**本篇尚未分析过**。

这篇论文最值得当前腕表实体匹配项目借鉴的，不是“再训练一个 CLIP”，而是一个非常具体的思想：

> **不要把商品标题里的所有词都当成同等可靠的多模态监督。模型应该学习哪些 token 真正属于当前商品、哪些 token 是噪声，并在训练时自动削弱噪声 token。**

论文提出 **MM-LTP（MultiModal alignment-guided Learned Token Pruning）**：

1. 从 self-attention 或 cross-attention 中计算每个文本 token 的重要度；
2. 为每一层学习一个阈值 `τ_l`；
3. 用可微 sigmoid mask 抑制低重要度 token；
4. 用额外 pruning loss 推动模型主动去掉冗余文本；
5. 方法既可以接 CLIP 这种“靠 alignment loss 隐式融合”的模型，也可以接 ALBEF 这种“有 cross-attention fusion encoder”的模型。

论文在 71 万+ Amazon 商品上证明，这种“在线文本去噪”可以显著改善视觉检索效果，ALBEF 配合多处 pruning 时 R@1 相比普通 fine-tuning 提升超过 5 个百分点。

但是，**当前腕表 Spec 绝不能原样复制 MM-LTP**。原因非常关键：

> 论文把“不能从图片直接看出来的文字”倾向视为视觉噪声；但在腕表实体识别里，`reference number` 恰恰可能是最重要的身份信息，却经常无法从商品主图直接视觉确认。

例如：

```text
劳力士 潜航者 126610LN 黑水鬼 全套
```

`126610LN` 对实体匹配是最高价值 token，但一张正面主图未必能直接读出这个编号。若直接用 MM-LTP 的“视觉可解释性”去剪 token，可能会把 **最重要的身份主键** 当噪声削弱。

因此，本报告建议将论文改造成一个更适合当前需求的方案：

> **Reference-Guarded MM-LTP / 身份保护型双通路多模态去噪**

核心不是用多模态模型直接决定 MATCH，而是：

```text
完整文本 ───────────────> 身份抽取通路（绝不剪 reference）
   |                         |
   |                         +--> brand/reference/serial/SKU/兼容型号角色识别
   |
   +--> MM-LTP 去噪通路 ----> 判断“哪些标题片段真正描述当前商品”
                               |
图片/OCR ---------------------+
                               |
                               v
                       Reference Evidence Graph
                               |
                               v
                    Precision-first Decision Gate
                               |
               +---------------+----------------+
               |               |                |
             MATCH          NO_MATCH          ABSTAIN
```

最终自动 MATCH 仍然必须收口到：

```text
canonical_brand 相同
AND
canonical_reference 相同
AND
双方 reference 都有足够强的“归属于当前商品”的证据
AND
不存在高可信 reference 冲突
```

**视觉相似度永远不能覆盖 reference 冲突。**

如果只做一个直接可落地的 MVP，我建议本篇论文只落地其中最有价值的一部分：

> **把“title 中抽到的 reference 是否真的属于当前商品”做成一个多模态/上下文证据判定问题，而不是直接把所有出现过的型号都当成商品型号。**

这可以直接解决二奢数据里最危险的一类 false positive：

```text
“适配 126610LN 的原装表带”
“116500LN / 126500LN 通用保护膜”
“升级 5711 风格表盘”
“盒证适用于 15400ST”
```

如果只做字符串 exact match，这些标题会非常容易误把“被适配/被提及的 reference”当成“当前售卖商品的 reference”。

---

## 1. 为什么选择这篇论文

当前 Spec 的核心约束是：

- 三个来源：雷小安、腕表之家、奢当家；
- 100 万–1000 万级数据；
- 持续增量；
- 字段高度稀疏；
- reference 有时在结构化字段，有时埋在标题；
- 有图片；
- “同一个商品”定义为同一参考号 / 型号；
- **绝对不能误匹配，precision 优先到极致，允许漏召回**；
- 可以人工标注几百对黄金样本。

表面上这是 Entity Matching，但真正危险的错误往往发生在更前面：

> **reference extraction 不是“找到一个长得像型号的字符串”就结束了，而是必须判断这个字符串在当前 listing 里扮演什么角色。**

二奢标题中的编号可能是：

```text
品牌 reference
机芯型号
系列编号
平台商品 ID
店铺 SKU
序列号 serial
保卡号
配件适配型号
前代/后代型号
比较对象
营销文本中的热门型号
```

因此当前项目如果只做：

```text
regex 抽取 -> normalize -> exact match
```

precision 仍然可能被“编号角色错误”击穿。

这篇论文正好提供了一个与这个问题高度同构的机制：

> **标题里确实存在很多与当前图片弱相关、甚至无关的 token；可以利用多模态 alignment 来判断 token 的有效性，而不是把整段标题等权输入。**

它虽然研究的是视觉检索，不是 reference matching，但它的 token-level 去噪思想可以迁移为：

> **Reference Ownership Verification：这个 reference 是当前商品自己的型号，还是标题里被提到的别的型号？**

对当前 precision-first 系统，这比简单地增加一个“图片 embedding 相似度”更有价值。

---

## 2. 论文原始问题：电商标题为什么会污染图文训练

论文指出，电商 image-text pair 并不像通用图片 caption 那么干净。

卖家为了提高曝光，标题往往塞入大量属性：

```text
品牌
功能
兼容性
材质
营销词
场景词
规格
认证
促销描述
```

其中很多词并没有直接视觉对应。

论文 Figure 1 用 BLIP-2 计算标题短语与图片 embedding 的相似度，展示同一个商品标题里，不同短语与图片的对齐程度差异很大。

如果训练时仍然直接把：

```text
整张图片 <-> 完整标题
```

当作强正样本，模型会被迫学习大量错误或弱相关的 image-text alignment。

这会导致：

1. text encoder 学到标题噪声；
2. vision encoder 被错误文字监督；
3. 对训练分布中的营销词过拟合；
4. 视觉检索泛化变差。

MM-LTP 的目标就是在训练过程中动态回答：

```text
这个 token 是否值得继续参与后续层的 multimodal learning？
```

---

## 3. MM-LTP 技术实现深入拆解

## 3.1 总体架构

论文的整体流程可以概括为：

```text
Text Tokens
    |
    v
Attention Matrix  <--------------------- Image Tokens（cross-attention 情况）
    |
    v
Token Importance Score
    |
    v
Learnable Threshold τ_l
    |
    v
Differentiable Mask
    |
    v
Suppress Low-importance Tokens
    |
    v
Next Transformer Layer / Fusion Network
    |
    +--> 原始 multimodal loss
    |
    +--> pruning regularization loss
```

MM-LTP 不要求特定模型架构，它覆盖两类常见情况。

### 情况 A：CLIP 类模型

```text
Image Encoder ------> image embedding
                         |
                         | contrastive alignment loss
                         |
Text Encoder -------> text embedding
     |
     +--> 从 self-attention 推导 token importance
```

CLIP 没有显式 cross-attention，因此作者使用 text encoder 内部 self-attention 估计 token 重要度。

### 情况 B：ALBEF 类模型

```text
Image Encoder -----------> image tokens
                              |
                              v
Text Encoder -> text tokens -> Cross-Attention Fusion Encoder
                              |
                              +--> 直接从 text-query / image-key attention
                                   估计 token 对视觉内容的重要度
```

这对我们非常重要，因为当前系统也可以拆成：

```text
纯文本 identity branch
+ 图文 cross-modal verification branch
```

不需要强行把所有任务塞进一个端到端模型。

---

## 3.2 Token Importance：从 attention 中计算文本 token 重要度

论文先从 attention score matrix 出发：

```text
Attn(x, z) = x Wq Wk^T z^T / sqrt(d)
```

其中：

- `x`：Query token 序列；
- `z`：Key token 序列；
- self-attention 时 Query/Key 都来自文本；
- cross-attention 时 Query 可以是文本，Key 来自图片 token。

最朴素的做法，是对一个 Query token 在：

```text
所有 attention heads
×
所有 Key tokens
```

上的 attention 做平均。

但作者认为所有 Key token 等权平均会稀释信号，所以做了一个重要改进：

> **只用 Key 侧 `[CLS]` 聚合 token 来评估 Query token 的重要性。**

论文定义大致为：

```text
s(x_i) = mean_h Attn(x_i, z_[CLS])
```

即：

```text
文本 token x_i
   |
   +--> 看它对 image/text 全局 CLS 表示的贡献
   |
   +--> 多头平均
   |
   v
importance score
```

论文的 ablation 结果显示，使用 Key `[CLS]` 的聚合信息比对所有 Key token 简单求平均更稳定。

### 对腕表项目的迁移意义

这里可以直接迁移成一个新特征：

```text
visual_grounding_score(token/span)
```

但不能直接把它解释成“身份重要度”。

正确解释应该是：

> **这个 token/span 是否受到当前商品视觉内容支持。**

于是我们可以把 reference 判断拆成两个维度：

```text
identity_likelihood(reference_span)
visual_ownership_support(reference_span)
```

例如：

```text
标题：原装表带 适配 Rolex 126610LN
```

`126610LN` 的：

```text
identity_likelihood = 高（非常像 Rolex reference）
visual_ownership_support = 低（图里是表带，不是潜航者整表）
```

因此不能把该 listing 归入 `126610LN` 腕表实体。

---

## 3.3 Learned Threshold：每层自己学习“剪多少”

如果只手工设置一个固定阈值：

```text
importance < 0.3 => prune
```

很难泛化，因为：

- 不同层 attention 分布不同；
- 不同任务需要不同 pruning 强度；
- 不同数据集噪声比例不同；
- 深层语义可能只需要少量 token。

因此论文为每层学习独立阈值：

```text
τ_l
```

并使用 tempered sigmoid 构造可微 mask：

```text
M_l(x_i) = sigmoid((s_l(x_i) - τ_l) / T)
```

当温度 `T` 很小时：

```text
s > τ  -> mask 接近 1
s < τ  -> mask 接近 0
```

然后把 mask 乘到该层 token 输出上：

```text
h_i' = M_l(x_i) * h_i
```

这不是物理删除 token，而是训练时把低价值 token 的后续影响压到接近 0。

论文设置中：

- `T = 1e-4`；
- layer-wise threshold 初始化为逐层上升；
- 最终层阈值初始化到约 `0.01`；
- pruning loss 正则系数 `λ = 0.1`。

### 为什么这比硬规则 pruning 更好

因为阈值会同时受到：

```text
下游任务 loss
+
pruning pressure
```

共同训练。

一个 token 如果对任务真的有贡献，模型会倾向保留；如果长期没有贡献，则会被压制。

---

## 3.4 Pruning Loss：明确奖励“少用 token”

仅有 sigmoid mask 不会自动让模型愿意剪 token。

因此论文增加 L1 风格 pruning loss：

```text
L_prune = average_l( ||M_l(x)||_1 / query_length_l )
```

最终训练目标：

```text
L = L_model + λ * L_prune
```

含义是：

```text
L_model：你仍然要把原任务做好
L_prune：在不损害任务的情况下，尽量少依赖 token
```

这会逼迫模型只留下真正有效的信息。

### 对当前项目的可迁移版本

如果我们做“reference ownership verifier”，不能直接最小化所有 token mask，因为 reference 本身需要保护。

建议改成：

```text
L = L_ownership
  + λ_visual * L_prune_visual
  + λ_role * L_role
  + λ_guard * L_identity_guard
```

其中：

```text
L_ownership
= reference 是否属于当前售卖商品

L_prune_visual
= 在视觉语义通路减少无效标题 token

L_role
= reference / serial / sku / compatible_ref / movement / unknown 的 span 角色分类

L_identity_guard
= 对高可信 identity span 加保护，防止被视觉 pruning 错删
```

---

## 3.5 论文在 CLIP 与 ALBEF 上的接法

论文刻意选择两种结构，说明 MM-LTP 不是依赖单一 backbone。

### CLIP

论文使用：

```text
ViT-B/16 image encoder
12 layers
约 86M 参数

CLIP text encoder
12 layers
约 63M 参数
```

CLIP 只有 image/text 两塔，没有显式 fusion layer，所以 pruning 接在 text self-attention。

论文 first-stage fine-tuning 结果：

```text
Off-the-shelf CLIP
R@1 = 42.56

CLIP-FT1
R@1 = 53.59

CLIP-FT1 + MM-LTP
R@1 = 55.21
```

即 MM-LTP 在普通 domain fine-tuning 之上仍带来约 `+1.62` 个百分点 R@1。

### ALBEF

ALBEF 有显式 cross-attention fusion，作者可以直接根据 text-query 和 image-key 的关系做 pruning。

论文结果：

```text
Off-the-shelf ALBEF
R@1 = 38.60

ALBEF-FT1
R@1 = 51.68

ALBEF-FT1 + cross-attention pruning
R@1 = 54.59

ALBEF-FT1 + text/self-attention + fusion/cross-attention pruning
R@1 = 57.06
```

相对普通 fine-tuning，最佳配置达到 `+5.38` 个百分点 R@1。

这说明：

> 对电商噪声文本，显式图文交互层非常适合做 token-level denoising。

对于当前项目，如果真的要训练多模态 ownership verifier，我更建议：

```text
轻量 text encoder
+
vision encoder
+
少量 cross-attention layers
+
identity-aware pruning
```

而不是直接训练一个巨大的 VLM 生成“是不是同款”的自然语言答案。

---

## 3.6 数据规模与训练工程

论文构建了基于 Amazon ESCI 的多模态数据：

```text
Train products: 637,511
Test products:   80,000
Total products: 717,511

Train image-text pairs: 858,231
Test image pairs:       186,084
Total pairs:          1,044,315
```

每个商品：

```text
title
main image
0~10 auxiliary images
```

训练工程：

```text
PyTorch
Ray distributed
8 × NVIDIA A100
```

CLIP 实验中：

```text
100+ epochs
batch size 1360
AdamW
weight decay 0.02
warmup + cosine decay
```

这说明 MM-LTP 原论文是一个真正的“大规模训练技术”，而不是几百样本即可从头训练。

### 当前项目不应该照抄这个训练规模

当前只有几百黄金标签时，不应该第一步就：

```text
训练一个新的 multimodal transformer
```

更合理的是：

1. 使用预训练 encoder；
2. 冻结大部分 backbone；
3. 只训练 span-role / ownership 的轻量 head；
4. 用规则和品牌 reference registry 提供弱监督；
5. 等积累数万审核样本后，再考虑 MM-LTP 式 domain fine-tuning。

---

## 3.7 Grad-CAM 实验给出的重要启发

论文对 ALBEF cross-attention 做 Grad-CAM 可视化。

结果显示：

- `valve`、`dog`、`handle` 这类可视觉对应词，attention 区域集中；
- 品牌名或视觉上无法直接定位的 token，attention 更分散。

这验证了论文核心假设：

```text
可视觉落地的词
和
非视觉/噪声词
在 cross-attention 分布上确实不同
```

对当前项目最直接的迁移不是“看 attention 高不高就保留 reference”，而是：

> **用 attention/视觉证据帮助识别“标题里这个 reference 是否在描述图中的主体”。**

也就是把模型目标从：

```text
这个词视觉上重要吗？
```

改成：

```text
这个 reference mention 是否属于当前商品主体？
```

---

## 4. 为什么原版 MM-LTP 不能直接用于腕表 reference

这是本篇最重要的结论。

## 4.1 reference 经常是“非视觉，但身份关键”

论文的基本假设大致是：

```text
越能被图片解释的 token
越适合作为视觉训练监督
```

对 visual search 是合理的。

但实体匹配目标不同。

腕表 reference 可能：

```text
126610LN
116500LN
5711/1A-010
15500ST.OO.1220ST.01
4500V/110A-B128
```

这些编号在正面商品图中通常不可读，甚至完全没有视觉文本。

但业务上：

```text
reference = entity identity
```

所以：

```text
视觉弱相关 != 可删除
```

这是原论文迁移时必须修正的第一原则。

---

## 4.2 中文与字母数字 reference 的 tokenization 问题

论文明确指出它使用英文 caption，并把中文 tokenization 的影响放在研究范围之外。

当前三源数据是中文电商/二奢文本，而且 reference 是高度特殊的字母数字串。

例如：

```text
AP爱彼15500ST.OO.1220ST.01灰盘
劳力士126610 LN黑水鬼
百达翡丽5711／1A-010
```

如果直接交给通用 tokenizer，可能出现：

```text
126610 + LN
15500 + ST + . + OO + ...
5711 + / + 1A + - + 010
```

此时 token-level pruning 可能只删掉 reference 的一部分，导致 reference span 被破坏。

### 建议

必须在 subword tokenizer 之前先运行 reference span detector：

```text
raw title
   |
   v
brand-aware regex / registry matcher
   |
   v
protected spans
   |
   v
<REF_0> / <REF_1> special span token
   |
   v
multimodal tokenizer
```

例如：

```text
输入：劳力士 126610 LN 黑水鬼

预处理：劳力士 <REF_0> 黑水鬼
REF_0.raw = "126610 LN"
REF_0.canonical = "126610LN"
```

这样多模态模型只能对整个 reference span 做保留/角色判断，而不是把一个型号拆坏。

---

## 4.3 “视觉不支持”有两种完全不同的语义

对 reference 来说：

```text
视觉不支持
```

不能直接解释为 reference 错误，因为可能只是商品图没有拍到表背或保卡。

必须区分：

### A. Missing visual evidence

```text
主图是腕表正面
标题写 126610LN
图片里读不到 reference
```

这只是：

```text
没有视觉证据
```

不是负证据。

### B. Contradictory visual evidence

```text
标题写 126610LN
但图片明显是表带/盒子/保卡配件
```

或者 OCR 在表背/保卡上读出另一个 reference。

这才是：

```text
冲突证据
```

因此模型输出不能只是一个 `visual_similarity_score`，而应该有三态：

```text
SUPPORT
UNKNOWN
CONTRADICT
```

其中 `UNKNOWN` 必须保持中性，不能当负样本。

---

## 5. 推荐落地架构：Reference-Guarded 双通路 MM-LTP

建议把原论文改造成两个完全不同的文本处理分支。

```text
                          ┌──────────────────────────────┐
                          │ Raw listing                 │
                          │ title/desc/fields/images    │
                          └──────────────┬───────────────┘
                                         │
                    ┌────────────────────┴────────────────────┐
                    │                                         │
                    v                                         v
        ┌────────────────────────┐               ┌────────────────────────┐
        │ Identity Extraction    │               │ Visual Semantic Branch │
        │ NO TOKEN PRUNING       │               │ MM-LTP style pruning   │
        └───────────┬────────────┘               └───────────┬────────────┘
                    │                                         │
          brand/reference/serial                    visual grounding
          sku/compatible-ref spans                  product type evidence
                    │                                         │
                    └────────────────────┬────────────────────┘
                                         v
                            ┌────────────────────────┐
                            │ Image Evidence Branch  │
                            │ image role + OCR       │
                            └───────────┬────────────┘
                                        │
                                        v
                            ┌────────────────────────┐
                            │ Evidence Fusion        │
                            │ reference ownership    │
                            └───────────┬────────────┘
                                        │
                                        v
                            ┌────────────────────────┐
                            │ Canonical Reference    │
                            │ Registry + Normalize   │
                            └───────────┬────────────┘
                                        │
                                        v
                            ┌────────────────────────┐
                            │ Precision Decision Gate│
                            └───────────┬────────────┘
                                        │
                      ┌─────────────────┼─────────────────┐
                      v                 v                 v
                    MATCH            NO_MATCH          ABSTAIN
```

这套结构保留了 MM-LTP 最有价值的“视觉引导去噪”，但不会让它碰最终身份主键。

---

## 6. 第一条通路：Identity Extraction 必须保留完整文本

这一层负责：

```text
brand
reference
serial
movement
seller_sku
platform_id
compatible_reference
other_model_mention
```

建议输出 span 级结构：

```json
{
  "raw": "劳力士原装表带 适配126610LN",
  "spans": [
    {
      "text": "劳力士",
      "role": "brand",
      "canonical": "rolex"
    },
    {
      "text": "126610LN",
      "role": "reference_candidate",
      "canonical": "126610LN"
    }
  ]
}
```

这里**不做任何视觉 pruning**。

原因是我们先要知道标题里到底出现过哪些身份候选，再判断这些候选属于谁。

---

## 7. 第二条通路：把 MM-LTP 改成“视觉主体去噪器”

这一通路的任务不是找 reference，而是判断：

```text
标题哪些词真正描述当前商品主体？
```

输入：

```text
title tokens
+
main image / auxiliary images
```

输出：

```text
visual_grounding_score per span
```

例如：

```text
标题：Rolex 126610LN 适配表带 黑色橡胶 20mm
图片：一根橡胶表带
```

模型可能得到：

```text
Rolex       LOW / UNKNOWN
126610LN    LOW / UNKNOWN
适配        text-only relation
表带         HIGH
黑色         HIGH
橡胶         HIGH
20mm         MEDIUM
```

此时“表带”与图片高度对齐，而 `126610LN` 只是 reference mention。

再结合上下文：

```text
适配 + 126610LN
```

就可以把它标成：

```text
compatible_reference
```

而不是：

```text
product_reference
```

---

## 8. Identity Guard：论文 mask 的关键改造

如果后续确实要把 MM-LTP 集成到一个模型里，建议修改 mask。

原论文：

```text
M_i = sigmoid((s_i - τ) / T)
```

当前项目增加 identity guard：

```text
g_i = 1
if token/span role in {
    brand,
    reference_candidate,
    serial_candidate,
    sku_candidate
}
else 0
```

融合 mask：

```text
M'_i = max(g_i, M_i)
```

或者更稳妥：

```text
Identity Branch: 永不应用 MM-LTP
Visual Branch:   允许应用 MM-LTP
```

我更推荐后者，因为逻辑更可审计：

> **身份抽取与视觉语义去噪从模型结构上隔离。**

这样即便多模态模型训练失败，也不会破坏 reference 抽取。

---

## 9. Reference Ownership Verifier：这篇论文最适合直接落地的任务

建议新增一个非常具体的模型任务：

```text
Input:
  listing title
  extracted reference span
  product type
  source fields
  1~N images
  OCR evidence

Output:
  OWNED_BY_PRODUCT
  COMPATIBLE_WITH_PRODUCT
  OTHER_PRODUCT_MENTION
  PLATFORM_OR_SELLER_CODE
  UNKNOWN
```

比直接做 pairwise `MATCH/NO_MATCH` 更安全。

### 示例 1：真正商品 reference

```text
标题：劳力士潜航者 126610LN 2024年 全套
图片：整表，多张

reference = 126610LN
role = OWNED_BY_PRODUCT
```

### 示例 2：配件适配 reference

```text
标题：劳力士 126610LN 适配 Rubber B 橡胶表带
图片：表带

reference = 126610LN
role = COMPATIBLE_WITH_PRODUCT
```

### 示例 3：比较/升级关系

```text
标题：116500LN 改 126500LN 新款风格表盘

116500LN = source/compatible/mentioned
126500LN = target-style/mentioned
```

都不能自动作为当前售卖整表 identity。

### 示例 4：图片 OCR 提供强证据

```text
标题：百达翡丽运动款 蓝盘
结构化 reference：空
保卡图片 OCR：5711/1A-010
```

若 OCR 区域属于保卡/吊牌 identity region，并且品牌 registry 校验通过，则可把 `5711/1A-010` 提升为强 reference evidence。

---

## 10. 图片不应该只做 embedding：先做 Image Role Classification

当前项目有图片，但不同图片对 reference 的证据价值差异巨大。

建议先分类：

```text
WATCH_FRONT
WATCH_BACK
CLASP
MOVEMENT
WARRANTY_CARD
HANG_TAG
BOX
ACCESSORY
OTHER
```

不同 image role 使用不同策略。

### WATCH_FRONT

适合：

```text
品牌/系列/盘面/外观辅助
候选召回
变体冲突辅助
```

但一般不能提供强 reference OCR。

### WATCH_BACK / MOVEMENT

适合：

```text
刻字 OCR
机芯信息
型号/编号证据
```

### WARRANTY_CARD / HANG_TAG

这是最有价值的 identity image。

可以：

```text
OCR reference
OCR serial
品牌
日期
经销商字段
```

### ACCESSORY / BOX

如果标题里出现腕表 reference，而图片主体被识别为：

```text
表带
表盒
保护膜
表扣
```

这是非常强的“reference 可能只是兼容型号”的信号。

这正是 MM-LTP 式视觉 grounding 可以发挥最大价值的场景。

---

## 11. 建议建立 Reference Evidence，而不是直接存一个 reference 字段

每个 listing 不要只存：

```text
reference = 126610LN
```

建议存证据集合：

```json
{
  "listing_id": "ldj_123",
  "brand": "rolex",
  "reference_evidence": [
    {
      "raw": "126610 LN",
      "canonical": "126610LN",
      "source": "title",
      "role": "owned_reference",
      "confidence_level": "T2",
      "span": [8, 17]
    },
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "source": "warranty_card_ocr",
      "role": "owned_reference",
      "confidence_level": "T3",
      "image_id": "img_3",
      "bbox": [120, 80, 440, 150]
    }
  ]
}
```

这样最终 decision engine 才能审计：

```text
为什么认为它是 126610LN？
```

而不是只有一个不可解释的模型分数。

---

## 12. 建议定义三级证据强度

### T3：直接身份强证据

例如：

```text
高质量来源的结构化 reference 字段
保卡/吊牌/reference 区域 OCR + 品牌规则校验通过
两个独立高质量 identity source 一致
```

### T2：强文本归属证据

例如：

```text
标题抽取 reference
+ role classifier 判定 owned_reference
+ 当前商品类型是腕表主体
+ 品牌 reference registry 合法
+ 无兼容/适配/for 等负上下文
```

### T1：辅助证据

例如：

```text
图片 embedding 很像
模型猜测系列
LLM 推断
弱 OCR
文本相似度
```

### 自动 MATCH 规则

至少要求：

```text
左侧 reference >= T2
AND
右侧 reference >= T2
AND
canonical_brand 相同
AND
canonical_reference 完全一致
AND
双方无 T2/T3 冲突
```

`T1` 只能：

```text
候选召回
排序
人工复核辅助
```

不能单独产生 MATCH。

---

## 13. Precision-first Decision Gate

建议 decision engine 明确只有四种状态：

```text
MATCH
NO_MATCH
CONFLICT
ABSTAIN
```

伪代码：

```python
def decide(a, b):
    if a.brand_id is None or b.brand_id is None:
        return "ABSTAIN"

    if a.brand_id != b.brand_id:
        return "NO_MATCH"

    if a.has_high_trust_reference_conflict:
        return "CONFLICT"

    if b.has_high_trust_reference_conflict:
        return "CONFLICT"

    if a.verified_reference and b.verified_reference:
        if a.verified_reference == b.verified_reference:
            return "MATCH"
        return "NO_MATCH"

    return "ABSTAIN"
```

注意：

```text
visual similarity > 0.99
```

也不进入自动 MATCH 条件。

原因是腕表同系列不同 reference 的外观可能极其相似。

---

## 14. 100万–1000万级别时，不需要做全量 pairwise comparison

如果最终定义就是：

```text
same entity = same canonical reference
```

那么大多数数据根本不需要 pairwise matcher。

主路径应该是：

```text
listing
 -> reference extraction
 -> ownership verification
 -> canonicalization
 -> hash/index lookup
 -> entity group
```

理想情况下复杂度接近：

```text
O(N)
```

而不是：

```text
O(N^2)
```

### unresolved listing 才走候选检索

对于没有可靠 reference 的记录：

```text
brand block
  -> series/product-type block
  -> text / image ANN Top-K
  -> candidate reference registry
  -> closed-set evidence verification
  -> ABSTAIN / review
```

这里多模态 embedding 只是：

```text
candidate generator
```

不是 matcher。

---

## 15. 推荐的千万级数据架构

```text
                         ┌──────────────────────┐
雷小安 ─┐                │                      │
腕表之家 ├──> Ingestion ->│ Raw Listing Store    │
奢当家 ─┘                │                      │
                         └──────────┬───────────┘
                                    │
                                    v
                         ┌──────────────────────┐
                         │ Normalization Queue   │
                         └──────────┬───────────┘
                                    │
                   ┌────────────────┼────────────────┐
                   v                v                v
              Brand Normalizer  Ref Extractor    Image/OCR Worker
                   │                │                │
                   └────────────────┼────────────────┘
                                    v
                         ┌──────────────────────┐
                         │ Evidence Store       │
                         └──────────┬───────────┘
                                    │
                                    v
                         ┌──────────────────────┐
                         │ Reference Resolver   │
                         │ + Ownership Verifier │
                         └──────────┬───────────┘
                                    │
               ┌────────────────────┴────────────────────┐
               v                                         v
        verified reference                         unresolved
               │                                         │
               v                                         v
       Reference Hash Index                    Brand-blocked ANN
               │                                         │
               v                                         v
       Exact Entity Group                       top-K candidate refs
               │                                         │
               └────────────────────┬────────────────────┘
                                    v
                         Precision Decision Gate
                                    │
                    ┌───────────────┼───────────────┐
                    v               v               v
                  MATCH          CONFLICT         ABSTAIN
                                                    │
                                                    v
                                               HITL Review
```

---

## 16. 建议的数据表

### listing

```text
listing_id
source
source_item_id
raw_title
raw_description
raw_brand
raw_reference
product_type
crawl_time
content_hash
```

### listing_image

```text
image_id
listing_id
image_url/object_key
image_role
ocr_text
ocr_json
embedding_version
```

### reference_evidence

```text
evidence_id
listing_id
raw_value
canonical_value
brand_id
source_type
role
trust_level
extractor_version
span_start/span_end
image_id
bbox
confidence
created_at
```

### listing_identity

```text
listing_id
brand_id
resolved_reference
resolution_status
resolution_version
conflict_reason
```

### canonical_reference

```text
brand_id
canonical_reference
aliases
series
known_patterns
status
```

### entity_group

```text
entity_id
brand_id
canonical_reference
```

由于业务定义直接等同于 reference，可以稳定使用：

```text
entity_key = hash(brand_id + "|" + canonical_reference)
```

### review_task

```text
task_id
listing_id
candidate_references
all_evidence
conflict_reason
review_result
reviewer
```

---

## 17. Reference Normalization 必须 brand-specific

不要设计一个全局：

```text
remove all punctuation
```

然后结束。

不同品牌 reference 结构不同。

建议：

```text
normalize_reference(brand, raw_value)
```

内部按品牌规则执行。

例如可以做：

```text
Unicode NFKC
全角 -> 半角
uppercase
统一 slash / dash
去无意义空格
保留有语义的分隔结构
registry lookup
pattern validation
```

例子：

```text
126610 LN
126610-LN
126610LN
```

可以归一到：

```text
126610LN
```

而：

```text
5711/1A-010
```

不能粗暴变成没有结构语义的随机串后就直接匹配，应该保留 canonical display 和 canonical key 两个版本。

---

## 18. MM-LTP 在当前项目里最适合做“负向 veto”，而不是“正向授权”

这是一个非常重要的工程原则。

因为当前：

```text
precision >> recall
```

所以多模态模型最好主要提供：

```text
否决信号
冲突信号
风险信号
```

而不是：

```text
自动 MATCH 信号
```

例如：

### 情况 A

```text
title_ref = 126610LN
image_role = ACCESSORY
ownership_model = COMPATIBLE_WITH_PRODUCT 0.98
```

结果：

```text
不能 resolve 为 126610LN
```

### 情况 B

```text
structured_ref = 126610LN
warranty_card_ocr = 126610LV
```

结果：

```text
CONFLICT
```

### 情况 C

```text
left_ref = 126610LN T3
right_ref = 126610LN T2
image_similarity = 0.42
```

如果两边 reference 高可信且没有冲突：

```text
仍可 MATCH
```

因为图片拍摄风格不同不能推翻 identity key。

这就是“reference-first”的正确优先级。

---

## 19. 训练数据如何利用“几百对黄金标签”

当前人工预算几百对，最不应该做的是随机抽几百 pair。

应该把标签集中在最危险的 decision boundary。

建议至少覆盖以下 hard cases。

### A. 同系列相邻 reference

```text
126610LN vs 126610LV
116500LN vs 126500LN
5711 vs 5811
15400ST vs 15500ST
```

这些是视觉最容易误导模型的负样本。

### B. reference 兼容/适配文本

```text
for 126610LN
适配 5711
compatible with 15500ST
```

专门训练：

```text
compatible_reference != owned_reference
```

### C. 多 reference 标题

```text
116500LN/126500LN 通用
5711 继任 5811
15400/15500 表带
```

### D. OCR 噪声

```text
0/O
1/I/L
5/S
8/B
斜杠/连字符
```

### E. 结构化字段错误

故意采集：

```text
structured field 和 title/OCR 冲突
```

### F. 配件/盒证/表带

这是最重要的 false-positive 压测集之一。

---

## 20. 几百标签不能“证明”99.99% precision

当前需求说“绝对不能误匹配”。

必须注意：

> 几百个黄金标签足够做模型选择和 hard-case 校准，但不足以仅靠统计验证证明极高 precision。

例如，如果 500 个自动 MATCH 样本全部正确，单侧 95% 置信下界大约也只有 **99.4%** 左右，而不是 99.99%。

因此“绝对不能误匹配”不能只靠：

```text
模型在验证集 precision = 100%
```

真正的保障必须来自：

```text
强业务不变量
+
reference exact key
+
冲突 veto
+
abstain
+
hard-negative 定向测试
+
生产抽样审计
```

换句话说：

> 模型负责提高 coverage；规则和证据门负责守 precision。

---

## 21. 建议的训练任务不是 Pair Matching，而是三个更小的任务

### Task 1：Reference Span Detection

```text
输入：title/description/structured fields
输出：reference candidate spans
```

可以大量用规则/registry 弱监督，不需要很多人工标签。

### Task 2：Reference Role Classification

```text
owned_reference
compatible_reference
serial
movement
sku
platform_id
other_reference
unknown
```

这是最值得人工标几百条的任务。

### Task 3：Multimodal Ownership Verification

输入：

```text
title
reference span
image roles
visual embeddings
OCR
```

输出：

```text
SUPPORT
UNKNOWN
CONTRADICT
```

最终实体匹配本身反而可以主要使用 deterministic rule。

---

## 22. 一个可直接实现的 Ownership Model

如果希望尽快落地，不需要完整复刻 ALBEF。

可以先做轻量特征模型。

### 文本特征

```text
reference 前后 ±20 字符
是否出现：适配 / 兼容 / for / replacement / strap / band / case / box / protector
商品类型
品牌
reference 在标题中的位置
是否出现多个 reference
是否结构化字段也存在同值
```

### 图片特征

```text
image_role distribution
main image embedding
OCR 是否出现同 reference
OCR 是否出现冲突 reference
```

### Registry 特征

```text
reference 是否符合品牌 pattern
是否存在于 canonical registry
是否和标题系列一致
```

先训练：

```text
LightGBM / small MLP / shallow transformer head
```

即可。

等样本积累后，再升级到 MM-LTP cross-attention model。

这种顺序更符合当前成本和 precision 约束。

---

## 23. 如果要真正实现 MM-LTP 式模型，建议结构

```text
Text Encoder
  |
  |-- protected identity spans
  |
  v
Cross Attention Block <------ Vision Encoder
  |
  +--> token grounding score
  |
  +--> learned visual-pruning mask
  |
  v
Fusion Representation
  |
  +--> product-type head
  +--> reference-role head
  +--> ownership head
```

### Identity span 不参与 pruning

```python
mask = sigmoid((score - tau) / temperature)
mask[identity_span] = 1.0
```

### 更推荐 span-level，而不是 token-level

```text
劳力士 | <REF:126610LN> | 黑水鬼 | 适配 | 表带
```

MM-LTP 对：

```text
<REF:126610LN>
```

作为整体打分。

这样不会出现：

```text
保留 126610
剪掉 LN
```

这种灾难性错误。

---

## 24. 增量更新流程

当前是持续爬取，必须让每条新 listing 只处理一次昂贵任务。

推荐：

```text
new listing
  |
  v
content hash dedupe
  |
  v
text normalize + reference extraction
  |
  v
image download / perceptual hash
  |
  v
only new image -> OCR / embedding
  |
  v
reference evidence resolution
  |
  +--> verified -> exact entity lookup
  |
  +--> unresolved -> ANN candidate lookup
```

模型版本升级后，不需要把所有原始数据重抓，只需要对：

```text
unresolved
conflict
low-trust
```

队列重跑即可。

建议所有结果带：

```text
extractor_version
normalizer_version
ocr_version
ownership_model_version
decision_rule_version
```

方便回滚与重算。

---

## 25. Offline 评估指标必须重做，不能照论文只看 Recall@K

论文的目标是 visual search，因此核心指标是：

```text
Recall@1 / @5 / @10
```

当前项目完全不同。

必须至少看：

```text
Auto-Match Precision
Auto-Match Coverage
False Match Count
Abstain Rate
Conflict Detection Recall
Reference Extraction Precision
Owned-reference Precision
Hard-negative False Positive Rate
```

其中第一优先级：

```text
False Match Count
```

然后才是 coverage。

建议评估集按 bucket 分层：

```text
品牌
来源组合
reference 格式
商品主体/配件
是否多 reference
是否 OCR
新旧时间段
```

否则整体 precision 很容易掩盖某一个高风险 bucket。

---

## 26. Shadow 模式和上线策略

第一阶段不要让模型直接写实体关系。

### Phase A：只抽证据

```text
抽 reference
做 normalization
输出 evidence
不自动合并
```

### Phase B：影子决策

系统同时输出：

```text
would_match
would_abstain
would_conflict
```

但不影响正式数据。

重点人工检查：

```text
所有 would_match
+ 所有规则/模型冲突
```

### Phase C：只放行最强规则

例如：

```text
T3 + T3 same canonical reference
```

### Phase D：逐步扩 coverage

再考虑：

```text
T3 + T2
T2 + T2
```

每扩大一档，都独立验证 false-positive。

不要直接用一个连续阈值：

```text
score > 0.95 自动匹配
```

因为不同品牌、不同来源、不同商品类型的 score 分布不一定可比。

---

## 27. 直接可执行的 MVP

如果当前要尽快做一版，我会按下面顺序实现。

### MVP-1：Reference-first baseline

```text
1. brand canonicalization
2. brand-specific reference regex
3. reference normalization
4. exact entity key
5. conflict detection
6. ABSTAIN
```

先建立 precision 上限很高的基础系统。

### MVP-2：Reference role classifier

专门解决：

```text
当前商品 reference
vs
适配/兼容/提及 reference
```

这是本篇 MM-LTP 思想最直接的业务价值。

### MVP-3：Image role + OCR

优先做：

```text
WARRANTY_CARD
HANG_TAG
WATCH_BACK
ACCESSORY
```

而不是先追求一个全能视觉 embedding。

### MVP-4：MM-LTP style visual grounding

对困难标题加入：

```text
span-level visual grounding
```

主要作为：

```text
compatible/accessory veto
conflict signal
review explanation
```

### MVP-5：ANN unresolved candidate retrieval

只给没有可靠 reference 的 listing 使用。

---

## 28. 与论文方案的对应关系

| 论文 MM-LTP | 当前腕表系统 |
|---|---|
| 商品标题存在非视觉噪声 | 标题存在营销词、配件兼容型号、其它 reference mention |
| token importance | span 的 visual grounding / ownership support |
| learned threshold | 可学习的视觉语义过滤阈值 |
| prune noisy token | 弱化与当前商品主体无关的描述 |
| CLIP self-attention | 低成本纯文本/双塔辅助分支 |
| ALBEF cross-attention | 图文 ownership verifier |
| visual search R@K | reference ownership precision / conflict recall |
| 图文 pair 对齐 | listing/reference 证据归属 |
| 去掉 non-visual token | **不能直接删除 identity token，必须 identity guard** |

最关键的改造只有一句话：

> **把“视觉相关性”从身份判定依据降级为 reference 归属/冲突证据。**

---

## 29. 失败模式与防护

## 29.1 主图没有 reference 可见信息

不能因为视觉 grounding 低就否定 reference。

处理：

```text
LOW visual grounding -> UNKNOWN
不是 CONTRADICT
```

---

## 29.2 同系列腕表外观非常像

处理：

```text
image similarity only candidate retrieval
never auto match
```

---

## 29.3 标题同时出现两个型号

处理：

```text
role classification
如果无法判断归属 -> ABSTAIN
```

---

## 29.4 OCR 把 serial 当 reference

处理：

```text
brand-specific pattern
image-role context
registry validation
role classifier
```

---

## 29.5 平台 SKU 看起来像型号

处理：

```text
source-specific field mapping
编号角色分类
不要只靠 regex pattern
```

---

## 29.6 老款/新款对比文本

处理：

```text
关系词：新款 / 老款 / 对比 / 升级 / 同款风格 / replacement
reference span relation parsing
```

---

## 30. 最终推荐方案

结合论文与当前 Spec，我推荐最终系统采用：

> **Canonical Reference Registry + Identity-preserving Text Extraction + MM-LTP-inspired Visual Ownership Verification + OCR Evidence + Precision Decision Gate + Abstention**

完整链路：

```text
[1] Source ingestion
      |
[2] Brand canonicalization
      |
[3] Protected reference-span extraction
      |
[4] Reference role classification
      |  owned / compatible / serial / sku / other
      |
[5] Brand-specific canonicalization + registry validation
      |
[6] Image role classification
      |
[7] OCR on identity-bearing images
      |
[8] MM-LTP-inspired visual grounding
      |  SUPPORT / UNKNOWN / CONTRADICT
      |
[9] Evidence fusion
      |
[10] Verified canonical reference
      |
      +--> same reference + no conflict -> MATCH
      +--> different verified ref       -> NO_MATCH
      +--> evidence conflict            -> CONFLICT
      +--> insufficient evidence        -> ABSTAIN
      |
[11] unresolved only -> brand-blocked multimodal ANN -> HITL
```

这里的优先级必须固定为：

```text
reference identity evidence
>
reference conflict evidence
>
OCR / role context
>
visual semantic support
>
embedding similarity
```

而不是反过来。

---

## 31. 本篇论文对当前项目最值得保留的三个点

### 1. 标题不能整段等权相信

电商 title 本身就是噪声源。

当前项目中尤其要防：

```text
兼容型号
配件型号
对比型号
营销型号
```

### 2. 用多模态 alignment 做 token/span 归属判断

这比简单增加一个 image similarity score 更有价值。

它可以回答：

```text
标题中的 reference 是否真的在描述图片主体？
```

### 3. 去噪模型只能做辅助，身份 token 必须被保护

这是迁移时最重要的修正。

> **腕表 reference 是一个“可能视觉不可见，但业务上最高优先级”的身份属性。**

所以绝不能把“看不见”当成“没用”。

---

## 32. 一句话落地建议

如果只从这篇论文拿一个可以马上做的功能，我会做：

> **Reference Ownership Verifier：对标题里每个 reference 候选，结合上下文、商品类型、图片主体、OCR 和多模态 grounding，判断它是“当前商品型号”还是“适配/提及的其它型号”；只有 owned reference 才能进入 canonical exact-match 主链路。**

这会直接提升当前系统最重要的指标：

> **减少 false positive，而不是单纯提高 recall。**

这也是 MM-LTP 思想在“同一 reference = 同一商品、绝不误匹配”的业务约束下最合理的改造方式。