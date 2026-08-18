# TransClean：用传递一致性给跨源二奢 Reference 匹配加一层“错边清洗 / 组件熔断”安全审计

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取论文 **TransClean: Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency** 做深入分析。

- 论文：<https://arxiv.org/abs/2506.04006>
- PDF：<https://arxiv.org/pdf/2506.04006>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前已先排除 `奢侈品调研/a` 中当前已有结果：

- `Ameli.md`
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
- `DeepBlocker.md`
- `Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes.md`
- `Tailoring entity resolution for matching product offers.md`
- `Using LLMs for the Extraction and Normalization of Product Attribute Values.md`
- `parts-distributor-sku-classifier.md`
- `pyJedAI.md`

`TransClean` 尚未出现在 `a` 目录，因此本次继续执行分析。

当前 Spec 的关键条件是：

1. 雷小安、腕表之家、奢当家三个来源；
2. 数据量约 100 万～1000 万，并持续增量；
3. “同一个商品”业务定义不是泛语义相似，而是 **同一 reference number / 型号**；
4. reference 字段稀疏，可能在结构化字段、标题、描述甚至图片中；
5. **precision 极端优先，绝对不能误匹配，允许漏匹配**；
6. 可人工标几百对黄金样本。

TransClean 最值得借鉴的不是某个 Transformer，而是一个很适合当前业务的系统思想：

> **不要只相信每一条 pairwise match；把 match 结果组成图后，利用同一实体应满足的传递一致性，主动寻找“把两个正确簇错误粘在一起”的 false-positive bridge edge，并优先切断这些错边。**

但是，本项目不能直接照搬论文。原因也非常关键：

> 当前业务已经把实体身份定义成 `reference equality`，因此最安全、最便宜、最可解释的主链路不应该是“商品 A 与商品 B 是否相同”的 pairwise ML，而应该是：
>
> ```text
> 商品记录 -> 唯一 Reference Entity
> ```
>
> 两条商品记录只有都安全地挂到同一个 `reference_entity_id`，才被视作“同一商品”。

所以我推荐把 TransClean 改造成 **Reference-first 系统的 Safety Auditor**：

```text
主链路：Reference 抽取 -> 保守规范化 -> Reference Entity -> exact join
安全层：冲突检测 -> 传递一致性审计 -> 低可信边切断 -> 人工复核
```

这样既利用了论文最有价值的图一致性思想，又不会让 ML/LLM/图片相似度越权定义“同款”。

---

# 1. 论文到底解决什么问题

传统 Entity Matching 一般把问题建模成二分类：

```text
f(record_i, record_j) -> Match / NoMatch
```

问题是，在多源环境里，真正输出的不是一堆彼此独立的 pair，而是由这些 pair 形成的连通分量。

例如：

```text
A --Match-- B --Match-- C
```

即使从未直接判断 `A vs C`，系统也已经通过传递关系隐式声明：

```text
A == B == C
=> A == C
```

于是，一条 false-positive edge 可能把两个本来完全独立的实体簇粘在一起，并产生大量错误的隐式匹配。

这对三源腕表尤其危险。

例如：

```text
雷小安记录 A：Rolex 126610LN
腕表之家记录 B：标题/OCR 被错误规范为 126610LN
奢当家记录 C：Rolex 126610LV

A --错误 Match-- B --错误/模糊 Match-- C
```

如果只看局部 pair，B 可能都“很像”；但整个组件已经出现硬冲突：

```text
A 的可信 reference = 126610LN
C 的可信 reference = 126610LV
```

在当前业务定义下，这个组件 **绝对不能作为一个实体发布**。

TransClean 就是在解决这种“pairwise 分数看起来不错，但组件语义已经被 false positive 污染”的问题。

---

# 2. Transitive Consistency：核心概念

论文把 pairwise matcher 的正预测构成无向图：

```text
G = (V, E)

V = records
E = matcher 判断为 Match 的 record pairs
```

如果两个节点：

- 属于同一 connected component；
- 但它们之间没有直接 edge；

那么这两个节点构成一个 **transitive match（隐式匹配）**。

例如：

```text
A -- B -- C
```

`A-C` 就是一个 transitive match。

论文定义的 Transitive Consistency 可以简化成：

```text
对于 G 中每一个 transitive pair (u, v)：
matcher(u, v) 都应该仍然预测 Match
```

如果出现：

```text
A --Match-- B --Match-- C
matcher(A, C) = NoMatch
```

说明 matcher 自己都不同意自己通过前两条边隐式得到的结论，组件中很可能存在 false-positive edge。

论文进一步把 transitive pair 的模型结果分成：

```text
Positive Transitive Prediction
Negative Transitive Prediction
```

实验中：

- Positive transitive predictions 与 true positives 数量有较强关联；
- Negative transitive predictions 与 false positives 数量有较强关联。

因此，即使没有整个数据集的人工真值，也可以用这两个量作为 matching 健康度 proxy。

这个思想非常适合生产环境，因为百万级 matching 不可能全部人工复核。

---

# 3. TransClean 原论文的完整处理流程

## 3.1 Step 1：Initial Steps with Finetuning

第一阶段目标不是把所有错边都找到，而是先找“破坏力最大”的 false positives。

论文会：

1. 找出 matching graph 的 connected components；
2. 对组件计算 negative transitive predictions；
3. 优先处理负传递预测多的组件；
4. 从组件里选一小批最可疑的显式边；
5. 人工标注或用外部模型 pseudo-label；
6. false-positive edge 从图中移除；
7. 新标注样本回流继续 fine-tune matcher；
8. 重算 transitive predictions；
9. 反复执行若干轮。

论文在选“最值得标”的 edge 时用了两个图结构信号。

### 3.1.1 Minimum Edge Cut

Minimum Edge Cut 是：

> 删除最少数量的边，就能把一个 connected component 切开的边集合。

如果一个组件的 negative transitive predictions 很多，那么把两个本来独立实体簇粘起来的 false-positive edge 往往处于图的“桥接位置”。

所以 minimum cut 很容易命中这种 bridge edge。

直觉示例：

```text
正确簇 1                   正确簇 2
A---B---C ---- x ---- D---E---F
               ^
          false-positive bridge
```

这条 `x` 边在结构上非常关键，删掉它就可以恢复两个簇。

### 3.1.2 Negative transitive pair 的 shortest path

假设：

```text
matcher(A, F) = NoMatch
```

但图上 A 与 F 处在同一个 connected component。

则从 A 到 F 的路径里，不可能所有 edge 都是真正的 Match，否则 A、F 也应表示同一个实体。

因此，对 negative transitive pair 找 shortest path，路径上的 edge 是高价值的“嫌疑边集合”。

这比全量人工检查组件中的所有边高效得多。

### 3.1.3 预算式标注

TransClean 明确以 labeling budget 工作。

不是：

```text
把所有可疑 pair 都检查完
```

而是：

```text
只消费 LB_total 允许的标注量
```

这和当前“只愿意人工标几百对”的约束高度一致。

---

# 4. Step 2：Post Finetuning Cleanup & Checks

初始几轮之后，模型和图都已经被清洗过，但仍可能存在不一致组件。

论文继续做：

```text
while 存在明显传递不一致的 component:
    找 Minimum Edge Cut
    prune edge
    重新计算 components / transitive predictions
```

然后再对：

- 超大组件；
- 仍有 negative transitive prediction 的组件；
- 之前标注信息与当前图发生冲突的组件；

做更强的检查。

论文在实验中设置：

```text
initial finetuning iterations n = 5
component size threshold S = 50
```

要注意：`S=50` 是论文实验参数，不应该直接复制到腕表系统。

当前系统中热门 reference 可能天然有成千上万条 listing；如果按 listing 建 clique/普通 pairwise component，那么“大组件”是业务正常现象，而不是错误信号。

这也是为什么我后面建议做 Reference Entity 压缩，而不是把 TransClean 直接跑在原始商品图上。

---

# 5. Step 3：Edge Recovery

前两个阶段为了消灭 false positives，会主动删边，其中一定可能误删少量 true positives。

所以论文最后增加 Edge Recovery：

对每个之前删除、当前 matcher 仍认为是 Match 的 edge `e`：

1. 假设把 `e` 加回来；
2. 计算它会新产生哪些 transitive matches；
3. 如果所有新增 transitive matches 仍被 matcher 判断为 Match，则恢复这条 edge；
4. 如果出现 NoMatch，则进入人工检查；
5. 标注预算不足就保持删除。

这是一个很合理的 recall 恢复策略。

但是 **当前二奢需求应该比论文更保守**。

由于业务明确：

```text
precision >> recall
```

因此我的建议是：

> 不要实现“模型一致就自动恢复”的完整版 Edge Recovery。
>
> 只允许满足 reference 硬证据的边自动恢复；其余继续 abstain / 人工复核。

也就是说，论文追求的是“清错后再尽量拿回 recall”，当前系统应该改成“清错后默认不拿回 recall”。

---

# 6. 论文实验对当前需求最有启发的结果

论文使用 DistilBERT 和 CLER 两类 pairwise matcher，并在多源实体数据上验证。

几个结果非常值得注意。

## 6.1 Pairwise 指标好，不代表实体组件安全

Synthetic Companies 上，DistilBERT 单看 pairwise：

```text
Precision 85.86
Recall    77.63
F1        81.54
```

看起来不是一个灾难模型。

但是把 pairwise edges 所隐含的 transitive matches 也计入后，Pre-TransClean precision 会下降到极低水平。

这说明：

> false positive 的真实损失不是“错一对”，而可能是“通过连通分量扩散成很多错对”。

当前商品系统绝对应该按 **component / entity** 评估，而不是只算 pair-level F1。

## 6.2 好 matcher + TransClean 才是合理组合

Synthetic Companies 上，CLER 产生的 matching 本身较好，TransClean 后：

- 移除了大比例 false positives；
- 只损失少量 true positives；
- Post-TransClean precision 进一步提升。

反过来，当底层 matcher 很差、false positive 很多时，TransClean 为了恢复一致性也会牺牲较多 true positives。

论文自己的结论也很明确：

> TransClean 最适合叠加在一个已经比较可靠的 matcher 上，而不是拿来挽救任意垃圾 matcher。

这正好支持当前项目采用：

```text
高精度 Reference 硬规则
        +
TransClean 风格图审计
```

而不是：

```text
模糊 embedding/LLM matcher
        +
期待 TransClean 把它救回来
```

## 6.3 LLM pseudo-label 不能当最终安全依据

论文还测试了用 7B LLM 做 pseudo-label。

结果总体上：

- 当底层 matching 已经不错时，LLM pseudo-label 还可以工作；
- 当数据复杂、底层模型较弱时，LLM pseudo-label 会导致更多 true positive 被错误删除，也会漏掉更多 false positive；
- LLM 推理还会明显增加运行时间。

因此当前项目里：

> LLM 适合当 candidate extractor、解释器、人工复核助手；不应该当“最终同 reference 裁判”。

---

# 7. 对当前 Spec 的关键改造：不要做 Pairwise Matching 主架构

当前最重要的业务定义是：

```text
same product <=> same reference number
```

在这个定义下，最自然的数据模型其实不是：

```text
record A --Match--> record B
```

而是：

```text
record A ----> ReferenceEntity(ROLEX, 126610LN)
record B ----> ReferenceEntity(ROLEX, 126610LN)
record C ----> ReferenceEntity(ROLEX, 126610LV)
```

最终：

```text
A 与 B：同一商品
A 与 C：不是同一商品
```

这样有四个巨大好处。

## 7.1 从 O(N²) pairwise 退化成 O(N) entity linking

1000 万商品如果做全 pair：

```text
N*(N-1)/2
```

完全不可行。

但如果每条 record 只做一次 reference extraction + dictionary lookup：

```text
O(N)
```

数据库的 `(brand_id, canonical_reference)` 索引负责聚合即可。

## 7.2 新增来源不会改变历史 pair 数量级

增量进来一条 record，只需要：

```text
extract reference
-> lookup reference_entity
-> attach / abstain
```

不需要和已有数百万 listing 全比较。

## 7.3 业务规则天然可解释

匹配理由不是：

```text
embedding = 0.938
LLM says yes
image looks similar
```

而是：

```text
brand = Rolex
canonical_reference = 126610LN
reference evidence = structured field + title
no conflicting evidence
```

对运营和错误排查友好很多。

## 7.4 图污染范围变小

错误边不再直接把两个巨大 listing component 合并。

每条 record 只有一个“候选挂载到 reference entity”的决策，审计可以直接针对这个 link 做 quarantine。

---

# 8. 推荐落地架构

```text
                    ┌────────────────────────┐
雷小安 ────────────>│                        │
腕表之家 ──────────>│  Ingestion / Normalize │
奢当家 ────────────>│                        │
                    └───────────┬────────────┘
                                │
                                v
                    ┌────────────────────────┐
                    │ Brand / Product Role   │
                    │ Canonicalization       │
                    └───────────┬────────────┘
                                │
                                v
                    ┌────────────────────────┐
                    │ Reference Candidate    │
                    │ Extraction             │
                    │                        │
                    │ structured / title     │
                    │ description / OCR      │
                    │ LLM candidate only     │
                    └───────────┬────────────┘
                                │
                                v
                    ┌────────────────────────┐
                    │ Conservative Reference │
                    │ Normalizer + Grammar   │
                    └───────────┬────────────┘
                                │
                                v
                    ┌────────────────────────┐
                    │ Decision Engine        │
                    │ ACCEPT / ABSTAIN       │
                    │ / REJECT               │
                    └───────────┬────────────┘
                                │ ACCEPT
                                v
                    ┌────────────────────────┐
                    │ Reference Entity Store │
                    │ UNIQUE(brand, ref)     │
                    └───────────┬────────────┘
                                │
                                v
                    ┌────────────────────────┐
                    │ TransClean-inspired    │
                    │ Safety Auditor         │
                    │ conflict / cut / queue │
                    └───────────┬────────────┘
                                │ healthy only
                                v
                    ┌────────────────────────┐
                    │ Published Match View   │
                    │ cross-source grouping  │
                    └────────────────────────┘
```

---

# 9. Reference Candidate Extraction：只“提候选”，不要直接 Match

每条商品先抽一个或多个 reference candidate，并记录来源证据。

建议候选表至少保留：

```text
record_id
candidate_raw
candidate_canonical
brand_id
channel
span / image_id
role
extractor_version
confidence
created_at
```

`channel` 可以是：

```text
structured_reference
title
description
image_ocr
llm
catalog_lookup
```

## 9.1 结构化 Reference

平台自己明确提供的型号字段，是最高优先级候选。

但仍然不能无脑信任，因为字段里可能出现：

- 平台内部货号；
- 商家 SKU；
- 库存编号；
- 证书号；
- 多个 reference 混填。

所以需要做 `identifier_role`：

```text
brand_reference
platform_sku
seller_sku
serial_number
certificate_number
compatible_reference
unknown_identifier
```

只有 `brand_reference` 才有资格进入最终匹配主键。

## 9.2 标题 / 描述

标题中不要让 LLM 自由生成 reference，而是：

```text
regex / grammar / known catalog
        -> candidate spans
        -> role classifier / contextual verifier
```

例如：

```text
“适配 Rolex 126610LN 的表带”
```

里面虽然存在 `126610LN`，但它不是当前商品自身 reference，应标成：

```text
compatible_reference
```

而不是：

```text
self_reference
```

## 9.3 图片 OCR

图片很有价值，但只能作为独立证据源。

优先 OCR：

- 保卡；
- 吊牌；
- 表背；
- 盒证；
- 机芯 / 壳体刻字。

OCR 结果必须保留 raw token。

尤其不能直接做：

```text
O -> 0
I -> 1
B -> 8
```

这类“看起来合理”的 fuzzy normalization，因为当前需求最怕的就是把相邻 reference 合并。

图片视觉 embedding 可以用来：

- 辅助判断“这个 reference 候选明显不可能属于该系列”；
- 给人工复核排序；
- 检查商品是不是配件/盒证而不是腕表主体；

但不能因为两块表“长得很像”就越过 reference 规则自动 Match。

---

# 10. Conservative Reference Normalizer：这是整个系统最重要的组件之一

不要做一个全品牌统一、激进的字符串 normalize。

应该：

```text
raw_reference
-> unicode normalization
-> conservative lexical normalization
-> brand-specific grammar
-> canonical_reference
```

## 10.1 可以默认做的低风险变换

例如：

```text
Unicode NFKC
英文统一大写
去首尾空白
统一全角/半角
把已确认语义等价的 Unicode dash 统一
清理明确的展示噪声
```

## 10.2 不应全局默认做的变换

```text
O <-> 0
I <-> 1
L <-> 1
B <-> 8
删除所有 . / - / /
模糊编辑距离自动纠错
根据视觉相似猜型号
```

是否允许去掉某种 separator，应由 **品牌规则版本** 决定。

例如：

```text
normalize_reference(brand_id, raw_ref, rule_version)
```

必须版本化，否则未来更新 normalization 规则后无法解释历史为什么被合并。

---

# 11. Reference Entity 数据模型

推荐主表：

```sql
CREATE TABLE reference_entity (
    id BIGSERIAL PRIMARY KEY,
    brand_id BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    reference_family TEXT,
    rule_version TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);
```

record 到 entity 的挂接不要只存最终 id，要完整保留证据：

```sql
CREATE TABLE record_reference_link (
    record_id BIGINT PRIMARY KEY,
    reference_entity_id BIGINT,
    decision TEXT NOT NULL, -- ACCEPT / ABSTAIN / REJECT
    evidence_grade TEXT,
    decision_reason JSONB NOT NULL,
    extractor_version TEXT NOT NULL,
    normalizer_version TEXT NOT NULL,
    decision_version TEXT NOT NULL,
    reviewed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
```

真正对外发布的“同款关系”只来自：

```sql
WHERE decision = 'ACCEPT'
```

然后按：

```text
reference_entity_id
```

分组。

---

# 12. Decision Engine：必须是三态，而不是普通二分类

论文里的 matcher 是：

```text
Match / NoMatch
```

当前系统应该改成：

```text
ACCEPT
REJECT
ABSTAIN
```

因为“证据不足”不等于“NoMatch”，更不应该被模型强行猜一个结果。

推荐最低限度策略：

```python
if trusted_references_conflict(record):
    return REJECT

if brand_is_uncertain(record):
    return ABSTAIN

if one_high_trust_self_reference(record) and no_conflict(record):
    return ACCEPT

if two_independent_channels_agree_on_same_reference(record) and no_conflict(record):
    return ACCEPT

return ABSTAIN
```

对于跨记录同款：

```python
def same_product(a, b):
    return (
        a.decision == "ACCEPT"
        and b.decision == "ACCEPT"
        and a.reference_entity_id == b.reference_entity_id
    )
```

不要存在：

```python
if embedding_similarity > 0.95:
    return MATCH
```

这种旁路。

---

# 13. 把 TransClean 改造成 Safety Auditor

这是本次分析最建议直接落地的部分。

## 13.1 图不应该是所有 listing 的 pairwise 图

如果 Rolex 某个热门 reference 有 10 万条 listing，按普通 EM 建图会出现巨大组件。

而当前业务里这 10 万条本来就可以共享一个 `reference_entity_id`，根本没必要计算：

```text
10万 * 99999 / 2
```

个 transitive pairs。

所以审计图只需要覆盖“可能产生错误挂接的 evidence links”。

可以把图建成：

```text
Record Node
Reference Candidate / Reference Entity Node
Evidence Edge
```

示例：

```text
record_1 --structured(high)--> ref:126610LN
record_2 --title(mid)--------> ref:126610LN
record_2 --ocr(low)----------> ref:126610LV
record_3 --structured(high)--> ref:126610LV
```

现在 record_2 本身就是冲突桥。

不需要全 pairwise comparison 就能发现。

---

# 14. 把“Negative Transitive Prediction”升级成“Hard Conflict”

论文用 matcher 对 transitive pair 再跑一次，得到 Match / NoMatch。

但当前业务拥有比 generic matcher 更强的业务不变量：

```text
可信 canonical reference 不同 => 必须 NoMatch
可信 brand 不同 => 必须 NoMatch
商品主体 vs 配件 => 不能按主体 reference Match
```

所以我建议定义三值 verifier：

```text
VERIFY_MATCH
VERIFY_CONFLICT
VERIFY_UNKNOWN
```

其中：

```python
def verify(a, b):
    if trusted_brand(a) != trusted_brand(b):
        return VERIFY_CONFLICT

    if trusted_ref(a) and trusted_ref(b):
        if trusted_ref(a) == trusted_ref(b):
            return VERIFY_MATCH
        return VERIFY_CONFLICT

    return VERIFY_UNKNOWN
```

与原论文相比，这个版本更适合 precision-first：

- `UNKNOWN` 不应该被当作 negative；
- 只有硬冲突才产生 Safety Alarm；
- 不用强迫模型对所有 transitive pair 猜 Match/NoMatch。

---

# 15. “任何 Hard Conflict 都熔断”，不要照搬论文的多数投票阈值

论文有一个重要 pruning 条件，是比较组件里的 positive / negative transitive predictions。

对通用 EM 这很合理。

但当前需求说：

```text
绝对不能误匹配
```

所以组件发布条件应该更严格：

```text
hard_conflict_count == 0
```

而不是：

```text
positive_count > negative_count
```

一旦发现高可信 reference 冲突：

```text
component.status = QUARANTINED
```

整个组件暂时不对外输出自动合并结果，直到冲突边被定位并切掉。

这个改造比原论文更符合业务风险函数。

---

# 16. Minimum Cut 也要改：从“最少边”变成“最小可信证据损失”

原论文使用 Minimum Edge Cut。

但在实际商品系统里，两条边的可信度完全不同：

```text
结构化 reference 精确字段
```

和：

```text
低清图片 OCR + LLM 猜测
```

不能有同样的删除成本。

因此实际应使用 **weighted cut**。

给 edge 定义 retention cost：

```text
structured validated reference       1000
manual verified reference            1000
2 independent channels agree          500
known catalog + exact title span       300
title only                              80
OCR only                                30
LLM inferred                            10
fuzzy similarity                         1
```

然后对两组冲突 anchor 做：

```text
minimum total weight cut
```

含义是：

> 为了解开冲突组件，优先切掉总可信度最低的那组 edge。

这比最小 cardinality cut 更符合生产语义。

实现上小组件可以直接使用：

```text
networkx.minimum_cut
```

大规模不建议对全图做通用 max-flow，而应先通过 reference key 和冲突 anchor 把问题裁成很小的局部子图。

---

# 17. Anchor：让高可信 Reference 成为图里的“不可轻易移动节点”

建议定义 `anchor`：

```text
人工确认 reference
或
结构化 reference + brand grammar 校验通过 + 无冲突
或
两个独立高可信通道完全一致
```

每个 component 最终只能有：

```text
0 个 canonical anchor reference
或
1 个 canonical anchor reference
```

如果出现：

```text
2 个不同 canonical references 的 anchor
```

则组件天然不合法。

这相当于把“业务主键不变量”放到了 TransClean 的图一致性之上。

---

# 18. 推荐的组件审计伪代码

```python
def audit_component(component):
    anchors = get_high_trust_reference_anchors(component)
    refs = {a.canonical_reference for a in anchors}

    # 没有可信 anchor，不自动发布
    if len(refs) == 0:
        quarantine(component, reason="NO_TRUSTED_REFERENCE")
        return

    # 唯一可信 ref：继续检查成员冲突
    if len(refs) == 1:
        canonical_ref = next(iter(refs))
        for member in component.members:
            verdict = verify_member_against_ref(member, canonical_ref)
            if verdict == "CONFLICT":
                quarantine_link(member, canonical_ref)
        publish_if_no_conflict(component)
        return

    # 多个可信 ref：一定有错边
    conflict_groups = group_anchors_by_reference(anchors)

    suspect_subgraph = build_local_subgraph_between(conflict_groups)
    cut_edges = weighted_min_cut(suspect_subgraph)

    for edge in cut_edges:
        quarantine_edge(edge)

    recompute_components()
```

这就是 TransClean 在当前业务里的精简、可落地版本。

---

# 19. Shortest Path 仍然很有价值：主要用于“解释为什么冲突”

原论文用 negative transitive pair 的 shortest paths 挖嫌疑 edge。

当前系统也可以保留，但用途建议从“主要算法”改成：

```text
审计解释 + 人工复核排序
```

例如两个可信 anchor 冲突：

```text
126610LN anchor
    |
  record A
    |
  OCR edge
    |
  record B
    |
 title edge
    |
126610LV anchor
```

系统把最短冲突路径直接展示给审核员：

```text
冲突来源：
1. A 结构化字段 = 126610LN
2. A/B 因 OCR "126610L?" 被关联
3. B 标题明确 = 126610LV
```

审核员不必理解整个 component，只看这条路径即可。

---

# 20. 人工标注的几百对应该怎么花

不要随机标 500 对。

随机样本极可能大部分都是容易样本，对 precision tail 没帮助。

应该让 Safety Auditor / Active Review Queue 专门抽：

1. 同品牌、同系列、reference 只差 1 个字符；
2. OCR `0/O`、`1/I`、`8/B` 混淆；
3. 同标题同时出现多个 reference；
4. “适配某型号”的配件；
5. 平台 SKU 与品牌 reference 形态相似；
6. 结构化字段与标题冲突；
7. 标题与图片 OCR 冲突；
8. 多源组件中的 bridge edge；
9. 新品牌 / 新来源刚上线后的分布漂移样本；
10. 系统原本想 ACCEPT、但 Safety Auditor 熔断的样本。

这几百对的价值主要用于：

```text
identifier role classifier
reference extractor
brand-specific grammar
conflict verifier
threshold / policy calibration
```

而不是训练一个“万能同款 LLM”。

---

# 21. 一个重要统计边界：几百个 0-error 样本不能证明 99.99% precision

当前需求虽然说“绝对不能错”，工程上一定还需要可度量的验收方式。

如果测试集中 500 个自动匹配全部正确，直觉上会觉得 precision=100%。

但从统计置信下界看，500 个 0-error 样本并不足以证明真实 precision 达到 99.99%。

以单侧 95% 置信下界为例，500/500 全对时，下界大约只有：

```text
99.4%
```

想靠“零错误抽样”把单侧 95% 下界推到 99.99% 左右，需要约 3 万个全对样本量级。

因此当前“几百对黄金样本”更适合：

```text
找 hard cases + 校准策略 + 回归测试
```

不能拿来宣称四个 9 的统计保证。

真正降低线上风险要靠：

```text
硬业务不变量
+ 强拒识
+ 组件冲突熔断
+ 持续抽检
+ 错误可回滚
```

而不是只靠一次离线 precision 数字。

---

# 22. 增量更新怎么做

当前系统持续增量，因此每条新记录应走事件式流程。

```text
record_ingested
    -> extract_candidates
    -> normalize_candidates
    -> decision
    -> attach_reference_entity / abstain
    -> audit affected reference component
    -> publish
```

关键点是只重算受影响局部：

```text
record_id
brand_id
candidate_reference
reference_entity_id
```

对应的 component。

不要每天重跑 1000 万全量 pairwise matcher。

如果 normalizer / extractor 升级：

```text
rule_version / extractor_version
```

发生变化，只重放受影响品牌或规则命中的 records。

---

# 23. 推荐存储与服务拆分

10M 级数据不需要一上来引入重型 graph database。

一个可维护的第一版：

```text
PostgreSQL
  - record 元数据
  - reference candidates
  - reference entities
  - accepted links
  - review queue

Object Storage (S3 / OSS / COS)
  - 原始页面
  - 图片
  - OCR artifact

Python Workers
  - extraction
  - normalization
  - decision
  - local graph audit

Redis / MQ（可选）
  - 增量任务
  - 幂等重试
```

如果后续批处理量进一步增加，可把离线抽取放到 Spark / Ray，但最终 authority store 仍应保持简单。

图审计本身只跑异常局部，不需要把全部商品搬进 Neo4j。

---

# 24. 生产表建议

至少需要下面几类数据。

## 24.1 `product_record`

```text
id
source
source_record_id
brand_raw
title
description
structured_attributes
image_ids
crawl_version
created_at
updated_at
```

## 24.2 `reference_candidate`

```text
id
record_id
raw_value
canonical_value
brand_id
channel
identifier_role
span_or_image
confidence
extractor_version
normalizer_version
```

## 24.3 `reference_entity`

```text
id
brand_id
canonical_reference
reference_family
rule_version
```

唯一约束：

```text
UNIQUE(brand_id, canonical_reference)
```

## 24.4 `record_reference_link`

```text
record_id
reference_entity_id
decision
evidence_grade
decision_reason
decision_version
review_state
```

## 24.5 `audit_event`

```text
event_id
component_key
event_type
conflict_path
cut_edges
before_state
after_state
policy_version
created_at
```

这样任何误合并都可以复盘与回滚。

---

# 25. 自动 ACCEPT 的建议证据等级

为了 precision-first，可以把自动路径做得非常窄。

## Grade A：自动 ACCEPT

例如：

```text
品牌确定
+
结构化 reference 明确是 brand_reference
+
brand-specific grammar 校验通过
+
没有任何高可信冲突证据
```

或者：

```text
标题 exact reference
+
独立 OCR / 结构化字段 exact 同值
+
无冲突
```

## Grade B：可选自动 ACCEPT，但应先离线验证

```text
标题 exact reference
+
已存在于高可信 reference catalog
+
品牌与系列上下文一致
+
无其他 reference 候选
+
无配件语义
```

是否开启 Grade B 自动路径，应由黄金样本和持续抽检决定。

## Grade C：ABSTAIN

```text
OCR only
LLM only
fuzzy corrected reference
多个候选
跨通道冲突
低可信品牌
```

## Grade D：REJECT

```text
两个高可信 reference 明确不同
品牌硬冲突
商品被识别为配件但 reference 属于被适配腕表
```

---

# 26. 图片应该如何参与

图片最大的风险是被错误用成“同款主证据”。

建议图片只进入两条路径。

## 26.1 Reference OCR

```text
image
-> region / document type detection
-> OCR
-> reference candidate grammar
-> evidence channel=image_ocr
```

这条路径仍然由 Reference Entity 收口。

## 26.2 Conflict veto

假设文本说：

```text
126610LN
```

但图片 OCR 明确得到另一个已知 reference：

```text
126610LV
```

不要让系统“平均两个分数”。

应该：

```text
HARD CONFLICT
-> ABSTAIN / review
```

这和 TransClean 的思想完全一致：

> 一个强 negative consistency signal 应该阻止错误关系向组件扩散。

---

# 27. LLM 应该放在哪里

推荐允许：

1. 从复杂标题 / 描述中找 identifier span；
2. 判断 identifier 是 `self_reference` 还是 `compatible_reference`；
3. 给人工审核解释冲突；
4. 对 hard-negative review queue 生成复核摘要；
5. 帮忙从人工结果归纳新的 brand rule 候选。

不推荐：

```text
LLM(A, B) = “是同一商品”
=> 自动合并
```

原因既来自当前 precision 约束，也被 TransClean 的 LLM pseudo-label 实验侧面支持。

---

# 28. 推荐直接落地的 MVP

## Phase 1：Reference-first 安全基线

先完全不做复杂 pairwise ML。

完成：

1. 统一商品 schema；
2. brand canonicalization；
3. structured/title reference candidate extraction；
4. conservative normalizer；
5. `reference_entity`；
6. `ACCEPT / ABSTAIN / REJECT`；
7. exact join；
8. 冲突日志。

这一步就能安全覆盖大量有明确 reference 的数据。

## Phase 2：图片 + Identifier Role

增加：

```text
OCR
商品主体/配件判断
self_reference vs compatible_reference
platform_sku vs brand_reference
```

重点不是提高 recall，而是降低“错误 reference 被当主键”的概率。

## Phase 3：TransClean Safety Auditor

增加：

```text
component anchor
hard conflict detection
conflict shortest path
weighted cut
quarantine
manual review queue
```

## Phase 4：Hard-negative 学习闭环

人工复核结果回流：

```text
review labels
-> extractor training
-> role classifier
-> grammar update
-> regression suite
```

但 Reference equality 硬规则不被模型替代。

---

# 29. Safety Auditor 的核心状态机

推荐 component / reference entity 有明确状态：

```text
HEALTHY
SUSPECT
QUARANTINED
REVIEWED
```

发布规则：

```text
只有 HEALTHY / REVIEWED
可以对外产生自动跨源同款关系
```

发生下面事件立即从 `HEALTHY -> QUARANTINED`：

```text
trusted_ref_conflict
trusted_brand_conflict
manual_negative_edge
new extractor version changes canonical ref
new source introduces conflicting anchor
```

这非常适合持续增量环境。

---

# 30. 回滚必须是一等能力

precision-first 系统仍然不能假设永远零错误。

所以每个自动 link 都要能回答：

```text
谁在什么时候
依据什么 raw evidence
通过哪个 extractor / normalizer / decision policy
把 record 挂到了哪个 reference_entity
```

如果某条规则被证明错误，应支持：

```text
invalidate policy version
-> re-evaluate affected records
-> detach wrong links
-> recompute affected components
```

不要把 merge 做成不可逆的数据覆盖。

---

# 31. 如何评测：不要只看 Pair F1

至少需要四层指标。

## 31.1 Extraction

```text
reference candidate precision
reference candidate recall
identifier role precision
```

## 31.2 Auto-link

最重要：

```text
auto_accept_precision
coverage / acceptance_rate
```

当前业务宁愿：

```text
coverage 30%, precision 极高
```

也不要：

```text
coverage 95%, precision 98%
```

## 31.3 Component Safety

```text
hard_conflict_component_rate
quarantine_rate
false_merge_component_rate
largest_wrong_component_size
```

尤其要看：

```text
一个 false positive 最多污染了多少 records
```

## 31.4 增量稳定性

```text
new-source conflict rate
rule-version regression
brand drift alert
OCR drift alert
```

---

# 32. 测试集必须包含真正危险的 hard negatives

重点构造：

```text
126610LN vs 126610LV
116500LN vs 126500LN
同 reference stem 不同 suffix
同系列不同材质/尺寸 reference
平台货号恰好长得像 reference
盒证/表带标题引用腕表 reference
OCR 0/O 误读
标题一个 reference、图片另一个 reference
卖家复制粘贴了其他型号描述
同品牌同系列外观高度相似但 reference 不同
```

系统验收要明确：

> 这些 hard negatives 只要有一个进入 auto-merge，就应阻断上线或降级对应自动规则。

---

# 33. TransClean 思想与当前业务的一一映射

| TransClean 原论文 | 当前二奢系统建议 |
|---|---|
| Pairwise Match edge | record -> reference candidate/entity evidence edge |
| Connected Component | 一个 reference 候选关系局部组件 |
| Transitive Match | 同组件内隐含“应为同 reference”的关系 |
| Negative Transitive Prediction | trusted canonical reference / brand 的 Hard Conflict |
| Minimum Edge Cut | weighted low-trust cut |
| Shortest Path | 两个冲突 anchor 之间的解释路径 |
| Labeling Budget | 几百条人工 hard-negative 预算 |
| Finetuning | extractor / role classifier / verifier 回流训练 |
| Post Cleanup | quarantine + 反复重算局部组件 |
| Edge Recovery | 只允许 hard-reference evidence 恢复；否则 abstain |

这个映射比直接照搬论文算法更适合当前 Spec。

---

# 34. 为什么这个方案比“先做 embedding matching，再阈值拉高”更安全

embedding / cross encoder / LLM 的高分都存在一个共同问题：

```text
“非常像” != “reference 完全相同”
```

腕表恰恰存在大量：

- 同系列；
- 同盘面；
- 同外壳；
- 只差年份 / 材质 / 尺寸 / suffix；
- reference 只差一个字符；

的 hard negatives。

如果业务真值已经明确是 reference equality，最合理的 inductive bias 就是直接把 reference 做成实体主键。

ML 负责：

```text
把埋在脏文本/图片里的 reference 找出来
```

而不是负责：

```text
重新定义“什么叫同款”
```

TransClean 则负责：

```text
当某个错误抽取/错误 link 试图污染实体组件时，及时发现并切断
```

三者职责完全分离。

---

# 35. 最终推荐

如果只能从 TransClean 中拿一个思想，我会选：

> **把实体匹配的安全性从“单条 pair 分数”提升到“整个组件是否自洽”，任何一条高可信冲突都必须阻断发布。**

如果要直接落地，我推荐最终系统形态是：

```text
                Reference-first Entity Resolution

Record
  |
  v
Reference Candidate Extraction
  |
  v
Conservative Canonicalization
  |
  +---------------------> conflict -> ABSTAIN / REVIEW
  |
  v
Reference Entity (brand, canonical_ref)
  |
  v
Exact Grouping
  |
  v
TransClean-inspired Safety Auditor
  |
  +---- hard conflict ----> QUARANTINE + weighted cut
  |
  v
Publish Cross-source Match
```

核心规则：

```text
1. 不做全量 pairwise matching。
2. 不允许 fuzzy reference 自动等价。
3. LLM / 图片不拥有最终 Match 权限。
4. 同款只由同一 accepted reference_entity_id 导出。
5. 所有不确定情况显式 ABSTAIN。
6. 所有组件出现 trusted reference 冲突立即熔断。
7. 图切边优先切低可信 evidence，而不是简单切最少边。
8. 自动恢复边只能依赖 reference 硬证据。
9. 人工预算优先花在冲突 bridge / hard negatives。
10. 所有规则、证据、决策版本化，可回滚。
```

这套架构既能直接满足三源 100 万～1000 万级增量数据，又把“绝对不能误匹配”的约束放到了系统最底层，而不是寄希望于某个模型阈值。

---

# 36. 一句话总结论文对当前项目的价值

**TransClean 证明了：在多源实体解析中，少量 false-positive edge 会通过传递关系放大成严重的组件污染；当前腕表项目应进一步利用业务已知的 reference equality，把“传递一致性”升级成硬冲突审计，并用 Reference Entity + 强拒识 + 组件熔断构成最终生产安全线。**
