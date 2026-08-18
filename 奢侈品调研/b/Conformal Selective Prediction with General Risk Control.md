# Conformal Selective Prediction with General Risk Control：用 SCoRE 给跨源腕表 Reference 匹配增加有限样本风险控制闸门

- 分析人：b
- 调研文章：**Conformal Selective Prediction with General Risk Control**
- 方法/项目名：**SCoRE（Selective Conformal Risk control with E-values）**
- 论文：https://arxiv.org/abs/2603.24704
- 官方实现：https://github.com/Tian-Bai/SCoRE
- PyPI 包：`score-select`
- 代码许可：MIT
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 结论先行

这篇工作的价值不在于再提供一个新的 Entity Matching 模型，而在于解决当前 Spec 中更困难的最后一公里问题：**已经有一个会产出候选匹配的系统后，哪些结果可以安全地自动放行，哪些必须拒识或进入人工复核？**

当前需求把“同一个商品”严格定义为“同一 reference number / 型号”，并明确要求 **precision 优先到极致，允许漏匹配**。因此生产系统不应该把一个神经网络/LLM 的概率直接解释为“匹配置信度”，然后手工设 `0.99`、`0.999` 之类阈值。模型分数没有天然概率语义，数据漂移后也很容易失效。

SCoRE 的合适位置是：

> `Reference 抽取与规范化 -> 强规则候选 -> 风险分数 -> SCoRE 风险控制闸门 -> 自动匹配 / 拒识`

其中，SCoRE 不负责“猜 reference”，也不允许图片相似度覆盖 reference 冲突。它只负责把一个任意的、甚至校准不完美的风险分数转换成具有有限样本风险控制含义的“可放行/不可放行”决策。

对本需求，我建议落地成 **Hard Gate + SCoRE Gate + Abstain** 三层：

1. **Hard Gate**：品牌、reference 角色、canonical reference 一致性等确定性规则先拦截；
2. **SCoRE Gate**：对仍然存在“抽错型号/归一化错/配件标题误带主品型号”等残余风险的候选边做风险控制；
3. **Abstain**：未满足统计放行条件的记录不合并，进入待补证据或人工复核。

这是比“训练更强 matcher”更贴合当前业务约束的一条路线。

---

## 2. 为什么选 SCoRE，而不是继续堆一个 matcher

当前三源数据（雷小安、腕表之家、奢当家）的关键困难并不是传统意义上的“两个商品看起来像不像”，而是：

- reference 有时是结构化字段，有时埋在标题；
- 平台自有 SKU、货号、库存 ID 很容易长得像 reference；
- 表带、盒证、配件标题可能包含其适配腕表的 reference；
- 同系列不同 reference 的文本和图片可能极其相似；
- 数据规模 100 万～1000 万且持续增量，分布会漂移；
- 业务允许漏掉，但不能轻易把不同 reference 合成一组。

所以真正危险的是 **false positive**。

传统 matcher 典型做法是学习：

```text
P(match | title_a, title_b, attrs_a, attrs_b, image_a, image_b)
```

然后设一个阈值：

```text
if P(match) > 0.995:
    accept
```

但 `0.995` 往往只是模型 softmax / sigmoid 输出，不代表线上真的只有 0.5% 错误，更不代表更换品牌、来源或时间批次后还成立。

SCoRE 则把问题改写成：

```text
我已经有一个候选匹配系统 f，
也有一个“越小越安全”的风险分数 s(X)，
再给我一批独立的人工黄金标签，
能否只放行一部分候选，使被放行部分的未知风险满足指定上限？
```

这正好与“允许拒识”的需求结构一致。

---

## 3. 论文方法的核心：把“信任模型”变成风险受控的选择问题

论文设定：

- 有 calibration 数据 `D_calib={(X_i,Y_i)}_{i=1}^n`；
- 有待决策 test 数据 `X_{n+j}`；
- 已有任意黑盒模型 `f`；
- 对每个样本定义真实部署风险 `L(f,X,Y)∈[0,1]`；
- 已有一个 score `s(X)`，数值越小表示初步判断越安全；
- 最终输出 `psi∈{0,1}`：`1` 表示信任/部署，`0` 表示 abstain。

一个非常重要的性质是：**SCoRE 的有限样本有效性不依赖 `s(X)` 一定是准确的概率估计。** score 越好，通常只是覆盖率/放行率越高；正确性主要依赖 calibration 与 test 的交换性等假设。

这意味着我们可以把大量工程经验写入 score，而不是非要先得到一个完美的深度模型。

### 3.1 MDR：Marginal Deployment Risk

论文定义：

```text
MDR = E[L * psi]
```

它控制的是“总体部署所产生的期望风险”。

如果将本需求中的风险定义为：

```text
L = 1  当系统自动合并了一对实际上 reference 不同的记录
L = 0  当自动合并确实是同 reference
```

那么 MDR 可以控制单个测试候选边被错误自动放行的边际风险。

### 3.2 SDR：Selective Deployment Risk

对一批 `m` 个候选，论文定义：

```text
SDR = E[
  sum_j L_j * psi_j
  -----------------
  max(1, sum_j psi_j)
]
```

它表示 **被自动放行结果中的平均风险**。当 `L` 为二元 false-positive loss 时，它与“自动合并集合中的错误比例”最接近，因此语义上比 MDR 更符合本项目的 precision-first KPI。

可把目标理解为：

```text
在系统真正自动合并的那一小部分候选中，
让 false match 的平均比例受控；
其余大量不确定候选全部 abstain。
```

### 3.3 风险调整 e-value

SCoRE 用 risk-adjusted e-value 连接选择性部署与假设检验。其关键性质是：

```text
E >= 0
E[L * E] <= 1
```

然后：

- MDR：通过 e-value 阈值产生放行集合；
- SDR：把 e-values 送入 e-BH 类多重检验流程，控制被选择集合上的平均风险。

这个框架的价值是：它不要求对每个样本都输出“可信概率”，而是只保证最终被选择/信任的集合满足风险约束。

---

## 4. 论文代码实现拆解

官方仓库 `Tian-Bai/SCoRE` 已公开可安装实现，README 给出的入口是：

```python
from SCoRE import SCoRE_MDR, SCoRE_SDR
```

核心包目录：

```text
SCoRE/
  SCoRE.py
  utility.py
  __init__.py
applications/
  drug/
  icu/
  llm/
simulation/
simulation_w/
tests/
```

仓库使用 MIT License，可以直接集成或改写到内部服务。

### 4.1 官方 MDR 快速实现

官方 `SCoRE_MDR` 在推荐 `gamma=alpha` 的情况下，核心判定可以概括为：

```python
phi = (
    1 + np.sum(Lcalib * (Scalib <= Stest[j]))
) / (Ncalib + 1) <= gamma
```

其中：

- `Lcalib`：calibration 样本真实风险；
- `Scalib`：calibration 样本风险 score；
- `Stest[j]`：当前待放行候选的风险 score；
- 越小的 score 表示越安全。

直观解释：

> 在 calibration 中，看“风险分数不高于当前候选”的那些更安全样本里，真实错误累计有多少；只有这个经验风险经过 conformal 修正后仍足够低，才允许放行当前候选。

这比直接规定 `score<0.01` 更有实际意义，因为阈值来自人工黄金标签的真实错误表现。

### 4.2 SDR 实现

官方 `SCoRE_SDR` 会：

1. 合并并排序 calibration score 与 test score；
2. 构造 prefix statistics；
3. 为每个 test 候选计算 risk-adjusted e-value；
4. 使用 e-BH 选择满足总体 SDR 目标的测试集合。

官方实现的 docstring 给出的复杂度是：

```text
O(m(n+m) + (n+m) log(n+m))
```

这对论文实验是合理的，但**不能直接把 `m=1000万` 的全量候选一次性塞进去**。生产系统必须先强力 blocking / hard gate，再按品牌、来源对、时间窗口做 micro-batch。

### 4.3 Covariate shift

仓库还提供：

```python
SCoRE_MDR_w(...)
SCoRE_SDR_w(...)
```

用 calibration/test 权重处理 covariate shift。

这对本需求很重要，因为：

- 新品牌上线；
- 某平台标题模板发生变化；
- 某来源突然增加大量配件；
- OCR 引擎升级；
- 数据源字段覆盖率变化；

都会破坏“历史 calibration 与线上 test 同分布”的近似。

不应该把 3 个月前的一套阈值永久固定使用。

---

## 5. 把论文问题精确映射到腕表 Reference 匹配

我建议不要把 `X` 定义成单条商品，而定义成 **候选匹配边**：

```text
X = (record_a, record_b, evidence_a, evidence_b, pair_features)
```

其中 `f(X)` 是现有匹配流水线给出的“建议合并”，而未知标签 `Y` 是黄金 reference 真值。

### 5.1 风险 L 的定义

最直接版本：

```python
L = 1.0 if canonical_reference_truth_a != canonical_reference_truth_b else 0.0
```

注意：

- false negative 不进入这个 loss；
- abstain 也不处罚；
- 这完全顺应“允许漏匹配”的业务目标；
- 自动合并不同 reference 是唯一的一等事故。

如果后续要对特别严重错误加权，也可以使用 `[0,1]` 连续 loss，但 MVP 不建议复杂化，先保持二元 FP loss，指标更容易解释。

### 5.2 score s(X) 的含义

这里不要把 score 简化成“文本 cosine distance”。应该让它表示 **这条自动合并边可能出错的风险等级**。

建议特征如下。

#### A. reference 证据

```text
ref_exact_equal
ref_normalized_equal
ref_extraction_source_a
ref_extraction_source_b
ref_mention_count_a
ref_mention_count_b
ref_from_structured_field
ref_from_title
ref_from_ocr
ref_from_description
ref_regex_valid
ref_in_brand_catalog
```

#### B. 冲突特征

```text
multiple_ref_candidates_a
multiple_ref_candidates_b
structured_vs_title_conflict
text_vs_ocr_conflict
brand_conflict
series_conflict
size_conflict
material_conflict
accessory_suspected
platform_sku_suspected
```

对于 precision-first 系统，**冲突特征通常比相似特征更重要**。

#### C. 语义/视觉辅助

```text
title_embedding_similarity
image_similarity
ocr_token_similarity
series_similarity
```

但它们只能用于提高/降低风险 score，不能覆盖 reference 硬冲突。

#### D. 来源与漂移特征

```text
source_pair
brand
category
crawl_time_bucket
extractor_version
normalizer_version
```

这些特征用于发现某一来源/版本的特殊误差模式，也方便后续做加权校准。

---

## 6. 推荐的生产架构

```text
┌──────────────────────────────────────────────┐
│ 1. Ingestion                                │
│ 雷小安 / 腕表之家 / 奢当家 增量商品          │
└─────────────────────┬────────────────────────┘
                      │
                      v
┌──────────────────────────────────────────────┐
│ 2. Reference Evidence Extractor              │
│ structured field / title / description / OCR │
│ 每个 evidence 保留来源、span、置信度、版本    │
└─────────────────────┬────────────────────────┘
                      │
                      v
┌──────────────────────────────────────────────┐
│ 3. Canonicalizer + Role Classifier           │
│ brand canonicalization                       │
│ reference vs platform SKU vs inventory ID    │
│ brand-specific normalization rules           │
└─────────────────────┬────────────────────────┘
                      │
                      v
┌──────────────────────────────────────────────┐
│ 4. Candidate Generator / Reference Index     │
│ brand block + canonical ref lookup           │
│ 禁止全量 Cartesian join                      │
└─────────────────────┬────────────────────────┘
                      │
                      v
┌──────────────────────────────────────────────┐
│ 5. HARD GATE                                 │
│ brand/ref 冲突直接 REJECT                    │
│ ref 证据不足直接 ABSTAIN                     │
│ 只留下“理论上可合并”的小候选集合             │
└─────────────────────┬────────────────────────┘
                      │
                      v
┌──────────────────────────────────────────────┐
│ 6. Pair Risk Scorer s(X)                     │
│ GBDT / Logistic / RF / small NN / heuristic  │
│ score 越小越安全                             │
└─────────────────────┬────────────────────────┘
                      │
                      v
┌──────────────────────────────────────────────┐
│ 7. SCoRE Gate                                │
│ online: conservative MDR                     │
│ micro-batch: SDR / e-BH                      │
└──────────────┬───────────────────┬───────────┘
               │                   │
          safe │                   │ unsafe/unknown
               v                   v
      AUTO_ACCEPT            ABSTAIN / REVIEW
               │
               v
┌──────────────────────────────────────────────┐
│ 8. Entity Graph / Match Table                │
│ 记录 policy_version / score / reason /       │
│ calibration_version / audit trail            │
└──────────────────────────────────────────────┘
```

SCoRE 必须位于 hard gate 后，而不是前面。

原因：统计风险控制不是业务逻辑替代品。如果已知两个品牌不同、reference 明确不同，就不应该让统计模型有“翻盘”的机会。

---

## 7. Hard Gate 应该如何设计

### 7.1 Reference 不是任意字符串归一化

不能简单做：

```python
normalized = re.sub(r'[^A-Za-z0-9]', '', ref).upper()
```

因为某些品牌的点、斜杠、后缀、材质/表带编码可能具有语义。

应做品牌级规则：

```python
normalize_ref(brand, raw_ref) -> CanonicalRef
```

并返回：

```json
{
  "canonical": "...",
  "raw": "...",
  "rule_version": "rolex-v3",
  "valid": true,
  "ambiguity": false
}
```

### 7.2 编号角色先分类

所有“像型号”的 token 都需要先判断角色：

```text
BRAND_REFERENCE
PLATFORM_SKU
SHOP_SKU
INVENTORY_ID
ACCESSORY_COMPATIBLE_REFERENCE
SERIAL_NUMBER
UNKNOWN_CODE
```

只有 `BRAND_REFERENCE` 能直接进入匹配主键候选。

### 7.3 强冲突一票否决

例如：

```python
if brand_a != brand_b:
    REJECT

if strong_ref_a and strong_ref_b and ref_a != ref_b:
    REJECT

if current_item_is_accessory and ref_only_comes_from_compatibility_text:
    ABSTAIN

if multiple_conflicting_refs:
    ABSTAIN
```

视觉相似不能解除这些否决。

---

## 8. SCoRE Gate 的直接工程实现

### 8.1 MVP：先做在线 MDR

官方 MDR 在 `gamma=alpha` 下可以做得比逐样本扫描 calibration 更快。

先离线构建：

```python
import numpy as np

class MDRGate:
    def __init__(self, calib_scores, calib_losses, alpha):
        order = np.argsort(calib_scores)
        self.scores = np.asarray(calib_scores)[order]
        losses = np.asarray(calib_losses)[order]
        self.bad_prefix = np.cumsum(losses)
        self.n = len(self.scores)
        self.alpha = alpha

    def accept(self, score):
        # 找到 calibration 中 s_i <= 当前 score 的最大位置
        k = np.searchsorted(self.scores, score, side="right")
        bad = 0.0 if k == 0 else self.bad_prefix[k - 1]
        calibrated_risk = (1.0 + bad) / (self.n + 1)
        return calibrated_risk <= self.alpha, calibrated_risk
```

这样：

- calibration 预处理：`O(n log n)`；
- 每个新候选：`O(log n)`；
- 可以直接用于持续增量流；
- 不需要对 1000 万商品做全局重算。

生产版还应加入 tie 处理、连续 loss、weighted shift、policy version 等，但整体思路不变。

### 8.2 批量自动合并：再做 SDR

SDR 的语义更接近“自动合并集合的 precision”。

但不建议全量执行：

```text
千万商品 -> 亿级候选 -> SCoRE_SDR
```

而应：

```text
千万商品
 -> brand/reference blocking
 -> hard gate
 -> 只剩少量高可信候选
 -> 按 source_pair + brand_group + time_window 微批
 -> SCoRE_SDR
```

例如每 5～30 分钟形成一个 post-hard-gate 批次，再运行 SDR。

### 8.3 输出不是只有 yes/no

每条 decision 至少落库：

```json
{
  "pair_id": "...",
  "decision": "AUTO_ACCEPT",
  "hard_gate_pass": true,
  "risk_score": 0.0127,
  "score_model_version": "risk-gbdt-v4",
  "score_policy": "SCORE_SDR",
  "alpha": 0.001,
  "calibration_version": "2026-08-18-brand-global-v7",
  "extractor_version": "ref-extractor-v12",
  "normalizer_version": "canonicalizer-v9",
  "reason_codes": [
    "REF_EXACT_EQUAL",
    "STRUCTURED_FIELD_BOTH",
    "NO_CONFLICTING_REF"
  ]
}
```

这样未来发生误合并时可以完整追溯：到底是 extractor、normalizer、score，还是 calibration policy 出错。

---

## 9. 一个很关键的现实约束：几百条黄金标签不够支撑“任意小 alpha”

这是这篇论文对当前 Spec 最有价值的提醒之一。

在 MDR 推荐形式 `gamma=alpha` 下，即使 calibration 中当前阈值以内 **一个错误都没有**，判定式仍至少有：

```text
1 / (n + 1)
```

这一项。

因此想要：

### 99.9% precision 级别

若希望 `alpha = 0.001`，至少需要：

```text
1 / (n + 1) <= 0.001
n >= 999
```

即 calibration 集本身至少接近 1000 条。

### 99.99% precision 级别

若希望 `alpha = 0.0001`，至少需要：

```text
n >= 9999
```

所以“人工标注几百对”适合：

- 建第一版风险分数；
- 做大粒度风险校准；
- 找 hard-negative 模式；

但如果产品文案真的要求“万分之一以下误匹配”并希望得到严格有限样本统计控制，**必须继续积累 calibration 标签，不能靠把阈值从 0.01 手调成 0.0001**。

这是一个应该直接写进上线验收标准的工程事实。

另外必须强调：SCoRE 给的是有限样本的期望风险控制，不等于数学上保证“线上永远 0 个 false positive”。如果业务字面要求绝对零错误，唯一可靠的策略仍是：

```text
只自动合并具有可验证 reference 硬证据的记录；
任何不确定情况 abstain。
```

SCoRE 是对硬规则剩余风险的统计安全层，不是“零错误证明器”。

---

## 10. 几百条标注该怎么花，才能最大化收益

论文假设 score `s` 最好独立于 calibration 数据训练。如果我们把仅有的 300～500 条标签全部拿去训练风险模型，再拿同一批做 calibration，会破坏理论前提，也容易过拟合。

因此第一阶段不建议把所有标签都喂给复杂模型。

### 方案 A：规则 score + 大部分标签做 calibration

先手工定义：

```text
score =
  + w1 * ref_source_weak
  + w2 * multiple_ref_conflict
  + w3 * accessory_suspected
  + w4 * platform_sku_suspected
  + w5 * brand_catalog_missing
  + w6 * title_ocr_conflict
  - w7 * structured_ref_both_exact
  - w8 * independent_evidence_agreement
```

然后尽量把人工黄金标签留给 calibration。

SCoRE 的有效性不要求这个 score 是完美概率，只会影响覆盖率，所以这是数据很少时非常合适的策略。

### 方案 B：历史弱标签训练 score，新鲜人工标签专门 calibration

如果已有高可信历史数据：

- 来源结构化 reference 完全一致 -> 弱正样本；
- 明确 reference 冲突 -> 强负样本；

可以用这些训练 `GBDT / LightGBM / RandomForest` 风险排序器，再把人工的 300～500 对作为独立 calibration。

### 黄金集必须故意加入 hard negatives

不能随机从千万商品里抽几百对，因为随机负例太简单，会产生虚假安全感。

应过采样：

```text
同品牌同系列、reference 只差少数字符
同品牌近似 reference
标题包含多个型号
配件/表带/盒证
平台 SKU 像 reference
OCR 将 O/0、I/1、S/5 混淆
同款不同材质/尺寸且 reference 不同
同图片/盗图但 reference 不同
来源字段之间互相冲突
```

线上事故几乎都来自这些边界情况，而不是完全不相关的随机商品对。

---

## 11. 数据表建议

### `product_record`

```text
id
source
source_item_id
brand_raw
brand_canonical
title
description
images
crawl_time
payload_hash
```

### `reference_evidence`

```text
record_id
raw_value
canonical_value
role
source_field        # structured/title/description/ocr
span_or_image_id
extract_confidence
normalizer_version
extractor_version
is_conflicting
```

### `candidate_pair`

```text
pair_id
record_a
record_b
blocking_reason
feature_blob
hard_gate_status
risk_score
score_model_version
```

### `match_decision`

```text
pair_id
decision            # AUTO_ACCEPT / ABSTAIN / REJECT / HUMAN_ACCEPT
policy_type         # HARD / SCORE_MDR / SCORE_SDR
policy_version
alpha
calibration_version
reason_codes
created_at
```

### `calibration_sample`

```text
sample_id
pair_id
truth_same_reference
loss
risk_score_at_label_time
brand
source_pair
evidence_regime
labeler
review_status
created_at
```

必须保存 **当时的 score** 和版本，不能以后模型升级后重新计算再假装它属于旧 calibration。

---

## 12. 持续增量更新方案

当前数据不是一次性离线任务，而是持续抓取。推荐事件化：

```text
new/updated record
  -> extract reference evidence
  -> canonicalize
  -> upsert reference index
  -> generate bounded candidates
  -> hard gate
  -> score
  -> SCoRE gate
  -> write match edge / abstain queue
```

### 12.1 Reference Index

主索引可以按：

```text
(brand_canonical, canonical_reference)
```

组织。

新增记录优先做精确索引命中，不扫描历史全表。

### 12.2 重新处理只影响局部

如果 extractor 或 normalizer 升级：

- 找出对应版本产生过 reference evidence 的记录；
- 重跑 evidence；
- 只重建其邻接候选与 match edges；
- 不做全库 Cartesian recompute。

### 12.3 Calibration 滚动更新

建议每次策略更新形成不可变版本：

```text
calibration-v001
calibration-v002
...
```

并持续监控：

```text
source_pair drift
brand mix drift
extractor version drift
risk score distribution drift
manual-review FP rate
```

一旦显著漂移，旧校准策略只允许降级为更保守，不允许自动放宽。

---

## 13. Exchangeability 与数据漂移：本项目最容易踩的理论坑

SCoRE 的基础有效性依赖 calibration/test 的交换性或相应的 covariate-shift 条件。

本项目天然会破坏它：

```text
calibration：主要是劳力士/欧米茄，来源 A+B
线上：突然出现大量小众品牌，来源 C
```

这时全局 calibration 未必还能代表线上。

但也不能简单把 calibration 切成：

```text
品牌 × 来源对 × 类目 × 月份 × 证据类型
```

因为几百条标签会被切得过碎，统计上几乎没有任何放行能力。

建议分三阶段：

1. **MVP**：全球 calibration + source/brand/evidence 作为 risk-score 特征；
2. **监控**：对子群单独做 precision/coverage 审计；
3. **数据够后**：对高风险/高流量 brand group 建独立 calibration；或使用 weighted SCoRE 处理可估计的 covariate shift。

---

## 14. 图片应该放在哪一层

Spec 明确“有图片可用”，但这里必须避免一个危险设计：

```text
reference 不同
但图片很像
=> 模型仍判 same
```

腕表同系列相邻 reference 可能外观非常接近，甚至不同商家可能复用同一张宣传图。

所以图片的正确角色是：

### 可做

- OCR 表背/保卡/吊牌 reference；
- 当文本 reference 缺失时提高候选检索覆盖率；
- 图文冲突时提高风险 score；
- 人工复核辅助证据。

### 不可做

- reference 明确冲突时用视觉相似度覆盖冲突；
- 把 CLIP 相似度当“同 reference”的最终事实；
- 用图片近似替代 canonical reference 主键。

一句话：**视觉只能补证据或否决，不能越权。**

---

## 15. MVP 可直接落地的代码结构

```text
src/
  ingest/
    adapters.py

  reference/
    extract.py
    normalize.py
    role_classifier.py
    catalog.py

  blocking/
    reference_index.py
    candidate_generator.py

  match/
    hard_gate.py
    features.py
    risk_scorer.py

  risk_control/
    mdr_gate.py
    sdr_gate.py
    weighted_gate.py
    calibration_store.py

  jobs/
    process_incremental.py
    run_sdr_microbatch.py
    refresh_calibration.py

  audit/
    review_queue.py
    metrics.py
```

核心接口：

```python
class MatchDecisionEngine:
    def decide(self, a, b):
        evidence = build_pair_evidence(a, b)

        hard = hard_gate(evidence)
        if hard.status == "REJECT":
            return Decision.reject(hard.reasons)
        if hard.status == "ABSTAIN":
            return Decision.abstain(hard.reasons)

        score = risk_scorer.predict(evidence)
        accepted, calibrated_risk = mdr_gate.accept(score)

        if not accepted:
            return Decision.abstain([
                "SCORE_RISK_NOT_CERTIFIED"
            ])

        return Decision.accept(
            reasons=hard.reasons,
            risk_score=score,
            calibrated_risk=calibrated_risk,
        )
```

后续把“在线 MDR”换成“微批 SDR”时，前面的 extraction / blocking / hard gate / score 都无需改。

---

## 16. 上线评估指标：不要再以 F1 为主

这个项目最不应该优化的是单一 F1。

推荐主指标：

```text
Auto-Accept Precision
False Matches per 1M Auto-Accepted Edges
Auto-Accept Coverage
Abstention Rate
Human Review Rate
```

分组必须看：

```text
per source_pair precision
per brand precision
per evidence_regime precision
per extractor_version precision
```

另外保留：

```text
Reference extraction exact accuracy
Candidate recall
Hard-negative precision
```

### 测试集切分

不能随机 pair split，因为同一 reference/同一商品可能泄漏到训练和测试。

至少做：

- reference-disjoint split；
- time-based holdout；
- brand holdout / long-tail slice；
- source-template change slice。

测试重点是“未来新数据会不会误合并”，不是“见过的 reference 能否被记住”。

---

## 17. 与现有调研方法的组合位置

SCoRE 与此前已经分析的几个方向不是竞争关系，而是最后的风险闸门：

```text
DeepBlocker / blocking
        ↓
Ditto / AnyMatch / rule matcher / LLM extractor
        ↓
reference evidence & hard conflict checking
        ↓
风险 score
        ↓
SCoRE
        ↓
AUTO_ACCEPT / ABSTAIN
        ↓
图级一致性审计（例如 TransClean / GraLMatch 类思想）
```

其中最重要的原则是：

> 上游模型负责提高召回和排序；SCoRE 负责“是否值得信任”；reference 硬证据负责最终业务语义。

---

## 18. 推荐实施顺序

### Phase 0：定义“错误”

先冻结：

```text
same_product := canonical brand reference 完全相同
false_match  := 系统自动合并，但黄金 reference 不同
```

不要在过程中把“外观相似”“同系列”“同款不同尺寸”偷偷混入 same_product。

### Phase 1：两周级可做的防线

1. source-specific reference extractor；
2. brand-specific canonicalizer；
3. code role classifier；
4. reference exact blocking；
5. hard gate；
6. reason-code 全链路日志；
7. 300～500 条 hard-negative 黄金集。

此时宁可大量 abstain。

### Phase 2：SCoRE MVP

1. 用弱标签/规则生成风险 score；
2. 人工黄金集作为独立 calibration；
3. 实现在线 MDR gate；
4. 记录 `alpha/calibration_version/policy_version`；
5. 建立每日 auto-accept precision 审计。

### Phase 3：提升 coverage

1. 继续积累人工复核标签；
2. 训练 GBDT/RF 风险排序器；
3. calibration 独立留出；
4. 对 high-volume brand/source 建微批 SDR；
5. 引入 weighted SCoRE 应对来源/时间漂移。

### Phase 4：高精度 SLA

若目标达到：

```text
99.9%+
99.99%+
```

需要按统计目标反推 calibration 样本量，而不是继续手调模型阈值。同时叠加：

- hard rule；
- SCoRE；
- graph consistency audit；
- 人工抽检；
- 自动回滚/停放策略。

---

## 19. 最终建议

**建议直接落地 SCoRE，但把它定位为“Reference Match Safety Gate”，不是主 matcher。**

对于当前 Spec，最稳妥的生产方案是：

```text
强 reference 证据
    +
确定性冲突否决
    +
候选风险排序
    +
SCoRE 有限样本风险控制
    +
abstention
    +
持续人工黄金标签回流
```

最关键的三点：

1. **自动合并永远需要 canonical reference 证据，图片/LLM 只辅助，不可越权。**
2. **SCoRE 的价值是把“模型高分”变成“在 calibration 假设下可受控的部署风险”，特别适合 precision-first + 可拒识系统。**
3. **“几百条标签”与“万分之一误匹配”在统计上存在硬样本量矛盾。想把 alpha 压到极低，必须持续积累独立 calibration 标签，而不能只提升模型分数。**

因此，如果要从这篇论文中提取一个可立即进入系统设计的实现，就是新增一个独立服务/模块：

```text
ReferenceMatchRiskGate
```

其输入是 hard-gate 后候选边及风险 score，输出是：

```text
AUTO_ACCEPT / ABSTAIN
```

并携带可审计的 risk-control policy/version。这样既可以在当前几百条黄金标签阶段保守上线，也能随着标注积累逐步把 coverage 提高，而不需要牺牲 precision-first 的核心原则。

---

## 20. 参考资料

1. Tian Bai, Ying Jin. **Conformal Selective Prediction with General Risk Control**. arXiv:2603.24704, 2026.  
   https://arxiv.org/abs/2603.24704
2. SCoRE 官方实现：  
   https://github.com/Tian-Bai/SCoRE
3. 当前业务 Spec：  
   https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31
