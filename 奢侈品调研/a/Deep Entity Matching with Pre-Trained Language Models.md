# Deep Entity Matching with Pre-Trained Language Models：面向二奢腕表 Reference-First 匹配系统的技术实现与落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **Deep Entity Matching with Pre-Trained Language Models（Ditto）** 进行深入分析；执行前已检查 `奢侈品调研/a/`，该标题尚未由 `a` 分析过。

- 论文：<https://arxiv.org/abs/2004.00584>
- 官方实现：<https://github.com/megagonlabs/ditto>
- 目标需求：<https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31>

Ditto 的核心做法非常简洁：把左右两条实体记录序列化成文本对，交给 BERT / DistilBERT / RoBERTa 一类预训练 Transformer，再用 `[CLS]` 表征接一个二分类头判断 `match / non-match`。论文另外提供三类增强：

1. **Domain Knowledge Injection**：标出 Product ID、品牌、数字等关键 span；
2. **Summarization**：长文本时按 TF-IDF 保留信息量更高的 token；
3. **Data Augmentation / MixDA**：删除、交换 span、删除字段、移动字段，并在隐层混合原样本与增强样本。

Ditto 非常适合当前项目的若干现实问题：

- 三个平台 schema 不一致；
- 字段稀疏，reference 有时在独立字段、有时藏在标题；
- 商品标题噪声大；
- 有大量“同系列、很像但 reference 不同”的困难负例；
- 百万到千万规模不能做笛卡尔积，需要先 blocking；
- 只有几百对人工黄金标签。

但 **Ditto 官方实现不能直接成为最终自动合并器**。当前 Spec 的业务定义比通用 Entity Matching 更严格：

> 同一个商品 = 同一个 reference number / 型号；绝对不能误匹配，precision 优先到极致，允许漏匹配。

官方 Ditto 最终仍是概率二分类器，并且：

- 验证集阈值按 F1 调优；
- 默认推理阈值是 0.5；
- 没有 `abstain / review` 状态；
- 没有 reference 硬冲突否决；
- `ProductDKInjector` 对“像 ID 的字符串”的识别过于宽松，可能把平台 SKU 当品牌 reference；
- 默认 data augmentation 可能删除整个字段或 ID span，却仍保留原标签；
- 默认 truncation / TF-IDF summarization 都无法保证 reference 永远不会丢；
- 不处理图片；
- 只负责 matcher，不解决千万级候选生成。

因此推荐方案不是：

```text
商品 A + 商品 B
      ↓
    Ditto
      ↓
match / non-match
```

而是把 Ditto 降级为 **Reference-First 系统中的二阶段复核器**：

```text
雷小安 / 腕表之家 / 奢当家原始商品
              │
              ▼
      Reference Evidence Extraction
  （结构字段 / 标题 / OCR / 来源规则）
              │
              ▼
      Identifier Role Classification
(reference / platform_sku / seller_sku / accessory_id)
              │
              ▼
      Brand-aware Canonicalization
              │
              ▼
      Canonical Reference Registry
        (brand_id, canonical_ref)
              │
       ┌──────┴──────┐
       │             │
   exact lookup   ref 不完整/多候选
       │             │
       ▼             ▼
   Hard Gate      Candidate Blocking
       │             │
       └──────┬──────┘
              ▼
    Ditto-style Secondary Matcher
  （只做疑难候选复核，不可越权）
              │
              ▼
      Precision-first Decision Gate
       ├── AUTO_LINK
       ├── REVIEW
       └── REJECT
              │
              ▼
       reference_entity_id
```

最重要的落地原则是：

> **模型只能增加“需要复核”的证据，不能覆盖 reference 冲突；AUTO_LINK 必须建立在可解释、可追溯、无冲突的 canonical reference 证据上。**

---

## 1. Ditto 原论文到底解决什么问题

Entity Matching（EM）的输入是两个数据集合 `D` 和 `D'`，目标是在候选记录对中判断哪些记录表示同一个现实实体。完整 EM 系统通常拆成：

```text
D × D'
  │
  ▼
Blocking
  │  高召回地去掉明显不可能匹配的 pair
  ▼
Candidate Pairs
  │
  ▼
Matcher
  │
  ▼
Matched Pairs
```

Ditto 重点优化的是第二步 **Matcher**，并不试图用 Transformer 直接对所有记录做两两比较。

论文特别强调：Blocking 应该高召回，而 matcher 再对“已经很相似”的困难候选做精细判断。这个分工对当前 100 万–1000 万规模尤其重要，因为即便只有两个各 100 万的集合，笛卡尔积也已经是 `10^12` 个 pair。

当前项目比论文的一般 EM 任务更容易利用业务规则，因为 Spec 已经给出强定义：最终实体不是一个模糊的“同款商品”，而是 **reference identity**。这意味着我们应该从“pair classifier”进一步收敛为“记录链接到 reference entity”。

---

## 2. Ditto 的原始技术架构

### 2.1 记录序列化：把异构 schema 统一成 token sequence

Ditto 将一条结构化记录序列化为：

```text
COL title VAL microsoft visio standard 2007 version upgrade
COL manufacturer VAL microsoft
COL price VAL 129.95
```

两条记录组成一个 sequence pair，由 tokenizer 生成类似：

```text
[CLS] left_record [SEP] right_record [SEP]
```

这样做的优势是：**左右表不要求预先做 schema alignment**。

例如来源 A 叫：

```text
款号 = 126610LN
品牌 = 劳力士
```

来源 B 可能只有：

```text
model = 126610LN
title = Rolex Submariner Date ...
```

甚至来源 C 没有独立型号字段，reference 只出现在标题里。序列化后，Transformer 可以对两边不同字段位置的信息做 cross-attention，而不需要人为要求“第 3 列只能和第 3 列比较”。

这也是 Ditto 对当前三源脏数据最有价值的能力之一。

### 2.2 Transformer + 线性二分类头

官方 `ditto_light/ditto.py` 的模型非常简单：

```python
self.bert = AutoModel.from_pretrained(...)
hidden_size = self.bert.config.hidden_size
self.fc = torch.nn.Linear(hidden_size, 2)
```

普通前向过程：

```python
enc = self.bert(x1)[0][:, 0, :]
return self.fc(enc)
```

即：

```text
serialized pair
      ↓
Tokenizer
      ↓
BERT / DistilBERT / RoBERTa
      ↓
first token / [CLS] hidden state
      ↓
Linear(hidden_size → 2)
      ↓
match / non-match logits
```

训练损失是普通 `CrossEntropyLoss`。

这说明 Ditto 的主要价值不来自复杂定制网络，而是：

- 一个合适的数据序列化协议；
- 预训练 LM 的上下文理解能力；
- 把领域知识、摘要、增强放在数据层；
- 让模型对困难候选做联合语义比较。

对于工程团队，这类结构也比较容易替换 backbone，例如中文或多语言 Transformer，而不用重写匹配框架。

### 2.3 Ditto 对腕表数据不是纯“跨领域”迁移

官方 `configs.json` 直接包含：

```text
wdc_watches_small
wdc_watches_medium
wdc_watches_large
wdc_watches_xlarge
```

官方 WDC watches 训练样本里也能看到真实的字母数字产品标识，例如：

```text
Nixon ... A1186-001
Junghans ... 027/4700.00
Suunto ... SS020674000
```

所以 Ditto 的原始设计本身就接触过“腕表 + 产品 ID + 多语言标题”的场景，这比只在论文、餐馆或人物数据上验证的通用 EM 方法更贴近当前业务。

但 WDC benchmark 的“同商品”标签仍不等于当前业务的“必须同一 reference”，因此只能借架构，不能照抄标签语义。

---

## 3. 三个优化模块的代码级分析

### 3.1 Domain Knowledge Injection

论文思想是：让模型看到不仅是 token，还看到关键 token 的**类型**。

官方 `ProductDKInjector` 使用 spaCy NER，再做简单规则：

- `NORP / GPE / LOC / PERSON / PRODUCT` → `PRODUCT`；
- `DATE / QUANTITY / TIME / PERCENT / MONEY` → `NUM`；
- `token.like_num` 做数字规范化；
- 长度至少 7 且含数字的 token 前面加 `ID`。

例如：

```text
126610LN
```

可能被强调为：

```text
ID 126610LN
```

论文的思路对腕表非常对路，因为“型号 / reference”确实比营销词重要得多。

但官方实现中的这一条：

```python
elif len(token.text) >= 7 and any(ch.isdigit() for ch in token.text):
    res += 'ID ' + token.text + ' '
```

**对二奢业务不够安全**。

原因是以下字符串全部可能满足“长且含数字”：

- 品牌 reference；
- 平台商品 ID；
- seller SKU；
- 订单号；
- 内部库存号；
- 配件适配型号；
- 保卡序列号；
- 年份 + 批次编码。

如果全部统一打成 `ID`，模型会学到“两个长数字字符串相同就很重要”，但无法理解这些 ID 的业务角色不同。

因此应把 Ditto 的单一 `ID` 标签升级为**稳定语义角色**：

```text
REF_EXPLICIT
REF_TITLE
REF_OCR
PLATFORM_SKU
SELLER_SKU
SERIAL_NO
ACCESSORY_FOR_REF
BRAND
SERIES
YEAR
SIZE
```

平台原始字段名可以变化，但这些语义角色必须稳定。

### 3.2 Summarization：TF-IDF 不是默认安全选项

官方 `Summarizer` 会用训练、验证、测试集建立 TF-IDF vocabulary，然后在每条左右记录中保留高 TF-IDF token，控制在 `max_len` 内。

其基本逻辑是：

```text
长记录
  ↓
统计 token IDF
  ↓
优先保留高信息量 token
  ↓
压到 Transformer 最大长度
```

对大段商品描述有意义，但当前业务有两个问题。

#### 问题一：reference 不能被概率式摘要掉

reference 可能是：

```text
126610LN
IW371604
311.30.42.30.01.005
WSSA0037
```

它可能很稀有、通常 TF-IDF 很高，但“通常”不够。当前约束是**绝不能因为摘要算法改变 reference 证据**。

所以生产摘要必须分层：

```text
protected tokens
  - REF_EXPLICIT
  - REF_TITLE
  - REF_OCR
  - BRAND
  - identifier role tags
永远保留

soft tokens
  - title marketing words
  - description
  - seller prose
  - condition description
允许按预算裁剪
```

即：

```python
protected = extract_protected_spans(record)
remaining_budget = max_len - token_len(protected)
summary = tfidf_or_other_summary(soft_text, remaining_budget)
```

而不是让 reference 与“九成新”“热门”“专柜同款”竞争 token budget。

#### 问题二：官方实现建立 IDF 时读取 testset

`Summarizer.build_index()` 同时读取：

```python
trainset
validset
testset
```

对于论文复现也许是实现便利，但如果我们据此评估模型，严格来说会把测试集分布信息提前带入 preprocessing。

生产系统应当：

- IDF 只从训练窗口 / 历史线上数据构建；
- 对测试和未来增量只 transform；
- 版本化 `summarizer_version`；
- 更推荐 reference-aware truncation，而不是把 TF-IDF 当核心安全机制。

### 3.3 Data Augmentation + MixDA

官方 `augment.py` 支持多种操作：

```text
del
swap
drop_col
append_col
drop_token
drop_len
drop_sym
drop_same
ins
all
```

`all` 模式会随机多次选择：

```text
del / swap / drop_col / append_col
```

而 `DittoModel.forward()` 在同时拿到原样本 `x1` 与增强样本 `x2` 时，会计算两份 embedding，再通过 Beta 分布随机生成 `aug_lam`：

```python
enc = enc1 * aug_lam + enc2 * (1.0 - aug_lam)
```

这就是 MixDA 的关键。

它的好处是让模型对字段缺失、位置变化、局部噪声更鲁棒，很符合三平台数据稀疏问题。

但对当前 reference identity 有一个**反向风险**：

假设正样本：

```text
A: Rolex 126610LN
B: 劳力士 126610LN
label = 1
```

如果 `drop_col` 恰好删除 `REF_EXPLICIT` 字段，增强样本可能变成：

```text
A: Rolex Submariner black
B: 劳力士 潜航者 黑盘
label = 1
```

模型被迫学习：**即使 reference 消失，只靠系列、颜色和标题相似也可以判正**。

这恰好与业务规则相反。

因此建议实现 `ReferenceProtectedAugmenter`：

```python
PROTECTED_ROLES = {
    "REF_EXPLICIT",
    "REF_TITLE",
    "REF_OCR",
    "BRAND",
}
```

增强操作只能作用于：

- 营销词；
- condition；
- description；
- 非关键 title token；
- 字段顺序；
- 来源特有冗余字段。

不能：

- 删除 reference；
- 改写 reference；
- 把 reference suffix 当标点删除；
- 删除决定“不同 reference”的 hard-negative 差异位。

例如：

```text
126610LN vs 126610LV
```

`LN / LV` 是必须保护的判别信息，任何 normalization 都不能把它们抹平。

---

## 4. 官方训练与推理逻辑中不适合 precision-first 的部分

### 4.1 验证阈值优化的是 F1

官方 `evaluate()` 在没有传 threshold 时：

```python
for th in np.arange(0.0, 1.0, 0.05):
    pred = [1 if p > th else 0 for p in all_probs]
    new_f1 = metrics.f1_score(all_y, pred)
```

选择的是最佳 F1 阈值。

这不符合当前约束。

当 false positive 成本远大于 false negative 时，目标函数应该从：

```text
maximize F1
```

改成类似：

```text
maximize coverage / recall
subject to precision_lower_bound >= target
```

更极端地，AUTO_LINK 甚至可以不依赖 matcher threshold，而是先要求 deterministic reference evidence 完全一致，再让模型只做冲突检测 / review routing。

### 4.2 官方推理默认 0.5

`matcher.py::classify()`：

```python
if threshold is None:
    threshold = 0.5
```

然后输出：

```json
{
  "match": 1,
  "match_confidence": 0.97
}
```

但 softmax 0.97 并不等于“现实 precision 是 97%”。神经网络概率常常没有校准，且分布漂移后更不可靠。

所以生产输出必须从单一 `match` 改为：

```json
{
  "candidate_pair_id": "...",
  "model_score": 0.997,
  "calibrated_score": 0.993,
  "hard_gate": "PASS",
  "decision": "REVIEW",
  "reason_codes": [
    "REF_FROM_TITLE_ONLY",
    "NO_EXPLICIT_REF_CROSS_CHECK"
  ]
}
```

即便模型分数极高，只要硬证据不足，仍然 REVIEW。

### 4.3 Pairwise 分类不是最终数据模型

Ditto 输出的是：

```text
record_a ↔ record_b
```

当前业务更适合输出：

```text
record_a → reference_entity_123
record_b → reference_entity_123
```

这样三源聚合时不需要维护大量 pairwise 边，也避免传递误差：

```text
A ~ B
B ~ C
=> A ~ C
```

如果 B 是一条错边，普通 pair cluster 很容易整簇污染。

使用 canonical reference registry 后，实体中心天然是：

```text
reference_entity = (brand_id, canonical_reference)
```

在没有证据证明 reference 全局唯一前，**必须带 brand namespace**，否则不同品牌碰巧使用相同数字型号可能被错误合并。

---

## 5. 直接可落地的 Reference-First Ditto 架构

## 5.1 Layer 0：原始记录不可变存储

保留来源原始字段，永远不覆盖：

```sql
raw_product (
    source              text,
    source_product_id   text,
    fetched_at          timestamptz,
    title_raw           text,
    brand_raw           text,
    model_raw           text,
    description_raw     text,
    image_urls          jsonb,
    payload_raw         jsonb,
    content_hash        text,
    primary key (source, source_product_id)
)
```

所有抽取和匹配结论都应是派生表，方便以后升级规则后重跑。

### 5.2 Layer 1：Reference Evidence Extraction

对每条记录生成**多条证据**，而不是直接生成一个最终型号。

```sql
reference_evidence (
    source              text,
    source_product_id   text,
    evidence_id         uuid,
    raw_value           text,
    normalized_value    text,
    evidence_type       text,
    semantic_role       text,
    confidence          double precision,
    extractor_version   text,
    span_start          int,
    span_end            int,
    image_url           text,
    created_at          timestamptz
)
```

推荐 `evidence_type`：

```text
STRUCTURED_FIELD
TITLE_REGEX
TITLE_MODEL
DESCRIPTION_REGEX
IMAGE_OCR
MANUAL
```

推荐 `semantic_role`：

```text
REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
ACCESSORY_COMPATIBLE_REFERENCE
UNKNOWN_ID
```

这是对 Ditto `ID` domain knowledge 的关键升级。

### 5.3 Reference 抽取器使用分层策略

优先级建议：

```text
人工确认字段
   > 来源明确的 reference structured field
   > 品牌规则 + title 中唯一 reference
   > 多字段一致的 title/description reference
   > OCR reference + 文本 reference 一致
   > 单一 OCR / 弱模型候选
```

不要直接让一个 LLM 自由生成型号。更安全的做法是：

1. 先从文本中找候选 span；
2. 再做 role classification；
3. 再进入品牌字典 / reference registry 校验；
4. 原 span 始终保留以便审计。

### 5.4 Brand-aware Canonicalization

reference normalization 必须可逆、保守。

推荐先做低风险规范化：

```python
import re
import unicodedata


def canonicalize_ref(raw: str) -> str:
    s = unicodedata.normalize("NFKC", raw).upper().strip()
    s = s.replace("–", "-").replace("—", "-")
    s = re.sub(r"\s+", "", s)
    return s
```

但**不要全局直接删除所有 `/ . -`**。

因为某些品牌 reference 中标点或 suffix 可能有语义；更安全的是维护 brand-specific parser：

```python
canonicalize_ref(brand="OMEGA", raw="311.30.42.30.01.005")
canonicalize_ref(brand="ROLEX", raw="126610 LN")
canonicalize_ref(brand="IWC", raw="IW371604")
```

并同时保存：

```text
raw_ref
canonical_ref
parser_version
```

### 5.5 Layer 2：Canonical Reference Registry

```sql
reference_entity (
    reference_entity_id bigserial primary key,
    brand_id            bigint not null,
    canonical_ref       text not null,
    status              text not null,
    created_at          timestamptz not null,
    updated_at          timestamptz not null,
    unique (brand_id, canonical_ref)
)
```

这张表是系统核心。

后续“两个商品是否同一个商品”的判断可以退化为：

```text
product_a.reference_entity_id
    ==
product_b.reference_entity_id
```

不需要在每次查询时重新跑 pair matcher。

---

## 6. Blocking：千万级数据必须先缩小候选

Ditto 官方架构本身就把 Blocking 放在 matcher 前面。当前业务可以利用 reference 定义做比通用向量 blocking 更强的候选策略。

### 6.1 第一层：Exact Reference Blocking

最便宜也最安全：

```sql
SELECT reference_entity_id
FROM reference_entity
WHERE brand_id = :brand_id
  AND canonical_ref = :canonical_ref;
```

有高可信 reference 时基本是 O(log N) / 索引查找，而不是和 1000 万条商品逐一比较。

### 6.2 第二层：Reference Prefix / Format Blocking

只用于产生 REVIEW 候选，不可直接自动合并：

```text
同 brand
+ ref 共享长前缀
+ edit distance 很小
```

例如：

```text
126610LN
126610LV
```

它们正是**最危险的 hard negative**，应该进入“冲突 / 相邻 reference”候选集，而不是因为字符串很像就合并。

### 6.3 第三层：无 reference 记录的语义候选

如果某记录没有任何可信 reference，可按：

```text
brand
+ series
+ title embedding
+ size / material / gender / movement 等属性
```

召回少量候选 reference entity。

但这类 candidate 即使 Ditto 认为高度相似，也只能：

```text
REVIEW
```

不能 `AUTO_LINK`，除非后续从结构字段、文本或 OCR 获得 reference 硬证据。

---

## 7. Hard Gate：模型之前先执行不可越权规则

建议任何模型推理前先生成 `reason_codes`。

### 7.1 直接 REJECT 的典型情况

```text
BRAND_CONFLICT
EXPLICIT_REF_CONFLICT
MULTIPLE_INCOMPATIBLE_REFS
ACCESSORY_LISTING
REFERENCE_ROLE_CONFLICT
```

例如：

```text
A: Rolex Submariner 126610LN
B: Rolex Submariner 126610LV
```

即使：

- 标题 95% 一致；
- 图片视觉几乎一样；
- 系列一致；
- 尺寸一致；

仍应：

```text
REJECT: EXPLICIT_REF_CONFLICT
```

Ditto 不应被调用，或者调用结果只记录日志，绝不能翻转结论。

### 7.2 可以直接 AUTO_LINK 的极强证据

最初版本可以非常保守，例如同时满足：

```text
brand canonical 一致
AND 两侧均有高可信 reference
AND canonical_ref 完全一致
AND 无 accessory / SKU role 冲突
AND 无另一条高可信 conflicting reference
```

这甚至不需要 Ditto。

这是好事，因为当前业务目标不是“尽可能多用 AI”，而是“尽可能不误匹配”。

Ditto 的第一阶段价值应该是：

- 给 REVIEW 队列排序；
- 发现字段冲突；
- 在弱 reference 情况下辅助人审；
- 帮助抽取器发现难例；
- 在 shadow mode 统计模型是否有增益。

---

## 8. 改造后的 Ditto 输入协议

不建议直接序列化平台原字段名：

```text
COL goods_no VAL ...
COL style_id VAL ...
```

因为平台 schema 会变，而且 `goods_no` 在不同来源不一定表示同一语义。

建议先做语义映射：

```text
ROLE SOURCE VAL xiaoleian
ROLE BRAND VAL ROLEX
ROLE REF_EXPLICIT VAL 126610LN
ROLE REF_TITLE VAL 126610LN
ROLE SERIES VAL SUBMARINER
ROLE TITLE VAL 劳力士潜航者型黑盘日历型腕表...
ROLE CATEGORY VAL WATCH
```

右侧：

```text
ROLE SOURCE VAL sheDangJia
ROLE BRAND VAL ROLEX
ROLE REF_EXPLICIT VAL 126610LN
ROLE SERIES VAL SUBMARINER
ROLE TITLE VAL Rolex Submariner Date 41mm 黑水鬼...
```

然后组成 sequence pair。

推荐实现：

```python
ROLE_ORDER = [
    "SOURCE",
    "BRAND",
    "REF_EXPLICIT",
    "REF_TITLE",
    "REF_OCR",
    "SERIES",
    "CATEGORY",
    "TITLE",
]


def serialize(record):
    chunks = []
    for role in ROLE_ORDER:
        value = record.get(role)
        if value:
            chunks.append(f"ROLE {role} VAL {value}")
    return " ".join(chunks)
```

这种做法保留 Ditto “schema-agnostic serialization”的优点，同时避免把高风险 identifier 当普通字段。

---

## 9. Precision-first Decision Gate

推荐最终决策不是二分类，而是三分类状态机：

```text
AUTO_LINK
REVIEW
REJECT
```

### 9.1 建议规则

```python
def decide(evidence, ditto_score, calibrated_score):
    if evidence.brand_conflict:
        return "REJECT", "BRAND_CONFLICT"

    if evidence.high_conf_ref_conflict:
        return "REJECT", "REFERENCE_CONFLICT"

    if evidence.is_accessory:
        return "REJECT", "ACCESSORY_LISTING"

    if (
        evidence.both_have_high_conf_ref
        and evidence.canonical_ref_equal
        and not evidence.has_other_conflict
    ):
        return "AUTO_LINK", "EXACT_REFERENCE"

    if (
        evidence.at_least_one_weak_ref
        and evidence.no_hard_conflict
        and calibrated_score >= REVIEW_MODEL_THRESHOLD
    ):
        return "REVIEW", "MODEL_SUPPORTS_WEAK_REFERENCE"

    return "REVIEW", "INSUFFICIENT_REFERENCE_EVIDENCE"
```

最保守的 V1 甚至可以完全取消“模型触发 AUTO_LINK”。

这样可以保证模型失败时的 blast radius 很小：最多增加人工复核，不会直接污染实体簇。

### 9.2 为什么不建议“score > 0.999 就自动匹配”

因为 score 不是业务风险保证。

模型可能在以下场景非常自信地错：

- 同系列不同 suffix；
- 标题复制错误；
- 配件标题写兼容 reference；
- 卖家把多个 reference 堆在标题里；
- 平台 SKU 恰好长得像型号；
- 新品牌分布漂移；
- OCR 把 `O/0`、`I/1`、`S/5` 识错。

因此“极高模型概率”只能是附加证据，不能替代业务硬规则。

---

## 10. 几百对黄金标签应该怎样花

Spec 允许人工标注几百对。不要随机抽几百对。

建议主要花在**决策边界**上。

### 10.1 正样本

选：

```text
同 brand + 同 canonical_ref
但来源不同
且标题写法、语言、字段缺失模式差异大
```

例如：

- 中文 vs 英文；
- reference 独立字段 vs 藏标题；
- 一侧只有型号，另一侧有完整系列；
- 标点格式不同；
- 一侧 OCR 才能看到 reference。

### 10.2 Hard Negative 是最重要的

优先标：

```text
同品牌
同系列
同尺寸
标题高度相似
图片高度相似
但 reference 不同
```

例如：

```text
126610LN vs 126610LV
IW371604 vs IW371605
```

还应加入：

- 手表 vs 原装表带；
- 手表 vs 表盒 / 保卡；
- “适配 126610LN”的配件 vs 126610LN 手表；
- 平台 SKU 相同格式但跨来源无意义；
- title 中同时出现多个兼容 reference。

### 10.3 不要随机 pair split

随机切分容易让同一个 reference 同时出现在 train 和 test，造成严重过乐观。

建议：

```text
split by canonical_reference
```

再加：

```text
leave-one-source-out
leave-one-brand-out
time-based split
```

评估真正的新来源、新品牌、新型号。

---

## 11. 训练策略：保留 Ditto 优点，但改掉危险默认值

### 11.1 Backbone

Ditto 架构不依赖具体 LM。生产可使用中文 / 多语言 Transformer 做 sequence-pair classification。

重点不是追求最大模型，而是：

- 能理解中英混合标题；
- GPU 批量吞吐稳定；
- 可本地部署；
- 能输出 logits 供后续校准；
- 模型版本可复现。

因为 blocking 后 candidate 已经很少，一个中小型 encoder 通常比“每个 pair 调闭源大 LLM”更可控。

### 11.2 Loss

基础可以继续 CrossEntropy。

真正的 precision-first 主要通过：

- 数据分布；
- hard negatives；
- hard gate；
- calibration；
- selective decision；

来实现，而不是期待换一个 loss 就能“保证不误匹配”。

### 11.3 Protected Augmentation

只对 soft fields 做：

```text
marketing token deletion
word order perturbation
source-specific field dropout
non-ID punctuation noise
```

reference 及其上下文角色不可删除。

Hard negative 的不同 reference 字符更不能被归一化掉。

### 11.4 Threshold Calibration

弃用 Ditto 的“最佳 F1 threshold”。

建议：

1. 单独留 calibration set；
2. 对 logits 做 temperature scaling / isotonic 一类校准；
3. 在高分区间统计真实 precision；
4. 选择阈值时看 precision 的保守下界，而不是点估计；
5. 分来源、分品牌监控 calibration drift。

但再次强调：即使校准很好，V1 AUTO_LINK 仍建议只由 exact reference hard rule 触发。

---

## 12. 图片应该如何加入，而不破坏 reference-first 原则

Ditto 本身不处理图像。

当前 Spec 明确说有图片可用。最安全的用法不是“图片相似 = 同款”，而是：

### 12.1 OCR 作为 reference 证据来源

优先 OCR：

- 表背刻字；
- 保卡；
- 吊牌；
- 标签；
- 证书。

生成：

```text
REF_OCR = 126610LN
ocr_confidence = ...
image_url = ...
```

如果：

```text
REF_TITLE == REF_OCR == REF_EXPLICIT
```

证据强度大幅提升。

### 12.2 视觉相似只能做辅助

同系列不同 reference 往往外观极近，所以 CLIP / perceptual similarity 不能成为自动合并依据。

视觉最适合：

- 识别明显品类冲突；
- 手表 vs 表带 / 盒证；
- 为人工 REVIEW 排序；
- 找重复图片 / 同一卖家复用图；
- 辅助 OCR 选图。

不能：

```text
image similarity high → AUTO_LINK
```

---

## 13. 增量架构：不需要每天重跑千万级全量 pair

当前数据持续增量更新，推荐事件级流程：

```text
new / changed product
       │
       ▼
content_hash 是否变化
       │
       ├── no → skip
       ▼
extract reference evidence
       │
       ▼
canonicalize
       │
       ▼
exact registry lookup
       │
       ├── strong exact → AUTO_LINK
       ├── conflict → REJECT
       └── weak / missing
                 │
                 ▼
          candidate retrieval
                 │
                 ▼
            Ditto batch
                 │
                 ▼
              REVIEW
```

只有这些情况需要重算历史记录：

- extractor_version 升级；
- canonicalizer_version 升级；
- brand alias 字典变化；
- reference registry 修订；
- 人工纠正某一批错误 reference。

这样 1000 万存量不会因为每天新增几万条而反复做全库匹配。

---

## 14. 推荐数据库与服务拆分

V1 不需要过度微服务化。

一个实用部署可以是：

```text
现有爬虫
   │
   ▼
Raw DB / Object Storage
   │
   ▼
Reference Extractor Workers (Python)
   │
   ▼
PostgreSQL Reference Registry
   │
   ├── exact lookup
   │
   └── review candidates
            │
            ▼
      Ditto Matcher Service
       (GPU batch inference)
            │
            ▼
        Decision Service
            │
      ┌─────┴─────┐
      ▼           ▼
  entity links   review queue
```

PostgreSQL 对 `(brand_id, canonical_ref)` 唯一索引足够支撑千万级 reference / product linkage；如果后续分析日志特别大，可再把事件和模型日志放 ClickHouse / 数据湖，不必一开始就上复杂流式栈。

如果现有抓取链路本身已经有 Kafka，可直接接入；没有的话不建议为了这个需求单独引入 Kafka。

---

## 15. 关键数据表

### 15.1 product_reference_link

```sql
product_reference_link (
    source                text,
    source_product_id     text,
    reference_entity_id   bigint,
    decision              text,
    decision_reason       text[],
    evidence_version      text,
    matcher_version       text,
    decided_at            timestamptz,
    reviewed_by           text,
    primary key (source, source_product_id)
)
```

### 15.2 match_decision_log

```sql
match_decision_log (
    decision_id            uuid primary key,
    source                 text,
    source_product_id      text,
    candidate_reference_id bigint,
    hard_gate_result       text,
    reason_codes           text[],
    raw_model_score        double precision,
    calibrated_score       double precision,
    final_decision         text,
    extractor_version      text,
    canonicalizer_version  text,
    matcher_version        text,
    created_at             timestamptz
)
```

必须保留 decision log，才能回答：

> “为什么这两条被合并？”

而不是只剩一个不可解释的 `match=1`。

---

## 16. 线上评估指标应该换掉 F1 中心主义

Ditto 论文主要报告 F1，这是通用 EM 合理指标，但当前项目优先级不同。

### 16.1 第一指标：AUTO_LINK Precision

```text
AUTO_LINK 中人工抽检 / 黄金集真正正确的比例
```

并且最好报告置信下界，而不只是：

```text
99.9%
```

### 16.2 第二指标：False Positive Count

直接追踪：

```text
FP / day
FP / source
FP / brand
FP / extractor_version
FP / matcher_version
```

### 16.3 第三指标：Coverage

在 precision 达标后，再看：

```text
AUTO_LINK coverage
REVIEW rate
UNRESOLVED rate
```

### 16.4 分层指标

至少分：

```text
explicit reference
reference from title
reference from OCR
no reference
new brand
new reference
accessory-like listing
```

否则总体数字会掩盖最危险的长尾区域。

---

## 17. 上线顺序

### Phase 1：Deterministic Reference MVP

只做：

```text
brand normalization
identifier role classification
reference extraction
reference canonicalization
exact registry match
conflict reject
review queue
```

目标：先建立**不会被模型轻易污染**的正确数据骨架。

### Phase 2：Ditto Shadow Mode

Ditto 对 REVIEW candidates 打分，但：

```text
不自动修改任何实体链接
```

比较：

- 高分 candidate 人审 precision；
- 哪些 hard negative 最常错；
- 不同来源 / 品牌的 calibration；
- 有模型 vs 无模型的人审排序效率。

### Phase 3：模型辅助 REVIEW

使用 Ditto：

- REVIEW 排序；
- 建议候选 reference；
- 解释关键冲突字段；
- active learning 选最值得标注的 pair。

### Phase 4：有限扩大 AUTO_LINK

只有在离线黄金集 + shadow 线上抽检显示长期稳定后，才考虑让模型参与更弱证据的自动放行。

即使到这一步，reference conflict 仍然是不可覆盖的 hard reject。

---

## 18. 最值得从 Ditto 直接复用的东西

| Ditto 设计 | 是否复用 | 当前项目改造 |
|---|---:|---|
| `COL/VAL` 序列化 | 是 | 改成稳定 `ROLE/VAL` 语义角色 |
| 预训练 Transformer pair classifier | 是 | 作为 secondary matcher，不做最终事实源 |
| Schema 不要求预对齐 | 是 | 很适合三源字段不一致 |
| Domain Knowledge | 强烈复用 | `ID` 升级为 reference / SKU / serial 等细粒度角色 |
| Number normalization | 部分复用 | reference 单独走 brand-aware parser，不能粗暴数字归一化 |
| TF-IDF summarization | 谨慎 | reference protected，IDF 不读测试/未来数据 |
| `del/swap/drop_col` augmentation | 部分复用 | 只能动 soft fields，reference 永远 protected |
| MixDA | 可实验 | 只在 protected augmentation 后使用 |
| F1 threshold tuning | 不复用 | 改 precision-constrained selective gate |
| 默认 0.5 推理阈值 | 不复用 | 模型不直接控制 AUTO_LINK |
| `match_confidence` | 仅日志 | 必须校准，不能当业务正确率 |
| Blocking + Matcher 分层 | 强烈复用 | Blocking 以 exact canonical reference 为核心 |

---

## 19. 与当前 Spec 最契合的一条实现路线

如果只选一条近期可以开始编码的路线，我建议：

```text
Step 1
给三个来源做 source adapter，统一输出：
brand_raw / explicit_model_fields / title / description / images

Step 2
实现 IdentifierCandidateExtractor：
结构字段 + title regex + title model + OCR

Step 3
实现 IdentifierRoleClassifier：
REFERENCE / PLATFORM_SKU / SELLER_SKU / SERIAL / ACCESSORY_REF

Step 4
实现 BrandAwareReferenceCanonicalizer

Step 5
建 PostgreSQL reference_entity 唯一索引：
(brand_id, canonical_ref)

Step 6
V1 只允许 high-confidence exact reference AUTO_LINK

Step 7
从 REVIEW 中人工标 300–500 个 hard pairs

Step 8
基于 Ditto 架构训练 pair reviewer：
ROLE serialization + multilingual encoder + protected augmentation

Step 9
Ditto 只以 shadow / review ranking 上线

Step 10
持续记录 FP，并按 source / brand / evidence type 调阈值与规则
```

这样做有三个明显优势：

1. **先用业务定义拿到最强 precision，不被模型能力绑架；**
2. **Ditto 仍然能充分发挥在脏文本、字段缺失、异构 schema 上的能力；**
3. **系统复杂度和风险随阶段逐步增加，任何阶段都可以停在一个可用版本。**

---

## 20. 一个具体例子

输入：

```json
{
  "source": "leixiaoan",
  "brand": "劳力士",
  "model": null,
  "title": "劳力士潜航者黑水鬼 41mm 126610LN 全套"
}
```

抽取：

```json
{
  "brand_id": "ROLEX",
  "reference_candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "role": "REFERENCE",
      "evidence_type": "TITLE_REGEX",
      "confidence": 0.98
    }
  ]
}
```

另一个来源：

```json
{
  "source": "watchhome",
  "brand": "Rolex",
  "reference": "126610LN",
  "title": "Submariner Date 41mm"
}
```

抽取：

```json
{
  "brand_id": "ROLEX",
  "canonical_ref": "126610LN",
  "evidence_type": "STRUCTURED_FIELD",
  "confidence": 1.0
}
```

Hard Gate：

```text
brand equal = yes
canonical ref equal = yes
conflicting high-confidence ref = no
accessory = no
```

可直接：

```text
AUTO_LINK(EXACT_REFERENCE)
```

如果第二条改成：

```text
126610LV
```

即使 Ditto score 是 `0.9999`，仍然：

```text
REJECT(REFERENCE_CONFLICT)
```

这才符合当前 Spec。

---

## 21. 对 Ditto 的最终评价

Ditto 值得参考，但最值得参考的不是它的最终 `match` 标签，而是它把 Entity Matching 工程化拆解的方式：

```text
Blocking
→ structured serialization
→ domain knowledge
→ pretrained encoder
→ difficult-example training
→ batch matcher
```

对于“雷小安 × 腕表之家 × 奢当家”的二奢腕表系统，建议进一步做一次业务语义收缩：

```text
通用 Entity Matching
        ↓
Reference Evidence Linking
        ↓
Canonical Reference Registry
        ↓
Selective / Abstaining Matcher
```

**最终推荐：先实现 Reference Registry + Hard Gate，再接 Ditto secondary matcher。**

如果顺序反过来，直接训练一个“这两条是否同款”的强模型，再试图通过阈值保证绝不误匹配，会把最关键的业务定义埋进概率模型；而把 reference 先提升为一等实体后，系统的自动链接条件、失败模式、人工复核和审计链路都会清晰得多。

因此 Ditto 在本需求里的最佳角色是：

> **高质量、低成本、可批量部署的疑难候选复核器，而不是 reference identity 的最终裁判。**
