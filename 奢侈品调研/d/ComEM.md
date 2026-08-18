# ComEM：Match, Compare, or Select? An Investigation of Large Language Models for Entity Matching

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次分析前已检查 `奢侈品调研/d/` 中已有结果，排除了已经分析过的文章/项目。特别是调研表第一条实际对应的 **Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration** 已经存在于 d 目录，因此本次不重复分析。

本次选取尚未分析的项目/论文：

**ComEM — Match, Compare, or Select? An Investigation of Large Language Models for Entity Matching**

- 论文（ACL Anthology）：<https://aclanthology.org/2025.coling-main.8/>
- 论文 PDF：<https://aclanthology.org/2025.coling-main.8.pdf>
- 官方代码：<https://github.com/tshu-w/ComEM>
- 对应调研项：`奢侈品文章调研.md` 中的 ComEM 条目
- 对应需求：<https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31>

这篇工作的核心价值不是“再提供一个 LLM 二分类器”，而是指出传统 Entity Matching 的根本问题之一：

> **逐对独立判断 `(anchor, candidate)` 会忽略同一候选集内部的相互关系；把相似候选放在一起比较/选择，往往更容易识别真正匹配项。**

论文比较了三种 LLM Entity Matching 策略：

```text
Matching   : anchor + candidate_i            -> Yes / No
Comparing  : anchor + candidate_i + candidate_j -> A / B
Selecting  : anchor + [candidate_1 ... candidate_n] -> candidate_k / NONE
```

并提出 ComEM：

```text
Blocking 得到 n 个候选
        │
        ▼
中型 LLM 做 local ranking / filtering
(Matching 或 Comparing)
        │
        ▼
只保留 Top-K
        │
        ▼
更强 LLM 做 global Selecting
        │
        ▼
Match / None-of-the-above
```

论文默认的最佳实践是：先用较便宜的 **Matching** ranker，把候选压缩到 Top-4，再用更强的 selector 选择；官方 `compound.py` 的默认实现也是 `flan-t5-xl` 排序、`gpt-4o-mini` 选择，并在主程序中使用 `topK=4`。

但对于当前腕表需求，**不能直接把 ComEM 原样用作最终同款判定器**。原因有三点：

1. Spec 对“同一个商品”的定义不是语义相似，而是 **同一个 reference number / 型号**；
2. 业务要求 **precision 极端优先，宁可漏匹配也不能误匹配**；
3. ComEM 的 selecting 假设更接近 clean-clean ER：一个 anchor 在候选 records 中选择一个 match（或 none），而二奢数据中**同一 reference 在一个来源中可能有多条在售/历史记录**，因此 record-to-record 的“只选一个”并不等价于本业务的实体定义。

因此本次推荐的落地方式是把 ComEM 改造成：

> **RefComEM：Record → Canonical Reference Entity 的候选集级消歧器，而不是 Record → Record 的最终合并器。**

核心架构如下：

```text
雷小安 / 腕表之家 / 奢当家 raw record
                │
                ▼
      Reference Evidence Extraction
  structured field / title span / OCR span
                │
                ▼
       Brand-aware Conservative Normalize
                │
                ▼
        Canonical Reference Dictionary
                │
      ┌─────────┴─────────┐
      │                   │
唯一 exact 且无冲突      不完整/歧义/缺失
      │                   │
      ▼                   ▼
 VERIFIED          Candidate Retrieval Top-N
                          │
                          ▼
               Cheap Local Ranker
                 (ComEM Matching)
                          │
                          ▼
                       Top-4
                          │
                          ▼
              Global Selector + NONE
             + 多候选交互 + 顺序一致性
                          │
                          ▼
                Reference Hard Gate
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
      VERIFIED         ABSTAIN          REJECT
          │
          ▼
 record_reference_link(reference_entity_id)
          │
          ▼
按 (brand_id, canonical_reference) 形成跨源商品实体组
```

最终原则：

> **ComEM 只拥有“候选排序、消歧、拒识建议”的权限；真正自动合并必须由可审计的 canonical reference 证据硬门收口。**

这样既利用了论文最有价值的“候选集交互”，又不会让 LLM 的语义相似度越权覆盖 reference 规则。

---

## 1. 论文到底解决了什么问题

传统 Entity Resolution 通常是：

```text
Blocking
  -> 候选生成
  -> Pairwise Entity Matching
  -> Clustering / Linking
```

其中 Pairwise Matching 把每个候选对独立处理：

```text
(query, c1) -> 0.92
(query, c2) -> 0.89
(query, c3) -> 0.87
```

但它看不到：`c1/c2/c3` 其实可能是同系列、只差一个 reference 后缀的竞争候选。

腕表场景尤其典型：

```text
anchor:
Rolex Submariner Date 126610LN

candidates:
1. Rolex Submariner Date 126610LN
2. Rolex Submariner Date 126610LV
3. Rolex Submariner Date 116610LN
4. Rolex Submariner 124060
```

如果逐对判断，四个 pair 都有大量共享 token：品牌、系列、尺寸、材质、外观甚至图片都可能非常相似。

如果放在同一个 context 中，模型反而更容易观察到：

```text
126610LN != 126610LV
126610LN != 116610LN
126610LN != 124060
```

论文把这种能力称为通过 record interaction 改善 global consistency。

论文在 8 个 ER 数据集、10 个 LLM 上实验，发现 incorporating record interactions 的 comparing/selecting 通常明显优于独立 matching；其中 selecting 往往最具成本效率。论文同时发现 selecting 存在明显 **position bias**，候选太多或 true match 位置靠后时表现会下降，因此才进一步提出 ComEM 的“先过滤、再选择”复合结构。

这与腕表的需求非常同构：真正危险的不是随机负例，而是**高度相似的 hard negatives**。

---

## 2. 三种策略的技术实现

### 2.1 Matching：独立点式判断

官方 `src/matching_sq.py` 的 prompt 很简单：

```text
Do the two entity records refer to the same real-world entity?
Answer "Yes" if they do and "No" if they do not.

Record 1: {anchor}
Record 2: {candidate}
```

如果模型支持 label log probability，代码会分别计算 `Yes` / `No` 的 log prob，并构造带正负号的分数，然后给所有 candidate 排序：

```python
scores = matcher.score(instance, use_prob=True)
indexes = sorted(candidate_indexes, key=score, reverse=True)
```

复杂度大致是：

```text
n candidates -> O(n) LLM matching
```

优点：

- 简单；
- 适合做 cheap local ranker；
- 对候选位置不敏感；
- 比 comparing 更便宜。

缺点：

- 每个 pair 独立判断；
- 对同系列/邻近 reference 的相对差异利用不足；
- chat LLM 容易给多个相似 candidate 同时产生 false positive。

论文最后的 Finding 6 也指出：**用于 ranking / filtering 时，matching 比 comparing 更合适**。这也是我们落地时优先使用 cheap Matching ranker，而不是直接复制 comparing 的原因。

---

### 2.2 Comparing：让两个候选直接竞争

官方 `src/comparing_sq.py` 使用：

```text
Given entity record: anchor

Record A: candidate_i
Record B: candidate_j

Which one is more likely to refer to the same entity?
```

输出只能是：

```text
Record A
Record B
```

一个很值得借鉴的实现细节是：**代码会把 A/B 顺序交换，再问一次。**

也就是：

```text
(anchor, A, B)
(anchor, B, A)
```

然后把两次结果合并成比较分数，用来缓解 prompt order bias。

如果做全 pair 比较，复杂度接近：

```text
O(n^2)
```

论文/代码使用 bubble-style top-K ranking，把复杂度降低到：

```text
O(k * n)
```

这个策略对“126610LN 和 126610LV 到底哪个更像 anchor”非常直观，但在千万级生产系统里仍然不适合作为主链路，因为每个比较都要额外调用模型。

因此我们建议只把 comparing 用于：

- 离线误差诊断；
- Top-2 极难候选的二次 tie-break；
- 发现 selector 对位置敏感时的审计。

不建议让它承担全量 ranking。

---

### 2.3 Selecting：一次把整个候选集给模型

官方 `src/selecting.py` 的核心 prompt 是：

```text
Select a record from the following candidates that refers to the same
real-world entity as the given record.
Answer with the corresponding record number surrounded by "[]"
or "[0]" if there is none.

Given entity record:
{anchor}

Candidate records:
[1] ...
[2] ...
[3] ...
...
```

模型最终只允许输出一个编号：

```text
[3]
```

或者：

```text
[0]
```

`[0]` 是非常关键的 **none-of-the-above / abstain** 机制。

官方实现还做了几个适合生产化的限制：

```text
temperature = 0
seed = 42
max_tokens = 3
```

也就是说 selector 不是让模型写一段长解释，而是强制它完成一个受限选择任务。

Selecting 的优势：

1. anchor 只输入一次；
2. 一次看到多个相似 candidate；
3. 可以利用 candidate 间差异；
4. 可以显式 NONE；
5. 调用次数近似 O(1)。

但它有两个重要风险：

- **position bias**：正确 candidate 在不同位置时准确率显著变化；
- **长 context**：blocking 返回几十甚至几百候选时，selector 不适合直接全部读取。

这正是 ComEM 要解决的问题。

---

## 3. ComEM 的 Compound 架构

官方 `src/compound.py` 非常直接：

```python
if ranking_strategy == "matching":
    indexes = ranker.pointwise_rank(instance)
elif ranking_strategy == "comparing":
    indexes = ranker.pairwise_rank(instance, topK=topK)

indexes_k = indexes[:topK]
instance_k = {
    "anchor": instance["anchor"],
    "candidates": [instance["candidates"][idx] for idx in indexes_k],
}
preds_k = selector(instance_k)
```

也就是说：

```text
candidate set R
    │
    ▼
local ranker
    │
    ▼
Top-K
    │
    ▼
global selector
```

论文实验中使用：

```text
Ranker   = Flan-T5-XL
Strategy = Matching
Top-K    = 4
Selector = stronger LLM
```

为什么 Top-4 很合理：

- 候选过多，selector 出现位置偏差和长上下文问题；
- 候选太少，ranker miss 会导致 true match 被提前截掉；
- 论文 ablation 也专门分析了 `k=1..5`。

这里最适合迁移到腕表的不是模型名，而是**任务拆解方式**：

```text
便宜模型解决“粗排”
昂贵模型只解决“小集合消歧”
最终业务规则解决“能不能自动落库”
```

这比“每个候选对都调用一个最强 LLM”更符合 100 万～1000 万条记录的成本约束。

---

## 4. Blocking 的真实代码与生产差异

项目 README 要求 clone pyJedAI 的数据集，但官方 `src/blocking.py` 实际使用 `retriv.SparseRetriever` 构建右侧语料索引：

```python
retriever = SparseRetriever(index_name=f"{dataset}-index")
retriever.index(generate_docs(right_df))
candidates = retriever.bsearch(queries, cutoff=topK)
```

实验先取 Top-20，再构造 Top-10 测试候选。

此外，代码会从每个数据集抽取：

```text
300 个有 true match 的 anchor
100 个 no-match anchor
```

这 100 个 no-match anchor 对本需求很重要，因为真实三源数据里大量商品可能只存在于一个来源，系统必须支持：

```text
NONE / ABSTAIN
```

而不是“只要 blocker 召回了候选，就必须选一个”。

不过生产上不建议继续用“把整条商品所有字段拼成一个稀疏检索文本”作为第一 blocking 层。腕表有更强的结构知识：

```text
brand
reference token
series
model name
OCR reference
```

因此应先做 reference-aware blocking，再用文本/图片作为 fallback。

---

## 5. 论文结果里最值得迁移的几个结论

### 5.1 Record interaction 确实有价值

论文 Table 4 中，以 GPT-4o Mini 为例，平均 F1：

```text
Matching   67.80
Comparing  84.36
Selecting  82.26
ComEM      86.42
```

这些数值不能直接外推到腕表，但说明一个稳定方向：

> **把 hard negatives 放在同一个决策上下文里，比独立二分类更容易消歧。**

腕表相邻 reference 正是最典型的 hard negatives。

### 5.2 先过滤再选择，主要提升 precision

论文指出 ComEM 的 filtering → selecting pipeline 可以在保持 selecting 高 recall 的同时显著提高 precision。

这与 Spec “宁可漏、不能错”一致，但我们还需要在其后再加一层业务 hard gate，才能达到生产要求。

### 5.3 Selecting 便宜，但有位置偏差

论文发现：

- selecting 的调用成本低于重复 matching；
- 但候选位置会显著影响结果；
- ComEM 通过先 ranking 再 Top-K selecting 缓解。

对于 precision-first 系统，可以进一步做：

```text
Top-4 selector run #1: [A, B, C, D]
Top-4 selector run #2: [C, A, D, B]
```

只有两个不同排列仍然选择同一个 canonical reference，并且 reference hard gate 也通过，才允许进入自动匹配候选；否则直接 `ABSTAIN`。

因为只在疑难尾部做两次 selector，这个额外成本是可控的。

### 5.4 没有一个 LLM 在所有策略上都绝对最好

论文的另一个重要发现是：同一个模型在 matching / comparing / selecting 上能力差异很大。

因此生产系统不应该设计成：

```text
“选一个最强模型解决全部问题”
```

而应该设计成：

```text
规则 / parser          -> reference evidence
轻量模型               -> ranking
强模型                 -> difficult selection
确定性 hard gate        -> final decision
```

---

## 6. 为什么不能把原版 ComEM 直接用于本需求

### 6.1 Record-to-record 的“选一个”与 reference 聚类语义不一致

论文主要针对 clean-clean entity resolution，通常假设：

> 一个来源的一条记录，在另一个来源最多有一个对应记录。

但当前业务定义是：

> 同一个 reference 即“同一个商品”。

二奢平台上可能出现：

```text
雷小安：Rolex 126610LN listing A
雷小安：Rolex 126610LN listing B
腕表之家：Rolex 126610LN listing C
奢当家：Rolex 126610LN listing D
奢当家：Rolex 126610LN listing E
```

它们都是同一 reference group。

如果强行让 ComEM 在 records 里只选一个，会把业务上本来应该 many-to-one 聚合的数据错误建模成 one-to-one linkage。

### 6.2 正确改造：选择 Canonical Reference Entity

建立：

```text
reference_entity
----------------
ref_entity_id
brand_id
canonical_reference
series
model_name
aliases
status
```

每条商品记录最终只需要链接一个 canonical reference：

```text
record_1 -> ref_entity(ROLEX, 126610LN)
record_2 -> ref_entity(ROLEX, 126610LN)
record_3 -> ref_entity(ROLEX, 126610LN)
```

这样 selector 的 one-of-K 假设反而完全成立：

```text
一条商品记录原则上对应一个 reference entity 或 NONE
```

然后所有三源 records 通过相同 `ref_entity_id` 自然形成商品实体组。

这是本次分析最推荐的架构变化。

---

## 7. RefComEM：可直接落地的生产架构

### 7.1 总体数据流

```text
            ┌────────────────────┐
            │  三源 Raw Products │
            └─────────┬──────────┘
                      │
                      ▼
            ┌────────────────────┐
            │ Source Normalizer  │
            │ 品牌/字段/文本清洗 │
            └─────────┬──────────┘
                      │
                      ▼
        ┌────────────────────────────┐
        │ Reference Evidence Extract │
        │ field / title / OCR / card │
        └─────────────┬──────────────┘
                      │
                      ▼
        ┌────────────────────────────┐
        │ Conservative Canonicalize  │
        │ 品牌感知 reference 规范化 │
        └─────────────┬──────────────┘
                      │
           ┌──────────┴───────────┐
           │                      │
           ▼                      ▼
    Exact Unique Gate       Ambiguous / Missing
           │                      │
           │                      ▼
           │             Reference Candidate Index
           │                 Top-N candidates
           │                      │
           │                      ▼
           │              Local Matching Ranker
           │                      │
           │                      ▼
           │                    Top-4
           │                      │
           │                      ▼
           │                Global Selector
           │                + NONE option
           │                      │
           └──────────┬───────────┘
                      ▼
              Reference Hard Gate
         exact evidence / conflict veto
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   VERIFIED        ABSTAIN        REJECT
       │
       ▼
 record_reference_link
       │
       ▼
 canonical product/reference cluster
```

---

## 8. Reference Evidence：不要只存“最终抽出来的字符串”

为了可审计，建议把 reference 抽取结果设计成 evidence table，而不是直接在商品表塞一个 `normalized_ref`：

```sql
reference_evidence (
    evidence_id,
    record_id,
    source,
    brand_id,
    evidence_type,       -- structured_field/title/description/ocr/card/backcase
    raw_span,
    normalized_value,
    token_role,          -- reference/platform_sku/seller_sku/compatible_model/unknown
    extractor,
    extractor_version,
    confidence,
    image_id,
    char_start,
    char_end,
    created_at
)
```

为什么 `token_role` 很重要：

商品标题里可能同时出现：

```text
平台商品编号
卖家 SKU
腕表 reference
机芯型号
表带适配 reference
配件型号
```

一个“长得像型号”的字母数字串，不一定就是当前售卖腕表的 reference。

例如：

```text
原装表带 适配 Rolex 126610LN
```

这里 `126610LN` 是被适配对象，不是当前商品本身。

如果不保存 evidence span 和 role，后续模型即使匹配正确，也很难审计“为什么自动合并”。

---

## 9. Reference 规范化必须保守、品牌感知

不要简单做：

```python
re.sub(r'[^A-Z0-9]', '', ref)
```

因为不同品牌 reference 的点、斜线、后缀、前导零可能有语义。

建议分两层：

### L1：Lossless normalize

只做不会改变语义的操作：

```text
Unicode NFKC
trim
统一全半角
英文字母 uppercase
标准化明显的 Unicode dash/space
保存 raw value
```

### L2：Brand-specific canonicalize

按品牌规则处理：

```text
Rolex: 126610 LN -> 126610LN
Omega: 310 30 42 50 01 002 -> 310.30.42.50.01.002
Cartier / AP / Patek: 各自单独规则
```

任何可能产生碰撞的 normalize 规则都必须经过：

```text
collision audit
```

即检查两个不同真实 reference 是否被归到同一个 canonical value。

**宁可保留两个 aliases 等待人工确认，也不要激进归一造成误匹配。**

---

## 10. Candidate Retrieval：千万级不能做全量 record pair

1,000 万条数据做三源笛卡尔积不可行，而且完全没必要。

RefComEM 把问题从：

```text
record ↔ record
```

改成：

```text
record -> canonical reference dictionary
```

Reference dictionary 的规模通常远小于商品记录数。

推荐候选生成优先级：

### Route A：强 reference evidence

```text
(brand_id, normalized_reference) hash lookup
```

复杂度接近 O(1)。

如果：

```text
brand 唯一
reference 唯一
role=reference
没有冲突 evidence
```

则无需 LLM，直接进入 hard gate。

### Route B：reference 不完整

例如：

```text
126610
126610L?
310.30.42.50.01
```

只在同品牌 reference dictionary 中做：

```text
prefix / edit-distance / char-ngram / trigram retrieval
```

### Route C：无可靠 reference

在品牌/系列分区内用：

```text
BM25 + dense embedding + OCR token
```

召回 `Top-N` canonical reference entities。

图片视觉 embedding 可用于候选召回，但不得作为最终 positive gate。

推荐：

```text
N = 10~20
```

再交给 local ranker 压到 Top-4。

---

## 11. Local Ranker：借 ComEM 思路，但不要让它决定最终 Match

对每条待消歧 record，构造：

```json
{
  "anchor": {
    "brand": "Rolex",
    "title": "劳力士潜航者 ... 126610 LN",
    "reference_evidence": ["126610 LN"],
    "ocr_evidence": ["126610LN"]
  },
  "candidate_reference": {
    "canonical_reference": "126610LN",
    "series": "Submariner Date",
    "known_aliases": ["126610 LN"]
  }
}
```

Ranker 只输出用于排序的分数：

```text
candidate score
```

不直接产出数据库 match。

推荐特征优先级：

```text
1. exact reference evidence
2. reference edit distance / prefix consistency
3. brand exact
4. series/model textual consistency
5. OCR consistency
6. title embedding similarity
7. image similarity（弱）
```

可以用一个 3B~11B 级 instruction / seq2seq 模型，也可以先用更便宜的特征模型；关键是遵循论文的“便宜 ranker 处理大部分计算”思想。

---

## 12. Global Selector：把相邻 reference 一次放进上下文

对 Top-4 构造：

```text
Anchor Record
- brand
- title
- explicit reference spans
- OCR spans

Candidates
[1] ref=126610LN, series=Submariner Date
[2] ref=126610LV, series=Submariner Date
[3] ref=116610LN, series=Submariner Date
[4] ref=124060,   series=Submariner
[0] NONE
```

生产 prompt 不建议只问“哪个最像”，而应强调：

```text
只有当候选的 canonical reference 与当前商品自身 reference 证据一致时才选择。
不能因为系列、外观、品牌相似而选择。
如果证据不足、存在冲突、reference 只出现在“适配/兼容”语境，选择 NONE。
```

建议输出受限 JSON：

```json
{
  "selected_candidate": 1,
  "decision": "candidate_or_none",
  "anchor_reference_spans": ["126610 LN"],
  "normalized_anchor_reference": "126610LN",
  "conflicts": [],
  "evidence_sufficient": true
}
```

但要再次强调：

> **即使 selector 输出 candidate 1，也不代表最终 VERIFIED。**

它必须再经过 hard gate。

---

## 13. Precision-first 的 Reference Hard Gate

建议最终状态只有：

```text
VERIFIED
ABSTAIN
REJECT
REVIEW
```

### 13.1 可以自动 VERIFIED 的典型路径

#### Path A：结构化 reference 直接一致

```text
brand_A == brand_B
structured_ref -> canonical_ref
canonical_ref 唯一
无冲突
```

直接 VERIFIED，不需要 LLM。

#### Path B：标题 + OCR 独立证据一致

```text
title span -> 126610LN
image/card/backcase OCR -> 126610LN
brand -> Rolex
candidate canonical_ref -> 126610LN
无其它冲突 reference
```

在品牌规则验证通过后可 VERIFIED。

#### Path C：标题唯一 reference + 高置信 role classifier

只有经过足够 gold data 验证后才开放。

### 13.2 必须 ABSTAIN 的路径

```text
只有模型语义相似，没有 literal reference evidence
只有视觉相似
Top-1/Top-2 reference 只差 1 个字符且证据模糊
OCR 与 title reference 冲突
selector 两个候选顺序排列结果不一致
selector 选择候选但无法给出 anchor 中的 reference evidence span
只有截断 reference
```

### 13.3 必须 REJECT 的路径

```text
brand 冲突
两个高置信 explicit reference 不同
当前商品是配件/表带/盒证，而 reference 属于“适配对象”
平台 SKU 被误识别为 reference
```

一条非常重要的规则：

```python
if high_conf_ref_a and high_conf_ref_b and ref_a != ref_b:
    return REJECT
```

任何 LLM、图片相似度、embedding 都不得覆盖这个 veto。

---

## 14. 用 ComEM 的 position-bias 结论再加强拒识

论文明确证明 selector 对候选位置敏感。

对于本业务，不能只“接受这个风险”，可以直接利用业务允许漏匹配的特点，把它变成额外安全门：

```python
order1 = [c1, c2, c3, c4]
order2 = deterministic_permutation(order1, record_id)

s1 = select(order1)
s2 = select(order2)

if s1 != s2:
    return ABSTAIN
```

还可以加入：

```text
Top-4 vs Top-3 consistency
```

如果删除最弱候选后 selector 结果发生变化，也进入 REVIEW。

这不是为了提高 recall，而是主动牺牲 coverage 换 precision。

---

## 15. 图片怎么用：OCR 是证据，视觉相似只是辅助

Spec 明确有商品图片。

腕表图片最值得利用的并不是“两个表看起来很像”，因为同系列不同 reference 往往几乎一样。

图片证据按可信度建议分层：

```text
高价值：
保卡 / 吊牌 / 表背 / 证书上的 reference OCR

中价值：
表盘文字、材质/颜色/圈口等可验证属性

低价值：
纯视觉 embedding 相似度
```

建议增加：

```sql
image_evidence (
    image_id,
    record_id,
    evidence_type,
    ocr_text,
    reference_span,
    normalized_reference,
    ocr_confidence,
    image_quality,
    extractor_version
)
```

图片可以：

- 补充 reference；
- 发现 title/reference 冲突；
- 给人工审核提供证据。

图片不能：

- 在 reference 不一致时“投票翻盘”；
- 单凭外观相似自动判同 reference。

---

## 16. 少量黄金标签怎么花最值

Spec 允许标几百对 gold labels。

不建议只标：

```text
record A, record B, match=1/0
```

这样信息密度太低。

建议每个 gold item 同时标：

```json
{
  "record_id": "...",
  "brand": "Rolex",
  "true_reference": "126610LN",
  "reference_spans": [
    {"source": "title", "span": "126610 LN", "role": "reference"}
  ],
  "wrong_number_spans": [
    {"span": "SKU88219", "role": "seller_sku"}
  ],
  "candidate_refs": ["126610LN", "126610LV", "116610LN"],
  "decision": "VERIFIED"
}
```

一条标签就可以同时评估：

- reference extraction；
- token role classification；
- normalization；
- candidate recall；
- selector；
- final gate。

### gold set 要刻意包含 hard negatives

优先采样：

```text
同品牌同系列相邻 reference
仅后缀不同
前代/后代型号
O/0、I/1 OCR 混淆
标题没有 reference
平台 SKU 像 reference
“适配/兼容/for”配件标题
reference 只在图片
title 与 OCR 冲突
跨语言/中文别名
```

随机抽样不能充分覆盖真正危险的 false positive。

---

## 17. “绝对不能误匹配”在统计上意味着什么

只有几百条 gold pair 时，不能靠“测试集 0 个错误”就宣称模型达到了 99.99% precision。

用常见的 zero-failure rule-of-three 粗略理解：

```text
N 个独立 accepted case 中 0 error
95% 置信下，真实 error rate 上界约为 3/N
```

例如：

```text
N = 300    -> 上界约 1%
N = 3,000  -> 上界约 0.1%
N = 30,000 -> 上界约 0.01%
```

所以几百条人工标签足以：

- 找高风险错误模式；
- 校准规则；
- 比较方案；
- 建 PoC。

但不足以证明一个纯模型 matcher 可以达到“几乎零 FP”。

这也是为什么最终自动匹配必须尽量依赖：

```text
可验证的 reference exact evidence
```

而不是依赖一个“0.9999 probability”的神经网络分数。

---

## 18. 评测体系：不要只看 F1

本需求首要指标应是：

```text
1. Auto-Match Precision
2. False Positive Count
3. Accepted Coverage
4. Abstain Rate
5. Candidate Recall@K
6. Reference Extraction Exact Accuracy
7. Conflict Detection Recall
```

建议分层测：

### Stage A：Reference extraction

```text
Exact reference accuracy
Span accuracy
Role accuracy
```

### Stage B：Candidate generation

```text
Recall@4
Recall@10
Recall@20
```

candidate miss 不应该造成 false positive，而应该造成：

```text
ABSTAIN
```

### Stage C：ComEM-style selector

```text
Top-1 accuracy
NONE accuracy
permutation consistency
hard-negative accuracy
```

### Stage D：Final gate

```text
Precision among VERIFIED
FP count
Coverage
```

并按以下维度切片：

```text
source pair
brand
reference format
with/without structured reference
with/without OCR
same-series hard negatives
new references
```

上线验收时，**不能因为 recall/F1 更高而接受 FP 更多的方案**。

---

## 19. 千万级工程实现建议

### 19.1 存储

推荐：

```text
Object Storage + Parquet : 原始历史数据 / 图片元数据 / 中间批结果
PostgreSQL               : canonical reference、证据、最终链接、审计状态
OpenSearch / FAISS       : 模糊候选召回
Redis（可选）            : 热 reference lookup / 去重缓存
```

如果已有大数据平台：

```text
Spark -> 首次 1000 万级 backfill
```

如果实际规模更接近 100~300 万且字段简单，Polars/DuckDB 也能先快速 PoC，不必一开始就引入完整流式体系。

### 19.2 精确 reference path 不需要 ANN

最重要的主路径其实是：

```sql
SELECT ref_entity_id
FROM reference_entity
WHERE brand_id = ?
  AND canonical_reference = ?;
```

只要有 B-tree / hash 索引，千万商品量不是问题，因为 lookup 的对象是 reference dictionary，而不是全量 record pair。

### 19.3 LLM 只跑疑难尾部

假设：

```text
90% 有强 reference evidence -> rule path
9% 可通过轻量 parser/normalizer -> rule path
1% 真正 ambiguous -> RefComEM
```

即使 1000 万 records：

```text
1% = 10 万个 selector task
```

远比对数十亿 candidate pairs 调 LLM 可控。

实际比例要用三源 profiling 测，但架构应该从第一天就按照“LLM 只处理 tail”设计。

### 19.4 缓存与幂等

每个模型调用键：

```text
hash(
  normalized_anchor_payload,
  candidate_ref_ids,
  prompt_version,
  model_version
)
```

结果落库。

只要输入没变化，就不重复调用。

新增数据只处理：

```text
new/changed record
```

而不是重跑全库。

---

## 20. 推荐表结构

```sql
CREATE TABLE reference_entity (
    ref_entity_id BIGSERIAL PRIMARY KEY,
    brand_id BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    series TEXT,
    model_name TEXT,
    status TEXT NOT NULL,
    normalization_version TEXT NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);

CREATE TABLE record_reference_evidence (
    evidence_id BIGSERIAL PRIMARY KEY,
    record_id BIGINT NOT NULL,
    source TEXT NOT NULL,
    evidence_type TEXT NOT NULL,
    raw_span TEXT NOT NULL,
    normalized_value TEXT,
    token_role TEXT NOT NULL,
    confidence DOUBLE PRECISION,
    extractor_version TEXT NOT NULL,
    evidence_meta JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reference_candidate (
    record_id BIGINT NOT NULL,
    ref_entity_id BIGINT NOT NULL,
    retrieval_score DOUBLE PRECISION,
    rank_score DOUBLE PRECISION,
    selector_run_1 BOOLEAN,
    selector_run_2 BOOLEAN,
    conflict_flags JSONB,
    pipeline_version TEXT NOT NULL,
    PRIMARY KEY (record_id, ref_entity_id, pipeline_version)
);

CREATE TABLE record_reference_link (
    record_id BIGINT PRIMARY KEY,
    ref_entity_id BIGINT,
    decision TEXT NOT NULL,       -- VERIFIED/ABSTAIN/REJECT/REVIEW
    reason_code TEXT NOT NULL,
    evidence_ids JSONB,
    model_trace JSONB,
    rule_version TEXT NOT NULL,
    model_version TEXT,
    decided_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

最终跨来源聚合只需要：

```sql
SELECT ref_entity_id, array_agg(record_id)
FROM record_reference_link
WHERE decision = 'VERIFIED'
GROUP BY ref_entity_id;
```

不需要对三源 records 再做全量 pairwise clustering。

---

## 21. 可直接实现的决策伪代码

```python
def resolve_record(record):
    evidence = extract_reference_evidence(record)
    evidence = classify_number_roles(evidence)
    refs = conservative_normalize(evidence, brand=record.brand)

    # 1. 强冲突先拒绝
    high_conf_refs = unique_high_conf_current_product_refs(refs)
    if len(high_conf_refs) > 1:
        return Decision.REJECT("CONFLICTING_EXPLICIT_REFERENCES")

    # 2. 明确 reference 直接查 canonical dictionary
    if len(high_conf_refs) == 1:
        ref = high_conf_refs[0]
        candidate = exact_lookup(record.brand, ref)
        if candidate and no_conflict(record, candidate, evidence):
            return Decision.VERIFIED(
                candidate.id,
                reason="EXPLICIT_REFERENCE_EXACT"
            )

    # 3. 只对歧义数据做候选召回
    candidates = retrieve_reference_candidates(
        record=record,
        evidence=evidence,
        top_n=20,
        brand_scope=True,
    )

    if not candidates:
        return Decision.ABSTAIN("NO_CANDIDATE")

    # 4. ComEM local ranking
    ranked = local_ranker(record, candidates)
    top4 = ranked[:4]

    # 5. Selector 必须允许 NONE，并做顺序一致性测试
    s1 = selector(record, top4, allow_none=True)
    s2 = selector(record, deterministic_permute(top4), allow_none=True)

    if s1 is None or s2 is None:
        return Decision.ABSTAIN("SELECTOR_NONE")

    if s1.ref_entity_id != s2.ref_entity_id:
        return Decision.ABSTAIN("SELECTOR_POSITION_UNSTABLE")

    selected = s1

    # 6. 最终 hard gate：模型不能越权
    if explicit_ref_conflicts(selected, refs):
        return Decision.REJECT("REFERENCE_CONFLICT")

    if not has_verifiable_reference_evidence(selected, refs):
        return Decision.REVIEW("MODEL_ONLY_NO_HARD_REFERENCE")

    if image_ocr_conflicts(selected, evidence):
        return Decision.ABSTAIN("OCR_CONFLICT")

    return Decision.VERIFIED(
        selected.ref_entity_id,
        reason="COMEM_ASSISTED_REFERENCE_VERIFIED"
    )
```

这里最关键的一行是：

```python
if not has_verifiable_reference_evidence(...):
    return REVIEW
```

也就是说：

> **ComEM 选得再自信，如果没有 reference 可验证证据，也不能自动合并。**

---

## 22. 分阶段落地计划

### Phase 0：数据 profiling

先统计三源：

```text
品牌覆盖率
独立 reference 字段覆盖率
title 中 reference 覆盖率
图片/OCR 可读率
平台 SKU 格式
同 reference 多 record 比例
```

输出品牌级 reference 格式规则。

### Phase 1：只做 deterministic baseline

实现：

```text
brand normalize
reference extraction
reference role
canonicalize
exact join
conflict veto
```

这一版可能 recall 不高，但应作为 precision 上界基线。

### Phase 2：加入 RefComEM 疑难消歧

只处理：

```text
截断 reference
多个候选 reference
title 脏数据
字段缺失
```

先 Top-20 retrieval，再 cheap ranker，再 Top-4 selector + NONE。

### Phase 3：加入图片 OCR

优先：

```text
保卡
吊牌
表背
证书
```

将 OCR reference 作为新的 evidence source，而不是简单和 image embedding 做融合分类。

### Phase 4：人工闭环

REVIEW 队列按风险排序：

```text
selector 不稳定
同系列 1-char hard negative
OCR/title 冲突
未知品牌 reference 格式
新 reference
```

人工结果回流：

```text
normalization rule
role classifier
candidate ranker
prompt examples
```

---

## 23. 与原 ComEM 的差异总结

| 维度 | 原 ComEM | 推荐 RefComEM |
|---|---|---|
| Anchor | record | product record |
| Candidate | other records | canonical reference entities |
| 最终语义 | same real-world record/entity | same brand + same canonical reference |
| Blocking | sparse retrieval top-K | reference-aware exact/prefix/BM25/ANN |
| Ranker | LLM Matching / Comparing | cheap rule/model hybrid Matching ranker |
| Selector | one candidate / NONE | one canonical reference / NONE |
| Position bias | 通过 Top-K 缓解 | Top-K + permutation consistency + abstain |
| 图片 | 原论文不处理 | OCR 作为 reference evidence，视觉仅辅助 |
| 最终决定 | selector 输出 | deterministic reference hard gate |
| 多条同 reference | 不适合 one-to-one record 选择 | 多 records -> 一个 ref_entity，自然支持 |
| Precision-first | 主要优化 F1/precision/recall | FP veto + abstain + 可审计 evidence |
| 规模 | benchmark candidate sets | 100万~1000万 records，LLM 仅跑 tail |

---

## 24. 最值得直接复用的代码思想

可以直接借鉴官方项目以下模块设计：

### 1. `src/matching_sq.py`

复用：

```text
pointwise rank
label probability ranking
```

改成 record → canonical reference 的 ranking。

### 2. `src/comparing_sq.py`

复用：

```text
A/B swap
order-bias check
bubble top-K
```

不必上主链路，但很适合 Top-2 疑难审计。

### 3. `src/selecting.py`

直接借鉴：

```text
编号化候选
[0] NONE
temperature=0
极短结构化输出
```

生产上再加 JSON evidence 与两次候选顺序一致性。

### 4. `src/compound.py`

保留最核心的：

```text
rank -> topK -> select
```

但在 select 后新增：

```text
reference_hard_gate()
```

形成：

```text
rank -> topK -> select -> verify/abstain
```

---

## 25. 风险与边界

### 风险 1：LLM 可能“理解”出并不存在的型号

解决：

```text
必须返回 literal evidence span
不得接受自由生成但原始 record 中不存在的 reference
```

### 风险 2：reference normalize 过度

解决：

```text
品牌规则版本化
collision test
保存 raw value
新增规则先 shadow run
```

### 风险 3：selector position bias

解决：

```text
Top-4
两种候选排列一致性
不一致直接 abstain
```

### 风险 4：candidate miss

解决：

```text
miss -> ABSTAIN
绝不“从错误候选里硬选一个”
```

### 风险 5：视觉相似导致同系列误合并

解决：

```text
视觉只做召回/反证
reference 冲突拥有最高 veto 权
```

### 风险 6：公开 benchmark 结果过于乐观

论文 Limitations 也指出 LLM 可能见过公开 Web 数据中的相似记录/数据集，并建议未来在新或非公开数据上评估。

因此绝不能用论文 F1 直接作为腕表生产精度预期，必须用三源私有 hard-negative gold set 验证。

---

## 26. 最终推荐

如果现在就开始实现，我建议优先级是：

```text
P0  Canonical Reference Entity 数据模型
P0  三源 reference evidence extraction
P0  品牌感知保守 normalize + collision audit
P0  exact reference hard gate
P1  reference candidate index
P1  ComEM-style cheap ranking -> Top-4
P1  selector + NONE + candidate permutation consistency
P1  VERIFIED / ABSTAIN / REJECT / REVIEW 审计表
P2  图片 OCR reference evidence
P2  人工 hard-negative review loop
P3  对 ranker/selector 做领域微调或蒸馏
```

对当前 Spec 最重要的工程结论不是“采用哪一个 LLM”，而是：

> **把实体匹配的主键语义从 record similarity 收敛到 canonical reference entity；把 ComEM 放在 reference 歧义消解层；把 LLM 永远放在可拒识、可审计、不能越过硬 reference 规则的位置。**

这能同时满足：

- 100 万～1000 万规模：通过 reference dictionary 和 blocking 避免笛卡尔积；
- 三源持续增量：新 record 只做一次 evidence extraction + lookup；
- 字段稀疏：title/OCR 补证；
- 图片可用：重点转为 OCR evidence；
- 少量人工标签：集中标 hard negatives 与规则边界；
- precision-first：模型不直接拥有自动合并权；
- 可追溯：每个 link 都能回放 evidence / rule / model version。

因此，我认为 **ComEM 非常适合作为当前系统“疑难候选消歧 + 全局一致性审计”模块的架构参考，但不应该作为最终 record-to-record matcher 原样部署。**

---

## 27. 参考资料

1. Wang et al., **Match, Compare, or Select? An Investigation of Large Language Models for Entity Matching**, COLING 2025  
   <https://aclanthology.org/2025.coling-main.8/>
2. 官方 PDF  
   <https://aclanthology.org/2025.coling-main.8.pdf>
3. 官方代码：`tshu-w/ComEM`  
   <https://github.com/tshu-w/ComEM>
4. 官方实现关键文件：
   - `src/blocking.py`
   - `src/matching_sq.py`
   - `src/comparing_sq.py`
   - `src/selecting.py`
   - `src/compound.py`
