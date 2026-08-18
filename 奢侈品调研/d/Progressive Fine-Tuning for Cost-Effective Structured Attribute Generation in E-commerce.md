# Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce** 进行深入分析。

- 论文主页：<https://aclanthology.org/2026.acl-industry.40/>
- PDF：<https://aclanthology.org/2026.acl-industry.40.pdf>
- 论文：Lakshman Kolasani, Fatemeh Taheri Dezaki, ACL 2026 Industry Track
- 需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

这篇论文对当前需求最有价值的点，不是“直接拿它的模型来判断两个腕表是不是同一商品”，而是给出了一套已经在大规模电商生产环境验证过的 **小模型 + LoRA + 难例驱动数据选择 + 持续渐进微调** 的模型生命周期。

当前 Spec 的核心定义非常严格：

> **只有同一 reference number / 型号才算同一商品；precision 优先到极致，允许漏匹配，但绝不能误匹配。**

因此推荐把论文方法迁移到 **reference 抽取器**，而不是迁移到最终的 pairwise matcher：

```text
多源商品原始数据
  -> 规则/结构化字段抽取 reference 候选
  -> 小模型负责“抽取 + 编号角色识别 + 拒识”
  -> deterministic canonicalizer 做保守规范化
  -> evidence verifier 验证候选是否有可追溯原始证据
  -> canonical reference exact join
  -> 冲突检查
  -> VERIFIED / REVIEW / REJECT
```

训练侧借鉴论文的 Progressive Fine-Tuning，但需要做一个关键改造：

```text
论文：优先训练“catalog 比模型更完整”的样本
当前业务：优先训练“会制造 false positive / 错 reference”的高风险样本
```

也就是把原论文的 **Completeness-Deficit Guided Curation** 改造成：

> **Precision-Risk Guided Curation / Reference-Error Guided Curation**

生产系统最终必须坚持：

> **模型只能发现 reference 候选，不能凭语言理解创造一个不存在于原始证据中的 reference；自动合并必须由 canonical reference 的可验证硬证据收口。**

如果要从这篇论文直接落一个版本，我建议做成 **“规则硬门控 + 小模型 reference extractor + progressive LoRA hard-case loop”**，这比训练一个泛化的“同款分类器”更符合当前业务。

---

## 1. 为什么这篇论文值得用于当前系统

当前三源商品匹配系统有几个很典型的生产约束：

1. 数据量是 100 万到 1000 万级，持续增量；
2. reference 不是稳定结构化字段，经常埋在标题、描述甚至图片里；
3. 三个平台字段覆盖率和文本风格不同；
4. 新品牌、新卖家模板、新 OCR 噪声会持续造成分布漂移；
5. 允许几百个黄金标签，但不适合长期依赖大规模人工标注；
6. 最终 precision 要远高于普通属性抽取场景。

这篇论文解决的是相邻的一类生产问题：电商商品上架时，用图像 + 文本生成 50+ 个结构化属性。大模型虽然效果好，但成本和延迟太高，因此作者把能力迁移到小模型，并持续用线上数据发现模型弱点，再针对弱点做 LoRA 微调。

这与当前系统的共同点是：

```text
输入：脏、稀疏、多模态电商数据
输出：严格 schema 的结构化字段
线上：持续有新分布
约束：成本、延迟、吞吐量必须可控
```

区别在于风险函数完全不同：

- 原论文追求“尽量多生成合法属性”，用户会逐项审核；
- 当前需求追求“自动合并绝不把两个 reference 搞混”。

所以真正应该复制的是 **训练与持续迭代架构**，而不是论文的 completeness 指标和自动放行逻辑。

---

## 2. 原论文解决什么问题

论文把任务定义为：给定产品输入 `x`，其中可以包含：

- product image；
- product description；

生成结构化属性集合：

```text
y = {a1, a2, ..., an}
```

每个属性都必须符合预先定义的 schema，包括：

- allowed values；
- format；
- constraints。

论文定义的 `Completeness` 是：

```text
Completeness(y)
= 有效且非空的已生成属性数量 / 当前类目要求的属性总数
```

注意它不是 exact-match accuracy。

例如 schema 同时允许 `Navy` 和 `Dark Blue`，模型输出其中任意一个合法值都算 complete。作者另外用人工 audit 测 correctness。

这恰好暴露了它不能原样用于腕表 reference 的原因：

```text
126610LN != 126610LV
```

对于 reference，没有“语义上差不多也算合法”这一说。只要一个字符不同，就可能是完全不同的款。

因此在迁移时必须把 schema 从“语义属性生成”改成“有证据约束的 identifier extraction”。

---

## 3. 原论文整体技术架构

论文的核心训练闭环可以抽象成：

```text
               ┌──────────────────────┐
               │  Production Samples  │
               └──────────┬───────────┘
                          │
                          ▼
               ┌──────────────────────┐
               │ Current Model Mt-1   │
               └──────────┬───────────┘
                          │ inference
                          ▼
               ┌──────────────────────┐
               │ Model Completeness   │
               └──────────┬───────────┘
                          │ compare
                          ▼
               ┌──────────────────────┐
               │ Catalog Completeness │
               └──────────┬───────────┘
                          │
                          ▼
               ┌──────────────────────┐
               │ Completeness Gap     │
               └──────────┬───────────┘
                          │ rank
                          ▼
               ┌──────────────────────┐
               │ Top-k Hard Samples   │
               └──────────┬───────────┘
                          │
                          ▼
               ┌──────────────────────┐
               │ LoRA Fine-Tuning     │
               └──────────┬───────────┘
                          │
                          ▼
               ┌──────────────────────┐
               │ New Checkpoint Mt    │
               └──────────────────────┘
```

最关键的是下面两点：

### 3.1 数据不是随机采样，而是根据当前模型弱点动态选

作者计算：

```text
Gap(x) = Ccat(x) - Cmodel(x)
```

其中：

- `Ccat(x)`：catalog 已存在属性的 completeness；
- `Cmodel(x)`：当前模型输出的 completeness。

每轮从大池子中选择 gap 最大的 `k` 个样本：

```text
St = TopK_x [ Ccat(x) - Cmodel(x) ]
```

直觉是：

> catalog 已经有比较多有效属性、但模型漏掉很多的商品，代表当前模型最明显的可学习弱点。

这是一种非常实用的 active-learning-like data curation，不需要每轮重新让人标大量数据。

### 3.2 每一轮从上一轮 checkpoint 继续，而不是从 base model 重训

论文比较两种方式。

#### Cumulative Training

```text
Mt = FineTune(M0, S1 ∪ S2 ∪ ... ∪ St)
```

每次都从原始 base model `M0` 开始，训练数据越来越大。

#### Progressive Training

```text
Mt = FineTune(Mt-1, St)
```

只用本轮新筛选出来的 hard examples，并从上一轮模型状态继续。

论文在固定训练步数预算下发现，progressive 更有效。原因很直观：

- cumulative 数据越来越多；
- 但训练 steps 不变；
- 每轮有效 epoch 越来越少；
- 每次又需要从 base model 重新学习旧模式。

而 progressive 是在已经学到的表示上继续针对当前错误分布修正。

---

## 4. 原论文的训练配置与生产结果

论文使用 LoRA 做 parameter-efficient fine-tuning。

训练配置：

```text
LoRA rank             = 32
每轮新训练样本 k       = 1,000
每轮 validation       = 100
training steps        = 100
learning rate         = 1e-5
global batch size     = 32
sequence length       = 8,192
GPU                   = 4 × NVIDIA A100
属性生成 batch 数 B    = 4
```

模型对比：

- 大模型 baseline：Claude Sonnet 4.5；
- 小模型：Amazon Nova 2 Lite；
- 小模型通过 LoRA progressive fine-tuning 适配业务。

### 4.1 Offline

论文固定 benchmark 上：

```text
Claude Sonnet 4.5        completeness = 70.40%
Nova 2 Lite base         completeness = 66.03%
Custom iteration 1       completeness = 71.09%
Progressive iteration 2  completeness = 74.20%
Cumulative iteration 2   completeness = 73.72%
```

人工 audit 的 `Correct Completeness`：

```text
Nova base      17.70%
Progressive    20.50%
```

也就是说论文证明：只用两轮、每轮 1K 的 targeted samples，就能让小模型在这个特定结构化生成任务上追上甚至超过大模型的 offline completeness。

### 4.2 Fixed compute budget 下 progressive 的优势

论文展示 progressive 第二轮因为继承前一轮 checkpoint，初始 loss 从约 0.80 降到约 0.59。

相反，cumulative 每次重启，随着数据从 1K 增长到 5K，而每轮仍只有 100 steps：

```text
Round 1: 约 3 epochs
Round 5: 约 0.6 epochs
```

最终 loss 反而从约 0.38 变差到约 0.52。

作者还做了一个额外 ablation，把 cumulative iteration 2 的 steps 翻倍到 200，使有效 epoch 更接近 progressive，但结果仍没有超过 progressive，因此论文把优势主要归因于 checkpoint-based progressive learning，而不是单纯训练步数不足。

### 4.3 生产 A/B

生产环境中：

```text
Claude Sonnet 4.5
  User Acceptance Rate = 88.3%
  Completeness          = 63.0%

Nova 2 Progressive
  User Acceptance Rate = 86.4%
  Completeness          = 59.1%
```

论文同时报告相对大模型：

```text
Inference cost 约降低 98%
p90 latency     约降低 70%
```

但这里有一个比成本数据更重要的结论：

> offline 上小模型 74.2% 高于大模型 70.4%，到了 production 反而 59.1% 低于 63.0%。

作者明确归因于 distribution shift：只用约 2K 样本微调的小模型，对 live traffic 的覆盖还没有大模型广。

这对三源腕表系统非常关键：**不要因为离线黄金集上 precision 很高就直接把模型设成自动合并器。**

---

## 5. 这篇论文不能直接照搬的地方

### 5.1 Completeness 不是我们的核心指标

论文的目标是：

```text
能不能多给用户填一些合法属性
```

当前目标是：

```text
自动合并出去的 pair 是否几乎没有 false positive
```

因此模型输出 100 个 reference，其中 99 个正确、1 个错误，在论文式“属性完整度”视角里可能还很不错；在当前业务里这个错误可能造成整个 reference group 污染。

当前必须优先优化：

```text
Reference Extraction Precision
Auto-Merge Precision
False Positive Count
Wrong-Reference Rate
```

而不是 F1 / completeness。

### 5.2 原论文有人审，当前自动匹配不能假设每条都有人审

论文线上属性建议需要用户确认，所以模型只要生成“可能有用”的值就可以创造价值。

当前系统如果自动把两条记录合并，错误会直接进入下游实体组。因此 reference 抽取必须有更强的硬门控和 `ABSTAIN`。

### 5.3 纯 progressive 可能有遗忘和偏置累积风险

论文只验证了两轮 progressive，并在 Limitations 中明确指出：

- 只验证单一领域和单一 base model；
- 生产有 distribution shift；
- progressive 只验证两轮；
- 长期迭代可能积累由数据选择导致的 distribution bias。

所以当前系统不建议完全照搬：

```text
Mt = FineTune(Mt-1, St)
```

更安全的是：

```text
Mt = FineTune(Mt-1, St ∪ SafetyReplay ∪ DriftAnchors)
```

其中 `SafetyReplay` 是历次最关键的 hard negative / near-reference 样本，永远保留。

### 5.4 论文的 8192 context 对 reference 抽取通常没必要

腕表 reference 大多来自：

- 标题；
- 短描述；
- 结构化属性；
- OCR 文本。

没必要默认使用 8192 token。生产中更应该：

- 先规则裁剪可能含 reference 的局部上下文；
- 只把相关 title / structured fields / OCR snippets 送给模型；
- 使用 2K～4K 甚至更短 context；
- 缩小输出 schema。

这会进一步降低成本和幻觉空间。

---

## 6. 推荐迁移：把 Completeness Gap 改造成 Reference Risk Gap

论文最值得借鉴的是“让当前模型自己暴露最需要训练的数据”。

当前可以定义一个风险优先级函数：

```text
RiskGap(x) =
    w_fp       * FalseReferenceRisk(x)
  + w_conflict * EvidenceConflict(x)
  + w_near     * NearReferenceConfusion(x)
  + w_miss     * VerifiedReferenceMiss(x)
  + w_drift    * SourceDriftScore(x)
  + w_ocr      * OCRDisagreement(x)
```

其中权重必须明显不对称：

```text
w_fp, w_conflict, w_near  >>  w_miss
```

因为当前允许漏匹配，但不允许错匹配。

### 6.1 FalseReferenceRisk

以下情况优先级最高：

- 模型输出了 reference，但原文/结构化字段/OCR 中找不到可追溯证据；
- 模型把平台 SKU 当成品牌 reference；
- 模型把“适配 XX 型号”里的 reference 当成当前商品本体 reference；
- 模型把系列号当具体 reference；
- 模型把一个合法 reference 自动“纠错”成另一个合法 reference。

这些都属于 future false positive seed。

### 6.2 EvidenceConflict

例如：

```text
structured_ref = 126610LN
title_ref      = 126610LN
ocr_ref        = 126610LV
```

不能投票后自动选择 LN；应该进入 hard case / review。

### 6.3 NearReferenceConfusion

腕表最危险的数据不是随机负例，而是：

```text
126610LN vs 126610LV
116500LN vs 126500LN
5711/1A vs 5712/1A
15500ST vs 15510ST
```

这些字符串极像、图片也极像，但业务上明确不是同 reference。

应当把 edit distance 小、同品牌同系列的不同 reference 对作为持续 hard-negative 池。

### 6.4 VerifiedReferenceMiss

如果已经有可信结构化 reference，但模型没抽出来，这是 recall 问题，可以进入训练，但优先级低于 false-reference 风险。

### 6.5 SourceDriftScore

每个来源单独监控：

- 新标题模板；
- 新字段；
- 分隔符变化；
- 品牌分布变化；
- OCR 图片类型变化；
- reference token 长度/字符分布变化。

只要某个来源出现明显漂移，优先从该来源抽 hard cases。

---

## 7. 直接落地架构

### 7.1 总体架构

```text
雷小安 ───────┐
腕表之家 ─────┼──> Raw Product Lake
奢当家 ───────┘          │
                         ▼
                ┌──────────────────┐
                │ Field Normalizer │
                └────────┬─────────┘
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
  Deterministic Candidate       OCR / Image Evidence
       Extraction                    Extraction
             │                       │
             └───────────┬───────────┘
                         ▼
                ┌──────────────────┐
                │ SLM Ref Extractor│
                │ + Role Classifier│
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ Evidence Verifier│
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ Canonicalizer    │
                │ deterministic    │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ Reference Index  │
                └────────┬─────────┘
                         │ exact join only
                         ▼
                ┌──────────────────┐
                │ Conflict Guard   │
                └────────┬─────────┘
                         │
                ┌────────┴────────┐
                ▼                 ▼
           VERIFIED            REVIEW
```

核心原则：

> **SLM 不直接决定 record A == record B；SLM 只产生受约束、可审计的 reference observation。**

最终匹配是：

```text
same_brand
AND canonical_reference_a == canonical_reference_b
AND reference evidence verified
AND no hard conflict
```

---

## 8. Reference Extractor 的输出协议

不要让模型只返回：

```json
{"reference": "126610LN"}
```

这不够审计，也很容易幻觉。

建议输出：

```json
{
  "brand": {
    "raw": "ROLEX",
    "canonical": "rolex"
  },
  "reference_candidates": [
    {
      "raw": "126610LN",
      "canonical_hint": "126610LN",
      "evidence_type": "title",
      "evidence_text": "劳力士潜航者 126610LN 全套",
      "start": 7,
      "end": 15,
      "role": "self_reference",
      "model_confidence": 0.997
    }
  ],
  "decision": "single_candidate"
}
```

`role` 建议至少支持：

```text
self_reference       当前商品自己的品牌 reference
compatible_reference 配件/兼容对象的 reference
platform_sku         平台/店铺 SKU
serial_number        序列号
family_or_series     系列/家族编号
ambiguous_identifier 无法判断的编号
```

只有 `self_reference` 才可能进入 canonical reference。

模型只给 `canonical_hint`，最终 canonicalization 应由 deterministic code 完成，避免模型擅自“修正”字符。

---

## 9. Evidence Verifier：阻止模型创造 reference

生产环境增加一条极重要规则：

```text
模型输出的 reference 必须绑定原始 evidence span。
```

验证逻辑可以是：

```python
def verify_candidate(candidate, product):
    raw = candidate.raw

    if candidate.evidence_type == "structured_field":
        return exact_or_allowed_normalization(raw, product.raw_ref_field)

    if candidate.evidence_type in {"title", "description"}:
        return span_exists(raw, candidate.evidence_text, product.raw_text)

    if candidate.evidence_type == "ocr":
        return raw in product.ocr_tokens and product.ocr_quality >= OCR_MIN

    return False
```

禁止下面这种流程：

```text
标题：劳力士黑水鬼 41mm
LLM：我知道它大概是 126610LN
系统：自动写入 reference=126610LN
```

这在知识问答里可能是“聪明”，在 identifier matching 里是灾难。

应该变成：

```text
没有可追溯 reference token -> ABSTAIN
```

如果业务以后确实希望根据图片/知识库推断 reference，也只能作为 `suggested_reference` 进入人工审核，不能进入 auto-match key。

---

## 10. Canonicalization 必须保守且 deterministic

推荐 canonicalization 只做明确无语义变化的转换：

```text
Unicode normalize
全角 -> 半角
大小写统一
去首尾空格
已验证的 separator 统一
已验证的品牌专用 format rule
```

禁止通用 fuzzy correction：

```text
O -> 0
I -> 1
S -> 5
删除任意横杠
自动补字符
按编辑距离纠错到最近合法 reference
```

除非某个品牌已经有明确格式规则并经过黄金集验证。

建议每条 canonicalization 都返回 provenance：

```json
{
  "raw": "126610-LN",
  "canonical": "126610LN",
  "rules": ["uppercase", "rolex_verified_separator_rule_v2"],
  "canonicalizer_version": "2026-08-18.3"
}
```

只要不能证明转换安全，就保留原值并进入 review。

---

## 11. 匹配决策不要训练成一个黑盒分类器

最终 matcher 可以非常简单：

```python
def auto_match(a, b):
    if a.brand_id is None or b.brand_id is None:
        return "UNKNOWN"

    if a.brand_id != b.brand_id:
        return "NO_MATCH"

    if not a.reference_verified or not b.reference_verified:
        return "UNKNOWN"

    if a.canonical_reference != b.canonical_reference:
        return "NO_MATCH"

    if hard_conflict(a, b):
        return "REVIEW"

    return "MATCH"
```

这里 text embedding / image embedding / LLM similarity 不参与最后的 `MATCH` 放行。

它们可以做：

- reference 候选发现；
- OCR；
- 数据质量冲突发现；
- REVIEW 队列排序；
- 没 reference 时给人工找可能同款候选。

但不允许：

```text
图片 0.99 相似 + 标题很像
=> 即使 reference 不同也自动 MATCH
```

---

## 12. 千万级数据如何扩展

### 12.1 不做全笛卡尔 pairwise

如果 1000 万条记录两两比较，复杂度不可接受。

当前业务的天然 blocking key 就是：

```text
(brand_id, canonical_reference)
```

数据结构：

```sql
CREATE TABLE reference_observation (
    product_id            BIGINT NOT NULL,
    source_id             SMALLINT NOT NULL,
    brand_id              BIGINT,
    reference_raw         TEXT,
    reference_canonical   TEXT,
    role                   TEXT,
    evidence_type          TEXT,
    evidence_text          TEXT,
    verified               BOOLEAN NOT NULL,
    extractor_version      TEXT NOT NULL,
    canonicalizer_version  TEXT NOT NULL,
    created_at             TIMESTAMP NOT NULL
);

CREATE INDEX idx_ref_exact
ON reference_observation (
    brand_id,
    reference_canonical
)
WHERE verified = TRUE
  AND role = 'self_reference';
```

最终聚合：

```sql
SELECT
    brand_id,
    reference_canonical,
    ARRAY_AGG(product_id) AS products
FROM reference_observation
WHERE verified = TRUE
  AND role = 'self_reference'
GROUP BY brand_id, reference_canonical;
```

复杂度主要落在：

- 每条商品一次 reference extraction；
- exact key index；
- 小组内冲突审计。

而不是 `O(N²)`。

### 12.2 内容 hash 缓存

reference extractor 的输入可以计算：

```text
hash(title + desc + structured_ref + ocr_text + image_hashes)
```

内容没变则复用旧 inference，新增/变更记录才跑模型。

### 12.3 增量流

```text
new/updated product
 -> normalize
 -> deterministic candidate extraction
 -> only unresolved cases call SLM
 -> verify
 -> canonicalize
 -> upsert reference index
 -> re-evaluate affected reference group only
```

这样持续增量不会触发全量重跑。

---

## 13. 小模型应该只处理规则解决不了的部分

生产成本最低、风险最低的方式不是“所有商品都走 VLM”。

建议 cascade：

```text
Level 0: 结构化 reference 字段 + 强校验
         -> 直接 VERIFIED

Level 1: 标题/描述 deterministic regex + 品牌 pattern
         -> 高置信候选

Level 2: SLM/VLM 做候选角色判断和歧义解析
         -> VERIFIED 或 REVIEW

Level 3: OCR/image-only
         -> 默认更保守，必要时人工
```

例如：

```text
来源字段明确：reference_number=126610LN
且品牌=Rolex
且 pattern 合法
```

没必要调用大模型。

论文证明小模型在结构化电商任务上通过针对性 fine-tune 可以大幅降低成本；当前还可以进一步通过 cascade 降低调用量。

---

## 14. 训练数据怎么构造

### 14.1 从可信结构化字段生成弱监督正样本

如果某来源已有高质量 `reference_number` 字段：

```text
input: 标题 + 描述 + 其他字段
label: 已验证 reference + evidence span/role
```

这相当于论文用 catalog attributes 做 proxy label。

但不能默认所有平台结构化字段都是真值，应先抽样审计来源质量。

### 14.2 黄金标签重点标“难负例”而不是随机 pair

几百对人工标签建议这样花：

```text
约 40%：同品牌同系列、不同 reference 的近邻负例
约 20%：配件/表带/盒证/兼容型号场景
约 15%：平台 SKU / 店铺货号 / 序列号混淆
约 15%：OCR 冲突、I/1/O/0 等字符问题
约 10%：完全正样本和普通负样本做 sanity check
```

随机抽两个完全不同品牌的商品作为 negative 几乎没有训练价值。

### 14.3 一个 pair 最好同时转成 record-level extraction label

如果人工确认：

```text
A.reference = 126610LN
B.reference = 126610LV
A != B
```

不要只保存 pair label：

```text
(A, B) -> NO_MATCH
```

还应该保存：

```text
A -> self_reference=126610LN + evidence span
B -> self_reference=126610LV + evidence span
```

这样标签既能训练 extractor，也能训练 verifier 和 hard-negative 逻辑，复用价值更高。

---

## 15. 推荐的 Hybrid Progressive Fine-Tuning

论文的 progressive 思路值得保留，但当前要增加 safety replay。

每轮训练集合：

```text
Train_t = HardBatch_t
        + SafetyReplay
        + SourceDriftAnchors
        + SmallRandomAnchor
```

建议比例从下面起步再实验：

```text
HardBatch_t        60%
SafetyReplay       25%
SourceDriftAnchors 10%
RandomAnchor        5%
```

其中 `SafetyReplay` 永远包括：

- 历史产生过 false positive 的样本；
- 同品牌相邻 reference；
- accessory / compatible 型号；
- OCR 混淆；
- 平台 SKU 混淆。

训练状态：

```text
M0
 -> LoRA round1
 -> checkpoint M1
 -> LoRA round2 on new hard batch + safety replay
 -> checkpoint M2
 -> ...
```

每次新模型都必须通过固定 safety regression suite 才能发布。

---

## 16. Loss 也要按 precision-first 改造

如果把 reference extraction 做成生成任务，单纯 token-level cross entropy 不会表达业务中的巨大 false-positive 成本。

可以从工程上先通过数据重采样实现非对称风险：

- hard negative 大量进入 batch；
- false-reference 样本重复采样；
- `ABSTAIN` 样本必须足够多；
- 训练输出要求 evidence span。

如果做额外 classifier / verifier，则可以显式使用 cost-sensitive loss：

```text
loss =
  λ_fp * BCE(false_positive)
+ λ_fn * BCE(false_negative)
```

并设置：

```text
λ_fp >> λ_fn
```

最终阈值也不要按 F1 选择，而是：

```text
在 validation 上先满足 precision constraint
再在约束内最大化 coverage
```

---

## 17. 模型发布门禁

建议每个模型 checkpoint 都有以下 gate。

### Gate 1：Reference Extraction Precision

必须在固定黄金集达到极高 precision。

### Gate 2：Zero Regression on Critical Near-Reference Set

例如已知危险对：

```text
126610LN / 126610LV
116500LN / 126500LN
```

任何一条旧的 critical regression 重新出错，禁止发布。

### Gate 3：Source Slice

分别检查：

```text
雷小安
腕表之家
奢当家
```

避免总体指标掩盖某一来源崩坏。

### Gate 4：Brand Slice

高频品牌和高风险品牌分开看。

### Gate 5：Abstention

`ABSTAIN` 不是失败，而是安全特性。

必须监控：

```text
auto_accept_rate
review_rate
abstain_rate
false_positive_rate
```

如果某版模型 coverage 提升但 FP 也提高，不应上线。

---

## 18. 不要只看离线随机切分

论文最值得警惕的实验结果就是：

```text
offline：fine-tuned SLM > 大模型
online ：fine-tuned SLM < 大模型
```

当前评测集必须模拟真实 drift。

建议至少四种切分：

### 18.1 Time Split

旧时间训练，新时间测试。

### 18.2 Source Holdout / Template Drift

故意保留某些新卖家模板或字段格式做测试。

### 18.3 Unseen Reference Split

测试集中放训练没见过的 reference。

系统不能靠记忆型号表取得虚假高分。

### 18.4 Near-Reference Challenge Set

专门测：

```text
同品牌
同系列
图片高度相似
reference 只差 1～2 个字符
```

这个集合对当前业务比普通随机 F1 更重要。

---

## 19. 图片应该怎么用

当前有图片，但图片不能越权替代 reference。

推荐三种用途：

### 19.1 OCR Reference Candidate

从：

- 表背；
- 保卡；
- 吊牌；
- 证书；

抽取 identifier 候选。

### 19.2 Evidence Conflict

如果文本是 `126610LN`，OCR 明确是 `126610LV`：

```text
=> REVIEW
```

而不是让视觉 embedding 投票。

### 19.3 REVIEW Ranking

没有 reference 的记录，可以用图像相似度把可能相关的候选排给人工，但：

```text
image similarity != auto-match authorization
```

---

## 20. Reference Master / Dictionary 的作用

建议维护一个 versioned reference master：

```text
reference_master(
  brand_id,
  canonical_reference,
  family,
  known_alias_format,
  active_from,
  active_to,
  source,
  confidence,
  version
)
```

它的用途：

- 校验格式；
- 构造 hard negatives；
- 生成品牌专用 canonicalization rule；
- 做候选合法性检查；
- 发现新 reference。

但注意：

> reference master 只能帮助“验证候选”，不能让模型在原文没有 reference 时凭最近邻自动补 reference。

例如标题出现 `126610LV`，不能因为 master 中 `126610LN` 更常见就改成 LN。

---

## 21. Progressive 数据选择伪代码

```python
def risk_score(sample, model_output, trusted_evidence):
    score = 0.0

    if model_output.reference and not has_traceable_evidence(model_output, sample):
        score += 100.0

    if is_wrong_role(model_output):
        score += 80.0

    if conflicts_with_trusted_reference(model_output, trusted_evidence):
        score += 100.0

    if is_near_reference_confusion(model_output, trusted_evidence):
        score += 90.0

    if trusted_evidence.reference and model_output.decision == "abstain":
        score += 10.0

    score += 20.0 * source_drift_score(sample.source)
    score += 30.0 * ocr_text_conflict_score(sample)

    return score


def select_progressive_batch(pool, current_model, k):
    scored = []
    for x in pool:
        y = current_model.predict(x)
        s = risk_score(x, y, trusted_evidence(x))
        scored.append((s, x))

    return top_k(scored, k)
```

与论文的最大差异是：

```text
论文 top-k：模型少填属性的样本
这里 top-k：模型可能造成错误 reference 的样本
```

---

## 22. 在线 hard-case 回流

推荐把以下线上事件自动写入 `hard_case_queue`：

```text
1. 模型 reference 与结构化 reference 冲突
2. 标题和 OCR reference 冲突
3. 同一个商品抽出多个 self_reference
4. canonical reference 合并后组内出现不同强证据 reference
5. 人工把模型建议从 MATCH 改成 NO_MATCH
6. 人工把 reference 修改为另一个值
7. 新品牌/新格式无法识别
8. 模型输出没有 evidence span
9. near-reference candidate 得分过高
```

队列表：

```sql
CREATE TABLE reference_hard_case (
    case_id          BIGSERIAL PRIMARY KEY,
    product_id       BIGINT NOT NULL,
    source_id        SMALLINT NOT NULL,
    case_type        TEXT NOT NULL,
    risk_score       DOUBLE PRECISION NOT NULL,
    model_version    TEXT NOT NULL,
    payload          JSONB NOT NULL,
    human_label      JSONB,
    created_at       TIMESTAMP NOT NULL,
    labeled_at       TIMESTAMP
);

CREATE INDEX idx_hard_case_priority
ON reference_hard_case (human_label, risk_score DESC);
```

这就是论文“implicit feedback + hard-example curation”在当前系统中的生产化版本。

---

## 23. 模型/规则全部版本化

每个 match 必须能回答：

```text
为什么这两条商品被合并？
```

建议保存：

```text
extractor_model_version
prompt_schema_version
ocr_model_version
canonicalizer_version
reference_master_version
verification_rule_version
match_decision_version
```

匹配记录：

```json
{
  "decision": "MATCH",
  "brand": "rolex",
  "canonical_reference": "126610LN",
  "a_evidence": {
    "type": "structured_field",
    "raw": "126610LN"
  },
  "b_evidence": {
    "type": "title",
    "raw": "126610LN",
    "span": [12, 20]
  },
  "extractor_version": "ref-slim-v7",
  "canonicalizer_version": "canon-v3",
  "decision_rule": "exact_verified_ref_v2"
}
```

这样发生误匹配时可以精确定位：

- extractor 错；
- OCR 错；
- role classification 错；
- canonicalization 错；
- rule 错。

否则只看到一个最终 similarity score，很难修。

---

## 24. 最小可行落地版本

### Phase A：先做完全确定的 Reference Core

只实现：

1. 品牌 canonicalization；
2. 结构化 reference 字段；
3. 品牌专用保守 regex；
4. deterministic canonicalization；
5. `(brand, canonical_reference)` exact index；
6. evidence provenance；
7. 不确定全部 `UNKNOWN`。

目标：先拿到一个 precision 极高但 coverage 可以低的 baseline。

### Phase B：加 SLM Reference Extractor

只处理 Phase A 未解决数据：

```text
title / desc -> candidate + role + evidence span
```

模型不直接参与 match。

### Phase C：加图片 OCR

OCR 结果也作为 observation，而不是自动真值。

### Phase D：加 Progressive Fine-Tuning

从线上 hard cases 中按 `RiskGap` 选 top-k，做 LoRA incremental fine-tune。

### Phase E：加长期 drift loop

新来源/新品牌/新格式进入：

```text
Drift detection
 -> hard-case sampling
 -> human label
 -> progressive LoRA
 -> safety regression
 -> shadow deploy
 -> gradual release
```

---

## 25. 与论文训练配置的具体映射

可以把论文配置当成起始实验参考，而不是照抄：

| 论文 | 当前建议 |
|---|---|
| LoRA rank=32 | 可以从 16/32 做小范围比较 |
| 每轮 1000 samples | 第一版几百黄金标签 + 弱监督 hard case 即可启动 |
| 100 validation | 当前应建立更大的长期固定 safety set，尤其 near-reference negatives |
| lr=1e-5 | 可作为起点，但要按 base model 调 |
| batch=32 | 可复用为起点 |
| seq=8192 | reference extraction 建议显著缩短，通常 2K～4K 足够 |
| 4×A100 | 不是业务约束，按所选小模型和 LoRA/QLoRA 调整 |
| 4 个属性 batch | 当前只抽 reference/role/evidence，不需要照搬 B=4 |
| completeness gap | 替换为 precision-risk gap |
| pure progressive | 改成 progressive + safety replay |

最应该保留的是：

```text
current model -> find current weaknesses -> train only valuable examples -> continue from previous checkpoint
```

而不是某个固定硬件或超参数。

---

## 26. 线上指标体系

当前不要把“整体 F1”作为第一看板。

### 26.1 P0 指标

```text
AutoMergePrecision
AutoMergeFalsePositiveCount
WrongReferenceExtractionCount
CriticalRegressionErrorCount
```

### 26.2 P1 指标

```text
VerifiedReferenceCoverage
AutoMergeCoverage
AbstainRate
ReviewRate
ReferenceExtractionPrecision
```

### 26.3 Drift 指标

按：

```text
source
brand
category
extractor_version
OCR / non-OCR
structured / title / image-only
```

切片。

特别是：

> **只要线上出现已确认 false positive，应立即进入 safety replay，而不是等下一次随机重训。**

---

## 27. 置信度不要直接当真值

LLM/SLM 的 token probability 或 classifier score 可以用于排序，但不能直接解释成业务 precision。

建议把最终自动放行做成多条件 gate：

```text
Gate A: candidate role == self_reference
Gate B: evidence traceable
Gate C: deterministic canonicalization successful
Gate D: brand consistent
Gate E: no contradictory strong reference
Gate F: source-specific quality rule satisfied
Gate G: model/reference pattern in calibrated safe region
```

只有全部通过：

```text
AUTO_ACCEPT
```

否则：

```text
REVIEW / UNKNOWN
```

这样即使模型 confidence calibration 漂移，也不会单点导致错误合并。

---

## 28. 一个具体腕表示例

原始数据：

```text
A / 腕表之家
品牌：Rolex
标题：劳力士潜航者型 126610LN 黑盘 41mm

B / 奢当家
品牌：劳力士
标题：黑水鬼 41 自动机械 全套
描述：Ref. 126610LN

C / 雷小安
品牌：ROLEX
标题：潜航者 126610LV 绿盘
```

抽取后：

```text
A -> rolex / 126610LN / title / verified
B -> rolex / 126610LN / description / verified
C -> rolex / 126610LV / title / verified
```

结果：

```text
A-B = MATCH
A-C = NO_MATCH
B-C = NO_MATCH
```

即使：

```text
image_similarity(A, C) = 0.98
text_similarity(A, C)  = 0.95
```

也不能改变结果。

另一种场景：

```text
D 标题：Rolex Submariner 41mm 黑盘
```

模型知道它“很可能”是 126610LN，但原始数据没有 reference evidence：

```text
D -> reference = NULL / ABSTAIN
```

可以把 126610LN 放到 `review_suggestion`，但不能自动加入 A/B 的实体组。

---

## 29. 这篇论文对当前需求最值得吸收的三个点

### 29.1 模型不用一直追求更大，关键是把业务 hard cases 喂对

论文用小模型 + targeted LoRA 接近大模型效果并显著降低成本。

当前千万级持续增量尤其适合：

```text
规则覆盖 easy cases
小模型覆盖 ambiguous extraction
大模型只用于离线 teacher / 人工辅助（可选）
```

### 29.2 数据选择比无脑累积训练集更重要

对于当前系统，最有价值的第 1001 个训练样本，不是随机商品，而是：

```text
刚刚差点把 126610LV 错识成 126610LN 的那个样本
```

### 29.3 线上分布漂移必须成为模型训练闭环的一部分

论文 offline/online 排名反转已经说明：静态 benchmark 不够。

当前应该把每次新抓取模板、新品牌、新 OCR 格式都视为潜在新 domain，并让 hard-case curation 自动捕获。

---

## 30. 最终推荐方案

如果只给当前 Spec 一个可直接实施的方案，我会选：

```text
1. 先构建 Brand + Canonical Reference 的强规则实体键
2. 模型只用于 Reference Observation Extraction
3. 所有 observation 必须带原始 evidence provenance
4. Canonicalization 完全 deterministic、品牌感知、保守
5. exact verified reference 才允许自动跨源合并
6. 图片只做 OCR / 冲突发现 / REVIEW 排序
7. 几百个人工标签优先做 near-reference hard negatives
8. 线上错误和冲突自动进入 RiskGap hard-case queue
9. 用 LoRA 从上一 checkpoint progressive fine-tune
10. 每轮混入 SafetyReplay，避免长期遗忘/偏置
11. 固定 critical regression set 作为模型发布门禁
12. 未通过任何硬门控的样本全部 ABSTAIN / REVIEW
```

建议系统职责边界最终明确成：

```text
Extractor：我看到了哪些 identifier？
Role Classifier：这个 identifier 是不是当前商品自己的 reference？
Canonicalizer：如何无损规范化？
Verifier：证据是否真实存在、是否冲突？
Matcher：两个 VERIFIED canonical reference 是否严格相等？
Human：只处理 UNKNOWN / CONFLICT / 高风险 drift case。
```

这比把所有工作塞进一个“多模态相似度模型”更安全、更可解释，也更容易在千万级数据上持续迭代。

---

## 31. 对论文方法的最终评价

这篇论文的生产价值很高，但它对当前 Spec 的价值主要在 **模型工程和持续学习机制**，不是 matcher 本身。

最值得复用：

- Completeness-deficit 类的难例定向采样思想；
- 从上一 checkpoint 继续的 progressive LoRA；
- 小模型替代昂贵大模型的生产策略；
- offline + production A/B 双重评估；
- 用线上反馈持续找当前模型弱点。

最不能照搬：

- completeness 作为核心目标；
- “schema-valid 就有价值”的宽松正确性定义；
- 用户会审核所有输出的风险假设；
- 纯 progressive 长期迭代；
- 把模型生成字段直接当可靠 identifier。

迁移到二奢/腕表场景后，我认为最佳形态是：

> **Progressive Fine-Tuning 不负责教系统“更大胆地匹配”，而负责教 Reference Extractor 在越来越多真实难例上“更准确地抽取、识别错误角色，并在没有证据时更愿意拒识”。**

这与当前“precision 极端优先、允许漏匹配”的目标是一致的。

---

## 32. 参考资料

1. Kolasani, L.; Taheri Dezaki, F. **Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce**. ACL 2026 Industry Track.  
   <https://aclanthology.org/2026.acl-industry.40/>

2. PDF：  
   <https://aclanthology.org/2026.acl-industry.40.pdf>

3. 当前需求 Spec：  
   <https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

4. LoRA：Hu et al., **LoRA: Low-Rank Adaptation of Large Language Models**.  
   <https://arxiv.org/abs/2106.09685>
