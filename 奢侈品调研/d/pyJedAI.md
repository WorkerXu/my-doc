# pyJedAI：面向跨源二奢/腕表 reference-first 实体解析系统的技术分析与落地方案

> 分析对象：[`AI-team-UoA/pyJedAI`](https://github.com/AI-team-UoA/pyJedAI)  
> 当前仓库版本：`0.3.6`（见 `pyproject.toml`）  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量；字段高度稀疏；reference 有时独立、有时埋在标题；可用图片；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，允许漏匹配**。

---

## 1. 结论先行

pyJedAI 很适合拿来参考“实体解析系统应该怎样拆层”，但**不应该直接把它的默认 matcher / clustering 当作本需求的最终判定逻辑**。

它最值得复用的地方有四个：

1. **把 ER 拆成可替换阶段**：Blocking → Block/Comparison Cleaning → Entity Matching → Clustering → Evaluation；
2. **Blocking 与 Matching 解耦**：先把千万级笛卡尔积压到小候选集，再做昂贵判断；
3. **同时提供词法 Blocking 与 embedding + FAISS Blocking**，便于快速对比召回与成本；
4. **统一的数据对象、评估接口和 workflow 编排方式**，非常适合做离线实验平台。

但当前业务的身份定义比通用 ER 更强：

> 同一商品 = 同一品牌语义下的同一 canonical reference。

因此生产系统的核心不应是：

```text
相似度 > threshold => match
```

而应该是：

```text
品牌规范化
  -> reference 候选抽取
  -> reference 角色判定
  -> conservative canonicalization
  -> 证据冲突检查
  -> canonical reference exact match
  -> ACCEPT / REJECT / ABSTAIN
```

pyJedAI 应承担两类职责：

- **生产前离线算法试验台**：快速评测各种 blocking / comparison cleaning / embedding 召回方案；
- **生产中的候选召回参考实现**：尤其用于 reference 缺失、埋在标题、OCR 才能得到编号的记录。

最终自动归并必须由 reference-first 的硬规则收口；文本相似、图片相似、向量相似、LLM 判断都不能越权覆盖明确的 reference 冲突。

---

## 2. 为什么本轮选 pyJedAI

调研清单对 pyJedAI 的推荐理由是：它提供端到端实体解析工作流，覆盖 Blocking、Comparison Cleaning、Entity Matching 和 Clustering，可以把 reference exact rule 作为强约束，与多种候选生成策略统一评测。

这个点正好补足当前 Spec 中最容易被低估的问题：

- 100 万–1000 万条数据，不能直接两两比较；
- 三个来源字段分布不同，必须允许多种 Blocking 策略共存；
- 最终 precision-first，但 Blocking 本身又必须尽量保 recall；
- 需要少量黄金标签评估不同阶段，而不是只看最后一个 F1。

与只提供单一模型的论文相比，pyJedAI 更像一个“ER 实验操作系统”，更容易映射成工程架构。

---

## 3. pyJedAI 的代码架构

核心代码位于 `src/pyjedai/`：

```text
src/pyjedai/
├── datamodel.py                 # Data / PYJEDAIFeature，统一数据与组件接口
├── block_building.py            # Standard/QGram/Suffix 等词法 Blocking
├── block_cleaning.py            # Block Purging / Filtering
├── comparison_cleaning.py       # Meta-blocking、边权、候选比较裁剪
├── vector_based_blocking.py     # embedding + FAISS Top-K Blocking
├── matching.py                  # 字符串/集合/向量相似度 Entity Matching
├── llm_matching.py              # LLM Matching 扩展
├── clustering.py                # Connected Components / Unique Mapping 等
├── evaluation.py                # Precision / Recall / F1 与候选评估
├── prioritization.py            # Progressive ER
├── joins.py                     # Similarity Join / Top-K Join
└── workflow.py                  # End-to-end workflow 编排
```

依赖也反映了其定位：

```text
pandas / numpy / scipy
networkx
ray
transformers / sentence-transformers
faiss-cpu
nltk
py-stringcompare
```

它不是一个单模型，而是一组有统一接口的 ER 算法组件。

### 3.1 `PYJEDAIFeature`：统一组件协议

`datamodel.py` 中绝大多数算法组件都继承 `PYJEDAIFeature`，统一暴露：

```python
_configuration()
evaluate(...)
stats()
method_configuration()
report()
```

这个设计很值得复用，因为生产系统最需要的并不是“再加一个模型”，而是让每个阶段都有：

- 固定输入/输出契约；
- 可记录配置；
- 可单独评估；
- 可统计 runtime；
- 可做 A/B pipeline 对比。

对于本需求，建议把类似协议改造成：

```python
class Stage:
    name: str
    version: str

    def run(self, batch, context): ...
    def metrics(self): ...
    def config(self): ...
```

并强制所有 reference 抽取、规范化、Blocking、OCR、匹配规则都带版本号，保证后续能解释“为什么这两条在某天被合并”。

---

## 4. `Data` 数据模型：适合实验，不适合直接承载三源生产主存

`datamodel.py` 中 `Data` 的核心形态是：

```python
Data(
    dataset_1=...,
    dataset_2=...,
    id_column_name_1=...,
    id_column_name_2=...,
    ground_truth=...
)
```

如果没有 `dataset_2`，它把任务看作 Dirty ER；如果有两个数据集，则看作 Clean-Clean ER。

内部会：

- `fillna("")`；
- 转成字符串；
- 维护原始 ID ↔ 连续整数 ID 的映射；
- 将两个 DataFrame 拼接成统一 `entities`；
- 可挂 ground truth 做评估。

### 4.1 对当前需求的限制

本需求是雷小安、腕表之家、奢当家三个来源，而且以后可能继续增加来源。pyJedAI 的核心 `Data` 是两表优先设计，因此不建议直接把生产实体存储建成它的结构。

更好的生产数据模型是：

```text
source_product
  source_id
  source_product_id
  raw_payload
  title
  brand_raw
  ref_raw
  images
  ingest_version

ref_evidence
  product_pk
  value_raw
  value_canonical
  evidence_type
  source_field
  extractor_version
  confidence
  is_authoritative
  conflict_group

reference_entity
  entity_id
  brand_id
  reference_canonical
  status

entity_member
  entity_id
  product_pk
  decision_id

match_decision
  decision_id
  product_pk
  entity_id
  decision
  reason_codes
  evidence_snapshot
  rule_version
```

pyJedAI 的 `Data` 可以作为离线 benchmark adapter，把生产表按两个来源切成 DataFrame 做实验，而不是反过来让生产存储迁就实验框架。

---

## 5. Standard Blocking：实现非常朴素，但思想很重要

`block_building.py` 的 `StandardBlocking` 做法是：

```text
选择属性
 -> 每行属性拼成字符串
 -> lower-case
 -> 按非单词字符切 token
 -> 每个 token 建一个倒排 block
 -> 一个 block 内出现的两侧实体形成候选关系
```

核心逻辑近似：

```python
for eid, entity in enumerate(entities):
    for token in entity:
        blocks[token].add(eid)
```

并删除只含单实体、无法产生比较的 block。

代码还提供 Ray 分批 tokenization / building 路径，默认 batch size 为 `10000`。

### 5.1 对腕表 reference 的启发

不要直接对完整标题做 generic token block；应该构造**业务感知 block key**：

```text
brand_id + exact_reference
brand_id + reference_prefix
brand_id + ref_alnum_shape
brand_id + series_id
brand_id + title_ref_candidate
brand_id + OCR_ref_candidate
```

例如：

```text
Rolex 126610LN
Rolex 126610LV
```

从语义和视觉上极近，但业务定义上必须分开，所以最强 block 应直接是：

```text
(roleх, 126610LN)
(roleх, 126610LV)
```

而不是共同落到“潜航者 / 水鬼 / 126610”大 block 后，再依赖模型判断。

### 5.2 推荐的三级 Blocking

#### L0：exact reference block

```text
key = (canonical_brand, canonical_reference)
```

这是主路径。只要 reference 证据足够强，不需要 pairwise matcher，直接映射到 `reference_entity`。

#### L1：reference-shape block

仅给 unresolved 记录用：

```text
brand + normalized_prefix
brand + alnum_length
brand + series + numeric_stem
```

目的是召回可能的正确 reference，不允许直接自动合并。

#### L2：semantic / image block

对 L1 仍无法定位的少量记录：

```text
brand constrained ANN
image ANN
OCR text ANN
```

仍然只产候选，不产最终 match。

---

## 6. EmbeddingsNNBlockBuilding：可以借，但不能原样拿到千万级生产

`vector_based_blocking.py` 的 `EmbeddingsNNBlockBuilding` 支持多种向量化方式：

- word2vec / fastText / GloVe；
- BERT / DistilBERT / RoBERTa / XLNet / ALBERT；
- SentenceTransformer：MPNet、T5、MiniLM 等；
- 自定义 Hugging Face 模型。

它会把指定属性拼成字符串，再生成向量，然后使用 FAISS Top-K 近邻检索。

### 6.1 当前代码的 FAISS 路径

实现中直接使用：

```python
index = faiss.IndexFlatL2(dim)
```

如果距离是 cosine，则把 metric 切到 inner product，并对向量做 L2 normalize；最终：

```python
index.add(vectors_1)
distances, neighbors = index.search(vectors_2, top_k)
```

这说明当前实现使用的是 FAISS Flat exact search，而不是 IVF/HNSW/PQ 等近似索引。

### 6.2 对 100 万–1000 万级的影响

`IndexFlat` 很适合 benchmark，因为实现简单、结果稳定；但千万级持续增量如果对全部商品频繁全局 Top-K，内存和扫描成本都会快速上升。

生产建议：

```text
先按 brand / category / series 做硬分区
      ↓
只在 unresolved 子集内建 ANN
      ↓
HNSW / IVF-PQ / DiskANN 类索引
      ↓
Top-K <= 20~50
```

并且 embedding 不应该编码全部字段的裸拼接，而要至少保留字段角色：

```text
[BRAND] ROLEX
[SERIES] SUBMARINER
[REF_CANDIDATES] 126610LN
[TITLE] 劳力士 潜航者 黑水鬼 自动机械
[CATEGORY] watch
```

### 6.3 向量召回在本需求中的权限边界

允许：

- 找“可能包含同一 reference 的候选记录”；
- 帮 reference 抽取器检索相似历史样本；
- 给人工复核排序；
- 在一条记录没有 reference 时寻找可能的 reference 候选。

禁止：

- 因为图片/文本相似就直接自动合并；
- 覆盖两个明确 reference 不一致的记录；
- 把同系列不同 variant 当成同一商品。

---

## 7. Comparison Cleaning：非常适合解决“大 block 候选爆炸”

pyJedAI 的 `comparison_cleaning.py` 会先建立 entity → block 的倒排索引，然后在候选图上计算边权并裁剪。

其 Meta-blocking 支持多种权重概念，例如：

- CBS：共同 block 数；
- COSINE；
- DICE；
- JS；
- EJS；
- ECBS；
- 一些 cardinality / size normalized 变体。

抽象逻辑是：

```text
一对实体共同出现在哪些 blocks
      ↓
根据 block 稀有度/共现次数算 edge weight
      ↓
删除低价值 comparison
```

### 7.1 在二奢场景中的改造

建议不要使用“共同 token 越多越像”的通用权重，而是做**证据等级权重**：

```text
+100  authoritative ref exact
 +40  title single ref exact
 +30  OCR single ref exact
 +10  same canonical brand
  +5  same series
  +2  high image similarity
-100  authoritative ref conflict
 -80  brand conflict
 -50  accessory/compatible-context ref
 -40  multiple competing ref candidates
```

注意：这个分数只用于**候选优先级与人工排序**；最终自动 ACCEPT 仍使用硬条件，而不是“总分过阈值”。

最关键的设计是把 conflict 做成 veto：

```python
if has_authoritative_ref_conflict:
    return REJECT
```

而不是：

```python
score = positive_similarity - conflict_penalty
if score > threshold:
    return MATCH
```

后者存在“很多弱正证据把一个强冲突抵消掉”的风险，不符合 precision-first。

---

## 8. Entity Matching：pyJedAI 默认模型为什么不适合作为最终规则

`matching.py` 中的 `EntityMatching` 支持：

- Levenshtein / Jaro；
- cosine / Jaccard / generalized Jaccard / Dice；
- overlap coefficient；
- TF/TF-IDF 等向量路径；
- 属性列表或属性权重字典。

输出是一个 `networkx.Graph`：

```text
node = entity id
edge = candidate pair
weight = similarity score
```

只有当：

```python
similarity > similarity_threshold
```

边才被加入图。

### 8.1 这与当前 Spec 的根本冲突

腕表型号常见这种 hard negative：

```text
126610LN
126610LV
```

标题、系列、尺寸、图片、价格区间都可能非常相似，字符串本身也只差少数字符。一个统一的相似度 threshold 很难同时做到：

- 对脏格式保持召回；
- 对相邻 reference 保持近乎零误匹配。

所以最终 matcher 应由“相似度函数”改成“reference evidence decision engine”。

---

## 9. 推荐的 StrictReferenceMatcher

### 9.1 先定义证据对象

```python
@dataclass
class RefEvidence:
    value_raw: str
    value_canonical: str | None
    source: str              # ref_field/title/description/ocr/card/backcase
    role: str                # brand_reference/platform_sku/compatible_ref/unknown
    confidence: float
    authoritative: bool
    extractor_version: str
    span: tuple[int, int] | None
```

### 9.2 reference 证据优先级

建议：

```text
A0  平台明确 reference 字段，且通过品牌 validator
A1  官方/结构化品牌型号字段
A2  标题中唯一 reference，且 role classifier 判为当前商品型号
A3  商品描述中唯一 reference
A4  图片 OCR：保卡/吊牌/表背中唯一且通过 validator
A5  LLM / embedding 推测出的 reference 候选
```

只有 A0–A2 可以直接进入第一版自动归并；A3/A4 可在离线验证后逐步放开；A5 永远只做候选或人工辅助。

### 9.3 决策状态机

```python
def decide(record):
    if brand_is_ambiguous(record):
        return ABSTAIN("brand_ambiguous")

    refs = high_conf_product_refs(record)

    if has_authoritative_internal_conflict(refs):
        return ABSTAIN("internal_ref_conflict")

    if len(unique_canonical(refs)) != 1:
        return ABSTAIN("zero_or_multiple_reference")

    ref = only_ref(refs)

    if not brand_specific_validator(record.brand, ref):
        return ABSTAIN("reference_validation_failed")

    return RESOLVED(entity_key=(record.brand, ref))
```

注意：生产系统最好**先把每条记录解析成唯一 entity key**，再归并，而不是先生成海量 pair 再判断两两是否 match。

因为业务定义已经告诉我们 entity key 的本质：

```text
entity_key = canonical_brand + canonical_reference
```

这比通用 pairwise ER 更简单、更安全。

---

## 10. Conservative Reference Canonicalization

这是整个系统最关键的模块之一。

### 10.1 可以安全做的规范化

默认只做信息不丢失或风险极低的转换：

```text
Unicode NFKC
trim
uppercase latin
统一全角/半角
清理明确无语义的外围空白
品牌白名单内统一确定无语义的分隔符
```

例如：

```text
" 126610ln " -> "126610LN"
```

### 10.2 不要默认做的激进转换

```text
O <-> 0
I <-> 1
删除所有 / - .
删除所有 suffix
只保留数字
模糊 edit-distance 自动归一
```

这些都可能把不同 reference 合并。

应该改为：

```text
strict_canonical
relaxed_candidate_key
```

两套值分开：

- `strict_canonical` 才允许自动归并；
- `relaxed_candidate_key` 只允许 Blocking / review retrieval。

### 10.3 品牌级 parser registry

建议实现：

```python
PARSERS = {
    "rolex": RolexReferenceParser(),
    "omega": OmegaReferenceParser(),
    "cartier": CartierReferenceParser(),
    ...
}
```

每个 parser 独立定义：

- 合法字符集；
- 长度/分段规则；
- suffix 是否有语义；
- 已知 reference dictionary；
- 标题 context 规则；
- OCR 常见错误但只能用于候选，不可静默改写。

版本化：

```text
rolex_ref_parser:v3
omega_ref_parser:v2
```

这样回溯错误时能定位是哪套规则导致。

---

## 11. “编号角色分类”必须先于 exact match

二奢标题中的字母数字串不一定是品牌 reference，还可能是：

- 平台商品 ID；
- 店铺 SKU；
- 库存编号；
- 鉴定证书号；
- 机芯号；
- 表带/配件适配的腕表型号；
- 系列代码；
- 价格、年份、尺寸。

例如：

```text
适配 Rolex 126610LN 的橡胶表带
```

如果直接 regex 抽到 `126610LN` 并归并到 Rolex 126610LN 腕表实体，会产生严重 false positive。

建议加一个 `reference_role_classifier`：

```text
输入：
  title + category + 字符串 span + 周边 context + 来源字段名

输出：
  CURRENT_PRODUCT_REFERENCE
  COMPATIBLE_PRODUCT_REFERENCE
  PLATFORM_SKU
  SERIAL_OR_CERT
  UNKNOWN
```

第一版不必训练大模型，规则也能覆盖大量风险：

```text
“适配 / 适用于 / 表带 / 配件 / 保护壳 / compatible with”
      -> compatible_ref

字段名 sku / goods_id / item_id
      -> platform_sku

结构化 model/reference 字段
      -> product_reference
```

分类不确定就 `ABSTAIN`。

---

## 12. 三源生产架构：不要做三次全量 pairwise join

推荐架构：

```text
                ┌────────────────────┐
雷小安 CDC ────>│                    │
腕表之家 CDC ──>│  Ingestion / Normalize │
奢当家 CDC ────>│                    │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Brand Normalizer   │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Reference Extractor│
                │ field/title/OCR    │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Ref Role Classifier│
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Strict Canonicalize│
                │ + Conflict Check   │
                └──────┬───────┬─────┘
                       │       │
               resolved│       │unresolved
                       v       v
          ┌────────────────┐  ┌────────────────────┐
          │ Reference Index│  │ Candidate Retrieval│
          │ brand + ref    │  │ lexical/ANN/image  │
          └───────┬────────┘  └─────────┬──────────┘
                  │                     │
                  v                     v
          ┌────────────────┐   ┌────────────────────┐
          │ Entity Registry│   │ Re-extract / Review│
          └───────┬────────┘   └────────────────────┘
                  │
                  v
          ┌────────────────┐
          │ Audit / Metrics│
          └────────────────┘
```

### 12.1 主路径复杂度

如果每条 record 都能得到可信 `(brand, ref)`：

```text
lookup(entity_key)
```

即可完成归并。

数据库对：

```sql
UNIQUE (brand_id, reference_canonical)
```

建 B-tree/hash 索引后，就不需要三源之间做 `N×M` 比较。

这也是本需求和普通 Entity Matching 最大的不同：**业务已经提供了天然实体主键，只是主键需要被抽取和规范化。**

---

## 13. Reference Index 的直接落地设计

### 13.1 PostgreSQL 主存即可起步

`reference_entity`：

```sql
CREATE TABLE reference_entity (
    entity_id BIGSERIAL PRIMARY KEY,
    brand_id BIGINT NOT NULL,
    reference_canonical TEXT NOT NULL,
    parser_version TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, reference_canonical)
);
```

`entity_member`：

```sql
CREATE TABLE entity_member (
    entity_id BIGINT NOT NULL,
    source_id SMALLINT NOT NULL,
    source_product_id TEXT NOT NULL,
    decision_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, source_product_id)
);
```

10M 级记录对 PostgreSQL 并不是必须引入图数据库的规模；真正需要特殊索引的是 unresolved ANN，而不是已经得到 exact reference 的主路径。

### 13.2 每次增量的幂等逻辑

```python
key = (brand_id, strict_reference)

entity = upsert_reference_entity(key)
attach_product(entity, source_product)
write_decision_log(...)
```

如果重新抓取同一 `source_product_id`：

```text
raw payload hash 未变 -> skip
抽取器版本未变 -> skip
reference 结果未变 -> no-op
结果变化 -> 进入 reassignment safety check
```

不能悄悄把一条已经归并的数据从 A entity 移到 B entity，必须留完整审计日志。

---

## 14. Multi-source Clustering：pyJedAI 的两个聚类器给我们的正反教材

### 14.1 ConnectedComponentsClustering

pyJedAI 的 Connected Components 会对 similarity graph 做连通分量：

```text
A -- B -- C
=> {A,B,C}
```

这是通用 ER 很常见的做法，但如果一条 false-positive edge 混进图，可能通过传递闭包污染整簇。

对当前需求，只有在边代表：

```text
same canonical brand + same strict canonical reference
```

时才能安全用连通分量。

实际上既然 entity key 已经确定，连通分量都不是必需的，直接 group by key 更简单。

### 14.2 UniqueMappingClustering

`UniqueMappingClustering` 会：

1. 把高于阈值的边按相似度降序放进 priority queue；
2. 每次取最高分边；
3. 只要任一实体已匹配，就跳过；
4. 最终得到 clean-clean 的一对一匹配。

它适用于“两个数据集各自无重复、实体最多一一对应”的场景。

但我们的“同一商品 = 同 reference”意味着：

```text
同一来源可以有多个二手 listing 指向同一 reference
```

例如腕表之家可能有多个不同卖家的 `126610LN` 商品记录。它们在业务定义上都属于同一个 reference entity。

所以生产端**不能使用一对一 Unique Mapping**；应该允许：

```text
many records -> one reference_entity
```

---

## 15. 图片怎么用：只把视觉转成 reference 证据或候选召回

Spec 明确有图片，这是重要优势，但需要限制权限。

### 15.1 推荐顺序

```text
图片
 -> 图像类型分类
    表盘 / 表背 / 保卡 / 吊牌 / 盒标 / 普通商品图
 -> OCR
 -> reference candidate extraction
 -> brand-specific validation
 -> 与文本 reference 做 cross-check
```

最有价值的不是“这两块表看起来像不像”，而是：

- 表背刻字是否出现型号；
- 保卡是否出现 reference；
- 吊牌是否有型号；
- 图片 OCR 与标题/结构化字段是否一致。

### 15.2 强冲突优先

例如：

```text
title_ref = 126610LN
card_ocr_ref = 126610LV
```

无论图片 embedding 多相似，都必须：

```text
ABSTAIN / MANUAL_REVIEW
```

而不是投票取多数。

### 15.3 图片 ANN 的正确用途

可以把图片 embedding 用于：

```text
unresolved 商品 -> Top-K 已知 reference 商品图
```

然后把 Top-K 对应的 reference 作为**待验证候选**交给 OCR / 文本 extractor / 人工复核。

---

## 16. pyJedAI 风格的 Reference-first Workflow

可以保留 pyJedAI 的编排思想，但换成业务专用 stage：

```python
workflow = ReferenceResolutionWorkflow([
    BrandNormalization(),
    StructuredReferenceExtractor(),
    TitleReferenceExtractor(),
    ReferenceRoleClassifier(),
    StrictReferenceCanonicalizer(),
    EvidenceConflictCleaner(),
    ExactReferenceResolver(),
    UnresolvedCandidateGenerator(),
    AuditEvaluator(),
])
```

### 16.1 每个阶段输出 reason codes

例如：

```text
BRAND_CANONICALIZED
REF_FROM_STRUCTURED_FIELD
REF_FROM_TITLE_SINGLE_CANDIDATE
REF_VALID_BY_BRAND_PATTERN
REF_CONFLICT_TEXT_VS_OCR
REF_ROLE_COMPATIBLE_PRODUCT
AUTO_LINK_EXACT_REF
ABSTAIN_MULTIPLE_REF
ABSTAIN_LOW_CONF_OCR
```

最终任何自动合并都应该能解释为：

```json
{
  "decision": "AUTO_LINK",
  "brand": "ROLEX",
  "reference": "126610LN",
  "evidence": [
    "REF_FROM_STRUCTURED_FIELD",
    "REF_VALID_BY_BRAND_PATTERN"
  ],
  "rule_version": "strict_ref_match:v4"
}
```

这比保存一个 `similarity=0.9821` 更适合高风险 precision-first 场景。

---

## 17. unresolved 候选生成：这里最适合借 pyJedAI

只有无法形成唯一 strict reference 的记录才进入这一支。

### 17.1 第一层：词法 Blocking

```text
brand + ref prefix
brand + digit pattern
brand + series token
brand + rare title tokens
```

类似 pyJedAI Standard/QGram Blocking，但要加业务字段权重与大 block 限制。

### 17.2 第二层：Comparison Cleaning

删除：

- 大量共同通用词导致的边；
- 只有“二手/正品/男表/机械/九五新”等词重合的边；
- category 冲突；
- brand 冲突；
- 明确 reference 冲突。

### 17.3 第三层：ANN

只在同品牌或极小 brand candidate set 中：

```text
text embedding Top-K
image embedding Top-K
OCR embedding Top-K
```

Top-K 不需要很大，20–50 即可做第二轮抽取/复核。

### 17.4 第四层：重新抽取 reference，而不是直接 match

这是整个设计和常见 RAG/EM 系统的关键区别：

```text
candidate retrieval
    ↓
得到“它可能是 126610LN”
    ↓
回到原始 title / description / image
    ↓
验证是否存在可解释的 reference 证据
    ↓
若无证据，仍 ABSTAIN
```

不能因为 ANN 第一名是 `126610LN` 就把记录贴上该 reference。

---

## 18. 黄金标签应该怎样标，才能真正服务 precision-first

Spec 允许人工标几百对。不要做普通随机正负样本；应优先构造 hard negatives。

建议标签结构：

```text
50  结构化 reference exact positives
50  标题抽取 exact positives
50  OCR 辅助 positives
100 同系列相邻 reference hard negatives
50  平台 SKU / reference 混淆 negatives
50  配件标题中出现腕表 reference negatives
50  O/0、I/1、连字符、空格等 canonicalization 边界
50  多 reference / 冲突 / 应拒识样本
```

如果标注预算只有几百条，宁可少做普通 easy positives，也要把边界样本覆盖完整。

### 18.1 需要分阶段指标

#### Reference extraction

```text
candidate recall
strict extraction precision
multi-candidate rate
abstain rate
```

#### Blocking

```text
pair completeness / candidate recall
candidates per record
P95 candidates per record
runtime
```

#### Final auto-link

```text
precision
false positives count
coverage / auto-link rate
abstain rate
conflict rate
```

本项目最重要的不是最高 F1，而是：

```text
在可以接受的 coverage 下，把 false positive 压到接近 0。
```

---

## 19. 必做 hard-negative 回归用例

### 19.1 同系列不同 suffix

```text
126610LN != 126610LV
```

### 19.2 同 numeric stem 不同完整 reference

```text
AB1234-CD != AB1234-EF
```

不能删除 suffix 后比较。

### 19.3 配件适配型号

```text
“适配 126610LN 的橡胶表带”
```

不得归入 126610LN 腕表实体。

### 19.4 多型号标题

```text
“126610LN / 116610LN 通用配件”
```

必须多候选并 `ABSTAIN`。

### 19.5 OCR 混淆

```text
0/O
1/I/L
5/S
8/B
```

不能直接自动纠正为某个已知 reference 后归并。

### 19.6 品牌冲突

两个相同 reference 字符串如果品牌不同，默认不能匹配：

```text
entity_key = brand + reference
```

### 19.7 来源自有 SKU

```text
sku=126610LN
```

如果字段语义是平台 SKU，就算字符串恰好像品牌 reference，也不能直接信任。

---

## 20. 生产接口建议

### 20.1 `resolve_product`

```http
POST /v1/resolve-product
```

输入：

```json
{
  "source": "watchhome",
  "source_product_id": "xxx",
  "title": "...",
  "brand": "劳力士",
  "reference": "126610LN",
  "description": "...",
  "images": ["..."]
}
```

输出：

```json
{
  "decision": "AUTO_LINK",
  "entity_id": 123456,
  "brand_canonical": "ROLEX",
  "reference_canonical": "126610LN",
  "reason_codes": [
    "STRUCTURED_REF",
    "BRAND_VALIDATED",
    "STRICT_EXACT_ENTITY_KEY"
  ],
  "rule_version": "strict_ref:v1"
}
```

### 20.2 `ABSTAIN` 输出必须保留候选

```json
{
  "decision": "ABSTAIN",
  "reason_codes": ["MULTIPLE_REF_CANDIDATES"],
  "reference_candidates": [
    {"value": "126610LN", "score": 0.91},
    {"value": "116610LN", "score": 0.88}
  ],
  "review_priority": 73
}
```

这个 score 只用于人工排序，不能变成自动匹配阈值。

---

## 21. 增量更新架构

100 万–1000 万级并不可怕，真正难点是持续增量与规则版本升级。

建议每条商品保存：

```text
raw_hash
normalizer_version
brand_parser_version
ref_extractor_version
ocr_version
resolver_rule_version
resolved_entity_id
resolution_status
```

当某个 parser 升级时，不需要全库无差别重算，而是：

```text
找受该品牌/parser 影响的记录
 -> shadow re-resolve
 -> diff old/new entity key
 -> 如果 entity 变化，进入 safety review
 -> 通过后再切换
```

### 21.1 不要自动 merge 已有 entity

如果新规则认为：

```text
旧 entity A
旧 entity B
应该合并
```

必须生成 `MERGE_PROPOSAL`，而不是直接执行。

因为 entity merge 是最容易把历史 false positive 扩散到整库的操作。

---

## 22. 推荐的离线实验方式：直接借 pyJedAI 做候选层 benchmark

可以从每个来源抽样一批记录，构造两两 Clean-Clean ER benchmark：

```text
雷小安 vs 腕表之家
雷小安 vs 奢当家
腕表之家 vs 奢当家
```

对 unresolved 子集比较：

```text
A. StandardBlocking
B. QGramBlocking
C. 业务自定义 reference-shape blocking
D. EmbeddingsNNBlockBuilding + MiniLM
E. EmbeddingsNNBlockBuilding + 中文/多语商品 embedding
```

主要看：

```text
candidate recall
avg candidates per record
P95 candidates
runtime
memory
```

不要先比较最终 F1。

### 22.1 业务自定义 blocking 的目标

理想结果应该是：

```text
candidate recall >= 通用 embedding blocking
candidate 数显著更少
```

因为 reference、brand、series 都是非常强的结构化先验。

---

## 23. 一个可以直接实现的核心代码骨架

```python
from dataclasses import dataclass
from enum import Enum

class Decision(str, Enum):
    AUTO_LINK = "AUTO_LINK"
    REJECT = "REJECT"
    ABSTAIN = "ABSTAIN"

@dataclass
class Resolution:
    decision: Decision
    brand: str | None
    reference: str | None
    entity_key: tuple[str, str] | None
    reasons: list[str]

class StrictReferenceResolver:
    def resolve(self, product) -> Resolution:
        brand = normalize_brand(product)
        if not brand:
            return Resolution(Decision.ABSTAIN, None, None, None,
                              ["BRAND_UNRESOLVED"])

        evidences = extract_reference_evidence(product, brand)
        product_refs = [
            e for e in evidences
            if e.role == "CURRENT_PRODUCT_REFERENCE"
            and e.confidence >= trusted_threshold(e.source)
        ]

        authoritative = [e for e in product_refs if e.authoritative]
        if len({e.value_canonical for e in authoritative}) > 1:
            return Resolution(Decision.ABSTAIN, brand, None, None,
                              ["AUTHORITATIVE_REF_CONFLICT"])

        strict_refs = {
            e.value_canonical for e in product_refs
            if e.value_canonical and strict_validator(brand, e.value_canonical)
        }

        if len(strict_refs) != 1:
            return Resolution(Decision.ABSTAIN, brand, None, None,
                              ["ZERO_OR_MULTIPLE_STRICT_REFS"])

        ref = next(iter(strict_refs))
        return Resolution(
            Decision.AUTO_LINK,
            brand,
            ref,
            (brand, ref),
            ["STRICT_REFERENCE_RESOLVED"]
        )
```

这段代码刻意不做 pairwise similarity；只有记录本身被解析出唯一可信 reference，才允许进入实体注册表。

---

## 24. pyJedAI 中哪些组件可以直接借，哪些只能参考

| pyJedAI 组件 | 建议 | 原因 |
|---|---|---|
| `PYJEDAIFeature` / workflow 编排 | **直接借思想** | 很适合版本化、可评估的 pipeline |
| `Data` | **仅离线 benchmark** | 两数据集优先，不适合作为三源生产主存 |
| `StandardBlocking` | **改造后可用** | 倒排 block 思路好，但 token key 需业务化 |
| QGram/Suffix Blocking | **仅 unresolved 候选召回** | 模糊召回可用，不能直接判同 |
| Block Filtering/Purging | **可借** | 控制大 block 与比较数量 |
| Comparison Cleaning | **可借框架** | 候选图裁剪有价值，但权重要改成业务证据 |
| `EmbeddingsNNBlockBuilding` | **可借接口，不建议原样上生产** | 当前 FAISS Flat，千万级需换 ANN/分区 |
| `EntityMatching` | **不作为最终 matcher** | 通用 similarity threshold 不满足零误匹配目标 |
| Connected Components | **仅 strict-ref 图可用** | 模糊边会产生传递污染 |
| UniqueMappingClustering | **不适用最终业务** | 业务是 many listings → one reference，不是一对一 |
| `Evaluation` | **可借思想** | 应扩展为阶段指标与 hard-negative 精度评估 |

---

## 25. 分阶段落地方案

### Phase A：只做最高精度主路径

实现：

```text
brand normalization
structured reference extraction
title single-reference extraction
strict canonicalization
brand validator
reference_entity exact index
audit log
```

自动合并只接受：

```text
唯一 reference + 无冲突 + exact entity key
```

其余全部 unresolved。

### Phase B：扩大 reference 抽取覆盖

增加：

```text
description extractor
reference role classifier
brand-specific parser
known reference dictionary
```

仍不改变 final exact rule。

### Phase C：图片证据

增加：

```text
image type classifier
OCR
card/backcase/tag reference extractor
text-vs-image conflict check
```

先只用于拒绝/人工辅助，再根据黄金集表现考虑放开部分来源。

### Phase D：pyJedAI-style 候选召回

对 unresolved：

```text
business blocking
comparison cleaning
text/image ANN Top-K
```

目的是“找到可能的 reference”，不是直接匹配。

### Phase E：主动学习/人工闭环

优先把以下样本送人工：

```text
多 reference
高相似但 reference 冲突
OCR 与文本冲突
高频未知品牌格式
候选 Top-1/Top-2 很接近
```

人工结果回流：

- parser 单元测试；
- brand validator；
- role classifier；
- hard-negative benchmark。

---

## 26. 关键监控指标

线上 dashboard 至少要有：

```text
resolved_rate
abstain_rate
ref_extract_rate_by_source
ref_conflict_rate
brand_unresolved_rate
multi_ref_rate
ocr_ref_rate
entity_reassignment_rate
manual_review_accept_rate
manual_review_reject_rate
```

尤其要监控：

```text
same numeric stem but different suffix 被合并的数量
```

目标应该恒为 0。

每次规则发布前跑固定 hard-negative 回归集；只要出现一个明确误合并，就阻断发布。

---

## 27. pyJedAI 本身的几个工程风险/不足

### 27.1 Pandas-first

`Data` 会把数据装入 Pandas DataFrame，并做字符串化/拼接，适合研究与离线实验，不适合直接用作 10M 持续流式主处理层。

生产应该把：

```text
批处理 / 流式计算
```

与：

```text
pyJedAI benchmark adapter
```

分开。

### 27.2 两源模型

核心 Clean-Clean 数据结构天然面向 D1/D2。本需求要建立全局 reference entity registry，来源只是 member 的属性。

### 27.3 FAISS Flat

当前 `EmbeddingsNNBlockBuilding` 使用 `IndexFlatL2`，适合基准，不适合作为所有 10M 数据的高频全局 ANN 主索引。

### 27.4 默认文本 flatten

多个组件把选定属性直接 `' '.join`，这会丢失字段角色。对 reference、SKU、series 这种强语义字段不够安全。

### 27.5 similarity threshold 不是风险控制

`EntityMatching` 超阈值即加边，无法表达：

```text
强冲突一票否决
证据不足拒识
不同证据来源的可信等级
```

### 27.6 图聚类可能放大 false positive

通用 connected components 对 precision-first 场景非常危险；只有 strict entity key 产生的边才应允许传递。

---

## 28. 最终推荐架构

一句话版本：

> **用 pyJedAI 的“分阶段 ER + Blocking/候选清理 + 可评估 workflow”思想搭实验框架；生产主链改成 `品牌规范化 → reference 证据抽取 → 角色分类 → 保守规范化 → 冲突否决 → (brand, reference) 精确实体键`，ANN/图片/LLM 只服务 unresolved 候选与 reference 恢复，不直接拥有归并权限。**

生产数据流：

```text
                ┌───────────────┐
                │ 3-source feed │
                └───────┬───────┘
                        v
                ┌───────────────┐
                │ Normalize     │
                └───────┬───────┘
                        v
                ┌───────────────┐
                │ Ref Evidence  │
                │ field/text/OCR│
                └───────┬───────┘
                        v
                ┌───────────────┐
                │ Role + Strict │
                │ Canonicalize  │
                └───────┬───────┘
                        v
                ┌───────────────┐
                │ Conflict Veto │
                └────┬─────┬────┘
                     │     │
              trusted│     │unresolved
                     v     v
            ┌────────────┐ ┌────────────────┐
            │ Exact Key  │ │ pyJedAI-style  │
            │ Resolver   │ │ Candidate Layer│
            └─────┬──────┘ └───────┬────────┘
                  │                │
                  v                v
            ┌────────────┐  ┌───────────────┐
            │ Entity     │  │ re-extract /  │
            │ Registry   │  │ manual review │
            └─────┬──────┘  └───────────────┘
                  v
            ┌────────────┐
            │ Audit + GT │
            └────────────┘
```

这套架构的优势是：

1. **安全**：不依赖模糊相似度自动归并；
2. **可解释**：每次归并都有 reference 证据与 reason code；
3. **可扩展**：新来源只需接入 normalization/extraction，不改变全局实体定义；
4. **可增量**：新记录按 `(brand, ref)` O(1)/索引 lookup 归入实体；
5. **可逐步提高 recall**：后续加 OCR、ANN、LLM 都只改善 unresolved，不破坏主链 precision；
6. **可借 pyJedAI 快速做离线 benchmark**：尤其适合比较 blocking 与候选裁剪算法。

---

## 29. 参考源码

- pyJedAI README：<https://github.com/AI-team-UoA/pyJedAI/blob/main/README.md>
- `pyproject.toml`：<https://github.com/AI-team-UoA/pyJedAI/blob/main/pyproject.toml>
- 数据模型：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/datamodel.py>
- Blocking：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/block_building.py>
- Comparison Cleaning：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/comparison_cleaning.py>
- Embedding + FAISS Blocking：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/vector_based_blocking.py>
- Entity Matching：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/matching.py>
- Clustering：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/clustering.py>
- Workflow：<https://github.com/AI-team-UoA/pyJedAI/blob/main/src/pyjedai/workflow.py>
