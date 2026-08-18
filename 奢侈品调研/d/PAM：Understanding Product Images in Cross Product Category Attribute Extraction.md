# PAM: Understanding Product Images in Cross Product Category Attribute Extraction

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择此前 **d 尚未分析** 的论文：

**PAM: Understanding Product Images in Cross Product Category Attribute Extraction**（Rongmei Lin et al., KDD 2021）。

- 论文：<https://arxiv.org/abs/2106.04630>
- DOI：<https://doi.org/10.1145/3447548.3467164>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

PAM 的核心思想非常适合当前任务中的 **reference number 抽取层**：

```text
商品标题 / 描述
       +
图片 OCR token + OCR 位置/视觉区域
       +
商品视觉区域
       +
按品类约束的候选属性词表
       ↓
多模态 Transformer
       ↓
受限候选 token 解码
       ↓
属性值
```

它最值得迁移的不是“多模态 Transformer 本身”，而是下面 4 个工程思想：

1. **把图片中的文字当作独立证据源，而不是只做整图视觉相似度**；
2. **输出不完全自由生成，而是在标题 token、OCR token、受限词表之间选择**；
3. **用动态候选词表缩小输出空间**，避免跨品类/跨领域乱猜；
4. **用多模态做交叉验证**：标题和图片同时出现同一属性值时，证据显著增强。

但 PAM **不能直接作为当前系统的最终 matcher**。论文的实验很能说明问题：对 `Item Form` 属性，PAM 从 text-only 的 `P=94.5 / R=60.1 / F1=73.4` 变成完整多模态的 `P=91.3 / R=75.3 / F1=82.5`——召回显著提高，但 precision 下降了 3.2 个百分点。当前 Spec 的要求却是：

> **绝对不能误匹配，precision 优先到极致，可以漏。**

因此正确迁移方式是：

> **把 PAM 改造成“多源 reference 证据抽取器 / 验证器”，而不是“同款判定模型”。最终自动 MATCH 仍然只能由 `canonical_brand + VERIFIED canonical_reference` 的严格等值和冲突门控决定。**

推荐的最终架构：

```text
雷小安 / 腕表之家 / 奢当家
        ↓
字段标准化 + 图片归档
        ↓
Reference Evidence Extractor
  ├─ structured reference field
  ├─ title / description parser
  ├─ image OCR parser
  ├─ reference-role classifier
  └─ brand-conditioned reference vocabulary
        ↓
Lossless Canonicalizer
        ↓
Evidence Aggregator
  ├─ VERIFIED
  ├─ CONFLICT
  └─ ABSTAIN
        ↓
只有 VERIFIED 才进入
(brand_id, canonical_reference) inverted index
        ↓
跨源精确分组 / 聚合
```

自动匹配规则应保持非常简单：

```text
record_a.reference_status == VERIFIED
AND record_b.reference_status == VERIFIED
AND record_a.brand_id == record_b.brand_id
AND record_a.canonical_reference == record_b.canonical_reference
AND no trusted conflicting_reference
=> MATCH

otherwise
=> ABSTAIN / CONFLICT
```

特别重要的一点：**PAM 原论文使用 edit-distance 作为候选 token 的补充打分，但在腕表 reference 上不能照搬到最终规范化/匹配环节。** 一位字符、一个后缀甚至一个连字符段都可能代表不同型号。编辑距离只能用于“候选召回 / OCR 错字提示”，绝不能把两个近似 reference 自动改成相同值。

---

## 1. 为什么这篇论文值得分析

当前需求表面看是 entity matching，实际上业务定义已经把问题收缩得很明确：

```text
same product := same reference number / 型号
```

所以核心难点不是学习一个“商品相似度”，而是：

1. 从高度不统一的字段中找到 reference；
2. reference 可能埋在标题、描述甚至图片中；
3. 图片里可能同时出现表背刻字、保卡编号、包装文字、店铺水印；
4. 同一个页面可能出现多个“看起来像型号”的字符串；
5. 同系列相邻 reference 通常长得非常像，不能模糊归并；
6. 需要在 100 万–1000 万级数据持续增量运行。

PAM 恰好研究的是：

> 从商品文本 + 图片视觉区域 + 图片 OCR 文本中抽取结构化属性值。

论文观察到，当网页结构化/文本信息缺失时，有超过一部分目标属性只能从图片中恢复；此外图片还可以帮助对标题中多个候选值做交叉验证。这与二手腕表非常同构：

- 标题没 reference，但保卡 / 吊牌 / 表背有；
- 标题里同时写系列名、机芯、尺寸、平台货号和品牌 reference；
- 图片能确认“这个编号确实属于当前商品”，而不是标题里的兼容/关联型号。

因此 PAM 对当前 Spec 最有价值的位置是 **reference acquisition**，也就是 entity resolution 之前的数据结构化层。

---

## 2. PAM 原论文的技术架构

## 2.1 输入不是简单“图 + 文”，而是三个可定位的信息源

PAM 把输入拆成：

```text
A. Product Text
   - title
   - bullet / textual profile

B. Image Objects
   - Faster R-CNN 检测区域
   - 每个区域的视觉向量
   - bounding box 坐标

C. OCR Tokens
   - OCR 字符串
   - OCR bounding box
   - OCR 对应图像区域视觉向量
```

另外还有一个非常重要但经常被忽略的输入：

```text
D. External Attribute Vocabulary
```

也就是从训练数据 / 类目知识中得到的“该属性可能有哪些值”。

这使 PAM 不是纯粹开放生成，而是一个 **copy + constrained vocabulary generation** 系统。

对腕表系统，可以一一映射：

```text
Product Text
=> title / subtitle / description / structured fields

Image Objects
=> 表盘、表背、保卡、吊牌、盒标、证书等区域

OCR Tokens
=> 图片中的型号、reference、序列文本候选

External Vocabulary
=> brand-conditioned reference dictionary / model catalog
```

这里最有价值的是最后一项。当前系统如果有品牌参考号知识库，就不应该让模型自由“生成一个看起来像 reference 的字符串”，而应该优先在该品牌的合法 reference 空间里进行受限识别。

---

## 2.2 文本表示

PAM 对商品文本使用 BERT 的前几层作为文本 encoder，得到 token-level embedding。

论文实现中：

- 文本最多约 100 token；
- BERT 参数参与 fine-tune；
- 输出被投影到与其他模态相同的 embedding dimension。

对当前任务，文本部分不必照搬 BERT。真正需要保留的是：

> **token-level 表示 + 原始字符串位置必须保留。**

原因是 reference 是 identifier，而不是语义概念。最终必须能够追溯：

```text
candidate = "126334"
source = title
span = [18, 24]
raw_context = "劳力士日志型 126334 蓝盘..."
```

模型得分只能作为证据，不能丢掉原始 span。

---

## 2.3 图像视觉表示

PAM 使用 Faster R-CNN：

```text
image
  -> object detector
  -> top object regions
  -> RoI feature
  + bounding box coordinate
```

论文实现使用 ResNet-101 backbone，并使用 Visual Genome 预训练检测器。

这套设计在 2021 年很合理，但对腕表 reference 的直接价值有限，因为 reference 本质上通常是字符串。整表视觉相似度容易产生最危险的一类误匹配：

```text
同品牌
同系列
同尺寸
同盘面布局
但 reference 不同
```

所以本项目不建议把通用视觉 embedding 当“正向匹配证据”。视觉部分应重新定义为：

```text
1. image-type routing
   - 表盘图
   - 表背图
   - 保卡图
   - 吊牌图
   - 盒证图

2. OCR region quality
   - 是否存在清晰文字区域
   - OCR 框是否位于可信区域

3. negative / conflict evidence
   - 图片明显是配件、表带、盒子而不是腕表本体
   - 图片中的 reference 与标题 reference 冲突
```

也就是说：

> **视觉用于找到“哪里值得 OCR”以及检测冲突，而不是用于绕过 reference 硬规则。**

---

## 2.4 OCR 表示是这篇论文最值得迁移的部分

PAM 对 OCR token 不只编码字符串，还同时保留：

- OCR 文本；
- OCR bounding box；
- OCR 区域视觉特征；
- 字符级特征；
- 词向量特征。

原论文使用 fastText 处理词级/OOV 表示，并使用 PHOC 一类字符级表达增强 OCR 文本鲁棒性。

对 reference number 来说，这一点比自然语言属性更加重要：

```text
116500LN
26240OR.OO.D002CR.01
IW500705
AB0121A11B1A1
```

这类字符串：

- 很多根本不在普通 tokenizer 词表里；
- 一个字符错就可能变成另一个 reference；
- OCR 常见 `0/O`、`1/I/l`、`.`、`-`、`/` 混淆；
- 子词语义 embedding 本身意义有限。

所以落地时应把 OCR reference 表示设计成：

```text
raw_string
normalized_string
char_sequence_embedding (optional)
ocr_confidence
bbox
image_type
image_id
candidate_pattern_id
brand_lexicon_hit
```

尤其不能只保存“模型最后预测的字符串”，必须保留原始 OCR token 和位置以便审计。

---

## 2.5 单一多模态 Transformer 融合

PAM 的 encoder / decoder 主体是一个多模态 Transformer。论文的设计不是为每个模态单独做一个最终分类器，而是把不同模态统一投影到相同维度，再拼成一个序列，让 self-attention 建立跨模态关联。

简化后可以理解为：

```text
[text token embeddings]
[image region embeddings]
[ocr token embeddings]
[previous decoder outputs]
        ↓ concatenate
multimodal transformer
        ↓
decoder state
```

论文实现的核心配置包括：

```text
transformer layers: 4
attention heads: 12
embedding dim: 768
batch size: 128
optimizer: Adam
base LR: 1e-4
max decoding steps: 10
```

这里的工程启示不是“我们也要训练 4 层 12 头”，而是：

> **标题候选、OCR 候选、图片位置和 reference 词典信息要在 record 内一起推理，而不是分别打分后粗暴平均。**

例如：

```text
标题：欧米茄海马 xxx 210.30.42.20.03.001 ...
OCR：21030422003001
图片类型：保卡
brand：OMEGA
```

系统应该知道 OCR 的无标点串与标题中的分段 reference 是同一个 canonical candidate；但如果 OCR 来自“配件兼容型号”区域，就应该降低其角色可信度。

---

## 2.6 Decoder 不是完全开放生成：三路候选 token

PAM 每一步 decoder 的 token 可以来自：

```text
1. product text token
2. OCR token
3. external attribute vocabulary
```

这本质上是 pointer/copy 与受限词表的结合。

对于 reference 系统，这是非常正确的方向。

我们不需要模型输出：

```text
"我认为它可能是 126334"
```

而要它输出：

```json
{
  "candidate_id": 17,
  "candidate": "126334",
  "selected_from": ["title_span_3", "ocr_span_8", "brand_lexicon"]
}
```

即从明确候选集合中选，而不是创造新的 identifier。

---

## 2.7 Dynamic Vocabulary：可以直接映射成品牌级 reference 词典

PAM 为不同 product category 使用不同候选属性词表，避免在一个超大公共词表中搜索。

对当前任务，category-conditioned vocabulary 应进一步改造成：

```text
brand-conditioned reference vocabulary
```

例如概念上：

```text
Rolex -> {valid Rolex references...}
Omega -> {valid Omega references...}
Cartier -> {...}
AP -> {...}
```

如果还有系列信息，可以进一步分层：

```text
brand
  -> family / collection
      -> reference candidates
```

收益有三个：

1. 降低 OCR 误识别后“误撞到别的品牌 reference”的概率；
2. 可以做格式合法性验证；
3. 可以把 OCR 的近似串只用于词典候选召回，再通过原始证据确认。

重要限制：

> reference 词典必须支持 `UNKNOWN / NEW_REFERENCE`，不能因为候选不在词典中就强制映射到最相似的已知 reference。

否则新型号会被错误吸附到旧型号，直接制造 false positive。

---

## 2.8 PAM 的 edit distance：只能借鉴一半

论文为了处理领域词和 OCR 候选，在 token selection 中引入候选与已知属性值的 edit-distance 相似度作为额外特征。

对自然语言属性值，例如：

```text
botanica / botanlca
```

这种策略合理。

但 reference 是 identifier，风险完全不同：

```text
126334
126333
```

编辑距离只有 1，但业务语义可能是两个不同型号。

因此本项目要严格区分两个阶段：

```text
candidate retrieval:
  可以使用 edit distance / character ANN

canonical decision:
  禁止 fuzzy merge
  必须回到可解释硬证据
```

推荐规则：

```text
OCR "12633A"
  -> fuzzy retrieve [126334, 126333, ...]
  -> 只是 candidates

如果没有标题 / 独立字段 / 第二图片 OCR 等证据消歧
  -> ABSTAIN

绝不能：
  nearest(reference_lexicon) -> 自动写成 126334
```

---

## 2.9 Multi-task category prediction 的迁移方式

PAM 使用两种方式把 category 信息注入模型：

1. 把 category 名放进目标序列前缀；
2. 加一个辅助 category classifier，与属性生成联合训练。

对腕表需求，最值得改成的辅助任务不是“商品大类”，而是：

```text
brand classification
reference-role classification
image-type classification
```

其中 `reference-role classification` 非常关键。

同一个商品页面中出现一个 reference-like 字符串，并不代表它是当前出售商品的 reference。它可能是：

```text
SOLD_ITEM_REFERENCE
COMPATIBLE_MODEL_REFERENCE
ACCESSORY_FOR_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
MOVEMENT_CALIBER
UNKNOWN_IDENTIFIER
```

如果不先判角色，单纯抽到正确字符也可能产生错误匹配。

因此 PAM 的 category auxiliary task 可以迁移成：

> **让模型同时回答“这个串是什么”与“这个串扮演什么编号角色”。**

---

## 3. 论文实验对当前 Spec 的真正启示

## 3.1 多模态提高了召回，但并不天然满足 precision-first

论文主要结果：

### Item Form

```text
PAM text-only : P 94.5 / R 60.1 / F1 73.4
PAM full      : P 91.3 / R 75.3 / F1 82.5
```

完整多模态提高了很多 recall / F1，但 precision 从 94.5 降到 91.3。

### Brand

```text
PAM text-only : P 81.2 / R 78.4 / F1 79.8
PAM full      : P 86.6 / R 83.5 / F1 85.1
```

Brand 上多模态同时提高 precision 与 recall。

所以不能笼统地说：

```text
加图片 => 更安全
```

正确结论是：

> **不同属性、不同图片类型、不同噪声分布下，多模态可能提高也可能降低 precision。必须把它作为证据，而不是最终规则。**

这正是当前系统应该保留 `ABSTAIN` 的原因。

---

## 3.2 OCR 的价值高于“纯视觉相似”的直觉

论文在 Item Form 上做模态消融：

```text
PAM w/o text  : P 79.9 / R 63.4 / F1 70.7
PAM w/o image : P 88.7 / R 72.1 / F1 79.5
PAM w/o OCR   : P 82.0 / R 69.4 / F1 75.1
PAM full      : P 91.3 / R 75.3 / F1 82.5
```

OCR 去掉后下降明显。

对腕表系统，这进一步支持：

> **优先投入高质量图片 OCR、区域检测和证据追踪，而不是先投入整图 embedding 商品相似度。**

如果预算只能优先做一件图片能力，建议顺序是：

```text
图片分类 / 区域定位
    -> OCR
    -> reference pattern parser
    -> OCR 与文本 cross-check
    -> 最后才是视觉语义 embedding
```

---

## 3.3 OCR 引擎质量直接影响最终效果

PAM 比较过不同 OCR 方案。论文中一个 OCR 服务平均提取更多 token，并获得明显更高 F1；另一个 OCR 方案由于检测出的 token 更少，最终表现更差。

当前系统要注意，这不意味着“token 越多越好”。二奢图片可能存在：

- 店铺水印；
- 价格标签；
- 其他商品背景；
- 保卡持有人信息；
- 盒子/配件编号；
- 防伪码；
- 序列号。

因此正确做法是：

```text
OCR recall 可以高
但 reference acceptance 必须极严
```

所有 OCR token 都可以保留，最终只把满足 reference pattern + role + multi-evidence 条件的候选提升为 VERIFIED。

---

## 4. 当前需求应该如何改造 PAM

下面给出一个可以直接落地的 **PAM-inspired Reference Evidence Architecture**。

## 4.1 总体数据流

```text
                ┌──────────────────────────┐
                │ 雷小安 / 腕表之家 / 奢当家 │
                └────────────┬─────────────┘
                             │
                             ▼
                   Raw Record Normalizer
                  /         |          \
             fields       texts       images
                │           │            │
                │           │            ▼
                │           │      Image Type Router
                │           │            │
                │           │            ▼
                │           │        OCR Service
                │           │            │
                └──────┬────┴──────┬─────┘
                       ▼           ▼
                  Candidate Extraction
                       │
                       ▼
              Reference Role Classification
                       │
                       ▼
              Brand-conditioned Lexicon
                       │
                       ▼
                 Lossless Canonicalizer
                       │
                       ▼
                  Evidence Aggregator
             ┌─────────┼──────────┐
             ▼         ▼          ▼
          VERIFIED   CONFLICT   ABSTAIN
             │
             ▼
      (brand_id, canonical_reference)
             │
             ▼
          Inverted Index
             │
             ▼
        Cross-source Groups
             │
             ▼
       Consistency / Audit
```

这里没有“大规模 pairwise matching model”。原因很简单：业务实体键已经由 reference 定义，只要 reference 被可靠抽取出来，就应该直接做 key-based resolution。

---

## 4.2 第一层：Raw Record Normalizer

统一三源数据为同一 schema：

```json
{
  "record_id": "source:123",
  "source": "leixiaoan | xxxxx | shedangjia",
  "title": "...",
  "description": "...",
  "brand_raw": "...",
  "reference_raw": "...",
  "structured_fields": {},
  "images": [
    {"image_id": "...", "url": "..."}
  ],
  "crawl_time": "..."
}
```

必须保存原始字段，后面所有规范化结果都不能覆盖原值。

---

## 4.3 第二层：候选 reference 抽取

候选来源按可信度分层：

```text
E0: 平台独立 reference / 型号字段
E1: 标题中识别出的 reference
E2: 描述 / 参数表中的 reference
E3: 图片 OCR 中识别出的 reference
E4: 模型/LLM 推断出的 reference candidate
```

E4 只能产生候选，不能单独 VERIFIED。

每个候选必须记录 lineage：

```json
{
  "record_id": "...",
  "candidate_raw": "210.30.42.20.03.001",
  "candidate_norm": "210.30.42.20.03.001",
  "brand_id": "omega",
  "source_type": "TITLE",
  "source_location": "title[12:29]",
  "ocr_confidence": null,
  "image_id": null,
  "image_type": null,
  "role": "SOLD_ITEM_REFERENCE",
  "role_confidence": 0.997,
  "lexicon_hit": true,
  "extractor_version": "ref-extractor-v1"
}
```

这样发生误匹配时，可以快速回答：

> 这个 reference 到底是从哪个字段 / 哪张图片 / 哪个字符 span 得来的？

---

## 4.4 第三层：图片路由 + OCR

不要所有图直接混在一起 OCR 后投票。

先做 image-type classifier：

```text
WATCH_FRONT
WATCH_BACK
WARRANTY_CARD
HANG_TAG
CERTIFICATE
BOX
ACCESSORY
OTHER
```

不同图片类型的 reference 证据权重不同，并且 role 不同。

例如：

```text
WARRANTY_CARD 上 reference 候选
  通常比普通详情图更可信

ACCESSORY 图上的 reference
  可能表示兼容对象，不一定是出售商品
```

OCR 输出必须包含 bounding box 和 confidence，不只要纯文本。

建议对 reference-like token 再做字符级二次识别：

```text
OCR full image
  -> detect candidate boxes
  -> crop candidate region
  -> second-pass OCR / char recognizer
  -> compare two OCR outputs
```

只有两次识别一致或有其他独立证据支持时，才提升置信等级。

---

## 4.5 第四层：Reference Role Classifier

这是从 PAM 迁移时必须新增的关键模块。

输入：

```text
candidate string
candidate context
field name
image type
nearby text
brand
```

输出：

```text
SOLD_ITEM_REFERENCE
COMPATIBLE_WITH_REFERENCE
ACCESSORY_TARGET_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
CALIBER
UNKNOWN_ID
```

初版甚至不需要大模型，可以先用规则 + 小分类器：

```text
"适用 / compatible / for / 表带适配"
=> COMPATIBLE_WITH_REFERENCE

字段名 "货号 / 商品ID / SKU"
=> PLATFORM/Seller SKU

字段名 "型号 / reference / ref"
=> SOLD_ITEM_REFERENCE 高先验
```

人工标注几百条时，应重点标这个任务，而不是只标“两个商品是不是同款”。

因为一旦 role 做对，后面的实体解析就非常简单。

---

## 4.6 第五层：Brand-conditioned Reference Lexicon

建立：

```text
brand_reference_catalog
```

结构至少包含：

```text
brand_id
canonical_reference
reference_aliases
format_pattern
collection(optional)
valid_from(optional)
valid_to(optional)
source
status
```

查询顺序：

```text
brand exact filter
  -> exact alias lookup
  -> normalized exact lookup
  -> fuzzy candidate retrieval ONLY
```

Fuzzy 结果永远不直接写入 canonical reference。

如果品牌词典中不存在一个高质量候选：

```text
NEW_REFERENCE_CANDIDATE
```

而不是强行映射到最近邻。

---

## 4.7 第六层：Lossless Canonicalization

reference normalization 必须是“尽可能无损”的。

安全操作示例：

```text
Unicode normalization
trim whitespace
统一大小写
统一全角/半角
把视觉等价分隔符转换成品牌明确允许的 canonical delimiter
```

危险操作：

```text
删除所有 '-' '.' '/'
随意去掉 leading zero
把 O 自动换成 0
把 I 自动换成 1
删除 suffix
只保留数字
```

推荐保留两份：

```text
reference_display_norm
reference_match_key
```

其中 `reference_match_key` 的生成逻辑必须按品牌 parser 版本化：

```text
canonicalizer_version = rolex-v3
canonicalizer_version = omega-v2
```

不要用一套全局 regex 处理所有品牌。

---

## 4.8 第七层：Evidence Aggregator

建议不输出一个 0~1 的统一概率后硬阈值，而输出状态机：

```text
VERIFIED
CONFLICT
ABSTAIN
```

### VERIFIED 条件（建议初版）

满足任意一条：

#### Rule A：可信结构化字段

```text
独立 reference 字段存在
AND brand parser 合法
AND role == SOLD_ITEM_REFERENCE
AND record 内不存在另一个高可信冲突 reference
```

#### Rule B：跨模态双证据

```text
title/description 中 reference = X
AND OCR 中 reference = X
AND brand lexicon / format 合法
AND role == SOLD_ITEM_REFERENCE
AND no conflicting X'
```

#### Rule C：两张独立高价值图片一致

```text
warranty card OCR = X
AND watch back / hang tag OCR = X
AND brand parser 合法
AND no conflict
```

### CONFLICT

任何高可信来源发生冲突：

```text
structured ref = X
OCR warranty card = Y
X != Y
```

直接 `CONFLICT`，不能让“多数投票”覆盖冲突。

### ABSTAIN

例如：

```text
只有一处低 confidence OCR
只有 LLM 推断
两个 fuzzy 候选都合理
reference 不在词典且格式异常
标题出现多个 reference 无法判 role
```

允许漏，正是当前业务的优势。

---

## 4.9 Entity Resolution：不要 pairwise，直接做倒排实体键

一旦 record 的 reference 变成 VERIFIED：

```text
entity_key = hash(brand_id + "\0" + canonical_reference)
```

建立：

```text
entity_key -> [record_id...]
```

这样 1000 万记录的主要成本是：

```text
O(N) reference extraction
+ O(N) index write / lookup
```

而不是：

```text
O(N^2) pair comparisons
```

即使有三个来源，也不需要对：

```text
雷小安 × 腕表之家
雷小安 × 奢当家
腕表之家 × 奢当家
```

分别训练 matcher。

统一抽成 canonical entity key 即可。

---

## 5. 一套可以直接实现的数据模型

## 5.1 product_record

```sql
product_record(
  record_id            string primary key,
  source               string,
  source_item_id       string,
  title_raw            string,
  description_raw      string,
  brand_raw            string,
  brand_id             string,
  structured_ref_raw   string,
  crawl_time           timestamp,
  payload_uri          string
)
```

## 5.2 reference_evidence

```sql
reference_evidence(
  evidence_id          string primary key,
  record_id            string,
  candidate_raw        string,
  candidate_norm       string,
  source_type          string,
  source_field         string,
  text_start           int,
  text_end             int,
  image_id             string,
  image_type           string,
  bbox                  json,
  ocr_confidence       float,
  role                  string,
  role_score            float,
  lexicon_hit           boolean,
  extractor_version     string,
  created_at            timestamp
)
```

## 5.3 reference_resolution

```sql
reference_resolution(
  record_id              string primary key,
  brand_id               string,
  canonical_reference    string,
  status                 string, -- VERIFIED/CONFLICT/ABSTAIN
  rule_id                string,
  evidence_ids           array<string>,
  conflicting_candidates array<string>,
  canonicalizer_version  string,
  resolver_version       string,
  updated_at              timestamp
)
```

## 5.4 entity_group

```sql
entity_group(
  entity_key            string,
  brand_id              string,
  canonical_reference   string,
  record_id             string,
  source                string,
  active                boolean,
  first_seen_at         timestamp,
  last_seen_at          timestamp
)
```

这样所有决策都可追溯。

---

## 6. 推荐技术组件

技术栈不是核心，下面是一个易于部署的组合。

### 存储

```text
Object Storage
  原始 JSON / HTML / 图片 / OCR crop

PostgreSQL / MySQL
  reference catalog、规则、人工复核状态

ClickHouse / BigQuery / Doris
  大规模 evidence / resolution 事件、离线分析

OpenSearch / Elasticsearch（可选）
  标题/OCR 候选检索与调试
```

### 流式/批处理

```text
Kafka / Pulsar
  增量事件

Flink / Spark
  大批量历史回填、规则版本重算
```

### OCR

优先选择支持：

```text
文字检测 + 识别
bounding box
confidence
旋转文字
小字符
```

的 OCR 引擎。

不要把 OCR 封装成只返回一大段 text 的黑盒 API。

### Reference Extractor

初版建议：

```text
brand-specific regex
+ reference lexicon trie/hash lookup
+ field-name rules
+ role rules/classifier
```

后续再加入 compact Transformer / multimodal reranker。

原因是：几百条黄金标注不足以稳定训练一个 PAM 规模的多模态生成器，但足够：

- 校准规则；
- 训练 role classifier；
- 训练 candidate reranker；
- 做 high-precision threshold validation。

---

## 7. 直接可落地的算法伪代码

## 7.1 record-level reference resolution

```python
def resolve_reference(record):
    brand = normalize_brand(record)

    candidates = []
    candidates += extract_structured_ref(record, brand)
    candidates += extract_text_refs(record.title, record.description, brand)

    for image in record.images:
        image_type = classify_image(image)
        ocr_items = run_ocr(image)
        candidates += extract_ocr_refs(
            ocr_items=ocr_items,
            image_type=image_type,
            brand=brand,
        )

    candidates = classify_reference_roles(candidates, record)
    candidates = canonicalize_losslessly(candidates, brand)
    candidates = attach_brand_lexicon_hits(candidates, brand)

    trusted = [c for c in candidates if c.role == "SOLD_ITEM_REFERENCE"]

    conflicts = find_trusted_conflicts(trusted)
    if conflicts:
        return Resolution(status="CONFLICT", conflicts=conflicts)

    # 独立结构化字段
    strong_field = unique_strong_structured_candidate(trusted)
    if strong_field:
        return Resolution(
            status="VERIFIED",
            reference=strong_field.canonical_reference,
            rule_id="R_STRUCTURED_REF"
        )

    # 文本 + OCR 独立证据一致
    x = same_reference_across_independent_channels(trusted)
    if x:
        return Resolution(
            status="VERIFIED",
            reference=x,
            rule_id="R_CROSS_MODAL_AGREE"
        )

    # 两张独立可信图片一致
    x = same_reference_across_high_value_images(trusted)
    if x:
        return Resolution(
            status="VERIFIED",
            reference=x,
            rule_id="R_MULTI_IMAGE_AGREE"
        )

    return Resolution(status="ABSTAIN")
```

## 7.2 cross-source matching

```python
def entity_key(resolution):
    if resolution.status != "VERIFIED":
        return None
    return f"{resolution.brand_id}\0{resolution.canonical_reference}"
```

没有模型 pair score。

---

## 8. 图片证据在最终决策中的推荐权限

为了避免视觉模型越权，建议把不同信息源权限写死：

| 信息源 | 可生成候选 | 可独立 VERIFIED | 可否决 | 可直接 MATCH |
|---|---:|---:|---:|---:|
| 独立 reference 字段 | 是 | 条件允许 | 是 | 否 |
| 标题 reference | 是 | 一般否 | 是 | 否 |
| 描述 reference | 是 | 否 | 是 | 否 |
| 保卡/吊牌 OCR | 是 | 一般需第二证据 | 是 | 否 |
| 普通图片 OCR | 是 | 否 | 是 | 否 |
| 整图视觉相似度 | 是（仅召回辅助） | 否 | 可作为异常提示 | 否 |
| LLM/VLM 自由回答 | 是 | 否 | 否 | 否 |
| `brand + VERIFIED reference` entity key | - | - | - | **是** |

这样可以在架构层面保证：

> 多模态模型再强，也没有权限绕过 reference exact gate。

---

## 9. 人工标注应该怎么花：不要随机标几百对

Spec 允许标注几百对黄金标签。建议把标签投入 hard cases，而不是随机 pair。

### 9.1 建议标 4 类数据

#### A. reference span

```text
标题 / 描述 / OCR 中哪一段是当前商品 reference
```

#### B. reference role

```text
SOLD_ITEM_REFERENCE
COMPATIBLE_WITH_REFERENCE
PLATFORM_SKU
SERIAL_NUMBER
CALIBER
...
```

#### C. canonicalization pair

```text
两个 raw string 是否允许规范化成同一个 reference
```

重点采集：

- 点号 / 连字符差异；
- OCR `0/O`；
- suffix；
- leading zero；
- 同系列只差一位的 hard negative。

#### D. record-level resolution

```text
VERIFIED / CONFLICT / ABSTAIN
```

这样几百条数据会比“随机标商品 pair 是否同款”更能直接训练/校准关键模块。

---

## 10. 评估方法必须跟 F1 思维切开

论文主要用 Precision / Recall / F1，而且是 exact-match 属性抽取评估。

当前需求必须单独监控：

```text
auto_match_precision
verified_reference_precision
auto_coverage
abstain_rate
conflict_rate
reference_extraction_recall
```

其中核心 KPI 不是 F1，而是：

```text
在自动放行集合上的 precision
```

建议测试集专门增加 adversarial slices：

```text
同品牌同系列不同 reference
只差 1 字符的 reference
图片极相似不同 reference
标题带“适配某型号”
表带 / 配件 / 盒证商品
OCR 0/O、1/I 混淆
一个 record 同时出现多个型号
新增词典外 reference
```

部署门槛建议采用：

```text
任何新 resolver / canonicalizer 版本：
  必须在 hard-negative regression suite 中 0 observed false-positive
  才允许进入 auto-match lane
```

生产中一旦人工确认一个 false positive，应把它立即变成永久回归用例。

---

## 11. 增量更新架构

百万到千万级持续更新，不需要每次全量重跑。

事件：

```json
{
  "record_id": "...",
  "source": "...",
  "event_type": "UPSERT",
  "payload_version": "..."
}
```

处理：

```text
UPSERT
 -> 检查 title/field/image hash 是否变化
 -> 只重算变化的 evidence
 -> reference resolver
 -> 如果 resolution 变化：
      old entity_key remove
      new entity_key insert
 -> 触发该 entity group consistency audit
```

规则/模型升级则使用版本化重算：

```text
extractor_version
canonicalizer_version
resolver_version
```

可以按 brand 分片批量回填，无需停机。

---

## 12. Reference KB 不完整时怎么办

真实系统中 reference catalog 一定不完整。

因此 dynamic vocabulary 不能变成封闭世界。

设计两个 lane：

```text
KNOWN_REFERENCE
NEW_REFERENCE_CANDIDATE
```

对 `NEW_REFERENCE_CANDIDATE`：

- 不 fuzzy 映射到最近已有型号；
- 如果跨两个来源都出现同一个高可信 raw/canonical candidate，仍然先人工确认；
- 确认后进入 reference catalog；
- 后续同型号可以正常自动识别。

这样可以持续扩展词典，又不会因为知识库滞后产生误匹配。

---

## 13. 为什么不建议直接复刻 PAM 训练

论文训练集有 6 万级样本，而且任务是较一般的商品属性抽取。当前 Spec 只承诺几百个黄金标签。

直接复刻会遇到：

1. 训练数据远远不足；
2. reference 是 identifier，生成损失不等于业务风险；
3. 需要大量图片 OCR 与 region 特征预处理；
4. 训练出来的模型依然无法提供“绝不误匹配”保证；
5. 最终业务定义其实允许我们用 deterministic key resolution，没必要把简单的最终规则模型化。

所以更合理的路线是：

```text
PAM architecture ideas
        ↓
受限 candidate extraction
+ multimodal evidence fusion
+ dynamic reference vocabulary
+ OCR cross validation
        ↓
deterministic precision-first resolver
```

而不是：

```text
重新训练 PAM
  -> decoder 输出 reference
  -> 直接拿输出做 match
```

---

## 14. 推荐实施顺序

### Stage 1：先做不依赖训练的 precision-first 主链

```text
brand normalization
brand-specific reference regex
structured/title reference extraction
lossless canonicalization
reference inverted index
VERIFIED / CONFLICT / ABSTAIN
```

这一阶段就可以解决有明确 reference 的大批数据。

### Stage 2：增加 OCR evidence lane

```text
image type classification
OCR + bbox
reference crop re-OCR
OCR/text cross-check
```

目标是提高“标题没有 reference，但图片有”的覆盖率。

### Stage 3：增加 reference-role classifier

使用几百条 hard-case 标注训练：

```text
SOLD_ITEM_REFERENCE vs other identifiers
```

优先解决配件/兼容型号/平台 SKU 误认。

### Stage 4：如果仍需要，提高 multimodal reranking

再引入类似 PAM 的 transformer fusion：

```text
record candidates
  + text context
  + OCR context
  + image type/region
  + brand reference lexicon
  -> candidate reranker
```

注意输出仍然是“选择 candidate”，而不是自由生成 reference。

---

## 15. 与 PAM 原论文相比，我建议的关键修改

| PAM 原设计 | 当前腕表系统改造 |
|---|---|
| category-conditioned vocabulary | brand / collection-conditioned reference lexicon |
| 可从 external vocab 生成属性 token | 只能选择已有 candidate；词典仅辅助召回 |
| edit distance 直接参与 token score | 只允许用于候选召回，禁止最终 fuzzy merge |
| generic image object features | image-type / reference-bearing-region / conflict evidence |
| OCR token 是一个模态 | OCR token + bbox + image type + second-pass OCR + lineage |
| 属性生成结果直接评价 | 先得到 VERIFIED / CONFLICT / ABSTAIN，再建立 entity key |
| 主要优化 P/R/F1 | 主要优化 auto-match precision，coverage 次之 |
| category auxiliary task | brand + reference-role + image-type auxiliary tasks |
| 单模型承担抽取 | 规则、词典、OCR、小模型分层，模型无最终放行权限 |

---

## 16. 最终推荐方案

如果只从这篇论文拿一个可直接实施的核心思想，我建议是：

> **把 reference number 抽取做成“受限候选、多模态证据、可拒识”的结构化信息抽取问题。**

而当前整体系统应采用：

```text
Reference-centric ER
```

完整决策原则：

```text
1. 先 normalize brand
2. 从 structured field / title / description / OCR 抽 candidate
3. 判定 candidate 的编号角色
4. 使用品牌级规则做无损 canonicalization
5. 允许词典 exact lookup；fuzzy 只召回，不改写
6. 多证据一致 => VERIFIED
7. 任何高可信冲突 => CONFLICT
8. 证据不足 => ABSTAIN
9. 仅 VERIFIED 的 (brand, reference) 可形成 entity_key
10. 相同 entity_key 的跨源记录直接聚合
```

这套方案既吸收 PAM 最有价值的多模态属性抽取能力，又主动去掉了不符合当前 precision-first 需求的部分。

最终可以把整个系统的安全边界概括为一句话：

> **图片和模型负责“找到/验证 reference”，但只有严格验证后的 canonical reference 有权决定跨源 MATCH。**
