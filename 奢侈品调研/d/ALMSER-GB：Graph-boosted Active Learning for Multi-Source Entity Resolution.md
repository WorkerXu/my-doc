# ALMSER-GB：Graph-boosted Active Learning for Multi-Source Entity Resolution

## 1. 结论先行

本次选取项目 **ALMSER-GB（Graph-boosted Active Learning for Multi-Source Entity Resolution）** 深入分析。

- 项目：https://github.com/wbsg-uni-mannheim/ALMSER-GB
- 论文：Graph-boosted Active Learning for Multi-Source Entity Resolution，ISWC 2021
- 预印本：https://www.uni-mannheim.de/media/Einrichtungen/dws/Files_Research/Web-based_Systems/pub/Primpeli-Bizer-ALMSER-ISWC2021-Preprint.pdf

它与当前 Spec 的核心约束非常匹配：**多来源实体解析、标注预算有限、需要利用跨源图结构主动寻找最有价值的人工标注样本**。

但不能原样用于线上自动匹配。ALMSER-GB 原始目标主要是提高实体解析 F1，它会把分类器预测的正边构造成 correspondence graph，并通过连通性推断额外正样本；这在“reference number 必须完全正确、绝对不能误匹配”的二奢腕表场景中风险过高。一条错边可能通过传递关系污染整个连通分量。

**推荐落地方式：保留 ALMSER 的图增强主动学习思想，但把图从“自动产生匹配真值”降级为“错误发现、冲突审计、人工样本选择和训练数据治理层”；线上最终 AUTO_MATCH 只能由 canonical reference 的高可信严格一致性触发。**

建议最终系统采用：

> Reference 抽取/规范化 → 高召回候选生成 → 严格规则自动匹配 → 图一致性审计 → ALMSER 式主动标注 → 模型仅处理疑难候选并允许拒识 → 规则/模型更新

其中最重要的原则是：

1. **reference 是身份键，不是普通相似特征。**
2. **模型不能覆盖 reference 冲突。**
3. **图片、LLM、向量相似度均不能单独触发 AUTO_MATCH。**
4. **所有不满足强证据条件的样本进入 REVIEW/ABSTAIN，而不是强行二分类。**
5. **图算法用于找错，不用于越权“补真值”。**

---

## 2. 当前 Spec 对技术方案的真实要求

当前需求可以抽象为三源多实体解析：

- 来源：雷小安、腕表之家、奢当家；未来可能增加来源。
- 数据量：约 100 万～1000 万级，并持续增量。
- “同一个商品”的定义：**同一 reference number / 型号**，并非物理意义上的同一只表。
- reference 可能存在于独立字段、标题、描述、图片 OCR 中，也可能完全缺失。
- 字段稀疏、来源 schema 和标题风格不同。
- 有图片，但图片只能作为辅助证据。
- 最重要约束：**precision 极端优先，宁漏勿错。**
- 可接受几百对人工黄金标签。

因此这个问题不是普通的“相似商品检索”，而是一个带有强业务身份约束的 **precision-first multi-source entity resolution**。

真正困难的部分不是“找相似”，而是：

- 标题中可能出现多个像 reference 的字符串；
- 店铺 SKU、平台货号、机芯号、表径、年份、配件兼容型号可能被误识别为 reference；
- 同系列腕表 reference 只差 1～2 个字符，但图片极其相似；
- 一边有 reference，另一边只有中文标题/图片；
- 同一 reference 在不同来源存在空格、连字符、点号、大小写、前后缀差异；
- 增量数据会不断出现新品牌、新型号和新的卖家标题模板；
- 一个 false positive 可能把整个实体簇错误合并。

这正是 ALMSER 的“多源图 + 主动学习”值得借鉴的地方，但必须重新定义图的权限边界。

---

## 3. ALMSER-GB 原始实现架构

### 3.1 输入不是原始记录，而是候选 pair feature vector

ALMSER 的核心类 `ALMSER` 接收：

- `feature_vector_train`
- `feature_vector_test`
- `unique_sources`
- query budget / quota
- classifier
- query strategy
- data-source pair 信息

项目把每个候选记录对组织成：

```text
(source_record, target_record)
      ↓
pairwise feature vector
      ↓
source-pair metadata + label / unsupervised_label
```

这意味着 ALMSER 本身不是 Blocking 系统，而是位于 **Blocking 之后的主动学习与 matcher 训练层**。

这一点非常适合当前场景：百万～千万级原始数据绝对不能先生成笛卡尔积；必须先通过品牌、reference 候选、系列、向量召回等方式把候选 pair 压缩，再交给图和模型。

### 3.2 初始 bootstrap

代码在初始化时使用 `unsupervised_label` 训练一个初始 classifier，并从每个 datasource pair 中选择：

- aggregate score 最大的 pair 作为初始正样本；
- aggregate score 最小的 pair 作为初始负样本。

然后进入主动学习循环。

这个 bootstrap 思路对二奢可以保留，但初始标签不能来自普通相似度，而应该来自强规则：

**正 bootstrap：**

- canonical brand 一致；
- 两侧均存在高可信 reference；
- canonical reference 严格一致；
- 无任何冲突 reference；
- 编号角色均判断为 `manufacturer_reference`。

**负 bootstrap：**

- 同品牌、同系列但高可信 canonical reference 不同；
- 或明确 reference 冲突。

这样生成的负样本会天然包含最有价值的 hard negatives，而不是随机负样本。

### 3.3 多模型与 source-pair 建模

ALMSER 会维护：

- 全局 matcher；
- bootstrap matcher；
- graph-boosted matcher；
- 某些策略下按 datasource pair / task group 建立模型；
- all-minus-one 模型用于衡量任务间 transferability。

这对应当前系统中的：

```text
雷小安 ↔ 腕表之家
雷小安 ↔ 奢当家
腕表之家 ↔ 奢当家
```

三个 pair 的数据分布并不相同。例如：

- A 平台 reference 独立字段覆盖高；
- B 平台 reference 常埋标题；
- C 平台图片 OCR 更有价值。

所以不应该假设同一个阈值/模型在所有 source-pair 上具有相同风险。

建议保留 `source_pair` 维度，线上任何阈值、规则命中率、模型 precision 都按 source-pair 分桶统计。

### 3.4 correspondence graph

ALMSER 最有价值的部分在 `graphutils.py`。

原实现将：

- 模型预测为 match 的 pair 加入无向图；
- 已人工确认的正边加入图，并赋极高权重；
- 已人工确认的负边从图中删除；
- 对部分 bridge edge 做剪枝；
- 如果一个人工负样本的两个节点仍然通过其他路径连通，则计算 `minimum_cut`，删除最小割上的边，使已知负样本两端不再连通。

之后，若两个节点之间存在 path，则产生 `graph_inferred_label=True`。

这实际上把多源实体解析从 pairwise classification 提升成了图一致性问题。

### 3.5 graph disagreement 作为主动学习信号

ALMSER 会同时维护：

- classifier prediction；
- graph inferred prediction；
- committee disagreement；
- datasource pair frequency；
- connected component size；
- task relatedness。

核心 query 思路是：

> 优先询问“分类器和图结构发生冲突”的 pair。

例如：

```text
模型：match
图：non-match
```

或者：

```text
模型：non-match
图：match
```

如果当前没有图-模型冲突，则退化到 Query-by-Committee disagreement。

这是 ALMSER 对当前项目最有价值的部分：**人工标签应该花在最可能改变系统边界、最可能暴露 false positive 的样本上，而不是随机标注。**

### 3.6 graph-boosted training

ALMSER 还会从较小、相对干净的 connected components 中取得图推断标签，作为额外训练数据，与人工 labeled set 一起训练 `boost_graph` 模型。

这可以降低人工成本，但对于当前 precision-first 系统必须做更严格改造，因为“连通”不等于 reference 真值。

---

## 4. ALMSER 原方案不能直接上线的原因

### 4.1 传递闭包会放大 false positive

假设：

```text
A = Rolex 126610LN
B = Rolex 126610LN
C = Rolex 126610LV
```

模型误把 `B-C` 判成 match，则：

```text
A -- B -- C
```

原 ALMSER 图逻辑可能让 `A` 与 `C` 通过 path 被推断为同实体。

对于一般 ER，这可能只是一个聚类错误；对于当前业务，这是不可接受的 reference 污染。

因此：

> **线上不能把 `nx.has_path(A, C)` 当作 same_reference 判据。**

### 4.2 F1 目标不符合当前风险函数

ALMSER 论文和代码主要报告 precision / recall / F1，并围绕 F1 改善主动学习效率。

当前需求的风险函数明显不同：

```text
Cost(false_positive) >>> Cost(false_negative)
```

因此优化目标应该改成：

```text
maximize coverage / recall
subject to precision >= P_target
```

甚至第一阶段直接采用：

```text
hard-negative validation set 上 0 false positive
```

再逐步增加 coverage。

### 4.3 图伪标签不能直接作为正训练标签

ALMSER 会将 graph-inferred labels 作为训练增强数据。当前场景中可以使用图伪标签，但建议：

- 图产生的 **冲突负样本** 可以高优先级送人工；
- 图产生的 **潜在正样本** 只能作为候选，不能直接进入强正训练集；
- 只有 reference 证据链闭合的样本才能成为高可信正伪标签。

---

## 5. 推荐落地架构：Reference-first + Graph Audit + Active Learning

## 5.1 总体架构

```text
┌────────────────────────────────────────────┐
│ 雷小安 / 腕表之家 / 奢当家 Raw Records    │
└──────────────────────┬─────────────────────┘
                       ↓
┌────────────────────────────────────────────┐
│ 1. Canonicalization                        │
│ 品牌统一 / 文本清洗 / source schema 映射   │
└──────────────────────┬─────────────────────┘
                       ↓
┌────────────────────────────────────────────┐
│ 2. Reference Evidence Extraction           │
│ 独立字段 + 标题规则 + NER/LLM + OCR        │
│ 每个候选保留 provenance / role / confidence│
└──────────────────────┬─────────────────────┘
                       ↓
┌────────────────────────────────────────────┐
│ 3. Reference Canonicalizer                 │
│ 品牌级语法规范化 + reference registry      │
└──────────────────────┬─────────────────────┘
                       ↓
        ┌──────────────┴───────────────┐
        ↓                              ↓
┌────────────────────┐       ┌────────────────────────┐
│ 4A. Exact Auto Join │       │ 4B. Ambiguous Candidate│
│ 高可信 reference    │       │ Blocking / ANN / OCR   │
└─────────┬──────────┘       └────────────┬───────────┘
          ↓                               ↓
┌────────────────────┐       ┌────────────────────────┐
│ AUTO_MATCH          │       │ Pair Feature Builder   │
│ 证据链完整才放行    │       └────────────┬───────────┘
└─────────┬──────────┘                    ↓
          │                    ┌────────────────────────┐
          └───────────────────→│ 5. Graph Audit Layer   │
                               │ ALMSER-style graph     │
                               └────────────┬───────────┘
                                            ↓
                               ┌────────────────────────┐
                               │ 6. Active Label Queue  │
                               │ FP-risk first          │
                               └────────────┬───────────┘
                                            ↓
                               ┌────────────────────────┐
                               │ 7. Matcher / Extractor │
                               │ Training + Calibration │
                               └────────────┬───────────┘
                                            ↓
                               REVIEW / ABSTAIN / NO_MATCH
```

关键区别：**Exact Auto Join 在模型之前，并且模型无权推翻明确 reference 冲突。**

---

## 5.2 Reference Evidence 数据模型

不要把 `reference` 只存成一个字符串字段。建议保留“证据对象”：

```json
{
  "raw_value": "126610-LN",
  "canonical_value": "126610LN",
  "brand": "ROLEX",
  "role": "manufacturer_reference",
  "source": "title_regex",
  "confidence": 0.998,
  "span": [14, 23],
  "extractor_version": "rolex-ref-v3",
  "evidence_record_id": "..."
}
```

`role` 至少需要区分：

```text
manufacturer_reference
platform_sku
seller_sku
movement_number
serial_number
compatible_reference
accessory_reference
unknown_identifier
```

这是避免 false positive 的第一道核心防线。

例如标题：

```text
适配劳力士 126610LN 原装风格表带 SKU A23991
```

如果只做“像型号的字符串抽取”，很容易把 `126610LN` 当成当前商品 reference，但实际商品可能只是表带。

所以抽取必须同时做：

```text
identifier extraction + identifier role classification
```

---

## 5.3 Canonical Reference 规范化

规范化必须 **brand-aware**，不能用全局 aggressive string normalization。

允许的安全变换示例：

```text
大小写统一
Unicode 归一
首尾空白
品牌明确允许的 separator 删除
品牌明确允许的固定前缀映射
```

禁止作为 AUTO_MATCH 依据的变换：

```text
Levenshtein 距离 <= 1
任意删除字符
数字模糊纠错
O/0、I/1 无条件替换
前后缀猜测
LLM 自由生成 reference
```

建议维护：

```text
reference_registry(
    brand,
    canonical_reference,
    aliases[],
    grammar_version,
    first_seen,
    source_count,
    verified_status
)
```

品牌级 grammar 可以从历史高可信 reference 自动统计，但所有新 rewrite rule 上线前必须经过 hard-negative 测试。

---

## 5.4 AUTO_MATCH 的最小安全规则

建议第一版只放行最窄、最安全路径：

```python
def auto_match(a, b):
    if a.brand_canonical != b.brand_canonical:
        return False

    if not a.has_high_conf_reference:
        return False
    if not b.has_high_conf_reference:
        return False

    if a.reference_role != "manufacturer_reference":
        return False
    if b.reference_role != "manufacturer_reference":
        return False

    if a.canonical_reference != b.canonical_reference:
        return False

    if a.has_reference_conflict or b.has_reference_conflict:
        return False

    return True
```

这里的 `high_conf_reference` 不应简单等价于模型 probability，而应满足证据规则。例如：

**Tier A：**

- 平台结构化 reference 字段；
- 且通过品牌 grammar；
- role 明确；
- 无冲突。

**Tier B：**

- 标题 regex / extractor 得到 reference；
- reference 在品牌 registry 中；
- 上下文 role classifier 确认是当前商品型号；
- 无其他冲突 reference。

**Tier C：**

- 仅 OCR / 图片模型发现；
- 不允许独立 AUTO_MATCH，只能补证或 REVIEW。

---

## 6. 如何把 ALMSER 的图真正用对

## 6.1 图中的节点与边

建议图不是全局 1000 万节点 NetworkX 图，而是按品牌 / reference 候选局部分片。

节点：

```text
OfferNode(source, record_id)
ReferenceNode(brand, canonical_reference)  # 可选
```

边分成至少四类：

```text
HARD_POSITIVE   高可信 exact reference
HARD_NEGATIVE   人工确认不同 reference / 明确冲突
SOFT_POSITIVE   模型或向量认为可能相同
EVIDENCE_EDGE   OCR / title / structured field 指向 reference
```

任何图推理都必须保留 edge type，不能把所有边扁平成“match”。

## 6.2 参考 ALMSER 的 minimum-cut，但用途改为找可疑边

ALMSER 中，当一个人工负样本的两个节点仍在同一 connected component，会计算 minimum cut 并删除边。

在当前系统中建议改成：

```text
发现 HARD_NEGATIVE 两端仍被 SOFT_POSITIVE 路径连通
        ↓
计算最小割 / 找最弱路径
        ↓
将割上的 soft edges 标记为 SUSPECT
        ↓
进入 active-label queue
        ↓
人工确认后更新规则/模型
```

**不要自动删除真实业务关系，也不要据此自动生成正标签。**

这会把图算法变成“false-positive detector”。

## 6.3 图冲突规则

可以直接落地以下冲突检测：

### Conflict A：一个 soft component 出现多个高可信 canonical reference

```text
component.references = {126610LN, 126610LV}
```

这是最高优先级风险，应立即冻结该 component 的任何模型自动放行。

### Conflict B：模型判正，但两侧高可信 reference 不同

直接记为模型 hard negative，并加入回归集。

### Conflict C：同一记录抽取出多个 manufacturer_reference

进入 extraction review，而不是 matcher review。

### Conflict D：图通过多跳认为两记录同实体，但缺少共同 reference 证据

只能作为人工标注候选，禁止自动 match。

### Conflict E：图片高度相似但 reference 冲突

图片证据被判定为“相似变体”，不能覆盖 reference。

---

## 7. ALMSER 式主动学习如何适配“几百对黄金标签”

人工预算不应该随机抽样。

建议每轮从候选池计算：

```text
risk_score =
    w1 * reference_conflict
  + w2 * model_rule_disagreement
  + w3 * graph_contradiction
  + w4 * hard_negative_similarity
  + w5 * source_pair_undercoverage
  + w6 * extractor_uncertainty
```

其中第一版权重不一定需要学习，可以先做 lexicographic priority：

```text
P0: 高可信 reference 冲突但模型预测 match
P1: 图 component 含多个 reference
P2: 同系列相邻 reference + 图文极像
P3: 一侧有 reference、一侧缺失但模型高置信
P4: 新品牌 / 新来源 / 新标题模板
P5: 普通 uncertainty
```

这比标准 margin sampling 更符合当前目标，因为普通 uncertainty 往往会把大量预算花在“模型不确定但业务风险一般”的样本上。

### 推荐 400 对人工标签的分配

不是固定值，可作为第一轮：

```text
150 对：同品牌同系列、reference 极接近的 hard negatives
80 对：一侧结构化 reference、一侧标题抽取
60 对：OCR / 图片出现 reference 的疑难样本
50 对：三类 source-pair 均衡覆盖
40 对：新品牌 / 长尾 reference
20 对：随机审计样本，用于估计选择偏差
```

主动学习每 50～100 对更新一次，而不是一次性标完。

---

## 8. Pair Feature 设计

如果要训练 matcher，建议特征显式区分“身份证据”和“相似证据”。

### 8.1 Identity Features（最高优先级）

```text
brand_exact
reference_exact
reference_conflict
reference_edit_distance
reference_prefix_equal
reference_numeric_core_equal
reference_role_equal
reference_source_tier_a/b/c
registry_verified
```

其中 `reference_edit_distance` 仅供模型学习和 hard-negative 选择，不能直接放行。

### 8.2 Text Features

```text
title_embedding_cosine
series_exact
model_name_similarity
material_match
size_match
year_overlap
keyword_jaccard
```

### 8.3 Image Features

```text
image_embedding_cosine
ocr_reference_exact
ocr_reference_conflict
logo_brand_match
image_text_reference_agree
```

### 8.4 Source Features

```text
source_pair
field_coverage_pattern
extractor_version
source_reference_reliability
```

建议第一版 matcher 使用 LightGBM / CatBoost / Logistic Regression 即可，不需要先上大 LLM。

原因：

- 几百标签时树模型更容易稳定；
- 特征可解释；
- 能做 source-pair 分析；
- 便于分析 false positive；
- 最终 AUTO_MATCH 仍由 rule gate 控制。

LLM 更适合：

```text
reference role classification
标题中候选 identifier 解释
人工 review 辅助
生成新的规则候选
```

而不是最终 match decision。

---

## 9. 三态决策，而不是二分类

必须把线上结果设计成三态：

```text
AUTO_MATCH
REVIEW
ABSTAIN / NO_MATCH
```

建议：

### AUTO_MATCH

必须满足：

```text
canonical brand exact
+ canonical reference exact
+ reference evidence >= required tier
+ no conflict
+ rule version 已通过回归集
```

### REVIEW

典型情况：

```text
一侧 reference 缺失
OCR 与标题冲突
reference 候选多值
模型强相似但缺身份键
graph contradiction
```

### NO_MATCH / ABSTAIN

```text
明确 reference 不同 → NO_MATCH
证据不足 → ABSTAIN
```

不要把 ABSTAIN 偷偷映射成模型概率最大的类别。

---

## 10. 百万～千万级工程实现

ALMSER 原项目使用 Pandas + NetworkX，适合论文实验，不适合直接处理 1000 万级生产数据。

建议把思想迁移，而不是直接部署原代码。

### 10.1 存储

建议：

```text
Object Storage / OSS / S3
    保存 raw snapshot、图片、OCR 结果

ClickHouse / BigQuery / Spark tables
    offer_canonical
    reference_evidence
    match_candidate
    match_decision
    audit_event
    label_gold
```

如果现有基础设施偏 MySQL/PostgreSQL，也可先用分区表落第一版，只要避免 pair 笛卡尔积。

### 10.2 Blocking

候选生成优先顺序：

```text
1. brand + canonical_reference exact bucket
2. brand + series + reference candidate token
3. brand + title embedding ANN（仅缺 reference 的 REVIEW 流）
4. brand + image ANN（仅辅助召回）
```

千万级数据不能：

```text
for a in sourceA:
    for b in sourceB:
        compare(a,b)
```

而应变成：

```sql
SELECT ...
FROM a
JOIN b
  ON a.brand = b.brand
 AND a.reference = b.reference
```

模糊候选再走 ANN/倒排索引。

### 10.3 图计算

不建议建立一个全局 NetworkX graph。

可采用：

```text
按 brand / candidate reference 分区
+ 局部 union-find / connected component
+ 只物化 suspicious edges
```

绝大多数 exact reference 样本根本不需要图算法。

图层只维护：

```text
soft candidate edges
hard negative constraints
conflict components
active-learning queue
```

这会把图规模从千万记录降低到“疑难候选”的小子集。

---

## 11. 增量更新架构

每条新商品进入时：

```text
1. normalize brand/text
2. extract reference evidence
3. canonicalize reference
4. 查询 brand+reference bucket
5. 满足强规则 → AUTO_MATCH
6. reference 缺失/冲突 → candidate retrieval
7. 更新局部 graph audit
8. 若触发 contradiction → REVIEW queue
```

无需每次全量重跑。

建议每条 decision 保存：

```json
{
  "decision": "AUTO_MATCH",
  "rule_version": "match-v4",
  "extractor_version": "rolex-ref-v3",
  "reference": "126610LN",
  "evidence_ids": ["e1", "e9"],
  "source_pair": "leixiaoan-watchhome",
  "created_at": "..."
}
```

这样一旦某个 extraction rule 被发现有 bug，可以精确回滚受影响的历史决策，而不是重新扫描全部数据。

---

## 12. Precision-first 验收方案

不能只看整体 F1。

必须单独维护：

### 12.1 Hard-negative regression set

重点包含：

```text
同品牌同系列相邻 reference
仅一个字符不同
后缀不同
表壳材质/盘面不同导致 reference 不同
腕表 vs 表带/配件
标题出现“适配型号”
店铺 SKU 冒充 reference
OCR O/0、I/1 混淆
```

每次规则/模型发布必须跑全量回归。

### 12.2 指标

建议分层统计：

```text
AUTO_MATCH precision
AUTO_MATCH coverage
reference extraction precision
reference extraction coverage
source-pair precision
brand precision
review yield
graph contradiction rate
post-release false-positive incidents
```

第一阶段目标不是追求高 coverage，而是：

```text
AUTO_MATCH hard-negative set: 0 FP
```

然后逐步放宽能证明安全的证据路径。

注意：只有几百条黄金标签时，无法用统计学声称“生产环境绝对 100% precision”。因此工程上要用：

```text
窄规则 + hard-negative 专项集 + abstention + 持续随机审计
```

共同降低风险，而不是只相信一个 0.9999 模型概率。

---

## 13. 可直接落地的第一版（MVP）

如果需要尽快上线，不建议一开始实现完整 ALMSER。

### Phase 1：Reference-first baseline

实现：

```text
brand canonicalization
reference regex / structured-field extractor
reference role classifier
brand-aware canonicalizer
exact join
conflict detector
```

输出：

```text
AUTO_MATCH / ABSTAIN
```

这一步就能覆盖 reference 字段质量较好的大量数据，同时风险最小。

### Phase 2：ALMSER-style audit queue

在 exact baseline 周围建立：

```text
soft candidate graph
reference conflict component detector
model-vs-rule disagreement
minimum-cut suspicious edge discovery
active review queue
```

人工开始标 hard negatives。

### Phase 3：Missing-reference matcher

仅对 reference 缺失记录增加：

```text
brand blocking
series/title ANN
image ANN
OCR reference
pair classifier
```

但输出仍然只能进入：

```text
REVIEW / ABSTAIN
```

直到某类证据经过足够验证，才考虑升级为自动放行规则。

### Phase 4：持续主动学习

每轮使用 ALMSER 式图冲突 + source-pair 覆盖选择 50～100 对样本，更新：

```text
reference extractor
role classifier
pair matcher
source-specific threshold
```

并固定保留历史 hard-negative 回归集。

---

## 14. 推荐数据库表

```sql
CREATE TABLE reference_evidence (
    record_id String,
    source LowCardinality(String),
    brand String,
    raw_value String,
    canonical_value String,
    role LowCardinality(String),
    evidence_source LowCardinality(String),
    confidence Float32,
    extractor_version String,
    created_at DateTime
);
```

```sql
CREATE TABLE match_candidate (
    left_id String,
    right_id String,
    source_pair LowCardinality(String),
    candidate_reason LowCardinality(String),
    brand_exact UInt8,
    reference_exact UInt8,
    reference_conflict UInt8,
    title_score Float32,
    image_score Float32,
    model_score Float32,
    graph_conflict UInt8,
    created_at DateTime
);
```

```sql
CREATE TABLE match_decision (
    left_id String,
    right_id String,
    decision LowCardinality(String),
    canonical_reference String,
    decision_reason String,
    rule_version String,
    model_version String,
    created_at DateTime
);
```

```sql
CREATE TABLE label_gold (
    left_id String,
    right_id String,
    label UInt8,
    error_type LowCardinality(String),
    annotator String,
    evidence String,
    created_at DateTime
);
```

---

## 15. 主动学习队列伪代码

```python
def active_learning_priority(edge):
    # 绝对优先找可能导致 FP 的样本
    if edge.reference_conflict and edge.model_predict_match:
        return (0, -edge.model_score)

    if edge.graph_component_has_multiple_references:
        return (1, -edge.graph_risk)

    if edge.same_series and edge.reference_edit_distance <= 2:
        return (2, -edge.multimodal_similarity)

    if edge.one_side_reference_missing and edge.model_predict_match:
        return (3, -edge.model_score)

    if edge.is_new_source_or_brand:
        return (4, -edge.model_uncertainty)

    return (5, -edge.model_uncertainty)
```

这实际上是“precision-first ALMSER”：

- ALMSER 原版优先图-模型 disagreement；
- 当前版本进一步把 **reference contradiction** 放到图 disagreement 之前。

---

## 16. 一个具体例子

假设：

```text
A / 雷小安
标题：劳力士 潜航者 黑水鬼 126610LN 2023
结构化 reference：126610LN

B / 腕表之家
标题：Rolex Submariner Date 126610 LN 全套
reference：缺失

C / 奢当家
标题：劳力士 绿水鬼 126610LV
reference：126610LV
```

抽取后：

```text
A → ROLEX / 126610LN / Tier A
B → ROLEX / 126610LN / Tier B
C → ROLEX / 126610LV / Tier A
```

则：

```text
A-B：reference exact → AUTO_MATCH
A-C：reference conflict → NO_MATCH
B-C：reference conflict → NO_MATCH
```

即使图片模型给出：

```text
sim(A,C)=0.98
```

也不能改变 A-C 的 NO_MATCH。

如果 B 没能抽到 reference：

```text
A-B → REVIEW candidate
```

图片/标题可以帮助排序 B，但不能直接将 B 自动并入 126610LN。

这就是当前系统和普通 multimodal product matching 最大的差别。

---

## 17. 对 ALMSER-GB 的最终取舍

### 建议直接借鉴

1. **多 source-pair 分桶建模与评估**。
2. **graph-vs-model disagreement 用于主动标注**。
3. **已知负边作为图约束**。
4. **minimum-cut / bridge 思想用于发现可疑 soft edge**。
5. **主动学习优先选择会暴露模型错误的样本**。
6. **利用跨源图信号提升几百条标签的价值**。
7. **connected-component-aware train/test split，避免传递泄漏**。

### 不建议直接照搬

1. `has_path == match` 的图推断。
2. 普通 classifier positive 直接成为图正边并参与自动聚类。
3. graph-inferred positive 未经 reference 校验直接作为训练真值。
4. F1 作为主要上线目标。
5. Pandas + NetworkX 全量处理千万级数据。

### 推荐改造后的定位

```text
ALMSER 原版：
matcher → graph → 推断更多 match → boost matcher

当前推荐：
reference rule → exact auto decision
                  ↓
soft matcher → audit graph → 找冲突/找高风险样本 → 人工标签
                                      ↓
                              更新 extractor/matcher
```

图从 **decision engine** 变成 **risk discovery engine**。

---

## 18. 最终推荐方案

如果只选一个能够从本项目直接吸收的实现思想，我认为最值得落地的是：

> **建立“Reference Constraint Graph”，并用 ALMSER 的 graph-model disagreement + negative-edge minimum-cut 思路，把人工标注预算优先花在最可能造成 false positive 的边上。**

同时，生产匹配主路径保持极简：

```text
高可信 brand
+ 高可信 canonical reference
+ reference exact
+ 无冲突
= AUTO_MATCH
```

其他所有智能能力，包括：

```text
LLM
文本 embedding
图片 embedding
OCR
图传播
pair classifier
```

只负责三件事：

```text
1. 找 reference
2. 找需要审核的候选
3. 找现有系统可能错在哪里
```

而不负责越过 reference 强约束做最终自动合并。

这能同时满足：

- 100 万～1000 万级可扩展；
- 三源及未来多源；
- 字段稀疏；
- reference 埋标题；
- 图片可辅助；
- 几百对人工标签；
- 持续增量；
- **precision 极端优先，宁漏勿错**。

从工程优先级看，建议立即做 **Phase 1（reference evidence + exact join）** 和 **Phase 2（ALMSER-style graph audit + active label queue）**。这两层上线后，再决定是否需要更复杂的 multimodal matcher。