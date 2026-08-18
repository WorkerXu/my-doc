# LATEX-Numeric: Language Agnostic Text Attribute Extraction for Numeric Attributes：面向二奢腕表 Reference 高精度抽取的技术与落地分析

## 1. 分析对象与结论

本次选择的对象是：

- 论文：**LATEX-Numeric: Language Agnostic Text Attribute Extraction for Numeric Attributes**
- 作者：Kartik Mehta、Ioana Oprea、Nikhil Rasiwasia（Amazon）
- 会议：NAACL 2021 Industry Track
- 论文页：https://aclanthology.org/2021.naacl-industry.34/
- PDF：https://aclanthology.org/2021.naacl-industry.34.pdf
- 当前需求 Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31

在分析前已检查 `奢侈品调研/b`。此前已经分析过 Ameli、AnyMatch、Confidence Classifiers、DeepBlocker、Ditto、End-to-end multi-modal product matching、7B LLM Entity Matching、Selective Entity Matching、GraLMatch、Large Scale Generative Multimodal Attribute Extraction、Multi-Value Product RAG、Tailoring Entity Resolution、TransClean、WDC-PAVE、parts-distributor-sku-classifier、pyJedAI 等，本论文此前未被 `b` 分析，因此本次不存在重复。

当前 Spec 的核心约束是：

- 雷小安、腕表之家、奢当家三源，数据规模约 100 万～1000 万；
- “同一个商品”定义为**同一 reference number / 型号**；
- reference 可能有独立字段，也可能埋在标题/描述/图片里；
- 字段稀疏、来源异构、持续增量；
- **precision 极端优先，绝不能因为相似而错误合并，允许大量 abstain / 漏匹配**；
- 可人工标注几百对黄金样本。

### 一句话结论

LATEX-Numeric 对当前需求最大的价值，不是它处理“数值属性”的具体任务，而是它解决了一个和三源腕表数据高度同构的训练问题：

> **当结构化字段缺失时，不能把文本中真实存在但没有结构化标签的内容错误标成负样本；应该用远程监督生成高精度正样本，并对缺失标签对应的任务关闭 loss。**

把论文的 MAMT（Multi Attribute Multi Task）架构迁移到腕表后，可以做成一个 **Masked Multi-Head Identifier Extractor**：共享文本编码器，同时分别识别品牌 reference、平台 SKU、商家货号、序列号、配件适配型号等编号角色；某条记录缺哪个结构化字段，就只屏蔽对应 head 的 loss，而不是把全部未命中 token 当作 `O`。

再把论文的 auto-aliasing 从“单位别名发现”改造成“reference 表面形式/规范化操作自动发现”，就能低成本学习：

- `126610LN` / `126610 LN`
- `Ref.126610LN` / `REF 126610LN` / `型号：126610LN`
- 全角/半角、大小写、不同连接符、空格和标点

但**最终自动匹配绝不能直接使用模型相似度**。推荐的生产闭环是：

```text
高精度远程监督
       │
       ▼
Masked Multi-Head 编号抽取器
       │
       ▼
编号角色识别 + 保守规范化
       │
       ▼
Reference Catalog / 品牌规则校验
       │
       ▼
(brand_id, canonical_reference) 严格等值匹配
       │
       ├── ACCEPT
       ├── REVIEW
       └── ABSTAIN / REJECT
```

这套结构把“模型”限制在**找证据**，而不是让模型拥有“合并实体”的最终权力，非常适合当前 Spec。

---

## 2. 论文原始问题：为什么远程监督会悄悄制造假负样本

论文将电商属性抽取建模为 NER：输入商品标题/描述，输出属性值的 token span。

传统远程监督做法很直观：

1. 商品有结构化属性，例如 `RAM=16GB`；
2. 在商品描述中找 `16 GB`；
3. 找到后给对应 token 打 BIO 标签；
4. 其余 token 标 `O`；
5. 用这些自动标签训练 NER。

问题出现在结构化字段本身缺失时。

论文的例子中，文本包含：

```text
... weighs 1.2 kg ...
```

但结构化 `Weight` 字段缺失。普通远程监督无法知道 `1.2 kg` 是 Weight，于是会把它标成 `O`。

这不是普通 label noise，而是论文称为 **Missing-PA（Missing Partial Annotation）** 的问题：

> “没有结构化标签”不等于“文本里没有这个属性”。

对三源腕表而言，这个问题会更加严重。

例如：

```text
标题：劳力士潜航者 126610LN 全套 2023
reference_field: null
platform_sku: LX2023088812
```

如果我们简单用“有 reference_field 才是正样本”的远程监督方式训练，那么 `126610LN` 会被错误标成非 reference。

再例如：

```text
标题：适配 Rolex 126610LN 的黑色表带
reference_field: null
seller_sku: STRAP-610-BLK
```

这里 `126610LN` 虽然是合法 reference，但它是**被适配对象**的 reference，而不是当前售卖商品本身的 reference。

因此当前业务的训练问题至少包含两层：

1. **Missing-PA**：结构化字段缺失造成的假负样本；
2. **Identifier Role Confusion**：标题里出现的编号不一定属于当前商品。

LATEX-Numeric 给了第一层问题非常实用的工程解法，而我们可以在其上扩展第二层。

---

## 3. 论文技术实现拆解

### 3.1 基础 NER：CNN + BiLSTM + CRF

论文将属性抽取建模为 BIO 序列标注，基础模型使用经典 `CNN-BiLSTM-CRF`：

```text
字符序列
  │
  ├── Character CNN ──► 字符级表示
  │
词序列 / word embedding
  │
  ▼
BiLSTM ──► 上下文 token 表示
  │
  ▼
CRF ──► BIO 标签序列
```

三部分职责：

- **Character CNN**：学习拼写、前后缀、形态信息；
- **BiLSTM**：学习上下文，例如 `16 GB RAM` 与 `16 GB storage` 的区别；
- **CRF**：利用相邻标签约束，避免不合法 BIO 序列。

论文也验证了 MAMT 思路可以挂到 BERT 上，因此我们落地时没必要照搬 2021 年的 BiLSTM。更合理的是保留“共享 encoder + task-specific heads + masked loss”这个核心思想，encoder 换成适合中文电商文本的 Transformer。

建议起步选择：

```text
共享 encoder：Chinese RoBERTa / MacBERT / multilingual MiniLM 类小模型
任务 head：Token Classification (+ 可选 CRF)
```

如果线上吞吐优先，可以蒸馏为 6 层小模型；如果 reference 很多是纯字母数字串，字符级/byte-level 特征依然值得保留。

---

### 3.2 MAST：一个模型里把所有属性塞进一套标签

论文先定义了常见的 MAST（Multi Attribute Single Task）模式。

假设有 K 个属性，则标签空间为：

```text
B-attr_1
I-attr_1
B-attr_2
I-attr_2
...
O
```

总共 `2K + 1` 个标签。

这类架构在完整人工标注数据上没问题，但在远程监督中很危险：如果某属性结构化值缺失，那个属性在文本中真实出现的位置会被错误当成 `O`，并直接参与 loss。

对于当前业务，如果把：

- reference
- platform_sku
- seller_sku
- serial_number
- accessory_target_reference

全部塞进一个 BIO 标签空间，某一列缺失就很容易制造大量假负样本。

---

### 3.3 MAMT：共享 encoder，每个属性一个任务 head

论文真正关键的设计是 MAMT（Multi Attribute Multi Task）。

其结构可抽象为：

```text
                   ┌── Head A：属性 A BIO
Input ─ Shared ────┼── Head B：属性 B BIO
        Encoder     ├── Head C：属性 C BIO
                   └── Head D：属性 D BIO
```

共享：

- 字符编码；
- 词向量；
- BiLSTM / 上下文 encoder。

不共享：

- 每个属性自己的输出层。

最重要的是 **loss masking**。

对训练样本 `i`、任务 `t` 定义：

```text
m(i,t) = 1  当该样本的结构化属性 t 已知
m(i,t) = 0  当该样本的结构化属性 t 缺失
```

总 loss 可理解为：

```text
L = Σ_i Σ_t m(i,t) * L(i,t)
```

也就是说：

- `RAM` 已知 -> RAM head 参与训练；
- `Weight` 缺失 -> Weight head 不产生 loss；
- 共享 encoder 仍可以从其他任务得到更新。

这避免了把“未知”训练成“负类”。

论文实验中，MAMT 相比 MAST 在数值属性上带来约 **9.2% F1 提升**；作者也在 BERT 上验证了同样思路，并显示它不仅适用于数值属性。

### 对当前需求的直接改造

不要把“属性”理解成腕表颜色、尺寸，而应把任务定义为不同的**编号语义角色**：

```text
Head 1: product_reference
Head 2: platform_sku
Head 3: seller_sku
Head 4: serial_number
Head 5: accessory_target_reference
Head 6: certificate_or_card_number
```

这样模型不是只问：

> “哪段文本像编号？”

而是在问：

> “这段编号在当前上下文中是什么角色？”

这正是避免 false positive 的关键。

---

## 4. Auto-Aliasing：论文如何自动发现表面形式

论文的第二个重要贡献是自动 alias 发现。

数值属性常有不同单位表面形式：

```text
inch / inches / in
kg / kilograms / lbs / pounds
GB / gigabyte
```

如果远程监督只拿 canonical unit 去匹配，会漏掉大量训练样本。

论文设计了两路 alias 发现：

### 4.1 `alias_dw`：从“已知属性值附近”发现单位

给定一个已知结构化值，例如：

```text
RAM = 8
```

在商品文本中找形如：

```text
8 <word>
```

从 `<word>` 统计候选 alias，例如 `8GB`、`8 gigabyte`。

为降低碰撞，如果同一文本中存在多个可能匹配，则论文倾向于放弃不确定样本，而不是强行打标签。

这个“**歧义就丢掉**”的策略非常适合当前 Spec 的 precision-first 哲学。

### 4.2 `alias_bp`：从所有“数字 + 单位”组合中发现候选

第二路不依赖已知属性值，而是扫描文本里的所有“数字 + 单词”组合，建立较宽的候选表。

因为候选噪声很高，论文再使用词向量和 canonical unit 做相似度过滤，得到属性特定的候选单位。

两路结果最后合并为 `alias_combined`。

论文报告：auto-aliasing 相比只使用 canonical alias 的远程监督，平均 F1 提升约 **20.2%**。

---

## 5. 把 Auto-Aliasing 改造成 Reference Surface-Form Induction

腕表 reference 通常不是“单位”，而是字母数字混合 identifier，因此不能照搬单位 embedding 相似度。

更安全的迁移方式是：

> 不学习“reference 值的模糊同义词”，只学习**reference 在文本中的合法表面变换操作**。

例如已知结构化：

```text
reference = 126610LN
```

在标题中观察到：

```text
126610 LN
Ref. 126610LN
REF126610LN
型号 126610LN
126610-LN
```

我们可以从大量高置信 seed 中归纳操作：

```text
op_1: 删除 reference 内部 ASCII 空格
op_2: 统一 '-' / '–' / '—'
op_3: 大小写统一
op_4: 去除固定上下文前缀 Ref / REF / 型号 / 型號
op_5: 全角字符 NFKC
```

注意：这些是**可审计、可版本化、可回放**的 normalization operators，而不是 fuzzy matching。

### 明确禁止自动学习的操作

以下规则不能因为“提高召回”就自动放开：

```text
删除任意前导 0
删除任意尾部字母
任意编辑距离 <= 1 视为同型号
把相似 reference 聚类
跨品牌共用激进格式规则
```

例如腕表中一个字符差异可能就是不同材质、表盘、尺寸、代际或市场版本。当前 Spec 的目标定义要求 reference 一致，因此 normalization 必须保证**不改变 identifier 语义**。

---

## 6. 建议落地：Reference-MAMT 架构

我建议基于论文实现一个专用的 **Reference-MAMT**。

### 6.1 总体架构

```text
              ┌─────────────────────┐
              │ 雷小安 / 腕表之家 / 奢当家 │
              └─────────┬───────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │ Raw Offer Store     │
              │ title/desc/fields   │
              │ images/source ids   │
              └─────────┬───────────┘
                        │
                        ▼
          ┌──────────────────────────────┐
          │ Safe Pre-normalization       │
          │ Unicode / punctuation / case │
          └─────────────┬────────────────┘
                        │
                        ▼
          ┌──────────────────────────────┐
          │ Deterministic Seed Builder   │
          │ 结构化字段 + exact span 对齐    │
          └─────────────┬────────────────┘
                        │
                        ▼
          ┌──────────────────────────────┐
          │ Reference-MAMT Extractor     │
          │ Shared Transformer Encoder   │
          │ + masked task heads          │
          └───────┬───────────┬──────────┘
                  │           │
                  ▼           ▼
       product_reference   sku/serial/accessory
                  │           │
                  └─────┬─────┘
                        ▼
          ┌──────────────────────────────┐
          │ Brand-aware Canonicalizer    │
          │ 可逆/保守 normalization       │
          └─────────────┬────────────────┘
                        ▼
          ┌──────────────────────────────┐
          │ Reference Catalog Validator  │
          │ pattern / known refs / conflicts│
          └─────────────┬────────────────┘
                        ▼
          ┌──────────────────────────────┐
          │ Decision Gate                │
          │ ACCEPT / REVIEW / ABSTAIN    │
          └─────────────┬────────────────┘
                        ▼
          ┌──────────────────────────────┐
          │ Exact Entity Key             │
          │ (brand_id, canonical_ref)    │
          └──────────────────────────────┘
```

核心原则：

> **模型只抽取和分类证据；最终 entity merge 必须经过 deterministic gate。**

---

## 7. 训练数据：利用三源现有结构化字段做远程监督

### 7.1 Seed 正样本

优先取最可信的数据：

```text
brand 已知且已 canonicalize
AND reference_field 非空
AND reference_field 能在 title/description 中安全 exact-align
AND 当前文本没有多个冲突 reference
```

例如：

```json
{
  "brand": "Rolex",
  "reference_field": "126610LN",
  "title": "劳力士 潜航者 126610LN 2023 全套"
}
```

可以自动得到：

```text
劳 力 士 潜 航 者 126610LN 2023 全 套
                 B-REF
```

这些 seed 的 precision 比“LLM 自己猜标签”更高，规模也更容易做大。

### 7.2 不要把 reference 缺失记录全部当负样本

这是论文最需要保留的原则。

```text
reference_field == null
```

只能说明：

```text
reference task label = UNKNOWN
```

不能推导：

```text
所有 token 都是 O
```

因此这条记录的 `product_reference` head 应：

```text
mask = 0
```

其他已知字段对应的 head 仍可训练。

### 7.3 SKU / 序列号等负责任务也用同一机制

如果平台已有：

```text
platform_sku = "LX-889991"
```

可以对 `platform_sku` head 造高精度标注。

这会给模型大量“看起来像 reference、但实际不是 reference”的真实上下文，远比随机生成负样本更有价值。

---

## 8. 模型任务设计：把“编号角色”拆开而不是二分类

推荐至少有如下任务：

| Head | 目标 | 典型来源 |
|---|---|---|
| `product_reference` | 当前售卖商品本身 reference | 标题、型号字段、描述 |
| `platform_sku` | 平台自己的商品号 | 平台字段、标题 |
| `seller_sku` | 商家/门店内部货号 | 描述、商家字段 |
| `serial_number` | 单件唯一序列号 | 保卡、表背、文本 |
| `accessory_target_reference` | 配件所适配的腕表型号 | “适配/for/compatible”上下文 |
| `certificate_number` | 保卡/证书类编号 | OCR、描述 |

模型输入可以把来源信息作为特殊 token：

```text
[SOURCE=LEXIAOAN] [FIELD=TITLE] 劳力士潜航者 ...
```

或把 title、description、structured snippets 拼接，并标记字段边界：

```text
[TITLE] ... [DESC] ... [OTHER_FIELD] ...
```

### 为什么不能只做 `is_reference` 二分类

因为线上最危险的错误不是“完全不像型号的文本被识别成型号”，而是：

> **一个真实存在的腕表 reference 被识别出来了，但它并不属于当前售卖商品。**

例如：

```text
“适用于 AP 15500ST 的橡胶表带”
```

`15500ST` 是真 reference，但当前商品是表带。

因此 role-aware extraction 比单纯 regex / NER 更符合业务风险。

---

## 9. Masked Multi-Head Loss 的落地实现

设共享 encoder 输出：

```text
H = Encoder(x)
```

每个 task 有自己的 token classifier：

```text
logits_t = Head_t(H)
```

任务 mask：

```text
mask_t = 1  标签可信且可用于训练
mask_t = 0  标签缺失/未知
```

loss：

```text
loss = 0
active = 0

for task in tasks:
    if mask[task] == 1:
        loss += task_loss(logits[task], labels[task])
        active += 1

loss = loss / max(active, 1)
```

关键点不是代码复杂度，而是**标签语义**：

- 空值不是负类；
- 未知不是 `O`；
- 只有可信结构化字段和可信规则才能打开对应 task 的 loss；
- 存在冲突的样本宁可不训练。

这恰好把 Spec 的 precision-first 原则提前到训练数据层。

---

## 10. Reference Canonicalization：保守、品牌化、可逆

建议 canonicalizer 分两层。

### Layer A：全局无语义变换

只做极安全操作：

```text
Unicode NFKC
trim
大小写统一为 upper
统一不可见空白
统一全角冒号/连字符到标准字符
```

任何操作都保存：

```text
raw_value
normalized_value
normalization_trace
rule_version
```

### Layer B：品牌级规则

例如某品牌经过黄金样本验证后，可以允许：

```text
126610 LN -> 126610LN
```

但规则必须带：

```text
brand_id
pattern
transform
support_count
conflict_count
gold_precision
version
status
```

推荐表：

```sql
reference_normalization_rule(
  rule_id,
  brand_id,
  pattern,
  transform,
  support_count,
  conflict_count,
  gold_precision,
  version,
  enabled
)
```

### 不接受“模型直接生成 canonical reference”作为自动合并依据

模型生成值可能产生：

- 漏字符；
- 多字符；
- 相近型号纠错；
- 根据上下文“脑补”型号。

因此生成式模型最多用于候选或 review 辅助，不能直接产生最终 entity key。

---

## 11. Reference Catalog：把抽取变成受约束识别

建议维护全局 catalog：

```sql
reference_catalog(
  brand_id,
  canonical_reference,
  series,
  family,
  first_seen_at,
  last_seen_at,
  source_count,
  trusted_evidence_count,
  status
)
```

来源可以包括：

1. 三源高可信结构化字段；
2. 官方/可信型号表；
3. 人工确认；
4. 多源独立一致的高可信发现。

抽取器得到候选后，先做 catalog 校验：

```text
候选 ref 在 catalog 内
   ├─ 是：继续 evidence gate
   └─ 否：默认 REVIEW / ABSTAIN，不直接建实体
```

长尾新 reference 当然会出现，所以 catalog 不能完全阻止新增；但新增 reference 应走更严格的 `new-reference admission`：

```text
brand pattern 合法
AND 至少一个强结构化来源
AND 无冲突编号
AND （人工确认 OR 多独立来源一致）
```

这样 catalog 同时承担：

- vocabulary constraint；
- 数据质量防火墙；
- 增量系统的稳定锚点。

---

## 12. 最终匹配决策：严格 key，而不是 fuzzy matcher

一旦每条商品记录获得：

```text
brand_id
canonical_reference
reference_evidence
```

跨源匹配应该极其简单：

```text
entity_key = hash(brand_id + "|" + canonical_reference)
```

即：

```sql
JOIN offers a
JOIN offers b
  ON a.brand_id = b.brand_id
 AND a.canonical_reference = b.canonical_reference
```

### ACCEPT 推荐条件

最小版：

```text
brand_id 相同
AND canonical_reference 严格相同
AND reference_role == PRODUCT_REFERENCE
AND 没有任何强冲突 reference
AND reference 证据达到可信级别
```

### 典型 REJECT / ABSTAIN

#### 情况 1：同系列视觉非常像，但 reference 不同

```text
126610LN != 126610LV
```

直接不匹配，图片不能覆盖 reference 冲突。

#### 情况 2：标题有 reference，但上下文是“适配”

```text
适配 126610LN 表带
```

识别成 `accessory_target_reference`，禁止形成当前商品 entity key。

#### 情况 3：模型抽到 ref，但 catalog 不认识且没有强结构化证据

进入 REVIEW / ABSTAIN。

#### 情况 4：多个证据抽出冲突 reference

```text
structured_ref = 126610LN
title_ref      = 126610LV
```

不要投票，不要相似度融合；进入冲突队列。

---

## 13. 图片怎么用：只能做证据增强或冲突否决

当前 Spec 有图片，图片对 reference 很有价值，但应放在正确位置。

建议图片链路：

```text
图片
 ├─ OCR：表背 / 保卡 / 吊牌 / 标签文字
 ├─ VLM：判断图中区域类型（表背、保卡、表盘、盒证）
 └─ 视觉相似度：只作 review 辅助
```

OCR 得到的 reference 也走同一个：

```text
角色识别 -> canonicalization -> catalog -> decision gate
```

### 图片可以把 REVIEW 提升到 ACCEPT，但不应覆盖文本硬冲突

例如：

```text
title_ref = 126610LN
OCR_card_ref = 126610LN
```

这是强一致证据。

但：

```text
title_ref = 126610LN
structured_ref = 126610LV
image_embedding 很像 126610LN
```

仍然必须冲突拒识。

视觉外观在同系列不同 reference 间往往高度相似，所以图片不适合拥有最终 merge 权。

---

## 14. 数据表与证据审计

建议不要只在 `offer` 表上存一个最终 ref，而要显式存 evidence。

### 14.1 `reference_evidence`

```sql
reference_evidence(
  evidence_id,
  offer_id,
  source_name,
  field_name,
  extractor_type,
  raw_span,
  raw_reference,
  normalized_reference,
  role,
  confidence,
  rule_version,
  model_version,
  created_at
)
```

`extractor_type` 示例：

```text
STRUCTURED_FIELD
EXACT_REGEX
MAMT_TEXT
OCR
MANUAL
```

### 14.2 `offer_reference_resolution`

```sql
offer_reference_resolution(
  offer_id,
  brand_id,
  canonical_reference,
  resolution_status,
  resolution_reason,
  winning_evidence_ids,
  conflict_evidence_ids,
  resolver_version,
  resolved_at
)
```

状态：

```text
ACCEPTED
REVIEW
ABSTAINED
REJECTED_CONFLICT
```

这样任何跨源合并都可以回答：

> “为什么这两个记录被认为是同一个 reference？”

这对“绝不能误匹配”的系统非常重要。

---

## 15. 增量处理架构

当前数据会持续更新，不应该每次全量重跑 pairwise matching。

推荐处理单元是“offer 解析”，而不是“offer pair 比较”。

```text
新/变更 offer
     │
     ▼
解析 brand + reference
     │
     ▼
得到 canonical entity_key
     │
     ▼
upsert 到 entity membership
```

复杂度从潜在的 pairwise：

```text
O(N²)
```

变成接近：

```text
O(N * extraction_cost)
```

最终实体查找是索引键查找。

### 推荐唯一索引

```sql
CREATE UNIQUE INDEX uq_reference_entity
ON reference_entity(brand_id, canonical_reference);
```

### offer membership

```sql
entity_offer_membership(
  entity_id,
  source,
  source_offer_id,
  valid_from,
  valid_to,
  is_current
)
```

这样同一来源商品更新、下架、重新抓取时，不需要重新计算整个实体图。

---

## 16. 训练集如何利用“几百对黄金标签”

几百对人工标签不应该主要用来从零训练大模型，而应该用在最值钱的位置。

### 用途 1：验证远程监督规则 precision

抽样检查：

```text
结构化 ref exact-align title
brand normalization rule
reference role rule
```

只有高精度规则才能进入 seed builder。

### 用途 2：标 hard negatives

优先标：

```text
同品牌、reference 仅差 1 字符
同系列不同 reference
配件标题包含目标腕表 reference
同一标题多个编号
平台 SKU 很像品牌 reference
保卡号/序列号与 reference 混杂
```

这些比随机负样本重要得多。

### 用途 3：阈值和 ACCEPT gate 校准

最终关注指标不应只是 F1，而应该分层统计：

```text
Precision@AUTO_ACCEPT
Auto-accept coverage
Conflict escape rate
Unknown-reference admission precision
Per-source / per-brand precision
```

### 一个重要统计现实

仅有几百个黄金样本，即使观察到 0 个 false positive，也**不能数学上证明线上千万级数据绝对零误匹配**。

因此“绝不能误匹配”必须主要通过以下机制实现：

```text
硬语义定义
+ exact reference key
+ role-aware extraction
+ conflict veto
+ conservative normalization
+ abstention
+ evidence audit
```

而不能只靠“验证集 precision=100%”。

---

## 17. 评测集切分：禁止随机行切分造成 reference 泄漏

如果同一 reference 的不同商品记录同时进入 train/test，模型可能记住型号，评估会虚高。

建议至少做三种切分：

### A. Reference-disjoint split

训练和测试 reference 集合不重叠，用来测长尾新型号泛化。

### B. Source holdout

例如训练雷小安 + 腕表之家，测试奢当家，验证跨来源迁移。

### C. Hard-negative benchmark

人工构造高风险测试：

```text
126610LN vs 126610LV
26240OR vs 26240ST
同系列相邻 reference
“适配 xxx”配件
标题多型号
OCR 错一字符
```

主指标应直接报告 false positive 个数和对应 case，而不是只看平均 F1。

---

## 18. 自动规则发现：把论文 alias discovery 变成可上线的安全流程

推荐离线任务：

```text
Step 1  收集强 seed：structured_ref + exact text occurrence
Step 2  对 raw span 与 structured_ref 做 alignment
Step 3  归纳 normalization operation 模板
Step 4  按 brand 聚合 support / collision
Step 5  用黄金集计算 precision
Step 6  规则进入 SHADOW
Step 7  无冲突后提升为 ENABLED
```

规则状态：

```text
CANDIDATE
SHADOW
ENABLED
DISABLED
```

### 例子

发现 Rolex 大量数据：

```text
structured: 126610LN
text:       126610 LN
```

候选规则：

```text
Rolex + pattern [0-9]{6}\s+[A-Z]{1,3}
=> remove internal whitespace
```

但是如果另一品牌存在：

```text
AB 12 C
AB12C
```

可能不是等价语义，就不能跨品牌共享该规则。

这比全局 `re.sub(r'[^A-Z0-9]', '')` 安全得多。

---

## 19. 与纯 LLM 抽取方案相比，这篇论文补上了什么

`b` 目录之前已经分析过 WDC-PAVE 的 LLM 抽取与规范化。LATEX-Numeric 提供的是另一块很重要的能力：**如何低成本构造规模化训练数据，并正确处理字段缺失导致的监督噪声**。

建议组合，而不是二选一：

```text
规则 / 结构化字段
      │
      ├── 高精度 seed -> MAMT 小模型训练
      │
      └── 规则无法覆盖的疑难样本
                    │
                    ▼
                 LLM fallback
```

LLM 不适合对千万级所有记录都跑，也不适合直接决定 merge；小模型适合大规模常规抽取，LLM 只处理：

```text
复杂上下文
多编号歧义
品牌新格式
人工 review 辅助
```

最终仍然统一进入同一个 deterministic resolver。

---

## 20. 推荐的 Decision Gate

可以先做一个不依赖“神奇模型分数”的规则型 gate。

### Evidence Level

```text
E5  人工确认
E4  可信结构化 reference + 文本 exact 支持
E3  两个独立模态 exact 一致（例如 title + OCR）
E2  MAMT 抽取 + catalog 命中 + 无冲突
E1  单模型抽取或未知 reference
E0  冲突 / 角色不明确
```

### 自动接受策略

初期建议：

```text
AUTO_ACCEPT: E4 / E5
REVIEW:      E2 / E3
ABSTAIN:     E1
REJECT:      E0
```

经过黄金集和线上 shadow 验证后，再考虑让部分 E3 自动进入 ACCEPT。

这里故意保守，因为当前目标是 precision-first，而不是最大覆盖率。

---

## 21. MVP：先不上复杂模型也能直接落地

第一版其实可以先把论文思想落在数据与规则层，而不是马上训练 Transformer。

### Stage 1：安全 Reference-First Pipeline

实现：

```text
brand canonicalization
structured reference parser
safe normalization
brand-specific regex
identifier role context rules
reference catalog
conflict veto
exact entity key
```

这一步就能覆盖结构化字段完整的记录，而且误匹配面最小。

### Stage 2：远程监督数据生成

从 Stage 1 的高可信结果生成：

```text
BIO spans
head masks
hard negative pool
normalization rule candidates
```

### Stage 3：Reference-MAMT 小模型

只用来补：

```text
reference 字段缺失
标题中有型号
多来源格式不同
编号角色歧义
```

### Stage 4：OCR / 多模态证据

图片用于：

```text
表背 / 保卡 / 吊牌 OCR
文本证据交叉确认
冲突告警
```

### Stage 5：LLM / 人工只处理尾部疑难

将昂贵推理集中在 REVIEW 队列。

---

## 22. 建议代码模块

```text
reference_matching/
├── ingestion/
│   ├── lexiaoan.py
│   ├── xwatch.py
│   └── shedangjia.py
├── brand/
│   └── canonicalizer.py
├── weak_supervision/
│   ├── seed_builder.py
│   ├── span_aligner.py
│   ├── task_mask_builder.py
│   └── normalization_rule_miner.py
├── extractor/
│   ├── model.py
│   ├── heads.py
│   ├── train.py
│   └── infer.py
├── identifier_role/
│   └── rules.py
├── normalization/
│   ├── safe_global.py
│   └── brand_rules.py
├── catalog/
│   └── reference_catalog.py
├── evidence/
│   ├── text.py
│   ├── ocr.py
│   └── resolver.py
├── matching/
│   ├── decision_gate.py
│   └── entity_key.py
└── evaluation/
    ├── hard_negatives.py
    ├── precision_report.py
    └── drift_report.py
```

这种模块划分的优点是模型可替换，但业务安全边界不会跟着模型变化。

---

## 23. 关键伪代码

### 23.1 Seed 构造

```python
def build_reference_seed(offer):
    if not offer.brand_id:
        return None

    if not offer.structured_reference:
        # UNKNOWN，不生成 reference 负标签
        return {
            "reference_mask": 0,
            "reference_labels": None,
        }

    ref = safe_normalize(offer.structured_reference)
    spans = exact_align_reference(ref, offer.title, offer.description)

    # precision-first：歧义样本不造训练标签
    if len(spans) != 1:
        return {
            "reference_mask": 0,
            "reference_labels": None,
        }

    return {
        "reference_mask": 1,
        "reference_labels": to_bio(spans[0]),
    }
```

### 23.2 Resolver

```python
def resolve_reference(offer, evidences, catalog):
    refs = [e for e in evidences if e.role == "PRODUCT_REFERENCE"]
    conflicting = unique_canonical_refs(refs)

    if len(conflicting) > 1:
        return "REJECTED_CONFLICT", None

    if not refs:
        return "ABSTAINED", None

    ref = strongest(refs)

    if ref.strength >= E4:
        return "ACCEPTED", ref.canonical_reference

    if ref.strength >= E2 and catalog.contains(offer.brand_id, ref.canonical_reference):
        return "REVIEW", ref.canonical_reference

    return "ABSTAINED", None
```

### 23.3 Entity Key

```python
def entity_key(brand_id, canonical_reference):
    if not brand_id or not canonical_reference:
        return None
    return f"{brand_id}:{canonical_reference}"
```

不要把 embedding 相似度、图片相似度、标题相似度放进最终 key。

---

## 24. 线上监控：模型漂移之外，更要监控规则与冲突

建议日报/监控至少包含：

```text
auto_accept_count
review_count
abstain_count
conflict_count
unknown_reference_count
new_reference_admission_count
per-source extraction rate
per-brand extraction rate
normalization rule hit rate
OCR/text disagreement rate
```

重点告警：

```text
某品牌 conflict rate 突增
某来源 unknown ref 比例突增
某 normalization rule collision 突增
某新爬虫字段导致 SKU 被识别成 ref
```

由于三个来源会持续变化，数据契约漂移往往比模型本身更容易导致生产错误。

---

## 25. 论文方案的局限，以及当前项目必须修改的地方

### 局限 1：论文主要是数值属性，reference 是高信息密度 identifier

数值属性允许单位转换、alias；reference 对单字符变化极其敏感。

因此只能迁移：

```text
远程监督
Missing-PA 处理
multi-task masked loss
auto discovery 的工程思想
```

不能迁移：

```text
基于 embedding 的值级模糊等价
```

### 局限 2：论文优化 F1，不等于我们的风险目标

论文的 MAMT 在某些文本属性实验里 F1 提升伴随 precision 相对下降。这提醒我们：

> MAMT 是“更好的抽取器”，不是“自动放行器”。

当前系统必须把自动 merge 的 precision 独立于 extractor F1 管理。

### 局限 3：论文只看文本

当前 Spec 有图片，应该增加 OCR 和图像区域识别，但仍保持 reference exact rule 收口。

### 局限 4：论文没有解决“编号角色”问题

我们新增 role heads 与 accessory context，是针对二奢/腕表业务最关键的扩展。

---

## 26. 最终推荐方案

如果只从这篇论文提炼一个可直接落地的架构，我建议采用：

### **Distant-Supervised Reference-MAMT + Deterministic Exact Resolver**

完整链路：

```text
三源原始商品
    │
    ▼
品牌规范化
    │
    ▼
安全规则抽取结构化/文本/OCR 编号
    │
    ▼
高精度 seed builder
    │
    ▼
Reference-MAMT
  - product reference
  - platform SKU
  - seller SKU
  - serial
  - accessory target ref
  - certificate no.
    │
    ▼
保守 normalization
    │
    ▼
Reference Catalog 校验
    │
    ▼
证据冲突 veto
    │
    ▼
ACCEPT / REVIEW / ABSTAIN
    │
    ▼
(brand_id, canonical_reference) exact entity key
```

### 为什么这套最符合 Spec

1. **规模可扩展**：远程监督可从已有百万级商品自动造标签；
2. **适合字段稀疏**：MAMT 的 masked loss 天生处理结构化字段缺失；
3. **适合多来源**：共享 encoder 学跨源上下文，每个 source 不需要一套独立模型；
4. **适合 precision-first**：歧义样本可以不造训练标签，线上也可以 abstain；
5. **可审计**：每次合并都有 raw reference、normalization trace 和 evidence；
6. **不需要 pairwise 全连接**：解析后直接按 exact key 聚合，适合 100 万～1000 万量级；
7. **能持续迭代**：人工复核可回流为 seed、hard negatives 和规则验证数据。

### 最重要的安全边界

必须坚持：

> **模型可以告诉系统“我在这里发现了一个很像 reference 的字符串”，但模型不能告诉系统“这两个商品应该合并”。**

最终跨源实体身份必须由：

```text
canonical brand 一致
+
canonical reference 严格一致
+
编号角色确认属于当前商品
+
无强冲突证据
```

共同决定。

这也是 LATEX-Numeric 最值得迁移到当前业务的工程思想：**先把高精度监督与证据抽取做好，再让最终决策尽可能简单、确定和可解释。**

---

## 27. 参考资料

- LATEX-Numeric 论文页：https://aclanthology.org/2021.naacl-industry.34/
- LATEX-Numeric PDF：https://aclanthology.org/2021.naacl-industry.34.pdf
- Ma & Hovy, End-to-end Sequence Labeling via Bi-directional LSTM-CNNs-CRF：https://aclanthology.org/P16-1101/
- 当前需求 Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
