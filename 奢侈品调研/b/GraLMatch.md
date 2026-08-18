# GraLMatch：从“记录两两匹配”改造成 Reference 实体 + 证据图的 precision-first 二奢匹配架构

- 分析人：b
- 调研文章 / 项目：GraLMatch: Matching Groups of Entities with Graphs and Language Models
- 论文：https://arxiv.org/abs/2406.15015
- 代码：https://github.com/FernandoDeMeer/GraLMatch
- 论文状态：EDBT 2025 Research Paper
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么选择 GraLMatch，以及它与前面调研的差异

当前 Spec 的核心约束是：

1. 数据来自雷小安、腕表之家、奢当家三个来源；
2. 总量约 100 万～1000 万，并且持续增量；
3. “同一个商品”被定义为 **同一 reference number / 型号**，并不是同一只物理腕表；
4. reference 有时在独立字段，有时埋在标题，图片也可用；
5. **绝对不能误匹配，precision 优先到极致，可以漏匹配**；
6. 可以人工标注几百对黄金数据。

`b/TransClean.md` 已经分析过“如何用传递一致性发现 pairwise false positive”。本次不重复 TransClean 的迭代清洗逻辑，而重点研究它的前身 GraLMatch：

> 一个多源实体匹配系统从 Blocking、Pairwise Matcher 到 Graph Cleanup，具体怎样组织成完整工程；以及为什么这种“记录节点 + 匹配边 + 连通分量”的设计在当前 reference 需求里 **只能借鉴、不能直接照搬**。

GraLMatch 对当前需求最重要的启发不是 DistilBERT，而是两个工程结论：

- **pairwise 看起来很准，不代表最终 group matching 安全。** 一条 false-positive 边可以通过传递闭包污染整组；
- **最终 precision 的决定权必须高于 recall。** 论文在大规模组匹配实验中明确观察到，更高 pairwise precision 的模型往往得到更好的最终组匹配结果，即便其 recall 更低、训练数据更少。

但它还有一个对当前项目非常关键的反例：论文自己的 WDC Products 实验指出，原始 Graph Cleanup 不适合“实体组大小高度不均匀”的商品场景。当前需求恰好属于这种情况——同一个 Rolex reference 可能在一个平台上就有几十、几百条在售/历史商品，因此 **不能把 group size 限制为数据源数量 3**。

所以本次结论是：

> **不要用 GraLMatch 直接做“商品记录聚类”；要把它改造成“Reference 实体解析 + 证据图安全审计”。**

这样才能同时满足千万级规模、同平台多记录、reference 精确等值和极端 precision-first。

---

## 2. GraLMatch 原始系统到底怎么工作

论文把端到端流程拆成四步：

```text
Raw Records
   |
   v
Blocking
   |
   v
Pairwise Matcher (DistilBERT / DITTO)
   |
   v
Match Graph G=(V,E)
   |
   v
GraLMatch Graph Cleanup
   |
   v
Connected Components => Entity Groups
```

### 2.1 Blocking：先避免 O(N²)

论文首先承认多源实体匹配不可能直接全量两两比较。

如果记录数为 `N`，全比较为：

```text
N * (N - 1) / 2
```

1000 万记录显然不可执行，因此先 Blocking 生成少量候选。

项目代码 `datainc_code/src/matching/matcher.py` 里可以看到两类典型 blocking：

#### A. Identifier overlap

证券数据会检查 ISIN / CUSIP / VALOR / SEDOL 等 identifier 是否重叠，再把命中的记录放进候选集。

这和当前需求中的 reference 极其相似：

```text
金融证券：ISIN / CUSIP ...
腕表商品：brand + reference number
```

它说明强 identifier 最适合作为 blocking / hard evidence，而不是让文本模型覆盖它。

#### B. Token overlap

公司数据会把 tokenized records 转成稀疏矩阵：

```text
records × tokens => scipy csr_matrix
```

然后通过稀疏矩阵点积计算 token overlap，为每条记录取 top-k 候选，并排除同数据源记录。

这是一种比全量 pair 更现实的文本候选生成方式。

但是对当前需求而言，token blocking 只能用于 **reference 缺失时的人工复核候选**，不能进入自动合并主路径。

原因很简单：

```text
Rolex 126334
Rolex 126300
```

标题 token、品牌、系列、外观都可能极像，但 reference 不同就必须是 NoMatch。

---

## 3. Pairwise Matcher：论文为什么用 DistilBERT

GraLMatch 原论文把两条 record 串成文本后，使用 DistilBERT 做二分类：

```text
(record_a, record_b) -> Match / NoMatch
```

代码中的 `Matcher.run_matching()` 很清楚地体现了完整流程：

```text
A) blocking()
B) model.test() -> pairwise_matches_preds.csv
C) pre_cleanup()
D) graph_cleanup()
```

模型预测保存的不只是二值结果，还有：

```text
lid
rid
prob
match_type
```

其中 `match_type` 会记录候选来自 identifier overlap、text match 等哪种路径。

这点非常值得当前项目直接借鉴：

> **不要只存最后的 canonical reference；必须把“为什么得出这个 reference”的 provenance 一起保存。**

例如：

```json
{
  "listing_id": "lx_123",
  "brand": "ROLEX",
  "canonical_ref": "126334",
  "evidence": [
    {"type": "structured_field", "raw": "126334", "weight": "hard"},
    {"type": "title", "raw": "劳力士126334蓝盘", "span": "126334", "weight": "medium"}
  ]
}
```

如果日后发生误匹配，可以明确追溯：

- 是源字段写错；
- 标题抽取错；
- normalization 把两个 reference 合并了；
- OCR 误读；
- 还是品牌归一化错。

---

## 4. GraLMatch 的 Graph Cleanup 技术实现

GraLMatch 把正向 pairwise 预测组成无向图：

```text
G = (V, E)
V = records
E = predicted Match edges
```

如果：

```text
A -- B -- C
```

即使没有直接预测 `A-C`，最终 connected component 仍意味着：

```text
A == B == C
```

所以一条 false-positive bridge 的影响不是“错一对”，而可能是“合并两个簇”。

### 4.1 代码里的高置信阈值

`Matcher.pre_cleanup()` 默认：

```python
threshold = 0.999
```

即 pairwise probability 要到 0.999 才进入 match graph。

这个细节符合 precision-first 思路：

> 图清洗是保险丝，不是让一个低 precision matcher 放宽阈值后再靠图算法补救。

当前系统也应该同样处理：先让进入自动匹配路径的 reference assignment 本身非常保守，再做全局一致性检查。

### 4.2 Minimum Edge Cut

代码中第一轮 Cleanup：

```python
while component_size > 5 * num_of_datasources:
    min_edge_cut = nx.minimum_edge_cut(subgraph)
    remove(min_edge_cut)
```

直觉是：如果一个大 connected component 实际由两个密集实体簇通过少数边连接，那么 minimum edge cut 容易把“桥”找出来。

### 4.3 Edge Betweenness Centrality

第二轮 Cleanup：

```python
while component_size > num_of_datasources:
    e = edge_with_max_betweenness_centrality
    remove(e)
```

Edge Betweenness Centrality 衡量某条边处于多少最短路径上。

连接两个大簇的 bridge 会被大量最短路径经过，因此很可能具有较高中心性。

论文给出的复杂度量级是：

```text
O(mn)
```

其中 `n` 是 component 节点数，`m` 是边数。

这也说明为什么它只适合作为候选图清理，而不适合让千万级数据形成巨型 component 后再处理。

### 4.4 最终再补传递边

Cleanup 完成后，代码调用：

```python
generate_transitive_matches_graph(matches_graph, True)
```

把每个最终 component 内的隐式匹配关系补齐，再输出：

```text
post_graph_cleanup_matches.csv
graph_cleanup_deleted_edges.csv
```

这种“保留 deleted edges 作为审计日志”的做法也值得当前项目直接采用。

---

## 5. GraLMatch 最重要的实验结论

论文实验有一个非常适合当前 Spec 的结果：

> **高 pairwise precision 比高 recall 更重要，尤其在大规模 entity group matching 中。**

原因是 false positive 会造成传递扩散，而 false negative 通常只造成漏匹配。

这与当前产品约束完全同向：

```text
False Positive = 不可接受
False Negative = 可接受
```

论文还展示了一个看似反直觉的现象：

- 用更少训练样本得到的 DistilBERT 版本，某些大规模实验里反而因为 precision 更高，最终 group matching 更好；
- 加更多训练数据、提高 pairwise recall，并不保证最后结果更好。

这说明当前几百条人工标注不应追求“把模型喂饱”，而应优先构造：

- 同品牌同系列相邻 reference hard negative；
- 标题中出现多个编号的冲突样本；
- 平台 SKU / 店铺货号冒充 reference 的样本；
- OCR `0/O`、`1/I`、`5/S` 混淆；
- reference 是配件适配型号而不是当前商品型号的样本。

也就是：**标难例，不标随机样本。**

---

## 6. 为什么 GraLMatch 原版不能直接用于当前需求

这是本次分析最重要的部分。

### 6.1 原算法隐含“一来源一记录”假设

论文明确把：

```text
mu = number_of_datasources
```

并把 component 清理到最多 `num_of_datasources` 个节点。

在其金融数据里，一个真实实体通常每个数据源至多有一条 record，所以合理。

当前项目完全不同。

例如 reference `126334`：

```text
雷小安：80 条商品
腕表之家：120 条商品
奢当家：45 条商品
```

正确实体组大小应该是：

```text
245
```

而不是：

```text
3
```

如果照搬 GraLMatch，算法会把一个正确 reference 簇错误切碎。

### 6.2 论文自己的 WDC Products 实验已经暴露这个问题

论文在 WDC Products 上明确指出：

> Graph Cleanup 不适合需要发现 heterogeneous group sizes 的匹配设置。

当前需求比 WDC 更明确：reference 就是 group key，而且 group size 天生不固定。

所以不能继续把问题建模成：

```text
record <-> record pair classification
```

应该改成：

```text
record -> canonical reference entity assignment
```

这一步改变以后，整个系统会简单很多，同时 precision 更容易做硬约束。

---

# 7. 推荐的目标架构：Reference Entity Resolution，不做全量 Pairwise EM

## 7.1 核心实体模型

建议建立显式的 `ReferenceEntity`：

```text
ReferenceEntity
---------------
brand_id
canonical_reference
reference_entity_id
status
created_at
```

唯一键：

```sql
UNIQUE (brand_id, canonical_reference)
```

每条商品记录不是直接与其他商品连边，而是做：

```text
Listing ----ASSIGNED_TO----> ReferenceEntity
```

因此两个跨源商品是否是同一商品，只需要判断：

```text
listing_a.reference_entity_id == listing_b.reference_entity_id
```

而 `reference_entity_id` 只有在高置信 reference 抽取、校验、规范化后才生成/绑定。

这样从根上消除了：

```text
10M records -> O(N²) pairwise comparison
```

主路径变为近似：

```text
10M records -> O(N) extraction + hash/index lookup
```

---

# 8. 落地流水线

```text
                +---------------------+
                | Raw listing ingest  |
                +----------+----------+
                           |
                           v
                +---------------------+
                | Brand canonicalizer |
                +----------+----------+
                           |
                           v
              +---------------------------+
              | Reference candidate       |
              | extraction                |
              | - structured field        |
              | - title parser            |
              | - OCR (auxiliary)         |
              +-------------+-------------+
                            |
                            v
              +---------------------------+
              | Brand-aware canonicalizer |
              | + validator               |
              +-------------+-------------+
                            |
                            v
              +---------------------------+
              | Evidence Gate             |
              | ACCEPT / REVIEW / REJECT  |
              +-------------+-------------+
                            |
                 ACCEPT only|
                            v
              +---------------------------+
              | ReferenceEntity upsert    |
              | key=(brand, canonicalRef) |
              +-------------+-------------+
                            |
                            v
              +---------------------------+
              | Evidence Graph Audit      |
              | conflict => quarantine    |
              +---------------------------+
```

---

## 8.1 Step A：品牌先规范化

Reference 不能脱离品牌独立匹配。

推荐先得到：

```text
canonical_brand_id
```

例如：

```text
劳力士
Rolex
ROLEX
劳力士（ROLEX）
    -> BRAND_ROLEX
```

最终匹配键永远是：

```text
(brand_id, canonical_reference)
```

禁止仅用：

```text
canonical_reference
```

因为不同品牌可能存在相似编号格式。

---

## 8.2 Step B：Reference Candidate Extraction

不要让一个模型直接输出唯一答案，而是先保留候选和证据。

建议优先级：

### Tier 1：结构化 reference 字段

如果来源已有独立字段：

```text
source.reference_number
source.model_no
source.ref
```

它是最高优先级候选，但仍需做：

- 编号角色判断；
- 品牌格式校验；
- canonicalization；
- 与标题 / OCR 的冲突检测。

不能假设“字段名叫型号就一定没脏数据”。

### Tier 2：标题抽取

标题只做 span extraction：

```text
劳力士 日志 126334 蓝盘 41mm
              ^^^^^^
```

输出：

```json
{
  "raw_candidate": "126334",
  "source": "title",
  "start": 6,
  "end": 12
}
```

而不是让 LLM 自由生成型号。

### Tier 3：图片 OCR

图片仅用于：

- 表背；
- 保卡；
- 吊牌；
- 证书；
- 标签。

视觉相似度 **不能** 作为“同 reference”的直接证据。

同系列不同 reference 的腕表可能肉眼高度相似，使用 embedding nearest-neighbor 自动合并会直接破坏 precision-first。

OCR 也默认不能单独 ACCEPT，只作为独立 corroboration。

---

# 9. Canonicalization：只能做“无语义损失”归一化

这是最容易制造系统性误匹配的模块。

建议分两层。

## 9.1 全局安全 normalization

可以做：

```text
Unicode NFKC
trim
uppercase
统一全角/半角
标准化明显的空格
```

## 9.2 品牌专属 normalization profile

例如：

```python
normalize_reference(brand_id, raw_ref)
```

由 brand profile 决定：

- `-` 是否可去；
- `/` 是否有语义；
- `.` 是否有语义；
- 字母前后缀能否忽略；
- 空格是否可删除；
- 大小写是否等价。

### 禁止的做法

禁止使用模糊字符串距离自动把：

```text
126334
126300
```

或：

```text
116500LN
116500
```

归成同一个 canonical reference。

Levenshtein / embedding / LLM “看起来是同款”最多只能进入 REVIEW，绝不能产生自动 alias。

---

# 10. Evidence Gate：把“高 precision”写成硬规则

建议每个 listing 生成：

```text
ReferenceDecision
-----------------
listing_id
brand_id
candidate_ref
canonical_ref
decision
reason_code
extractor_version
normalizer_version
```

其中：

```text
decision ∈ {AUTO_ACCEPT, REVIEW, REJECT}
```

## 10.1 推荐 AUTO_ACCEPT 条件

可以先采用非常保守的规则：

### Rule A：结构化字段硬证据

```text
brand 已确定
AND structured_ref 非空
AND 通过 brand-specific format validator
AND 不与 title/OCR 的高置信候选冲突
=> AUTO_ACCEPT
```

### Rule B：标题 + 已知 reference catalog

```text
brand 已确定
AND title 精确抽到唯一 reference
AND reference 存在于该 brand 已验证 reference catalog
AND 没有第二个冲突 reference
=> AUTO_ACCEPT
```

### Rule C：双独立证据

```text
structured/title/OCR 中至少两个独立来源
给出相同 canonical reference
AND 无冲突
=> AUTO_ACCEPT
```

## 10.2 一律 REVIEW 的情况

```text
标题中出现两个不同 reference
structured_ref 与 title_ref 冲突
OCR 与结构化字段冲突
只有模糊相似候选
reference 格式不符合品牌规则
品牌不确定
型号可能是配件适配对象
```

不要试图用一个“总 confidence > 0.95”把这些冲突压平。

在 precision-first 系统里：

> **冲突是 veto，不是减分项。**

---

# 11. GraLMatch 思想如何真正改造后落地：Reference Evidence Graph

原 GraLMatch：

```text
Record --match--> Record
```

建议改成：

```text
                      +--> StructuredFieldEvidence
                     /
Listing --> CandidateRef --> TitleEvidence
                     \
                      +--> OCREvidence

CandidateRef --> ReferenceEntity
```

也可以理解为二部/多部证据图：

```text
Listing Node
   |
   | evidence edges
   v
Raw Reference Candidate Nodes
   |
   | normalization edge
   v
Canonical Reference Entity
```

### 11.1 这里的“Graph Cleanup”不再靠 group size

不要使用：

```text
component_size > num_sources
```

而改成一致性约束：

#### Constraint 1：一个 listing 最多只能 AUTO_ACCEPT 一个 canonical ref

如果：

```text
Listing X -> 126334
Listing X -> 126300
```

则整个 assignment 进入 REVIEW。

#### Constraint 2：一个 canonical ref component 不能包含两个互斥的“可信 structured ref”

例如 normalization 规则错误地把：

```text
116500LN -> 116500
116500   -> 116500
```

合到一起，但业务确认后缀 `LN` 有区分意义。

这时应当直接：

```text
quarantine normalization rule
rollback affected assignments
```

#### Constraint 3：低信任边不能桥接两个 hard-reference 簇

如果出现：

```text
[大量 structured=126334]
           |
       title fuzzy edge
           |
[大量 structured=126300]
```

GraLMatch 的 bridge 思想在这里仍然有效：

> 那条 title/fuzzy 边应该被当成高风险 bridge，而不是让两个 reference 簇被传递合并。

但是实际落地时不必运行昂贵的全局 minimum-cut；因为我们已经知道 hard-reference identity，可以直接用规则 veto：

```text
hard_ref_a != hard_ref_b => edge forbidden
```

规则比 graph heuristic 更安全、更快。

---

# 12. 数据表设计

## 12.1 `reference_entity`

```sql
CREATE TABLE reference_entity (
    id BIGSERIAL PRIMARY KEY,
    brand_id BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, canonical_reference)
);
```

## 12.2 `listing_reference_evidence`

```sql
CREATE TABLE listing_reference_evidence (
    id BIGSERIAL PRIMARY KEY,
    source TEXT NOT NULL,
    source_listing_id TEXT NOT NULL,
    brand_id BIGINT,
    evidence_type TEXT NOT NULL,
    raw_value TEXT,
    extracted_value TEXT,
    canonical_value TEXT,
    evidence_payload JSONB,
    extractor_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`evidence_type`：

```text
structured_ref
title_ref
ocr_ref
catalog_hit
manual_label
```

## 12.3 `listing_reference_assignment`

```sql
CREATE TABLE listing_reference_assignment (
    source TEXT NOT NULL,
    source_listing_id TEXT NOT NULL,
    reference_entity_id BIGINT REFERENCES reference_entity(id),
    decision TEXT NOT NULL,
    reason_code TEXT NOT NULL,
    extractor_version TEXT NOT NULL,
    normalizer_version TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source, source_listing_id)
);
```

状态：

```text
AUTO_ACCEPT
REVIEW
REJECT
```

## 12.4 `reference_decision_audit`

所有决策变化都追加写，不覆盖历史：

```text
old_ref
new_ref
old_decision
new_decision
reason
rule_version
operator
created_at
```

这相当于 GraLMatch 的 `graph_cleanup_deleted_edges.csv` 在生产系统中的升级版。

---

# 13. 自动匹配查询会变得非常简单

一旦两个 listing 已被安全绑定：

```sql
SELECT a.source, a.source_listing_id,
       b.source, b.source_listing_id
FROM listing_reference_assignment a
JOIN listing_reference_assignment b
  ON a.reference_entity_id = b.reference_entity_id
WHERE a.decision = 'AUTO_ACCEPT'
  AND b.decision = 'AUTO_ACCEPT'
  AND a.source <> b.source;
```

实际上很多业务根本不需要生成 pair 表。

只需查询：

```text
ReferenceEntity 126334
  - 雷小安 listings...
  - 腕表之家 listings...
  - 奢当家 listings...
```

这比 materialize 数十亿 pair 更合理。

---

# 14. 增量更新方案

当前数据持续更新，建议所有流程都做幂等 upsert。

唯一输入 key：

```text
(source, source_listing_id)
```

再保存：

```text
content_hash
extractor_version
normalizer_version
```

### 新商品

```text
ingest
-> extract
-> validate
-> assign reference entity
```

### 商品内容变化

只有当：

```text
reference field / title / relevant image hash / brand
```

发生变化时，才重新跑 reference pipeline。

### 规则升级

如果：

```text
normalizer_version: rolex_v3 -> rolex_v4
```

只重跑该品牌、受影响 raw pattern 的记录，不做全库重算。

---

# 15. 千万级性能设计

主路径不需要 GPU pairwise inference。

建议：

```text
CPU extraction workers
+ brand-specific regex / parser
+ PostgreSQL / MySQL indexed entity table
+ batch upsert
```

如果 OCR 量很大：

```text
OCR 独立异步队列
```

但 OCR 结果回来前，listing 可以保持：

```text
REVIEW / PENDING_AUX_EVIDENCE
```

不要为了等待图片证据阻塞全部结构化 reference 数据。

### 复杂度

原始 pairwise：

```text
O(N²) candidate universe
```

改造后：

```text
Extraction: O(N)
Reference lookup/upsert: O(log R) 或 hash O(1)
Group query: indexed lookup
```

其中 `R` 是 reference entity 数量，远小于 listing 数量。

---

# 16. 几百条人工标注应该怎么花

不建议随机抽 500 个 pair。

建议把预算用来标 **最可能产生 false positive 的边界**。

一个可执行分配：

```text
150：同品牌同系列、只差 1~2 个字符的 reference hard negatives
100：标题多编号 / 平台 SKU / 店铺货号 / reference 混杂
100：structured field 与 title 冲突
 75：OCR 字符混淆与低清图片
 50：配件 / 表带 / 盒证标题中出现“适配 reference”
 25：随机 sanity check
```

这些标签优先用于：

1. 验证 extractor；
2. 校准 brand-specific validator；
3. 发现 normalization 规则 bug；
4. 构建自动化 regression test；
5. 评估 AUTO_ACCEPT 区域 precision。

而不是首先训练一个大 matcher。

---

# 17. Precision-first 评测指标

不要把总 F1 当主 KPI。

建议至少分开统计：

```text
AUTO_ACCEPT precision
AUTO_ACCEPT coverage
REVIEW rate
reference extraction exact accuracy
conflict rate
brand-specific FP count
```

最关键指标：

```text
FP_auto_accept = 0 on gold / audit sample
```

另外要单独构造 hard-negative benchmark：

```text
126334 vs 126300
116500LN vs 116500
相同系列不同尺寸
相同表款不同材质后缀
商品本体 vs 兼容配件
```

随机测试集的 99.9% precision 不够有意义，因为真正事故集中在这些相邻 reference 上。

---

# 18. 一段可直接实现的决策伪代码

```python
def resolve_reference(listing):
    brand = canonicalize_brand(listing)
    if brand is None:
        return review("BRAND_UNKNOWN")

    evidences = []

    if listing.structured_ref:
        evidences += extract_structured_ref(listing.structured_ref, brand)

    evidences += extract_title_refs(listing.title, brand)

    if listing.ocr_result:
        evidences += extract_ocr_refs(listing.ocr_result, brand)

    valid = [
        e for e in evidences
        if validate_reference_format(brand, e.canonical_ref)
    ]

    hard = unique_refs(valid, evidence_type="structured_ref")
    all_refs = unique_refs(valid)

    # conflict = veto
    if len(hard) > 1:
        return review("MULTIPLE_HARD_REFERENCES")

    if len(hard) == 1:
        ref = only(hard)
        if has_conflicting_high_confidence_ref(valid, ref):
            return review("EVIDENCE_CONFLICT")
        return auto_accept(ref, "STRUCTURED_REFERENCE")

    # No structured hard ref
    if len(all_refs) != 1:
        return review("AMBIGUOUS_OR_MISSING_REFERENCE")

    ref = only(all_refs)

    if catalog_verified(brand, ref) and title_uniquely_supports(valid, ref):
        return auto_accept(ref, "TITLE_PLUS_CATALOG")

    if independent_evidence_count(valid, ref) >= 2:
        return auto_accept(ref, "TWO_INDEPENDENT_EVIDENCES")

    return review("INSUFFICIENT_HARD_EVIDENCE")
```

重点不是具体分数，而是：

```text
冲突 => REVIEW
证据不足 => REVIEW
模糊相似 => REVIEW
只有明确 reference => AUTO_ACCEPT
```

---

# 19. 与 GraLMatch 代码的对应关系

可以把论文代码里的概念映射为：

| GraLMatch | 当前项目 |
|---|---|
| record node | listing node |
| identifier overlap blocking | reference candidate extraction |
| text overlap blocking | REVIEW 候选生成 |
| pairwise DistilBERT | reference extractor / verifier |
| prediction probability | evidence trust level |
| `match_type` | evidence provenance |
| match graph | reference evidence graph |
| minimum edge cut | 冲突 reference bridge veto / 审计 |
| betweenness cleanup | 可疑 normalization / alias 边审计 |
| connected component | ReferenceEntity |
| deleted edge log | decision audit log |

其中最重要的变化是：

```text
connected component 不再由“模型说 Match 的 record-record 边”定义
```

而是由：

```text
严格的 (brand_id, canonical_reference) 实体键
```

定义。

图算法只负责 **找异常、做审计、隔离风险**，不负责创造 reference identity。

---

# 20. 可直接落地的 MVP 边界

第一版完全可以不训练 pairwise Transformer。

只做以下能力就能形成可用且安全的生产闭环：

1. 三源字段 mapping；
2. brand canonicalization；
3. structured reference extraction；
4. title reference span extraction；
5. brand-specific normalization / validator；
6. `(brand, canonical_ref)` ReferenceEntity 表；
7. AUTO_ACCEPT / REVIEW / REJECT evidence gate；
8. 冲突审计表；
9. 人工 review 回流；
10. hard-negative regression tests。

图片 OCR 可以在第二层逐渐增加 coverage。

LLM / Transformer 也应该只放在：

```text
reference 缺失 / 标题复杂 / REVIEW 队列
```

而不是主自动匹配路径。

---

# 21. 最终建议

GraLMatch 证明了一件对当前需求非常重要的事：

> **多源实体匹配真正危险的不是漏一条边，而是错一条边后通过传递关系污染整组。**

但当前“同商品 = 同 reference”的定义比普通 Entity Matching 更强，因此没必要继续把问题做成通用 pairwise ML。

最合适的落地方案是：

```text
先把 reference number 变成一等实体（ReferenceEntity）
        +
保留每次抽取/归一化的证据与 provenance
        +
只允许硬规则确认的 listing 绑定到 ReferenceEntity
        +
模糊模型、OCR、图片只增加证据，不拥有最终合并权
        +
用 GraLMatch 的图级思想检查冲突和高风险 bridge
```

这样既保留 GraLMatch 对 false-positive 传递污染的核心洞见，又避开它“不适合 heterogeneous group size”的原生限制。

对于 100 万～1000 万级别数据，这个设计还把主计算从 pairwise `O(N²)` 降为 reference extraction + indexed lookup，工程复杂度、推理成本和误匹配面都会显著降低。

**一句话落地原则：**

> **Reference 决定 identity；模型只负责找 Reference，图只负责阻止错误扩散。**

---

## 参考资料

- GraLMatch paper: https://arxiv.org/abs/2406.15015
- GraLMatch code: https://github.com/FernandoDeMeer/GraLMatch
- 核心 matcher / graph cleanup 实现：`datainc_code/src/matching/matcher.py`
- 当前需求：Notion `调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）`
