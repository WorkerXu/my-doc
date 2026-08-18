# TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency

> 分析人：c  
> 论文：Fernando De Meer Pardo, Branka Hadji Misheva, Martin Braschler, Kurt Stockinger, **TransClean: Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency**, arXiv:2506.04006, 2025  
> 论文：https://arxiv.org/abs/2506.04006  
> 论文 HTML：https://arxiv.org/html/2506.04006v1  
> 调研清单：`奢侈品文章调研.md`  
> 目标需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务定义：**“同一个商品” = 同一 reference number / 型号**；数据规模 100 万–1000 万、持续增量；字段稀疏；图片可用；**precision 极端优先，绝不能误匹配，允许漏匹配**。

---

## 1. 选题与去重

执行前检查 `奢侈品调研/c/`，c 已经分析过：

- `Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes.md`
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
- `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
- `pyJedAI.md`

本次选择 **TransClean**，c 目录中尚无该文章，因此不重复。

之所以优先分析它，是因为当前需求最大的业务风险不是“漏掉一个同款”，而是**一条错误匹配边把两个本不相同的 reference 簇连接起来，随后通过传递闭包污染整组数据**。TransClean 正是从这个问题出发：不只看 pairwise classifier 的单边准确率，而是检查“模型建立出来的整张匹配图是否自洽”，并重点寻找会造成错误合并传播的 false positive edge。

论文最值得迁移到本项目的一句话可以概括为：

> pairwise 模型的一条 false positive，不只是错一对；在多源场景里，它可能把两个连通分量错误地粘成一个实体组，从而隐含地产生大量错误 transitive matches。

对腕表 reference matching，这个问题甚至比论文的一般实体匹配场景更适合用“传递一致性”处理，因为“同一 reference”天然应该满足**等价关系**：自反、对称、传递。

---

## 2. 论文真正解决的问题

传统实体匹配通常把任务写成：

```text
f(record_i, record_j) -> MATCH / NO_MATCH
```

然后用 pairwise precision / recall / F1 评估。

但多源实体匹配的最终产物通常不是一堆孤立 pair，而是由 MATCH 边组成的图：

```text
record A --MATCH-- record B --MATCH-- record C
```

即使 A 与 C 从未直接评估，系统也已经隐含认为：

```text
A == B
B == C
=> A == C
```

论文把这种“同一连通分量中、没有直接边但由路径隐含出来的 pair”称为 **transitive match**。

### 2.1 Transitive Consistency

设 pairwise matcher 为 `fθ`，其预测出的 MATCH 边形成图：

```text
G_fθ = (V, E_fθ)
```

其中：

- `V`：record；
- `E_fθ`：pairwise matcher 判为 MATCH 的边。

如果两个节点属于同一 connected component，但没有直接边，则它们形成一个 transitive match。

论文定义的 **Transitive Consistency** 是：

> 对图中所有 transitive match，如果重新拿给同一个 matcher 判断，都应该仍然得到 MATCH。

因此：

```text
A --MATCH-- B --MATCH-- C
```

如果模型对 `A,C` 又输出：

```text
A --NO_MATCH-- C
```

那么这个 component 就不一致。

论文进一步把 transitive pair 的模型判断分成：

```text
positive transitive prediction -> MATCH
negative transitive prediction -> NO_MATCH
```

实验发现：

- positive transitive predictions 与 true positives 的数量相关；
- negative transitive predictions 与 false positives 的数量相关；
- 因而在没有完整 ground truth 时，两者可作为匹配图质量的可观测 proxy。

这正适合真实生产环境：不可能人工检查百万级匹配边，但可以用图结构把人工预算集中在最可能“污染整簇”的位置。

---

## 3. TransClean 的核心算法

TransClean 不是一个新的 matcher，而是一个可以包在任意 pairwise matcher 外面的 **graph cleanup / auditing layer**。

论文流程分三阶段。

### 3.1 阶段一：Initial Steps with Finetuning

目标：先发现影响最大的 false positive edge，并把这些难例反哺 matcher。

论文做法：

1. 对匹配图求 connected components；
2. 计算/抽样计算 component 中的 transitive pairs；
3. 按 negative transitive predictions 数量优先处理最可疑 component；
4. 在 component 中重点选两类边去人工标注：
   - **Minimum Edge Cut** 中的边；
   - negative transitive pair 两端之间 shortest path 上的边；
5. 把标成 false positive 的边删除；
6. 用得到的 hard cases 继续 fine-tune matcher；
7. 对 negative transitive 明显多于 positive transitive 的 component，继续用 min-cut 拆分。

为什么 min-cut 有用？

如果一条错误 MATCH 边恰好把两个原本独立的真实实体组连起来，那么它往往位于一个“窄桥”上。最小割就是寻找最少删除哪些边可以把 component 拆开的图算法，因此天然适合定位这种污染边。

为什么 shortest path 有用？

如果 `A` 和 `C` 是一个 negative transitive pair，但图上又存在：

```text
A - B - D - C
```

那么这条路径中至少应该有一条错误 MATCH，否则全是真正等价关系时 A 与 C 不可能 NO_MATCH。因此 shortest path 上的边是高价值 review 候选。

### 3.2 阶段二：Post Finetuning Cleanup & Checks

第一阶段之后，论文继续：

- 反复拆分仍存在大量 negative transitive predictions 的 component；
- 对超大 component 做额外 size check；
- 如果某个当前 transitive pair 过去已经被人工判定为 false，则说明该 component 里必然还有错误边，继续检查；
- 最后对所有仍有 negative transitive predictions 的 component 进行更完整的检查，直到图接近 transitively consistent。

这个阶段本质是：

```text
模型产生匹配图
  -> 图结构发现内部矛盾
  -> 将矛盾定位到少量边
  -> 删除高风险边
  -> 重新计算 component
```

### 3.3 阶段三：Edge Recovery

前两阶段为了 precision 会主动切边，必然可能误删少量 true positive。

所以论文最后尝试把被删边加回来，但加回有条件：

1. 假设把某条 removed edge 加回；
2. 计算它会新增哪些 transitive matches；
3. 如果新增的 transitive matches **全部被 matcher 判为 MATCH**，则恢复该边；
4. 如果会产生新的 negative transitive prediction，则需要人工检查；
5. 对会形成超大 component 的恢复动作，不做高成本全量推理，优先人工或直接放弃恢复。

论文明确指出：transitive pair 数随 component 大小近似二次增长，因此实现中使用 component size threshold `S`；实验设置为 `S=50`，大 component 不做完整 O(n²) transitive evaluation。

---

## 4. 论文实验中最值得本项目注意的结果

论文在 Synthetic Companies、MusicBrainz、Camera、Monitor、WDC Products 等多源数据上，把 TransClean 接到 DistilBERT 或 CLER 后面。

几个关键结果：

### 4.1 高 pairwise F1 可以掩盖严重的 cluster contamination

在 Synthetic Companies 上，DistilBERT 的 pairwise F1 是 **81.54**，看起来并不差；但把它产生的 MATCH 边做传递闭包后，Pre-TransClean precision 会跌到 **0.02**。

原因不是模型每条边都很差，而是少量 false positive 把大量不同真实实体连在一起，产生巨量错误 transitive matches。

这是本项目必须警惕的指标陷阱：

```text
pairwise precision 很高
!=
最终 reference cluster 很干净
```

真正上线要监控的是 cluster-level contamination，而不是只看随机 pair 测试集。

### 4.2 对较好的 matcher，TransClean 可以大量删 FP、少量伤 TP

Synthetic Companies + CLER：

- pairwise precision：97.69
- Pre-TransClean precision（含 transitive matches）：87.48
- Post-TransClean precision：**98.54**
- false positives removed：**78.06%**
- true positives removed：1.88%

使用 LLM pseudo-labeling 时 Post-TransClean precision 甚至达到 **99.02**，但 recall 更低。

这说明 TransClean 最适合的定位不是“救一个很差的 matcher”，而是：

> **在已有高精度 matcher/规则之后，再做 false-positive containment。**

论文结论也明确：TransClean 依赖一个本身有效的 pairwise matcher；上游太差时，transitive signal 自身也会变得不可靠。

### 4.3 人工标签不能完全被 LLM 替代

论文允许人工 labeling 或 LLM pseudo-labeling。实验中 LLM 版本经常会：

- 删除更多 true positives；
- 留下更多 false positives；
- 在困难数据集上退化明显。

所以本需求既然允许“几百对黄金标签”，不应该为了省这几百对而让 LLM 直接当最终裁判。

更合理的是：

```text
规则/模型找最值得标的冲突边
    -> 人工给黄金标签
        -> 用于阈值校准、hard negative、回归测试
```

LLM 可以做：

- 候选解释；
- 低风险预标注；
- reference 候选抽取；

但不应该拥有最终 auto-merge 权限。

---

## 5. 对当前腕表需求的关键改造：不要原样照搬 TransClean

论文是一般实体匹配，而本项目有一个极强的业务定义：

```text
同商品 <=> 同 reference number
```

这意味着我们不应该把一个神经网络 matcher 当作“真理中心”，再让 TransClean 修它；应该把架构改成：

```text
Reference Evidence Engine
        ↓
Hard Precision Gate
        ↓
Candidate / Accepted Edge Graph
        ↓
Transitive Consistency Auditor
        ↓
AUTO_MATCH / REVIEW / REJECT
```

**reference 硬证据优先级必须高于 learned matcher。**

TransClean 在这里最有价值的部分是：

1. 图级污染检测；
2. negative transitive conflict；
3. min-cut/shortest-path 定位污染边；
4. 用有限人工预算标最有价值的 hard cases；
5. 把 pairwise 指标提升为 cluster-level 安全审计。

而论文中“模型认为一致就恢复边”的逻辑，在本项目里必须更保守。

---

## 6. 可直接落地的目标架构

```text
┌────────────────────────────────────────────────────┐
│  Source Ingestion                                  │
│  雷小安 / 腕表之家 / 奢当家                        │
└──────────────────────────┬─────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────┐
│  Normalize & Evidence Extraction                   │
│  brand / title / structured reference / OCR / SKU │
└──────────────────────────┬─────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────┐
│  Reference Resolver                                │
│  canonical_brand + reference_candidates + role    │
└──────────────────────────┬─────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────┐
│  Hard Precision Gate                              │
│  exact ref agreement + no contradiction           │
│  => ACCEPTED / REJECTED / ABSTAIN                  │
└──────────────────────────┬─────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────┐
│  Match Graph                                       │
│  node = listing                                    │
│  edge = accepted/proposed same-reference evidence │
└──────────────────────────┬─────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────┐
│  TransClean-inspired Graph Auditor                 │
│  ref fingerprints / transitive checks / min-cut   │
└──────────────────────────┬─────────────────────────┘
                           ↓
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
   AUTO_MATCH            REVIEW             REJECT
```

---

## 7. 数据模型：不要只存一个 `reference` 字符串

建议每条 listing 保留完整证据链。

### 7.1 listing

```sql
listing(
    listing_id           bigint,
    source               varchar,   -- leixiaoan / xcar_watch / shedangjia
    source_item_id       varchar,
    title                text,
    brand_raw            varchar,
    brand_id             bigint,
    product_role         varchar,   -- watch / strap / box / card / accessory / unknown
    raw_payload_uri      text,
    created_at           timestamp,
    updated_at           timestamp
)
```

`product_role` 很重要。腕表标题中经常会出现：

```text
适配 Rolex 126610LN 表带
126610LN 原装盒
某型号保卡
```

如果只看到 reference 字符串就匹配，会把配件、盒证、表带错误并入腕表实体。

### 7.2 reference_evidence

```sql
reference_evidence(
    listing_id           bigint,
    evidence_id          bigint,
    raw_value            varchar,
    normalized_value     varchar,
    evidence_type        varchar,
    -- structured_field / title_regex / title_model / image_ocr / llm_extract
    confidence           float,
    source_location      text,
    extractor_version    varchar,
    is_negated           boolean,
    created_at           timestamp
)
```

### 7.3 reference_resolution

```sql
reference_resolution(
    listing_id           bigint,
    brand_id             bigint,
    canonical_ref        varchar,
    status               varchar,
    -- CERTAIN / AMBIGUOUS / CONFLICT / NONE
    strongest_evidence   varchar,
    resolution_version   varchar,
    updated_at           timestamp
)
```

### 7.4 match_edge

```sql
match_edge(
    left_listing_id      bigint,
    right_listing_id     bigint,
    edge_state           varchar,
    -- PROPOSED / ACCEPTED / REJECTED / REVIEW_REQUIRED
    decision_reason      varchar,
    evidence_snapshot    json,
    score                float,
    rule_version         varchar,
    created_at           timestamp,
    updated_at           timestamp
)
```

### 7.5 graph_component_audit

```sql
graph_component_audit(
    component_id             bigint,
    node_count               int,
    edge_count               int,
    trusted_ref_count        int,
    negative_transitive_cnt  bigint,
    positive_transitive_cnt  bigint,
    audit_status             varchar,
    -- CLEAN / SUSPECT / QUARANTINED
    audit_version            varchar,
    updated_at               timestamp
)
```

所有自动匹配都必须可追溯到：

```text
为什么匹配
用的哪个 reference
reference 从哪里抽出来
哪个版本的规则/模型
是否经过图审计
```

否则未来无法回滚错误簇。

---

## 8. Reference 规范化：这是主键工程，不是 NLP 小细节

同一个 reference 可能出现：

```text
126610LN
126610 LN
126610-LN
126610ln
Ref.126610LN
型号：126610LN
```

但不能简单“把所有非字母数字都删除”后就认为安全，因为不同品牌的 reference 语义规则不同。

建议：

### 8.1 通用层

```text
Unicode NFKC
trim
uppercase
统一全角/半角
统一常见 dash/space
删除明确的 ref 前缀：REF / REFERENCE / 型号 等
```

### 8.2 品牌层

为 Rolex / Omega / Cartier / AP / Patek 等分别维护：

```text
brand_ref_normalizer(brand_id, raw_ref) -> canonical_ref | INVALID
```

品牌规则负责：

- 合法长度；
- 字母位置；
- 前后缀语义；
- 是否允许分隔符；
- 是否允许 leading zero；
- 常见 OCR 混淆字符；
- 已知 reference dictionary。

**绝不能全局把 O/0、I/1 等直接互换。** 这类 OCR 修复只能生成候选，不能直接自动 canonicalize。

---

## 9. Pair Verifier：三态而不是二态

为了符合“宁可漏、不许错”，pair verifier 应返回：

```text
MATCH
NO_MATCH
ABSTAIN
```

建议优先级：

### 9.1 Hard NO_MATCH

以下任一成立，直接 NO_MATCH：

```text
可信 brand 不同
两个 CERTAIN canonical_ref 不同
一边是 watch、一边明确是 strap/box/card/accessory
同一 listing 内存在无法解释的高可信 reference 冲突
```

### 9.2 Hard MATCH 候选

至少要求：

```text
brand_id 相同
canonical_ref 完全一致
双方 reference 都来自可接受证据级别
双方无高可信冲突 reference
双方 product_role = watch
```

建议进一步按证据等级划分：

```text
T1: structured_ref == structured_ref
T2: structured_ref == title_regex_ref
T3: title_regex_ref == OCR_ref 且证据互相独立
T4: model/LLM only
```

AUTO_MATCH 只放行 T1/T2，以及经过验证的部分 T3。

T4 永远 REVIEW，不直接合并。

### 9.3 ABSTAIN

例如：

- 只有一边有 reference；
- 两边都从低可信 LLM 抽到了相同值；
- 标题 ref 和图片 OCR ref 冲突；
- reference 可能是兼容型号而不是当前商品型号；
- 品牌无法确定。

这类数据进入 review / 后续抽取增强，而不是为了 recall 强行 MATCH。

---

## 10. TransClean 在本项目中的改造版：Reference Graph Guard

建议不要直接把模块叫 matcher，而是实现为一个独立的安全层：

```text
Reference Graph Guard
```

核心职责：

> 已经被局部规则/模型认为同 reference 的边，在形成全局 component 后，有没有产生语义矛盾？

### 10.1 第一层：O(n) reference fingerprint 检查

对于每个 component，先收集所有高可信 canonical reference：

```python
trusted_refs = set(
    node.canonical_ref
    for node in component
    if node.ref_status == "CERTAIN"
)
```

如果：

```python
len(trusted_refs) > 1
```

则不需要任何模型：

```text
component = QUARANTINED
```

因为业务定义已经说明同实体必须同 reference。

这比论文中的 learned transitive check 更强、更便宜。

### 10.2 第二层：anchor consistency，避免 O(n²)

一个 reference 可能在三个平台上有大量 listings，同 reference component 可能很大。

不能完整比较所有 pair。

选 component 中证据最强的 anchor：

```text
优先：structured_ref + known brand dictionary
其次：structured_ref
其次：title + OCR agreement
```

然后：

```python
for node in component:
    verify(anchor, node)
```

复杂度 O(n)。

如果某个 node 与 anchor 是 NO_MATCH，则构造一个 negative transitive signal。

### 10.3 第三层：source-aware representative checks

三个来源分别选 1–k 个代表节点：

```text
雷小安 anchor
腕表之家 anchor
奢当家 anchor
```

做跨来源互检：

```text
LXA <-> XCAR
LXA <-> SDJ
XCAR <-> SDJ
```

优先选：

- 结构化 reference 最强；
- 图片/OCR 最完整；
- 与 component 多数证据一致。

这一步能很便宜地发现：

```text
A-B 看起来匹配
B-C 看起来匹配
但 A-C reference 明确冲突
```

### 10.4 第四层：小 component 完整 transitive evaluation

如果：

```text
component_size <= S
```

例如 `S=50`，可以做完整未连接 pair 检查。

如果大于 `S`：

- 不做 n²；
- 只做 anchor + representative + conflict sampling；
- 若出现任何 hard conflict，直接 quarantine；
- 必要时把大 component 根据证据子簇先拆开。

### 10.5 第五层：min-cut / bridge 定位污染边

发现 component 有冲突后，不应该把整组都人工检查。

优先找：

```text
graph bridges
minimum edge cut
negative transitive pair 的 shortest path
```

如果一条边：

- 是 bridge；
- 删除后刚好把两个不同 canonical_ref 指纹分开；

那么这条边是最高优先级 false-positive 候选。

对于 precision-first 业务，可设置：

```text
hard evidence conflict + bridge edge
=> 自动切边
```

而：

```text
只有模型 conflict
=> REVIEW，不自动切
```

这样可以减少模型误判导致的过度拆分。

---

## 11. 与论文不同：Edge Recovery 必须更严格

论文允许：

> 被删边重新加入后，如果所有新 transitive pair 都被 matcher 判为 MATCH，则可以恢复。

本项目不建议这样直接做。

原因：业务要求“绝对不能误匹配”，而 learned matcher 即使 99.9% precision 也不是证明。

建议改成：

### 自动恢复条件

只有同时满足：

```text
1. 两端 brand 相同
2. 两端存在高可信 canonical_ref
3. canonical_ref 完全一致
4. component 中不存在第二个高可信 canonical_ref
5. product_role 无冲突
6. 加回后不产生任何 hard negative transitive conflict
```

才可以 auto recover。

否则：

```text
REVIEW / 保持拆分
```

在本需求里，recall 本来就可以牺牲，所以**错误切开比错误合并便宜得多**。

---

## 12. 增量架构：100 万–1000 万数据怎么跑

不建议把 Neo4j 等图数据库作为核心前提。这个问题大部分操作都是：

- reference exact join；
- component 计算；
- 小图 min-cut；
- 审计队列。

一个更简单、容易上线的栈：

```text
Raw data:       Object Storage / Parquet
Normalized:     ClickHouse 或 PostgreSQL
Reference index: (brand_id, canonical_ref) B-Tree / ClickHouse sort key
Batch compute:  Spark SQL / DataFrame
Graph CC:       Spark GraphFrames 或自研 union-find
Small graph:    Python + NetworkX
Review queue:   PostgreSQL
API:            FastAPI / Go
```

### 12.1 为什么不需要先上向量库

业务主键是 reference。

向量检索可以辅助：

- reference 缺失时找候选；
- 人工 review 排序；
- OCR/标题疑难样本。

但不能成为自动合并依据。

核心索引应该是：

```sql
(brand_id, canonical_ref)
```

### 12.2 增量流程

每次新抓一批 listings：

```text
1. normalize
2. extract reference evidence
3. resolve canonical_ref
4. 用 (brand_id, canonical_ref) 查已有候选
5. 通过 Hard Precision Gate
6. 写入/拒绝 match edge
7. 更新受影响 component
8. 只重跑受影响 component 的 Graph Guard
9. 有冲突 -> quarantine + review queue
```

不需要每次对 1000 万历史数据全图重算。

### 12.3 component id 的维护

可以两层：

```text
实时：union-find / component mapping 增量更新
离线：每天或每小时 Spark 全量/分区校验
```

但注意：删除边会使 union-find 不容易在线拆分。

因此建议：

- 实时只做“新增边后的 suspect 标记”；
- 真正涉及拆边的 component 单独重算 connected components；
- 周期性做全量一致性校验。

这是比追求一个完全动态连通图算法更务实的工程方案。

---

## 13. Review Queue：几百条人工标签应该怎么花

不要随机标。

TransClean 最重要的工程价值之一，就是把人工预算用到“最可能是一条错边就污染很多节点”的位置。

建议 review priority：

```text
priority =
    hard_ref_conflict_weight
  + negative_transitive_count_impacted
  + bridge_or_mincut_weight
  + component_size_weight
  + source_pair_novelty
  + brand_novelty
```

人工先看：

1. 把两个不同高可信 reference 子簇连接起来的 bridge；
2. 同时位于多个 negative-transitive shortest paths 的边；
3. 大 component 的最小割边；
4. 新品牌、新来源格式、新 OCR 模式；
5. 同系列相邻 reference hard negative。

几百条黄金标签建议至少覆盖：

```text
source pair × brand × evidence regime × positive/negative
```

并刻意加入 hard negatives：

```text
126610LN vs 126710BLNR
同系列不同尺寸
同系列不同材质
腕表 vs 适配表带
腕表 vs 原装盒证
标题写 A，图片 OCR 出 B
平台 SKU 被误抽成 reference
```

---

## 14. 关键伪代码

### 14.1 Reference Gate

```python
def decide_pair(a, b):
    if a.brand_id and b.brand_id and a.brand_id != b.brand_id:
        return "NO_MATCH", "brand_conflict"

    if a.product_role != "watch" or b.product_role != "watch":
        return "NO_MATCH", "non_watch_role"

    if a.ref_status == "CERTAIN" and b.ref_status == "CERTAIN":
        if a.canonical_ref != b.canonical_ref:
            return "NO_MATCH", "trusted_ref_conflict"

        if no_internal_conflict(a) and no_internal_conflict(b):
            if evidence_tier(a, b) in {"T1", "T2"}:
                return "MATCH", "trusted_exact_reference"

    return "ABSTAIN", "insufficient_proof"
```

### 14.2 Component Audit

```python
def audit_component(component):
    trusted_refs = {
        n.canonical_ref
        for n in component.nodes
        if n.ref_status == "CERTAIN" and n.canonical_ref
    }

    # 最强 hard invariant
    if len(trusted_refs) > 1:
        component.status = "QUARANTINED"
        return find_conflicting_cut_edges(component)

    anchor = choose_strongest_anchor(component)

    negative_pairs = []
    for node in component.nodes:
        if node.id == anchor.id:
            continue
        decision, reason = decide_pair(anchor, node)
        if decision == "NO_MATCH":
            negative_pairs.append((anchor.id, node.id, reason))

    for a, b in source_representative_pairs(component):
        decision, reason = decide_pair(a, b)
        if decision == "NO_MATCH":
            negative_pairs.append((a.id, b.id, reason))

    if len(component.nodes) <= 50:
        for a, b in all_missing_pairs(component):
            decision, reason = verifier(a, b)
            if decision == "NO_MATCH":
                negative_pairs.append((a.id, b.id, reason))

    if negative_pairs:
        component.status = "SUSPECT"
        return rank_edges_on_negative_paths(component, negative_pairs)

    component.status = "CLEAN"
    return []
```

### 14.3 安全切边

```python
def can_auto_cut(edge, component):
    # 只有“可证明”的冲突才自动切
    left_refs = trusted_refs(component.side_after_cut(edge, "left"))
    right_refs = trusted_refs(component.side_after_cut(edge, "right"))

    return (
        edge.is_bridge
        and len(left_refs) == 1
        and len(right_refs) == 1
        and left_refs != right_refs
    )
```

这体现 precision-first 原则：

> 模型怀疑 -> review；reference 硬冲突 -> 可以自动阻断污染。

---

## 15. API/任务拆分建议

可以拆成 5 个独立服务/任务。

### 15.1 `reference-extractor`

输入：raw listing + image URLs  
输出：`reference_evidence[]`

职责：

- structured field；
- title regex / brand grammar；
- OCR；
- 低可信模型/LLM extraction。

### 15.2 `reference-resolver`

输入：evidence  
输出：canonical reference + status

```json
{
  "brand_id": 1,
  "canonical_ref": "126610LN",
  "status": "CERTAIN",
  "evidence": [
    {"type": "structured_field", "value": "126610LN", "confidence": 1.0},
    {"type": "title_regex", "value": "126610LN", "confidence": 0.99}
  ]
}
```

### 15.3 `match-gate`

只负责：

```text
MATCH / NO_MATCH / ABSTAIN
```

不负责图逻辑。

### 15.4 `reference-graph-guard`

负责：

- connected components；
- trusted ref fingerprint；
- transitive checks；
- bridge/min-cut/path；
- quarantine；
- edge recovery policy。

### 15.5 `review-service`

展示：

- 两边商品；
- reference 证据来源；
- 图片；
- 为什么进 review；
- 该边影响多少 component 节点；
- cut 后会拆成什么 reference 子簇。

人工不应该只看到“相似度 0.998”，而要看到可解释证据。

---

## 16. 监控指标：不要只看 F1

### 16.1 Primary

```text
Auto-Match Precision
False Positive count in audited auto-match sample
Cluster contamination rate
```

其中：

```text
cluster contamination =
component 内存在 >=2 个高可信 canonical_ref
```

目标应接近 0。

### 16.2 TransClean-inspired

```text
negative_transitive_component_count
negative_transitive_pair_count
negative / positive transitive ratio
suspect component size distribution
bridge conflict count
quarantined edge count
```

如果上线后：

```text
negative transitive suddenly ↑
```

往往说明：

- 某来源字段格式变了；
- extractor 版本有 bug；
- 新品牌 reference grammar 未覆盖；
- OCR 模型漂移；
- 平台 SKU 被误识别成 reference。

因此这不只是清洗指标，也可以作为**数据漂移报警**。

### 16.3 Secondary

```text
coverage / recall
abstain rate
review rate
review precision
reference extraction coverage
```

当前业务必须接受：

```text
abstain 很高并不等于系统失败
```

只要 auto-match 集合足够干净，就是符合目标函数的。

---

## 17. 测试用例必须覆盖的 graph failure modes

### Case 1：格式差异

```text
A: Rolex 126610LN
B: ROLEX 126610-LN
```

结果：MATCH。

### Case 2：同系列相邻型号

```text
A: 126610LN
B: 126710BLNR
```

结果：NO_MATCH，即使图片视觉非常相似。

### Case 3：配件污染

```text
A: Rolex 126610LN 腕表
B: 适配 Rolex 126610LN 黑色表带
```

结果：NO_MATCH。

### Case 4：传递污染

```text
A(ref=X) -- edge -- B(ref=X)
B        -- wrong edge -- C(ref=Y)
```

Graph Guard 必须发现：

```text
trusted_refs = {X, Y}
```

并定位 B-C 或对应 bridge/min-cut 边。

### Case 5：标题和 OCR 冲突

```text
title: 126610LN
image OCR: 126710BLNR
```

结果：ABSTAIN / QUARANTINE，不允许自动选一个。

### Case 6：平台 SKU 冒充 reference

```text
platform_sku = 126610LN-like string
真实 brand reference = other value
```

编号角色分类失败必须能被 evidence conflict 捕获。

### Case 7：三源链式矛盾

```text
雷小安 A <-> 腕表之家 B = MATCH
腕表之家 B <-> 奢当家 C = MATCH
雷小安 A <-> 奢当家 C = hard NO_MATCH
```

结果：component SUSPECT，沿 A-C shortest path 找候选错边。

---

## 18. 上线顺序

### Phase 0：只做 deterministic reference core

先不要上复杂 matcher。

完成：

- brand canonicalization；
- reference evidence schema；
- brand-aware reference normalization；
- product role；
- T1/T2 exact-reference auto-match；
- conflict => abstain。

这一阶段就能拿到最安全的第一批自动匹配。

### Phase 1：Graph Guard shadow mode

不自动改结果，只计算：

```text
component contamination
negative transitive signals
bridge/min-cut suspects
```

观察 1–2 个数据周期，验证它是否能找出真实错误。

### Phase 2：自动 quarantine，不自动 aggressive recover

允许：

```text
hard ref conflict -> component quarantine
hard bridge conflict -> auto cut
```

仍禁止“模型认为一致所以加回”。

### Phase 3：把几百条黄金标签投入 hard cases

用 Graph Guard 的 review priority 选数据，而不是随机抽样。

训练/校准：

- reference extractor；
- pair verifier；
- evidence tier threshold。

### Phase 4：模型只扩大 ABSTAIN 区域的处理能力

模型负责：

```text
ABSTAIN -> 更好的候选 / 更好的 review 排序
```

而不是：

```text
NO_MATCH conflict -> 被模型翻成 MATCH
```

---

## 19. 一个需要特别指出的论文实现细节

arXiv HTML 中 `Algorithm 2` 的伪代码条件与紧邻正文叙述存在表面不一致：正文多次说明应优先拆分“negative transitive predictions 多于 positive transitive predictions”的 component，而 HTML 展示的某一行 `if` 条件看起来写成了 `|Pos_tr| > |Neg_tr|` 后执行 prune。

因此工程上不应该机械复制伪代码这一行，而应该：

1. 以论文方法描述的目标——消除 transitive inconsistency——为准；
2. 在复现时加入单元测试，确认 prune 后 negative transitive count 应下降；
3. 本项目更进一步，直接用 `trusted_ref_count > 1` 作为更强的 deterministic conflict signal。

这也是为什么本方案建议“借鉴 TransClean 的图清洗思想”，而不是照搬论文代码。

---

## 20. 最终建议

对这个 Notion Spec，TransClean 最值得直接落地的不是某个 Transformer，也不是完整论文 pipeline，而是以下四个能力：

### 20.1 把“同 reference”当等价关系，显式建图

不要只保存 pairwise matches。

必须能回答：

```text
一条边最终把哪些 listing 归成了同一个 reference component？
```

### 20.2 引入 graph-level safety invariant

最重要的 invariant：

```text
一个 CLEAN component 里，不能同时出现两个不同的高可信 canonical_ref。
```

这是比模型相似度更强的线上安全规则。

### 20.3 用 negative transitive + min-cut 把人工预算集中到污染边

几百条人工标签不随机花，而是优先标：

```text
能拆掉最大污染 component 的最少边
```

这正是 TransClean 对本项目最直接的价值。

### 20.4 precision-first 时，恢复边必须比删除边更保守

策略应是：

```text
错分开 -> 后续还能修
错合并 -> 会污染聚合、统计、推荐、价格分析和后续训练数据
```

因此：

```text
AUTO_MATCH = reference 硬证据 + 无冲突 + graph audit 通过
```

而不是：

```text
AUTO_MATCH = 模型分数高
```

---

## 21. 可以直接实现的 MVP Definition of Done

第一版不需要复杂深度学习，做到下面这些就已经是可上线的 precision-first MVP：

- [ ] 三来源统一 `listing` schema；
- [ ] brand canonicalization；
- [ ] structured/title/OCR reference evidence 全量留痕；
- [ ] brand-specific reference normalizer；
- [ ] `CERTAIN / AMBIGUOUS / CONFLICT / NONE` resolver；
- [ ] `MATCH / NO_MATCH / ABSTAIN` 三态 gate；
- [ ] 仅高可信 canonical ref exact match 可以自动建 ACCEPTED edge；
- [ ] connected component 计算；
- [ ] `trusted_ref_count > 1 => QUARANTINED`；
- [ ] 小 component 全量 transitive check，大 component anchor/sample check；
- [ ] bridge/min-cut/shortest-path 生成 review queue；
- [ ] hard reference conflict 可以自动切边；
- [ ] edge recovery 只允许 reference proof，不允许纯模型恢复；
- [ ] 监控 cluster contamination / negative transitive count；
- [ ] 每次规则/模型升级做历史回放和 component-level regression test。

做到这些后，再逐步引入 RAG、OCR、多模态模型、cross-encoder 或 LLM 来提升 `ABSTAIN` 区域的覆盖率，会比一开始直接训练一个“大一统商品匹配模型”安全得多，也更符合当前 Spec 的风险偏好。
