# Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes

> 身份：b  
> 调研对象：Khandelwal et al., ACL 2023 Industry Track  
> 论文：**Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes**  
> 原文：https://aclanthology.org/2023.acl-industry.29/  
> 当前需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么选这篇

当前 Spec 的核心不是传统意义的“商品相似度”，而是一个 **reference number / 型号驱动的高精度实体匹配系统**：

- 数据来自雷小安、腕表之家、奢当家三个来源；
- 规模约 100 万～1000 万，且持续增量；
- “同一个商品”被定义为 **同一 reference number**；
- reference 有时是结构化字段，有时埋在标题、描述甚至图片里；
- precision 极端优先，宁可漏掉也不能误合并；
- 图片可用；
- 可接受几百对人工黄金标签。

这篇论文并不直接做 Entity Matching，而是做大规模电商 **多模态属性抽取**。它与当前问题的连接点非常直接：把 `reference number` 看成一个特殊的高价值商品属性，先从多源文本/图片中高精度抽取和规范化，再用严格相等规则完成实体匹配。

更重要的是，论文不是只给一个实验模型，而是给出了真实电商生产经验：

1. 单模型覆盖大量 `(product-type, attribute)` 组合；
2. 文本 + 图片联合抽取；
3. 支持文本中没有显式值时从图片补充；
4. 用 distant supervision 降低人工标注成本；
5. 实际部署时按属性寻找达到目标 precision 的阈值；
6. 作者明确遇到脏标签、归一化、长尾值、阈值标定等生产问题。

论文公开结果中，系统覆盖了超过 10K 的 PT-attribute 组合并抽取超过 150MM 属性值，说明其架构思想确实考虑了大规模生产，而不是小数据集 demo。

---

## 2. 原论文问题建模

论文把属性抽取改写为一个生成式 Question Answering 问题。

给定：

- 属性名 `a`，例如 `color`；
- 商品类型 `p`，例如 `dress`；
- 商品文本 `T`；
- 商品图片 `I`；

模型生成属性值：

```text
question = attribute_name
context  = product_type + product_text + product_image
answer   = attribute_value
```

如果迁移到腕表 reference：

```text
question = reference number
context  = watch + title + description + structured fields + images
answer   = 126610LN / 5711/1A-010 / 15500ST.OO.1220ST.01 / ...
```

但这里必须做一个重要改造：**原论文允许自由生成 attribute value，而我们的 reference 不能自由生成**。因为 reference 是标识符，只要多一个字符、少一个字符或把相邻型号混淆，就可能产生错误实体合并。因此后文会把论文的生成式输出改造成受限候选选择和证据验证。

---

## 3. MXT 的技术架构拆解

论文提出 MXT，整体可以拆成三段：

```text
                ┌──────────────────────────┐
                │ attribute + product type │
                │ + product text           │
                └────────────┬─────────────┘
                             │
                       T5 token embedding
                             │
Product Image ──ResNet-152──► MAG
                             │
                    image-aware text
                             │
                         T5 Encoder
                             │
                             ▼
                     text representation
                             │
Product Image ───Xception───► Cross Attention
                             │
                  attribute-aware fusion
                             │
                             ▼
                         T5 Decoder
                             │
                             ▼
                     attribute value
```

### 3.1 第一层：Image-aware Text Encoder

文本输入不是单纯商品描述，而是：

```text
[attribute name] + [product type] + [product textual description]
```

例如：

```text
attribute: sleeve type
product type: dress
text: Women's ...
```

输入经过 T5 embedding 后，论文使用预训练 ResNet-152 得到全局商品图片向量，再用 **MAG（Multimodal Adaptation Gate）** 把图像信息注入每一个文本 token 的表示。

其核心思想不是简单 concat，而是：每个 token 根据自身语义决定应该接收多少视觉信息。

论文中的门控过程可概括为：

```text
g_i = ReLU(W_g [T_i ; V] + b_g)
H_i = g_i * (W_H V) + b_H
T'_i = T_i + alpha * H_i
```

其中：

- `T_i`：第 i 个文本 token embedding；
- `V`：ResNet 图像 embedding；
- `g_i`：当前 token 对视觉信息的门控向量；
- `H_i`：视觉对该 token 的位移；
- `alpha`：限制视觉位移幅度，防止图像信息完全淹没文本。

这对腕表场景非常有意义。标题中类似：

```text
劳力士 黑水鬼 全套 2022 126610LN
```

有时卖家会同时写：

```text
适配 126610LN 表带
```

单看字符串都出现 `126610LN`，但图片内容可能是一条表带而不是腕表本体。视觉上下文可以帮助模型判断文本中的 reference 是“当前商品自身型号”还是“兼容/适配对象型号”。

不过视觉只能作为 **reference 归属验证信号**，不能直接把“长得像某型号”作为同款证据。

### 3.2 第二层：Attribute-aware Text-Image Fusion

MXT 不只使用 ResNet 的全局图片向量，还用 Xception 提取更局部、属性相关的视觉表示。

论文的直觉是：不同属性需要看图片不同区域。

- `sleeve type` 应看袖子；
- `neck style` 应看领口；
- 颜色可能看主体；
- 尺寸可能需要其他区域。

Xception 的 depthwise separable convolution 用于得到区域/通道相关视觉特征，再通过 multi-head cross attention 与已经经过 T5 Encoder 的文本特征融合。

对于腕表 reference，可以把这个思想改造成 **reference-aware visual attention**：

重点关注：

- 表盘文字；
- 表背刻字；
- 表耳/壳体刻字；
- 保卡型号区域；
- 吊牌；
- 盒证标签；
- 发票/鉴定卡中的型号区域。

但直接用 Xception 识别 reference 并不是最优生产选择。reference 是细粒度 OCR/字符识别问题，因此实际落地更适合：

```text
视觉 backbone / detector
        ↓
reference-related region proposal
        ↓
OCR / VLM OCR
        ↓
identifier candidate extraction
        ↓
品牌规则 + canonical dictionary 校验
```

论文的“attribute-aware region”思想保留，但视觉模块替换成更适合编号字符的 OCR + region detector。

### 3.3 第三层：T5 Decoder

MXT 最终用 T5 decoder 自回归生成属性值：

```text
P(y_j | y_<j, text, image)
```

训练目标是标准 token-level negative log-likelihood。

论文使用 greedy decoding。

对普通属性，例如颜色、版型，这种自由生成很好，因为可以输出训练集未见过的新值；论文也强调其 zero-shot attribute value 能力。

但对腕表 reference，**这是原论文最不能直接照搬的部分**。

原因：

- reference 是 ID，不是自然语言语义；
- `126610LN` 与 `126610LV` 只差一个字符但实体不同；
- `116500LN` 与 `126500LN` 在字符和外观上都高度接近；
- 模型生成一个“看起来合理但不存在”的 reference，是严重错误；
- 即使生成的是存在的 reference，也不意味着它属于当前商品。

因此 production 版不能让 decoder 自由生成，应改为：

1. **候选受限解码**；或
2. **候选 ranking / classification + NONE**；
3. 输出必须带 evidence provenance；
4. 无充分证据直接 abstain。

---

## 4. 论文训练与生产经验中最值得复用的部分

### 4.1 Distant Supervision

论文不依赖大规模人工逐条标注，而是用电商目录已有结构化属性值作为远程监督信号。

对于本项目可以直接利用：

- 平台已有 reference 字段；
- 标题中高置信 regex 抽出的 reference；
- 腕表之家标准型号页；
- 品牌/系列 reference 字典；
- 多来源中完全一致且结构字段可信的记录。

构造训练样本：

```text
输入：title + description + fields + image/OCR
标签：trusted_reference
```

这里不应该把所有平台字段都当真，而需要生成“可信弱标签”。例如：

```text
positive label =
    structured_ref passes brand grammar
    AND canonical_ref exists in reference dictionary
    AND title/OCR does not contain conflicting canonical ref
```

只用高可信部分训练，宁可样本少一些。

### 4.2 值归一化是生产瓶颈

论文部署时明确指出，远程监督数据中有大量 junk value，需要：

- heuristic merge similar values；
- trim tail values；
- 解决属性值不规范问题。

reference 场景更需要把 normalization 提升为一级模块。

建议 canonicalization 顺序：

```text
raw_ref
  ↓ Unicode / full-width normalization
uppercase
  ↓
去除无语义空格
  ↓
品牌特定 separator normalization
  ↓
品牌特定 prefix/suffix 规则
  ↓
reference dictionary exact lookup
  ↓
canonical_ref
```

关键原则：**不能做模糊纠错后直接匹配**。

例如：

```text
1266101N  --OCR 可能把 L 识别成 1
```

即使编辑距离 1 就能变成 `126610LN`，也不能自动改写并放行。正确方式是输出：

```json
{
  "raw": "1266101N",
  "candidate": "126610LN",
  "transform": "ocr_confusion_L_1",
  "status": "needs_secondary_evidence"
}
```

只有第二独立证据（例如标题明确出现 `126610LN`）一致时才提升可信度。

### 4.3 按属性/任务单独标定 precision 阈值

论文实际部署时没有使用一个全局 threshold，而是需要针对每个 PT-attribute 寻找达到所需 precision 的阈值。

这对本项目特别重要，应进一步细化到：

```text
(source, brand, evidence_type, extractor_version)
```

例如：

```text
(腕表之家, Rolex, structured_field, v1)
(奢当家, Rolex, title_regex, v2)
(雷小安, AP, image_ocr, v3)
```

每个桶单独统计 precision。

因为：

- 不同来源字段可信度不同；
- 不同品牌 reference 语法差异巨大；
- OCR 对纯数字、字母数字混合、斜杠型号的错误模式不同；
- 数据源页面模板更新会发生分布漂移。

### 4.4 用 Recall@固定 Precision，而不是只看 F1

论文使用 `Recall@90P` 做核心评估之一，这比纯 F1 更接近当前需求。

但当前 Spec 要求“绝对不能误匹配”，90% precision 远远不够。

建议评估指标改为：

```text
Recall@99.9P
Recall@99.99P
False Positive Per Million (FPPM)
Auto-Merge Coverage
Abstain Rate
```

对于自动合并层，目标不是最大化 F1，而是在高 precision 约束下最大化 coverage。

---

## 5. 结合 Spec 的推荐落地架构

### 5.1 总体原则

将论文的“多模态属性生成”改造成：

> **多模态 reference 候选抽取 + 受限规范化 + 证据验证 + exact canonical match**

最终匹配规则依然是：

```text
canonical_reference_A == canonical_reference_B
```

模型只负责帮助发现和验证 reference，不直接决定“两个商品是不是同一个”。

### 5.2 系统分层

```text
┌───────────────────────────────────────────────┐
│  Layer 0: Raw ingestion                      │
│  雷小安 / 腕表之家 / 奢当家，增量 CDC         │
└──────────────────────┬────────────────────────┘
                       ↓
┌───────────────────────────────────────────────┐
│  Layer 1: Source normalization                │
│  brand / title / description / fields / image │
└──────────────────────┬────────────────────────┘
                       ↓
┌───────────────────────────────────────────────┐
│  Layer 2: Reference candidate extraction      │
│  A. structured field                          │
│  B. regex / dictionary from text              │
│  C. image region OCR                          │
│  D. MXT-like multimodal verifier              │
└──────────────────────┬────────────────────────┘
                       ↓
┌───────────────────────────────────────────────┐
│  Layer 3: Canonicalization & role validation  │
│  brand grammar / ref dictionary / SKU-role    │
│  product-ref vs compatible-ref vs shop-SKU    │
└──────────────────────┬────────────────────────┘
                       ↓
┌───────────────────────────────────────────────┐
│  Layer 4: Evidence consensus / veto           │
│  agreement / conflict / provenance / abstain  │
└──────────────────────┬────────────────────────┘
                       ↓
┌───────────────────────────────────────────────┐
│  Layer 5: Exact reference entity resolution   │
│  brand + canonical_reference                  │
└──────────────────────┬────────────────────────┘
                       ↓
┌───────────────────────────────────────────────┐
│  Layer 6: Cluster consistency audit           │
│  cross-source contradiction / graph checks    │
└───────────────────────────────────────────────┘
```

---

## 6. Reference Extraction Service 设计

建议每条商品都产出一个结构化结果，而不是只返回字符串：

```json
{
  "item_id": "source:item-123",
  "brand": "Rolex",
  "candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "role": "PRODUCT_REFERENCE",
      "evidence": [
        {
          "type": "STRUCTURED_FIELD",
          "value": "126610LN",
          "source_field": "model_no"
        },
        {
          "type": "TITLE_TEXT",
          "value": "126610LN",
          "span": [18, 26]
        }
      ],
      "validators": {
        "brand_grammar": true,
        "dictionary_exact": true,
        "conflict": false
      },
      "decision": "TRUSTED"
    }
  ],
  "final_reference": "126610LN",
  "reference_status": "TRUSTED",
  "extractor_version": "ref-extractor-v1"
}
```

如果证据冲突：

```json
{
  "structured": "126610LN",
  "title": "126610LV",
  "decision": "ABSTAIN_CONFLICT"
}
```

**不要**通过模型分数二选一后自动放行。

---

## 7. 四类 reference 证据与优先级

建议建立有序证据层。

### Tier A：结构化 reference 字段

最高优先级，但仍需要验证：

```text
非空
AND 品牌语法合法
AND 不像平台 SKU
AND 若有 canonical dictionary，则 exact exists
```

不能因为字段名叫 `model` 就默认可信，有的平台可能混入系列名、库存号、内部货号。

### Tier B：文本显式 reference

标题、详情中用规则 + candidate dictionary 抽取。

推荐组合：

1. 品牌识别；
2. 品牌内 reference grammar；
3. dictionary trie / Aho-Corasick exact candidate scan；
4. 上下文 role classifier。

上下文 role classifier 重点区分：

```text
当前商品型号：Rolex 126610LN 黑水鬼
兼容型号：适配 Rolex 126610LN 表带
比较型号：相比 126610LN 新款...
库存/SKU：货号 126610LN-03291
```

### Tier C：图片 OCR reference

推荐做 region-first，而不是整图 OCR 后把所有数字都当 reference。

Pipeline：

```text
image
  ↓
region detector / VLM crop proposal
  ↓
OCR
  ↓
identifier candidate extractor
  ↓
brand grammar
  ↓
canonical dictionary
  ↓
OCR confusion validator
```

图片参考区域可以分：

```text
WATCH_CASE_ENGRAVING
WARRANTY_CARD
HANG_TAG
BOX_LABEL
INVOICE
OTHER_TEXT_REGION
```

不同区域可信度也应不同。例如保卡型号通常比背景中的网页截图可信。

### Tier D：MXT-like 多模态 verifier

这里真正复用论文架构。

输入：

```text
question: is <candidate_ref> the product's own reference number?
product_type: watch
brand: Rolex
title: ...
description: ...
image crops: ...
```

输出不再自由生成 reference，而是：

```text
OWN_REFERENCE
NOT_OWN_REFERENCE
UNKNOWN
```

或者候选 ranking：

```text
P(candidate_ref_i is own reference)
```

更安全的实现是三分类并强制保留 `UNKNOWN`。

这个 verifier 的职责仅限：

- 排除“兼容型号”；
- 排除“比较对象型号”；
- 排除“平台 SKU”；
- 在文本与图片之间做冲突检测；
- 在多个已抽取候选中给辅助排序。

它**没有权限创造新的 canonical reference**。

---

## 8. 把 MXT 改造成 Constrained-MXT

如果确实希望保留 T5 生成结构，可以做以下改造。

### 8.1 候选集合

先由确定性 extractor 得到：

```text
C = {c1, c2, ..., ck, NONE}
```

候选来源：

```text
structured field
+ title regex
+ description regex
+ OCR
+ same-brand reference dictionary retrieval
```

### 8.2 Decoder 不允许任意输出

方案 A：prefix constrained decoding

只允许 decoder 生成候选集合中存在的 token prefix。

方案 B：直接去掉 decoder，把 fused representation 与每个 candidate encoder 做点积/MLP ranking。

对于本项目更推荐方案 B：

```text
score_i = MLP([h_product ; h_candidate ; feature_i])
```

并显式加入：

```text
NONE / ABSTAIN
```

### 8.3 硬规则在模型外部

无论模型分数多高，以下情况硬拒绝：

```text
brand conflict
reference grammar invalid
canonical dictionary says impossible
two independent high-quality sources disagree
candidate role == COMPATIBLE_REFERENCE
candidate role == SHOP_SKU
```

这能保证模型永远不是最终的安全边界。

---

## 9. Entity Matching 层应极简

一旦每条记录都有可信 canonical reference，实体匹配不应继续堆复杂模型。

建议实体键：

```text
entity_key = canonical_brand_id + ':' + canonical_reference
```

例如：

```text
ROLEX:126610LN
PATEK:5711/1A-010
AP:15500ST.OO.1220ST.01
```

三源 record：

```text
雷小安:item_1 -> ROLEX:126610LN
腕表之家:item_9 -> ROLEX:126610LN
奢当家:item_31 -> ROLEX:126610LN
```

直接进入同一个 reference entity。

不要让图片 embedding、标题 embedding 再覆盖 reference 决策。

图片/文本相似度只用于：

- reference 缺失时辅助人工；
- reference 冲突时报警；
- 发现异常数据；
- 候选召回；
- 不自动合并。

---

## 10. 数据存储建议

### 10.1 商品原始表

```sql
product_record (
  source,
  source_item_id,
  crawl_time,
  raw_title,
  raw_description,
  raw_fields_json,
  image_urls,
  raw_brand,
  payload_hash,
  PRIMARY KEY(source, source_item_id)
)
```

### 10.2 Reference extraction 表

```sql
reference_extraction (
  source,
  source_item_id,
  extractor_version,
  canonical_brand_id,
  canonical_reference,
  status,
  confidence_bucket,
  evidence_json,
  conflict_reason,
  created_at
)
```

其中 `status`：

```text
TRUSTED
PROVISIONAL
CONFLICT
NO_REFERENCE
INVALID
NEEDS_REVIEW
```

### 10.3 Reference entity 表

```sql
reference_entity (
  entity_id,
  canonical_brand_id,
  canonical_reference,
  reference_family,
  metadata_json,
  UNIQUE(canonical_brand_id, canonical_reference)
)
```

### 10.4 Record-entity link

```sql
record_entity_link (
  source,
  source_item_id,
  entity_id,
  link_method,
  evidence_version,
  linked_at,
  PRIMARY KEY(source, source_item_id)
)
```

`link_method` 只允许：

```text
TRUSTED_EXACT_REFERENCE
HUMAN_CONFIRMED
```

不建议出现 `MODEL_SIMILARITY_AUTO`。

---

## 11. 千万级规模的在线/离线处理

当前规模不需要做任意两两比较。

错误方案复杂度：

```text
O(N²)
```

reference-first 后：

```text
extract each item: O(N)
hash/group by canonical reference: O(N)
```

推荐增量架构：

```text
Crawler / DB
   ↓ CDC / Kafka
Normalize Worker
   ↓
Reference Extract Worker
   ↓
Evidence Validator
   ↓
Reference Entity Upsert
   ↓
Conflict / Review Queue
```

每条新增记录独立处理，不需要重新扫描历史全部候选对。

如果 canonical reference 已存在：

```sql
SELECT entity_id
FROM reference_entity
WHERE canonical_brand_id = ?
  AND canonical_reference = ?;
```

命中即链接；否则创建新的 reference entity。

---

## 12. 几百对黄金标签应该怎么花

不要随机采样几百对普通 easy case。

建议把黄金标签集中在最可能制造 false positive 的 hard cases：

### 12.1 相邻 reference

```text
126610LN vs 126610LV
116500LN vs 126500LN
15500ST vs 15510ST
```

### 12.2 identifier role

```text
product reference vs compatible reference
product reference vs shop SKU
product reference vs serial number
product reference vs accessory target model
```

### 12.3 OCR confusion

```text
0/O
1/I/L
5/S
8/B
- / space / slash
```

### 12.4 来源冲突

例如结构字段 A，标题 B，图片 OCR C。

推荐 300～500 条先这样分：

```text
100：相邻 reference hard negative
100：reference role classification
100：OCR / 图片证据
100：跨源字段冲突
100：新增品牌/长尾/reference dictionary 外数据
```

用这批数据做：

- 阈值标定；
- rule regression test；
- model calibration；
- 版本发布 gate；
- 每日/每周 drift 检测基准。

---

## 13. Precision-first 决策矩阵

可以先用非常保守的规则上线。

| structured ref | title ref | OCR ref | dictionary | role | 决策 |
|---|---|---|---|---|---|
| A | A | A/空 | exact | own | AUTO TRUST |
| A | A | 空 | exact | own | AUTO TRUST |
| A | 空 | A | exact | own | AUTO TRUST |
| 空 | A | A | exact | own | AUTO TRUST |
| A | B | 任意 | - | - | CONFLICT |
| 空 | A | 空 | exact | own | PROVISIONAL / 高阈值后可 TRUST |
| 空 | 空 | A | exact | own | PROVISIONAL |
| A | A | B | exact | own | CONFLICT |
| 任意 | 任意 | 任意 | miss | - | REVIEW / NO LINK |
| A | A | A | exact | compatible | REJECT |

其中 A/B 表示不同 canonical reference。

第一版宁可只自动放行前 2～4 类，把覆盖率做低也没关系。

---

## 14. 为什么图片不能直接做实体合并

腕表场景中，同系列不同 reference 经常外观极近：

- 不同年份/机芯迭代；
- 盘面颜色或材质差异；
- 同壳型不同 reference；
- 同 reference 的不同拍摄环境反而差异很大。

因此：

```text
image similarity high
```

最多意味着“值得检查”，不能推出：

```text
same reference
```

论文中的多模态思想应该用在 **属性抽取和冲突否决**，而不是改变业务定义。

这也是将 MXT 放在 Reference Extraction 层，而不是 Entity Resolution 最终层的原因。

---

## 15. 品牌 Reference Grammar Registry

建议实现独立 registry：

```yaml
Rolex:
  patterns:
    - "^[0-9]{5,6}[A-Z]{0,3}$"
  separators: []

PatekPhilippe:
  patterns:
    - "^[0-9]{4}/[0-9A-Z]+(?:-[0-9]+)?$"
  separators: ["/", "-"]

AudemarsPiguet:
  patterns:
    - "^[0-9A-Z]+(?:\\.[0-9A-Z]+)+$"
  separators: ["."]
```

这里的 regex 仅示意，正式规则应通过真实 reference 全量字典反推和验证。

Registry 还应该保存：

```text
normalizer_version
known_prefixes
known_suffixes
forbidden_patterns
common_ocr_confusions
candidate_dictionary
```

所有规则必须版本化，否则后续无法解释为什么同一历史记录在不同时间得到不同 canonical reference。

---

## 16. 模型训练建议

如果要训练 MXT-like verifier，建议不要第一版直接训练一个庞大的端到端多模态生成器。

### Phase 1：确定性高精度 baseline

实现：

```text
brand normalization
reference dictionary
regex extraction
role keyword rules
OCR
conflict engine
```

先得到一个 precision 极高但 recall 较低的系统。

### Phase 2：文本 role classifier

训练：

```text
PRODUCT_REFERENCE
COMPATIBLE_REFERENCE
COMPARISON_REFERENCE
SHOP_SKU
SERIAL_NUMBER
UNKNOWN
```

输入 reference span 周围上下文。

### Phase 3：图片 OCR 与 region classifier

先解决图片中“有没有可靠型号证据”，而不是端到端识别整个商品型号。

### Phase 4：MXT-like multimodal verifier

只有 hard cases 才进入：

```text
candidate ref + title + description + image region
```

输出：

```text
SUPPORT
CONTRADICT
UNKNOWN
```

### Phase 5：calibration

对 verifier 输出进行 temperature scaling / isotonic calibration，最终由策略引擎按品牌和来源桶设置阈值。

---

## 17. 推荐的服务接口

### `/v1/reference/extract`

请求：

```json
{
  "source": "leidangjia",
  "item_id": "123",
  "brand": "Rolex",
  "title": "...",
  "description": "...",
  "fields": {},
  "images": []
}
```

响应：

```json
{
  "canonical_brand": "ROLEX",
  "final_reference": "126610LN",
  "status": "TRUSTED",
  "evidence": [],
  "conflicts": [],
  "extractor_version": "2026-08-18.1"
}
```

### `/v1/reference/verify`

请求候选：

```json
{
  "candidate_reference": "126610LN",
  "product": {}
}
```

响应：

```json
{
  "verdict": "SUPPORT",
  "reason_codes": [
    "TITLE_EXACT",
    "DICTIONARY_EXACT",
    "IMAGE_OCR_EXACT"
  ]
}
```

### `/v1/entity/resolve`

只接收 `TRUSTED` canonical reference：

```json
{
  "brand": "ROLEX",
  "reference": "126610LN"
}
```

返回固定 entity id。

---

## 18. 可观测性与防回归

每一次自动链接必须可追溯：

```text
raw data version
extractor version
normalizer version
brand grammar version
reference dictionary version
model checkpoint
threshold policy version
evidence list
```

核心 dashboard：

```text
AUTO_TRUST count
CONFLICT rate
NO_REFERENCE rate
source x brand coverage
reference dictionary miss rate
OCR disagreement rate
review overturn rate
false-positive incidents
```

最重要的是建立 false-positive kill switch：

如果某个：

```text
source + brand + extractor_version
```

出现确认误匹配，可以立刻停止该 bucket 的自动链接，回滚到 `PROVISIONAL`。

---

## 19. 与论文原方案相比，哪些保留、哪些舍弃

| MXT 原方案 | 本项目处理 | 原因 |
|---|---|---|
| 属性名作为 question | 保留 | reference 可视为专用属性 |
| 商品类型作为 prompt 条件 | 改成 brand + category + source | reference 语法高度品牌化 |
| ResNet 全局视觉信息 | 部分保留 | 用于商品/配件语义判断 |
| Xception 属性区域特征 | 思想保留，工程上换 OCR/region detector | reference 主要是字符证据 |
| T5 生成 attribute value | 不直接使用 | reference 自由生成风险太高 |
| Zero-shot value generation | 不用于自动链接 | 未知 reference 必须先验证 |
| Distant supervision | 强烈保留 | 可利用已有高可信 reference 字段 |
| 多 PT-attribute 单模型 | 保留思路 | 多品牌/多来源共享 extractor |
| per-attribute precision threshold | 强烈保留并细化 | precision-first 的核心生产机制 |
| 自动 catalog evaluation | 部分保留 | 需黄金标签校准，不能只信脏 catalog |

---

## 20. 最小可落地版本（MVP）

第一版甚至可以不训练多模态模型。

### Sprint 1：Reference 基础设施

1. 建 brand canonical mapping；
2. 导入品牌 reference dictionary；
3. 做结构字段 parser；
4. 做标题/描述 candidate extractor；
5. 建 normalization + role rules；
6. 输出 evidence provenance。

### Sprint 2：安全实体解析

1. exact `(brand, canonical_reference)` entity key；
2. conflict engine；
3. `TRUSTED / PROVISIONAL / CONFLICT / NO_REF` 状态机；
4. 只对 TRUSTED 做 auto link；
5. review queue。

### Sprint 3：图片证据

1. 图片分类：腕表本体/配件/盒证/保卡；
2. region OCR；
3. OCR candidate canonicalization；
4. 只作为支持或冲突证据。

### Sprint 4：MXT-like verifier

1. 采样 hard cases；
2. 训练三分类 `SUPPORT / CONTRADICT / UNKNOWN`；
3. 按 source × brand 做 precision calibration；
4. 只有达到极高 precision 的 SUPPORT 才允许把 `PROVISIONAL` 升级为 `TRUSTED`。

---

## 21. 直接给 Spec 的结论

这篇论文最值得借鉴的不是“用 T5 + Xception + MAG 端到端生成 reference”，而是一个更抽象、也更适合当前业务的生产模式：

> **把关键标识符作为专用属性，从结构字段、文本、图片中多路抽取；对每种属性/来源分别做高精度阈值标定；用弱监督扩大训练数据；模型负责补充证据，最终由确定性规则收口。**

对于三源二奢/腕表实体匹配，推荐最终架构为：

```text
多源商品
  ↓
reference 多路候选抽取
  ↓
品牌级 canonicalization
  ↓
identifier role 验证
  ↓
文本 / 结构字段 / OCR 多证据一致性
  ↓
高精度策略 + abstain
  ↓
brand + canonical_reference exact match
  ↓
跨源实体簇
```

最关键的安全边界有四条：

1. **模型不能自由生成 reference 后直接匹配**；
2. **图片相似度不能覆盖 reference 硬规则**；
3. **任何高质量证据冲突都必须 abstain**；
4. **自动合并只接受经过 canonicalization 与 role validation 的 exact reference**。

这样既使用了 MXT 的多模态属性抽取优势，又避免了生成模型在 identifier 上最危险的 hallucination / near-neighbor error，能够更贴合“precision 优先到极致、允许漏匹配”的 Spec。

---

## 22. 参考资料

- Khandelwal, A., Mittal, H., Kulkarni, S., & Gupta, D. **Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes.** ACL 2023 Industry Track. https://aclanthology.org/2023.acl-industry.29/
- 当前 Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
- `my-doc/奢侈品文章调研.md` 中该论文的推荐理由：工业级多模态属性抽取、12K PT-attribute、150MM 属性值、以固定 precision 下的 recall 评估，适合将 reference 作为高精度专用属性抽取。