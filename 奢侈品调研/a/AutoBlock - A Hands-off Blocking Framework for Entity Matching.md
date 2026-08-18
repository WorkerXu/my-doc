# AutoBlock：从多 Signature Blocking 到二奢腕表 Reference-Link 的直接落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取尚未由 `a` 分析过的论文 **AutoBlock: A Hands-off Blocking Framework for Entity Matching** 做深入分析。

- 论文：<https://arxiv.org/abs/1912.03417>
- WSDM 2020，Wei Zhang、Hao Wei、Bunyamin Sisman、Xin Luna Dong、Christos Faloutsos、David Page
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

先给结论：

> AutoBlock 最值得当前项目借鉴的并不是“再训练一个商品是否相同的二分类器”，而是它把 Blocking 建模成 **多视角表示学习 + 多路近邻检索**：一个商品可以因为标题像、结构化属性像、OCR/标识符像中的任意一路进入候选集，而不必把所有稀疏字段强行压进一个向量。

但是当前业务的约束比 AutoBlock 原论文更严格：

- “同商品”已经被定义为 **同一个 reference number / 型号**；
- 最终决策是 **precision 极端优先**，宁可漏掉，也不能错合；
- 数据规模 100 万～1000 万，且持续增量；
- 字段大量缺失，reference 可能在独立字段、标题、描述或图片里；
- 可以有几百对人工黄金标签。

所以不能照搬 AutoBlock 的“相似度超过阈值即进入匹配”的思路，而应该把它改造成：

```text
Multi-Signature Candidate Retrieval
        ↓
Reference Candidate Set
        ↓
Deterministic Reference Verifier
        ↓
ACCEPT / ABSTAIN / CONFLICT
```

最终生产原则仍然是：

> **模型只负责找到可能的 reference；自动匹配必须由 reference 的确定性证据放行。视觉、语义、向量相似都只能召回或否决，不能单独把两个商品合并。**

这篇论文对当前需求最大的新增价值，是给出了一个比“单 embedding + TopK”更适合字段稀疏场景的 Blocking 架构：**一个实体生成多个互补 signature，每个 signature 覆盖不同属性组合，然后分别做 ANN/LSH，最后取候选并集。**

---

## 1. 为什么选 AutoBlock，而不是重复已有分析

`a` 目录中已经分析过：

- DeepBlocker
- Ditto / Deep Entity Matching with Pre-Trained Language Models
- TransClean
- AnyMatch
- pyJedAI
- Confidence Classifier / Conformal Selective Prediction
- 多模态商品匹配、属性抽取、SKU/MPN 分类等

AutoBlock 尚未分析，而且与已经做过的 DeepBlocker 有一个非常关键的差异：

### DeepBlocker 的主要启发

```text
record -> single tuple embedding -> TopK vector retrieval
```

它很适合说明：

- Blocking 与 Matching 要分层；
- embedding 只能作为召回；
- 百万级不能做笛卡尔积。

### AutoBlock 的主要新增启发

```text
record
  ├── signature 1: 属性组合 A
  ├── signature 2: 属性组合 B
  └── signature 3: 属性组合 C
          ↓
    每路独立 NN search
          ↓
       union candidates
```

这对当前三个来源字段覆盖不一致的问题更重要。

例如同一个 Rolex 商品：

```text
雷小安：
brand=劳力士
reference=NULL
title=劳力士 黑水鬼 126610LN 全套

腕表之家：
brand=Rolex
reference=126610LN
series=Submariner

奢当家：
brand=劳力士
reference=NULL
title=潜航者日历型 黑盘
image_ocr=126610LN
```

如果只有一个统一 embedding，很容易出现：

- reference 强字段被长标题稀释；
- 某一路字段缺失导致向量质量下降；
- 图片 OCR 与文本字段难以合理合并；
- 为了兼容缺失字段只能降低阈值，候选量迅速膨胀。

AutoBlock 的多 signature 思路允许：

```text
sig_ref      = reference / identifier 视角
sig_title    = title / model-name 视角
sig_attr     = series + model + size + material 视角
sig_ocr      = image OCR / card / caseback 视角
```

某一 signature 缺失不会拖垮其他 signature，只要有一路能召回候选即可。

这非常适合当前 Spec。

---

# 2. AutoBlock 原论文到底做了什么

## 2.1 Blocking 的目标

设数据集有 `n` 个 tuple。

如果直接做 Entity Matching，需要考虑接近：

```text
O(n²)
```

的 pair。

Blocking 的作用是先生成一个候选集合 `C`：

```text
所有 pair
   ↓ Blocking
很小的 Candidate Pair Set C
   ↓ Matching
最终 match
```

论文关注两个核心指标：

```text
Blocking Recall = 被候选集覆盖的真实 match / 所有真实 match

P/E Ratio = Candidate Pair 数 / Entity 数
```

Blocking Recall 越高越好，P/E 越低越好。

这一点要与当前项目的最终业务指标严格区分：

```text
AutoBlock：候选召回层 → Recall First
当前最终判定层：Reference Match → Precision First
```

两层不能共用同一个阈值和同一个成功标准。

---

# 3. AutoBlock 的五阶段技术架构

论文把 AutoBlock 拆成五步：

```text
1. Token Embedding
        ↓
2. Attention-based Attribute Embedding
        ↓
3. Multiple Tuple Signatures
        ↓
4. Positive-Pair Training
        ↓
5. Fast NN Search with cosine LSH
```

## 3.1 Token Embedding：fastText

论文使用 fastText。

关键原因是 fastText 使用 character n-gram，可以自然处理：

- OOV；
- 拼写错误；
- 稀有 token；
- 表面形式接近的 token。

这对脏文本 Blocking 很有效。

但在腕表 reference 场景中，需要反向警惕这个优点。

例如：

```text
126610LN
126610LV
```

对普通 NLP 来说它们“非常像”；

对业务来说它们是两个明确不同的 reference，绝不能因为相似而自动合并。

因此生产设计必须将 token 分成至少两类：

```text
普通自然语言 token
identifier / reference token
```

普通 token 可以进入语义 embedding；identifier 必须保留原文、规范化值、来源位置和角色，不允许只剩一个模糊向量。

---

## 3.2 Attention-based Attribute Embedding

AutoBlock 不直接平均一个属性中的全部 token，而是先用序列编码器得到每个 token 的 hidden state，再得到 attention 权重。

论文的核心形式可以写成：

```text
h1...hl = SeqEncoder(v1...vl)
αk = softmax(wᵀ hk)
βk = ρ αk + (1-ρ)/l
attribute_embedding = Σ βk vk
```

其中 `SeqEncoder` 可以是：

- RNN；
- BiLSTM；
- 1D CNN；
- Transformer。

`ρ` 控制 learned attention 与 uniform average 的混合。

这个设计有两个值得当前系统借鉴的点。

### 第一：模型学习“字段内部哪些 token 对召回更稳定”

例如标题：

```text
劳力士 Rolex 潜航者 黑水鬼 126610LN 2023 全套 未使用
```

候选召回真正有价值的 token 可能是：

```text
劳力士 / Rolex
潜航者
126610LN
```

而：

```text
2023
全套
未使用
```

更多描述的是交易状态，不是 reference identity。

Attention 可以自动减少这些词的权重。

### 第二：这只能用于 Blocking，不能用于最终判同款

AutoBlock 的实验中甚至观察到：一些对下游匹配很有区分力的 token 会被 Blocking 模型故意忽略，因为这些 token 经常缺失；忽略它们反而能提高 Blocking Recall。

这与当前项目特别相关：

> 一个 Blocking 模型为了不漏召回，可能主动忽略“真正区分两个 reference 的细粒度 token”。

所以：

```text
Blocking attention 的输出
≠
最终 reference 判定证据
```

这是本项目必须在架构层保证的边界。

---

# 4. AutoBlock 最关键的设计：Multiple Tuple Signatures

## 4.1 为什么一个向量不够

论文举了一个典型问题：

同一个实体的不同记录可能分别只覆盖不同字段。

如果：

```text
record A：只有 title
record B：title + album + composer
record C：album + composer 比较完整，但 title 有噪声
```

为了让：

```text
A ≈ B
B ≈ C
```

一个单向量需要同时兼顾 title 和其他字段。

缺失字段一多，通常只能降低 similarity threshold；阈值一低，大量无关 pair 就会进入候选集。

## 4.2 AutoBlock 的解决方式

它不生成一个 signature，而是生成 `S` 个 signature：

```text
f^(1)(x)
f^(2)(x)
...
f^(S)(x)
```

每个 signature 是若干 attribute embedding 的非负加权组合：

```text
f^(s) = Σ w[s,j] * attribute_embedding[j]
```

然后两个 tuple 的总相似度定义为：

```text
σ(x, y) = max_s cosine(f_s(x), f_s(y))
```

也就是：

> 只要某一个 signature 视角认为二者足够接近，就允许它们进入 Blocking 候选集。

这本质上是一个 **OR-of-views**。

---

# 5. 对腕表数据，Multiple Signature 应该怎么改

我建议不要完全让模型自由发现 signature，而是使用：

> **业务定义好的 evidence group + group 内学习 attention/weight。**

原因是当前场景有 identifier 安全边界，不能允许优化器为了召回率把 reference、平台 SKU、店铺 SKU 等混成不可解释的同一语义空间。

推荐四个 signature group。

## Signature A：Reference / Identifier Signature

输入：

```text
reference_field
mpn/model_no-like token
标题中抽取的 reference candidate
描述中抽取的 reference candidate
```

建议不是普通中文 embedding，而是：

```text
character n-gram encoder
+ raw normalized identifier
+ identifier type
+ source position
```

输出用于：

- 找到拼写/OCR 很接近的 reference；
- 修复空格、连字符、大小写、OCR 误识别；
- 生成候选。

例如：

```text
126610 LN
126610-LN
12661OLN  # OCR 把 0/L/O 混淆
```

可以互相召回。

但最终 `ACCEPT` 时仍必须回到 canonical reference 的确定性证据。

## Signature B：Title / Model Name Signature

输入：

```text
brand
series
model_name
clean_title
```

训练时建议主动 mask 掉已知 reference token 的一部分样本。

目的：

> 防止模型只学会“标题里有 reference 就匹配”，使其在 reference 缺失的商品上仍可根据系列和型号名找候选。

例如：

```text
劳力士 潜航者日历型 黑盘
Rolex Submariner Date Black Dial
```

可以召回同一批 reference 候选，但不能因此直接判同款。

## Signature C：Structured Attribute Signature

输入：

```text
brand
series
case_size
material
movement
color
bezel
gender
```

这一路主要解决：

- 标题很短；
- 标题噪声很大；
- 不同平台字段拆分方式不同。

仍然只能作为候选检索。

## Signature D：Image OCR Signature

输入：

```text
表背 OCR
保卡 OCR
吊牌 OCR
盒证 OCR
图片中的 model/reference token
```

这一路建议只编码 OCR token 和版面/图片角色，不建议一开始直接用“纯视觉长得像”作为 reference signature。

腕表同系列不同 reference 的外观可能高度接近，纯图像相似是典型误合并源。

纯视觉 embedding 可以作为：

```text
manual-review ranking
conflict detector
```

但不要成为自动 ACCEPT 的主证据。

---

# 6. 原论文如何训练 Multiple Signature

AutoBlock 假设有一个正样本集合：

```text
L = {(xi, xj) | xi 与 xj 是同一实体}
```

对每一个正样本 pair：

```text
(xi, xj)
```

随机采一小组 irrelevant tuple：

```text
U = {u1, u2, ...}
```

训练目标要求：

```text
sim(xi, xj)
>
sim(xi, uk)

且

sim(xi, xj)
>
sim(xj, uk)
```

论文把这个构造成一个 softmax 多分类概率，并最大化所有正 pair、所有 signature 上的 log probability。

优化器使用 Adam。

## 6.1 Signature 去重/正交

如果所有 signature 同时自由训练，很容易出现：

```text
sig1 ≈ sig2 ≈ sig3
```

全部选择最强的同一组属性。

论文没有直接加复杂的正交 penalty，而是采用 sequential training：

```text
训练 sig1
→ 标记 sig1 已使用属性
→ 训练 sig2 时禁止再使用这些属性
→ 继续直到属性耗尽或达到 S 上限
```

这使 signature 天然关注不同属性集合。

---

# 7. 原论文训练法不能原样用于当前项目

## 7.1 随机负样本太容易

AutoBlock 的假设是：实体重复很稀疏，所以随机抽一个 tuple，基本可认为是 negative。

这对通用 Blocking 合理，但腕表的真实难点不是随机负样本，而是：

```text
Rolex 126610LN
Rolex 126610LV

AP 15500ST.OO.1220ST.01
AP 15500ST.OO.1220ST.02
```

这些同品牌、同系列、只差少量字符或属性的 pair，才是误匹配风险最高的 hard negative。

如果训练集中大量 negative 都是：

```text
Rolex Submariner
vs
Hermès Birkin
```

模型很容易获得漂亮的 loss，却没有学到真正有用的边界。

### 改造

负样本池建议至少分四类：

```text
N1: random negative
N2: same brand negative
N3: same brand + same series negative
N4: reference edit-distance / OCR-confusable hard negative
```

训练时混合：

```text
20% random
30% same brand
30% same series
20% reference hard negative
```

比例必须由真实数据调，不建议把上面数字当固定参数。

同时要注意：Blocking 层仍然需要高 recall，因此 hard-negative 训练的目的主要是控制候选爆炸，不是让召回器承担最终拒绝职责。

---

## 7.2 原论文的“属性绝对不重叠”不适合直接照搬

论文 sequential signature 的做法是：某个 attribute 被一个 signature 使用后，后续 signature 不再使用。

对于当前业务，这过于刚性。

例如 `brand` 是所有候选路径的重要约束：

```text
sig_ref 需要 brand
sig_title 需要 brand
sig_attr 也需要 brand
```

生产方案应该把特征分为：

```text
Shared hard-gating features
View-private soft features
```

其中：

### Shared hard-gating

```text
brand_id
category/watch-vs-accessory
source
identifier_role
```

可以在所有 signature 路径重复使用。

### View-private soft features

```text
reference char features
model/title text
structured attrs
OCR text
```

再要求这些 soft view 尽量独立。

这比论文的“所有 attribute 完全不重叠”更符合业务。

---

# 8. 最重要的建模重构：不做 Product-Product 全量匹配

当前 Spec 已经明确：

> 同一个商品 = 同一个 reference number。

因此系统没有必要把 100 万～1000 万商品互相两两匹配。

更好的对象是：

```text
Product Record
      ↓
Reference Entity
```

建立全局：

```text
Reference Registry
```

每条商品只需要解析它指向哪个 `ref_entity_id`。

这样：

```text
Product A -> ref_entity_id = R123
Product B -> ref_entity_id = R123

=> 同 reference
```

### 复杂度变化

原问题：

```text
products from source A × source B × source C
```

改造成：

```text
每条 product 查询一个远小于商品全集的 Reference Catalog
```

Reference Catalog 的规模通常会显著小于商品记录数；具体数量以实际 catalog 为准。

这会同时降低：

- 候选搜索量；
- ANN 索引量；
- 在线增量成本；
- cluster 污染风险；
- 可解释性成本。

---

# 9. 推荐最终架构：Reference-Link AutoBlock

```text
                ┌────────────────────┐
                │ 雷小安 / 腕表之家 / 奢当家 │
                └─────────┬──────────┘
                          │
                          ▼
                ┌────────────────────┐
                │ Canonical Product   │
                │ Schema + Normalizer │
                └─────────┬──────────┘
                          │
          ┌───────────────┴────────────────┐
          │                                │
          ▼                                ▼
┌──────────────────┐             ┌──────────────────┐
│ Hard Ref Extract │             │ Soft Evidence     │
│ field/title/OCR  │             │ title/attr/OCR    │
└────────┬─────────┘             └────────┬─────────┘
         │                                │
         │ exact high confidence          │
         ▼                                ▼
┌──────────────────┐          ┌─────────────────────────┐
│ Exact Ref Lookup │          │ Multi-Signature Retriever│
│ brand + ref      │          │ ref/title/attr/ocr ANN  │
└────────┬─────────┘          └────────────┬────────────┘
         │                                  │
         └──────────────┬───────────────────┘
                        ▼
               ┌────────────────────┐
               │ Candidate Reference │
               │ Set + Evidence      │
               └─────────┬──────────┘
                         ▼
               ┌────────────────────┐
               │ Hard Verifier       │
               │ conflict-first      │
               └───────┬───────┬────┘
                       │       │
                   ACCEPT    ABSTAIN
                       │       │
                       ▼       ▼
             ref_entity_id   人工复核
```

---

# 10. 数据模型

## 10.1 `reference_entity`

```sql
CREATE TABLE reference_entity (
    ref_entity_id        BIGINT PRIMARY KEY,
    brand_id             BIGINT NOT NULL,
    canonical_reference  VARCHAR(128) NOT NULL,
    series               VARCHAR(256),
    model_name           VARCHAR(512),
    status               VARCHAR(32) NOT NULL DEFAULT 'active',
    created_at           TIMESTAMP NOT NULL,
    updated_at           TIMESTAMP NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);
```

关键点：

```text
(brand_id, canonical_reference)
```

而不是裸 reference。

即使业务目前认为 reference 全局唯一，也不建议生产系统把这个假设写死。

## 10.2 `reference_alias`

```sql
CREATE TABLE reference_alias (
    alias_id              BIGINT PRIMARY KEY,
    ref_entity_id         BIGINT NOT NULL,
    raw_alias             VARCHAR(256) NOT NULL,
    normalized_alias      VARCHAR(256) NOT NULL,
    alias_type            VARCHAR(32) NOT NULL,
    source                VARCHAR(32),
    verified              BOOLEAN NOT NULL DEFAULT FALSE
);
```

例如：

```text
126610LN
126610 LN
126610-LN
REF.126610LN
```

都可以指向同一个 canonical reference，但 alias 必须有来源和验证状态。

## 10.3 `product_record`

```sql
CREATE TABLE product_record (
    product_id              BIGINT PRIMARY KEY,
    source                  VARCHAR(32) NOT NULL,
    source_item_id          VARCHAR(128) NOT NULL,
    brand_raw               VARCHAR(256),
    brand_id                BIGINT,
    title                   TEXT,
    description             TEXT,
    reference_raw           VARCHAR(256),
    reference_normalized    VARCHAR(256),
    product_type            VARCHAR(64),
    series                  VARCHAR(256),
    model_name              VARCHAR(512),
    attrs_json              JSON,
    ref_entity_id           BIGINT,
    link_status             VARCHAR(32) NOT NULL,
    extractor_version       VARCHAR(64),
    model_version           VARCHAR(64),
    index_version           VARCHAR(64),
    updated_at              TIMESTAMP NOT NULL,
    UNIQUE (source, source_item_id)
);
```

`link_status` 建议不是 boolean，而是：

```text
EXACT_ACCEPT
MULTI_EVIDENCE_ACCEPT
ABSTAIN
CONFLICT
NO_CANDIDATE
MANUAL_ACCEPT
MANUAL_REJECT
```

这样才有可追踪性。

## 10.4 `reference_evidence`

这是 precision-first 系统非常重要的一张表。

```sql
CREATE TABLE reference_evidence (
    evidence_id          BIGINT PRIMARY KEY,
    product_id           BIGINT NOT NULL,
    candidate_ref_id     BIGINT,
    evidence_type        VARCHAR(64) NOT NULL,
    raw_value            TEXT,
    normalized_value     TEXT,
    confidence           DOUBLE PRECISION,
    source_location      VARCHAR(64),
    is_support           BOOLEAN,
    extractor_version    VARCHAR(64),
    created_at           TIMESTAMP NOT NULL
);
```

例如：

```text
TITLE_REF_EXACT
STRUCTURED_REF_EXACT
OCR_REF_EXACT
MODEL_NAME_SIM
SERIES_MATCH
BRAND_MATCH
ACCESSORY_NEGATIVE_CONTEXT
REF_CONFLICT
```

最终任何自动 ACCEPT 都必须能回答：

> 为什么系统认为这条商品是这个 reference？

---

# 11. Reference Normalization 必须是可逆、分层的

不要只有一个暴力：

```python
re.sub(r'[^A-Z0-9]', '', ref)
```

因为这可能把本来不同的语义形式合并。

建议保留三层：

```text
raw_reference
lexical_normalized_reference
canonical_reference
```

### lexical normalization

只做低风险转换：

```text
Unicode normalize
uppercase
统一全角/半角
规范空白
规范连接符
去掉明确的 “REF / MODEL / 型号” 前缀
```

### canonicalization

必须是：

```text
brand-aware
pattern-aware
registry-aware
```

例如某品牌合法 reference pattern 与另一个品牌不一样。

如果 normalization 产生多个可能结果：

```text
不要猜
→ candidate set
→ ABSTAIN / review
```

---

# 12. Multi-Signature Retriever 的直接实现

不建议第一版复刻 2019 年论文里的 FALCONN cross-polytope LSH 代码。

论文的核心思想应该保留：

```text
多 signature
独立向量空间
独立近邻查询
union candidate
```

生产实现可以使用成熟 ANN 引擎，例如：

```text
FAISS HNSW / IVF
HNSWlib
Qdrant / Milvus / OpenSearch vector index
```

具体选型取决于当前基础设施。

## 12.1 按 brand 分区

查询前先做：

```text
brand canonicalization
```

然后：

```text
query only within brand partition
```

如果 brand 不确定：

```text
不自动 ACCEPT
```

不要让向量检索跨品牌自由找“长得像”的 reference。

## 12.2 每个 Reference Entity 建多份向量

```text
ref_entity_id=R123

sig_ref_vector
sig_title_vector
sig_attr_vector
sig_ocr_alias_vectors
```

商品 query 也按同样 view 生成向量。

## 12.3 查询

伪代码：

```python
def retrieve_reference_candidates(product):
    if product.brand_id is None:
        return []

    candidates = {}

    if product.ref_signature is not None:
        for hit in ref_index.search(
            brand_id=product.brand_id,
            signature='ref',
            vector=product.ref_signature,
            topk=20,
        ):
            merge(candidates, hit, route='ref')

    if product.title_signature is not None:
        for hit in ref_index.search(
            brand_id=product.brand_id,
            signature='title',
            vector=product.title_signature,
            topk=30,
        ):
            merge(candidates, hit, route='title')

    if product.attr_signature is not None:
        for hit in ref_index.search(
            brand_id=product.brand_id,
            signature='attr',
            vector=product.attr_signature,
            topk=30,
        ):
            merge(candidates, hit, route='attr')

    if product.ocr_signature is not None:
        for hit in ref_index.search(
            brand_id=product.brand_id,
            signature='ocr',
            vector=product.ocr_signature,
            topk=20,
        ):
            merge(candidates, hit, route='ocr')

    return rank_for_verification(candidates)
```

注意：

`topk=20/30` 只是示意，不应该直接作为生产参数。

需要用黄金集调到：

```text
Blocking Recall 接近 100%
同时 candidate count 可控
```

---

# 13. 不要把多个 signature 的相似度平均

一个常见错误是：

```python
score = (
    0.4 * ref_sim
    + 0.3 * title_sim
    + 0.2 * attr_sim
    + 0.1 * image_sim
)
```

这会产生一个非常危险的行为：

```text
reference 明确冲突
但 title/image 非常相似
→ 总分仍然很高
```

对腕表同系列不同 reference，这是不能接受的。

AutoBlock 原论文用：

```text
max(signature similarities)
```

做 Blocking 是合理的，因为它只是在“召回候选”。

当前系统建议：

### Candidate Retrieval

```text
OR / union
```

### Final Verification

```text
hard constraints + contradiction first
```

即：

```python
if has_reference_conflict(product, candidate):
    return CONFLICT

if exact_reference_supported(product, candidate):
    return ACCEPT

return ABSTAIN
```

永远不要用“其他相似分把 reference 冲突抵消掉”。

---

# 14. Final Hard Verifier：自动放行规则

建议把最终 verifier 设计成规则引擎，而不是一个黑盒总分。

## 14.1 第一原则：Conflict First

任何可信 evidence 出现不同 reference：

```text
candidate A = 126610LN
可信 OCR = 126610LV
```

结果：

```text
CONFLICT
```

不是：

```text
“综合得分 0.92，仍然 match”
```

## 14.2 自动 ACCEPT 的最低条件

建议第一版只接受类似：

```text
brand exact
AND
canonical reference exact
AND
reference role = current product itself
AND
no conflicting reference evidence
AND
product_type = target watch, not accessory/strap/box/manual
```

### reference role 特别重要

商品标题可能是：

```text
适配 Rolex 126610LN 原装表带
```

出现 `126610LN` 并不意味着当前商品就是 `126610LN` 腕表。

必须把 identifier 分成：

```text
OWN_REFERENCE
COMPATIBLE_REFERENCE
RELATED_REFERENCE
PLATFORM_SKU
SELLER_SKU
UNKNOWN_IDENTIFIER
```

只有：

```text
OWN_REFERENCE
```

可以进入自动放行逻辑。

---

# 15. Evidence Tier 建议

## Tier A：最高可信

```text
平台独立 reference 字段
+ brand 一致
+ pattern 合法
+ registry exact hit
+ 无冲突
```

可以自动 ACCEPT。

## Tier B：强可信

```text
标题明确 reference
+ identifier role = OWN_REFERENCE
+ category = watch
+ brand exact
+ registry exact hit
+ 无冲突
```

可以在黄金集证明误匹配为 0 后考虑自动 ACCEPT。

## Tier C：多证据

```text
OCR exact reference
+ title/model/series 一致
+ brand exact
+ 多张图片 OCR 一致或有第二独立证据
```

可以作为高优先级人工复核；是否自动放行取决于实测 precision。

## Tier D：只有语义/视觉相似

```text
title similar
image similar
attrs similar
but no exact reference
```

只能：

```text
ABSTAIN
```

绝不自动 match。

---

# 16. 训练数据怎么构造

Spec 允许人工标几百对，但其实可以利用已有强 reference 自动构造大量 Blocking 正样本。

## 16.1 Positive Pair

从三个来源中，已经能够确定：

```text
(brand_id, canonical_reference)
```

相同的商品记录直接组成 positive pair。

例如：

```text
雷小安商品 A -> Rolex / 126610LN
腕表之家商品 B -> Rolex / 126610LN

(A, B) = positive
```

这些 positive 用于训练 candidate retriever，不代表把模型输出直接作为最终匹配。

## 16.2 强制 Cross-source Sampling

训练 pair 应优先跨来源：

```text
雷小安 ↔ 腕表之家
雷小安 ↔ 奢当家
腕表之家 ↔ 奢当家
```

否则模型容易学习平台内部固定文本模板，而不是实体稳定特征。

## 16.3 Hard Negative

从同品牌、同系列中找：

```text
reference 不同
但 title / image / attrs 很像
```

重点建设：

```text
one-character difference
suffix difference
same family different size
same model different dial
accessory mentions watch reference
platform SKU looks like reference
OCR confusion pair
```

这些 hard negative 比随机 negative 对系统安全性更重要。

---

# 17. 训练时必须做 Reference Masking

如果所有训练正样本标题都直接带 reference：

```text
Rolex ... 126610LN
Rolex ... 126610LN
```

Title Signature 会学到一个捷径：

```text
只看 126610LN
```

于是碰到真正困难的：

```text
reference 缺失
```

就失效。

建议训练 `sig_title` 时随机 mask 已知 reference：

```text
Rolex Submariner 126610LN Black
↓
Rolex Submariner [REF] Black
```

但 `sig_ref` 保留 reference。

形成明确分工：

```text
sig_ref   学 identifier 近似
sig_title 学 reference 缺失时的语义召回
sig_attr  学结构化属性召回
sig_ocr   学图片文字召回
```

这才真正发挥 Multiple Signature 的价值。

---

# 18. 对 AutoBlock Loss 的生产改造

可以保留论文的核心 pairwise ranking / contrastive 思路，但生产实现建议把每个 view 单独训练。

例如：

```python
loss = (
    loss_ref
    + loss_title
    + loss_attr
    + loss_ocr
)
```

每个 loss 的 positive 是同 reference。

Negative 池使用：

```text
random
same brand
same series
reference hard negative
```

对于 identifier view，还可以额外加入字符级 hard negative：

```text
126610LN ↔ 126610LV
5711/1A-010 ↔ 5711/1A-011
```

但要谨记：

> Retriever 的 loss 只优化候选质量；最终 precision 不能从这个 loss 推导出来。

---

# 19. ANN / LSH：论文实现与生产实现的区别

AutoBlock 论文使用 cosine + cross-polytope LSH。

论文附录给出的实现信息包括：

```text
PyTorch
FALCONN cross-polytope LSH
n = 1,000,000
vector dim = 300
10 个 hash tables
每 table 2 个 hash functions
额外 multi-probe 一个相邻 bucket
```

论文在百万点实验中观察到：

```text
LSH 查询比 brute force 快约 40–80 倍
```

同时近似搜索对 recall 的损失很小；在 Grocery 数据上，不同阈值下相对 brute-force 的最大 recall gap 约 1.5%。

这些结果证明：

> 对百万级 Blocking，近似 NN 是合理架构。

但不要把论文的具体 FALCONN 参数当成 2026 年生产默认值。

生产建议只保留思想：

```text
cosine/inner-product representation
+ approximate nearest neighbor
+ threshold/topK
+ per-view index
```

ANN 具体实现由现有基础设施决定。

---

# 20. 增量更新怎么做

当前数据不是一次性离线任务，而是持续新增商品。

因此系统必须从第一天就区分：

```text
Reference Catalog Update
Product Event Update
```

## 20.1 新商品到达

```text
product_ingested
  ↓
normalize
  ↓
extract reference evidence
  ↓
exact lookup
  ├── hit + pass verifier -> link
  └── miss/uncertain -> multi-signature retrieval
                         ↓
                       verifier
                         ↓
                  accept / abstain
```

只需要查询已有 Reference Index，不需要重跑历史商品两两匹配。

## 20.2 新 reference 进入 Catalog

当人工确认或权威数据发现一个新 reference：

```text
insert reference_entity
  ↓
compute all signature vectors
  ↓
upsert corresponding ANN partitions
```

然后只需要重试：

```text
NO_CANDIDATE / ABSTAIN
```

的历史商品，不需要全库重算。

## 20.3 Version 必须落库

每个 link 保存：

```text
extractor_version
normalizer_version
model_version
index_version
rule_version
```

否则后续模型升级后无法解释：

```text
为什么昨天没匹配，今天匹配了？
```

---

# 21. Candidate Cache

对于千万级持续增量，不建议每次人工页面打开时实时跑所有模型。

候选结果落表：

```sql
CREATE TABLE reference_candidate (
    product_id         BIGINT NOT NULL,
    candidate_ref_id   BIGINT NOT NULL,
    route              VARCHAR(32) NOT NULL,
    similarity         DOUBLE PRECISION,
    rank_in_route      INT,
    model_version      VARCHAR(64),
    index_version      VARCHAR(64),
    created_at         TIMESTAMP NOT NULL,
    PRIMARY KEY (product_id, candidate_ref_id, route)
);
```

这样可以：

- 复用 candidate；
- 做 offline evaluation；
- 查看每个 signature 的贡献；
- 回放旧模型；
- 计算召回率和 P/E/候选量。

---

# 22. Multi-Signature 的可观测性

每个 candidate 必须记录：

```json
{
  "ref_entity_id": 123,
  "routes": {
    "ref":   {"rank": 1, "score": 0.98},
    "title": {"rank": 2, "score": 0.91},
    "attr":  {"rank": 4, "score": 0.84},
    "ocr":   null
  }
}
```

但这个 score 只用于：

```text
candidate ranking
manual-review priority
monitoring
```

不能直接做：

```text
score > 0.95 => same reference
```

最终仍走 evidence verifier。

---

# 23. 评测必须拆成两套指标

## 23.1 Retriever / Blocking 指标

借鉴 AutoBlock：

```text
Blocking Recall@K
Mean Candidates Per Product
P/E-like candidate ratio
Index query latency
Missing-field slice recall
```

需要按场景切片：

```text
reference 独立字段存在
reference 只在标题
reference 只在 OCR
reference 完全缺失
同系列 hard negatives
新品/未见 reference
```

## 23.2 Final Verifier 指标

这是业务真正关心的：

```text
Precision
False Positive Count
Abstain Rate
Coverage
Manual Review Rate
```

关键验收不能只看 F1。

建议上线门槛：

```text
黄金集 + hard-negative 集中
自动 ACCEPT 必须零 observed false positive
```

然后再根据样本量做统计置信评估，而不是因为“测试集没错”就声称数学意义上的 100% precision。

---

# 24. 黄金标签的几百对应该花在哪里

不要把几百对全花在随机 pair 上。

最有价值的是：

```text
同品牌同系列不同 reference
只有一个字符不同
标题中有多个 reference
配件/表带/盒证提到目标表 reference
OCR 0/O、1/I/L、5/S、8/B 混淆
平台 SKU 像 reference
brand alias 冲突
同型号不同后缀
reference 缺失但图像/标题很像
```

建议人工标签结构不是简单：

```text
MATCH / NOT_MATCH
```

而是：

```json
{
  "same_reference": false,
  "left_reference": "126610LN",
  "right_reference": "126610LV",
  "identifier_role_left": "OWN_REFERENCE",
  "identifier_role_right": "OWN_REFERENCE",
  "error_type": "NEAR_REFERENCE"
}
```

这样才能同时训练：

- reference extractor；
- role classifier；
- candidate retriever；
- hard verifier 测试集。

---

# 25. 一个直接可实现的 Verifier 伪代码

```python
from enum import Enum

class LinkStatus(str, Enum):
    ACCEPT = "ACCEPT"
    ABSTAIN = "ABSTAIN"
    CONFLICT = "CONFLICT"


def verify(product, candidate_ref, evidence):
    # 1. 品牌不确定或冲突：永不自动匹配
    if product.brand_id is None:
        return LinkStatus.ABSTAIN

    if product.brand_id != candidate_ref.brand_id:
        return LinkStatus.CONFLICT

    # 2. 当前商品若明确不是腕表本体，出现兼容 reference 不能作为自身 reference
    if product.product_type in {
        "strap", "bracelet", "box", "manual", "accessory", "spare_part"
    }:
        return LinkStatus.ABSTAIN

    # 3. 收集所有可信的 OWN_REFERENCE
    own_refs = {
        e.canonical_reference
        for e in evidence
        if e.identifier_role == "OWN_REFERENCE"
        and e.is_high_trust
        and e.canonical_reference is not None
    }

    # 4. 多个可信 ref 互相冲突，直接拒绝
    if len(own_refs) > 1:
        return LinkStatus.CONFLICT

    # 5. 唯一可信 reference 与 candidate exact 一致
    if len(own_refs) == 1:
        only_ref = next(iter(own_refs))
        if only_ref == candidate_ref.canonical_reference:
            return LinkStatus.ACCEPT
        return LinkStatus.CONFLICT

    # 6. 只有语义 / ANN / 图片相似度，不允许自动放行
    return LinkStatus.ABSTAIN
```

这个 verifier 看起来“保守”，但正符合 Spec：

```text
允许漏匹配
绝不能误匹配
```

---

# 26. 多来源聚合时不要用传递相似度直接合并

错误做法：

```text
A ≈ B
B ≈ C
=> A/B/C cluster
```

因为一个 false-positive edge 就可能污染整个 cluster。

正确做法：

```text
A -> Reference R
B -> Reference R
C -> Reference R

=> cluster by R
```

聚类依据是：

```text
ref_entity_id
```

而不是商品 pair 相似图的 connected component。

这样新增来源也很简单：

```text
Source D Product -> Reference R
```

直接加入已有实体。

---

# 27. 第一版不要做哪些东西

为了快速并安全上线，我建议第一版明确不做：

## 27.1 不做全库 Product-Product ANN

没有必要，Reference Catalog 才应该是主要索引对象。

## 27.2 不让 LLM 直接输出 match=true

LLM 可以：

```text
抽取 reference candidate
判断 identifier role
解释冲突
```

但最终 ACCEPT 由结构化规则执行。

## 27.3 不用纯图片相似自动匹配

相似 variant 外观太接近。

## 27.4 不做一个综合 probability

例如：

```text
P(match)=0.998
```

没有解释哪个 reference 证据支持它，而且会隐藏 hard conflict。

## 27.5 不把 Platform SKU 当 Reference

必须先做 identifier role/type 分类。

---

# 28. 推荐 MVP

## Phase 0：Reference Registry + Exact Path

先完成：

```text
brand canonicalization
reference normalization
reference registry
reference alias
identifier role
exact lookup
conflict-first verifier
```

这一阶段甚至可以不训练 AutoBlock。

目标是先拿到一批：

```text
高精度自动链接结果
```

同时这些结果自然成为后续 retriever 的 positive labels。

## Phase 1：Multiple Signature Candidate Retrieval

实现：

```text
sig_ref
sig_title
sig_attr
sig_ocr
```

每路独立索引，最后 union。

目标不是增加自动 ACCEPT，而是：

```text
减少 NO_CANDIDATE
提高人工复核效率
覆盖 reference 隐藏/缺失记录
```

## Phase 2：学习 Attention / Signature Weight

用 Phase 0 的高置信正样本训练：

```text
attribute attention
view-specific embedding
hard-negative-aware ranking
```

并对比：

```text
固定规则 signature
vs
learned signature
```

只有 Blocking Recall 明显提升且候选量可控才上线。

## Phase 3：增量闭环

人工复核产生：

```text
new aliases
new reference entities
hard negatives
identifier-role labels
```

持续进入训练和规则库。

---

# 29. 一个更现实的生产组件划分

```text
services/
  ingest/
    leixiaoan_adapter
    xiaohongshu?  # 仅示意，按真实来源
    xiaohongbiao? # 不要硬编码不存在的来源
    watchhouse_adapter
    shedangjia_adapter

  normalize/
    brand_normalizer
    reference_normalizer
    product_type_classifier
    identifier_role_classifier

  reference_registry/
    registry_api
    alias_store
    pattern_store

  extraction/
    structured_ref_extractor
    title_ref_extractor
    ocr_ref_extractor

  retrieval/
    ref_signature_encoder
    title_signature_encoder
    attr_signature_encoder
    ocr_signature_encoder
    ann_router
    candidate_union

  verification/
    conflict_detector
    deterministic_verifier
    evidence_ledger

  review/
    review_queue
    label_export

  evaluation/
    blocking_eval
    precision_eval
    hard_negative_suite
```

上面的 adapter 名称只应按真实数据源实现；当前明确来源是：

```text
雷小安
腕表之家
奢当家
```

---

# 30. 对 AutoBlock 论文实验结果的解读

论文在三个真实数据集上比较：

```text
Movie      结构化且相对干净
Music      脏数据、字段错位
Grocery    非结构化长文本
```

规模中最大的一侧超过 200 万记录。

AutoBlock 在 dirty / unstructured 数据上表现特别好。

原因不是它“更会做最终匹配”，而是：

1. attention 能忽略不稳定噪声；
2. 多 signature 能覆盖不同缺失字段模式；
3. 近似 NN 使高维 fuzzy blocking 可扩展；
4. positive labels 让 blocking metric 自动适应真实数据。

这与当前三个平台高度异构、reference 覆盖不一致的场景非常接近。

但论文自己的观察也提醒我们：

> Blocking 模型会主动忽略某些有区分力但容易缺失的 token，以换取 recall。

所以 AutoBlock 非常适合做“候选召回器”，却不适合直接承担当前系统的 precision-first 最终判定。

---

# 31. AutoBlock 与当前需求的最终对应关系

| AutoBlock 原概念 | 当前二奢腕表系统 |
|---|---|
| Tuple | 平台商品记录 / Reference Entity |
| Attribute | brand/title/reference/series/attrs/OCR |
| Positive labels | 已确认同 canonical reference 的跨源商品 |
| Token embedding | 文本、identifier char、OCR token embedding |
| Attribute attention | 学习字段内部稳定召回 token |
| Multiple signatures | ref/title/attr/OCR 多证据通道 |
| Cosine NN | Product → Reference candidate retrieval |
| LSH candidate union | 多 ANN route 结果 union |
| Blocking Recall | Reference Recall@K |
| P/E ratio | 每商品平均候选 reference 数 |
| Downstream matcher | Deterministic Reference Verifier |

这个映射几乎可以直接实现。

---

# 32. 最终推荐方案

如果现在就要确定技术路线，我建议：

```text
                    Reference Registry
                           ▲
                           │
                 authority / manual
                           │
Product ── normalize ── ref extraction
  │                        │
  │                 exact high-confidence
  │                        │
  │                        ├─────────────► Exact Lookup
  │                        │                  │
  │                        │                  ▼
  │                        │              Hard Verify
  │                        │
  └── uncertain ─► Multi-Signature Retrieval
                    ├─ ref-char ANN
                    ├─ title ANN
                    ├─ attrs ANN
                    └─ OCR ANN
                           │
                           ▼
                    candidate union
                           │
                           ▼
                     Hard Verify
                      │        │
                    ACCEPT   ABSTAIN
```

其中：

```text
ANN / AutoBlock-like model = Recall Layer
Reference Exact Evidence   = Decision Layer
```

两个层必须物理和逻辑分离。

---

# 33. 最终一句话

AutoBlock 对当前需求最值得直接落地的不是它 2019 年的具体 fastText + FALCONN 组合，而是它的架构思想：

> **面对字段稀疏和脏数据，不要把所有证据压成一个向量；为不同证据建立多个独立 signature，分别做高召回候选检索，再用业务确定性规则收口。**

针对腕表项目，再加一条最重要的安全约束：

> **任何 signature 的高相似都只能产生候选；只有 brand-aware canonical reference 的确定性证据、且无冲突时，才能自动写入同一个 `ref_entity_id`。**

这样既能吸收 AutoBlock 在百万级 Blocking、字段缺失和多属性组合上的优势，又不会让“语义很像”“图片很像”“型号只差一个字符”变成不可接受的 false positive。

这比直接训练一个端到端 `same_product=true/false` 模型更符合当前 Spec，也更适合 100 万～1000 万级持续增量数据。