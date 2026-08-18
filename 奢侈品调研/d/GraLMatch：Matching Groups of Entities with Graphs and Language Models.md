# GraLMatch：Matching Groups of Entities with Graphs and Language Models —— 技术实现、架构分析与跨源二奢/腕表实体匹配落地方案

> 分析对象：GraLMatch: Matching Groups of Entities with Graphs and Language Models（EDBT 2025）  
> 论文：https://arxiv.org/abs/2406.15015  
> 官方代码：https://github.com/FernandoDeMeer/GraLMatch  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

GraLMatch 最值得借鉴的并不是它使用 DistilBERT/DITTO 做 pairwise matching，而是它把**“两两匹配结果”提升为“图上的实体组一致性问题”**：一条 false-positive 边可能把两个本来正确的实体簇错误连起来，而传递闭包会把一次错误放大成大量错误匹配。因此，系统不能只看单边分数，还需要在成组后检查整体结构是否合理。

不过，GraLMatch 不能原样套到本需求。它的图清理代码把“一个真实实体在每个来源最多出现一条记录”作为重要先验，因此会把连通分量大小压缩到 `num_of_datasources` 附近。腕表二奢场景中，“同一个商品”在 Spec 中被定义为**同一 reference number / 型号**，同一平台完全可能同时存在多条同 reference 的不同在售/历史商品，所以一个 reference 分组可能有几十、几百甚至更多记录，绝不能因为分组大小超过 3 就拆开。

因此，推荐采用一个 **Reference-First + Evidence Graph + Fail-Closed（失败即拒识）** 的改造方案：

1. **reference 是唯一自动合并的主证据**：先抽取、规范化、角色识别、校验 reference，再做严格等值匹配。
2. **图结构用于一致性审计和阻断错误传播，不用于把软相似度升级成事实**。
3. 文本模型、图片 embedding、OCR、LLM 都只用于“召回、补证、冲突发现、人工复核排序”，默认不能单独产生自动合并边。
4. 自动分组只基于 `HARD_MATCH` 边；任何 reference 冲突、品牌冲突或证据来源异常都进入 `QUARANTINE/REVIEW`，不继续传递。
5. 千万级在线实现不应全量使用 NetworkX，也不应显式补齐连通分量的所有传递边；应以 **canonical reference 倒排索引 + Union-Find/DSU + 局部审计子图** 为核心。

这个方案比“训练一个很强的 pairwise classifier 然后调高阈值”更符合“**绝对不能误匹配，允许漏匹配**”的业务约束。

---

## 1. 为什么选 GraLMatch

`奢侈品文章调研.md` 对 GraLMatch 的推荐理由是：它将多源 Entity Resolution 从独立 pairwise 判定提升到实体组图匹配，并用图结构清理少量 false-positive 边，适合三源同 reference 聚类后做跨源一致性审计，避免一条错边通过传递关系污染整簇。

这与当前 Spec 的风险结构非常一致：

- 数据来自雷小安、腕表之家、奢当家多个来源；
- 数据规模 100 万–1000 万，并持续增量；
- reference 有时是结构化字段，有时埋在标题；
- 一条 reference 抽错，可能把两个型号的大量商品合成同一组；
- 业务允许漏，但不能错合并。

换句话说，本需求的核心不是“平均 F1 做高”，而是**如何限制单个错误的爆炸半径**。GraLMatch 正好提供了非常有价值的图视角。

---

## 2. GraLMatch 的问题建模

### 2.1 从 pairwise matching 到 entity group matching

传统 Entity Matching 通常对候选对 `(a, b)` 独立输出：

```text
Match(a, b) ∈ {0, 1}
```

如果最终要得到实体组，则通常对所有预测为正的 pair 做传递闭包：

```text
A == B
B == C
=> A == C
```

问题在于：只要其中一条边是 false positive，就可能把两个正确簇连接起来。

例如：

```text
真实组 1：A - B - C
真实组 2：D - E - F

若模型错误产生 C - D

传递闭包后：A,B,C,D,E,F 全部被视为同一实体
```

所以 pairwise precision 即使已经很高，放到 group matching 上仍可能出现非常明显的错误放大。

GraLMatch 的核心就是把正匹配构造成图：

```text
G = (V, E)
V = 数据记录
E = pairwise matcher 预测为正的边
```

然后利用连通分量内部的结构寻找最可疑的桥接边并删除，最后把清理后的 connected component 作为实体组。

### 2.2 论文/代码的整体流水线

官方代码中的 `Matcher.run_matching()` 很清楚地体现了三阶段：

```text
A) Blocking
   ↓
B) Pairwise Matching
   ↓
C) Graph Cleanup
```

可以抽象为：

```text
Raw multi-source records
        │
        ▼
Candidate Generation / Blocking
        │
        ▼
Language-model pairwise classifier
        │
        ▼
high-confidence positive edges
        │
        ▼
Graph construction
        │
        ▼
Minimum Edge Cut + Edge Betweenness Cleanup
        │
        ▼
Connected Components / Entity Groups
```

这个分层非常适合作为腕表方案的骨架，但每层的“证据权力”需要重新定义。

---

## 3. 官方代码级技术实现分析

官方仓库主要包含：

```text
GraLMatch/
├── datainc_code/          # GraLMatch / DistilBERT 主实现
│   ├── src/
│   │   ├── data/
│   │   ├── matching/
│   │   │   └── matcher.py
│   │   └── models/
│   └── scripts/
└── em_ditto/              # DITTO 训练与推理代码
```

### 3.1 Blocking：先大幅缩小候选空间

`matcher.py` 中实现了 token overlap 型 blocking。其做法不是遍历所有 pair，而是：

1. 把每条记录 token 化；
2. 构造稀疏 `record × token` CSR matrix；
3. 通过稀疏矩阵与转置点乘得到 token overlap；
4. 排除同来源；
5. 每条记录只保留 top-N 候选。

代码逻辑可概括为：

```python
indicators = csr_matrix(...)
lookup = indicators[i, :].dot(indicators.transpose())
lookup *= cross_source_mask
top_idx = np.argpartition(lookup, -number_of_candidates)[-number_of_candidates:]
```

另外，证券场景还利用 ISIN/CUSIP/VALOR/SEDOL 等标识符生成 `id_overlap` 候选。这一点对腕表非常重要：**reference 应对应证券标识符的角色，而不是对应普通文本字段。**

因此在腕表场景，Blocking 的优先级应变为：

```text
Brand + canonical reference exact bucket   （最高优先）
        ↓
Brand + reference candidate prefix/family
        ↓
Brand + series + token/embedding top-K     （只服务疑难召回）
```

而不是先用通用文本相似度制造大量 pair。

### 3.2 极高阈值建图

`full_data_utils.generate_matches_graph()` 的默认阈值为：

```python
threshold = 0.999
```

只把：

```python
pairwise_matches_preds['prob'] > threshold
```

的 pair 加入图。

这说明 GraLMatch 本身也非常依赖**高 precision 的第一阶段边**。图清理不是用于拯救一个随意、低阈值的 matcher；它只能作为高精度 pairwise 结果的二次防线。

这一点与 Spec 完全一致：腕表系统不能指望“图算法最后帮忙纠错”，必须在成边前就尽可能保守。

### 3.3 Minimum Edge Cut 清理大错误簇

`Matcher.graph_cleanup()` 第一阶段会检查异常大的 connected component，并对大子图执行：

```python
nx.minimum_edge_cut(subgraph)
```

直观上，如果两个内部连接较密的簇只是被少数几条错误边连起来，minimum edge cut 可以找到较小的割边集合，把两个簇分离。

图示：

```text
A--B--C
|  |  |
A2-B2-C2 ---- X ---- D2-E2-F2
                         |  |  |
                         D--E--F

X 是错误桥边时，minimum cut 很容易将其识别为结构脆弱连接。
```

### 3.4 Edge Betweenness Centrality 清理桥接边

第二阶段对仍然过大的 component 计算：

```python
nx.edge_betweenness_centrality(subgraph)
```

并反复删除 betweenness 最高的边。

边介数高，意味着大量节点间最短路径经过这条边；在“两个密集簇被一条误边连起来”的情况下，桥边往往有很高的 betweenness。

因此 GraLMatch 实际上把下面这种模式视为高危：

```text
高置信 pairwise edge
       +
强桥接结构特征
       =
优先怀疑为 false positive
```

### 3.5 最后再补传递关系

代码的 `generate_transitive_matches_graph()` 会对 connected component 中节点补齐传递关系。

这对离线评测方便，但实现上有明显的规模问题：大小为 `k` 的组件显式 clique 化需要 `O(k²)` 边。代码甚至对超过 1000 节点的 component 直接跳过显式添加，以避免内存问题。

所以在 1000 万级系统中，**不能把“所有同 reference 记录两两展开成边”作为存储模型**。

正确做法是保存：

```text
record_id -> group_id
```

以及必要的原始证据边，而不是保存一个 k 节点完全图。

---

## 4. GraLMatch 不能直接照搬的三个关键点

### 4.1 “组大小 ≤ 来源数”的假设不成立

GraLMatch 的清理逻辑中会使用 `num_of_datasources` 作为期望实体组大小的重要先验。例如，当 component 大于来源数时继续删边。

对本需求：

```text
来源数 = 3
```

但同一 reference 可能出现：

```text
雷小安：20 条
腕表之家：50 条
奢当家：8 条

总计 78 条，且全部可能真的是同一 reference
```

所以腕表系统**绝对不能因为 component > 3 就判定存在错误边**。

### 4.2 通用文本相似度不能成为最终事实证据

奢侈品标题中的相邻型号高度相似，例如：

```text
Rolex Submariner 126610LN
Rolex Submariner 126610LV
```

视觉外观和标题都可能非常相近，而业务定义明确要求 reference 一致才能判同。

更危险的是配件标题：

```text
“适配 Rolex 126610LN 表带”
```

标题里出现 `126610LN` 并不代表当前售卖商品本身就是 126610LN 腕表。

因此必须先做 **reference role classification**：

```text
PRODUCT_REFERENCE
ACCESSORY_TARGET_REFERENCE
PLATFORM_SKU
SHOP_SKU
SERIAL_NUMBER
UNKNOWN_ID
```

只有 `PRODUCT_REFERENCE` 才有资格形成自动匹配证据。

### 4.3 图片相似不能证明 reference 相同

同系列不同 reference 的表盘、表壳、表链可能极其相似。因此：

```text
image similarity = 辅助证据
image similarity ≠ reference equality
```

图片更适合做三件事：

1. OCR 表背/保卡/吊牌上的 reference；
2. 在 reference 文本证据不足时提供人工复核上下文；
3. 检测明显视觉冲突，作为 veto signal。

---

## 5. 面向 Spec 的直接落地架构

推荐架构如下：

```text
                       ┌──────────────────┐
雷小安 ───────────────►│                  │
腕表之家 ─────────────►│  Ingestion / CDC │
奢当家 ───────────────►│                  │
                       └────────┬─────────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │ Raw Record Versioning│
                    └──────────┬───────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │ Reference Extraction Pipeline   │
              │ structured / title / OCR / LLM │
              └───────────────┬────────────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │ Canonicalization + Role Verify │
              │ brand-aware rules + dictionary │
              └───────────────┬────────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
       ┌──────────────────┐       ┌────────────────────┐
       │ Hard Ref Index   │       │ Weak Candidate     │
       │ brand + ref      │       │ Retrieval top-K    │
       └─────────┬────────┘       └─────────┬──────────┘
                 │                          │
                 ▼                          ▼
       ┌──────────────────┐       ┌────────────────────┐
       │ HARD_MATCH Gate  │       │ Soft Evidence /    │
       │ fail-closed      │       │ Review Ranking     │
       └─────────┬────────┘       └─────────┬──────────┘
                 │                          │
                 └────────────┬─────────────┘
                              ▼
                 ┌──────────────────────────┐
                 │ Evidence Graph / Audit   │
                 │ conflict + bridge check  │
                 └────────────┬─────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ Group Membership / DSU │
                  │ record -> ref_group    │
                  └────────────┬───────────┘
                               │
                               ▼
                     Review / Feedback / QA
```

核心思想是：**先证明 reference，再分组；不是先判断两个商品像不像，再猜 reference。**

---

## 6. Reference 抽取与规范化层

这是整个系统最关键的一层。

### 6.1 证据来源优先级

建议保存每一个 reference candidate 的来源，而不是只保存一个最终字符串：

```text
STRUCTURED_FIELD
TITLE_REGEX
DESCRIPTION_REGEX
IMAGE_OCR
LLM_EXTRACTION
REFERENCE_DICTIONARY_MATCH
HUMAN_LABEL
```

不同来源有不同信任等级。

建议初始等级：

| 来源 | 默认信任 | 是否可单独自动成边 |
|---|---:|---|
| 人工确认 | 最高 | 是 |
| 官方/高可信结构化 reference 字段 | 很高 | 是，需通过格式与品牌校验 |
| 标题规则抽取 + reference 字典命中 | 高 | 可，建议再做角色校验 |
| 图片 OCR + 文本一致 | 高 | 可作为第二独立证据 |
| 单独标题 LLM 抽取 | 中 | 否 |
| 单独 OCR | 中低 | 否 |
| 通用模型猜测 | 低 | 否 |

### 6.2 Canonicalization

不能粗暴删除所有标点。reference 规范化必须品牌感知，因为某些品牌的点号、斜杠、后缀可能具有语义。

通用层先做：

```python
NFKC unicode normalize
uppercase
trim
normalize full-width punctuation
collapse whitespace
normalize common hyphen variants
```

然后进入 brand-specific rules：

```python
canonicalize_reference(brand, raw_ref)
```

示意：

```python
def canonicalize_reference(brand, raw_ref):
    s = nfkc(raw_ref).upper().strip()
    s = normalize_punctuation(s)
    s = normalize_spaces(s)

    if brand == "ROLEX":
        return rolex_rule(s)
    elif brand == "OMEGA":
        return omega_rule(s)
    elif brand == "CARTIER":
        return cartier_rule(s)
    return conservative_generic_rule(s)
```

原则是：**宁可把可能等价的两个形式暂时保留为不同值，也不要错误合并两个真实不同 reference。**

### 6.3 Reference 角色识别

每一个抽出的编号都必须同时输出：

```json
{
  "value": "126610LN",
  "canonical_value": "126610LN",
  "role": "PRODUCT_REFERENCE",
  "source": "TITLE_REGEX",
  "confidence": 0.998,
  "span": "...",
  "extractor_version": "ref-v3"
}
```

若标题是：

```text
“适配劳力士 126610LN 橡胶表带”
```

应输出：

```json
{
  "canonical_value": "126610LN",
  "role": "ACCESSORY_TARGET_REFERENCE"
}
```

这条 candidate 不能形成商品 reference 匹配。

---

## 7. 候选生成：千万级不能做全量 pair

1000 万条数据的全量两两比较约为：

```text
O(N²)
```

完全不可行。

但本需求有一个巨大优势：最终定义就是 reference 相同。因此主路径根本不需要 ANN 才能解决。

### 7.1 一级：精确 reference bucket

维护：

```text
(brand_id, canonical_reference) -> record_ids / group_id
```

新记录只需查询一次对应 bucket。

复杂度接近：

```text
建索引：O(N)
单条增量查询：O(1) hash / O(log N) B-tree
```

### 7.2 二级：弱 reference 候选

只有 reference 不完整、OCR 有噪声或格式不确定时，才进入弱召回：

```text
brand
+ reference prefix/suffix
+ series
+ token overlap
+ char ngram
+ embedding top-K
```

且必须在 brand partition 内做，严禁全库 ANN 后直接合并。

弱候选的结果只能进入：

```text
SOFT_CANDIDATE
```

不能自动成为 `HARD_MATCH`。

---

## 8. Match Gate：把“能否自动合并”写成显式安全策略

推荐不要输出简单的：

```text
match_probability = 0.9997
```

然后“一刀切”。

应输出结构化决策：

```json
{
  "decision": "AUTO_MATCH | REVIEW | NO_MATCH | ABSTAIN",
  "reason_codes": [],
  "hard_evidence": [],
  "conflicts": []
}
```

### 8.1 AUTO_MATCH 最低条件

保守版：

```python
def can_auto_match(a, b):
    if a.brand_id != b.brand_id:
        return False

    if not a.product_ref or not b.product_ref:
        return False

    if a.product_ref.canonical != b.product_ref.canonical:
        return False

    if a.product_ref.role != "PRODUCT_REFERENCE":
        return False
    if b.product_ref.role != "PRODUCT_REFERENCE":
        return False

    if has_hard_conflict(a, b):
        return False

    if not independent_reference_evidence_is_strong(a, b):
        return False

    return True
```

这里最重要的是：

```text
同 reference 是必要条件，但“字符串刚好相同”还不够；
必须先确认这个编号真的是当前商品的 PRODUCT_REFERENCE。
```

### 8.2 Veto / hard conflict

下列情况直接禁止自动合并：

```text
canonical brand 不同
两个高可信 PRODUCT_REFERENCE 不同
一个记录被确认是配件而另一条是整表
型号族/产品类别明确冲突
人工历史标注为 reject
结构化 reference 与 OCR/标题出现高可信相反证据
```

### 8.3 为什么 Fail-Closed 很重要

在普通推荐系统里，不确定时可以继续输出概率；本需求应该相反：

```text
证据不充分 = ABSTAIN
证据冲突   = NO_MATCH / QUARANTINE
证据强一致 = AUTO_MATCH
```

这才真正体现 precision-first。

---

## 9. Evidence Graph：借鉴 GraLMatch，但重新定义边

### 9.1 节点

节点代表具体记录版本：

```text
record_version_id
```

而不是只代表逻辑 record id。因为持续增量时标题/reference 可能发生更新，必须能够追踪某次错误是由哪个版本产生。

### 9.2 边类型

建议至少有：

```text
HARD_MATCH
SOFT_CANDIDATE
CONFLICT
HUMAN_CONFIRMED
HUMAN_REJECTED
SUPERSEDED
```

边上保存：

```json
{
  "left_id": "...",
  "right_id": "...",
  "edge_type": "HARD_MATCH",
  "reference": "126610LN",
  "evidence": ["STRUCTURED_FIELD", "TITLE_DICT"],
  "decision_rule_version": "gate-v4",
  "created_at": "..."
}
```

### 9.3 自动聚类只使用硬边

关键规则：

```text
AUTO GROUP = Connected Components(HARD_MATCH + HUMAN_CONFIRMED)
```

绝不能：

```text
Connected Components(HARD_MATCH + SOFT_CANDIDATE)
```

否则一个高相似但不同 reference 的 soft edge 就可能污染整个分组。

---

## 10. GraLMatch 式图清理如何改造成“局部异常审计器”

GraLMatch 的 minimum cut 和 edge betweenness 思路可以保留，但用途应从“自动决定真匹配”降级为：

```text
异常发现 / review priority / quarantine trigger
```

### 10.1 什么时候触发局部图审计

只对下列局部组件执行：

```text
一个组件内出现 >1 个高可信 canonical reference
一个新边会连接两个已有大组
同一组突然因为一次增量扩大很多
某条边两端来自弱提取器
人工 reject 与现有 hard group 冲突
```

### 10.2 Edge Betweenness 的用途

若一个新 edge 连接两个原本稳定的大组：

```text
Group A ===== edge X ===== Group B
```

edge X 的 betweenness 往往很高。此时可以：

```text
标记 X 为 BRIDGE_RISK
冻结该 union
进入人工复核
```

而不是先 union 再试图事后拆图。

### 10.3 Minimum Cut 的用途

对已被污染的小型局部组件，可以用 minimum cut 给出“最小可疑边集合”，辅助定位：

```text
哪些 edge 删除后，组件能恢复为 reference 一致的多个子组
```

但在腕表系统中，**拓扑证据不能推翻高可信 exact reference 证据**。它只是异常定位器。

---

## 11. 千万级实现：不要全局 NetworkX

GraLMatch 官方实现使用 NetworkX，适合研究实验和中等规模局部图，但不适合作为千万级在线主存储/主计算模型。

推荐分成两条路径：

### 11.1 主路径：Reference Index + DSU

对已经通过 `HARD_MATCH` gate 的记录：

```text
key = (brand_id, canonical_reference)
```

直接映射到稳定 `reference_group_id`。

如果仍希望用 union-find：

```text
find(record_id)
union(record_a, record_b)
```

摊还复杂度接近常数：

```text
O(alpha(N))
```

不过由于最终定义天然等价于 `(brand, canonical_ref)`，甚至可以不做复杂 union，直接把 canonical key 作为 group identity 的来源。

### 11.2 审计路径：只拉取局部子图

当发生冲突时，从边表中读取受影响 group 周边的有限子图，再用 NetworkX / igraph / graph-tool 做：

```text
connected components
bridge
edge betweenness
minimum cut
```

这样把最贵的图算法限制到很小的异常范围。

### 11.3 不保存传递闭包

不要为 1000 个同 reference 记录存：

```text
~ 1000 × 999 / 2 = 499,500 条 pair edge
```

只需要保存：

```text
1000 条 group membership
+
少量原始证据边/审计边
```

这是从研究代码走向生产必须做的变化。

---

## 12. 推荐数据模型

### 12.1 raw_record_versions

```text
record_version_id PK
source_id
source_record_id
version
raw_payload
raw_title
raw_brand
raw_reference
image_urls
fetched_at
content_hash
```

### 12.2 reference_candidates

```text
candidate_id PK
record_version_id
raw_value
canonical_value
role
brand_id
source_type
confidence
span_or_image_id
extractor_version
created_at
```

### 12.3 resolved_reference

```text
record_version_id PK
brand_id
canonical_reference
resolution_status
resolution_grade
reason_codes
resolver_version
```

`resolution_status`：

```text
VERIFIED
AMBIGUOUS
CONFLICT
MISSING
```

### 12.4 match_edges

```text
edge_id PK
left_record_version_id
right_record_version_id
edge_type
canonical_reference
score
reason_codes
decision_version
created_at
```

### 12.5 reference_groups

```text
group_id PK
brand_id
canonical_reference
group_status
member_count
created_at
updated_at
```

### 12.6 group_members

```text
group_id
record_version_id
membership_type
joined_by_rule
joined_at
```

### 12.7 review_tasks

```text
review_task_id PK
trigger_type
left_id
right_id
group_id
priority
snapshot_json
status
reviewer
result
```

### 12.8 gold_labels

```text
left_id
right_id
label
label_reason
reference_left
reference_right
annotator
created_at
```

---

## 13. 增量更新流程

Spec 明确要求持续增量，因此必须把“reference 改变”视为高危事件。

### 13.1 新增记录

```text
1. 写 raw_record_versions
2. 抽取 reference candidates
3. 规范化 + 角色识别
4. resolve reference
5. 若 VERIFIED：查询 hard ref index
6. 通过 match gate
7. 写 group membership / hard evidence
8. 做局部 invariant check
```

### 13.2 旧记录发生更新

假设：

```text
v1: 126610LN
v2: 126610LV
```

不能直接覆盖旧值。应：

```text
1. 新建 record version
2. 标记旧 membership 为 superseded
3. 重新执行 reference resolution
4. 对旧组和新组都做局部审计
5. 如果变化来自低可信 extractor，先 quarantine，不立即搬组
```

这样可以防止一次 extractor 回归让大量历史组被重新污染。

---

## 14. 组件级不变量：比“图大小”更适合腕表

GraLMatch 用 component size 作为结构约束；腕表更适合使用**语义不变量**。

建议至少实现：

### Invariant A：高可信 reference 唯一

一个自动组中：

```text
count(distinct high_confidence canonical_reference) <= 1
```

若大于 1：

```text
GROUP_STATUS = QUARANTINED
```

### Invariant B：canonical brand 唯一

```text
count(distinct verified brand_id) <= 1
```

### Invariant C：人工 reject 不可跨越

若 gold/human 数据中存在：

```text
A != B
```

任何自动传递路径都不能把 A、B 放进同一 auto-approved group。

### Invariant D：弱证据不能升级

`SOFT_CANDIDATE` 即使形成闭环，也不能自动变成 `HARD_MATCH`。

### Invariant E：新边连接两个稳定大组时先冻结

如果：

```text
size(group_a) > T
size(group_b) > T
```

且一个新边试图合并两组，默认进入审计，不直接 union。

这就是把 GraLMatch 的“桥边风险”提前到写入阶段。

---

## 15. 人工标注几百对应该怎么花

几百个标签很珍贵，不应随机抽样。

应主动覆盖最容易 false positive 的 hard negatives：

1. 同品牌、同系列、相邻 reference；
2. 只差一个字母/数字后缀；
3. 标题出现“适配/兼容/表带/盒证/配件”的 reference；
4. 平台 SKU 长得像 reference；
5. OCR 把 `0/O`、`1/I`、`5/S` 混淆；
6. 同一 reference 的不同格式写法；
7. 同外观但不同 reference；
8. 跨品牌存在相似纯数字编号；
9. 结构化字段和标题冲突；
10. 更新前后 reference 改变的记录。

### 15.1 标注目标不是训练一个万能 matcher

首批标注更适合用于：

```text
reference role classifier
reference extractor 校准
match gate 阈值验证
hard-negative regression suite
```

而不是直接训练一个“两个商品是否同款”的黑盒模型。

### 15.2 数据切分要按 reference 隔离

不要随机 pair split，否则同一 reference 很可能同时出现在训练和测试中，得到虚高结果。

推荐：

```text
train refs / validation refs / test refs 互斥
```

并额外建立 hard-negative 专用测试集。

---

## 16. 评估指标：不要让 F1 主导决策

主指标应是：

```text
Auto-Match Precision
False Positive Count
False Positive Rate per 1M decisions
Coverage / Auto-Match Rate
Abstain Rate
Review Rate
```

其中优先级建议：

```text
0 个可复现 false positive
    >
更高 coverage
    >
recall / F1
```

“绝对不能误匹配”在统计上无法仅凭几百个测试样本证明数学上的 100%，所以工程上应该靠：

```text
高门槛规则
+ fail-closed
+ 多证据交叉
+ component invariant
+ 人工复核
+ 版本化审计
```

共同降低风险，而不是声称某个模型置信度 0.9999 就“绝对安全”。

---

## 17. 三档决策策略

### Grade A：AUTO_MATCH

必须满足：

```text
brand verified equal
canonical product reference exact equal
reference role = PRODUCT_REFERENCE
至少一侧为高可信结构化/人工证据
另一侧至少达到字典校验 + 高可信抽取，或存在第二独立证据
无任何 hard conflict
不违反 group invariants
```

### Grade B：REVIEW

典型：

```text
两侧抽出的 canonical reference 相同
但一侧或两侧来自弱标题/LLM/OCR
```

### Grade C：SOFT_CANDIDATE / ABSTAIN

典型：

```text
reference 缺失
只有视觉相似
只有文本语义相似
reference 只有模糊前缀
```

### NO_MATCH / CONFLICT

```text
两个高可信 reference 不同
或品牌/品类存在硬冲突
```

---

## 18. 一个三源示例

### 记录 A：雷小安

```text
标题：劳力士 潜航者 126610LN 黑水鬼 全套
结构化 reference：空
```

规则抽取：

```text
ROLEX / 126610LN / PRODUCT_REFERENCE
source=TITLE_REGEX+DICT
```

### 记录 B：腕表之家

```text
品牌：Rolex
reference：126610LN
```

结构化字段 + 品牌字典校验通过。

A 与 B：

```text
brand equal
canonical ref equal
role valid
B 是高可信结构化证据
A 有规则+字典证据
=> Grade A / HARD_MATCH
```

### 记录 C：奢当家

```text
标题：适配劳力士 126610LN 黑色橡胶表带
```

抽取：

```text
126610LN
role=ACCESSORY_TARGET_REFERENCE
```

即便文字和图片都与 A/B 很相似：

```text
=> 不得 HARD_MATCH
=> NO_MATCH 或独立配件实体
```

这类 case 正是简单“reference 字符串出现相同就合并”会犯的致命错误。

---

## 19. 生产技术栈建议

技术栈不限时，可以分“最小可落地”和“规模化”两档。

### 19.1 MVP

```text
Python
PostgreSQL
Object Storage / Parquet
FastAPI worker
Redis（可选缓存）
OpenSearch（弱候选检索，可后上）
PaddleOCR / 云 OCR / 自选 OCR
```

PostgreSQL 先承载：

```text
reference index
group membership
edge audit
review task
```

只要 schema 和索引设计合理，首轮验证不需要先建设复杂大数据平台。

### 19.2 1000 万级持续增量

可演进为：

```text
Kafka/Pulsar        -> 增量事件
Flink/Spark         -> 批流抽取/规范化
Iceberg/Delta       -> 原始与版本化历史
PostgreSQL/CockroachDB -> 控制面、group、review
OpenSearch          -> 弱 reference / title candidate retrieval
ClickHouse          -> 指标、审计、离线分析
GPU inference service -> OCR / extractor / multimodal model
```

但主判定逻辑仍然应是确定性的 reference gate，而不是把基础设施复杂度等同于匹配质量。

---

## 20. 推荐服务拆分

```text
reference_matcher/
├── ingestion/
│   ├── source_adapters.py
│   └── versioning.py
├── reference/
│   ├── extract_structured.py
│   ├── extract_title.py
│   ├── extract_ocr.py
│   ├── role_classifier.py
│   ├── canonicalizer.py
│   └── resolver.py
├── blocking/
│   ├── exact_index.py
│   └── weak_retrieval.py
├── evidence/
│   ├── hard_gate.py
│   ├── conflicts.py
│   └── scoring.py
├── graph/
│   ├── membership.py
│   ├── invariants.py
│   └── local_audit.py
├── review/
│   ├── task_builder.py
│   └── feedback.py
└── evaluation/
    ├── hard_negative_suite.py
    ├── precision_metrics.py
    └── drift_monitor.py
```

其中最重要的接口不是：

```python
predict_match(a, b) -> float
```

而应是：

```python
resolve_reference(record) -> ResolvedReference

decide_edge(a, b) -> MatchDecision

validate_group(group_id) -> InvariantResult
```

这样系统的安全边界才清晰、可测试、可审计。

---

## 21. GraLMatch 与建议方案对比

| 维度 | GraLMatch 原方案 | 腕表落地改造 |
|---|---|---|
| 基本实体定义 | 多源记录对应同一现实实体 | 同一 `brand + canonical reference` |
| 候选生成 | ID overlap + token overlap | exact reference index 优先，弱检索兜底 |
| 主 matcher | LM pairwise classifier | deterministic reference gate 为主 |
| 成边依据 | 高概率 pairwise match | verified PRODUCT_REFERENCE exact match |
| 图作用 | 清理 false positive 后聚类 | 一致性审计、冲突阻断、异常定位 |
| 组大小先验 | 受来源数约束 | 不限制，可一来源多记录 |
| 图计算 | NetworkX connected components/cut/centrality | 主路径 DSU/index，异常时局部图计算 |
| 传递闭包 | 可显式补边 | 不显式补全，存 membership |
| 图片 | 非核心 | OCR/冲突/复核辅助，不直接成硬边 |
| 不确定结果 | 模型阈值 | ABSTAIN / REVIEW，fail-closed |
| 目标 | group matching 准确性 | 极端 precision，阻止任何错误扩散 |

---

## 22. 分阶段落地计划

### Phase 1：先做一个几乎不会错的规则系统

只支持：

```text
高可信结构化 reference
brand normalization
canonicalization
exact key grouping
完整 audit log
```

此阶段 recall 可以非常低。

目标：验证三平台数据中 structured reference 的质量和错误类型。

### Phase 2：标题 reference 抽取

加入：

```text
brand-aware regex
reference dictionary
role classifier
配件/兼容语境识别
几百对 gold labels
```

只把通过高标准校验的标题抽取升级为 Grade A。

### Phase 3：OCR 与图审计

加入：

```text
图片 OCR
独立证据交叉验证
component invariants
bridge-risk / minimum-cut 局部审计
```

### Phase 4：软模型提高 coverage

最后再加入：

```text
text encoder
multimodal embeddings
LLM extraction
ANN top-K
```

但这些模块默认只能推动：

```text
ABSTAIN -> REVIEW
```

而不是：

```text
ABSTAIN -> AUTO_MATCH
```

除非经过专门的高精度验证和明确 gate 放权。

---

## 23. 最值得从 GraLMatch 直接复用的思想与代码

### 可以直接借鉴

1. **Blocking → Pairwise Evidence → Graph Cleanup** 的分层思想；
2. 将 pairwise positive 视为图边，而不是直接视为最终实体组；
3. 保存 `match_type`、概率和被删除边，建立可审计流水；
4. 用 bridge / edge betweenness / minimum cut 发现“两个正常簇之间的可疑桥边”；
5. 高 precision 第一阶段优先于盲目追求 recall；
6. 小候选集 blocking 避免全量笛卡尔积。

### 不应直接复用

1. “component 大于来源数就持续删边”的规则；
2. 以通用 LM pairwise probability 作为最终自动合并权威；
3. 用 NetworkX 维护整个千万级在线图；
4. 显式生成所有 transitive edges；
5. 允许软相似度通过传递关系升级成同 reference 事实。

---

## 24. 最终建议

如果只选择一个可落地的核心方案，建议实现以下最小闭环：

```text
① brand normalization
② reference candidate extraction
③ reference role classification
④ brand-aware canonicalization
⑤ exact reference hard index
⑥ fail-closed match gate
⑦ group membership + semantic invariants
⑧ local graph audit
⑨ manual review + gold feedback
⑩ versioned incremental recomputation
```

其中 GraLMatch 的价值主要落在 `⑦/⑧`：它提醒我们**一个系统即使 pairwise precision 很高，也必须防止少数 false-positive 边通过传递关系放大**。

但在当前腕表需求中，更强的先验其实已经存在——业务明确给出了“同一 reference number 才算同一个商品”。因此最优架构不是让模型学习“像不像同一商品”，而是把模型能力集中在最难且真正需要学习的部分：

```text
reference 在哪里？
这个编号是什么角色？
不同写法如何安全规范化？
证据是否足够强？
是否存在冲突？
```

最终自动匹配层应尽量保持简单、确定、可解释：

```text
verified brand + verified canonical product reference
                  ↓
               exact match
                  ↓
         semantic invariant check
                  ↓
             AUTO_MATCH
```

其他所有情况，都宁可拒识或人工复核。

这才是 GraLMatch 的图一致性思想与当前 Spec“precision 极端优先”约束结合后，最适合直接落地的生产方案。
