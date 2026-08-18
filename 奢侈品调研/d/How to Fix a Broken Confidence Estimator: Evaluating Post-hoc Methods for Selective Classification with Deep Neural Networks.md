# How to Fix a Broken Confidence Estimator: Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks：面向跨源二奢/腕表 reference 实体链接的置信度修复层与高精度拒识方案

> 分析对象：Luís Felipe P. Cattelan, Danilo Silva, **How to Fix a Broken Confidence Estimator: Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks**（UAI 2024 / PMLR 244）  
> 论文主页：https://proceedings.mlr.press/v244/cattelan24a.html  
> arXiv：https://arxiv.org/abs/2305.15508  
> 官方代码：https://github.com/lfpc/FixSelectiveClassification  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”定义为同一 reference number / 型号；字段高度稀疏；reference 可能埋在标题或图片；可人工标注几百对；**precision 优先到极致，允许大量 abstain / 漏匹配**。

---

## 1. 为什么选这篇，以及它补的是哪一块

`d` 目录里已经分析过：

- `Confidence Classifiers with Guaranteed Accuracy or Precision`
- `Conformal Selective Prediction with General Risk Control`
- `TransClean`
- `GraLMatch`
- `DeepBlocker`
- `Ameli`
- 多模态属性抽取、LLM 属性规范化等方案

这些方案已经覆盖了“候选生成”“reference 抽取”“图一致性清洗”“风险保证”“多模态辅助”等部分。

这篇论文补的是一个非常容易被忽略、但对 production precision 很关键的中间层：

> **基础模型本身可能判断得不错，但它输出的 confidence 排序是坏的。**

也就是说，模型 accuracy 很高，并不意味着“分数最高的那些预测就是最不容易错的”。如果我们直接拿：

```text
softmax > 0.99
```

作为自动合并阈值，那么即使基础模型整体准确，也可能把最危险的 false positive 放进“最高置信区间”。

论文研究的不是重新训练一个更大的模型，而是：

```text
已有分类器 logits
    ↓
极轻量 post-hoc 变换
    ↓
更可靠的 confidence ranking
    ↓
按阈值只接受一小部分预测
```

这与当前需求非常契合，因为当前业务并不追求“每条都给答案”，而是：

```text
AUTO_LINK / AUTO_MATCH   —— 只接受极少量最安全样本
ABSTAIN                  —— 其余全部不自动合并
```

但本文最重要的落地结论不是“直接在 pairwise match 模型后加 pNorm”，而是：

> **应把二奢匹配主问题从 pairwise 二分类，重构成“商品记录 → canonical reference entity”的多候选实体链接；MaxLogit-pNorm 放在多候选 reference disambiguation 后面，作为置信度排序修复层。**

原因见第 5 节：对普通二分类 logits，论文的中心化 MaxLogit-pNorm 在数学上会严重退化。

---

## 2. 论文解决的问题：Selective Classification，而不是概率校准

论文把模型写成：

```text
h(x) = argmax_k z_k
```

其中 `z` 是网络输出的 logits。

Selective Classification 在普通分类器上再增加一个 confidence estimator：

```text
(h, g)
```

其中：

```text
g(x) >= threshold  -> 接受预测
g(x) <  threshold  -> reject / abstain
```

于是系统不再只有“预测对/预测错”，而是产生一个业务更有意义的 trade-off：

```text
coverage ↑  -> 自动处理更多，但风险变大
coverage ↓  -> 自动处理更少，但风险可下降
```

论文用 Risk-Coverage Curve（RC curve）描述这个关系，并用 AURC（Area Under Risk-Coverage Curve）衡量 confidence 排序质量。

对当前业务，risk 可以直接定义为：

```text
risk_i = 1[predicted_reference_i != gold_reference_i]
```

coverage 则是：

```text
被系统自动绑定到 canonical reference entity 的商品比例
```

因此我们的目标不是最大化全量 F1，而是：

```text
在风险极低的区域最大化 coverage
```

这比普通 pairwise EM 的 F1 目标更符合“绝对不能误匹配、允许漏匹配”。

### 2.1 Selective confidence 与 calibration 不是同一个问题

论文明确区分：

- calibration：`0.9` 是否真的约等于 90% 正确率；
- selective classification：confidence 能否把“最可能正确的样本”排在前面。

两者不一定一致。

因此本文方案里的：

```text
MaxLogit-pNorm score = 0.93
```

**不能解释成“93% 概率正确”**。

它首先是一个排序分数。生产自动合并还需要另外一层 operating-point calibration / precision gate。

---

## 3. 核心方法：MaxLogit-pNorm

论文发现，很多模型的 Maximum Softmax Probability（MSP）不是好的错误检测 / selective confidence 指标。

其最有效、又非常轻量的方法之一是：

1. 对 logits 做中心化；
2. 用 p-norm 对 logits 归一化；
3. 取归一化后的最大 logit 作为 confidence。

公式为：

```text
z_center = z - mean(z)

z_norm = z_center / ||z_center||_p

confidence = max(z_norm)
```

即：

```text
MaxLogit-pNorm(z)
= max_k (z_k - mean(z)) / ||z - mean(z)||_p
```

官方代码的核心实现非常短：

```python
def centralize(logits):
    return logits - logits.mean(-1).view(-1, 1)


def MaxLogit_pNorm(logits, p='optimal', centralize_logits=True, **kwargs):
    if centralize_logits:
        logits = centralize(logits)

    if p == 'optimal':
        p = optimize.p(...)

    if p == 'MSP':
        return MSP(logits)

    return max_logit(torch.nn.functional.normalize(logits, p, dim=-1))
```

它有几个工程上非常重要的特点：

- 不需要重新训练主模型；
- 不需要加 confidence head；
- 推理成本几乎只是一个向量归一化；
- `p` 是很小的超参数搜索；
- 可以保留 MSP fallback：如果 pNorm 没在 hold-out 数据上带来收益，就回退到原始 MSP。

### 3.1 为什么它可能比 softmax 更好

softmax 会把所有类别 logits 都通过指数函数放进分母：

```text
softmax(z_k) = exp(z_k) / Σ_j exp(z_j)
```

因此很多“小但数量很多”的 logits 仍可能影响最终 confidence。

论文认为，部分现代模型的问题并不是预测类别错，而是 logit scale / 非目标类别分布让 MSP 产生了不好的错误排序。

pNorm 的作用可以理解成一种 **sample-wise adaptive temperature**：

```text
temperature(x) ≈ ||z(x) - mean(z(x))||_p
```

它在不改变 argmax 类别的情况下，改变不同样本之间 confidence 的相对排序。

### 3.2 `p` 如何选择

论文采用非常简单的 grid search。

官方代码大致是：

```python
for p in p_range:
    score = method(normalize(logits, p))
    metric_value = AURC(score, risk)
    choose p with minimum metric
```

论文实验显示只搜索较小的 `p` 就够了；官方实现的默认范围是整数小范围。

对当前业务不建议无限调参。建议：

```text
p_candidates = [1, 2, 3, 4, 5, 6, 8, 10]
```

然后保持：

```text
if best_p is not materially better than baseline:
    fallback = baseline confidence
```

这样可以避免几百条黄金标签被超参数搜索过拟合。

---

## 4. 论文实验里对当前需求最有价值的结论

论文不是只在一个模型上试验，而是在 84 个 ImageNet 预训练分类器上做 post-hoc 评估，并补充 CIFAR-100、Oxford-IIIT Pet 和 distribution shift 实验。

与当前需求最相关的是四点。

### 4.1 很多“准确模型”其实有坏掉的 confidence estimator

论文在 84 个 ImageNet 模型里发现，多数模型都能通过 post-hoc confidence 修复获得明显 selective performance 提升。

这说明：

> 不要因为 reference linker 的 top-1 accuracy 很高，就直接信它的 softmax tail。

尤其我们的业务只会自动执行 top confidence 尾部，因此“confidence ranking 是否正确”比平均 accuracy 更关键。

### 4.2 post-hoc 修复不改分类结果，只改变“谁值得被信任”

这非常适合 production：

```text
主模型版本保持不动
    ↓
重新标几十/几百条最近数据
    ↓
重调 p + threshold
    ↓
不重新训练主模型即可调整自动放行集合
```

对持续增量、来源分布漂移的三平台数据，维护成本低很多。

### 4.3 数据效率高

论文专门做了 data efficiency 实验。

MaxLogit-pNorm 只有一个主要超参数 `p`，在论文实验里用很少的 hold-out 数据就接近最优 selective performance；论文报告其平均只需不到每类一个 hold-out 样本即可获得很强的结果。

这与 Spec “可人工标注几百对”很契合。

但这里必须加一个业务边界：

> **“几百条足够调 confidence ranking”不等于“几百条足够统计证明 99.99% precision”。**

例如，如果自动接受集合中 500 条都没有观察到 false positive，单侧 95% 置信下界仍大约只能支持 99.4% 左右的 precision；要在 0 个错误下统计支持 99.9%，需要约 3000 条独立 accepted 样本；支持 99.99% 则需要约 3 万条。

所以当前系统的“近乎零误匹配”不能只依赖少量 calibration labels，必须同时依赖：

- reference 硬身份规则；
- 编号角色识别；
- 多证据冲突否决；
- abstain；
- 线上审计与回滚。

### 4.4 对 distribution shift 仍有价值

论文在 ImageNet-C / ImageNetV2 上验证：只用原分布 hold-out 数据调出来的 post-hoc 方法，在 shift 下仍能保留明显收益。

这不代表“永远不用重校准”，但说明它很适合做成独立版本化组件：

```text
base linker model
    ↓
confidence repair config(p)
    ↓
acceptance policy(threshold / hard gates)
```

线上如果某个来源字段、卖家标题风格、OCR 管线发生变化，只需要首先监控 selective risk / coverage；必要时重新标少量近期数据调 `p` 和 acceptance threshold。

---

## 5. 对本项目最关键的技术陷阱：不要直接把 MaxLogit-pNorm 套在二分类 pair matcher 后

当前很多 entity matching 系统是：

```text
(record_a, record_b)
      ↓
二分类模型
      ↓
logits = [z_nonmatch, z_match]
      ↓
softmax / threshold
```

直觉上可能想直接把论文的 MaxLogit-pNorm 放在这里。

**这是不合适的。**

### 5.1 二分类中心化后 pNorm 会退化

令二分类 logits 为：

```text
z = [a, b]
```

中心化：

```text
mean = (a+b)/2

a - mean = (a-b)/2
b - mean = (b-a)/2
```

所以：

```text
z_center = [d/2, -d/2]
```

其中：

```text
d = a-b
```

对任何普通 `p > 0`：

```text
||z_center||_p
= (2 * |d/2|^p)^(1/p)
= |d| * 2^(1/p - 1)
```

最大归一化 logit：

```text
max(z_center / ||z_center||_p)
= 2^(-1/p)
```

**与样本的 `|a-b|` 无关。**

也就是说，在典型二分类场景下，只要 `p>0`，中心化 MaxLogit-pNorm 几乎把所有样本映射成同一个 confidence，失去了排序能力。

论文代码里允许 `p=0`；PyTorch 的 0-“norm”更接近非零元素计数，这种情况下不会完全产生上述常数，但它已经不再是通常意义上的 p-norm scale normalization，效果更接近保留 margin magnitude，不能把论文在大类数 ImageNet 上的结论直接迁移过来。

### 5.2 因此应改造业务建模方式

当前需求的定义本来就是：

```text
同一个商品 = 同一个 reference
```

所以最自然的建模不是：

```text
商品 A × 商品 B -> match / nonmatch
```

而是：

```text
商品记录 -> canonical reference entity
```

跨来源“是否同一个商品”由 canonical entity 自动派生：

```text
record_A.reference_entity_id == record_B.reference_entity_id
```

这样就把任务从二分类改成了：

```text
candidate reference 1
candidate reference 2
...
candidate reference K
NONE / UNKNOWN
```

的多候选实体链接问题。

此时 MaxLogit-pNorm 才真正有适用空间。

这也是本文对当前 Spec 最值得直接落地的架构改造。

---

## 6. 推荐总体架构：Reference-Centric Entity Linking，而不是全量 Pairwise Matching

### 6.1 核心实体

建议系统内部把身份主键定义为：

```text
reference_entity_key = canonical_brand_id + canonical_reference
```

即使业务文案说“同 reference 即同商品”，生产上也应加入 canonical brand，避免不同品牌之间出现碰巧相同的编号字符串。

核心表：

```text
ReferenceEntity
- reference_entity_id
- canonical_brand_id
- canonical_reference
- aliases[]
- status: VERIFIED / PROVISIONAL / BLOCKED
- created_at
- updated_at
```

每条平台商品不是先和另外两平台做笛卡尔积，而是先链接到 `ReferenceEntity`：

```text
雷小安 record ─┐
腕表之家 record ├──> ReferenceEntity(ROLEX, 126610LN)
奢当家 record ─┘
```

这样 100 万–1000 万记录的跨源实体匹配从：

```text
O(N_A * N_B + N_A * N_C + N_B * N_C)
```

变成近似：

```text
O(N * candidate_retrieval(K))
```

其中 `K` 可以固定在很小的 8–32。

而且不会因为一条错误 pair edge 的传递闭包把整个 cluster 污染。

---

## 7. 在线推理流水线

推荐把每条新/更新商品经过以下阶段。

```text
Source Record
   │
   ▼
[1] normalize + canonical brand
   │
   ▼
[2] reference evidence extraction
   │   ├─ structured field
   │   ├─ title / description
   │   ├─ OCR from caseback/card/tag
   │   └─ optional VLM evidence
   ▼
[3] number-role classifier
   │   ├─ BRAND_REFERENCE
   │   ├─ PLATFORM_SKU
   │   ├─ SELLER_SKU
   │   ├─ SERIAL_NUMBER
   │   ├─ COMPATIBLE_MODEL
   │   └─ UNKNOWN
   ▼
[4] candidate retrieval in ReferenceEntity catalog
   │   brand-constrained Top-K
   ▼
[5] candidate disambiguator / reranker
   │   logits over K candidates + NONE
   ▼
[6] MaxLogit-pNorm confidence repair
   │
   ▼
[7] hard evidence gate + selective threshold
   │
   ├─ pass -> LINK TO VERIFIED REFERENCE ENTITY
   │
   └─ fail -> ABSTAIN / HUMAN REVIEW
```

最重要的一条原则：

> **模型从不直接执行“merge 两条商品”。模型只能提出“这条记录属于哪个 canonical reference entity”；最终跨源匹配由相同 verified entity key 的确定性等值关系产生。**

---

## 8. Stage 1：reference evidence extraction

每条商品应保留“候选 reference + 证据来源”，而不是只保留一个扁平字符串。

推荐数据结构：

```json
{
  "record_id": "sdj:123",
  "brand_id": "rolex",
  "reference_evidence": [
    {
      "raw": "126610 LN",
      "normalized": "126610LN",
      "provenance": "title",
      "span": "...劳力士潜航者 126610 LN 黑盘...",
      "role": "BRAND_REFERENCE",
      "extractor": "regex+catalog",
      "score": 0.97
    },
    {
      "raw": "126610LN",
      "normalized": "126610LN",
      "provenance": "ocr:caseback",
      "role": "BRAND_REFERENCE",
      "extractor": "ocr-v3",
      "score": 0.84
    }
  ]
}
```

注意一定要存 `provenance` 和 `role`。

因为以下字符串都可能“很像型号”：

```text
品牌 reference
平台商品 ID
商家 SKU
序列号
配件适配型号
盒证编号
机芯编号
尺寸数字
```

真正危险的 false positive 往往不是模型“不懂语义”，而是**拿错了编号角色**。

---

## 9. Stage 2：canonical reference normalization

对腕表 reference，不建议做过度 fuzzy canonicalization。

可以安全统一的形式差异：

```text
大小写
空格
普通连字符 / 全角连字符
明显的展示分隔符
```

例如：

```text
126610 LN
126610-LN
126610ln
```

可以统一成：

```text
126610LN
```

但不能轻易做：

```text
去掉任意字母后缀
只保留数字主体
编辑距离 <= 1 就认为同 reference
```

因为腕表同系列相邻 reference 的差异本来就可能只在 1–2 个字符。

推荐 canonicalizer 输出：

```text
normalized_reference
normalization_operations[]
normalization_risk
```

如果 normalization 做了任何可能改变语义的操作，直接进入 `ABSTAIN`。

---

## 10. Stage 3：Candidate Retrieval

候选召回必须先品牌约束：

```sql
WHERE canonical_brand_id = :brand_id
```

优先级建议：

```text
1. canonical exact match
2. alias exact match
3. prefix/suffix aware match
4. character n-gram / pg_trgm
5. edit-distance top-K
```

不要一开始就使用语义 embedding 找 reference，因为字母数字型号的 identity 信息主要在字符本身。

一个直接可落地的 PostgreSQL 查询思路：

```sql
SELECT
    reference_entity_id,
    canonical_reference,
    similarity(canonical_reference, :normalized_ref) AS sim
FROM reference_entity
WHERE canonical_brand_id = :brand_id
  AND status IN ('VERIFIED', 'PROVISIONAL')
ORDER BY
    (canonical_reference = :normalized_ref) DESC,
    sim DESC
LIMIT 16;
```

`K=16` 是一个可从小规模实验开始的值，不是论文要求。

对于主流品牌，固定 K 有一个额外好处：MaxLogit-pNorm 的 class dimension 比较稳定。

如果某些品牌候选数明显小于 K，建议单独按 candidate-count bucket 校准 threshold，而不是用 dummy `-inf` logits 强行补齐，因为中心化与 norm 会被无效 logit 扰动。

---

## 11. Stage 4：多候选 Reference Disambiguator

输入：

```text
商品 record evidence
+
K 个 candidate ReferenceEntity
+
NONE / UNKNOWN
```

输出：

```text
logits = [z_ref1, z_ref2, ..., z_refK, z_none]
```

模型不一定要很大。

### 11.1 推荐第一版：轻量 listwise scorer

候选特征：

```text
exact_reference_hit
alias_hit
edit_distance
prefix_match
suffix_match
brand_match
series_match
title_contains_candidate
structured_field_contains_candidate
ocr_contains_candidate
candidate_is_compatible_model_only
role_classifier_score
text_evidence_count
image_evidence_count
conflicting_reference_count
```

可以先做一个小 MLP / LightGBM pair scorer，然后把每个 candidate 的 raw score 拼成固定长度 candidate logits：

```text
score(record, ref_1)
score(record, ref_2)
...
score(record, ref_K)
score(NONE)
```

如果后续需要更强语义理解，再替换成 cross-encoder；post-hoc confidence 层不需要改变。

### 11.2 为什么一定要有 `NONE`

如果候选集没有真实 reference，模型必须有能力说：

```text
NONE / UNKNOWN
```

否则 top-1 永远会选一个“最像但其实错误”的 reference。

对 precision-first 系统，`NONE` 是必需类，不是可选优化。

---

## 12. Stage 5：MaxLogit-pNorm Confidence Repair

对 `K+1` 维 logits：

```python
import torch
import torch.nn.functional as F


def maxlogit_pnorm(logits: torch.Tensor, p: float) -> torch.Tensor:
    z = logits - logits.mean(dim=-1, keepdim=True)
    z = F.normalize(z, p=p, dim=-1)
    return z.max(dim=-1).values
```

预测类别保持：

```python
pred_idx = logits.argmax(dim=-1)
```

confidence 使用：

```python
conf = maxlogit_pnorm(logits, p=best_p)
```

注意：

```text
pred_idx
```

不会因为 pNorm 改变；pNorm 只决定**哪些 top-1 prediction 更值得自动接受**。

### 12.1 p 的离线选择

黄金标签建议不是随机 pair，而是构造“reference disambiguation hard cases”：

```text
同品牌相邻 reference
只有后缀不同
标题里含兼容型号
结构化字段缺失
OCR 识别错 1 字符
不同来源格式不同
平台 SKU 与 reference 同时出现
新旧型号并列
套装 / 配件 / 表带包含适配 reference
```

然后：

```python
for p in [1, 2, 3, 4, 5, 6, 8, 10]:
    conf = maxlogit_pnorm(val_logits, p)
    aurc = compute_aurc(conf, val_wrong)
    choose lowest AURC
```

同时保留 baseline fallback：

```text
if pNorm 没有稳定优于 baseline confidence:
    使用 baseline
```

这里建议保留论文的核心思想：**修复必须是可回退的附加层，不能为了使用新方法而使用新方法。**

---

## 13. Stage 6：不要用单一阈值，必须有 Hard Gate

最终自动绑定条件建议写成一个显式 policy，而不是：

```python
if confidence > 0.99:
    auto_match()
```

推荐：

```python
AUTO_LINK = (
    predicted_entity.status == VERIFIED
    and canonical_brand_is_confident
    and predicted_reference != NONE
    and reference_role_is_brand_reference
    and no_reference_conflict
    and normalization_is_safe
    and hard_evidence_level >= MIN_HARD_EVIDENCE
    and repaired_confidence >= threshold
)
```

### 13.1 强制否决条件

下面任何一个出现都建议直接 `ABSTAIN`：

```text
品牌冲突
两个不同 reference 都有强证据
标题型号与结构化 reference 冲突
标题 reference 与 OCR reference 冲突
编号被判断为 PLATFORM_SKU / SELLER_SKU / SERIAL
候选 reference 只出现在“适配/兼容/for xxx”上下文
candidate entity 仍为 PROVISIONAL
来源 schema 发生未知变化
模型版本 / extractor 版本无对应 calibration
candidate set 异常（真实候选可能未召回）
```

这类 veto 比继续把 threshold 从 0.995 调到 0.9999 更有价值。

---

## 14. 证据分级：把“自动放行”限制在极少数安全路径

建议第一版只开放非常保守的 Tier A / Tier B。

### Tier A：确定性 reference 路径

```text
canonical brand 明确
+
结构化 reference 字段明确为品牌 reference
+
canonicalization 只做安全字符归一化
+
命中 VERIFIED ReferenceEntity exact canonical value
+
无任何冲突 reference
```

这类可以直接 AUTO_LINK，甚至不依赖 ML confidence。

### Tier B：抽取型 reference 路径

```text
canonical brand 明确
+
标题/描述抽到 reference
+
role classifier = BRAND_REFERENCE
+
命中 VERIFIED candidate
+
candidate disambiguator top-1
+
MaxLogit-pNorm 高 confidence
+
没有其他强候选冲突
```

这类才是本文方法主要发挥作用的地方。

### Tier C：OCR / 图像辅助路径

```text
没有可靠文本 reference
只有图片 OCR / VLM 证据
```

第一版建议：

```text
只用于候选召回、冲突否决、人工复核排序
不直接 AUTO_LINK
```

等有足够真实线上标签后再开放。

---

## 15. Threshold 选择：论文 AURC 负责排序，业务 Precision Gate 负责执行

论文用 AURC 优化 confidence ranking。

当前业务应该拆成两步。

### 15.1 第一步：调 p，优化“谁更可靠”的排序

```text
objective = minimize AURC
```

这一步尽量贴近论文。

### 15.2 第二步：独立选择 acceptance threshold

不要在同一小批数据上既调 `p` 又调极端高 precision threshold。

建议黄金标签切成：

```text
train / model fitting
calib_rank / tune p
calib_gate / tune acceptance threshold
shadow_test / final offline check
```

数据很少时可以 cross-validation，但必须保存严格的 out-of-fold score。

Threshold 目标不是普通 accuracy，而是：

```text
maximize coverage
subject to:
    precision_lower_bound >= target
```

例如使用单侧 Clopper-Pearson / Beta posterior lower bound，而不是只看点估计：

```text
observed precision = 100%
```

并不等于真实 precision 足够高。

### 15.3 几百条标签的现实边界

如果 accepted calibration 集中 0 个 false positive：

```text
n ≈ 500    -> 95% 单侧下界约 99.4%
n ≈ 3000   -> 才接近支持 99.9%
n ≈ 30000  -> 才接近支持 99.99%
```

因此：

> 少量标签适合“调排序、发现 hard case、校验规则”，不适合单独承担“绝不误匹配”的统计证明。

真正的保险丝仍然是 verified reference + hard gate + abstain。

---

## 16. 与 `Confidence Classifiers / Conformal` 调研的组合方式

本文方法和之前 `d` 已分析的 conformal / precision guarantee 不是替代关系。

推荐组合：

```text
candidate reference linker
        ↓
MaxLogit-pNorm
        ↓
修复 confidence ranking
        ↓
Conformal / precision risk gate（可选）
        ↓
Hard Rule Gate
        ↓
AUTO_LINK or ABSTAIN
```

理解上：

```text
MaxLogit-pNorm：
    让“最可信的一批”排得更正确

Conformal / precision control：
    尝试对被接受集合的风险做统计约束

Hard reference rules：
    业务身份定义和最后的确定性保险丝
```

这三层职责不同，不应该让其中任一层独自承担全部安全责任。

---

## 17. 数据表设计

### 17.1 `source_item`

```sql
CREATE TABLE source_item (
    source              TEXT NOT NULL,
    source_item_id      TEXT NOT NULL,
    canonical_brand_id  BIGINT,
    title               TEXT,
    raw_payload         JSONB NOT NULL,
    content_hash        TEXT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (source, source_item_id)
);
```

### 17.2 `reference_entity`

```sql
CREATE TABLE reference_entity (
    reference_entity_id BIGSERIAL PRIMARY KEY,
    canonical_brand_id  BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    aliases             JSONB,
    status              TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (canonical_brand_id, canonical_reference)
);
```

### 17.3 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    source              TEXT NOT NULL,
    source_item_id      TEXT NOT NULL,
    raw_token           TEXT NOT NULL,
    normalized_token    TEXT,
    provenance          TEXT NOT NULL,
    role                TEXT NOT NULL,
    extractor_version   TEXT NOT NULL,
    score               DOUBLE PRECISION,
    context             JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 17.4 `item_reference_link`

```sql
CREATE TABLE item_reference_link (
    source                  TEXT NOT NULL,
    source_item_id          TEXT NOT NULL,
    reference_entity_id     BIGINT,
    decision                TEXT NOT NULL,
    confidence_method       TEXT,
    confidence_score        DOUBLE PRECISION,
    pnorm_p                 DOUBLE PRECISION,
    evidence_tier           TEXT,
    model_version           TEXT,
    policy_version          TEXT NOT NULL,
    evidence_snapshot       JSONB NOT NULL,
    decided_at              TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (source, source_item_id)
);
```

`decision` 至少：

```text
AUTO_LINK
ABSTAIN
HUMAN_CONFIRMED
REVOKED
```

所有自动结果必须保留 `model_version + policy_version + evidence_snapshot`，否则后续无法审计为什么发生错误合并。

---

## 18. 增量处理架构

100 万–1000 万级别不需要每次全量重跑。

建议 event-driven / idempotent pipeline：

```text
crawler / import
    ↓
item_upsert event
    ↓
content_hash check
    ↓
changed?
  no  -> stop
  yes -> reference extraction
          ↓
       candidate retrieval
          ↓
       linker logits
          ↓
       confidence repair
          ↓
       policy gate
          ↓
       upsert item_reference_link
```

一个直接能落地的技术栈：

```text
PostgreSQL
  - source_item
  - reference_entity
  - evidence / links
  - pg_trgm reference retrieval

S3 / MinIO
  - 原图 / OCR 中间结果

Python + PyTorch
  - role classifier
  - reference disambiguator
  - MaxLogit-pNorm

Kafka / Redis Streams（数据持续高频更新时）
  - item_upsert 事件

Airflow / Dagster（离线回填、重跑、评估）
  - model/policy version reprocessing
```

如果当前数据更新频率不高，Kafka 不是第一天必须上；先用数据库 job queue 也足够。

---

## 19. 不要存 pairwise match 作为主事实

传统 EM 容易落成：

```text
match_edge(a, b)
```

然后再做 connected components。

当前 Spec 最好不要把它作为身份主事实。

主事实应该是：

```text
item -> reference_entity
```

pairwise relationship 只作为派生视图：

```sql
SELECT a.*, b.*
FROM item_reference_link a
JOIN item_reference_link b
  ON a.reference_entity_id = b.reference_entity_id
WHERE a.source <> b.source
  AND a.decision IN ('AUTO_LINK', 'HUMAN_CONFIRMED')
  AND b.decision IN ('AUTO_LINK', 'HUMAN_CONFIRMED');
```

这样天然避免：

```text
A-B 错一条边
B-C 又正确
=> A/B/C 全部被连成错误 cluster
```

与之前 `TransClean / GraLMatch` 的图清洗思路相比，这是一种更前置的架构规避：能不用不可靠 pair edge 做主身份，就尽量不用。

---

## 20. Reference Catalog 的状态机

长尾型号会不断出现，所以 catalog 不能假定完整。

建议：

```text
NEW STRING
   ↓
PROVISIONAL
   ↓
人工 / 强规则确认
   ↓
VERIFIED
```

另有：

```text
BLOCKED
```

用于已知危险字符串，例如：

```text
经常被误识别的平台 SKU
常见机芯编号
高频兼容型号
OCR 高频乱码模式
```

自动链接只允许：

```text
ReferenceEntity.status == VERIFIED
```

新型号可以先被抽取、聚合、统计，但不急着自动跨源合并。

这非常符合 precision-first。

---

## 21. 线上监控指标

不要只监控 overall accuracy/F1。

应监控 selective system 指标：

### 21.1 Coverage

```text
auto_link_count / eligible_item_count
```

按：

```text
source
source_pair
brand
evidence_tier
model_version
policy_version
```

分桶。

### 21.2 Selective Precision / False Positive Audit

人工抽检只抽 `AUTO_LINK`，而且重点 oversample：

```text
threshold 附近样本
新品牌
新 source schema
OCR 路径
新 reference
相邻 reference hard negative
```

### 21.3 Confidence Drift

监控：

```text
confidence quantiles
accepted score distribution
pNorm vs baseline rank disagreement
NONE rate
candidate retrieval miss rate
reference conflict rate
```

### 21.4 Coverage at fixed risk

比单纯 AURC 更直观的生产指标：

```text
Coverage @ target precision
```

以及：

```text
human-reviewed false positive rate among AUTO_LINK
```

---

## 22. 分布漂移处理

三平台是持续增量数据，很容易发生：

```text
标题模板改变
字段含义改变
卖家群体改变
图片压缩/OCR质量改变
新品牌/新系列进入
```

建议给 confidence repair 层单独版本：

```text
confidence_config
- base_model_version
- source_scope
- candidate_count_bucket
- p
- baseline_method
- threshold
- calibrated_at
- calibration_dataset_id
```

当监控触发：

```text
coverage 突变
conflict rate 上升
人工抽检 FP 上升
score distribution 漂移
```

优先重新收集少量近期 hard cases，重新调 `p + threshold`；如果 top-1 accuracy 本身没下降，不必马上重训大模型。

这正是本文 post-hoc 方法最现实的价值。

---

## 23. Offline Benchmark 应该怎么构造

不能只做随机 train/test split。

建议按风险类型分层：

```text
A. exact structured reference
B. reference in title
C. reference only in description
D. reference from OCR
E. one-character neighbor
F. same family / different suffix
G. accessory contains compatible watch reference
H. seller/platform SKU confusion
I. unseen reference
J. unseen brand/source schema
```

测试指标：

```text
Top-1 reference accuracy
Candidate Recall@K
AURC / NAURC
Coverage @ 100% observed precision
Coverage @ 99.9% observed precision
False Positive count
NONE recall
role-classification confusion matrix
```

其中真正的 release blocker：

```text
False Positive count on hard-negative suite
```

而不是整体 F1。

---

## 24. Shadow Mode 上线方式

第一阶段：

```text
系统正常产生 predicted reference
但不自动写业务 merge
```

只记录：

```text
candidate set
raw logits
MSP
MaxLogit-pNorm
predicted ref
hard-gate result
would_auto_link
```

持续人工审计一段真实增量流量。

第二阶段只打开 Tier A。

第三阶段如果 Tier B 在足够 accepted 样本上稳定，再只开放少数品牌 / source pair。

一旦发现 false positive：

```text
立即 revoke 对应 policy version
回放 evidence_snapshot
加入 hard-negative regression set
必要时 BLOCK 对应 pattern
```

---

## 25. 建议的代码模块边界

```text
src/
  normalization/
    brand.py
    reference.py

  extraction/
    structured.py
    text.py
    ocr.py
    role_classifier.py

  retrieval/
    reference_catalog.py

  linker/
    candidate_features.py
    model.py

  confidence/
    pnorm.py
    tune.py
    risk_coverage.py

  policy/
    hard_gates.py
    threshold.py

  persistence/
    reference_entity_repo.py
    item_link_repo.py

  jobs/
    process_item.py
    recalibrate.py
    shadow_audit.py
```

`confidence/pnorm.py` 应保持无业务逻辑，只做：

```text
logits -> repaired confidence
```

`policy/hard_gates.py` 才负责：

```text
是否允许自动执行
```

这样不会把“模型置信度”和“业务安全规则”混成一个不可解释阈值。

---

## 26. 最小可行实现（MVP）

如果希望最快落地，不需要先训练复杂多模态模型。

### Phase 1：Reference exact identity baseline

实现：

```text
canonical brand
reference extractor
safe normalizer
number-role rules
verified reference catalog
record -> reference_entity exact link
```

只开放 Tier A。

### Phase 2：Candidate disambiguator + pNorm

针对：

```text
reference 缺空格/连字符
OCR 单字符噪声
标题里有多个编号
reference alias
```

做 Top-K candidate classifier，输出多类 logits，再加 MaxLogit-pNorm。

仍只在 hard gate 全过时 AUTO_LINK。

### Phase 3：Selective statistical gate

接入已有 conformal / confidence classifier 调研中的方法，对 Tier B accepted set 做进一步风险控制。

### Phase 4：图片增强

图片只先做：

```text
OCR reference evidence
冲突否决
人工复核排序
```

不急着让视觉相似度定义 identity。

---

## 27. 一个可直接实现的决策函数

```python
def decide_reference_link(record, candidates, logits, config):
    pred_idx = logits.argmax(-1).item()
    pred = candidates[pred_idx]

    # 1. NONE / UNKNOWN 永不自动链接
    if pred.is_none:
        return "ABSTAIN"

    # 2. 只允许 VERIFIED canonical reference
    if pred.status != "VERIFIED":
        return "ABSTAIN"

    # 3. 业务硬冲突直接否决
    if record.brand_conflict:
        return "ABSTAIN"
    if record.reference_conflict:
        return "ABSTAIN"
    if record.best_reference_role != "BRAND_REFERENCE":
        return "ABSTAIN"
    if record.unsafe_normalization:
        return "ABSTAIN"

    # 4. Tier A 可走 deterministic exact path
    if record.has_trusted_structured_reference:
        if record.canonical_reference == pred.canonical_reference:
            return "AUTO_LINK"

    # 5. Tier B 才使用 repaired selective confidence
    conf = maxlogit_pnorm(logits.unsqueeze(0), config.p).item()

    if conf < config.threshold:
        return "ABSTAIN"

    if record.evidence_tier not in config.allowed_tiers:
        return "ABSTAIN"

    return "AUTO_LINK"
```

核心思想：

> **confidence 是最后一个允许条件，不是唯一条件。**

---

## 28. 论文方法哪些可以直接复用，哪些不能

### 可直接复用

1. **Post-hoc confidence repair 层**：不改主模型，只改 confidence ranking。
2. **中心化 + pNorm + MaxLogit** 的实现方式。
3. **小范围 grid search** 调 `p`。
4. **MSP fallback**：新方法无收益时回退。
5. **Risk-Coverage / AURC** 评估方式。
6. **单独监控 distribution shift 下 selective performance**。
7. **少量 hold-out 数据即可调 confidence ranking** 的工程思路。

### 不能直接复用

1. **不能把 ImageNet 多类实验结果直接解释成二奢实体匹配 precision 保证。**
2. **不能把 MaxLogit-pNorm score 当校准概率。**
3. **不能直接用于中心化二分类 pair matcher；会发生退化。**
4. **不能只依赖 AURC 决定 production merge threshold。**
5. **不能用几百条标签宣称统计意义上的“绝不误匹配”。**
6. **不能让视觉/文本模型越过 canonical reference 硬身份规则。**

---

## 29. 对当前 Spec 的最终推荐架构

最终建议不是：

```text
三平台两两 pairwise matcher
+
0.999 threshold
```

而是：

```text
                    ┌──────────────────────┐
                    │ Verified Reference KB │
                    └──────────┬───────────┘
                               │
Source Item                    │
   │                           │
   ▼                           │
Brand Normalize                │
   │                           │
   ▼                           │
Reference Evidence Extract     │
   │                           │
   ▼                           │
Role Classifier                │
   │                           │
   ▼                           │
Brand-constrained Top-K Retrieval
   │
   ▼
Multi-candidate Reference Linker
   │   logits(K + NONE)
   ▼
MaxLogit-pNorm Confidence Repair
   │
   ▼
Precision / Risk Gate
   │
   ▼
Hard Evidence Gate
   │
   ├──────── PASS ────────> item -> VERIFIED ReferenceEntity
   │                             │
   │                             ▼
   │                      cross-source match derived
   │
   └──────── FAIL ────────> ABSTAIN / Human Review
```

这里每层都有清晰职责：

```text
Reference KB            -> 定义“谁是谁”
Extraction              -> 找到可能的 reference 证据
Role classifier         -> 防止把 SKU/序列号当 reference
Retrieval               -> 保证真实 reference 进入小候选集
Linker                   -> 从相似 reference 中选择
MaxLogit-pNorm           -> 修复 confidence 排序
Risk gate               -> 控制自动接受集合
Hard gate               -> 对业务危险情况一票否决
ReferenceEntity equality -> 最终跨源 identity
```

---

## 30. 一句话结论

这篇论文对当前项目最有价值的不是“换一种 confidence 公式”，而是提醒我们：

> **高 accuracy 的 reference linker 仍可能有坏掉的 confidence ranking；在 precision-first 系统里，应把“预测什么”和“哪些预测值得自动执行”拆成两个模块。**

直接落地时，建议把商品匹配重构为 **record → canonical reference entity** 的多候选实体链接，在 `K + NONE` logits 上使用 **MaxLogit-pNorm** 做轻量 post-hoc confidence repair，再叠加独立的 precision/risk gate 与 reference 硬规则。

同时必须避免最危险的误用：

> **不要把中心化 MaxLogit-pNorm 直接套到二分类 pair matcher 上。对二类 logits，普通 `p>0` 时其分数会数学退化，无法承担 selective ranking。**

如果按本文方案实施，MaxLogit-pNorm 最适合成为现有 `reference 抽取 / candidate linker` 与已有 `conformal / precision guarantee / hard rule` 之间的一个非常轻量、可版本化、可回退的“置信度保险层”。
