# Conformal Selective Prediction with General Risk Control：用于跨源二奢/腕表匹配的高精度自动放行层

> 调研对象：Tian Bai, Ying Jin, **Conformal Selective Prediction with General Risk Control (SCoRE)**，arXiv:2603.24704，2026。
>
> 论文：https://arxiv.org/abs/2603.24704
>
> 官方代码：https://github.com/Tian-Bai/SCoRE
>
> 本文目标：结合当前 Spec——雷小安 × 腕表之家 × 奢当家，100 万–1000 万级持续增量商品；“同一商品”定义为**同一 reference number / 型号**；字段稀疏；**precision 极优先，宁可拒识也不能误匹配**；图片可用；可人工标注几百对——分析 SCoRE 的技术实现，并给出可直接落地的系统方案。

---

## 0. 结论先行

这篇论文对当前需求非常有价值，但**价值不在于替代实体匹配模型，而在于给自动匹配增加一个有统计风险控制的“放行/拒识层”**。

最重要的架构结论是：

```text
模型 / LLM / OCR / 图像
        ↓
只负责抽取、校验、给风险分数
        ↓
SCoRE / Conformal Selection
只决定“这条 canonical reference 能不能自动信任”
        ↓
硬规则：canonical_brand + canonical_reference 严格相等
        ↓
自动归并到同一 reference entity
```

即：

1. **reference 是唯一自动匹配键**；
2. 视觉相似、标题相似、embedding、LLM 判断都不能直接产生 match；
3. 模型只负责把脏数据转换成 `canonical_reference`，以及判断这个转换有多危险；
4. SCoRE/Conformal Selection 负责对“自动接受集合”的风险做校准；
5. 未通过统计门槛、存在冲突、无法证明 reference 的记录全部 abstain（拒识/人工复核）；
6. 最终跨源匹配就是一个确定性的 key join，而不是 pairwise classifier。

这比直接训练一个“商品 A 和 B 是否相同”的二分类器更符合当前 Spec，因为 pairwise 模型即使 99.9% precision，在千万级数据里仍可能产生大量误边；而且一条误边进入聚类后可能污染整个实体簇。

### 一个必须明确的边界

SCoRE 能提供的是**有限样本下的期望风险控制**，不是“数学上保证永远 0 个 false positive”。

因此 Spec 中“绝对不能误匹配”不能仅靠概率模型实现。真正的工程策略必须是：

- 用 deterministic hard guards 把错误空间先压到极小；
- 用 SCoRE/Conformal Selection 控制剩余自动放行风险；
- 任何证据不充分的情况 fail closed；
- 最终用人工复核承接被拒识的长尾；
- 对持续增量和分布漂移重新校准。

换句话说，SCoRE 应该是 **safety certificate layer**，不是 matcher 本身。

---

# 1. 为什么这篇论文正好命中当前 Spec

论文研究的问题是：

> 已经有一个任意黑盒模型 `f`，如何只在“足够可信”的输入上自动采用模型输出，并让被采用样本中的风险受到严格控制？

这和当前任务高度同构。

我们已经可以有任意 reference extractor：

- 正则/品牌规则；
- title NER；
- LLM structured extraction；
- OCR；
- 图像模型；
- 多模态模型；
- 多模型 ensemble。

但真正难点不是“能不能预测”，而是：

> **什么时候有资格自动相信这个 reference？**

当前需求允许漏匹配，因此可以把问题从“尽量猜出来”改写为：

> 对每条商品记录产生一个候选 canonical reference；只有当其风险被证明足够低时，才允许进入自动 join。

这正是 selective prediction / abstention 的使用场景。

---

# 2. SCoRE 的核心思想

## 2.1 输入不是模型概率，而是“风险排序分数”

SCoRE 假设有：

- calibration 样本：`(X_i, Y_i)`；
- 已训练模型 `f`；
- 一个 bounded loss `L(f, X, Y) ∈ [0,1]`；
- 一个 score `s(X)`，约定 **越小越安全**；
- 新的 test instances。

重要的是：论文的有限样本有效性**不要求 `s(X)` 是完美的概率**。只要 calibration/test 满足相应 exchangeability 条件，score 主要影响 power，也就是“能放行多少”，而不是理论风险控制本身。

这非常适合商品 matching，因为我们的风险分数可以是一个工程化组合，而不必强行解释成严格概率：

```text
s(x) =
    reference_role_risk
  + extraction_uncertainty
  + source_field_risk
  + conflict_penalty
  + out_of_registry_penalty
  + OCR_text_disagreement
  + model_margin_risk
  + source_drift_risk
```

只要保证：**越危险，score 越大**。

---

## 2.2 两种风险：MDR 与 SDR

论文定义两种风险。

### MDR：Marginal Deployment Risk

单条 test 记录的自动采用风险：

```text
MDR = E[L * ψ]
```

`ψ=1` 表示自动采用，`ψ=0` 表示拒识。

它更像“总风险预算”。如果系统部署得少，即使被部署样本的条件错误率不极低，也可能满足 MDR。

**因此 MDR 不是当前需求的首选。**

### SDR：Selective Deployment Risk

对一个 batch 中被自动采用的记录：

```text
SDR = E[ sum(L_j * ψ_j) / max(1, sum(ψ_j)) ]
```

如果 `L ∈ {0,1}` 表示“reference assignment 是否错误”，那 SDR 就非常接近：

```text
被自动放行记录中的平均错误率
```

也就是我们真正关心的：

```text
precision ≈ 1 - SDR
```

因此对于当前项目，我建议：

- **生产主门：SDR / binary conformal selection**；
- MDR 作为某些低风险异步任务或监控指标补充。

---

# 3. risk-adjusted e-value：为什么它能控制“被选中的风险”

论文定义一种 risk-adjusted e-value `E_j ≥ 0`，要求：

```text
E[L_j * E_j] <= 1
```

如果一个 test instance 的 e-value 足够大，就可以认为它有足够证据进入“自动部署集合”。

## 3.1 MDR 的 threshold

SCoRE-MDR 使用：

```text
ψ_j = 1{ E_j >= 1 / α }
```

论文直接证明：

```text
E[L_j ψ_j] <= α
```

核心就是：

```text
1{E >= 1/α} <= αE
```

再配合 `E[L E] <= 1`。

## 3.2 SDR 使用 e-BH

对多个 test instance，论文把所有 risk-adjusted e-values 输入 e-BH step-up procedure。

这使得最终被选集合的：

```text
E[average risk among selected] <= α
```

这个思路和 FDR control 类似，但论文把传统二元“错误发现”推广到了 bounded continuous risk。

对于本项目，如果 loss 是 binary：

```text
L = 1  -> 自动采用的 canonical reference 是错的
L = 0  -> reference 正确
```

SDR 就可以直接解释成自动放行集合中的 expected false assignment rate。

---

# 4. 论文具体算法

## 4.1 SCoRE-MDR

官方 Algorithm 1：

1. 在 calibration 集上计算真实 loss `L_i`；
2. 计算 calibration/test 的风险 score；
3. 对每个 test 样本构造 `E_{α,n+1}`；
4. 若 `E >= 1/α` 自动部署，否则拒识。

论文推荐 `gamma = alpha`。

官方代码对应：

```python
SCoRE_MDR(Dcalib, Dtest, alpha, gamma)
SCoRE_MDR_bf(...)
SCoRE_MDR_w(...)
```

其中：

- `SCoRE_MDR_bf` 显式算 e-values；
- `SCoRE_MDR` 使用论文里的 computation shortcut；
- `_w` 是 covariate shift 加权版本。

---

## 4.2 SCoRE-SDR

论文 Algorithm 2：

1. calibration 上算 loss；
2. calibration + test 上算 score；
3. 为每个 test 构造 risk-adjusted e-value；
4. 对 e-values 跑 e-BH；
5. e-BH 选中的集合自动部署。

官方仓库实现：

```python
SCoRE_SDR(
    Dcalib,
    Dtest,
    alpha,
    gamma,
    prune=None,
    return_evals=False,
    random_state=None,
)
```

代码注释给出的优化复杂度是：

```text
O(m(n+m) + (n+m) log(n+m))
```

其中：

- `n` = calibration size；
- `m` = 当前 test batch size。

实现内部会：

1. 把 calibration score 和 test score 合并排序；
2. 做 prefix sum：
   - `NUMER`：到 threshold 为止的 calibration loss 累积；
   - `DENOM`：到 threshold 为止的 test 数量；
3. 显式处理 score tie；
4. 对每个 test instance 构造 `FR_0 / FR_1`；
5. 找可行 threshold；
6. 算 e-value；
7. 最终调用 `eBH(evalues, alpha)`。

这个 repo 不是只有 notebook，核心 SCoRE 已经封装成可安装 Python package：

```bash
pip install score-select
```

目录结构也很清楚：

```text
SCoRE/
  SCoRE.py      # MDR / SDR / weighted algorithms
  utility.py    # BH / eBH / loss / evaluation helpers
applications/
  drug/
  icu/
  llm/
simulation/
simulation_w/
tests/
```

所以它可以直接作为实验依赖，而不需要重写论文算法。

---

# 5. 对本项目最重要的发现：二元 loss 时可以进一步简化

我们的核心风险天然是 binary：

```text
canonical reference 正确 -> L = 0
canonical reference 错误 -> L = 1
```

论文证明，在 binary risk 场景，SCoRE 和已有 conformal selection 有很强的等价关系；缩小对未知 loss 的搜索范围后，可以直接恢复 conformal selection。

官方仓库也直接提供：

```python
CS(Dcalib, Dtest, alpha, mult_test=True, return_pvals=False)
```

其中：

- `mult_test=False`：MDR 风格；
- `mult_test=True`：BH，多 test 的 SDR/FDR 风格。

所以**V1 最推荐的不是把完整连续 SCoRE-SDR 硬塞进生产，而是先用 binary Conformal Selection + BH 做 reference 自动放行。**

完整 SCoRE 的 continuous-risk 能力留给 V2：例如把一次错误 reference assignment 的下游污染成本做成连续 loss。

---

# 6. 当前 Spec 中“匹配对象”应该重新定义

传统 entity matching 很容易把 unit 定义成：

```text
pair = (source_record_A, source_record_B)
model(pair) -> match / no-match
```

我不建议这样做。

当前业务已经给出非常强的语义定义：

> 同一商品 = 同一 reference number。

那最安全的 unit 应该是：

```text
source product record
    ↓
canonical reference assignment
    ↓
reference entity
```

也就是：

```text
record -> (canonical_brand, canonical_reference)
```

而不是预测任意 record pair 是否相同。

理由：

1. 1000 万记录 pairwise 空间不可接受；
2. pairwise 相似模型会把同系列相邻 reference 弄混；
3. 外观近似不代表 same reference；
4. 一条错边可能污染整簇；
5. 严格 reference key 天然支持增量 upsert；
6. audit 非常容易：每个 match 都能回溯到 reference 证据。

---

# 7. 推荐的最终系统架构

```mermaid
flowchart TD
    A[雷小安/腕表之家/奢当家 Raw Records] --> B[字段标准化 + Source Adapter]
    B --> C[Brand Canonicalizer]
    B --> D[Reference Candidate Extractor]
    B --> E[Image/OCR Evidence Extractor]

    C --> F[Reference Role Classifier]
    D --> F
    E --> F

    F --> G[Brand-specific Reference Normalizer]
    G --> H[Reference Registry / Grammar Validator]

    H --> I[Evidence Feature Builder]
    I --> J[Risk Scorer s(x)]

    J --> K[SCoRE / Conformal Selection Calibrator]
    K -->|safe| L[Auto-eligible Reference Assignment]
    K -->|abstain| M[Manual Review Queue]

    L --> N[Hard Guard]
    N --> O[Exact Key: brand_id + canonical_reference]
    O --> P[Reference Entity Store]

    M --> Q[Human Gold Labels]
    Q --> J
    Q --> K
```

核心原则：

```text
没有可信 canonical_reference -> 永不自动匹配
reference 不完全一致 -> 永不自动匹配
brand 不一致 -> 永不自动匹配
reference role 不是 manufacturer reference -> 永不自动匹配
有冲突 reference -> 永不自动匹配
```

图片只能：

- 帮助 OCR 出 reference；
- 支持/否决文本 reference；
- 帮助人工复核。

图片**不能**因为“长得像”就把两个不同 reference 合并。

---

# 8. Reference 抽取层怎么实现

建议每条记录不是只保存一个字符串，而是保存**候选 + 证据 + provenance**。

例如：

```json
{
  "source": "watchhome",
  "product_id": "xxx",
  "brand_raw": "劳力士",
  "brand_id": "rolex",
  "reference_candidates": [
    {
      "raw": "126334",
      "canonical": "126334",
      "source": "structured_field",
      "role": "manufacturer_reference",
      "span": null,
      "confidence": 0.99
    },
    {
      "raw": "m126334-0014",
      "canonical": "126334",
      "source": "title",
      "role": "manufacturer_reference",
      "span": [18, 29],
      "confidence": 0.94
    }
  ],
  "ocr_candidates": [],
  "conflict": false
}
```

## 8.1 候选来源优先级

建议按来源构造 evidence tier：

```text
Tier A
  品牌官网/官方结构化 reference

Tier B
  平台明确的 型号/参考编号 字段
  + 符合品牌 grammar
  + 无其他冲突 reference

Tier C
  title / description 中抽取
  + 语义角色明确
  + registry 可验证

Tier D
  OCR / 图片抽取
  + 和文本证据一致

Tier E
  LLM 猜测 / 视觉相似推断
  只能辅助，不可单独自动通过
```

不要把这些 tier 直接写死成最终决策，而是作为 risk score 的 feature。

---

# 9. Reference Role Classifier 是防误匹配关键层

二奢场景里最大的坑之一是：

> 标题里出现“像型号的字符串”，不代表它是当前商品自身的 manufacturer reference。

常见错误角色：

- 平台内部商品 ID；
- 店铺 SKU；
- 库存号；
- 证书号；
- 序列号；
- 配件兼容型号；
- 表带适配的 watch reference；
- “同款 XXX”营销描述里的别的型号；
- 套装中另一件商品的 reference；
- OCR 误读。

因此必须先做：

```text
candidate token
  -> manufacturer_reference
  -> serial_number
  -> platform_sku
  -> seller_sku
  -> compatibility_reference
  -> certificate_id
  -> unknown
```

只有：

```text
role == manufacturer_reference
```

才有资格进入 canonicalization。

这一步甚至比“更强的 embedding 模型”更有价值，因为它直接消除系统性的 false positive 来源。

---

# 10. Canonicalization 必须是“保守规范化”，不能暴力清洗

最危险的实现是：

```python
canonical = re.sub(r'[^A-Z0-9]', '', raw.upper())
```

这种做法会错误地把不同型号折叠到一个 key。

正确策略是 brand-specific normalization：

```text
raw reference
  ↓
Unicode NFKC
  ↓
大小写标准化
  ↓
只应用该品牌已验证的 separator/alias rule
  ↓
品牌 grammar validation
  ↓
canonical reference
```

必须保留：

```text
raw_reference
normalized_reference
normalization_rule_id
normalization_version
```

如果某种去点号、去横线规则没有被品牌规则明确证明，就不要应用。

### 例子

一个品牌的：

```text
ABC-123
ABC123
```

可能是同一 reference；另一个品牌的 punctuation 却可能有结构意义。

因此不要设计全局 normalize rule。

---

# 11. Risk score `s(x)` 怎么设计

SCoRE 不要求 score 本身严格校准成概率，但它的排序质量会决定 coverage。

建议单独训练一个 **reference assignment risk model**。

输入不是商品是否相似，而是“当前 canonical reference assignment 有多容易错”。

## 11.1 Feature groups

### A. 来源证据

```text
source_name
field_origin
structured_reference_field?
title_extracted?
description_extracted?
ocr_extracted?
```

### B. reference grammar

```text
brand_grammar_valid
length_valid
prefix_valid
segment_pattern_valid
known_family_prefix
registry_exact_hit
registry_near_hit_distance
```

### C. role evidence

```text
role_classifier_prob_manufacturer_ref
role_classifier_margin
nearby_context_has_兼容/适用/表带/配件
nearby_context_has_型号/Ref./Reference
```

### D. 多证据一致性

```text
structured == title
structured == OCR
title == OCR
number_of_independent_sources
number_of_conflicting_refs
```

### E. 模型不确定性

```text
extractor_margin
LLM_self_consistency
ensemble_agreement
logit_entropy
```

### F. 数据漂移

```text
source_version
crawl_template_version
brand_newness
out_of_distribution_score
```

### G. 下游污染成本

```text
existing_entity_size(canonical_ref)
number_of_cross_source_members
```

最后两项尤其重要：如果一个错误 assignment 会把记录错误接入一个有几千条成员的大 reference entity，它的风险成本比落到一个孤立 key 更高。

---

# 12. 推荐的 loss 定义

## V1：binary loss

最直接：

```python
L = int(predicted_canonical_reference != gold_canonical_reference)
```

并把以下情况统一视为错误：

```text
reference role 错
brand 错
canonicalization 错
把 serial 当 reference
把兼容型号当自身 reference
```

优点：

- 可直接使用论文的 binary special case；
- 可直接跑官方 `CS()` + BH；
- 业务解释简单：`1 - SDR` 近似自动放行 reference assignment precision。

## V2：continuous contamination cost

如果希望体现“一条错 assignment 污染很多 pair”的严重程度，可定义：

```python
L = min(
    1.0,
    false_pairs_introduced / contamination_cap
)
```

或者：

```python
L = wrong_assignment * min(1.0, log1p(target_entity_size) / C)
```

这样一个会污染大实体簇的错误，比落到空 key 的错误代价更高。

这时使用完整 `SCoRE_SDR` 就有了比普通 FDR 更强的意义：控制的是**平均业务损失**，不只是错误个数。

---

# 13. 几百条黄金标签应该怎么用

这是落地成败的关键。

**训练集、hard-negative 测试集、conformal calibration 集不能混为一谈。**

建议：

```text
Gold labels
├─ train / fit score model
├─ hard-negative benchmark
└─ calibration set (必须独立)
```

## 13.1 Calibration 样本不能只挑困难样本

SCoRE 的有效性依赖 calibration/test 的 exchangeability 或加权 exchangeability。

如果人工只标：

- 最相似的邻近型号；
- 模型最不确定的样本；
- 所有异常样本；

然后直接拿这些做 calibration，分布就已经被人为选择，不能再把理论保证原样解释成线上总体风险。

正确做法：

### Calibration set

从真实待部署流量中**随机/分层随机抽样**，做 gold reference 标注。

### Hard-negative benchmark

额外主动构造：

```text
同品牌
同系列
reference 编辑距离 1~2
视觉高度相似
配件包含主体 reference
平台 SKU 看起来像 reference
```

这个集用于压力测试，不用于直接替代 exchangeable calibration。

### Training set

困难样本可以大量用于训练 risk scorer / role classifier。

---

# 14. 一个非常现实的限制：几百条标注不足以证明“万分之一错误率”

这一点必须在项目早期讲清楚。

对 binary conformal p-value，最小非零粒度大约是：

```text
1 / (n_calib + 1)
```

如果只有：

```text
n_calib = 300
```

最小粒度约：

```text
0.00332 = 0.332%
```

如果目标是：

```text
α = 0.001  # 0.1% error，99.9% precision
```

通常至少要有约：

```text
n_calib >= 999
```

才开始有合适的有限样本分辨率。

如果希望：

```text
α = 0.0001  # 99.99% precision
```

量级上就需要接近：

```text
10000 calibration labels
```

才能得到足够细的纯统计证据。

因此当前“可以标几百对”的约束意味着：

> **统计校准绝不能承担全部安全责任。**

V1 必须主要靠 deterministic hard guard 实现极高 precision；SCoRE 用于给剩余自动化决策增加风险约束。

这也解释了为什么本方案坚持“reference exact + fail closed”，而不是只提高模型 confidence threshold。

---

# 15. V1 可以直接使用官方 `CS` 接口

官方 README 的 public API 包含：

```python
from SCoRE import SCoRE_MDR, SCoRE_SDR, CS
```

对当前 binary risk，最简单的生产实验：

```python
import numpy as np
from SCoRE import CS

# calibration:
# 1 = reference assignment 错误
# 0 = reference assignment 正确
L_calib = np.asarray([...], dtype=int)

# 越小越安全
S_calib = np.asarray([...], dtype=float)
S_test = np.asarray([...], dtype=float)

selected, pvals = CS(
    (L_calib, S_calib),
    S_test,
    alpha=0.005,
    mult_test=True,
    return_pvals=True,
)
```

`selected` 就是本 batch **允许自动信任 reference assignment** 的记录索引。

后面还不能马上 match，必须再过 deterministic hard guard。

---

# 16. 推荐写一个独立的 `ReferenceSafetyGate`

生产代码不应该让业务层知道 SCoRE 的细节。

接口建议：

```python
class ReferenceSafetyGate:
    def calibrate(self, gold_rows): ...

    def select_batch(self, assignments): ...

    def explain(self, assignment_id): ...
```

输入：

```python
ReferenceAssignment(
    product_id,
    source,
    brand_id,
    raw_reference,
    canonical_reference,
    evidence,
    risk_score,
)
```

输出：

```python
SafetyDecision(
    product_id,
    accepted: bool,
    pvalue_or_evalue,
    alpha,
    calibration_version,
    rule_failures,
    reason,
)
```

这样可确保：

- calibration version 可追溯；
- 模型升级不影响 match entity schema；
- 可以回放历史决策；
- 可同时 A/B 测 `CS` 与 `SCoRE_SDR`。

---

# 17. 在当前场景中，官方 `CS` 还可以做一个等价的向量化加速

官方实现为了通用和可读性，对 test score 做循环。

binary loss 情况下，它实际构造：

```python
p_j = (1 + # { calibration error i: s_i <= s_j }) / (n + 1)
```

因为 calibration 中 `L=0` 的 score 被移出有效比较区间。

所以可以直接：

```python
bad_scores = np.sort(S_calib[L_calib == 1])
counts = np.searchsorted(bad_scores, S_test, side="right")
pvals = (1.0 + counts) / (len(S_calib) + 1.0)
```

复杂度从朴素的：

```text
O(n * m)
```

变成：

```text
sort calibration: O(n log n)
lookup:           O(m log n)
BH sort:          O(m log m)
```

对于 100 万–1000 万级商品，这比逐 pair 算模型分数可扩展得多。

注意：这个优化针对 **binary conformal selection**；完整 continuous SCoRE-SDR 仍按论文/官方实现处理。

---

# 18. Hard Guard：真正实现“宁漏不误”的地方

SCoRE 选中后还要过下列 hard guards：

```python
def hard_guard(a):
    return all([
        a.brand_id is not None,
        a.canonical_reference is not None,
        a.reference_role == "manufacturer_reference",
        a.brand_grammar_valid,
        a.no_reference_conflict,
        a.normalization_rule_is_verified,
        a.not_accessory_compatibility_reference,
        a.not_serial_number,
        a.not_platform_sku,
    ])
```

我建议额外加：

```text
如果 source structured field 和 title/OCR reference 冲突 -> reject
如果同一记录抽到两个不同 manufacturer reference -> reject
如果 canonicalization 用了 fallback/fuzzy rule -> reject
如果 brand unknown -> reject
如果 reference 只由 image similarity 猜出来 -> reject
```

自动匹配只处理：

```text
safe_assignment == true
```

的记录。

---

# 19. 最终 matching 根本不需要模型

当安全层输出：

```text
brand_id = "rolex"
canonical_reference = "126334"
safe_assignment = true
```

实体 key：

```python
reference_key = f"{brand_id}:{canonical_reference}"
entity_id = sha256(reference_key.encode()).hexdigest()
```

所有来源：

```text
雷小安 record A -> rolex:126334
腕表之家 record B -> rolex:126334
奢当家 record C -> rolex:126334
```

直接进入同一个 reference entity。

不是：

```text
A-B model score 0.99
B-C model score 0.97
A-C model score 0.93
```

而是：

```text
A.ref == B.ref == C.ref
```

复杂度从潜在 pairwise 爆炸降成：

```text
O(N) extraction + O(N log N) / hash-key grouping
```

---

# 20. 推荐数据库结构

## 20.1 raw_product

```sql
CREATE TABLE raw_product (
    source              TEXT NOT NULL,
    source_product_id   TEXT NOT NULL,
    crawl_ts             TIMESTAMPTZ NOT NULL,
    title                TEXT,
    description          TEXT,
    brand_raw            TEXT,
    reference_raw_field  TEXT,
    image_urls           JSONB,
    raw_payload          JSONB,
    PRIMARY KEY (source, source_product_id)
);
```

## 20.2 reference_assignment

```sql
CREATE TABLE reference_assignment (
    source                 TEXT NOT NULL,
    source_product_id      TEXT NOT NULL,
    brand_id               TEXT,
    raw_reference          TEXT,
    canonical_reference    TEXT,
    reference_role         TEXT,
    extractor_version      TEXT NOT NULL,
    normalizer_version     TEXT NOT NULL,
    normalization_rule_id  TEXT,
    evidence               JSONB NOT NULL,
    risk_score             DOUBLE PRECISION,
    hard_guard_pass        BOOLEAN NOT NULL DEFAULT FALSE,
    safety_selected        BOOLEAN NOT NULL DEFAULT FALSE,
    calibration_version    TEXT,
    decision_ts            TIMESTAMPTZ,
    PRIMARY KEY (source, source_product_id)
);
```

## 20.3 reference_entity

```sql
CREATE TABLE reference_entity (
    entity_id              TEXT PRIMARY KEY,
    brand_id               TEXT NOT NULL,
    canonical_reference    TEXT NOT NULL,
    created_at             TIMESTAMPTZ NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);
```

## 20.4 entity_membership

```sql
CREATE TABLE entity_membership (
    entity_id              TEXT NOT NULL,
    source                 TEXT NOT NULL,
    source_product_id      TEXT NOT NULL,
    assignment_version     TEXT NOT NULL,
    created_at             TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (source, source_product_id)
);
```

## 20.5 calibration_label

```sql
CREATE TABLE calibration_label (
    sample_id                 BIGSERIAL PRIMARY KEY,
    source                    TEXT NOT NULL,
    source_product_id         TEXT NOT NULL,
    predicted_brand_id        TEXT,
    predicted_reference       TEXT,
    gold_brand_id             TEXT,
    gold_reference            TEXT,
    loss                      DOUBLE PRECISION NOT NULL,
    risk_score                DOUBLE PRECISION NOT NULL,
    sampling_policy           TEXT NOT NULL,
    calibration_version       TEXT NOT NULL,
    labeled_at                TIMESTAMPTZ NOT NULL
);
```

---

# 21. 增量更新架构

100 万–1000 万并不是必须上非常重的实时架构。

V1 推荐：

```text
Crawler
  -> object storage/raw DB
  -> batch normalize/extract
  -> safety gate
  -> exact reference upsert
```

增量记录处理：

```python
for row in new_rows:
    assignment = extract_reference(row)
    features = build_risk_features(assignment)
    score = risk_model.predict(features)

batch_decision = safety_gate.select_batch(assignments)

for assignment in accepted:
    if hard_guard(assignment):
        entity = get_or_create(
            brand_id=assignment.brand_id,
            canonical_reference=assignment.canonical_reference,
        )
        add_membership(entity, assignment.product_id)
    else:
        enqueue_review(assignment)
```

核心实体表是 idempotent upsert，所以持续增量非常自然。

---

# 22. 为什么不能直接在全量千万数据上跑官方 SCoRE-SDR

官方 `SCoRE_SDR` 的优化实现复杂度仍包含：

```text
O(m(n+m))
```

如果一个 batch 的 `m` 是百万级，就不适合直接照搬研究实现。

因此生产建议分三层：

## Layer 1：deterministic easy path

结构化 reference 字段 + 品牌 grammar + 无冲突，直接进入一个极保守路径。

但即使 easy path，也要做 audit sampling。

## Layer 2：binary conformal selection

对需要模型抽取/语义判断的记录，使用高效 p-value 计算 + BH。

## Layer 3：完整 SCoRE-SDR

只用于：

- 高价值品牌；
- 高污染成本 batch；
- continuous business loss 实验；
- 离线风险评估；
- 比较研究。

不要为了“用了论文”而把研究算法无脑放在全量热路径。

---

# 23. Batch 怎么切

SDR 是对一个 test set / batch 的 selected set 定义的。

持续增量时需要明确 deployment unit。

建议初期：

```text
batch = source × crawl_day
```

或：

```text
batch = source × 6h window
```

不要切得太碎，否则每个 batch 的多重检验 power 降低；也不要一次放几百万进入完整 SCoRE-SDR。

同时要注意：论文明确把 online sequential extension 作为未来方向之一。

所以不能声称：

> “每天分别跑一次 batch SDR，就自动得到整个无限时间流上的统一全局 FDR guarantee。”

正确表述应该是：

> 每个经过相应假设的 deployment batch 都进行风险控制，同时对跨 batch 的长期 empirical precision 做独立 monitoring；如果未来确实需要严格 online guarantee，再引入 online e-value/multiple-testing 方法。

---

# 24. 分布漂移：这篇论文另一个很适合三平台的点

三个来源会持续改页面模板、字段和商品结构：

```text
P_calibration(X) != P_current(X)
```

SCoRE 论文专门扩展了 covariate shift。

假设：

```text
dQ/dP(x,y) = w(x)
```

也就是 conditional label mechanism 基本不变，但 covariate distribution 变了。

官方代码提供：

```python
SCoRE_MDR_w(...)
SCoRE_SDR_w(...)
```

使用 calibration/test density-ratio weights。

论文说明：

- 已知权重时可以保留 finite-sample 风险控制；
- 权重是估计的时，理论变成渐近性质；
- 论文还讨论了 doubly robust calibration。

## 实际工程建议

不要一有 drift 就自动信任 estimated-weight 理论。

先建立：

```text
source drift detector
```

feature：

```text
reference_field_missing_rate
reference_length_distribution
grammar_valid_rate
role_classifier_distribution
risk_score_distribution
OCR usage rate
brand mix
new prefix rate
```

如果 drift 超阈值：

```text
1. 暂停/降低 auto-match coverage
2. 抽样重新标注
3. 更新 score model
4. 重新建立 calibration set
5. 必要时再使用 weighted SCoRE
```

这是最符合“不能误匹配”的 fail-closed 策略。

---

# 25. 新品牌 / 新来源应该默认拒识

对于从未见过的新品牌、新抓取模板或新来源：

```text
auto_eligible = false
```

直到：

1. 建立 brand grammar；
2. 做 reference role 样本；
3. 验证 normalizer；
4. 有随机 calibration 数据；
5. hard-negative benchmark 通过。

不要让一个全局 LLM 的高 confidence 直接打开新 source 的自动匹配。

---

# 26. 图像怎么用才安全

图片非常有价值，但应该是 evidence，而不是 identity key。

推荐三个用途。

## 26.1 OCR reference

对：

- 表背；
- 保卡；
- 吊牌；
- 证书；
- 标签；

做 OCR，抽取 reference candidate。

## 26.2 冲突否决

例如：

```text
title -> 126334
structured field -> 126334
back-case OCR -> 126300
```

不应该让“2 票对 1 票”自动通过。

应该：

```text
conflict = true -> abstain
```

因为 precision 优先。

## 26.3 人工 review 排序

用 image similarity 把疑似同系列/相邻 reference 的记录放在一起，提高人工效率。

### 明确禁止

```text
图片 embedding cosine > 0.98 -> same product
```

这种规则对腕表尤其危险：同系列不同 reference 可能视觉几乎一样。

---

# 27. 一条错误 reference 为什么比一个普通分类错误更危险

假设某条记录被错误赋值到：

```text
rolex:126334
```

该实体已经有：

```text
1000 条记录
```

如果下游把 entity 内所有记录视为“同款”，一条 wrong assignment 可以造成大量 false pair relationships。

因此风险不应该只看“一个 record 错一次”。

建议额外做：

```text
entity contamination guard
```

当新 membership 加入大实体时：

```text
if entity_size > T:
    require stronger evidence
```

甚至可以把 `entity_size` 放进 risk score 或 V2 continuous loss。

---

# 28. 候选 reference 冲突图

可以维护一个轻量 conflict graph，不用复杂 entity matching graph。

节点：

```text
(record, candidate_reference)
```

冲突：

```text
同 record 出现多个 incompatible manufacturer reference
```

证据：

```text
structured / title / OCR / model
```

规则：

```text
只要存在 unresolved contradiction
-> record cannot be auto-selected
```

这比做“多数投票”更符合 precision-first。

---

# 29. Hard-negative benchmark 应该怎么造

为了专门测 false positive，建议 benchmark 不做普通随机 pair，而做 hardest cases。

## 29.1 相邻 reference

```text
126334 vs 126300
126334 vs 126333
```

## 29.2 相同系列不同尺寸/材质

外观接近，但 reference 不同。

## 29.3 配件

```text
“适用于 Rolex 126334 的表带”
```

当前商品是表带，不能归到 `126334` watch entity。

## 29.4 平台 SKU

内部 SKU 长得和 reference 一样。

## 29.5 OCR 近邻

```text
0/O
1/I
5/S
8/B
```

## 29.6 标题多型号

卖家写：

```text
“同 126334/126300 系列...”
```

## 29.7 跨品牌形似 reference

如果 brand canonicalization 错，会直接造成灾难性 collision。

### 指标重点

不要主看 F1。

至少看：

```text
Precision@AutoAccepted
FalseAcceptCount
Coverage
Precision by source
Precision by brand
Precision on hard negatives
Precision under drift
Entity contamination incidents
```

---

# 30. Evaluation：必须把 coverage 和 precision 分开

在这个项目里：

```text
precision > recall
```

所以 dashboard 应该画：

```text
x-axis: auto coverage
 y-axis: false assignment rate / precision
```

比较：

```text
1. hand threshold
2. Platt/isotonic probability threshold
3. conformal CS
4. SCoRE-SDR
5. hard guard + CS
6. hard guard + SCoRE-SDR
```

最终目标不是最高 coverage，而是在：

```text
达到风险要求的前提下最大化 coverage
```

这和论文的 selective prediction power 目标完全一致。

---

# 31. 为什么简单 confidence threshold 不够

常见做法：

```python
if model.prob > 0.999:
    auto_match()
```

问题：

1. neural probability 通常没有严格 calibration；
2. source 分布一漂，0.999 的含义就变；
3. 很难解释“0.999 到底保证什么”；
4. 不同品牌/来源 confidence distribution 不一致；
5. pairwise 数量巨大，微小尾部误差会放大。

SCoRE/Conformal Selection 的价值就是：

> 把“模型 confidence”转成一个基于真实 gold calibration 数据的**部署决策规则**。

这也是为什么 risk scorer 可以随便换模型，而 calibration layer 保持相对独立。

---

# 32. 推荐生产 Pipeline

```text
Stage 0 Raw ingest
  保存原始网页字段和图片，不覆盖

Stage 1 Canonical brand
  品牌别名 -> brand_id

Stage 2 Candidate extraction
  structured + title + description + OCR

Stage 3 Role classification
  manufacturer ref / SKU / serial / compatibility...

Stage 4 Conservative normalization
  brand-specific rule only

Stage 5 Evidence conflict detection
  有冲突 -> reject

Stage 6 Risk scoring
  产出 s(x)，越小越安全

Stage 7 Statistical safety gate
  CS / SCoRE-SDR

Stage 8 Deterministic hard guard
  所有必要条件必须满足

Stage 9 Exact key grouping
  brand_id + canonical_reference

Stage 10 Human review
  rejected high-value records

Stage 11 Feedback
  review labels -> training / calibration / benchmark
```

---

# 33. 建议的 Decision State，而不是简单 true/false

每条 reference assignment 用四态：

```text
AUTO_ACCEPT
MANUAL_REVIEW
REJECT_REFERENCE
NO_REFERENCE
```

不要把后 3 种都变成：

```text
match = false
```

因为：

- `NO_REFERENCE` 是不知道；
- `REJECT_REFERENCE` 是证据表明当前 reference 不可信；
- `MANUAL_REVIEW` 是可能可以恢复；
- `AUTO_ACCEPT` 才能进实体 join。

这样下游能正确理解“未匹配”不等于“不同商品”。

---

# 34. 人工复核队列如何最大化价值

人工不是随机处理所有 abstain，而应按业务价值排队：

```text
priority =
    entity_business_value
  * potential_cross_source_gain
  * review_resolvability
```

优先：

- 已在两个来源出现、第三来源疑似同 ref；
- 大品牌/高价值款；
- 只有一个局部冲突即可解决；
- risk score 靠近放行边界。

但注意：

> 这些主动挑出的 review label 可以用于训练，不应无脑混入 exchangeable calibration set。

Calibration 仍要独立随机抽样。

---

# 35. Calibration versioning

每次自动匹配都要记录：

```text
extractor_version
normalizer_version
risk_model_version
calibration_version
alpha
decision_batch_id
```

如果以后发现某个 normalizer 有 bug，可以精确找到受影响 records 并重算，而不是不知道哪些 entity 是由旧逻辑产生的。

建议 entity membership 是可重建派生数据，不要把它当不可逆真相。

---

# 36. 回滚能力

如果发现错误：

```text
canonical_reference assignment invalidated
```

系统应：

1. 移除旧 membership；
2. 重算该 reference entity；
3. 检查被污染 entity；
4. 把同规则版本产生的 assignment 全量扫描；
5. 提升对应 risk feature / blacklist rule。

因为“一个错误可能污染实体簇”，所以必须支持反向 provenance。

---

# 37. V1 实现建议：不要先做大而全模型

我建议两周级别的最小实验可以是：

## Step 1

先抽三源各一批记录，统一 schema。

## Step 2

人工做 500–1000 条 record-level gold reference，而不是只做 pair label。

Gold：

```text
brand
true manufacturer reference
reference source/span
wrong-role flag
```

## Step 3

做一个可解释 risk scorer：

先不需要 neural network，LightGBM / logistic regression 足够。

输入上述 evidence features。

## Step 4

拿独立 calibration split 跑：

```text
hand threshold
CS + BH
```

## Step 5

hard-negative benchmark 强压：

```text
配件
相邻型号
OCR confusion
多型号 title
platform SKU
```

## Step 6

只有：

```text
statistical_selected && hard_guard_pass
```

才生成 entity membership。

## Step 7

对所有 auto-accepted 样本再随机人工 audit。

这一步才是真正判断能否上线。

---

# 38. V2：引入完整 SCoRE continuous risk

等 binary V1 稳定后，可以定义：

```text
L = downstream contamination cost
```

例如：

```python
L = wrong_reference * normalized_expected_false_edges
```

这时同样一个错误：

- 加到孤立 entity，损失低；
- 加到几千成员大 entity，损失高。

完整 SCoRE 的 continuous risk 控制就比传统 binary FDR 更有意义。

可以实验：

```python
SCoRE_SDR(
    (L_calib, S_calib),
    S_test,
    alpha=...,
    gamma=...,
    prune=None,
)
```

官方还实现 `homo` / `hete` boosting。

不过生产初期我**不建议开启随机 boosting**：

- 虽然理论上保持 SDR 控制；
- 但随机放行对业务 audit 不友好；
- 相同记录多次执行可能产生不同选集；
- precision-first 场景更适合 deterministic 决策。

如果后续实验 power 确实不足，再考虑固定 `random_state` 的可重放方案。

---

# 39. V3：covariate-shift / source-specific calibration

三源数据差异大，未来可以比较三种方案：

### A. Global calibration

把 source/brand 作为 score feature，所有数据共享 calibration。

优点：样本量大。

### B. Source-specific calibration

每个平台一个 calibration。

优点：分布更一致；缺点：label 样本被拆小。

### C. Weighted global calibration

用 source/current-batch 的 density ratio weight 做 SCoRE `_w`。

建议从 A 开始，因为当前人工标签只有几百到一千，过早按品牌/来源拆 calibration 会让 finite-sample granularity 变得更差。

---

# 40. 一个实际的 production-fast binary CS 实现草图

下面是针对当前项目更适合的实现形态：

```python
import numpy as np


def conformal_pvalues_binary(loss_calib, score_calib, score_test):
    """
    score 越小越安全。
    loss=1 表示错误 reference assignment。
    """
    loss_calib = np.asarray(loss_calib, dtype=np.int8)
    score_calib = np.asarray(score_calib, dtype=np.float64)
    score_test = np.asarray(score_test, dtype=np.float64)

    bad_scores = np.sort(score_calib[loss_calib == 1])
    bad_before = np.searchsorted(bad_scores, score_test, side="right")

    return (1.0 + bad_before) / (len(loss_calib) + 1.0)


def bh_select(pvals, alpha):
    pvals = np.asarray(pvals)
    m = len(pvals)
    order = np.argsort(pvals, kind="mergesort")
    ps = pvals[order]
    threshold = alpha * np.arange(1, m + 1) / m

    ok = np.flatnonzero(ps <= threshold)
    if len(ok) == 0:
        return np.array([], dtype=np.int64)

    k = ok[-1] + 1
    return order[:k]
```

然后：

```python
pvals = conformal_pvalues_binary(
    loss_calib=L_calib,
    score_calib=S_calib,
    score_test=S_batch,
)

stat_selected = set(bh_select(pvals, alpha=ALPHA))

for i, assignment in enumerate(batch):
    assignment.safety_selected = (
        i in stat_selected
        and hard_guard(assignment)
    )
```

要先用官方 `CS()` 做随机测试验证这份向量化实现的结果完全一致，再放生产。

---

# 41. 与当前需求最匹配的 alpha 策略

由于 literal “0 error” 无法由有限样本 statistical guarantee 证明，建议不要假装一个极小 alpha 就解决问题。

更合理的是分层策略：

```text
Tier A deterministic verified reference
  hard rules extremely strict
  高 coverage

Tier B model-extracted reference
  conformal alpha 较小
  + hard guard

Tier C ambiguous
  100% manual / reject
```

并让 alpha 是配置，不写死。

上线前通过 gold audit 决定：

```text
alpha_candidate = [0.01, 0.005, 0.002, 0.001]
```

查看：

```text
coverage
false accept count
hard-negative false accept
source/brand breakdown
```

如果 calibration size 不支持目标 alpha，就增加随机标注，而不是靠 confidence threshold 假装有更高精度。

---

# 42. 对“几百对 pair 标签”的改造建议

当前 Spec 说可以人工标几百对。

如果只标：

```text
商品 A == 商品 B ?
```

对这个 reference-centric 架构信息不够。

建议把标注任务改成：

```text
Record A:
  gold_brand
  gold_reference
  current reference 是否属于当前商品

Record B:
  gold_brand
  gold_reference
  current reference 是否属于当前商品

Pair:
  same_reference = gold_ref_A == gold_ref_B
```

这样一对标注能同时服务：

- pair matching benchmark；
- reference extractor；
- role classifier；
- risk scorer；
- conformal calibration；
- normalizer evaluation。

单位人工成本价值更高。

---

# 43. 监控指标

线上至少监控：

## Safety

```text
random_audit_precision
auto_accept_error_count
hard_negative_escape_count
entity_contamination_incident
```

## Coverage

```text
auto_accept_rate
manual_review_rate
no_reference_rate
```

## Drift

```text
score PSI
reference field missing rate
brand grammar invalid rate
new reference prefix rate
source template fingerprint change
```

## Calibration

```text
calibration_size
calibration_error_rate
min_achievable_conformal_p
selected risk by batch
```

## Data quality

```text
multiple_reference_conflict_rate
OCR/text disagreement
platform-SKU confusion rate
```

---

# 44. Kill switch

必须有自动匹配 kill switch。

以下情况直接关自动放行：

```text
source HTML template changed
reference field missing rate sudden jump
brand mapping version rollback
normalizer bug detected
random audit false positive > threshold
new unrecognized reference pattern spike
risk score distribution severe shift
```

恢复需要：

```text
reprocess + re-audit + recalibrate
```

这个设计比“线上继续跑，之后清理”更符合 precision-first。

---

# 45. SCoRE 的优点

对当前项目：

1. **模型无关**：extractor/risk model 可替换；
2. **允许 abstention**：完全符合宁漏不误；
3. **binary / continuous risk 都支持**；
4. **有限样本 risk control**；
5. **可处理 batch selection**；
6. **有 covariate-shift 扩展**；
7. **官方代码已发布，可直接实验**；
8. **风险控制和模型训练解耦**；
9. 非常适合“高风险决策只自动处理一部分”。

---

# 46. SCoRE 的局限

也必须明确：

1. 它不是 entity matcher；
2. 它不能让错误概率变成绝对 0；
3. exchangeability 假设必须认真处理；
4. biased active-learning labels 不能直接当 calibration；
5. 极小 alpha 需要更多 calibration 样本；
6. 完整 SDR 研究实现对超大 batch 不够轻；
7. 持续无限流的严格 online guarantee 不是本文已解决问题；
8. score 质量差时虽然可能仍 valid，但 coverage 会很低；
9. estimated covariate weights 下的保证弱于 known-weight finite-sample；
10. e-value/BH 风险保证是期望意义，不是每个 batch 都必然无错。

因此不能把论文包装成“用了 conformal 就绝不会误匹配”。

---

# 47. 最推荐的最终方案

综合论文、官方实现和当前业务约束，我建议落成下面这个版本：

## 核心算法

```text
Reference-centric entity resolution
+ conservative brand-specific canonicalization
+ reference role classification
+ evidence conflict hard reject
+ binary conformal selection / SCoRE safety gate
+ exact canonical key join
+ human abstention queue
```

## 自动匹配的必要条件

```text
1. brand 已 canonicalized
2. manufacturer reference 已抽取
3. reference role 正确
4. brand grammar 通过
5. normalization 是 verified rule
6. 无多证据冲突
7. statistical safety gate 通过
8. canonical reference exact equal
```

只有 1–8 全部满足，才自动合并。

---

# 48. 推荐落地顺序

## Phase 0：一周

```text
统一三源 schema
建立 brand mapping
建立 reference evidence schema
实现 conservative normalizer
```

## Phase 1：一周

```text
500–1000 条 record-level gold
rule/LightGBM risk scorer
hard-negative benchmark
官方 CS baseline
```

## Phase 2

```text
ReferenceSafetyGate
exact key entity store
人工 review queue
random audit dashboard
kill switch
```

## Phase 3

```text
完整 SCoRE-SDR continuous contamination loss
covariate-shift experiment
OCR / image evidence
更强 role classifier
```

## Phase 4

```text
扩大 random calibration labels
降低目标 alpha
按来源/品牌做更细风险策略
长期 drift/recalibration
```

---

# 49. 最后的架构判断

如果业务定义已经明确：

> “同一个商品”就是同一个 reference number

那系统应该尽量避免重新发明一个通用 entity matching 问题。

真正需要解决的是两个子问题：

```text
A. 从脏、多模态、多来源数据中可靠地得到 canonical reference
B. 只在足够可靠时自动相信该 canonical reference
```

A 用：

```text
规则 + NER/LLM + OCR + 品牌 grammar + role classifier
```

B 用：

```text
SCoRE / Conformal Selection + hard guard + abstention
```

然后实体匹配本身：

```text
brand_id + canonical_reference exact join
```

这是当前 Spec 下比“端到端多模态 pair matcher”更稳、更可解释、更可扩展、也更容易真正做到 precision-first 的架构。

**SCoRE 最适合放在 matcher 之前，成为 reference assignment 的统计安全阀。**

---

## 参考

- Tian Bai, Ying Jin. *Conformal Selective Prediction with General Risk Control*. arXiv:2603.24704, 2026. https://arxiv.org/abs/2603.24704
- Official SCoRE code: https://github.com/Tian-Bai/SCoRE
- Python package shown in official README: `score-select`
