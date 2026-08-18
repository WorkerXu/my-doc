# Labeling Training Data for Entity Matching Using Large Language Models

> 分析人：c  
> 论文：Aaron Steiner, Christian Bizer, **Labeling Training Data for Entity Matching Using Large Language Models**, arXiv:2606.28823, 2026  
> 论文：https://arxiv.org/abs/2606.28823  
> 论文 HTML：https://arxiv.org/html/2606.28823  
> 官方实现：https://github.com/wbsg-uni-mannheim/Automatic-data-labeling  
> 调研清单：`奢侈品文章调研.md`  
> 目标需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务硬约束：**“同一个商品” = 同一 reference number / 型号**；100 万–1000 万级、持续增量；字段高度稀疏；图片可用；**precision 极端优先、绝不能误匹配，允许漏匹配**；可人工标注几百对黄金样本。

---

## 1. 选题与去重

执行前已检查 `奢侈品调研/c/` 的既有结果；其中已经有 AnyMatch、Ditto、DeepBlocker、TransClean、GraLMatch、Conformal Selective Prediction、属性抽取、多模态商品匹配等分析，但**没有**本篇 `Labeling Training Data for Entity Matching Using Large Language Models`，因此本次不存在重复分析。

调研清单对本文的推荐理由是：

> 用 LLM 作为教师自动标注实体匹配训练对，再训练更快的小模型；把几百条人工黄金标签主要用于验收与阈值校准，而让机器标注扩充 Blocking 和候选分类训练集，降低人工标注成本。

这个方向和当前需求有一个很重要的互补关系：

- 现有多篇方案解决的是“怎么匹配”；
- 本文重点解决的是“**在没有足够人工标签时，怎么低成本生产 matcher 的训练数据，并把大模型能力蒸馏到可大规模推理的小模型**”。

但必须先给出结论：

> **本文不能原样作为最终自动合并器。**它优化的是实体匹配模型的 F1/成本/吞吐，而当前腕表业务要求“reference 相同才算同一商品，且误匹配不可接受”。最合理的迁移方式是把论文变成一个 **离线标签工厂 + hard-case mining + 小模型疑难候选分流器**；最终自动 MATCH 仍由“高可信 reference 抽取 + brand-aware canonical reference 严格一致 + 冲突否决”硬规则收口。

这样既吃到论文“省标注、低成本、高吞吐”的优势，又不会让 LLM/学生模型越权制造 false positive。

---

## 2. 论文到底做了什么

### 2.1 核心问题

传统实体匹配存在两类成本：

1. 大 LLM 可以 zero-shot 判 pair，但百万级候选直接调用太慢、太贵；
2. Ditto / RoBERTa / XGBoost 等小模型推理快，但需要目标任务训练标签。

论文把两者串起来：

```text
无标签候选 pair
      ↓
有策略地选择“值得标”的 pair
      ↓
LLM Teacher 自动打标签
      ↓
标签后处理 / 清洗
      ↓
训练小 Student Matcher
      ↓
大规模低成本推理
```

这实际上是一个 entity matching 领域的 **knowledge distillation / machine labeling pipeline**。

### 2.2 论文的四个实验维度

作者系统比较了四个维度：

1. **Pair selection**
   - similarity search
   - feature-based active learning
   - Ditto-based active learning

2. **Teacher**
   - GPT-5.2
   - Qwen 3.6 Plus
   - Kimi K2.6

3. **Post-processing**
   - relabel
   - relabel-drop
   - closure-drop
   - closure + relabel 的组合

4. **Student**
   - Ditto / RoBERTa
   - XGBoost
   - Qwen3 fine-tuning

论文在 Abt-Buy、Walmart-Amazon、WDC Products、DBLP-ACM、DBLP-Scholar 上评测。

### 2.3 最关键的实验结论

论文最有工程价值的不是“哪个模型赢”，而是下面几点：

- 用 LLM 机器标注生成的训练集训练 student，和用 benchmark 官方人工训练集训练的 student，最终差异大多控制在 **2 F1 点以内**；
- GPT-5.2 为 5 个 benchmark 构造训练集的总标注成本约 **28.31–40.88 美元**，而作者估算同量人工标注约需要 **470 小时**；
- Ditto 在推理阶段比直接使用 LLM 做 pair matching 快约 **41.5–534 倍**；
- active learning 尤其 Ditto-based disagreement sampling 通常比单纯 similarity selection 更有效；
- 但是 teacher 仍然会错，甚至论文复核发现 benchmark 自己的标签也存在约 **0.53%–2.52%** 的错误；
- graph closure / bridge filtering 并不是稳定增益，在一些数据集上会伤害结果。

对当前“绝不能误匹配”业务来说，最后两点尤其重要：

> 即便先进 LLM 或 benchmark ground truth 都不是零错误，因此**任何 probabilistic matcher 都不应该拥有最终 merge authority**。

---

## 3. 官方代码的真实实现架构

官方仓库并不是概念 demo，而是把训练集生成、prompt、后处理、训练和复现拆成了完整流水线。

```text
Automatic-data-labeling/
├── artifacts/
│   ├── training_data/                 # 机器标注训练集
│   ├── prompts/                       # teacher / relabel prompt
│   ├── USAGE.md
│   └── SCRIPTS_AND_CONFIGS.md
├── benchmarks/                        # benchmark 数据和预计算 embedding
├── configs/labeling/                  # 标注配置
├── scripts/
│   ├── labeling/
│   │   ├── similarity_search.py
│   │   ├── active_learning_ml.py
│   │   └── active_learning_ditto.py
│   ├── post_processing/
│   │   ├── relabel_three_phase_generated_labels_batch.py
│   │   ├── build_drop_changed_profiles.py
│   │   └── build_closure_bridge_profiles.py
│   └── training/
│       ├── train_xgboost.py
│       ├── train_ditto.py
│       └── train_qwen.py
├── error_anlysis/                     # teacher label 人工审计
└── third_party/ditto_modern/          # Ditto 训练/推理实现
```

### 3.1 Candidate pool：先把笛卡尔积砍掉

论文先对记录做 embedding，并不是让 LLM 在全量两两组合上工作。

候选构造的核心是：

- 对每条左表记录取一批 embedding nearest neighbors；
- 再混入少量更低排名随机 pair，避免训练集只有“相似 pair”；
- 论文实验中每个 query 大致形成 20 个候选，其中 18 个近邻 + 2 个较低排名随机候选；
- 这种候选池对 benchmark 原始 positive pair 的覆盖率达到约 99.56%–99.92%。

这与千万级腕表数据完全同构：

> 大模型只应该看“经过 blocking/ANN 后的小候选池”，绝不能参与 10^12 级笛卡尔比较。

### 3.2 三种 pair selection

#### A. Similarity Search

最简单：按 embedding 相似度挑高相关 pair 给 teacher 标。

优点是实现简单；缺点是容易浪费标签预算在大量“很像、但模型其实早已学会”的样本上。

#### B. Feature-based Active Learning

`active_learning_ml.py` 使用一个传统模型 committee，特征包括：

- attribute-level similarity；
- record embedding cosine；
- 多个轻量分类器的预测。

committee 包含 Logistic Regression、Random Forest、ExtraTrees、GradientBoosting 和 rule-based matcher 等。

每轮优先拿模型分歧较大的 pair 交给 teacher。

#### C. Ditto-based Active Learning

`active_learning_ditto.py` 是最值得迁移的部分。

代码会：

1. 把已经获得 teacher label 的 pair 序列化成 Ditto 训练数据；
2. 用 bootstrap 方式训练多个 Ditto member；
3. 每个 member 使用不同随机 seed / bagging 样本；
4. 对未标注 pool 进行并行打分；
5. 用 ensemble 的**概率方差、vote entropy、离 0.5 的距离、正负票分裂程度**排序；
6. 把最值得学习的 pair 再送给 teacher；
7. 重复直到达到 label budget。

官方实现里的 `DittoMember` 保存：

```text
name
checkpoint_dir
threshold
val_f1
train_rows
valid_rows
TrainConfig
max_field_len
```

bagging 时会做 stratified train/validation split，再按目标正样本比例 bootstrap 正负样本。每个模型产生 positive probability，最后形成 disagreement ranking。

这说明它不是简单的“LLM 自动标一批数据”，而是：

> **Student 主动告诉 Teacher：下一批最值得花钱标哪些样本。**

对于我们只有几百人工黄金 pair 的场景，这个机制非常有价值。

### 3.3 Teacher prompt：论文版很简单

官方 teacher prompt 本质只有：

```text
You are an expert entity matcher.
Decide if two records refer to the same real-world entity.
Return only valid JSON:
{"match": true|false}
```

两条记录作为 JSON 描述传入。

这对通用 benchmark 足够，但**不适合直接用于腕表**，因为“same real-world entity”在二奢项目里有特殊且极严格的定义：

> 同一商品 = 同一 brand 下的同一 reference number；外观相似、同系列、同尺寸、同机芯都不能替代 reference 一致。

所以我们必须重写 prompt，让 teacher 只围绕 reference 证据工作，且允许 `UNKNOWN/ABSTAIN`，而不是强迫二分类。

### 3.4 Precision-oriented Relabel

仓库还提供第二次保守审核 prompt：

```text
predict match=true only when evidence strongly supports same entity
and there is no meaningful contradiction
```

并明确把：

```text
conflicting model numbers / editions / capacities / sizes / colors / years / variants
```

当作 strong negative evidence。

这个思路和腕表非常匹配，尤其是 **conflicting model numbers**。

但论文也表明：二次 relabel / closure 并不在所有数据集上稳定提升。因此不能理解成“多问一次 LLM 就安全”。

### 3.5 Student：Ditto 是最合适的工程折中

论文比较 XGBoost、Ditto、Qwen3 后，Ditto 是更适合作为生产候选判别器的折中：

- 比大 LLM 小很多；
- GPU 批推吞吐高；
- 能直接消费属性序列；
- 实验中性能通常接近更大模型；
- 官方代码路径和训练脚本完整。

论文配置中 Ditto 基于 RoBERTa-base，典型训练设置包括：

```text
batch size: 64
max length: 256
learning rate: 5e-5
up to 50 epochs + early stopping
```

对 100 万 pair，论文给出的量级大约是：

```text
XGBoost: ~5.5 min
Ditto:   ~37.3 min
```

而直接 LLM / 小型生成模型通常进入小时甚至天级。

因此：

> **LLM 更适合离线 teacher，Ditto 更适合在线/批量 student。**

---

## 4. 为什么论文不能直接拿来“自动判同款”

当前需求和论文 benchmark 的目标函数不同。

论文基本优化：

```text
maximize average F1 / matching quality
```

业务真正优化：

```text
subject to precision ≈ 100%
maximize coverage / recall
```

并且业务定义具有一个罕见优势：

```text
same product ⇔ same canonical reference number under same brand
```

也就是说我们不是在解决一个完全模糊的“语义相同”问题，而是可以把最终判定降维成**标识符解析与验证问题**。

因此以下做法必须禁止：

```text
text similarity high       → MATCH      ✗
image similarity high      → MATCH      ✗
Ditto score = 0.999         → MATCH      ✗
LLM says same watch         → MATCH      ✗
transitive cluster connected→ MATCH      ✗
```

这些都只能产生**证据 / 候选 / 审核优先级**。

最终 MATCH 应该满足：

```text
canonical_brand_left == canonical_brand_right
AND
trusted_reference_left == trusted_reference_right
AND
no hard contradiction
```

如果 reference 不足以可信解析，就宁可 `ABSTAIN`。

---

## 5. 对当前需求的改造：Reference-First Distillation Architecture

推荐落地架构如下：

```text
              ┌──────────────────────────────────┐
              │ 雷小安 / 腕表之家 / 奢当家增量数据 │
              └────────────────┬─────────────────┘
                               ↓
                  [1] Raw Ingestion / Snapshot
                               ↓
                  [2] Canonical Normalization
              brand / title / fields / images / OCR
                               ↓
                  [3] Reference Evidence Extractor
           explicit field / title / desc / OCR / catalog
                               ↓
          ┌──────────── trusted ref available? ────────────┐
          │ YES                                            │ NO/ambiguous
          ↓                                                ↓
 [4A] Exact Reference Join                        [4B] Candidate Retrieval
 brand + canonical_ref                           brand/category block + ANN
          ↓                                                ↓
 [5] Hard Contradiction Gate                      [6] Student / rules triage
          ↓                                                ↓
  AUTO_MATCH / REJECT                            ABSTAIN / HUMAN / TEACHER
          │                                                │
          └───────────────── audit log ─────────────────────┘
                               ↓
                    [7] Offline Label Factory
           Human Gold + LLM Teacher + Active Learning
                               ↓
                    [8] Student Retraining
                               ↓
                      versioned deployment
```

核心原则：

> 学习模型负责把“不确定世界”变得更有序；硬标识规则负责真正合并。

---

## 6. 第一层：建立 Reference Evidence，而不是直接做 pair matching

### 6.1 统一商品 schema

三源先映射成统一结构：

```json
{
  "source": "leixiaoan|xbiao|shedangjia",
  "source_product_id": "...",
  "brand_raw": "劳力士",
  "brand_id": "rolex",
  "title": "劳力士潜航者 126610LN 黑水鬼...",
  "model_field_raw": "126610LN",
  "description": "...",
  "platform_sku": "LX123456",
  "images": ["..."],
  "ocr_text": ["..."],
  "reference_candidates": [],
  "normalized_at": "...",
  "normalizer_version": "refnorm-v3"
}
```

特别重要：**平台 SKU 必须与品牌 reference 分开**。

例如：

```text
LX-88291            → platform_sku
126610LN            → brand_reference
M126610LN-0001      → potentially official full product code / catalog id
```

否则“看起来像编号”的平台自有货号会成为最危险的 false-positive 来源之一。

### 6.2 Reference candidate 带 provenance

不要只保存一个字符串，应保存证据：

```json
{
  "raw": "126610 LN",
  "canonical": "126610LN",
  "brand_id": "rolex",
  "source_type": "explicit_field",
  "source_location": "model_no",
  "extractor": "brand-regex-v7",
  "confidence": 0.9999,
  "role": "brand_reference",
  "catalog_hit": true,
  "span": "126610 LN"
}
```

来源可靠度建议分级：

```text
L0  官方/平台明确 reference 字段 + catalog 命中
L1  标题中品牌规则精确抽取 + catalog 命中
L2  description / spec table 抽取 + catalog 命中
L3  图片 OCR 抽取 + 多图/文本交叉验证
L4  LLM 自由抽取但无 catalog 验证
```

自动 MATCH 最好只消费 L0/L1，以及极严格交叉验证后的 L2/L3；L4 永远不能独立触发 MATCH。

### 6.3 Canonicalization 必须 brand-aware

不能用一个粗暴的：

```python
re.sub(r'[^A-Za-z0-9]', '', ref).upper()
```

因为不同品牌的连字符、点号、slash、后缀可能有语义。

正确做法：

```python
def canonicalize_ref(brand_id: str, raw: str) -> RefNormalizationResult:
    s = nfkc(raw).upper().strip()
    rule = BRAND_REF_RULES[brand_id]
    tokens = rule.tokenize(s)
    parsed = rule.parse(tokens)
    return rule.canonicalize(parsed)
```

输出不仅是 canonical string，还要返回：

```text
canonical
parse_ok
removed_tokens
suffixes
catalog_id
normalizer_version
```

必须保留原值和 normalization trace，方便审计。

### 6.4 Catalog 是关键安全设施

建立：

```text
reference_catalog(
    brand_id,
    canonical_reference,
    aliases[],
    series,
    family,
    official_variants[],
    valid_patterns[],
    status,
    provenance,
    version
)
```

catalog 作用不是提升 recall，而是**减少 hallucinated / mis-extracted reference**。

例如标题出现：

```text
“适配 Rolex 126610LN 表带”
```

虽然能抽出 `126610LN`，但商品其实不是腕表。

所以 reference 证据必须结合 `item_role`：

```text
watch
strap
box
certificate
accessory
part
unknown
```

自动 MATCH 只允许 `watch ↔ watch`。

---

## 7. 第二层：把论文的 Candidate Pool 改造成“reference 疑难候选池”

### 7.1 有可信 reference 的记录根本不需要 ANN

如果两边都已有可信 reference：

```sql
JOIN normalized_product a
JOIN normalized_product b
  ON a.brand_id = b.brand_id
 AND a.canonical_ref = b.canonical_ref
 AND a.source <> b.source
```

这是最便宜、最可解释、也是 precision 最容易做到极高的主路径。

千万级数据真正难的是：

- reference 字段为空；
- reference 埋在标题；
- reference 写法不规范；
- OCR 才能看到；
- 标题同时出现多个型号；
- 配件标题提到“被适配腕表”的 reference；
- 相邻型号只差一位字符或后缀。

所以 ANN / student 的计算预算应该集中在这些记录。

### 7.2 Blocking 顺序

推荐：

```text
brand exact block
    ↓
item_role compatible block
    ↓
series/family optional block
    ↓
reference candidate prefix/pattern block
    ↓
text embedding / BM25 / ANN top-K
```

不要一开始全品牌 ANN，更不要跨品牌 ANN。

### 7.3 Hard Negative 生成比随机 negative 更重要

论文 active learning 会找不确定 pair；腕表业务应进一步显式加入“高危误匹配对”。

例如：

```text
126610LN vs 126610LV
116610LN vs 126610LN
5711/1A-010 vs 5711/1A-011
同系列、同尺寸、同颜色，但 reference 不同
腕表 vs 原装表带/盒证
标题含“适配 126610LN”的表带 vs 126610LN 腕表
图片极像但 reference 不同
同 reference family 但 suffix 不同
OCR 只错一个字符：0/O、1/I、5/S、8/B
```

这些 pair 对降低 false positive 的价值远高于普通随机 negative。

建议 hard-negative sampler：

```python
risk = (
    4.0 * same_brand
  + 3.0 * same_series
  + 4.0 * near_ref_edit_distance
  + 3.0 * high_image_similarity
  + 2.0 * high_title_similarity
  + 5.0 * conflicting_trusted_ref
  + 5.0 * accessory_mentions_target_ref
)
```

按 risk 排序进入人工/teacher 队列。

---

## 8. 第三层：把论文的 LLM Teacher 改成“Reference Evidence Teacher”

论文原 prompt 强迫输出二分类。对本项目应改成**可拒识的结构化裁决**。

建议 teacher 输入：

```json
{
  "left": {
    "brand": "ROLEX",
    "title": "...",
    "reference_candidates": [...],
    "item_role": "watch",
    "ocr": [...]
  },
  "right": {...},
  "brand_catalog_candidates": [
    "126610LN", "126610LV", "116610LN"
  ]
}
```

建议输出：

```json
{
  "decision": "MATCH | NON_MATCH | UNKNOWN",
  "left_reference": "126610LN | null",
  "right_reference": "126610LN | null",
  "left_reference_evidence": "explicit_field | title | ocr | none",
  "right_reference_evidence": "explicit_field | title | ocr | none",
  "hard_conflicts": [],
  "item_role_conflict": false
}
```

Teacher 规则必须写死：

```text
1. MATCH 仅当同 brand 且两侧都能用明确证据落到同一 canonical reference。
2. 相似外观、系列名、机芯、尺寸、昵称都不能替代 reference。
3. 任一侧存在可信且不同的 reference => NON_MATCH。
4. 只能从受限 catalog 中选择 reference，不允许自由发明型号。
5. 无法确定 => UNKNOWN，不强迫二分类。
6. 配件/表带/盒证提到腕表 reference 时，不得视为该腕表本体。
```

这会把 LLM 从“语义裁判”降级成“受约束证据解析器”。

### 8.1 Teacher label 如何进入训练集

不能把所有机器 label 等权写入 student training。

建议：

```text
TIER_A: hard-rule / human gold                  weight 1.0
TIER_B: teacher + catalog evidence + no conflict weight 0.7
TIER_C: teacher only, reference incomplete       weight 0.2 或仅用于 mining
UNKNOWN                                           不训练 MATCH，可进入人工队列
```

正样本尤其保守：

```text
机器正样本必须满足 reference evidence gate；
机器负样本可以相对宽松，因为错杀只伤 recall，不产生 false merge。
```

这正好利用业务“precision 优先”的非对称成本。

---

## 9. 第四层：Active Learning 不只找“不确定”，而要找“最可能制造 FP”的样本

论文的 Ditto disagreement ranking 很适合直接改造。

原始信号：

```text
score variance
vote entropy
mean margin to 0.5
split votes
```

我们增加业务风险项：

```text
trusted_ref_conflict
near_neighbor_reference
image_high_text_conflict
accessory_risk
ocr_confusion_risk
new_brand_or_source_distribution
normalizer_low_confidence
```

最终采样分数：

```python
sample_score = (
    0.25 * model_disagreement
  + 0.15 * uncertainty
  + 0.35 * false_positive_risk
  + 0.15 * drift_score
  + 0.10 * diversity_score
)
```

这里 `false_positive_risk` 权重最高。

### 9.1 几百个人工标签怎么花

不建议把“几百对”全部拿去训练。

建议拆成：

```text
40%  immutable gold test / precision calibration
30%  hard-negative stress set
20%  active-learning seed / correction
10%  drift / 新品牌 / 新来源预留
```

如果总共 500 对，可以大致：

```text
200 对：永不参与训练的 gold holdout
150 对：高危 hard negatives
100 对：active learning seed
 50 对：后续漂移/新来源
```

为什么要留 immutable gold？

因为论文已经证明：机器 teacher 自己会错；如果几百条人工标签全拿去训练，就没有真正独立的 precision 验收集。

### 9.2 Gold set 必须分层

不能随机抽样。

至少覆盖：

```text
source pair:
  雷小安 × 腕表之家
  雷小安 × 奢当家
  腕表之家 × 奢当家

risk bucket:
  exact explicit reference
  reference only in title
  OCR reference
  same-series different-ref
  accessory mentions watch ref
  OCR confusion
  sparse fields
  image-high-sim but ref mismatch

brand bucket:
  高频品牌
  reference 规则复杂品牌
  长尾品牌
```

随机 benchmark F1 很容易掩盖真正致命的 false positive。

---

## 10. 第五层：Student 只做疑难分流，不拥有 Merge Authority

推荐 student 首选 Ditto/RoBERTa 类 cross-encoder。

### 10.1 输入序列不要直接塞三平台原始字段

建议序列化成统一 evidence schema：

```text
[LEFT]
[BRAND] ROLEX
[ROLE] WATCH
[REF_EXPLICIT] 126610LN
[REF_TITLE] 126610LN
[SERIES] SUBMARINER
[TITLE] ...
[OCR] ...

[RIGHT]
...
```

并显式加入：

```text
[REF_CONFLICT] false
[ROLE_CONFLICT] false
```

这样学生更容易学到“reference 冲突是强负信号”。

### 10.2 输出三段式，而不是单阈值二分类

```text
score < T_neg            → REJECT
T_neg <= score < T_review→ ABSTAIN / HUMAN REVIEW
score >= T_review        → HIGH_CONF_CANDIDATE
```

但注意：

> `HIGH_CONF_CANDIDATE` 仍然不是 `AUTO_MATCH`。

真正 AUTO_MATCH 还要再次过 Reference Gate。

### 10.3 推荐权限模型

```text
Rule Engine:
  can AUTO_MATCH
  can AUTO_REJECT

Student Model:
  can AUTO_REJECT
  can PRIORITIZE_REVIEW
  can REQUEST_EXTRACTION
  cannot AUTO_MATCH if trusted reference is absent

LLM Teacher:
  can LABEL_OFFLINE_TRAINING_DATA
  can EXTRACT_REFERENCE_CANDIDATES
  cannot AUTO_MATCH production records

Human:
  can approve/reject ambiguous cases
  corrections feed label store
```

从组织和代码层面把权限分开，比靠“prompt 里让模型谨慎”安全得多。

---

## 11. 最终 Auto-Match Gate：可直接落地的判定逻辑

建议生产决策函数：

```python
def decide_pair(left, right):
    # 1. 来源必须不同，且同一来源内部不做跨源 merge
    if left.source == right.source:
        return REJECT("same_source")

    # 2. 品牌必须可信且一致
    if not left.brand.trusted or not right.brand.trusted:
        return ABSTAIN("brand_untrusted")
    if left.brand.id != right.brand.id:
        return REJECT("brand_conflict")

    # 3. 商品角色必须允许
    if left.item_role != "watch" or right.item_role != "watch":
        return REJECT("non_watch_or_accessory")

    # 4. 如果存在高可信 reference 冲突，永久 hard reject
    if left.ref.trusted and right.ref.trusted:
        if left.ref.canonical != right.ref.canonical:
            return REJECT("trusted_reference_conflict")

    # 5. 自动 MATCH 只接受两边高可信 reference 完全相等
    if left.ref.trusted and right.ref.trusted:
        if left.ref.canonical == right.ref.canonical:
            if no_hard_contradiction(left, right):
                return AUTO_MATCH(
                    key=(left.brand.id, left.ref.canonical),
                    reason="trusted_exact_reference"
                )

    # 6. student / LLM 只能决定下一步，不越权
    candidate_score = student.score(left, right)
    if candidate_score < T_NEG:
        return REJECT("student_low_score")

    if candidate_score >= T_EXTRACT:
        return REQUEST_MORE_REFERENCE_EVIDENCE()

    return ABSTAIN("insufficient_reference_evidence")
```

这个函数最关键的安全性质是：

```text
模型错成 MATCH ≠ 系统错合并
```

因为模型输出还没有 merge 权限。

---

## 12. 图结构：借鉴论文，但不要复刻 closure-drop

论文尝试把 positive pair 形成图，再通过桥边/传递闭包寻找可疑标签；实验不是稳定收益。

当前三源腕表恰好适合图，但用途应改成**一致性审计**。

### 12.1 节点与边

```text
node = source product record
edge = candidate / matched evidence
entity_key = (brand_id, canonical_reference)
```

### 12.2 不允许普通传递性自动扩散

危险例子：

```text
A(ref=126610LN) --model says match-- B(ref unknown)
B(ref unknown)  --model says match-- C(ref=126610LV)
```

如果做 connected component clustering，就会错误把：

```text
126610LN 与 126610LV
```

合在一起。

正确规则是：

```text
任何 component 内出现 >1 个 trusted canonical_reference
=> 整个 component quarantine
=> 禁止新增自动边
=> 进入 conflict audit
```

### 12.3 三源一致性审计

如果：

```text
雷小安 A ↔ 腕表之家 B = exact ref
腕表之家 B ↔ 奢当家 C = exact ref
但雷小安 A 与奢当家 C 的可信 ref 冲突
```

那不是“多数表决后自动合并”，而应判定：

```text
数据提取或 reference normalization 至少有一处异常
```

并冻结整组。

图是**报警器**，不是“把一条错边扩散成整个簇”的工具。

---

## 13. 数据模型：保证可审计、可回放、可重算

建议至少有以下表。

### 13.1 raw_product_snapshot

```sql
CREATE TABLE raw_product_snapshot (
    source              TEXT NOT NULL,
    source_product_id   TEXT NOT NULL,
    snapshot_version    BIGINT NOT NULL,
    crawled_at           TIMESTAMP NOT NULL,
    payload_json         JSONB NOT NULL,
    payload_hash         TEXT NOT NULL,
    PRIMARY KEY (source, source_product_id, snapshot_version)
);
```

### 13.2 normalized_product

```sql
CREATE TABLE normalized_product (
    product_key          TEXT PRIMARY KEY,
    source               TEXT NOT NULL,
    source_product_id    TEXT NOT NULL,
    brand_id             TEXT,
    brand_confidence     DOUBLE PRECISION,
    item_role            TEXT,
    title_norm           TEXT,
    canonical_ref        TEXT,
    ref_trust_level      SMALLINT,
    ref_evidence_json    JSONB,
    normalizer_version   TEXT NOT NULL,
    updated_at           TIMESTAMP NOT NULL
);
```

关键索引：

```sql
CREATE INDEX idx_product_ref
ON normalized_product(brand_id, canonical_ref)
WHERE canonical_ref IS NOT NULL;
```

### 13.3 candidate_pair

```sql
CREATE TABLE candidate_pair (
    pair_id              BIGSERIAL PRIMARY KEY,
    left_key             TEXT NOT NULL,
    right_key            TEXT NOT NULL,
    block_reason         TEXT NOT NULL,
    retrieval_score      DOUBLE PRECISION,
    risk_features        JSONB,
    candidate_version    TEXT NOT NULL,
    UNIQUE(left_key, right_key, candidate_version)
);
```

### 13.4 match_decision

```sql
CREATE TABLE match_decision (
    pair_id              BIGINT NOT NULL,
    decision             TEXT NOT NULL, -- MATCH/REJECT/ABSTAIN
    authority            TEXT NOT NULL, -- RULE/HUMAN/MODEL
    reason_code          TEXT NOT NULL,
    canonical_entity_key TEXT,
    evidence_json        JSONB NOT NULL,
    rule_version         TEXT,
    model_version        TEXT,
    created_at           TIMESTAMP NOT NULL,
    PRIMARY KEY(pair_id, created_at)
);
```

必须 append-only，不能只保留“当前结果”，否则无法追查一次错误 merge 当时用了哪个规则/模型/字段。

### 13.5 label_store

```text
pair_id
label              MATCH/NON_MATCH/UNKNOWN
label_source       HUMAN_GOLD/LLM_TEACHER/RULE_WEAK
label_tier
teacher_model
prompt_version
evidence_json
reviewed_by_human
created_at
```

把 production decision 与 training label 分开，避免模型训练逻辑污染线上事实表。

---

## 14. 100 万–1000 万级部署建议

### 14.1 不要把全部 pair 存进 OLTP

粗估：

```text
10M products × 全量 cross-source pair
```

不可行。

必须通过：

```text
brand/ref exact index
        +
blocking
        +
top-K ANN / lexical retrieval
```

把每条疑难商品候选限制到个位数或几十个。

若 10M 商品中只有 20% reference 不可信，且每条生成 K=20 候选：

```text
2M × 20 = 40M candidate pairs
```

这已经从“无法处理的 10^12+”降到可以分区批处理的数量级。

### 14.2 推荐工程组件

技术栈不限时，可以这样拆：

```text
Object Storage / Parquet
  └─ 保存 raw snapshot、训练集、审计产物

PostgreSQL
  └─ reference catalog、当前 normalized record、match decision 元数据

OpenSearch / Elasticsearch
  └─ title/BM25、reference prefix/pattern retrieval

FAISS / Qdrant / Milvus
  └─ 只服务 reference 不确定记录的 ANN 候选召回

Kafka / queue（如确有实时增量）
  └─ ingest -> normalize -> extract -> candidate -> decide

GPU inference worker
  └─ Ditto student batch scoring

LLM labeling batch job
  └─ 仅离线主动学习，不在在线主路径阻塞
```

如果更新以小时/天为批次，不必强上 Kafka；Parquet + workflow scheduler + idempotent batch job 更简单。

### 14.3 增量处理

每条商品保存：

```text
payload_hash
normalizer_version
reference_extractor_version
candidate_version
model_version
```

只有以下变化时重算：

```text
raw payload changed
normalizer upgraded
reference catalog changed
extractor upgraded
model upgraded且该记录仍处于 unresolved
```

已经通过高可信 exact reference 形成的实体，不应因普通模型升级反复漂移。

---

## 15. Label Factory 的具体流水线

可以直接按下面实现。

### Phase A：Rule weak labels

从三源中自动构造高可信 weak labels：

**Positive**：

```text
same canonical brand
+ same trusted canonical reference
+ watch/watch
+ no contradiction
```

**Negative**：

```text
same brand + trusted different reference
```

特别是 near-reference negatives 优先保留。

### Phase B：Human seed

人工标注 50–100 个高风险 pair，保证正负都有，并覆盖三个来源 pair。

### Phase C：第一轮 student

用：

```text
human + rule weak labels
```

训练 5 个 bagged Ditto member。

### Phase D：Active learning

对 unresolved candidate pool 最多抽 2 万或可配置的样本做 student ensemble scoring。

排序：

```text
model disagreement
+ FP risk
+ source/brand diversity
```

每轮选 200–500 个给 LLM Teacher。

### Phase E：Teacher labeling

用 reference-constrained prompt 输出 MATCH/NON_MATCH/UNKNOWN。

正样本还要经过 catalog/ref evidence gate，才写入训练集。

### Phase F：Retrain

加入新 teacher labels，重新训练 ensemble。

停止条件不是“label 数量够了”，而是：

```text
1. immutable gold 上的 precision 不再提升；
2. 新增 round 对高风险桶错误率无明显改善；
3. unresolved coverage 收益已经很低。
```

### Phase G：Deploy student

部署单个最佳 Ditto 或 distill 后的小 encoder，ensemble 只保留在 active-learning job 里，不需要线上 5 倍成本。

---

## 16. 评测：不要再用一个 F1 盖住风险

论文主要用 F1；本需求必须改成 precision-first dashboard。

### 16.1 核心指标

第一优先级：

```text
FP_auto_match_count
AutoMatch Precision / PPV
```

其次：

```text
AutoMatch Coverage
Abstain Rate
Human Review Rate
Reference Extraction Precision
Reference Catalog Hit Rate
Candidate Recall@K
Student Precision@high-score
```

### 16.2 必须单独看 stress buckets

```text
same series / different ref
one-character ref difference
suffix difference
accessory mention
OCR confusable char
high visual similarity / ref conflict
new brand
new source
missing fields
```

如果整体 precision 是 99.99%，但“同系列不同 ref”桶只有 98%，系统仍然不能上线自动合并。

### 16.3 “绝对不能误匹配”的现实表达

统计模型无法数学上承诺绝对零 FP。

要把业务目标实现成工程策略：

```text
学习模型不直接产生最终 MATCH
+
自动 MATCH 只由 deterministic trusted reference gate 产生
+
所有决策有 provenance/version 可审计
+
冲突一律 fail closed / abstain
```

这比把 threshold 从 0.95 调到 0.999999 更可靠。

---

## 17. 分布漂移与持续增量

论文强调训练数据生成的低成本；在当前业务中它最大的长期价值其实是**应对来源漂移**。

每周/每天监控：

```text
brand distribution
ref extraction success rate
catalog miss rate
unknown item_role rate
student score distribution
abstain rate
human rejection rate
hard-conflict rate
```

例如腕表之家改了标题模板后：

```text
reference title extraction 92% → 51%
```

应该触发的是：

```text
extractor drift alert
→ 采样新格式
→ teacher/human label
→ 更新 extractor/student
```

而不是降低 match threshold 强行补 recall。

Active learning 的价值就在这里：每次只标新的“模型最不懂/风险最高”的小批量样本。

---

## 18. 与论文结果不同的两个关键取舍

### 18.1 不采用通用的 graph closure 自动清洗作为主机制

原因：

1. 论文自身实验显示 closure filtering 不是稳定增益；
2. 三源腕表存在“未知 reference 中间节点”，传递关系极易把两个不同 reference 误连；
3. 我们已经有更强的业务 invariant：一个实体簇只能有一个 `(brand, canonical_ref)`。

因此图结构只做：

```text
conflict detection
component quarantine
manual audit prioritization
```

### 18.2 不让 teacher 正样本直接驱动线上 merge

原因：

1. 论文人工 audit 表明 teacher 仍会犯错；
2. benchmark 本身也并非零错误；
3. 本业务 FP 成本远高于 FN。

Teacher 正样本的正确用途：

```text
训练 student
扩充 reference extractor
发现 canonicalization alias
构造 hard case
```

不是“因为 GPT-5.2 说 MATCH，所以把两条商品合并”。

---

## 19. 一个可以立即实现的 MVP

如果希望最快落地，不需要一开始把论文全部复现。

### Sprint 1：Reference-first baseline

实现：

```text
1. 三源统一 schema
2. brand canonicalization
3. reference regex + catalog
4. platform SKU / brand ref role separation
5. exact (brand, canonical_ref) join
6. watch/accessory hard rule
7. append-only decision log
```

先测：

```text
reference extraction precision
exact-match coverage
false positive = 0 的人工抽检
```

### Sprint 2：疑难候选与 hard negatives

实现：

```text
BM25/ANN top-K
same-series nearby ref sampler
OCR confusion sampler
accessory-risk sampler
人工 review queue
```

### Sprint 3：Teacher Label Factory

复用论文思想：

```text
100 个 human seed
5-member Ditto disagreement
每轮 200–500 pair 给 LLM teacher
reference-constrained JSON prompt
UNKNOWN/ABSTAIN
machine label tiering
```

### Sprint 4：Student serving

部署单 Ditto/RoBERTa：

```text
只处理 ref missing/ambiguous 的候选
低分自动 reject
高分进入 ref extraction / review
禁止直接 auto match
```

### Sprint 5：持续学习

上线：

```text
drift monitor
active-learning sampler
weekly/monthly teacher batch
human correction feedback
versioned model/extractor/catalog
```

---

## 20. 推荐的 API 边界

### `POST /normalize-product`

输出 canonical brand、item role、reference evidence。

### `POST /retrieve-candidates`

只对 unresolved record 生成候选。

### `POST /score-pairs`

Ditto student 批量打分，返回 triage score，不返回最终业务 MATCH。

### `POST /decide-pair`

唯一允许产生 `AUTO_MATCH` 的服务；内部执行 deterministic Reference Gate。

### `POST /label-batch`

离线 teacher job，产出 training label。

### `POST /human-review`

写人工 gold/correction，并保留 reviewer、时间和证据。

这种服务边界把“模型判断”和“业务合并”物理隔离，避免后续某个工程师因为模型分数很高就绕过 hard rule。

---

## 21. 示例：三个典型 case 如何走完整流程

### Case A：标题格式不同，但 reference 明确相同

```text
雷小安：劳力士 黑水鬼 126610 LN 全套
腕表之家：Rolex Submariner 126610LN
```

抽取：

```text
brand = rolex / rolex
ref   = 126610LN / 126610LN
role  = watch / watch
```

结果：

```text
AUTO_MATCH
```

不需要 LLM，不需要图片，不需要 Ditto。

### Case B：极像的相邻型号

```text
左：Rolex 126610LN
右：Rolex 126610LV
```

即使：

```text
text similarity = 0.98
image similarity = 0.99
Ditto score = 0.999
```

仍然：

```text
REJECT: trusted_reference_conflict
```

这是整个方案最核心的 invariant。

### Case C：reference 只在图片/脏标题里

```text
左：标题“劳力士潜航者黑盘全套”
右：标题“Submariner Date”
图片 OCR 左：126610LN
图片 OCR 右：126610 LN
```

如果 OCR 单图置信不足：

```text
不自动 MATCH
```

系统可以：

```text
candidate retrieval
→ student high score
→ 多图 OCR / catalog check
→ 如两边形成 trusted ref，再进入 exact gate
→ 否则 ABSTAIN / human review
```

图片最终帮助的是“把 reference 证据补完整”，不是直接靠视觉相似判同款。

---

## 22. 论文方案给当前项目的最大价值

如果只总结一句：

> **把昂贵的 LLM 从“线上判每一个 pair”改成“离线教会一个便宜 student 识别最值得处理的疑难 pair”，再让最终业务判定由 reference 硬规则收口。**

具体收益：

1. **显著降低标注成本**：有限人工黄金标签主要用于独立验收和最危险 hard negatives；
2. **显著降低推理成本**：student 可以批量处理百万候选；
3. **适合持续增量**：active learning 每次只补分布变化部分；
4. **适合多源扩展**：新来源不必重新全量人工标注；
5. **不会牺牲业务安全边界**：模型没有最终 merge 权限。

---

## 23. 最终建议

### 应该直接采用

- LLM teacher → small student 的知识蒸馏；
- Ditto bagged ensemble 做 active-learning disagreement；
- 候选池后再调用 teacher；
- 对机器标签做人工 audit；
- label/model/prompt/version 全量留痕；
- 小模型在线、大模型离线。

### 应该改造后采用

- teacher 二分类 → `MATCH / NON_MATCH / UNKNOWN`；
- 通用 same-entity prompt → reference-constrained evidence prompt；
- 普通 uncertainty sampling → FP-risk-aware active learning；
- student final classifier → triage / reject / review classifier；
- graph closure → reference conflict audit graph。

### 不应该采用

- LLM / Ditto 高分直接自动 merge；
- 无 reference 时靠图像相似补成 MATCH；
- 全量 pair 直接 LLM；
- connected component / transitive closure 自动扩散 MATCH；
- 用整体 F1 代替 precision-first stress test；
- 为提高 recall 而放松 trusted reference gate。

### 推荐最终生产判据

```text
AUTO_MATCH =
    same canonical brand
AND same trusted canonical reference
AND watch/watch
AND no hard contradiction
```

其余所有智能能力——LLM、Ditto、embedding、OCR、图片、图结构——都服务于：

```text
更快地找到 reference
更好地构造候选
更低成本地训练模型
更高效地发现冲突
更少地占用人工审核
```

而不是修改这条最终判据。

在“绝不能误匹配”的约束下，这是本文最值得迁移、也最容易直接落地的实现方式。

---

## 24. 参考资料与代码定位

- 论文：https://arxiv.org/abs/2606.28823
- 论文 HTML：https://arxiv.org/html/2606.28823
- 官方仓库：https://github.com/wbsg-uni-mannheim/Automatic-data-labeling
- 仓库总览：`README.md`
- Artifact 索引：`artifacts/README.md`
- Teacher prompt：`artifacts/prompts/entity_matching_labeling_prompt.txt`
- Precision-oriented relabel prompt：`artifacts/prompts/entity_matching_relabel_prompt.md`
- Similarity selection：`scripts/labeling/similarity_search.py`
- Feature active learning：`scripts/labeling/active_learning_ml.py`
- Ditto active learning：`scripts/labeling/active_learning_ditto.py`
- Ditto training：`scripts/training/train_ditto.py`
- XGBoost training：`scripts/training/train_xgboost.py`
- Qwen training：`scripts/training/train_qwen.py`
- Post-processing：`scripts/post_processing/`
- Teacher human audit：`error_anlysis/`
