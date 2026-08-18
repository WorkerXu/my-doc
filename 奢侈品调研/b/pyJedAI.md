# pyJedAI：把端到端实体解析框架改造成“Reference 硬判定 + 可拒识候选实验台”的三源二奢匹配方案

- 分析人：b
- 调研项目：https://github.com/AI-team-UoA/pyJedAI
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31
- 代码快照：`AI-team-UoA/pyJedAI@eb446095f921af3207bbefd570cc65e818194419`
- 调研日期：2026-08-18

## 1. 结论先行

pyJedAI 很适合当前需求，但**适合的位置不是线上最终“是否同一商品”的裁判，而是离线 ER 实验、候选生成和评测骨架**。

当前 Spec 的业务定义其实非常强：

> “同一个商品” = **同一个 reference number / 型号**，且绝对不能误匹配，precision 优先到极致，可以漏匹配。

在这个定义下，最终身份判定不应该是：

```text
text_similarity > 0.92
or image_similarity > 0.95
or entity_matcher.predict() == match
```

而应该是：

```text
trusted_brand_id 相同
AND trusted_canonical_reference 相同
AND 没有强冲突证据
```

因此推荐把系统拆成两条完全不同权限的通道：

1. **确定性通道（authoritative lane）**：拿到可信 reference 后，通过 `(brand_id, strict_reference_key)` 做哈希/唯一索引归组；这是唯一允许自动形成跨源实体关系的通道。
2. **不确定性通道（discovery lane）**：当 reference 缺失、含糊、埋在标题或图片里时，才使用 pyJedAI 风格的 Blocking / Meta-blocking / FAISS / 文本相似等找候选；候选分数只能帮助提取、复核、排序或发现冲突，**不能越权成为最终 match 依据**。

这比“把 pyJedAI 的 Blocking → Matching → Clustering 原样跑到底”更符合业务约束，也更适合 100 万～1000 万级持续增量数据。

---

## 2. 为什么这次选择 pyJedAI

`奢侈品文章调研.md` 对 pyJedAI 的推荐理由是：它覆盖完整的 `Blocking → Comparison Cleaning → Entity Matching → Clustering` 工作流，可以统一比较候选召回、计算量与误匹配。

这次分析前已检查 `奢侈品调研/b/`，b 已分析：

- Ameli
- DeepBlocker
- Ditto
- TransClean
- parts-distributor-sku-classifier

pyJedAI 尚未分析，因此符合“每次先排除已分析文章/项目”的要求。

它与前几篇研究的关系也比较清晰：

- `parts-distributor-sku-classifier` 解决“像型号的字符串到底是什么编号角色”；
- `DeepBlocker` 解决“全量笛卡尔积太大时怎样只召回候选”；
- `Ditto / Ameli` 更偏 pairwise matcher；
- **pyJedAI 的价值是把 Blocking、候选清洗、匹配、聚类、评测统一成可插拔工作流，适合作为我们自己的 ER 实验台骨架。**

---

## 3. pyJedAI 当前代码的真实架构

当前主分支代码不是一个单独 matcher，而是一套模块化 Entity Resolution 工具箱。

核心目录大致是：

```text
src/pyjedai/
├── datamodel.py
├── block_building.py
├── block_cleaning.py
├── comparison_cleaning.py
├── vector_based_blocking.py
├── matching.py
├── clustering.py
├── evaluation.py
├── workflow.py
└── ...
```

`workflow.py` 中的 `BlockingBasedWorkFlow` 把这些模块串成：

```text
Data
  │
  ▼
Block Building
  │
  ▼
Block Cleaning（可选，多步）
  │
  ▼
Comparison Cleaning / Meta-blocking（可选）
  │
  ▼
Entity Matching
  │
  ▼
Clustering（可选）
  │
  ▼
Pairs / Clusters + Evaluation
```

框架会记录每一步的 Precision、Recall、F1 和 Runtime，这对于我们比较不同候选生成策略非常方便。

### 3.1 Data：pandas-first 的两表 ER 数据模型

`datamodel.py` 的 `Data` 主要接收：

```python
Data(
    dataset_1,
    id_column_name_1,
    attributes_1=None,
    dataset_2=None,
    attributes_2=None,
    id_column_name_2=None,
    ground_truth=None,
)
```

其重要实现特点：

- 输入要求是 pandas DataFrame；
- 缺失值填成空字符串；
- 数据整体转成字符串；
- 在内部把业务 ID 映射为连续整数；
- `dataset_2 is None` 时视为 Dirty ER，否则视为 Clean-Clean ER；
- 原生抽象核心仍然是“一张表内部”或“两张表之间”的 ER。

这意味着：**三来源雷小安 × 腕表之家 × 奢当家并不是 pyJedAI 原生的一次 N-source 任务。**

如果拿它做实验，最直接是分别跑：

```text
雷小安 ↔ 腕表之家
雷小安 ↔ 奢当家
腕表之家 ↔ 奢当家
```

但生产系统不应该把这三批 pair 当作最终实体图再传递闭包，而应在外部维护一个全局 reference identity。

---

## 4. 一个非常关键的代码风险：默认清洗会破坏 Reference

`Data.clean_dataset()` 当前默认参数是：

```python
remove_stopwords=True
remove_punctuation=True
remove_numbers=True
remove_unicodes=True
```

代码会在指定属性列中执行类似：

```python
re.sub(r'\d+', '', x)
re.sub(r'[^\w\s]', '', x)
```

这对普通自然语言相似度可能有意义，但对本项目是**高危默认行为**。

reference 往往就是字母、数字、斜杠、短横线等组合，例如：

```text
126334
5711/1A-010
```

如果删除数字或无差别删除标点，reference 的身份语义会被直接摧毁。

所以本项目必须立一条硬规则：

> **任何 reference 字段、reference 候选 span、OCR reference 都禁止经过通用 stopword / remove-number / remove-punctuation 清洗。**

应单独做 `ReferenceCanonicalizer`，而不是复用普通文本 cleaner。

当前 pyJedAI 最新主分支提交已经修复过“清洗时不应破坏 identifier column 与内部映射一致性”的问题，这也侧面说明：对 ID / identifier 类字段，通用清洗必须非常谨慎。

---

## 5. Blocking：pyJedAI 最值得直接借鉴的部分

### 5.1 Standard Blocking

`StandardBlocking` 会把选中的字段按行拼接，再做 tokenization：

```text
selected attributes
  -> " ".join(row)
  -> lowercase
  -> re.split('[\W_]', text)
  -> token -> entity set
```

每个 token 形成一个 block，只有一条记录的 block 会被丢弃。

优点：

- 实现简单；
- 很适合快速建立 baseline；
- 可以把全量比较压缩成局部比较。

但腕表场景不能把它当 reference matcher，因为：

- 标题里大量营销词会产生大 block；
- 同系列不同 reference 的文本高度相似；
- punctuation split 可能把一个结构化 reference 拆成多个 token；
- 平台 SKU、库存号也会形成“看起来很强”的 token。

所以 Standard Blocking 只能承担“找可能相关记录”的职责。

### 5.2 Q-Gram / Suffix Blocking

pyJedAI 还提供 Q-Gram、Suffix Array 等 Blocking。

这对“reference 埋在标题里而且格式不统一”的长尾数据有价值，例如同一串字符出现了：

```text
AB-1234
AB1234
AB 1234
```

Q-Gram 可以帮助把它们召回到同一候选空间。

但需要强调：

```text
AB1234 ≈ AB1235
```

在字符空间也会高度相似，业务上却必须视为不同 reference。

因此 q-gram 只做召回，不做授权。

### 5.3 Ray 并行不是“千万级生产架构”的充分条件

`AbstractBlockBuilding` 支持 Ray batch tokenization，但最终核心结构仍然是 Python 层的：

```text
dict[token] -> Block
Block -> Python set(entity_id)
```

并且 serial/ray 两条路径都会把实体 tokenization 结果和 block collection 物化到内存。

1000 万级商品如果标题 token 丰富，Python dict/set 的对象开销会非常可观。

因此：

- pyJedAI 可以在抽样数据、品牌分区或小规模离线实验中直接运行；
- 生产候选索引应使用更适合磁盘/分布式/紧凑索引的数据结构；
- 对“可信 reference”根本不需要 Blocking，直接做 key lookup 即可。

---

## 6. Block Cleaning / Comparison Cleaning：很适合压缩候选，但不能承担身份判断

### 6.1 BlockFiltering

`BlockFiltering(ratio=...)` 的核心逻辑是：

1. 按 block cardinality 从小到大排序；
2. 建 entity → blocks 的倒排；
3. 每个实体只保留一定比例的小 block；
4. 重建 blocks。

它隐含了一个合理经验：**越小、越具体的 block 通常越有区分力。**

在腕表里可对应：

- 品牌 block 很大，区分力弱；
- 系列 block 中等；
- 稳定 reference 片段、稀有型号 token 的 block 很小，区分力更强。

这个思想值得迁移到候选召回层。

### 6.2 BlockPurging

`BlockPurging` 会估计每个 block 可接受的最大 comparisons，然后抛弃过大的 block。

在本项目里可以直接对应：

- `Rolex`
- `二手`
- `自动机械`
- `男表`

这类词形成的 block 可能太大，应在 candidate generation 阶段被降权或直接丢弃。

### 6.3 WeightedEdgePruning / Meta-blocking

`comparison_cleaning.py` 提供多个 weighting scheme，例如：

- CBS
- COSINE
- DICE
- JS
- EJS
- ECBS
- X2

`WeightedEdgePruning` 等方法根据两个实体共同出现在哪些 blocks 中，为候选边赋权并剪枝。

它非常适合回答：

> “哪些候选 pair 值得送去更昂贵的 reference extractor / OCR / 人工复核？”

但它不应该回答：

> “这两个商品是不是同一个 reference？”

因为 block overlap 本质仍然是软相似证据，无法满足零误匹配约束。

---

## 7. EntityMatching：作为 Review Ranker 可以，作为最终判定器不行

`matching.py` 的 `EntityMatching` 支持：

- edit distance
- Jaro
- cosine
- Jaccard
- Dice
- generalized Jaccard
- overlap coefficient
- char / word q-gram tokenizer
- TF / TF-IDF / boolean vectorizer
- 指定属性列表或属性权重

输出是一个 NetworkX `Graph`：

```text
entity A ---- similarity weight ---- entity B
```

超过 `similarity_threshold` 的边被保留。

对于普通 ER，这是标准做法；对于当前业务，有两个严重风险。

### 7.1 相邻 Reference 字符串天然相似，但身份完全不同

例如只差 1 位、1 个 suffix 的型号，在 edit distance / q-gram / cosine 上可能非常接近。

如果把“高 similarity”当 match，会刚好命中本需求最不能接受的 false positive 类型。

### 7.2 全字段拼接可能让强噪声盖过身份字段

如果标题、品牌、系列、年份、材质都一致，而 reference 不同，普通 matcher 仍可能给出很高分。

但本项目的业务定义是：**reference 不同就不是同一个商品。**

因此我们的生产规则必须是：

```text
soft_similarity_score
    -> 只能影响候选优先级 / 人工复核顺序
    -> 不能 create identity edge
```

---

## 8. Clustering：pyJedAI 的默认 ER 聚类思想与本业务有结构性冲突

### 8.1 Connected Components 的“单错边污染整簇”风险

`ConnectedComponentsClustering` 会对相似度图做连通分量。

如果：

```text
A -- B  真匹配
B -- C  错匹配
```

那么 connected component 会把：

```text
{A, B, C}
```

整体连起来。

对于普通 ER，这种 transitive closure 很常见；对于“绝不能误匹配”的当前需求，一条错边可能污染整个实体簇。

所以**禁止对 fuzzy edges 直接做 connected-components 自动合并**。

### 8.2 UniqueMappingClustering 也不适合“同 Reference 多 listing”

`UniqueMappingClustering` 面向 Clean-Clean ER，会按高权重边贪心地做 one-to-one mapping：一个实体匹配后，不再参与其他匹配。

但二手电商中，同一 reference 可能在同一个平台就有多条不同卖家的 listing，也可能存在历史/重复抓取记录：

```text
同一 reference
├── 雷小安 listing 1
├── 雷小安 listing 2
├── 腕表之家 listing 1
└── 奢当家 listing 1
```

这不是一对一关系，而是：

```text
many product records -> one reference identity
```

因此最终实体层不应采用 `UniqueMappingClustering`，而应采用确定性的 reference group。

---

## 9. Vector Blocking：适合 unresolved tail，不能替代 Reference

`vector_based_blocking.py` 的 `EmbeddingsNNBlockBuilding` 提供：

```text
商品文本
  -> embedding
  -> FAISS nearest-neighbor search
  -> Top-K candidates
```

可用的 representation 包括多种 word / Transformer / sentence embedding，并支持 FAISS 搜索。

这非常适合处理：

- reference 完全缺失；
- 标题很脏；
- 新来源 schema 没接好；
- 只知道品牌、系列、材质、尺寸等弱信息；
- 需要找若干可能候选给 OCR 或人工审核。

推荐把它放到：

```text
Reference Resolver = MISSING / AMBIGUOUS
                      │
                      ▼
             candidate retrieval only
```

而不是：

```text
FAISS top-1 == same product   # 禁止
```

图片同理：视觉近似可以找候选，不能越过 reference rule。

---

# 10. 针对当前 Spec 的直接落地架构

## 10.1 总体架构

推荐生产主链路：

```text
雷小安          腕表之家          奢当家
  │               │               │
  └───────┬───────┴───────┬───────┘
          ▼               ▼
       Raw Ingestion / Immutable Snapshot
          │
          ▼
 Source Adapter + Field Normalization
          │
          ├───────────────┐
          ▼               ▼
 Brand Canonicalizer   Image/OCR Pipeline
          │               │
          └───────┬───────┘
                  ▼
        Reference Candidate Extractor
       ┌──────────┼──────────┐
       │          │          │
 structured    title/desc    OCR/image text
       │          │          │
       └──────────┼──────────┘
                  ▼
        Reference Role Classifier
 manufacturer_ref / platform_sku /
 seller_sku / compatible_ref / unknown
                  │
                  ▼
          Brand-aware Canonicalizer
                  │
                  ▼
          Reference Resolver + Veto
        ┌─────────┴───────────┐
        │                     │
     TRUSTED             MISSING / AMBIGUOUS /
        │                 CONFLICT
        ▼                     │
 Strict Reference Key         ▼
(brand_id, strict_ref)   Candidate Retrieval
        │                (pyJedAI-inspired)
        ▼                     │
Reference Identity Store      ├─ token/qgram blocking
        │                     ├─ block cleaning
        ▼                     ├─ FAISS top-K
 Group Membership             └─ soft ranker
        │                     │
        ▼                     ▼
 automatic match         review / remain unmatched
```

这里最关键的架构原则是：

> **只有左边 TRUSTED 通道可以写“实体归属”；右边不确定通道只能产生 candidate / evidence / review task。**

这从系统权限上消除了“相似度模型偶尔误判却直接污染实体图”的风险。

---

## 10.2 Reference Evidence：不要只保存一个抽取字符串

每次抽到一个“像 reference”的值，建议保存完整证据：

```json
{
  "record_id": "watchhome:123456",
  "raw_value": "5711/1A-010",
  "canonical_value": "...",
  "role": "manufacturer_reference",
  "evidence_source": "title",
  "span_start": 18,
  "span_end": 29,
  "image_id": null,
  "extractor_version": "ref-ext-v3",
  "canonicalizer_version": "ref-can-v2",
  "brand_grammar_valid": true,
  "confidence": 0.998,
  "conflict_flags": []
}
```

推荐 evidence source 至少区分：

- `structured_reference`
- `title`
- `description`
- `spec_table`
- `image_ocr_caseback`
- `image_ocr_card`
- `image_ocr_tag`

这样发生争议时可以回答：

> “为什么这条记录被判成这个 reference？”

并支持 parser 升级后的重放与回滚。

---

## 10.3 先做“编号角色分类”，再谈 Canonical Reference

二奢数据最危险的错误之一不是“没抽到编号”，而是**抽到了错误角色的编号**。

标题里可能同时出现：

```text
品牌 reference
平台商品 ID
店铺 SKU
库存号
保卡号
序列号
兼容表款 reference
配件自身编号
```

所以建议 Reference Candidate Extractor 后面有一层：

```text
ReferenceRoleClassifier(candidate, context)
```

输出：

```text
MANUFACTURER_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
ACCESSORY_COMPATIBLE_REFERENCE
UNKNOWN
```

只有 `MANUFACTURER_REFERENCE` 才能进入 identity resolver。

对于“表带适配 126334”这种配件标题，即使 reference 字符串完全正确，也不能把这条配件商品归到 `126334` 腕表实体。

---

## 10.4 Canonicalization 必须 brand-aware，而且宁可不归一也不要错归一

建议分三层保存：

```text
raw_reference
normalized_display_reference
strict_reference_key
```

`normalized_display_reference` 可以做相对安全的格式统一，例如：

- Unicode NFKC；
- 全角/半角统一；
- 大小写统一；
- 去首尾空白；
- 统一明显的排版空格。

但 `strict_reference_key` 不能简单：

```python
re.sub(r'[^A-Z0-9]', '', ref)
```

因为 `/`、`-`、suffix、字母位置等在不同品牌中**可能**具有语义。

推荐：

```text
BrandReferenceParser
├── rolex_parser
├── patek_parser
├── omega_parser
├── cartier_parser
└── generic_conservative_parser
```

每个 parser 只做已经验证不会改变型号身份的转换。

对于未知品牌/未知格式：

```text
宁可 AMBIGUOUS
不要 aggressive normalize
```

这正符合 precision-first。

---

## 10.5 Trusted Reference 的自动放行规则

可以把 resolver 做成显式 rule engine，而不是一个不可解释的单分数模型。

伪代码：

```python
def resolve_reference(record, evidences, brand_id):
    strong = [
        e for e in evidences
        if e.role == "MANUFACTURER_REFERENCE"
        and e.brand_grammar_valid
    ]

    if has_strong_conflict(strong):
        return Resolution.CONFLICT

    keys = unique_strict_keys(strong)

    if len(keys) != 1:
        return Resolution.AMBIGUOUS if keys else Resolution.MISSING

    key = only(keys)

    if violates_product_type_rule(record, key):
        return Resolution.CONFLICT

    if not passes_trust_policy(strong):
        return Resolution.AMBIGUOUS

    return Resolution.TRUSTED(key)
```

`passes_trust_policy` 可以逐步开放：

### 第一阶段：最保守

只接受：

```text
来源明确的结构化 reference 字段
+ brand parser 合法
+ 没有任何强冲突证据
```

### 第二阶段：扩大覆盖

允许：

```text
标题抽取出的 manufacturer_reference
+ brand grammar 合法
+ role classifier 高置信
+ 无冲突
```

### 第三阶段：图片作为增强证据

例如：

```text
标题 reference 与保卡/表背 OCR 一致
```

可以提高 trust，但**图片相似度本身仍不能代替 reference**。

---

## 10.6 Reference Identity Store：生产系统不需要构造海量 pair graph

当 trusted reference 得到后，不应该生成所有跨源两两边：

```text
A1-B1
A1-C1
A2-B1
A2-C1
...
```

而应该直接映射到一个 reference entity：

```text
record ─── belongs_to ───> reference_identity
```

建议最小数据模型：

```sql
reference_identity (
    id UUID PRIMARY KEY,
    brand_id BIGINT NOT NULL,
    strict_reference_key TEXT NOT NULL,
    canonical_display TEXT,
    canonicalizer_version TEXT NOT NULL,
    status TEXT NOT NULL,
    UNIQUE (brand_id, strict_reference_key)
);

product_record (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    source_item_id TEXT NOT NULL,
    payload_version BIGINT NOT NULL,
    ...
);

reference_evidence (
    id BIGSERIAL PRIMARY KEY,
    record_id TEXT NOT NULL,
    raw_value TEXT NOT NULL,
    canonical_value TEXT,
    role TEXT NOT NULL,
    evidence_source TEXT NOT NULL,
    confidence DOUBLE PRECISION,
    extractor_version TEXT NOT NULL,
    canonicalizer_version TEXT NOT NULL,
    conflict_flags JSONB
);

group_membership (
    record_id TEXT PRIMARY KEY,
    reference_identity_id UUID NOT NULL,
    decision_type TEXT NOT NULL,
    decision_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);

review_task (
    id BIGSERIAL PRIMARY KEY,
    record_id TEXT NOT NULL,
    reason TEXT NOT NULL,
    candidate_payload JSONB,
    status TEXT NOT NULL
);
```

自动归组变成：

```python
identity = upsert_reference_identity(brand_id, strict_reference_key)
attach_record(record_id, identity.id)
```

平均意义上是索引/哈希 lookup，而不是 N² pairwise matching。

对于 100 万～1000 万级数据，这是数量级上的架构变化。

---

# 11. 增量更新怎么做

Spec 明确要求持续增量，因此每条 record 都应该可重放、可撤销、可审计。

推荐增量事件：

```text
ProductUpserted(source, source_item_id, payload_version)
```

处理流程：

```text
1. 读取最新 raw record
2. source adapter 标准化字段
3. brand normalize
4. reference extraction
5. role classification
6. brand-aware canonicalization
7. resolver
8a. TRUSTED:
      upsert reference_identity
      transactional replace group_membership
8b. MISSING / AMBIGUOUS / CONFLICT:
      不自动匹配
      删除/冻结旧的自动 membership（按规则）
      进入 candidate/review pipeline
9. 写 decision log + parser/model version
```

### 为什么一定要保存版本

例如 `ref-can-v2` 发现 v1 曾经把两个 suffix 错误归一成一个 key。

如果保存了：

```text
extractor_version
canonicalizer_version
decision_version
raw evidence
```

就可以精确找出受影响记录重新计算，而不是全库盲目重跑。

这对“绝对不能误匹配”非常重要：**系统必须能撤销错误，不只会追加 match。**

---

# 12. 三来源如何处理：不要依赖 pairwise 传递关系建立全局身份

pyJedAI 原生更偏两表 ER。

如果直接这样做：

```text
雷小安 ↔ 腕表之家  得到 edge A-B
腕表之家 ↔ 奢当家  得到 edge B-C
然后 Connected Components 得到 A-B-C
```

那么 B-C 的一个误判会污染 A。

生产应该反过来：

```text
A -> reference_identity R
B -> reference_identity R
C -> reference_identity R
```

A/B/C 并不需要靠彼此的相似边证明同一性，它们独立证明自己属于 R。

这等价于把实体解析问题从：

```text
record-to-record matching
```

改成：

```text
record-to-canonical-reference resolution
```

当前业务定义恰好允许我们这样做，这是最值得利用的先验约束。

---

# 13. pyJedAI 在项目中应该怎样真正落地

## 13.1 作为离线“候选层算法赛马”框架

在三组 source pair 的代表性样本上运行：

```text
Baseline A: StandardBlocking
Baseline B: QGramsBlocking
Baseline C: StandardBlocking + BlockFiltering
Baseline D: StandardBlocking + WEP(EJS)
Baseline E: EmbeddingsNNBlockBuilding + FAISS Top-K
```

比较：

```text
Candidate Recall@K
Pair Completeness
Candidate Pair Count
Reduction Ratio
Runtime
Peak Memory
```

目标不是选“最终 precision 最高”的 pyJedAI pipeline，而是：

> **在 unresolved records 上，用尽可能少的候选保住尽可能多的真实同-reference 对。**

最终自动 precision 仍由 strict resolver 决定。

## 13.2 一个安全的 pyJedAI 实验方式

示意：

```python
from pyjedai.datamodel import Data
from pyjedai.block_building import StandardBlocking, QGramsBlocking
from pyjedai.block_cleaning import BlockFiltering
from pyjedai.comparison_cleaning import WeightedEdgePruning

# 每次只做两个来源的离线候选实验
# 注意：不要对 reference 字段调用 Data.clean_dataset() 默认清洗。
data = Data(
    dataset_1=left_df,
    id_column_name_1="record_id",
    attributes_1=["brand", "series", "candidate_text"],
    dataset_2=right_df,
    id_column_name_2="record_id",
    attributes_2=["brand", "series", "candidate_text"],
    ground_truth=gold_pairs,
)

bb = StandardBlocking()
blocks = bb.build_blocks(
    data,
    attributes_1=["brand", "series", "candidate_text"],
    attributes_2=["brand", "series", "candidate_text"],
)

bf = BlockFiltering(ratio=0.9)
blocks = bf.process(blocks, data)

wep = WeightedEdgePruning(weighting_scheme="EJS")
candidates = wep.process(blocks, data)
```

这里 `candidates` 的含义必须是：

```text
值得进一步解析/复核的 pair
```

而不是：

```text
已经匹配成功的 pair
```

## 13.3 reference 本身最好不要进入“自由相似度”决策

候选层可以使用：

```text
brand
series
collection
material
size
normalized title text
image embedding
```

reference 候选可以用于 block key 或 exact filter，但最终 identity 必须由 `strict_reference_key` equality 形成。

特别禁止：

```python
levenshtein(ref_a, ref_b) > threshold -> auto_match
```

---

# 14. 图片应该怎样使用

Spec 明确有图片可用。

建议把图片拆成两个能力，不要混成一个“视觉相似度分数”。

## 14.1 OCR / Visual Text Extraction

优先识别：

- 表背刻字；
- 保卡；
- 吊牌；
- 证书；
- 盒标；
- 其他可能直接出现 reference 的区域。

这类图片能产生**可验证的字符串证据**，价值远高于仅做视觉 embedding。

OCR 结果仍然要进入：

```text
role classifier
-> brand grammar
-> canonicalizer
-> conflict resolver
```

而不是 OCR 一抽到就放行。

## 14.2 Image Similarity

图像 embedding 可用于：

- unresolved candidate retrieval；
- 人工 review 排序；
- 发现明显冲突。

例如文本说一个系列，但图像分类器认为完全不同系列，可以作为 veto / review signal。

不建议：

```text
图看起来一样 -> 同 reference
```

因为同系列不同 reference 的外观可能高度接近。

---

# 15. 几百对人工黄金标签应该怎么花

随机抽几百 pair 标注，价值不高；应刻意覆盖最容易产生 false positive 的 hard negatives。

建议标签结构至少包含：

### 正例

- 同 reference，不同空格/大小写形式；
- 同 reference，结构化字段 vs 标题抽取；
- 同 reference，标题 vs OCR；
- 新出现/长尾 reference；
- 字段大量缺失但 reference 确认一致。

### 关键负例

- 同品牌、同系列、相邻 reference；
- 只差 suffix 的 reference；
- 一个是平台 SKU，一个是品牌 reference；
- 配件标题提到了“兼容某腕表 reference”；
- OCR 常见混淆：`O/0`、`I/1`、`S/5`、`B/8`；
- 标题同时出现多个 reference；
- 结构化 reference 与标题 reference 冲突；
- 图像非常相似但 reference 不同；
- 同型号名字但不同真正 reference。

这几百条应主要用于：

1. 验收 extractor / role classifier / resolver；
2. 校准人工复核阈值；
3. 构造 hard-negative regression suite；
4. 比较候选 Blocking 的 Recall@K；
5. 以后每次 parser/model 升级跑回归。

---

# 16. 评测指标：不能只看 F1

pyJedAI 自带 Evaluation 可以统计 TP / FP / FN / Precision / Recall / F1，但当前业务要补充更贴合 production gate 的指标。

建议至少监控：

```text
1. Auto Match Precision
2. Auto False Positive Count
3. Auto Coverage
4. Trusted Reference Extraction Precision
5. Reference Conflict Rate
6. Candidate Recall@K（仅 unresolved lane）
7. Review Rate
8. Unresolved Rate
9. Hard-negative Precision
10. 每品牌/每来源分层指标
```

最重要的是：

```text
FP_auto = 0
```

应作为发布门槛之一，而不是被总体 F1 掩盖。

但还要说明一个统计事实：**几百条标签无法数学上证明“全量永远零 FP”。**

所以“绝对不能误匹配”不能只靠测试集指标保证，而应该靠：

- deterministic strict key；
- 保守 parser；
- 冲突即拒绝；
- 模糊模型无写权限；
- decision provenance；
- 可回滚；
- hard-negative regression。

也就是让架构本身尽量消灭错误通路。

---

# 17. 为什么不建议直接把 pyJedAI 跑到 1000 万级生产

## 17.1 pandas + Python object 内存模型

Data 是 pandas-first；Blocking 又大量使用 Python dict/set。

实验和中小批处理方便，但 1000 万记录、多个 text token、多个 block 时对象开销很高。

## 17.2 NetworkX Graph 不适合成为海量在线关系存储

`EntityMatching` / clustering 以 NetworkX graph 表示候选/相似边。

如果候选边 E 达到几千万级，Python graph object 的内存开销会成为核心瓶颈。

生产应该把候选边当流/表处理，最终 identity membership 用数据库唯一索引维护，而不是把全量实体关系放进 NetworkX。

## 17.3 原生两数据集抽象

三源及未来更多来源需要外层编排；如果继续靠 pairwise edge + transitive closure 扩源，风险会随来源数增长。

全局 `reference_identity` 则天然支持第 4、第 5 个来源。

## 17.4 通用 ER 目标与本业务的“身份硬约束”不同

pyJedAI 是 domain-independent ER 框架，它合理地允许 similarity-based match。

当前业务却已经明确：

```text
identity semantics = same reference
```

这时再让通用 similarity model 决定最终身份，等于丢掉最强的业务先验。

---

# 18. 推荐的生产技术拆分

下面是一个可直接执行的工程拆分，不要求一次把所有组件做完。

## 服务 A：Source Normalizer

输入：三来源 raw record

输出统一 schema：

```text
record_id
source
source_item_id
brand_raw
brand_id
product_type
title
description
structured_reference
image_urls
updated_at
raw_payload_pointer
```

## 服务 B：Reference Intelligence

内部模块：

```text
CandidateExtractor
RoleClassifier
BrandReferenceParser
Canonicalizer
ConflictResolver
```

输出：

```text
TRUSTED(ref_key)
AMBIGUOUS(candidates)
CONFLICT(evidences)
MISSING
```

## 服务 C：Reference Identity Store

核心接口：

```python
get_or_create_identity(brand_id, strict_ref_key)
attach_record(record_id, identity_id, decision_metadata)
detach_record(record_id, reason)
get_members(identity_id)
```

必须有唯一约束：

```text
UNIQUE(brand_id, strict_reference_key)
```

## 服务 D：Unresolved Candidate Service

只处理非 TRUSTED 数据：

```text
brand blocking
series blocking
q-gram blocking
ANN/FAISS
image ANN
meta-blocking
```

可以把 pyJedAI 作为算法实验参考或直接在 sampled batch 上使用。

## 服务 E：Review Console

给人工展示：

- 原始商品字段；
- 所有 reference evidence；
- 文本 span；
- 图片/OCR region；
- 冲突原因；
- 候选 reference identity；
- 模型分数仅作排序参考。

人工动作应优先生成“reference 标签 / parser 修正规则”，而不是只生成 pair label，这样反馈更容易推广到同类记录。

---

# 19. 最小可落地版本（MVP）

如果希望最快验证价值，不需要先做复杂深度模型。

## Phase 0：数据审计

对三来源各抽样：

- reference 独立字段覆盖率；
- 标题包含 reference 的比例；
- 多 reference 比例；
- SKU/库存号混入比例；
- 品牌覆盖；
- 图片中可见 reference 的比例。

## Phase 1：只上最安全自动通道

实现：

```text
brand normalization
+ structured reference parser
+ conservative canonicalization
+ conflict veto
+ exact identity store
```

此时覆盖率可能不高，但 precision 最容易做到极高。

## Phase 2：标题/描述抽取

加入：

```text
brand-aware regex / parser
+ reference role classifier
```

仍然只有 resolver 给出 TRUSTED 才自动归组。

## Phase 3：OCR

从保卡、表背、吊牌等补 reference evidence。

优先把 OCR 当“交叉验证”和“冲突发现”工具，而不是自由识别后直接放行。

## Phase 4：pyJedAI Candidate Lab

对 MISSING / AMBIGUOUS 尾部数据：

- 比较 Standard / QGram / WEP / FAISS Top-K；
- 选 candidate recall 高且成本可控的方案；
- 结果进入人工复核，不进入自动 identity。

## Phase 5：持续学习

人工复核反馈到：

```text
brand parser rules
role classifier
OCR correction dictionary
hard-negative suite
candidate retriever
```

每次升级先跑 regression，再扩大 auto coverage。

---

# 20. 一个更接近生产的核心伪代码

```python
def process_product(record):
    normalized = normalize_source_record(record)
    brand_id = resolve_brand(normalized)

    evidences = extract_reference_evidence(
        normalized,
        include_structured=True,
        include_title=True,
        include_description=True,
        include_ocr=True,
    )

    evidences = classify_reference_roles(evidences, normalized)
    evidences = canonicalize_with_brand_rules(evidences, brand_id)

    resolution = resolve_reference(
        record=normalized,
        brand_id=brand_id,
        evidences=evidences,
    )

    persist_evidences(evidences)

    if resolution.status == "TRUSTED":
        identity_id = get_or_create_reference_identity(
            brand_id=brand_id,
            strict_reference_key=resolution.strict_key,
        )

        replace_membership_transactionally(
            record_id=normalized.record_id,
            identity_id=identity_id,
            decision_type="STRICT_REFERENCE",
            decision_version=resolution.version,
        )
        return

    # 模糊模型没有 attach 权限
    candidates = retrieve_unresolved_candidates(normalized, brand_id)

    create_or_update_review_task(
        record_id=normalized.record_id,
        reason=resolution.status,
        candidates=candidates,
        evidences=evidences,
    )
```

最重要的一行其实是：

```text
# 模糊模型没有 attach 权限
```

这是把“precision-first”从模型参数变成系统访问控制。

---

# 21. pyJedAI 可以直接复用什么，应该替换什么

| pyJedAI 能力 | 建议 | 当前项目里的角色 |
|---|---|---|
| Data / ground truth 接口 | 部分复用 | 离线双源实验数据封装 |
| Standard / QGram Blocking | 复用思想或实验代码 | unresolved candidate recall |
| BlockFiltering / BlockPurging | 可复用 | 压缩高频噪声候选 |
| WeightedEdgePruning / Meta-blocking | 可实验 | 候选排序/削减 |
| EmbeddingsNNBlockBuilding + FAISS | 可实验 | 长尾候选召回 |
| EntityMatching | 不作为自动判定 | review ranker / baseline |
| ConnectedComponents | 禁止用于 fuzzy 自动合并 | 仅 strict-edge 的离线验证可用 |
| UniqueMappingClustering | 不适合 | 同 reference 多 listing 不是一对一 |
| Evaluation | 强烈复用并扩展 | 基准、回归、候选召回评测 |
| `Data.clean_dataset()` 默认策略 | reference 上禁用 | 另写 ReferenceCanonicalizer |
| pandas/NetworkX 全量运行 | 不作为 10M 生产主存储 | 样本/品牌分区实验 |

---

# 22. 最终推荐

如果只吸收 pyJedAI 的一个核心思想，我会选：

> **把 Entity Resolution 做成模块化、逐层可评测的 pipeline，而不是一个端到端黑盒分数。**

但当前项目还必须进一步做一层业务约束升级：

> **把“Reference 是否可信”放在所有相似度模型之前，并把 exact reference identity 作为唯一可自动落库的身份键。**

最终建议的职责边界是：

```text
pyJedAI / ANN / LLM / image embedding
          │
          └── 找候选、找证据、找异常、排序

Reference Extractor + Role Classifier + Brand Parser + Resolver
          │
          └── 决定是否存在可信 strict reference

Reference Identity Store
          │
          └── 唯一拥有自动“归组/匹配”写权限
```

这样既能利用现代 ER 的召回能力，又不会让软相似度突破“同一 reference 才算同一商品”的业务红线。

对于 100 万～1000 万级持续增量数据，可信 reference 记录的主路径会从潜在的 pairwise matching 问题降维成：

```text
extract -> canonicalize -> indexed exact lookup -> attach
```

真正昂贵的 Blocking / ANN / OCR / LLM 只服务于少量 unresolved tail。

这应当是当前 Spec 最安全、也最容易逐步上线的实现方式。

---

## 参考代码

- pyJedAI 仓库：https://github.com/AI-team-UoA/pyJedAI/tree/eb446095f921af3207bbefd570cc65e818194419
- README：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/README.md
- Data Model：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/datamodel.py
- Block Building：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/block_building.py
- Block Cleaning：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/block_cleaning.py
- Comparison Cleaning：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/comparison_cleaning.py
- Entity Matching：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/matching.py
- Clustering：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/clustering.py
- Vector Blocking：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/vector_based_blocking.py
- Workflow：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/workflow.py
- Evaluation：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/src/pyjedai/evaluation.py
- pyproject：https://github.com/AI-team-UoA/pyJedAI/blob/eb446095f921af3207bbefd570cc65e818194419/pyproject.toml
