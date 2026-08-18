# PAM: Understanding Product Images in Cross Product Category Attribute Extraction

> 身份：b  
> 对应调研条目：`https://arxiv.org/abs/2106.04630`  
> 论文：Rongmei Lin, Xiang He, Jie Feng, Nasser Zalmout, Yan Liang, Li Xiong, Xin Luna Dong. KDD 2021.  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 1. 结论先行

这篇论文值得参考，但**不应该把 PAM 原样当成最终实体匹配模型**。它真正适合本需求的部分，是放在实体匹配之前，作为一个“高精度 reference 抽取与多模态证据融合层”。

PAM 的核心思想是：把商品页面文本、图片中的 OCR 文本、图片视觉区域放进同一个多模态 Transformer，解码时不是无限制生成，而是从三类候选中选择 token：

1. 商品文本中的 token；
2. OCR 中识别出的 token；
3. 与商品类别/属性相关的动态词表。

同时它允许输出 `unknown`，并用商品类别约束动态词表，在 OCR/text 候选打分中加入编辑距离作为领域先验。

把这个思想迁移到腕表二奢场景后，最合适的架构不是“模型判断两个商品像不像”，而是：

**先对每条商品记录抽取并校验 canonical reference，再用 canonical reference 做严格等值匹配。图片只用来补全/交叉验证 reference，不能越过 reference 硬规则直接判同款。**

对于当前 Spec 的“绝对不能误匹配、precision 优先到极致、允许漏匹配”，建议直接落成：

```text
原始商品
  -> 多源 reference 候选抽取
  -> 候选角色识别（品牌型号 / 平台 SKU / 序列号 / 兼容型号 / 其他数字）
  -> 品牌/系列条件下的 canonical reference 归一化
  -> 文本 + OCR + 结构化字段证据交叉校验
  -> 高精度安全门（可拒识）
  -> canonical_ref_key
  -> 三源 exact join / grouping
```

实体匹配层本身因此不需要做 100 万～1000 万记录之间的全量 pairwise 相似度比较。只要 reference 抽取结果足够可信，主路径可以退化成 O(N) 的哈希分组/索引查询，既便宜又容易审计。

---

## 2. 为什么选择 PAM

当前需求最难的部分并不是“相似商品检索”，而是：

- reference 有时是独立字段，有时埋在标题；
- reference 还可能只出现在表背、保卡、吊牌等图片里；
- 标题里可能同时出现多个“像型号”的字符串；
- 腕表配件、表带、盒证、兼容描述中可能出现“被适配商品”的 reference，而不是当前售卖商品的 reference；
- 平台 SKU、店铺货号、序列号、reference 都是字母数字混合串，形态很接近；
- 同系列不同 reference 的表在图片上可能极其相似，因此视觉相似度不能成为最终匹配依据。

PAM 正好研究“文本信息不完整/含歧义时，如何让图片 OCR 和视觉区域补充并交叉验证商品属性”。论文报告，在其样本中，有一部分网页文本缺失的属性值只能从图片中识别出来；同时它展示了文本里出现多个候选值时，图片/OCR 可以帮助消歧。

这与当前需求里的 reference 抽取高度同构。

---

## 3. PAM 的技术实现拆解

### 3.1 总体架构

PAM 把属性抽取建模成 sequence-to-sequence generation，但它的输出空间是受约束的。

输入分成三路：

- **Product Text**：商品标题/描述；
- **Image Objects**：图片中的目标区域和视觉特征；
- **OCR Tokens**：图片里识别出的文字，以及 OCR token 对应的视觉/位置特征。

论文中的多模态 encoder/decoder 实际使用一个 Transformer，通过 attention mask 区分编码和解码计算。不同模态先投影到同一维度，再拼成统一 embedding sequence，所以文本位置可以直接 attend OCR 或图像区域。

论文实现参数：

- 4 层 Transformer；
- 12 attention heads；
- embedding dimension 768；
- batch size 128；
- Adam；
- base learning rate `1e-4`；
- 最大解码步数 10；
- 文本最多 100 tokens；
- 图像最多 10 个 object regions；
- OCR 最多 100 tokens。

### 3.2 文本表征

商品文本 token 通过 BERT 前三层得到 word-level embedding，训练时会继续 fine-tune。

PAM 没有只把标题当一个整体向量，而是保留 token 级表示，原因很重要：最终解码器可以直接从输入文本中 copy 候选 token。

迁移到 reference 抽取时，这比“把标题整体编码后做一个分类”更合适，因为 reference 往往是稀疏的字母数字串，例如：

```text
劳力士 宇宙计型迪通拿 116500LN 黑盘 全套 2020
```

我们真正关心的是 `116500LN` 这个局部 span，而不是标题的整体语义。

### 3.3 图像表征

论文使用 Faster R-CNN，ResNet-101 backbone，并在 Visual Genome 上预训练。每个检测框会产生：

- ROI visual feature；
- bounding box 四坐标。

视觉特征与坐标分别投影，再进行归一化。

对腕表场景而言，视觉 object feature 本身不应该直接判断 reference，但可以帮助判断 OCR token 的“所处区域是什么”：

- 表盘；
- 表背；
- 表扣；
- 保卡；
- 吊牌；
- 外盒；
- 平台截图；
- 水印/店铺 logo。

例如 OCR 读到 `116500LN`：

- 如果它位于保卡/吊牌区域，是强正证据；
- 如果它位于网页截图的“相关推荐”区域，应该降权；
- 如果它出现在表带包装上的“compatible with 116500LN”，则应标成兼容型号而不是当前商品 reference。

因此真正值得迁移的不是 Faster R-CNN 这个具体模型，而是“**OCR token 必须绑定视觉区域和坐标上下文**”这一设计。

### 3.4 OCR 表征

PAM 的 OCR token 不只使用 OCR 字符串，还组合：

- fastText 文本 embedding；
- PHOC 字符级特征；
- OCR 区域对应的视觉特征；
- OCR bounding box 坐标。

这些特征投影到同一维度再相加/归一化。

对 reference number 这是非常关键的，因为型号通常包含大量 OOV 字母数字组合，常规语言模型的语义 embedding 反而不一定有优势。

对于腕表 reference，更建议把字符级特征进一步强化：

```text
116500LN
126500LN
116503
116515LN
IW371604
WSSA0037
Q1368420
```

这些字符串的业务意义更多来自字符结构和品牌内编码规则，而不是自然语言语义。

### 3.5 受约束 token selection

PAM 每个解码步不会对整个自然语言词表做自由生成，而是从三种候选中选择：

```text
candidate_tokens =
    product_text_tokens
  + ocr_tokens
  + external_dynamic_vocabulary
```

解码器输出向量分别与三类候选打分，选择最高分 token。

这一点可以直接迁移成 reference candidate scoring：

```text
candidate_refs =
    structured_field_candidates
  + title_candidates
  + description_candidates
  + image_ocr_candidates
  + brand_reference_catalog_candidates
```

但在本项目中我建议进一步收紧：**最终 reference 不做 token-by-token 自由拼接，直接对“完整候选字符串”做排序/分类。**

原因是当前 Spec 的损失函数极不对称：生成一个不存在的 reference 是不可接受的 false positive。

### 3.6 动态词表

PAM 为每个 `(product category, attribute type)` 维护不同的 frequent value vocabulary，并显式加入 `unknown`。推理时只查询当前类别对应的词表。

这是整篇论文对本需求最有价值的设计之一。

迁移到腕表后应改成：

```text
V(brand, series?) = 该品牌/系列已知 canonical reference 集合
```

例如：

```text
Rolex -> {116500LN, 126500LN, 126610LN, ...}
Omega -> {...}
Cartier -> {...}
IWC -> {...}
```

当品牌/系列明确时，不让模型在全世界所有 reference 中搜索，而只在极小的品牌内候选空间中判断。

这会同时带来：

- 更低的误纠正风险；
- 更容易处理 OCR 字符混淆；
- 更容易做 hard negative；
- 更容易校准阈值；
- 更容易解释为什么模型选择了某个 reference。

`unknown` 也应该被保留，并且在当前需求中它不是异常，而是**一等公民输出**。

### 3.7 领域特定编辑距离

PAM 发现普通预训练 embedding 无法正确表达很多电商领域词，因此在 OCR/text token 打分上额外加编辑距离相似度。论文使用基于 FuzzyWuzzy 的 ratio，权重 `lambda=0.05`。

迁移到腕表时不要简单套普通 Levenshtein，而要做 **reference-aware edit model**：

高风险混淆：

```text
0 <-> O
1 <-> I <-> L
5 <-> S
8 <-> B
- / . / space
```

但注意：

**不能因为 `116500LN` 与 `126500LN` 编辑距离很近，就自动纠正。**

正确的使用方式是：编辑距离只负责“候选召回/打分”，最终自动接受必须有其他独立证据。

### 3.8 类别感知与多任务训练

PAM 尝试两种 category-aware 方式：

1. 把商品类别作为目标序列前缀一起解码；
2. 在 encoder 中加入 `<CLS>`，额外训练商品类别分类头。

两个任务和属性生成任务联合训练。

论文还指出，推理时如果真实商品类别已知，直接使用真实类别而不是模型预测类别，会提高 precision。

这个结论对我们很重要：

**只要上游已经有确定性信息，就不要让模型重新猜。**

因此本项目里优先级建议是：

```text
平台结构化品牌 > 高置信品牌词典解析 > 模型品牌识别
平台结构化 reference > 严格解析的标题 reference > OCR reference > 模型推断
```

模型应该填空，而不是覆盖确定性证据。

---

## 4. 论文结果中最值得注意的部分

PAM 在 14 个商品类别、61,308 个样本上实验；测试集使用人工标注值。论文把 `unknown` 明确作为不可观察属性值时的输出。

对于 Brand 属性，论文结果中：

```text
PAM text-only: P 81.2 / R 78.4 / F1 79.8
PAM full:      P 86.6 / R 83.5 / F1 85.1
```

对于 Item Form：

```text
PAM text-only: P 94.5 / R 60.1 / F1 73.4
PAM full:      P 91.3 / R 75.3 / F1 82.5
```

这同时说明两个事实：

1. OCR/图片能显著补 recall；
2. 多模态并不会天然提升 precision，某些属性甚至会用 precision 换 recall。

第二点决定了我们**不能把论文的完整模型原样作为自动 merge gate**。

当前 Spec 要的是 precision-first，因此多模态模型适合：

- 补充候选；
- 交叉验证；
- 冲突检测；
- 人工复核排序；

不适合仅凭一个高 softmax score 就自动合并两个商品。

论文的 ablation 还显示：去掉 OCR、图片或文本任一路，效果都会下降；动态词表与类别相关 target sequence 也都有收益。这支撑了“多证据 + 受限候选空间”的总体方向。

---

## 5. 对当前 Spec 的直接落地方案

## 5.1 不做 pairwise 商品相似度主链路，先做 reference evidence extraction

建议把系统拆成两个完全不同的阶段：

```text
Stage A: Record -> Trusted Canonical Reference
Stage B: Canonical Reference -> Cross-source Entity Group
```

### Stage A 输入

每条商品记录：

```json
{
  "source": "leixiaoan | xcar_watch | shedangjia",
  "source_product_id": "...",
  "brand": "...",
  "title": "...",
  "description": "...",
  "structured_reference": "...",
  "images": ["..."],
  "other_fields": {}
}
```

### Stage A 输出

不要只输出一个字符串，必须输出完整证据：

```json
{
  "brand_id": "rolex",
  "canonical_reference": "116500LN",
  "decision": "trusted | review | unknown | conflict",
  "confidence": 0.9997,
  "evidence": [
    {
      "type": "structured_field",
      "raw": "116500LN",
      "normalized": "116500LN",
      "role": "product_reference",
      "score": 1.0
    },
    {
      "type": "image_ocr",
      "image_role": "warranty_card",
      "bbox": [121, 88, 312, 121],
      "raw": "116500 LN",
      "normalized": "116500LN",
      "role": "product_reference",
      "score": 0.996
    }
  ],
  "conflicts": []
}
```

这个 evidence object 是整个系统可审计性的基础。

---

## 5.2 reference 候选抽取器

第一版甚至不需要训练大型模型。

### 结构化字段

直接解析平台已有 reference/model 字段：

```python
candidates += parse_structured_reference(record.reference)
```

这是最高优先级证据，但仍要做“字段语义是否可信”的平台级配置，因为某个平台名为 `model` 的字段可能实际存的是系列名。

### 标题/描述

对字母数字混合串做宽召回：

```text
[A-Z0-9][A-Z0-9./\-]{3,20}
```

然后再通过品牌规则和角色分类器过滤。

不要一开始就用非常严格的 regex，因为腕表品牌 reference 形态差异很大。

### 图片 OCR

只对“结构化/标题没有 trusted reference”或“存在冲突”的记录进入 OCR 重路径，以降低千万级数据成本。

图片先做 image-role 分类：

```text
watch_front
caseback
clasp
warranty_card
hang_tag
box
receipt
seller_watermark
web_screenshot
other
```

对 `warranty_card / hang_tag / caseback` 提高 reference OCR 权重；对 `seller_watermark / web_screenshot` 降权。

---

## 5.3 必须增加“编号角色分类”

仅仅抽出一个像 reference 的字符串远远不够。

每个候选必须判定：

```text
product_reference
compatible_product_reference
platform_sku
seller_sku
serial_number
year
price
size
movement_caliber
other_identifier
unknown_identifier
```

输入特征：

- 候选字符串本身；
- 候选前后 20～50 个字符；
- 字段来源；
- OCR 所在 image role；
- bbox 区域；
- 品牌；
- 是否存在 `ref / reference / 型号 / model / compatible / 适配 / for` 等上下文词；
- 是否命中品牌 reference catalog；
- 字符模式。

对当前需求而言，这个 role classifier 的价值可能高于一个通用商品 pair matcher。

---

## 5.4 canonical normalization 必须“保守”

目标不是把所有相似串归一到一起，而是只做已证明安全的格式变换。

安全例子：

```text
116500 LN -> 116500LN
116500-LN -> 116500LN
116500ln -> 116500LN
```

危险例子：

```text
116500LN -> 116500
116500LN -> 126500LN
Q1368420 -> 1368420
```

推荐接口：

```python
def normalize_reference(raw: str, brand_id: str) -> NormalizationResult:
    # 只做 brand-specific allowlist transform
    # 返回 transform trace，便于审计
    ...
```

每个品牌维护：

```yaml
brand: rolex
safe_transforms:
  - uppercase
  - remove_spaces
  - remove_hyphen_if_between_alnum
preserve_suffix: true
confusable_autocorrect: false
```

默认**禁止** O/0、I/1、S/5 的自动改写。

---

## 5.5 把 PAM 动态词表改造成“品牌 reference catalog”

建立：

```text
reference_catalog(
  brand_id,
  canonical_reference,
  series,
  aliases[],
  source,
  status,
  evidence_count,
  first_seen_at,
  last_seen_at
)
```

catalog 来源可以分层：

1. 官方/可靠目录；
2. 三个平台结构化 reference 的高置信聚合；
3. 人工审核加入；
4. 模型发现但未确认的候选，只放 shadow catalog，不参与自动纠错。

推理时：

```text
brand -> series(optional) -> candidate reference vocabulary
```

这就是 PAM dynamic vocabulary 的生产化版本。

---

## 5.6 推荐的 candidate scorer

不建议把完整 reference 当成自由生成任务，建议改成 candidate ranking：

```text
score(candidate_ref | record, evidence)
```

特征可以分四组：

### A. 确定性特征

```text
structured_exact
brand_catalog_exact
title_exact
ocr_exact
num_independent_sources
has_conflicting_reference
role_is_product_reference
```

### B. 字符特征

```text
length
alpha_digit_pattern
brand_pattern_valid
edit_distance_to_catalog
confusable_char_count
suffix_valid
```

### C. 上下文特征

```text
left_context_embedding
right_context_embedding
field_type
image_role
bbox_region
```

### D. 多模态交互特征

小型 cross-modal Transformer 或 late-fusion MLP 即可，输出候选分数。

如果训练样本只有几百条，我建议先使用：

```text
规则 + LightGBM/小 MLP + calibrated probability
```

而不是从零训练 PAM 那种 4-layer multimodal Transformer。

等人工复核累计到数万条候选级标签，再考虑更复杂模型。

---

## 5.7 precision-first 安全门

核心规则：**模型分数只是一项证据，不是最终决策。**

建议自动接受 `trusted` 的最小条件：

```python
AUTO_ACCEPT = (
    role == "product_reference"
    and canonical_ref is not None
    and brand_guard_passed
    and not conflicting_high_conf_reference
    and (
        structured_exact_and_valid
        or independent_evidence_count >= 2
    )
    and calibrated_score >= threshold_for_brand_source
)
```

其中 `independent_evidence_count >= 2` 可以是：

```text
title + warranty_card OCR
structured field + title
structured field + caseback OCR
title + hangtag OCR
```

而这些组合不能算真正独立：

```text
title + description（可能复制同一卖家文本）
同一张图 OCR 两次
OCR + 由 OCR 生成的 LLM 解释
```

### 冲突时一票否决

例如：

```text
structured: 116500LN
title:      116500LN
warranty:   126500LN
```

即使两个文本字段一致，也应该进入 `conflict/review`，不能自动 merge。

### 图片只能确认/否决，不能越权

```text
视觉看起来像 Daytona
```

绝不能把 `116500LN` 和 `126500LN` 合并。

图片视觉 embedding 最多做：

- 检测明显品类冲突；
- 判断 OCR token 所在区域；
- 给人工复核提供候选；
- 对已经相同 canonical reference 的记录做异常检测。

---

## 6. 三源实体匹配层如何实现

一旦 Stage A 得到 trusted canonical reference，Stage B 非常简单。

### 6.1 实体 key

建议：

```text
entity_key = brand_id + "::" + canonical_reference
```

虽然需求描述中“同一个商品”以 reference 为核心定义，但生产环境仍建议加 `brand_id` 做防碰撞 guardrail。品牌不同而 reference 字符串偶然相同，不能被哈希键直接合并。

### 6.2 数据库表

```sql
CREATE TABLE product_record (
  id BIGINT PRIMARY KEY,
  source SMALLINT NOT NULL,
  source_product_id TEXT NOT NULL,
  raw_payload JSONB NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  UNIQUE(source, source_product_id)
);

CREATE TABLE reference_extraction (
  record_id BIGINT PRIMARY KEY,
  brand_id TEXT,
  canonical_reference TEXT,
  decision SMALLINT NOT NULL,
  confidence DOUBLE PRECISION,
  extractor_version TEXT NOT NULL,
  evidence JSONB NOT NULL,
  conflicts JSONB,
  updated_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_reference_trusted
ON reference_extraction(brand_id, canonical_reference)
WHERE decision = 1;

CREATE TABLE product_entity_member (
  entity_key TEXT NOT NULL,
  record_id BIGINT NOT NULL,
  source SMALLINT NOT NULL,
  PRIMARY KEY(entity_key, record_id)
);
```

### 6.3 匹配

```sql
SELECT brand_id, canonical_reference,
       array_agg(record_id) AS members,
       count(DISTINCT source) AS source_count
FROM reference_extraction
WHERE decision = 1
GROUP BY brand_id, canonical_reference;
```

这避免了：

```text
雷小安 × 腕表之家 × 奢当家
```

之间的笛卡尔积。

1000 万条记录只需要对每条记录做一次 reference extraction，并对 trusted key 建索引/分组。

---

## 7. 增量架构

推荐事件驱动：

```text
Crawler / DB CDC
      |
      v
Raw Record Topic
      |
      v
Cheap Reference Extractor
  | trusted
  |------------------------> Reference Index -> Entity Grouper
  |
  | unresolved/conflict
  v
Image Fetch / OCR Queue
      |
      v
Multimodal Evidence Scorer
  | trusted ----------------> Reference Index
  |
  v
Review Queue / unknown
```

### 7.1 为什么分 cheap path / expensive path

千万级数据里，很多记录已经有结构化 reference 或标题里有明显型号，没必要全部跑图片模型。

建议分层成本：

```text
L0 结构化字段 + 正则 + catalog：毫秒级
L1 标题上下文模型：低成本 CPU/GPU batch
L2 OCR：只处理缺失/冲突记录
L3 多模态模型：只处理 OCR 后仍不确定的 hard cases
L4 人工复核：最难、最高风险
```

这样 OCR/视觉成本不会与总记录数线性等比例膨胀。

### 7.2 幂等和版本化

每条 extraction 记录必须保存：

```text
extractor_version
normalizer_version
catalog_version
ocr_version
policy_version
```

当模型/规则升级时，允许只重跑受影响记录。

---

## 8. 几百条黄金标签应该怎么用

Spec 允许人工标注几百对，但如果目标是“几乎不能错”，不要把这几百对全部拿去训练一个黑盒 pair classifier。

更高价值的标注方式是给 hard cases 做候选级标注：

```json
{
  "record_id": 123,
  "candidate": "116500LN",
  "label": "product_reference | not_product_reference",
  "reason": "title | warranty_card | compatible_accessory | sku | ..."
}
```

建议 300～500 条优先覆盖：

- 同系列相邻 reference；
- 一字符差异；
- O/0、I/1、S/5 OCR 混淆；
- 配件/表带兼容型号；
- 多型号标题；
- 平台 SKU 与 reference 混淆；
- OCR 与标题冲突；
- 不同来源字段 schema 差异；
- 新旧款 reference 共现。

### 统计上的现实

如果只有 300 个自动接受样本，哪怕 0 个错误，用常见的“rule of three”粗略估计，95% 置信下错误率上界仍约为：

```text
3 / 300 = 1%
```

所以几百标签不足以从统计上“证明 99.9%+ precision”。

因此“绝不能误匹配”不能只靠模型离线 precision 数字，而要靠：

- 确定性 hard gate；
- 多独立证据；
- 冲突一票否决；
- `unknown/abstain`；
- 线上持续抽检；
- 可回滚、可追溯 evidence。

---

## 9. 训练集与 hard negative 的构造

reference 抽取的 hard negative 不应该随机采。

推荐品牌内构造：

```text
116500LN vs 126500LN
126610LN vs 126610LV
IW371604 vs IW371606
```

再加入“非 reference 但长得像 reference”的负例：

```text
平台商品 ID
库存 SKU
序列号
机芯号
年份
尺寸
价格
订单号
适配商品型号
```

训练目标更应该是：

```text
正确 candidate 的分数 > 所有 hard negative + margin
```

而不是只对随机负例做 binary classification。

---

## 10. 一个可以直接做 MVP 的算法

```python
def resolve_reference(record):
    brand = resolve_brand_deterministically_first(record)

    candidates = []

    # 1. cheapest / strongest evidence
    candidates += extract_from_structured_fields(record, brand)
    candidates += extract_from_title(record.title, brand)

    # 2. role classification before normalization
    candidates = [
        classify_identifier_role(c, record, brand)
        for c in candidates
    ]

    # 3. conservative normalization
    candidates = [
        normalize_with_brand_allowlist(c, brand)
        for c in candidates
    ]

    # 4. if already decisive, no OCR needed
    result = deterministic_gate(candidates, brand)
    if result.is_trusted:
        return result

    # 5. expensive multimodal path
    image_roles = classify_images(record.images)
    ocr_candidates = run_ocr_on_useful_images(record.images, image_roles)
    ocr_candidates = bind_visual_context(ocr_candidates, image_roles)
    ocr_candidates = classify_identifier_roles(ocr_candidates, record, brand)
    ocr_candidates = conservative_normalize(ocr_candidates, brand)

    all_candidates = candidates + ocr_candidates

    # 6. PAM-inspired dynamic candidate vocabulary
    catalog = load_reference_catalog(brand, infer_series(record))
    scored = score_candidates(
        record=record,
        candidates=all_candidates,
        catalog=catalog,
        allow_unknown=True,
    )

    # 7. precision-first policy, not argmax
    return safety_policy(scored, all_candidates)
```

关键点是最后一步不是：

```python
return argmax(scored)
```

而是：

```python
return trusted only if evidence policy passes else unknown/review
```

---

## 11. 模型接口建议

```json
POST /v1/reference/resolve
{
  "source": "shedangjia",
  "brand": "Rolex",
  "title": "劳力士迪通拿116500 LN 黑盘全套",
  "structured_fields": {},
  "images": ["s3://.../1.jpg", "s3://.../2.jpg"]
}
```

返回：

```json
{
  "canonical_reference": "116500LN",
  "decision": "trusted",
  "confidence": 0.9996,
  "reason_codes": [
    "TITLE_EXACT",
    "OCR_WARRANTY_CARD_EXACT",
    "BRAND_CATALOG_EXACT",
    "NO_CONFLICT"
  ],
  "evidence": [...],
  "versions": {
    "extractor": "ref-extractor-0.3.1",
    "normalizer": "brand-rules-2026-08-18",
    "catalog": "catalog-2026-08-18"
  }
}
```

---

## 12. 线上策略与阈值

不要设一个全局 `score > 0.95`。

阈值至少按以下维度校准：

```text
brand
source platform
field combination
image role
是否 catalog exact
是否存在 OCR correction
```

例如：

```text
Policy A:
structured_ref exact + brand valid + no conflict
=> 可自动 trusted

Policy B:
title exact + warranty-card OCR exact + catalog exact
=> 可自动 trusted

Policy C:
只有 OCR，且通过一次 O/0 autocorrect 才命中 catalog
=> 不自动 trusted，进入 review

Policy D:
视觉非常像某 reference，但没有任何文字 reference
=> unknown
```

这比一个统一神经网络阈值更符合业务风险。

---

## 13. 对 PAM 需要明确舍弃的部分

### 13.1 不追求自由生成 unseen reference

PAM 的 sequence generation 可以生成未完整出现在输入中的属性值，这对普通属性抽取有价值。

本需求不适合。

reference 属于 identifier，应遵循：

```text
copy / catalog select / exact normalize > generate
```

新 reference 如果确实未进 catalog，可以作为 `candidate_new_reference` 输出，但必须进人工/高可信规则确认后才能进入 trusted catalog。

### 13.2 不以 F1 为主要指标

论文主要以 Precision/Recall/F1 衡量。当前项目应改为：

主指标：

```text
Precision@AutoAccept
False Merge Count
False Merge Rate
```

辅助指标：

```text
AutoAccept Coverage
Review Rate
Unknown Rate
Reference Extraction Recall
```

优化顺序：

```text
0 false merge > coverage > recall
```

### 13.3 不让图片相似度承担身份判断

论文里的 visual object feature 对颜色、形态等属性有效，但腕表同系列不同 reference 外观高度相似。

所以视觉信息的权限必须被限制：

```text
视觉 = 辅助证据 / OCR region context / conflict detector
reference exact evidence = 身份证据
```

---

## 14. 推荐的两阶段实施计划

### Phase 1：1～2 周可落地的 precision-first baseline

实现：

- 三平台字段 schema 梳理；
- 品牌 canonicalization；
- reference 候选正则/解析；
- brand-specific conservative normalization；
- identifier role rules；
- trusted/review/unknown/conflict 状态机；
- exact reference index/grouping；
- evidence JSON 与版本记录；
- 人工复核页面最小版本。

这一版甚至可以不使用视觉模型，就先测出：

```text
仅结构化字段 + 标题
```

能覆盖多少记录，以及 false-positive 主要来自哪里。

### Phase 2：PAM-inspired multimodal reference resolver

只对 Phase 1 未解决记录：

- 图片角色分类；
- OCR；
- OCR bbox 与 image role 绑定；
- 品牌 reference dynamic catalog；
- candidate scorer；
- hard-negative 训练；
- calibration；
- 多证据 auto-accept policy。

这样可以避免一开始就构建成本高、难调试的端到端多模态系统。

---

## 15. 最终推荐架构

```text
                         +----------------------+
                         | Brand/Reference      |
                         | Canonical Catalog    |
                         +----------+-----------+
                                    |
                                    v
+-------------+    +-------------------------------+
| 三源增量数据 | -> | Cheap Extractor               |
+-------------+    | structured/title/parser       |
                   +---------------+---------------+
                                   |
                  trusted ----------+---------- unresolved/conflict
                    |                              |
                    v                              v
          +------------------+          +------------------------+
          | Safety Gate L0   |          | Image Role + OCR       |
          +--------+---------+          +-----------+------------+
                   |                                |
                   |                                v
                   |                    +------------------------+
                   |                    | PAM-inspired Candidate |
                   |                    | Evidence Scorer        |
                   |                    +-----------+------------+
                   |                                |
                   |                       +--------+--------+
                   |                       | Safety Gate L1 |
                   |                       +---+---------+---+
                   |                           |         |
                   |                     trusted       review/
                   |                           |        unknown
                   +--------------+------------+         |
                                  |                      v
                                  v               +-------------+
                        +------------------+       | Human Review|
                        | Trusted Ref Index|       +------+------+ 
                        +--------+---------+              |
                                 |                        |
                                 +-----------+------------+
                                             |
                                             v
                                   +--------------------+
                                   | Exact Entity Grouper|
                                   | brand::reference    |
                                   +--------------------+
```

---

## 16. 最终判断

PAM 对本项目最大的启发不是“上一个多模态 Transformer”，而是三件事：

1. **不要只看商品标题；OCR 是真实商品属性的重要来源，而且必须和视觉位置上下文一起使用。**
2. **输出空间应该被品牌/类别先验约束，并且必须允许 `unknown`。**
3. **多模态的价值是补充信息和交叉验证，而不是让视觉相似度替代 identifier。**

因此我建议把 PAM 改造成一个 **PAM-inspired Reference Evidence Resolver**：

- 所有来源先抽 `reference candidate`；
- 所有候选先判 identifier role；
- 所有归一化都采用品牌级安全 allowlist；
- OCR/图片只负责补证和冲突检测；
- 最终只在多证据、无冲突、canonical reference 严格一致时 auto-accept；
- 其他记录全部 `abstain/review`；
- 匹配层只对 trusted `brand::reference` 做 exact join。

这个方案比直接训练 pairwise 商品匹配模型更符合当前 Spec 的定义，也更容易在 100 万～1000 万数据规模下稳定扩展和审计。

---

## 参考资料

- PAM 论文：https://arxiv.org/abs/2106.04630
- DOI：https://doi.org/10.1145/3447548.3467164
- 当前需求 Notion：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
