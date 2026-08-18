# Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中排除我已经分析过的 Ameli、DeepBlocker、Tailoring entity resolution for matching product offers、TransClean 后，选取以下未分析文章进行深入分析：

**Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration**

- 论文：<https://www.sciencedirect.com/science/article/pii/S1474034625004318>
- DOI：<https://doi.org/10.1016/j.aei.2025.103538>
- 官方代码：<https://github.com/quickhdsdc/LLM4EntityMatching>
- 需求：<https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31>

这篇文章对当前需求最有价值的点是：**它不再把候选商品逐对独立做 Match/NoMatch，而是把同一个 query 的一组近邻候选同时放进竞争关系中，专门学习“真正匹配项必须压过最像它的 hard negatives”。**

这正对应二奢腕表里最危险的错误：

```text
Rolex Submariner 126610LN   <- 真正 reference
Rolex Submariner 126610LV   <- 只差后缀、外观也很像
Rolex Submariner 116610LN   <- 同系列上一代
Rolex Submariner 124060     <- 同系列无历款
```

普通 pairwise matcher 分别看 `(query, candidate)` 时，很容易把多个相似型号都打成高分；Selective EM 强制这些相似候选**在同一候选集内互相竞争**，因此非常适合作为“reference 歧义消解器”。

但**不能原样照搬**论文实现。当前 Spec 有两个比论文更强的业务约束：

1. “同一个商品”被定义为 **同一 reference number / 型号**；
2. **绝对不能误匹配，precision 优先到极致，允许漏匹配。**

而论文/代码中的 selector 最终使用 `argmax`，即默认必须选出一个候选，没有业务需要的 `ABSTAIN / NO_MATCH`；另外代码在构造 top-10 训练数据时，如果真实匹配没有被 blocker 召回，甚至会直接把 gold truth 塞到第 10 位，这在实验训练中可以保证 selector 看得到正样本，但会掩盖真实生产中的 blocker miss。

因此对本需求的正确迁移方式不是“商品记录 -> 商品记录做 SelectEM”，而是进一步改造成：

> **商品记录 -> Canonical Reference Entity 的高精度实体链接（Reference Entity Linking）**。

推荐架构：

```text
雷小安 / 腕表之家 / 奢当家 record
        │
        ▼
Reference Observation 抽取
(结构化字段 / title / OCR / 图片)
        │
        ▼
品牌感知的保守规范化
        │
        ▼
候选 Reference Entity 检索 Top-K
        │
        ├─ 明确 exact + 唯一 + 无冲突 ───────► 直接 VERIFIED
        │
        └─ 歧义候选集
              │
              ▼
      SelectEM / CMRL listwise 排序
              │
              ▼
  hard gate + score margin + conflict veto
       │               │
       ▼               ▼
   VERIFIED         ABSTAIN/REVIEW
       │
       ▼
record_reference_link(reference_entity_id)
       │
       ▼
跨平台同 reference 自动形成商品实体组
```

最终原则：

> **模型只负责“在候选 reference 中排序”和发现疑难项，不拥有越过 reference 硬规则直接合并商品的权限。**

这比直接做 record-to-record matching 更符合 Spec，也把千万级数据的潜在笛卡尔比较降维成 `record -> reference dictionary` 的链接问题。

---

## 1. 论文解决的核心问题：Pairwise EM 不知道“旁边还有一个更像的候选”

传统 Entity Matching 通常是两阶段：

```text
Blocking / Retrieval
    -> 得到 K 个候选
    -> 对每个 (query, candidate) 独立做二分类
```

问题出在第二步。

例如 query 是：

```text
Omega Speedmaster 310.30.42.50.01.002
```

候选集中可能同时存在：

```text
310.30.42.50.01.001
310.30.42.50.01.002   <- true
310.32.42.50.01.001
311.30.42.30.01.005
```

一个 pairwise 模型看到 `(query, 310.30.42.50.01.001)` 时，只知道这两个文本高度相似，并不知道候选集中还有 `.002` 这个更合理的 true match。

论文认为这种 **Competitive / Hard Negative** 是传统 EM benchmark 被低估的问题：随机负例往往太容易，生产中的 blocker 却天然会把“最像但不相同”的记录放在一起。

因此作者把任务改成 listwise selection：

```text
query
   ├─ candidate_1
   ├─ candidate_2
   ├─ candidate_3
   └─ ... candidate_K

一次比较整组，学习 true positive 应该排在 hard negative 前面。
```

这个视角对腕表 reference 极其重要，因为腕表型号的“最危险错误”恰恰不是完全不相关商品，而是**同品牌、同系列、相邻 reference、相近外观**。

---

## 2. 官方项目的真实实现架构

官方仓库：`quickhdsdc/LLM4EntityMatching`。

核心文件：

```text
EM_data_convert.py
EM_training_contrastive.py

tasks/SelectiveEntityMatching/selection_llm/
├── data_preprocessor.py
├── model_loader.py
├── modelling_llama.py
├── model_finetuner.py
└── evaluater.py
```

### 2.1 候选生成：Linq-Embed-Mistral + FAISS

`EM_data_convert.py` 的思路是：

1. 把 target corpus 每条 entity 的多个字段拼成文本；
2. 用 `Linq-AI-Research/Linq-Embed-Mistral` 编码；
3. embedding 做 L2 normalize；
4. 使用 `faiss.IndexFlatIP`；
5. 对每个 query 搜索 top-20；
6. 取 top-10 作为 SelectEM 的候选集合。

因为 embedding 已归一化，`IndexFlatIP` 的 inner product 等价于 cosine ranking。

代码中的 query 还会加 instruction：

```text
Instruct: Given a query entity, retrieve similar entity that matches the query
Query: ...
```

这一步的目标不是直接决定 match，而是构造“高度相似、因此难以区分”的候选集。

#### 一个非常重要的实验细节

官方代码有如下逻辑：

```python
if 1 not in label_k10:
    candidates_k10[-1] = truth_list[0]
    label_k10[-1] = 1
```

也就是说：**如果 true match 不在 blocker 的 top-10 中，代码会强制把 truth 放进候选集。**

这使训练/选择器实验始终有正样本，但生产系统不能这么做，因为线上没有 gold truth 可以“补进去”。

因此我们的评估必须拆成两个指标：

```text
Candidate Recall@K
×
Selector Precision / Accuracy
```

不能只看 Selector 在“已保证 true match 存在”的候选集里表现多好。

对于 precision-first 业务还要额外加入：

```text
NO_MATCH candidate set
```

即很多 query 的 top-K 中根本不存在正确 reference；系统必须允许一个都不选。

---

### 2.2 数据表示：每个 query 对应一个候选列表与 binary vector

官方 recompiled dataset 不只保存 pairwise row，而是把一条 query 表示为：

```text
text_src: query entity
candidates: [c1, c2, ... c10]
labels:     [0, 0, 1, 0, ... 0]
```

训练时：

- 保留 positive candidates；
- 从 negatives 中采样 `num_neg`；
- positive + negatives 混在一起 shuffle；
- 同一候选集一次进入模型。

官方默认：

```text
num_neg = 9
K = 10
```

这比把 9 个 hard negatives 拆成 9 个互不相干的训练 pair 更适合“相邻 reference 必须互斥”的业务。

---

### 2.3 Siamese 编码：query 与 candidates 共用一套 Mistral embedding 权重

`modelling_llama.py` 中的 `EntityRetrieverMistral` 是核心。

Query：

```python
query_hidden = Mistral(query)
q = last_token_embedding(query_hidden)
q = normalize(q)
```

Candidates：

```python
candidate_hidden = Mistral(candidates)
c = last_token_embedding(candidate_hidden)
c = normalize(c)
```

再计算：

```python
similarities = q @ c.T
```

因此结构是标准 shared-weight Siamese / bi-encoder：

```text
                  ┌────────────────────┐
query ───────────►│                    │──► q
                  │ Shared Mistral     │
                   │ Embedding Encoder │
                  │                    │──► c1
candidate_1 ─────►│                    │
                  │                    │──► c2
candidate_2 ─────►│                    │
                  └────────────────────┘

score_i = cosine(q, c_i)
```

相比 cross-encoder，它有两个生产优势：

1. candidate/reference embedding 可以离线预计算；
2. query 与 K 个候选不是 K 次完整 cross-encoder 推理，扩展性更好。

论文的 ablation 也显示，保持 embedding backbone 原本的 Siamese 结构，比改造成 cross-encoder 更稳定。

---

## 3. CMRL：为什么它比普通 CE/InfoNCE 更适合 reference hard negatives

官方实现把总 loss 写成：

```text
L = (1 - α) * L_CE + α * L_CL
```

### 3.1 第一部分：listwise Cross Entropy

先得到同一候选集的 similarity vector：

```text
[s1, s2, ..., sK]
```

然后用真正的 positive position 做 Cross Entropy。

作用：

> 让模型学会“这 K 个候选里谁应该是第一名”。

### 3.2 第二部分：Hard-Negative Margin Ranking

代码对每个 query：

1. 取所有 negative scores；
2. 选 score 最高的 top-H hard negatives；
3. 对每个 hard negative 与 positive 计算：

```text
d = s_neg - s_pos + margin
```

如果：

```text
s_pos >= s_neg + margin
```

则 `relu(d)=0`，不再处罚。

如果 hard negative 太靠近 positive，则产生 loss。

这与腕表相邻 reference 的目标完全一致：

```text
126610LN score = 0.91  <- true
126610LV score = 0.90  <- hard negative
```

普通 CE 只要求 true 排第一；CMRL 还要求它与最危险 negative 拉开一个安全 margin。

### 3.3 官方实现还对 hard negatives 做 soft weighting

代码中对 margin violation 做：

```python
exp_differences = exp(differences)
softmax_weights = exp_differences / sum(exp_differences)
loss_cl += sum(softmax_weights * relu(differences))
```

即越接近甚至超过 positive 的 negative，权重越大。

本质上就是：

> **训练预算优先花在最可能制造 false positive 的近邻上。**

这比随机负采样非常适合二奢型号。

### 3.4 论文超参数与仓库默认值存在一个值得注意的差异

论文 ablation 报告最佳附近是：

```text
α = 0.6
margin = 0.5
H = 3
```

而仓库 `EM_training_contrastive.py` 当前示例默认：

```text
alpha = 0.6
margin = 0.5
top_k = 1
```

因此如果复现实验，不要只照抄当前 repo example；`H` 应独立验证，尤其我们的 hard negatives 比普通电商 benchmark 更密集。

腕表建议至少比较：

```text
H ∈ {1, 3, 5}
```

并以**高精度 acceptance 区间**而不是总 F1/MRRP 选参数。

---

## 4. 参数高效训练：4-bit + LoRA

官方 `model_finetuner.py` 使用：

```text
prepare_model_for_kbit_training
PEFT / LoRA
gradient checkpointing
paged_adamw_32bit
fp16
```

示例参数：

```text
LoRA r = 64
LoRA alpha = 64
target_modules = all-linear
learning_rate = 2e-4
batch_size = 32
gradient_accumulation_steps = 8
train_epochs = 10
```

这意味着不需要全参数训练 7B embedding model，可以用相对有限的可训练参数适配业务 hard cases。

对当前需求尤其有价值，因为用户只愿意提供“几百对黄金标签”。

但我的建议是：

> **不要一开始就用几百条人工标签训练。先从规则能确定的 reference exact-match 数据自动产生高置信 pseudo-label，再把人工标签集中用在 hard negative / ambiguous reference 上。**

例如：

```text
Positive:
品牌一致 + canonical reference 完全一致 + reference 来源可信

Hard Negative:
品牌一致 + 系列一致 + reference 高字符串相似但 canonical reference 不同
```

这样几百条人工预算可以专门覆盖：

- `O/0`、`I/1` OCR 混淆；
- 后缀 LN/LV/BLNR/BLRO；
- 老款/新款相邻 reference；
- 标题同时出现多个 reference；
- “适配某型号”的表带/配件文本；
- 平台 SKU 被误认成 reference；
- reference 截断或缺分隔符。

---

## 5. 论文效果值得参考，但不能直接作为业务安全指标

论文在 recompiled hard-negative datasets 上报告 Mistral4SelectEM 优于 pairwise matcher 和 reranker；文章结果中 Mistral4SelectEM 在四个 recompiled 数据集上的 MRRP 约为：

```text
rAB: 0.971
rWA: 0.919
rAG: 0.886
rAE: 0.723
```

并报告平均相对提升：

```text
+9.6%  vs baseline embedding model
+12.4% vs best pairwise matcher
+6.7%  vs best reranker
```

单 query 推理时间约 `1.2s`（论文实验环境）。

这些结果说明 listwise hard-negative training 的方向有效，但对当前 Spec 不能直接用 MRRP 做上线标准。

当前业务必须把核心指标改成：

```text
1. AutoMatch Precision
2. False Merge Rate
3. Coverage / AutoMatch Rate
4. Abstain Rate
5. Candidate Recall@K
6. Reference Extraction Precision
```

排序指标只能是辅助。

业务真正要优化的是：

> 在 precision 达到极高目标的前提下，最大化 coverage。

而不是：

> 最大化所有样本平均排名/F1。

---

## 6. 原方案与当前 Spec 的 5 个关键冲突

### 6.1 冲突一：论文 selector 默认“必须选一个”

核心推理代码：

```python
predicted_labels = torch.argmax(softmax_similarities, dim=1)
```

这与“允许漏匹配”冲突。

当前业务需要：

```text
MATCH(reference_id)
ABSTAIN
CONFLICT
```

而不是单纯 argmax。

### 6.2 冲突二：论文把 generic entity similarity 当主要语义

当前需求已经明确：

```text
same entity <=> same reference number
```

因此 reference 不是普通 feature，而是**实体主键级证据**。

`品牌、系列、尺寸、材质、图片外观` 都只能辅助 reference 识别与冲突检测，不能在 reference 不同的情况下通过相似度“投票翻盘”。

### 6.3 冲突三：实验会强制把 gold truth 放进 top-K

前面提到，代码会把 blocker miss 的 truth 填到最后一位。

生产中必须允许：

```text
Top-K 中没有正确答案
```

并让模型选择 `NO_MATCH` / abstain。

### 6.4 冲突四：训练数据几乎都假设有 positive

当前真实数据会大量出现：

- reference 缺失；
- reference 无法解析；
- 新型号不在 reference catalog；
- 标题里的编号其实是 SKU；
- 候选库没有该型号。

如果训练时缺少 no-positive list，模型会学到“总有一个最像的就是答案”。这正是 false positive 来源。

### 6.5 冲突五：MRRP 不是 precision guarantee

论文新增 MRRP 对 false positive / missed detection 做 penalty，但它仍是整体排序质量指标。

业务 acceptance 必须单独校准阈值，并在 hard-negative holdout 上统计 precision/false merge；模型 score 不能天然当概率使用。

---

# 7. 面向当前需求的直接落地方案：RefSelectEM

我建议把论文方法改成 **RefSelectEM（Reference-First Selective Entity Matching）**。

核心思想：

> 不直接判断“雷小安商品 A 是否等于腕表之家商品 B”；先分别判断 A、B 属于哪个 Canonical Reference Entity。只要两个 record 都以可审计证据链接到同一个 reference entity，它们天然同款。

这样从：

```text
record × record
```

变为：

```text
record -> reference_entity
```

对于百万到千万数据更容易扩展，也与业务定义完全一致。

---

## 8. 数据模型

### 8.1 `product_record`

```sql
product_record(
  record_id,
  source,              -- leixiaoan / xcar_watch / shedangjia
  source_item_id,
  brand_raw,
  title_raw,
  reference_raw,
  description_raw,
  image_urls,
  updated_at,
  payload_hash
)
```

### 8.2 `reference_observation`

不要只保存最终抽取值，要保存每一个“观察证据”：

```sql
reference_observation(
  observation_id,
  record_id,
  channel,             -- structured / title / description / image_ocr
  raw_value,
  normalized_value,
  span_or_bbox,
  extractor_version,
  confidence,
  role,                -- BRAND_REFERENCE / PLATFORM_SKU / COMPATIBLE_MODEL / UNKNOWN
  evidence_uri
)
```

`role` 极其重要。

例如标题：

```text
适用劳力士 126610LN 的第三方表带 SKU-RX-8891
```

如果只抽数字串，会同时看到：

```text
126610LN
RX-8891
```

但这个商品本身并不是 126610LN 手表。

因此必须先做 **identifier role classification**，不能“看到像 reference 的字符串就当本品型号”。

### 8.3 `reference_entity`

```sql
reference_entity(
  reference_entity_id,
  brand_id,
  canonical_reference,
  series,
  model_name,
  status,
  catalog_version
)
```

唯一约束建议：

```text
UNIQUE(brand_id, canonical_reference)
```

不要只对 `canonical_reference` 全局唯一，因为不同品牌可能重复编号格式。

### 8.4 `record_reference_link`

```sql
record_reference_link(
  record_id,
  reference_entity_id,
  decision,            -- VERIFIED / REVIEW / REJECTED
  decision_score,
  score_margin,
  evidence_level,
  model_version,
  rule_version,
  created_at
)
```

所有自动合并都必须可追溯到：

```text
哪条 observation
哪版 canonicalizer
哪版候选库
哪版模型
哪组阈值
```

---

## 9. Reference 规范化：必须“保守”，不能为了召回把不同型号洗成一样

典型规范化可以做：

```text
Unicode NFKC
trim
uppercase
统一全角/半角
统一明确等价的分隔符
品牌别名字典映射
```

但不要无脑做：

```text
删除所有字母
删除所有后缀
只保留数字
模糊 edit distance 后直接视为相同
```

例如：

```text
126610LN != 126610LV
126710BLNR != 126710BLRO
```

这些后缀就是 reference 身份的一部分。

建议 canonicalizer 输出的不只是字符串，而是：

```json
{
  "brand": "ROLEX",
  "raw": "126610 LN",
  "canonical": "126610LN",
  "transformations": ["remove_allowed_space"],
  "lossy": false
}
```

任何 `lossy=true` 的规范化结果都不能独立触发自动 VERIFIED。

---

## 10. 候选生成：Reference Entity Top-K，而不是全量商品 Top-K

### 10.1 强路径：exact reference index

如果得到高置信 canonical reference：

```text
(brand_id, canonical_reference)
```

直接查唯一索引：

```sql
SELECT reference_entity_id
FROM reference_entity
WHERE brand_id = ?
  AND canonical_reference = ?
```

命中唯一实体且无冲突时，不需要 LLM。

这是最快、最安全、最应该覆盖大多数自动匹配流量的路径。

### 10.2 弱路径：模糊候选只用于消歧，不用于直接判同

reference 不完整时，例如：

```text
126610L?
```

先在同品牌内找候选：

```text
126610LN
126610LV
```

候选生成可以结合：

- reference prefix/trigram；
- edit distance；
- 系列；
- 标题 embedding；
- OCR token；
- 图像属性。

然后把这组候选 reference entity 交给 SelectEM。

注意：

> 模糊检索只是“谁值得比较”，不是“谁可以自动合并”。

---

# 11. 把 CMRL 改造成腕表专用 hard-negative learner

训练样本单位：

```text
query record
+
K 个 canonical reference candidates
```

### 11.1 Positive

只使用高置信来源，例如：

```text
结构化 reference 字段
+ 品牌一致
+ catalog 唯一命中
+ 无其他冲突 observation
```

### 11.2 Hard Negative Mining

不要随机抽品牌外负样本，应该主动生成最危险样本：

```text
同品牌 + 同系列
同 prefix
Levenshtein distance 小
仅 suffix 不同
数字位数相同
图片高度相似
标题 embedding 高相似
```

例：

```text
Positive: 126610LN
Negatives:
  126610LV
  116610LN
  124060
  126613LN
```

这正是 CMRL 的优势场景。

### 11.3 加入 `NO_MATCH` sentinel

论文原始实现最重要的业务改造之一：

候选列表增加一个显式项：

```text
__NO_MATCH__
```

训练数据必须包含三类：

```text
A. 正确 reference 在 top-K
B. 正确 reference 不在 top-K
C. query 本身没有足够证据确认 reference
```

B/C 的 label 都指向 `NO_MATCH`。

但生产最终仍不应只依赖 `NO_MATCH` score；hard gate 负责最后放行。

---

## 12. Precision-first 的最终 Decision Gate

建议模型输出：

```text
s1 = top1 reference score
s2 = top2 reference score
margin = s1 - s2
```

同时计算 evidence：

```text
structured_ref_match
text_ref_match
ocr_ref_match
brand_consistency
series_consistency
conflict_count
identifier_role
catalog_unique
```

最终自动 VERIFIED 必须满足类似：

```python
def can_auto_verify(x):
    if not x.catalog_unique:
        return False

    if x.identifier_role != "BRAND_REFERENCE":
        return False

    if x.strong_reference_conflict:
        return False

    if x.canonical_reference != x.top1_reference:
        return False

    if x.model_score < T_SCORE:
        return False

    if x.top1_score - x.top2_score < T_MARGIN:
        return False

    if x.lossy_normalization and x.independent_evidence_count < 2:
        return False

    return True
```

这里的核心不是具体阈值，而是**模型分数永远不是唯一条件**。

推荐状态机：

```text
VERIFIED
  只有强证据、唯一 reference、无冲突

REVIEW
  有合理候选，但证据不足/多候选接近

REJECTED / CONFLICT
  存在强矛盾 reference

UNRESOLVED
  没有可靠 reference
```

如果 precision 真的是“绝不能误匹配”，宁可大量停在 REVIEW/UNRESOLVED。

---

# 13. 图片应该怎么用：只做 reference evidence，不做“长得像就是同款”

当前数据有图片，因此可加：

```text
图片
 -> OCR
 -> reference token candidate
```

高价值区域：

- 表背刻字；
- 保卡；
- 吊牌；
- 证书；
- 商品说明牌；
- 表盘局部型号相关文字。

建议把 OCR 结果也写入 `reference_observation`：

```text
channel = image_ocr
bbox = ...
raw_value = ...
```

图片视觉 embedding 可以帮助：

```text
候选召回
冲突检测
人工 review 排序
```

但不能用：

```text
图片很像 -> 覆盖 reference 不一致 -> 自动匹配
```

因为同系列相邻 reference 正是视觉最相似的 hard negatives。

---

# 14. 增量架构：适配 100 万–1000 万记录持续更新

推荐服务拆分：

```text
                ┌─────────────────────┐
Crawler/Event ─►│ Record Ingestion    │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Ref Extractor       │
                │ field/title/OCR     │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Ref Canonicalizer   │
                └──────────┬──────────┘
                           │
             ┌─────────────┴──────────────┐
             │                            │
             ▼                            ▼
    Exact Reference Index        Candidate Retriever
             │                   (only ambiguous)
             │                            │
             │                            ▼
             │                     RefSelectEM CMRL
             │                            │
             └─────────────┬──────────────┘
                           ▼
                   Decision Gate
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          VERIFIED       REVIEW       UNRESOLVED
             │
             ▼
       Reference Entity Group
```

### 14.1 存储建议

核心关系和唯一约束放关系数据库即可：

```text
PostgreSQL
```

百万到千万级本身并不要求为了规模放弃关系约束。

日志/离线分析可放：

```text
ClickHouse / object storage
```

候选 reference dictionary 如果量级只是几万/几十万，完全没必要为每条记录做 7B 向量全库搜索；先按 brand/series/ref prefix 缩小集合。

只有标题/描述语义 fallback 才走 ANN：

```text
FAISS / HNSW / pgvector
```

### 14.2 增量更新必须幂等

`product_record` 使用：

```text
(source, source_item_id, payload_hash)
```

控制重复处理。

任何 extractor/model/canonicalizer 升级后，不直接静默覆盖旧结果，而是保留 version，支持：

```text
重跑
回溯
误匹配审计
规则回滚
```

---

# 15. 几百条黄金标签应该怎么花

不要平均随机标 500 对。

建议先用规则生成大量 easy pseudo-label，再让人工只标高价值候选集。

优先级：

```text
1. top1/top2 score margin 最小
2. reference 编辑距离最小但不同
3. OCR 与 title 冲突
4. 结构化 ref 与 title ref 冲突
5. 同标题出现多个 reference
6. identifier role 不确定
7. 新品牌/新模板/新来源
8. 模型高置信但规则拒绝的样本
```

标注单元也不要只标 pair：

推荐让标注员看到：

```text
query record
+ top-K reference candidates
+ 每条 reference evidence
+ 图片/OCR crop
```

最终标：

```text
reference_entity_id
或 NO_MATCH / UNKNOWN
```

这样标签天然适配 listwise training。

---

# 16. 测试集必须专门制造“差一个字符就错”的场景

随机 train/test split 很容易高估效果。

建议建立 `LuxuryRef-Hard` 专用测试集，至少包含：

### 16.1 相邻 reference

```text
126610LN vs 126610LV
```

### 16.2 新旧代

```text
116610LN vs 126610LN
```

### 16.3 OCR confusion

```text
O / 0
I / 1
B / 8
S / 5
```

### 16.4 分隔符与空格

```text
AB-123
AB123
AB 123
```

只有品牌 catalog 明确等价时才能 canonicalize。

### 16.5 标题里的 compatible model

```text
表带/配件适配 126610LN
```

应判当前商品 reference unknown，而不是 126610LN。

### 16.6 平台 SKU

```text
平台货号：126610LN-like 字符串
```

必须先识别编号角色。

### 16.7 多 reference 冲突

```text
structured = 126610LN
title = 126610LV
OCR = 126610LN
```

不能简单 majority vote，应该进入 conflict policy。

### 16.8 No-match candidate set

正确 reference 不存在于 catalog/top-K 时必须 abstain。

---

# 17. 上线指标：不要再用普通 F1 作为主指标

建议 dashboard：

```text
CandidateRecall@K
ReferenceExtractionPrecision
AutoMatchPrecision
AutoMatchCoverage
FalseMergeCount
ConflictRate
ReviewRate
UnknownRate
Top1-Top2 Margin Distribution
By-brand Precision
By-source Precision
By-extractor-version Precision
```

其中上线 gate 最重要：

```text
AutoMatchPrecision
```

并且要单独看 hard-negative slice：

```text
same-brand
same-series
near-reference
OCR-derived
multi-reference conflict
```

如果总体 precision 很高、但 `near-reference` slice 出现 false positive，仍不应该放开自动匹配。

---

# 18. 可直接实现的 MVP

不需要一开始就训练 Mistral4SelectEM。

第一版可以先落以下链路：

```text
1. 建 reference_entity 表
2. 做品牌感知 canonicalizer
3. 从结构化字段/title 抽 reference observations
4. 区分 BRAND_REFERENCE 与 SKU/COMPATIBLE_MODEL
5. (brand, canonical_reference) exact link
6. 冲突一律 REVIEW
7. 建 hard-negative 数据集
8. 再训练 RefSelectEM 处理 ambiguous cases
```

这会比一上来做“大模型判断两个商品是否相同”更快得到高 precision baseline。

### 伪代码

```python
def link_record(record):
    observations = extract_reference_observations(record)
    observations = classify_identifier_roles(observations)

    refs = [
        conservative_canonicalize(x)
        for x in observations
        if x.role == "BRAND_REFERENCE"
    ]

    strong_refs = get_strong_refs(refs)

    # 强冲突：不自动链接
    if has_reference_conflict(strong_refs):
        return Decision("REVIEW", reason="REFERENCE_CONFLICT")

    # 强 exact path
    exact = lookup_unique_reference_entity(record.brand, strong_refs)
    if exact and exact.is_unambiguous:
        return Decision(
            "VERIFIED",
            reference_entity_id=exact.id,
            reason="EXACT_REFERENCE"
        )

    # 无候选/证据太弱，直接 abstain
    candidates = retrieve_reference_candidates(record, refs, k=10)
    if not candidates:
        return Decision("UNRESOLVED", reason="NO_CANDIDATE")

    ranked = ref_select_em(record, candidates)

    # 模型不能越过硬规则
    if ranked.top1.id == "__NO_MATCH__":
        return Decision("UNRESOLVED", reason="MODEL_NO_MATCH")

    if ranked.top1.score < T_SCORE:
        return Decision("REVIEW", reason="LOW_SCORE")

    if ranked.top1.score - ranked.top2.score < T_MARGIN:
        return Decision("REVIEW", reason="LOW_MARGIN")

    if not evidence_supports_reference(record, ranked.top1):
        return Decision("REVIEW", reason="NO_HARD_EVIDENCE")

    if has_conflict_with_candidate(record, ranked.top1):
        return Decision("REVIEW", reason="CANDIDATE_CONFLICT")

    return Decision(
        "VERIFIED",
        reference_entity_id=ranked.top1.id,
        reason="SELECTIVE_REFERENCE_LINK"
    )
```

---

# 19. 与我之前几个调研结果如何拼起来

这篇论文不是孤立方案，正好补上之前几个方向之间的空档：

```text
DeepBlocker
  -> 大规模 candidate/blocking 思路

Ameli
  -> 多模态 + 细粒度属性辅助 entity linking

本篇 Mistral4SelectEM / CMRL
  -> 在 hard-negative 候选集中进行 listwise reference 消歧

TransClean
  -> 链接之后做跨源图一致性审计与 false-positive 清洗
```

如果组合成最终系统：

```text
Observation Extraction
      │
      ▼
Reference Candidate Retrieval
      │
      ▼
RefSelectEM (CMRL hard-negative ranking)
      │
      ▼
Hard Precision Gate / Abstain
      │
      ▼
Reference Entity Linking
      │
      ▼
TransClean-style group consistency audit
```

其中最重要的控制关系是：

> **embedding / LLM 永远位于 reference 规则之下，而不是 reference 规则之上。**

---

# 20. 最终建议

这篇论文值得直接吸收的有四点：

1. **候选集级 listwise selection**，不要把相邻 reference 独立 pairwise 判断；
2. **hard-negative mining**，训练重点放在同系列相邻型号；
3. **CMRL margin**，不仅要求 true top-1，还要求与最危险 negative 拉开安全距离；
4. **Siamese embedding + LoRA**，适合可扩展候选检索与有限标注适配。

但必须修改的也有四点：

1. 从“必须 argmax 一个候选”改成 `MATCH / ABSTAIN / CONFLICT`；
2. 从“record-to-record generic EM”改成“record-to-canonical-reference entity linking”；
3. 训练与评测必须包含 **正确答案不在 top-K** 的 no-match 场景，不能像官方数据转换脚本一样强制插入 gold truth；
4. 最终自动 VERIFIED 必须由 **reference 硬证据 + 无冲突 + 模型 margin 阈值**共同决定，模型只能辅助，不能越权。

如果只做一个最小但安全的落地版本，我会优先实现：

```text
Canonical Reference Registry
+ Reference Observation
+ Identifier Role Classification
+ Exact Reference Link
+ Conflict Abstention
+ Hard-negative Dataset
```

然后再把 Mistral4SelectEM 的 CMRL 训练方式加入 ambiguous reference resolver。

这个顺序最符合当前 Spec 的真实目标：

> **不是尽可能多地“猜中同款”，而是在证据不足时坚决不匹配，只自动提交可解释、可审计、可回滚的同 reference 链接。**
