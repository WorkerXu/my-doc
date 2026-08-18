# Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce：面向跨源二奢 reference 解析的持续学习落地方案

> 分析人：c  
> 原文：ACL 2026 Industry Track，*Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce*  
> 论文：https://aclanthology.org/2026.acl-industry.40/  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心约束：**“同一个商品”= 同一 reference number；precision 极端优先；允许漏匹配；字段稀疏；图片可用；100 万–1000 万级且持续增量。**

---

## 1. 为什么这篇论文值得单独分析

此前围绕当前需求已经分析过 blocking、selective matching、图一致性清洗、多模态实体链接、属性抽取、编号角色分类等方法。这篇论文补上的不是“又一个 matcher”，而是一个生产系统经常被低估的能力：

> **如何让一个低成本、小模型的结构化抽取器在持续到来的线上数据上不断变强，同时把训练成本和推理成本控制在可接受范围内。**

对当前跨源二奢/腕表项目而言，真正应该长期学习的组件不是最终的“pairwise 是否匹配”分类器，而是：

```text
record -> canonical_brand + canonical_reference + evidence + abstain
```

因为 Spec 已经把实体身份定义为 reference number。一旦每条记录都能被高精度解析出可信的 canonical reference，最终跨源匹配可以退化成确定性的 key join，而不需要在千万级数据上做大规模 pairwise 语义判断。

这篇论文的两个核心思想非常适合这层“reference parser”持续迭代：

1. **Completeness-deficit guided curation**：只挑当前模型最弱的样本继续训练；
2. **Progressive fine-tuning**：下一轮从上一轮 checkpoint 继续 LoRA 微调，而不是每次回到 base model 用所有历史数据重训。

但必须强调：**论文原任务的优化目标不是 precision-first entity matching**。它优化的是电商商品属性生成的 completeness，而且线上流程里所有建议最终由用户审核。因此不能原样照搬。当前需求必须把“completeness gap”改造成“reference 风险/缺口驱动的难例选择”，并在模型之外增加不可绕过的确定性验证门。

---

# 2. 原论文到底做了什么

## 2.1 任务定义

论文研究的是电商 listing 的结构化属性生成。输入是：

```text
product image + product description
```

输出是满足预定义 schema 的一组属性：

```text
brand
color
material
size
...
```

平均每个商品需要生成 50+ 属性。

论文把 completeness 定义成：请求的属性中，有多少属性生成了**非空且 schema-valid** 的值。

形式上：

```text
Completeness(y)
= valid non-empty attributes / all requested attributes
```

这里有一个对当前需求极其重要的差异：

> 论文中的 completeness 不要求“精确等于 catalog ground truth”。

例如 schema 同时允许 `Navy` 和 `Dark Blue`，模型生成 `Navy`、catalog 是 `Dark Blue`，在 completeness 指标里仍然算完整。论文另外通过人工 audit 测 correctness。

这对普通 listing assistance 合理，但对 reference number 完全不够：

```text
126610LN ≠ 126610LV
```

两个值都“像合法 Rolex reference”，但对当前业务而言是不同实体。reference 必须使用**精确身份语义**，而不是“schema-valid 即可”。

---

## 2.2 Completeness-Deficit Guided Curation

论文最核心的数据选择思想如下。

对同一个商品 x，分别计算：

```text
C_cat(x)   = catalog 中已有属性的 completeness
C_model(x) = 当前模型生成属性的 completeness
```

定义：

```text
Gap(x) = C_cat(x) - C_model(x)
```

每一轮不随机采样训练数据，而是取 Gap 最大的 k 个样本：

```text
S_t = top-k samples by [C_cat(x) - C_M(t-1)(x)]
```

它本质上是一个生产化 active learning/continual learning 采样器：

```text
当前模型在哪些真实商品上最“缺”
    -> 把这些商品作为下一轮训练数据
    -> 更新模型
    -> 再重新扫描当前弱点
```

论文强调，catalog 中已经存在、且经过用户/业务流程确认的结构化属性可以充当 proxy label，因此不需要每一轮都重新人工标注。

### 对当前需求的直接启发

我们也有大量“隐式监督”可以利用：

- 来源中明确独立 reference 字段；
- 已人工确认的黄金对；
- 同品牌官方/可信 catalog 中的 reference；
- 人工复核队列中的 accept/reject/correction；
- 多来源中高可信结构化 reference 的一致结果；
- 旧模型自动结果被后续数据推翻或产生冲突的记录。

这些信号可以组成一个**reference deficit / conflict score**，持续挑选最值得训练的样本，而不是不断随机追加海量数据。

---

## 2.3 Progressive Training vs Cumulative Training

论文比较两种方式。

### Cumulative Training

每轮都从 base model `M0` 开始，用到目前为止所有数据重训：

```text
M_t = FineTune(M0, S1 ∪ S2 ∪ ... ∪ St)
```

### Progressive Training

每轮从上一轮模型继续训练，只使用这一轮新增的 hard samples：

```text
M_t = FineTune(M_(t-1), S_t)
```

也就是：

```text
M0
 ↓ + hard batch 1
M1
 ↓ + hard batch 2
M2
 ↓ + hard batch 3
M3
 ...
```

论文在固定每轮 100 training steps 的预算下发现，progressive 更容易收敛。

原因也很直观：累计训练每次从 base 开始，随着累计数据从 1K 变成 5K，固定 100 steps 对数据的有效 epoch 会越来越少。论文报告累计方式从约 3 epochs 降到约 0.6 epochs，最终 loss 反而变差；progressive 则保留了上一轮已经学到的表示，只需针对当前 hard batch 做增量适配。

这个思路非常适合当前“来源持续增量、品牌/型号分布持续变化”的情况。

---

## 2.4 参数高效微调配置

论文的实验配置很明确：

```text
Base/SLM: Amazon Nova 2 Lite
PEFT: LoRA
LoRA rank: 32
每轮新增训练样本: 1,000
每轮 validation: 100
training steps: 100
learning rate: 1e-5
global batch size: 32
sequence length: 8,192
GPU: 4 × NVIDIA A100
```

论文用 Claude Sonnet 4.5 作为大模型 baseline。

值得注意：这**不是经典意义上的 teacher-student distillation**。论文的主要训练标签来自已经 review/verified 的 catalog listing attributes；核心是“高质量 proxy labels + 难例筛选 + LoRA progressive fine-tuning”。因此在当前项目里也没有必要先引入复杂蒸馏体系，只要能积累可信的 reference label，就可以先训练专用小模型。

---

## 2.5 Batched Attribute Generation

论文没有一次让模型生成全部 50+ 属性，而是把属性集合分成 B 组，分批生成。实验中 `B = 4`。

动机：输出上下文越长，结构化生成越容易漏属性；分批后输出更短，completeness 更高。

代价：

- API/推理调用数增加；
- 输入 token 会重复；
- 成本会上升。

对 reference 解析任务，我们不需要 50+ 属性，因此不应该机械使用 B=4；但这个思想可以改成**分阶段、短输出**：

```text
Pass 1: 找出所有 identifier candidate
Pass 2: 判断 identifier role / ownership
Pass 3: 做 reference canonicalization + evidence validation
```

比“一次 prompt 同时完成品牌、型号、reference、SKU、序列号、配件关系、实体匹配”更容易控制错误。

---

# 3. 原论文结果与真正值得关注的风险

论文离线和在线结果很有生产意义。

## 3.1 成本与时延

论文报告 fine-tuned SLM 相比大模型 baseline：

```text
inference cost 约降低 98%
p90 latency 约降低 70%
```

这解释了为什么在千万级持续增量场景中，“专用小模型 + 少量 hard samples 持续微调”比“所有记录每次调用大模型”更有吸引力。

## 3.2 离线指标提升

Nova 2 Lite base completeness：

```text
66.03%
```

第 1 轮后：

```text
71.09%
```

Progressive 第 2 轮：

```text
74.20%
```

Cumulative 第 2 轮：

```text
73.72%
```

论文人工 audit 的 correct completeness 从 17.70% 提升到 20.50%。

## 3.3 在线分布漂移：这部分对当前项目比“74.2%”更重要

生产 A/B 中：

```text
Claude Sonnet 4.5 user acceptance: 88.3%
Nova 2 Prog user acceptance:       86.4%
```

但 production completeness：

```text
Claude: 63.0%
SLM:    59.1%
```

也就是说，离线 benchmark 上 SLM 74.2% > Claude 70.4%，到了线上排序反转。

论文明确归因为：

- 离线 benchmark 与线上流量存在 distribution shift；
- SLM 只用约 2K 样本 fine-tune，泛化面仍较窄；
- 后续应继续从生产弱点中挑样本 progressive fine-tuning。

当前三源二奢系统也会遇到完全相同的问题，而且风险更高：

```text
今天主要是 Rolex/Omega
下周新增冷门品牌
某个平台开始改标题模板
OCR 引擎升级
卖家把库存号放进“型号”字段
某来源字段语义发生变化
```

因此不能把一次离线 F1/precision 当成永久结论，必须把**数据漂移检测 + 持续难例回流**设计成一等公民。

---

# 4. 当前 Spec 不应该直接做“商品 pairwise matcher”

需求已经明确：

```text
same product = same reference number
```

因此推荐把系统主问题从：

```text
(a, b) -> same / different
```

重写为：

```text
record -> verified canonical reference
```

然后：

```text
entity_key = hash(canonical_brand_id + "\x1f" + canonical_reference)
```

最终匹配规则：

```python
def same_entity(a, b):
    return (
        a.entity_key is not None
        and b.entity_key is not None
        and a.entity_key == b.entity_key
    )
```

这样会带来三个巨大收益：

1. **复杂度从潜在 O(N²) pairwise 比较变成 O(N) record parsing + hash/index join**；
2. precision 的控制点从“神秘的相似度模型”变成可审计的 reference 证据链；
3. 模型只拥有“提出 candidate”的权限，最终实体合并由 deterministic verifier 决定。

论文的 progressive fine-tuning 最适合放在 `record -> reference candidate` 这一层，而不是最终 join 层。

---

# 5. 推荐的直接落地架构

## 5.1 总体架构

```text
               ┌─────────────────────────────┐
雷小安 -------->│                             │
腕表之家 ------->│ Raw Ingest / Change Feed    │
奢当家 -------->│                             │
               └──────────────┬──────────────┘
                              │
                              ▼
                  Source Schema Normalizer
                              │
                              ▼
                  Identifier Candidate Miner
             ┌────────────────┼────────────────┐
             │                │                │
       structured field    title/desc        image OCR
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                  Reference Extraction SLM
                    (LoRA, progressive)
                              │
                              ▼
                   Candidate + provenance
                              │
                              ▼
                Deterministic Reference Gate
          ┌───────────────┬───────────────┬───────────────┐
          │               │               │               │
       brand rule      role/ownership   ref registry    conflicts
          │               │               │               │
          └───────────────┴───────┬───────┴───────────────┘
                                  ▼
                         VERIFIED / ABSTAIN
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                 VERIFIED                    ABSTAIN
                    │                           │
                    ▼                           ▼
             canonical entity_key         Review Queue
                    │                           │
                    ▼                           ▼
          deterministic cross-source join   feedback log
                                                │
                                                ▼
                                      Hard-example Curator
                                                │
                                                ▼
                                      Progressive LoRA Loop
```

核心原则：

> **模型可以提议 reference，但不可以直接合并实体。**

---

# 6. 数据层设计

为了让整个系统可审计、可重放、可渐进训练，不要只在商品表加一个 `reference` 字段。建议至少保留以下结构。

## 6.1 `raw_product_record`

```text
record_id
source
source_item_id
raw_payload_uri
raw_title
raw_description
raw_brand
raw_reference_field
image_uris[]
ingested_at
source_updated_at
```

所有原始字段不可覆盖，后续算法升级时可以重放。

## 6.2 `identifier_candidate`

每一个可能是编号的字符串都独立存储：

```text
candidate_id
record_id
raw_token
normalized_surface
origin_type        # structured_field/title/description/ocr
origin_field
span_start
span_end
image_id
ocr_bbox
left_context
right_context
extractor_version
```

不要只保留“最终 candidate”。错误分析最需要的是 provenance。

## 6.3 `reference_prediction`

```text
record_id
brand_id_candidate
reference_raw
reference_canonical_candidate
identifier_role
ownership_label
model_score
model_version
prompt_schema_version
evidence_ids[]
decision_candidate
created_at
```

`decision_candidate` 只是模型意见，不是业务最终结论。

## 6.4 `reference_verification`

```text
record_id
brand_id
canonical_reference
verdict              # VERIFIED / ABSTAIN / CONFLICT
reject_reason[]
verification_rules[]
model_version
rule_version
registry_version
verified_at
```

## 6.5 `canonical_reference_registry`

```text
brand_id
canonical_reference
known_aliases[]
format_family
status              # trusted / observed / deprecated / blocked
source_provenance[]
verified_by
verified_at
```

这是整个 precision-first 系统最重要的资产之一。

## 6.6 `entity_assignment`

```text
record_id
entity_key
brand_id
canonical_reference
assignment_version
assigned_at
```

其中：

```text
entity_key = sha256(brand_id + "\x1f" + canonical_reference)
```

---

# 7. Reference Extraction SLM：如何把论文改造成腕表专用模型

## 7.1 输出不要只给一个字符串

推荐 schema：

```json
{
  "brand": {
    "raw": "劳力士",
    "canonical_id": "rolex"
  },
  "identifiers": [
    {
      "raw": "126610LN",
      "canonical_candidate": "126610LN",
      "role": "BRAND_REFERENCE_SELF",
      "origin": "title",
      "evidence": "劳力士潜航者 126610LN ..."
    },
    {
      "raw": "LX20260818001",
      "canonical_candidate": null,
      "role": "PLATFORM_SKU",
      "origin": "structured_field"
    }
  ],
  "record_type": "WATCH",
  "model_decision": "CANDIDATE_FOUND"
}
```

标签至少包含：

```text
BRAND_REFERENCE_SELF
PLATFORM_SKU
SELLER_INVENTORY_ID
SERIAL_NUMBER
COMPATIBLE_REFERENCE
ACCESSORY_TARGET_REFERENCE
OTHER_IDENTIFIER
UNKNOWN
```

最关键的是把：

```text
“这个字符串像 reference”
```

和：

```text
“这个 reference 属于当前售卖商品本体”
```

分开。

例如：

```text
原装表带，适配 Rolex 126610LN
```

`126610LN` 是真实 Rolex reference，但它不是当前“表带”商品的 reference。模型必须识别 ownership/record_type，否则 exact match 仍会产生灾难性 false positive。

---

# 8. 不要照搬论文的 Completeness Gap：改造成 Reference Risk Gap

论文的 Gap：

```text
catalog completeness - model completeness
```

在当前任务里应该换成面向 precision-first 的 hard sample score。

建议：

```text
RiskGap(x) =
    w1 * trusted_ref_exists_but_model_abstains
  + w2 * model_ref_conflicts_with_trusted_ref
  + w3 * multiple_reference_candidates
  + w4 * reference_near_collision
  + w5 * cross_source_transitive_conflict
  + w6 * source_field_vs_title_conflict
  + w7 * text_vs_ocr_conflict
  + w8 * unseen_brand_or_reference_pattern
  + w9 * human_rejected_prediction
```

其中最应该优先进入训练集的不是“容易的正确样本”，而是：

### A. 一字符近邻型号

```text
126610LN vs 126610LV
311.30.42.30.01.005 vs 311.30.42.30.01.006
```

这类错误对视觉/文本相似模型最危险。

### B. 真实 reference vs 平台 SKU

两者字符串形态都可能高度规则。

### C. 配件标题中的目标表 reference

```text
表带 / 表盒 / 保卡 / 表扣 / 保护膜 / 配件
```

它们经常包含主表型号，但不能因此归到主表实体。

### D. 多 candidate 冲突

标题一个 reference、结构化字段一个、OCR 又一个。

### E. 新来源、新品牌、新模板

这是线上 distribution shift 的主要来源。

---

# 9. Progressive Fine-Tuning 的生产化改造

论文原始 progressive 方式每轮只训练新 hard batch。对普通 listing generation 可行，但对“绝不能误匹配”的 reference 系统，建议增加**稳定锚点 replay**，避免模型在连续 hard batch 上逐步偏移。

推荐：

```text
M_t = FineTune(
    M_(t-1),
    HardBatch_t
    + AnchorReplay
    + CriticalNegatives
)
```

每轮数据可以类似：

```text
60% 当前最新 hard cases
20% 历史 verified positives anchor
20% 历史 catastrophic hard negatives
```

比例需要按验证集调，但思想很重要。

为什么不直接完全照搬论文“只训最新 1K”？

论文自己在 Limitations 中指出：

- progressive 只验证了两轮；
- 长期是否会积累 selection bias 尚未确认；
- 线上存在明显 distribution shift。

当前任务比论文更不能容忍长期漂移，因此必须保留稳定 replay set。

---

# 10. 训练样本怎么来：尽量把人工用在刀刃上

Spec 可接受“几百对人工黄金标签”，这个预算足以启动，但不能拿来随机标注。

## 10.1 Seed Gold Set

第一批 300–500 条建议分层抽样：

```text
品牌 × 来源 × 信息完整度 × record_type × reference难度
```

必须故意包含大量 hard negatives：

- 同品牌相邻 reference；
- 同系列外观近似不同 reference；
- 平台 SKU 与品牌 reference 混杂；
- accessory；
- 一个标题多个 reference；
- OCR 错一位；
- 中英文品牌别名；
- 连字符/点号/空格差异；
- 缺失 reference；
- 错误结构化字段。

## 10.2 Proxy Labels

可以低成本扩张训练集的来源：

1. 结构化 reference 字段长期稳定、且人工抽检通过的来源；
2. 已验证 canonical registry；
3. 多来源独立证据完全一致且无冲突的记录；
4. 人工 review queue 的修正记录；
5. 高可信 OCR + 标题双证据一致的样本。

但 proxy label 不能直接进入 evaluation benchmark。

## 10.3 Fixed Gold Benchmark

论文有独立 held-out benchmark；当前项目更应该如此。

固定 benchmark 永远不参与 hard-sample 选择，也不参与训练。按：

```text
source
brand
reference family
record_type
时间窗口
```

分层。

否则 progressive loop 会逐渐“刷熟”评测集附近分布，导致 precision 虚高。

---

# 11. Deterministic Reference Gate：precision 的真正保险丝

这是当前方案与原论文最大的不同。

论文里模型输出最终会由用户审核；当前系统如果自动合并实体，就必须把用户审核替换成严格的机器门禁。

推荐 gate：

```python
def verify_reference(pred, record, registry):
    if pred.role != "BRAND_REFERENCE_SELF":
        return ABSTAIN("not_self_reference")

    if pred.record_type != "WATCH":
        return ABSTAIN("non_watch_or_accessory")

    if pred.brand_id is None:
        return ABSTAIN("brand_unknown")

    ref = canonicalize_with_brand_rules(
        pred.brand_id,
        pred.reference_raw
    )

    if ref is None:
        return ABSTAIN("canonicalization_failed")

    if has_conflicting_high_trust_reference(record, ref):
        return CONFLICT("trusted_source_conflict")

    if looks_like_source_sku(record.source, ref):
        return ABSTAIN("source_sku_pattern")

    if not registry.accepts(pred.brand_id, ref):
        return ABSTAIN("unknown_reference_pattern_or_registry_miss")

    if insufficient_evidence(pred.evidence):
        return ABSTAIN("evidence_weak")

    return VERIFIED(ref)
```

最终 join **只接受 VERIFIED**。

任何以下情况都不要“猜一个最像的”：

```text
UNKNOWN
CONFLICT
MULTIPLE_CANDIDATES
LOW_EVIDENCE
OUT_OF_DISTRIBUTION
```

全部进入 abstain/review。

---

# 12. Canonicalization 不能简单“去掉所有符号”

一个常见但危险的做法：

```python
re.sub(r'[^A-Z0-9]', '', ref)
```

不建议全品牌通用。

原因：不同品牌 reference 的点号、斜杠、后缀、空格可能携带语义，粗暴删除可能把不同型号映射到同一字符串。

推荐品牌级规则：

```text
brand_id
normalization_rule_version
allowed_patterns
safe_separator_rules
alias_map
negative_patterns
```

规范化过程必须可逆、可审计：

```text
raw_reference
 -> normalized_surface
 -> canonical_reference
 -> applied_rules[]
```

任何“非显然安全”的变换都需要 registry/人工确认。

---

# 13. 图片怎么用：辅助确认，不越权决定实体

论文任务本身是 multimodal 的，这一点非常契合二奢。

腕表 reference 可能出现在：

- 表背；
- 保卡；
- 吊牌；
- 鉴定证书；
- 包装标签；
- 商品详情截图。

但是视觉相似度不能直接用于“同 reference”判定，因为同系列不同 reference 往往视觉极近。

推荐图片路径：

```text
image
  -> OCR / document region detection
  -> identifier candidates
  -> role/ownership classification
  -> text evidence cross-check
```

只有在以下场景里让图片提高可信度：

```text
title/structured ref == OCR ref
```

如果：

```text
text ref != OCR ref
```

应该变成 `CONFLICT`，而不是让某个模型融合出“平均答案”。

第一版甚至不需要全量 VLM：

- 全量商品走文本 + OCR；
- 只有 hard cases 走更昂贵的 multimodal model；
- 这样更接近论文“把昂贵能力转移到专用小模型”的成本逻辑。

---

# 14. 千万级数据的计算架构

## 14.1 Backfill

对历史 100 万–1000 万记录，不做全量 pairwise。

流程：

```text
Object Storage / Parquet Raw Data
       │
       ▼
Batch Normalize
       │
       ▼
Candidate Mining
       │
       ▼
Batch SLM Inference
       │
       ▼
Deterministic Verification
       │
       ▼
(entity_key, record_id)
       │
       ▼
Sort/Hash Group By entity_key
```

复杂度近似线性。

可以用 Spark/Ray 做 batch orchestration，模型推理用 vLLM/TGI/自研服务均可。具体框架不是关键，关键是：

```text
parser result 可重放
model_version 可追溯
rule_version 可追溯
```

## 14.2 增量

新记录到来后：

```text
ingest event
 -> parse reference
 -> verify
 -> if VERIFIED:
        lookup entity_key
        upsert assignment
    else:
        review/conflict queue
```

无需重新跑历史 N×M 比较。

当 reference parser 或 canonicalization rule 升级时，只重跑受影响的：

```text
brand/source/rule_version cohort
```

而不是全库重算。

---

# 15. 在线 serving 与训练建议

## 15.1 serving 分层

推荐三档：

### Tier 0：纯规则快路

如果来源有高可信结构化 `reference` 字段，并且：

```text
brand known
+ reference pattern valid
+ no conflicting candidate
```

直接走 verifier，不调用模型。

### Tier 1：专用 SLM

标题/描述中抽 reference，输出严格 JSON schema。

### Tier 2：昂贵 fallback

只对：

```text
multi-candidate
OCR conflict
new brand
new pattern
```

调用大模型/VLM或者进入人工 review。

这样能显著压低平均推理成本。

## 15.2 模型版本

建议：

```text
ref-parser-base
ref-parser-r001
ref-parser-r002
...
```

每一版都保存：

```text
base model
LoRA adapter
training sample ids
anchor replay version
prompt/schema version
metric report
registry version
```

这样出现 false positive 可以完整追责。

---

# 16. 评估指标必须从“F1”改成 precision-first 指标组

论文主要看 completeness，但当前任务必须换指标。

## 16.1 一级指标

### Auto-Match Precision

```text
正确自动匹配 / 所有自动匹配
```

这是首要指标。

### False Positives Per Million (FPPM)

千万级数据上，99.9% 看起来很高，但仍可能产生大量错误。因此建议同时报告：

```text
每百万自动合并中错误数
```

### Coverage

```text
VERIFIED / all records
```

precision 固定后才优化 coverage。

## 16.2 二级指标

```text
reference extraction recall
abstain rate
conflict rate
manual review yield
p90/p99 latency
cost / 1M records
```

## 16.3 分桶指标

必须按：

```text
source
brand
reference family
record_type
是否图片可用
是否结构化字段可用
时间窗口
```

分别统计。

全局 99.99% 不能掩盖某个新来源只有 97%。

---

# 17. 上线不能直接 A/B 自动合并：先 Shadow

论文的线上结果显示离线排名会因 distribution shift 反转。当前系统错误合并的代价更大，因此建议：

## Phase 0：Offline

只在固定 gold benchmark 上评估。

## Phase 1：Shadow

新模型实时跑，但不改变生产实体关系。

记录：

```text
old decision
new decision
conflicts
evidence
```

## Phase 2：Human-confirmed Canary

只在少量品牌/来源开启新模型，但所有新增 VERIFIED 仍抽检。

## Phase 3：Precision-gated Auto Merge

只有统计置信度和分桶 precision 达标的 cohort 才允许自动合并。

## Phase 4：Progressive Expansion

逐步扩大品牌/来源，持续监控 drift。

任何新 source schema 或新 reference family 默认回到 abstain/canary，而不是继承旧阈值。

---

# 18. 一个可以直接实现的训练循环

伪代码：

```python
model = load_model("ref-parser-r000")
anchor = load_anchor_replay()
critical_negatives = load_critical_negatives()

for round_id in range(1, N + 1):
    candidates = sample_recent_and_unresolved_records()

    preds = infer(model, candidates)

    scored = []
    for record, pred in zip(candidates, preds):
        score = reference_risk_gap(
            record=record,
            prediction=pred,
            trusted_registry=current_registry,
            review_history=review_history,
        )
        scored.append((score, record))

    hard_batch = top_k_diverse(
        scored,
        k=1000,
        group_by=["brand", "source", "reference_family", "record_type"],
    )

    labeled_hard_batch = attach_proxy_or_human_labels(hard_batch)

    train_set = mix(
        hard=labeled_hard_batch,
        anchor=anchor,
        negatives=critical_negatives,
        ratio=(0.6, 0.2, 0.2),
    )

    new_model = lora_finetune(
        init_from=model,
        train_data=train_set,
    )

    report = evaluate_on_fixed_gold(new_model)

    if precision_gate_passed(report):
        register_candidate_model(new_model, report)
        model = new_model
    else:
        reject_round_and_analyze_failures(round_id)
```

几个关键点：

1. `top_k_diverse` 防止一轮 1000 条全被某个热门品牌占满；
2. 继续从上一 checkpoint 训练，继承论文 progressive 思路；
3. 加 replay，降低长期 selection bias；
4. 每轮都必须通过固定 gold benchmark 的 precision gate；
5. 模型失败不会自动部署。

---

# 19. Reference Risk Gap 的一个可用初版

可以先不做复杂学习，用规则打分：

```python
def reference_risk_gap(record, pred, registry, reviews):
    s = 0

    trusted = trusted_reference(record)

    if trusted and pred.reference is None:
        s += 3

    if trusted and pred.reference and pred.reference != trusted:
        s += 10

    if len(pred.reference_candidates) > 1:
        s += 5

    if pred.text_ref and pred.ocr_ref and pred.text_ref != pred.ocr_ref:
        s += 8

    if is_near_collision(pred.reference, registry, edit_distance=1):
        s += 7

    if unseen_pattern(record.brand, pred.reference):
        s += 5

    if is_accessory(record) and pred.reference:
        s += 8

    if reviews.was_rejected(record.id, pred.reference):
        s += 12

    return s
```

之后再根据人工 review 的实际收益学习权重。

---

# 20. 直接可落地的 MVP

## Week 1：先把“确定性主干”搭起来

1. 三来源 source schema mapping；
2. 建立 brand canonical table；
3. 建立 reference candidate 抽取规则；
4. 建立 reference registry；
5. 品牌级 conservative canonicalization；
6. `entity_key = brand_id + canonical_reference`；
7. 所有无法确定的记录进入 abstain。

这一阶段甚至不需要训练模型，就可以先得到一批极高精度匹配。

## Week 2：建立专用 parser + gold benchmark

1. 选 300–500 条分层 hard cases；
2. 标注 identifier role、ownership、canonical reference；
3. 建固定 benchmark；
4. 用小模型结构化输出 reference candidates；
5. 只让模型补 Tier 0 规则覆盖不到的记录。

## Week 3+：引入 Progressive Loop

1. 扫描新数据和 abstain/conflict；
2. 计算 Reference Risk Gap；
3. 每轮挑 500–1000 个多样化 hard cases；
4. LoRA 从上一 checkpoint 继续训练；
5. 加 anchor replay + catastrophic negatives；
6. gold precision gate；
7. shadow/canary。

---

# 21. 与论文相比，当前方案必须做的六个关键改造

| 论文 | 当前二奢系统 |
|---|---|
| 优化 completeness | 优化 verified-reference precision |
| schema-valid 即可算完整 | reference 必须身份精确一致 |
| 用户最终审核所有建议 | 自动系统必须有 deterministic gate |
| 每轮只训最新 hard batch | progressive checkpoint + anchor replay |
| catalog attributes 作为 proxy label | trusted structured ref + registry + review feedback |
| 允许生成更多有效属性 | 宁可 abstain，不可猜错 reference |

这六点决定了能否真正满足“绝对不能误匹配”。

---

# 22. 这篇论文最值得拿走的三个结论

## 结论 1：不要为持续增量数据反复全量重训

持续从当前 checkpoint 学当前最难的新样本，比每次回到 base model 重训所有历史样本更符合固定算力预算。

对千万级二奢数据尤其重要：训练资源应该用于新模式/冲突/难例，而不是反复学习已经解决的 Rolex 常见 reference。

## 结论 2：生产反馈本身就是训练资产

系统中的：

```text
ABSTAIN
CONFLICT
人工修正
来源字段变化
新 reference pattern
```

都应该直接驱动下一轮 sample curation。

这比每个月“随机抽 1 万条重新标注”有效得多。

## 结论 3：小模型能降本，但必须尊重线上分布漂移

论文已经展示：离线优势在生产流量上可能反转。

因此对当前高风险匹配系统，模型升级的正确流程不是：

```text
offline F1 更高 -> 全量替换
```

而是：

```text
fixed gold precision
 -> shadow
 -> cohort canary
 -> drift monitoring
 -> progressive hard-case retraining
```

---

# 23. 最终推荐方案

针对 Notion Spec，我建议把这篇论文用于建设一个**持续演进的 reference 解析器工厂**，而不是直接复制其“属性生成模型”。

最终生产系统应为：

```text
                 Continuous Reference Parser Factory

trusted labels / review / conflicts / drift
                 │
                 ▼
        Reference Risk Gap Curator
                 │
                 ▼
       Progressive LoRA Fine-Tuning
        + Stable Anchor Replay
                 │
                 ▼
            Ref Parser SLM
                 │
                 ▼
       Deterministic Reference Gate
                 │
        ┌────────┴────────┐
        ▼                 ▼
     VERIFIED           ABSTAIN
        │                 │
        ▼                 └──> human / future training
brand + canonical ref
        │
        ▼
 deterministic entity_key join
```

最关键的一句话：

> **让 ML 持续提高“找到正确 reference 的覆盖率”，但永远不要让 ML 单独决定“两个实体可以合并”。**

这样才能同时利用论文的持续学习与低成本优势，并满足当前系统最苛刻的 precision-first 约束。

---

## 参考资料

1. Kolasani, Lakshman; Taheri Dezaki, Fatemeh. *Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce*. ACL 2026 Industry Track.  
   https://aclanthology.org/2026.acl-industry.40/
2. 论文 PDF：  
   https://aclanthology.org/2026.acl-industry.40.pdf
3. LoRA：Hu et al., *LoRA: Low-Rank Adaptation of Large Language Models*.  
   https://arxiv.org/abs/2106.09685
