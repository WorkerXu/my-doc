# TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **TransClean: Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency** 进行深入分析。

- 论文：<https://arxiv.org/abs/2506.04006>
- PDF：<https://arxiv.org/pdf/2506.04006>
- 需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

TransClean 对当前需求最有价值的地方，不是再提供一个“判断两个商品是否相同”的模型，而是提供了一层 **匹配结果的图一致性审计与 false-positive 清洗机制**。

当前需求已经把“同一个商品”定义得非常强：

> **同一 reference number / 型号 = 同一个商品实体；precision 极端优先，允许漏匹配，绝不能误匹配。**

因此我建议不要把系统核心设计成“商品 A 和商品 B 做语义相似度分类”，而是重构为：

```text
商品记录
  -> 抽取多个 reference observation
  -> 保守规范化 canonical reference
  -> 链接 Reference Entity
  -> Reference Entity 内形成跨源商品组
  -> TransClean-inspired 图一致性审计
  -> 只有无冲突组才进入 VERIFIED
```

最终原则：

> **Reference 的可追溯硬证据负责放行；文本/图片/LLM/embedding 只负责发现候选或发现冲突；图一致性负责阻止单条错误边污染整个实体组。**

与原论文相比，当前业务还应该做一个关键改造：

> 原始 TransClean 使用二值 pairwise matcher 的 `Match / NoMatch` 预测检查传递一致性；当前场景应使用 **三值 verifier：MATCH / CONFLICT / UNKNOWN**。只有存在强矛盾证据时才算 CONFLICT，字段缺失不能被当成 NoMatch。

这是将 TransClean 真正安全地迁移到二奢 Reference Matching 的关键。

---

## 1. 为什么选 TransClean

当前三源数据有几个特征与论文问题设定高度一致：

1. 多来源：雷小安、腕表之家、奢当家；
2. 百万到千万级，并持续增量；
3. 字段稀疏，reference 可能在结构化字段，也可能埋在标题或图片中；
4. 分布会漂移：新品牌、新卖家文案、新抓取模板、新 OCR 噪声；
5. 少量 false positive 的代价极高；
6. 只能接受有限人工标注。

TransClean 关注的正是传统 pairwise matcher 在多源场景中的一个严重问题：

```text
A --match-- B --match-- C

系统会隐式把 A、B、C 放进同一实体组，
即使 A 和 C 从未直接比较过。
```

这意味着一条 false-positive 边不只制造一个错误 pair，而可能通过 connected component / transitive closure 污染整个实体簇。

在当前业务中，这类错误非常典型：

```text
Rolex 126610LN
      │
      │ 错误抽取/错误规范化
      ▼
Rolex 126610LV
```

两者字符串和外观都非常接近，但 reference 不同。只要错误地把它们连成一个组，后续来自三个平台的所有同型号记录都可能被错误传递合并。

因此，当前系统不能只看“单边准确率”，还必须审计“边连接之后形成的组是否自洽”。

---

## 2. TransClean 原论文的核心技术

### 2.1 Matching Graph

论文把匹配结果表示为无向图：

```text
G_f = (V, E_f)
```

其中：

- `V`：所有 record；
- `E_f`：pairwise matcher 预测为 Match 的边；
- 一个 connected component：被系统隐式认为是同一个实体的 record group。

只要两个 record 位于同一 connected component，即使没有直接边，也会因为传递关系被隐式视为匹配。

### 2.2 Transitive Match

如果两个节点：

- 位于同一个 connected component；
- 但两者之间没有直接 edge；

那么它们形成一个 **transitive match**。

例如：

```text
A -- B -- C
```

`A-C` 就是 transitive match。

### 2.3 Transitive Consistency

论文再让原来的 pairwise matcher 去判断这些“被图结构隐式推导出来，但模型从未直接判断过”的 pair。

如果：

```text
A-B = Match
B-C = Match
但模型认为 A-C = NoMatch
```

则该 component 不具备 Transitive Consistency。

论文把 transitive pair 的模型结果分成：

- Positive Transitive Prediction：模型也认为 Match；
- Negative Transitive Prediction：模型认为 NoMatch。

重要观察是：

> Negative Transitive Predictions 往往集中在存在 false-positive edge 的 component 中，因此可以在没有完整 ground truth 的情况下作为错误匹配的 proxy。

而且数据源越多，一个错误 edge 通常会制造更多 transitive contradiction，因此多源本身反而变成错误检测信号。

---

## 3. TransClean 三阶段清洗流程

### 3.1 第一阶段：Initial Step with Finetuning

目标是先处理最明显的错误 component。

论文大致流程：

1. 对 component 计算 Negative Transitive Predictions 数量；
2. 优先处理负传递预测多的 component；
3. 从 component 中选择高价值 edge 做人工标注：
   - Minimum Edge Cut 中的 edge；
   - 位于 negative transitive pair 两端之间 shortest path 上的 edge；
4. 标注出的 false-positive edge 从图中删除；
5. 用这些 hard cases 继续 fine-tune matcher；
6. 重复直到耗尽阶段 labeling budget。

为什么 Minimum Edge Cut 有用？

如果一个 false-positive edge 把本来应该分开的两个实体组桥接起来，它往往正是让 component 连通所需的少量“桥”。把 min-cut 边拿出来检查，单位标注成本能够影响更多错误传递 pair。

为什么 shortest path 有用？

如果 `A` 和 `C` 是一个模型明确认为 NoMatch 的 transitive pair，那么连接 A 到 C 的 path 上不可能全部是真正的 match。沿 path 找 edge，相当于把人工预算集中到“必有问题”的局部结构里。

### 3.2 第二阶段：Post Finetuning Cleanup & Checks

论文在 matcher 经多轮 hard-case fine-tune 后进一步做 cleanup：

- 对依然存在明显 transitive inconsistency 的 component 继续剪除 Minimum Edge Cut；
- 对异常大 component 做 size check；
- 对已有 label 可以验证的 transitive 关系做检查；
- 最终检查仍含 Negative Transitive Prediction 的 component。

该阶段的目标是让图逐渐趋向 transitively consistent。

### 3.3 第三阶段：Edge Recovery

前两阶段为了 precision 会有意“多砍一些”，因此可能误删真阳性边。

论文最后尝试恢复被删除 edge：

1. 假设把某条已删 edge 加回；
2. 计算它会新产生哪些 transitive matches；
3. 如果所有新 transitive matches 都被 matcher 判断为 Match，则恢复该 edge；
4. 如果存在 NoMatch，则进入人工检查（预算允许时）；
5. 如果 edge 只连接两个孤立 record、无法产生新传递关系，则人工检查。

这一步本质上是：

> **恢复 recall，但不能破坏已经得到的全局一致性。**

这非常符合当前 precision-first 业务，不过我们应该比原论文更保守：没有强证据时宁可不恢复。

---

## 4. 原论文实验对当前需求最重要的结论

论文同时把 TransClean 接在普通 DistilBERT matcher 和 CLER 后面，说明它不是某一种 matcher 专属模块，而是一个 graph-level wrapper。

几个值得直接记住的数据：

### 4.1 Pairwise 指标可能严重误导

在 Synthetic Companies 上，DistilBERT pairwise F1 为 **81.54**，看起来尚可；但把传递匹配也算进去后，Pre-TransClean precision 直接降到 **0.02**，F1 为 **0.04**。

这说明：

> 在多源 group matching 中，一小批 false-positive pair 可以被图传递放大几个数量级，单看 pairwise F1 不足以表示最终实体簇质量。

这对当前业务尤其重要，因为如果一个 reference 错误归一化，错误会扩散到该 reference 下所有平台记录。

### 4.2 强 matcher + TransClean 更适合 precision-first

Synthetic Companies 的 CLER：

- Pairwise precision：97.69；
- 包含 transitive match 的 Pre-TransClean precision：87.48；
- Post-TransClean precision：98.54；
- 删除 false positives：78.06%；
- 删除 true positives：1.88%。

它说明 TransClean 最合适的定位不是“救一个很差的 matcher”，而是：

> **在已经相对可靠的基础 matcher / 硬规则之上做最后一道 false-positive 清洗。**

这与当前 Reference exact-rule 架构十分匹配。

### 4.3 LLM 不适合承担最终标注真值

论文同时测试 LLM pseudo-labeling。整体上 LLM 版本会：

- 删除更多 true positives；
- 删除更少 false positives；
- 当基础 matcher 较差时问题更严重。

例如 WDC Products 中，人工标签版本删除约 12.22% TP、86.77% FP；LLM 标签版本删除约 46.7% TP、77.52% FP。

这对当前需求的含义非常明确：

> **LLM 可以抽取 reference candidate、解释冲突、给人工复核排序，但不能把“LLM 说是同型号”作为最终自动合并真值。**

---

## 5. 不能原样照搬 TransClean 的地方

### 5.1 当前业务不是开放式“同一实体”判断，而是 Reference Equality

论文里的 matcher 学习广义 entity matching；当前 Spec 已经给出业务定义：同一 canonical reference 即同实体。

因此最重要的不是训练更强 pairwise classifier，而是：

1. 正确发现 reference；
2. 正确判断某个字符串是不是“当前商品本体”的 reference；
3. 保守规范化；
4. 防止平台 SKU / 兼容型号 / 配件适配型号冒充 reference；
5. 防止 OCR/LLM 把相邻 reference 纠错成另一个合法型号。

### 5.2 字段缺失不能等于 NoMatch

原论文 matcher 是二值：Match / NoMatch。

当前数据字段高度稀疏，如果 A 有 reference、C 没有 reference：

```text
A.reference = 126610LN
C.reference = NULL
```

不能因为无法证明 Match，就判 NoMatch。

因此必须改造成：

```text
MATCH      = 有足够强的同 reference 证据
CONFLICT   = 有足够强的矛盾 reference 证据
UNKNOWN    = 证据不足
```

**Transitive conflict 只统计 CONFLICT，不统计 UNKNOWN。**

否则字段缺失越严重，图会被错误拆得越碎。

### 5.3 不建议使用普通 unweighted minimum cut

原论文的 min-cut 用于从 matcher graph 中找桥接错误。

当前场景不同 edge 的可信度差异巨大：

- 官方/结构化字段 exact reference；
- 标题 regex 抽取；
- LLM candidate；
- OCR；
- 图片 embedding。

因此建议实现 **Weighted Minimum Cut**：

```text
edge capacity = evidence reliability
```

强硬证据 edge capacity 高，不容易被 cut；弱证据 edge capacity 低，遇到 contradiction 时优先移除。

注意：capacity 不是一个统一 ML probability，而是“证据等级 + 数据来源 + extractor 版本 + 是否有原文 span”组成的可解释分值。

---

## 6. 推荐的直接落地架构

## 6.1 总体架构

```text
                    ┌─────────────────────┐
雷小安 ────────────>│                     │
腕表之家 ──────────>│ Raw Product Ingest  │
奢当家 ────────────>│                     │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Source Normalizer   │
                    │ brand/title/fields  │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Reference Observer  │
                    │ 1. structured field│
                    │ 2. title/span       │
                    │ 3. OCR              │
                    │ 4. constrained LLM │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Conservative Canon. │
                    │ brand-specific rule │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Reference Registry  │
                    │ (brand, canon_ref)  │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Proposed Link Graph │
                    └─────────┬───────────┘
                              │
                              ▼
             ┌────────────────────────────────┐
             │ TransClean-inspired Audit     │
             │ - MATCH/CONFLICT/UNKNOWN      │
             │ - transitive contradiction    │
             │ - weighted min-cut            │
             │ - hard negative review        │
             └──────────────┬─────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
      ┌───────────────┐           ┌────────────────┐
      │ VERIFIED LINK │           │ ABSTAIN/REVIEW │
      └───────────────┘           └────────────────┘
```

核心思想是把“匹配”和“审计”分开：

- Extraction Layer 尽量召回 reference candidate；
- Decision Layer 极端保守；
- Graph Audit Layer 专门发现那些单条看似合理、放到整个组里却产生矛盾的边。

---

## 6.2 数据模型

建议最少保留以下表。

### products

```sql
products(
  product_id,
  source,
  source_item_id,
  brand_raw,
  title_raw,
  attrs_json,
  image_urls,
  source_updated_at,
  ingest_version
)
```

### reference_observations

一条商品可以有多个 observation，不要一开始就覆盖成一个值。

```sql
reference_observations(
  observation_id,
  product_id,
  evidence_type,       -- structured/title/ocr/llm_candidate
  raw_value,
  canonical_candidate,
  source_span,         -- 能回到标题/OCR原文的位置
  image_id,
  extractor_version,
  evidence_level,
  created_at
)
```

关键点：**必须保存 raw value + provenance**。

否则以后发现 normalizer 有 bug 时无法批量回滚和重算。

### reference_entities

```sql
reference_entities(
  reference_entity_id,
  brand_id,
  canonical_reference,
  normalizer_version,
  status,
  UNIQUE(brand_id, canonical_reference)
)
```

Reference Entity 才是全局主实体。

### product_reference_links

```sql
product_reference_links(
  product_id,
  reference_entity_id,
  state,               -- PROPOSED / VERIFIED / REJECTED / REVIEW
  decision_reason,
  evidence_snapshot,
  decision_version,
  verified_at
)
```

### graph_audit_events

```sql
graph_audit_events(
  component_key,
  product_a,
  product_b,
  verifier_result,     -- MATCH / CONFLICT / UNKNOWN
  conflict_reason,
  suspect_edge_ids,
  audit_version,
  created_at
)
```

---

## 7. Reference 抽取：必须“Grounded”，不能自由生成

### 7.1 结构化字段优先

优先级最高的是平台明确给出的：

- 型号；
- reference；
- model number；
- 厂商编号。

但即使字段名叫“型号”，仍要做 role check，因为部分站点可能混入平台货号、内部 SKU。

### 7.2 标题抽取必须返回 span

LLM/NER/regex 输出不要只是：

```json
{"reference": "126610LN"}
```

而要保存：

```json
{
  "candidate": "126610LN",
  "raw_span": "126610LN",
  "start": 18,
  "end": 26,
  "role": "product_reference"
}
```

这样最终 reference 必须能“指回原始证据”。

如果 LLM 生成了一个标题里根本不存在、OCR 里也不存在的 reference，不能自动采用。

### 7.3 LLM 只做 constrained extraction

推荐 prompt 逻辑：

```text
只允许从输入文本中的连续或可解释规范化片段中抽取 reference。
禁止根据品牌/系列知识补全缺失数字。
若文本只有系列名、昵称或不完整型号，返回 UNKNOWN。
同时区分：
- product_reference
- compatible_reference
- accessory_reference
- platform_sku
- seller_sku
- unknown_identifier
```

比“请推断这是什么型号”安全得多。

### 7.4 OCR 是独立证据，不是最终真值

图片可用于：

- 表背刻字；
- 保卡；
- 吊牌；
- 证书；
- 商品标签。

但 OCR 容易出现：

```text
O <-> 0
I <-> 1
S <-> 5
B <-> 8
```

因此 OCR candidate 进入 reference registry 前必须和品牌 reference grammar / 候选库校验。

尤其不能做“最相似合法型号自动纠错”，因为：

```text
126610LN
126610LV
```

这种 edit distance 很小但业务含义不同。

---

## 8. Canonical Reference：只允许确定性、可逆的规范化

最终 equality 不能用 fuzzy match。

可以做的规范化：

- 大小写统一；
- Unicode 全半角统一；
- 明确等价的空格标准化；
- 明确等价的分隔符标准化；
- 品牌规则中已验证的 prefix/suffix 形式标准化。

不应该做：

- edit-distance 自动纠错；
- embedding nearest reference 自动替换；
- LLM 猜测缺失后缀；
- “看起来很像”就 canonical 到同一编号。

建议 normalizer 设计成 brand-specific deterministic transducer：

```text
normalize(brand, raw_ref) ->
  VALID(canonical_ref, rule_id)
  or AMBIGUOUS(candidates)
  or INVALID(reason)
```

例如只要规范化过程中发生非确定性选择，就进入 AMBIGUOUS，不自动 link。

---

## 9. 三值 Pair Verifier：MATCH / CONFLICT / UNKNOWN

这是当前方案中最重要的 TransClean 改造。

### 9.1 MATCH

只有满足可解释强证据时返回 MATCH，例如：

```text
brand canonical 一致
AND canonical reference exact 一致
AND 至少一侧 reference 有高可信 grounded evidence
AND 双方不存在其他高可信 reference 冲突
```

更严格版本可以要求双方都存在 grounded evidence。

### 9.2 CONFLICT

只有“有明确矛盾”才返回 CONFLICT，例如：

```text
A 高可信 reference = 126610LN
B 高可信 reference = 126610LV
且两者品牌均为 Rolex
```

或：

```text
同一商品同时出现两个互斥的高可信本体 reference
```

### 9.3 UNKNOWN

以下全部是 UNKNOWN，而不是 NoMatch：

- 一边缺 reference；
- OCR 模糊；
- LLM 只抽出可能 candidate；
- title 中存在多个 compatible model；
- 图片只说明外观相似；
- 无法确认编号是不是平台 SKU。

这正好落实“允许漏匹配”。

---

## 10. TransClean-inspired 图审计如何落地

## 10.1 图的构造

MVP 不需要把 1000 万商品做全连接图。

先通过 Reference Registry 将商品分桶：

```text
bucket key = (brand_id, canonical_reference)
```

每个 bucket 内只处理已有 proposed link 产生的小 component。

这把复杂度从全局 `O(N^2)` 变成“只审计受影响 reference bucket”。

### 10.2 生成 transitive checks

对于 component 中未直接比较的 pair：

```python
result = verify(product_i, product_j)
```

统计：

```text
positive_transitive = MATCH 数
negative_transitive = CONFLICT 数
unknown_transitive  = UNKNOWN 数
```

和论文不同：

> `unknown_transitive` 只代表缺证据，不应贡献冲突分数。

### 10.3 Component Risk

可以定义一个可解释 risk ranking，而不是直接训练黑盒分数：

```text
risk(component) =
  high_conflict_count
  + cross_source_conflict_count
  + reference_cardinality_anomaly
  + low_evidence_bridge_edges
  + extractor_version_drift_signal
```

其中 high_conflict_count 权重最高。

### 10.4 Suspect Edge 定位

对于产生 CONFLICT 的 transitive pair `(u, v)`：

1. 找 u 到 v 的 shortest paths；
2. 收集 path 上所有 edge；
3. 计算 edge evidence reliability；
4. 优先检查最低可信 edge；
5. 如果 component 由少量 bridge edge 串联，做 weighted minimum cut。

示例：

```text
A(ref=126610LN, structured)
   |
   | high-confidence
   |
B(ref=126610LN, title exact)
   |
   | LLM-only weak edge   <--- suspect bridge
   |
C(ref=126610LV, structured)
```

`A-C = CONFLICT`。

这时不应该平均怀疑所有 edge，而应优先切 B-C 这条 LLM-only bridge。

### 10.5 Weighted Min-Cut

建议 edge capacity 由规则生成：

```text
structured exact + grounded       -> very high
brand-regex title exact           -> high
OCR exact + grammar valid         -> medium/high
LLM grounded candidate            -> medium/low
LLM inferred / ungrounded         -> never auto edge
image embedding similarity        -> candidate only, no merge edge
```

数值不应现在拍脑袋固定，应该用黄金标签校准，但顺序必须保持。

### 10.6 Edge Recovery

恢复逻辑比论文更保守：

```text
只有：
1. 被删 edge 自身重新获得强 reference evidence；
2. 加回后不会产生任何 CONFLICT；
3. 所有新增重要 transitive pair 至少不是冲突；
4. 对高风险来源/规则可要求人工确认；
才恢复。
```

没有证据时保持 abstain，不为了 recall 恢复。

---

## 11. Incremental：千万级持续更新应该怎样跑

不要每天全量重建 graph。

每条新增/更新商品执行：

```text
1. ingest record
2. extract reference observations
3. canonicalize candidates
4. 找到受影响的 (brand, canonical_ref) buckets
5. 生成 PROPOSED link
6. 只重算受影响 component 的 transitive checks
7. 若无 CONFLICT -> VERIFIED
8. 若有 CONFLICT -> quarantine + review/min-cut
9. 持久化 decision_version / extractor_version
```

更新 reference extractor 或 normalizer 时，也不需要全库立即重算：

- 根据 `extractor_version` 找受影响商品；
- 生成 changed-product stream；
- 只重新审计被触达 bucket。

### 推荐状态机

```text
UNRESOLVED
   │
   ▼
PROPOSED
   │
   ├── strong evidence + graph consistent ──> VERIFIED
   │
   ├── explicit conflict ───────────────────> REJECTED
   │
   └── insufficient / ambiguous ────────────> REVIEW
```

不要让“模型一次推理”直接写最终 merge 表。

---

## 12. 图片在本方案中的正确位置

图片非常有价值，但只能是辅助证据。

### 可以做

1. OCR 找 reference；
2. 检测保卡/吊牌/表背区域；
3. 判断标题中的某个编号是否出现在图片上；
4. 图像相似度用于候选召回；
5. 图像显著冲突用于否决/人工排序。

### 不应该做

```text
两只表长得很像
=> 自动判同 reference
```

同系列不同 reference 的外观可能只有颜色、圈口、尺寸、材质等很小差异。

所以视觉模型应该是：

```text
Recall / Evidence / Conflict Layer
```

而不是：

```text
Final Identity Decision Layer
```

---

## 13. 与 a / b / c 已有方案的组合方式

当前团队已经分别分析：

- `a/DeepBlocker.md`：适合做大规模 candidate retrieval / blocking；
- `b/parts-distributor-sku-classifier.md`：适合区分 manufacturer reference 与平台 SKU/店铺编号；
- `c/AmeLi...md`：适合多模态细粒度属性与 candidate disambiguation。

TransClean 最适合放在它们之后做最后一道 group-level safety layer：

```text
                 ┌─ SKU / identifier role classifier ─┐
Raw Product ─────┤                                     ├─> Reference candidates
                 └─ text/OCR/multimodal extractor ────┘
                                      │
                                      ▼
                          Conservative canonicalizer
                                      │
                                      ▼
                              Reference Registry
                                      │
                     ┌────────────────┴───────────────┐
                     │                                │
                DeepBlocker                     exact bucket
             (疑难候选召回)                         │
                     │                                │
                     └────────> candidate verifier <─┘
                                      │
                                      ▼
                          TransClean-style Graph Audit
                                      │
                           ┌──────────┴──────────┐
                           ▼                     ▼
                        VERIFIED              ABSTAIN
```

也就是说，四个方向并不冲突：

- DeepBlocker 解决规模与召回；
- SKU classifier 解决“编号角色是否正确”；
- AmeLi 类多模态方法解决稀疏字段和图片辅助；
- TransClean 解决少量错误边被传递放大的系统性风险。

---

## 14. 黄金标签应该怎么花

Spec 允许人工标注几百对。不要把这几百对随机抽样后全部拿去训练普通 matcher。

应该优先构造 **hard-negative / conflict golden set**：

### 必须覆盖

1. 同品牌同系列、相邻 reference：
   - `126610LN` vs `126610LV`；
2. reference 缺 suffix / prefix；
3. 平台 SKU 被误识别为 reference；
4. 标题含“适用/兼容/同款/配件”型号；
5. 表带、盒证、保卡、配件商品带有主表 reference；
6. OCR `O/0`、`I/1`、`S/5`、`B/8`；
7. 中文昵称相同但 reference 不同；
8. 同 reference 不同标题写法；
9. 图像极相似但 reference 不同；
10. 多来源字段冲突。

### 标签用途

几百对优先用于：

- canonicalization rule regression test；
- identifier role classifier 校准；
- MATCH / CONFLICT / UNKNOWN verifier regression；
- edge evidence tier 的阈值校准；
- 人工 review 排序策略评估；
- source / brand drift detection。

而不是承诺“几百对足以训练出绝不误匹配的神经网络”。

如果随机抽取 300 个自动接受 pair 且 0 错误，简单 rule-of-three 下 95% 上界的错误率仍大约是 1%，远达不到业务语义上的“绝不能误匹配”。所以 precision 必须主要来自结构化硬约束、拒识和可审计流程，而不是仅靠测试集上 0 error。

---

## 15. 评估指标：不要只看 F1

建议核心 dashboard 按优先级展示：

### P0

- `false_positive_verified_count`
- `verified_precision`
- `hard_negative_false_accept_count`

### P1

- `abstain_rate`
- `manual_review_rate`
- `reference_extraction_grounded_rate`
- `structured/title/ocr/llm evidence distribution`

### 图一致性指标

- `negative_transitive_conflict_count`
- `positive_transitive_count`
- `unknown_transitive_count`
- `conflicted_component_count`
- `max_component_size`
- `weak_bridge_edge_count`

### Drift

按：

```text
source × brand × extractor_version × day
```

监控：

- conflict rate；
- abstain rate；
- reference parse failure rate；
- OCR confusion rate；
- 新 canonical reference 数量。

其中 `negative_transitive_conflict_count` 是一个非常适合线上告警的指标：如果某个平台抓取模板变化、某版 normalizer 出 bug，它往往会突然上升。

---

## 16. 建议的 MVP 技术栈

当前定义以 exact reference 为核心，因此没必要一开始上重型 graph database。

### 存储

**PostgreSQL** 可以先承担：

- Reference Registry；
- Product -> Reference 状态；
- provenance；
- review queue；
- 唯一约束；
- 增量事务。

100 万到 1000 万级的 exact-key/index 查询不是必须上图数据库的问题。

大规模原始抓取数据如果已放 ClickHouse / Hive / Lakehouse，可以继续保留，不需要迁移。

### 图计算

component 通常已经被 `(brand, canonical_reference)` blocking 限制得很小，可用：

- Python + NetworkX 做 MVP；
- 后续大批量可改成 igraph / graph-tool / Spark GraphFrames；
- weighted min-cut 只对 conflicted component 运行，不对全库运行。

### 图片

- 原图：S3 / OSS / MinIO；
- OCR worker 异步处理；
- OCR 结果作为 observation 回流。

### 向量召回

只在 unresolved 路径需要：

- FAISS / Milvus / pgvector 任选；
- 只返回 candidate reference；
- 永远不能因为 vector top-1 就自动 merge。

---

## 17. MVP 决策逻辑伪代码

```python
def resolve_product(product):
    observations = collect_reference_observations(product)

    candidates = []
    for obs in observations:
        if not is_grounded(obs):
            continue

        role = classify_identifier_role(obs)
        if role != "product_reference":
            continue

        norm = normalize_reference(product.brand, obs.raw_value)
        if norm.status == "VALID":
            candidates.append((norm.canonical_ref, obs))

    strong = select_non_conflicting_strong_candidates(candidates)

    if len(strong) == 0:
        return REVIEW("no strong canonical reference")

    if has_conflicting_strong_reference(strong):
        return REVIEW("reference conflict")

    canonical_ref = strong[0].canonical_ref
    ref_entity = get_or_create_reference_entity(product.brand, canonical_ref)

    proposed = propose_link(product, ref_entity, evidence=strong)

    audit = audit_affected_component(ref_entity)

    if audit.has_conflict:
        quarantine(proposed)
        enqueue_graph_review(audit)
        return REVIEW("transitive conflict")

    return VERIFIED(ref_entity)
```

Graph audit：

```python
def audit_component(component):
    conflicts = []
    unknowns = []

    for u, v in implied_transitive_pairs(component):
        result = verify_pair(u, v)

        if result == "CONFLICT":
            conflicts.append((u, v))
        elif result == "UNKNOWN":
            unknowns.append((u, v))

    if not conflicts:
        return PASS()

    suspect_edges = set()

    for u, v in conflicts:
        suspect_edges |= edges_on_shortest_paths(component, u, v)

    ranked = rank_by_evidence_reliability(suspect_edges)
    cut_plan = weighted_min_cut_if_needed(component, conflicts, ranked)

    return FAIL(conflicts, ranked, cut_plan)
```

---

## 18. 一条具体示例

假设三源出现：

### 雷小安

```text
劳力士 潜航者 126610LN 黑水鬼 41mm
structured_ref = 126610LN
```

### 腕表之家

```text
劳力士潜航者型系列 m126610ln-0001
reference = 126610LN
```

### 奢当家

```text
Rolex Submariner 126610LV 绿圈
structured_ref = 126610LV
```

如果一个语义 matcher 因为三者品牌、系列、尺寸、图片都很像，错误产生：

```text
A-B Match
B-C Match
```

则传统 connected component 会把三者合并。

当前 verifier 会直接得到：

```text
verify(A, C) = CONFLICT
126610LN != 126610LV
```

Graph Audit 会检查 A-C 路径：

```text
A -- B -- C
```

若 A-B 来自两侧 exact structured/title reference，而 B-C 是 embedding/LLM 弱边，则 B-C 被优先 cut。

最终：

```text
Reference Entity: Rolex/126610LN
  - 雷小安 A
  - 腕表之家 B

Reference Entity: Rolex/126610LV
  - 奢当家 C
```

这个例子说明 TransClean 的真正价值：

> 它不是帮我们把“相似的东西连起来”，而是在已经连起来以后，用组内矛盾发现一条危险的桥接边。

---

## 19. 建议的实施顺序

### Phase 1：先把确定性路径做对

1. Brand canonicalization；
2. identifier role 分类；
3. reference raw observation 保存；
4. brand-specific canonicalizer；
5. `(brand, canonical_reference)` Reference Registry；
6. strong exact link；
7. 所有不确定样本 abstain。

这一步甚至不需要通用 ML matcher，就能覆盖高 precision 的主路径。

### Phase 2：补 recall，但不改变最终规则

1. title NER / constrained LLM extraction；
2. OCR；
3. candidate reference retrieval；
4. DeepBlocker / embedding 只做 unresolved candidate retrieval；
5. 多模态只做辅助。

### Phase 3：加入 TransClean-inspired Safety Layer

1. 为 Reference Entity 构建 proposed-link component；
2. 三值 pair verifier；
3. transitive conflict 统计；
4. shortest-path suspect edge；
5. weighted minimum cut；
6. quarantine / review queue；
7. conservative edge recovery。

### Phase 4：持续增量与 drift

1. 按 affected reference bucket 局部重算；
2. extractor/normalizer versioning；
3. 每日 conflict dashboard；
4. hard-negative regression；
5. 人工 review 结果回流。

---

## 20. 最终建议

如果只从 TransClean 论文里拿一个设计原则，我认为应该是：

> **不要把 pairwise prediction 当最终结果；真正需要保证的是它们连接之后形成的 entity group 是否自洽。**

但结合当前 Spec，还要再向前走一步：

> **不要把通用 ML matcher 当 primary truth。业务已经明确 truth 是 reference equality，因此 primary truth 应该是 grounded + conservative canonical reference；TransClean 只作为 false-positive safety net。**

推荐最终生产架构：

```text
Grounded Reference Extraction
        ↓
Identifier Role Check
        ↓
Conservative Canonicalization
        ↓
Reference Entity Exact Link
        ↓
PROPOSED
        ↓
TransClean-inspired Group Audit
        ↓
 ┌───────────────┬────────────────┐
 ↓               ↓                ↓
VERIFIED       REVIEW          REJECTED
```

这种设计有几个直接收益：

1. **precision 可解释**：每一个自动 link 都能回到具体 reference 原文；
2. **能拒识**：没有足够证据时不做猜测；
3. **能发现系统性 bug**：错误 normalizer / OCR / extractor 会通过 transitive conflict 暴露；
4. **适合千万级**：按 `(brand, reference)` blocking 与增量 component 局部审计；
5. **能与已有三个研究方向组合**，而不是重复建设；
6. **人工预算利用率高**：只审最危险的 bridge edge / conflict component；
7. **方便回滚**：raw evidence、extractor version、decision version 全部保留。

对于“绝对不能误匹配”的要求，我不建议追求一个更复杂、更大的端到端 matcher，而建议把系统设计成：

> **高召回候选层 + 确定性 reference 决策层 + 图一致性安全层 + 默认拒识。**

这比单纯提高模型 F1 更贴合当前业务目标，也更容易直接进入生产实现。