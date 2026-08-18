# Confidence Classifiers with Guaranteed Accuracy or Precision：面向跨源腕表实体匹配的高精度拒识架构

## 0. 结论先行

本次选择并深入分析的文章是：

- **Confidence Classifiers with Guaranteed Accuracy or Precision**
- Ulf Johansson, Cecilia Sönströd, Tuwe Löfström, Henrik Boström
- COPA / PMLR 204, 2023
- 论文页：https://proceedings.mlr.press/v204/johansson23a.html
- PDF：https://proceedings.mlr.press/v204/johansson23a/johansson23a.pdf

选择它的原因不是它能直接解决 reference 抽取，而是它解决了当前 Spec 中最难落地的一层：**当系统已经有了一个“候选是否同 reference”的打分模型时，如何把模型变成一个可以大量拒识、只自动放行极少数高可信匹配的决策器，并且使放行集合的 precision 可被校准和解释。**

当前 Notion Spec 的关键约束是：

1. 三个来源：雷小安、腕表之家、奢当家；
2. 数据规模 100 万–1000 万，并持续增量；
3. “同一个商品”定义为 **同一 reference number / 型号**；
4. 字段稀疏，reference 既可能是结构化字段，也可能埋在标题、描述或图片中；
5. **precision 极端优先，绝对不能误匹配，允许漏匹配**；
6. 可以接受人工标注几百对黄金标签；
7. 有图片可用。

针对这个约束，我的核心建议是：

> **不要把任何模型的 `match_probability > 0.99` 直接等价为“可以自动合并”。**
>
> 应将系统拆成“reference 证据抽取/规范化 → 候选生成 → 基础判别器 → Mondrian conformal precision gate → 硬规则否决 → 保守聚类”几层。模型只负责排序与评分，真正控制自动合并的是 precision gate 和 fail-closed 规则。

同时有一个很重要的现实限制：

> **仅靠“几百对”黄金标签，无法从统计上证明 99.99% 甚至 99.9% 的自动匹配 precision。**
>
> Conformal 的有限样本分辨率决定了：若一个 Mondrian 校准桶里只有 `q` 个有效校准样本，最理想情况下可分辨的错误概率量级大约也只能到 `1/(q+1)`。例如 q=300 时上限量级约 99.67%，q=1000 才到约 99.90%，q=10000 才到约 99.99%。
>
> 因而本项目不能指望“一个模型 + 300 标签 + 0.999 阈值”来获得近零误匹配；真正的高 precision 必须由 **reference 硬证据 + 角色判断 + 冲突否决 + conformal 拒识** 共同实现。

---

## 1. 已排除我此前分析过的条目

在 `奢侈品调研/b/` 下已经存在以下分析，因此本次不重复：

- Ameli
- DeepBlocker
- Ditto
- Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration
- GraLMatch
- Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes
- Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification
- TransClean
- parts-distributor-sku-classifier
- pyJedAI

本次文章不在上述历史结果中。

---

## 2. 论文真正解决了什么问题

传统二分类模型一般输出：

```text
P(match | pair) = 0.997
```

工程上最常见的做法是：

```text
if p >= 0.99:
    auto_match()
else:
    reject_or_review()
```

问题是，**模型分数并不是 precision 保证**。

即使经过 Platt scaling / temperature scaling，`0.99` 也只是在某种整体校准意义上接近概率，并不意味着“所有 >=0.99 的预测集合，实际 precision 一定接近 99%”。特别是在：

- hard negative 很多；
- 新品牌/新来源持续进入；
- 分布漂移；
- 正负样本极不平衡；
- 模型只见过少量黄金标签；

这些情况下，高分 false positive 恰恰是最危险的。

论文采用 **Conformal Classification + Reject Option**，关注的不是“每个单独样本到底有多大概率正确”，而是：

> 当我只保留 confidence 足够高的一批预测、拒绝其余样本时，这一批被保留预测的整体 accuracy / precision 应该是多少？

这个思路与 Spec 高度一致，因为 Spec 明确允许漏匹配。

---

## 3. 论文技术实现深挖

### 3.1 基础模型并不特殊

论文底层模型只使用了：

- Decision Tree
- Random Forest

也就是说，Conformal 层不要求底座必须是某种神经网络。

对本项目而言，这反而是好事：

- 底座可以是 LightGBM / XGBoost；
- 可以是文本 cross-encoder；
- 可以是 LLM 打分器；
- 也可以是规则 + 模型混合打分。

**Conformal precision gate 是“决策层”，不是“表征层”。**

---

### 3.2 Inductive Conformal Classification（ICP）

论文把带标签的数据拆成：

```text
proper training set Z_t
calibration set     Z_c
```

基础分类器 `h` 只在 `Z_t` 上训练。

对 calibration 样本，计算 nonconformity score。论文采用的是基于基础模型概率的 hinge 形式：

```text
alpha_i = 1 - P_h(y_i | x_i)
```

直观上：

- 模型越确信真实标签，alpha 越小；
- 模型越不符合真实标签，alpha 越大。

对测试样本 `x`，分别假设它属于每一个可能标签，再计算相应 nonconformity score，并与 calibration score 排序比较，得到每个标签的 conformal p-value。

工程化的非随机近似写法可以理解为：

```text
p_y(x) = (#{i in calibration: alpha_i >= alpha(x, y)} + 1) / (q + 1)
```

论文正式公式还对 score tie 使用随机项做平滑。

Conformal 的理论基础依赖 calibration 与 test 的 **exchangeability**。这意味着：若线上分布完全变化，保证也不能无条件成立。

---

### 3.3 Confidence 与 Credibility

对每个测试样本会得到两个类别的 p-value，例如：

```text
p(match)     = 0.82
p(nonmatch)  = 0.01
```

论文定义：

```text
predicted_label = argmax(p_y)
confidence      = 1 - second_largest(p_y)
credibility     = largest(p_y)
```

在二分类中，confidence 本质上取决于“次优标签有多容易被排除”。

这是和普通 softmax probability 很不同的一点：

- softmax 主要描述当前单条预测；
- conformal confidence 被解释为与**一个被筛选出来的预测集合**相关。

因此它天然适合 reject option。

---

### 3.4 拒识后的 accuracy 估计

论文最关键的公式是：

假设当前测试集合共有 `n` 个预测，只保留 confidence 不低于 `lambda` 的 `m` 个预测，其余全部拒绝，则保留集合的期望 error rate 为：

```text
error_est = n * (1 - lambda) / m
```

于是：

```text
accuracy_est = 1 - n * (1 - lambda) / m
```

这不是把 `lambda` 当单样本概率，而是在解释“经过当前 reject threshold 后的集合”。

---

### 3.5 为什么 Mondrian Conformal 可以瞄准 precision

普通 ICP 对总体 error rate 做有效性约束，更自然对应 accuracy。

论文进一步使用 **Mondrian Conformal Classification**：

- 先按某种 taxonomy 把样本分桶；
- 每个桶单独做 conformal 校准；
- 每个桶都拥有自己的有效性性质。

论文在 precision 实验里使用的 taxonomy 是：

```text
由基础模型预测出来的 label
```

只看“基础模型预测为正类”的那个类别，那么这个桶中的错误就对应：

```text
预测为 match，但真实是 non-match
```

也就是 false positive。

所以在该桶里：

```text
1 - error_rate ≈ precision
```

这一步是整篇论文对本项目最有价值的点。

---

### 3.6 论文的实验结论

论文使用 10 个公开二分类数据集，比较：

1. Uncalibrated probability；
2. Platt scaling；
3. Conformal confidence。

实验会按置信度从高到低保留预测，并逐步把 10% 到 90% 的样本拒绝。

结果的核心不是“Conformal 基础分类准确率最高”，而是：

> **当系统需要估计“当前被自动放行这一批预测到底有多高 precision”时，Conformal 的估计偏差明显小于普通 probability 和 Platt scaling。**

论文 Table 13 中，precision estimation 的平均 signed error 接近 0，且 absolute error 在不同 rejection level 上都明显更稳定。

这说明它最适合做：

```text
score -> reject -> auto-accept subset
```

而不是替代实体匹配模型本身。

---

## 4. 论文方案不能直接照搬的地方

### 4.1 论文明确不是 streaming 方法

论文要求能看到“一批 test instances”，然后对这一批做 reject 分析；作者明确指出它不能直接按纯 streaming 方式理解。

而我们的数据是持续增量。

因此不能做：

```text
每来一条 pair
-> 单独计算一个 confidence
-> 立刻宣称当前 precision 有保证
```

更合理的是：

```text
增量流
  -> 微批次窗口
  -> 对一个 batch 内的 predicted-positive pairs 统一排序
  -> 选择 accept cutoff
  -> batch commit
```

推荐窗口：

- 高频业务：15 分钟 / 1 小时微批；
- 低频业务：每日批处理；
- 若强实时：只有“reference 双端结构化 exact match + 无冲突”的硬规则路径可以实时，模型疑难路径等待批处理。

---

### 4.2 Exchangeability 是最重要的前提

新增来源、品牌、标题模板变化、OCR 模型升级，都可能破坏 calibration/test exchangeability。

因此必须有：

```text
source_pair
brand_family
reference_evidence_tier
extractor_version
model_version
```

这些维度上的漂移监控。

一旦漂移超阈值，应：

```text
auto-match disabled
-> review only
-> collect labels
-> recalibrate
```

也就是 fail closed。

---

### 4.3 “几百标签”不足以支撑 99.99% 形式保证

这是最容易被忽略但最重要的工程结论。

Conformal p-value 的有限样本分辨率与 calibration size `q` 有直接关系。

最小 p-value 量级约是：

```text
1 / (q + 1)
```

因此极理想情况下，单一 calibration 桶可以支撑的 precision 分辨率大约是：

| calibration q | 理想 precision 分辨率量级 |
|---:|---:|
| 100 | 99.01% |
| 300 | 99.67% |
| 500 | 99.80% |
| 1,000 | 99.90% |
| 10,000 | 99.99% |

如果再把 calibration 分成：

```text
3 个 source pair
× 多种 evidence tier
× brand family
```

每个桶里的 q 会更小。

所以不能把少量黄金标签“切太细”。

本项目的正确策略是：

> **把 precision 的主要保障交给 reference deterministic evidence；Conformal 只负责对模型辅助路径做保守拒识。**

---

## 5. 针对当前 Spec 的推荐总体架构

```text
                         ┌──────────────────────────┐
                         │ 雷小安 / 腕表之家 / 奢当家 │
                         └─────────────┬────────────┘
                                       │
                              Raw ingestion
                                       │
                         ┌─────────────▼────────────┐
                         │  Canonical Record Layer  │
                         │ brand/title/desc/images  │
                         └─────────────┬────────────┘
                                       │
                         ┌─────────────▼────────────┐
                         │ Reference Evidence Layer │
                         │ field / text / OCR       │
                         │ role + canonical ref     │
                         └─────────────┬────────────┘
                                       │
                         ┌─────────────▼────────────┐
                         │ Blocking / Candidate Gen │
                         │ exact ref first          │
                         │ retrieval only fallback  │
                         └─────────────┬────────────┘
                                       │
                         ┌─────────────▼────────────┐
                         │ Pair Validator / Scorer  │
                         │ rule + GBM / encoder     │
                         └─────────────┬────────────┘
                                       │
                         ┌─────────────▼────────────┐
                         │ Mondrian Conformal Gate  │
                         │ precision-oriented reject│
                         └─────────────┬────────────┘
                                       │
                         ┌─────────────▼────────────┐
                         │ Hard Negative Veto Layer │
                         │ role/conflict/accessory  │
                         └─────────────┬────────────┘
                                       │
             ┌─────────────────────────┼────────────────────────┐
             │                         │                        │
      AUTO_MATCH                 HUMAN_REVIEW                REJECT
             │
      ┌──────▼──────┐
      │ Safe Cluster │
      │ union-find   │
      │ consistency  │
      └──────────────┘
```

---

## 6. 第一层：reference evidence，而不是直接 pair matching

因为 Spec 已经定义：

```text
same product := same reference
```

所以真正核心不是先训练一个“同商品/不同商品”黑盒模型，而是先建立一个可审计的 `reference_evidence` 层。

推荐数据结构：

```sql
reference_evidence (
    evidence_id          bigint,
    record_id            bigint,
    source               varchar,
    brand_canonical      varchar,

    raw_value            varchar,
    canonical_reference  varchar,

    evidence_source      varchar,  -- structured_field/title/description/ocr
    evidence_location    varchar,  -- field name / text span / image id
    evidence_role        varchar,  -- PRODUCT_REF / COMPATIBLE_REF / SKU / SERIAL / ...

    extractor_name       varchar,
    extractor_version    varchar,
    confidence           float,

    parser_valid         boolean,
    created_at           timestamp
)
```

必须保留 raw value，不能只存 canonical reference。

因为一旦发生误匹配，需要追溯：

```text
“这个 126610LN 到底来自平台 reference 字段、标题正文、表背 OCR，还是模型猜的？”
```

---

## 7. reference 规范化不要做成简单 `remove_non_alnum`

腕表 reference 常包含：

- 数字主体；
- 字母后缀；
- 斜杠；
- 点号；
- 连字符；
- 空格变体；
- 品牌特定格式。

如果统一写成：

```python
re.sub(r'[^A-Z0-9]', '', ref.upper())
```

风险很高，因为不同品牌的分隔符可能具有语义。

建议：

```text
global normalization
  -> Unicode NFKC
  -> uppercase
  -> normalize whitespace
  -> normalize full-width chars
  -> normalize visually equivalent hyphens

brand parser
  -> regex / dictionary / check digit / known family grammar
  -> canonical form
```

例如：

```text
normalize_global(raw)
-> parse_by_brand(brand, normalized)
-> canonical_ref
```

而不是无品牌全局强行压平。

---

## 8. 必须有“编号角色分类”

系统中所有“像型号”的字符串都不能默认是 PRODUCT_REFERENCE。

至少要区分：

```text
PRODUCT_REFERENCE       当前售卖主体的 reference
COMPATIBLE_REFERENCE    配件标题中“适配 XXX 型号”
ACCESSORY_REFERENCE     表带/表扣/盒证自身编号
PLATFORM_SKU            平台商品号
SHOP_SKU                商家内部货号
SERIAL_NUMBER           独立序列号
ORDER_ID / OTHER_ID     其它编号
```

一个典型高危标题：

```text
适用 Rolex 116610 / 126610 黑水鬼表带 原装规格
```

如果直接抽到 `116610` 或 `126610` 并用 exact match 合并，会把配件错误合到整表记录上。

因此：

```text
reference exact equality
```

只是必要条件之一，必须再加：

```text
role == PRODUCT_REFERENCE
```

---

## 9. Candidate Generation：优先 hash join，不要一上来向量召回

100 万–1000 万数据量若做三源全量 pairwise，数量级会爆炸。

但 reference 是高辨识度键，因此候选生成应分层。

### 9.1 Path A：双端有可信 reference

```text
brand_canonical + canonical_reference
```

直接 hash join。

复杂度接近：

```text
O(N)
```

而不是全量：

```text
O(N^2)
```

### 9.2 Path B：一端 reference 缺失

只能用于“候选发现”，不能直接自动匹配。

可以用：

- brand；
- series/model family；
- title embedding；
- image embedding；
- OCR token；

召回少量候选。

但真正自动放行前，缺失端必须重新得到可信 `PRODUCT_REFERENCE` 证据。

### 9.3 Path C：双方都没有 reference

建议：

```text
永不自动合并
```

仅用于：

- review queue；
- 生成标注样本；
- 训练 reference extractor；
- 发现可能存在的型号词典。

这是 precision-first 系统应该主动接受的 recall 损失。

---

## 10. Pair Validator 应该验证“reference 是否属于当前商品”

推荐基础模型的任务不是：

```text
这两张表看起来像不像同款？
```

而是更窄的问题：

```text
两个 record 中被抽取出来的 canonical_reference，
是否都高可信地描述当前售卖主体？
```

推荐特征：

### 10.1 Reference 特征

```text
brand_equal
canonical_ref_equal
ref_parser_valid_left/right
ref_source_trust_left/right
ref_role_score_left/right
ref_span_context_score
ref_dictionary_hit
```

### 10.2 文本一致性

```text
brand / collection / family
case size
material
movement
color
year range
```

注意：这些只能做**冲突检测**，不能替代 reference。

例如同 reference 在不同描述里可能漏字段，所以不应要求所有属性完全相同。

### 10.3 负证据

```text
accessory keywords
compatible-with context
replica/fake context
box/papers only
strap/bracelet/buckle only
platform sku pattern
serial number pattern
```

### 10.4 图片特征

图片建议只做：

```text
conflict detector / secondary evidence
```

例如：

- 一边明显是整表，一边明显是表带 → veto；
- 一边圆形表盘，一边方形表盘 → veto；
- OCR 出现同一 canonical ref → 加强证据。

**不要允许高 image similarity 覆盖 reference 冲突。**

---

## 11. 训练数据怎么标：不要随机抽几百 pair

若随机抽 pair，99.999% 都是非常容易的负例，模型会学得毫无意义。

黄金标签必须集中在 hard cases。

建议标注结构：

```text
Positive:
- 跨源同 brand + 同 canonical ref
- 不同字段覆盖
- title 缺 ref / OCR 有 ref
- reference 格式不同但 canonical 后相同

Hard Negative:
- 同 brand + 编辑距离 1 的相邻 reference
- 同系列不同尺寸
- 同系列不同材质/颜色后缀
- 配件标题包含主体 reference
- 平台 SKU 恰好像 reference
- OCR 错一位
- title 中同时出现多个 reference
- 新旧代际高度视觉相似
```

优先把几百标签用在：

```text
决定 false positive 的边界
```

而不是易负例。

---

## 12. 对本项目改造后的 Mondrian Conformal Precision Gate

### 12.1 基础分类器

输入 pair features：

```text
x_pair
```

输出：

```text
s = P_base(match | x_pair)
```

基础模型可以先用 LightGBM：

优点：

- 小样本更稳；
- 缺失值支持好；
- 易解释；
- 训练/推理便宜；
- 便于加入大量 hard-rule features。

如果后期文本复杂，再把 cross-encoder 分数作为一个 feature 喂进去，而不是直接让大模型决定 auto merge。

---

### 12.2 Calibration

数据拆分：

```text
train_gold
calibration_gold
shadow_test_gold
```

务必按实体 / reference group 切分，而不是随机 pair 切分，避免同 reference 泄漏到 train 和 test。

Nonconformity：

```python
def nonconformity(prob, y):
    return 1.0 - prob[y]
```

---

### 12.3 推荐 Mondrian taxonomy

论文只按 predicted label 做 taxonomy。

我们的数据有明显 source domain 差异，因此推荐：

```text
(predicted_label, source_pair, evidence_tier)
```

例如：

```text
MATCH + 雷小安-腕表之家 + TIER_A
MATCH + 雷小安-奢当家   + TIER_B
MATCH + 腕表之家-奢当家 + TIER_B
```

暂时不要按 brand 继续细分，除非 calibration 样本量已经足够。

因为样本过少时，桶越细，conformal 分辨率越差。

---

### 12.4 Evidence Tier

建议明确分三档：

#### TIER_A

```text
两边都有结构化 reference 字段
+ brand parser valid
+ exact canonical reference
+ role 无冲突
```

这类可以主要靠规则。

#### TIER_B

```text
至少一边是结构化 reference
另一边来自 title/description/OCR
+ 有第二独立证据确认
```

这类进入模型 + conformal gate。

#### TIER_C

```text
两边 reference 都主要依赖弱抽取/模型猜测
```

不自动合并。

---

## 13. Batch Gate 算法

下面是推荐的生产逻辑。

```python
def process_batch(candidate_pairs, model, conformal_calibrator, policy):
    # 1. 先硬规则
    pairs = [p for p in candidate_pairs if not hard_veto(p)]

    # 2. 基础模型打分
    scored = []
    for p in pairs:
        prob = model.predict_proba(p.features)
        predicted = int(prob[1] >= 0.5)
        scored.append((p, prob, predicted))

    # 3. 只看 predicted-positive
    positives = [x for x in scored if x[2] == 1]

    # 4. 计算 Mondrian conformal p-values/confidence
    items = []
    for p, prob, _ in positives:
        cell = (
            1,
            p.source_pair,
            p.evidence_tier,
        )
        conf = conformal_calibrator.confidence(prob, cell)
        items.append((p, conf))

    # 5. 每个 cell 内按 confidence 排序
    grouped = group_by_cell(items)

    decisions = []
    for cell, xs in grouped.items():
        xs.sort(key=lambda x: x[1], reverse=True)
        n = len(xs)

        cutoff = None
        for m, (_, lam) in enumerate(xs, start=1):
            estimated_error = n * (1.0 - lam) / m
            estimated_precision = 1.0 - estimated_error

            if estimated_precision >= policy.target_precision(cell):
                cutoff = lam

        # fail closed：找不到可行 threshold，不自动放行
        if cutoff is None:
            for p, _ in xs:
                decisions.append((p, "REVIEW"))
            continue

        for p, lam in xs:
            if (
                lam >= cutoff
                and hard_positive_requirements(p)
                and not hard_veto(p)
            ):
                decisions.append((p, "AUTO_MATCH"))
            else:
                decisions.append((p, "REVIEW"))

    return decisions
```

真正实现时需要按论文的 conformal p-value 公式处理 tie/randomization，而不是只用示意代码。

---

## 14. 硬规则应该放在 Conformal 前后各一次

### 前置 veto

避免明显脏候选污染模型：

```text
brand conflict
reference canonical conflict
accessory-only
known platform sku
multiple unresolved references
reference role != PRODUCT_REFERENCE
```

### 后置 veto

即使模型 + conformal 通过，仍再做一次业务否决：

```text
cluster already contains incompatible reference
record has conflicting high-confidence evidence
source duplicate contradiction
image/text detects object-type mismatch
```

最终策略：

```text
AUTO_MATCH =
    candidate_retrieved
AND base_predicted_positive
AND conformal_gate_pass
AND hard_positive_requirements
AND no_hard_veto
```

只要任何一层不确定：

```text
ABSTAIN
```

---

## 15. “同 reference”系统最适合的自动匹配政策

建议上线初期只放下面两类。

### Rule-A：纯硬证据自动合并

```text
brand canonical exact
AND ref canonical exact
AND left evidence = structured field
AND right evidence = structured field
AND both parser valid
AND both role = PRODUCT_REFERENCE
AND no contradictory ref
```

这类甚至可以不依赖模型。

### Rule-B：一强一弱，需要 conformal gate

```text
brand canonical exact
AND ref canonical exact
AND one side structured field
AND other side title/OCR extracted
AND weak side has >= 2 independent evidence paths
AND role = PRODUCT_REFERENCE
AND conformal precision gate pass
AND no veto
```

### Rule-C：弱弱匹配

```text
永不自动合并
```

review only。

---

## 16. 图片如何真正发挥价值

图片的价值不应该是：

```text
两张图 embedding 很像 -> 自动匹配
```

腕表同系列相邻 reference 外观可能极其相似，这会制造危险 false positive。

图片应该做三件事：

### 16.1 OCR reference evidence

优先区域：

```text
表背
保卡
吊牌
证书
盒标
```

OCR token 经：

```text
brand parser
+ role classifier
+ canonicalizer
```

再进入 evidence store。

### 16.2 Object type veto

分类：

```text
watch
strap
buckle
box
papers
accessory
other
```

若 pair 两边主体类别不同：

```text
hard veto
```

### 16.3 Gross visual conflict

图像模型只检测明显矛盾：

```text
圆 / 方
表盘主色
金属 / 皮带
明显不同 bezel
```

图片低相似不能自动判 nonmatch，但可作为 review priority。

---

## 17. 聚类层不能简单“pair match 就 union”

即使 pair precision 很高，错误边一旦进入 union-find，会产生传递污染。

例如：

```text
A(ref=R1) --match--> B(ref=R1)
B          --错边--> C(ref=R2)
```

如果直接 union：

```text
A/B/C 全部一个簇
```

这是不可接受的。

推荐 cluster invariant：

```text
一个自动簇只能有一个 canonical_reference
```

伪代码：

```python
def can_union(cluster_a, cluster_b):
    refs = high_conf_refs(cluster_a) | high_conf_refs(cluster_b)

    if len(refs) > 1:
        return False

    if has_source_internal_conflict(cluster_a, cluster_b):
        return False

    return True
```

任何冲突：

```text
不 union
+ raise review alert
```

---

## 18. 增量架构

论文不是 streaming，因此建议采用“流入 + 微批决策”。

```text
Kafka / queue
   │
   ▼
Raw Record Store
   │
   ▼
Normalize + Reference Extract
   │
   ▼
Evidence Store
   │
   ▼
Candidate Index
   │
   ├── fast hard-rule path -> immediate auto-match
   │
   └── uncertain path
          │
          ▼
       micro-batch
          │
          ▼
       model score
          │
          ▼
       conformal gate
          │
          ▼
       review / auto-match
```

建议：

```text
raw ingestion      实时
reference extract  准实时
hard exact path    实时
model/conformal    15min~1h batch
recalibration      每日/每周 + 漂移触发
```

---

## 19. 推荐的存储和计算组件

技术栈不限的情况下，我会优先选择容易审计、成本低、可横向扩展的组合。

### 数据湖 / Raw

```text
S3 / OSS / MinIO + Parquet
```

### Canonical / Evidence metadata

```text
PostgreSQL
```

### 离线批处理

100 万–1000 万级并不必须上复杂大数据平台。

第一版可以：

```text
Polars / DuckDB
```

如果单机不够再上：

```text
Spark
```

### Candidate index

reference exact：

```text
PostgreSQL btree/hash
```

文本 fallback：

```text
OpenSearch
```

向量只用于 fallback candidate retrieval：

```text
FAISS / pgvector / OpenSearch vector
```

### 模型

第一版：

```text
LightGBM
```

后续：

```text
text cross-encoder score
image classifier score
OCR evidence score
```

全部作为 features，并保留可解释原始证据。

---

## 20. Calibration Artifact 必须版本化

不要只保存模型文件。

需要保存：

```yaml
model_version: matcher_v3
extractor_version: ref_extractor_v5
canonicalizer_version: ref_norm_v4
calibration_version: conf_2026_08_18

cells:
  - predicted_label: 1
    source_pair: leixiaoan_watchhome
    evidence_tier: B
    q: 214
    calibration_scores: [...]

policy:
  target_precision: 0.995
  min_calibration_size: 100
```

每一条 auto-match 结果都必须能回答：

```text
用哪个模型？
哪个 reference parser？
哪个 calibration set？
哪个 threshold？
哪些证据？
```

否则线上 false positive 发生后无法审计。

---

## 21. 几百黄金标签的最优使用方式

假设只拿到 400–600 个 pair 标签，我建议不要平均分给所有品牌。

第一批：

```text
150 hard negatives
100 positives with format variation
100 accessory/compatible traps
50 OCR/conflict cases
```

用途：

1. 训练 pair validator；
2. 验证 reference role classifier；
3. 估计 Rule-B 的可行 coverage；
4. 建立第一版 conformal calibration。

之后人工 review 的结果直接进入 active learning：

```text
最高风险 auto-match near threshold
+ 模型最不确定
+ 新品牌/new source
+ ref conflict
```

优先标。

---

## 22. 不能把人工 review 当“兜底垃圾桶”

review queue 应有明确优先级：

```text
P0: 两个高可信 reference 冲突
P1: conformal cutoff 附近
P2: accessory/compatible ambiguity
P3: OCR vs title conflict
P4: both ref missing
```

review 页面必须展示：

```text
左右原始标题
结构化 reference
抽取 span
OCR crop
canonical ref
role prediction
brand/model evidence
negative signals
model score
conformal confidence
```

让人工是在“验证证据”，而不是纯看两条商品猜。

---

## 23. 生产指标不要只看 F1

该 Spec 中 F1 不是核心指标。

上线 dashboard 至少需要：

### 23.1 Auto-match precision

最核心。

```text
目标：尽可能接近 100%
```

### 23.2 Auto-match coverage

```text
auto_matched / total_matchable
```

precision 达标后再优化 coverage。

### 23.3 Abstention rate

高不是坏事。

### 23.4 Reference extraction precision

最好分：

```text
structured
text
OCR
```

分别统计。

### 23.5 Role classification precision

尤其：

```text
PRODUCT_REFERENCE
```

不能把 accessory/compatible reference 放进自动通道。

### 23.6 Cluster conflict rate

```text
cluster 中出现 >1 个高可信 canonical_ref 的比例
```

理论上应接近 0。

### 23.7 Drift

分 source_pair/evidence_tier 看：

```text
score distribution
confidence distribution
ref parser reject rate
unknown ref pattern rate
review overturn rate
```

---

## 24. Precision Target 应该分阶段，而不是第一天写死 99.99%

因为数据不足。

建议：

### Shadow Stage

```text
不自动合并
只计算假想 AUTO_MATCH
100% 人工复核
```

### Stage 1

只开放 Rule-A。

### Stage 2

Rule-B 开放极小 coverage：

```text
只放 conformal top slice
```

### Stage 3

随着 review label 增多：

```text
扩大 calibration q
提高统计分辨率
按 evidence/source 分桶
逐步扩大 coverage
```

系统优化方向始终应该是：

```text
在 precision 不下降的前提下扩大 coverage
```

而不是：

```text
最大化总体 F1
```

---

## 25. Literal “零误匹配”无法由 ML 数学保证

需要明确：

论文里的 guarantee 有前提：

- exchangeability；
- calibration 流程正确；
- 标签正确；
- batch 设置符合方法假设。

现实数据还有：

- 数据抓取错误；
- 同平台字段语义变更；
- OCR 系统性错误；
- reference 字典错误；
- fake/replica/配件污染；
- 同一个 reference 的语义定义与业务定义不一致。

所以“绝对不能误匹配”只能转成工程策略：

```text
系统宁愿拒绝，也不猜；
自动合并只接受多层独立证据；
任何冲突 fail closed；
自动簇保持 reference invariant；
持续抽检和回滚。
```

这是比宣称“模型概率 0.9999”更可信的落地方式。

---

## 26. 推荐直接落地的 MVP

### Step 1：Canonical Brand

先建立三源品牌映射：

```text
source_brand -> canonical_brand_id
```

### Step 2：Reference Evidence Extractor

实现：

```text
structured field extractor
regex/dictionary title extractor
OCR extractor
role classifier
brand-specific parser
```

### Step 3：Exact Candidate Join

```sql
SELECT ...
FROM source_a a
JOIN source_b b
  ON a.brand_id = b.brand_id
 AND a.canonical_reference = b.canonical_reference
```

### Step 4：Rule-A 自动结果

先把最安全的双结构化 exact match 跑出来。

### Step 5：收集 hard labels

从冲突/弱证据候选中标 400–600 pair。

### Step 6：训练 Pair Validator

LightGBM 即可。

### Step 7：实现 Mondrian Conformal

cell 先只用：

```text
predicted_label + source_pair
```

样本多后再加 evidence tier。

### Step 8：微批 precision gate

每小时对 positive candidate batch 做 reject threshold。

### Step 9：人工 review 回流

每周更新：

```text
model
calibration
policy
```

### Step 10：Safe clustering

cluster union 前检查 canonical_reference invariant。

---

## 27. 推荐表结构

```sql
CREATE TABLE product_record (
    record_id           BIGINT PRIMARY KEY,
    source              TEXT NOT NULL,
    source_item_id      TEXT NOT NULL,
    brand_raw           TEXT,
    brand_id            BIGINT,
    title               TEXT,
    description         TEXT,
    structured_ref_raw  TEXT,
    raw_payload_uri     TEXT,
    ingest_time         TIMESTAMP NOT NULL
);

CREATE TABLE reference_evidence (
    evidence_id         BIGSERIAL PRIMARY KEY,
    record_id           BIGINT NOT NULL,
    raw_value           TEXT NOT NULL,
    canonical_reference TEXT,
    evidence_source     TEXT NOT NULL,
    evidence_role       TEXT NOT NULL,
    confidence          DOUBLE PRECISION,
    parser_valid        BOOLEAN NOT NULL,
    extractor_version   TEXT NOT NULL,
    provenance          JSONB NOT NULL
);

CREATE TABLE match_candidate (
    pair_id             BIGSERIAL PRIMARY KEY,
    left_record_id      BIGINT NOT NULL,
    right_record_id     BIGINT NOT NULL,
    source_pair         TEXT NOT NULL,
    evidence_tier       TEXT NOT NULL,
    features            JSONB NOT NULL,
    model_score         DOUBLE PRECISION,
    conformal_conf      DOUBLE PRECISION,
    decision            TEXT,
    decision_reason     JSONB,
    model_version       TEXT,
    calibration_version TEXT,
    created_at          TIMESTAMP NOT NULL
);

CREATE TABLE entity_cluster (
    cluster_id          BIGINT NOT NULL,
    record_id           BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    PRIMARY KEY(cluster_id, record_id)
);
```

---

## 28. 推荐 decision_reason 格式

每个 AUTO_MATCH 必须可解释。

```json
{
  "brand": {
    "left": "ROLEX",
    "right": "ROLEX",
    "equal": true
  },
  "reference": {
    "left_raw": "126610LN",
    "right_raw": "126610 LN",
    "canonical": "126610LN",
    "equal": true
  },
  "left_evidence": {
    "source": "structured_field",
    "role": "PRODUCT_REFERENCE",
    "parser_valid": true
  },
  "right_evidence": {
    "source": "title+ocr",
    "role": "PRODUCT_REFERENCE",
    "parser_valid": true
  },
  "model_score": 0.9972,
  "conformal_confidence": 0.9961,
  "policy": "RULE_B",
  "vetoes": [],
  "cluster_invariant_pass": true
}
```

不要只存：

```json
{"score": 0.9972}
```

---

## 29. 与 Spec 的一一对应

| Spec 约束 | 本方案应对 |
|---|---|
| 同一商品 = 同一 reference | reference evidence/canonicalization 作为主轴 |
| 字段稀疏 | 结构化字段 + title + description + OCR 多路证据 |
| 绝不能误匹配 | reject option + hard veto + fail closed |
| 可漏匹配 | TIER_C 永不自动合并 |
| 有图片 | OCR 与冲突否决，不让视觉相似越权 |
| 100万–1000万 | reference hash join + blocking，避免全量 pairwise |
| 持续增量 | 流式 ingest + 微批 conformal gate |
| 几百黄金标签 | hard-case sampling + validator + 初始 calibration |
| 三来源差异 | source_pair Mondrian taxonomy |

---

## 30. 最终建议

这篇论文最值得借鉴的并不是“Conformal Prediction”这个名词，而是一个架构原则：

> **把“模型能否判断”与“系统是否应该自动行动”分开。**

对当前跨源二奢/腕表实体匹配需求，我建议最终生产判定链路是：

```text
可信 brand
  AND
可信 PRODUCT_REFERENCE evidence
  AND
canonical reference exact equality
  AND
候选验证器预测 match
  AND
Mondrian conformal precision gate 通过
  AND
没有任何 hard conflict
  AND
cluster reference invariant 通过
  => AUTO_MATCH

否则
  => ABSTAIN / REVIEW
```

其中最重要的两点是：

1. **Conformal gate 不应该取代 reference exact rule，而应该保护模型辅助路径；**
2. **几百标签不足以“证明”超高 precision，必须用保守规则降低问题难度，再随着人工回流逐步扩大可自动化的 coverage。**

如果按这一方案落地，第一版完全不需要复杂大模型：

```text
brand/ref parser
+ OCR
+ LightGBM pair validator
+ Mondrian conformal calibrator
+ review queue
+ safe union-find
```

就可以搭出一个可审计、fail-closed、适合百万到千万级增量数据的 precision-first 实体匹配系统。