# pyJedAI：端到端 Entity Resolution 架构拆解与二奢腕表 Reference-First 落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **pyJedAI** 做深入分析。

- 项目：<https://github.com/AI-team-UoA/pyJedAI>
- 文档：<https://pyjedai.readthedocs.io/>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

开始前已排除我此前已经分析并输出过的内容：

- `Ameli`
- `DeepBlocker`
- `parts-distributor-sku-classifier`

pyJedAI 的价值在于它不是单一 matcher，而是把 Entity Resolution 拆成了一条完整流水线：

```text
Data
  -> Block Building
  -> Block Cleaning
  -> Comparison Cleaning / Meta-blocking
  -> Entity Matching
  -> Clustering
  -> Evaluation / Workflow
```

这个分层非常适合当前三源二奢腕表场景，但**不能原样把 pyJedAI 当生产系统直接跑 100 万～1000 万持续增量数据**。原因主要有三点：

1. 当前代码的数据模型以 Pandas DataFrame 和 Python dict/set 为核心，适合研究、评测和中等规模离线实验，不是流式分布式状态系统；
2. `Data` 的 Clean-Clean ER 一次主要表达两个数据源，而当前业务有雷小安、腕表之家、奢当家三个来源；
3. 最关键的是，pyJedAI 默认的相似度边和图聚类面向“通用实体解析”，而本需求已经明确规定：**同一个商品 = 同一个 reference number / 型号，且误匹配几乎不可接受。**

因此推荐的生产化结论是：

> **借鉴 pyJedAI 的分层架构与评测能力，但把最终决策层改造成 Reference-First 的确定性实体链接。Blocking、Meta-blocking、Embedding、文本相似度都只负责“找候选/找证据”，不负责最终自动合并。**

更具体地说，生产系统应把问题从：

```text
商品 A <-> 商品 B 是否相同？
```

重构为：

```text
商品记录 -> 解析出品牌 + Canonical Reference
       -> 链接到唯一 Reference Entity
       -> 所有链接到同一 Reference Entity 的记录属于同一商品组
```

最终实体主键建议是：

```text
(brand_id, canonical_reference)
```

而不是文本 embedding、图片 embedding 或图上 connected component。

---

## 1. 需求约束决定了系统必须“Reference-First”

目标 Spec 有几个非常强的约束：

- 三个来源：雷小安、腕表之家、奢当家；
- 数据量约 100 万～1000 万，并持续增量；
- 字段可能稀疏；
- Reference 可能有独立字段，也可能埋在 title/description；
- 有图片；
- 可以人工标几百条；
- “同一个商品”的业务定义就是“同一个 reference / 型号”；
- precision 极端优先，宁可漏掉，也不能误合并。

这意味着一个常规的：

```text
title similarity + brand + image similarity -> 二分类器 -> 阈值
```

即使离线 F1 很高，也不适合作为最终自动合并规则。

腕表恰好存在大量“视觉和标题都很像、Reference 只差 1～2 位”的 hard negative。例如同系列、同尺寸、同材质、同表盘风格的变体，图片和自然语言可能高度相似，但业务上必须是不同实体。

所以应把决策分成两层：

```text
Recall / Candidate Layer
    可以模糊、可以高召回

Decision / Membership Layer
    必须确定、可审计、可拒识
```

pyJedAI 最适合帮助我们搭建第一层和实验框架；第二层需要按业务语义额外加硬约束。

---

## 2. pyJedAI 当前代码架构拆解

以下分析基于当前 `main` 分支，重点阅读：

- `src/pyjedai/datamodel.py`
- `src/pyjedai/block_building.py`
- `src/pyjedai/block_cleaning.py`
- `src/pyjedai/comparison_cleaning.py`
- `src/pyjedai/matching.py`
- `src/pyjedai/vector_based_blocking.py`
- `src/pyjedai/clustering.py`
- `src/pyjedai/workflow.py`

### 2.1 `Data`：统一的双源 ER 数据模型

`datamodel.py` 中的 `Data` 接收：

- `dataset_1`
- 可选 `dataset_2`
- 两侧 ID 列
- 两侧参与匹配的 attribute 列
- 可选 ground truth

它会：

1. 将 NaN 填成空字符串；
2. 将字段统一转为字符串；
3. 为两侧记录构造内部连续整数 ID；
4. 建立外部 ID 与内部 ID 的正反映射；
5. 如果有 ground truth，则把真实匹配 pair 存起来用于每一步评测。

当 `dataset_2 is None` 时是 Dirty ER；存在 `dataset_2` 时是 Clean-Clean ER。

这个设计有一个很值得复用的点：

> **每一个流水线阶段都共享同一套数据与 ground truth，Blocking、Cleaning、Matching 都能独立评测。**

对当前项目应保留这种“每阶段可评测”的思想，但生产数据模型要扩成多源、增量和证据可追踪结构。

同时它也暴露了 pyJedAI 不适合整库在线化的原因：两侧 DataFrame、内部映射、实体列表都驻留在进程内；1000 万级加大量文本字段后，Pandas + Python 对象开销会明显放大。

### 2.2 Block Building：先把笛卡尔积压成候选集

`block_building.py` 提供多种传统 Blocking：

- `StandardBlocking`
- `QGramsBlocking`
- `SuffixArraysBlocking`
- `ExtendedSuffixArraysBlocking`
- 其他扩展 Blocking

#### StandardBlocking

当前实现大致是：

```text
选定属性
 -> 每行属性用空格拼接
 -> lowercase
 -> 按非 word 字符 / 下划线分 token
 -> 每个 token 创建一个 block
 -> 共享 token 的记录进入同一 block
```

并且代码支持 Ray 批量 tokenization，默认 batch size 为 10000。

这对通用 ER 很自然，但当前腕表场景不能直接把全字段都丢进去。

例如：

```text
ROLEX Submariner 126610LN 黑水鬼 2022 全套
ROLEX Submariner 126610LV 绿水鬼 2022 全套
```

两条记录会共享大量 token。作为候选召回这是好事；但它们是典型的 hard negative，最终不能靠 token 相似度决定匹配。

更大的风险是：

- 品牌词会生成超大 block；
- “自动”“机械”“男表”“全套”等业务常见词会生成超大 block；
- seller SKU、平台内部 ID、配件兼容型号可能被混到一起；
- 如果 Reference 被拆成多个 token，组合语义会丢失。

因此生产系统不能把 `StandardBlocking` 等价成最终 Reference Index。

#### QGrams / Suffix Blocking

`QGramsBlocking` 会从 token 里继续生成 q-gram（默认 q=6），Suffix 类方法会产生后缀或子串 block。

这对于以下疑难情况很有价值：

- 标题中 Reference 少一个分隔符；
- OCR 漏了一位；
- 某个平台写成不同空格/连字符格式；
- Reference 候选只截到一部分。

例如：

```text
126610LN
126610LV
```

q-gram 会让它们共享大量候选特征，从而互相进入候选集。

但这也说明它只能做 Recall Layer：

> **越擅长找“相似型号”，越不能直接把相似当相同。**

### 2.3 Block Cleaning：抑制超大、低信息 block

`block_cleaning.py` 主要有：

#### `BlockPurging`

根据 block 的 comparison cardinality 自动找阈值，把比较量过大的 block 删掉。

业务上可以理解为：

```text
“ROLEX” 这种几乎人人都有的 block 信息量太低
=> 不值得让其中所有记录两两比较
```

#### `BlockFiltering`

对每个实体保留其一部分较小 block，默认 ratio=0.8。

这和 IR 中“稀有 token 信息量更大”直觉一致。

但在当前需求里需要做一个关键分流：

> **Verified Reference 形成的 exact block 绝不能被 BlockPurging / BlockFiltering 当普通大 block 丢掉。**

因为某个热门 reference 可能有几千条甚至更多历史商品记录。它虽然是大 block，但语义上却是最强证据。

所以生产系统应该有两种 block：

```text
Hard Block: (brand_id, canonical_reference)
  -> 永远保留，直接进入确定性实体

Soft Block: title token / qgram / series / embedding neighbor
  -> 可以 purging/filtering，仅用于疑难候选
```

### 2.4 Comparison Cleaning / Meta-blocking：把候选图再瘦身

`comparison_cleaning.py` 的核心思想是：Blocking 后很多 pair 会通过多个 block 重复出现，甚至产生大量弱关联 pair，所以继续对候选边做图式压缩。

当前代码支持多种 weighting scheme，例如：

- CBS
- Cosine
- Dice
- Jaccard / JS
- ECBS
- EJS
- Chi-square (`X2`)
- 多种 cardinality / size normalization 变体

它们大体都在利用：

- 两个实体共享了多少 block；
- block 自身有多稀有；
- 两边各自参与多少比较；

来估计一条候选边有多重要。

另外 `ComparisonPropagation` 会把重叠 block 中重复 comparison 去重。

这层对 1000 万级系统很有价值，因为真正昂贵的不是 hash blocking，而是后续要跑多少 candidate pair。

但仍然需要坚持：

> Meta-blocking 的高分只表示“值得进一步看”，不表示“可以自动认为相同 Reference”。

### 2.5 Entity Matching：相似度图，而不是业务真值图

`matching.py` 中 `EntityMatching` 支持：

- edit distance
- Jaro
- cosine
- Jaccard
- generalized Jaccard
- Dice
- overlap coefficient
- TF/TF-IDF 等向量方式

它可以：

- 对某些字段做匹配；
- 给字段设置权重；
- 设置 similarity threshold；
- 高于阈值的 pair 会写成 NetworkX graph edge，edge 上保存 similarity weight。

这非常适合做实验，例如：

```text
brand      0.2
series     0.2
reference  0.5
color      0.1
```

但对本需求，**Reference 不能只是 0.5 权重的一个 feature**。

应该是：

```text
reference conflict => hard reject
reference unavailable => abstain / fallback
reference exact + evidence valid => allow
```

而不是：

```text
reference 不同，但其他字段非常像
=> 总分仍然 > 0.9
=> 自动匹配
```

因此需要在 pyJedAI matcher 外再包一层业务决策门。

### 2.6 `EmbeddingsNNBlockBuilding`：Sentence Transformer + FAISS Top-K

`vector_based_blocking.py` 是 pyJedAI 里对当前场景较有价值的一部分。

它支持多类预训练 embedding，包括：

- BERT / DistilBERT / RoBERTa / XLNet / ALBERT 类 word embedding；
- SentenceTransformer 系列句向量；
- 部分 Gensim word vectors；
- 自定义预训练模型。

随后用 FAISS 做近邻检索，参数包括：

- `top_k`，默认 30；
- cosine / euclidean 距离；
- 可保存 embedding；
- 可只对前面 Cleaning 后保留下来的 subset 做 embedding / search。

这里的架构思想非常好：

```text
Cheap blocking
 -> candidate subset
 -> embedding only for retained subset
 -> FAISS top-K
```

而不是所有数据无脑跑最昂贵模型。

不过当前实现使用 `IndexFlatL2`（cosine 时改 inner product + L2 normalization）进行 flat search，本质上仍是 exact nearest-neighbor search；如果把千万级商品全部互相检索，成本和内存仍然不合适。

生产环境更合理的是：

1. 先按 `brand` / `category` / reference pattern 分区；
2. 只对 **Reference 缺失或冲突的小尾部**做向量候选；
3. 候选库优先是数量更小的 Reference Catalog / 已确认 Reference Entity，而不是全量商品互搜；
4. 如果 fallback 索引仍很大，再换成可增量的 ANN 索引，而不是 Flat 全检索。

### 2.7 Clustering：当前业务最需要“克制使用”的模块

`workflow.py` 会引用 `ConnectedComponentsClustering`、`UniqueMappingClustering` 等聚类方法，`clustering.py` 也建立在 NetworkX 图和加权边上。

对一般 ER，这是合理的：

```text
A ~ B
B ~ C
=> 可能把 A/B/C 组成一个实体 cluster
```

但对 precision-first 的腕表 Reference 匹配，这是风险最大的部分之一。

如果：

```text
A(reference=126610LN) --错误边--> B(reference=126610LV)
B --相似边--> C(reference=126610LV)
```

Connected Component 可能把错误边的影响传递到整个 cluster。

这就是典型的 **single false edge contaminates whole component**。

而我们的业务已经有更强的实体定义，因此不应使用相似图自动聚类：

```text
cluster_id = Reference Entity ID
Reference Entity ID 唯一对应 (brand_id, canonical_reference)
```

相似图应该降级为：

- 候选图；
- 冲突审计图；
- 人工复核辅助图；

而不是 membership truth。

### 2.8 Workflow：pyJedAI 最值得直接复用的部分是“实验编排”

`workflow.py` 把：

- Blocking
- Block Purging
- Block Filtering
- Comparison Cleaning
- Matching
- Clustering

组织成可配置 workflow，并记录每一步：

- F1
- Recall
- Precision
- Runtime
- 参数配置

这非常适合当前项目做**离线算法选型实验**。

建议把 pyJedAI 定位成：

> “Reference Entity Resolution 的实验台 / benchmark harness”，而不是最终生产 serving engine。

---

## 3. 生产架构：Reference-First + pyJedAI-inspired Fallback

推荐最终架构如下：

```text
               ┌──────────────────────┐
雷小安 -------->|                      |
腕表之家 ------>|  Raw Ingestion      |
奢当家 -------->|  + Versioned Record |
               └──────────┬───────────┘
                          |
                          v
               ┌──────────────────────┐
               | Brand Normalization  |
               | Category / Role Gate |
               └──────────┬───────────┘
                          |
                          v
               ┌─────────────────────────────┐
               | Reference Evidence Extractor|
               | - dedicated field           |
               | - title/description         |
               | - image OCR                 |
               | - catalog validation        |
               └──────────┬──────────────────┘
                          |
                 ┌────────┴─────────┐
                 |                  |
                 v                  v
        Strong / Unique        Missing/Ambiguous/Conflict
                 |                  |
                 v                  v
      ┌──────────────────┐  ┌───────────────────────────┐
      | Exact Ref Router |  | Soft Candidate Retrieval  |
      | brand + ref      |  | block/qgram/vector Top-K  |
      └────────┬─────────┘  └────────────┬──────────────┘
               |                         |
               v                         v
      ┌──────────────────┐     ┌──────────────────────┐
      | Reference Entity |     | Strict Verifier /    |
      | deterministic    |     | Human Review         |
      | membership       |     └──────────┬───────────┘
      └────────┬─────────┘                |
               |                          | 只有升级出强 Reference 证据
               |<─────────────────────────┘ 才允许进入
               v
      ┌───────────────────────┐
      | Audit + Metrics +     |
      | Incremental Reconcile |
      └───────────────────────┘
```

这里最重要的规则是：

> **Soft Candidate Retrieval 永远不能因为相似度高就直接写 membership。它只能帮助找到新的 Reference 证据，或进入人工复核。**

---

## 4. Reference Evidence 模型：不要只存一个 `reference` 字符串

要满足“绝不能错匹配”，应把 Reference 当成带 provenance 的证据集合。

建议一条商品记录解析出：

```json
{
  "brand_id": "ROLEX",
  "reference_candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "source_type": "dedicated_field",
      "field": "reference_no",
      "role": "product_reference",
      "confidence": 1.0,
      "extractor_version": "ref-v3"
    },
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "source_type": "title",
      "span": "劳力士潜航者126610LN",
      "role": "product_reference",
      "confidence": 0.99,
      "extractor_version": "ref-v3"
    }
  ],
  "reference_resolution": {
    "status": "VERIFIED",
    "canonical": "126610LN"
  }
}
```

至少保留这些属性：

- raw value；
- canonical value；
- 从哪里抽到；
- 原文 span；
- 编号角色；
- 置信度；
- validator 结果；
- extractor / normalization 版本。

这样后续规则升级时可以重放，而不是只剩一个不可解释字符串。

---

## 5. Reference 提取与规范化

### 5.1 品牌先规范化

Reference 的命名空间必须带品牌。

不能只用：

```text
canonical_reference
```

应该用：

```text
(brand_id, canonical_reference)
```

原因：不同品牌可能存在形式相近甚至完全相同的编号字符串。

品牌规范化先处理：

- 中英文别名；
- 大小写；
- 常见拼写；
- 平台自己的品牌 ID；

但品牌置信度不足时，同样应拒识而不是跨品牌模糊匹配。

### 5.2 Reference 候选来源按可靠性分级

建议证据优先级：

```text
Tier A
- 平台独立 reference/model 字段
- 官方/可信 catalog 明确命中

Tier B
- title/description 中唯一、符合该品牌格式的候选
- 多个文本位置彼此一致

Tier C
- OCR 单独识别
- 模糊字符串恢复
- 仅靠 embedding/LLM 推断
```

自动 membership 推荐至少满足：

- 两边都有 A/B 级强证据；或
- 一边 A，另一边 B 且无任何冲突；

C 级证据只做候选和复核，不独立放行。

### 5.3 规范化必须“品牌感知”，不能全局暴力清洗

安全的通用步骤：

```text
Unicode NFKC
trim
uppercase
统一全角/半角
规范可确认等价的 separator
```

但不要全局做：

```text
O -> 0
I -> 1
S -> 5
```

也不要默认把所有标点删除后就认为相同。

因为 identifier 中一个字符通常就是型号边界。

更安全的方案：

```text
raw_reference
  -> brand-specific parser
  -> canonical_reference
  -> format validator
  -> optional catalog validator
```

如果 OCR 得到 `12661OLN`，不能自动把 `O` 改 `0` 后直接合并；最多在同品牌 catalog 内生成候选，并要求其他证据确认。

### 5.4 先判断“编号角色”，再判断值

Reference 出现在标题里，并不一定表示当前商品本体 Reference。

常见危险场景：

- “适配 126610LN 的表带”；
- “126610LN 原装盒”；
- “兼容 xxx 型号”；
- 平台货号、店铺 SKU 长得像型号；
- 保卡/说明书关联型号出现在配件商品里。

因此提取器输出必须包含：

```text
role = product_reference
     | compatible_reference
     | accessory_target_reference
     | seller_sku
     | platform_id
     | serial_number
     | unknown
```

只有 `product_reference` 才能进入自动实体链接。

---

## 6. 最终决策门：Exact Reference 是硬条件，不是高权重 feature

推荐把自动决策写成可审计规则，而不是一个不可解释阈值。

伪代码：

```python
def resolve_membership(item):
    if item.brand.status != "VERIFIED":
        return ABSTAIN("brand_unverified")

    if item.category.is_accessory:
        return ABSTAIN("accessory_or_non_watch")

    refs = strong_product_reference_evidence(item)

    if has_strong_conflict(refs):
        return ABSTAIN("reference_conflict")

    ref = unique_canonical_reference(refs)
    if ref is None:
        return ABSTAIN("reference_missing_or_ambiguous")

    return UPSERT_REFERENCE_ENTITY(
        brand_id=item.brand.id,
        canonical_reference=ref,
        evidence_ids=refs.ids,
    )
```

跨源是否同商品不需要再跑 pairwise classifier：

```python
def same_product(a, b):
    return (
        a.reference_entity_id is not None
        and a.reference_entity_id == b.reference_entity_id
    )
```

如果一定要做 pair 级 verifier，则必须至少：

```python
def strict_pair_gate(a, b):
    if a.brand_id != b.brand_id:
        return REJECT
    if a.has_reference_conflict or b.has_reference_conflict:
        return ABSTAIN
    if not a.strong_ref or not b.strong_ref:
        return ABSTAIN
    if a.canonical_ref != b.canonical_ref:
        return REJECT
    return ACCEPT
```

这里不存在“0.93 相似度所以 accept”的逻辑。

---

## 7. Clustering 的生产替代：确定性 Reference Entity

建议数据库里显式建立：

```text
reference_entity
---------------
id
brand_id
canonical_reference
reference_display
normalization_version
status
created_at
updated_at
```

并加唯一约束：

```sql
UNIQUE (brand_id, canonical_reference)
```

每条 source item 最终只能有一个 active membership：

```text
entity_membership
-----------------
source_item_id
reference_entity_id
decision_type
resolver_version
evidence_snapshot
created_at
```

必须满足 cluster invariant：

1. 同一 cluster 内所有自动确认成员都有同一个 `(brand_id, canonical_reference)`；
2. 任何 strong reference 冲突都阻止 membership；
3. 不允许因为 A-B、B-C 两条相似边而传递式合并；
4. fuzzy/embedding edge 永远不能直接修改 cluster membership。

图可以保留，但作为：

```text
candidate_edge / evidence_edge
```

而不是 `entity_membership`。

这一步是对 pyJedAI clustering 最重要的业务改造。

---

## 8. pyJedAI 在本系统里的正确位置

推荐将流程拆为 `fast path` 与 `fallback path`。

### 8.1 Fast Path：绝大多数可解析 Reference 的记录

```text
raw item
 -> brand normalize
 -> reference extraction
 -> canonicalize
 -> conflict check
 -> exact key upsert
 -> Reference Entity
```

复杂度接近 O(N)，本质上是解析 + hash/index lookup，不做 N×M 比较。

这应该覆盖尽可能多的正常数据。

### 8.2 Fallback Path：只有 Reference 缺失/歧义的小尾部

这时才借鉴 pyJedAI：

```text
Soft Blocking
 -> Block Purging / Filtering
 -> Comparison Cleaning
 -> Optional Vector Top-K
 -> Strict verifier
 -> Human review / abstain
```

其中 candidate 特征建议只用：

- canonical brand；
- series/family；
- title 中高信息 token；
- reference fragment / qgram；
- 结构化尺寸、材质、机芯等；
- OCR 候选；
- 图片 embedding 只作为辅助。

### 8.3 三源如何使用 pyJedAI 的双源 Data

离线 benchmark 可跑三组 Clean-Clean ER：

```text
雷小安 <-> 腕表之家
雷小安 <-> 奢当家
腕表之家 <-> 奢当家
```

分别测试：

- candidate recall；
- comparisons 数；
- runtime；
- 最终 hard-gate precision。

但生产系统不需要把三组 pairwise 结果再做 connected component；最终仍然写到统一 `Reference Entity`。

---

## 9. 对 pyJedAI 各模块的建议改造

| pyJedAI 模块 | 原用途 | 当前业务建议 |
|---|---|---|
| `Data` | 双源数据 + GT | 仅离线实验；生产改为多源版本化 item/evidence 表 |
| `StandardBlocking` | token block | 仅 soft candidate，不用于 hard reference membership |
| `QGramsBlocking` | 字符近似候选 | Reference 模糊/缺失时找候选，绝不自动放行 |
| `BlockPurging` | 去超大 block | 只作用于 soft blocks；exact reference block 绕过 |
| `BlockFiltering` | 每实体保留较小 blocks | fallback 可用，需测 candidate recall |
| `ComparisonPropagation` | 去重复 comparisons | 可直接借鉴 |
| Meta-blocking | 边权剪枝 | candidate ranking/pruning，不作为 truth |
| `EntityMatching` | 字符/集合相似度 | 作为 review score / 辅助 verifier |
| `EmbeddingsNNBlockBuilding` | embedding + FAISS top-K | 只给 unresolved tail 或 Reference Catalog 使用 |
| `ConnectedComponentsClustering` | 图聚类 | 不用于最终 membership |
| `UniqueMappingClustering` | 映射约束聚类 | 不建议直接套；同 Reference 可有多条 listing |
| `Workflow` / `Evaluation` | 端到端实验评估 | 很适合复用为 benchmark harness |

---

## 10. 为什么不要直接用 StandardBlocking 构造 `(brand, reference)` 硬块

这里有一个代码级细节值得特别指出。

`StandardBlocking` 会把多属性拼起来后，再按 `[^word]` 和 `_` 一类字符切 token。

如果预先生成：

```text
ROLEX_126610LN
```

它很可能仍会被拆成：

```text
ROLEX
126610LN
```

这不再是复合 key。

因此生产中的 hard block 不应该“曲线救国”塞进 StandardBlocking，而应该直接：

```text
hard_key = (brand_id, canonical_reference)
```

使用数据库唯一索引、hash table 或 distributed key-value join。

pyJedAI 的 token blocking 留给 soft path 即可。

---

## 11. 向量检索的正确对象：优先 Reference Catalog，不是全量商品

如果我们已经整理出一个 Canonical Reference Catalog：

```text
Reference Entity
- brand
- ref
- series
- aliases
- title exemplars
- images / OCR exemplars
```

那么 unresolved 商品更适合：

```text
商品 -> Top-K Reference Entity
```

而不是：

```text
商品 -> Top-K 另外 1000 万商品
```

好处：

1. 索引规模显著变小；
2. 候选天然带 canonical ID；
3. 人工复核是在“选哪个 Reference”而不是“这两个商品是不是同一个”；
4. 新来源接入时不需要重新构造跨来源全量 pair；
5. 一条 Reference Entity 可以聚合多个可信样本，增强检索表现。

这也是把 pyJedAI 的 Vector Blocking 从 pairwise ER 改造成 Entity Linking 的关键一步。

---

## 12. 图片的角色：补证据和否决，不越权替代 Reference

当前数据有图片，可以用，但应该限制权力。

推荐两类用途：

### 12.1 OCR 抽 Reference

重点看：

- 表背；
- 保卡；
- 吊牌；
- 盒证；
- 可见刻字。

OCR 输出仍然是 `reference_evidence`，必须保留：

- 图片 ID；
- bbox；
- raw OCR；
- normalization；
- confidence。

如果 OCR 与独立 reference 字段冲突：

```text
=> quarantine / review
```

不能简单按“谁分数高听谁的”。

### 12.2 图片相似度做冲突信号

图片 embedding 可以：

- 给无 Reference 商品找相似 Reference 候选；
- 发现文本说一个型号、图片明显像另一个系列的异常；
- 帮助人工复核。

但禁止：

```text
图片很像 + 文本很像 + reference 不同
=> 自动合并
```

Reference conflict 永远优先。

---

## 13. 生产数据模型建议

### 13.1 `source_item`

```text
id
source                 # leixiaoan / xbiao / shedangjia
source_item_id
source_version
brand_raw
brand_id
category_raw
category_normalized
title
description
raw_payload_uri
first_seen_at
last_seen_at
is_active
```

唯一约束：

```text
UNIQUE(source, source_item_id, source_version)
```

### 13.2 `reference_evidence`

```text
id
source_item_pk
raw_value
canonical_value
evidence_type          # field/title/description/ocr/catalog/model
role                   # product_ref/compatible_ref/platform_sku/...
field_or_image_id
span_or_bbox
confidence
validator_status
extractor_version
normalizer_version
created_at
```

### 13.3 `reference_entity`

```text
id
brand_id
canonical_reference
display_reference
status
created_at
updated_at
```

核心唯一约束：

```text
UNIQUE(brand_id, canonical_reference)
```

### 13.4 `entity_membership`

```text
source_item_pk
reference_entity_id
decision                # auto_exact / human_verified / migrated
decision_reason
evidence_ids
resolver_version
valid_from
valid_to
```

### 13.5 `candidate_edge`

只用于 soft path：

```text
left_item_id
right_item_or_reference_id
retrieval_method
blocking_keys
text_score
vector_score
image_score
reference_conflict
final_status             # rejected/reviewed/abstained/upgraded
created_at
```

这样可以完全复盘为什么某条记录没有自动匹配。

---

## 14. 百万～千万级扩展策略

### 14.1 主链路不要做 O(N²)

Fast Path 本质是：

```text
extract -> normalize -> indexed upsert
```

每条新记录只查自己的 Reference Key。

历史全量也可按 `(brand_id, canonical_reference)` 做 hash/group-by，不需要跨表笛卡尔积。

### 14.2 分区

推荐至少按：

```text
brand_id
```

分区。

如果单品牌仍太大，可再按：

```text
reference prefix / family / hash bucket
```

分片。

所有 soft candidate retrieval 都先限定品牌，避免跨品牌误召回和无效计算。

### 14.3 增量事件

每个新/变更 item：

```text
UPSERT source_item
 -> re-run reference evidence extraction
 -> compare old resolved ref vs new resolved ref
 -> unchanged: no-op
 -> changed: close old membership + open new membership + audit event
 -> conflict: quarantine
```

不要直接覆盖旧 membership，否则无法追踪平台数据修正导致的实体迁移。

### 14.4 生产运行时不建议整库直接使用 pyJedAI DataFrame

pyJedAI 可继续用于：

- 品牌分片样本；
- 候选算法比较；
- hard-negative benchmark；
- 参数实验；
- regression test。

生产主链路建议采用：

- 列式批处理 / SQL 做全量 Reference 解析结果聚合；
- 带唯一约束的关系型存储维护 Reference Entity 与 membership；
- soft path 用独立候选索引；
- 原始文本和图片放对象存储；
- 流式要求不高时也可以先用周期 micro-batch，不必一开始过度建设复杂流系统。

---

## 15. 几百条人工标注应该怎么花

只有几百条预算时，不要随机采样普通 pair。

随机负样本太容易，会高估系统可靠性。

应该主要构造 hard negatives：

1. 同品牌 + 同系列 + Reference edit distance 1；
2. 同品牌 + 同前缀 + 末位不同；
3. LN/LV、材质后缀、盘面后缀等相邻变体；
4. 标题里出现“适配某型号”的配件；
5. seller SKU / platform ID 长得像 Reference；
6. OCR 的 O/0、I/1、S/5 混淆；
7. 独立 Reference 字段与 title 冲突；
8. 图片和文本强相似、Reference 不同；
9. Reference 相同但标题很不同的 true positive；
10. 新品牌/新 Reference 的 unseen case。

建议标注对象从“pair 是否 match”扩展为：

```text
- brand 是否正确
- reference candidate span
- reference role
- canonical reference
- 是否存在 strong conflict
- 最终是否应自动 membership
```

同一批人工信息能同时训练/评估抽取器和最终决策器。

---

## 16. 评测指标：不要只看 F1

pyJedAI 默认可以逐阶段报告 Precision / Recall / F1，很适合实验；但生产验收要换成更贴合风险的指标。

### 16.1 Fast Path

重点看：

```text
Reference Extraction Precision
Reference Resolution Precision
Auto-Match Precision
Auto-Match Coverage
Conflict Detection Recall
Abstention Rate
```

其中最重要的是：

```text
Auto-Match Precision
```

### 16.2 Soft Candidate Path

重点看：

```text
Candidate Recall@K
Candidates per Item
Comparison Reduction Ratio
Runtime / cost
Review Hit Rate
```

这条链路允许 recall 优先，因为它还没有写最终 membership。

### 16.3 一个很重要的统计现实

几百条标注即使 0 个 false positive，也不足以“统计证明”极低误匹配率。

粗略使用零失败的 rule-of-three：

```text
95% 上界约 ≈ 3 / n
```

如果验收集只有 500 条自动匹配样本且 0 错误，上界仍大约是：

```text
3/500 = 0.6%
```

这离“千万级数据上几乎不能错”差得非常远。

因此不能把安全性寄托在“模型离线测了 100% precision”。真正的安全来源应该是：

- Reference exact invariant；
- 强证据来源；
- conflict hard reject；
- fuzzy path abstain；
- 全链路可审计。

---

## 17. 推荐的离线 pyJedAI Benchmark

可以用 pyJedAI 快速搭一个三组 pairwise benchmark，但只把它当实验工具。

### 17.1 数据准备

为每个平台构造标准化字段：

```text
item_id
brand_id
series
reference_raw
reference_candidate
reference_fragment
title_clean
attributes_clean
```

同时为人工标注 pair 准备 ground truth。

### 17.2 Baseline A：Standard Blocking

只选择受控字段，例如：

```text
brand_id
series
reference_fragment
```

测：

- blocking recall；
- comparison count。

不要把长 description、seller SKU 全部混进去。

### 17.3 Baseline B：QGram Blocking

对 unresolved reference fragment 做 qgram blocking，测试是否能召回：

- 少分隔符；
- 少一字符；
- OCR 轻微错误；

同时重点观察 hard negative 候选量。

### 17.4 Baseline C：Block Cleaning + Meta-blocking

在 B 上继续加入：

```text
BlockPurging
BlockFiltering
ComparisonPropagation / Node/Edge pruning
```

目标是：

> 在 Candidate Recall 几乎不掉的前提下，大幅减少 candidate pair 数量。

### 17.5 Baseline D：Embedding Top-K

只对 unresolved subset：

```text
EmbeddingsNNBlockBuilding(top_k=20~50)
```

并限制在同品牌候选库。

重点看 `Recall@K`，不看 embedding score 是否足以直接 accept。

### 17.6 最后统一跑 Strict Gate

所有 Baseline 的候选最终统一经过：

```text
brand exact
+ resolved reference exact
+ no strong conflict
+ product_reference role
```

只比较：

```text
最终自动匹配覆盖率
在 0 FP 约束下谁覆盖最多
```

这比比较最高 F1 更符合需求。

---

## 18. 一个可直接落地的 PoC 代码结构

建议代码按以下模块拆：

```text
reference_er/
├── ingestion/
│   ├── leixiaoan.py
│   ├── xbiao.py
│   └── shedangjia.py
├── normalization/
│   ├── brand.py
│   ├── reference.py
│   └── category.py
├── extraction/
│   ├── field_extractor.py
│   ├── title_extractor.py
│   ├── ocr_extractor.py
│   └── role_classifier.py
├── resolution/
│   ├── evidence.py
│   ├── conflict.py
│   ├── strict_gate.py
│   └── reference_entity.py
├── candidates/
│   ├── blocking.py
│   ├── qgram.py
│   ├── vector_index.py
│   └── pyjedai_benchmark.py
├── review/
│   ├── queue.py
│   └── audit.py
├── evaluation/
│   ├── gold.py
│   ├── hard_negatives.py
│   └── metrics.py
└── jobs/
    ├── backfill.py
    └── incremental.py
```

这比把全部逻辑塞进一个 pairwise matching notebook 更容易维护。

---

## 19. pyJedAI 实验代码示意

下面是概念级实验方式，目的是验证 Blocking/Comparison Cleaning，而不是直接生产自动合并：

```python
from pyjedai.datamodel import Data
from pyjedai.block_building import QGramsBlocking
from pyjedai.block_cleaning import BlockPurging, BlockFiltering
from pyjedai.comparison_cleaning import ComparisonPropagation

# d1 / d2 已预先完成 brand/reference 相关标准化
# gt 只用于离线 benchmark

data = Data(
    dataset_1=d1,
    id_column_name_1="item_id",
    attributes_1=["brand_id", "series", "reference_fragment", "title_clean"],
    dataset_name_1="leixiaoan",
    dataset_2=d2,
    id_column_name_2="item_id",
    attributes_2=["brand_id", "series", "reference_fragment", "title_clean"],
    dataset_name_2="xbiao",
    ground_truth=gt,
)

bb = QGramsBlocking(qgrams=4)
blocks = bb.build_blocks(
    data,
    attributes_1=["reference_fragment", "series"],
    attributes_2=["reference_fragment", "series"],
)

bp = BlockPurging()
blocks = bp.process(blocks, data)

bf = BlockFiltering(ratio=0.8)
blocks = bf.process(blocks, data)

cp = ComparisonPropagation()
candidates = cp.process(blocks, data)

# 到这里为止只得到候选 pair。
# 不允许直接根据 pyJedAI 相似度写入 entity_membership。
```

然后生产/实验共同调用同一个 strict gate：

```python
for left_id, right_ids in candidates.items():
    for right_id in right_ids:
        decision = strict_reference_gate(
            resolved[left_id],
            resolved[right_id],
        )
        if decision == "ACCEPT":
            link_to_reference_entity(left_id, right_id)
        else:
            record_candidate_decision(left_id, right_id, decision)
```

真正生产时甚至更推荐直接把 `link_to_reference_entity` 改成 item -> reference entity，而不是先 pairwise。

---

## 20. 增量处理与幂等性

对于持续增量，关键不是“每天重跑全量 pyJedAI”，而是保持解析结果幂等。

建议每条 item 生成：

```text
record_fingerprint
extractor_version
normalizer_version
resolver_version
```

如果原始关键字段和图片没变，且处理版本没变：

```text
=> skip
```

如果版本升级：

```text
=> 只重算受影响 brand / rule / item
```

Reference Entity 的稳定 ID 不应因为 parser 版本升级随意改变。

如果规范化规则发现历史 canonical key 有误，应通过：

```text
alias / merge / migration event
```

显式迁移，而不是静默改 key。

---

## 21. 失败模式与对应防线

| 失败模式 | 如果直接用通用 ER 会怎样 | 推荐防线 |
|---|---|---|
| 同系列相邻 Reference | 文本/图像高相似，容易 FP | ref exact hard gate |
| 标题有兼容型号 | 被误当商品型号 | reference role classifier |
| seller SKU 像型号 | 错跨源关联 | 编号角色 + source-aware 字段规则 |
| OCR O/0 | 模糊纠错后可能错合 | OCR 只弱证据，需文本/字段确认 |
| 品牌别名不统一 | 漏召回或跨品牌冲突 | canonical brand namespace |
| 热门品牌 block 超大 | 候选爆炸 | soft BlockPurging；hard ref block 绕过 |
| 一条错误相似边 | Connected Component 污染全簇 | 不用相似图做 membership |
| 新 Reference 未见过 | 模型不认识 | parser + 格式验证，不依赖 closed-set 分类 |
| 分布漂移 | 阈值失效 | resolver version + abstain + hard-negative regression |
| 来源字段修正 | 旧实体静默错误 | versioned membership + reconcile event |

---

## 22. 推荐实施顺序

### Milestone A：先把确定性主链路跑通

完成：

- 品牌规范化；
- Reference 独立字段解析；
- title 中 Reference 抽取；
- 品牌级规范化规则；
- `(brand_id, canonical_reference)` 唯一实体；
- conflict quarantine；
- 全链路 evidence/audit。

先测能覆盖多少数据。

### Milestone B：建立 hard-negative Benchmark

集中标：

- 相邻 reference；
- 配件；
- 内部 SKU；
- OCR 混淆；
- 字段冲突。

验收 strict gate。

### Milestone C：只给 unresolved tail 加 pyJedAI-inspired Candidate Retrieval

依次比较：

1. qgram/reference fragment；
2. block cleaning；
3. comparison cleaning；
4. embedding top-K；
5. 图片辅助。

每一步只看：

```text
在不增加 FP 的情况下，能减少多少人工 / 提高多少可确认覆盖率
```

### Milestone D：再做持续增量和规模优化

- brand partition；
- micro-batch / queue；
- ANN 索引增量；
- reprocessing version；
- dashboard / conflict audit。

---

## 23. 最终建议

pyJedAI 非常值得参考，但应该把它的价值拆开看。

### 值得直接借鉴

1. **流水线分层**：Blocking -> Cleaning -> Matching -> Clustering；
2. **候选削减**：Block Purging、Filtering、Meta-blocking；
3. **多种 Blocking 快速对比**；
4. **Embedding + FAISS Top-K** 的 fallback 思路；
5. **每阶段独立评测**；
6. **Workflow 参数化实验**。

### 不应直接照搬

1. 把所有属性拼字符串后直接做通用相似；
2. 把 Reference 当普通 weighted feature；
3. 只用 similarity threshold 决定“同商品”；
4. 用 Connected Components 把概率边传递聚类；
5. 用 Pandas/Python dict 直接承载千万级长期在线状态；
6. 全量商品互做向量 Top-K。

### 当前需求的推荐最终形式

```text
                         ┌───────────────────────────────┐
                         |      Reference Resolver       |
                         | brand + ref + evidence + role |
                         └──────────────┬────────────────┘
                                        |
                         strong exact   |   weak/unknown
                              ┌─────────┴─────────┐
                              v                   v
                    Reference Entity        Candidate Layer
                    deterministic           pyJedAI-inspired
                         membership          blocking/vector
                              |                   |
                              |            找更多证据/人工复核
                              |                   |
                              └─────────<─────────┘
```

一句话总结：

> **用 pyJedAI 解决“千万级数据里该比较谁”，用 Reference Evidence System 解决“谁真的可以被合并”。最终实体不是相似度图里的 cluster，而是有唯一 `(brand, canonical_reference)` 约束、可追踪证据且允许拒识的 Reference Entity。**

这套结构既利用了 pyJedAI 在大规模 ER 候选压缩上的成熟设计，又避免了通用 entity matching 最危险的 false-positive 传播问题，和当前 Spec 的 precision-first 约束是对齐的。
