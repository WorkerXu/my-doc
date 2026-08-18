# Efficient Model Repository for Entity Resolution: Construction, Search, and Integration

> 分析人：c  
> 论文：Victor Christen, Peter Christen, **Efficient Model Repository for Entity Resolution: Construction, Search, and Integration**, EDBT 2026  
> 论文主页：https://dbs.uni-leipzig.de/research/publications/efficient-model-repository-for-entity-resolution-construction-search-and  
> PDF：https://openproceedings.org/2026/conf/edbt/paper-245.pdf  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级、持续增量；“同一个商品”严格定义为**同一 reference number / 型号**；字段稀疏；reference 有时在结构化字段、有时需要从标题或图片 OCR 抽取；图片可用；**precision 极端优先，绝不能误匹配，允许漏匹配**；可人工标注几百对黄金样本。

---

## 1. 选题与去重

执行前已检查 `奢侈品调研/c/`，c 已经分析过以下工作：

- Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes
- AnyMatch – Efficient Zero-Shot Entity Matching with a Small Language Model
- Confidence Classifiers with Guaranteed Accuracy or Precision
- DeepBlocker
- End-to-end multi-modal product matching in fashion e-commerce
- Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration
- GraLMatch：Matching Groups of Entities with Graphs and Language Models
- How to Fix a Broken Confidence Estimator：Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks
- Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification
- PAM：Understanding Product Images in Cross Product Category Attribute Extraction
- Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce
- Tailoring entity resolution for matching product offers
- TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency
- Using LLMs for the Extraction and Normalization of Product Attribute Values
- parts-distributor-sku-classifier
- pyJedAI

本次选择的 **Efficient Model Repository for Entity Resolution: Construction, Search, and Integration（下文简称 MoRER）** 尚未由 c 分析，因此不重复。

同时检查了 a / b / d 当前目录，本工作也没有被其他成员覆盖。相比继续分析一个“新的 pair matcher”，MoRER 补的是另一个更接近生产架构的问题：

> 当三源数据持续增长、品牌不断增加、标题模板/OCR/字段覆盖持续变化时，怎样判断“当前这一批数据应该继续复用哪个 matcher”，什么时候必须拒绝复用、重新标注和更新模型？

这对本需求非常关键。因为 100 万–1000 万级、持续增量系统最大的隐患不只是“某个模型今天准不准”，而是**把一个在旧分布上很准的模型错误复用到新分布**。对 precision-first 系统，这种错误路由会批量制造 false positive。

---

## 2. 论文解决的核心问题

多源 Entity Resolution 中，如果有 `n` 个数据源，潜在 source-pair ER 任务会快速增长。最直接的方案是：

```text
D1 × D2 -> M12
D1 × D3 -> M13
D2 × D3 -> M23
D1 × D4 -> M14
...
```

每来一个新来源都重新标注并训练模型，成本高；但反过来，把所有来源的数据混在一起训练一个“万能模型”也会遇到问题：不同 source pair 的相似度特征分布可能明显不同。

MoRER 的核心假设是：

> **相似的 ER task 可以共用一个模型；不相似的 task 不应该强行共用。**

因此它不直接把“记录”聚类，而是把**ER 任务**聚类，并建立模型仓库（model repository）。论文的整体工作流可以概括成 5 步：

```text
1. Similarity Distribution Analysis
        ↓
2. ER Problem Clustering
        ↓
3. Model Generation
        ↓
4. Process New ER Problems
        ↓
5. Classification with Appropriate Model
```

论文 Figure 3 正是这个架构：已有 ER problems 先按 feature distribution 计算相似性，构建 ER problem similarity graph，再聚类；每个 cluster 训练一个模型；新 ER problem 到来时先找到最适合的 cluster/model，必要时重新聚类并追加训练数据。

这与当前腕表需求高度契合，因为我们也不应该默认：

```text
雷小安 × 腕表之家 的 Rolex 标题抽取数据
```

和：

```text
奢当家 × 腕表之家 的 Cartier OCR 数据
```

是同一个统计任务。即使最终业务规则都叫“同 reference”，其输入证据质量、字段缺失率、reference 格式、负例类型完全可能不同。

---

## 3. MoRER 的技术实现拆解

## 3.1 ER problem 不是单条 pair，而是一组 similarity feature vectors

论文把两个数据源之间的 ER problem 记作：

```text
p_{i,k}
```

其中每一个候选 record pair 会被转换为 similarity feature vector：

```text
w = [f1, f2, ..., fd]
```

例如论文产品示例里可以有：

```text
simTitle
simBrand
simModelNumber
simPrice
```

一个 ER problem 就是大量这样的向量形成的分布。

这点非常重要：MoRER 判断两个任务是否相似，不是比较“来源名字”，而是比较**这些任务上相似度特征的统计分布是否相近**。

对腕表系统，我们应该做同样的转换，但特征必须改造成 reference-first，而不能照抄通用商品匹配特征。

建议一个 candidate pair 的 feature vector 至少包含：

```text
# reference 证据
structured_ref_present_left
structured_ref_present_right
ref_candidate_count_left
ref_candidate_count_right
canonical_ref_exact
canonical_ref_char_similarity
canonical_ref_len_delta
ref_role_left                 # brand_reference / platform_sku / seller_sku / unknown
ref_role_right
ref_source_left               # structured / title / ocr
ref_source_right
ref_extraction_conf_left
ref_extraction_conf_right
ocr_title_ref_agree_left
ocr_title_ref_agree_right
explicit_ref_conflict

# 品牌 / 系列
brand_exact
brand_alias_conflict
series_similarity

# 文本辅助证据
title_char_similarity
title_embedding_cosine
accessory_keyword_flag
compatibility_phrase_flag

# 图片辅助证据
image_embedding_cosine
caseback_ocr_present
card_ocr_present

# 缺失与来源质量
missingness_signature
source_pair_id
parser_version
```

这里最重要的不是训练一个超级复杂模型，而是让“任务指纹”能够反映：

- structured reference 覆盖率是否变化；
- title extraction 的模式是否变化；
- OCR 是否成为主要证据；
- platform SKU 被误识别成 reference 的比例是否变化；
- hard negative（相邻 reference、兼容配件、表带/盒证）比例是否变化。

这些恰好是最可能让 precision 崩掉的 domain shift。

---

## 3.2 任务相似度：KS / Wasserstein / PSI / C2ST

MoRER 对两个 ER problems 的 feature distribution 做统计比较，论文实现了两类方案。

### 3.2.1 单变量分布比较

对每个 feature 分别比较分布，再汇总：

- **Kolmogorov–Smirnov Test（KS）**：比较两个经验 CDF 的最大距离；
- **Wasserstein Distance（WD）**：比较两个分布搬运距离；
- **Population Stability Index（PSI）**：将分布分桶，衡量比例变化。

论文不是简单平均所有 feature，而是使用 feature 的标准差作为权重，目的是让区分能力更强的 feature 获得更高影响。

### 3.2.2 多变量 C2ST

论文还使用 **Classifier Two-Sample Test（C2ST）**：

```text
数据集 A 的 feature vectors -> label 0
数据集 B 的 feature vectors -> label 1
                 ↓
训练一个分类器区分 A / B
```

如果分类器很容易区分，说明两个 ER task 分布不同；如果难以区分，则分布相似。论文因为 ER task 大小不平衡，使用 F1，并把 similarity 定义成分类器 F1 的反向量。

### 论文实验给出的工程启示

论文在 heterogeneous / noisy 数据上发现：

- PSI 整体较稳健；
- KS 和 C2ST 整体表现也很可靠；
- Wasserstein 对数据集和 AL 组合较敏感，并不总是稳定。

因此本项目不建议一开始就“只押一个距离函数”。第一版可做：

```text
PSI              -> 快速、可解释的在线 drift 指标
KS               -> 每个关键 feature 的稳定比较
C2ST              -> 离线 / 周期性的多变量二次确认
```

而且对于 precision-first 的 reference 匹配，不能只算一个总体平均分。建议把 reference 关键 feature 设成 **sentinel features**：

```text
structured_ref_missing_rate
ref_role_distribution
explicit_ref_conflict_rate
ref_candidate_count
ref_extraction_confidence
```

只要这些 sentinel 明显漂移，即使 title/image 等通用特征总体看起来相似，也不能直接复用旧模型。

---

## 3.3 构建 ER Problem Similarity Graph

MoRER 把每一个 ER problem 作为图节点：

```text
G_P = (P, E)
```

边表示两个 ER problems 之间的 distribution similarity，边权就是聚合后的相似度。

然后使用 **Leiden algorithm** 做社区发现，把相似 ER tasks 分到同一个 cluster。论文选择 Leiden 的原因包括：

- 能找到弱连通大图里的高内聚子群；
- 可扩展性较好；
- 适合 ER problems 数量不断变多的场景。

这一步的意义是把：

```text
一个 source-pair 一个模型
```

变成：

```text
一簇相似任务一个模型
```

从而降低模型数量和标注成本。

### 对腕表业务的关键改造

“ER task”的粒度不能只定义成 source pair。

如果只定义：

```text
雷小安 × 腕表之家
雷小安 × 奢当家
腕表之家 × 奢当家
```

那只有 3 个 task，MoRER 的价值非常有限，也掩盖品牌/证据形态差异。

建议任务键定义为：

```text
(source_pair,
 brand_family,
 evidence_regime,
 batch_or_time_window,
 feature_schema_version)
```

例子：

```text
雷小安→腕表之家 | Rolex   | structured_ref | 2026-W33 | v3
雷小安→腕表之家 | Rolex   | title_extract  | 2026-W33 | v3
奢当家→腕表之家 | Cartier | OCR+title      | 2026-W33 | v3
奢当家→腕表之家 | Omega   | title_extract  | 2026-W34 | v3
```

这样 MoRER 才真正成为**数据分布路由器**：它决定某个新批次、品牌、证据模式到底能不能复用已有模型。

---

## 3.4 每个 cluster 一个 matcher，而不是所有 task 一个 matcher

MoRER 对每个 cluster `C_i` 生成一个分类模型：

```text
C1 -> M_C1
C2 -> M_C2
C3 -> M_C3
```

论文认为同 cluster 内的 ER tasks 特征分布相似，因此可以共享 classifier。

这对当前需求非常实用，但应改变 classifier 的职责。

### 论文原始职责

```text
feature vector -> MATCH / NON-MATCH
```

### 本项目建议职责

```text
feature vector
    -> evidence quality / ambiguity / hard-negative risk
    -> support score
```

最终自动合并仍由 reference 硬规则收口：

```text
if trusted canonical reference A != trusted canonical reference B:
    NO_MATCH

elif trusted canonical reference A == trusted canonical reference B:
    if no hard conflict:
        AUTO_MATCH
    else:
        REVIEW

else:
    ABSTAIN / REVIEW
```

换句话说：

> **MoRER 决定“应该用哪个辅助 matcher / extractor”，但不能让模型越权改写 reference equality。**

这就是本论文最适合本 Spec 的定位：模型生命周期与路由层，而不是最终真值定义层。

---

## 3.5 标注预算与 Active Learning

MoRER 给定总标注预算 `b_tot`，每个 cluster 至少分配 `b_min`。剩余预算再按 cluster 的规模等因素分配。

如果 cluster 数量太多，导致：

```text
cluster_count × b_min > b_tot
```

论文会把 singleton clusters 与非 singleton clusters 合并，以避免每个小簇都消耗独立最低预算。

在训练数据选择上，论文集成两种 Active Learning：

1. **Almser**：专门面向 multi-source ER；
2. **Bootstrap uncertainty**：对当前训练集做 bootstrap，训练 `k` 个分类器，用模型之间的不确定性选择待标注 pair。

Bootstrap 的核心不确定性可理解为：

```text
p = k 个模型中预测 MATCH 的比例
uncertainty = p * (1 - p)
```

当模型投票接近 50/50 时，不确定性最大。

论文还引入类似 IDF 的 uniqueness score：优先选择那些关联 record 在多个 cluster 中并不反复出现的 feature vector，避免标注预算被重复、低信息量实例占满。

### 对本项目的标注策略不能照抄“纯 uncertainty”

我们的业务损失函数极端偏向 false positive，因此更应该主动采样：

```text
40%  高分但可能为 false positive 的边界样本
30%  hard negatives：
     - 同系列不同 reference
     - reference 只差 1~2 字符
     - 标题包含“适配某型号”
     - 表带 / 盒证 / 配件
     - 平台 SKU 像 reference
20%  新来源 / 新品牌 / 新模板 / OCR 漂移样本
10%  高置信 clean positives，用于校准基线
```

即：

> **Active Learning 目标不是最快提高总体 F1，而是最快发现“会让系统误合并”的边界。**

如果几百条黄金标签是全部人工预算，这种定向标注比随机标注更符合需求。

---

## 3.6 新任务如何选择已有模型：sel_base

论文的 `sel_base` 假设 domain shift 较小：

1. 为新 ER problem 计算 feature distribution；
2. 与各 cluster 中用于训练的 feature-vector sets 比较；
3. 找到 similarity 最高的 cluster；
4. 直接使用对应 `M_Ci`。

概念上是：

```python
cluster = argmax(task_similarity(new_task, cluster_training_distribution))
model = model_repo[cluster]
```

这种策略很适合：

- 同一品牌；
- 同一 evidence regime；
- 同一 parser/OCR 版本；
- 只是新增一批相同来源模板的数据。

但对“绝不能误匹配”的业务，必须再加一个最低 similarity 门槛：

```python
if best_similarity < ROUTING_THRESHOLD:
    return UNKNOWN_TASK
```

论文的目标是 ER 质量，不等价于我们的“宁可不做，也不能错误复用”。因此不能永远 `argmax` 后强行选一个 cluster。

---

## 3.7 处理 domain shift：sel_cov

MoRER 更有价值的是第二种 `sel_cov` 策略。

当新 task 到来时：

1. 把新 task 与全部已知 ER tasks 比较；
2. 将新 task 加入 ER problem similarity graph；
3. 重新运行 Leiden；
4. 根据新 cluster 与旧 cluster 的重叠决定复用哪个模型；
5. 如果新 task 所在 cluster 全部都是未训练任务，则训练新模型；
6. 如果新数据在 cluster 中占比太高，则触发模型更新。

论文定义了 coverage：

```text
cov(C) =
  cluster C 中来自尚未训练 ER problems 的 feature vectors 数
  ---------------------------------------------------------
  cluster C 中所有 feature vectors 数
```

当：

```text
cov(C) > t_cov
```

就用新任务中的 feature vectors 再做 Active Learning，补标数据并更新模型。

论文实验比较了：

```text
t_cov = 0.1 / 0.25 / 0.5
```

较低 threshold 会更频繁 retrain，质量通常更好但标注成本也更高，而且不同数据集并不存在一个永远最优的固定值。

### 对腕表系统的改造

不建议只用“新 feature vectors 占比”触发更新，而应该做**双触发**：

```text
触发条件 A：coverage > t_cov
或
触发条件 B：reference sentinel drift > threshold
```

例如即使新批次只占 cluster 的 5%，但出现：

```text
reference structured field 覆盖率从 85% -> 20%
OCR 成为主证据
平台 SKU 格式发生变化
```

这已经是高风险 shift，应立即进入 `REVIEW / UPDATE`，不能等 coverage 达到 25%。

---

## 4. 论文原方案不能直接用于本需求的地方

## 4.1 MoRER 优化的是模型复用，不提供“零 false positive”保证

论文的最终输出仍是普通 ER classifier 的 match / non-match，主要评估指标也是 F1 等 ER 质量指标。

而当前 Spec 的业务目标是：

```text
false positive cost >> false negative cost
```

甚至明确要求：

```text
绝对不能误匹配，允许漏匹配
```

因此不能部署为：

```text
MoRER router -> cluster matcher -> MATCH -> 自动合并
```

必须部署为：

```text
MoRER router
    -> 选择合适的 extractor / matcher
    -> 生成证据
    -> reference hard gate
    -> conflict veto
    -> MATCH / REVIEW / ABSTAIN
```

MoRER 在这里解决的是：

> **“当前数据应该信任哪套模型，以及什么时候旧模型已经不该被复用。”**

而不是：

> “模型高分就证明 reference 相同。”

---

## 4.2 相同 source pair 不代表相同任务

Rolex、Omega、Cartier、AP 的 reference 格式不同；结构化字段、标题模板和 OCR 质量也不同。

如果直接把每个 source pair 当作一个 ER task，任务分布会被严重平均化。

所以必须把 `brand_family + evidence_regime` 纳入 task 定义。

---

## 4.3 论文假设各 ER problems 有 common feature space

论文在 limitation 中明确指出，它主要考虑 feature-space 上的异质性，并要求 ER problems 有公共 features；如果属性完全不同，需要先建立可比表示。

本项目刚好存在字段稀疏：

```text
来源 A：有 reference 独立字段
来源 B：reference 埋在 title
来源 C：title 也不可靠，需要 OCR
```

所以必须先做 **canonical evidence schema**，再进入 MoRER：

```text
原始字段
  ↓
统一证据抽取层
  ↓
canonical_ref_candidates[]
brand
series
reference_role
source_of_evidence
confidence
conflicts[]
  ↓
统一 feature schema
  ↓
MoRER task fingerprint / matcher
```

不能直接拿三个来源原始 column 做分布比较。

---

## 4.4 模型路由相似度也必须可拒绝

MoRER `sel_base` 的思想是选最相似 cluster，但 precision-first 场景中必须存在：

```text
UNKNOWN_TASK
```

如果所有 cluster 都不够像，系统应该：

```text
拒绝模型复用
-> 人工标注少量样本
-> 建立新 cluster/model
```

而不是“在不合适的模型里选一个最不差的”。

---

## 5. 面向当前 Spec 的直接落地架构

## 5.1 总体架构

```text
                    ┌──────────────────────────┐
                    │ 雷小安 / 腕表之家 / 奢当家 │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │  Raw / Incremental Ingest │
                    └────────────┬─────────────┘
                                 │
                                 ▼
              ┌────────────────────────────────────┐
              │ Canonicalization & Evidence Extract │
              │ - brand normalization              │
              │ - ref role classification          │
              │ - title ref extraction             │
              │ - OCR ref extraction               │
              │ - ref canonicalization              │
              └──────────────┬─────────────────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │ Precision-first Blocking│
                 │ brand / candidate ref   │
                 └───────────┬────────────┘
                             │ candidate pairs
                             ▼
                 ┌────────────────────────┐
                 │ Feature Vector Builder  │
                 └───────────┬────────────┘
                             │
                 ┌───────────┴────────────┐
                 │                        │
                 ▼                        ▼
       ┌──────────────────┐      ┌──────────────────┐
       │ Task Fingerprint │      │ Hard Conflict Gate│
       │ PSI/KS/C2ST      │      │ ref mismatch etc. │
       └────────┬─────────┘      └──────────────────┘
                │
                ▼
       ┌──────────────────────┐
       │ MoRER Task Router     │
       │ Leiden + Model Repo   │
       └────────┬─────────────┘
                │
                ▼
       ┌──────────────────────┐
       │ Cluster-specific Model│
       │ extractor/support/risk│
       └────────┬─────────────┘
                │
                ▼
       ┌───────────────────────────────┐
       │ Reference Admission Gate      │
       │ strict equality + no conflict │
       └──────────────┬────────────────┘
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
   AUTO_MATCH      REVIEW         ABSTAIN
```

其中最关键的边界是：

> **Task Router 和 Cluster Matcher 永远不能绕过 Reference Admission Gate。**

---

## 5.2 Canonical reference 数据模型

建议每条商品不要只保存一个 `reference` 字符串，而保存候选与证据：

```json
{
  "listing_id": "...",
  "brand_id": "rolex",
  "ref_candidates": [
    {
      "raw": "126610 LN",
      "canonical": "126610LN",
      "role": "brand_reference",
      "source": "title",
      "extractor_version": "ref-v3",
      "confidence": 0.997,
      "evidence_span": "..."
    },
    {
      "raw": "A928172",
      "canonical": "A928172",
      "role": "platform_sku",
      "source": "structured_field",
      "confidence": 1.0
    }
  ],
  "conflicts": []
}
```

这能防止最危险的一类错误：

```text
“看到一个像型号的字母数字串” == “把它当品牌 reference”
```

MoRER 的 feature distribution 也应该基于这些语义化证据，而不是基于任意字符串相似度。

---

## 5.3 Task Registry

建立 `er_task` 表：

```text
er_task
-------
task_id
source_left
source_right
brand_family
evidence_regime
time_window
feature_schema_version
extractor_version
candidate_count
fingerprint_uri
cluster_id
routing_status
created_at
```

例如：

```text
T_2026W33_RL_WB_ROLEX_TITLE
T_2026W33_SD_WB_CARTIER_OCR
```

任务粒度应可配置，避免品牌太长尾时 task 爆炸。可采用：

```text
Top brands -> brand-level task
Long tail  -> reference-format family / brand-family task
```

---

## 5.4 Task Fingerprint

不需要保存全量 candidate pair 才能做路由。对每个 task 采样例如 5 万条 candidate pairs，保存：

```text
feature histogram
quantiles
missing rate
mean/std
categorical distribution
sentinel metrics
```

例如：

```json
{
  "canonical_ref_exact": {
    "rate": 0.118
  },
  "explicit_ref_conflict": {
    "rate": 0.032
  },
  "structured_ref_present_left": {
    "rate": 0.87
  },
  "ref_source_right": {
    "structured": 0.12,
    "title": 0.73,
    "ocr": 0.15
  },
  "ref_extraction_conf_right": {
    "p10": 0.42,
    "p50": 0.91,
    "p90": 0.998
  }
}
```

因此对 1000 万级 listings，MoRER 路由层的计算量仍然主要与：

```text
任务数 × fingerprint sample size
```

相关，而不是对所有 listings 做任务两两比较。

---

## 5.5 Model Repository 的元数据

模型仓库不能只有一个模型文件。建议至少记录：

```text
model_registry
--------------
model_id
cluster_id
model_type
artifact_uri
feature_schema_version
extractor_version
training_task_ids
training_label_ids
training_time
routing_distribution_signature
precision_on_gold
false_positive_count_on_gold
coverage_on_gold
brand_scope
evidence_regime_scope
activation_status        # shadow / canary / active / retired
parent_model_id
```

额外保存：

```text
cluster_registry
----------------
cluster_id
member_task_ids
leiden_version
similarity_metric
similarity_threshold
sentinel_thresholds
centroid_or_representative_fingerprint
created_at
```

这样一个错误匹配才能追溯到：

```text
原始 listing
-> extractor version
-> task routing
-> cluster
-> model version
-> hard rule version
-> 最终 decision
```

对于“绝不能误匹配”的系统，可审计性本身就是核心能力。

---

## 5.6 New Task Router 的建议实现

```python
def route_task(new_task):
    fp = build_fingerprint(new_task)

    # 1. feature schema / reference 语义不兼容时直接拒绝复用
    if not schema_compatible(fp):
        return {"action": "NEW_TASK"}

    # 2. sentinel feature 先做 precision-first 漂移检查
    if severe_reference_drift(fp):
        return {"action": "REVIEW_AND_UPDATE"}

    # 3. 与已训练 cluster 比较
    scores = {}
    for cluster in model_repository.active_clusters():
        scores[cluster.id] = task_similarity(fp, cluster.fingerprint)

    best_cluster, best_score = max(scores.items(), key=lambda x: x[1])

    # 4. 与论文 sel_base 不同：允许 UNKNOWN，而不是强行 argmax
    if best_score < ROUTING_THRESHOLD:
        return {"action": "NEW_TASK"}

    # 5. 判断新任务在该簇中的 coverage / drift
    cov = projected_coverage(new_task, best_cluster)
    if cov > TCOV:
        return {
            "action": "UPDATE_CLUSTER_MODEL",
            "cluster_id": best_cluster,
            "label_budget": allocate_precision_first_budget(cov)
        }

    return {
        "action": "REUSE_MODEL",
        "cluster_id": best_cluster
    }
```

如果系统初期 task 少，不必马上完整实现动态图 reclustering；可以先：

```text
PSI/KS router + static clusters + UNKNOWN_TASK
```

验证有效后再引入 Leiden 和 `sel_cov`。

---

## 5.7 最终 Auto-Match 决策必须独立于 MoRER

推荐决策状态：

```text
AUTO_MATCH
NO_MATCH
REVIEW
ABSTAIN
```

伪代码：

```python
def decide(pair, routed_model):
    a = pair.left
    b = pair.right

    # 1. 品牌冲突
    if trusted_brand(a) != trusted_brand(b):
        return "NO_MATCH"

    # 2. 两侧都有可信 canonical reference，且明确不同
    if trusted_ref(a) and trusted_ref(b) and a.ref != b.ref:
        return "NO_MATCH"

    # 3. accessory / compatibility / SKU-role 等硬冲突
    if has_hard_conflict(pair):
        return "NO_MATCH"

    # 4. 没有得到 reference，不允许模型凭“很像”自动合并
    if not trusted_ref(a) or not trusted_ref(b):
        return "ABSTAIN"

    # 5. reference 完全一致才有资格进入自动放行候选
    if a.ref != b.ref:
        return "NO_MATCH"

    # 6. matcher 只能否决/升级人工，不可把 ref mismatch 改成 match
    risk = routed_model.predict_risk(build_features(pair))
    if risk.has_conflict or risk.is_ood:
        return "REVIEW"

    return "AUTO_MATCH"
```

这满足 Spec 的核心定义：

```text
same product = same reference number
```

而图片、标题 embedding、LLM、cluster matcher 都只能用于：

- 抽取 reference；
- 判断 reference 角色；
- 发现冲突；
- 路由模型；
- 决定是否拒识；

不能把“视觉很像”升级成 reference 相同。

---

## 6. 图片在 MoRER 架构里的正确位置

当前数据有图片，但同系列不同 reference 的腕表外观可能极其相似，因此图片不适合作为最终同款证明。

建议三个用途：

### 6.1 OCR 产生独立 reference 证据

优先识别：

```text
表背刻字
保卡
吊牌
盒标
发票 / 证书
```

如果 title reference 与 OCR reference 一致：

```text
ocr_title_ref_agree = 1
```

这是强辅助证据。

### 6.2 视觉用于 hard-negative / conflict veto

比如两个 listing reference 字符串被错误规范到同一个值，但图片明显是不同系列，可升级为 REVIEW。

### 6.3 视觉分布也可加入 task fingerprint

若某来源图片从“商品主体图”突然变为“直播截图 / 拼图 / 水印图”，image embedding / OCR 质量分布会漂移。MoRER 可以据此判断旧视觉模型不再适合。

但要保持原则：

> 图片可以增加拒绝理由，不能单独创造自动合并理由。

---

## 7. 持续增量场景的运行方式

## 7.1 在线路径

```text
新增 listing
  -> canonical extraction
  -> blocking
  -> candidate pairs
  -> 根据 task_id 路由当前 active model
  -> reference admission gate
  -> decision
```

在线请求不应该每条都跑 Leiden。

## 7.2 准实时 / 离线路径

每个时间窗口（如小时/天）聚合 task fingerprint：

```text
new batch
  -> fingerprint
  -> PSI / KS sentinel monitor
  -> task similarity
  -> route / unknown / update
```

## 7.3 周期性 graph maintenance

例如每天或每周：

```text
更新 task graph
-> Leiden reclustering
-> 检查 cluster membership 变化
-> shadow 新模型
-> 黄金集验证
-> canary
-> active
```

不是每个新 listing 都重训，而是**任务分布发生变化时才重训**。

---

## 8. 100 万–1000 万规模下的工程实现建议

### 数据层

```text
PostgreSQL     -> 模型/任务/决策元数据
ClickHouse     -> 大量 feature / fingerprint / audit analytics
Object Storage -> 图片、模型、fingerprint artifacts
Kafka/Pulsar   -> 增量 listing 与 task events
```

### 计算层

```text
Polars / Spark -> 批量特征与 distribution aggregation
scipy          -> KS / Wasserstein
自定义 SQL     -> PSI
LightGBM       -> C2ST / cluster matcher baseline
igraph + leidenalg -> task graph clustering
MLflow         -> model registry / lineage
```

模型不需要一开始就上大型 LLM。reference-first 任务大量信息是结构化规则、字符串和 role classification，小模型更容易做版本化、校准和审计。

### 为什么可扩展

主数据是千万级，但 task 数通常远小于 listing 数。MoRER 只需维护：

```text
每个 task 的采样分布
每对 task 的 similarity
task graph
每个 cluster 的模型
```

因此 model routing 层不是最大的性能瓶颈；真正要避免的是候选对笛卡尔爆炸，所以 Blocking 仍必须在前面完成。

---

## 9. 黄金标签如何分配

用户允许几百对人工黄金标签，这个预算很宝贵。

建议第一轮 300 对示例：

```text
120 对：同品牌、同系列、不同 reference 的 hard negatives
 60 对：平台 SKU / seller SKU 与品牌 reference 混淆
 50 对：标题 reference 与 OCR / structured 字段冲突
 40 对：真实同 reference、但字段高度缺失的 positives
 30 对：新增品牌 / 新模板 / 新 evidence regime 的 drift cases
```

标签字段不要只存 `match=0/1`，还要存：

```text
canonical_ref_left
canonical_ref_right
ref_role_left
ref_role_right
reason_code
reviewer
```

reason code 例如：

```text
SAME_REFERENCE
DIFFERENT_REFERENCE
PLATFORM_SKU_CONFUSION
ACCESSORY_COMPATIBILITY
OCR_CONFLICT
UNKNOWN_REFERENCE
```

这样黄金标签既能评估最终匹配，也能训练 reference role / extraction 组件，并解释 false positive 的来源。

---

## 10. 评测指标：不要照论文只看 F1

MoRER 论文以 F1 等 ER 指标验证模型复用是合理的，但本项目上线门槛应该改成：

### Primary

```text
Auto-Match Precision
False Positive Count
```

其中测试黄金集上：

```text
FP > 0 -> 不允许自动放行
```

### Secondary

```text
Auto-Match Coverage
Abstention Rate
Review Rate
Reference Extraction Precision
Reference Role Classification Precision
```

### MoRER 路由层指标

```text
Model Reuse Rate
Unknown Task Rate
Drift Detection Recall
Wrong-Model Routing Rate
Labels Required per New Task
Retrain Frequency
```

尤其要单独监控：

```text
wrong_model_routing -> downstream FP
```

因为这正是引入模型仓库后新的系统性风险。

---

## 11. 分阶段落地方案

## Phase 0：先把 reference 语义做正确

目标：建立可靠的 canonical reference 与编号角色。

完成：

```text
brand normalization
reference parser / normalizer
reference role classifier
structured/title/OCR evidence schema
hard conflict rules
```

没有这一层，不应该开始 MoRER。

---

## Phase 1：固定 3 source pairs + Top 品牌做 task fingerprints

先选高量品牌：

```text
Rolex / Omega / Cartier / AP / Patek ...
```

按 evidence regime 分 task，采样 candidate pairs，计算：

```text
PSI
KS
missingness
reference sentinel metrics
```

先可视化哪些 task 真正相似，不急着自动聚类。

交付物：

```text
Task Registry v1
Fingerprint Pipeline v1
Distribution Drift Dashboard
```

---

## Phase 2：建立 Model Repository + Static Router

人工把分布相近 task 先划成少量 clusters；每簇训练一个轻量 matcher / risk model。

路由支持：

```text
REUSE_MODEL
UNKNOWN_TASK
```

先不自动 recluster。

任何 UNKNOWN_TASK：

```text
ABSTAIN
+ 进入人工采样池
```

这是最符合 precision-first 的 MVP。

---

## Phase 3：引入 Leiden + sel_cov

当 task 足够多后再做：

```text
ER task similarity graph
Leiden clustering
cluster model lifecycle
coverage-triggered update
```

模型先 shadow，必须通过黄金集和 replay 才能 active。

---

## Phase 4：持续漂移与自动标注编排

每日 / 每周自动生成：

```text
Top drift tasks
Unknown tasks
High-risk false-positive candidates
Recommended labeling batch
Model update candidates
```

人工只标最值得标的几十条，而不是重复从头训练每个来源。

---

## 12. 与 c 已分析方案的组合关系

MoRER 最适合当“模型编排层”，与此前结果不是替代关系：

```text
DeepBlocker / Blocking
        ↓
reference extraction + normalization
        ↓
MoRER task router / model repository     ← 本文
        ↓
cluster-specific matcher / extractor
        ↓
Confidence / selective gate
        ↓
TransClean / GraLMatch cluster audit
        ↓
最终实体簇
```

更精确地说：

- **DeepBlocker**：控制千万级候选规模；
- **parts-distributor-sku-classifier**：防止把平台 SKU 当 reference；
- **Using LLMs / RAG / PAM**：补 reference 抽取与规范化；
- **MoRER**：决定新来源/品牌/批次应复用哪一套模型，以及何时 domain shift 需要更新；
- **Confidence Classifier / selective prediction**：让模型不确定时拒识；
- **TransClean / GraLMatch**：在多源图层面发现错误边和传递冲突。

这套组合比单一“多模态相似度模型”更符合本需求，因为它把风险分成了：

```text
候选风险
编号语义风险
模型路由风险
单 pair 决策风险
多源图传播风险
```

每一层都可以拒绝，而不是把所有责任压给一个模型分数。

---

## 13. 推荐的最终生产规则

### 可以自动 MATCH

仅当：

```text
1. canonical brand 一致
2. 两侧均获得可信 brand_reference
3. canonical reference 严格一致
4. 不存在 platform SKU / accessory / compatibility 等角色冲突
5. 不存在结构化字段、title、OCR 间的强冲突
6. 当前 task 被路由到已验证 cluster/model，且非 OOD
```

### 必须 NO_MATCH

任一条件成立：

```text
可信 reference 明确不同
可信 brand 明确不同
一侧是配件 / 兼容品，而另一侧是腕表主体
编号被识别为平台 SKU / seller SKU
```

### 必须 ABSTAIN / REVIEW

```text
reference 缺失
reference 多候选冲突
新品牌 / 新模板无法路由
PSI/KS sentinel drift 超阈值
模型 cluster coverage 失衡
OCR 与 title 冲突
```

这意味着模型只能帮助增加 coverage，不能降低 precision gate。

---

## 14. 最终建议

MoRER 最值得直接落地的不是“它的 classifier”，而是以下三个架构思想：

### 14.1 把持续增量匹配建模成一组不断出现的 ER tasks

不要假设一个模型永久适配所有品牌、来源和时间窗口。

### 14.2 用 feature distribution 判断“能不能复用模型”

将模型选择从：

```text
if source == X: use model_X
```

升级成：

```text
if current_task_distribution is similar to validated_cluster:
    reuse model
else:
    abstain + label + update
```

### 14.3 建立真正的 Model Repository 和模型生命周期

模型仓库必须知道：

```text
它在哪些 task 上训练过
适配哪些 evidence regime
输入分布是什么
使用了哪些标签
precision / FP 结果如何
何时因为 drift 被替换
```

对于当前 Spec，我建议把 MoRER 作为**“Reference Matching Model Router & Lifecycle Layer”** 落地，而不是当最终匹配器。最终自动匹配仍坚持：

> **canonical reference 的可信严格一致是必要条件；任何模型、LLM、图片相似度都不能把 reference 不一致或缺失的 pair 越权升级为自动 MATCH。**

这样既利用了 MoRER 对多源、持续增量、有限标注预算的优势，又不会引入与“绝不能误匹配”相冲突的模型复用风险。

---

## 15. 一句话结论

**MoRER 最适合作为三源腕表系统的“任务分布检测 + 模型路由 + 漂移重训”控制平面：用 PSI/KS/C2ST 描述不同品牌/来源/证据模式的 ER task，以 Leiden 聚类并复用 cluster-specific 模型；新任务不够相似时允许 `UNKNOWN_TASK` 并拒绝自动匹配，发生覆盖/关键 reference 特征漂移时用少量 hard-negative 主导的 Active Learning 更新模型；最终业务真值仍由可信 canonical reference 严格一致和 conflict veto 收口。**
