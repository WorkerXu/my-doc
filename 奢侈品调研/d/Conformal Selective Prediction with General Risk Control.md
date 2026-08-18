# Conformal Selective Prediction with General Risk Control：面向跨源二奢/腕表实体匹配的 SCoRE 风险控制层与可落地架构

> 分析对象：Tian Bai, Ying Jin, **Conformal Selective Prediction with General Risk Control**（arXiv:2603.24704, 2026）  
> 论文：https://arxiv.org/abs/2603.24704  
> HTML：https://arxiv.org/html/2603.24704  
> 官方代码：https://github.com/Tian-Bai/SCoRE  
> Python 包：`score-select`  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”定义为**同一 reference number / 型号**；字段稀疏，reference 可能埋在标题；图片可用；可人工标注几百对；**precision 极端优先，允许大量拒识/漏匹配**。

---

## 1. 结论先行

这篇论文最适合当前需求的地方，不是再提供一个更强的 entity matching 模型，而是提供一层**独立于基础模型的“是否允许自动执行”风险控制器**。

对本 Spec，推荐直接落成下面的结构：

```text
原始商品记录
  ↓
品牌归一化
  ↓
reference 候选抽取 + 编号角色识别
  ↓
canonical reference 归一化
  ↓
强规则否决
  ↓
查询 canonical entity: (brand_id, canonical_reference)
  ↓
生成「record → entity」加入候选
  ↓
风险模型给出 score（越小越安全）
  ↓
SCoRE-SDR 批量选择
  ├── selected  → AUTO_ATTACH / AUTO_MATCH
  └── rejected  → ABSTAIN / HUMAN_REVIEW
```

这里最关键的架构调整是：

> **不要把主问题建模成 N×N 的 record-pair matching，而要建模成“每条记录能否可信地声明自己的 canonical reference，并加入这个 reference 对应的实体簇”。**

因为 Spec 已经把“同一个商品”定义成同一 reference，所以真正要解决的是：

1. 记录里的字符串是不是品牌 reference，而不是平台 SKU、店铺货号、配件适配号；
2. reference 是否抽对、归一化对；
3. 在证据不够硬时是否应该拒识。

SCoRE 正好适合第 3 步：它允许任意风险排序器先给候选排序，再使用单独的 calibration 标签决定**这一批里哪些候选能在目标风险下被自动执行**。

对于本需求，应优先采用论文的 **SDR（Selective Deployment Risk）**，而不是 MDR：

```text
L_i = 1{一次自动加入实体簇是错误的}
```

此时：

```text
SDR ≈ 自动执行集合中的错误加入比例
    ≈ 1 - precision
```

所以在理想假设成立时，把 `alpha` 设为 `0.001`，其业务含义接近：

```text
自动执行集合的期望错误比例 <= 0.1%
期望 precision >= 99.9%
```

但必须强调：

> **SCoRE 提供的是统计风险控制，不是“单条永不出错”的逻辑证明。**

因此它应该是 reference 硬规则后的**保险丝**，不能替代 reference 身份定义本身。

---

## 2. 为什么这篇比“再做一个 matcher”更值得落地

`d` 目录已经分析过 DeepBlocker、Ameli、pyJedAI、TransClean、Confidence Classifier、LLM 属性抽取等方案。SCoRE 与这些工作的关系不是重复，而是向外再包一层：

```text
DeepBlocker / reference index
    解决：候选从哪里来

LLM / OCR / attribute extractor
    解决：reference 怎么抽

matcher / CatBoost / multimodal model
    解决：候选看起来有多可信

SCoRE
    解决：在给定风险预算下，哪些结果“允许自动执行”
```

它和之前的 `Confidence Classifiers with Guaranteed Accuracy or Precision` 也有明显区别：

- Confidence Classifier 更像“用 conformal confidence 排序后做 reject，并估计 selective precision”；
- SCoRE 直接把部署动作写成风险控制问题，构造 **risk-adjusted e-value**；
- 对批量测试，SCoRE-SDR 配合 **e-BH** 直接控制 selected 集合的平均风险；
- 风险不仅能是 0/1，还可以是 `[0,1]` 连续损失；
- 论文和官方实现都给出了 **covariate shift 加权版本**；
- 官方包已经包含 `SCoRE_MDR`、`SCoRE_SDR`、`SCoRE_MDR_w`、`SCoRE_SDR_w`，可以直接做工程验证。

因此更适合成为当前系统的最终“自动合并闸门”。

---

## 3. 论文核心：把“信不信模型”变成带风险预算的选择问题

### 3.1 基本设定

论文假设有：

```text
calibration:
  (X_i, Y_i), i=1...n

test:
  X_{n+j}, j=1...m
```

基础模型 `f(X)` 已经训练好。

对每条样本定义一个实际部署损失：

```text
L = Loss(f, X, Y) ∈ [0,1]
```

同时有一个固定的 score：

```text
s(X)
```

约定：

```text
score 越小 → 预估风险越小 → 越应该优先自动执行
```

论文特别重要的一点是：

> **风险控制的有效性不要求 score 一定非常准。**

score 越准，只会提高 selection power / coverage；在 exchangeability 条件下，风险控制来自 calibration + conformal/e-value 构造，而不是“模型概率本身很可信”。

这非常适合当前只有少量标注的情况：第一版甚至可以用规则型 score，后续再逐步替换成 CatBoost / LightGBM 风险模型。

### 3.2 两种风险：MDR 与 SDR

论文定义两种部署风险。

#### MDR：Marginal Deployment Risk

可以理解成：

```text
总体候选空间里，自动执行带来的期望风险
```

其形式为：

```text
E[L * ψ]
```

其中 `ψ=1` 表示自动执行。

它适合“总风险预算”问题，但不一定保证 selected 集合本身非常纯。

例如系统只执行 1% 的候选，即使那 1% 里面错误率偏高，总体 MDR 仍可能很低。

所以本 Spec 不应把 MDR 作为主 KPI。

#### SDR：Selective Deployment Risk

SDR 关注：

```text
自动执行的那一批结果里，平均风险是多少
```

对于二元错误：

```text
L = 1{wrong auto-match}
```

SDR 就退化成类似 FDR 的量：

```text
wrong_selected / selected
```

这和业务的 precision-first 完全同构。

结论：

> **生产主闸门应该用 SCoRE-SDR。MDR 可作为“全局总错误预算”辅助指标，但不要代替 SDR。**

---

## 4. risk-adjusted e-value：为什么它能做“自动执行闸门”

论文定义 risk-adjusted e-value：

```text
E_j >= 0
E[L_j * E_j] <= 1
```

直觉上：

```text
E_j 越大
→ 越有证据说明这个 test item 的真实风险很小
```

对于单条 MDR，论文给出的决策形式是：

```text
如果 E_j >= 1 / alpha
→ deploy
否则
→ abstain
```

因为：

```text
1{E >= 1/alpha} <= alpha * E
```

于是可把部署风险压到目标 `alpha` 以下。

对一批 test item，SCoRE-SDR 不再逐条用固定阈值，而是把所有 e-values 交给 `e-BH`：

```text
E_1, E_2, ... E_m
      ↓
     e-BH
      ↓
selected set R
```

选择规则本质上是：

```text
E_j >= m / (alpha * k)
```

其中 `k` 是最终被选中的数量。

因此 selection threshold 会随着整批候选的质量和数量自适应，而不是手工拍一个：

```text
match_probability > 0.999
```

这也是本方案相比普通 calibrated probability 最大的价值之一。

---

## 5. SCoRE-SDR 的技术过程

论文与官方代码都围绕同一个结构：

### Step 1：准备 calibration loss

对 calibration 中每个“自动加入实体簇”候选，保存：

```text
Lcalib[i] = 0   # 加入目标 canonical entity 是正确的
Lcalib[i] = 1   # 加入目标 canonical entity 是错误的
```

第一版就用二元 loss，不要一开始设计复杂 continuous loss。

原因：当前业务真正要控制的是 false positive，二元 loss 最容易解释成 precision。

### Step 2：准备 calibration/test risk score

```text
Scalib[i] = risk_score(calib_candidate_i)
Stest[j]  = risk_score(batch_candidate_j)
```

score 越小表示越安全。

### Step 3：用 calibration risk 在 score threshold 上估计 selected 风险

SCoRE 会在候选 score 阈值上寻找满足风险约束的区域。

SDR 版本的核心是估计：

```text
当我只选 score <= t 的 test 候选时，
对应 calibration 中的损失能否支持这个 selected 集合？
```

论文把这个构造成 risk-adjusted e-value，并用 e-BH 选出最终集合。

### Step 4：只对 selected 执行

```text
selected_index = SCoRE_SDR(...)
```

只有 selected 才允许：

```text
entity_membership.status = AUTO_APPROVED
```

其他全部：

```text
ABSTAIN
```

注意：

> **未被 selected 不是 NON_MATCH。**

它只表示“证据不足以自动执行”。

这与当前允许漏匹配的约束完全一致。

---

## 6. 官方代码实现值得直接复用的细节

官方仓库：

```text
Tian-Bai/SCoRE
```

核心包非常小：

```text
SCoRE/
  SCoRE.py
  utility.py
  __init__.py
```

README 给出的推荐入口：

```python
from SCoRE import SCoRE_MDR, SCoRE_SDR
```

安装：

```bash
python -m pip install score-select
```

### 6.1 `SCoRE_SDR` 的输入接口非常适合做独立 batch service

官方 API 形态：

```python
selected = SCoRE_SDR(
    Dcalib=(Lcalib, Scalib),
    Dtest=Stest,
    alpha=alpha,
    gamma=gamma,
    prune=None,
)
```

这意味着它完全不关心 matcher 是什么模型。

生产上可以把它做成非常薄的一层：

```text
RiskScorer
  输出 score

SCoREGate
  只读取：
  - calibration loss
  - calibration score
  - current batch score
  - alpha / gamma

EntityWriter
  只写 selected
```

模型、OCR、LLM、数据库都不需要侵入 SCoRE 本身。

### 6.2 官方 SDR 实现并不是暴力连续搜索

论文给出的高效算法复杂度：

```text
O(m(n+m) + (n+m) log(n+m))
```

官方实现的关键工程手法是：

1. 把 calibration/test score 合并并排序；
2. 建 `NUMER` prefix sum，表示阈值 `t` 以下的 calibration risk 累计；
3. 建 `DENOM` prefix sum，表示阈值 `t` 以下 test 数量；
4. 显式处理 score ties，保证相同阈值取到一致的累计值；
5. 对每个 test candidate 计算 `FR_0`、`FR_1`、`ELL`；
6. 把连续 `ell ∈ [0,1]` 搜索化简到有限的 score threshold 集合；
7. 计算 e-values 后统一交给 `eBH`。

这个实现对当前项目有两个启示：

- **不要给每条候选单独调用一个远程“conformal API”**，而应该按 batch 一次性运行；
- risk score 最好是稳定的标量，并保证 tie 处理一致。

### 6.3 官方 `eBH` 非常简单

`utility.py` 中的实现本质上是：

```text
p_like = 1 / evalue
再对 p_like 跑 BH
```

所以生产系统不需要单独引入复杂统计框架，直接调用官方包即可。

### 6.4 第一版不要打开随机 boosting

官方 `SCoRE_SDR` 支持：

```text
prune = None
prune = "hete"
prune = "homo"
```

后两者通过随机 Uniform 因子放大 e-values 来提高 power，同时论文证明仍可保持 SDR control。

但对当前“所有自动合并必须可审计、可复现”的系统，第一版建议：

```python
prune=None
```

原因：

- 同一数据在不同随机 seed 下 selection 可能变化；
- 会增加事故复盘难度；
- 当前目标不是最大 coverage，而是最小 false positive。

等系统稳定后可以 shadow test `homo`，不建议一开始进入生产自动写路径。

---

## 7. 对当前 Spec 最重要的建模调整：不要做 pair matching，做 cluster admission

如果有：

```text
N = 1,000,000 ~ 10,000,000
```

任何试图构造大规模 pair 的系统都会把主要资源浪费在候选生成上。

但本 Spec 的实体定义已经非常特殊：

```text
same entity ⇔ same canonical reference
```

因此应建立：

```text
canonical_entity
  key = (brand_id, canonical_reference)
```

例如：

```text
(ROLEX, 126610LN)
(ROLEX, 126610LV)
(OMEGA, 310.30.42.50.01.001)
```

一条新记录进入后，不应该去找“最像的旧记录”，而应该：

```text
record
  ↓
extract reference claim
  ↓
canonicalize reference
  ↓
lookup (brand_id, ref)
  ↓
找到唯一 canonical entity
  ↓
决定：是否允许 record 加入这个 entity
```

这样每条记录只需要一次索引 lookup。

### 7.1 为什么这比 record-pair SCoRE 更合理

如果一个 reference cluster 里已有 500 条记录，新记录进入时，没有必要生成：

```text
500 个 pair
```

只需要一个动作：

```text
record → entity_cluster
```

其 label 是：

```text
这个 record 是否真的属于该 reference entity？
```

这直接把复杂度从潜在 pair explosion 降成 O(N) 级实体归属。

更重要的是，审计也更简单：

```text
为什么 record 123 被挂到了 ROLEX:126610LN？
```

只需要查看一条 membership decision，而不是追踪数百条 pair edge 的传递闭包。

---

## 8. 推荐生产架构

```text
                 ┌──────────────────────────────┐
                 │   雷小安 / 腕表之家 / 奢当家   │
                 └──────────────┬───────────────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │ Raw Record Store     │
                    │ 原始字段/图片/抓取版本 │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Brand Normalizer     │
                    └──────────┬───────────┘
                               │
                               ▼
             ┌──────────────────────────────────┐
             │ Reference Evidence Extractor     │
             │ structured/title/desc/OCR/LLM    │
             └────────────────┬─────────────────┘
                              │
                              ▼
             ┌──────────────────────────────────┐
             │ Reference Role Classifier        │
             │ brand_ref / sku / store_id /     │
             │ compatible_ref / accessory_ref   │
             └────────────────┬─────────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ Canonicalizer          │
                  │ 品牌级 reference 归一化 │
                  └───────────┬────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Hard Safety Gate    │
                    └───────┬─────────────┘
                            │ pass
                            ▼
               ┌──────────────────────────┐
               │ Canonical Entity Index   │
               │ (brand_id, ref) -> id    │
               └───────────┬──────────────┘
                           │
                           ▼
               ┌──────────────────────────┐
               │ Admission Candidate      │
               │ record -> entity         │
               └───────────┬──────────────┘
                           │
                           ▼
               ┌──────────────────────────┐
               │ Risk Scorer              │
               │ s(x): smaller = safer    │
               └───────────┬──────────────┘
                           │ micro-batch
                           ▼
               ┌──────────────────────────┐
               │ SCoRE-SDR Gate           │
               │ calibration + e-values   │
               │ + e-BH                   │
               └───────────┬──────────────┘
                    selected│      │rejected
                            ▼      ▼
                    AUTO_ATTACH   ABSTAIN
                            │      │
                            ▼      ▼
                 Entity Membership  Review Queue
```

图片在这里的地位是：

```text
辅助 reference OCR
辅助识别当前商品/配件
辅助冲突否决
```

而不是：

```text
图片看起来像 → 直接判同 reference
```

---

## 9. Hard Safety Gate：必须先于 SCoRE

SCoRE 不是用来修复明显错误候选的。

下面这些条件建议直接 `ABSTAIN`，不要让风险模型尝试“学过去”。

### 9.1 品牌冲突

```text
brand_id_a != brand_id_entity
→ reject
```

### 9.2 reference 不唯一

标题里同时出现：

```text
126610LN
126610LV
```

但无法确定哪一个描述当前售卖商品：

```text
→ reject
```

### 9.3 reference role 不是品牌 reference

如果分类为：

```text
platform_sku
seller_sku
inventory_id
compatible_model
accessory_target_ref
```

则：

```text
→ reject
```

### 9.4 明确的 reference 冲突

例如：

```text
structured field = 126610LN
OCR/back-card     = 126610LV
```

不能让模型平均掉冲突：

```text
→ reject
```

### 9.5 商品类型冲突

例如标题说明是：

```text
表带 / 表盒 / 保卡 / 配件 / 镜面 / 表扣
```

但其中出现某款腕表 reference：

```text
→ 不得直接加入腕表 entity
```

### 9.6 canonicalization 高风险变换

可以接受：

```text
大小写
Unicode 全半角
确定性的空格/连字符格式
品牌官方已知格式
```

不建议自动接受：

```text
删除任意字母
模糊编辑距离
相邻数字纠错
LLM 猜测缺失位
```

因为：

```text
126610LN ≠ 126610LV
```

即使只差两个字符，也必须保持独立实体。

---

## 10. Risk Scorer 应该预测什么

SCoRE 的 score 不应再预测“两个商品像不像”。

应该预测：

> **如果现在把这条 record 自动加入它声明的 `(brand, reference)` entity，发生错误归属的风险有多大？**

建议特征：

```text
# reference evidence
ref_from_structured_field
ref_from_title
ref_from_description
ref_from_ocr
ref_evidence_count
ref_occurrence_count
ref_candidate_count

# role / ownership
role_brand_reference_prob
role_platform_sku_prob
role_compatible_model_prob
current_product_ownership_score

# agreement
structured_title_agree
structured_ocr_agree
title_ocr_agree
all_evidence_agree

# ambiguity
multiple_ref_in_title
multiple_ref_in_page
accessory_keyword_score
watch_product_type_score

# canonicalization
normalization_operation_count
normalization_risk_level
brand_pattern_valid

# source
source_id
source_pair
crawler_template_version
extractor_version

# entity context
entity_size
reference_frequency
entity_existing_sources
same_source_duplicate_count
```

### 10.1 第一版不一定要训练模型

由于 SCoRE validity 不要求 score 本身完美，初期可以用纯规则 risk score：

```text
0.01 structured ref + title ref + OCR 全一致
0.05 structured ref + title 一致
0.15 title 独立抽取且上下文明确
0.30 title 抽取但存在多个编号
0.60 OCR only
0.90 有角色/配件风险
```

只要这个 score **没有使用 calibration label 训练**，就能作为固定排序函数进入 calibration。

等标签增加后再替换为：

```text
CatBoost / LightGBM
```

目标不是概率 calibration，而是把“更可能出错”的 candidate 排到后面。

---

## 11. calibration 数据必须与训练数据彻底隔离

论文要求 score/model 与 calibration/test 独立训练，或者至少使最终 `(score, prediction, Y)` 仍满足交换性条件。

工程上直接做三份：

```text
train_label_set
  用于训练 reference role / risk scorer

calibration_label_set
  永不用于训练
  只用于 SCoRE

validation/audit_set
  只用于独立验收
```

不能：

```text
300 条标签
→ 全拿去训练 risk model
→ 再用同 300 条做 SCoRE calibration
```

这样会让 score 对 calibration 过拟合，破坏方法前提。

### 11.1 Active Learning 标签不能直接当普通 calibration

为了训练 matcher，最值得标的是 hard case：

```text
模型最不确定的样本
```

但这种样本并不是线上候选分布的随机样本。

如果直接把它们当 calibration：

```text
exchangeability 会明显变差
```

建议：

```text
主动学习池 → train
随机/分层随机抽样池 → calibration
```

calibration 至少要保持和真实 admission candidate 分布一致，或者后续使用明确的 covariate-shift weighting。

---

## 12. 生产调用示例

第一版可以直接调用官方包：

```python
import numpy as np
from SCoRE import SCoRE_SDR


def score_batch(calib_rows, batch_rows, risk_model, alpha):
    # calibration labels: 1 = wrong admission, 0 = correct admission
    lcalib = np.asarray([r.loss for r in calib_rows], dtype=float)

    # score model must be fixed independently from calibration labels
    scalib = np.asarray([
        risk_model.score(r.features) for r in calib_rows
    ], dtype=float)

    stest = np.asarray([
        risk_model.score(r.features) for r in batch_rows
    ], dtype=float)

    selected = SCoRE_SDR(
        (lcalib, scalib),
        stest,
        alpha=alpha,
        gamma=alpha,
        prune=None,
        return_evals=False,
    )

    selected_ids = {batch_rows[i].candidate_id for i in selected}

    return selected_ids
```

执行逻辑：

```python
selected = score_batch(...)

for candidate in batch:
    if candidate.id in selected:
        attach_record_to_entity(candidate)
    else:
        abstain(candidate)
```

必须保存：

```text
batch_id
candidate_id
score
alpha
gamma
calibration_version
risk_model_version
extractor_version
canonicalizer_version
selected / abstained
```

这样任何自动归属都能复盘。

---

## 13. 持续增量应该 micro-batch，而不是逐条实时 SCoRE

SCoRE-SDR 是批量选择问题。

当前抓取数据完全没必要为了毫秒延迟牺牲风险控制。

推荐：

```text
ingestion stream
  ↓
staging
  ↓
每 5/15/30 分钟形成 batch
  ↓
Hard Gate
  ↓
Risk scoring
  ↓
SCoRE-SDR
  ↓
commit selected memberships
```

不同风险分布建议不要随意混 batch。

第一版可以按以下维度粗分：

```text
source_pair
  雷小安 ↔ 腕表之家
  雷小安 ↔ 奢当家
  腕表之家 ↔ 奢当家
```

标签足够后再增加：

```text
evidence_tier
  structured
  title_extract
  OCR_assisted
```

但不要分得过细，否则每个 bucket 的 calibration 样本太少，SCoRE 会没有 selection power。

---

## 14. “只有几百条黄金标签”意味着什么

这是本方案必须提前说清楚的现实限制。

### 14.1 极低 alpha 需要足够 calibration 才可能有 coverage

在 binary loss 的简单情形下，SCoRE 与 conformal selection 有紧密联系。

从官方实现也可以看到，当低风险区域 calibration error 很少时，e-value 的量级仍大体受 `n+1` 限制。

而 e-BH 最宽松的有效阈值量级约是：

```text
1 / alpha
```

因此一个非常实用的粗略判断是：

```text
想让 alpha = 0.001 有非零自动放行能力
通常至少要有 ~1000 量级 calibration

想让 alpha = 0.0001
通常需要 ~10000 量级 calibration 才有现实空间
```

这不是说 `n >= 1/alpha` 就自动获得对应 precision，而是说：

> **低于这个量级时，连统计闸门“有能力放行”的空间都可能不足。**

### 14.2 几百标签的正确用途

几百条可以做：

```text
reference role 规则验证
hard negative 模式发现
第一版 risk ranker
shadow calibration
流程打通
```

但如果产品方真的要求：

```text
自动 precision >= 99.9% / 99.99%
```

正确策略不是把阈值硬写成 `0.9999`，而是：

```text
标签不够
→ AUTO_MATCH 关闭或只开放极窄 Tier-0
→ 大部分全部人工/拒识
→ 持续随机抽检积累 calibration
→ 达到足够样本后再逐步打开 coverage
```

这和“允许漏匹配”的业务约束并不冲突。

---

## 15. 推荐的 evidence tier 与灰度策略

### Tier 0：最强证据

```text
品牌明确
structured reference 存在
reference 格式满足品牌 pattern
标题中同 reference 再次出现
无第二 reference
非配件
无 OCR/描述冲突
```

风险 score 最低。

### Tier 1：中强证据

```text
品牌明确
没有 structured reference
标题可抽出唯一 reference
上下文明确属于当前商品
描述/OCR 至少一个独立证据一致
```

### Tier 2：弱证据

```text
OCR only
LLM 推断
标题多 reference
存在 compatible / 配件语义
```

建议：

```text
Tier 2 永远不 AUTO_MATCH
```

至少在足够大规模验证之前如此。

SCoRE 不是为了把 Tier 2 “洗白”，而是为了从 Tier 0/1 中再做最后一次风险筛选。

---

## 16. reference 缺失时不要让视觉模型越权

如果某条记录完全没有可靠 reference：

```text
不要因为图片很像就 AUTO_MATCH
```

正确做法：

```text
图片 embedding / OCR / 文本 ANN
    ↓
召回可能 reference
    ↓
进入 HUMAN_REVIEW
```

这些能力可以提高人工审核效率，也可以产生新的训练标签，但不能改变身份定义。

对于 current Spec：

```text
reference 不足
→ ABSTAIN
```

是正确答案，不是系统失败。

---

## 17. 分布漂移：SCoRE 的 weighted 版本怎么用

三个来源会持续变化：

```text
新抓取模板
新品牌
标题格式变化
OCR 模型升级
LLM extractor 升级
来源占比变化
```

普通 SCoRE 依赖 calibration/test exchangeability。

论文给出了 covariate shift 扩展：

```text
w(x) = dQ(x) / dP(x)
```

官方代码对应：

```python
SCoRE_MDR_w(...)
SCoRE_SDR_w(...)
```

输入：

```text
wcalib
wtest
```

### 17.1 第一版不要把 weighted SCoRE 当作 drift 万能药

论文在已知 density ratio 时能给出有限样本性质；当权重需要估计时，保证会变成渐近性质并依赖估计质量。

所以生产优先级应该是：

```text
1. drift detection
2. bucket shutdown
3. 新增随机 calibration 标签
4. 重新校准
5. 最后才考虑 weighted SCoRE
```

### 17.2 推荐 drift guard

至少监控：

```text
reference extraction method distribution
ref pattern distribution
source mix
brand mix
score distribution
abstain rate
AUTO_MATCH rate
calibration error by score band
new crawler template ratio
new extractor version ratio
```

出现显著 drift：

```text
该 segment AUTO_MATCH = OFF
```

直到重新校准。

---

## 18. 数据模型建议

### `raw_product_record`

```text
record_id
source
source_product_id
raw_title
raw_fields
image_urls
crawl_time
crawler_version
```

### `reference_claim`

```text
record_id
raw_reference
canonical_reference
brand_id
extraction_channel
context
role
role_score
canonicalization_ops
extractor_version
```

### `canonical_entity`

```text
entity_id
brand_id
canonical_reference
status
created_at
```

唯一键：

```text
UNIQUE(brand_id, canonical_reference)
```

### `entity_admission_candidate`

```text
candidate_id
record_id
entity_id
feature_json
risk_score
hard_gate_status
batch_id
```

### `score_calibration_sample`

```text
sample_id
candidate_snapshot
human_label
loss
calibration_partition
feature_version
```

### `entity_membership`

```text
record_id
entity_id
decision_type       # AUTO / HUMAN
decision_batch_id
score
alpha
gamma
calibration_version
model_version
created_at
active
```

不要物理删除或覆盖原始记录。

错误发现后应该可以：

```text
active = false
```

快速回滚 membership。

---

## 19. 一个更安全的完整决策状态机

不要只用：

```text
MATCH / NON_MATCH
```

推荐：

```text
NO_REFERENCE
AMBIGUOUS_REFERENCE
REFERENCE_ROLE_REJECT
REFERENCE_CONFLICT
HARD_GATE_PASS
SCORE_REJECT
SCORE_SELECTED
AUTO_ATTACHED
HUMAN_APPROVED
HUMAN_REJECTED
QUARANTINED
```

其中真正会改变 entity membership 的只有：

```text
AUTO_ATTACHED
HUMAN_APPROVED
```

其他都是可解释的拒识状态。

---

## 20. 生产指标不要再以 F1 为中心

### 第一优先级

```text
Auto-Match Precision
False Positive Count
SDR / empirical FDP
```

### 第二优先级

```text
Auto-Match Coverage
Abstain Rate
Human Review Rate
```

### 诊断指标

```text
precision by source_pair
precision by brand
precision by evidence_tier
precision by extractor_version
precision by score_decile
precision by entity_size
```

### 绝对要监控

```text
hard conflict escaped count
reference-role false positive count
cluster contamination incident count
rollback count
```

对于这个业务：

```text
coverage 从 20% 提升到 40%
```

远没有：

```text
FP 从 2 降到 0
```

重要。

---

## 21. 推荐 rollout 路线

### Phase 0：先只做 reference identity layer

目标：

```text
不做任何模型自动合并
```

实现：

```text
brand normalization
reference extraction
reference role classification
canonicalization
canonical entity index
```

所有 candidate 进入人工审核。

### Phase 1：SCoRE shadow mode

收集：

```text
candidate
score
SCoRE would_select
human label
```

但不自动执行。

验证：

```text
selected precision
coverage
各 source/evidence bucket 风险
```

### Phase 2：只打开 Tier 0

条件：

```text
calibration 足够
独立 audit 无明显 FP
无 drift
```

且设置极低 alpha。

### Phase 3：逐步开放 Tier 1

必须独立看：

```text
structured→title
title→title
OCR-assisted
```

不要共享一个乐观阈值。

### Phase 4：增量校准与 drift 自动熔断

```text
定期随机抽检 selected + abstained
更新 calibration
版本化 score / extractor
发生 drift 自动关闭 segment
```

---

## 22. 可以直接落地的最小 MVP

如果现在就实现，推荐只做下面 6 个组件。

### 1. `reference_extractor`

输出：

```json
{
  "brand_id": "ROLEX",
  "candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "channel": "title",
      "role": "brand_reference"
    }
  ]
}
```

### 2. `reference_hard_gate`

返回：

```text
PASS / ABSTAIN + reason_code
```

### 3. `canonical_entity_index`

```text
(brand_id, canonical_reference) -> entity_id
```

### 4. `risk_scorer`

第一版可用规则函数。

接口：

```python
score = scorer(features)  # lower = safer
```

### 5. `score_gate_worker`

每个 micro-batch：

```python
selected = SCoRE_SDR(
    (Lcalib, Scalib),
    Stest,
    alpha=policy.alpha,
    gamma=policy.alpha,
    prune=None,
)
```

### 6. `entity_membership_writer`

只写 selected，并保存完整 decision snapshot。

这 6 个组件就能把论文方法真正变成一个可上线的 precision-first matching system。

---

## 23. 论文方法的边界与本项目必须额外加的防护

### 23.1 SDR control 不是单条 guarantee

不能对业务说：

```text
SCoRE 选中的每条都保证正确
```

只能说在方法假设成立时，对 selected 集合的期望风险提供控制。

### 23.2 exchangeability 是最大风险点

真实线上：

```text
品牌、来源、抓取器、extractor 都会变
```

所以必须有 version/bucket/drift guard。

### 23.3 calibration 少时可能什么都不选

这不是算法坏了，而是 ultra-high precision 的正常代价。

当前业务明确允许漏匹配，因此：

```text
0 auto-match
```

在证据不足时甚至是正确生产行为。

### 23.4 不能把 calibration labels 偷回训练

要严格冻结 calibration partition。

### 23.5 不要因为模型变强就删除 hard rule

即使未来换成更强 multimodal LLM：

```text
reference conflict
brand conflict
role conflict
```

仍应 hard reject。

---

## 24. 最终推荐方案

针对 Notion Spec，我推荐把整体系统定义成：

> **Reference-Centric Entity Resolution + Hard Safety Rules + Risk Ranking + SCoRE-SDR Selective Admission**

核心原则：

```text
1. identity 由 canonical reference 定义，而不是由模型相似度定义
2. 模型只负责判断 reference claim / cluster admission 有多危险
3. SCoRE 只负责决定哪些低风险 admission 允许自动执行
4. 所有未通过都 ABSTAIN，不强行 NON_MATCH
5. 图片只提供 evidence，不覆盖 reference 硬事实
6. calibration 与训练严格隔离
7. 标签不够时宁可 0 coverage
8. 持续增量必须 micro-batch + drift guard
9. 所有自动 membership 可追溯、可撤销
```

一句话总结：

> **当前系统真正要优化的不是“匹配率”，而是“在不确定时敢于不匹配”。SCoRE 的价值，就是把这种拒识能力从经验阈值提升为一个可校准、可批量控制、可在分布漂移下扩展的统计决策层。**

这篇论文最适合直接落在最终自动合并闸门上，而不是取代 reference 抽取、编号角色识别和 canonicalization。