# How to Fix a Broken Confidence Estimator: Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks

## 0. 结论先行

本文分析 Cattelan & Silva 在 UAI 2024 的论文 **How to Fix a Broken Confidence Estimator: Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks** 及其官方实现 `lfpc/FixSelectiveClassification`，并针对 Notion 需求「跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）」给出可直接落地的架构。

这篇论文最值得借鉴的不是“再训练一个更大的 matcher”，而是把已有模型输出的 logits 重新构造成一个更适合 **selective classification / 可拒识分类** 的置信度排序，从而把系统拆成：

1. 模型负责产生候选判断；
2. 置信度估计器负责判断“这次判断是否值得信任”；
3. policy gate 只自动接受极少数高置信结果，其余一律拒识/人工复核。

这与本需求“**precision 优先到极致，允许漏匹配**”高度一致。

但有一个非常关键的工程结论：**论文的 MaxLogit-pNorm 不能直接套到二分类 pairwise matcher 上。** 对二分类 logits 做中心化后，p-norm 最大值会退化为常数，失去排序能力。正确落地方式应把“两个商品是否相同”的二分类改造成“当前商品属于哪个 canonical reference 候选，或 NONE”的**多候选选择问题**，再用 MaxLogit-pNorm 做拒识置信度。

最终建议采用：

> **硬 reference 规则收口 + 多候选 reference classifier + MaxLogit-pNorm 后处理 + 高精度阈值门控 + 人工复核闭环**

其中图片/OCR/LLM 只能提供“reference 属于当前商品”的辅助证据，不能越过 reference 硬规则直接合并商品。

---

## 1. 为什么选这篇

### 1.1 与当前 Spec 的直接对应

Notion Spec 的约束是：

- 100 万～1000 万级多源商品，持续增量；
- “同一个商品”严格定义为 **同一 reference number / 型号**；
- 字段稀疏，reference 可能在结构化字段，也可能埋在标题中；
- 有图片；
- **绝对不能误匹配，允许漏匹配**；
- 可以人工标注几百对黄金样本。

这不是一个追求总体 F1 的普通 entity matching 问题，而是典型的 selective decision：

- 高置信度 → 自动接受；
- 中低置信度 → 不做决定；
- precision 通过牺牲 coverage 保证。

论文专门研究“模型预测可以不变，只替换 confidence estimator，就能显著改善 selective classification 的 risk-coverage 排序”，非常适合作为整个匹配系统的最后一道风险闸门。

### 1.2 本次排除项

`奢侈品调研/b` 已经分析过的内容包括：

- Ameli
- AnyMatch
- Confidence Classifiers with Guaranteed Accuracy or Precision
- DeepBlocker
- Ditto
- End-to-end multi-modal product matching in fashion e-commerce
- Entity Matching with 7B LLMs
- Fine-tuning large language models with contrastive margin ranking loss...
- GraLMatch
- LATEX-Numeric
- Large Scale Generative Multimodal Attribute Extraction...
- Multi-Value-Product Retrieval-Augmented Generation...
- Progressive Fine-Tuning...
- Tailoring entity resolution for matching product offers
- TransClean
- Using LLMs for the Extraction and Normalization of Product Attribute Values
- parts-distributor-sku-classifier
- pyJedAI

本次论文不在已有分析中。

---

## 2. 原论文方法与官方实现拆解

论文/项目：

- PMLR: https://proceedings.mlr.press/v244/cattelan24a.html
- 官方代码: https://github.com/lfpc/FixSelectiveClassification

### 2.1 Selective Classification 的核心

普通分类器对每条样本都输出类别：

\[
\hat y = \arg\max_k z_k
\]

Selective Classification 多一个动作：**abstain（拒识）**。

给定 confidence function \(g(x)\)，只在：

\[
g(x) \ge \tau
\]

时输出自动判断，否则拒绝。

因此一个好的 confidence estimator 不一定要改变分类准确率，但必须把“容易正确的样本”排在“容易错误的样本”前面。

对本项目来说：

- classifier：判断最可能的 canonical reference；
- confidence estimator：判断这次 reference 选择是否可信；
- threshold：决定是否允许自动写入 `matched_reference_id`。

### 2.2 论文提出的 MaxLogit-pNorm

官方 README 给出的定义为：

\[
\text{MaxLogit-pNorm}(\mathbf{z})
=
\max_k
\frac{z_k-\mu(\mathbf{z})}
{\|\mathbf{z}-\mu(\mathbf{z})\|_p}
\]

步骤：

1. logits 减去均值；
2. 对中心化后的 logits 做 p-norm；
3. 用归一化后的最大 logit 作为 confidence。

官方 `post_hoc.py` 实现基本就是：

```python
def centralize(logits):
    return logits - logits.mean(-1).view(-1, 1)


def normalize(logits, p, centralize_logits=True):
    if centralize_logits:
        logits = centralize(logits)
    return torch.nn.functional.normalize(logits, p, -1)


def MaxLogit_pNorm(logits, p='optimal', centralize_logits=True, **kwargs):
    if centralize_logits:
        logits = centralize(logits)
    if p == 'optimal':
        p = optimize.p(...)
    if p == 'MSP':
        return MSP(logits)
    return max_logit(normalize(logits, p, False))
```

`p` 通过验证集 grid search 优化，官方代码还提供 `MSP_fallback`：如果 pNorm 不能优于 Maximum Softmax Probability，则退回 MSP。

这个设计非常适合线上系统，因为它：

- 不要求重训主模型；
- 只依赖 logits；
- 可以针对新的来源/品牌单独重新校准；
- 适合在分布漂移后快速更新置信度层。

### 2.3 Risk-Coverage

官方实现把样本按 confidence 从高到低排序，再计算不同 coverage 下累计风险：

```python
confidence, indices = confidence.sort(descending=True)
risk = risk[indices]
coverages = (1 + indices) / n
risks = risk.cumsum(0)[indices] / n
risks /= coverages
```

然后用 AURC（Area Under Risk-Coverage Curve）衡量排序质量。

对当前业务，不应该只优化整体 AURC，而应该额外计算：

- `Precision@Coverage`
- `Coverage@Precision>=target`
- 高风险品牌/来源上的 `FP count`
- hard-negative 子集上的 `FP count`

因为业务损失并不对称：false positive 的成本远高于 false negative。

---

## 3. 一个容易踩坑但非常重要的数学问题：二分类会退化

假设 pairwise matcher 输出两个 logits：

\[
\mathbf z=(a,b)
\]

中心化后：

\[
\mathbf z'=(\frac{a-b}{2},\frac{b-a}{2})=(d/2,-d/2)
\]

其 p-norm 为：

\[
\|\mathbf z'\|_p
=
(2(|d|/2)^p)^{1/p}
=
|d|\cdot 2^{1/p-1}
\]

因此最大归一化 logit 为：

\[
\frac{|d|/2}{|d|2^{1/p-1}}=2^{-1/p}
\]

只要不是完全平局，它就是常数。

**结论：如果把当前任务建模成 `[match, non-match]` 二分类，MaxLogit-pNorm 不能提供有意义的 confidence ranking。**

这也是为什么本方案建议从 pairwise binary EM 改成：

> **给一个商品 record，选择一个 canonical reference 候选，或选择 NONE。**

即：

\[
\{r_1,r_2,...,r_K, NONE\}
\]

至少三类，pNorm 才有空间表达“第一候选相对整个候选集合到底突出多少”。

对于腕表，这种建模反而更自然：真正危险的错误往往不是“完全不相关商品”，而是同品牌、同系列、只有一两个字符/后缀不同的 reference hard negatives。

---

## 4. 推荐的生产架构

```text
雷小安 / 腕表之家 / 奢当家
          │
          ▼
┌──────────────────────┐
│ 1. Raw Ingestion      │
│ 原始字段、HTML、图片   │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 2. Canonicalization   │
│ 品牌、文本、编号角色   │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 3. Reference Extract  │
│ structured/title/OCR  │
│ 多路抽取 + provenance │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 4. Candidate Retrieval│
│ brand内召回 K 个ref   │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 5. Hard Conflict Gate │
│ 冲突直接拒绝          │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 6. Candidate Matcher  │
│ K refs + NONE logits  │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 7. pNorm Confidence   │
│ post-hoc recalibration│
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 8. Precision Gate     │
│ ACCEPT/REVIEW/REJECT  │
└─────────┬────────────┘
          ▼
┌──────────────────────┐
│ 9. Canonical Ref Link │
│ 跨源商品按ref关联     │
└──────────────────────┘
```

核心原则：**模型不能直接产生“两个商品被合并”这个副作用。** 模型只产生候选 reference 和 confidence，最终合并必须经过 deterministic policy。

---

## 5. 数据模型：不要直接存 pairwise match

### 5.1 `product_record`

```sql
product_record(
  id bigint primary key,
  source varchar,          -- leixiaoan/watchclub/luxury...
  source_item_id varchar,
  brand_raw varchar,
  brand_id bigint,
  title text,
  attrs jsonb,
  image_urls jsonb,
  crawl_time timestamp,
  content_hash varchar,
  version int
)
```

### 5.2 `reference_evidence`

每个抽取到的编号都保留来源，不要只保留最终字符串。

```sql
reference_evidence(
  id bigint,
  product_id bigint,
  surface varchar,
  canonical_lossless varchar,
  retrieval_skeleton varchar,
  role varchar,            -- brand_reference / platform_sku / seller_sku / compatible_ref
  channel varchar,         -- structured / title / description / ocr / llm
  span_start int,
  span_end int,
  image_id varchar,
  extractor_version varchar,
  confidence float
)
```

### 5.3 `canonical_reference`

```sql
canonical_reference(
  id bigint primary key,
  brand_id bigint,
  reference_lossless varchar,
  family varchar,
  aliases jsonb,
  status varchar,
  unique(brand_id, reference_lossless)
)
```

### 5.4 `match_decision`

```sql
match_decision(
  product_id bigint,
  canonical_reference_id bigint,
  decision varchar,        -- ACCEPT / REVIEW / REJECT
  matcher_version varchar,
  confidence_method varchar,
  confidence float,
  p_value float,
  threshold_version varchar,
  hard_rule_trace jsonb,
  evidence_ids jsonb,
  decided_at timestamp
)
```

必须保存 `hard_rule_trace` 和模型版本，未来出现一次 false positive 时才能追溯“到底是抽取错、规范化错、候选错、模型错还是阈值错”。

---

## 6. Reference 规范化：必须区分“召回字符串”和“判同字符串”

这是高 precision 系统最重要的工程细节之一。

不能做一个 aggressive normalize 后直接 exact match，例如：

```text
15500ST.OO.1220ST.01
15500ST.OO.1220ST.03
```

只差后缀，但可能就是不同 reference。

建议生成两个版本。

### 6.1 `canonical_lossless`

用于最终判同，只做**确定不会改变语义**的规范化：

- Unicode NFKC；
- 英文字母统一大写；
- 全角转半角；
- trim；
- 已验证等价的分隔符规则；
- 品牌级白名单规则。

不能随便：

- 删除所有点号；
- 删除所有连字符；
- 删除结尾颜色/材质后缀；
- 把相似字符 O/0、I/1 无条件互换。

### 6.2 `retrieval_skeleton`

只用于召回，可更激进：

- 去空格；
- 统一分隔符；
- 字母数字 token 化；
- OCR confusion 候选展开；
- trigram / edit-distance key。

**召回可以模糊，自动接受不能模糊。**

---

## 7. Reference 抽取器设计

不要用单个 LLM prompt 直接抽 reference。推荐多路独立证据：

### 7.1 路 1：结构化字段

如果来源有：

- 型号
- 参考编号
- reference
- model number

优先级最高。

但仍需做编号角色分类，排除平台 SKU、商家货号、库存号。

### 7.2 路 2：标题/描述规则抽取

品牌级 pattern registry：

```python
PATTERNS = {
    "rolex": [...],
    "omega": [...],
    "ap": [...],
    "patek": [...],
}
```

规则只负责产生候选，不直接判定。

### 7.3 路 3：轻量序列标注模型

使用 BERT/DeBERTa token classification 标注：

- `B-REFERENCE`
- `I-REFERENCE`
- `B-PLATFORM_SKU`
- `B-COMPATIBLE_REFERENCE`

这里“compatible reference”非常重要，例如表带、盒证、配件标题可能写着适配某型号，但当前商品并不是那只表。

### 7.4 路 4：图片 OCR

优先 OCR：

- 保卡；
- 吊牌；
- 表背；
- 证书；
- 包装标签。

图片视觉 embedding 只用于辅助，不允许根据“长得很像”直接推断同 reference。

### 7.5 冲突规则

如果不同高可信通道出现不同 reference：

```text
structured = 126610LN
title      = 126610LV
```

直接 `REVIEW`，而不是让模型投票。

因为在 precision-first 系统里，**冲突本身就是拒识证据**。

---

## 8. Candidate Retrieval：从千万 pair 降到固定 K

直接三源笛卡尔积不可行。

推荐候选召回顺序：

1. `brand_id` 硬 blocking；
2. exact `canonical_lossless`；
3. `retrieval_skeleton`；
4. brand 内 trigram / edit distance；
5. series/family filter；
6. text/OCR embedding 仅补充召回。

最终给 matcher 的是：

```text
record X
  -> ref_1
  -> ref_2
  -> ...
  -> ref_K
  -> NONE
```

建议生产上固定 `K=16` 或 `K=32`。

固定 K 的理由不仅是推理方便，还因为 pNorm confidence 与类别维数有关。如果不同请求的候选数差别很大，confidence 分布会漂移。

如果某品牌真实候选不足 K：

- 可以 mask padding；
- pNorm 的均值和 norm 必须只在 valid logits 上计算；
- 阈值至少按 `valid_candidate_count bucket` 分桶校准。

不要简单把 padding 设成 `-inf` 后直接调用原始 `torch.normalize`。

---

## 9. Candidate Matcher：模型只做“reference 消歧”

### 9.1 输入特征

对每个 `(record, candidate_reference)` 构造：

```text
reference_exact_structured
reference_exact_title
reference_exact_ocr
reference_edit_distance
reference_role_score
brand_match
family_match
series_match
text_cross_encoder_score
ocr_cross_encoder_score
image_attribute_score
conflict_count
source_pair
```

### 9.2 推荐模型

第一版不需要大模型，可使用：

- LightGBM/XGBoost：可解释、训练快；或
- 小型 MLP：输出 candidate logit；或
- cross-encoder：只在 hard cases 使用。

对每个候选得到一个 `candidate_logit`，额外训练一个 `NONE_logit`，然后 softmax/argmax 选候选。

### 9.3 为什么不要让图片模型成为主模型

腕表同系列不同 reference 可能外观几乎一样。

因此视觉相似度适合：

- 排除明显不一致；
- OCR；
- 补充材质/盘面/颜色属性；

不适合：

- “视觉很像，所以 reference 一样”。

最终 `ACCEPT` 必须回到 reference 硬证据。

---

## 10. pNorm Confidence 层如何接到 matcher

假设 matcher 输出：

```python
logits.shape == [B, K + 1]
# K candidate refs + NONE
```

实现一个 mask-aware 版本：

```python
def masked_pnorm_confidence(logits, valid_mask, p):
    # valid_mask 包含候选和 NONE
    count = valid_mask.sum(-1, keepdim=True)

    masked = logits * valid_mask
    mean = masked.sum(-1, keepdim=True) / count

    centered = (logits - mean) * valid_mask
    norm = centered.abs().pow(p).sum(-1, keepdim=True).pow(1.0 / p)
    norm = norm.clamp_min(1e-12)

    normalized = centered / norm
    normalized = normalized.masked_fill(~valid_mask.bool(), float('-inf'))

    return normalized.max(-1).values
```

生产中建议同时保留：

```text
confidence_msp
confidence_margin
confidence_pnorm
candidate_top1
candidate_top2
raw_logits
```

便于离线比较不同 confidence estimator。

---

## 11. p 的优化方式应从论文 AURC 改成业务目标

官方实现：

```python
for p in p_range:
    metric_value = metric(method(normalize(logits, p)), risk)
    if metric_value < metric_min:
        p_opt = p
```

本项目可以直接复用这个结构，但 `metric` 改成业务损失。

例如：

```python
def business_metric(confidence, y_correct):
    # 目标：首先杜绝 FP，然后才提高 coverage
    thresholds = torch.quantile(confidence, torch.linspace(0.8, 0.999, 100))

    best = float('inf')
    for t in thresholds:
        selected = confidence >= t
        if selected.sum() == 0:
            continue

        fp = (~y_correct[selected]).sum().item()
        coverage = selected.float().mean().item()

        # FP 极重惩罚
        loss = fp * 1_000_000 - coverage
        best = min(best, loss)

    return best
```

更正规的做法是：

> 最大化 coverage，约束 `precision >= P_target`。

例如：

```text
max coverage
s.t. validation precision >= 99.9%
     hard-negative FP = 0
     cross-source conflict FP = 0
```

---

## 12. Threshold 不应该只有一个

建议阈值按风险层分桶：

```text
brand
source_pair
candidate_count_bucket
reference_extraction_channel
has_conflict
model_version
```

例如：

```text
Rolex + structured reference + 雷小安→腕表之家
    tau = 0.91

Rolex + OCR-only + 雷小安→奢当家
    tau = 0.995

未知品牌 + title-only
    永不自动 ACCEPT
```

不要为了追求简单而设置一个“全站 similarity > 0.95 自动匹配”的阈值。

---

## 13. 最终自动 ACCEPT Policy

推荐第一版使用非常保守的规则。

### 13.1 ACCEPT-A：强规则直接接受

满足：

```text
brand 一致
AND canonical_lossless reference 完全一致
AND reference role = brand_reference
AND 没有第二个冲突 reference
AND 商品类型不是 accessory/parts/strap/box/certificate
```

这种情况甚至可以不调用深度 matcher。

### 13.2 ACCEPT-B：抽取有歧义，但模型完成消歧

满足：

```text
brand 一致
AND top1 candidate reference 在原始文本/结构化/OCR 中有真实 span/evidence
AND top1 != NONE
AND no hard conflict
AND pnorm_confidence >= bucket_tau
AND top1-top2 margin >= margin_tau
AND reference role classifier = brand_reference
```

注意：模型只能在“已经被真实 evidence 提到的 reference 候选”中选择。

### 13.3 REVIEW

任一条件：

- structured/title/OCR reference 冲突；
- top1/top2 太接近；
- OCR-only；
- accessory 类型；
- 新品牌；
- 新来源；
- distribution shift；
- pNorm 低于自动阈值但高于垃圾阈值。

### 13.4 REJECT

- 品牌冲突；
- 明确不同 canonical reference；
- 当前商品没有任何可信 reference evidence；
- 明确是配件，而 reference 只是适配型号。

---

## 14. 从 reference 到跨源实体匹配：不要做图传递式“猜合并”

Spec 定义“同一商品 = 同一 reference”，因此最终实体图应以 `canonical_reference_id` 为中心：

```text
canonical_reference 126610LN
 ├── 雷小安 record A
 ├── 雷小安 record B
 ├── 腕表之家 record C
 └── 奢当家 record D
```

而不是先判断：

```text
A ~= C
C ~= D
=> A ~= D
```

这种 pairwise transitive closure 容易出现“一条错边污染整簇”。

正确方式是每条 source record 独立链接到 canonical reference。

这样：

- 新来源接入时无需重做旧 pair；
- 增量复杂度近似 O(N log M)；
- 回滚只需要撤销单条 record→reference link；
- 审计非常清晰。

---

## 15. 千万级规模的工程实现

### 15.1 存储

建议：

- 原始抓取：S3/OSS + Parquet；
- 主数据/决策：PostgreSQL；
- 批量统计与离线评测：ClickHouse；
- 模糊候选检索：OpenSearch/Elasticsearch；
- 图片/OCR 中间结果：对象存储；
- 模型服务：Python + FastAPI/Triton。

如果当前数据只有几百万，第一版甚至可以 PostgreSQL + pg_trgm，不必提前引入过重架构。

### 15.2 初始全量

```text
raw ingest
-> brand normalize
-> reference extraction
-> canonical ref build
-> candidate retrieve
-> matcher
-> confidence
-> policy
-> decision table
```

### 15.3 持续增量

每条新商品只需要：

1. 解析自身；
2. 检索 canonical reference；
3. 跑候选消歧；
4. 写一条 decision。

不需要和历史千万商品逐个比较。

### 15.4 幂等

用：

```text
(source, source_item_id, content_hash)
```

作为版本判断。

商品内容没变化则跳过；reference 相关字段变化则重新决策，并保存旧 decision 版本。

---

## 16. 几百条黄金标签怎么用最划算

不要随机标几百条。

应优先构造 high-risk hard-negative：

1. 同品牌同系列、只差 1 个字符；
2. suffix 不同；
3. 表带/配件标题含兼容 reference；
4. OCR O/0、I/1、S/5 混淆；
5. 多个 reference 同时出现在标题；
6. 结构化字段与标题冲突；
7. 三个平台字段覆盖差异最大的样本；
8. 新旧 reference 格式；
9. 翻新/改装/后镶商品；
10. 套装、盒证、附件。

标签格式也不要只有 `match/non-match`，建议标：

```json
{
  "true_reference": "126610LN",
  "evidence_spans": [...],
  "reference_role": "brand_reference",
  "conflict": false,
  "auto_match_safe": true,
  "reason": "structured + title exact"
}
```

这样一套标签可以同时训练：

- reference extractor；
- role classifier；
- candidate matcher；
- confidence threshold。

---

## 17. “99.99% precision”与几百标签的现实限制

如果只靠几百条验证数据，即使 observed false positive = 0，也很难从统计上证明真实 precision 达到 99.99%。

所以本项目不能把“黄金集 300 条全对”理解成获得了 99.99% guarantee。

正确策略是多层防线：

1. reference 硬规则；
2. 冲突拒识；
3. candidate matcher；
4. pNorm confidence；
5. 超高阈值；
6. hard-negative 回归测试；
7. shadow mode；
8. 人工复核；
9. 错误样本永久加入 regression set。

也就是说，pNorm 是“排序和拒识层”，不是形式化安全证明。

---

## 18. 分布漂移处理：这是本论文最适合生产的点之一

三个来源持续抓取后一定会发生：

- 页面模板变化；
- 商家标题风格变化；
- OCR 质量变化；
- 新品牌/新系列；
- 新 reference 格式；
- 数据比例变化。

因为 pNorm 是 post-hoc 层，主 matcher 不一定要立刻重训。

可以每周/每天监控：

```text
confidence histogram
NONE ratio
ACCEPT coverage
review ratio
top1-top2 margin
channel conflict rate
brand/source PSI
manual review FP
```

发现分布漂移：

1. 先提高 threshold；
2. 再用最新 review 样本重新选择 p；
3. 仍异常才重训 matcher。

这比“模型每次漂移都全量重训”便宜得多。

---

## 19. Shadow Mode 上线方案

### Phase 0：只跑规则

自动接受：

- brand exact；
- canonical_lossless reference exact；
- no conflict。

记录 coverage 与人工抽检 precision。

### Phase 1：模型只观察，不写主数据

运行 candidate matcher + pNorm，但所有输出只写 shadow table。

对比：

```text
rule decision
model top1
pnorm confidence
human label
```

### Phase 2：开放极高阈值 ACCEPT-B

只开放最安全 bucket，例如：

```text
known brand
structured/title 两路一致
无 OCR conflict
pNorm > tau_high
```

### Phase 3：逐步扩 coverage

每个 bucket 独立放量，不做全局一刀切。

---

## 20. 推荐 API

### 20.1 `/extract-reference`

```json
{
  "product_id": 123,
  "references": [
    {
      "surface": "126610LN",
      "canonical_lossless": "126610LN",
      "role": "brand_reference",
      "channel": "title",
      "confidence": 0.997
    }
  ]
}
```

### 20.2 `/reference-candidates`

```json
{
  "product_id": 123,
  "candidates": [
    {"reference_id": 1, "ref": "126610LN"},
    {"reference_id": 2, "ref": "126610LV"}
  ]
}
```

### 20.3 `/match-reference`

```json
{
  "product_id": 123,
  "top1_reference_id": 1,
  "raw_logits": [8.1, 3.4, -1.2],
  "pnorm_confidence": 0.93,
  "margin": 4.7,
  "decision": "ACCEPT",
  "rule_trace": [
    "brand_exact",
    "reference_seen_in_title",
    "no_reference_conflict",
    "pnorm_above_bucket_threshold"
  ]
}
```

---

## 21. 推荐离线评测集

不要只做随机 train/test split。

建议至少六个 slice：

| Slice | 目标 |
|---|---|
| random | 基础性能 |
| hard-ref | 相邻 reference false positive |
| cross-source | 三来源字段差异 |
| OCR | 图片编号质量 |
| accessory | 兼容型号陷阱 |
| temporal | 新增数据/分布漂移 |

关键指标：

```text
FP count
precision
coverage
review rate
AURC
Coverage@100% observed precision
hard-ref FP
accessory FP
```

对这个系统，`FP count` 应放在 dashboard 第一位，而不是 F1。

---

## 22. 可以直接开发的第一版任务拆分

### Week 1：数据与规则

- 建 `canonical_reference` / `reference_evidence` / `match_decision`；
- 做 brand normalize；
- 从三个来源各抽 1000 条，整理 reference field；
- 实现 lossless normalize；
- 实现编号角色分类规则；
- 建 hard-negative 测试集。

### Week 2：候选与 matcher

- OpenSearch/pg_trgm 候选召回；
- 固定 K 候选；
- LightGBM/MLP candidate scorer；
- 加 `NONE`；
- 记录 raw logits。

### Week 3：selective confidence

- 实现 MSP / margin / MaxLogit-pNorm；
- p grid search；
- 按 brand/source/candidate-count 评测；
- 找 `Coverage@target precision` 最优 threshold。

### Week 4：shadow + review

- shadow pipeline；
- review UI；
- 误判原因结构化；
- review 数据回流；
- 阈值版本化。

---

## 23. 最小可用代码骨架

```python
class MatchService:
    def match(self, record):
        brand = normalize_brand(record)

        evidence = extract_reference_evidence(record, brand)

        if has_hard_reference_conflict(evidence):
            return Decision.review("reference_conflict")

        direct = direct_exact_reference(evidence, brand)
        if direct and direct.is_safe:
            return Decision.accept(
                reference_id=direct.reference_id,
                reason="hard_exact_rule"
            )

        candidates = retrieve_candidates(
            brand=brand,
            evidence=evidence,
            k=16,
        )

        if not candidates:
            return Decision.reject("no_candidate")

        logits, valid_mask = matcher.score(record, evidence, candidates)
        top1 = select_top1(logits, candidates)

        conf = masked_pnorm_confidence(
            logits,
            valid_mask,
            p=threshold_registry.p_for(record, evidence),
        )

        if top1.is_none:
            return Decision.reject("model_none")

        if not top1.has_literal_reference_evidence:
            return Decision.review("model_cannot_invent_reference")

        tau = threshold_registry.tau_for(
            brand=brand,
            source=record.source,
            channels=evidence.channels,
            candidate_count=len(candidates),
        )

        if conf < tau:
            return Decision.review("low_selective_confidence")

        if top1.margin < threshold_registry.margin_tau(brand):
            return Decision.review("small_candidate_margin")

        return Decision.accept(
            reference_id=top1.reference_id,
            reason="candidate_matcher+pnorm"
        )
```

---

## 24. 最终建议

这篇论文适合当前需求的地方，是它提供了一个很轻的**可拒识后处理层**：不用把主 matcher 训练成“无所不知”，而是重点学会哪些结果应该自动相信、哪些应该放弃。

不过落地时必须做三处改造：

1. **不要直接用于二分类 pairwise matcher**，改成 `K candidate references + NONE`；
2. **不要只优化 AURC**，改成 precision-first 的 coverage constrained threshold；
3. **不要让置信度模型越过硬 reference 规则**，reference 冲突、编号角色不明、配件兼容型号等情况直接拒识。

最推荐的系统边界是：

> **机器学习负责消歧，规则负责授权。**

在这个边界下，pNorm confidence 的价值很明确：它不是用来“发现更多匹配”，而是用来从已有候选判断中筛出一个非常窄、非常可信的自动 ACCEPT 区间，把其余样本安全地留给 REVIEW/REJECT。

这比追求更高 recall/F1 更符合当前二奢跨源匹配的真实目标，也更容易在 100 万～1000 万持续增量规模上稳定运行。