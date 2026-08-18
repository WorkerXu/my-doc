# AutoBlock: A Hands-off Blocking Framework for Entity Matching —— 面向千万级腕表 Reference 匹配的多路候选召回层

- 分析人：b
- 调研文章：AutoBlock: A Hands-off Blocking Framework for Entity Matching
- 论文：https://arxiv.org/abs/1912.03417
- 会议：WSDM 2020
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31
- 本次分析前已确认：`奢侈品调研/b/` 中不存在本文章分析文件。

---

## 1. 先给结论

AutoBlock 很适合当前需求，但**只适合做 Blocking / Candidate Retrieval，不适合成为“同一商品”的最终判定器**。

当前 Spec 对“同一商品”的定义非常特殊：

> **同一 reference number / 型号，才是同一商品。**

同时还有一个比一般 entity matching 更苛刻的业务约束：

> **绝对不能误匹配，precision 优先到极致，可以漏匹配。**

因此，本项目不应该把最终问题建模成普通的：

```text
record A + record B -> neural matcher -> match / non-match
```

更合理的架构是：

```text
record
  -> 品牌规范化
  -> reference 候选抽取
  -> 编号角色判断
  -> canonical reference 规范化
  -> 连接到唯一 (brand_id, canonical_reference) 实体
```

只有在 reference 缺失、抽取失败、字段错位、文本极脏时，才进入 AutoBlock 风格的候选召回层：

```text
reference 不确定记录
  -> 多路 signature
  -> ANN/Blocking
  -> 找到少量可能的 reference entity / 商品记录
  -> 再做 reference 证据验证
  -> ACCEPT / REVIEW / REJECT
```

**最重要的边界：Blocking 的相似度只能回答“值得检查谁”，不能回答“它们就是同一个 reference”。**

本篇最值得迁移的不是 2019 年的 fastText + FALCONN 技术栈，而是三个思想：

1. **把 Blocking 重写成 Nearest Neighbor Search 问题；**
2. **不要把一条稀疏记录压成单一表示，而是学习多个互补 signature；**
3. **通过少量高置信 positive pairs 学习“哪些字段/哪些 token 对召回有用”，而不是手写大量 blocking key。**

对当前三源腕表数据，我建议把它改造为：

> **Reference-Centric Multi-Lane Blocking：以 `(brand, canonical_reference)` 为最终实体键，用 deterministic reference lane + semantic lane + OCR lane + visual lane 并行召回，候选取并集；任何 ANN lane 都无权直接产生自动匹配。**

---

## 2. 为什么本次选 AutoBlock，而不是重复 DeepBlocker

`b/` 下已经有 `DeepBlocker.md`，两篇都属于深度 Blocking，但关注点并不完全一样。

DeepBlocker 更像是一个“blocking design space + 可组合 tuple embedding/vector pairing”的框架，适合快速比较 embedding 与 ANN 方案；AutoBlock 则把问题进一步拆成：

```text
Token Embedding
    ↓
Attribute Encoder
    ↓
Multiple Tuple Signatures
    ↓
Similarity-Preserving Training
    ↓
Fast NN Search
```

它对当前需求最有价值的是 **Multiple Signatures**。

腕表三源数据天然存在：

- reference 在某来源是独立字段，在另一个来源埋在 title；
- brand / series / caliber / diameter 的覆盖率不同；
- 平台字段定义不一致；
- 同一语义可能出现在不同属性；
- 图片 OCR 可能提供文本没有的 reference；
- 同一系列不同 reference 的文本、视觉极其相似。

如果把所有字段直接拼起来形成一个向量，很容易发生：

```text
记录 1：brand + title + reference
记录 2：brand + title，reference 缺失
记录 3：brand + series + caliber，title 缺失
```

一个 embedding 很难同时让 1↔2 和 1↔3 都靠得很近，又不把大量同系列不同 reference 拉进来。

AutoBlock 的设计是：**一个 tuple 可以拥有多个 signature，只要某一 signature 相近就进入候选。**

这正好适合稀疏、字段错位的数据。

但是，需要明确反驳一个可能的误用：

> AutoBlock 的“max similarity over signatures”非常适合 Blocking，却非常不适合 precision-first 的最终匹配。

只要一个 signature 相似就放行，在候选召回层是优点；在最终实体合并层则会制造严重 false positive。

因此本篇的落地点不是“用 AutoBlock 替代 reference 规则”，而是“用 AutoBlock 补 reference 规则覆盖不到的候选搜索”。

---

## 3. AutoBlock 原论文的核心架构

论文把 Blocking 看作一个近邻搜索问题。

传统 Blocking 常常是：

```text
blocking_key(record) -> bucket
bucket 内两两比较
```

例如：

```text
title exact match
brand + first two title tokens
manufacturer + model prefix
```

这类方法的问题是：

- 字段需要人工清洗；
- schema 不一致时容易漏；
- typo、缺失、属性错位时 exact key 失效；
- 为了提高 recall 不断增加 key，会导致候选爆炸。

AutoBlock 的改写是：

```text
如果能学习一个 similarity(record_a, record_b)
并能高效做 nearest-neighbor search
那么 blocking 就不必依赖人工 key。
```

论文的完整链路是：

```text
Raw Tuple
  │
  ├─ attribute 1 -> token embedding -> attention encoder -> g1
  ├─ attribute 2 -> token embedding -> attention encoder -> g2
  ├─ ...
  └─ attribute m -> token embedding -> attention encoder -> gm
                        │
                        ▼
             multiple signature functions
              f1(g1...gm), f2(...), ...
                        │
                        ▼
             cosine-similarity signatures
                        │
                        ▼
              cross-polytope LSH
                        │
                        ▼
                 Candidate Pairs
```

论文强调 Blocking 的四个目标：

1. 高 recall：真实匹配不能大量在候选阶段被漏掉；
2. 候选数量小：P/E ratio 低；
3. 少人工调 blocking key；
4. 能扩展到百万级。

注意：这四个目标都没有要求 Blocking 自己拥有极高 precision。

这和当前系统的职责划分非常重要：

```text
Blocking 优化 candidate recall / cost
Final Decision 优化 precision / safety
```

两个目标必须解耦。

---

## 4. Step 1：fastText Token Embedding —— 对普通脏文本有用，对 reference 有隐患

论文第一步用 fastText。

fastText 和普通 word embedding 最大区别是利用 character n-gram，因此：

- 拼写错误仍可能得到相似表示；
- rare token/OOV 仍能得到向量；
- 对脏文本比固定词表模型稳。

这对于普通商品词非常合理：

```text
Datejust
Date Just
Date-just
```

或者：

```text
Oystersteel
Oyster steel
```

都有机会保持近似语义。

### 4.1 但 reference number 不是自然语言语义

腕表 reference 的关键问题是：

```text
126334
126333
126300
```

在字符层面高度相似，但业务意义可能完全不同。

一个 subword embedding 很可能认为：

```text
sim(126334, 126333) 很高
```

而业务规则要求：

```text
canonical_ref(126334) != canonical_ref(126333)
=> 最终绝不能自动合并
```

因此不能把 reference token 当成普通自然语言 token 交给语义 embedding 决定身份。

### 4.2 建议的改造：Code Token 与 Semantic Token 分轨

预处理时先做 token role tagging：

```text
TITLE:
“劳力士 日志型 126334 蓝盘 41mm 全套”

=>
BRAND_TOKEN: 劳力士
SERIES_TOKEN: 日志型
CODE_TOKEN: 126334
ATTR_TOKEN: 蓝盘
SIZE_TOKEN: 41mm
OTHER_TOKEN: 全套
```

之后生成两类表示：

#### Lane A：Code / Reference Lane

不做语义近似，强调字符串身份：

- exact canonical reference；
- separator-normalized exact；
- 品牌限定的 alias map；
- 可选 character edit-distance 仅用于“候选”，不能直接确认。

#### Lane B：Semantic Lane

把 code-like token mask 掉：

```text
劳力士 日志型 [REF] 蓝盘 41mm 全套
```

再做 text embedding。

这样 semantic lane 学的是：

- 品牌；
- 系列；
- 材质；
- 尺寸；
- 机芯；
- 表盘；
- 商品类别。

它可以在 reference 缺失时找到“可能同系列”的候选，但不会把相邻 reference 的字符相似度误当成身份。

这是迁移 AutoBlock 时必须做的第一处结构性修改。

---

## 5. Step 2：Attention-based Attribute Encoder —— 自动学“哪些 token 对 Blocking 有用”

AutoBlock 对每个 attribute 单独编码。

论文使用的核心形式本质上是：

```text
attribute tokens
  -> sequence encoder
  -> per-token attention score
  -> weighted average
  -> attribute embedding
```

论文实验配置中：

- token embedding：300D fastText；
- sequence encoder：单层 Bi-LSTM；
- hidden units：64；
- 最终 attribute embedding 仍是 token embedding 的加权平均。

这里有一个很值得迁移的思想：

> 不直接人工写 stopword/字段清洗规则，而让 positive pairs 教模型哪些 token 在跨源匹配中更稳定。

论文观察到：

- title 前部品牌类 token 往往权重大；
- stop words/标点会自然被降权；
- 某些版本描述会被降权；
- 一些看起来“很有区分度”的 token 反而会被 Blocking 模型忽略，因为它们跨源经常缺失或表达不一致。

### 5.1 这点对腕表既有价值，也有风险

价值在于：

三平台很可能有大量：

```text
“全套”
“附件齐全”
“附件：盒证”
“原装盒卡”
```

这种字段对 identity 不稳定，attention 可以自动降低它们的重要性。

风险在于：

如果训练目标只追求 Blocking recall，模型也可能学习到：

> reference 经常缺失，因此 reference token 不稳定，应该降权。

这在 Blocking 中可以接受，在 Final Match 中绝对不可接受。

因此系统必须明确：

```text
Attention score ≠ Identity evidence weight
```

attention 只能控制候选召回表示，不能控制最终 reference 证据权重。

---

## 6. Step 3：Multiple Tuple Signatures —— 本文最值得迁移的部分

如果一个 tuple 只有一个 embedding，会遇到一个不可避免的问题：

```text
A 有 title，没有 caliber
B 有 title + caliber
C 没 title，但有 caliber + series
```

为了让 A↔B 相近，向量可能被 title 主导；
为了让 B↔C 相近，又希望 caliber/series 主导。

一个 embedding 很难同时满足。

AutoBlock 的解决方案不是强行找一个“全能 embedding”，而是为每个 tuple 生成多个 signature。

原论文每个 signature 是对不同 attribute embedding 的加权组合：

```text
sig_s(record) = Σ w_sj * attribute_embedding_j
```

并要求不同 signature 尽可能使用不同字段，从而覆盖不同“相似理由”。

最终 Blocking similarity 是：

```text
max_s cosine(sig_s(A), sig_s(B))
```

也就是说：

> 只要有一条 signature lane 很像，就进入候选。

### 6.1 对三源腕表的正确改造

我不建议完全照搬“模型自动决定 signature”。

当前业务里有明确的 identity hierarchy，所以应该采用 **显式 lane + lane 内学习权重**：

```text
Lane 0: exact_reference
Lane 1: reference_candidate_lexical
Lane 2: semantic_title
Lane 3: structured_attributes
Lane 4: OCR_reference_context
Lane 5: visual
```

这样既保留 AutoBlock 的“multiple signatures”，又不会让模型越权。

#### Lane 0：Exact Reference Lane

输入：

```text
brand_id + canonical_reference
```

索引：hash / BTree / KV。

这是最强路径，不需要 ANN。

#### Lane 1：Reference Candidate Lexical Lane

输入：

```text
brand_id + raw_reference_candidate
```

用途：

- 分隔符差异；
- OCR 1/I、0/O 等疑似错字；
- 少量字符噪声；
- reference 版本写法差异。

注意：只用于找候选 canonical reference，最终仍需规则确认。

#### Lane 2：Semantic Title Lane

输入：

```text
去掉所有 code-like token 后的 title/description
```

例如：

```text
“Rolex Datejust blue dial oyster bracelet 41mm”
```

用途：在 ref 缺失时找同系列候选。

#### Lane 3：Structured Attribute Lane

输入字段：

```text
brand
series
collection
caliber
case_size
material
dial_color
movement
watch_type
```

不同字段覆盖稀疏，可以使用 AutoBlock 风格的 attribute encoder + multi-signature。

#### Lane 4：OCR Reference Context Lane

输入：

```text
OCR 识别出的 code token + 周围文本 + 图片类型
```

例如：

```text
image_type=caseback
context="ROLEX ... 126334 ..."
```

用途：标题没 reference，但表背、保卡、吊牌有 reference。

#### Lane 5：Visual Lane

图片 embedding 只能回答：

```text
“外观可能属于同一系列/近似变体”
```

不能回答：

```text
“reference 完全一致”
```

因为同系列不同 reference 可以几乎同外观。

因此 visual lane 的输出只能：

- 补召回；
- 在人工复核页排序；
- 做冲突提示；

绝不能单独触发 ACCEPT。

---

## 7. Step 4：Positive-only + Random Negative Training —— 原论文在腕表场景必须修改

AutoBlock 只需要 positive pair 标签。

对每个正对：

```text
(x_i, x_j) = match
```

再随机采若干其他 tuple 作为 irrelevant negatives。

论文每个 positive pair 在线随机采 10 个 irrelevant tuples。

训练目标是：

```text
positive pair similarity
>
positive-to-random-negative similarity
```

这个设计减少了人工负例标注。

### 7.1 为什么腕表不能只用 random negative

对普通数据：

```text
Rolex 126334
vs
Omega 210.30.42.20.03.001
```

这种随机负例太容易。

真正危险的是：

```text
Rolex 126334
vs
Rolex 126333
```

或者：

```text
Rolex 116610LN
vs
Rolex 126610LN
```

以及：

```text
商品 reference = 126334
标题里出现 compatible/strap for 126334
```

这类 near-reference hard negative 才是 precision-first 的核心。

如果只训练 random negative，模型极容易得到很漂亮的 loss，却完全学不会真正的“禁止边界”。

### 7.2 建议训练集组成

几百对人工黄金标签不要平均撒，而应该专门标注 hard cases。

建议构成：

#### Positive

- 同 brand + 同 canonical reference，跨来源；
- reference 字段 vs title 抽取；
- title vs OCR；
- 不同写法但同 canonical ref；
- 字段大量缺失但已被人工确认的同 ref。

#### Hard Negative A：同品牌相邻 reference

```text
126334 vs 126333
126610LN vs 116610LN
```

#### Hard Negative B：同系列不同 reference

文本和图片非常近，但 ref 不同。

#### Hard Negative C：编号角色错误

```text
平台 SKU
店铺库存号
序列号
订单号
reference of accessory target
```

#### Hard Negative D：OCR 混淆

```text
0/O
1/I/L
5/S
8/B
```

#### Hard Negative E：跨来源 schema 错位

比如 reference 被错误抓到 `sku`、`serial`、`description` 子字段。

### 7.3 训练目标建议

Blocking lane 可以继续使用 contrastive / InfoNCE 类目标，但 batch 中必须强制加入 hard negatives：

```text
loss = positive_loss
     + λ1 * same_brand_adjacent_ref_negative
     + λ2 * same_series_diff_ref_negative
     + λ3 * sku_role_negative
```

并且：

```text
semantic lane 训练时 mask reference token
reference lane 不使用 semantic embedding
```

这是防止“相邻型号被 embedding 拉近后误授权”的关键。

---

## 8. Step 5：Cross-Polytope LSH —— 思想可用，具体实现不建议照搬

AutoBlock 通过 cosine similarity 做 nearest neighbor search，然后用 cross-polytope LSH 加速。

论文使用 FALCONN，实验配置大致为：

- 300 维 signature；
- 10 个 hash tables；
- 每 table 2 个 hash functions；
- multi-probe 额外 probe 一个邻近 bucket；
- similarity threshold 0.8；
- 每 tuple 最大邻居数约为 `max(1000, sqrt(n_large))`。

论文在 `10^6` 点规模上报告：

- LSH 相对 brute-force 单核查询有约 40～80 倍加速；
- 在 Grocery 的 exact NN vs LSH 对比中，平均 recall gap 很小；
- 其三个百万级实验从 signature 计算到候选输出均能在较短时间完成。

### 8.1 但是不要照搬 FALCONN 参数

原因：

1. 论文是 2019/2020 年技术环境；
2. 当前数据量最高 1000 万，并且是持续增量；
3. 我们不是做单一 embedding，而是多 lane；
4. 候选必须带可审计的 provenance；
5. source / brand partition 可以大幅缩小检索空间，没必要全局 NN。

### 8.2 更适合当前项目的索引策略

建议：

```text
先 deterministic partition
再 ANN
```

优先 partition：

```text
brand_id
```

可继续二级 partition：

```text
watch/product category
series family（仅做候选，不做最终约束）
```

索引可使用支持增量 ANN 的成熟方案，例如：

- FAISS HNSW / IVF 系列；
- Milvus / Qdrant / Elasticsearch vector；
- 现有基础设施如果已经有向量数据库，则直接复用。

选择标准不是“论文复现”，而是：

```text
incremental insert
filter by brand
stable id mapping
snapshot/version
rebuild ability
auditability
```

---

## 9. 原论文实验结果应该怎么解读

论文三个数据集：

- Movie：结构化、较干净；
- Music：脏、字段错位；
- Grocery：非结构化长文本。

AutoBlock 在 Music / Grocery 上表现尤其好。

论文表格里，Music 上 AutoBlock Blocking recall 约 96.3%，P/E 约 48.8；Grocery 上 recall 约 89.1%，P/E 约 0.72。

这说明它确实擅长：

- dirty schema；
- missing fields；
- raw title；
- lexical mismatch；
- attribute misplacement。

这与三源二奢数据相当接近。

但不要错误解读为：

> “AutoBlock 可以给当前系统 96% 的实体匹配准确率。”

完全不是。

论文评的是：

```text
真实 pair 有没有进候选集
```

不是：

```text
候选 pair 是不是最终 match
```

而当前业务最关键的是 final precision。

所以 AutoBlock 结果最多证明：

> “这种候选生成方式值得尝试。”

不能证明：

> “这种 embedding 能代替 reference 身份规则。”

---

## 10. 当前 Spec 最应该做的架构反转：从 Record Pair Matching 改为 Reference Entity Linking

这是本次分析最重要的落地建议。

既然业务已经明确：

```text
same product == same reference
```

那么最终 entity 不应该是通过 record cluster 猜出来的，而应该显式建模：

```text
ReferenceEntity
```

主键：

```text
(brand_id, canonical_reference)
```

每条来源商品记录只是 link 到它。

### 10.1 数据关系

```text
SourceRecord
  ├─ 雷小安 item A
  ├─ 腕表之家 item B
  └─ 奢当家 item C
          │
          ▼
ReferenceEvidence
  ├─ structured field
  ├─ title extraction
  ├─ OCR
  ├─ manual
  └─ source rule
          │
          ▼
ReferenceEntity
  brand_id = ROLEX
  canonical_reference = 126334
```

最终跨源“同商品”自然变成：

```sql
SELECT *
FROM source_record
WHERE reference_entity_id = ?;
```

而不是每次进行三源 pairwise join。

### 10.2 规模优势

假设 1000 万 records，如果 pairwise：

```text
O(N²)
```

根本不可行。

Reference-Centric 后，正常路径是：

```text
record -> reference_entity
```

复杂度接近：

```text
O(N * extraction_cost)
+ O(uncertain_subset * ANN_cost)
```

AutoBlock 只服务 `uncertain_subset`。

---

## 11. 推荐的完整生产架构

```text
                    ┌──────────────────────────────┐
                    │ 雷小安 / 腕表之家 / 奢当家 │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                         Raw Ingestion Layer
                    raw payload + source_item_id
                                   │
                                   ▼
                       Normalization / Parsing
                    brand / title / attrs / images
                                   │
                                   ▼
                    Reference Candidate Extractor
                 structured + title + description + OCR
                                   │
                                   ▼
                       Reference Role Classifier
         brand_ref / platform_sku / serial / accessory_target / unknown
                                   │
                                   ▼
                    Canonical Reference Normalizer
                                   │
                   ┌───────────────┴─────────────────┐
                   │                                 │
              high confidence                    uncertain
                   │                                 │
                   ▼                                 ▼
        Exact Reference Resolver          Multi-Lane AutoBlock
      (brand, canonical_reference)     ref lexical / semantic / attr /
                   │                   OCR context / visual candidates
                   │                                 │
                   │                                 ▼
                   │                       Evidence Verifier
                   │                                 │
                   └──────────────┬──────────────────┘
                                  ▼
                         Precision Decision Gate
                       ACCEPT / REVIEW / REJECT
                                  │
                                  ▼
                         ReferenceEntity Link
                                  │
                                  ▼
                           Cross-source View
```

---

## 12. 建议的数据模型

### 12.1 `source_record`

```sql
CREATE TABLE source_record (
    id                  BIGINT PRIMARY KEY,
    source              VARCHAR(32) NOT NULL,
    source_item_id      VARCHAR(128) NOT NULL,
    raw_title           TEXT,
    raw_brand           TEXT,
    structured_ref      TEXT,
    raw_payload_uri     TEXT,
    ingest_time         TIMESTAMP NOT NULL,
    payload_hash        VARCHAR(64) NOT NULL,
    UNIQUE(source, source_item_id)
);
```

### 12.2 `normalized_record`

```sql
CREATE TABLE normalized_record (
    record_id           BIGINT PRIMARY KEY,
    brand_id            BIGINT,
    normalized_title    TEXT,
    series_id           BIGINT,
    caliber             TEXT,
    case_size_mm        DECIMAL(6,2),
    material            TEXT,
    dial_color          TEXT,
    category            TEXT,
    parser_version      VARCHAR(64),
    updated_at          TIMESTAMP NOT NULL
);
```

### 12.3 `reference_evidence`

任何 reference 结论必须保留 evidence，不允许只存一个最终字符串。

```sql
CREATE TABLE reference_evidence (
    id                      BIGINT PRIMARY KEY,
    record_id               BIGINT NOT NULL,
    evidence_source         VARCHAR(32) NOT NULL,
    raw_value               TEXT NOT NULL,
    normalized_candidate    TEXT,
    token_role              VARCHAR(32),
    role_confidence         DECIMAL(8,6),
    extraction_confidence   DECIMAL(8,6),
    span_start              INT,
    span_end                INT,
    image_id                BIGINT,
    extractor_version       VARCHAR(64) NOT NULL,
    created_at              TIMESTAMP NOT NULL
);
```

`evidence_source` 可以是：

```text
structured_field
title_regex
title_model
description
ocr_caseback
ocr_card
ocr_tag
manual
```

### 12.4 `reference_entity`

```sql
CREATE TABLE reference_entity (
    id                    BIGINT PRIMARY KEY,
    brand_id              BIGINT NOT NULL,
    canonical_reference   VARCHAR(128) NOT NULL,
    status                VARCHAR(16) NOT NULL,
    created_at            TIMESTAMP NOT NULL,
    UNIQUE(brand_id, canonical_reference)
);
```

### 12.5 `record_reference_link`

```sql
CREATE TABLE record_reference_link (
    record_id              BIGINT PRIMARY KEY,
    reference_entity_id    BIGINT,
    decision               VARCHAR(16) NOT NULL,
    decision_reason        VARCHAR(128) NOT NULL,
    decision_version       VARCHAR(64) NOT NULL,
    reviewed_by            VARCHAR(128),
    updated_at             TIMESTAMP NOT NULL
);
```

`decision`：

```text
ACCEPT
REVIEW
REJECT
UNRESOLVED
```

### 12.6 `blocking_candidate`

AutoBlock 层必须可审计：

```sql
CREATE TABLE blocking_candidate (
    record_id              BIGINT NOT NULL,
    candidate_type         VARCHAR(32) NOT NULL,
    candidate_entity_id    BIGINT,
    candidate_record_id    BIGINT,
    lane                   VARCHAR(32) NOT NULL,
    score                  DECIMAL(8,6),
    rank_no                INT,
    index_version          VARCHAR(64),
    created_at             TIMESTAMP NOT NULL
);
```

candidate 不能只存 pair；还必须存：

```text
为什么被召回
哪一条 lane
哪个模型版本
哪个 index snapshot
rank / score
```

否则误匹配事故很难复盘。

---

## 13. Reference 抽取与规范化：必须在 Blocking 之前优先做

因为“同商品 = 同 reference”，所以最有价值的模型预算应该优先给 reference extraction，而不是通用 entity matcher。

### 13.1 抽取顺序

推荐优先级：

```text
1. source-specific structured reference field
2. title code candidate
3. description/spec table
4. OCR on card/tag/caseback
5. semantic candidate retrieval
```

### 13.2 不要直接 regex 后 exact match

regex 只能找出“像编号”的 token：

```text
126334
M126334-0001
IW371605
Q1238420
```

还需要 role classifier：

```text
BRAND_REFERENCE
PLATFORM_SKU
SHOP_SKU
SERIAL_NUMBER
ACCESSORY_TARGET_REFERENCE
ORDER_ID
UNKNOWN_CODE
```

只有 `BRAND_REFERENCE` 才能进入身份规则。

### 13.3 Canonicalization 规则必须品牌化

不要全局做：

```python
remove_all_non_alphanumeric(ref)
```

因为不同品牌 reference 的：

- 点号；
- 横线；
- 前缀；
- 后缀；
- variation suffix；

可能有业务意义。

建议：

```text
canonicalize(brand_id, raw_reference)
```

而不是：

```text
canonicalize(raw_reference)
```

并维护：

```text
brand_reference_rule_version
```

---

## 14. Multi-Lane Blocking 的推荐实现

### 14.1 Lane 0：Exact Reference

```python
key = (brand_id, canonical_reference)
```

如果：

- brand 高置信；
- reference role = BRAND_REFERENCE；
- canonicalization 成功；
- 没有冲突 evidence；

就直接定位 reference entity。

这里不需要向量模型。

### 14.2 Lane 1：Reference Lexical Candidate

用于：

```text
OCR typo
分隔符差异
可能的输入错误
```

可以使用：

- character n-gram inverted index；
- edit-distance bounded search；
- BK-tree（小规模字典）；
- Elasticsearch fuzzy query；
- reference dictionary ANN（如果确实需要）。

但必须先：

```text
brand partition
```

防止跨品牌相似编号碰撞。

### 14.3 Lane 2：Semantic Title

把：

```text
brand token
platform marketing words
code-like tokens
```

做不同处理。

建议输入：

```text
[BRAND=ROLEX] datejust blue dial oyster 41mm
```

reference token全部 mask。

输出 top-K `reference_entity` 或代表性 records。

### 14.4 Lane 3：Structured Attribute Multi-Signature

显式构造几个 signature：

```text
sig_A = series + caliber
sig_B = series + case_size + movement
sig_C = material + dial + case_size
sig_D = source-specific normalized attrs
```

然后可以让模型学习 lane 内权重。

为什么不完全自动学习？

因为当前业务有很强的领域约束，完全自动可能把 identity-critical 属性错误降权。

### 14.5 Lane 4：OCR

OCR 不应该只产一个字符串。

需要保留：

```text
image_type
bbox
raw_text
normalized_code
OCR confidence
surrounding text
```

例如：

```text
126334
```

出现在：

```text
保卡
```

比出现在：

```text
商品背景里的盒子标签
```

证据强度不同。

### 14.6 Lane 5：Visual

建议只在：

- ref 全缺失；
- title 过短；
- OCR 失败；

时用于补召回。

不要把视觉 score 与 reference exact rule 加权求平均得到最终 match score。

错误做法：

```text
final_score = 0.5 * text + 0.3 * image + 0.2 * ref_string
if final_score > 0.9 -> match
```

因为这会允许强视觉相似度“补偿”reference 冲突。

正确做法是：

```text
reference conflict => hard reject
visual => only auxiliary evidence
```

---

## 15. Precision-first Decision Gate：最终必须是可解释规则，不是一个混合分数

建议最终输出三态：

```text
ACCEPT
REVIEW
REJECT
```

而不是二分类。

### 15.1 ACCEPT

建议至少满足：

```text
brand canonical 一致
AND
至少一个高可信 BRAND_REFERENCE evidence
AND
双方 canonical_reference exact equal
AND
没有任何高可信冲突 reference evidence
AND
没有 accessory / SKU / serial 角色冲突
```

如果是 candidate entity linking：

```text
record canonical_reference == entity canonical_reference
```

### 15.2 REJECT

以下任一成立直接拒绝：

```text
高可信 brand 冲突
高可信 canonical_reference 不同
reference role = PLATFORM_SKU / SERIAL
商品类型冲突（watch vs strap/box/card/accessory）
同 title 中有多个互斥 reference 且无法消解
```

### 15.3 REVIEW

包括：

```text
只有 semantic/visual candidate，没有 exact ref
OCR 疑似一个字符错
structured ref 与 title ref 冲突
reference role unknown
新品牌/新格式
```

### 15.4 关键原则

```text
模型只能把 UNRESOLVED 推到 REVIEW
不能靠相似度把 UNRESOLVED 直接推到 ACCEPT
```

这条非常符合“绝不能误匹配”的要求。

---

## 16. 决策伪代码

```python
def decide(record, candidate_entity, evidences):
    # 1. brand hard gate
    if record.brand_confident and candidate_entity.brand_id != record.brand_id:
        return "REJECT", "BRAND_CONFLICT"

    refs = [e for e in evidences if e.role == "BRAND_REFERENCE"]
    strong_refs = [e for e in refs if e.confidence >= STRONG_REF_THRESHOLD]

    # 2. evidence conflict hard gate
    strong_values = {e.canonical_reference for e in strong_refs}
    if len(strong_values) > 1:
        return "REVIEW", "MULTIPLE_STRONG_REF_CONFLICT"

    # 3. exact reference path
    if len(strong_values) == 1:
        ref = next(iter(strong_values))
        if ref == candidate_entity.canonical_reference:
            if has_forbidden_role_conflict(evidences):
                return "REJECT", "REFERENCE_ROLE_CONFLICT"
            return "ACCEPT", "EXACT_CANONICAL_REFERENCE"
        else:
            return "REJECT", "CANONICAL_REFERENCE_CONFLICT"

    # 4. no strong reference => ANN cannot authorize
    if has_only_semantic_or_visual_candidate(record, candidate_entity):
        return "REVIEW", "NO_IDENTITY_GRADE_REFERENCE"

    return "UNRESOLVED", "INSUFFICIENT_EVIDENCE"
```

AutoBlock 的 ANN score 甚至不应该出现在 `ACCEPT` 条件里。

---

## 17. 增量架构：千万级持续更新时怎么跑

当前不是一次性离线 dedupe，而是持续增量。

建议 event-driven：

```text
source_record_upsert
    │
    ▼
normalize_record
    │
    ▼
extract_reference_evidence
    │
    ├─ exact success -> resolve_reference_entity
    │
    └─ uncertain -> enqueue_blocking
                       │
                       ▼
                  multi-lane ANN
                       │
                       ▼
                   verify evidence
                       │
                       ▼
                  decision event
```

### 17.1 每次更新要幂等

主幂等键：

```text
(source, source_item_id, payload_hash)
```

如果 payload 没变化，不重新做昂贵模型推理。

### 17.2 模型版本化

每条结果记录：

```text
parser_version
extractor_version
role_classifier_version
embedding_version
index_version
decision_version
```

这样升级模型后可以：

```text
只重算需要重算的阶段
```

### 17.3 Index 更新

不要每条商品更新都重建全量 ANN。

推荐：

```text
实时/准实时增量 index
+
周期性 compact/rebuild snapshot
```

并保留旧 index snapshot 一段时间，便于复盘。

---

## 18. 如何控制 1000 万级候选数量

AutoBlock 原论文的 P/E ratio 是很有用的思想。

当前项目更应该监控：

```text
Candidates Per Uncertain Record (CPUR)
```

例如不要全量每条 record top-100。

建议：

```text
exact reference route: 0 ANN candidates
uncertain reference route: top 3~10
no reference route: top 5~20
```

具体 K 用黄金标签调。

最重要的是：

> **只有 uncertain subset 才做 ANN。**

如果 70% records 能通过结构化/title reference 直接解析，就完全没有必要让这 70% 参与语义 NN。

这样千万级系统的主要复杂度会从：

```text
10M records × global ANN
```

降到：

```text
uncertain subset × brand-local ANN
```

而且候选最好优先指向 `reference_entity`，不是所有历史 records。

### 18.1 Reference Prototype

每个 `reference_entity` 可以维护 prototype：

```text
canonical title tokens
known aliases
common series
known calibers
representative image embeddings
known OCR forms
```

然后：

```text
uncertain record -> reference prototypes
```

比：

```text
uncertain record -> 10M raw records
```

更稳定，也更省。

---

## 19. 用几百对人工标签，怎样最划算

Spec 允许人工标几百对，这其实足够建立一个非常有价值的安全集，但前提是不要随机抽。

### 19.1 50% 标 hard negatives

优先：

- 同品牌同系列不同 ref；
- 相邻数字 reference；
- 同外观不同 ref；
- 平台 SKU vs brand ref；
- 配件标题包含目标手表 ref；
- OCR 一字符混淆。

### 19.2 30% 标 positive sparse cases

- 一边结构化 ref，一边 title；
- 一边 title，一边 OCR；
- 字段错位；
- 只有部分字段重合；
- 中英文/别名混合。

### 19.3 20% 留作完全冻结验收集

不参与：

```text
training
threshold tuning
prompt tuning
```

只用于最后验收。

否则很容易把阈值调到测试集上。

---

## 20. 评估指标：不要再以 F1 为主

当前业务目标不是平均 F1。

### 20.1 Blocking 层

测：

```text
candidate_recall
candidates_per_record
candidate_count
latency
index_size
```

必须分桶：

```text
structured ref present
ref only in title
ref only in OCR
no ref
same-series hard negatives
new unseen references
```

### 20.2 Final Decision 层

主指标：

```text
precision(ACCEPT)
false_accept_count
coverage(ACCEPT)
review_rate
```

因为业务允许漏匹配，建议把：

```text
false_accept_count
```

直接作为 release blocker。

### 20.3 安全验收

建议不是问：

```text
F1 有没有 95%
```

而是问：

```text
在冻结 hard-negative gold set 上，是否出现任何自动 ACCEPT 的 false positive？
```

如果有，就不应该发布当前 Gate。

---

## 21. Shadow Mode 是必须的

上线前不要直接写实体关系。

第一阶段：

```text
系统只生成 candidate + proposed decision
不修改生产 reference_entity link
```

人工对比：

```text
current mapping
new proposed mapping
reason/evidence
```

重点看：

- false accept；
- conflict rate；
- 哪条 lane 召回了错误候选；
- 哪种 source schema 最脏；
- 哪些品牌 canonicalization 最不稳定。

跑到 hard-negative false accept 足够低后，再只开放最强 ACCEPT path。

---

## 22. 分阶段上线方案

### Phase 0：Reference Dictionary + Audit

先不做任何 ANN。

交付：

- canonical brand；
- canonical reference 规则；
- `reference_evidence`；
- `reference_entity`；
- exact match；
- 冲突审计。

这是收益最高、风险最低的一步。

### Phase 1：Title / Structured Reference Extraction

加入：

- code candidate extractor；
- token role classifier；
- 品牌化 canonicalization；
- ACCEPT / REVIEW / REJECT gate。

### Phase 2：Reference Lexical Candidate Lane

只处理：

- OCR typo；
- separator / formatting；
- 轻微字符串噪声。

仍不做通用 semantic ANN。

### Phase 3：AutoBlock-style Semantic / Attribute Lanes

只处理 unresolved records。

实现：

- code-mask semantic title embedding；
- structured multi-signature；
- brand partition ANN；
- top-K 候选；
- provenance。

### Phase 4：OCR + Visual Auxiliary Lanes

图片进入：

- OCR evidence；
- visual candidate recall；
-人工复核 UI。

仍然不允许视觉直接授权 identity。

### Phase 5：持续学习

把人工 review 结果回流：

- role classifier；
- reference extractor；
- semantic blocking hard-negative mining；
- brand canonicalization exception rules。

---

## 23. AutoBlock 原设计中不建议直接照搬的地方

### 23.1 fastText 直接编码 reference

原因：相邻 reference character overlap 太高。

改：

```text
reference lane 与 semantic lane 分离
```

### 23.2 Random negatives

原因：太简单，不覆盖真实 false positive。

改：

```text
same brand / same series / adjacent reference hard-negative mining
```

### 23.3 Max-over-signatures 直接拿来做 match

原因：任意 lane 高相似就会被放行。

改：

```text
max-over-signatures 只生成 candidate union
final decision 走 hard gates
```

### 23.4 全局 ANN

原因：品牌是强先验，跨品牌搜索浪费且增加碰撞。

改：

```text
brand partition first
```

### 23.5 tuple-to-tuple 全历史搜索

原因：1000 万 raw records 搜索成本和重复结果都高。

改：

```text
优先 record -> reference_entity prototype
```

### 23.6 模型自动学习所有属性权重

原因：身份关键字段不能被模型任意降权。

改：

```text
显式 lane + lane 内学习
```

---

## 24. 可以直接落地的服务拆分

建议先保持简单，不需要一开始做微服务大拆分。

### 24.1 `normalizer`

API：

```json
{
  "source": "leixiaoan",
  "source_item_id": "...",
  "title": "...",
  "attrs": {}
}
```

输出：

```json
{
  "brand_id": 12,
  "normalized_title": "...",
  "attrs": {},
  "code_candidates": []
}
```

### 24.2 `reference-extractor`

输出：

```json
{
  "evidences": [
    {
      "raw": "126334",
      "role": "BRAND_REFERENCE",
      "canonical": "126334",
      "source": "TITLE",
      "confidence": 0.998
    }
  ]
}
```

### 24.3 `candidate-retriever`

输入：

```json
{
  "record_id": 123,
  "brand_id": 12,
  "normalized_record": {}
}
```

输出：

```json
{
  "candidates": [
    {
      "reference_entity_id": 999,
      "lane": "SEMANTIC_TITLE",
      "score": 0.91,
      "rank": 1
    }
  ]
}
```

### 24.4 `decision-gate`

输出：

```json
{
  "decision": "REVIEW",
  "reason": "NO_IDENTITY_GRADE_REFERENCE",
  "candidate_reference_entity_id": 999
}
```

这比一个黑盒：

```json
{"match_probability": 0.97}
```

安全得多，也更容易审计。

---

## 25. 可直接实现的 Blocking 代码骨架

```python
class CandidateRetriever:
    def __init__(self, exact_ref_index, ref_lexical_index,
                 semantic_index, attr_index, ocr_index, visual_index):
        self.exact_ref_index = exact_ref_index
        self.ref_lexical_index = ref_lexical_index
        self.semantic_index = semantic_index
        self.attr_index = attr_index
        self.ocr_index = ocr_index
        self.visual_index = visual_index

    def retrieve(self, record):
        out = []

        # 1. strongest deterministic route
        if record.strong_canonical_ref:
            entity = self.exact_ref_index.get(
                (record.brand_id, record.strong_canonical_ref)
            )
            if entity:
                out.append((entity, "EXACT_REFERENCE", 1.0))
                return out

        # 2. all fuzzy/semantic routes remain candidates only
        if record.ref_candidates:
            out += self.ref_lexical_index.search(
                brand_id=record.brand_id,
                refs=record.ref_candidates,
                k=5,
            )

        out += self.semantic_index.search(
            brand_id=record.brand_id,
            vector=record.semantic_vector,
            k=10,
        )

        out += self.attr_index.search(
            brand_id=record.brand_id,
            signatures=record.attr_signatures,
            k=10,
        )

        if record.ocr_vector is not None:
            out += self.ocr_index.search(
                brand_id=record.brand_id,
                vector=record.ocr_vector,
                k=5,
            )

        if record.visual_vector is not None:
            out += self.visual_index.search(
                brand_id=record.brand_id,
                vector=record.visual_vector,
                k=5,
            )

        return dedupe_keep_provenance(out)
```

`dedupe_keep_provenance()` 不应该只保留最大 score，而应保留：

```text
entity 999:
  exact_lexical: 0.88
  semantic_title: 0.92
  attr_signature_2: 0.84
  visual: 0.95
```

因为后面的 verifier 需要知道候选来源。

---

## 26. Hard Negative Mining 可以从线上候选直接产生

上线 shadow mode 后，可以自动构造最有价值的 hard negatives：

```text
ANN top-1 很高
BUT
canonical_reference conflict
```

例如：

```text
semantic sim = 0.97
visual sim = 0.99
brand same
series same
canonical ref different
```

这就是绝佳 hard negative。

训练时应高权重采样。

这样系统会持续学会：

> 外观和文本再像，只要 identity reference 不同，就不能把它们当同一实体。

这与普通 product matching 训练目标非常不同。

---

## 27. 监控项

### 27.1 Reference 抽取

```text
ref_candidate_rate
strong_ref_rate
multi_ref_conflict_rate
role_unknown_rate
ocr_ref_rate
```

### 27.2 Blocking

```text
uncertain_record_rate
candidate_recall_on_gold
avg_candidates_per_record
P95 candidates per record
lane_hit_rate
lane_overlap_rate
```

### 27.3 Decision

```text
accept_rate
review_rate
reject_rate
unresolved_rate
false_accept_on_gold
reference_conflict_count
```

### 27.4 Drift

按：

```text
source
brand
week
extractor_version
```

监控。

如果某品牌突然：

```text
role_unknown_rate ↑
ref extraction rate ↓
```

很可能是来源格式变化或新 reference pattern 上线。

---

## 28. 对“图片可用”的具体建议

图片在这个需求里有价值，但不能喧宾夺主。

推荐证据层级：

```text
一级：结构化可靠 reference
二级：title/description reference + role/canonical validation
三级：OCR reference
四级：semantic attributes
五级：visual similarity
```

图片最有价值的不是“看起来像不像同一块表”，而是：

```text
从保卡/表背/吊牌恢复 reference
```

所以图片工程优先级应该是：

```text
image classification -> identify card/caseback/tag
              -> OCR
              -> code candidate extraction
              -> role classification
```

而不是先训练一个巨大的 image matcher。

---

## 29. 与 AutoBlock 思路最一致的最终系统边界

AutoBlock 原论文其实已经隐含一个非常正确的工程边界：

> Blocking 是为了缩小 downstream matching 的工作量。

把它放到当前项目，就是：

```text
AutoBlock-style retrieval
≠
Reference identity decision
```

如果强行把两者合并，会出现：

```text
同系列不同 reference
文本高度相似
图片高度相似
ANN top-1 极高
=> 错误自动合并
```

而拆开后：

```text
ANN top-1 极高
=> 找到候选
=> 再检查 exact canonical reference
=> ref 不同
=> REJECT
```

这样既得到大规模可扩展性，又保住 precision-first 约束。

---

## 30. 最终推荐方案

我建议直接落地以下架构，而不是复现 AutoBlock 原论文代码：

### 核心实体

```text
ReferenceEntity = (brand_id, canonical_reference)
```

### 主路径

```text
SourceRecord
  -> reference extraction
  -> role validation
  -> canonicalization
  -> exact ReferenceEntity lookup
```

### 兜底路径

```text
Uncertain Record
  -> AutoBlock-inspired multi-lane signatures
  -> brand-local ANN candidate retrieval
  -> exact reference evidence verifier
  -> ACCEPT / REVIEW / REJECT
```

### 强制安全规则

```text
1. ANN score 永远不能直接触发 ACCEPT
2. visual score 永远不能覆盖 reference conflict
3. strong canonical reference 不同 => REJECT
4. strong canonical reference 相同 + role 合法 + brand 一致 => 才允许 ACCEPT
5. 所有不够 identity-grade 的证据 => REVIEW / UNRESOLVED
```

### 技术迁移表

| AutoBlock 原设计 | 当前项目建议 |
|---|---|
| fastText 全 token | code/semantic 分轨，reference 不走语义身份判断 |
| attention attribute encoder | 保留，用于候选表示 |
| multiple learned signatures | 改为显式 multi-lane + lane 内学习 |
| random negatives | 加 same-series / adjacent-reference hard negatives |
| max cosine over signatures | 只用于 candidate union |
| cross-polytope LSH | 换成支持增量和过滤的 ANN 基础设施 |
| tuple-to-tuple | 优先 record-to-reference-entity prototype |
| blocking recall/P-E | 增加 final precision / false accept / review rate |

---

## 31. 一句话总结

AutoBlock 真正值得当前需求借鉴的不是“学一个更强的相似度模型”，而是：

> **把千万级数据的 pairwise matching 问题拆成多路、可扩展、只负责召回的近邻搜索；然后把身份判定从相似度模型中拿出来，重新交还给可审计的 canonical reference 硬证据。**

如果这样使用，AutoBlock 可以成为整个系统的“规模扩展器”；如果把它当最终 matcher，它反而会放大同系列相邻 reference 的 false positive 风险。
