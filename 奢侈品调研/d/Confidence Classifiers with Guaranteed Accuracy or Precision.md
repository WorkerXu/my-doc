# Confidence Classifiers with Guaranteed Accuracy or Precision：面向跨源二奢/腕表实体匹配的高精度拒识层技术分析与落地方案

> 分析对象：Johansson et al., **Confidence Classifiers with Guaranteed Accuracy or Precision**（PMLR 204, 2023）  
> 论文主页：https://proceedings.mlr.press/v204/johansson23a.html  
> PDF：https://proceedings.mlr.press/v204/johansson23a/johansson23a.pdf  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31  
> 需求核心：100 万–1000 万级持续增量商品；“同一个商品”定义为**同一 reference number / 型号**；字段稀疏；可用图片；可人工标注几百对；**绝对不能误匹配，precision 优先到极致，允许漏匹配**。

---

## 1. 结论先行

这篇论文最值得当前需求复用的，不是 decision tree / random forest，也不是某个具体分类器，而是它给出了一套非常适合 **precision-first + 可拒识（abstain）** 系统的决策范式：

```text
基础模型负责“排序/打分”
        ↓
Conformal confidence 负责“这个批次里哪些预测值得自动执行”
        ↓
低置信或无法满足目标 precision 的样本全部 reject / abstain
```

它比直接使用 `match_probability > 0.99` 更有价值，因为普通模型概率即使经过 Platt scaling，也可能在高拒绝比例、高置信尾部发生明显的过度自信；论文实验中，Mondrian conformal 方案在不同 reject level 上的 precision 估计偏差显著小于未校准概率和 Platt scaling。

但对本 Spec，必须先明确一个边界：

> **Conformal prediction 的统计保证不能被解释成“单条记录绝不会误匹配”。**

论文的保证依赖 calibration/test 的 exchangeability，并针对一组预测的误差率 / precision 进行控制或估计；论文还明确指出它的方案需要拿到一个 test instance 集合，**不能直接照搬到逐条 streaming 场景**。

因此，当前业务不能让 conformal classifier 成为“最终身份定义”。推荐把它放在一个更保守的架构里：

1. **品牌 + reference 的确定性硬规则定义身份；**
2. reference 从标题、描述或图片 OCR 中抽出来时，要先判断“这个编号是不是当前商品自己的品牌 reference”，而不是平台 SKU、兼容型号、配件适配号；
3. 对“reference 已相等，但证据来源不够硬”的候选对，用基础模型输出风险分数；
4. 用 **Mondrian conformal + reject option** 只挑出最可信的一小部分自动放行；
5. 任何 reference 冲突、角色冲突、校准不足、分布漂移，都直接 `ABSTAIN`；
6. 图片只做辅助证据或冲突否决，不能把不同 reference 的商品因为“长得像”而合并。

一句话概括推荐落地方式：

> **把论文的方法做成“自动合并保险丝”，而不是“自动匹配发动机”。**

这样最符合 Spec 的“错一个都很贵，漏一些没关系”。

---

## 2. 论文到底解决了什么问题

### 2.1 普通二分类默认“每条都必须给答案”

传统 entity matching 常做成二分类：

```text
pair(record_a, record_b)
        ↓
matcher
        ↓
P(match)
        ↓
P > threshold ? MATCH : NON_MATCH
```

问题是，无论模型多不确定，它通常都被迫输出一个类别。

而我们的业务成本是不对称的：

```text
False Positive（把不同 reference 合并） >>> False Negative（漏掉同 reference）
```

所以最合理的输出不是二态：

```text
MATCH / NON_MATCH
```

而是三态：

```text
AUTO_MATCH
REJECT
ABSTAIN / HUMAN_REVIEW
```

甚至生产上可以进一步简化为：

```text
AUTO_MATCH
ABSTAIN
```

因为非匹配不需要“证明不同”，只要不要自动合并即可。

### 2.2 为什么直接看模型概率不可靠

论文比较了三种方案：

```text
Uncal：直接使用模型原始概率
Platt：使用 Platt scaling 做概率校准
Conf：使用 conformal classifier 的 confidence
```

实验使用 decision tree 和 random forest，但核心结论与具体模型无关：

- 未校准模型会明显过度自信；
- Platt scaling 能改善整体 calibration，但在 selective / reject 场景中，尤其针对 precision，高置信尾部仍可能偏；
- conformal confidence 是一种**集合层面的置信信息**，在不同 reject 比例下对实际 precision 的估计更稳。

论文的关键观察是：普通 probability 是“这一条样本属于正类的概率估计”；conformal confidence 的意义更接近“当我只保留 confidence 至少这么高的一组样本时，这一组的错误率应该是什么水平”。

这恰好与我们真正需要的问题一致：

> 不是“这两个商品有 99.7% 概率相同吗？”，而是“如果今天自动执行这一批匹配，我们能否把这一批的误匹配风险压到极低？”

---

## 3. Conformal Classification 的技术机制

### 3.1 数据拆成 training + calibration

Inductive Conformal Prediction（ICP）的基本做法：

```text
已标注数据 Z
  ├── proper training set Z_t
  └── calibration set Z_c
```

基础模型只在 `Z_t` 上训练。

对于 calibration 样本，计算 nonconformity score：

```text
alpha_i = A(x_i, y_i)
```

论文采用的示例是基于基础分类器概率的 hinge-style nonconformity：

```text
alpha(x, y) = 1 - P_h(y | x)
```

也就是说：

- 模型越相信真实标签，nonconformity 越小；
- 模型越不相信真实标签，nonconformity 越大。

### 3.2 对测试样本计算每个候选标签的 p-value

对于测试样本 `x`，分别假设它是每个标签，计算对应 nonconformity，再与 calibration score 排序比较，得到 conformal p-value。

二分类时会得到：

```text
p(match)
p(non_match)
```

这里的 p-value 不是传统统计检验中某个模型参数的 p-value，而是“这个测试样本在假设标签下，与 calibration 数据相比有多不异常”的 conformal p-value。

### 3.3 confidence 与 credibility

论文不用传统 conformal set 直接做最终决策，而是从 p-values 计算：

```text
predicted_label = p-value 最大的标签
confidence      = 1 - 第二大的 p-value
credibility     = 最大的 p-value
```

二分类下，如果：

```text
p(match)     = 0.995
p(non_match) = 0.003
```

那么：

```text
predicted = MATCH
confidence ≈ 0.997
credibility = 0.995
```

论文重点使用 `confidence` 做 reject ranking。

### 3.4 reject option

把测试样本按 confidence 从高到低排序：

```text
最高置信样本 ------------------------ 最低置信样本
      自动预测                         reject
```

如果保留 `m` 条、原批次共有 `n` 条，并选择对应 confidence threshold `lambda`，论文给出的期望错误率估计形式为：

```text
estimated_error_rate = n * (1 - lambda) / m
```

于是：

```text
estimated_accuracy = 1 - n * (1 - lambda) / m
```

直觉上，当我们想提高自动执行精度时，就不断提高 `lambda`、扩大 reject 比例，只保留最安全的头部样本。

这与“允许漏匹配”的业务约束完全契合。

---

## 4. 为什么要用 Mondrian Conformal 才能对 precision 有意义

普通 conformal validity 关注整体 error rate / accuracy。

但当前业务最关心的是：

```text
Precision = TP / (TP + FP)
```

也就是：

> 在所有被系统自动判成 MATCH 的 pair 里，有多少真的应该 MATCH？

论文通过 **Mondrian conformal classifier** 把 calibration 分成类别，并让 validity 在类别内部成立。

论文实验里使用的 taxonomy 是：

```text
按基础模型预测类别分组
```

于是正类组可以写成：

```text
κ_positive = {基础模型预测为 MATCH 的样本}
```

只在这个组里做 calibration / confidence 分析，就能把问题转成：

```text
“被预测成 MATCH 的那些样本里，实际误差多少？”
```

这对应的就是 precision 风险。

对于我们的系统，应把论文思想保留下来，但不能一开始就把 taxonomy 切得很细，因为只有几百条黄金标签。

### 第一版建议的 Mondrian taxonomy

只按：

```text
predicted_label ∈ {MATCH, NON_MATCH}
```

对于自动合并，只使用 `MATCH` 类别。

### 标签积累后再扩展

当 calibration 样本够多，可以逐步增加：

```text
source_pair:
  leixiaoan ↔ watch之家
  leixiaoan ↔ 奢当家
  watch之家 ↔ 奢当家

ref_evidence_type:
  structured_field ↔ structured_field
  structured_field ↔ title_extract
  title_extract ↔ title_extract
  OCR ↔ structured_field
  OCR ↔ title_extract
```

甚至：

```text
brand_group
product_type
```

但前提是每个 bucket 有足够 calibration 样本，否则“分得越细”反而让 p-value 离散、方差大、无法支持极端 precision。

---

## 5. 论文实验最值得关注的结果

论文在 10 个二分类数据集上比较：

```text
Uncalibrated probability
Platt calibrated probability
Conformal confidence
```

并在多个 reject 比例（10%–90%）上评估。

对于 precision estimation，论文 Table 13 给出的平均绝对误差显示：

```text
Decision Tree:
  Uncal mean abs error ≈ 0.200
  Platt mean abs error ≈ 0.055
  Conf  mean abs error ≈ 0.009

Random Forest:
  Uncal mean abs error ≈ 0.086
  Platt mean abs error ≈ 0.036
  Conf  mean abs error ≈ 0.014
```

这不是说 conformal matcher 本身一定比所有模型“匹配更准”，而是：

> **当系统决定只自动处理最可信的一部分样本时，conformal confidence 对“这一部分到底有多准”的估计明显更可靠。**

对当前需求，这比平均 F1 提升 1–2 个点重要得多。

我们真正关心的是 selective risk curve：

```text
coverage ↑  -> precision 往往 ↓
precision ↑ -> reject / abstain ↑
```

而不是最大化一个混合指标 F1。

---

## 6. 这篇论文不能直接照搬的 5 个问题

### 6.1 它不能保证“0 个 false positive”

Conformal 的 guarantee 是统计意义上的，并依赖 exchangeability。

业务里的：

```text
绝对不能误匹配
```

不能被翻译成：

```text
用了 conformal prediction，所以每条自动 match 都一定正确
```

这是错误理解。

因此生产架构仍然必须保留**确定性硬约束**：

```text
brand 不一致 -> 永不自动合并
canonical reference 不一致 -> 永不自动合并
reference role 有冲突 -> 永不自动合并
高风险编号变换 -> 永不自动合并
```

Conformal 只决定：

> 在已经通过硬规则的“候选相等”里，哪些证据足够强，可以自动执行；哪些需要 abstain。

### 6.2 论文明确不是 streaming 方案

论文指出该方法需要一个 test instance 集合，不适合逐条 streaming 直接使用。

但 Spec 要持续增量。

解决办法是**micro-batch 化**：

```text
实时抓取
  ↓
进入事件流 / staging
  ↓
每 5 分钟 / 30 分钟 / 1 小时组成一个 decision batch
  ↓
对 batch 做 conformal confidence + selective acceptance
  ↓
AUTO_MATCH / ABSTAIN
```

对二奢数据，没有必要为了几十毫秒延迟牺牲安全性。

### 6.3 exchangeability 在持续分布漂移下可能失效

新品牌、新来源、新抓取模板、新标题风格、新 OCR 模型，都会让分布变。

必须增加 drift guard：

```text
PSI / KS / feature drift
ref pattern drift
source extraction error rate
brand-level match-rate shift
abstain-rate shift
```

一旦漂移超阈值：

```text
该 bucket 自动匹配关闭
→ 全部 ABSTAIN
→ 采样人工标注
→ 更新 calibration
```

### 6.4 论文数据集远小于 100 万–1000 万

论文的 conformal 层本身计算并不重，真正的千万级问题发生在**候选生成**。

当前需求不能做全量 pair：

```text
10^6 × 10^6
```

更不能做到三源全笛卡尔积。

由于身份定义已经是 reference，正确的 scalable 方案是：

```text
record
  -> brand normalization
  -> reference candidate extraction
  -> canonical reference
  -> inverted index (brand_id, ref_canonical)
  -> 只在同 key 内形成候选
```

Conformal matcher 只处理这个极小候选集合。

### 6.5 几百条黄金标签无法证明 99.99% precision

Spec 说可以人工标注几百对，这足够：

- 做第一版规则验证；
- 训练轻量基础模型；
- 做初始 calibration；
- 找出 hard negative 模式。

但如果业务真的想从统计上证明：

```text
precision >= 99.99%
```

几百条验证样本远远不够。

一个直观估算是：如果验证集中 **0 个错误**，常见“rule of three”近似认为 95% 上界的错误率约为：

```text
3 / n
```

因此：

```text
n = 300     -> 错误率上界约 1%
n = 3,000   -> 错误率上界约 0.1%
n = 30,000  -> 错误率上界约 0.01%
```

所以第一版不要把几百标注包装成“已经证明万分之一误匹配率”。

更现实的策略是：

> 用硬 reference 规则保证业务语义，用 conformal reject 最大限度缩小自动执行面，再通过线上持续抽检积累足够 validation 样本。

---

## 7. 当前需求的正确建模方式

### 7.1 不要直接问“这两个商品是否看起来相同”

这种模型非常危险：

```text
标题很像
图片很像
系列一样
机芯一样
尺寸一样
```

但：

```text
126610LN != 126610LV
```

根据 Spec 就必须是不同商品。

### 7.2 正确问题应是“这两个已抽取 reference 是否可信地代表当前商品”

建议把 pair-level 模型的正类定义为：

```text
safe_same_reference =
    两条记录中的 canonical reference 相同
    AND 两侧 reference 都确实是当前售卖商品的品牌 reference
    AND 没有任何强冲突证据
```

模型不是去“猜 reference 是否等价”，而是去识别：

```text
虽然字符串相等，但其中一侧是不是：
- 平台 SKU
- 店铺货号
- 兼容型号
- 配件适配的腕表型号
- 描述中提到的对比型号
- 盒证/保卡上的另一商品型号
- OCR 误读
```

这是一个更适合 precision-first 的问题。

---

## 8. 推荐生产架构

```text
                           ┌──────────────────────────┐
                           │ 雷小安 / 腕表之家 / 奢当家 │
                           └────────────┬─────────────┘
                                        │
                                        v
┌─────────────────────────────────────────────────────────────────┐
│ 1. Ingestion / Raw Store                                         │
│ raw title / attrs / desc / images / source id / crawl version    │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               v
┌─────────────────────────────────────────────────────────────────┐
│ 2. Identifier Extraction                                         │
│ candidate strings + provenance + role + confidence               │
│ structured / title-regex / NER-LLM / OCR                         │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               v
┌─────────────────────────────────────────────────────────────────┐
│ 3. Canonicalization + Hard Guard                                  │
│ brand canonicalization                                           │
│ conservative ref normalization                                   │
│ role validation                                                   │
│ conflict detection                                                │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                ref invalid/conflict│              valid
                       ┌─────────────┘                │
                       v                              v
                  [ABSTAIN]              ┌──────────────────────────┐
                                         │ 4. Exact Ref Index       │
                                         │ (brand_id, canonical_ref)│
                                         └────────────┬─────────────┘
                                                      │
                                                      v
                                         ┌──────────────────────────┐
                                         │ 5. Pair / Group Features │
                                         └────────────┬─────────────┘
                                                      │
                                                      v
                                         ┌──────────────────────────┐
                                         │ 6. Base Risk Matcher     │
                                         │ score only               │
                                         └────────────┬─────────────┘
                                                      │
                                                      v
                                         ┌──────────────────────────┐
                                         │ 7. Mondrian Conformal    │
                                         │ confidence + reject      │
                                         └────────────┬─────────────┘
                                                      │
                                         ┌────────────┴────────────┐
                                         v                         v
                                  [AUTO_MATCH]                [ABSTAIN]
                                         │                         │
                                         v                         v
                                  Entity Cluster             Human Review
                                         │                         │
                                         └────────────┬────────────┘
                                                      v
                                         Gold Labels / Recalibration
```

这个架构最重要的特点是：

> **任何 ML / LLM / 图片模型都没有权力覆盖 reference 硬冲突。**

---

## 9. Reference 抽取层必须输出“值 + 角色 + 来源”

不要只存：

```json
{"reference": "126610LN"}
```

至少存：

```json
{
  "raw": "126610LN",
  "canonical": "126610LN",
  "brand_id": "rolex",
  "role": "manufacturer_reference",
  "provenance": "structured_field",
  "extractor": "source_parser_v3",
  "confidence": 0.999,
  "span": "...126610LN...",
  "image_id": null,
  "normalization_version": "refnorm_v2"
}
```

图片 OCR 得到的则可能是：

```json
{
  "raw": "12661OLN",
  "canonical": null,
  "brand_id": "rolex",
  "role": "reference_candidate",
  "provenance": "image_ocr",
  "confidence": 0.71,
  "image_id": "img_123",
  "normalization_version": "refnorm_v2"
}
```

注意这里**不要自动把 `O` 改成 `0`**。

可以产生候选：

```text
12661OLN
126610LN
```

但第二个只能作为待验证 candidate，不能因为编辑距离 1 就自动认定。

---

## 10. Canonical Reference 规范化必须“保守而可逆”

### 10.1 可以安全做的规范化

通常可以考虑：

```text
Unicode NFKC
trim
ASCII upper-case
已知品牌规则下的分隔符标准化
连续空格折叠
```

例如在经过品牌规则确认后：

```text
" 126610 ln " -> "126610LN"
```

### 10.2 不应全局自动做的变换

```text
O ↔ 0
I ↔ 1
S ↔ 5
B ↔ 8
删除所有 / - .
任意截断后缀
去掉字母后缀
只保留数字
```

腕表 reference 的一个字符就可能区分不同材质、颜色、尺寸或版本。

因此规范化建议实现为：

```python
normalize(raw, brand_id) -> {
    canonical,
    safe_transforms[],
    unsafe_candidates[]
}
```

而不是只返回一个字符串。

---

## 11. 候选生成：百万级不需要向量全库匹配

既然业务身份定义是 reference，最高优先级的索引就是：

```text
key = (brand_id, canonical_reference)
```

数据库可以维护：

```sql
CREATE TABLE reference_index (
    brand_id            VARCHAR NOT NULL,
    canonical_reference VARCHAR NOT NULL,
    source              VARCHAR NOT NULL,
    source_record_id    VARCHAR NOT NULL,
    evidence_level      SMALLINT NOT NULL,
    extractor_version   VARCHAR NOT NULL,
    PRIMARY KEY (source, source_record_id, canonical_reference)
);

CREATE INDEX idx_reference_lookup
ON reference_index (brand_id, canonical_reference);
```

增量一条商品时：

```text
1. 抽 reference
2. 规范化
3. 查 (brand, ref)
4. 只与命中的其他来源记录比较
```

复杂度从全量 pair：

```text
O(N × M)
```

变成更接近：

```text
O(N log M) / hash lookup
```

实际 pair 数通常会缩小几个数量级。

### 异常大 group 必须报警

如果一个 `(brand, ref)` 突然挂了 10 万条记录，很可能 reference 抽取错了，比如把：

```text
DATEJUST
AUTOMATIC
116
```

这种非唯一文本误当成 reference。

因此加入：

```text
max_group_size_by_brand
ref_uniqueness_stats
source_cardinality_guard
```

超过阈值不自动合并。

---

## 12. 基础风险模型应该吃什么特征

Conformal 不替代基础模型，它需要一个能够排序风险的 underlying model。

第一版建议用可解释、稳定的小模型：

```text
LightGBM / XGBoost / Logistic Regression
```

不用急着上大 LLM。

### 12.1 强特征

```text
brand_exact
ref_canonical_exact
ref_raw_exact
both_structured_ref
left_ref_role
right_ref_role
left_ref_provenance
right_ref_provenance
ref_pattern_valid_left/right
known_brand_ref_regex_match
source_pair
```

### 12.2 冲突特征

```text
brand_conflict
ref_multiple_candidates_left/right
another_strong_ref_conflict
accessory_keyword
compatible_with_keyword
strap_box_card_only_keyword
platform_sku_like_pattern
ocr_only_ref
```

### 12.3 辅助文本特征

```text
title_series_equal
case_size_equal
material_equal
movement_equal
year_consistency
text_embedding_similarity
```

### 12.4 图片特征

```text
image_embedding_similarity
ocr_ref_exact
ocr_ref_conflict
logo_brand_consistency
dial_text_consistency
```

图片相似度只能加分或产生冲突，不能让：

```text
ref A != ref B
```

变成 AUTO_MATCH。

---

## 13. 建议的 hard guard 规则

在模型前直接执行：

```python
def hard_guard(a, b):
    if a.brand_id != b.brand_id:
        return "REJECT"

    if not a.ref_canonical or not b.ref_canonical:
        return "ABSTAIN"

    if a.ref_canonical != b.ref_canonical:
        return "REJECT"

    if a.ref_role not in ALLOWED_REF_ROLES:
        return "ABSTAIN"

    if b.ref_role not in ALLOWED_REF_ROLES:
        return "ABSTAIN"

    if a.has_strong_ref_conflict or b.has_strong_ref_conflict:
        return "ABSTAIN"

    if a.is_accessory_like or b.is_accessory_like:
        return "ABSTAIN"

    return "PASS_TO_RISK_MODEL"
```

`ALLOWED_REF_ROLES` 第一版可以极保守：

```text
manufacturer_reference
source_official_model_field
```

标题抽取 / OCR 一开始都不进入直接 AUTO_MATCH 范围。

待有足够标注再逐步放开。

---

## 14. 如何把论文的 Mondrian Conformal 真正接到系统里

### 14.1 训练阶段

假设人工有 600 对黄金标签。

不要随机 600 对都来自容易样本，应主动采 hard cases：

```text
150：structured ref == structured ref
150：structured ref == title extracted ref
100：title extracted == title extracted
100：OCR involved
100：hard negatives
     - 同系列差 1 个字符
     - 配件标题含腕表型号
     - 平台 SKU 与 reference 混淆
     - OCR O/0 I/1
```

拆分示例：

```text
60% proper training
40% calibration
```

小数据时甚至可以：

```text
训练基础模型使用规则 + 预训练特征
尽量把更多人工标签留给 calibration / validation
```

因为当前任务更需要可靠拒识，而不是追求复杂模型拟合。

### 14.2 建立基础 matcher

```python
base_model.fit(X_train, y_train)
```

输出：

```text
score_match = P_base(match | x)
```

### 14.3 构造 nonconformity

最简单沿用论文：

```python
alpha = 1 - p_true_label
```

例如 calibration 真实标签是 MATCH：

```python
alpha_i = 1 - model.predict_proba(x_i)[MATCH]
```

真实标签 NON_MATCH 则：

```python
alpha_i = 1 - model.predict_proba(x_i)[NON_MATCH]
```

### 14.4 Mondrian bucket

论文按预测标签分组。

生产第一版：

```python
bucket = predicted_label
```

只关注：

```text
bucket == MATCH
```

### 14.5 推理一个 micro-batch

```python
for pair in batch:
    hard = hard_guard(pair.left, pair.right)
    if hard != "PASS_TO_RISK_MODEL":
        emit(hard)
        continue

    proba = base_model.predict_proba(features(pair))
    p_values = conformal_p_values(proba, calibration_scores)

    pred = argmax(p_values)
    confidence = 1 - second_largest(p_values)
    credibility = largest(p_values)

    if pred != MATCH:
        emit("ABSTAIN")
        continue

    candidates.append({
        "pair": pair,
        "confidence": confidence,
        "credibility": credibility,
    })
```

然后对当前 `MATCH` candidates 按 confidence 排序。

### 14.6 只接受满足目标 selective precision 的前缀

伪代码：

```python
candidates.sort(key=lambda x: x["confidence"], reverse=True)

accepted = []
N = len(candidates)

for rank, item in enumerate(candidates, start=1):
    m = rank
    lam = item["confidence"]

    estimated_error = N * (1 - lam) / m
    estimated_precision = 1 - estimated_error

    if estimated_precision >= TARGET_PRECISION:
        accepted.append(item)
    else:
        break
```

注意：这是论文集合置信思想的工程化表达。实际实现应严格按照选定 conformal 公式、随机化 tie handling 和 calibration bucket 计算，不建议只靠上面十几行伪代码直接上线。

---

## 15. 我建议的业务阈值不是一个数字，而是多重门

不要配置：

```yaml
match_threshold: 0.999
```

应该配置：

```yaml
policy:
  require_same_brand: true
  require_same_canonical_reference: true
  allow_reference_conflict: false
  allow_accessory_like_record: false

  reference_evidence:
    auto_allow:
      - structured_field
      - verified_source_parser
    conditional_allow:
      - title_extractor
      - image_ocr

  conformal:
    enabled: true
    min_calibration_size: 200
    target_precision: 0.999
    min_credibility: 0.05

  safety:
    unknown_bucket: abstain
    drifted_bucket: abstain
    oversized_reference_group: abstain
    normalization_version_mismatch: abstain
```

这里 `target_precision: 0.999` 只是系统策略目标，不表示已经获得法律意义或绝对数学意义上的“千分之一以下错误保证”。真正生产承诺还必须结合足量审计样本。

---

## 16. 对“持续增量”的落地改造：Micro-batch Conformal Gate

论文原方案需要一组 test instances。

因此把流式数据做成：

```text
Kafka / MQ / DB CDC
        ↓
5–60 分钟窗口
        ↓
Match Decision Batch
        ↓
Conformal Gate
```

### 推荐批次维度

不要把所有品牌混成一个 batch。

可以先按：

```text
source_pair × ref_evidence_level
```

分组。

例如：

```text
雷小安-腕表之家 / structured-structured
雷小安-腕表之家 / structured-title
腕表之家-奢当家 / structured-structured
```

但小流量 bucket 应回退到父 bucket，而不是硬算一个只有 8 条 calibration 的 Mondrian p-value。

### 分层回退

```text
source_pair + evidence_type
      ↓ insufficient calibration
source_pair
      ↓ insufficient calibration
global predicted-positive
      ↓ insufficient calibration
ABSTAIN
```

这是生产上非常重要的细节。

---

## 17. Calibration 数据版本化

至少维护：

```text
calibration_set_id
model_version
feature_version
normalization_version
extractor_version
source_schema_version
created_at
valid_from
valid_to
```

每一次 AUTO_MATCH 都要记录：

```json
{
  "decision": "AUTO_MATCH",
  "brand_id": "rolex",
  "canonical_reference": "126610LN",
  "model_version": "risk_lgbm_v4",
  "calibration_set_id": "cal_2026_08_18_01",
  "confidence": 0.99972,
  "credibility": 0.93,
  "bucket": "lx_watch_struct_struct",
  "normalization_version": "refnorm_v2",
  "evidence": ["structured_ref_left", "structured_ref_right"]
}
```

以后发现误匹配，才能回答：

```text
哪一版抽取器？
哪一版 normalization？
哪个 conformal bucket？
当时 calibration 是什么？
为什么系统自动放行？
```

---

## 18. 数据表建议

### 18.1 商品原始表

```sql
CREATE TABLE product_record (
    id                  BIGINT PRIMARY KEY,
    source              VARCHAR NOT NULL,
    source_record_id    VARCHAR NOT NULL,
    title               TEXT,
    attrs_json          JSONB,
    description         TEXT,
    raw_payload         JSONB,
    crawl_time          TIMESTAMP NOT NULL,
    schema_version      VARCHAR NOT NULL,
    UNIQUE(source, source_record_id)
);
```

### 18.2 Identifier evidence

```sql
CREATE TABLE identifier_evidence (
    id                    BIGSERIAL PRIMARY KEY,
    product_record_id     BIGINT NOT NULL,
    brand_id              VARCHAR,
    raw_value             VARCHAR NOT NULL,
    canonical_value       VARCHAR,
    identifier_role       VARCHAR NOT NULL,
    provenance            VARCHAR NOT NULL,
    extractor_version     VARCHAR NOT NULL,
    normalization_version VARCHAR NOT NULL,
    confidence            DOUBLE PRECISION,
    metadata              JSONB,
    created_at            TIMESTAMP NOT NULL
);

CREATE INDEX idx_identifier_ref
ON identifier_evidence (brand_id, canonical_value);
```

### 18.3 Match decision

```sql
CREATE TABLE match_decision (
    id                    BIGSERIAL PRIMARY KEY,
    left_record_id        BIGINT NOT NULL,
    right_record_id       BIGINT NOT NULL,
    decision              VARCHAR NOT NULL,
    reason_code           VARCHAR NOT NULL,
    model_version         VARCHAR,
    calibration_set_id    VARCHAR,
    conformal_bucket      VARCHAR,
    base_score            DOUBLE PRECISION,
    confidence            DOUBLE PRECISION,
    credibility           DOUBLE PRECISION,
    evidence_snapshot     JSONB NOT NULL,
    created_at            TIMESTAMP NOT NULL
);
```

### 18.4 人工复核

```sql
CREATE TABLE match_review (
    id                BIGSERIAL PRIMARY KEY,
    match_decision_id BIGINT NOT NULL,
    gold_label        SMALLINT NOT NULL,
    error_type        VARCHAR,
    reviewer          VARCHAR,
    reviewed_at       TIMESTAMP NOT NULL,
    notes             TEXT
);
```

---

## 19. Reference Evidence Level：建议直接做成业务等级

第一版定义：

```text
L0：无 reference
L1：OCR / 自由文本弱抽取
L2：标题高置信抽取
L3：来源明确的型号字段
L4：来源明确型号字段 + 第二独立证据一致
```

自动合并策略：

```text
L4 + L4 -> 可进入 AUTO_MATCH gate
L4 + L3 -> 可进入 AUTO_MATCH gate
L3 + L3 -> 可进入 AUTO_MATCH gate
L3 + L2 -> conformal 严格 gate，默认高 reject
L2 + L2 -> 第一版 ABSTAIN
任何 L1 -> ABSTAIN
任何 L0 -> ABSTAIN
```

随着人工标注积累再放开，不要一开始为了 recall 把所有路径都自动化。

---

## 20. “同 reference 聚类”也必须防止错误传播

三个来源最后不是简单 pair 表，而会形成 entity cluster：

```text
雷小安 A
  ↕
腕表之家 B
  ↕
奢当家 C
```

即使 pair AB 被放行，也不能无条件靠 transitive closure 把所有东西并起来。

建议 cluster key 仍然以：

```text
(brand_id, canonical_reference)
```

为核心。

并满足：

```text
cluster 内所有强 reference evidence 不得冲突
```

如果出现：

```text
A strong ref = 126610LN
B weak ref   = 126610LN
C strong ref = 126610LV
```

必须：

```text
整个可疑 cluster 冻结
→ 人工复核
```

不能因为 `A-B` 和 `B-C` 两条模型边都高分，就把 LN 与 LV 合并。

---

## 21. 图片应该放在哪一层

Spec 明确有图片。

图片很有价值，但最安全的角色是：

### 21.1 OCR reference evidence

对：

```text
表背
保卡
吊牌
说明标签
```

做 OCR，产生 identifier candidate。

### 21.2 冲突检测

文本抽出：

```text
126610LN
```

图片 OCR 强证据却是：

```text
126610LV
```

则：

```text
ABSTAIN
```

### 21.3 人工复核排序

图片 embedding 很像，可以让复核员优先看。

### 21.4 不能做的事情

```text
image_similarity > 0.99
=> 自动认为 reference 相同
```

这是高风险错误。

同系列不同 reference 的视觉差异可能极小。

---

## 22. 基础模型选择：为什么第一版推荐 LightGBM 而不是 LLM

当前 pair 特征多数是结构化离散 / 布尔 / 数值：

```text
ref 是否 exact
证据来源
role
是否冲突
source pair
pattern validity
OCR consistency
文本辅助相似度
```

LightGBM 的优点：

- 小样本可用；
- 训练快；
- 推理便宜；
- 特征贡献可解释；
- 输出 score 可供 conformal 排序；
- 版本可控；
- 没有生成式 hallucination。

LLM 可以放在：

```text
reference / identifier role extraction
困难文本解析
人工复核辅助
```

但不建议让 LLM 直接决定 AUTO_MATCH。

---

## 23. 最小可运行代码结构

建议拆成：

```text
luxury_matcher/
├── ingestion/
│   ├── models.py
│   └── source_adapters/
├── identifier/
│   ├── brand_normalizer.py
│   ├── ref_extractor.py
│   ├── role_classifier.py
│   ├── ref_normalizer.py
│   └── evidence.py
├── blocking/
│   └── reference_index.py
├── features/
│   └── pair_features.py
├── matcher/
│   ├── base_model.py
│   └── model_registry.py
├── conformal/
│   ├── calibration.py
│   ├── mondrian.py
│   ├── confidence.py
│   └── policy.py
├── decision/
│   ├── hard_guard.py
│   ├── batch_decider.py
│   └── reason_codes.py
├── review/
│   ├── sampler.py
│   └── feedback.py
└── monitoring/
    ├── precision_audit.py
    ├── drift.py
    └── dashboards.py
```

---

## 24. Conformal Calibrator 接口建议

```python
from dataclasses import dataclass
from typing import Dict, List


@dataclass
class ConformalResult:
    predicted_label: int
    confidence: float
    credibility: float
    p_values: Dict[int, float]
    bucket: str
    calibration_size: int


class MondrianConformalCalibrator:
    def fit(self, model, X_cal, y_cal, bucket_fn):
        ...

    def predict_one(self, model, x, bucket) -> ConformalResult:
        ...

    def predict_batch(self, model, X, buckets) -> List[ConformalResult]:
        ...
```

关键要求：

```text
calibrator 与 base model 解耦
```

以后基础模型可以从：

```text
LightGBM
```

换成：

```text
CrossEncoder / multimodal model
```

conformal gate 不需要重写。

---

## 25. 决策引擎必须返回 reason code

```python
class Reason:
    BRAND_CONFLICT = "BRAND_CONFLICT"
    REFERENCE_CONFLICT = "REFERENCE_CONFLICT"
    NO_REFERENCE = "NO_REFERENCE"
    WEAK_REFERENCE_ROLE = "WEAK_REFERENCE_ROLE"
    ACCESSORY_RISK = "ACCESSORY_RISK"
    CALIBRATION_TOO_SMALL = "CALIBRATION_TOO_SMALL"
    DRIFT_DETECTED = "DRIFT_DETECTED"
    CONFORMAL_REJECT = "CONFORMAL_REJECT"
    AUTO_MATCH_SAFE = "AUTO_MATCH_SAFE"
```

输出例子：

```json
{
  "decision": "ABSTAIN",
  "reason": "WEAK_REFERENCE_ROLE",
  "left_ref": "126610LN",
  "right_ref": "126610LN",
  "left_evidence": "structured_field",
  "right_evidence": "compatible_text_extract"
}
```

这会极大提升人工复核效率，也方便做错误分析。

---

## 26. 人工标注几百对，应该优先标什么

不要均匀随机抽。

随机样本很可能大量是：

```text
明显不同
```

对提高 precision 没什么帮助。

优先标：

### 26.1 同系列近邻 reference

```text
126610LN vs 126610LV
116500LN-0001 vs 116500LN-0002
```

### 26.2 相同字符串但角色不同

```text
平台 SKU 恰好等于另一条商品 reference
配件标题包含兼容腕表 reference
描述中列举多个型号
```

### 26.3 抽取边界

```text
reference 前后有中文
连字符
斜杠
空格
全半角
```

### 26.4 OCR 混淆

```text
O / 0
I / 1
S / 5
B / 8
```

### 26.5 来源差异

三个 source pair 都要覆盖。

这几类样本对 calibrating extreme precision 比随机 easy negative 更有价值。

---

## 27. 线上反馈回路

```text
ABSTAIN 样本
   ↓
按风险 / 不确定度 / 新模式抽样
   ↓
人工复核
   ↓
错误类型归因
   ↓
┌────────────────────────────┐
│ 更新 parser / normalization │
│ 更新基础模型                 │
│ 更新 calibration             │
│ 新增 hard guard              │
└────────────────────────────┘
```

这里特别要区分：

```text
模型错误
```

和：

```text
数据语义 / parser 错误
```

如果问题其实是“来源字段从 `model_no` 改名了”，重训 matcher 没意义，应该修 source adapter。

---

## 28. Precision-first 监控指标

不要只看：

```text
Accuracy
F1
AUC
```

生产 dashboard 应优先看：

```text
accepted_precision
accepted_false_positive_count
selective_coverage
auto_match_rate
abstain_rate
human_review_rate
precision_by_source_pair
precision_by_evidence_type
precision_by_brand
calibration_bucket_size
reference_conflict_rate
oversized_ref_group_count
drift_alarm_count
```

最核心的是 selective risk-coverage：

```text
X: auto-match coverage
Y: false-positive risk / precision
```

业务应自己选择愿意牺牲多少 coverage 来换安全性。

---

## 29. 自动匹配上线建议：三阶段，而不是一步到位

### Phase 0：只上 deterministic baseline

自动合并仅允许：

```text
brand canonical exact
AND canonical reference exact
AND 两边均来自可信 structured field
AND 无冲突
```

其他全部 abstain。

目标：

- 验证 reference 语义；
- 建 normalization 规则；
- 积累错误类型；
- 验证三源字段质量。

### Phase 1：Conformal 只做 shadow / monitor

不改变自动决策，只记录：

```text
base score
confidence
credibility
estimated selective precision
```

与人工复核对比。

目标：确认 calibration 在真实数据上有效。

### Phase 2：开放 structured ↔ title 的高置信自动匹配

只有：

```text
hard guard PASS
AND conformal gate PASS
AND audit precision 达标
```

才自动执行。

### Phase 3：逐步探索 OCR / multimodal

OCR 先只用于：

```text
冲突否决 + 人工辅助
```

有足够标注后再讨论是否进入自动匹配路径。

---

## 30. 针对“绝不能误匹配”的最终决策策略

我建议生产策略是“多保险丝串联”：

```text
保险丝 1：品牌一致
保险丝 2：canonical reference exact
保险丝 3：reference role 合法
保险丝 4：无强冲突证据
保险丝 5：来源证据等级达到要求
保险丝 6：基础风险模型高分
保险丝 7：Mondrian conformal selective gate 通过
保险丝 8：bucket calibration 足够且无 drift
```

任意一个不满足：

```text
ABSTAIN
```

这比一个超级复杂的端到端 matcher 更符合真实需求。

---

## 31. 一个可直接落地的 `decide_match` 伪代码

```python
def decide_match(left, right, ctx):
    # 1. Hard identity semantics
    if left.brand_id != right.brand_id:
        return Decision.reject("BRAND_CONFLICT")

    if not left.reference or not right.reference:
        return Decision.abstain("NO_REFERENCE")

    if left.reference.canonical != right.reference.canonical:
        return Decision.reject("REFERENCE_CONFLICT")

    # 2. Identifier role / provenance safety
    if not left.reference.role_is_safe:
        return Decision.abstain("WEAK_REFERENCE_ROLE")

    if not right.reference.role_is_safe:
        return Decision.abstain("WEAK_REFERENCE_ROLE")

    if left.has_strong_conflict or right.has_strong_conflict:
        return Decision.abstain("STRONG_CONFLICT")

    if left.is_accessory_like or right.is_accessory_like:
        return Decision.abstain("ACCESSORY_RISK")

    # 3. Build interpretable pair features
    x = ctx.feature_builder.build(left, right)

    # 4. Underlying model only gives ranking score
    base_proba = ctx.base_model.predict_proba(x)
    pred = int(base_proba[1] >= 0.5)

    if pred != 1:
        return Decision.abstain("BASE_MODEL_NON_MATCH")

    # 5. Resolve conformal bucket
    bucket = ctx.bucket_resolver.resolve(left, right, x)

    if ctx.drift_monitor.is_drifted(bucket):
        return Decision.abstain("DRIFT_DETECTED")

    if ctx.calibrator.size(bucket) < ctx.min_calibration_size:
        bucket = ctx.bucket_resolver.fallback(bucket)

    if bucket is None:
        return Decision.abstain("CALIBRATION_TOO_SMALL")

    # 6. Conformal score
    conf = ctx.calibrator.predict_one(ctx.base_model, x, bucket)

    if conf.predicted_label != 1:
        return Decision.abstain("CONFORMAL_NON_MATCH")

    # 7. Batch selective gate is applied after ranking all pending positives.
    return PendingPositive(
        left_id=left.id,
        right_id=right.id,
        confidence=conf.confidence,
        credibility=conf.credibility,
        bucket=bucket,
        base_score=base_proba[1],
    )
```

随后：

```python
accepted, rejected = selective_batch_gate(pending_positives)
```

只有 `accepted` 才真正写入 entity cluster。

---

## 32. 与当前 Spec 的逐条映射

### 约束：100 万–1000 万级

方案：

```text
reference inverted index
+ 增量 lookup
+ 小候选集
+ conformal 只处理候选
```

不做全量 pair。

### 约束：持续增量

方案：

```text
stream ingestion
+ micro-batch decision
+ rolling calibration
+ drift guard
```

解决论文本身不支持 streaming 的限制。

### 约束：同一个商品 = 同一 reference

方案：

```text
canonical reference exact 是不可被 ML 覆盖的 hard identity rule
```

### 约束：字段稀疏

方案：

```text
reference evidence 可以来自：
structured field / title / description / OCR
```

但每种 evidence 单独建风险等级。

### 约束：绝对不能误匹配

方案：

```text
hard guard + reject option + conformal gate + drift fail-closed
```

默认拒识，不追求全覆盖。

### 约束：图片可用

方案：

```text
OCR identifier evidence
+ conflict veto
+ review assist
```

图片不越过 reference 冲突。

### 约束：可标几百对

方案：

```text
主动采 hard negatives
+ 简单基础模型
+ 初始 Mondrian calibration
+ 线上持续审计扩大样本
```

不把几百样本误解成足以证明 99.99% precision。

---

## 33. 与“直接调一个很高概率阈值”相比，为什么值得加这一层

### 普通方案

```text
if model.p_match > 0.999:
    auto_merge()
```

问题：

- 模型概率可能未校准；
- 全局校准不代表高置信尾部校准；
- 新来源分布变化后 0.999 不再具有同样含义；
- 无法自然表达 coverage / reject tradeoff。

### Conformal selective 方案

```text
模型 score
  ↓
calibration data
  ↓
confidence ranking
  ↓
只接受满足安全目标的集合
```

它更符合“这批自动执行结果要有怎样的 precision”这个运营问题。

---

## 34. 仍然建议增加一个论文之外的安全层：线上统计审计

因为业务说“绝不能误匹配”，只依赖模型内部 guarantee 不够。

建议每个周期从 AUTO_MATCH 中抽样复核：

```text
高置信随机样本
+ 各品牌
+ 各 source pair
+ 各 evidence type
+ 新 reference pattern
```

维护：

```text
observed_fp_count
reviewed_auto_match_count
upper_confidence_bound_of_fp_rate
```

如果任何关键 bucket 的风险上界超过政策值：

```text
自动关闭该 bucket
```

这是一种 fail-closed 运维方式。

---

## 35. 失败场景演练

### Case A：同 reference，双方结构化字段

```text
A.brand = ROLEX
A.ref   = 126610LN (structured)

B.brand = 劳力士 -> ROLEX
B.ref   = 126610LN (structured)
```

如果无冲突：

```text
hard guard PASS
→ conformal gate
→ 高概率 AUTO_MATCH
```

### Case B：同系列不同 reference

```text
126610LN
126610LV
```

无论标题、图片多像：

```text
REFERENCE_CONFLICT
→ REJECT
```

### Case C：表带兼容 126610LN

```text
A = 劳力士 126610LN 腕表
B = 适配劳力士 126610LN 橡胶表带
```

字符串 reference 相同。

但 B：

```text
product_type = strap/accessory
ref_role = compatible_target_reference
```

因此：

```text
ACCESSORY_RISK / WEAK_REFERENCE_ROLE
→ ABSTAIN
```

这是当前系统最需要防的 false positive 类型之一。

### Case D：OCR 误读

```text
text ref = 126610LN
image OCR = 12661OLN
```

不能自动 `O -> 0`。

```text
OCR_REF_CONFLICT
→ ABSTAIN
```

### Case E：新来源抓取模板变化

原来：

```text
model_no = 5711/1A
```

新版字段变成：

```text
sku = 5711/1A
```

parser 误把平台 SKU 当 reference。

若线上：

```text
reference cardinality / role distribution 突变
```

drift monitor 应自动触发：

```text
DRIFT_DETECTED
→ 该来源自动匹配暂停
```

---

## 36. 推荐评测集组织方式

不要只随机切 train/test。

至少建立：

```text
golden/
├── easy_positive
├── easy_negative
├── same_series_diff_ref
├── accessory_mentions_ref
├── platform_sku_confusion
├── multi_ref_title
├── ocr_confusion
├── structured_vs_title
├── title_vs_title
├── unseen_brand_or_pattern
└── source_schema_change
```

核心通过标准：

```text
accepted_false_positive_count = 0   # 在当前 golden set 上必须为 0
```

但必须在报告里注明：

> 测试集 0 FP 是上线门槛，不等于已经数学证明线上永远 0 FP。

---

## 37. 推荐第一版工程工作量

### 1. Reference 层

```text
brand canonicalization
reference parser
reference role
conservative normalization
evidence schema
```

### 2. Index 层

```text
PostgreSQL / Elasticsearch exact index
```

无需第一天引入向量数据库。

### 3. Matcher 层

```text
hand-crafted features
LightGBM
```

### 4. Conformal 层

```text
ICP / Mondrian calibration
confidence / credibility
batch reject policy
```

### 5. 安全层

```text
hard guard
reason codes
drift monitor
manual review
precision audit
```

第一版最难的其实不是模型，而是：

```text
reference 的业务语义、角色识别和可审计证据链
```

---

## 38. 推荐的 MVP 验收标准

### 功能

- [ ] 三个来源都能解析统一 `brand_id`
- [ ] reference evidence 保留 raw / canonical / role / provenance
- [ ] 可以按 `(brand_id, canonical_reference)` 增量查候选
- [ ] reference 冲突永不自动合并
- [ ] accessory / compatible reference 能被阻断
- [ ] 基础 matcher 与 conformal calibrator 解耦
- [ ] 支持 `AUTO_MATCH / ABSTAIN / REJECT`
- [ ] 每个决策有 reason code 和版本链
- [ ] 支持 micro-batch
- [ ] drift 后 fail closed

### 评测

- [ ] golden hard-negative 上 AUTO_MATCH FP = 0
- [ ] source-pair 分桶统计 precision
- [ ] evidence-type 分桶统计 precision
- [ ] selective risk-coverage 曲线
- [ ] conformal estimated precision vs empirical precision
- [ ] calibration bucket 最小样本监控

### 运维

- [ ] 自动匹配抽样审计
- [ ] 误匹配可回溯到 extractor/model/calibration 版本
- [ ] 支持一键关闭某品牌 / 来源 / bucket 自动匹配

---

## 39. 最终推荐方案

如果只选一条可以直接落地的路线，我建议：

```text
                         ┌─────────────────────┐
                         │ 三源原始商品数据     │
                         └──────────┬──────────┘
                                    ↓
                         Brand Normalization
                                    ↓
                 Identifier Extraction + Role Classification
                                    ↓
                    Conservative Ref Canonicalization
                                    ↓
                   Exact (brand_id, canonical_ref) Index
                                    ↓
                         Candidate Group / Pair
                                    ↓
                     Hard Conflict / Accessory Guard
                                    ↓
                         Interpretable Base Matcher
                                    ↓
                       Mondrian Conformal Confidence
                                    ↓
                           Selective Reject Gate
                          ↙                     ↘
                   AUTO_MATCH                  ABSTAIN
                       ↓                         ↓
                 Entity Cluster             Human Review
                       ↓                         ↓
                       └──── Audit / Feedback ──┘
```

关键原则按优先级排序：

1. **Reference identity rule 永远高于模型相似度。**
2. **reference 的角色正确性，比字符串相似度更重要。**
3. **Conformal 用于控制自动执行面，不用于推翻硬冲突。**
4. **系统必须有 abstain；不确定就是不合并。**
5. **增量流式需求用 micro-batch 适配论文的 batch 假设。**
6. **分布漂移时默认关闭自动匹配，而不是继续沿用旧阈值。**
7. **几百黄金标签用于启动，不足以证明万分之一级别错误率；必须持续线上抽检积累证据。**
8. **图片首先用作 OCR / 冲突否决 / 人工辅助，不作为不同 reference 间的越权合并证据。**

这篇论文最大的启发，是把问题从：

```text
“怎样让 matcher 猜得更准？”
```

改写成：

```text
“怎样让系统只在自己有足够证据时自动行动？”
```

对于一个“precision 可以牺牲 recall，但不能容忍误合并”的跨源二奢实体系统，这个方向比继续堆更大的端到端模型更重要。

---

## 40. 参考资料

1. Ulf Johansson, Cecilia Sönströd, Tuwe Löfström, Henrik Boström. **Confidence Classifiers with Guaranteed Accuracy or Precision**. PMLR 204:513–533, 2023.  
   https://proceedings.mlr.press/v204/johansson23a.html
2. 论文 PDF：  
   https://proceedings.mlr.press/v204/johansson23a/johansson23a.pdf
3. 当前需求 Spec：  
   https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
