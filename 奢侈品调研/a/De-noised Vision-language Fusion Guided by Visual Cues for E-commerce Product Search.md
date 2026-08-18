# De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search：从 MM-LTP 到 Precision-First 腕表 Reference Matching 的落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取尚未由 `a` 分析过的论文：

**De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search**（CVPR 2024 Workshop）。

- CVF：<https://openaccess.thecvf.com/content/CVPR2024W/MULA/html/Hu_De-noised_Vision-language_Fusion_Guided_by_Visual_Cues_for_E-commerce_Product_CVPRW_2024_paper.html>
- 作者 PDF：<https://kevin-hu.com/publication/mula_2024/mula_2024.pdf>
- Amazon Science：<https://www.amazon.science/publications/de-noised-vision-language-fusion-guided-by-visual-cues-for-e-commerce-product-search>
- 目标 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前已检查 `奢侈品调研/a`，当前目录不存在该论文对应结果，因此不是重复分析。

当前 Spec 的核心约束是：

```text
数据源：雷小安 / 腕表之家 / 奢当家
规模：100 万 ~ 1000 万商品，持续增量
同一商品定义：同一 reference number / 型号
字段：高度稀疏，reference 可能在结构化字段，也可能埋在标题
图片：可用
人工：可接受几百对黄金标签
最重要约束：绝对不能误匹配，precision 极端优先，允许漏匹配
```

这篇论文解决的恰好是一个容易被忽略、但会直接制造 false positive 的问题：

> 电商标题中经常混入大量“与当前商品图片并不对应”的词，例如营销词、功能描述、兼容对象、配件适配型号、品牌堆词等；如果把整段标题无差别地与图片做多模态融合，模型会学到错误的图文对齐。

论文提出 **MM-LTP（MultiModal alignment-guided Learned Token Pruning）**，利用多模态 attention 自动学习哪些文本 token 应该被抑制，从而在训练过程中完成 online text denoising。

但对当前腕表 matching 需求，有一个必须先指出的关键矛盾：

> **原版 MM-LTP 不能直接照搬。**

原因是：论文倾向于把“不具视觉描述性”的 token 视为噪声，而腕表 `reference number` 本身很可能就是**非视觉描述 token**。例如标题里的 `126610LV`、`311.30.42.30.01.005`，在商品正面图里未必能看到对应字符，但它却是当前业务定义里最重要、甚至是最终唯一的 identity key。

因此本次方案不是“复现 MM-LTP，然后用图文 embedding cosine threshold 直接判同款”，而是把它改造成一个 **Protected-Identifier MM-LTP（下文简称 PI-MM-LTP）**：

```text
Reference / 型号 token：永远保护，不允许被视觉剪枝
普通标题 token：允许由视觉 attention 去噪
图片：只负责候选召回、文本去噪、冲突检测、人工复核辅助
最终 AUTO_LINK：只能由 canonical reference 的确定性证据门禁放行
```

推荐最终架构：

```text
             ┌────────────────────────┐
             │ 三来源原始商品增量数据 │
             └────────────┬───────────┘
                          ▼
                Normalization Layer
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
结构化 Reference      标题/描述抽取        图片 OCR
       │                  │                  │
       └──────────┬───────┴───────┬──────────┘
                  ▼               ▼
          Reference Role      Image Role
          Classification     Classification
                  │               │
                  └──────┬────────┘
                         ▼
                Reference Evidence Store
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
      Exact Reference Index    PI-MM-LTP Encoder
              │             （标题视觉去噪 + embedding）
              │                     │
              └──────────┬──────────┘
                         ▼
                 Candidate Generator
                         │
                         ▼
             Multimodal Conflict Verifier
                         │
                         ▼
              Precision-First Decision Gate
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          AUTO_LINK   NEED_REVIEW  REJECT
              │
              ▼
        Global Reference Entity
    key = (brand_id, canonical_reference)
```

一句话原则：

> **MM-LTP 负责把“脏标题”变得更适合做候选召回和冲突识别；Reference Evidence 负责最终身份判定。**

---

## 1. 原论文解决的真实问题

### 1.1 电商图文数据为什么天然很脏

论文指出，真实电商训练数据通常自动采集，缺少逐条人工清洗，因此商品图片和商品标题并不是严格对齐的。

卖家为了搜索曝光，往往在标题里塞入大量属性：

```text
产品主体名称
品牌
颜色
尺寸
功能
营销词
兼容型号
适用人群
材质
认证
配件信息
系列名
平台关键词
```

其中只有一部分能从图片中直接观察到。

论文展示的核心问题可以抽象成：

```text
Image = 当前正在卖的商品
Title = 当前商品信息 + 非视觉属性 + 搜索关键词 + 冗余信息
```

如果训练模型时直接：

```text
full_title <-> product_image
```

那么模型会被迫把所有 title token 都和图片建立联系，即使某些词根本没有视觉对应物。

这会造成：

1. 图像 encoder 被错误文本监督污染；
2. 文本 encoder 过度拟合营销/功能词；
3. 跨模态 embedding 被“标题垃圾信息”拖偏；
4. 下游视觉检索更容易出现局部视觉 false positive。

这与二奢数据非常同构。

### 1.2 映射到腕表场景

二奢标题里常见更危险的噪声不是普通营销词，而是**看起来非常像 identifier 的字符串**：

```text
劳力士 绿水鬼 126610LV 全套
适配 Rolex 126610LN 表带
可配 116610 126610 系列
店铺 SKU: LX-126610-7788
平台货号 20260818123
柜台同款 126610LV 风格
```

如果一个系统只做 regex：

```text
抽到像型号的字符串 -> 作为 reference
```

就可能把：

```text
兼容对象 reference
配件目标 reference
店铺 SKU
平台 ID
搜索堆词
```

错误当成当前商品的 reference。

而一旦这个错误 reference 被用于跨源 exact join，由于 Spec 把 reference 定义为商品身份，错误会被放大成**高置信误合并**。

因此这篇论文最值得借鉴的不是“视觉检索效果更好”，而是一个工程思想：

> **必须识别标题里哪些 token 真正被当前商品视觉内容支持，不能默认所有标题词都属于当前商品。**

---

## 2. MM-LTP 原始技术实现

## 2.1 支持两类多模态架构

论文特意让 MM-LTP 同时适配两类模型。

### A. CLIP 类：隐式图文融合

```text
image -> vision encoder -> image embedding
text  -> text encoder   -> text embedding
                     \   /
                 contrastive loss
```

CLIP 没有文本 token 对图像 patch 的显式 cross-attention。

论文的处理方式是：

> 在 text encoder 的 self-attention 中估算 token 重要性，再通过最终 image-text alignment loss 间接学习“哪些词对图文对齐有帮助”。

### B. ALBEF 类：显式 cross-attention

```text
image -> vision encoder --------------------┐
                                             ▼
text -> text encoder -> multimodal fusion encoder
                         (cross-attention)
```

这里可以直接观察：

```text
text token -> image patch
```

的 attention，因此更容易判断某个词是否真的有视觉 grounding。

论文实验也表明，在有显式 cross-attention 的 ALBEF 上，对 self-attention + cross-attention 都做 pruning 的收益更明显。

---

## 2.2 Token Importance：如何量化某个文本 token 是否重要

Transformer attention 可以抽象成：

```text
Attn(Q, K) = QK^T / sqrt(d)
```

对某个 query token `x_i`，最直接的做法是把它对所有 key、所有 attention head 的分数取平均。

论文进一步发现：

> 与其对所有 Key token 平均，不如看 Query token 对 Key 的 `[CLS]` 聚合表示的 attention。

因此可简化理解成：

```text
importance(token_i)
    = mean_head Attention(token_i, key_CLS)
```

在 ALBEF cross-attention 中：

```text
Query = text token
Key   = image token
key_CLS = 聚合后的视觉概念
```

在 CLIP text self-attention 中：

```text
Query = text token
Key   = text token
key_CLS = 聚合后的文本上下文
```

这样得到每个文本 token 的 importance score。

论文的 ablation 显示，利用 `[CLS]` 聚合信息得到的重要性计算方案比直接粗暴平均所有 key 更好，作者把它解释为又做了一次降噪。

---

## 2.3 Learnable Threshold：不是手写阈值，而是每层学习

如果已经有：

```text
s_i = importance(token_i)
```

最简单的方法是：

```text
if s_i < 0.2:
    prune(token_i)
```

但不同模型、数据集、层的 attention 分布完全不同，所以论文认为静态 threshold 不可靠。

于是每一层学习一个阈值：

```text
tau_l
```

并通过 tempered sigmoid 得到 soft mask：

```text
M_l(token_i)
    = sigmoid((s_l(token_i) - tau_l) / T)
```

论文实现中：

```text
T = 1e-4
```

当 `T` 很小时，sigmoid 会非常接近 hard binary mask：

```text
importance > threshold -> mask ≈ 1
importance < threshold -> mask ≈ 0
```

但在训练阶段仍然可微。

然后把 mask 乘到 token 的 layer output：

```text
h_i' = M_i * h_i
```

低重要性 token 不是真的立即从 tensor 中删除，而是被软抑制，后续层基本感知不到它。

这对工程实现很重要，因为：

- 不需要动态改变 tensor shape；
- 训练更稳定；
- threshold 可以 end-to-end 学习。

---

## 2.4 Pruning Loss：让模型真的愿意删 token

如果只靠原模型 loss，模型可能倾向于把所有 token 都保留。

因此论文额外加入 pruning loss，大体上是对各层 mask 的 L1 做归一化：

```text
L_prune
    = mean_layer( ||M_l||_1 / sequence_length_l )
```

总损失：

```text
L = L_model + lambda * L_prune
```

论文实验：

```text
lambda = 0.1
```

这相当于告诉模型：

> 在不损伤原任务的情况下，尽量少保留无用 token。

---

## 2.5 原论文工程参数

论文实验使用：

```text
Framework: PyTorch
Distributed: Ray
GPU: 8 x NVIDIA A100
Vision backbone: ViT-B/16
CLIP vision encoder: 12 layers / 86M parameters
CLIP text encoder: 12 layers / 63M parameters
ALBEF text + fusion: 6-layer transformer，整体约 124M
```

CLIP fine-tuning：

```text
batch size = 1360
AdamW
weight decay = 0.02
learning rate:
    5e-6 -> warmup 到 2e-5 -> cosine decay 到 5e-6
100+ epochs
```

Pruning：

```text
layer threshold：线性递增初始化
final layer threshold = 0.01
T = 1e-4
lambda = 0.1
```

这些参数本身不建议照抄到腕表数据，但说明 MM-LTP 的实现并不依赖特殊算子，本质上只是：

```text
attention statistic
+ learnable threshold
+ differentiable mask
+ one regularization loss
```

因此可以比较容易地挂到现有 Transformer 上。

---

## 3. 论文实验结果说明了什么、又没有说明什么

论文构建了约 71.7 万 Amazon 商品的多模态数据集：

```text
Train products: 637,511
Test products:   80,000
Total products: 717,511
Train pairs:     858,231
Test pairs:      186,084
Total pairs:   1,044,315
```

它主要评估 image retrieval 的 Recall@K。

### CLIP first-stage fine-tuning

```text
CLIP-FT1:
R@1  = 53.59
R@5  = 63.24
R@10 = 69.03

CLIP-FT1 + MM-LTP:
R@1  = 55.21
R@5  = 65.13
R@10 = 70.97
```

R@1 约提升 1.62 个百分点。

### ALBEF first-stage fine-tuning

```text
ALBEF-FT1:
R@1  = 51.68
R@5  = 62.27
R@10 = 68.68

ALBEF-FT1 + cross-attention pruning:
R@1  = 54.59

ALBEF-FT1 + self-attention + cross-attention pruning:
R@1  = 57.06
R@5  = 67.54
R@10 = 73.74
```

全部 pruning 时 R@1 相对 baseline 提升约 5.38 个百分点。

论文还通过 Grad-CAM 显示：

- 视觉描述词的 attention 更集中；
- 品牌名或没有直接视觉对应的词，attention 更分散。

这支持“视觉信号能够帮助区分标题 token 是否有视觉 grounding”。

但是必须注意：

> **Recall@K 提升不等于 entity matching precision 提升。**

论文没有证明：

```text
embedding similarity 高 -> 一定是同一个 reference
```

也没有评估腕表这种：

```text
同品牌
同系列
外观几乎一样
reference 只差 1~2 字符
```

的极端 hard negative。

所以我们只能借鉴其**去噪机制**，不能把论文的 retrieval 结论外推为自动合并依据。

---

## 4. 原版 MM-LTP 在腕表 Reference Matching 中最危险的点

这是本次分析最重要的结论。

论文的问题定义是：

```text
视觉无法体现的文本 token 往往是噪声
```

但当前业务中：

```text
视觉无法体现的 token
```

可能正好是：

```text
126610LV
5711/1A-010
311.30.42.30.01.005
IW500705
```

也就是说：

```text
non-visual token != useless token
```

对商品视觉搜索，brand/reference 不是视觉词，可能应该弱化；

对 reference matching，它们却是最重要的 identity evidence。

如果直接复现原版算法，很可能出现：

```text
标题：Rolex Submariner 126610LV 黑盘 绿圈 全套

视觉 attention：
Rolex       低
Submariner  中
126610LV    低
黑盘         高
绿圈         高
全套         低

原 MM-LTP：
可能把 126610LV 一起 soft-prune
```

这会让模型最终只看到：

```text
Submariner 黑盘 绿圈
```

而这恰恰会放大同系列 sibling references 的误匹配风险。

因此必须增加一层业务先验：

> **identifier 的重要性不能由视觉 attention 决定。**

---

## 5. 改造方案：Protected-Identifier MM-LTP（PI-MM-LTP）

## 5.1 Token Role 必须先于 Token Pruning

在进入多模态模型前，先对标题中的 span 做角色识别。

建议定义：

```text
PROTECTED_REFERENCE
OWN_PRODUCT_REFERENCE
COMPATIBILITY_REFERENCE
PLATFORM_SKU
SHOP_SKU
SERIES_NAME
BRAND
VISUAL_ATTRIBUTE
NON_VISUAL_ATTRIBUTE
MARKETING_NOISE
UNKNOWN
```

其中：

```text
PROTECTED_REFERENCE
```

不是最终语义角色，而是一个安全标记：

> 任何“有可能是 reference 的字符串”，在 token pruning 阶段都不能删。

后续再由 Reference Role Classifier 判定它到底属于：

```text
当前商品 reference
兼容对象 reference
平台 SKU
店铺 SKU
其它编号
```

这样避免：

```text
模型还没搞清楚它是什么 -> 先把它剪掉
```

---

## 5.2 Subword Tokenization 也要保护

腕表 reference 经常被 tokenizer 拆开：

```text
126610LV
-> 126 / 610 / LV
```

或者：

```text
311.30.42.30.01.005
-> 311 / . / 30 / . / 42 / ...
```

所以保护逻辑必须发生在 tokenization 之前：

```text
raw title
  -> char-level ReferenceSpanDetector
  -> span offsets
  -> tokenizer
  -> offset mapping
  -> 将 span 对应的所有 subword 标记 protected
```

不能只在 tokenized sequence 上跑 regex。

---

## 5.3 改造 soft mask

原论文：

```text
M_i = sigmoid((s_i - tau) / T)
```

改造后：

```text
if protected_identifier(i):
    M_i = 1
else:
    M_i = sigmoid((s_i - tau) / T)
```

数学上可写为：

```text
M'_i = P_i + (1 - P_i) * M_i

P_i = 1  -> identifier protected
P_i = 0  -> normal MM-LTP pruning
```

这样可以保证：

```text
reference token pruning rate = 0
```

这应成为生产监控里的硬指标。

---

## 5.4 再增加 Identifier Preservation Loss

仅写死 mask 已经足够安全，但训练时还可以增加一项：

```text
L_id_preserve
    = mean_{i in protected}(1 - M_i)^2
```

总目标：

```text
L_total
  = L_multimodal
  + lambda_prune * L_prune
  + lambda_id * L_id_preserve
  + lambda_role * L_reference_role
```

其中 `lambda_id` 应明显高于普通 pruning regularization。

更直接的工程实现甚至可以：

```text
protected token mask 完全 stop-gradient，固定为 1
```

这样没有任何学习过程能把 identifier 删掉。

对于当前“绝对不能误匹配”的约束，我更推荐固定保护，而不是依赖 loss 惩罚。

---

## 6. Reference Role Classifier：解决“标题里出现型号，但不是当前商品型号”

这是仅靠 MM-LTP 仍然解决不了的问题。

示例：

```text
Rolex 126610LN 专用橡胶表带
适配 126610LN / 126610LV
```

图片里可能真的有一个表带。

模型可能认为：

```text
表带
黑色
橡胶
```

有视觉 grounding，而：

```text
126610LN
```

视觉 grounding 弱。

这只能说明 reference 不是视觉属性，不能说明 reference 是否属于当前商品。

因此要单独做 `Reference Role Classification`。

推荐输入：

```text
candidate reference span
+ 左右上下文窗口
+ 完整标题
+ 商品类别
+ 来源
+ 图片 category / image role
```

输出：

```text
OWN_REFERENCE
COMPATIBILITY_TARGET
ACCESSORY_TARGET
SHOP_SKU
PLATFORM_ID
UNKNOWN_ID
```

### 训练样本重点

黄金标签不应大量浪费在显然不同品牌的 pair 上，而应重点标：

```text
“标题里这个编号到底是谁的编号？”
```

例如 300~500 条人工可以优先覆盖：

```text
腕表本体
表带
表盒
保卡
替换件
表扣
镜片
兼容配件
维修服务
同系列堆词
店铺内部编码
```

这个小分类任务对 precision 的贡献，可能比再训练一个更大的 matching 模型更高。

---

## 7. Reference Evidence Store：不要只存一个 extracted_ref

如果只存：

```json
{"reference": "126610LV"}
```

后续根本无法审计为什么得到这个结论。

建议保存完整 evidence。

### 表：`reference_evidence`

```text
id
record_id
source
raw_value
normalized_value
canonical_reference
span_start
span_end
extraction_method
role
role_confidence
evidence_strength
image_id
ocr_bbox
model_version
rule_version
created_at
```

`extraction_method`：

```text
STRUCTURED_FIELD
TITLE_REGEX
TITLE_NER
DESCRIPTION_NER
IMAGE_OCR_CASEBACK
IMAGE_OCR_CARD
IMAGE_OCR_TAG
MANUAL
```

`role`：

```text
OWN_REFERENCE
COMPATIBILITY_TARGET
PLATFORM_SKU
SHOP_SKU
UNKNOWN
```

最终商品表不要只保留单值，而是通过 evidence resolver 得到：

```text
resolved_reference
resolved_reference_status
resolved_reference_confidence
conflict_count
```

---

## 8. Canonical Reference Normalization：只能做“可证明安全”的归一化

腕表 reference 不适合通用 aggressive normalization。

### 安全操作

```text
Unicode NFKC
trim
统一大小写
移除明确无语义的首尾空白
按品牌白名单统一已知分隔符
```

### 高风险操作

不能全局：

```text
删除所有 - / . 空格
O -> 0
I -> 1
S -> 5
删除后缀
只保留数字
模糊 edit-distance 合并
```

这些操作可以用于：

```text
候选召回
OCR纠错候选
```

但不能直接改变 canonical key。

建议每个品牌有：

```text
brand_reference_normalizer
brand_reference_validator
```

例如：

```python
canonical_ref = normalize(raw_ref, brand_rules)
valid = validate(canonical_ref, brand_rules)
```

而不是一个全平台统一 regex。

---

## 9. 图片在系统中的四种角色

当前 Spec 有图片，但图片不应该只有一个 embedding。

## 9.1 Candidate Recall

当 reference 缺失时：

```text
图片 + 去噪标题 -> multimodal embedding
```

在品牌/系列内部召回 top-K 可能候选。

但：

```text
embedding top1 != AUTO_LINK
```

它只产生候选。

## 9.2 Title Denoising

这是 MM-LTP 最直接的用途。

例如：

```text
劳力士 绿水鬼 现货 专柜同款 原装 全套 送礼 126610LV
```

PI-MM-LTP 可弱化：

```text
现货 / 专柜同款 / 送礼
```

保留：

```text
劳力士 / 绿水鬼 / 126610LV
```

## 9.3 Conflict Detector

如果标题说：

```text
腕表本体 126610LV
```

但所有图片都被 `ImageRoleClassifier` 判成：

```text
STRAP / BOX / ACCESSORY
```

则不能自动信任标题 reference。

应输出：

```text
PRODUCT_TYPE_CONFLICT
```

## 9.4 OCR Evidence

图片先分类：

```text
WATCH_FRONT
WATCH_BACK
WARRANTY_CARD
TAG
BOX
ACCESSORY
OTHER
```

只有高价值 image role 才跑 reference OCR：

```text
WATCH_BACK
WARRANTY_CARD
TAG
```

避免所有图片都做昂贵 OCR。

OCR 结果必须保存：

```text
raw text
bbox
image role
OCR confidence
normalization candidates
```

并且 OCR 相似字符修正只能进入 candidate set，不直接 canonicalize。

---

## 10. 完整 Matching Pipeline

## 10.1 Stage 0：数据标准化

```text
record_id
source
source_item_id
brand_raw
brand_id
category
raw_title
raw_description
structured_ref_raw
images[]
updated_at
```

标准化：

```text
brand entity linking
Unicode cleanup
HTML cleanup
source-specific field mapping
image dedup
```

---

## 10.2 Stage 1：Reference Extraction

按优先级抽：

```text
1. 结构化 reference 字段
2. title reference candidates
3. description candidates
4. 高价值图片 OCR
```

输出不是一个字符串，而是一组 evidence。

---

## 10.3 Stage 2：Reference Role + Conflict Resolution

每个候选编号做：

```text
role classification
brand grammar validation
context validation
cross-field consistency
OCR consistency
```

结果：

```text
CONFIRMED_OWN_REFERENCE
LIKELY_OWN_REFERENCE
AMBIGUOUS_REFERENCE
CONFLICT_REFERENCE
NO_REFERENCE
```

---

## 10.4 Stage 3：Exact Blocking

如果有：

```text
brand_id
canonical_reference
```

先直接查：

```text
reference_index[(brand_id, canonical_reference)]
```

这一步复杂度近似 O(N)，不需要全量 pairwise comparison。

在 1000 万级数据上非常关键。

理论上的笛卡尔积：

```text
10,000,000^2
```

完全不可接受。

而 reference hash index：

```text
key -> list<record_id>
```

可以把绝大多数确定性匹配变成常数级查找。

---

## 10.5 Stage 4：PI-MM-LTP Candidate Recall

只针对：

```text
reference 缺失
reference 模糊
存在冲突
新品牌/新格式
```

才进入 embedding recall。

建议 partition：

```text
brand_id
+ product_type
+ optional series
```

然后在 partition 内：

```text
HNSW / FAISS topK = 20~100
```

输入 embedding：

```text
E_text  = denoised_title_encoder(title)
E_image = aggregate(image_embeddings)
E_mm    = fusion(E_text, E_image)
```

这里可以用 PI-MM-LTP 训练的 ALBEF-like encoder，也可以 MVP 先用较轻方案。

候选召回目标是：

```text
candidate recall 尽量高
```

不是最终 precision。

---

## 10.6 Stage 5：Multimodal Conflict Verification

对候选 pair `(A, B)` 计算：

```text
brand_equal
canonical_ref_equal
ref_edit_distance
ref_role_conflict
structured_ref_conflict
ocr_ref_conflict
title_ref_conflict
image_type_conflict
text_similarity
image_similarity
multimodal_similarity
same_family_hard_negative_score
stock_image_flag
```

其中最重要的是：

```text
conflict > similarity
```

即：

> 强冲突具有否决权，相似度没有越权权力。

---

## 10.7 Stage 6：Precision-First Decision Gate

推荐不是普通二分类：

```text
MATCH / NOT_MATCH
```

而是三态：

```text
AUTO_LINK
NEED_REVIEW
REJECT
```

### 推荐规则

#### AUTO_LINK

必须满足：

```text
brand_id 相同
AND canonical_reference 完全相同
AND A 的 reference 是 OWN_REFERENCE
AND B 的 reference 是 OWN_REFERENCE
AND 两边至少达到最低 evidence strength
AND 没有任何 strong conflict
```

图片、embedding 只能：

```text
提高 confidence
或制造 conflict
```

不能替代 reference exact evidence。

#### REJECT

任一：

```text
强 reference 不同
brand 不同
一边 reference 被判定为 compatibility target
商品类型冲突
高置信 OCR / 结构化字段与标题强冲突
```

#### NEED_REVIEW

```text
只有一边有强 reference
reference 缺失
OCR 模糊
标题抽取角色不确定
embedding 很像但 reference 未确认
```

这符合 Spec 的业务哲学：

> 漏掉可以，误合并不可以。

---

## 11. 为什么最终应该“按 Reference 建实体”，而不是 pairwise 聚类

业务定义已经给得很明确：

```text
same product = same reference number
```

因此全局实体建议直接定义：

```text
reference_entity_id
brand_id
canonical_reference
```

唯一键：

```text
UNIQUE(brand_id, canonical_reference)
```

每条 source record 通过经过验证的 evidence 绑定到这个实体。

不要让模型先做：

```text
A ~ B
B ~ C
=> A,B,C cluster
```

否则一条 false-positive edge 可能污染整个 cluster。

更安全的是：

```text
Record A -> verified ref -> Entity X
Record B -> verified ref -> Entity X
Record C -> verified ref -> Entity X
```

没有 verified reference 的记录：

```text
保持 unlinked
```

直到得到足够证据。

---

## 12. 训练数据怎么构造：几百条人工标签不应直接拿去训“大 Matcher”

当前最强的 supervision 其实是 reference 本身。

## 12.1 自动构造高精度 Positive

只从满足以下条件的历史记录生成：

```text
brand 相同
canonical_reference 相同
至少一个来源有结构化 ref
另一来源 title/structured/OCR 至少有独立证据
无冲突
```

生成：

```text
same-reference positive pairs
```

---

## 12.2 Hard Negative 才是训练重点

不要大量随机负采样：

```text
Rolex vs Cartier
```

这种样本没有价值。

重点采：

```text
same brand
same series
reference 只差少数字符
图片极像
标题极像
```

例如：

```text
126610LV vs 126610LN
116610LV vs 126610LV
311.30.42.30.01.005 vs 311.30.42.30.01.006
```

还要加一种非常重要的 accessory hard negative：

```text
腕表 126610LV
vs
“适配 126610LV” 的表带
```

这类负例对 false positive 的抑制价值最大。

---

## 12.3 几百条黄金标签建议怎么花

建议不要全标 pair matching。

例如 500 条预算可以拆成：

```text
150 条：Reference Role Classification
150 条：同系列 hard-negative pair
100 条：OCR / title / structured ref 冲突案例
100 条：最终 AUTO_LINK / REVIEW 阈值校准
```

或者根据误差分析动态调整。

人工标签应优先覆盖：

```text
系统最不确定
最容易误匹配
最接近自动放行边界
```

而不是随机抽样。

---

## 13. PI-MM-LTP 的训练目标建议

可以把论文原始目标扩展成：

```text
L_total =
    L_image_text_alignment
  + lambda_prune * L_prune
  + lambda_role  * L_reference_role
  + lambda_hn    * L_hard_negative
  + lambda_cons  * L_intra_record_consistency
```

其中 identifier mask 直接强制固定为 1，不建议只靠 loss。

### `L_image_text_alignment`

保持论文 CLIP / ALBEF 类目标。

### `L_prune`

让普通标题 token 尽量干净。

### `L_reference_role`

识别 candidate reference 的语义角色。

### `L_hard_negative`

重点拉开 sibling reference：

```text
same series + different canonical_reference
```

### `L_intra_record_consistency`

约束同一条记录内部：

```text
structured_ref
<-> title own_ref
<-> OCR own_ref
```

的一致性。

---

## 14. 一个比“完全复现论文”更快的 MVP

完整训练 MM-LTP 不是第一阶段的必需品。

可以先做一个 **MM-LTP-inspired Zero/Low-Training Denoiser**。

### Step 1：Title phrase segmentation

把标题切成 phrase：

```text
[品牌]
[系列]
[reference candidate]
[颜色]
[材质]
[营销词]
[兼容描述]
```

### Step 2：ReferenceSpanDetector

任何可能 reference：

```text
PROTECTED
```

### Step 3：Visual grounding score

用现成 CLIP/SigLIP/BLIP 类模型计算：

```text
sim(phrase_embedding, image_embedding)
```

或者使用 cross-attention。

### Step 4：对非 protected phrase 降权

```text
final_phrase_weight = visual_grounding_score
```

高风险营销/兼容 phrase 可以进一步基于规则降权。

### Step 5：重建 denoised title

```text
brand + series + protected_reference + high-grounding-visual-attributes
```

用于：

```text
candidate retrieval
pair verifier features
```

这可以在没有重训练大型模型的情况下快速验证：

> 视觉去噪是否真的能降低三来源候选中的 false positive。

如果有效，再训练 PI-MM-LTP。

---

## 15. 生产数据模型建议

## 15.1 `product_record`

```text
record_id
source
source_item_id
brand_id
product_type
raw_title
raw_description
resolved_reference
resolved_reference_status
source_updated_at
ingested_at
```

## 15.2 `product_image`

```text
image_id
record_id
url/object_key
phash
image_role
role_confidence
embedding_version
embedding_key
```

## 15.3 `reference_evidence`

```text
record_id
raw_value
canonical_reference
method
role
role_confidence
evidence_strength
image_id
ocr_bbox
model_version
rule_version
```

## 15.4 `reference_entity`

```text
reference_entity_id
brand_id
canonical_reference
created_at
```

唯一约束：

```text
UNIQUE(brand_id, canonical_reference)
```

## 15.5 `match_decision`

```text
decision_id
record_id
reference_entity_id
candidate_record_id
decision
reason_codes
feature_snapshot
rule_version
model_version
created_at
```

`reason_codes` 示例：

```text
EXACT_CANONICAL_REFERENCE
STRUCTURED_REF_SUPPORT
TITLE_OWN_REF_SUPPORT
OCR_SUPPORT
COMPATIBILITY_REFERENCE_REJECT
STRONG_REF_CONFLICT
IMAGE_TYPE_CONFLICT
MISSING_REFERENCE
```

这样任何 AUTO_LINK 都可以追溯。

---

## 16. 决策 Gate 的推荐伪代码

```python
def decide(record_a, record_b):
    # 1. 品牌先硬门禁
    if record_a.brand_id != record_b.brand_id:
        return REJECT("BRAND_CONFLICT")

    # 2. 强冲突永远优先于相似度
    if has_strong_reference_conflict(record_a, record_b):
        return REJECT("STRONG_REF_CONFLICT")

    if has_product_type_conflict(record_a, record_b):
        return REJECT("PRODUCT_TYPE_CONFLICT")

    # 3. reference 必须是当前商品自身编号
    ref_a = resolve_own_reference(record_a)
    ref_b = resolve_own_reference(record_b)

    if ref_a.is_compatibility_target or ref_b.is_compatibility_target:
        return REJECT("COMPATIBILITY_REFERENCE")

    # 4. 缺 reference 时，embedding 再高也不能自动合并
    if not ref_a.confirmed or not ref_b.confirmed:
        return NEED_REVIEW("REFERENCE_NOT_CONFIRMED")

    # 5. canonical reference 不同 -> reject
    if ref_a.canonical != ref_b.canonical:
        return REJECT("CANONICAL_REFERENCE_DIFFERENT")

    # 6. 多模态只做额外 conflict check
    mm = multimodal_verify(record_a, record_b)
    if mm.strong_conflict:
        return NEED_REVIEW("MULTIMODAL_CONFLICT")

    # 7. 最终由 exact reference 放行
    return AUTO_LINK(
        reason="EXACT_CANONICAL_REFERENCE",
        evidence=[ref_a.evidence, ref_b.evidence],
    )
```

注意：

```text
multimodal_similarity
```

没有任何一行可以直接：

```python
if cosine > threshold:
    AUTO_LINK
```

这是刻意设计的。

---

## 17. 百万到千万级数据的工程架构

建议拆成离线/增量两条链路。

## 17.1 离线全量 Backfill

```text
Object Storage / Raw DB
        │
        ▼
Spark / Ray Batch Normalize
        │
        ▼
Reference Extraction
        │
        ├──> Evidence DB
        │
        ▼
Image Role + OCR + Embedding GPU Batch
        │
        ▼
Exact Reference Index
        │
        ▼
Unresolved Records Only
        │
        ▼
Vector Candidate Recall
        │
        ▼
Verifier + Decision Gate
```

论文已经使用 Ray 做多 GPU 实验，因此如果现有团队熟悉 Ray，可以直接沿用到：

```text
OCR batch
image embedding
PI-MM-LTP inference
pair verification
```

但这不是必须绑定的技术栈。

---

## 17.2 持续增量

一个商品更新时，只重新处理它，不重新跑全库 pairwise matching。

```text
new/updated record
    -> normalize
    -> extract reference evidence
    -> image pipeline if image changed
    -> resolve reference
```

如果 reference confirmed：

```text
reference_index lookup
    -> attach to reference_entity
    -> conflict audit
```

如果 unresolved：

```text
brand partition vector search
    -> top-K candidates
    -> verifier
    -> REVIEW / REJECT
```

复杂度从：

```text
O(N^2)
```

下降为近似：

```text
O(N) preprocessing
+ O(log N) ANN query / record
```

而且真正进入模型 pair verifier 的只是小部分 unresolved 数据。

---

## 18. 推荐技术栈

这不是强绑定，但一个比较现实的组合是：

### 计算

```text
Python
PyTorch
Ray Data / Ray Train（GPU batch）
```

### 结构化存储

```text
PostgreSQL：核心 record / entity / decision
ClickHouse：离线指标、模型分析、错误统计（可选）
```

### 图片

```text
S3 / OSS / COS 等对象存储
```

### 向量

MVP：

```text
FAISS
```

服务化：

```text
Milvus / Qdrant / pgvector
```

前提是按 brand 分区，避免全库无约束视觉近邻。

### 队列/增量

```text
Kafka / Pulsar / 云消息队列
```

如果数据更新频率不高，也可以先用数据库 job queue，不需要第一天就引入 Kafka。

---

## 19. 评估指标必须从论文的 Recall@K 改成 Precision-First

论文用：

```text
Recall@1 / Recall@5 / Recall@10
```

但当前项目第一指标应该是：

```text
AUTO_LINK Precision
```

推荐 dashboard：

```text
1. AUTO_LINK precision
2. false-positive count
3. false-positive rate
4. auto-link coverage
5. abstention / review rate
6. candidate recall@K
7. confirmed-reference extraction precision
8. reference-role precision
9. OCR own-reference precision
10. protected reference token pruning rate
```

其中：

```text
protected reference token pruning rate = 0
```

应做 hard assertion。

---

## 20. 测试集必须专门包含“最像但不是同款”的腕表

普通随机切分会严重高估效果。

建议单独构造：

### Sibling Reference Set

```text
同品牌
同系列
reference 只差 1~3 字符
外观非常接近
```

### Compatibility Trap Set

```text
配件标题出现腕表 reference
```

### OCR Trap Set

```text
0/O
1/I/L
5/S
8/B
```

### Stock Image Trap Set

不同 reference 使用相同或相似官方宣传图。

### Sparse Field Set

```text
只标题
只图片
只有一个模糊 reference candidate
```

### Cross-source Drift Set

分别统计：

```text
雷小安 -> 腕表之家
雷小安 -> 奢当家
腕表之家 -> 奢当家
```

不要只报整体平均值。

---

## 21. Image Similarity 不能作为正向强证据的原因

腕表是典型 fine-grained 商品：

```text
同系列不同 reference
```

可能共享：

```text
表壳
表链
盘面布局
宣传图角度
甚至官方 stock image
```

因此：

```text
image_similarity 高
```

只说明：

```text
值得进入候选
```

不能说明：

```text
同 reference
```

反过来，图片可以做一些**负向证据**：

```text
一个是腕表本体，一个是表带 -> conflict
一个是圆形表，一个是方形表 -> conflict
一个主色/结构完全不符 -> conflict
```

因此多模态在 precision-first 系统中应更偏向：

```text
recall + veto
```

而不是：

```text
final judge
```

---

## 22. 与 `a` 已有调研的互补关系

`a` 目录已有多项工作覆盖：

```text
Blocking
Deep Entity Matching
多模态 Product Matching
Selective / Conformal Prediction
Transitive Consistency
Reference Extraction / Normalization
```

本篇不是重复再做一个 Matcher，而是补一个此前容易被忽略的环节：

```text
“进入 Matcher 之前，title 里的词到底哪些应该相信？”
```

尤其是：

```text
兼容型号
平台 SKU
营销堆词
配件目标 reference
```

这类噪声如果没在输入层处理好，后面的任何高阶 matching 模型都会被污染。

因此本方案建议把 PI-MM-LTP 放在：

```text
ReferenceSpan protection
        ↓
Title visual denoising
        ↓
Candidate representation
```

这个位置，而不是替换已有的 reference hard gate。

---

## 23. 最小可落地方案（建议优先做）

如果目标是最快拿到可上线结果，我建议不要先训练完整 ALBEF + MM-LTP，而是按以下顺序。

### Phase 1：Reference-First Baseline

先完成：

```text
brand normalization
reference regex / NER
brand-specific canonicalization
Reference Evidence Store
exact reference index
three-state decision gate
```

这一阶段不需要多模态模型就可以上线第一版。

### Phase 2：Reference Role Classifier

解决：

```text
OWN_REFERENCE
vs
COMPATIBILITY_TARGET
vs
SKU
```

优先吃掉最危险的 false positive。

### Phase 3：图片角色 + OCR

```text
WATCH_BACK
WARRANTY_CARD
TAG
```

优先提供 identifier evidence。

### Phase 4：MM-LTP-inspired 视觉去噪

先做 phrase-level visual grounding，不训练复杂模型。

目标：

```text
降低 unresolved candidate pair 数
降低 compatibility/marketing noise 导致的错误召回
```

### Phase 5：PI-MM-LTP Fine-tuning

当已经有足够 pseudo-label：

```text
高置信 same-reference positives
same-family hard negatives
compatibility negatives
```

再训练完整模型。

---

## 24. 预期收益拆分

不要把所有收益写成“matching accuracy 提升”。

PI-MM-LTP 在当前架构的收益应拆成：

### 收益 1：标题 embedding 更干净

降低：

```text
营销词
功能词
无关兼容型号上下文
```

对 candidate retrieval 的污染。

### 收益 2：减少 accessory false positive

图片帮助识别：

```text
标题说 Rolex 126610
但卖的是表带/表盒
```

### 收益 3：减少跨来源字段稀疏带来的召回损失

当一侧 reference 缺失时，去噪后的 title+image embedding 可以帮助找到正确候选，进入人工复核。

### 收益 4：提高可解释性

每个 token 可以输出：

```text
protected?
attention importance
mask score
reference role
```

人工复核界面可以直接高亮：

```text
保留词 / 降权词 / reference 候选 / 冲突词
```

这对排查错误非常实用。

---

## 25. 生产可观测性

每次 AUTO_LINK 必须保存：

```text
record A/B
canonical reference
raw reference evidence
reference role
source
规则版本
模型版本
image/OCR evidence
conflict flags
decision reason
```

每次模型发布后，应跑固定 regression set。

关键告警：

```text
AUTO_LINK false positive > 0
protected reference pruning > 0
某品牌 reference conflict 激增
某来源 OWN_REFERENCE role precision 下降
OCR 字符混淆率突增
```

对于这个系统，**一次误合并应该被当成事故样本，而不是被平均 F1 掩盖。**

---

## 26. 论文方案的局限性

### 26.1 论文是视觉检索，不是实体解析

不能把结果直接外推为 matching precision。

### 26.2 论文数据是英文 Amazon 商品

当前三源很可能包含：

```text
中文
英文
品牌缩写
数字/字母混合 reference
OCR 字符
```

需要重新验证 tokenizer 与 attention 行为。

### 26.3 非视觉 token 不一定是噪声

对 reference matching，这是最大 domain mismatch，因此必须有 identifier protection。

### 26.4 图片可能是假信号

例如：

```text
stock photo
重复宣传图
配件图
盒证图
```

图片不是 identity proof。

### 26.5 论文没有 selective abstention

当前系统必须额外加入：

```text
NEED_REVIEW
```

并接受 coverage 下降。

---

## 27. 推荐的最终生产 Policy

```text
Policy 1:
Reference identity evidence 永远高于 embedding similarity。

Policy 2:
Reference token 永远不能被视觉 denoiser 删除。

Policy 3:
图片只能用于 candidate recall、token denoising、OCR 和 conflict veto。

Policy 4:
只有双方 OWN canonical reference exact equal 才能 AUTO_LINK。

Policy 5:
任何强 reference conflict 直接 REJECT，不做相似度投票。

Policy 6:
一边无 confirmed reference 时，不允许 embedding 自动补齐身份，只能 REVIEW。

Policy 7:
按 (brand_id, canonical_reference) 建全局实体，不用无约束 pairwise transitive clustering。

Policy 8:
所有 AUTO_LINK 必须可解释、可回放、可按规则/模型版本重算。
```

---

## 28. 可直接排进开发任务的 Backlog

### P0

- [ ] 建 `brand_id + canonical_reference` 唯一实体模型
- [ ] 建 `reference_evidence` 表
- [ ] 实现 brand-specific reference normalizer / validator
- [ ] 实现 ReferenceSpanDetector
- [ ] 保护所有候选 reference span，不允许任何文本清洗删除
- [ ] 建 AUTO_LINK / REVIEW / REJECT 三态 Gate
- [ ] 强 reference conflict 一票否决

### P1

- [ ] Reference Role Classifier
- [ ] Image Role Classifier
- [ ] 高价值图片 OCR
- [ ] 同系列 hard-negative 数据集
- [ ] Compatibility Trap 数据集
- [ ] Source/brand 分桶指标

### P2

- [ ] phrase-level visual grounding MVP
- [ ] 去噪标题 embedding
- [ ] brand-partition ANN candidate recall
- [ ] 多模态 conflict verifier

### P3

- [ ] PI-MM-LTP 模型训练
- [ ] self-attention + cross-attention pruning 对比
- [ ] identifier preservation 单元测试
- [ ] token importance 可视化
- [ ] gold-set 阈值校准

---

## 29. 最终建议

这篇论文对当前项目真正值得落地的不是“再加一个多模态大模型”，而是把一个非常具体的问题工程化解决：

> **电商标题中的所有词并不都属于当前商品，必须在进入实体匹配之前做视觉引导的输入去噪。**

但腕表 reference matching 又比论文原任务更严格，因此必须反向增加一个业务硬约束：

> **Reference number 即使完全没有视觉 grounding，也必须被保护。**

因此推荐最终方案是：

```text
ReferenceSpanDetector
        │
        ├─ reference candidate -> PROTECTED
        │
        ▼
PI-MM-LTP / Visual Grounding Denoiser
        │
        ▼
Denoised Text + Image Embedding
        │
        ├─ Candidate Recall
        └─ Conflict Detection
                 │
                 ▼
Reference Evidence Resolver
                 │
                 ▼
Precision-First Hard Gate
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
   AUTO_LINK   REVIEW    REJECT
       │
       ▼
(brand_id, canonical_reference)
```

对于当前 Spec，最值得立即执行的版本不是大规模复现论文，而是：

1. 先完成 reference evidence / role / canonicalization 的确定性链路；
2. 把 MM-LTP 思想作为标题去噪和候选召回增强；
3. 图片和 embedding 永不越过 reference hard gate；
4. 用几百条人工标签重点校准 compatibility trap、同系列 sibling references 和决策边界；
5. 等积累出大量高置信 pseudo-label 后，再训练完整 PI-MM-LTP。

这样既吸收了论文的多模态去噪价值，又不会把它的视觉搜索假设错误迁移到一个“绝对不能误匹配”的 reference identity 系统里。

---

## 参考资料

1. Hu, Zhizhang, et al. **De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search.** CVPR Workshops 2024, pp. 1986-1996.  
   <https://openaccess.thecvf.com/content/CVPR2024W/MULA/html/Hu_De-noised_Vision-language_Fusion_Guided_by_Visual_Cues_for_E-commerce_Product_CVPRW_2024_paper.html>
2. 作者 PDF：<https://kevin-hu.com/publication/mula_2024/mula_2024.pdf>
3. Amazon Science publication page：<https://www.amazon.science/publications/de-noised-vision-language-fusion-guided-by-visual-cues-for-e-commerce-product-search>
4. 当前需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
