# How to Fix a Broken Confidence Estimator: Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks

> 分析人：c  
> 论文：Luís Felipe P. Cattelan, Danilo Silva, **How to Fix a Broken Confidence Estimator: Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks**, UAI 2024 / PMLR 244:547–584  
> 论文主页：https://proceedings.mlr.press/v244/cattelan24a.html  
> arXiv：https://arxiv.org/abs/2305.15508  
> 官方代码：https://github.com/lfpc/FixSelectiveClassification  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级、持续增量；字段高度稀疏；“同一商品”定义为**同一 reference number / 型号**；图片可用；可人工标几百对；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 1. 选题与去重

执行前先检查 `奢侈品调研/c/`。截至本次分析前，c 已覆盖：

- Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes
- AnyMatch – Efficient Zero-Shot Entity Matching with a Small Language Model
- Confidence Classifiers with Guaranteed Accuracy or Precision
- DeepBlocker
- End-to-end multi-modal product matching in fashion e-commerce
- Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration
- GraLMatch：Matching Groups of Entities with Graphs and Language Models
- Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification
- PAM：Understanding Product Images in Cross Product Category Attribute Extraction
- Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce
- Tailoring entity resolution for matching product offers
- TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency
- Using LLMs for the Extraction and Normalization of Product Attribute Values
- parts-distributor-sku-classifier
- pyJedAI

本次论文对应 `奢侈品文章调研.md` 中这一条推荐：

> “研究如何在不重训主模型的情况下修复置信度估计器，提升 selective classification，并覆盖分布漂移场景；适合三个来源持续增量、品牌和数据分布不断变化时，单独校准拒识分数来减少边界误匹配。”

该论文此前未由 c 分析，因此符合去重要求。

它与 c 已分析的 **Confidence Classifiers with Guaranteed Accuracy or Precision** 不是重复关系，而是上下游互补：

- 本论文解决：**模型排序错误——高分样本未必比低分样本更可信，怎样只改 logits 就修复 confidence ranking。**
- 前一篇 conformal 论文解决：**给定一个 confidence/ranking 后，怎样通过校准和拒识控制被接受集合的风险。**

因此最合理的组合并不是二选一，而是：

```text
Base model logits
    -> post-hoc confidence repair（本论文）
    -> hard business guards
    -> calibration / conformal admission gate（前一篇）
    -> AUTO_MATCH / REVIEW / REJECT
```

这正好补足当前 precision-first 方案里“主模型高分不等于可信”的薄弱环节。

---

## 2. 论文到底解决什么问题

普通深度分类器通常输出 logits：

```text
z = [z1, z2, ..., zC]
```

然后通过 softmax 得到概率：

```text
p_k = exp(z_k) / sum_j exp(z_j)
```

最自然的 confidence 是 Maximum Softmax Probability：

```text
MSP(z) = max softmax(z)
```

Selective Classification 的流程则是：

```text
if confidence >= threshold:
    ACCEPT prediction
else:
    ABSTAIN
```

问题在于：**分类准确率高，不代表 confidence 能正确排序“哪些预测更容易错”。**

论文发现部分 ImageNet 强模型存在非常反直觉的现象：

- 模型 top-1 accuracy 很高；
- 但使用 MSP 做拒识时，越往高置信区筛选，风险并没有稳定下降；
- 个别模型的 Risk-Coverage Curve 甚至在低 coverage 区出现非单调；
- 也就是说模型会把一批错误预测放在“最自信”的位置。

这就是论文所谓的 **broken confidence estimator**。

对当前腕表需求，完全存在同构风险：

```text
cross-encoder score = 0.9997
```

并不能推出：

```text
P(同 reference | score=0.9997) = 99.97%
```

更不能推出：

```text
所有 score >= 0.9997 的自动匹配几乎不会出现 false positive
```

尤其以下 hard cases 很容易产生“错误但高分”：

- 同品牌同系列、只差 reference 尾码；
- 标题出现兼容型号而非当前商品型号；
- 平台 SKU 被错误识别为 manufacturer reference；
- OCR 把 `126610LN` 读成 `126610LV`；
- 同外观不同 reference；
- 模型只看到品牌、系列、图片非常相似，却没有真正读懂 reference；
- 新品牌/新来源分布漂移后，模型仍输出很高 softmax。

论文的价值是：**不用重训主模型，只重新定义如何从 logits 计算 confidence，就可能显著改善“正确样本在前、错误样本在后”的排序。**

---

## 3. 核心方法：MaxLogit-pNorm

### 3.1 公式

官方实现定义：

```text
z_c = z - mean(z)
```

然后做 p-norm normalization：

```text
z_norm = z_c / ||z_c||_p
```

最终 confidence：

```text
g(z) = max(z_norm)
```

也就是：

```text
g(z) = max_k (z_k - mean(z)) / ||z - mean(z)||_p
```

直观上它做了两件事：

1. **去平移**：减掉均值，让所有 logits 加同一个常数不会改变 confidence；
2. **去尺度**：除以整个 logit 向量的 p-norm，减少不同模型/样本 logit 整体放大造成的虚假高置信。

最终关注的是：

> top logit 在当前整个 logit 几何结构中有多突出，而不是 softmax 指数化以后被放大到多接近 1。

官方代码核心只有几行：

```python
def centralize(logits):
    return logits - logits.mean(-1).view(-1, 1)


def normalize(logits, p, centralize_logits=True):
    if centralize_logits:
        logits = centralize(logits)
    return torch.nn.functional.normalize(logits, p, -1)


def MaxLogit_pNorm(logits, p):
    logits = centralize(logits)
    return normalize(logits, p, False).max(-1).values
```

它的工程优点非常明显：

- 不需要重训 encoder / matcher；
- 不需要额外网络；
- 不改变原模型预测类别 `argmax(z)`；
- 只增加一次均值、范数和 max 计算；
- 可以作为现有 matcher 的旁路组件快速上线验证。

### 3.2 p 怎么选

官方仓库不是手工拍一个 `p=2`，而是在 hold-out 数据上网格搜索：

```python
p = post_hoc.optimize.p(logits, risk, metric=metric)
```

实现里：

```text
p_range = 0,1,...,9
```

对每个 p：

```text
confidence = max_logit(normalize(logits, p))
metric_value = metric(confidence, risk)
```

选择 metric 最优的 p。

而且官方实现默认有：

```text
MSP_fallback = True
```

即如果所有 pNorm 都比原始 MSP 差，就直接退回 MSP，不强行启用新方法。

这个设计非常适合生产系统：**新 confidence estimator 必须以验证集指标证明自己更安全，否则不改变线上行为。**

---

## 4. Risk-Coverage 比“概率校准”更贴合本需求

论文重点不是 Expected Calibration Error，而是 selective classification：

```text
Coverage = 被系统自动处理的比例
Risk     = 被接受集合中的错误率
```

随着 confidence threshold 增大：

```text
coverage ↓
risk 理想情况下也 ↓
```

论文使用 Risk-Coverage Curve，并以 AURC / normalized AURC 等指标比较 confidence estimator。

官方代码 `RC_curve` 的核心就是：

```python
confidence, indices = confidence.sort(descending=True)
risk = risk[indices]
risks = risk.cumsum(0) / coverage_count
```

这与当前业务目标高度一致，因为我们真正需要的问题不是：

> `score=0.99` 是否真的是 99% 概率？

而是：

> **如果只把最可信的前 0.1%、1%、5% 自动合并，被放行集合里会不会出现 false positive？**

因此建议内部评估也切换为 selective metrics，而不是只看：

```text
Accuracy / F1 / ROC-AUC
```

应该重点看：

```text
FP count @ AUTO_MATCH
Precision @ coverage
Risk-Coverage Curve
Coverage @ target precision
Worst-stratum precision
```

这里尤其要注意：本需求“允许漏匹配”，所以 coverage 本来就是可牺牲变量；这正是 selective classification 最适合的风险结构。

---

## 5. 最重要的迁移结论：不能把论文方法直接套在二分类 MATCH / NO_MATCH 上

这是本论文迁移到实体匹配时最容易踩的坑，也是本次分析最关键的结论。

假设 pairwise matcher 只有两个 logits：

```text
z = [a, b]
```

中心化以后：

```text
mean = (a+b)/2
z_c = [(a-b)/2, (b-a)/2]
```

令：

```text
d = a-b
```

则：

```text
z_c = [d/2, -d/2]
```

对于任意正常的 `p > 0`：

```text
||z_c||_p
= (|d/2|^p + |-d/2|^p)^(1/p)
= |d| * 2^(1/p - 1)
```

而最大中心化 logit：

```text
max(z_c) = |d|/2
```

因此 MaxLogit-pNorm：

```text
g(z)
= (|d|/2) / (|d| * 2^(1/p - 1))
= 2^(-1/p)
```

**结论：对二分类中心化 logits，它对所有非 tie 样本都是常数。**

也就是说如果直接这样做：

```text
pair -> [MATCH_logit, NO_MATCH_logit]
     -> MaxLogit-pNorm
```

那么几乎所有 pair 的 confidence 一样，完全失去排序能力。

所以：

> **该论文最有价值的算法不能直接作为二分类 pair matcher 的 confidence。**

这也是为什么本需求最合理的迁移不是继续做 pairwise 二分类，而是把问题重构为 **候选 reference 的多类选择 + NONE/ABSTAIN**。

建议线上实现直接加防呆：

```python
assert logits.shape[-1] >= 3, \
    "Centered MaxLogit-pNorm degenerates for binary logits; use candidate-set logits."
```

---

## 6. 最适合本需求的重构：从“pair 是否匹配”变成“这个 listing 属于哪个 reference”

Spec 已经明确：

```text
同一个商品 = 同一 reference number / 型号
```

因此实际上没必要让模型最终直接做：

```text
listing A + listing B -> MATCH / NO_MATCH
```

更安全的建模是：

```text
listing
   -> canonical reference assignment
      -> 同 canonical reference 的记录再合并
```

对于每条 listing，先构造一个小候选集：

```text
R = [ref_1, ref_2, ..., ref_K, NONE]
```

模型输出：

```text
z = [z_ref1, z_ref2, ..., z_refK, z_none]
```

这里类别数：

```text
C = K + 1 >= 3
```

MaxLogit-pNorm 就不再退化，可以真正衡量：

> top candidate 在整个 reference 候选集合里是否足够突出。

这样做还把业务规则与 ML 模型天然对齐：

```text
模型只负责 reference 消歧
最终实体合并仍由 canonical_reference exact equality 完成
```

这比让一个黑盒 matcher 直接决定两个商品“是否同款”更可审计、更容易追错，也更符合“绝不能误匹配”。

---

## 7. 推荐的完整生产架构

```text
                    ┌──────────────────────────┐
3-source raw data ->│ 1. Normalize & Parse     │
                    │ title / fields / OCR     │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │ 2. Reference Candidate   │
                    │ Generator                │
                    │ brand-scoped top-K       │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │ 3. Candidate-set Scorer  │
                    │ K refs + NONE logits     │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │ 4. MaxLogit-pNorm        │
                    │ confidence repair        │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │ 5. Hard Evidence Guards  │
                    │ exact ref / brand / OCR  │
                    │ conflict veto            │
                    └────────────┬─────────────┘
                                 │
                                 v
                    ┌──────────────────────────┐
                    │ 6. Calibrated Admission  │
                    │ conformal / risk gate    │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────┴─────────────┐
                    │                          │
                    v                          v
              AUTO_ASSIGN_REF              REVIEW
                    │
                    v
          canonical_reference equality
                    │
                    v
             cross-source merge
```

关键原则：

> **pNorm 可以提高 confidence 排序质量，但它永远不能绕过 reference 硬证据，也不能自己充当“零误匹配保证”。**

---

## 8. 各层具体怎么实现

## 8.1 Layer 1：Reference Normalize & Evidence Extraction

每条商品先生成统一结构：

```json
{
  "listing_id": "...",
  "source": "leixiaoan|xbiao|shedangjia",
  "brand_raw": "ROLEX",
  "brand_canonical": "rolex",
  "title": "劳力士 潜航者 126610LN ...",
  "structured_reference_raw": null,
  "title_reference_tokens": ["126610LN"],
  "ocr_reference_tokens": ["126610LN"],
  "platform_sku_tokens": ["SDJ-983771"],
  "image_ids": ["..."],
  "evidence": []
}
```

reference normalization 必须是**保守规范化**，只处理确定不改变语义的形式差异：

```text
大小写统一
Unicode 统一
首尾空格
确定性的分隔符规则
品牌特定已验证 alias
```

禁止模糊地做：

```text
126610LN -> 126610LV
```

或把 edit distance 很近直接视为同 reference。

对于腕表，尾码往往就是区分变体的核心信息。

## 8.2 Layer 2：候选 reference 生成

候选生成一定先按 brand 限定：

```text
listing brand = Rolex
    -> 只查 Rolex reference dictionary
```

候选来源可按强弱排序：

1. 独立 reference 字段 exact normalized lookup；
2. title token exact lookup；
3. OCR token exact lookup；
4. brand-specific regex / trie；
5. 受限 fuzzy candidate retrieval，仅用于召回，不用于确认；
6. 同系列/相邻 reference hard negative 候选补充。

建议固定：

```text
K = 5 或 K = 10
```

然后永远附加：

```text
NONE
```

原因不只是工程简单，还与 pNorm 有关：**不同类别数 C 的 pNorm confidence 分布并不天然可直接比较。**

如果 K 动态变化，至少应：

```text
按 K 分 bucket 分别调 p 和 threshold
```

更推荐直接固定 K，并用 mask / dummy candidates 保持 logit 向量长度一致。

## 8.3 Layer 3：Candidate-set Scorer

这里不需要从零训练一个巨型模型。

可以对每个 `(listing, candidate_reference)` 构造特征：

```text
structured_ref_exact
structured_ref_conflict
title_ref_exact
title_ref_token_source
title_ref_edit_distance
ocr_ref_exact
ocr_ref_conflict
brand_exact
series_consistent
platform_sku_like_penalty
candidate_frequency
source_reliability
text_cross_encoder_score
image_aux_score
```

然后得到每个 candidate 的 scalar logit：

```text
s_i = scorer(listing, ref_i)
```

再额外学习或规则生成：

```text
s_none
```

拼成：

```text
z = [s_1, ..., s_K, s_none]
```

可选模型：

- LightGBM / XGBoost：几百标注时更稳、可解释；
- 小型 MLP：特征足够结构化时成本低；
- cross-encoder：只用于 top-K，成本可控；
- LLM：只做离线 feature/extraction，不建议直接作为最终自动合并判定器。

在只有几百黄金标签的条件下，我更推荐：

```text
规则特征 + LightGBM/小 MLP + listwise candidate logits
```

而不是端到端训练视觉/文本大模型。

## 8.4 Layer 4：MaxLogit-pNorm confidence

生产版建议自己实现一个很薄的函数，而不是强绑定论文仓库：

```python
import torch


def maxlogit_pnorm(logits: torch.Tensor, p: float) -> torch.Tensor:
    if logits.shape[-1] < 3:
        raise ValueError(
            "Centered MaxLogit-pNorm is not useful for binary logits; "
            "use K references + NONE."
        )

    z = logits - logits.mean(dim=-1, keepdim=True)
    denom = torch.linalg.vector_norm(
        z, ord=p, dim=-1, keepdim=True
    ).clamp_min(1e-12)
    z = z / denom
    return z.max(dim=-1).values
```

保留：

```text
predicted_reference = argmax(original logits)
```

confidence 只由 pNorm 生成：

```text
confidence = maxlogit_pnorm(logits, p)
```

不要用归一化后的 logits 改 prediction，因为论文方法本来就是 post-hoc confidence estimator，不应该改主模型类别。

---

## 9. 不要照搬论文的 AURC 调参目标，要改成 precision-first metric

官方实现 `optimize.p` 接受任意 `metric`：

```python
optimize.p(logits, risk, metric=metric)
```

论文研究的是通用 selective classification，所以主要关注 AURC / NAURC。

但本业务损失函数明显不对称：

```text
False Positive（误合并） >> False Negative（漏合并）
```

因此线上应该把 `p` 的选择目标改成：

```text
maximize coverage subject to FP risk <= epsilon
```

或者：

```text
minimize risk in highest-confidence region
```

示意代码：

```python
def coverage_at_target_risk(confidence, wrong, target_risk):
    order = torch.argsort(confidence, descending=True)
    wrong = wrong[order].float()

    n = torch.arange(1, len(wrong) + 1, device=wrong.device)
    prefix_risk = torch.cumsum(wrong, 0) / n
    coverage = n.float() / len(wrong)

    ok = prefix_risk <= target_risk
    if not ok.any():
        return 0.0
    return coverage[ok].max().item()
```

然后：

```text
best_p = argmax_p coverage_at_target_risk(...)
```

同时保留官方的 fallback：

```text
if pNorm 不优于 MSP：继续用 MSP
```

这可以非常低风险地插入现有系统。

---

## 10. pNorm score 不是概率，更不是安全保证

这一点必须写进设计文档，否则非常容易误用。

论文解决的是：

```text
confidence ranking quality
```

不是：

```text
P(correct | score)
```

也不是：

```text
P(FP) <= 10^-6
```

因此不能写：

```python
if pnorm_score > 0.95:
    auto_match()
```

然后把 `0.95` 解读成 95% 置信概率。

正确方法是：

```text
pNorm score
    -> 在独立 calibration set 上选择 threshold
    -> 再用 conformal / binomial risk bound / conservative rule gate
```

本项目可以直接与 c 已分析的 Mondrian conformal 方案串起来：

```text
listwise matcher logits
    -> MaxLogit-pNorm score
    -> calibration cohort by source-pair / brand / evidence-type
    -> conformal / empirical acceptance threshold
```

从而让本论文承担：

```text
把错误样本尽量排到低 confidence
```

让 conformal / risk layer 承担：

```text
决定到底放行多少
```

职责更清晰。

---

## 11. “几百黄金标签”能做什么，不能做什么

Spec 允许人工标注几百对，这足够：

- 比较 MSP vs pNorm 的排序质量；
- 调一个很低维的 `p`；
- 训练轻量 candidate scorer；
- 构造 hard-negative validation；
- 粗略确定 conservative threshold；
- 找出明显坏品牌/坏来源 bucket。

但**几百条标签不足以统计证明“绝对零误匹配”**。

例如某个自动放行策略在 300 条验证样本中 0 FP，也不意味着真实 FP rate 接近 0；小样本下真实风险的置信上界仍然不可忽略。

所以“绝不能误匹配”的工程实现必须来自多层设计，而不是只靠验证集分数：

```text
1. canonical reference exact invariant
2. 编号角色识别，SKU/平台 ID 不能冒充 reference
3. evidence conflict veto
4. listwise confidence repair
5. calibrated reject gate
6. OOD / drift fallback to REVIEW
7. 自动合并可追溯、可回滚
```

机器学习只负责减少 REVIEW，不负责取消这些不变量。

---

## 12. Hard Evidence Guards：pNorm 只能帮助放行，不能越权

建议 AUTO_ASSIGN_REF 必须满足全部条件：

```text
A. top candidate != NONE
B. brand canonical exact
C. candidate reference 通过品牌 reference grammar
D. 至少一个高可信文本/OCR证据能映射到该 canonical reference
E. 没有任何高可信证据明确指向另一个 reference
F. pNorm confidence >= bucket threshold
G. calibrated admission gate pass
H. 非 OOD / 非 drift quarantine bucket
```

### 12.1 强冲突直接否决

例如：

```text
structured_ref = 126610LN
title_ref      = 126610LN
ocr_ref        = 126610LV
```

不应该让模型多数投票后直接选 LN；应该：

```text
CONFLICT -> REVIEW
```

尤其 OCR 来源如果是表背/保卡，则可能比标题更可信。

### 12.2 图片只做辅助，不得推翻 reference

图片可用于：

- 候选排序；
- 检测“明显不是这个系列”；
- OCR 表背/保卡/吊牌；
- 冲突否决；
- 人工 REVIEW 排序。

不建议：

```text
image similarity high
    -> 即使 reference 不一致也 MATCH
```

因为腕表同系列不同 reference 外观可能非常接近，而 Spec 又把 reference 定义为最终真值。

---

## 13. 数据集怎么构造：必须大量加入“几乎一样但 reference 不同”的 hard negatives

如果随机采负样本，任务会太容易：

```text
Rolex vs Cartier
126610LN vs HPI00716
```

模型很快就会学到品牌/系列，离线上分数极高，却完全不会处理真正危险的 FP。

黄金集建议至少 60% 是 hard negatives：

```text
同品牌
同系列
同尺寸
同颜色/材质
图片近似
标题高度重合
reference 只差 1-3 个字符
```

例如：

```text
126610LN vs 126610LV
116610LN vs 126610LN
M126710BLRO vs M126710BLNR
```

另外单独加入：

```text
reference vs platform SKU
reference vs compatibility model
OCR one-char error
缺 reference
多个 reference 同时出现在标题
配件/盒证/表带标题携带主表型号
```

这些样本才真正决定 pNorm 排序在业务里有没有价值。

---

## 14. 推荐的 calibration 分桶

论文证明 post-hoc 方法在 distribution shift 下仍有价值，但本业务不能因此假设“一个全局 threshold 永久有效”。

建议至少按以下维度分 bucket：

```text
source
source_pair / ingestion pipeline
brand
candidate_set_size K
reference evidence type
OCR present / absent
structured reference present / absent
```

不一定所有维度做笛卡尔积；可以做层级回退：

```text
brand + source + evidence
    -> 样本不足
brand + evidence
    -> 样本不足
evidence
    -> global
```

每个 bucket 保存：

```json
{
  "model_version": "resolver-v3",
  "confidence_method": "maxlogit_pnorm",
  "p": 3.0,
  "threshold": 0.7421,
  "fallback": "MSP",
  "calibration_set_version": "gold-2026-08-18",
  "accepted_count": 182,
  "observed_fp": 0
}
```

上线后如果出现：

```text
score distribution shift
reference grammar hit-rate shift
NONE rate shift
manual-review FP
```

则直接：

```text
raise threshold
或 quarantine bucket -> REVIEW
```

不要自动降低 threshold 去保 coverage。

---

## 15. 持续增量场景的处理

100 万–1000 万商品、持续增量，pNorm 本身几乎没有性能压力。

对每条 listing，假设固定：

```text
K = 10
```

则 pNorm 只需：

```text
10 维均值
10 维范数
10 维 max
```

复杂度：

```text
O(K)
```

真正昂贵的是：

- OCR；
- 图片 embedding；
- cross-encoder；
- candidate retrieval。

因此可以将 pNorm 放在 scorer 后同步执行，不需要独立服务。

推荐流式架构：

```text
Kafka / queue
  -> normalize worker
  -> ref-candidate service
  -> scorer batch inference
  -> pNorm + guardrail service/library
  -> accepted reference table / review queue
```

其中 pNorm 最好实现成共享 library，而不是网络 RPC 服务，减少复杂度。

---

## 16. 推荐的数据表与可审计字段

最终每条自动决策必须能回答：

> 为什么它被自动分配到这个 reference？

建议保存：

```sql
listing_resolution (
    listing_id,
    source,
    brand_canonical,

    candidate_refs_json,
    raw_logits_json,
    predicted_ref,
    predicted_rank,

    confidence_method,
    pnorm_p,
    pnorm_score,
    msp_score,
    top2_margin,

    structured_ref_evidence,
    title_ref_evidence,
    ocr_ref_evidence,
    image_conflict_score,

    hard_guard_pass,
    hard_guard_reason,
    calibration_bucket,
    calibration_threshold,

    decision,            -- AUTO_ASSIGN / REVIEW / REJECT
    model_version,
    rules_version,
    created_at
)
```

这样后续如果发现某品牌规则错误，可以按：

```text
model_version + rules_version + calibration_bucket
```

快速定位和回滚受影响记录。

---

## 17. 一个可以直接落地的推理伪代码

```python
def resolve_listing(listing):
    # 1. normalize / extract
    evidence = extract_reference_evidence(listing)
    brand = normalize_brand(listing)

    if brand is None:
        return review("missing_brand")

    # 2. candidate retrieval
    candidates = retrieve_reference_candidates(
        brand=brand,
        evidence=evidence,
        k=10,
    )

    # Always add NONE and keep a fixed candidate-set size.
    candidates = pad_to_fixed_k(candidates, k=10)
    classes = candidates + [NONE]

    # 3. listwise logits
    logits = candidate_scorer(listing, evidence, classes)
    pred_idx = logits.argmax(-1)
    pred_ref = classes[pred_idx]

    # 4. post-hoc confidence repair
    p = calibration_registry.get_p(
        brand=brand,
        source=listing.source,
        evidence_type=evidence.type,
    )
    score = maxlogit_pnorm(logits[None, :], p=p)[0]

    # 5. hard vetoes
    guard = validate_reference_assignment(
        predicted_ref=pred_ref,
        evidence=evidence,
        brand=brand,
    )
    if not guard.pass_:
        return review(guard.reason)

    # 6. threshold / calibrated admission
    threshold = calibration_registry.get_threshold(...)
    if score < threshold:
        return review("low_selective_confidence")

    if not conformal_or_risk_gate_pass(...):
        return review("risk_gate_reject")

    # 7. assign canonical reference; merge happens later by exact equality
    return auto_assign_reference(
        canonical_reference=pred_ref,
        audit={
            "pnorm_score": float(score),
            "p": p,
            "evidence": evidence,
            "logits": logits,
        },
    )
```

最终跨源合并不要重新过模型：

```sql
SELECT canonical_reference, array_agg(listing_id)
FROM resolved_listing
WHERE decision = 'AUTO_ASSIGN'
GROUP BY canonical_reference;
```

即：

```text
ML 负责 canonical reference assignment
数据库 exact equality 负责 merge
```

---

## 18. p 的离线选择与上线门槛

建议流程：

### Step 1：准备 calibration set

严格按 entity/reference 划分，避免同 reference 同时进入 train/calibration 导致泄漏。

最好再按时间做一份未来批次验证：

```text
train: old batch
calibration: later batch
stress-test: newest batch
```

模拟持续增量 distribution shift。

### Step 2：比较 baseline

至少比较：

```text
MSP
max logit
logit margin
MaxLogit-pNorm
```

因为官方代码已经提供：

```text
MSP
max_logit
margin_logits
margin_softmax
entropy / negative_entropy
energy
```

不要只证明 pNorm 比“没有 confidence”强，而要证明它比当前最简单 baseline 强。

### Step 3：只优化低风险区

我们最关心：

```text
Risk <= target
```

而不是整条 RC 曲线平均。

可以记录：

```text
coverage @ 100% observed precision
coverage @ 99.9% observed precision
FP count in top 0.1%
FP count in top 1%
worst-brand FP count
```

### Step 4：必须有 fallback

如果：

```text
pNorm 没改善
或某个 bucket validation 太少
或 drift 监控报警
```

则：

```text
MSP / stricter hard-rule-only / REVIEW
```

不能因为系统已经接入 pNorm 就默认它永远生效。

---

## 19. 与已有 c 方案的组合关系

本论文最适合放在现有方案的中间，而不是替代已有模块。

### 19.1 与 DeepBlocker / pyJedAI

```text
DeepBlocker / Blocking
    -> 解决“不要做全量笛卡尔积”

本论文
    -> 解决“候选已经出来后，模型 confidence 是否能正确排序风险”
```

### 19.2 与 Ameli / 多模态属性抽取

```text
Ameli / PAM / OCR
    -> 补 reference / 属性证据

本论文
    -> 不新增证据，只修正已有模型的拒识排序
```

### 19.3 与 TransClean / GraLMatch

```text
TransClean / GraLMatch
    -> merge 后从图一致性发现可疑错误边

本论文
    -> merge 前尽量把最危险的预测拒绝掉
```

### 19.4 与 Confidence Classifiers with Guaranteed Accuracy or Precision

两者最值得直接串联：

```text
Base resolver
    -> MaxLogit-pNorm：修 ranking
    -> Mondrian / conformal：校准 accepted MATCH cohort
    -> auto admission
```

可以把 pNorm score 作为 conformal nonconformity / ranking 的一个输入，或者直接用 pNorm 排序后再做经验风险校准。

一句话：

> **pNorm 负责让“谁更可信”排得更对；conformal 负责决定“可信到什么程度才允许自动放行”。**

---

## 20. 需要特别防的工程陷阱

### 20.1 二分类退化

这是第一优先级防呆。pairwise binary logits 不能直接使用 centered MaxLogit-pNorm。

### 20.2 动态 K 导致 confidence 不可比

固定 K，或者按 K 分桶校准。

### 20.3 把 pNorm 当概率

严禁。它只是排序 score。

### 20.4 用随机负样本调 p

会得到虚假的“完美 confidence”。必须用同系列 hard negatives。

### 20.5 只看总体 AURC

总体表现改善不代表 top 0.1% 自动放行区更安全。必须单独看极低风险 operating point。

### 20.6 用图片相似度覆盖 reference 冲突

禁止。图片只辅助或否决。

### 20.7 新品牌继续沿用老 threshold

新品牌、新来源、新 OCR 模型、新解析规则都应触发重新校准或 REVIEW-only。

### 20.8 NONE 类训练不足

NONE 非常重要。候选集中真实 reference 不存在时，模型必须有能力拒绝，而不是被迫选择最像的错误 reference。

NONE 训练样本至少包括：

- reference 缺失；
- 全部候选都错；
- 新 reference 未进入 catalog；
- OCR 严重错误；
- 平台 SKU 被误抽取；
- 配件标题只有兼容型号。

---

## 21. 推荐上线判定矩阵

| 场景 | reference 证据 | pNorm | 风险校准 | 动作 |
|---|---|---:|---:|---|
| 独立字段 exact，其他证据无冲突 | 强 | 高 | pass | AUTO_ASSIGN |
| 标题 exact + OCR exact 一致 | 强 | 高 | pass | AUTO_ASSIGN |
| 标题 exact，但 OCR 指向另一个 ref | 冲突 | 任意 | 任意 | REVIEW |
| 只有图片相似，无 reference 文本/OCR证据 | 弱 | 高 | pass | REVIEW |
| top1 / top2 相邻 reference 很接近 | 中 | 低 | fail | REVIEW |
| candidate set 不含真实 ref，NONE top1 | 合理拒识 | 高 | pass | REJECT/UNRESOLVED |
| 新品牌/新来源 bucket | 未知 | 高 | 未校准 | REVIEW |
| binary pair matcher 直接输出两类 logits | 任意 | 不可用 | 任意 | 不启用 pNorm |

这个矩阵体现核心思想：

> **confidence 只能加强自动化，不能削弱硬规则。**

---

## 22. 最小可实施版本（MVP）

不需要先重构整个系统，可以先做一个旁路实验：

```text
现有 candidate scorer
    -> 保存 top-K candidate scalar scores
    -> 拼成 K + NONE logits
    -> 离线计算 MSP / margin / pNorm
    -> 在几百黄金标签上画 RC curve
    -> 比较极低风险区 coverage
```

如果 pNorm 明显更好，再上线：

```text
pnorm score + conservative threshold
```

如果无明显收益：

```text
fallback MSP / margin
```

因为本方法完全 post-hoc，这个实验不要求先重训主模型，验证成本很低。

真正值得做的改造只有一个：

> **把原来的 pairwise binary 输出保留作 candidate feature，但新增一个 K-reference + NONE 的 listwise resolution head。**

这既解决 pNorm 二分类退化，也更贴近“最终赋 canonical reference”的业务定义。

---

## 23. 最终推荐方案

本论文不应该被理解为“又一个置信度技巧”，而应该用于重构实体匹配决策边界：

### 推荐最终链路

```text
[三源 listing]
    ↓
保守 reference extraction / OCR / 编号角色分类
    ↓
brand-scoped candidate references (fixed top-K)
    ↓
K references + NONE listwise scorer
    ↓
MaxLogit-pNorm 修复 confidence ranking
    ↓
reference / brand / evidence hard guards
    ↓
Mondrian conformal 或 conservative empirical risk gate
    ↓
AUTO_ASSIGN canonical reference
    ↓
canonical reference exact equality merge
```

### 为什么它适合当前 Spec

1. **precision-first**：天然允许大量 abstain；
2. **不重训主模型也可加**：适合已有 scorer 快速增强；
3. **成本极低**：百万到千万规模都不是瓶颈；
4. **适合持续增量**：confidence layer 可独立重调，不必重训整个系统；
5. **与图片兼容**：图片可进入 scorer，但不能覆盖 reference 硬规则；
6. **几百标签可启动**：p 只有低维超参数，调参数据要求远小于端到端大模型；
7. **可与 conformal 串联**：先修排序，再做风险控制；
8. **可审计**：每次自动赋 reference 都能保存 candidate logits、pNorm score、证据和阈值。

### 最重要的限制

> **论文的 MaxLogit-pNorm 在中心化二分类 logits 上会退化为常数，因此不能直接加在 MATCH / NO_MATCH pair classifier 后。必须改成多候选 reference + NONE 的输出空间，或者只借鉴论文“post-hoc 修 confidence”的方法论而使用其他 binary confidence。**

这不是小实现细节，而是决定方案能不能工作的结构性条件。

---

## 24. 结论

如果当前系统已经有一个准确率不错的 candidate matcher，但线上仍然担心：

```text
“为什么这个 0.999 的分数可以信？”
```

本论文给出的最有用答案不是“0.999 就可信”，而是：

> **先检查 confidence estimator 是否真的能把错误排到后面；必要时只用 logits 做 post-hoc 修复，再通过拒识把不可信部分全部交给人工。**

对跨源腕表实体匹配，我建议直接落地为：

```text
candidate reference listwise resolution
+ MaxLogit-pNorm ranking
+ hard reference invariant
+ calibrated abstention
```

而不是：

```text
pairwise MATCH probability > 某阈值就合并
```

最终安全边界仍然应该是：

```text
AUTO merge
= canonical reference 一致
AND reference 证据无冲突
AND confidence ranking 足够高
AND calibrated risk gate 通过
```

其余全部 `REVIEW / ABSTAIN`。

这套方案与 Spec 的风险偏好完全一致：**宁可少合并，也不要错合并。**