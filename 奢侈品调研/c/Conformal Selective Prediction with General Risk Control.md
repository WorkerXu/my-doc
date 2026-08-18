# Conformal Selective Prediction with General Risk Control

> 分析人：c  
> 论文：Tian Bai, Ying Jin, **Conformal Selective Prediction with General Risk Control**, 2026  
> 论文：https://arxiv.org/abs/2603.24704  
> 官方代码：https://github.com/Tian-Bai/SCoRE  
> Python 包：`score-select`  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”严格定义为**同一 reference number / 型号**；字段稀疏且 reference 可能埋在标题或图片中；图片可用；允许几百对人工黄金标签；**precision 极端优先，绝不能误匹配，允许漏匹配**。

---

## 1. 选题与去重

执行前先检查 `奢侈品调研/c/`。c 已经覆盖的方向包括：多模态实体链接、DeepBlocker、AnyMatch、TransClean、GraLMatch、属性抽取/规范化、商品 offer matching、置信度估计与拒识、pyJedAI 等；当前目录中**不存在** `Conformal Selective Prediction with General Risk Control.md`，因此本次分析不重复既有结果。

本次选择这篇论文/项目的原因，是它正好补当前方案最关键的一层：

> **不是再造一个 matcher，而是给“哪些模型结果可以自动落库”增加一个带有限样本风险控制的 admission controller（准入控制器）。**

现有的实体匹配论文大多优化 F1、Recall、AUC 或 pairwise accuracy，但本需求的损失函数极不对称：

- false negative：漏掉一个同 reference 的匹配，可以以后人工补、再次召回；
- false positive：把两个不同 reference 错合并，会污染 canonical entity，后续还会通过聚类/传递关系扩大污染。

所以最有价值的问题不是“怎样把模型分数再提高一点”，而是：

```text
给定一个已训练的 matcher / reference extractor，
怎样只部署其中风险足够低的一小部分结果，
其余全部 ABSTAIN / REVIEW，
并且对被自动接受集合的误匹配风险做统计控制？
```

SCoRE（Selective Conformal Risk control with E-values）正是为这个问题设计的。

---

## 2. 论文解决的核心问题

论文把普通预测系统拆成两层：

```text
prediction model
      │
      ├── prediction / candidate
      └── risk-estimating score s(x)
                  │
                  ▼
        conformal risk controller
                  │
          ┌───────┴────────┐
          ▼                ▼
       DEPLOY            ABSTAIN
```

这里的关键不是强迫主模型永远输出正确结果，而是允许系统拒绝。

设每个候选 `j` 有损失：

```text
L_j ∈ [0, 1]
```

在腕表实体匹配里最自然的定义是：

```text
L_j = 0  自动 MATCH 后确实是同一 canonical reference
L_j = 1  自动 MATCH 后实际是不同 reference（false positive）
```

然后系统学习/构造一个风险分数 `s(x)`：

```text
分数越小 => 越安全
```

SCoRE 不要求这个分数本身是“真实错误概率”，而是用独立 calibration set 把分数转换成**risk-adjusted e-value**，再进行假设检验，决定哪些实例可以部署。

### 2.1 MDR 与 SDR

论文定义两类风险目标。

#### Marginal Deployment Risk（MDR）

关注“一个随机样本被部署且出错”的总体风险：

```text
MDR = E[L · ψ]
```

其中 `ψ=1` 表示部署，`ψ=0` 表示拒绝。

这更适合“每条任务独立控制是否执行”的场景。

#### Selective Deployment Risk（SDR）

关注**被选择出来部署的集合内部平均风险**：

```text
SDR = E[ Σ L_j ψ_j / max(1, Σ ψ_j) ]
```

对本项目，如果 `L=1` 就表示 false match，则：

```text
SDR ≈ 自动 MATCH 集合中的 false-positive rate
    = 1 - precision
```

因此本需求应该优先使用 **SDR**，因为业务真正关心的是：

> 所有被系统自动声称“同 reference”的结果里，有多少是错的？

这比整体 accuracy 更符合“precision-first”。

---

## 3. SCoRE 的技术原理

## 3.1 输入：校准数据、风险损失和 risk score

SCoRE 的最小输入不是原始图片或标题，而是：

```text
D_calib = {(L_i, S_i)}
D_test  = {S_j}
```

其中：

- `L_i`：人工确认的 calibration loss；
- `S_i`：校准样本风险分数；
- `S_j`：待部署样本风险分数；
- score 越小代表越安全。

这意味着它可以很容易接在任意 matcher 后面：

```text
LightGBM / XGBoost
Cross Encoder
LLM judge
reference extractor
多模态模型
规则 + 模型融合器
        │
        ▼
统一输出 risk_score
        │
        ▼
SCoRE
```

论文强调：score 的质量影响**能选出多少样本（power / coverage）**，但在满足方法假设时，风险控制并不要求 score 一定等于真实概率。

对本业务，一个更合理的 `risk_score` 不应只来自单一模型概率，而应该来自“reference 证据包”：

```text
risk_score = f(
    reference_source,
    canonical_ref_exactness,
    extraction_confidence,
    number_role_confidence,
    source_pair,
    title/OCR agreement,
    brand consistency,
    series consistency,
    hard-negative similarity,
    image conflict score,
    matcher margin
)
```

注意：**真正的 reference 冲突不能只作为一个软特征。它必须先 hard reject。**

---

## 3.2 Risk-adjusted e-value

论文构造非负 e-value `E`，满足核心条件：

```text
E ≥ 0
E[L · E] ≤ 1
```

直觉上：

- 一个样本越可能安全，它的 e-value 越大；
- e-value 不是普通概率；
- 它被设计成可以直接放进统计检验中控制 deployment risk。

论文构造中会在 calibration score 与 test score 上寻找一个数据依赖阈值 `t_γ`，使阈值以内的经验风险足够低，再根据 calibration losses 构造该测试点的 e-value。

对于二元风险 `L∈{0,1}`，可理解为：

```text
只要 test 的 risk score 足够低，
并且 calibration 数据显示“这么低的分数区域里很少出错”，
则该 test 获得较大的 e-value。
```

这比手写：

```python
if p_match > 0.995:
    auto_merge()
```

更可靠，因为 `0.995` 本身并没有“线上 99.5% precision”的统计含义，而 e-value 是由 calibration errors 对当前 score 区域重新校准出来的。

---

## 3.3 MDR：单样本准入

论文证明，如果 `E` 是 risk-adjusted e-value，则用：

```text
ψ = 1{E ≥ 1/α}
```

即可控制 MDR。

这意味着可以把它理解为一个“自动执行许可”：

```text
风险不够低 -> 不执行
风险足够低 -> 执行
```

官方代码提供：

```python
SCoRE_MDR(Dcalib, Dtest, alpha, gamma)
```

以及显式计算 e-value 的 brute-force 版本：

```python
SCoRE_MDR_bf(...)
```

官方实现还提供了计算快捷路径。对于生产系统，这个模式很适合：

- 单条/小批量增量决策；
- 延迟要求高；
- 不希望为了每个微批次重新做完整 batch selection。

但从本需求“自动 MATCH 集合的 precision”看，**SDR 的业务语义更直接**。

---

## 3.4 SDR：控制被自动接受集合的平均风险

SCoRE 对一批 test candidate 先计算 e-values，然后通过 **e-BH**（e-value 版本的 multiple testing）选择部署集合。

可理解成：

```text
输入 m 个待自动 MATCH 候选
        │
        ▼
每个候选得到 e-value
        │
        ▼
按 e-value 强弱排序
        │
        ▼
e-BH 自适应决定最终接受多少个
        │
   ┌────┴────┐
   ▼         ▼
AUTO_MATCH  ABSTAIN
```

相比固定阈值，它的优势是：

- 接受数量由**这一批候选的整体证据强度**决定；
- 不需要假设每个候选互相独立；
- 可以直接以 `alpha` 表达可容忍的 selective risk。

官方代码入口：

```python
SCoRE_SDR(Dcalib, Dtest, alpha, gamma,
          prune=None,
          return_evals=False,
          random_state=None)
```

README 推荐 `gamma=alpha` 作为默认起点。

官方实现明确包含 score tie 的修正逻辑，这一点很重要：生产中大量 exact-rule candidate 的 risk score 可能完全相同，如果对 ties 的处理不一致，线上同一批数据可能产生不稳定选择结果。

---

## 3.5 Covariate shift：三平台持续增量时非常关键

本项目不会长期满足“所有来源同分布”：

```text
雷小安      标题模板 / 字段覆盖 / 图片风格 A
腕表之家    标题模板 / 字段覆盖 / 图片风格 B
奢当家      标题模板 / 字段覆盖 / 图片风格 C
```

而且品牌差异也非常明显：

```text
Rolex reference 格式
Omega reference 格式
Cartier reference 格式
AP reference 格式
```

官方实现提供 weighted 版本：

```python
SCoRE_MDR_w(...)
SCoRE_SDR_w(...)
```

通过 calibration/test 的 covariate-shift weights 做加权控制。

这对持续增量系统有直接价值，但需要谨慎：

1. 权重估计错了会降低实际可信度；
2. 极端权重会导致方差巨大；
3. 论文中估计权重场景的性质与“已知真实权重”不同，不能把它包装成业务上的绝对保证。

因此生产优先级建议是：

```text
优先：分层 calibration + abstain
其次：稳定、可解释的 density-ratio weighting
最后才是：完全依赖黑盒 drift reweighting
```

---

## 4. 这篇论文不能原样当成“实体匹配算法”

这是本次调研最重要的边界。

SCoRE 解决的是：

> **已有候选结果里，哪些可以安全部署？**

它不解决：

- reference number 如何抽取；
- 型号字符串怎样品牌内规范化；
- 哪个数字是品牌 reference、哪个是平台 SKU；
- 千万商品怎样 blocking；
- 图片怎样 OCR；
- 同系列近邻 reference 怎样 hard-negative 建模。

因此错误架构是：

```text
标题/图片
   -> 黑盒多模态 matcher
   -> SCoRE
   -> 自动合并
```

正确架构应该是：

```text
Reference-first + Hard Constraints + SCoRE-last
```

也就是：**SCoRE 是最后一道统计风控门，不是 reference truth 的替代品。**

---

## 5. 针对 Notion Spec 的直接落地架构

推荐完整链路：

```text
┌────────────────────────────────────────────┐
│ 1. Ingestion                              │
│ 雷小安 / 腕表之家 / 奢当家                 │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 2. Listing Normalization                  │
│ 品牌、标题、结构化字段、图片、来源 ID      │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 3. Reference Evidence Extraction          │
│ structured field / title / OCR / description│
│ 输出候选 reference + provenance            │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 4. Number Role Classification             │
│ brand reference / platform SKU / 店铺货号  │
│ accessory model / serial / unknown         │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 5. Brand-aware Canonicalization           │
│ 保留有语义 suffix，禁止过度 normalize       │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 6. Hard Gate                              │
│ ref conflict / brand conflict => REJECT    │
│ ref missing/ambiguous => REVIEW            │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 7. Candidate Retrieval                    │
│ brand + canonical_reference index          │
│ 必要时加轻量 blocking                      │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 8. Candidate Risk Scorer                  │
│ 生成越小越安全的 risk_score                │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│ 9. SCoRE Admission Gate                   │
│ MDR: 单条准入 / SDR: 微批 precision 风控   │
└───────────────────┬────────────────────────┘
                    ▼
       ┌────────────┴────────────┐
       ▼                         ▼
 AUTO_MATCH                  ABSTAIN/REVIEW
       │                         │
       ▼                         ▼
canonical entity          人工复核回流 calibration
```

### 核心原则

**只有 reference 证据成立，SCoRE 才有资格决定是否自动落库。**

如果存在可信的 reference 冲突：

```text
126610LN  != 126610LV
```

无论图片多像、标题 embedding 多像、matcher score 多高：

```text
直接 NO_MATCH
```

不能让统计模型“覆盖”品牌 reference 的硬冲突。

---

## 6. Reference Evidence 层如何设计

建议不要只存一个：

```text
reference = "126610LN"
```

而是存完整 provenance：

```json
{
  "raw_value": "126610 LN",
  "canonical_value": "126610LN",
  "brand_id": "rolex",
  "role": "brand_reference",
  "source": "title",
  "extractor_version": "ref-extractor-v3",
  "span": "劳力士潜航者 126610 LN",
  "confidence": 0.98,
  "normalizer_version": "rolex-ref-v2"
}
```

同一 listing 允许多个 evidence：

```text
structured_ref = 126610LN
标题抽取       = 126610LN
OCR 表背       = 126610LN
```

这时证据互相一致，是强 positive evidence。

如果：

```text
structured_ref = 126610LN
标题抽取       = 126610LV
```

必须形成：

```text
REFERENCE_CONFLICT
```

并进入 REVIEW，而不是做平均分。

### 特别重要：canonicalization 不能过度

腕表 reference 的 suffix 往往有业务语义。

不能把：

```text
126610LN
126610LV
```

都简化成：

```text
126610
```

建议：

```text
global normalization
    只处理 Unicode、空白、明显分隔符、大小写

brand-specific normalization
    再根据品牌规则处理合法前后缀
```

所有 normalization 都保留：

```text
raw_value
canonical_value
rule_version
```

以便回溯。

---

## 7. 千万级数据的候选生成：不要做 pairwise 全比较

三个来源总量 100 万–1000 万时，全量笛卡尔积不可行，而且没有必要。

既然业务定义明确：

> 同一商品 = 同一 reference number

则主索引应该直接围绕：

```text
(brand_id, canonical_reference)
```

建立。

### 强证据路径

```text
listing
  -> canonical brand
  -> unique canonical ref
  -> hash / B-tree lookup
  -> candidate canonical entity
```

复杂度基本接近线性 ingest，而不是 `O(N²)` pair comparison。

### 弱证据路径

reference 缺失或有多个候选时才使用：

- brand + series blocking；
- BM25 / lexical retrieval；
- embedding ANN；
- image retrieval；
- cross encoder。

但这些弱召回候选**只进入 REVIEW / evidence enrichment**，不能直接自动合并。

这样可以把昂贵模型集中在少量疑难样本上。

---

## 8. SCoRE 应该控制什么风险

### 8.1 最简单版本：pair false match

对每个候选 pair：

```text
L = 0  两条 listing 的真实 reference 相同
L = 1  真实 reference 不同
```

risk scorer 输出：

```text
s(pair)
```

越小越安全。

然后：

```python
selected = SCoRE_SDR(
    Dcalib=(calib_losses, calib_scores),
    Dtest=batch_scores,
    alpha=alpha,
    gamma=alpha,
)
```

只有：

```text
hard_reference_gate == PASS
AND index in selected
```

才允许自动 MATCH。

### 8.2 更推荐：把“reference hypothesis 错误”也纳入 score

因为本项目最大的风险不是两个已有 reference 的字符串比较，而是：

> 从脏标题/图片里抽错 reference，却被后续 exact match 当成真值。

所以建议 risk score 至少包含：

```text
reference extraction risk
number-role risk
normalization ambiguity
cross-field conflict
cross-modal conflict
pair matcher risk
```

最后控制的是“这次 canonical entity 归属是否安全”，而不只是文本 pair 是否相似。

---

## 9. 可直接复用的代码方案

官方仓库已封装 PyPI 包：

```bash
pip install score-select
```

一个最小生产原型可以这样写：

```python
import numpy as np
from SCoRE import SCoRE_SDR


def hard_reference_gate(candidate):
    # 任何可信 reference 冲突都不可自动合并
    if candidate.brand_conflict:
        return False
    if candidate.reference_conflict:
        return False
    if candidate.reference_role != "brand_reference":
        return False
    if not candidate.shared_canonical_reference:
        return False
    return True


def score_candidate(candidate):
    # 约定：越小越安全
    # 实际可由 LightGBM / calibrated ranker / small MLP 输出
    return candidate.risk_score


def select_auto_matches(calib_rows, candidates, alpha):
    safe_candidates = [c for c in candidates if hard_reference_gate(c)]

    if not safe_candidates:
        return []

    lcalib = np.asarray([r.loss for r in calib_rows], dtype=float)
    scalib = np.asarray([r.risk_score for r in calib_rows], dtype=float)
    stest = np.asarray([score_candidate(c) for c in safe_candidates], dtype=float)

    selected_idx = SCoRE_SDR(
        (lcalib, scalib),
        stest,
        alpha=alpha,
        gamma=alpha,
        prune="homo",
        random_state=2026,
    )

    return [safe_candidates[i] for i in selected_idx]
```

注意三点：

1. `risk_score` 是**越小越安全**，不要把 `p_match` 不经变换直接传进去；
2. calibration rows 必须和 scorer 训练集分离；
3. SCoRE 前面仍有 hard gate。

如果现有 matcher 只有：

```text
p_match
```

最简单可先用：

```python
risk_score = 1.0 - p_match
```

但更推荐单独训练一个 error/risk scorer，让它重点识别 false-positive hard cases。

---

## 10. Risk scorer 推荐实现

本项目不需要先上大模型，最实用的首版可以用 LightGBM/XGBoost。

输入特征：

### reference 特征

```text
ref_exact
ref_edit_distance
ref_prefix_equal
ref_suffix_equal
ref_length_diff
structured_ref_agree
ocr_ref_agree
title_ref_agree
reference_conflict_count
```

### 编号角色特征

```text
number_role_brand_ref_prob
number_role_platform_sku_prob
number_role_serial_prob
number_role_accessory_ref_prob
```

### 品牌/系列

```text
brand_equal
series_equal
brand_parser_confidence
```

### 文本

```text
title_token_jaccard
BM25 score
cross_encoder score
```

### 图片

```text
image_similarity
OCR conflict
visual_reference_present
```

### 来源

```text
source_pair
crawler_version
field_coverage_pattern
```

模型输出 raw risk score，SCoRE 再使用 calibration loss 做最后准入。

这样模型负责：

> 排出“谁更危险”。

SCoRE 负责：

> 在当前 calibration 证据下，“哪些危险程度低到可以自动部署”。

两个职责解耦。

---

## 11. 几百对黄金标签应该怎么标

如果只随机抽几百 pair，大概率都是太容易的负例，几乎没有意义。

黄金数据应该主动覆盖**最容易制造 false positive 的边界**：

### 必须包含的 hard negatives

```text
同品牌 + 同系列 + reference 只差 1 个字符
同一基础数字 + 不同 suffix
同外观不同尺寸 reference
同款不同代 reference
腕表 vs 表带/表盒/配件
标题同时出现“当前商品 reference”和“兼容型号 reference”
平台 SKU 伪装成型号
序列号被误当 reference
OCR 字符混淆：0/O、1/I、5/S、8/B
标题 reference 与图片 OCR reference 冲突
```

### 必须包含的 positives

```text
不同来源不同格式但 canonical reference 相同
空格/连字符/大小写差异
品牌缩写差异
reference 只存在于标题
reference 只存在于结构化字段
reference 只在 OCR 中可见
```

### 数据切分

至少保持：

```text
train
calibration
final audit holdout
```

严格分离。

不要用 calibration set 调完几十次规则后仍声称它是独立 calibration；规则和模型一旦根据这批数据反复调整，这批数据就被“训练化”了。

---

## 12. 一个非常关键的现实限制：几百条标签不足以证明 99.9%+ precision

这是本需求最容易误判的地方。

业务写“绝对不能误匹配”，但任何有限样本统计方法都不能把：

```text
有限 calibration 样本
```

变成：

```text
每一条未来数据数学上绝不出错
```

而且当目标 `alpha` 极小时，有限样本会导致明显的分辨率/统计功效问题。

例如只有几百个校准样本时，即便校准样本里一个错误也没看到，也不能据此可信宣称：

```text
线上 false-positive rate < 0.1%
```

跨品牌、跨 source pair 再切 bucket 后，每个 bucket 的样本更少。

因此正确的上线策略应该是：

### 阶段 A：确定性自动合并

只放行：

```text
可信 structured reference
+ 品牌一致
+ brand-aware canonical reference exact match
+ 无任何冲突
```

### 阶段 B：模型只做 shadow / review ranking

标题抽取、OCR、图片、LLM 等结果先不自动 merge，积累 hard-case 标签。

### 阶段 C：SCoRE 逐步扩大自动化覆盖

当某个 evidence regime 的 calibration 数据足够时，再为它打开很小的自动接受集合。

这比一开始就追求很高 coverage 安全得多。

---

## 13. Calibration 不应该只按一个全局池做

理想 taxonomy：

```text
source_pair
× brand_family
× evidence_regime
× extractor_version
```

例如：

```text
雷小安 -> 腕表之家 | Rolex | structured_ref_exact
雷小安 -> 奢当家   | Rolex | title_ref_exact
腕表之家 -> 奢当家 | Omega | OCR+title_agree
```

但几百条黄金标签不足以切得这么碎，所以建议采用层级回退：

```text
L3: source_pair + brand + evidence_regime
      ↓ 样本不足
L2: source_pair + evidence_regime
      ↓ 样本不足
L1: evidence_regime
      ↓ 样本仍不足
ABSTAIN
```

不要为了“总有一个阈值”而强行使用样本量极小的 bucket。

---

## 14. 持续增量与分布漂移的生产策略

每一个自动决策必须记录：

```text
policy_version
risk_model_version
reference_extractor_version
normalizer_version
calibration_snapshot_id
alpha
gamma
risk_score
e_value（如果使用返回 e-value 的接口）
hard_gate_result
decision_reason
```

推荐建立 drift monitor：

```text
source field coverage 变化
reference 缺失率变化
reference 字符长度分布变化
risk score 分布变化
品牌占比变化
OCR 成功率变化
人工 review FP 率变化
```

触发明显 drift 时：

```text
AUTO_MATCH bucket -> shadow / review-only
```

直到重新校准。

这是“绝不能误匹配”系统应有的 fail-closed 行为。

---

## 15. SCoRE 在千万级上的性能边界

官方代码对 SDR 给出的优化复杂度仍包含：

```text
O(m(n+m) + (n+m) log(n+m))
```

其中：

- `n`：calibration 数量；
- `m`：当前 test batch 数量。

因此绝不能把千万 candidate 一次性塞进 `SCoRE_SDR`。

好在本系统根本不需要这么做：

### 先通过 reference index 把候选数量压到很低

```text
10M listings
  -> reference extraction
  -> hard filtering
  -> indexed candidate lookup
  -> 少量“可能自动匹配”的 candidates
  -> SCoRE
```

### 再做 micro-batch

建议按：

```text
source_pair + evidence_regime + policy_version
```

形成微批次。

批量大小不要写死，应该用真实基准测试决定；原则是：

- 不让 SDR 变成 ingest 性能瓶颈；
- 又保证每批有足够候选让 selective selection 有意义。

如果业务必须逐条低延迟，则用 `SCoRE_MDR` 作为准入层；如果目标是“接受集合 precision”更可解释，则定时用 `SCoRE_SDR` 做微批提交。

---

## 16. 推荐的数据表结构

### `listing`

```text
id
source
source_listing_id
brand_raw
brand_id
title
description
structured_fields
image_urls
crawler_version
created_at
updated_at
```

### `reference_evidence`

```text
listing_id
raw_value
canonical_value
brand_id
role
source_type        # structured/title/ocr/description
span_or_image_id
confidence
extractor_version
normalizer_version
conflict_group
```

### `canonical_reference_entity`

```text
entity_id
brand_id
canonical_reference
status
created_at
```

核心唯一性建议围绕：

```text
(brand_id, canonical_reference)
```

而不是只用裸 reference。

### `match_candidate`

```text
left_listing_id
right_entity_id
shared_reference
risk_score
hard_gate_status
hard_gate_reason
risk_model_version
```

### `match_decision`

```text
candidate_id
decision            # AUTO_MATCH / REVIEW / REJECT
alpha
gamma
score
e_value
calibration_snapshot_id
policy_version
decided_at
```

### `calibration_sample`

```text
candidate_id
human_label
loss
score_at_label_time
source_pair
evidence_regime
brand_id
reviewer
labeled_at
```

---

## 17. 数据库层也要做最后一道防线

“零误合并”不能只靠模型服务约定。

建议 canonical entity 写入事务包含：

1. 再次读取当前 canonical reference；
2. 校验 listing 的 accepted reference 未发生版本变化；
3. 校验不存在 hard conflict；
4. 校验 decision policy / calibration snapshot 仍有效；
5. 原子提交 listing → entity 关系；
6. 写审计日志。

如果任何版本在候选生成后发生变化：

```text
fail closed
```

重新评估，而不是沿用旧决策。

---

## 18. 图片在本方案中的正确角色

图片非常有价值，但不能越过 reference 定义。

推荐作用：

### 1. OCR reference evidence

```text
表背
保卡
吊牌
标签
```

抽取型号作为额外 reference evidence。

### 2. conflict veto

如果标题声称某 reference，但视觉/OCR 明显显示另一个 reference：

```text
REVIEW
```

### 3. hard-negative 特征

同系列不同 reference 外观很像，所以视觉 similarity 高不能成为充分匹配条件；但明显不相似可以提高风险分数。

因此应遵循：

```text
image can veto / support
image cannot override a trusted reference conflict
```

---

## 19. 与 c 之前置信度方案的关系

c 已分析过 `Confidence Classifiers with Guaranteed Accuracy or Precision`。两者并不重复，而是可以形成递进关系：

### Confidence Classifier 思路

重点是：

```text
分类器需要拒识
高模型概率不能直接当高 precision
conformal confidence 可以校准接受集合
```

### SCoRE 的进一步价值

SCoRE 把问题推广为：

```text
一般 bounded loss
+ selective deployment
+ e-value hypothesis testing
+ MDR / SDR risk control
+ covariate-shift variants
```

也就是说：

> 前者更像“可信度估计/拒识”；SCoRE 更适合直接实现为生产中的“部署风险控制器”。

对于本项目，建议把 SCoRE 放在 policy engine 里，而不是塞进 matcher 模型本身。

---

## 20. 推荐的首版技术栈

技术栈不限时，一版足够务实的实现：

```text
采集/队列：Kafka / Pulsar（已有队列则直接复用）
处理服务：Python worker + FastAPI 管理接口
结构化主库：PostgreSQL
分析/审计：ClickHouse 或 Parquet + DuckDB
图片：S3 / MinIO
OCR：独立 worker，可 GPU batch
候选索引：Postgres B-tree/hash；弱召回需要时再加 Elasticsearch/FAISS
risk scorer：LightGBM / XGBoost
selective gate：score-select / 自建经测试的 SCoRE wrapper
模型与规则版本：MLflow 或内部 registry
```

重点不是堆模型，而是把：

```text
reference provenance
hard gate
calibration snapshot
decision audit
```

做成一等公民。

---

## 21. 一个可直接实现的 Decision Policy

建议线上决策明确成 4 类，不要只有 MATCH / NO_MATCH：

```text
AUTO_MATCH
REVIEW
REJECT
NO_CANDIDATE
```

伪代码：

```python
def decide(candidate, score_gate):
    if candidate.brand_conflict:
        return "REJECT"

    if candidate.trusted_ref_conflict:
        return "REJECT"

    if candidate.ref_role != "brand_reference":
        return "REVIEW"

    if not candidate.unique_shared_canonical_ref:
        return "REVIEW"

    if candidate.reference_ambiguity:
        return "REVIEW"

    if not score_gate.accept(candidate):
        return "REVIEW"

    return "AUTO_MATCH"
```

这里的 `score_gate.accept()` 才由 SCoRE 控制。

这个顺序非常关键：

> **hard rule 负责“绝对禁止什么”，SCoRE 负责“剩余候选里哪些风险足够低”。**

---

## 22. 上线验收指标不要再看总体 F1

至少分成：

### 自动区

```text
AUTO_MATCH precision
AUTO_MATCH count / coverage
false positive count
按 source_pair / brand / evidence_regime 的 precision
```

### 拒识区

```text
REVIEW volume
review hit rate
review disagreement rate
```

### reference extraction

```text
reference exact accuracy
reference abstention rate
number-role confusion matrix
canonicalization conflict rate
```

### 系统风险

```text
calibration age
bucket sample size
drift alerts
policy fallback count
```

首要 KPI 应是：

```text
AUTO_MATCH false positive = 0 on audited samples
```

而不是为了 coverage 去牺牲 hard gate。

---

## 23. 建议的落地顺序

### P0：先做 reference authority

- 品牌规范化；
- reference evidence schema；
- number-role 分类；
- brand-aware normalizer；
- `(brand_id, canonical_reference)` 索引；
- conflict veto；
- 只开放最强 exact evidence 自动合并。

### P1：建立黄金 hard-case 数据

- 优先标同系列近邻 reference；
- 标平台 SKU / 配件 / OCR 混淆；
- train/calibration/audit 严格分开。

### P2：训练 risk scorer + SCoRE shadow

- matcher/risk scorer 只做候选风险排序；
- SCoRE 输出拟接受集合；
- 暂不自动写 canonical relation；
- 和人工结果对比。

### P3：只开放最安全 bucket

例如：

```text
structured_ref_exact
+ title_ref_agree
+ no_conflict
+ SCoRE accept
```

### P4：随着人工回流逐步扩大 evidence regime

可以再考虑：

```text
title-only
OCR+title
multimodal-assisted
```

但每扩大一类都单独 shadow、校准和验收。

---

## 24. 关键风险与防护

### 风险 1：Calibration data 被反复调参污染

**防护：** calibration 和 audit holdout 固定版本；每次 scorer / rule 改版重新建立 snapshot。

### 风险 2：几百条数据切 bucket 后太少

**防护：** 层级回退；不足则直接 abstain，不强行自动化。

### 风险 3：来源或品牌分布漂移

**防护：** drift monitor + fail closed；必要时再使用 weighted SCoRE。

### 风险 4：错误 canonicalization 把不同 reference 合并

**防护：** brand-specific rule；保留原始值；suffix 不随意删除；规则版本化。

### 风险 5：模型对 hard negative 过度自信

**防护：** hard-negative 主动采样；risk scorer + SCoRE；reference conflict 永远 hard reject。

### 风险 6：同一 listing 产生大量相关 candidate

**防护：** candidate 去重、唯一 shared reference、entity-level transactional constraint；不要靠“pair 分数都高”决定聚类。

### 风险 7：一条错边污染整个实体簇

**防护：** canonical entity 以 reference authority 为中心，不允许单纯通过图传递关系创建跨 reference merge。

---

## 25. 最终建议

这篇论文最值得直接落地的不是某个新的实体匹配网络，而是一个架构角色：

```text
SCoRE = Statistical Admission Controller
```

对当前跨源二奢/腕表实体匹配需求，推荐最终原则是：

```text
1. Reference is authority
2. Candidate retrieval is recall-oriented
3. Hard conflicts are non-negotiable
4. Models estimate risk, not truth
5. SCoRE controls which low-risk candidates may auto-deploy
6. Everything else abstains
7. Human review continuously expands calibration evidence
```

因此最可落地的方案不是“让 SCoRE 判断两块表是不是同款”，而是：

> **先用 brand-aware reference evidence 系统证明两条记录确实共享同一 reference，再让 SCoRE 决定当前证据包是否足够安全到可以自动落库。**

这与“绝对不能误匹配、允许漏匹配”的约束完全一致：

- 硬规则负责业务意义上的零容忍；
- SCoRE 负责模型区间的有限样本统计风控；
- abstention 是正常输出，不是失败；
- 图片、LLM、embedding 都只能提高召回或提供辅助证据，不能覆盖 reference 冲突。

如果现在只有几百对黄金标签，建议**先把 SCoRE 作为 shadow gate 上线**，积累真实 hard-case review，再逐步扩大自动匹配覆盖率；不要因为论文有 risk control 就过早承诺 99.9%/99.99% 的线上 precision。

---

## 26. 参考资料

1. Tian Bai, Ying Jin. **Conformal Selective Prediction with General Risk Control**  
   https://arxiv.org/abs/2603.24704
2. 官方代码：**Tian-Bai/SCoRE**  
   https://github.com/Tian-Bai/SCoRE
3. PyPI 包：`score-select`（官方 README 给出的安装方式）
4. 当前需求 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）
