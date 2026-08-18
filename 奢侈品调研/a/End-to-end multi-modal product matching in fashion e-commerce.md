# End-to-end multi-modal product matching in fashion e-commerce：从工业级多模态召回到 Reference-First 腕表实体匹配系统

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取尚未在 `奢侈品调研/a` 分析过的论文：

- 论文：**End-to-end multi-modal product matching in fashion e-commerce**
- arXiv：<https://arxiv.org/abs/2403.11593>
- 作者：Sándor Tóth、Stephen Wilson、Alexia Tsoukara、Enric Moreu、Anton Masalovich、Lars Roemheld（Zalando）
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

这篇论文非常值得当前项目借鉴，因为它不是只做离线 benchmark，而是描述了一个真正面向大规模电商数据、跨域分布漂移和生产 Human-in-the-loop 的端到端多模态商品匹配系统。它的核心架构是：

```text
商品图片集 ──> 冻结的图像 Encoder ──> Pooling ┐
                                           │
商品文本   ──> 冻结的文本 Encoder ──────────┼─> concat ─> Linear Projection ─> 统一商品向量
                                           │
数值特征   ────────────────────────────────┘

统一商品向量
   │
   ├─> Blocking
   │
   ├─> KNN Retrieval
   │
   ├─> Similarity Threshold
   │
   └─> 高置信自动接受 / 低置信人工审核
```

论文最值得直接复用的工程思想有五点：

1. **先编码、再检索，而不是对海量商品做全量 pairwise 分类**；
2. **冻结大模型，只训练很小的线性投影层**，可预计算图片/文本 embedding，训练与增量成本极低；
3. **Blocking + ANN/KNN** 将大规模匹配压缩为很小的候选集合；
4. **把 high-precision 区域单独优化**，不要只看 F1；
5. **模型只负责筛选候选，人审只负责提高 precision，不让人工承担全库搜索**。

但是，论文不能原样应用到当前腕表需求。

当前 Spec 有一个比论文更强的业务定义：

> **“同一个商品”严格定义为同一个 reference number / 型号，并且 precision 极端优先，宁可漏匹配，不能误匹配。**

因此当前系统必须在论文架构上增加一个更强的 `Reference Gate`：

> **多模态模型负责找候选，Reference 证据负责最终放行；模型相似度永远不能越过 reference 冲突。**

最终推荐架构不是“一个更强的商品相似度模型”，而是：

```text
Reference-First Entity Linking
        +
Multimodal Candidate Retrieval
        +
Hard Conflict Gate
        +
Selective / Human Review
```

如果只做一个最重要的架构决策，就是：

> **不要把 100 万～1000 万条商品互相聚类；把每条商品链接到一个 Canonical Reference Entity。**

两个商品只有在最终链接到同一个 `(brand_id, canonical_reference)` 时才允许进入同一个实体簇。

---

## 1. 当前需求与论文问题的对应关系

### 1.1 Spec 的关键约束

当前需求具有以下特征：

- 来源：雷小安、腕表之家、奢当家；
- 数据规模：100 万～1000 万级；
- 持续增量；
- 字段非常稀疏；
- reference 有时有独立字段，有时埋在标题；
- 有图片；
- “同商品”= **同 reference**；
- precision 极端优先；
- 可以接受几百对人工黄金标签。

这里真正困难的不是“找相似商品”，而是三个问题：

1. 从脏字段、标题、图片/OCR 中**可靠找到 reference 候选**；
2. 把多个 reference 表达**保守规范化**到 canonical form；
3. 在证据不足、冲突、同系列近邻 reference 的情况下**主动拒识**。

### 1.2 论文解决了什么

论文研究的是多个卖家/域之间的“同一个 fashion product”识别。它的生产数据约有 200 万 offer、5 个 domain，每个 offer 有多张图片、品牌、标题、价格、尺码等信息。

论文的难点与当前项目高度相似：

- 不同 seller/domain 的拍摄风格差异很大；
- title 不一定唯一；
- 新 domain 会造成分布漂移；
- 大多数 index 商品根本没有对应 match；
- 最终业务需要高 precision。

论文测试集中超过 99% 的 index offer 在对应 query 集中没有匹配，这一点尤其重要：真实大规模匹配不是一个正负平衡的二分类问题，而是极端稀疏的 retrieval 问题。

### 1.3 最大差异

论文中的“same product”主要通过商品 ID 真值定义；视觉、标题、价格等都可以共同支持匹配。

当前业务则规定：

```text
same_product(a, b)
    ⇔
canonical_reference(a) == canonical_reference(b)
```

因此视觉相似并不能成为最终 match 证据。

腕表场景尤其危险：

- 同系列不同 reference 外观可能几乎一样；
- 同一 reference 不同年份/成色/表带拍摄方式差异可能很大；
- 标题里可能出现配件“适配某 reference”；
- 图片里可能出现盒证、吊牌、保卡、序列号、内部 SKU，不能把任意字母数字串当 reference；
- `126610LN` 与 `126610LV` 视觉高度接近，但必须判不同；
- 不同平台自己的 SKU 可能长得很像品牌 reference。

所以论文的 embedding + threshold 在当前项目里只能放在 **Recall Layer**，不能直接当 **Decision Layer**。

---

## 2. 论文技术架构拆解

## 2.1 两阶段：Encoding + Retrieval

论文明确把匹配拆成两步：

```text
1. Encoding
   把每个 offer 的图片、文本和数值特征编码到统一 metric space

2. Retrieval
   对 query offer 在 index offer 中找最近邻
   只保留足够近的 candidate
```

这是比 pairwise cross-encoder 更适合百万级数据的架构。

如果有 `N` 个 query、`M` 个 index，pairwise classifier 需要接近：

```text
O(N × M)
```

而 embedding retrieval 可以变成：

```text
离线编码：O(N + M)
在线查询：O(N × ANN_cost)
```

这也是当前 100 万～1000 万规模必须采用的基本范式。

## 2.2 fashionID Encoder

论文提出的 `fashionID` 结构很简单：

```text
images[1..n]
  │
  ├─ image encoder -> embedding_1
  ├─ image encoder -> embedding_2
  └─ image encoder -> embedding_n
                 │
                 ▼
             average pool
                 │
                 ├────────────┐
text -> text encoder ----------┤
                              ├-> concat -> linear -> 192d -> L2 normalize
numeric features --------------┘
```

关键点不是网络复杂，而是：

- image encoder 冻结；
- text encoder 冻结；
- 只训练最后一个小的线性投影层；
- 训练目标是 contrastive learning。

论文最佳配置使用 CLIP ViT-bigG-14 的 image/text encoder，冻结权重；拼接后只训练一个约 0.6M 参数的 linear projection，输出 192 维 embedding。

这使得预训练大模型变成一个稳定、可预计算的 feature extractor，而不是每次都端到端 fine-tune。

## 2.3 为什么冻结 Encoder 对生产很重要

论文指出：预先计算 image/text embedding 后，只训练 projection 层，50 epoch 在单张 A100 上大约 10 分钟。

这件事对当前项目的价值很大：

### 成本可控

如果图片有几千万张，真正贵的是首次 image embedding。只要原始 encoder 不换，后续：

- 改 projection；
- 改 contrastive sampling；
- 加新数值/结构化特征；
- 重新校准 threshold；

都不需要重新跑全部图片 encoder。

### 便于迭代

可以把特征分成两层缓存：

```text
Raw Feature Cache
- image_embedding_raw
- text_embedding_raw
- OCR tokens
- structured fields

Task Embedding
- projection_version
- final_match_embedding
```

以后只重算 Task Embedding，不必重新解码图片。

### 对新来源更稳

论文实验显示，更复杂的 MLP 在 validation 上略好，但简单 linear projection 在 in-domain 和 out-domain test 上反而略优，说明小容量 projection 更能保留大预训练模型原有的泛化能力。

这对持续新增来源/品牌的当前系统很有价值：不要一开始堆复杂网络，让小样本把模型拟合到某个平台的拍摄模板或标题格式。

---

## 3. Contrastive Learning 怎么训练

论文使用归一化 embedding 上的 supervised contrastive loss。

简化理解：

```text
同一个 product 的 offer embedding 要靠近
不同 product 的 offer embedding 要远离
```

### 3.1 Batch 组织比模型结构更重要

论文为了增加 batch 内正样本密度，会按 product id 抽样，把同一 product 的所有 offer 尽可能放到一个 mini-batch。

同时，大 batch 自动提供大量负样本，论文发现 batch size 增大到 16k 能明显提高效果。

由于 encoder 冻结、只训练小 projection，可以使用超大 batch，而不需要复杂的多 GPU end-to-end 训练。

### 3.2 对腕表场景必须改造成 hard-negative-first

论文里的负样本对 fashion 已经足够，但腕表的真正危险负样本不是随机负样本，而是：

```text
同品牌
+ 同系列
+ 外观高度接近
+ reference 只差一个字符/后缀
```

例如：

```text
126610LN  vs 126610LV
116500LN  vs 126500LN
Datejust 同尺寸不同 reference
同一系列钢款 / 金款 / 间金款
```

因此当前训练 batch 应该强制包含：

```text
Positive:
  相同 canonical reference，跨来源的商品

Hard Negative:
  同 brand
  同 series（如果有）
  reference 不同
  字符编辑距离很近
  图片 embedding 很近
  标题 embedding 很近
```

这比随机 negative 更重要。

推荐 batch sampler：

```python
for each batch:
    sample B_ref canonical references

    for each ref in B_ref:
        add 2~4 cross-source positives
        add hard negatives from:
            same_brand
            same_series
            nearest_ref_string
            nearest_image_embedding
```

训练目标不是让模型学会“看起来像”，而是让模型重点学习：

> **看起来很像但 reference 不同，必须拉开。**

---

## 4. 论文 Retrieval 架构

论文 retrieval 流程：

```text
index + query
    │
  Encoder
    │
  Blocker
    │
   KNN
    │
距离阈值 discriminator
```

### 4.1 Blocking

论文使用品牌名 fuzzy string similarity 做 blocking，只在品牌相近的集合内找最近邻。

这对当前需求可以进一步强化为硬 blocking：

```text
brand canonicalization
        │
        ├─ brand 明确且一致 -> 进入同 brand 检索
        └─ brand 冲突       -> 禁止自动匹配
```

腕表比 fashion 更适合 brand hard block，因为 reference 的语义必须附着在品牌命名空间中。

生产唯一键建议始终是：

```text
(brand_id, canonical_reference)
```

而不是裸 reference。

### 4.2 KNN

论文使用 cosine distance：

```text
d(i, j) = 1 - dot(v_i, v_j)
```

embedding 都做归一化，因此 dot product 等价于 cosine similarity。

对百万级/千万级不能直接 brute-force，需要 ANN。

但是当前项目还有一个更重要的优化：

> **尽量不要建立“千万商品互相查”的主索引，而是建立数量更小的 Reference Registry 索引，让商品去找 Reference Entity。**

如果有 1000 万 offer，但 canonical reference 只有几十万级，那么主检索就从：

```text
10M offer -> 10M offer
```

变成：

```text
10M offer -> 0.1M~1M reference entities
```

查询复杂度和错误传播都大幅下降。

### 4.3 Threshold 不应是一个全局常数

论文通过 test PR curve 调 production threshold。

当前系统不建议全局一个 `cosine > 0.92` 就放行，因为：

- 不同品牌视觉可区分度不同；
- 不同来源图片质量不同；
- 某些系列高度同质；
- title/OCR/reference 字段完整度不同；
- 新 domain 会发生 score distribution shift。

推荐阈值至少分层：

```text
threshold_key = (
    brand,
    source_pair,
    evidence_level,
    model_version
)
```

更安全的是使用“选择性决策（selective prediction）”：模型可以输出 `ABSTAIN`，而不是所有候选必须二选一。

---

## 5. 论文结果对当前项目最有价值的几条结论

### 5.1 多模态明显优于单模态

论文最佳 multimodal 配置把图像、文本、数值信息融合后，in-domain AUCPR 约 66.1，out-domain AUCPR 约 63.3；R@1 约 84.2 / 82.1，R@3 约 95.2 / 92.6。

图像-only 明显强于 text-only，是 fashion 数据的特性。

当前腕表不能直接照搬“图像最重要”的结论，因为 reference 本质是 identifier，文本/OCR 里的型号字符可能比视觉更具有决定性。

因此腕表应调整模态权重：

```text
最终决策优先级：
Reference identifier evidence
    > identifier-role evidence
    > brand/series structured evidence
    > OCR evidence
    > text semantics
    > visual similarity
```

视觉应该主要用于：

- 找候选；
- 找 hard negative；
- 在 reference 缺失时辅助人工；
- 发现明显冲突。

视觉不应该单独触发自动合并。

### 5.2 Linear projection 的跨域泛化更好

论文发现一层 hidden MLP 在 validation 指标更高，但 linear 在 in-domain/out-domain test 略好。

对当前只有几百黄金标签的项目，这支持一个非常实用的策略：

> 先用强预训练 encoder + 极小 projection，不要一开始训练复杂端到端 matcher。

### 5.3 大 batch 比复杂 hard-negative infra 更简单

论文冻结 encoder 后把 batch 提高到 16k，用大 batch 获取大量 in-batch negatives。

当前项目可以结合两者：

```text
large batch
+
same-brand / same-series hard-negative sampler
```

这样既保持训练简单，又能专门压制最危险的近邻 reference false positive。

### 5.4 不要只看 F1

论文明确指出，F1 无法体现 high-precision / low-recall 区域的提升，因此使用 PR curve / AUCPR。

当前业务应该更进一步：

真正 KPI 不是 F1，也不是 AUCPR，而是：

```text
Precision@AutoAccept
Coverage@TargetPrecision
FalseMergeCount
AbstainRate
HumanReviewRate
```

最关键：

```text
在满足目标 precision 下，系统能自动覆盖多少数据？
```

而不是：

```text
总体 F1 最高是多少？
```

---

## 6. Human-in-the-loop：论文怎么做，当前应该怎么改

论文的最终人工 UI 会展示：

- 左边一个 query 商品；
- 右边三个模型预测的最近邻；
- 多张商品图片；
- 标题/品牌；
- reviewer 选择哪个 candidate 匹配或 none。

论文经过迭代发现：

1. 给人工展示过多信息不一定有帮助；
2. Top-3 候选比只展示 Top-1 更利于找正确 match；
3. 高 precision 场景，多张图片和 close-up 很重要；
4. 专职、有反馈训练的审核员明显优于普通众包；
5. Human review 最适合做“验证候选”，不适合让人从全库搜索。

### 6.1 论文 human precision 还不够当前需求

论文一个较好的验证实验中，输入模型 precision 约 28.5%，经过 3 人 majority vote 后 HITL output precision 约 93.7%。

对一般电商这是很有价值的提升，但对当前“绝不能误匹配”仍然远远不够。

所以不能把论文的 Human-in-the-loop 理解为：

```text
模型不确定 -> 扔给三个人投票 -> 自动合并
```

当前更安全的定义应该是：

```text
模型不确定 -> 人工补充/确认 reference 证据
              │
              ├─ 找到可审计 reference -> 链接到 canonical reference
              └─ 仍然无法确认          -> ABSTAIN
```

人工审核的任务不是“凭感觉看两块表像不像”，而是确认：

- reference 原文在哪；
- 是商品自身 reference 还是“适配型号”；
- OCR 框是不是读错；
- 是 reference、serial、SKU 还是其他 identifier；
- 是否存在冲突 reference。

### 6.2 推荐审核 UI

```text
┌──────────────── Query Offer ────────────────┐
│ source / item_id                            │
│ title                                       │
│ structured reference fields                │
│ extracted ref candidates + origin span     │
│ OCR ref candidates + image bounding box    │
│ 4~6 images, 优先 close-up / 表背 / 保卡     │
└─────────────────────────────────────────────┘

Top-3 Candidate Reference Entities

[1] Rolex 126610LN
    aliases
    series
    supporting offers
    representative images

[2] Rolex 126610LV
...

[3] Rolex 116610LN
...

Decision:
( ) Confirm candidate 1
( ) Confirm candidate 2
( ) Confirm candidate 3
( ) None of the above
( ) Cannot determine

必须填写：
- evidence source
- observed reference string
- identifier role
```

这会把人工结论变成可回流的结构化训练数据，而不是不可解释的 yes/no。

---

## 7. 当前需求推荐的最终架构

## 7.1 总体架构

```text
                   ┌────────────────────────────┐
3 Sources -------->│ Ingestion / CDC / Versioning│
                   └──────────────┬─────────────┘
                                  │
                                  ▼
                   ┌────────────────────────────┐
                   │ Normalize / Field Typing   │
                   │ brand / sku / ref / title  │
                   └──────────────┬─────────────┘
                                  │
                                  ▼
                   ┌────────────────────────────┐
                   │ Reference Extraction       │
                   │ structured -> title -> OCR │
                   └──────────────┬─────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
          confident exact ref         missing / ambiguous ref
                    │                           │
                    ▼                           ▼
        ┌────────────────────┐       ┌───────────────────────┐
        │ Reference Registry │       │ Multimodal Encoder    │
        │ exact lookup       │       │ image/text/meta       │
        └─────────┬──────────┘       └──────────┬────────────┘
                  │                             │
                  │                             ▼
                  │                  ┌───────────────────────┐
                  │                  │ Blocking + ANN Top-K  │
                  │                  └──────────┬────────────┘
                  │                             │
                  └──────────────┬──────────────┘
                                 ▼
                    ┌────────────────────────┐
                    │ Evidence / Conflict Gate│
                    └────────────┬───────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
         AUTO_ACCEPT          REVIEW            ABSTAIN
              │                  │                  │
              └──────────┬───────┴──────────────────┘
                         ▼
              ┌────────────────────────┐
              │ Offer -> Reference Link│
              │ Decision Ledger        │
              └────────────────────────┘
```

核心是：

```text
Offer 不直接 link Offer
Offer link Reference Entity
```

---

## 8. Reference Registry：系统真正的中心

推荐表：

```sql
reference_entity (
    ref_entity_id          bigint primary key,
    brand_id               bigint not null,
    canonical_reference    varchar not null,
    series                 varchar null,
    model_name             varchar null,
    aliases                jsonb,
    valid_patterns         jsonb,
    evidence_level         smallint,
    status                 varchar,
    created_at             timestamp,
    updated_at             timestamp,
    unique(brand_id, canonical_reference)
)
```

### 8.1 canonical_reference 规范化必须“保守”

允许：

```text
Unicode NFKC
trim
uppercase
统一普通空格
统一全角/半角
把视觉上无意义的分隔符映射到统一形式（需品牌规则白名单）
```

不要做：

```text
O -> 0
I -> 1
S -> 5
删除所有连字符
删除任意前后缀
模糊编辑距离自动改号
```

因为这些“纠错”会制造 false merge。

应同时保存：

```text
raw_reference
normalized_reference
canonical_reference
normalization_rule_version
```

### 8.2 品牌规则独立版本化

不同品牌 reference 语法不同，建议：

```text
reference_normalizer/
  rolex.yaml
  omega.yaml
  cartier.yaml
  patek_philippe.yaml
  audemars_piguet.yaml
```

例如每个规则定义：

- 合法字符集；
- 长度范围；
- 常见分隔符；
- 是否有后缀；
- 是否允许 `/`；
- 系列已知 pattern；
- 明确禁止的平台 SKU pattern。

所有规则改动都必须可重放、可回滚。

---

## 9. Reference Extraction：先规则，后模型，模型只出 candidate

推荐提取顺序：

```text
Tier 1：独立 reference 字段
Tier 2：品牌专用 regex / dictionary 从标题抽取
Tier 3：identifier-role classifier
Tier 4：OCR / VLM 从图片找候选
Tier 5：LLM/VLM 只作为候选生成器
```

### 9.1 为什么要做 identifier-role classifier

标题可能是：

```text
劳力士表带 适配 126610LN
```

如果只 regex，`126610LN` 很容易被错当当前商品 reference。

所以每个 identifier candidate 至少需要分类角色：

```text
PRODUCT_REFERENCE
COMPATIBLE_REFERENCE
SERIAL_NUMBER
PLATFORM_SKU
SELLER_SKU
INTERNAL_ID
UNKNOWN
```

只有 `PRODUCT_REFERENCE` 可以进入自动放行路径。

### 9.2 每个提取结果都要有 provenance

```json
{
  "candidate": "126610LN",
  "normalized": "126610LN",
  "brand_id": 1,
  "role": "PRODUCT_REFERENCE",
  "source_type": "TITLE",
  "source_field": "title",
  "span_start": 8,
  "span_end": 16,
  "extractor_version": "rolex_ref_v3",
  "confidence": 0.998
}
```

OCR 则额外保存：

```text
image_id
bbox
ocr_engine_version
image_role（表背/保卡/吊牌/表盘/其他）
```

这样人工可以看到模型“为什么认为 reference 是这个”。

---

## 10. 多模态 Encoder：如何把论文架构改成腕表版本

论文直接：

```text
pooled image embedding + text embedding + numeric -> linear projection
```

当前建议：

```text
Image Set
   └─ frozen image encoder
      └─ image pooling

Title / Description
   └─ frozen multilingual text encoder

Reference Evidence
   ├─ character-level ref embedding
   ├─ role one-hot
   ├─ ref_present
   └─ ref_conflict

Structured Metadata
   ├─ brand
   ├─ series
   ├─ material
   ├─ diameter
   └─ year / movement（如果有）

concat
  │
  └─ small Linear Projection (128~256d)
       │
       └─ L2 normalize
```

### 10.1 图片多张怎么 pooling

论文用 average pooling，简单且稳定。

当前可以先复现 average pooling 作为 baseline：

```python
image_vec = normalize(mean([encoder(img) for img in images]))
```

以后再比较：

- attention pooling；
- max/mean hybrid；
- 按 image role 加权；
- 去掉低质量/广告/包装图。

不建议第一版就训练复杂 set transformer，因为当前主要误差源更可能在 reference extraction，而不是 pooling。

### 10.2 Embedding 不参与最终自动合并

这一点必须写死：

```text
embedding score 再高
只要 reference 冲突
=> REJECT
```

例如：

```text
A.ref = 126610LN
B.ref = 126610LV
image_similarity = 0.997
```

决策必须是：

```text
NOT_MATCH / CONFLICT
```

不能让模型“覆盖”identifier。

---

## 11. Blocking 和 ANN 的生产设计

## 11.1 第一层：Hard Block

```text
brand_id 必须一致
商品大类必须兼容
明确 reference 冲突直接停止
```

### 11.2 第二层：Reference Lexical Block

如果有候选 reference：

```text
exact canonical lookup
prefix/suffix index（仅召回）
character n-gram / edit distance Top-K（仅召回）
```

注意 edit distance 只能用于找“可能 OCR/格式有误”的候选，不允许直接自动纠错。

### 11.3 第三层：Multimodal ANN

只在 reference 缺失或不确定时调用：

```text
brand shard
   └─ ANN top 20~100
       └─ rerank top 10
           └─ reference evidence gate
```

ANN 可以使用任意成熟向量索引实现，但接口要抽象成：

```python
search(
    brand_id,
    embedding,
    top_k,
    index_version
) -> candidates
```

避免业务和某个特定向量数据库绑定。

### 11.4 为什么优先索引 Reference Entity

Reference Entity 可以有聚合向量：

```text
ref_embedding = robust_pool(
    high_confidence offer embeddings linked to same ref
)
```

并保存多个 prototype：

```text
ref_image_prototypes
ref_text_prototypes
source_specific_prototypes
```

查询时：

```text
Offer -> Reference ANN
```

比：

```text
Offer -> 1000 万 Offer ANN -> 再聚类
```

更稳、更便宜，也避免一条错误边通过 transitive closure 污染整个 cluster。

---

## 12. 最终 Evidence / Conflict Gate

这是当前系统和论文最大的区别。

推荐决策输出不是二分类，而是四态：

```text
AUTO_ACCEPT
REVIEW
REJECT_CONFLICT
ABSTAIN
```

### 12.1 自动接受的最低条件

第一版建议非常保守：

```python
def decide(a, candidate_ref):
    if a.brand_id != candidate_ref.brand_id:
        return REJECT_CONFLICT

    if a.has_conflicting_product_reference:
        return REJECT_CONFLICT

    if a.ref_role != "PRODUCT_REFERENCE":
        return REVIEW

    if a.canonical_reference == candidate_ref.canonical_reference \
       and a.ref_confidence >= VERY_HIGH:
        return AUTO_ACCEPT

    return REVIEW_OR_ABSTAIN
```

也就是：

> **没有可审计的 reference 证据，不自动合并。**

### 12.2 可以逐步增加“强证据组合”

等黄金数据积累后，可增加：

```text
title ref candidate
+ OCR ref candidate
+ 二者一致
+ brand 一致
+ 无任何冲突
```

这种双证据路径也可进入自动接受。

但不建议使用：

```text
image_similarity > x
+ title_similarity > y
=> 自动 match
```

因为这与“同商品 = 同 reference”的业务定义不等价。

### 12.3 负证据优先

建议建立 veto list：

```text
不同 canonical reference          -> veto
不同明确品牌                      -> veto
identifier 被识别为 compatible ref -> veto auto-accept
序列号/平台 SKU 被当 reference      -> veto
series/material/diameter 明确冲突   -> veto or review
```

在 precision-first 系统里，**冲突证据的权重应高于相似证据**。

---

## 13. 数据模型建议

## 13.1 raw_offer

```sql
raw_offer (
    offer_id bigint primary key,
    source varchar,
    source_item_id varchar,
    raw_payload jsonb,
    crawl_version varchar,
    source_updated_at timestamp,
    ingested_at timestamp
)
```

## 13.2 normalized_offer

```sql
normalized_offer (
    offer_id bigint primary key,
    brand_id bigint,
    title_norm text,
    category_id bigint,
    structured_ref_raw varchar,
    structured_ref_norm varchar,
    normalize_version varchar,
    updated_at timestamp
)
```

## 13.3 reference_candidate

```sql
reference_candidate (
    id bigint primary key,
    offer_id bigint,
    candidate_raw varchar,
    candidate_norm varchar,
    role varchar,
    source_type varchar,
    source_locator jsonb,
    confidence double precision,
    extractor_version varchar,
    created_at timestamp
)
```

## 13.4 offer_embedding

```sql
offer_embedding (
    offer_id bigint,
    encoder_version varchar,
    projection_version varchar,
    image_embedding_uri varchar,
    text_embedding_uri varchar,
    final_embedding_uri varchar,
    updated_at timestamp,
    primary key(offer_id, encoder_version, projection_version)
)
```

## 13.5 offer_reference_link

```sql
offer_reference_link (
    offer_id bigint,
    ref_entity_id bigint,
    decision varchar,
    decision_score double precision,
    evidence jsonb,
    model_version varchar,
    rule_version varchar,
    reviewer_version varchar,
    created_at timestamp,
    revoked_at timestamp null
)
```

注意 `link` 必须可撤销，不要直接覆盖。

---

## 14. Decision Ledger：绝不能只保存最终 cluster_id

precision 极端优先的系统必须保存完整决策链：

```json
{
  "offer_id": 123,
  "ref_entity_id": 456,
  "decision": "AUTO_ACCEPT",
  "brand_rule": "PASS",
  "reference_rule": {
    "raw": "126610LN",
    "canonical": "126610LN",
    "role": "PRODUCT_REFERENCE",
    "source": "structured_field"
  },
  "conflicts": [],
  "ann_candidate_rank": 1,
  "multimodal_similarity": 0.971,
  "extractor_version": "ref_extract_v4",
  "projection_version": "watch_embed_v2",
  "gate_version": "gate_v7",
  "timestamp": "..."
}
```

如果规则或 extractor 升级，可以：

```text
replay -> diff -> revoke wrong links
```

这是持续增量系统必须具备的能力。

---

## 15. 增量处理架构

不要每天全量重跑 1000 万。

推荐：

```text
New/Updated Offer Event
        │
        ▼
Normalize
        │
        ▼
Reference Extraction
        │
        ├─ exact ref -> registry lookup -> gate -> link
        │
        └─ uncertain -> embed -> ANN -> gate -> review/abstain
```

只有以下变化需要触发重算：

```text
offer 内容变化
reference normalizer 版本变化
extractor 版本变化
encoder/projection 版本变化
gate 规则变化
reference registry merge/split
```

每个派生结果都带 version，可按版本增量回放。

---

## 16. 训练数据：几百黄金标签应该怎么花

Spec 允许几百对人工黄金标签，但几百对不应该主要用于“随机正负样本”。

最有价值的标注应集中到边界：

### 16.1 黄金集分层

```text
30% 明确正样本：同 ref，跨来源、字段稀疏
50% hard negatives：同品牌同系列不同 ref
10% identifier-role cases：配件/适配/平台 SKU/序列号
10% OCR cases：图中 ref / serial / 保卡 / 吊牌混淆
```

### 16.2 可以自动生成大量 weak positives

如果两个来源都有独立 reference 字段，并且：

```text
brand canonical 一致
reference canonical exact match
无冲突
```

这些可以作为高质量 weak positives 训练 retrieval encoder，而不占人工预算。

### 16.3 hard negatives 自动挖掘

利用当前模型挖：

```text
same brand
+ different ref
+ top multimodal similarity
```

这是最有价值的主动学习队列。

每轮人工只标几十到一两百个最危险的 false-positive candidates，再回流训练。

---

## 17. 评估：不能用普通随机切分

### 17.1 Test split 必须模拟真实增量

论文专门做了：

- in-domain 时间切分；
- out-domain 新 domain；

当前也应该：

```text
Train:
  老时间段 + 两个平台

Test A:
  新时间段，同平台

Test B:
  一个完整来源 hold-out

Test C:
  新品牌 / 新系列 hold-out
```

### 17.2 必须单独建 hard-negative benchmark

随机负样本太容易。

必须专门保留：

```text
同品牌 / 同系列 / 相邻 reference
视觉极近
标题极近
只有一个字符不同
适配型号场景
平台 SKU 冒充 reference
```

### 17.3 核心指标

```text
auto_accept_precision
false_merge_count
coverage_at_target_precision
review_rate
abstain_rate
reference_extraction_precision
reference_extraction_recall
candidate_recall@K
```

模型层可以看：

```text
R@1
R@3
R@10
AUCPR
```

但业务上线必须看自动接受层的 precision。

---

## 18. “绝对不能误匹配”在统计上意味着什么

需要明确：机器学习无法通过几百条黄金标签证明“绝对零错误”。

如果测试集 `n` 条自动接受样本中观察到 0 个错误，一个常用近似是“rule of three”：

```text
95% 置信下错误率上界约 ≈ 3 / n
```

例如：

```text
n = 300，0 error
=> 仍只能大致说明 95% 上界错误率约 1%
```

如果希望用测试数据证明：

```text
error rate < 0.01%
```

就需要数量级约 3 万条以上零错误样本，而不是几百条。

所以“几百黄金标签”最适合做：

- 模型训练；
- hard case 发现；
- 规则设计；
- 初始阈值校准；

而不能当成极高 precision 的充分统计证明。

生产安全应依赖：

```text
强 Reference 规则
+ conflict veto
+ abstention
+ audit/replay
+ 长期线上抽检
```

---

## 19. 第一版可以直接落地的 MVP

不要第一版就训练复杂多模态网络。

## Phase 1：Reference-First Baseline

### Step 1

统一三个来源：

```text
brand canonicalization
reference field mapping
raw title
images
source IDs
```

### Step 2

对 Top 品牌实现 reference normalizer：

```text
Rolex
Omega
Cartier
Patek Philippe
Audemars Piguet
...
```

### Step 3

只做高置信自动链接：

```text
独立 reference 字段
或
标题中品牌规则明确抽到 PRODUCT_REFERENCE
```

### Step 4

建立 canonical Reference Registry。

### Step 5

只按：

```text
(brand_id, canonical_reference)
```

聚合。

这个 baseline 很可能已经能安全覆盖一部分数据，而且是后续模型的高质量训练集来源。

---

## 20. 第二版：加入论文式 Multimodal Retrieval

只处理：

```text
reference 缺失
reference 多候选
OCR 可疑
字段冲突
```

流程：

```text
frozen image encoder
+
frozen text encoder
+
reference/meta features
        │
        ▼
small linear projection
        │
        ▼
brand-sharded ANN
        │
        ▼
Top-20 candidates
        │
        ▼
reference gate
```

训练数据优先使用 Phase 1 产生的高可信同 reference offer。

---

## 21. 第三版：Human Review + Active Learning

审核队列优先级：

```text
P0: 高相似但 reference 冲突
P1: 同系列相邻 reference
P2: OCR/title ref 不一致
P3: 新品牌 / 新来源
P4: 模型 Top1/Top2 margin 很小
```

每次人工结论回流：

```text
reference extraction label
identifier role label
match/reject label
hard-negative pool
```

这样人工不是一次性成本，而是系统持续改进的训练数据生产线。

---

## 22. 推荐代码模块边界

```text
src/
  ingestion/
    source_leixiaoan.py
    source_xbiao.py
    source_shedangjia.py

  normalization/
    brand.py
    reference/
      base.py
      rolex.py
      omega.py
      cartier.py

  extraction/
    structured_ref.py
    title_ref.py
    identifier_role.py
    ocr_ref.py

  registry/
    reference_entity.py
    aliases.py

  embedding/
    image_encoder.py
    text_encoder.py
    pooling.py
    projection.py
    feature_store.py

  retrieval/
    blocker.py
    ann.py
    reranker.py

  decision/
    evidence.py
    conflict_gate.py
    selective_policy.py

  review/
    queue.py
    aggregation.py

  evaluation/
    candidate_recall.py
    precision_at_coverage.py
    hard_negative_benchmark.py
    calibration.py

  pipeline/
    incremental.py
    replay.py
```

原则：

- extraction 和 matching 分开；
- retrieval 和 decision 分开；
- encoder 和 projection 版本分开；
- rule version 必须可追踪；
- ANN 只是基础设施，不承载业务真值。

---

## 23. 一个可执行的决策伪代码

```python
from enum import Enum

class Decision(Enum):
    AUTO_ACCEPT = "AUTO_ACCEPT"
    REVIEW = "REVIEW"
    REJECT_CONFLICT = "REJECT_CONFLICT"
    ABSTAIN = "ABSTAIN"


def decide_offer_to_reference(offer, ref_entity, retrieval_score):
    # 1. 品牌是硬约束
    if offer.brand_id and offer.brand_id != ref_entity.brand_id:
        return Decision.REJECT_CONFLICT

    product_refs = [
        x for x in offer.ref_candidates
        if x.role == "PRODUCT_REFERENCE"
    ]

    # 2. 出现多个互相冲突的高置信 PRODUCT_REFERENCE
    high_refs = {
        x.canonical for x in product_refs
        if x.confidence >= 0.98
    }
    if len(high_refs) > 1:
        return Decision.REJECT_CONFLICT

    # 3. 有高置信 reference 时，只接受严格 canonical 相等
    if len(high_refs) == 1:
        only_ref = next(iter(high_refs))
        if only_ref == ref_entity.canonical_reference:
            return Decision.AUTO_ACCEPT
        return Decision.REJECT_CONFLICT

    # 4. 模型高相似只能触发复核，不能直接 match
    if retrieval_score >= REVIEW_SCORE:
        return Decision.REVIEW

    return Decision.ABSTAIN
```

第一版宁可让 `AUTO_ACCEPT` 很少，也不要给 embedding score 自动放行权。

---

## 24. 论文可以直接复用与必须修改的部分

| 论文设计 | 当前是否复用 | 当前改造 |
|---|---:|---|
| 预计算 image/text embedding | 是 | 增加 OCR/reference provenance |
| 冻结大 Encoder | 是 | 中文文本换成合适的多语言 encoder，接口保持可替换 |
| Average image pooling | 是，作为 baseline | 后续再做 image-role pooling |
| 小 Linear Projection | 是 | 拼入 reference/meta/conflict features |
| Contrastive training | 是 | 强制 same-series/different-ref hard negatives |
| Brand Blocking | 是 | fuzzy brand 改为 canonical brand hard block |
| KNN / ANN candidate retrieval | 是 | 优先 Offer -> Reference Entity，而非 Offer -> Offer |
| Similarity threshold auto accept | 否 | similarity 只做 selective routing |
| PR/AUCPR 评估 | 是 | 再增加 precision@auto-accept / false merge |
| Top-3 Human review | 是 | 人工必须确认 reference provenance |
| Majority vote 即接受 | 不直接复用 | 证据不足仍 ABSTAIN |
| 商品 pair merge | 不建议 | 改为 Offer -> Canonical Reference Link |

---

## 25. 最终推荐

这篇论文对当前项目最大的价值不是 CLIP 本身，而是提供了一个经过生产验证的模式：

```text
大预训练 Encoder
+ 小任务投影层
+ 预计算 embedding
+ Blocking
+ ANN/KNN
+ high-precision selective routing
+ Human validation
```

当前腕表需求应在这个模式上加一层更强的业务约束：

```text
Reference Entity Linking
+ Deterministic Conflict Gate
```

最终系统原则建议固定为：

### 原则 1：Reference 是唯一真值

```text
same product = same canonical reference under same brand
```

### 原则 2：Embedding 只负责候选召回

```text
embedding_similarity != identity
```

### 原则 3：任何强冲突都优先拒绝

```text
conflict > similarity
```

### 原则 4：允许拒识

```text
不确定 -> REVIEW / ABSTAIN
```

而不是强行给出 match/non-match。

### 原则 5：不做无约束 transitive clustering

只有 Offer 都指向同一个 canonical Reference Entity，才能形成同实体簇。

### 原则 6：所有自动决策可审计、可撤销、可重放

持续增量下，规则和模型都会变化，必须保留 decision ledger。

---

## 26. 建议的最短落地路线

如果从今天开始实施，我建议按这个顺序：

```text
Week 1
- 三源 schema 对齐
- 品牌 canonicalization
- Top 品牌 reference pattern/normalizer
- Reference Registry

Week 2
- structured/title reference extractor
- identifier-role classifier baseline
- exact canonical reference 自动链接
- 冲突/拒识规则

Week 3
- 图片/text raw embedding 离线计算
- frozen encoder + linear projection baseline
- brand-sharded ANN Top-K
- hard-negative benchmark

Week 4
- Review UI：Top-3 Reference candidates + OCR bbox + provenance
- selective routing
- active-learning queue
- precision/coverage dashboard
```

四周后即使多模态模型效果一般，也已经有一个安全、可扩展、可持续迭代的 Reference-First 实体匹配骨架。

多模态模型越强，只会提高：

```text
candidate recall
review efficiency
coverage
```

而不会改变最终安全边界。

这正是当前“precision 极端优先”的需求最需要的架构属性。

---

## 参考

1. Tóth, S. et al. **End-to-end multi-modal product matching in fashion e-commerce**. arXiv:2403.11593, 2024.  
   <https://arxiv.org/abs/2403.11593>
2. 当前调研清单：`my-doc/奢侈品文章调研.md`
3. 当前需求 Spec：  
   <https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
