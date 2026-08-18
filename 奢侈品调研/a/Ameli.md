# Ameli：从多模态商品实体链接到 Precision-First 腕表 Reference Linking 的落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **Ameli: Enhancing Multimodal Entity Linking with Fine-Grained Attributes** 进行深入分析。

- 论文：<https://aclanthology.org/2024.eacl-long.172/>
- arXiv：<https://arxiv.org/abs/2305.14725>
- 官方代码：<https://github.com/PLUM-Lab/Ameli>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

Ameli 与当前需求高度相关，因为它没有把商品实体链接简单做成一个“图文相似度二分类器”，而是显式拆成：

```text
商品/Review
   │
   ├─ Text Retrieval
   ├─ Image Retrieval
   ▼
Top-K Candidate Entities
   │
   ├─ Fine-grained Attribute Extraction
   ├─ Attribute Filtering
   ├─ NLI Text Disambiguation
   └─ Contrastive Image Disambiguation
   ▼
Final Entity
```

尤其重要的是，Ameli 会从文本和图片中抽取细粒度属性，并把 **Model Number** 作为高价值属性之一，再用这些属性过滤相似候选。这个思路和当前三源二奢/腕表场景非常契合：同系列腕表外观、标题和语义都可能极其相似，而业务定义已经明确——**同一个商品 = 同一个 reference number / 型号**。

不过，Ameli 不能原样落地。当前 Spec 的约束比论文任务严格得多：

> **绝对不能误匹配，precision 极端优先，允许大量漏匹配。**

Ameli 的目标是提高 end-to-end entity linking F1，并且最终总会从候选中选一个得分最高的实体；而我们的系统必须允许 `ABSTAIN / NEED_REVIEW / NO_LINK`，不能因为某个候选“最像”就自动合并。

因此本次建议不是直接部署 Ameli，而是把它重构成一套 **Reference-Centric Multimodal Entity Linking（R-MEL）**：

```text
每条来源商品记录
       │
       ▼
Reference Evidence Extraction
       │
       ├─ 结构化字段
       ├─ 标题/描述
       ├─ OCR
       └─ 受约束模型抽取
       │
       ▼
Reference Candidate Retrieval
       │
       ▼
Strict Verification + Conflict Veto
       │
       ├─ AUTO_LINK -> Global Reference Entity
       ├─ NEED_REVIEW
       ├─ CONFLICT_REJECT
       └─ NO_LINK
```

最关键的设计原则是：

> **模型负责“找候选”和“发现冲突”，Reference 的确定性证据负责“自动放行”；任何强证据冲突都必须拒识。**

相比直接做三源记录之间的 pairwise matching，这个架构更安全，也更适合 100 万～1000 万级持续增量数据。

---

## 1. Ameli 解决的原始问题

Ameli 研究的是 **Multimodal Entity Linking（多模态实体链接）**：给定一条包含文本和图片的商品评论，需要把它链接到知识库里的正确商品实体。

这个任务的难点与腕表数据非常相似：

- 商品有大量相近变体；
- 标题、描述、评论里的信息并不完整；
- 图片在很多情况下有帮助，但相似型号外观可能几乎一致；
- 真正区分实体的往往是很细粒度的属性；
- 候选库规模较大，不能对所有实体做完整深模型打分。

论文构建的商品知识库包含约 3.47 万商品实体，同时包含大量商品图片和细粒度属性。整个系统因此采用标准的两阶段结构：

1. **Candidate Retrieval**：快速召回 Top-K 可能实体；
2. **Entity Disambiguation**：在候选集中进行更昂贵、更细粒度的判断。

这与大规模 Entity Matching 的 Blocking + Matching 架构类似，但 Ameli 多了一层非常值得当前项目借鉴的 **Fine-grained Attribute**。

---

## 2. Ameli 的候选召回架构

### 2.1 文本召回

Ameli 使用 SBERT 把 mention/review 文本和商品实体文本编码成向量。

商品实体侧文本大致会把：

- Product Title
- Description
- Attributes

拼接后编码。

训练时使用 InfoNCE/对比学习思想：

```text
review/query --------> encoder ----> q
positive product ----> encoder ----> p+
negative products ---> encoder ----> p-

目标：sim(q, p+) > sim(q, p-)
```

负样本既有普通负例，也利用 batch 内其他实体作为 in-batch negatives。

这部分非常适合当前需求作为 **reference 缺失时的 recall layer**，但不适合直接决定是否相同，因为：

```text
Rolex Submariner 126610LV
Rolex Submariner 126610LN
```

在自然语言 embedding 中极可能非常接近，而业务上它们是两个不同 reference，必须严格区分。

### 2.2 图像召回

Ameli 使用 CLIP 编码 query 图片和商品实体图片。

实体通常有多张图，query 也可能有多张图。代码和论文中采用多图相似度聚合，典型策略是取图片集合之间的最大 cosine similarity：

```text
image_score(query, entity)
  = max cosine(query_image_i, entity_image_j)
```

这个策略很适合商品实体链接中的 recall，因为只要其中一张图比较相关，就能把候选召回。

但当前腕表任务中必须非常谨慎：

- 同系列不同 reference 可能共用几乎相同的表壳、盘面布局；
- 商家可能复用官网图、宣传图；
- 商品图里可能出现包装盒、保卡、表带等；
- 二手商品拍摄角度和背景差异很大。

所以图像在当前系统中不应拥有“单独把两个 record 判为同一 reference”的权限。

### 2.3 文本与图片融合召回

Ameli 将文本 retrieval score 与 image retrieval score 加权融合：

```text
retrieval_score
  = wt * text_score
  + wi * image_score
```

权重在开发集上调优，然后输出 Top-K candidate entities。

论文结果显示，多模态融合比单独文本或图片召回效果更好。经过 fine-tuning 后，融合检索在论文数据集上的 Recall@10 约为 85.84%，Recall@100 约为 95.11%。

这个结果说明一个重要工程事实：

> 即使多模态检索已经很强，Top-K 仍然会漏掉真实实体。

因此，对于当前需求，**绝不能把“真实 reference 没出现在 embedding Top-K”解释成“应该匹配最相似的另一个 reference”**。

有明确 reference 的记录应直接走 exact reference index，不应依赖 embedding retrieval。

---

## 3. Ameli 最值得借鉴的部分：Fine-Grained Attribute

Ameli 的核心贡献不是简单叠加 SBERT + CLIP，而是把细粒度属性单独建模。

### 3.1 属性来源

论文使用多种方式从 query/review 中产生属性：

#### 方式 A：String Match

把 Top-K candidate entities 的属性值作为受限字典，在 review/title 中做字符串匹配。

这种方法看起来简单，却非常重要，因为它有两个优点：

1. **候选闭集**：只允许抽出候选实体中真实存在的属性值；
2. **低幻觉**：模型不会自由创造一个不存在的型号。

论文的属性抽取实验里，String Match 的 precision 非常高，约 97.82%，但 recall 只有约 37.88%。

这正好符合当前业务理念：

> 低 recall 可以接受，高 precision 才有资格进入自动匹配链路。

#### 方式 B：OCR

Ameli 使用 EasyOCR 从 review 图片中抽取文字。

论文中 OCR 属性抽取 precision 约 98.46%，recall 约 12.54%。

对一般商品评论来说 OCR recall 很低，但对腕表场景反而可能很有价值，因为 reference 可能直接出现在：

- 保卡；
- 吊牌；
- 表背刻字；
- 盒标；
- 发票/鉴定卡；
- 商家拍摄的型号标签。

因此我们应该保留 Ameli 的 OCR 思路，但把它从通用 OCR 改造成 **Reference OCR Evidence Extractor**，并保留图片位置、OCR bounding box、字符置信度等审计证据。

#### 方式 C：LLM / QA

Ameli 会针对常见商品属性构建多选问答，在候选实体已有属性值的范围内，让 ChatGPT 选择对应属性。

论文显式使用的属性类别包括：

- Brand
- Color
- Model Number
- Product Title
- 其他高频属性

其中 **Model Number** 对当前需求尤其关键。

但论文中的 zero-shot ChatGPT 属性抽取 precision 并不够高，约 64.57%。这意味着：

> LLM 可以帮助发现候选，但绝不能直接作为“reference 自动确认器”。

当前系统应该进一步收紧成：

```text
不要问：
“这个商品的型号是什么？”

而应该问：
“以下候选 reference 中，文本/图片是否明确支持其中一个？
A. 126610LV
B. 126610LN
C. 124060
D. 无法判断”
```

同时输出必须允许 `无法判断`，并且模型不能生成候选集合之外的新 reference。

#### 方式 D：生成式模型补全

Ameli 也尝试用 GPT-2 等生成模型做属性补全。

这对于提高 recall 有价值，但当前需求不适合把自由生成的型号直接放进 AUTO_LINK。最多只能生成 **candidate hint**，随后必须用：

- Reference KB；
- 品牌规则；
- 原始文本 span；
- OCR span；
- 结构化字段

重新验证。

### 3.2 候选属性闭集约束

Ameli 有一个非常关键的实现思想：

> 抽出的属性值只保留那些能与 Top-K candidate entities 中已知属性值对应的值。

这本质上是“检索增强的闭集抽取”。

当前项目可以把这个思想强化成：

```text
先得到品牌级 Reference Candidate Set
              │
              ▼
抽取器只能输出 Candidate Set 中的 canonical reference
              │
              ├─ exact support -> 强证据
              ├─ fuzzy support -> 弱证据
              └─ no support    -> abstain
```

这样可以显著减少 LLM/OCR/NER 自由生成错误型号的问题。

---

## 4. Ameli 的 Entity Disambiguation

### 4.1 Attribute Filtering

得到系统属性后，Ameli 会先过滤与属性不一致的 candidate entities。

这一步是当前项目最值得直接复用的地方之一。

传统多模态模型通常会：

```text
text embedding + image embedding -> classifier -> score
```

而 Ameli 会先问：

```text
候选实体的已知属性
是否与 query 抽出的属性冲突？
```

如果冲突，就在进入更昂贵的 neural reranker 前删除。

对腕表而言，reference 就应该升级成最强属性：

```text
如果 source A 明确抽出 126610LV，
候选 reference = 126610LN，
无论标题和图片多相似，立即 hard reject。
```

### 4.2 NLI 文本消歧

Ameli 把文本消歧建模成 Natural Language Inference。

代码中 `disambiguation/train_nli.py` 使用 Sentence-Transformers 的 `CrossEncoder`，标签为：

```text
contradiction = 0
entailment     = 1
neutral        = 2
```

训练时还会根据类别频率做 loss reweighting，以缓解样本不平衡。

概念上可以把：

```text
Query / Review
```

和：

```text
Candidate entity 的 attribute / description
```

组成句子对，判断 candidate 是否被 query 支持。

当前系统不建议用 NLI 直接选最终 reference，而建议把它变成 **Contradiction Detector**：

```text
Candidate = Rolex 126610LV
Query evidence = “黑色陶瓷圈 / LN / 126610LN”

NLI -> contradiction 高
=> veto 126610LV
```

也就是说：

> NLI 的最佳业务角色不是“证明相同”，而是“发现不一致并阻止误匹配”。

这更符合 precision-first。

### 4.3 Contrastive Image Disambiguation

Ameli 的 image disambiguation 会：

1. 从 query 和 candidate entity 中挑选最相关图片；
2. 用 CLIP 得到图像表示；
3. 经过 feed-forward/residual 映射；
4. 用 contrastive loss 训练区分正确实体和负例。

代码中的 `model_contrastive.py` 还提供了一套可组合的多模态 encoder，包括：

- mention multimodal encoder；
- entity multimodal encoder；
- text embedding；
- attribute embedding；
- visual features；
- concatenate/self-attention fusion；
- MLP projection；
- L2 normalization。

仓库中还能看到大量实验性 LXMERT、cross-attention 等代码路径，说明它更像研究框架而不是可直接复制进生产的轻量 SDK。

对我们的系统，这部分不应该成为必须上线的主路径。更建议把图像模块降级为：

```text
1. OCR reference 证据
2. 图像冲突发现
3. 人工复核排序
4. reference 缺失时的候选召回
```

而不是“视觉相似 = 相同 reference”。

---

## 5. Ameli 的最终融合为什么不适合直接照搬

Ameli 最后会融合：

- retrieval score；
- NLI/text score；
- image score；

然后选一个最终候选。

论文实验中，完整 System Attribute 方案 end-to-end F1 约 51.54%，使用 Gold Attribute 可以提升到约 62.87%，Human 约 74.0%。

这组结果非常有启发：

### 5.1 Attribute 是最大杠杆之一

系统属性提升明显，Gold Attribute 更明显，说明：

> 商品实体链接的瓶颈很大程度不是更复杂的融合网络，而是“有没有可靠的区分性属性”。

对当前项目，这几乎直接指向：

```text
Reference extraction quality
    >
通用 multimodal similarity model complexity
```

### 5.2 Retrieval 会传播错误

论文错误分析提到，一部分 disambiguation 错误是因为真实实体根本不在 Top-K candidate 中。

因此我们不能用：

```text
Top-K 里选最像的
```

替代：

```text
没有足够证据 -> 不链接
```

### 5.3 多模态相似度无法给“绝不误匹配”保证

即使 text + image + attribute 的平均效果很好，也不意味着高分区间没有 false positive。

当前业务不是优化普通 F1，而是优化：

```text
Precision(AUTO_LINK) -> 尽可能接近 100%
Coverage             -> 在上述条件下尽量提高
```

所以决策函数必须从 `argmax()` 改成 **selective prediction / abstention policy**。

---

# 6. 当前需求应该如何重构：不是 Record-to-Record，而是 Record-to-Reference

当前 Spec 已经明确：

> “同一个商品”的定义是同一个 reference number / 型号。

因此最重要的架构重构是：

```text
不要：
雷小安记录 A <-> 腕表之家记录 B ?
雷小安记录 A <-> 奢当家记录 C ?
腕表之家记录 B <-> 奢当家记录 C ?

而是：
A -> Global Reference Entity X
B -> Global Reference Entity X
C -> Global Reference Entity X

=> A/B/C 自动属于同一 reference group
```

这把一个复杂的 `O(N×M)` pairwise entity matching 问题，变成一个更可控的 **Entity Linking** 问题。

这也是 Ameli 最适合当前需求的真正原因。

---

# 7. 推荐的生产架构：R-MEL

我建议实现一套：

**R-MEL = Reference-Centric Multimodal Entity Linking with Abstention**

整体架构如下：

```text
                 ┌─────────────────────────────┐
                 │ 雷小安 / 腕表之家 / 奢当家 │
                 └──────────────┬──────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │ Ingestion / Normalize│
                    └──────────┬──────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │ Reference Evidence Extraction  │
              │                                │
              │ 1. structured field            │
              │ 2. title / description         │
              │ 3. OCR                         │
              │ 4. constrained model           │
              └───────────────┬────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Evidence Conflict │
                    │      Check        │
                    └───────┬───────────┘
                            │
               conflict ────┴──── no conflict
                  │                    │
                  ▼                    ▼
          CONFLICT_REJECT      Reference Candidate
                                      Retrieval
                                          │
                                          ▼
                               ┌─────────────────────┐
                               │ Strict Verification │
                               │ + Policy Engine     │
                               └───────┬─────────────┘
                                       │
          ┌────────────────────────────┼───────────────────────┐
          ▼                            ▼                       ▼
      AUTO_LINK                  NEED_REVIEW                NO_LINK
          │
          ▼
 Global Reference Entity
          │
          ▼
Cross-source records naturally grouped
```

---

## 8. Global Reference KB

### 8.1 核心表

第一阶段不要急着训练大模型，先建立可靠的 `ReferenceEntity`。

建议字段：

```sql
ReferenceEntity
---------------
reference_entity_id
brand_id
canonical_reference
reference_display
series
family
status
created_at
updated_at
```

必须有：

```sql
UNIQUE (brand_id, canonical_reference)
```

因为同一个字符串在不同品牌可能有不同含义，reference 的实体键应该是：

```text
(brand_id, canonical_reference)
```

而不是只有 reference string。

### 8.2 Reference Alias

再建立：

```sql
ReferenceAlias
--------------
brand_id
reference_entity_id
raw_alias
normalized_alias
alias_type
source
verified
```

例如某些来源可能写成：

```text
126610LV
126610 LV
126610-LV
```

它们是否可以折叠成同一个 canonical value，应该由 **品牌级规则** 决定，而不是全局粗暴删除所有标点。

---

# 9. Reference Normalization：必须“保守”而不是“聪明”

这是整个系统最容易产生灾难性误匹配的地方。

一个错误的 normalizer 可能把两个本来不同的型号变成同一个字符串。

因此建议只做安全变换：

```text
1. Unicode NFKC
2. trim
3. uppercase
4. 全角字符 -> 半角
5. 只对已验证品牌规则处理空格/连接符
6. 保留有业务意义的字母后缀
7. 不做通用数字抽取
8. 不做任意前后缀删除
```

例如：

```text
126610LV
126610LN
```

只差一个字符，但必须永远是两个 reference。

禁止这样的 normalization：

```python
re.sub(r'[^0-9]', '', ref)
```

因为它会把：

```text
126610LV -> 126610
126610LN -> 126610
```

直接制造 false positive。

### 9.1 推荐 API

```python
normalize_reference(
    brand_id,
    raw_reference,
) -> {
    "canonical": "126610LV",
    "rule_id": "rolex_ref_v3",
    "is_valid": True,
    "warnings": []
}
```

所有 normalization 必须可审计：

- 使用了哪一版规则；
- 原始字符串是什么；
- canonical 是什么；
- 是否发生了字符删除/替换。

---

# 10. Evidence Extraction：把 Reference 当作“证据”，而不是一个模型输出字段

建议把每次抽取结果保存成独立证据对象：

```sql
ReferenceEvidence
-----------------
evidence_id
source_record_id
evidence_type
raw_value
canonical_value
brand_id
confidence
provenance
extractor_version
image_id
bbox
created_at
```

其中 `evidence_type` 至少分：

```text
STRUCTURED_FIELD
TITLE_EXACT
DESCRIPTION_EXACT
OCR_CARD
OCR_CASEBACK
OCR_TAG
OCR_OTHER
CONSTRAINED_LLM
FUZZY_CANDIDATE
```

这样后续 policy 可以区别对待不同来源，而不是把所有结果平均融合成一个 score。

---

## 11. Trust Tier：不同证据必须有不同权限

建议定义：

### Tier A：强确定性证据

典型：

- 来源独立 reference 字段，且通过品牌 grammar 校验；
- 标题中清晰、唯一、上下文确认属于当前商品的 exact reference；
- 两个独立高可信来源都得到同一 canonical reference。

Tier A 可以进入 AUTO_LINK 判断。

### Tier B：可确认型证据

典型：

- 高置信 OCR，且结果在 Reference KB 中 exact hit；
- 标题 + OCR 同时支持同一 reference；
- 两张不同类型图片均 OCR 到相同 reference。

Tier B 在满足额外一致性规则后可以升级放行。

### Tier C：候选证据

典型：

- LLM 从受限候选中选择；
- fuzzy string match；
- embedding retrieval；
- CLIP image similarity；
- NLI entailment score。

Tier C 只能：

- 扩大候选；
- 排序人工复核；
- 发现冲突；

**不能单独产生 AUTO_LINK。**

---

# 12. 一个常被忽略的问题：先判断字符串“是什么编号”

商品标题里出现一个很像 reference 的字母数字串，不代表它就是商品 reference。

常见混淆包括：

```text
品牌 reference
平台商品 ID
商家 SKU
库存编号
鉴定编号
订单编号
配件兼容型号
机芯编号
表壳编号
序列号
```

因此建议增加 `ReferenceRoleClassifier`：

```text
BRAND_REFERENCE
PLATFORM_SKU
STORE_SKU
SERIAL_NUMBER
MOVEMENT_NUMBER
ACCESSORY_COMPATIBLE_REFERENCE
UNKNOWN_IDENTIFIER
```

尤其要防止：

```text
“适配 Rolex 126610LV 的表带”
```

被系统理解成“当前售卖商品的 reference = 126610LV”。

这类错误在纯字符串 exact match 系统里非常危险。

建议把：

- 商品类型；
- reference 附近上下文；
- 标题关键词（适配、兼容、表带、盒、卡、配件等）；
- 来源字段语义

一起用于角色判定。

---

# 13. Candidate Retrieval：只有在 Reference 不明确时才需要复杂召回

### 13.1 Fast Path

如果已经获得唯一 Tier A reference：

```text
brand = Rolex
canonical_ref = 126610LV
```

直接执行：

```sql
SELECT reference_entity_id
FROM reference_entity
WHERE brand_id = ?
  AND canonical_reference = ?;
```

这是最安全、最低成本的路径，不需要 SBERT、CLIP、FAISS 或 LLM。

### 13.2 Slow Path

只有 reference 缺失、OCR 有噪声、标题写法异常时，才进入 Ameli 风格的 candidate retrieval。

候选可以来自：

```text
A. brand + reference prefix
B. brand-specific character n-gram
C. edit distance
D. series/family
E. SBERT text ANN
F. CLIP image ANN
G. OCR fuzzy candidate
```

最终形成：

```text
CandidateSet = union(A, B, C, D, E, F, G)
```

但每个 candidate 必须保存 `retrieval_reason`：

```json
{
  "reference": "126610LV",
  "reasons": [
    "same_brand",
    "ocr_edit_distance_1",
    "title_embedding_top_3"
  ],
  "text_score": 0.91,
  "image_score": 0.94,
  "ocr_distance": 1
}
```

不要只留下一个融合后的 `0.932`，否则无法解释为什么模型选中了它。

---

# 14. 不建议照搬 Ameli 的固定 Top-10

Ameli 的 disambiguation 通常围绕 Top-K（代码默认很多场景最大候选数 10）做计算。

当前系统不应该把 10 当作固定业务边界。

因为我们的最终 verification 很严格，候选多一些只影响成本，不直接损害 precision。

可以采用：

```text
reference prefix / lexical candidate <= 50
embedding candidate             <= 50
image candidate                 <= 20
union + dedupe                   <= 100
```

再通过 cheap rules 快速过滤。

如果唯一 strong reference 根本不在候选中，则直接 `NO_LINK / NEED_REVIEW`，不能强选。

---

# 15. Strict Verification：真正保证 Precision 的核心

Candidate retrieval 之后，不应该训练一个普通二分类器说：

```text
P(match) > 0.95 => match
```

而应该设计一个可解释的 rule/model policy。

推荐的 AUTO_LINK 条件：

```text
1. brand 已确认且一致
2. candidate canonical reference 在 Reference KB 中存在
3. 至少一个允许自动放行的强 reference evidence
4. 所有高可信 reference evidence 都指向同一个 canonical reference
5. 不存在另一条高可信冲突 reference
6. identifier role = BRAND_REFERENCE
7. product role 不属于明显配件/兼容件
8. normalization 无危险 warning
9. policy 版本允许该来源 + 品牌 + evidence_type 自动放行
```

注意：

> `SBERT score = 0.99`、`CLIP score = 0.99` 本身不应该出现在 AUTO_LINK 的必要充分条件里。

它们只能是辅助证据。

---

# 16. Conflict Veto：一票否决比加权平均更重要

对“绝不能误匹配”的系统，冲突应该有一票否决权。

例如：

```text
结构化字段：126610LV
标题抽取：126610LV
OCR 保卡：126610LN
```

普通模型可能做：

```text
2 vs 1 => 126610LV
```

当前系统应该做：

```text
CONFLICT_REJECT
```

因为冲突可能意味着：

- 结构化字段填错；
- 图片串图；
- OCR 误识别；
- 商品标题复制错误；
- 配件场景；
- 商家自己就存在脏数据。

错误数据不应该通过“多数投票”被自动洗成一个确定结果。

---

# 17. 推荐把 NLI 改造成 Veto Model

Ameli 用 NLI 做 candidate ranking。

在这里可以做一个更安全的改造：

```text
candidate reference = 126610LV
候选知识：绿色陶瓷圈、Submariner Date ...

record evidence：
“126610LN 黑水鬼 黑色陶瓷圈”
```

构造多个 hypothesis：

```text
H1: 当前商品的 reference 是 126610LV
H2: 当前商品属于绿色版本
H3: 当前商品属于 126610LV 对应系列
```

NLI 只用于发现：

```text
CONTRADICTION -> reject candidate
```

而：

```text
ENTAILMENT -> 仅增加辅助置信，不自动放行
```

这种“不对称使用模型”的方式更符合业务目标。

---

# 18. Image 的正确角色：辅助，不越权

### 可以做

```text
1. OCR 读取 reference
2. 检查图片是否明显是不同品类
3. 检查颜色/盘面/表圈等是否与 candidate reference 明显冲突
4. 在人工复核队列中排序
5. Reference 缺失时召回可能候选
```

### 不应该做

```text
CLIP 很像
=> 自动认为两个商品同 reference
```

因为：

```text
126610LV vs 126610LN
```

可能只有表圈颜色等细节不同；还有更多型号外观差异比这更小。

图像模型最适合作为 **negative evidence / contradiction evidence**，不是唯一 positive identity evidence。

---

# 19. OCR 应专门做 ROI，而不是整图无差别识别

可以把图片先分类：

```text
DIAL
CASEBACK
WARRANTY_CARD
TAG
BOX_LABEL
INVOICE
OTHER
```

再按图片类型运行不同策略：

```text
WARRANTY_CARD -> OCR 高优先级
TAG           -> OCR 高优先级
CASEBACK      -> OCR + pattern detector
DIAL          -> 品牌/系列视觉辅助
OTHER         -> 低优先级
```

这样既能提升 precision，也能减少所有图片都跑 OCR/CLIP 的成本。

另外 OCR evidence 必须保留：

- 原图 ID；
- bbox；
- raw OCR text；
- per-character/word confidence；
- normalization 结果；
- candidate exact/fuzzy status。

人工复核才能真正解释模型为什么判断某 reference。

---

# 20. Decision State 必须至少有四类

建议不要只用 `matched = true/false`。

应该定义：

```text
AUTO_LINK
NEED_REVIEW
CONFLICT_REJECT
NO_LINK
```

语义分别是：

### AUTO_LINK

存在足够强、无冲突的 reference 证据，可以自动绑定到全局 Reference Entity。

### NEED_REVIEW

有较强候选，但缺少允许自动放行的证据。

### CONFLICT_REJECT

存在两个或更多强证据冲突。此状态比普通 NEED_REVIEW 更应该优先人工处理，因为可能暴露数据源质量问题。

### NO_LINK

没有可靠候选，或者 reference 不在当前 KB。

这四态模型比二分类更适合持续增量系统。

---

# 21. 推荐的数据模型

## 21.1 SourceRecord

```sql
SourceRecord
------------
record_id
source
source_record_id
brand_raw
brand_id
title
description
attrs_json
image_urls
product_role
ingested_at
source_updated_at
raw_payload_hash
```

`UNIQUE(source, source_record_id)`。

## 21.2 ReferenceEvidence

前面已给出，重点是保留 provenance。

## 21.3 CandidateScore

```sql
CandidateScore
--------------
record_id
reference_entity_id
lexical_score
edit_distance
text_embedding_score
image_score
nli_entailment
nli_contradiction
retrieval_reason
model_version
```

## 21.4 LinkDecision

```sql
LinkDecision
------------
record_id
reference_entity_id nullable
decision
reason_code
policy_version
extractor_version
model_version
created_at
```

## 21.5 LinkEvidenceMap

```sql
LinkEvidenceMap
---------------
link_decision_id
evidence_id
role  -- SUPPORT / CONFLICT / AUXILIARY
```

最终所有自动链接都能回答：

> “为什么这条数据被绑定到这个 reference？”

---

# 22. 推荐的核心决策伪代码

```python
def resolve_reference(record):
    brand = resolve_brand(record)
    if brand is None:
        return Decision.NO_LINK("brand_unresolved")

    product_role = classify_product_role(record)

    evidences = extract_reference_evidence(
        record=record,
        brand=brand,
    )

    # 只看高可信证据是否互相冲突
    strong = [e for e in evidences if e.trust_tier in {"A", "B"}]
    strong_refs = {
        e.canonical_value
        for e in strong
        if e.is_valid_brand_reference
    }

    if len(strong_refs) > 1:
        return Decision.CONFLICT_REJECT(
            reason="multiple_strong_reference_conflict",
            evidences=strong,
        )

    # Fast path：唯一强 reference + KB exact hit
    if len(strong_refs) == 1:
        ref = next(iter(strong_refs))
        entity = reference_kb.get_exact(brand, ref)

        if (
            entity is not None
            and product_role.allows_reference_link
            and policy.allow_auto_link(record, strong, entity)
            and not has_any_strong_conflict(record, entity, evidences)
        ):
            return Decision.AUTO_LINK(entity, evidences=strong)

    # Slow path：只召回候选，不直接判同
    candidates = retrieve_reference_candidates(
        record=record,
        brand=brand,
        evidences=evidences,
    )

    verified = []
    for candidate in candidates:
        result = strict_verify(record, candidate, evidences)

        if result.has_hard_conflict:
            continue

        if result.has_auto_link_grade_reference_support:
            verified.append((candidate, result))

    if len(verified) == 1:
        candidate, result = verified[0]
        return Decision.AUTO_LINK(candidate, evidences=result.evidences)

    if len(verified) > 1:
        # 不选 top1，直接拒识
        return Decision.CONFLICT_REJECT(
            reason="multiple_verified_candidates"
        )

    if candidates:
        return Decision.NEED_REVIEW(candidates[:10])

    return Decision.NO_LINK("no_reliable_reference_candidate")
```

这里最关键的不是模型，而是：

```text
len(verified) != 1
=> 不自动链接
```

---

# 23. 规模化：100 万～1000 万记录如何跑

当前数据规模并不要求所有数据都进入 GPU 模型。

建议做明显的 Fast Path / Slow Path 分流。

### Fast Path

大概率记录只需要：

```text
结构化字段 / regex extraction
        ↓
canonicalization
        ↓
PostgreSQL / KV exact lookup
        ↓
AUTO_LINK
```

这部分 CPU 即可，成本很低。

### Slow Path

只对：

- reference 缺失；
- reference 不合法；
- 多个候选；
- OCR 噪声；
- 强证据冲突；

运行：

- OCR；
- SBERT；
- CLIP；
- NLI；
- LLM；
- 人工复核。

这样能把 GPU/LLM 成本控制在少数 difficult cases 上。

---

# 24. 推荐的服务拆分

可以按下面方式落地：

```text
                      ┌────────────────────┐
                      │ Kafka / Queue      │
                      └─────────┬──────────┘
                                │
                                ▼
                  ┌────────────────────────┐
                  │ Record Normalizer      │
                  └─────────┬──────────────┘
                            │
                            ▼
               ┌─────────────────────────────┐
               │ Reference Extractor Service │
               │ regex / parser / role       │
               └──────────┬──────────────────┘
                          │
             ┌────────────┴─────────────┐
             │                          │
        clear exact ref             ambiguous
             │                          │
             ▼                          ▼
     Reference KB Lookup      Candidate Retrieval Service
                                       │
                                       ▼
                              OCR / SBERT / CLIP / LLM
                                       │
                                       ▼
                              Strict Verifier Service
                                       │
                                       ▼
                              Decision / Audit Store
                                       │
                          ┌────────────┴───────────┐
                          ▼                        ▼
                    AUTO_LINK                Review Queue
```

### 存储建议

初期完全可以：

```text
PostgreSQL
+ object storage for images
+ Redis optional cache
```

如果后续 embedding candidate retrieval 占比变高，再增加：

```text
FAISS / Qdrant / Milvus / pgvector
```

没有必要一开始就为了 1000 万条记录引入复杂向量基础设施，因为 reference exact index 的吞吐远高于模型检索。

---

# 25. 增量更新策略

每条来源记录计算：

```text
raw_payload_hash
```

当抓取到新版本时：

```text
hash 未变化 -> 跳过
hash 变化   -> 重新抽取 evidence
```

同时保存：

```text
extractor_version
normalization_rule_version
policy_version
model_version
```

如果未来更新了 Rolex 的 normalization rule，可以只重算受影响品牌，而不是重跑全部数据。

---

# 26. Hard Negative：人工标注几百对应该标什么

Spec 允许人工标注几百对黄金标签。

不要随机采样，因为随机负例太简单，对 precision-first 几乎没价值。

建议集中标注以下 hard negatives：

### 26.1 相邻 reference

```text
126610LV vs 126610LN
```

只差一个后缀字符。

### 26.2 OCR 易混字符

```text
O / 0
I / 1 / L
S / 5
B / 8
Z / 2
```

### 26.3 同系列不同尺寸/材质/年代

标题高度相似、图片高度相似，但 reference 不同。

### 26.4 配件兼容场景

```text
“适配 126610LV 表带”
```

负例必须明确标注为：

```text
商品本身 reference != 126610LV
```

### 26.5 平台 SKU 冒充 reference

不同来源各自的内部编号很可能也是字母数字串。

### 26.6 串图/错图

文字和图片指向不同 reference。

### 26.7 相同标题模板

商家批量复制标题，只改一个型号字符。

### 26.8 Reference 缺失但视觉很像

验证系统是否会因为 CLIP 高分产生 false positive。

这几百条数据应该优先用于：

```text
1. policy safety test
2. reference role classifier
3. OCR confusion calibration
4. NLI contradiction detector
5. 阈值/拒识策略校准
```

而不是一开始训练一个大而泛化不足的 pairwise classifier。

---

# 27. Precision-First 的评估指标

传统 EM 常看：

```text
Precision / Recall / F1
```

当前系统需要更细分。

建议至少记录：

```text
reference_extraction_precision
reference_extraction_recall
candidate_recall@K
AUTO_LINK_precision
AUTO_LINK_coverage
false_link_count
conflict_rate
abstain_rate
manual_review_precision
source_pair_precision
```

其中发布门槛最重要的是：

```text
AUTO_LINK_precision
false_link_count
```

### 27.1 不要被“0 个错误”误导

如果安全测试集只有 500 个 AUTO_LINK 样本，观察到 0 个错误，并不能证明真实误匹配率是 0。

粗略的“rule of three”说明：当观察到 0 个错误时，95% 置信下错误率上界约为：

```text
3 / N
```

例如：

```text
N = 500
上界约 0.6%
```

这距离“千万级数据里几乎不出现误合并”仍然很远。

因此几百条黄金标签适合开发阶段找 hard case，但生产上线后必须持续扩大安全评测集和人工抽检样本。

---

# 28. 线上监控不能只看平均 Precision

应该按切片监控：

```text
source
brand
series
extractor_type
reference_pattern
OCR / non-OCR
structured / unstructured
new / seen reference
policy_version
model_version
```

尤其要监控：

```text
new brand
new reference pattern
new source field mapping
OCR 分布变化
```

当某个切片没有足够验证数据时，自动降级成 NEED_REVIEW，而不是沿用全局阈值。

---

# 29. 直接可以从 Ameli 借鉴的 7 个实现思想

## 29.1 两阶段架构

```text
Candidate Retrieval
      ->
Fine-grained Verification
```

直接保留。

## 29.2 Embedding 预计算

Ameli 把 entity embedding 预计算后用于检索。

我们也可以对 Reference Entity 的：

- title/series description；
- representative images；

预计算 embedding，避免每条增量数据重复编码实体库。

## 29.3 候选闭集属性抽取

这是最应该复制的思想：

```text
先检索候选
再只在候选已有属性值中抽取
```

对 reference 能显著降低自由生成错误。

## 29.4 Attribute Filter 先于 Neural Rerank

先用 deterministic reference/brand/role 冲突把不可能候选删掉，再运行 NLI/CLIP。

这比把所有特征交给一个大模型学习权重更可控。

## 29.5 多模态分数分开保存

不要只保存最终 fused score。

保存：

```text
text score
image score
OCR evidence
attribute match
NLI contradiction
```

便于审计和重新制定 policy。

## 29.6 Hard Negative 来自最近邻候选

Ameli 数据构造会让人工在相似候选中做判断。

这正适合当前项目：优先把最容易误匹配的“近邻 reference”交给人工标注，而不是随机抽负例。

## 29.7 模型是插件，不是唯一决策器

Ameli 代码已经把 retrieval、attribute、disambiguation 模块分开。

当前工程也应该保持：

```text
ReferenceExtractor
CandidateRetriever
OCRExtractor
NLIConflictDetector
ImageConflictDetector
DecisionPolicy
```

可独立升级，而不是训练一个不可解释的 end-to-end 模型。

---

# 30. Ameli 原方案中必须修改的 6 点

## 30.1 从“必选一个”改成“默认拒识”

Ameli：

```text
argmax candidates
```

当前项目：

```text
no unique verified candidate
=> ABSTAIN
```

## 30.2 System Attribute 的 precision 不够用于自动合并

论文中多来源 System Attribute 联合 precision 约 94.33%。

这对普通 ML benchmark 已经很好，但对“绝不能误匹配”远远不够。

所以必须按 evidence type 独立设权限，而不是把所有属性抽取器 union 后直接当真值。

## 30.3 图像不能作为强正证据

Ameli 会把 image score 融入最终选择。

我们只能把视觉用作辅助召回、冲突检测和人工排序。

## 30.4 目标实体从 Product Entity 改成 Reference Entity

这是最大重构。

一个 reference 下可能有很多不同平台 listing、不同成色、不同年份的二手记录，它们都应该链接到同一个 reference entity。

## 30.5 NLI 从 selector 改成 veto

模型更适合说“这个候选与证据矛盾”，而不是独立证明 identity。

## 30.6 Retrieval 召回率不应决定最终匹配

真实 reference 不在 Top-K 时，应 NO_LINK，而不是选择另一个相似 reference。

---

# 31. 一个实际例子：126610LV vs 126610LN

假设有三条数据：

```text
雷小安：
劳力士 潜航者 绿水鬼 126610LV 全套

腕表之家：
Rolex Submariner Date 126610LV

奢当家：
劳力士 黑水鬼 126610LN 41mm
```

通用 SBERT：

```text
三条文本都高度相似
```

CLIP：

```text
同系列外观也高度相似
```

传统多模态 matcher 可能把三条都聚到一起。

R-MEL：

```text
记录 1 -> canonical 126610LV
记录 2 -> canonical 126610LV
记录 3 -> canonical 126610LN

=> 1/2 同 Reference Entity
=> 3 明确不同
```

这说明 reference identity 应该高于所有语义/视觉相似度。

---

# 32. 一个更危险的例子：配件

商品标题：

```text
“劳力士 126610LV 适配橡胶表带 绿色”
```

简单 reference parser 会抽到：

```text
126610LV
```

如果直接 exact join，就会把这条表带记录链接到腕表本体。

因此正确流程是：

```text
identifier extraction -> 126610LV
product role          -> STRAP / ACCESSORY
identifier role       -> ACCESSORY_COMPATIBLE_REFERENCE

=> 禁止 AUTO_LINK 到腕表 Reference Entity
```

这也是为什么“reference 抽到了”还不够，必须有 role semantics。

---

# 33. 一个 OCR 冲突例子

结构化字段：

```text
126610LV
```

保卡 OCR：

```text
1266101V
```

模型 fuzzy match 认为：

```text
1266101V ~ 126610LV
edit distance = 1
```

不应立刻自动修正为 126610LV。

正确做法：

```text
OCR raw = 1266101V
candidate = 126610LV
confusion = 1/L
confidence = low

=> OCR 只作为弱辅助证据
```

如果另一张清晰标签图又 OCR 出 `126610LV`，再提升证据等级。

---

# 34. 为什么应该先做规则基线，而不是先训练 Ameli 全模型

当前任务的定义极度结构化：identity 由 reference 决定。

因此 Phase 0 很可能已经能解决大量数据：

```text
brand normalization
+ reference exact extraction
+ conservative normalization
+ reference KB exact lookup
+ conflict veto
```

相比直接训练 SBERT/CLIP/NLI：

- 更快上线；
- 更便宜；
- 更容易审计；
- 更符合业务定义；
- 更容易做到高 precision。

Ameli 应该用于难例，不应该反过来成为所有记录必须经过的主干模型。

---

# 35. 推荐实施阶段

## Phase 0：Reference KB + Exact Pipeline

目标：先拿下最安全的自动链接覆盖率。

实现：

```text
brand canonicalization
reference parser
brand-specific grammar
reference role classifier
ReferenceEntity KB
strict decision policy
audit table
```

上线前优先验证：

- 相邻型号；
- 兼容配件；
- 来源 SKU；
- 后缀字符；
- 标题复制错误。

## Phase 1：OCR Evidence

只对 reference 缺失或冲突记录跑 OCR。

优先识别：

```text
保卡
吊牌
盒标
表背
```

OCR exact hit 可以成为高价值辅助证据。

## Phase 2：Ameli-style Candidate Retrieval

加入：

```text
SBERT text retrieval
CLIP image retrieval
lexical/fuzzy reference retrieval
```

用途只有：

```text
找候选
```

## Phase 3：Attribute + NLI Conflict Detector

把 title/description/image 中的：

```text
series
color
material
size
reference
```

与 candidate reference metadata 对比。

NLI 重点学习 contradiction。

## Phase 4：Human-in-the-loop

人工处理：

```text
CONFLICT_REJECT
NEED_REVIEW
high-value new reference
new brand pattern
```

结果回流：

- 新 alias；
- 新 normalization rule；
- 新 hard negative；
- OCR confusion table；
- role classifier；
- NLI contradiction model。

---

# 36. 最小可落地版本（MVP）

如果希望最快开始，可以先只做 5 个组件：

```text
1. Brand Resolver
2. Reference Extractor
3. Reference Normalizer
4. Reference KB
5. Strict Decision Policy
```

核心接口：

```python
result = resolve_reference(record)
```

输出统一 JSON：

```json
{
  "record_id": "sdj_123",
  "decision": "AUTO_LINK",
  "reference_entity_id": "rolex:126610lv",
  "brand": "ROLEX",
  "canonical_reference": "126610LV",
  "reason_code": "STRUCTURED_AND_TITLE_EXACT_AGREE",
  "evidences": [
    {
      "type": "STRUCTURED_FIELD",
      "raw": "126610LV",
      "canonical": "126610LV"
    },
    {
      "type": "TITLE_EXACT",
      "raw": "126610LV",
      "canonical": "126610LV"
    }
  ],
  "policy_version": "ref_policy_v1"
}
```

这已经可以对三来源做稳定 join。

---

# 37. 三源最终匹配不需要再训练 pairwise model

得到：

```text
record_id -> reference_entity_id
```

以后查询同款商品只需要：

```sql
SELECT *
FROM source_record r
JOIN link_decision d
  ON r.record_id = d.record_id
WHERE d.reference_entity_id = ?
  AND d.decision = 'AUTO_LINK';
```

或者离线生成：

```text
reference_entity_id
  -> 雷小安 records
  -> 腕表之家 records
  -> 奢当家 records
```

不需要每次重新做跨平台 pairwise comparison。

这对持续增量尤其重要：新来一条记录只需链接一次到 Reference Entity，就自动与历史所有同 reference 记录关联。

---

# 38. 与千万级数据的复杂度对比

假设三来源总量 1000 万。

### Pairwise 方案

跨源笛卡尔组合可能达到不可接受的数量级。

即便 blocking 后，仍然需要管理大量 pair。

### Reference Linking 方案

每条记录独立执行：

```text
extract -> normalize -> KB lookup
```

大多数 fast path 接近：

```text
O(N)
```

仅少量 ambiguous records 进入 ANN retrieval / model inference。

Reference KB 本身的实体数量通常远小于 listing 数量，因此工程成本会显著降低。

---

# 39. 推荐 reason_code 设计

所有结果都应该带机器可统计的 reason code。

例如：

```text
AUTO_STRUCTURED_REF_EXACT
AUTO_TITLE_REF_EXACT
AUTO_MULTI_EVIDENCE_AGREE
AUTO_OCR_TEXT_AGREE
REJECT_REFERENCE_CONFLICT
REJECT_PRODUCT_ROLE_ACCESSORY
REJECT_BRAND_CONFLICT
REJECT_INVALID_REFERENCE_PATTERN
REVIEW_ONLY_FUZZY_REFERENCE
REVIEW_ONLY_IMAGE_SIMILARITY
REVIEW_ONLY_LLM_CANDIDATE
NO_REFERENCE_FOUND
REFERENCE_NOT_IN_KB
```

这样后续可以准确回答：

> 哪种自动路径产生了错误？

而不是只有一个不可解释的模型 probability。

---

# 40. 可以直接参考 Ameli 代码的模块映射

官方仓库当前结构主要包括：

```text
attribute/
retrieval/
disambiguation/
candidate_retrieval.py
entity_disambiguation_v2.py
```

与当前系统可对应为：

```text
Ameli retrieval/
    -> ReferenceCandidateRetriever

Ameli attribute/
    -> ReferenceEvidenceExtractor
       + CandidateConstrainedAttributeExtractor

Ameli disambiguation/train_nli.py
    -> ReferenceContradictionDetector

Ameli model_contrastive.py
    -> Optional ImageConflictModel

Ameli entity_disambiguation_v2.py
    -> Offline training/evaluation orchestration
```

不建议把仓库整体 fork 后直接改成生产服务，因为它：

- 研究代码路径较多；
- 有多种实验模型和注释掉的架构；
- CLI/训练参数与 Best Buy 数据格式耦合明显；
- 默认 GPU 训练/多进程配置不是当前业务的最小需求。

更合理的是抽取它的 architecture pattern 和 loss/design，再重写成当前领域的轻量服务。

---

# 41. 训练资源与工程现实

论文训练中 retrieval/disambiguation 都使用 GPU，官方仓库也依赖：

- PyTorch；
- Sentence-Transformers；
- CLIP；
- Ray Tune；
- 多 GPU / DDP 路径；
- WandB。

这些对于研究复现合理，但当前线上系统无需把每一条数据都送进这些模型。

建议：

```text
CPU rules / exact lookup 处理 80%~95%+ 的明确记录
GPU/LLM 只处理 ambiguous tail
```

具体比例以真实数据统计为准，不应预设。

---

# 42. 最值得做的一项离线分析

在训练任何新模型之前，先统计三来源：

```text
1. reference 独立字段覆盖率
2. 标题中 reference 可 exact 抽取比例
3. reference 格式按品牌分布
4. 同一 reference 的写法变体数
5. 单条记录出现多个“像 reference”的 identifier 比例
6. accessory/compatible 商品比例
7. 图片中可 OCR 到 reference 的比例
8. 结构化字段 vs 标题 vs OCR 的冲突率
```

这些数据将直接决定：

- 是否需要 LLM；
- 是否需要 CLIP；
- OCR 应跑哪些图片；
- 哪些品牌最需要 brand-specific parser；
- 自动链接能覆盖多少。

很可能真正的大头工作会是数据质量和 reference parser，而不是模型结构。

---

# 43. 最终推荐方案

针对当前 Spec，我建议最终采用：

```text
Global Reference KB
        +
Conservative Reference Normalization
        +
Multi-source Reference Evidence Extraction
        +
Ameli-style Candidate Retrieval
        +
Candidate-constrained Attribute Extraction
        +
Hard Conflict Filtering
        +
NLI / Image as Veto or Auxiliary Evidence
        +
Explicit Abstention
        +
Human Review / Feedback Loop
```

其中优先级排序是：

```text
P0 Reference identity / normalization / role
P1 Exact evidence + conflict veto
P2 OCR evidence
P3 Candidate retrieval
P4 NLI / image difficult-case verifier
```

而不是：

```text
P0 训练一个 multimodal matcher
```

---

# 44. 一句话总结 Ameli 对当前项目的真正价值

Ameli 最值得借鉴的不是它最终的多模态融合分数，而是它证明了：

> **在高度相似的商品候选之间，细粒度属性比泛化的“整体相似度”更关键。**

把这个思想迁移到腕表场景后，`reference number` 就应该成为拥有最高决策权的 identity attribute；SBERT、CLIP、OCR、LLM、NLI 都只能围绕它做候选召回、证据补全和冲突检测。

最终生产原则可以压缩成三句话：

```text
Embedding 找候选。
Reference 定身份。
有冲突就拒识。
```

这比直接复制 Ameli 的 `Top-K + fused argmax` 更符合当前“precision 优先到极致、允许漏匹配、绝不能误匹配”的实际要求。
