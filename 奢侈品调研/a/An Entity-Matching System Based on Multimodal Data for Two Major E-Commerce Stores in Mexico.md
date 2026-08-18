# An Entity-Matching System Based on Multimodal Data for Two Major E-Commerce Stores in Mexico：从 ImageBERT 到 ReferenceGate 的二奢落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取论文 **An Entity-Matching System Based on Multimodal Data for Two Major E-Commerce Stores in Mexico** 做深入分析。

- 论文主页：<https://www.mdpi.com/2227-7390/10/15/2564>
- DOI：<https://doi.org/10.3390/math10152564>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前先检查了 `奢侈品调研/a`，当前已有：

- `Ameli.md`
- `AnyMatch -- Efficient Zero-Shot Entity Matching with a Small Language Model.md`
- `ComEM.md`
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
- `DeepBlocker.md`
- `Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration.md`
- `Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes.md`
- `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
- `PAM - Understanding Product Images in Cross Product Category Attribute Extraction.md`
- `Tailoring entity resolution for matching product offers.md`
- `TransClean - Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency.md`
- `Using LLMs for the Extraction and Normalization of Product Attribute Values.md`
- `parts-distributor-sku-classifier.md`
- `pyJedAI.md`

本论文标题尚未出现，因此继续执行分析。

当前 Spec 的业务约束非常明确：

1. 数据来自雷小安、腕表之家、奢当家三个来源；
2. 总量约 100 万～1000 万，并持续增量；
3. “同一个商品”的定义不是泛化的商品相似，而是 **同一 reference number / 型号**；
4. reference 字段稀疏，可能在结构化字段、标题、描述甚至图片中；
5. **precision 极端优先，绝对不能误匹配，允许漏匹配**；
6. 有图片；
7. 可以人工标注几百对作为黄金标签。

这篇论文最值得借鉴的是两点：

> 第一，文本与图片在商品匹配中确实存在互补性，中间层融合 `ImageBERT` 比单模态模型得到更高 precision；第二，Siamese 图像表征如果过早压缩成一个距离标量，再与文本融合，反而可能损失信息。

但论文的原始方案 **不能直接作为当前需求的自动匹配器**。论文最好的 `ImageBERT` precision 约为 **86.26%**，意味着每 100 个模型判为匹配的 pair 中，仍可能有十几个错误；这与“绝对不能误匹配”的业务目标完全不兼容。

因此，最适合当前需求的落地方式不是：

```text
文本 + 图片 -> 一个 Match / NoMatch 神经网络 -> 自动合并
```

而是把论文改造成一个 **Reference-first、Multimodal-as-Veto** 的系统：

```text
                    ┌──────────────────────────┐
三个来源商品记录 --->│ Reference Evidence Layer │
                    │ 结构化字段 / 文本 / OCR   │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │ Brand-aware Canonicalizer│
                    │ reference 规范化 + 校验   │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │      ReferenceGate       │
                    │ exact equality + 硬冲突否决│
                    └────────────┬─────────────┘
                                 │
                     仅对可疑/边界记录
                                 v
                    ┌──────────────────────────┐
                    │  Multimodal Verifier     │
                    │ 文本 + 图片 + OCR + 属性  │
                    │ 只做冲突发现/拒识/复核排序 │
                    └────────────┬─────────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 v               v               v
              ACCEPT           REVIEW          REJECT
```

核心原则只有一句：

> **模型可以阻止一次自动合并，但不能在 reference 硬证据不成立时创造一次自动合并。**

这使论文的多模态能力从“身份裁判”变成“安全校验器”，更符合当前 Spec。

---

# 1. 论文到底解决了什么问题

论文研究的是跨两个墨西哥电商平台的 Product Matching / Entity Matching。

每个商品 offer 主要包含：

```text
title
brand
category
price
image
```

任务被建模成二分类：

```text
f(offer_a, offer_b) -> Match / NoMatch
```

也就是说，它解决的是：

> 两个平台上的两个商品 offer 是不是同一个商品？

这和当前二奢业务表面上很像，但定义层面有一个重要区别。

论文里的“同商品”由人工 pair label 定义，是一个整体语义身份判断；而当前 Spec 已经明确把身份定义收缩为：

```text
canonical_reference(a) == canonical_reference(b)
```

因此，当前项目实际上不是一个纯 Entity Matching 问题，而更接近：

```text
Reference Extraction
        +
Reference Normalization
        +
Reference Entity Linking
        +
Strict Equality Join
        +
Multimodal Safety Verification
```

这是后续架构设计最重要的前提。

---

# 2. 论文的数据构造流程：这里隐藏着一个非常实用的工程思想

论文没有对两个网站全量做笛卡尔积。

它的 pair 构造思路大致是：

1. 从 e-shop 1 抓取目标品类商品；
2. 用 e-shop 1 的商品 title 作为 query 去 e-shop 2 搜索；
3. 从 e-shop 2 搜索结果取前几个候选；
4. 再人工标注 pair 是 Match 还是 NoMatch。

也就是：

```text
全量商品
  ↓
先检索候选
  ↓
只对少量候选做昂贵判别
```

这其实就是典型的两阶段架构：

```text
Candidate Retrieval -> Pair Verification
```

对 100 万～1000 万规模尤其重要。

如果三源数据各有数百万条，直接计算：

```text
雷小安 × 腕表之家
雷小安 × 奢当家
腕表之家 × 奢当家
```

会进入不可接受的 O(N²) 比较量级。

不过当前项目还能比论文更进一步：

> 因为同商品已经被定义为同 reference，所以大多数可自动发布的候选根本不需要语义 ANN blocking，直接用 `(brand_id, canonical_reference)` 做 hash/blocking 即可。

因此，论文的“title 检索候选”应只保留给 **reference 缺失或无法确定的 REVIEW 队列**，而不是主链路。

---

# 3. 论文的图像分支：ResNet50 + 两种 pair 建模方式

论文先使用 ImageNet 预训练的 `ResNet50` 对商品图像做特征抽取，再设计两类 pair 模型。

## 3.1 2-CNN

逻辑可以简化成：

```text
image_a -> ResNet50 -> emb_a
image_b -> ResNet50 -> emb_b

concat(emb_a, emb_b)
        ↓
regularized dense layers
        ↓
Match / NoMatch
```

它的优点是：

- 两侧完整图像 embedding 都保留到融合层；
- 分类器可以学习复杂的非线性交互；
- 不强迫图像关系提前被压缩成一个标量。

它的缺点是：

- 方向性更强；
- pair 推理成本较高；
- 对大规模召回不适合，只适合小候选集后的 rerank / verify。

## 3.2 Image-Siamese

另一条路线是 Siamese：

```text
image_a -> shared encoder -> z_a
image_b -> shared encoder -> z_b

       EuclideanDistance(z_a, z_b)
                  ↓
             classifier
```

论文实验中，单模态的 `Image-Siamese` 表现很好：

```text
Accuracy    91.07%
Precision   81.81%
Sensitivity 86.00%
F1          83.22%
```

这个结果说明：

> 商品图片确实能给跨平台实体匹配提供较强信号。

但这并不意味着“图片像就可以当同 reference”。

腕表里恰好存在大量视觉极近的不同 reference：

```text
同系列
同壳型
同盘面
同尺寸
不同机芯/年份/材质/细小配置
=> reference 不同
```

所以在二奢里，图像模型尤其容易遇到：

```text
视觉近邻 ≠ reference equality
```

最安全的使用方式是：

```text
高相似 -> 只能增加人工复核优先级
低相似/明确冲突 -> 可以成为自动拒绝或 REVIEW 的证据
```

而不是：

```text
高相似 -> 自动 Match
```

---

# 4. 论文的文本分支：BERT pair encoding

论文针对西班牙语商品描述使用 BETO（Spanish BERT）。

输入格式沿用 BERT pair 输入：

```text
[CLS] Text_A [SEP] Text_B [SEP]
```

商品文本由多个字段组合，例如：

```text
name
brand
category
price
```

BERT 后继续使用：

```text
BiLSTM
  ↓
max pooling + average pooling
  ↓
concat
  ↓
regularized dense layers
```

然后完成二分类或输出给多模态融合层。

论文中的文本模型结果：

```text
BERT
Accuracy    87.35%
Precision   70.26%
Sensitivity 92.14%
F1          78.87%
```

一个非常值得注意的现象是：

> BERT 的 recall/sensitivity 很高，但 precision 明显不够。

这正是当前业务最危险的模型类型。

对于一般商品匹配，高 recall 可能值得；但对“绝不能误匹配”的 reference 系统来说，70% precision 完全不能进入自动合并链路。

因此当前系统里，如果使用文本 cross-encoder，它的角色应该改成：

```text
candidate ranking
conflict detection
review prioritization
```

而不是最终 identity decision。

---

# 5. 论文最关键的两种多模态融合架构

## 5.1 ImageBERT

论文最好的模型是 `ImageBERT`。

整体结构可以理解为：

```text
image pair
   ↓
2-CNN / image feature branch
   ↓
image representation

text pair
   ↓
BERT-based branch
   ↓
text representation

concat(image_repr, text_repr)
              ↓
       dense classifier
              ↓
        Match / NoMatch
```

结果：

```text
Accuracy    92.41%
Precision   86.26%
Sensitivity 86.45%
F1          85.84%
```

相比单独 BERT：

```text
Precision: 70.26% -> 86.26%
```

提升非常明显。

这说明：

> 当文本信息高度相似、字段不完整或有噪声时，图片作为独立模态能显著减少一部分文本 false positive。

这正是当前 Spec 可以利用的地方。

## 5.2 BERTSiamese

第二种融合方式是：

```text
text pair -> BERT representation
image pair -> Siamese -> Euclidean distance

concat(text_repr, image_distance)
              ↓
          classifier
```

结果反而下降：

```text
Accuracy    87.20%
Precision   70.34%
Sensitivity 89.80%
F1          78.35%
```

论文解释的核心原因也很有启发性：

> Siamese 图像分支最终只给融合层一个距离值，这个一维摘要可能丢掉太多图像信息，导致文本模态占据主导。

这对当前系统的直接启发是：

> 多模态 verification 不要只给下游一个 `image_similarity=0.92`；应该保留更丰富、可解释的图像/OCR冲突特征。

例如：

```text
visual_cosine
ocr_reference_equal
ocr_reference_conflict
logo_brand_equal
watch_face_cluster_distance
case_shape_distance
color_conflict
image_quality
number_of_images
```

然后让安全规则或轻量模型使用这些特征。

---

# 6. 论文训练与评估方式

论文的深度学习模型使用：

```text
binary weighted cross entropy
Adam
最多 160 epochs
early stopping patience = 20
```

并分别划分 train / validation / test。

论文还使用了经典模型作为基线：

| 模型 | Accuracy | Precision | Sensitivity | F1 |
|---|---:|---:|---:|---:|
| Logistic Regression | 0.772 | 0.628 | 0.378 | 0.472 |
| SVM | 0.778 | 0.674 | 0.340 | 0.452 |
| Naive Bayes | 0.734 | 0.504 | 0.729 | 0.596 |
| KNN | 0.794 | 0.636 | 0.548 | 0.589 |
| Random Forest | 0.838 | 0.644 | 0.725 | 0.682 |

单模态深度模型：

| 模型 | Accuracy | Precision | Sensitivity | F1 |
|---|---:|---:|---:|---:|
| 2-CNN | 0.8542 | 0.7210 | 0.8041 | 0.7492 |
| Image-Siamese | 0.9107 | 0.8181 | 0.8600 | 0.8322 |
| BERT | 0.8735 | 0.7026 | 0.9214 | 0.7887 |

多模态模型：

| 模型 | Accuracy | Precision | Sensitivity | F1 |
|---|---:|---:|---:|---:|
| ImageBERT | 0.9241 | 0.8626 | 0.8645 | 0.8584 |
| BERTSiamese | 0.8720 | 0.7034 | 0.8980 | 0.7835 |

论文在一般 Product Matching 语境下认为 `ImageBERT` 是明显改善。

但当前项目必须换一种方式读这张表：

```text
86.26% precision
```

对于“只要模型判断相同就直接合并”来说，仍然是不可接受的。

因此论文提供的是 **特征和架构层面的证据**，而不是满足当前安全目标的最终决策策略。

---

# 7. 为什么原论文不能直接照搬到腕表 reference 系统

## 7.1 目标函数不一样

论文：

```text
maximize pair classification performance
```

当前需求：

```text
subject to almost-zero false accept
maximize automatic coverage
```

两者完全不同。

更准确地说，当前系统应该优化：

```text
最大化：AutoMatchCoverage
约束：FalseAcceptRate <= 极低阈值
```

而不是最大化 F1。

## 7.2 论文的标签是“商品身份”，当前标签是 reference equality

当前业务已经有更强的先验：

```text
same product := same reference
```

既然身份键已经被业务定义，就不应该让一个神经网络重新学习一个模糊身份函数。

## 7.3 腕表存在大量“视觉几乎相同但 reference 不同”的 hard negative

这是多模态模型最危险的区域。

例如同系列相邻 reference，可能：

- 外观极像；
- 品牌相同；
- 系列相同；
- 标题高度重合；
- 价格接近；
- 只有一小段字母数字 reference 不同。

普通二分类模型很容易为了提高 recall 把它们合并。

## 7.4 图片在当前业务里应该是“辅助证据”，不是主键

图片更适合：

```text
OCR reference
发现明显配置冲突
发现品牌冲突
发现类别冲突
发现配件/盒证/表带并非本体
人工复核辅助
```

不适合：

```text
图片很像 -> 自动认定同 reference
```

## 7.5 pairwise 分类会把 1000 万规模推向不必要的计算量

如果 reference 已经能被规范化，最优解是：

```text
hash(reference)
```

不是：

```text
cross_encoder(record_i, record_j)
```

大模型应该只处理小比例的边界样本。

---

# 8. 推荐落地架构：ReferenceGate + MultimodalVerifier

下面给出一个可以直接实现的架构。

```text
┌────────────────────────────────────────────────────────────┐
│                        Source Layer                         │
│         雷小安             腕表之家             奢当家      │
└─────────────────────────────┬──────────────────────────────┘
                              │
                              v
┌────────────────────────────────────────────────────────────┐
│                       Raw Record Store                      │
│ source / source_item_id / title / attrs / images / version │
└─────────────────────────────┬──────────────────────────────┘
                              │
                              v
┌────────────────────────────────────────────────────────────┐
│                     Evidence Extraction                     │
│  1. structured reference                                  │
│  2. title/description reference spans                      │
│  3. OCR reference spans from images                       │
│  4. brand / series / category / accessory-role            │
└─────────────────────────────┬──────────────────────────────┘
                              │
                              v
┌────────────────────────────────────────────────────────────┐
│                Brand-aware Reference Canonicalizer          │
│ normalize -> parse -> validate -> canonical_ref -> version │
└─────────────────────────────┬──────────────────────────────┘
                              │
                              v
┌────────────────────────────────────────────────────────────┐
│                        ReferenceGate                        │
│   hard equality / hard conflict / evidence-grade policy    │
└───────────────┬────────────────────┬───────────────────────┘
                │                    │
       strong exact match       ambiguous / sparse
                │                    │
                v                    v
┌──────────────────────┐   ┌─────────────────────────────────┐
│ Multimodal Safety    │   │ Candidate Retrieval for REVIEW  │
│ Verifier / Veto      │   │ brand/series/text/image ANN     │
└───────────┬──────────┘   └────────────────┬────────────────┘
            │                               │
            └──────────────┬────────────────┘
                           v
              ┌──────────────────────────┐
              │ ACCEPT / REVIEW / REJECT │
              └────────────┬─────────────┘
                           │
                           v
              ┌──────────────────────────┐
              │ Reference Entity + Audit │
              │ immutable decision log   │
              └──────────────────────────┘
```

---

# 9. 第一层：不要先匹配 pair，先抽取 Reference Evidence

每条商品记录都先产生若干 reference observation。

建议统一表示为：

```json
{
  "record_id": "lx_12345",
  "brand_id": "rolex",
  "raw_reference": "126610 LN",
  "canonical_reference": "126610LN",
  "source_type": "title",
  "source_location": "title[12:21]",
  "extractor": "regex_brand_rolex_v3",
  "extractor_score": 0.99,
  "catalog_valid": true,
  "evidence_grade": "B"
}
```

一条记录允许有多个 observation：

```text
结构化字段：126610LN
标题：126610 LN
图片 OCR：126610LN
```

这样系统不会把“某一次抽取结果”直接当真，而是形成 evidence set。

---

# 10. Reference Evidence 分级

建议至少定义 4 级。

## Grade A：高可信结构化证据

例如：

```text
来源明确标注为 型号 / reference / ref / 款号 的字段
且字段角色经过该来源 parser 验证
且通过品牌 reference grammar / catalog 校验
```

这是最高优先级。

## Grade B：强文本证据

例如标题/描述出现：

```text
型号：126610LN
Ref. 126610LN
参考编号 126610LN
```

要求：

- span 与型号关键词有明确语法关系；
- 通过品牌 grammar；
- 不处于“适配 / compatible with / 适用于”等上下文。

## Grade C：图片 OCR 证据

来源可能是：

- 表背；
- 保卡；
- 吊牌；
- 盒证标签；
- 商品铭牌。

OCR 不能只存最终字符串，还应该存：

```text
image_id
bbox
ocr_text
ocr_confidence
image_role
```

因为后续人工要能回看证据。

## Grade D：模型推断 / 相似候选

例如：

```text
文本 embedding 猜测是 126610LN
视觉检索最像 126610LN
LLM 根据描述推断型号
```

这一级 **永远不能单独触发自动 ACCEPT**。

它只用于：

```text
REVIEW 排序
候选生成
辅助解释
```

---

# 11. Canonicalizer：这是整个系统真正的主干

全局统一做：

```text
Unicode NFKC
trim
uppercase
统一全角/半角
统一视觉等价分隔符
清理纯装饰空格
```

但不能做一个“所有品牌都删掉标点”的暴力规则。

腕表 reference 的结构具有品牌特异性。

推荐接口：

```python
canonicalize(
    brand_id,
    raw_reference,
    context
) -> CanonicalizationResult
```

返回：

```json
{
  "canonical": "21030422001001",
  "display_reference": "210.30.42.20.01.001",
  "brand_id": "omega",
  "is_valid": true,
  "grammar_version": "omega_ref_v4",
  "warnings": []
}
```

关键是：

> canonical representation 可以为了 join 去掉品牌允许的格式差异，但 display representation 要保留业务可读形式。

并且每次 canonicalization 都要记录规则版本：

```text
canonicalizer_version
```

否则将来规则升级以后无法解释为什么旧记录会被归到某个 reference entity。

---

# 12. ReferenceGate：自动合并必须由硬条件驱动

推荐最终决策不是概率，而是三态：

```text
ACCEPT
REVIEW
REJECT
```

## 12.1 ACCEPT 的必要条件

至少：

```text
brand_id 相同
canonical_reference 完全相同
没有任何高可信 reference 冲突
没有类别/角色硬冲突
证据强度满足发布策略
```

推荐自动接受策略可以从最保守版本开始：

### Policy P0

两条记录都必须有 `Grade A`：

```text
A(ref=x) + A(ref=x) -> ACCEPT
```

### Policy P1

允许：

```text
一侧 A(ref=x)
另一侧至少两个独立证据 B/C 同时支持 ref=x
且无冲突
-> ACCEPT
```

例如：

```text
雷小安 structured_ref = 126610LN        [A]
腕表之家 title = 126610LN                [B]
腕表之家 保卡 OCR = 126610LN             [C]
```

可以成为非常强的自动链路。

## 12.2 一票否决条件

只要出现下面任一高可信冲突，就不能 ACCEPT：

```text
A(ref=x) vs A(ref=y), x != y
A(ref=x) vs B(ref=y), x != y 且 B 高可信
本体 vs 配件
品牌冲突
明确系列/类别冲突
可信 OCR reference 与结构化 reference 冲突
```

此时：

```text
REVIEW 或 REJECT
```

而不是让神经网络“综合打分后覆盖掉冲突”。

这与论文原始 ImageBERT 最大的设计差异就在这里。

---

# 13. MultimodalVerifier：把 ImageBERT 从 Match 模型改造成 Veto 模型

论文给出的最重要启发是：

```text
text + image
```

比单独 text 更能减少 false positive。

当前项目可以继续使用这个思路，但输出目标改为：

```text
P(conflict | record_a, record_b)
```

而不是：

```text
P(match | record_a, record_b)
```

这非常关键。

## 13.1 输入特征建议

### Reference 特征

```text
canonical_ref_equal
structured_ref_equal
text_ref_equal
ocr_ref_equal
has_ref_conflict
reference_edit_distance
reference_prefix_equal
reference_suffix_equal
```

注意：

`reference_edit_distance` 只能帮助找 hard negative，不能把 `126610LN` 与 `126610LV` 判成相同。

## 13.2 文本特征

可以使用 pair cross-encoder 或更轻量的离线 embedding + pair features：

```text
title semantic similarity
brand equality
series equality
category equality
accessory keyword conflict
model-token overlap
normalized title token overlap
```

特别需要识别：

```text
适用于 XXX
compatible with XXX
表带 XXX
盒子 XXX
保卡 XXX
配件 XXX
```

否则“配件标题中出现被适配腕表 reference”会造成非常危险的误匹配。

## 13.3 图像特征

不要只保留一个 Siamese distance。

建议保留：

```text
main_image_similarity
max_pairwise_image_similarity
min_pairwise_image_similarity
ocr_reference_equal
ocr_reference_conflict
logo_brand_conflict
image_role_distribution
visual_outlier_score
```

多个图片时，可以做：

```text
record A images x record B images
```

但不用全组合跑重模型，可以先向量化，每条记录离线缓存 image embedding，再做矩阵相似度。

## 13.4 结构化属性特征

腕表可加入：

```text
brand
series
case_size
material
dial_color
movement
watch_type
gender/category
year range
```

这些字段只作为 conflict / verification，不允许覆盖 reference mismatch。

## 13.5 价格

价格只做弱信号。

二手市场受：

```text
成色
附件
年份
渠道
地区
是否大全套
维修记录
```

影响太大，不能作为身份键。

最多使用：

```text
log_price_ratio
price_outlier_within_reference
```

作为异常检测。

---

# 14. 为什么建议使用“冲突模型”而不是“同款模型”

假设 reference 已经严格一致：

```text
A.ref = 126610LN
B.ref = 126610LN
```

此时系统真正想知道的是：

> 有没有任何证据表明，这两个 reference assignment 中至少有一个是错的？

所以模型任务天然应该是：

```text
is_reference_assignment_suspicious ?
```

而不是再次问：

```text
are_the_products_same ?
```

这会形成一个更安全的职责边界：

```text
ReferenceGate 定义身份
MultimodalVerifier 发现错误
```

也更符合论文中图片能降低文本 false positive 的观察。

---

# 15. 推荐的融合方式：保留丰富特征，不要只压成一个距离

论文 `BERTSiamese` 的退化是一个重要警告。

推荐融合向量类似：

```python
x = [
    ref_structured_equal,
    ref_title_equal,
    ref_ocr_equal,
    ref_conflict,
    brand_equal,
    series_equal,
    category_equal,
    accessory_conflict,
    text_cross_encoder_score,
    image_cosine_main,
    image_cosine_max,
    image_cosine_min,
    ocr_confidence_a,
    ocr_confidence_b,
    image_quality_a,
    image_quality_b,
    log_price_ratio,
]
```

然后用：

```text
Logistic Regression / GBDT / 小 MLP
```

都可以。

当前场景不一定需要一个巨大的端到端多模态模型。

因为：

- identity 已由 reference 定义；
- 特征数量有限；
- 人工标签只有几百对；
- 需要解释 false positive；
- 模型主要输出 REVIEW / veto。

从工程可控性上看，`GBDT + calibrated probability` 往往比大端到端模型更容易审计。

---

# 16. 100 万～1000 万规模下的候选生成

## 16.1 已解析 reference 的记录

不要做 ANN。

直接：

```text
block_key = hash(brand_id, canonical_reference)
```

所有来源记录进入同一个 reference bucket。

复杂度近似：

```text
O(N) ingestion
O(1) / O(logN) reference lookup
```

而不是 O(N²)。

## 16.2 未解析 reference 的记录

这些记录不能自动 Match，只能进入候选/复核链路。

候选 Blocking 可按：

```text
brand
series
category
高置信型号 token 前缀
text embedding ANN
image embedding ANN
```

产生少量候选：

```text
TopK = 10~50
```

再交给 MultimodalVerifier / 人工复核。

重点：

> 这条 ANN 链路的输出是“可能是什么 reference”，不是“可以自动合并”。

---

# 17. 推荐的数据模型

## 17.1 `product_record`

```sql
product_record(
    record_id              bigint primary key,
    source                 varchar,
    source_item_id         varchar,
    brand_id               bigint,
    title                  text,
    category               varchar,
    price                  decimal,
    raw_payload_uri        text,
    source_updated_at      timestamp,
    payload_hash           varchar,
    ingest_version         varchar
)
```

唯一约束：

```text
(source, source_item_id)
```

## 17.2 `record_image`

```sql
record_image(
    image_id               bigint primary key,
    record_id              bigint,
    image_uri              text,
    image_hash             varchar,
    image_role             varchar,
    embedding_uri          text,
    ocr_status             varchar
)
```

## 17.3 `reference_observation`

```sql
reference_observation(
    observation_id         bigint primary key,
    record_id              bigint,
    raw_reference          varchar,
    canonical_reference    varchar,
    evidence_type          varchar,
    evidence_grade         varchar,
    source_location        varchar,
    extractor_version      varchar,
    extractor_score        decimal,
    catalog_valid          boolean,
    conflict_group         varchar,
    created_at             timestamp
)
```

## 17.4 `reference_entity`

```sql
reference_entity(
    reference_entity_id    bigint primary key,
    brand_id               bigint,
    canonical_reference    varchar,
    display_reference      varchar,
    catalog_status         varchar,
    canonicalizer_version  varchar,
    created_at             timestamp,
    updated_at             timestamp
)
```

唯一约束：

```text
UNIQUE(brand_id, canonical_reference)
```

## 17.5 `record_reference_link`

```sql
record_reference_link(
    record_id              bigint primary key,
    reference_entity_id    bigint,
    decision               varchar,
    decision_reason        varchar,
    evidence_policy        varchar,
    resolver_version       varchar,
    verifier_version       varchar,
    decision_at            timestamp
)
```

`decision`：

```text
ACCEPT
REVIEW
REJECT
```

## 17.6 `match_audit_log`

```sql
match_audit_log(
    audit_id               bigint primary key,
    record_id              bigint,
    old_reference_entity   bigint,
    new_reference_entity   bigint,
    action                 varchar,
    reason_code            varchar,
    evidence_snapshot_uri  text,
    actor                  varchar,
    rule_version           varchar,
    created_at             timestamp
)
```

关键是：

> 所有自动 ACCEPT 都必须可回放。

---

# 18. 统一决策函数

推荐把最终逻辑集中在一个纯函数里：

```python
def resolve_record(record, evidences, verifier):
    refs = collect_valid_reference_candidates(evidences)

    if has_high_trust_conflict(refs):
        return Decision("REVIEW", reason="REFERENCE_CONFLICT")

    best = choose_reference_by_policy(refs)

    if best is None:
        return Decision("REVIEW", reason="NO_SAFE_REFERENCE")

    if not best.passes_auto_accept_policy:
        return Decision("REVIEW", reason="INSUFFICIENT_REFERENCE_EVIDENCE")

    conflict = verifier.check(record, best.reference_entity)

    if conflict.hard_conflict:
        return Decision("REJECT", reason=conflict.reason)

    if conflict.score >= REVIEW_THRESHOLD:
        return Decision("REVIEW", reason="MULTIMODAL_SUSPICION")

    return Decision(
        "ACCEPT",
        reference_entity_id=best.reference_entity_id,
        reason="REFERENCE_GATE_PASSED"
    )
```

注意调用方向：

```text
先 ReferenceGate
后 MultimodalVerifier
```

绝对不要：

```text
MultimodalVerifier 先猜 Match
再找 reference 理由
```

---

# 19. 自动接受规则示例

## Case 1：双结构化字段一致

```text
A structured: 126610LN
B structured: 126610LN
brand: Rolex == Rolex
```

结果：

```text
ACCEPT
```

前提是两个来源对应字段已经过字段角色验证，不是平台 SKU。

## Case 2：一侧结构化，一侧标题 + OCR 双证据

```text
A structured: 126610LN          [A]
B title: Ref 126610LN            [B]
B warranty-card OCR: 126610LN    [C]
```

结果：

```text
ACCEPT
```

## Case 3：标题看起来相同，但 reference 不同

```text
A: Rolex Submariner 126610LN
B: Rolex Submariner 126610LV
```

即使：

```text
text_similarity = 0.99
image_similarity = 0.98
```

结果也必须：

```text
REJECT as same-reference match
```

不能让模型覆盖 reference mismatch。

## Case 4：Reference 相同，但图片 OCR 冲突

```text
A structured: 126610LN
B structured: 126610LN
B image OCR: 126610LV
```

结果：

```text
REVIEW
```

因为这很可能是：

- 来源字段脏数据；
- 图片错挂；
- OCR 错误；
- 商品描述/图片不一致。

不应该直接 ACCEPT。

## Case 5：只有视觉非常像

```text
A no reference
B no reference
image_similarity = 0.995
```

结果：

```text
REVIEW
```

绝不能：

```text
ACCEPT
```

---

# 20. 增量更新设计

数据持续变化时，最危险的问题不是第一次解析，而是 **旧结论在新证据出现后仍然存活**。

因此每条记录建议维护：

```text
payload_hash
extractor_version
canonicalizer_version
resolver_version
verifier_version
```

更新流程：

```text
source item update
       ↓
compare payload_hash
       ↓
changed ?
  no -> skip
  yes
       ↓
re-run evidence extraction
       ↓
re-run reference canonicalization
       ↓
compare old/new evidence
       ↓
if reference changed or conflict introduced
       ↓
withdraw old ACCEPT
       ↓
re-resolve
       ↓
write immutable audit log
```

特别重要：

> ACCEPT 不是永久真理，而是“在某个证据版本和规则版本下的可追溯决策”。

---

# 21. 回填 / 重算策略

规则升级时不要全库暴力重跑所有模型。

可以按影响面重算。

例如：

```text
rolex_ref_v3 -> rolex_ref_v4
```

只找：

```text
brand_id = rolex
AND canonicalizer_version = rolex_ref_v3
```

进行重解析。

OCR 模型更新时只重跑：

```text
image OCR 质量不足
或
当前为 REVIEW 且 OCR 可能提供 reference
```

MultimodalVerifier 更新时也不必影响所有 ACCEPT：

可以先 shadow scoring，只有发现明显风险的历史 ACCEPT 才进入 audit queue。

---

# 22. 几百对黄金标签应该怎么标

如果随机抽几百对，大多数样本会太容易，无法改善 precision。

应该故意标 hard negative。

建议黄金集至少覆盖：

## A. 同系列相邻 reference

```text
126610LN vs 126610LV
```

这是最重要的一类。

## B. Reference 字符只差 1～2 位

例如：

```text
末尾字母不同
一位数字不同
连字符/斜杠混淆
OCR O/0、I/1、S/5 混淆
```

## C. 配件标题含主商品 reference

例如：

```text
适用于 126610LN 的表带
126610LN 原装表盒
```

标签必须是：

```text
not same product entity
```

## D. 图片错挂

```text
标题 reference = X
图片 OCR = Y
```

## E. 同 reference 的格式差异

```text
ABC-123
ABC 123
abc123
```

用于验证 canonicalizer。

## F. 缺 reference，但图文高度相似

这些应训练系统保持 abstain，而不是追求 recall。

---

# 23. 不要误解“几百个黄金标签”能证明零误匹配

这是当前项目里需要特别明确的一点。

如果只验证 300 个自动 ACCEPT，且一个错误都没有，统计上也不能证明 precision 已经达到 99.9% 或 99.99%。

在 0 observed error 的情况下，用 95% 单侧置信上界粗略看：

```text
n = 300
错误率上界约 0.99%
=> 只能支持大约 99% 级 precision 的统计证据
```

如果希望在 95% 置信水平下，用“0 个错误”证明错误率低于：

```text
1%      -> 约需 299 个无错样本
0.1%    -> 约需 2995 个无错样本
0.01%   -> 约需 29956 个无错样本
```

因此：

> 几百个黄金标签适合训练 hard-negative verifier、调规则、做回归集，但不足以单独证明“绝对不会误匹配”。

真正的安全来自：

```text
业务身份硬定义
+ reference exact gate
+ 高可信证据策略
+ 冲突一票否决
+ 模型只做 veto / abstain
+ shadow audit
+ 持续抽检
```

而不是来自一个漂亮的 F1。

---

# 24. 评估指标也要跟论文不同

论文主要看：

```text
Accuracy
Precision
Sensitivity
F1
```

当前项目建议主指标改成：

## 24.1 Auto-Accept Precision

```text
自动 ACCEPT 中真正 reference 相同的比例
```

这是第一指标。

## 24.2 False Accept Count

```text
直接数错误自动合并的数量
```

因为业务对这个指标极度敏感。

## 24.3 Auto-Accept Coverage

```text
所有记录中有多少能无人工 ACCEPT
```

在 precision 达标后再优化它。

## 24.4 Review Rate

```text
进入人工队列比例
```

## 24.5 Reference Conflict Rate

例如：

```text
structured != title
structured != OCR
title != OCR
```

这是非常有价值的数据质量指标。

## 24.6 Source-specific Precision

分别监控：

```text
雷小安
腕表之家
奢当家
```

因为三个来源字段质量一定不同。

## 24.7 Brand-specific Precision

不同品牌 reference grammar 差异大，必须拆开看。

---

# 25. MultimodalVerifier 的阈值不要用一个全局值

推荐分层阈值：

```text
source_pair
brand
reference_evidence_grade
image_availability
```

例如：

```text
雷小安 A-grade structured
  +
腕表之家 A-grade structured
```

可以使用非常宽松的 verifier，只处理明显冲突。

而：

```text
title B-grade
  +
OCR C-grade
```

则应该更严格。

也就是说，Verifier score 不应该脱离 evidence policy 单独解释。

---

# 26. 为什么 MultimodalVerifier 适合采用“特征融合 + 轻量分类器”

论文中的 ImageBERT 用深度中间融合得到最好结果，但当前环境有三个现实变化：

1. identity 已被 reference equality 强约束；
2. 模型只需要找冲突，不需要学习完整身份语义；
3. 标注预算只有几百对。

因此最先落地的版本可以是：

```text
预训练文本 encoder
预训练图像 encoder
OCR
规则特征
        ↓
离线生成 pair features
        ↓
GBDT / Logistic Regression
        ↓
conflict score
```

优点：

- 容易解释；
- 容易加入 hard rule；
- 容易做特征消融；
- 小样本更稳；
- 推理成本低；
- 更适合 1000 万规模中的小比例边界记录。

如果之后标签和流量足够，再升级为 learned multimodal fusion。

---

# 27. 生产部署建议

可以按职责拆成几类组件。

## 27.1 Ingestion

```text
crawler output
  -> normalized source event
  -> raw record store
```

要求：

```text
idempotent upsert
(source, source_item_id) 唯一
```

## 27.2 Reference Extraction Worker

输入：

```text
raw record
```

输出：

```text
reference_observation[]
```

支持：

```text
structured parser
regex/parser
NER/LLM candidate extractor
OCR
```

## 27.3 Canonicalization Service

要求纯函数化：

```text
same input + same version -> same output
```

便于批量回放。

## 27.4 Reference Registry

存储：

```text
brand_id
canonical_reference
display_reference
catalog metadata
```

它是全局实体层。

## 27.5 Resolver

只做：

```text
evidence policy
hard conflict checks
ACCEPT / REVIEW / REJECT
```

## 27.6 Verifier

异步执行昂贵模型。

只有满足以下之一才需要跑：

```text
边界 evidence
高风险品牌
历史冲突来源
随机 shadow audit
```

不需要所有明确 Grade-A exact match 都跑昂贵 cross-encoder。

---

# 28. 一个更直接的 API 设计

## `POST /resolve-record`

请求：

```json
{
  "record_id": "watchhome_98765"
}
```

响应：

```json
{
  "decision": "ACCEPT",
  "reference_entity_id": 812739,
  "canonical_reference": "126610LN",
  "evidence_policy": "P1",
  "evidence": [
    {
      "type": "title",
      "grade": "B",
      "reference": "126610LN"
    },
    {
      "type": "ocr",
      "grade": "C",
      "reference": "126610LN"
    }
  ],
  "conflicts": [],
  "resolver_version": "resolver_v5",
  "canonicalizer_version": "rolex_ref_v4"
}
```

REVIEW 示例：

```json
{
  "decision": "REVIEW",
  "reason": "REFERENCE_CONFLICT",
  "candidates": [
    "126610LN",
    "126610LV"
  ]
}
```

这样上游和人工系统都不需要理解模型内部细节。

---

# 29. 人工复核 UI 应该展示什么

不要只展示一个：

```text
match score = 0.97
```

应该并排展示证据：

```text
Source A                      Source B
------------------------------------------------
Title                         Title
Structured reference          Structured reference
OCR reference + bbox          OCR reference + bbox
Main image                    Main image
Brand / series                Brand / series
Reference observations        Reference observations
Conflict reason               Conflict reason
```

并且默认把最关键差异高亮：

```text
126610LN
126610LV
      ^
```

人工标注动作不要只是 Match / NoMatch，建议至少：

```text
CONFIRM_REFERENCE
CORRECT_REFERENCE
NOT_ENOUGH_EVIDENCE
ACCESSORY_NOT_PRODUCT
IMAGE_MISMATCH
SOURCE_DATA_ERROR
```

这样人工结果才能真正反馈到 extractor/canonicalizer，而不是只给一个 pair label。

---

# 30. 论文架构与推荐架构的映射

| 论文模块 | 论文作用 | 当前系统改造后作用 |
|---|---|---|
| title-based candidate search | 构造 pair 候选 | 仅用于无 reference 的 REVIEW 候选 |
| ResNet50 image encoder | 学习图像相似 | 离线图像 embedding / 冲突证据 |
| Image-Siamese | 判断图片是否同商品 | image similarity / outlier signal，不定义身份 |
| BERT pair encoder | 文本 Match 分类 | 文本冲突检测、候选排序 |
| ImageBERT | 最终 Match 分类 | MultimodalVerifier / Veto |
| weighted BCE | 处理 pair 分类不平衡 | 可改为 hard-negative 加权或 cost-sensitive conflict training |
| Match / NoMatch | 最终输出 | ACCEPT / REVIEW / REJECT |
| F1 | 核心指标 | Auto-Accept Precision + False Accept Count |

这就是这篇论文真正可复用的方式。

---

# 31. MVP：建议按安全价值递增，而不是按模型复杂度递增

## Phase 0：Reference Exact Baseline

先实现：

```text
来源结构化 reference parser
brand normalization
canonicalization
reference_entity
exact join
conflict log
```

只允许：

```text
A + A exact -> ACCEPT
```

这一步应该成为可独立上线的安全基线。

## Phase 1：文本 Reference Extraction

加入：

```text
品牌 grammar
型号关键词上下文
配件否定规则
catalog validation
```

目标：提高 reference 可解析覆盖率。

## Phase 2：OCR

重点处理：

```text
保卡
吊牌
表背
标签
```

OCR 输出仍然是 observation，不直接改最终 identity。

## Phase 3：MultimodalVerifier

使用论文 ImageBERT 的思想，但只做：

```text
veto
REVIEW ranking
shadow audit
```

## Phase 4：Hard-negative Active Learning

人工标签集中到：

```text
最相似的不同 reference
最冲突的相同 reference assignment
新品牌
新来源
OCR 易错模式
```

逐步提升自动 coverage。

---

# 32. 建议的测试矩阵

上线前必须有明确的 hard cases 回归测试。

## Canonicalization

```text
same logical ref, formatting differs -> equal
one meaningful char differs -> not equal
brand differs -> not equal entity key
```

## OCR

```text
0/O
1/I/l
5/S
8/B
分隔符缺失
局部遮挡
```

## Accessory

```text
表带适配 reference
盒子标题含 reference
保卡单卖
零件 reference
```

必须避免挂到腕表本体 entity。

## Multimodal

```text
same ref + very different seller photos -> 不应自动 REJECT，最多 REVIEW
same-looking watch + different ref -> 必须 REJECT as same-reference
same ref + OCR conflict -> REVIEW
```

## Incremental

```text
旧记录 ref=X
更新后 ref=Y
=> 旧 link 必须撤销
```

## Version replay

```text
canonicalizer v3 -> v4
=> 可定位所有受影响记录并回放
```

---

# 33. 一个必须坚持的安全不变量

整个系统建议写成以下不可违反的 invariant：

```text
Invariant 1:
No automatic match may be created without canonical reference equality.

Invariant 2:
Any high-trust reference conflict forces abstention or rejection.

Invariant 3:
Image/text similarity can never override a reference mismatch.

Invariant 4:
Every automatic match is reproducible from versioned evidence.

Invariant 5:
A model may reduce auto-match coverage, but may not expand it beyond ReferenceGate policy.
```

其中最关键的是第五条。

它让模型即使发生分布漂移，也更难制造 false positive 灾难。

---

# 34. 对论文结果最值得保留的三个技术判断

## 判断一：图片确实有价值，但价值主要是降低文本误判

论文里：

```text
BERT precision      70.26%
ImageBERT precision 86.26%
```

这说明图像提供了很强的补充信息。

当前系统应该利用这种信息来：

```text
发现 reference assignment 异常
```

而不是让图片决定 reference。

## 判断二：融合前不要过度压缩模态信息

论文：

```text
ImageBERT > BERTSiamese
```

其中一个重要原因是 Siamese 最后只留下一个距离值。

当前实现也应保留：

```text
OCR
图像向量关系
图像角色
图像质量
结构化属性
```

而不是只留下 `similarity`。

## 判断三：候选生成和判别必须拆层

论文先检索，再 pair classify。

当前系统进一步拆成：

```text
Reference exact bucket
        ↓
Safety verification
```

只有无 reference 样本才进入：

```text
ANN candidate retrieval
        ↓
REVIEW
```

这样才能撑到千万级。

---

# 35. 最终推荐方案

如果现在就要给 Spec 落一个最稳妥的实现，我推荐：

```text
1. 为雷小安、腕表之家、奢当家分别实现 source adapter。
2. 把所有“像编号的字符串”先分角色：brand reference / platform SKU / seller SKU / accessory target ref。
3. 建品牌级 reference grammar 和 canonicalizer。
4. 建全局 Reference Entity Registry，唯一键为 (brand_id, canonical_reference)。
5. 每条商品记录收集 structured / text / OCR 三路 reference observation。
6. 用 evidence-grade policy 决定是否允许自动挂到 reference entity。
7. 任何高可信 reference 冲突立即 REVIEW/REJECT。
8. 同 reference 的记录通过 exact join 聚合，不做全量 pairwise matcher。
9. 借鉴论文 ImageBERT 的多模态中间融合思想，训练一个 MultimodalVerifier，但它只拥有 veto / REVIEW 权限。
10. 对无 reference 记录用文本/图片 ANN 召回候选，仅用于人工复核，不自动合并。
11. 所有 extractor/canonicalizer/resolver/verifier 都版本化，所有 ACCEPT 都记录 evidence snapshot。
12. 用几百条人工标签优先覆盖相邻 reference hard negatives，并持续抽检自动 ACCEPT。
```

一句话总结：

> **这篇论文证明了“文本 + 图片”比单独文本更能减少商品匹配误判，但当前二奢系统不应该照搬 ImageBERT 做最终 Match 分类；最稳的方案是 ReferenceGate 负责身份，ImageBERT 风格的 MultimodalVerifier 负责否决和复核。**

这样既吸收论文的多模态架构价值，也完全服从当前业务的 precision-first 定义。

---

# 36. 参考

- Estrada-Valenciano, R.; Muñiz-Sánchez, V.; De-la-Torre-Gutiérrez, H. **An Entity-Matching System Based on Multimodal Data for Two Major E-Commerce Stores in Mexico.** Mathematics 2022, 10, 2564. <https://doi.org/10.3390/math10152564>
- 论文主页：<https://www.mdpi.com/2227-7390/10/15/2564>
- 当前需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
