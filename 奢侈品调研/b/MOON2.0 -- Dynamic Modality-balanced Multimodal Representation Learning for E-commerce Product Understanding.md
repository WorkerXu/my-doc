# MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding：把多模态模型降级为 Reference 证据增强与冲突否决层

- 分析人：b
- 调研文章：MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding
- 作者：Zhanheng Nie, Chenghan Fu, Daoze Zhang, Junxian Wu, Wanxian Guan, Pengjie Wang, Jian Xu, Bo Zheng（Alibaba Group）
- CVPR 2026：https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html
- arXiv HTML：https://arxiv.org/html/2511.12449
- arXiv：https://arxiv.org/abs/2511.12449
- MBE2.0：https://huggingface.co/datasets/ZHNie/MBE2.0
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 本次为什么选 MOON2.0

当前 Spec 的业务定义非常严格：

1. 数据来自雷小安、腕表之家、奢当家三个来源；
2. 数据量 100 万～1000 万级，且持续增量；
3. “同一个商品”严格定义为 **同一 reference number / 型号**；
4. reference 有时在独立字段，有时埋在标题、描述甚至图片中；
5. 图片可用；
6. **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
7. 可接受人工标注几百对作为黄金标签。

此前 `b` 已经分析过 AnyMatch、DeepBlocker、Ditto、TransClean、GraLMatch、pyJedAI、Confidence Classifier、Conformal Selective Prediction、Zalando 多模态商品匹配、属性抽取/规范化等方向，因此本次先排除这些已经给出过结果的条目。

MOON2.0 尚未在 `奢侈品调研/b/` 中分析。它值得补充的原因不是“又一个更强的多模态模型”，而是它正面处理了当前项目图片使用时最危险的三个问题：

- **模态不平衡**：有的记录只有标题，有的图片多、文字少，有的两者都有；
- **商品内部图文不一致**：标题可能写错、图可能是盒证/配件/参考图；
- **电商原始数据噪声**：伪正例、伪负例、营销词、背景、错误图片都会污染训练和判断。

这和三源腕表数据非常像。

但本文最重要的结论不是“把 MOON2.0 直接拿来做同款判断”，而是：

> **MOON2.0 的多模态表征适合作为候选召回、Reference 证据补全和冲突检测器；绝不能越权成为最终自动合并规则。最终自动合并必须由 canonical reference 的确定性证据收口。**

这是下面方案的总原则。

---

## 2. MOON2.0 原论文到底解决什么问题

MOON2.0 的目标是学习一个通用电商商品 embedding，使文本、图片、图文组合都能映射到统一空间，并支持：

- text -> multimodal retrieval；
- image -> multimodal retrieval；
- multimodal -> multimodal retrieval；
- text -> image；
- image -> text；
- 商品分类；
- 属性预测。

论文指出传统双塔模型通常是：

```text
image -> visual encoder -> image embedding
text  -> text encoder   -> text embedding

image/text embedding -> shared latent space
```

这种结构对“一张图对应一段文本”很自然，但电商商品经常是：

```text
一个商品
  ├── 主图
  ├── 细节图
  ├── SKU 图
  ├── 场景图
  ├── 标题
  ├── 描述
  └── 属性
```

即典型 many-to-one 多模态关系。

MOON2.0 用生成式 MLLM 统一编码这些输入，并专门处理：

1. image-only / text-only / multimodal 混合训练引起的模态失衡；
2. 只做跨商品对比而忽略“同一商品内部图文关系”的问题；
3. 电商标题、图片与行为日志本身存在的大量噪声。

论文当前 arXiv v3 为 2026-07-29 版本，CVPR 2026 正式收录。

---

## 3. MOON2.0 技术架构拆解

## 3.1 输入组织：同一个样本显式生成三种模态视图

对于训练三元组：

```text
(query, positive, negative)
```

MOON2.0 不只生成一个 multimodal 输入，而是为 q/p/n 分别构造：

```text
x^t  = text-only
x^i  = image-only
x^mm = image + text
```

因此模型不是被动接受数据里“刚好有什么模态”，而是在训练目标中显式要求：

```text
text query       -> multimodal positive
image query      -> multimodal positive
multimodal query -> multimodal positive
```

这对三源腕表非常重要，因为三个来源字段覆盖率必然不同。

如果直接把所有字段拼接训练一个 matcher，模型很容易学到某个平台的字段风格；而 MOON2.0 强迫模型在缺图、缺文或同时存在图文时都能产生可比较表示。

### 对腕表的迁移

可以把一条商品记录组织成：

```text
text view:
  brand + title + description + extracted_reference + structured attrs

image view:
  主图 + 表盘 + 表背 + 保卡/证书 + 吊牌/OCR crop

multimodal view:
  text view + selected images
```

但这里要加一条比论文更严格的规则：

> extracted_reference 不是普通文本 token，而是必须保留来源、抽取方式、置信度和原始证据位置的强结构化字段。

---

## 3.2 统一 MLLM Encoder

论文对文本进行 token embedding，对图片先经过 vision encoder 和 projector 得到 visual tokens，然后一起输入 LLM：

```text
text ----------------------> text tokens ---\
                                            \
image -> vision encoder -> projector -> visual tokens -> LLM -> hidden states -> mean pooling -> embedding
```

设：

```text
text token length = L
visual token count = V
LLM hidden dim = D
```

原始标题、增强标题、原图和若干增强图最后共同形成 hidden states：

```text
h ∈ R^((2L + (nc + 1)V) × D)
```

最后对 LLM 最后一层做 mean pooling：

```text
r = mean_pool(h)
r ∈ R^D
```

得到统一商品向量。

### 对腕表项目的实际取舍

这套完整 MLLM 训练成本很高。论文使用其内部电商 MLLM，在 64 张 NVIDIA A100 上训练约 18 小时，batch size 为每卡 4，学习率 1e-5。

因此 **MVP 不建议复现整套 MOON2.0**。

更实际的做法是：

```text
Phase 1:
  现成 text embedding + image embedding + OCR/reference rule

Phase 2:
  用几百对黄金标签训练一个轻量 evidence fusion / conflict classifier

Phase 3:
  如果图片确实能显著降低人工率，再训练腕表领域 multimodal encoder
```

不要一开始把项目变成 64-A100 级基础模型工程。

---

## 3.3 Modality-driven Mixture-of-Experts

这是 MOON2.0 第一个核心组件。

普通 MoE 在每个 FFN 层根据 token hidden state 做 expert routing：

```text
G = softmax(Wg h)
```

然后选择若干 expert MLP：

```text
h_hat = Σ Gz * fz(h)
```

MOON2.0 认为仅 token routing 还不够，因为它不知道当前 token 最终服务于哪个对齐任务。

例如：

```text
image query -> multimodal positive
text query  -> multimodal positive
image       -> paired text
```

这些目标对专家能力需求不同。

所以论文又定义一个可学习的：

```text
Dual-alignment Matrix W* ∈ R^(Z × M)
```

其中：

- Z = expert 数量；
- M = alignment objective 数量；
- W*[z,m] = expert z 对 objective m 的偏好。

再通过 softmax 得到：

```text
p[z,m]
```

并把 token routing 权重和 objective preference 合并成 objective-specific weight：

```text
omega_m
```

最后每个 alignment loss 的权重不是纯手工常量，而和 expert 当前对该模态任务的支持程度相关。

同时论文增加两个正则：

```text
L_aux      -> 防止 expert 使用极度不均
L_sparsity -> 让每个 expert 对少数 alignment objective 专门化
```

`sparsity loss` 本质上是压低 expert 的 objective preference entropy。

### 对腕表项目的迁移，不必真的上 MoE

MOON2.0 的真正启发是：

> **不同证据通道不能用一个固定线性加权公式粗暴相加。**

腕表里可以先做一个极简“业务版 modality routing”：

```text
if structured_reference exists:
    route = REFERENCE_STRONG
elif title_reference exists:
    route = TEXT_REFERENCE
elif OCR_reference exists:
    route = IMAGE_REFERENCE
else:
    route = NO_REFERENCE
```

然后每个 route 使用不同判定策略：

```text
REFERENCE_STRONG:
  canonical exact match + brand一致 -> 可进入自动候选

TEXT_REFERENCE:
  需要 reference grammar 校验 + 多字段一致性

IMAGE_REFERENCE:
  需要 OCR/VLM 双读一致或人工确认

NO_REFERENCE:
  只允许生成候选，不允许自动 merge
```

这在工程上相当于把 MoE 的思想先实现成 **规则驱动 evidence experts**，成本比训练稀疏大模型低几个数量级，而且更符合 precision-first。

---

## 3.4 Dual-level Alignment：最值得迁移的组件

MOON2.0 同时训练两种关系。

### 3.4.1 Inter-product Alignment

训练三元组：

```text
q = query
p = positive
n = negative
```

分别对：

```text
text q       -> multimodal p
image q      -> multimodal p
multimodal q -> multimodal p
```

做 contrastive learning，让正例近、负例远。

原论文核心形式可以理解为：

```text
L_inter = -log exp(sim(q,p)/tau)
                ------------------------------
                exp(sim(q,p)/tau)+Σexp(sim(q,n)/tau)
```

### 3.4.2 Intra-product Alignment

论文额外要求：

```text
同一个商品的 image ↔ text 应靠近
不同商品的 image ↔ text 应远离
```

也就是对同一个 product 内部再做一层 image-text contrastive alignment。

这一点在消融实验里非常关键。

论文 MBE2.0 的 R@10 消融结果里，完整模型与去掉 Alignment 的差距非常大，例如：

```text
完整 MOON2.0:
  text -> multimodal    63.09
  image -> multimodal   91.08
  multimodal -> mm      94.21
  text -> image         73.12
  image -> text         64.91

w/o Alignment:
  text -> multimodal    37.99
  image -> multimodal   65.72
  multimodal -> mm      67.45
  text -> image         31.45
  image -> text         23.35
```

这说明“商品内部图文一致性”不是装饰项，而是重要信号。

### 对腕表项目的关键迁移

我们可以把 intra-product alignment 重新定义得更业务化：

```text
标题里的 reference
      ↕
结构化 reference
      ↕
表背 OCR reference
      ↕
保卡 OCR reference
      ↕
品牌/系列视觉语义
```

目标不是让“整张图”和“整段标题”粗粒度接近，而是：

> **建立 Reference Evidence Alignment。**

即同一条记录内部，各证据源对 reference 的结论应该尽量一致。

例如：

```text
structured_ref = 126610LN
text_ref       = 126610LN
ocr_caseback   = 126610LN
```

强一致。

而：

```text
structured_ref = 126610LN
text_ref       = 126610LN
ocr_caseback   = 126710BLNR
```

应被当成严重冲突，哪怕图片整体看起来“都是劳力士 GMT”。

这就是多模态模型在 precision-first 系统里最有价值的角色：**不是证明两个商品相似，而是尽早发现证据不一致。**

---

## 3.5 MLLM-based Image-text Co-augmentation

MOON2.0 对训练数据做两类增强。

### 文本增强

论文先从标题、描述里抽取实体候选，再结合图片让 MLLM 生成更完整的 enriched title：

```text
T+ = MLLM_text(T, I, E)
```

其中：

- T：原始标题；
- I：图片；
- E：实体候选。

### 图片增强

先抽取主体，去除无关内容，再做：

- 背景变化；
- 视角变化；
- logo / 细节强化；
- 多粒度局部增强。

论文最终用 CLIP 检查 image-title consistency，低质量增强样本过滤掉。

### 对腕表项目的迁移

这里**不能直接生成“新表图”加入生产证据**。

生成式图片增强可能改变：

- 表圈刻度；
- 字盘颜色；
- 指针；
- logo；
- 小字；
- reference 刻字。

这些恰恰是不同 reference 之间最敏感的区别。

因此建议只把增强用于训练鲁棒性，而且设置不变量约束：

```text
允许：
  crop
  resize
  轻度亮度/对比度
  背景 mask
  轻度透视

谨慎：
  生成式背景替换

禁止作为 reference 证据：
  生成 logo
  重绘表盘文字
  生成表背刻字
  生成保卡编号
```

更推荐做 **evidence-preserving augmentation**，不是普通 generative augmentation。

---

## 3.6 Dynamic Sample Filtering

电商日志和弱标签很容易出现伪正/伪负样本。

MOON2.0 给每个 triplet 计算一个训练时可靠度：

```text
phi = sigmoid(kappa * ((sim(q,p) - sim(q,n)) - margin_bar))
```

论文固定 reliability threshold：

```text
delta = 0.6
```

并让 margin 随训练逐渐衰减：

```text
训练早期：只相信容易、高置信的样本
训练后期：逐步吸收困难样本
```

低可靠 triplet 被 down-weight，而不是一刀切删除。

### 对当前项目的直接价值

我们有大量可以自动构造的弱监督样本：

```text
positive:
  canonical reference 严格一致

hard negative:
  同品牌
  同系列
  reference 只差 1~2 个字符
  外观极其相似
```

但自动 reference 抽取本身也会错。

因此训练任何辅助模型时，绝不能把“抽取器结果”无条件当真值。

可以给样本定义 `label_reliability`：

```text
1.00  结构化 ref + 标题 ref + OCR ref 三者一致
0.95  两个独立证据一致
0.80  单一结构化 ref，通过品牌 grammar 校验
0.65  标题抽取 ref，通过 dictionary 校验
0.40  仅 OCR/VLM 单通道读出
0.20  仅 embedding 相似推测
```

然后训练 loss 乘该权重。

这比一次性硬构造百万“伪黄金标签”安全得多。

---

## 4. MOON2.0 不能直接用于当前 Spec 的地方

这一节非常重要。

## 4.1 原论文优化的是“语义相关”，我们定义的是“reference 相同”

MOON2.0 的正例来自购买行为和检索行为。

而当前业务定义是：

```text
same entity := canonical(reference_a) == canonical(reference_b)
```

两块同系列、同颜色、同尺寸、外观几乎相同的腕表，如果 reference 不同，就必须判不同。

因此：

```text
high multimodal similarity != same reference
```

图片相似度只能提供：

- candidate recall；
- conflict alert；
- 缺失 reference 时的人工复核排序；

不能提供最终 positive authority。

## 4.2 论文追求综合 retrieval/classification 指标，不是极端 precision

论文主要报告 R@k、accuracy、precision、recall、F1。

我们的线上核心 KPI 应该不同：

```text
AutoMerge Precision
False Merge Count
False Merge Rate
Abstain Rate
Human Review Rate
Reference Extraction Precision
Reference Conflict Detection Recall
```

最关键目标应该类似：

```text
P(false merge | auto_merge) <= 10^-5 ~ 10^-6
```

具体上线阈值由业务可承受风险和黄金集规模决定。

只看 F1 会把系统带偏。

## 4.3 图片应有“否决权”，但不应有“单独批准权”

建议权力设计：

```text
Reference exact evidence  -> 可以批准
Image conflict evidence   -> 可以否决/降级人工
Image similarity evidence -> 不可以单独批准
```

也就是：

```text
图片：negative authority > positive authority
```

这是本文对 Spec 最重要的工程改造。

---

# 5. 建议直接落地的总体架构

最终系统不要做成“一个大模型 API”，而是做成 **Reference Evidence Graph + Deterministic Gate + Multimodal Conflict Detector**。

整体链路：

```text
                 ┌─────────────────────────────┐
                 │ 雷小安 / 腕表之家 / 奢当家 │
                 └─────────────┬───────────────┘
                               │
                         ingest / CDC
                               │
                 ┌─────────────▼───────────────┐
                 │ 1. Canonicalization Layer   │
                 │ 品牌、标题、reference、URL   │
                 └─────────────┬───────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
  structured ref       title/desc ref          image OCR/VLM
          │                    │                    │
          └────────────┬───────┴─────────┬──────────┘
                       │                 │
              Reference Evidence Store  │
                       │                 │
                 ┌─────▼─────────────────▼─────┐
                 │ 2. Reference Resolver       │
                 │ canonical ref + confidence │
                 │ conflict state             │
                 └─────────────┬───────────────┘
                               │
                  ┌────────────▼─────────────┐
                  │ 3. Exact-ref Blocking   │
                  │ brand + canonical_ref   │
                  └────────────┬─────────────┘
                               │
                 candidate pairs / entity group
                               │
                  ┌────────────▼─────────────┐
                  │ 4. Deterministic Gate   │
                  │ brand/ref/hard conflict │
                  └───────┬───────────┬─────┘
                          │           │
                    pass candidate   reject
                          │
             ┌────────────▼────────────┐
             │ 5. Multimodal Conflict │
             │ Detector / MOON-lite   │
             └────────────┬────────────┘
                          │
                 ┌────────▼─────────┐
                 │ Decision Engine  │
                 ├──────────────────┤
                 │ AUTO_MATCH       │
                 │ REVIEW           │
                 │ NO_MATCH         │
                 │ ABSTAIN          │
                 └──────────────────┘
```

核心思想：

> **先解析 reference，再做 exact blocking；多模态放在 hard gate 之后做一致性审计，而不是先用 embedding 找“最像的商品”然后直接合并。**

---

# 6. 第一层：Reference Evidence Extraction

每条商品不要只保存一个 `reference` 字符串，而要保存“证据集合”。

建议表：

```sql
reference_evidence (
    product_id          bigint,
    source              varchar,
    evidence_type       varchar,   -- structured/title/description/ocr/vlm/manual
    raw_value           varchar,
    canonical_value     varchar,
    brand               varchar,
    image_id            varchar null,
    bbox                 jsonb null,
    text_span_start     int null,
    text_span_end       int null,
    extractor_version   varchar,
    confidence          double precision,
    grammar_valid       boolean,
    dictionary_hit      boolean,
    created_at          timestamp
)
```

一条商品可能是：

```text
product_id = 10086

structured:
  raw = "126610 LN"
  canonical = "126610LN"
  conf = 0.99

title:
  raw = "126610LN"
  canonical = "126610LN"
  conf = 0.96

ocr_caseback:
  raw = "126610LN"
  canonical = "126610LN"
  conf = 0.91
```

也可能存在冲突：

```text
structured -> 126610LN
title      -> 126610LN
ocr        -> 126710BLNR
```

不要在 extractor 阶段偷偷覆盖冲突，必须原样保留。

---

# 7. Reference Canonicalization

Canonicalization 只能处理“格式差异”，不能把“不确定的字符”强行修成同一个型号。

安全操作：

```text
Unicode NFKC
uppercase
去首尾空白
规范常见 separator
统一全角/半角
品牌特定可逆格式化
```

例如：

```text
"126610 LN"
"126610-LN"
"126610LN"
```

可规范到：

```text
126610LN
```

危险操作：

```text
O -> 0
I -> 1
B -> 8
S -> 5
```

这种 OCR correction 只能产生候选：

```text
raw_ref
candidate_ref
correction_reason
correction_confidence
```

不能直接改 canonical truth。

建议：

```text
canonical_ref_strict
canonical_ref_fuzzy_candidate
```

严格 matching 只使用前者。

---

# 8. Brand-specific Reference Grammar

腕表 reference 不是任意字符串。

应维护：

```text
brand_reference_rule
```

示意：

```yaml
brand: ROLEX
patterns:
  - regex: '^[0-9]{5,6}[A-Z]{0,6}$'
separators: [' ', '-', '.', '/']
allow_fuzzy_ocr: false
```

不同品牌单独维护 grammar、长度、字符集、前后缀和 known reference dictionary。

Reference Resolver 输出：

```json
{
  "canonical_reference": "126610LN",
  "status": "CONSISTENT",
  "confidence": 0.998,
  "support_count": 3,
  "evidence_types": ["structured", "title", "ocr"],
  "conflicts": []
}
```

或者：

```json
{
  "canonical_reference": null,
  "status": "CONFLICT",
  "confidence": 0.0,
  "conflicts": [
    ["structured", "126610LN"],
    ["ocr", "126710BLNR"]
  ]
}
```

---

# 9. Exact Reference Blocking：千万级主索引不需要先依赖向量库

如果“同商品 = 同 reference”是业务定义，那么最便宜、最可靠的 blocking key 就是：

```text
canonical_brand + canonical_reference
```

索引：

```sql
CREATE INDEX idx_product_reference
ON product_resolution(canonical_brand, canonical_reference)
WHERE resolution_status = 'CONSISTENT';
```

对于 100 万～1000 万数据，这种 B-tree/hash lookup 本身并不困难。

每来一条增量记录：

```text
1. parse evidence
2. resolve canonical ref
3. query same brand + same canonical ref
4. 只和这个 bucket 内的跨源记录建立候选关系
```

这样主流程从潜在的：

```text
O(N^2)
```

降为接近：

```text
O(N log N) ingest + O(bucket_size) match
```

向量检索只服务于：

- reference 缺失；
- OCR 模糊；
- 找可疑候选供人工确认；
- 发现疑似错误 reference；

而不是作为主 blocking。

---

# 10. MOON-lite：多模态冲突检测器，而不是 Matcher

建议把模型目标改成：

```text
输入：两个商品 + 各自 reference evidence
输出：是否存在“与 same-reference 假设冲突”的视觉/文本证据
```

即模型输出不是：

```text
P(same_product)
```

而是：

```text
P(conflict | reference_equal)
```

这会让任务定义和业务目标更加一致。

## 10.1 输入特征

### Reference features

```text
ref_exact
ref_source_count_a
ref_source_count_b
ref_confidence_a
ref_confidence_b
structured_ref_agree
ocr_ref_agree
brand_grammar_valid
```

### Text features

```text
brand
series
case_size
material
dial_color
movement
year
gender
set completeness
normalized title embedding
```

### Image features

```text
global image embedding similarity
main image similarity
caseback similarity
card/certificate OCR agreement
logo/brand visual consistency
fine-grained crop similarity
```

### Source/meta features

```text
source_a/source_b
seller/store
listing type
new/used/accessory
image count
missing-field mask
```

## 10.2 模态路由

不用一开始训练真正 MoE，可以先分专家：

```text
ReferenceExpert
TextAttributeExpert
ImageSemanticExpert
OCRExpert
AccessoryDetector
ConflictRuleExpert
```

最终 fusion：

```text
conflict_score = calibrated_model(expert outputs + missing masks)
```

重要的是必须显式输入 `missing mask`，否则缺失字段容易被模型当成“无冲突”。

---

# 11. 把 Dual-level Alignment 改造成 Reference-aware Dual Alignment

## 11.1 Inter-product：同 reference / hard negative

正样本：

```text
brand 相同
canonical_ref 相同
证据可靠度高
来源不同
```

hard negative：

```text
brand 相同
series 相同
reference 不同
编辑距离很近
图片相似
```

例如：

```text
126610LN vs 126710BLNR
116500LN vs 126500LN
```

这些才是项目最需要训练的负样本。

随机负样本几乎没有价值，因为：

```text
Rolex Submariner vs Cartier Tank
```

太容易。

## 11.2 Intra-product：Reference Evidence Alignment

对同一商品，构造：

```text
text reference ↔ structured reference
OCR reference  ↔ structured reference
image crop      ↔ OCR/reference token
title attributes ↔ product image
```

训练目标不是要求所有 modality 都像，而是要求关键 reference evidence 一致。

## 11.3 Contrastive training 数据格式

建议：

```json
{
  "anchor": {
    "product_id": 1,
    "brand": "ROLEX",
    "reference": "126610LN",
    "text": "...",
    "images": ["..."],
    "evidence": ["structured", "title"]
  },
  "positive": {
    "product_id": 2,
    "brand": "ROLEX",
    "reference": "126610LN"
  },
  "hard_negative": {
    "product_id": 3,
    "brand": "ROLEX",
    "reference": "126710BLNR"
  },
  "label_reliability": 0.99
}
```

---

# 12. Decision Engine：四态而不是二分类

不要只输出 match / non-match。

建议四态：

```text
AUTO_MATCH
NO_MATCH
REVIEW
ABSTAIN
```

## 12.1 AUTO_MATCH

建议必要条件：

```text
brand_strict_equal
AND canonical_ref_strict_equal
AND ref_confidence_a >= T_ref_high
AND ref_confidence_b >= T_ref_high
AND no_reference_conflict
AND no_accessory_conflict
AND multimodal_conflict_score < T_conflict_low
```

注意：最后一个多模态条件只负责“否决”。

即使 multimodal_similarity 很低，如果 reference 证据非常硬，也可能只是不同拍摄风格；可进入 REVIEW，而不要轻易 NO_MATCH。

## 12.2 NO_MATCH

确定性负规则优先：

```text
canonical_ref_a != canonical_ref_b
AND both ref confidence high
```

或者：

```text
brand conflict
```

或者：

```text
一个是腕表，一个明确是表带/盒/证书/配件
```

## 12.3 REVIEW

典型：

```text
same canonical ref
BUT image/OCR/attribute conflict
```

或：

```text
OCR ref 与 title ref 只有 1 字符差异
```

## 12.4 ABSTAIN

```text
reference 缺失
reference 只有低置信 OCR
证据不足
模型相似但没有 reference hard evidence
```

**ABSTAIN 不是失败，而是 precision-first 的核心能力。**

---

# 13. 推荐的决策伪代码

```python
def decide(a, b):
    # 1. 品牌硬冲突
    if a.brand_confident and b.brand_confident:
        if a.brand != b.brand:
            return "NO_MATCH", "BRAND_CONFLICT"

    # 2. 两边都有高置信 reference，但不相等
    if a.ref.high_conf and b.ref.high_conf:
        if a.ref.canonical != b.ref.canonical:
            return "NO_MATCH", "REFERENCE_CONFLICT"

    # 3. 没有足够 reference 证据，绝不自动批准
    if not (a.ref.high_conf and b.ref.high_conf):
        return "ABSTAIN", "INSUFFICIENT_REFERENCE_EVIDENCE"

    # 4. reference 相等后才进入视觉/文本冲突审计
    if a.ref.canonical == b.ref.canonical:
        conflicts = collect_hard_conflicts(a, b)
        if conflicts:
            return "REVIEW", conflicts

        conflict_score = multimodal_conflict_detector(a, b)
        if conflict_score >= T_REVIEW:
            return "REVIEW", "MULTIMODAL_CONFLICT"

        return "AUTO_MATCH", "REFERENCE_EXACT_AND_NO_CONFLICT"

    return "ABSTAIN", "UNRESOLVED"
```

这段逻辑有意让 AI 很难“批准”，但很容易“报警”。

这是符合 Spec 的。

---

# 14. 图片的实际处理方案

## 14.1 图片先分类，不要全部混在一起平均

腕表商品图片角色差异巨大：

```text
watch_front
watch_back
watch_side
clasp
movement
certificate
warranty_card
box
receipt
accessory
marketing/reference image
unknown
```

先训练/规则识别 image role。

然后不同 role 进入不同 extractor：

```text
watch_front -> global visual embedding + dial attributes
watch_back  -> OCR/reference/serial crop
certificate -> OCR/reference
warranty    -> OCR/reference
box         -> 只做辅助，不做强 positive
```

直接把 8 张图 embedding 平均，会把保卡/盒/腕表本体混在一个向量里，反而损失精细信号。

## 14.2 OCR 双通道

高风险 OCR 建议至少两条独立通道：

```text
OCR engine A
VLM constrained transcription B
```

只有：

```text
canonical(A) == canonical(B)
```

才提高图片 reference evidence 置信度。

否则：

```text
OCR_CONFLICT -> REVIEW/ABSTAIN
```

## 14.3 不要让视觉相似覆盖 reference 冲突

必须硬编码：

```python
if high_conf_ref_a != high_conf_ref_b:
    visual_similarity = ignored_for_positive_decision
```

不允许模型说：

```text
虽然型号不一样，但图片 0.99 相似，所以 merge
```

---

# 15. 数据与服务拆分

推荐拆成以下服务。

## 15.1 Ingest Service

职责：

- 接收三源增量；
- 分配全局 product_id；
- 保存 raw payload；
- 图片落对象存储；
- 发解析任务。

数据：

```text
product_raw
image_object
source_checkpoint
```

## 15.2 Canonicalization Service

职责：

- 品牌规范；
- 文本清洗；
- reference 候选抽取；
- strict canonicalization；
- brand grammar 校验。

## 15.3 Image Evidence Service

职责：

- image role；
- OCR；
- VLM transcription；
- image embedding；
- attribute extraction。

GPU 异步执行，不阻塞主 ingest。

## 15.4 Reference Resolver

职责：

把多个 evidence 聚合成：

```text
CONSISTENT
CONFLICT
INSUFFICIENT
```

以及：

```text
canonical_ref
confidence
provenance
```

## 15.5 Candidate Service

优先级：

```text
P0 exact brand+reference index
P1 OCR fuzzy candidate
P2 text/vector candidate
P3 image ANN candidate
```

P1/P2/P3 只能进入 REVIEW 或辅助诊断，不直接 AUTO_MATCH。

## 15.6 Decision Service

纯 deterministic gate + calibrated conflict model。

每次输出完整解释：

```json
{
  "decision": "AUTO_MATCH",
  "reason": "REFERENCE_EXACT_AND_NO_CONFLICT",
  "reference": "126610LN",
  "reference_evidence_a": [...],
  "reference_evidence_b": [...],
  "conflict_score": 0.012,
  "rule_version": "2026-08-18.1",
  "model_version": "conflict-v3"
}
```

保证可审计和可回滚。

---

# 16. Entity Group 结构

三源不是只做 pairwise link，最终应形成 entity group：

```text
entity_group
  group_id
  canonical_brand
  canonical_reference
  status
  created_at

entity_group_member
  group_id
  product_id
  source
  evidence_strength
```

约束：

```text
同一个 group 内所有高置信 canonical_ref 必须一致
```

如果新成员加入时出现：

```text
existing group ref = A
new strong ref     = B
```

禁止写入，生成 incident。

这相当于用数据库约束实现一层 TransClean/GraLMatch 类图一致性防线。

---

# 17. 人工复核界面应该展示什么

人工页不能只给“两张主图 + match/no match”。

应该按证据排序：

```text
[Reference]
A structured: 126610LN
A title:      126610LN
A OCR:        126610LN

B structured: 126610LN
B title:      126610LN
B OCR:        126710BLNR  <-- conflict

[Attributes]
brand       ROLEX == ROLEX
series      Submariner == Submariner
case size   41 == 41
dial        black == black

[Images]
front/front
back/back
certificate/certificate

[Model]
conflict_score = 0.87
reason = OCR_REFERENCE_CONFLICT
```

复核员操作：

```text
CONFIRM_SAME_REFERENCE
CONFIRM_DIFFERENT_REFERENCE
FIX_REFERENCE_A
FIX_REFERENCE_B
IMAGE_NOT_PRODUCT
ACCESSORY_LISTING
```

不要只收 match/non-match，因为最有价值的反馈通常是“哪条 evidence 错了”。

---

# 18. 几百对黄金标签怎么用最值钱

不要随机抽样。

建议 400～800 对按以下 strata 标注：

```text
25% same ref, easy
25% same brand+series, different near-neighbor ref
15% OCR one-char confusion
10% accessory / strap / box / certificate
10% title ref vs image ref conflict
10% source-specific dirty formatting
5% no-reference / impossible
```

尤其要覆盖：

```text
最容易 false positive 的边界
```

而不是追求总体分布代表性。

这些标签主要用于：

1. 校准 Reference Resolver；
2. 训练/校准 conflict detector；
3. 定 AUTO_MATCH threshold；
4. 建 precision 的统计置信区间；
5. 做回归测试。

---

# 19. Offline Evaluation：禁止只看 F1

建议主指标：

```text
1. AutoMatch Precision
2. False AutoMatch Count
3. AutoMatch Coverage
4. Review Rate
5. Abstain Rate
6. Ref Extraction Precision
7. Ref Conflict Detection Recall
8. Same-ref Candidate Recall
```

阈值选择方式：

```text
maximize AutoMatch Coverage
subject to:
  lower_confidence_bound(precision) >= business_target
```

如果黄金集中没有观察到 false positive，也不能简单声称 precision=100%。

必须报告置信区间。

例如可以用 Clopper-Pearson / Wilson 下界约束自动区。

这与当前“绝不能误匹配”的要求比 ROC-AUC 更匹配。

---

# 20. Shadow Mode 上线

第一阶段不要直接写 entity merge。

建议：

```text
生产数据 -> 新 pipeline -> shadow decision
                         |
                         +-> 不影响现网
```

每天抽查：

```text
AUTO_MATCH high-risk sample
REVIEW sample
reference conflict sample
```

累计足够证据后，再放开：

```text
source pair A-B + brand whitelist + high-confidence ref
```

然后逐品牌扩展。

不要全品牌一次上线。

---

# 21. 增量更新与幂等

每个商品生成：

```text
content_hash
reference_evidence_hash
image_set_hash
```

只有相关部分变化才重新计算。

例如：

```text
price changed:
  不需要 OCR / embedding

title changed:
  重跑 text reference extraction

image changed:
  重跑 image role/OCR/embedding

reference changed:
  必须撤销旧 group candidate 并重新 resolve
```

所有 match decision 记录：

```text
input_version
rule_version
model_version
```

方便在规则升级后重算受影响子集。

---

# 22. 建议技术栈

技术栈不限的情况下，优先选普通、可维护的组件。

## 数据与流

```text
Kafka/Pulsar            增量事件
Object Storage          图片/raw payload
PostgreSQL/ClickHouse   结构化 evidence / audit / analytics
Redis/RocksDB           热 reference lookup，可选
```

## 批处理

```text
Spark/Flink/Ray
```

用来做：

- 历史回填；
- 批量 OCR；
- embedding；
- 候选重算；
- golden evaluation。

## 模型服务

```text
Python + PyTorch
FastAPI/gRPC
GPU batch inference
```

## 向量索引

只有 secondary candidate discovery 才需要：

```text
FAISS / Milvus / pgvector
```

主 exact-reference lookup 不应依赖向量库。

---

# 23. 三阶段落地路线

## Phase A：2 周级可验证 MVP

目标：先证明 reference hard pipeline。

实现：

```text
structured/title reference extraction
strict canonicalization
brand grammar
exact brand+ref index
four-state decision
review queue
```

图片只做：

```text
OCR proof-of-concept
```

此阶段不训练 MOON-like 模型。

验收：

```text
AutoMatch Precision
coverage
reference extraction precision
```

## Phase B：加入图片 Reference Evidence

实现：

```text
image role classifier
caseback/card OCR
OCR/VLM 双通道 transcription
reference evidence conflict
```

目标：

```text
减少 title/structured ref 的错误自动 merge
```

即优先提升安全性，不先追求 coverage。

## Phase C：MOON-lite Conflict Detector

实现：

```text
text embedding
image embedding
reference features
hard-negative triplets
reference-aware dual alignment
reliability weighted training
calibration
```

模型只拥有：

```text
REVIEW / conflict escalation 权限
```

经过足够 shadow test 后，才允许它帮助减少 REVIEW，但仍不能覆盖 reference hard rule。

---

# 24. 一个更具体的训练方案

如果最终确实要训练 MOON-lite，可这样做。

## 24.1 样本生成

```python
for ref_group in high_conf_reference_groups:
    positives = cross_source_pairs(ref_group)

    negatives = nearest_reference_groups(
        same_brand=True,
        same_series=True,
        ref_edit_distance_le=3,
        image_similarity_high=True,
    )
```

目标是把最危险的“长得像但 reference 不同”放进训练集。

## 24.2 Loss

```text
L = L_inter_ref
  + lambda_intra * L_intra_evidence
  + lambda_conflict * L_conflict
```

其中：

```text
L_inter_ref:
  same ref closer, near-ref negative farther

L_intra_evidence:
  title/OCR/structured ref consistency

L_conflict:
  对明确冲突 pair 做 binary/ranking loss
```

每个样本再乘：

```text
label_reliability
```

吸收 MOON2.0 Dynamic Sample Filtering 的思想。

## 24.3 Calibration

训练模型后不要直接用 raw score。

用独立黄金验证集做：

```text
isotonic regression / Platt / temperature scaling
```

然后只基于 calibrated conflict probability 定 REVIEW 阈值。

---

# 25. 最关键的 hard-negative 库

建议专门维护：

```text
reference_hard_negative_catalog
```

字段：

```text
brand
ref_a
ref_b
series
visual_similarity
edit_distance
reason
confirmed_by
```

来源：

- 人工复核；
- 历史 false positive；
- same-series reference dictionary；
- ANN 找到的高视觉相似异 reference；
- OCR 常见混淆。

这是比“不断换更大模型”更有长期价值的数据资产。

---

# 26. 如何借鉴 Dynamic Sample Filtering 做持续学习

生产中每周收集：

```text
manual_confirmed_positive
manual_confirmed_negative
reference_fix
ocr_fix
model_conflict
```

生成训练池。

按可靠度分层：

```text
Tier 1: 人工确认 + 多证据一致
Tier 2: 高置信硬规则
Tier 3: 弱规则/模型伪标签
```

训练早期：

```text
Tier 1 + Tier 2
```

稳定后逐步加入 Tier 3 中模型 margin 足够大的样本。

这就是 MOON2.0 curriculum-like dynamic filtering 在本项目的工程化版本。

---

# 27. 风险清单

## 风险 1：reference 本身抓错

缓解：

```text
保留 provenance
多证据一致
图片 OCR 交叉验证
人工修正回流
```

## 风险 2：型号字符串其实是配件兼容型号

缓解：

```text
listing type classifier
context role classification
reference ownership validation
```

例如标题中的“适配 Rolex 126610LN 表带”不能被解析成该 listing 的商品 reference。

## 风险 3：二手表存在换盘/换圈/改装

这会造成：

```text
reference 相同，但视觉细节不一致
```

因此图片冲突更适合触发 REVIEW，而不是直接 NO_MATCH。

## 风险 4：同 reference 可能有年份/配色细分语义

需要和业务再次确认“same reference”是否始终等价于“same entity group”。

当前 Spec 明确定义同一 reference 即同商品，所以系统应按这个定义实现；其他属性只用于冲突告警，不改变主键语义。

## 风险 5：生成式增强破坏 reference 细节

缓解：

```text
训练增强限制在不改变文字/刻度/型号的操作
生成图永不作为线上硬证据
```

---

# 28. 与 MOON2.0 原方案的映射

| MOON2.0 | 当前项目落地 | 是否原样照搬 |
|---|---|---|
| Modality-driven MoE | Evidence routing / 专家通道 | 否，先规则化简 |
| Multimodal Joint Learning | text/image/mm 缺失鲁棒训练 | 是，后期使用 |
| Inter-product Alignment | 同 reference 跨源正样本 | 是，修改 label 定义 |
| Intra-product Alignment | title/structured/OCR/reference evidence 对齐 | 强烈建议 |
| Image-text Co-augmentation | evidence-preserving augmentation | 只部分采用 |
| Dynamic Sample Filtering | 伪标签可靠度加权 + curriculum | 强烈建议 |
| Unified embedding | ANN 候选/冲突检测 | 采用 |
| embedding similarity 做 match | 禁止直接用于 AutoMatch | 不采用 |

---

# 29. 最小可落地规则集

第一版甚至可以先不训练模型，只实现：

```yaml
AUTO_MATCH:
  - brand_strict_equal: true
  - canonical_ref_equal: true
  - both_ref_confidence: '>= 0.98'
  - evidence_count_each: '>= 1 high-quality'
  - reference_conflict: false
  - listing_type_conflict: false

REVIEW:
  - canonical_ref_equal: true
  - any_strong_conflict: true

NO_MATCH:
  - both_high_conf_ref_present: true
  - canonical_ref_equal: false

ABSTAIN:
  - otherwise: true
```

然后统计真实数据：

```text
多少数据直接进 AUTO_MATCH
多少卡在 ABSTAIN
哪些冲突最常见
```

再决定是否值得上 MOON-lite。

这比先造一个复杂大模型更稳。

---

# 30. 推荐的最终生产原则

整个系统应把“证据权力”按如下顺序排列：

```text
Level 0  人工黄金标签 / 官方可靠 reference
Level 1  多独立通道一致的 canonical reference
Level 2  单一高可信结构化 reference
Level 3  title/description 抽取 reference
Level 4  OCR/VLM reference
Level 5  text/image embedding similarity
```

自动 merge 只能主要依赖 Level 0～2，必要时接受经过严格校准的 Level 3。

Level 4～5 更适合：

```text
补充证据
候选发现
冲突报警
人工排序
```

这就是 MOON2.0 在当前项目中的正确位置。

---

# 31. 最终建议

MOON2.0 最值得借鉴的不是它本身的 SOTA embedding，而是四个系统设计思想：

1. **显式处理模态缺失与不平衡**，不要假设每个平台字段齐全；
2. **同时建模跨商品关系和商品内部图文关系**，尤其要把 Reference Evidence Alignment 做成一等公民；
3. **训练数据必须动态过滤噪声**，不能把弱抽取结果当硬真值；
4. **多模态模型应该服务于证据补全和冲突检测，不应覆盖 deterministic reference 规则。**

因此对当前 Spec，我推荐的最终架构不是：

```text
所有字段 + 图片 -> 大模型 -> same / different
```

而是：

```text
多源原始数据
    ↓
Reference Evidence Extraction
    ↓
Strict Canonicalization + Brand Grammar
    ↓
Exact brand + canonical reference Blocking
    ↓
Deterministic Precision Gate
    ↓
MOON-inspired Multimodal Conflict Detection
    ↓
AUTO_MATCH / REVIEW / NO_MATCH / ABSTAIN
    ↓
Entity Group + Audit Log
```

一句话总结：

> **Reference 决定“能不能匹配”，MOON2.0 类多模态模型决定“这次匹配有没有值得怀疑的地方”。**

在“绝对不能误匹配、允许漏匹配”的约束下，这比让多模态模型直接决定 same/not-same 更安全，也更容易在 100 万～1000 万级持续增量数据上工程化落地。

---

## 参考资料

1. MOON2.0, CVPR 2026 Open Access：
   https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html
2. MOON2.0 arXiv HTML：
   https://arxiv.org/html/2511.12449
3. MOON2.0 arXiv：
   https://arxiv.org/abs/2511.12449
4. MBE2.0：
   https://huggingface.co/datasets/ZHNie/MBE2.0
5. 需求 Spec：
   https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31
