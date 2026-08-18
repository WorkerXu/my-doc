# Confidence Classifiers with Guaranteed Accuracy or Precision：用 Conformal Reject Option 给二奢 Reference 匹配增加“可拒识”的安全闸

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取论文 **Confidence Classifiers with Guaranteed Accuracy or Precision** 进行深入分析。

- 论文：<https://proceedings.mlr.press/v204/johansson23a.html>
- PDF：<https://proceedings.mlr.press/v204/johansson23a/johansson23a.pdf>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

当前 Spec 的关键约束是：

1. 三个来源、100 万～1000 万级商品，持续增量；
2. “同一个商品”被业务定义为 **同一 reference number / 型号**；
3. reference 字段稀疏，可能在结构化字段、标题或图片中；
4. **precision 极端优先，绝对不能误匹配，允许漏匹配**；
5. 有图片，可人工标注几百对黄金样本。

这篇论文最值得借鉴的不是某个具体分类模型，而是一层独立于主模型的 **Reject / Abstain 安全闸**：

> 先让任意底层 matcher 对候选进行二分类打分，再用 Conformal Classification 的 confidence 做“保留 / 拒识”排序；对于正类使用 Mondrian（分组）Conformal，可以把关注点从整体 accuracy 转到 **正类 precision**。

但它不能被误解成“给每条匹配一个 99.99% 概率，然后直接自动合并”。论文的方法有两个重要边界：

- 它给的是一批预测在 reject 机制下的统计性质，不是单条样本的绝对正确概率；
- 有效性依赖 calibration 与待预测数据近似 exchangeable，论文也明确指出其方法需要一组 test instances，不适用于逐条无上下文的纯 streaming 判定。

因此，对当前项目最合理的落地方式是：

> **Reference 硬规则负责最终业务真值，Conformal Reject Option 负责把“模型辅助路径”压缩成极小的高可信自动区间，其余全部 abstain。**

推荐最终系统不要做“商品 A ↔ 商品 B”的不可逆 pairwise merge，而是做：

```text
商品记录 -> 全局 Reference Entity
```

两条记录只有在都可靠链接到同一个 `reference_entity_id` 后，才视为“同一个商品”。

---

## 1. 为什么这篇论文特别适合当前 Spec

大多数 Entity Matching 工作默认目标是综合优化 F1，或者在 recall 和 precision 之间取平衡；但当前需求非常不对称：

```text
False Positive（错合并） >> False Negative（漏合并）
```

所以真正的问题不是：

> “怎样把 matcher 做得更准？”

而是：

> “怎样只让极小一部分足够可信的预测进入自动路径，并让模型能够明确说‘不知道’？”

这正是 classification with reject option 的目标。

论文进一步把普通 reject classifier 和 conformal prediction 结合，强调不能把普通分类器的 softmax / probability score 直接当作可信概率。即使做 Platt Scaling，极端高置信区间也可能过度自信；而当前业务恰恰最关心这个尾部区域。

这对二奢腕表非常重要。典型 hard negative 例如：

```text
126610LN  vs  126610LV
116500LN  vs  126500LN
平台 SKU   vs  品牌 Reference
主表型号   vs  “适配 XX 型号”的表带/配件标题
OCR 误读 0/O、1/I、8/B
```

这类样本在 embedding、编辑距离甚至 LLM 中都很容易得到“很像”的高分，但业务上必须严格拒绝误合并。

---

## 2. 论文方法的技术实现

## 2.1 基础层：Inductive Conformal Classifier

论文从 Inductive Conformal Prediction（ICP）开始。

训练数据被拆成两部分：

```text
Labeled Data
   ├── Proper Training Set  -> 训练底层分类器 h
   └── Calibration Set      -> 构造 nonconformity 分布
```

底层模型 `h` 可以是任意输出分类分数/概率的模型。论文实验使用的是 scikit-learn 的 Decision Tree 和 Random Forest，因此 Conformal 层本身并不依赖深度模型。

论文使用的 nonconformity score 是非常直接的形式：

```text
alpha(x, y) = 1 - P_h(y | x)
```

含义是：

- 模型越支持标签 `y`，`alpha` 越小；
- 模型越不支持标签 `y`，`alpha` 越大。

对 calibration set 中每个已知标签样本计算 `alpha`，就得到经验 nonconformity 分布。

对一个新样本 `x`，分别假设它属于每个候选标签，然后计算对应的 nonconformity，再和 calibration 分布比较，得到各标签的 conformal p-value。

二分类时可以得到：

```text
p(match)
p(non_match)
```

注意，这里的 `p(...)` 是 conformal p-value，不是普通意义上的 posterior probability。

---

## 2.2 Confidence / Credibility：论文真正用于 Reject 的分数

标准 Conformal Classification 往往输出一个 label set，但论文没有直接拿 singleton set 做自动决策，因为 singleton 的条件正确率不能简单等同于用户设定的显著性水平。

论文改为使用 confidence-credibility 表示：

- predicted label：p-value 最大的标签；
- confidence：`1 - 第二大 p-value`；
- credibility：最大的 p-value。

二分类时：

```text
confidence = 1 - min(p_match, p_non_match)
credibility = max(p_match, p_non_match)
```

两者含义不同：

- `confidence` 更像“第一名和其它标签相比有多确定”，适合做 reject 排序；
- `credibility` 更像“当前样本和 calibration 分布是否像同分布样本”，可用来辅助发现 OOD / 脏数据。

对当前项目，推荐二者都保留：

```text
confidence  -> 自动放行候选的排序依据
credibility -> 新品牌、新来源、异常 OCR、异常标题的拒识信号
```

---

## 2.3 从 Accuracy 保证转成 Precision：Mondrian Conformal

普通 ICP 主要对整体错误率提供 validity。当前项目关心的是：

```text
在所有“自动判为 match”的样本里，有多少是真的 match？
```

也就是 precision。

论文用 Mondrian Conformal 解决这个问题：把 calibration/test 样本按照一个 taxonomy 分组，然后在组内做 conformal calibration。

论文的 precision 实验使用“底层模型预测的类别”作为 taxonomy：

```text
Predicted Negative Bucket
Predicted Positive Bucket  <- 当前最关心
```

当只看 `Predicted Positive Bucket` 时，组内错误对应的就是 false positive，因此可以对正类 precision 做 reject-aware 的估计。

对当前腕表系统，第一版也应该保持 taxonomy 足够粗：

```text
bucket = predicted_match
```

而不是一开始就切成：

```text
brand × source_pair × evidence_type × category × year
```

原因是 calibration 样本只有几百对。Mondrian bucket 切得越细，每桶 calibration 数越少，p-value 分辨率越粗，极端尾部几乎不可用。

后续只有在积累足够人工复核样本后，才逐步拆成：

```text
predicted_match × evidence_mode
predicted_match × source_pair
predicted_match × major_brand
```

用于处理不同来源/证据模式的 score distribution 差异。

---

## 2.4 Reject 后的 Precision 估计

论文的关键点是：Conformal confidence 不应该被解释成某一条样本的正确概率，而是和一组 test predictions 一起解释。

假设在某个正类 bucket 中：

- 一共 `n` 条 predicted-positive；
- 设定 confidence cutoff 为 `lambda`；
- cutoff 后剩下 `m` 条自动接受样本；

论文使用下面的期望错误率形式：

```text
estimated_error_rate = n * (1 - lambda) / m
estimated_precision  = 1 - estimated_error_rate
```

因此真正的部署逻辑不是：

```python
if confidence > 0.99:
    merge()
```

而是：

```text
1. 收集一批 predicted-positive candidates
2. 按 conformal confidence 从高到低排序
3. 枚举 cutoff
4. 计算该 cutoff 下的 estimated precision
5. 找到满足目标 precision 的最大可覆盖集合
6. 其它样本全部 abstain
```

这是一种 **batch-level selective decision**。

---

## 2.5 论文方法的关键限制

### 限制 1：不是纯 Streaming 算法

论文明确指出，该过程需要“一组 test instances”才能计算 reject 后集合的统计量，因此不适合一条来一条立即独立决策。

当前需求虽然持续增量，但没有要求毫秒级实时合并，所以可以改造成：

```text
实时/准实时摄取
      ↓
候选与模型分数实时生成
      ↓
进入 decision buffer
      ↓
按小批次 release
      ↓
Conformal reject + 自动放行/人工复核
```

即：**流式摄取，微批决策**。

### 限制 2：Exchangeability / Distribution Drift

Conformal validity 依赖 calibration 与未来样本的可交换性假设。

但二奢业务很容易漂移：

- 新品牌/新系列上线；
- 某平台改标题模板；
- OCR 模型升级；
- 来源字段规则变化；
- 某批商家大量发布配件而非主表。

因此不能把“Conformal 有 guarantee”理解为“线上永远 100% 安全”。

正确做法是：

- calibration 数据必须和实际自动决策路径一致；
- extractor / normalizer / matcher 版本变化后重新校准；
- 新来源或明显漂移 bucket 默认进入 abstain/shadow；
- 只有重新收集标签并校准后再打开自动 lane。

### 限制 3：几百标签不足以证明“四个 9”的业务安全

几百对黄金标签足够：

- 建立初版 verifier；
- 做 hard-negative 评估；
- 验证 Conformal reject 流程；
- 找明显错误模式。

但它不足以从统计上证明 99.99% 甚至“绝对零误匹配”。

因此当前 Spec 中的“绝对不能误匹配”必须主要由 **业务确定性约束 + 可拒识机制** 实现，而不能只靠概率阈值。

---

## 3. 对当前需求的问题重构

如果“同一个商品”就是“同一个 reference”，最安全的实体模型是：

```text
Source Product Record
        │
        │ link
        ▼
Global Reference Entity
```

而不是：

```text
Product A ----match?---- Product B
Product B ----match?---- Product C
```

原因有三点：

1. pairwise matcher 会产生传递污染：一条错误正边可能把多个商品并进同一簇；
2. reference 本身就是天然 canonical entity key；
3. record -> reference 的错误更容易定位、回滚和审计。

建议数据模型中把“匹配”定义为一条可追溯链接：

```text
record_reference_link
- record_id
- reference_entity_id
- decision_state
- evidence_json
- rule_version
- model_version
- calibrator_version
- confidence
- credibility
- created_at
```

不要直接把原始商品数据物理 merge 掉。

---

## 4. 推荐的整体架构

```text
┌──────────────────────────────────────────────┐
│  雷小安 / 腕表之家 / 奢当家 Raw Records      │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  1. Normalize & Evidence Extraction          │
│  - brand canonicalization                    │
│  - title reference token extraction          │
│  - structured reference extraction           │
│  - OCR reference extraction                  │
│  - identifier role classification            │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  2. Global Reference Registry                │
│  canonical_ref + brand + aliases + provenance│
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  3. Candidate Retrieval                      │
│  只在 brand / series / ref-token block 内召回 │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  4. Deterministic Hard Gates                 │
│  - 品牌冲突 -> reject                         │
│  - 明确不同 reference -> reject               │
│  - SKU/配件编号 -> reject                     │
│  - 多证据冲突 -> reject                       │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  5. Base Verifier                            │
│  输出 match / non-match score                │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  6. Mondrian Conformal Reject Layer          │
│  confidence / credibility / batch cutoff     │
└──────────────────────┬───────────────────────┘
                       │
           ┌───────────┼────────────┐
           ▼           ▼            ▼
    AUTO_LINK       REVIEW       ABSTAIN
           │           │            │
           └───────────┴─────┬──────┘
                             ▼
                  Human Feedback / Gold Set
                             │
                             └──> retrain + recalibrate
```

核心原则：

> **Hard Gate 拥有否决权，Conformal 只有“缩小自动集合”的权力，没有推翻硬冲突的权力。**

---

## 5. Reference 抽取与规范化：必须先于 Matcher

## 5.1 保存 raw 与 canonical 两份值

任何 reference 都必须保留：

```text
raw_reference
canonical_reference
normalizer_version
source_field
```

不要只保存清洗后的字符串，否则后续无法审计“为什么两个型号被合并”。

## 5.2 Normalization 必须保守

建议只做确定性、可逆或高度安全的规范化：

```text
Unicode NFKC
trim
uppercase
全角 -> 半角
品牌明确允许的 separator normalization
```

谨慎处理：

```text
- / . 空格
```

因为某些品牌 reference 中 separator、后缀、材质码、表盘码可能有业务意义。

绝对不要用“编辑距离很近”直接 canonicalize：

```text
126610LN != 126610LV
```

## 5.3 Identifier Role Classification

标题里出现一个像型号的字符串，不代表它就是当前商品的 reference。

至少要区分：

```text
BRAND_REFERENCE
PLATFORM_SKU
SELLER_SKU
ACCESSORY_COMPATIBLE_REFERENCE
SERIAL_NUMBER
ORDER_ID
UNKNOWN_IDENTIFIER
```

特别是“适配某型号”的表带、表盒、配件，可能在标题中包含主表 reference。

因此 reference extraction 应输出结构：

```json
{
  "raw": "126610LN",
  "canonical": "126610LN",
  "role": "BRAND_REFERENCE",
  "evidence_source": "title",
  "span": [12, 20],
  "extractor_score": 0.98
}
```

而不是只输出一个裸字符串。

---

## 6. Deterministic Hard Gates

在任何机器学习模型之前先执行强约束。

推荐第一版规则：

### Gate A：品牌冲突

```text
canonical_brand(A) != canonical_brand(B)
=> REJECT
```

除非存在明确的品牌映射错误修正规则，否则不要让模型覆盖。

### Gate B：强 reference 冲突

如果两边都从高可信字段获得了品牌 reference：

```text
ref_A != ref_B
=> REJECT
```

### Gate C：多证据冲突

例如：

```text
structured_ref = 126610LN
title_ref      = 126610LV
```

不要“投票”选一个，而是直接：

```text
ABSTAIN / REVIEW
```

因为当前业务错误成本极高。

### Gate D：编号角色冲突

如果候选 token 被判为：

```text
PLATFORM_SKU
SELLER_SKU
ACCESSORY_COMPATIBLE_REFERENCE
```

则不得作为自动 Reference Link 的主证据。

### Gate E：Reference Entity 唯一性

在 `brand + canonical_ref` 下应有唯一 `reference_entity_id`。

如果同一个 canonical key 对应多个知识库实体，先解决 registry 冲突，不要把歧义下推给商品 matcher。

---

## 7. Candidate Retrieval：只负责找候选，不负责确认

当 reference 在结构化字段中明确存在时，实际上不需要通用 ANN Matching：

```text
brand + canonical_ref -> exact lookup
```

只有以下情况进入 candidate retrieval：

- reference 只在标题中；
- OCR 有少量字符噪声；
- 有品牌/系列但 reference 缺失；
- 标题存在多个可能 identifier；
- reference 只解析出部分前缀/后缀。

候选召回优先级建议：

```text
1. 品牌内 Reference Trie / Hash Index
2. 品牌内 token / n-gram 候选
3. 品牌内 edit-neighbor（仅召回）
4. 系列 / 属性 block
5. 文本 embedding / ANN（最后兜底）
```

任何 fuzzy / ANN 相似度都只能用于生成候选集合，不能直接产生 `AUTO_LINK`。

---

## 8. Base Verifier：模型应该学什么

Conformal 层是 model-agnostic，所以第一版不需要重模型。

建议从可解释的 LightGBM / Logistic Regression 开始，输入候选 `record -> reference_entity` 的结构化特征：

```text
ref_exact_match
ref_edit_distance
ref_prefix_match
ref_suffix_match
brand_match
series_match
model_name_match
structured_field_support
text_span_support
ocr_support
text_ocr_agreement
num_reference_candidates
identifier_role
accessory_keyword_flag
conflicting_reference_count
source_pair
```

模型输出：

```text
P(match | features)
```

但这只是生成 nonconformity 的底层 score，不是最终业务概率。

### 为什么不建议第一版直接用 LLM / 多模态大模型做最终判定

因为当前难点不是语义理解能力不足，而是：

- 同系列近邻 reference 极其相似；
- 模型通常倾向“给一个答案”，而不是拒绝；
- 高置信度分数不天然校准；
- 错一次的成本远高于漏一次。

LLM/VLM 更适合：

```text
标题/OCR 属性抽取
候选解释
人工复核辅助
```

而不是最终 merge oracle。

---

## 9. Conformal Reject Layer 的工程实现

## 9.1 离线训练

```python
# 伪代码
proper_train, calibration = split_labeled_data(gold_pairs)

model.fit(proper_train.X, proper_train.y)

cal_probs = model.predict_proba(calibration.X)
cal_pred  = argmax(cal_probs)

# 论文中的 hinge-style nonconformity
alpha_i = 1 - cal_probs[i, calibration.y[i]]

# Mondrian bucket：先按底层预测类分桶
bucket_i = cal_pred[i]
store(alpha_i, bucket_i)
```

当前项目最关心：

```text
bucket = predicted_match
```

## 9.2 在线/微批打分

对每个候选：

```python
prob = model.predict_proba(x)
pred = argmax(prob)

# 对每个 tentative label 算 nonconformity，
# 再与同 Mondrian bucket 的 calibration scores 比较得到 conformal p-value
p_match = conformal_pvalue(...)
p_non_match = conformal_pvalue(...)

confidence = 1 - min(p_match, p_non_match)
credibility = max(p_match, p_non_match)
```

输出进入 `decision_buffer`：

```text
candidate_id
predicted_label
raw_model_score
p_match
p_non_match
confidence
credibility
hard_gate_state
calibrator_version
```

## 9.3 Batch Precision Cutoff

只处理：

```text
hard_gate_state = PASS
predicted_label = MATCH
```

设这一批共有 `n` 个 predicted match，按 `confidence DESC` 排序。

对每个候选 cutoff `lambda_k`：

```python
m = k
estimated_error = n * (1 - lambda_k) / m
estimated_precision = 1 - estimated_error
```

然后找到满足目标 precision 的最大 `m`。

伪代码：

```python
def select_auto_link(candidates, target_precision):
    cands = sorted(candidates, key=lambda x: x.confidence, reverse=True)
    n = len(cands)
    best_k = 0

    for k, c in enumerate(cands, start=1):
        lam = c.confidence
        est_precision = 1.0 - n * (1.0 - lam) / k

        if est_precision >= target_precision:
            best_k = k

    return cands[:best_k], cands[best_k:]
```

这里最重要的工程纪律是：

> `confidence` 不要在 API 层被当作“单条正确率”展示，也不要写成固定 `confidence > 0.99` 的业务规则。

---

## 10. 如何处理“持续增量”与论文的 Batch 限制

推荐使用 **Streaming Ingest + Micro-batch Decision**。

```text
Kafka / Queue / DB CDC
        ↓
Normalizer / Extractor
        ↓
Candidate + Verifier Score
        ↓
Decision Buffer
        ↓
按数量或时间组成 micro-batch
        ↓
Conformal Reject
        ↓
Release Auto Links
```

每个 batch 记录：

```text
decision_batch_id
calibrator_version
model_version
extractor_version
source_distribution
candidate_count
auto_link_count
abstain_count
estimated_precision
```

这样既支持持续增量，又保留论文方法需要的集合语义。

如果业务未来要求单条毫秒级立即返回，则不能直接照搬本文公式；那时要么：

- 返回 `PENDING`，后台微批完成后再落 link；
- 要么改用专门的 online conformal / risk-control 方法重新设计，不能假装当前 batch guarantee 仍然成立。

---

## 11. Calibration 与数据漂移管理

建议把 calibrator 当成独立生产资产，不要绑死在 matcher 代码里。

```text
calibrator_registry
- calibrator_version
- model_version
- feature_schema_version
- extractor_version
- taxonomy
- calibration_data_snapshot
- created_at
- active
```

以下事件触发重新校准或关闭自动 lane：

```text
新来源上线
新品牌大批量进入
标题模板明显变化
OCR / Extractor 升级
Reference Normalizer 规则变化
Base Verifier 重训
Gold Set 错误分布发生明显变化
```

如果检测到分布漂移但还没有新标签，默认行为应该是：

```text
AUTO_LINK -> REVIEW / ABSTAIN
```

而不是继续沿用旧阈值。

---

## 12. 黄金标签应该怎么标，才真正帮助 precision-first

不要随机抽几百对。

随机样本里绝大部分 pair 很容易，无法覆盖 false-positive 风险。

应主动构造 hard negatives：

```text
同品牌 + 同系列 + reference 只差 1 个字符
同品牌 + 同型号名 + 不同尺寸/材质后缀
主表 vs 表带/配件
标题含“适配 reference”
平台 SKU 长得像 reference
OCR 0/O、1/I、5/S、8/B 混淆
同图/相似图但 reference 不同
同 reference 不同写法（空格、连字符、大小写）
```

建议 Gold Set 分层：

```text
Easy Positive
Easy Negative
Hard Positive
Hard Negative       <- 最高优先级
Conflict / Abstain  <- 单独标记
```

而且测试集不要只做随机 split，应至少增加：

```text
按 reference 留出
按时间留出
按来源组合留出
按品牌留出
```

避免同一 reference 的近重复记录同时出现在 train/test 中造成虚高指标。

---

## 13. 自动决策状态不要只有 Match / Non-match

建议状态机：

```text
AUTO_LINK
REVIEW
ABSTAIN
REJECT
```

语义：

### AUTO_LINK

满足确定性业务约束，并通过当前 calibrator 的 selective release 条件。

### REVIEW

证据可能足够，但存在可解释歧义，需要人工确认。

### ABSTAIN

模型/证据不足，不做任何实体链接。

### REJECT

存在明确冲突，例如品牌不同、两个强 reference 明确不一致。

这能避免把“没有足够证据”错误地当成“确定不匹配”。

---

## 14. 图片在这套架构中的正确位置

图片不应该直接成为“看起来一样，所以同 reference”的最终证据。

更合理的作用是：

```text
图片
 ├── OCR 表背 / 保卡 / 吊牌 -> reference evidence
 ├── 识别品牌 / 系列 -> candidate filter
 └── 视觉冲突 -> veto / review signal
```

推荐优先做 OCR evidence，而不是先训练端到端视觉 matcher。

例如：

```text
title_ref = 126610LN
ocr_ref   = 126610LN
=> 独立证据一致，可信度上升
```

而：

```text
title_ref = 126610LN
ocr_ref   = 126610LV
=> 直接 REVIEW / ABSTAIN
```

不要让模型通过“图像整体很像”覆盖 reference 冲突。

---

## 15. 推荐的数据表

## 15.1 `product_record`

```text
record_id
source
source_record_id
raw_title
raw_attributes_json
image_urls
created_at
updated_at
```

## 15.2 `reference_evidence`

```text
evidence_id
record_id
raw_reference
canonical_reference
brand_id
role
evidence_source   # structured/title/ocr/manual
extractor_score
extractor_version
span_or_image_id
```

## 15.3 `reference_entity`

```text
reference_entity_id
brand_id
canonical_reference
series
model_name
status
registry_version
```

唯一索引：

```text
UNIQUE(brand_id, canonical_reference)
```

## 15.4 `candidate_link`

```text
candidate_id
record_id
reference_entity_id
feature_json
hard_gate_state
model_score
p_match
p_non_match
confidence
credibility
model_version
calibrator_version
```

## 15.5 `record_reference_link`

```text
record_id
reference_entity_id
decision_state
candidate_id
decision_batch_id
decision_reason
created_at
revoked_at
```

建议 link 可撤销，不要直接修改历史事实。

---

## 16. P0 / P1 / P2 落地顺序

## P0：先做“确定性 Reference Entity Link”

目标：不依赖复杂模型，立即获得一条高 precision 主路径。

实现：

```text
品牌规范化
Reference 保守规范化
结构化字段提取
标题 identifier 提取
Identifier role 分类规则
Reference Registry
brand + canonical_ref exact lookup
冲突即 abstain
全链路 provenance
```

此阶段就能处理三个来源中 reference 明确的很大一部分记录。

## P1：加入 Base Verifier + Conformal Reject

只处理 P0 无法直接确定的 ambiguous records：

```text
候选 Reference 召回
LightGBM / Logistic verifier
proper train / calibration split
Mondrian positive bucket
confidence / credibility
micro-batch selective release
人工 REVIEW 队列
```

这里推荐先把模型路径放在 shadow mode：

```text
模型给 AUTO_LINK 建议
但先不真正写生效 link
人工抽检/复核一段时间
确认 hard-negative 零事故后再逐步开放
```

## P2：图像/OCR + 漂移治理

加入：

```text
表背/保卡 OCR
图片品牌/系列辅助
多证据冲突 veto
calibrator registry
drift detection
按 evidence/source 的 Mondrian taxonomy
active learning hard negatives
```

---

## 17. 一个可直接实施的决策矩阵

| 场景 | 决策 |
|---|---|
| 两边都有高可信 structured reference，规范化后完全一致，品牌一致，无冲突 | `AUTO_LINK` |
| 两边都有高可信 reference，但不一致 | `REJECT` |
| 一边 structured ref，另一边 title/OCR ref，完全一致，且无其它 ref 冲突 | 可进入高可信 lane；建议初期仍经 conformal/shadow |
| 只有标题抽取，只有 fuzzy similarity | `ABSTAIN/REVIEW` |
| 标题和 OCR reference 一致，且落到唯一 Reference Entity | 可进入 verifier + conformal lane |
| 标题和 OCR reference 冲突 | `REVIEW` |
| 品牌不同 | `REJECT` |
| “适配 XX 型号”配件 | 不允许用 XX 直接 link 主表 Reference |
| 模型分数极高但 hard gate 冲突 | `REJECT` |
| 新品牌/新来源没有有效 calibrator | `ABSTAIN` |

---

## 18. 线上监控指标

不要只看 F1/AUC。

最重要的是：

```text
Auto-link False Positive Count
Auto-link Precision
Auto-link Coverage
Abstain Rate
Review Rate
Conflict Rate
Reference Extraction Conflict Rate
Per-source / Per-brand Error
Calibration Drift
```

尤其需要单独维护：

```text
false_positive_incident
- record_id
- wrong_reference_entity_id
- correct_reference_entity_id
- evidence
- root_cause
- model_version
- calibrator_version
- normalizer_version
```

每次 false positive 都必须变成：

```text
新 hard-negative
+ 新回归测试
+ 必要时新增 hard gate
```

因为当前业务里，错误不是普通指标波动，而是规则系统需要吸收的事故样本。

---

## 19. 这篇论文可以直接借鉴什么，不能照搬什么

### 可以直接借鉴

1. **Reject Option 是一等公民**，不是分类阈值失败后的补丁；
2. Conformal 层与底层 matcher 解耦，底层可以是规则、树模型、Cross Encoder；
3. 用 Mondrian bucket 把统计关注点放到 predicted-positive，从而面向 precision；
4. 用 batch-level confidence cutoff 选择自动处理覆盖率，而不是固定 softmax 阈值；
5. probability calibration 在极端 reject 区域可能仍过度自信，必须单独验证。

### 不能照搬

1. 论文数据集规模小，不能直接等同于 100 万～1000 万生产数据；
2. 论文方法要求 test batch，不能伪装成逐条 streaming guarantee；
3. exchangeability 在来源持续变化时可能被破坏；
4. 几百 calibration 样本不足以支撑业务意义上的“绝对零错”；
5. 论文解决的是通用二分类 reject，而当前业务已有极强的 `reference == entity key` 结构，应优先利用业务规则。

---

## 20. 最终推荐方案

对当前 Spec，推荐把系统设计成 **“Reference Entity Resolution + Hard Gates + Conformal Selective Release”**：

```text
第一层：Reference Extraction / Normalization
第二层：Global Reference Registry
第三层：Candidate Retrieval
第四层：Deterministic Conflict Gates
第五层：Base Verifier
第六层：Mondrian Conformal Reject
第七层：AUTO_LINK / REVIEW / ABSTAIN / REJECT
第八层：人工反馈 -> Hard Negative -> Recalibration
```

最终自动合并原则建议写死为：

> **模型永远不能推翻明确 reference 冲突；fuzzy/embedding/image similarity 永远不能单独触发 AUTO_LINK；Conformal 的职责是继续减少模型路径的自动覆盖率，而不是为不确定证据制造“高概率正确”的幻觉。**

如果必须在“更多自动化”和“绝对不误匹配”之间选，当前 Spec 应始终选择后者。

从工程收益看，这篇论文最大的价值是给系统补上一层传统 Entity Matching 项目常缺少的能力：

```text
我不知道，所以我不匹配。
```

而这恰好是当前 precision-first 二奢实体匹配最重要的能力之一。

---

## 21. 参考资料

1. Johansson, U., Sönströd, C., Löfström, T., Boström, H. **Confidence Classifiers with Guaranteed Accuracy or Precision**. PMLR 204, 2023.  
   <https://proceedings.mlr.press/v204/johansson23a.html>
2. 论文 PDF：  
   <https://proceedings.mlr.press/v204/johansson23a/johansson23a.pdf>
3. 当前需求 Spec：  
   <https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
4. 调研来源：`WorkerXu/my-doc/奢侈品文章调研.md`
