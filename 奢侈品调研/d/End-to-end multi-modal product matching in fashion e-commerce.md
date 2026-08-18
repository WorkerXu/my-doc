# End-to-end multi-modal product matching in fashion e-commerce

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本轮从 `奢侈品文章调研.md` 中选择 **End-to-end multi-modal product matching in fashion e-commerce**（Tóth et al., Zalando，arXiv:2403.11593）进行深入分析。

- 论文：<https://arxiv.org/abs/2403.11593>
- HTML 全文：<https://arxiv.org/html/2403.11593v1>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

在分析前已检查 `奢侈品调研/d`，该文章此前没有被 d 分析过。

这篇论文最值得当前项目借鉴的不是“用 CLIP 做商品相似度”，而是它提供了一条真实工业系统已经走通的完整链路：

```text
多模态编码
  -> Blocking
  -> ANN / Top-K retrieval
  -> threshold discriminator
  -> 高置信自动通过
  -> 中低置信进入 Human-in-the-loop
```

论文使用冻结的预训练图像/文本编码器，将多张图片平均池化，与文本及数值特征拼接，再只训练一个低维线性投影；全部商品预先编码后，在线/离线只做向量近邻检索。它还专门评估了跨时间、跨新域的数据漂移，并把人工复核作为生产链路的一部分。

但是，**当前腕表 Spec 不能直接复制论文的最终判定方式**。原因是业务定义已经非常明确：

> **同一个商品 = 同一品牌下同一个 reference number / 型号。**

而且要求 **绝对不能误匹配，precision 优先到极致**。

因此建议把论文架构改造成：

> **Reference-first + Multimodal Retrieval + Evidence Gate + HITL**

其中：

- reference 是“身份主键”；
- 多模态 embedding 只负责召回、排序、发现疑难样本和辅助人工；
- 模型相似度永远不能单独产生自动 MATCH；
- 自动 MATCH 必须收口到 `(canonical_brand, canonical_reference)` 的硬一致性；
- 一旦出现高可信 reference 冲突，直接 `CONFLICT`，不能让图像相似度覆盖；
- 没有足够 reference 证据时宁可 `ABSTAIN`；
- 人工复核也应被设计为“验证 reference 证据”，而不是“凭外观看是不是同一只表”。

推荐的最终架构如下：

```text
雷小安 / 腕表之家 / 奢当家
          |
          v
[1] Raw Ingestion + Source Snapshot
          |
          v
[2] Brand Canonicalization
          |
          v
[3] Reference Evidence Extraction
    - structured field
    - title / description
    - OCR from caseback/card/tag
    - source-specific parser
          |
          v
[4] Reference Role Classifier
    reference / serial / sku / internal_id / movement / unknown
          |
          v
[5] Brand-specific Reference Normalizer + Registry Lookup
          |
          +----------------------------+
          |                            |
          | verified reference         | unresolved / ambiguous
          v                            v
[6A] Hard Exact Entity Key       [6B] fashionID-like Multimodal Retrieval
(brand_id, canonical_ref)            brand blocking + vector Top-K
          |                            |
          |                            v
          |                      Candidate references/items
          |                            |
          +------------+---------------+
                       v
[7] Evidence Gate / Conflict Veto
    - same verified reference
    - trusted source agreement
    - no conflicting trusted reference
    - multimodal only support/veto
                       |
           +-----------+-----------+
           |           |           |
         MATCH      ABSTAIN      CONFLICT
                       |           |
                       +-----+-----+
                             v
                     [8] Expert HITL
                     top-3 candidate references
                     evidence highlighted
                             |
                             v
                     Verified Reference Registry
                             |
                             v
                     Incremental feedback loop
```

如果只做一个 MVP，我建议甚至不要先训练复杂的 pairwise matcher，而是先做：

```text
品牌规范化
+ reference 多源抽取
+ 编号角色识别
+ brand-specific canonicalization
+ exact entity key
+ 冲突拒绝
+ 多模态 Top-K 人工辅助
```

这会比“直接训练一个同款分类器”更符合当前 precision-first 约束，也更容易解释、回滚、审计和持续增量。

---

## 1. 为什么选择这篇论文

当前 Spec 有几个关键难点：

1. 数据量 100 万–1000 万级；
2. 来源有 3 个且持续增量；
3. 字段高度稀疏、格式不统一；
4. reference 有时是结构化字段，有时埋在标题或图片；
5. 图片可用；
6. 允许漏匹配，但不能误匹配；
7. 未来还可能增加新来源，存在数据分布漂移；
8. 可以人工标注几百对，但不适合海量人工搜索。

这篇 Zalando 论文恰好从“生产系统”视角处理了：

- 大规模商品集合；
- 多 seller/domain；
- 标题表达方式不同；
- 图片拍摄角度和风格不同；
- 新域（out-domain）；
- 时间漂移；
- 大量商品没有跨域匹配对象；
- 自动模型无法达到业务 precision 时如何接 HITL。

论文训练集约包含：

```text
index offers: 1.5M
query offers: 90k
positive pairs: 0.2M
train domains: 4
```

测试阶段还有完全未见过的新域，并且测试集合中超过 99% 的 index offers 在 query 域里没有匹配对象。

这和当前实际环境非常像：

> 绝大多数跨平台组合都不是 match，所以系统设计的核心不只是“找到像的”，而是要在海量 negative 中不误合并。

论文还有一个非常重要的工程取舍：

> 不端到端微调整个大模型，而是冻结预训练 encoder，预计算 embedding，只训练一个很小的 projection layer。

这样可以获得：

- 训练便宜；
- 快速迭代；
- embedding 可缓存；
- 大 batch contrastive training 成本低；
- 新域部署容易；
- 线上检索架构简单。

对当前项目来说，这非常适合把“多模态模型”定位为一个 **候选生成基础设施**，而不是最终裁决器。

---

## 2. 论文原始技术实现

## 2.1 问题定义

论文中每个 domain 可以理解为一个卖家或电商网站，每个 offer 包含：

```text
Offer
  ├── image set
  ├── brand
  ├── title
  ├── price
  ├── number of sizes
  └── optional product codes
```

任务是在两个 domain 之间找到表示同一商品的 offers。

形式化：

```text
C(offer) -> real product id

match(i, j) = 1
iff
C(i) == C(j)
```

论文虽然也有 product codes，但它的任务并不要求“只有 code 一致才算 match”，所以仍然是一般化的商品 identity matching。

当前腕表任务比它更受约束：

```text
match(i, j) = 1
iff
brand(i) == brand(j)
AND
canonical_reference(i) == canonical_reference(j)
```

这一点决定了我们应该借它的 **retrieval architecture**，而不是照搬其 **semantic match decision**。

---

## 2.2 数据预处理

论文对图片做统一 resize / crop；每个 offer 平均约 4.5 张图片。

图片包含：

- 标准商品图；
- 模特图；
- 局部细节图。

文本只做很轻量的预处理：

```text
normalize unicode(brand)
normalize unicode(title)
join(brand, title)
```

数值特征为：

```text
v_num = [num_sizes, log(num_sizes), log(price)]
```

这体现了论文的设计原则：

> 尽量减少 source-specific feature engineering，让新 domain 可以快速接入。

对腕表系统，可以保留这个原则，但 `reference` 是例外：

- reference 是业务身份定义；
- 必须进行 brand-specific normalization；
- 必须保留 provenance；
- 必须区分 reference 与 serial/SKU/internal id。

因此我们的预处理应该是：

```text
通用轻量预处理
+
极重的 identifier normalization
```

而不是对所有属性都做复杂规则。

---

## 2.3 fashionID 多模态 Encoder

论文最核心的模型是 `fashionID`。

高层结构：

```text
image_1 -> pretrained image encoder --\
image_2 -> pretrained image encoder ----> average pooling --\
image_n -> pretrained image encoder --/                    |
                                                            |
text ------> pretrained text encoder -----------------------+--> concat
                                                            |
numeric features -------------------------------------------/
                                                                  |
                                                                  v
                                                        Linear Projection
                                                                  |
                                                                  v
                                                       normalized embedding
```

论文最佳配置使用 CLIP ViT-bigG-14 同时得到图像和文本 embedding。

特点：

1. pretrained encoder 全部冻结；
2. 多张图片先分别编码，再 average pooling；
3. 图像、文本、数值向量 concat；
4. 只训练最后一个 linear projection；
5. 最终输出 192 维 embedding；
6. projection 参数只有约 0.6M。

论文还比较了：

```text
linear projection
vs
1-hidden-layer MLP
```

MLP 在验证集稍好，但 linear projection 在 in-domain / out-domain 测试集略优，说明低容量投影更能保留预训练模型的泛化能力。

### 对当前项目的直接启示

我们不需要一开始就训练“腕表专用大模型”。

可以做：

```text
image_embedding = frozen_image_encoder(images)
text_embedding  = frozen_text_encoder(title + description)
ocr_embedding   = frozen_text_encoder(ocr_text)
meta_vector      = lightweight numerical/categorical features

multimodal = concat(...)
projection = trainable Linear(... -> 192/256)
```

只训练最后一层。

但它的作用应该是：

> **candidate retrieval score**

不是：

> **最终 identity score**。

---

## 2.4 Supervised Contrastive Learning

论文使用 supervised contrastive loss。

对于同一商品的 offers，让 embedding 靠近；其他 batch 内 offers 作为 negatives。

核心形式：

```text
positive: same product_id
negative: other product_id in batch
```

一个很关键的实现细节是 batch sampling：

```text
sample product_id
-> put all offers of that product into same mini-batch
```

这样能保证 batch 里有足够 positive pairs。

论文还发现：

> large mini-batch 对 contrastive retrieval 很重要。

因为 encoder 都被冻结、embedding 都可以预计算，所以训练时实际上只读取缓存 embedding，然后训练一个小 linear layer，因此可以使用非常大的 batch。

论文实验最终固定到 16k batch size。

### 对腕表的训练集设计

当前我们不应该随机采 negative。

最重要的是构造 **reference hard negatives**：

```text
Positive:
  same brand + same canonical reference

Hard Negative A:
  same brand + same collection + different reference

Hard Negative B:
  same brand + visually almost identical + different reference

Hard Negative C:
  same reference-like token shape but token is platform SKU / serial

Hard Negative D:
  title includes compatible/related reference but sold item itself不是该 reference

Hard Negative E:
  same base reference, different meaningful suffix
```

例如：

```text
126610LN-0001
126610LV-0002
```

如果业务 registry 认定为不同 reference，则它们必须被强制作为 hard negative，即使图片极像。

这类 hard negative 对当前价值远高于随机抽两个不同品牌商品。

---

## 2.5 Blocking

论文指出：

> 在百万级 query/index 上直接全量 nearest-neighbor 或 pairwise 比较不现实，需要先 blocking。

论文用品牌名 fuzzy similarity 做 blocking：

```text
query brand
  -> find compatible index brands
  -> only ANN inside compatible brands
```

这是一种非常朴素但很有效的结构。

### 当前项目的 Blocking 应更强

腕表可以做分层 blocking：

```text
Level 0: canonical_brand
Level 1: exact canonical reference prefix / family（若有）
Level 2: collection / model family
Level 3: multimodal ANN
```

建议：

```text
if verified_reference exists:
    不需要 vector retrieval 做自动 matching
    直接 key lookup

elif candidate_reference exists:
    block = brand + reference_family
    vector retrieve only inside this block

else:
    block = brand
    vector retrieve top-K
```

这样向量检索不是第一层，而是 reference 缺失时的补救层。

---

## 2.6 ANN Retrieval

论文对所有商品预先存 embedding，然后执行 nearest-neighbor search：

```text
similarity(i, j) = cosine(v_i, v_j)
```

并在 Top-K 结果之后再做 threshold filtering。

论文引用 FAISS 相关工作，说明其目标就是标准的大规模向量近邻检索架构。

当前可以实现成：

```text
brand_id -> one ANN shard/index
```

而不是一个 1000 万商品全局向量库。

原因：

1. 不同品牌绝不该自动 match；
2. brand shard 搜索更快；
3. 可按品牌重建模型/索引；
4. 热门品牌和长尾品牌可以使用不同索引策略；
5. 更容易排查误召回。

初版甚至可以用：

```text
每品牌 FAISS HNSW / FlatIP
```

规模上升后再考虑 IVF/PQ 或独立向量数据库。

不要为了架构“先进”一开始就上全局 distributed vector database。

---

## 2.7 Threshold Discriminator

论文的 retrieval 最后一层是：

```text
if nearest_neighbor_distance <= threshold:
    accept candidate
else:
    no-match
```

threshold 通过测试集 precision-recall curve 进行选择。

论文特别强调：

> 对真实生产业务，高 precision、低 recall 区域本身就是重要工作点，F1 不足以衡量。

这个观点与当前 Spec 完全一致。

但当前需要再走一步：

```text
论文：embedding threshold 可以决定自动 match

当前：embedding threshold 只能决定
      “是否值得继续验证 / 是否进入 HITL / 是否做 OCR”
```

最终自动 MATCH 仍需要 reference hard evidence。

---

## 2.8 Human-in-the-loop

这是这篇论文最值得落地借鉴的地方之一。

论文不是简单地说“低置信度人工看一下”，而是实际设计和验证了审核 UI。

论文的经验包括：

1. 给审核员展示 Top-3 nearest neighbors，比只展示 Top-1 更好；
2. 高 precision 场景下，展示多张图片和细节图很重要；
3. 不是信息越多越好，价格、颜色、size 等有时反而干扰人工；
4. 需要持续给审核员做错误反馈和训练；
5. 专业固定审核人员比未训练 crowdsourcing 更可靠；
6. 多人独立投票 + majority aggregation 可以形成稳定生产流程。

论文使用 3 judgments / row，并展示了模型输入 precision 会影响最终 HITL precision。

它还给出：

```text
P_hitl = {1 + (1/P_model - 1) / LR+}^-1
```

其中：

```text
LR+ = TPR / FPR
```

这个公式的意义是：

> 人工审核不是一个固定“99% 准”的黑盒；其最终 precision 与送给人工的候选质量相关。

这对当前项目非常重要。

如果把大量很烂的 multimodal candidates 全送人工，即使审核员能力不变，最终误匹配风险也会上升。

所以 HITL 前必须有强规则过滤。

---

## 3. 论文方案哪些可以直接复用，哪些必须改

## 3.1 可以直接复用的部分

### A. 两阶段结构

```text
candidate retrieval
-> precise validation
```

这是千万级规模必需的。

### B. Embedding 预计算

把图片和文本大模型 embedding 当作可复用资产：

```text
raw image
  -> image embedding cache

raw text
  -> text embedding cache
```

后续可以不断换 projection / scoring，不需要反复跑大 encoder。

### C. 冻结 encoder + 小投影层

极适合少量标注与快速迭代。

### D. Brand Blocking

腕表里甚至可以变成硬规则。

### E. Top-K 而不是 pairwise 全比较

百万级/千万级下必须这么做。

### F. Out-domain / time split evaluation

当前三平台会持续增量，不能只做随机 train/test split。

### G. HITL 产品化

不仅要有“人工审核”，还要设计：

- 候选组织方式；
- 证据展示；
- 投票策略；
- 审核员质量指标；
- 审核反馈闭环。

---

## 3.2 不能直接复用的部分

### A. 不能让 embedding similarity 决定同一 reference

论文任务里视觉非常关键，但腕表存在大量：

```text
视觉极相似
但 reference 不同
```

因此：

```text
visual similarity high
!=
same reference
```

### B. 不能 average pool 后丢掉每张图片的角色

论文把所有商品图片平均池化。

腕表更适合先做图片角色识别：

```text
front / dial
caseback
warranty card
tag
box label
movement
wrist shot
other
```

因为：

- 表背、保卡、吊牌可能直接包含 reference；
- 正面图主要提供系列/外观证据；
- movement 图可能出现机芯号但不是 reference。

推荐：

```text
retrieval embedding: 可以 average/set pooling
reference OCR: 必须逐图保留 provenance
```

### C. Price 只能作为弱冲突信号

二手商品价格受：

- 成色；
- 年份；
- 附件；
- 渠道；
- 商家定价；
- 市场波动

影响很大。

不能用“价格接近”作为身份硬证据。

### D. 人工不能靠“看起来一样”判 MATCH

审核 UI 必须围绕 reference 证据，而非一般视觉相似度。

---

## 4. 针对当前 Spec 的直接落地架构

## 4.1 核心设计原则

建议写进系统设计文档的原则：

```text
P0. Identity is reference-centric.
P1. Precision > Recall.
P2. Model similarity never overrides trusted identifier conflict.
P3. No evidence => abstain.
P4. Every automatic decision must be explainable by evidence.
P5. Every extracted identifier must carry provenance.
P6. New sources are untrusted until calibrated.
P7. Matching is incremental and reversible.
```

---

## 4.2 服务拆分

建议最少拆成以下逻辑服务/worker，不必一开始都独立部署：

```text
1. ingest_worker
2. brand_normalizer
3. reference_extractor
4. identifier_role_classifier
5. reference_normalizer
6. embedding_worker
7. candidate_retriever
8. decision_engine
9. review_service
10. audit/replay_worker
```

MVP 可以是一个 Python monorepo + 多个 queue worker，成熟后再拆微服务。

不要一开始为了“架构”把 10 个逻辑模块变成 10 个 Kubernetes service。

---

## 4.3 建议数据流

```text
raw_offer_ingested
      |
      v
normalize_brand
      |
      v
extract_identifier_candidates
      |
      v
classify_identifier_roles
      |
      v
normalize_reference_candidates
      |
      +---- verified reference ----> exact entity lookup
      |
      +---- ambiguous -------------> OCR / multimodal retrieval
                                      |
                                      v
                                  top-K candidates
                                      |
                                      v
                                  evidence gate
                                      |
                     +----------------+----------------+
                     |                |                |
                   MATCH           ABSTAIN          CONFLICT
                                      |                |
                                      +------> review queue
```

---

## 5. Reference Evidence：系统最重要的数据结构

不要只在商品表上存一个：

```text
reference = "126610LN"
```

必须存“证据”。

建议结构：

```sql
reference_evidence (
    evidence_id           bigint primary key,
    offer_id              bigint not null,
    source                varchar,
    raw_value             varchar,
    canonical_value       varchar,
    role                  varchar,   -- reference/serial/sku/internal_id/movement/unknown
    extraction_channel    varchar,   -- field/title/description/ocr/manual
    field_name            varchar,
    image_id              bigint null,
    span_start            int null,
    span_end              int null,
    parser_version        varchar,
    normalizer_version    varchar,
    role_model_version    varchar,
    extractor_score       float,
    role_score            float,
    registry_hit          boolean,
    registry_entity_id    bigint null,
    trust_level           varchar,   -- HIGH/MEDIUM/LOW
    created_at            timestamp
);
```

一个 offer 可以有多个 evidence：

```text
field: "型号：126610LN"
OCR:   "126610LN"
title: "劳力士潜航者 126610LN 黑水鬼"
image: 保卡照片
```

这样决策层可以判断：

```text
2 个独立高可信 channel 同意
```

比单个 `reference` 字段稳得多。

---

## 6. Reference 抽取

## 6.1 多通道抽取

建议至少：

### Channel A：结构化字段

优先级最高，但必须 source-specific mapping。

例如：

```yaml
sources:
  x:
    reference_fields:
      - model_no
      - reference
  y:
    reference_fields:
      - 型号
  z:
    unsafe_fields:
      - 商品编号
      - 货号
```

“商品编号”在很多平台可能只是平台内部 SKU，不能默认等价于 reference。

### Channel B：标题

先用 regex/tokenizer 发现 reference-like tokens：

```text
字母 + 数字
数字 + 字母
数字 + 分隔符 + 后缀
```

但不直接认定 role。

### Channel C：描述

同样抽 token，但上下文窗口要保留：

```text
"型号 126610LN"
"适配 126610LN"
"同款 126610LN"
"附件编号 ..."
```

这些上下文语义完全不同。

### Channel D：OCR

优先 OCR：

```text
caseback
warranty card
hang tag
box label
certificate
```

正面表盘 OCR 可能只有品牌/系列字样，价值其次。

### Channel E：人工

人工修正必须作为最高 provenance，但仍需保留 reviewer_id 和时间。

---

## 6.2 Identifier Role Classification

这是 precision-first 的关键模块。

输入：

```text
raw token
left/right context
source
field name
image role
brand
format features
```

输出：

```text
REFERENCE
SERIAL
PLATFORM_SKU
SHOP_SKU
MOVEMENT_ID
CASE_ID
CERTIFICATE_ID
YEAR
PRICE
UNKNOWN
```

建议先规则 + 小模型混合，而不是纯 LLM。

### Rule 示例

```python
if field_name in trusted_reference_fields[source]:
    role = REFERENCE

elif field_name in known_sku_fields[source]:
    role = PLATFORM_SKU

elif context contains ["型号", "reference", "ref."]:
    role_score[REFERENCE] += strong_weight

elif context contains ["货号", "商品编号", "库存编号"]:
    role_score[PLATFORM_SKU] += strong_weight

elif image_role == "warranty_card" and token hits brand reference registry:
    role_score[REFERENCE] += strong_weight
```

之后再用一个轻量 classifier 处理规则覆盖不到的情况。

### 为什么不能用 LLM 直接输出 reference

LLM 最大问题不是平均准确率，而是：

> 可能在不存在证据时猜一个看起来合理的 reference。

当前任务无法接受这种 hallucination。

因此 LLM 如果使用，只允许：

```text
candidate extraction
or
role classification
```

并且输出必须回到 registry / parser 做验证。

---

## 7. Brand-specific Reference Normalization

reference normalization 不能只做：

```python
upper().replace('-', '').replace(' ', '')
```

因为分隔符或后缀有时有业务语义。

推荐设计成：

```text
Global Normalizer
      |
      v
Brand Plugin
      |
      v
Reference Registry
```

### 7.1 Global Normalizer

只做安全操作：

```text
Unicode NFKC
全角 -> 半角
trim
统一大小写
统一不可见字符
保留原值
```

### 7.2 Brand Plugin

例如：

```python
normalize_rolex(raw)
normalize_omega(raw)
normalize_cartier(raw)
...
```

只有明确验证过的规则才允许去掉字符。

### 7.3 Registry

建议建：

```sql
reference_registry (
    reference_id           bigint primary key,
    brand_id               bigint not null,
    canonical_reference    varchar not null,
    reference_family       varchar,
    collection_id          bigint null,
    aliases                jsonb,
    status                 varchar,
    source_of_truth        varchar,
    created_at             timestamp,
    updated_at             timestamp,
    unique(brand_id, canonical_reference)
);
```

再建 alias：

```sql
reference_alias (
    brand_id,
    alias,
    canonical_reference,
    alias_type,
    confidence,
    provenance
);
```

重要：alias 必须可审计，不要自动学习后无审核地写入。

---

## 8. VERIFIED Reference 的定义

不要把每个 parser 输出都叫 VERIFIED。

建议 evidence state：

```text
RAW
CANDIDATE
SUPPORTED
VERIFIED
CONFLICT
```

一种初版 gate：

### VERIFIED 条件 1：可信结构化字段 + registry hit

```text
source_field_trust == HIGH
AND role == REFERENCE
AND registry_hit
```

### VERIFIED 条件 2：两个独立通道一致

```text
title candidate == OCR candidate
AND role both REFERENCE
AND registry_hit
```

### VERIFIED 条件 3：人工专家确认

```text
reviewer verified canonical reference
```

### 任何高可信冲突

```text
field reference = A
warranty OCR reference = B
A != B
```

直接：

```text
CONFLICT
```

而不是选择“分数更高”的 A。

---

## 9. fashionID 如何改造成腕表 Multimodal Retriever

## 9.1 输入

建议：

```text
image features
  - front/dial embeddings
  - side/case embeddings
  - detail embeddings

text features
  - title
  - short description

ocr features
  - normalized OCR tokens

structured weak features
  - brand
  - collection
  - diameter bucket
  - material
  - movement family
```

### 重要边界

reference token 本身如果已经 VERIFIED，不应该再主要靠 embedding 找 identity；直接 exact lookup 即可。

Retriever 的主要对象是：

```text
reference unresolved offers
```

---

## 9.2 图片聚合

可以比论文的简单 average pooling 稍作改进。

MVP：

```python
img_vec = mean(valid_image_embeddings)
```

V2：

```text
role-aware pooling
```

例如：

```python
retrieval_vec =
    0.6 * mean(front_and_side_embeddings)
  + 0.4 * mean(detail_embeddings)
```

OCR 类图片不要只压成视觉 embedding；保留其 OCR evidence。

---

## 9.3 文本编码

标题可做：

```text
[BRAND] + title_without_reference_tokens + normalized attributes
```

有意把 reference token 从 retrieval text 中剥离有一个好处：

> 可以独立测量“没有 reference 硬证据时，图文模型到底能召回多少正确 reference”。

否则模型可能只是复制字符串匹配能力，掩盖视觉/语义真正价值。

---

## 9.4 训练目标

使用论文的 supervised contrastive 结构即可：

```text
same canonical reference -> positive
other references -> negative
```

但 batch sampler 改成：

```python
sample brand
sample reference_family
sample multiple references within family
sample offers across sources
```

这样一个 batch 内天然充满 hard negatives。

建议 batch 示例：

```text
Rolex / Submariner
  124060
  126610LN
  126610LV
  116610LN
  116610LV
```

这远比：

```text
Rolex vs Chanel bag vs Cartier ring
```

更能训练出实际需要的边界。

---

## 9.5 输出与使用方式

Retriever 只输出：

```json
{
  "offer_id": 123,
  "candidates": [
    {"reference_id": 10, "score": 0.94},
    {"reference_id": 11, "score": 0.92},
    {"reference_id": 12, "score": 0.89}
  ]
}
```

不要输出：

```json
{"is_match": true}
```

候选分数进入后面的 evidence engine。

---

## 10. Candidate Reference Retrieval，而不是 Candidate Offer Retrieval

这是我建议对论文最大的业务化改造。

论文检索的是具体 offer。

当前更适合检索：

```text
canonical reference entity
```

也就是一个 reference 可以聚合多个已经 VERIFIED 的 offers：

```text
ReferenceEntity
  126610LN
    ├── verified offer from source A
    ├── verified offer from source B
    ├── verified offer from source C
    ├── prototype image embeddings
    ├── text prototype
    └── known attributes
```

对于 unresolved offer：

```text
offer -> Top-K ReferenceEntity
```

这样比 offer-to-offer 检索更稳，原因：

1. 同一 reference 有多个样本可以形成 prototype；
2. 减少候选重复；
3. 候选天然对应业务实体；
4. 审核员直接选择 reference；
5. 一旦确认 reference，跨源聚合自动完成。

可以使用 prototype：

```text
reference_visual_centroid
reference_text_centroid
reference_attribute_profile
```

也可以保留多个 exemplar，防止 centroid 抹掉不同拍摄风格。

---

## 11. 最终 Decision Engine

这是整个系统最需要保守设计的部分。

建议不要训练一个 end-to-end score，而是显式规则 gate。

### 11.1 自动 MATCH

```python
def decide(a, b):
    if a.brand_id != b.brand_id:
        return NON_MATCH

    if has_trusted_reference_conflict(a) or has_trusted_reference_conflict(b):
        return CONFLICT

    if a.verified_reference is None or b.verified_reference is None:
        return ABSTAIN

    if a.verified_reference != b.verified_reference:
        return NON_MATCH

    return MATCH
```

换成跨源实体映射时：

```python
entity_key = (brand_id, verified_canonical_reference)
```

### 11.2 Multimodal 分数的权限

允许它做：

```text
- 候选排序
- 触发 OCR
- 触发人工
- 发现 possible metadata error
- 低相似度时做 veto warning
```

不允许它做：

```text
reference 不同但视觉像 -> MATCH
reference 缺失但视觉极像 -> MATCH
trusted OCR 与 field 冲突但模型分高 -> MATCH
```

---

## 12. Negative Evidence / Conflict Veto

Precision-first 系统的关键不是堆 positive feature，而是构造强 negative evidence。

建议定义：

```text
N1. trusted_reference_mismatch
N2. brand_mismatch
N3. identifier_role_conflict
N4. incompatible reference family
N5. mutually exclusive attributes
N6. OCR trusted reference conflict
N7. one candidate contains explicit “compatible with / 适配” context
N8. accessory vs watch-body mismatch
```

任何 N1/N2 都直接阻断自动 match。

N3-N8 进入：

```text
CONFLICT / REVIEW
```

而不是简单扣 0.2 分。

这和传统：

```text
score = 0.4 * text + 0.4 * image + 0.2 * metadata
```

有本质区别。

当前业务不适合“加权平均后阈值”，因为一个强冲突不能被多个弱相似信号抵消。

---

## 13. HITL 审核界面

强烈建议借论文的 Top-3 思路，但界面改成“Reference Verification”。

### 左侧：待解析商品

显示：

```text
source
brand
原始 title
reference-like token 高亮
结构化字段
OCR 图片
OCR token 高亮
正面/背面/保卡/吊牌图片
```

### 右侧：Top-3 reference candidates

每个候选显示：

```text
canonical_reference
brand
collection
known aliases
代表图
已 VERIFIED 的跨源 offers
候选命中原因
冲突原因
```

### 操作按钮

不要只有：

```text
MATCH / NO MATCH
```

建议：

```text
[确认 reference A]
[确认 reference B]
[确认 reference C]
[不是以上任何一个]
[证据冲突]
[无法判断]
```

“无法判断”必须是一等公民。

### 双人审核策略

对进入最终 entity merge 的高风险候选，可以：

```text
2 个专家独立判断
只有 2/2 一致才 VERIFIED
否则 escalated review
```

如果量大，可以根据风险分层：

```text
高风险：2/2
普通疑难：1 expert + rule verification
```

---

## 14. 如何利用“几百对黄金标签”

不要把几百对全拿去随机训练二分类器。

建议分成 4 类。

### 14.1 100–150 对：reference extraction / role

重点覆盖：

```text
reference
serial
platform SKU
movement no.
compatible reference
```

### 14.2 100–150 对：same-family hard negatives

例如：

```text
同品牌同系列不同 reference
```

这是最重要的误匹配边界。

### 14.3 50–100 对：跨 source positive

同 reference，不同平台、不同标题、不同图。

### 14.4 50–100 对：冲突案例

例如：

```text
field=A, OCR=B
标题多个 reference
配件标题引用主表 reference
```

用于测试 reject / conflict gate。

### 训练与校准分离

一定要预留 calibration/test：

```text
train: 60%
calibration: 20%
test: 20%
```

样本少时可以做重复交叉验证，但最终 precision threshold 不能在训练数据上直接选。

---

## 15. 评估方法

论文一个非常好的点是不用 F1 作为唯一指标。

当前应该更激进：

## 15.1 Primary Metric

```text
Automatic-Match Precision
```

目标不是：

```text
F1 最大
```

而是：

```text
precision >= 业务允许下限
```

如果业务口径真的是“绝对不能误匹配”，上线时应把自动接受阈值设置为非常保守，并通过 audit 持续验证。

## 15.2 Coverage

```text
auto_match_coverage
```

在满足 precision 之后，再最大化 coverage。

## 15.3 Reference Extraction Metrics

按 channel 分开：

```text
structured-field precision
text-extraction precision
OCR extraction precision
role classification precision
canonicalization precision
```

## 15.4 Candidate Recall@K

只对 unresolved offers：

```text
R@1
R@3
R@10
```

Retriever 的任务是高 recall，不是最终 precision。

## 15.5 Conflict Recall

专门构造已知冲突数据：

```text
trusted reference mismatch detected?
```

## 15.6 人工队列指标

```text
review precision
review throughput
abstain rate
inter-reviewer agreement
escalation rate
```

---

## 16. 数据切分必须模拟真实上线

不要随机 split 商品对。

至少做：

### Split A：Time split

```text
过去 -> train
未来增量 -> test
```

### Split B：Source holdout

例如：

```text
train: 雷小安 + 腕表之家
holdout: 奢当家
```

测试新来源接入。

### Split C：Reference holdout

测试从未见过的新 reference。

### Split D：Brand holdout（可选）

测试规则/模型是否能扩展到新品牌。

### Split E：Hard family split

确保同系列不同 reference 同时出现在 test。

论文专门做 out-domain 与 time shift，这一点应直接继承。

---

## 17. 增量处理架构

当前不是一次性全量任务，所以每条 offer 都需要状态机。

建议：

```text
INGESTED
 -> BRAND_NORMALIZED
 -> IDENTIFIERS_EXTRACTED
 -> REFERENCE_RESOLVED
 -> MATCHED

or
 -> NEEDS_EMBEDDING
 -> CANDIDATES_READY
 -> REVIEW_REQUIRED
 -> VERIFIED

or
 -> CONFLICT
```

建议每个阶段都记录：

```text
input_version
code/model_version
output
status
error
created_at
```

这样 parser 更新后可以 replay。

---

## 18. 新数据到达时的具体流程

```python
def process_new_offer(offer):
    brand = normalize_brand(offer)
    if not brand.verified:
        return enqueue_review("brand")

    evidences = extract_reference_evidences(offer, brand)
    evidences = classify_identifier_roles(evidences)
    evidences = normalize_against_registry(evidences, brand)

    resolved = resolve_reference(evidences)

    if resolved.state == "CONFLICT":
        return enqueue_review("reference_conflict")

    if resolved.state == "VERIFIED":
        entity = lookup_entity(brand.id, resolved.reference)
        if entity:
            attach_offer(entity, offer, reason="verified_reference_exact")
            return MATCHED
        else:
            entity = create_reference_entity(brand.id, resolved.reference)
            attach_offer(entity, offer)
            return MATCHED

    # reference not verified
    ensure_embeddings(offer)
    candidates = retrieve_reference_candidates(
        brand_id=brand.id,
        offer_embedding=offer.embedding,
        top_k=3,
    )

    candidates = apply_negative_evidence_filters(offer, candidates)

    return enqueue_reference_review(offer, candidates)
```

这段流程最重要的性质：

> 没有 VERIFIED reference，不会因为相似度高就自动 merge。

---

## 19. 全量历史数据初始化

对于已有 100 万–1000 万数据，不需要先算全部 pair。

顺序：

### Stage 1：只跑品牌 + reference

```text
全量 parse
-> 得到大量 deterministic matches
```

这部分成本最低，precision 最高。

### Stage 2：构建 Reference Registry

从高可信数据中收集：

```text
brand + reference + aliases + verified examples
```

### Stage 3：只对 unresolved offers 算 embedding

不是所有商品都必须立即计算昂贵的多模态 embedding。

### Stage 4：按 brand 建 ANN

优先热门品牌。

### Stage 5：生成 review backlog

只把：

```text
高价值 + 高可判别候选
```

送人工。

---

## 20. 计算与存储建议

一个务实初版：

```text
PostgreSQL / MySQL
  - offer metadata
  - reference registry
  - evidence
  - entity membership
  - audit

Object Storage
  - raw images
  - thumbnails

Parquet / object storage
  - image embeddings
  - text embeddings
  - feature snapshots

FAISS per brand
  - unresolved offer retrieval
  - reference prototype retrieval

Python workers
  - parsing
  - OCR
  - embedding
  - decision

Simple queue
  - Redis/Rabbit/Kafka 任选现有基础设施
```

如果当前已有 ClickHouse / ES / Milvus，可复用；没有必要为了本需求全部新建。

### 为什么 embedding 建议同时落 Parquet

不要把唯一副本锁在向量数据库里。

保留：

```text
entity_id
model_version
embedding
created_at
```

后续更换索引或向量库时可以直接重建。

---

## 21. 模型版本管理

论文的一大优势是 frozen embedding + small projection 很容易版本化。

建议：

```text
image_encoder_version
text_encoder_version
projection_version
reference_parser_version
role_classifier_version
normalizer_version
registry_version
threshold_config_version
```

每个自动 match 写入：

```json
{
  "decision": "MATCH",
  "reason": "verified_reference_exact",
  "canonical_reference": "126610LN",
  "evidence_ids": [101, 102],
  "versions": {
    "parser": "ref-parser-0.3",
    "normalizer": "rolex-norm-0.2",
    "registry": "2026-08-18"
  }
}
```

否则半年后出现误匹配，无法解释当时为什么合并。

---

## 22. 不要直接做 pairwise O(N²)

10M 级数据如果做三源全笛卡尔完全不可行。

假设三个来源分别百万级，即使只做两个源：

```text
1M x 1M = 10^12 pairs
```

正确的复杂度结构应该接近：

```text
O(N * extraction)
+
O(U * embedding)
+
O(U * log/index_search)
+
O(review_budget)
```

其中 `U` 是 reference unresolved 的子集。

论文的 encoding + retrieval 架构本质上就是为避免 pairwise cross product。

---

## 23. 直接可执行的三阶段上线计划

## Phase 1：Deterministic Reference Engine

目标：最快获得第一批极高 precision 结果。

实施：

```text
1. source schema audit
2. brand dictionary
3. reference-like token extractor
4. role rules
5. 头部品牌 normalizer
6. reference evidence table
7. exact entity key
8. conflict reject
9. manual audit sample
```

自动 MATCH 只允许：

```text
verified brand + verified reference exact
```

### 验收

对人工审计样本：

```text
0 known false positive
```

如果出现一个误匹配，先修规则，不扩大 coverage。

---

## Phase 2：Multimodal Retrieval for Unresolved

目标：提高 reference 缺失商品的可处理率。

实施：

```text
1. image/text embedding cache
2. frozen encoder
3. optional linear projection
4. same-reference contrastive training
5. per-brand ANN
6. Top-3 candidate reference generation
7. retrieval evaluation R@K
```

此阶段仍然不增加 semantic auto-match。

### 验收

```text
Top-3 candidate recall 足够高
review queue size 可控
```

---

## Phase 3：HITL + Feedback Loop

目标：把 unresolved 中的一部分稳定变成 VERIFIED reference。

实施：

```text
1. reference-centric review UI
2. expert reviewers
3. explicit abstain
4. conflict workflow
5. double review for high-risk
6. reviewed aliases -> registry proposal
7. hard negatives -> training set
8. per-source drift dashboard
```

注意：

> 人工确认结果不要直接无限制写入 alias registry，应经过 reviewer/规则校验。

---

## 24. 可以进一步尝试的“高精度自动化”

如果 Phase 1–3 之后仍想提高自动覆盖率，可以增加“多证据联合验证”，但仍不要把模型相似度作为单独依据。

例如：

```text
candidate reference R

Evidence 1: title parser -> R
Evidence 2: OCR warranty card -> R
Evidence 3: registry hit -> R
Evidence 4: multimodal rank #1, large margin to #2
Evidence 5: no negative conflict
```

可以定义：

```text
VERIFIED_REFERENCE
```

但核心依然是 identifier evidence，而非 visual match。

### Top-1 / Top-2 Margin

论文只主要用 distance threshold；当前可额外看：

```text
margin = score(top1) - score(top2)
```

如果：

```text
Top1 = 0.94
Top2 = 0.939
```

说明很可能是同系列相邻 reference，风险高。

即使 Top1 分数很高，也应拒识。

如果：

```text
Top1 = 0.95
Top2 = 0.71
```

可以优先展示给人工，但仍不自动 MATCH。

---

## 25. 与论文 HITL 公式结合做队列质量控制

论文指出最终人工 precision 受输入模型 precision 影响。

当前可以把 review queue 按 bucket 监控：

```text
Bucket A: strong identifier evidence + missing one confirmation
Bucket B: only title candidate
Bucket C: only visual candidate
Bucket D: conflicting identifiers
```

分别测：

```text
review TPR
review FPR
LR+
```

然后决定哪些 bucket 可以进入普通审核，哪些需要专家二审。

不要只统计一个全局“人工准确率”。

---

## 26. 典型误匹配场景与系统行为

## Case 1：同系列不同 reference，图片几乎一样

```text
A: Rolex 126610LN
B: Rolex 126610LV
```

视觉 embedding 很高。

正确行为：

```text
reference mismatch -> NON_MATCH
```

无需看模型分。

---

## Case 2：标题无 reference，但保卡图有

```text
title: 劳力士潜航者 黑盘 全套
OCR(card): 126610LN
```

正确行为：

```text
OCR token
-> role=REFERENCE
-> registry hit
-> VERIFIED
-> exact entity attach
```

图像模型只做辅助。

---

## Case 3：标题出现“适配 126610LN”的表带

```text
title: 适配劳力士 126610LN 橡胶表带
```

抽取到 126610LN。

如果不做 role/context verification，会把配件合并到腕表实体。

正确行为：

```text
context=compatible/accessory
-> token not identity reference of sold item
-> reject
```

---

## Case 4：平台商品编号像 reference

```text
SKU: 126610LN-12345
```

正确行为：

```text
source field mapping -> PLATFORM_SKU
```

不能只根据字符串前缀匹配。

---

## Case 5：field 与 OCR 冲突

```text
field: 126610LN
warranty OCR: 126610LV
```

正确行为：

```text
CONFLICT
-> human review
```

不要：

```text
选择 extractor_score 更高的一边
```

---

## 27. Reference Entity 的版本与合并

如果 reference registry 后续修正，也必须支持可逆。

例如：

```text
alias X 过去错误映射到 R1
后来发现应该是 R2
```

需要能够：

```text
replay affected offers
-> detach old entity
-> re-resolve
-> attach new entity
```

所以不要在原始 offer 上永久覆写、丢失 provenance。

---

## 28. 与 d 已有调研的关系

这篇论文与此前 d 已经分析过的几篇工作有互补关系：

```text
DeepBlocker / pyJedAI
  -> 候选生成、ER pipeline

AMELI
  -> 多模态 entity linking + fine-grained attributes

Reference extraction / normalization 相关工作
  -> reference 身份证据

Confidence / conformal selective prediction
  -> precision-first abstention

TransClean
  -> 多源图上的 false-positive 清理

本篇 fashionID
  -> 生产级多模态 retrieval + domain shift + HITL
```

它最独特的价值是：

> 把“一个还不错的模型”变成“一个真正可以在工业环境中运行的端到端检索 + 人工链路”。

因此可以把已有调研整合成统一架构：

```text
Reference Extraction / Normalization
        |
        v
Reference Registry
        |
        +--------> deterministic exact match
        |
        v
DeepBlocker / fashionID retrieval
        |
        v
AMELI-style fine-grained evidence
        |
        v
Selective / confidence gate
        |
        v
HITL
        |
        v
Transitive consistency audit / TransClean
```

---

## 29. 建议的最终数据库核心表

```text
offer
source_offer_snapshot
brand
brand_alias
reference_registry
reference_alias
reference_evidence
offer_reference_resolution
reference_entity
offer_entity_membership
embedding_feature
candidate_reference
match_decision
review_task
review_judgment
model_version
pipeline_run
```

### offer_reference_resolution

```sql
offer_reference_resolution (
    offer_id               bigint primary key,
    brand_id               bigint,
    state                  varchar, -- VERIFIED/CANDIDATE/CONFLICT/UNKNOWN
    canonical_reference    varchar null,
    reference_id           bigint null,
    confidence_bucket      varchar,
    decision_reason        varchar,
    evidence_ids           jsonb,
    resolver_version       varchar,
    updated_at             timestamp
);
```

### match_decision

```sql
match_decision (
    decision_id            bigint primary key,
    offer_id               bigint,
    entity_id              bigint,
    decision               varchar, -- MATCH/ABSTAIN/CONFLICT/NON_MATCH
    reason                 varchar,
    evidence_snapshot      jsonb,
    model_versions         jsonb,
    rule_version           varchar,
    created_at             timestamp
);
```

---

## 30. 监控面板

建议至少监控：

```text
按 source：
  reference extraction coverage
  VERIFIED rate
  conflict rate
  review rate

按 brand：
  reference registry hit rate
  unresolved rate
  candidate R@3
  auto-match coverage

按 pipeline version：
  sampled precision
  new false positives
  abstain rate

按时间：
  distribution drift
  OCR failure rate
  title format change
```

一旦某 source 突然：

```text
verified rate 70% -> 20%
```

大概率是字段结构或抓取模板变化，需要暂停该 source 的自动决策，而不是继续让模型硬猜。

---

## 31. 生产安全阀

强烈建议增加：

```text
source-level kill switch
brand-level kill switch
parser-version rollback
registry-version rollback
```

例如：

```text
Rolex parser v0.5 出现异常
```

可以只暂停：

```text
brand=Rolex
parser=v0.5
```

而不是关掉整个系统。

---

## 32. 最终推荐

如果现在就要从研究转开发，我建议直接按以下顺序实施：

```text
1. 建 Reference Evidence 数据模型
2. 梳理三源字段与字段可信度
3. 建 brand canonical dictionary
4. 实现 reference-like token 抽取
5. 实现 identifier role classifier/rules
6. 为头部品牌写安全 normalizer
7. 建 reference registry
8. 用 (brand, verified reference) 做 exact entity key
9. 加 conflict veto + abstain
10. 从 VERIFIED offers 构建 reference prototypes
11. 参考 fashionID 预计算 image/text embedding
12. 训练一个小 linear projection 做 unresolved retrieval
13. 每品牌 Top-3 reference ANN
14. 做 reference-centric HITL UI
15. 把人工结果回流 registry + hard-negative set
16. 按 source/time/reference-family 做 precision audit
```

最重要的三条实现红线：

```text
红线 1：多模态相似度不能单独产生自动 MATCH。
红线 2：高可信 reference 冲突不能被任何模型分数抵消。
红线 3：没有足够证据必须允许 ABSTAIN。
```

### 一句话架构决策

> **把 Zalando fashionID 的“冻结多模态 encoder + 低成本 projection + brand blocking + ANN Top-K + HITL”作为 reference 缺失场景的候选检索与人工辅助层；把 `(canonical_brand, VERIFIED canonical_reference)` 作为唯一自动合并主键，并通过 provenance-aware evidence gate、强冲突否决和拒识机制满足“绝对不能误匹配”的业务约束。**

这比直接训练一个“两个商品是不是同款”的大模型更容易达到高 precision，也更适合 100 万–1000 万级持续增量数据的工程落地。