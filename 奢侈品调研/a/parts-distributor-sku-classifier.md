# parts-distributor-sku-classifier：从 SKU/MPN 字符分类器到二奢腕表 Reference 角色识别门禁

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **parts-distributor-sku-classifier** 进行深入分析。

- 项目：<https://github.com/pcbng/parts-distributor-sku-classifier>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
- 本目录此前已经分析过：`DeepBlocker`；本次已排除，不重复分析。

这个项目解决的问题非常小，但和当前 Spec 中最危险的一类 false positive 几乎同构：

> **一个“长得像型号”的字符串，到底是品牌/厂商真正的型号（MPN / Reference），还是平台自己的 SKU / 货号？**

原项目用字符级 LSTM 把电子元器件编号分成三类：

1. Manufacturer Part Number（MPN）
2. Mouser SKU
3. Digi-Key SKU

最值得借鉴的并不是“LSTM 本身”，而是一个更重要的工程分层：

> **在做实体匹配之前，先判断 identifier 的“角色”。只有被确认是品牌 Reference 的 identifier，才允许进入 Reference Entity Linking；平台 SKU、店铺货号、序列号、兼容型号等全部必须被隔离。**

对当前需求，推荐直接把这个思想升级成一层 **Identifier Role Gate（编号角色门禁）**：

```text
商品字段 / 标题 / OCR
        │
        ▼
Identifier Candidate Extraction
        │
        ▼
Identifier Role Classification
  ├─ BRAND_REFERENCE
  ├─ PLATFORM_SKU
  ├─ SELLER_SKU
  ├─ SERIAL_NUMBER
  ├─ INTERNAL_ID
  ├─ COMPATIBILITY_REFERENCE
  └─ UNKNOWN
        │
        ▼
只有 BRAND_REFERENCE + SELF（属于当前商品）
才进入 Canonical Reference Resolver
        │
        ▼
严格 Reference Entity Linking
        │
        ▼
同一 ref_entity_id => 同一个商品
```

并且必须对原项目做一个关键反转：

> **Reference 不能像原项目的 MPN 一样成为“剩下的都算它”的 catch-all 类。Reference 必须是高置信、可证明的正类；不确定的字符串默认 UNKNOWN / ABSTAIN。**

这是本文最核心的落地结论。

---

## 1. 为什么这个项目和当前 Spec 高度相关

当前 Spec 的约束是：

- 数据来自雷小安、腕表之家、奢当家三个来源；
- 规模约 100 万～1000 万；
- 持续增量；
- “同一个商品”已经明确等价为 **同一 Reference Number / 型号**；
- Reference 有时在独立字段，有时埋在标题；
- 可以用图片；
- **precision 极端优先，宁可漏匹配也不能误匹配。**

因此真正的核心问题并不是传统的：

```text
商品 A 和商品 B 文本/图片像不像？
```

而是：

```text
商品 A 中出现的哪个字符串，才是“当前商品自身的品牌 Reference”？
这个 Reference 能否规范化并链接到唯一的 Reference Entity？
```

例如一条标题里可能同时出现：

- 品牌 Reference；
- 平台商品 ID；
- 商家 SKU；
- 序列号；
- 尺寸参数；
- 年份；
- “适用/兼容某型号”的另一个 Reference；
- 表带、盒证、配件所对应的主表型号。

只要把其中一个非 Reference 编号误当成 Reference，后续 exact match 反而会非常“自信”地制造错误实体合并。

所以，当前系统需要的不是更强的模糊匹配，而是更强的 **编号语义隔离**。

原项目正好提供了一个非常干净的最小实验：仅看字符序列，也能学到 SKU 的格式语法；但它同时暴露了开放世界分类和 false positive 的根本风险。

---

## 2. 原项目究竟做了什么

README 对问题的定义很清楚：电子元器件分销商会给同一厂商零件分配自己的 SKU。例如：

```text
厂商 MPN:      SN74LVC541APWR
Digi-Key SKU:  296-8521-1-ND
Mouser SKU:    595-SN74LVC541APWR
```

人一眼能发现：

- Digi-Key 经常有 `-ND` 后缀；
- Mouser 经常是若干数字前缀 + `-` + MPN；
- MPN 本身则来自大量不同厂商，格式最杂。

作者也明确说：正则完全可以解决一部分问题，但该仓库故意用一个 LSTM 作为机器学习玩具问题。

这点对当前需求很重要：

> 原项目本质上不是“语义理解模型”，而是一个 **字符格式语法学习器（format grammar recognizer）**。

而 Reference / SKU 这类 identifier 恰好非常适合字符级建模。

---

## 3. 原项目的数据与训练流程

### 3.1 三分类数据

训练数据只有两个核心字段：

```text
partnum, class
```

三类定义：

```text
0 = Manufacturer Part Number (MPN)
1 = Mouser SKU
2 = Digi-Key SKU
```

原始数据类别不平衡，Digi-Key 样本更多。作者先取最小类别的样本数作为上限：

```python
limit_rows_per_class = int(df_raw.groupby('class').count().min())
```

仓库 notebook 中该值是：

```text
4711
```

于是每类截取 4711 条，合计：

```text
4711 × 3 = 14133
```

随后 shuffle，再随机按 80% / 20% 切 train / validation：

```python
np.random.seed(20181203)
df['dataset'] = np.random.choice(
    ['train', 'val'],
    size=len(df),
    replace=True,
    p=[0.80, 0.20]
)
```

实际 notebook 输出：

```text
Train on 11344 samples, validate on 2789 samples
```

### 3.2 字符级编码

项目没有分词，也没有 WordPiece/BPE，而是直接建立字符字典：

```python
unique_chars = set()
for s in df['partnum'].values:
    unique_chars |= set(c for c in s)

partnum_dict = {c: i+1 for i, c in enumerate(unique_chars)}
```

每个编号变成整数序列：

```text
"595-TPS..." -> [10, 46, 10, 9, ...]
```

然后使用 post-padding 补齐到 `maxlen + 1`：

```python
sequence.pad_sequences(
    df['x'],
    maxlen=maxlen+1,
    padding='post'
)
```

0 既作为 padding，也事实上形成“序列结束”信号。

仓库保存的 Keras model config 显示字符词表输入维度是 52：

```text
Embedding input_dim = 52
```

也就是一个极小字符表。

### 3.3 模型结构

模型只有三层：

```text
Character IDs
    │
    ▼
Embedding(vocab=52, dim=32)
    │
    ▼
LSTM(hidden=32,
     dropout=0.2,
     recurrent_dropout=0.2)
    │
    ▼
Dense(3, softmax)
    │
    ▼
MPN / Mouser SKU / Digi-Key SKU
```

代码基本就是：

```python
model = Sequential()
model.add(Embedding(len(partnum_dict)+1, 32))
model.add(LSTM(32, dropout=0.2, recurrent_dropout=0.2))
model.add(Dense(3, activation='softmax'))

model.compile(
    loss='categorical_crossentropy',
    optimizer='adam',
    metrics=['accuracy']
)
```

训练配置：

```text
batch_size = 32
epochs     = 7
```

这是一个非常小的网络，参数量和推理成本都很低。

---

## 4. 原项目的效果，以及真正值得关注的不是 Accuracy

Part 1 notebook 的一次可见训练结果是：

```text
整体 validation accuracy: 98.49%

MPN:          96.19%
Mouser SKU:   99.26%
Digi-Key SKU: 100.00%
```

Part 2 载入仓库中保存的权重后得到的数值略有不同：

```text
整体 accuracy: 99.01%

MPN:          97.49%
Mouser SKU:   99.51%
Digi-Key SKU: 100.00%
```

说明模型确实非常擅长识别有稳定前后缀格式的 SKU。

但对当前需求，**98%/99% accuracy 完全不是可以接受的指标**。

Part 2 的 confusion graph 显示一组很关键的错误分布：

```text
真实 MPN -> 预测 MPN          96.187%
真实 MPN -> 预测 Mouser SKU   3.050%
真实 MPN -> 预测 Digi-Key SKU 0.763%

真实 Mouser SKU -> MPN        0.737%
真实 Mouser SKU -> Mouser     99.263%

真实 Digi-Key SKU -> Digi-Key 100.000%
```

换句话说：

> 即使整体 Accuracy 接近 99%，某个业务上最危险的方向依然可以有 3% 左右的错误。

对于“绝对不能误匹配”的业务，这已经足够让整个系统不可用。

---

## 5. 原项目暴露出的四个关键问题

### 5.1 MPN 是一个 noisy catch-all，导致开放世界假阳性

作者在 Part 2 里直接指出：

> MPN 类本质上是一个没有明确格式规律的 noisy catch-all。

Mouser、Digi-Key SKU 有很强的可学习格式；但 MPN 是“其他所有厂商编号”的合集。

这会产生一个非常典型的问题：

```text
已知格式类：容易学
未知/长尾类：没有封闭边界
```

如果强迫 softmax 必须在三类里选一个，它一定会把未知东西硬塞进某一类。

这就是当前腕表系统最不能照搬的部分。

如果把腕表任务定义成：

```text
REFERENCE
PLATFORM_SKU
OTHER
```

然后把所有识别不了的东西都训练成 `REFERENCE` 或 `OTHER`，最终还是会出现 open-set false positive。

所以生产系统必须有：

```text
UNKNOWN / ABSTAIN
```

而且自动匹配路径默认应该是拒识，而不是“总要选一个标签”。

### 5.2 标签污染会被模型忠实学进去

Part 2 还发现：训练和验证数据中，有部分 **Digi-Key SKU 被错误标成了 MPN**。

作者检查错误样本后明确写道：

```text
Looks like both our training and validation data contains
some Digi-Key SKUs disguised as MPNs.
```

这是一个对当前项目极重要的警告。

三平台数据里的字段名很可能并不可靠：

- `model` 不一定就是品牌 Reference；
- `sku` 不一定都是平台 SKU；
- 标题中出现的编号也不一定属于当前商品；
- OCR 可能把 `O/0`、`I/1`、`S/5` 混淆；
- 配件页面可能出现它“适用的主表 Reference”。

如果直接把弱标签当真，模型会把数据源历史错误固化成规则。

因此：

> **弱监督数据只能用于预训练/扩充，最终阈值和自动放行策略必须依赖人工黄金集和可解释硬约束。**

### 5.3 只看字符串，不知道“这个编号属于谁”

原项目只解决：

```text
这个字符串像哪种编号？
```

但腕表场景还需要第二个问题：

```text
这个 Reference 是当前商品自身的 Reference，
还是标题里提到的“兼容/适用/相关型号”？
```

例如：

```text
“适用 Rolex 126610LN 的表带”
```

`126610LN` 确实是一个合法 Reference，但它不是当前“表带商品”的 Reference。

如果只有 Role Classifier，它会正确识别 `126610LN` 是品牌 Reference，随后却错误地把表带链接到该 Reference Entity。

所以腕表系统必须把问题拆成两层：

```text
Identifier Role:  这是什么编号？
Mention Relation: 这个编号是否描述当前商品本身？
```

后者必须支持：

```text
SELF
COMPATIBILITY
RELATED
UNKNOWN
```

### 5.4 随机 train/val 会高估跨来源、跨时间泛化

原项目只在同一数据分布里随机切 80/20。

当前系统面对的是：

- 三个来源；
- 不同品牌；
- 新增品牌；
- 新增 Reference；
- 数据格式不断变化；
- OCR 噪声；
- 新商家/新 SKU 规则。

因此评测不能只做随机切分。

至少要做：

```text
按 source holdout
按 brand holdout
按时间 holdout
按 reference holdout
hard-negative 专项集
```

否则一个“记住雷小安 SKU 前缀”的模型可能在随机验证集上看起来 99.9%，一上线遇到新格式就失效。

---

## 6. 对当前需求的正确问题重构

当前 Spec 已经把“同一个商品”定义成“同一个 Reference”。

因此推荐把整个系统重构为：

```text
通用 Entity Matching
        ↓
Reference Entity Linking
```

即每条商品只需要回答：

```text
这个商品能否高置信链接到唯一 ref_entity_id？
```

如果两个来源的商品都成功链接到同一个 `ref_entity_id`：

```text
product_a.ref_entity_id == product_b.ref_entity_id
```

那么才认为它们匹配。

不需要让 pairwise 模型直接判断商品对。

这一重构把复杂度从潜在的：

```text
O(N × M)
```

变成接近：

```text
O(N)
```

每条商品独立抽取、规范化、查 Reference Dictionary 即可。

---

## 7. 推荐的生产架构

### 7.1 总体数据流

```text
雷小安 ─┐
腕表之家 ├─> Source Adapter
奢当家 ─┘        │
                  ▼
          Raw Product Record
                  │
        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
 Structured    Title      Images
  Fields       Text         │
        │         │         ▼
        │         │        OCR
        │         │         │
        └────┬────┴────┬────┘
             ▼         ▼
      Identifier Candidate Extractor
                  │
                  ▼
       Identifier Role Classifier
                  │
                  ▼
       Mention Ownership Verifier
        SELF / COMPAT / UNKNOWN
                  │
                  ▼
       Brand-aware Normalizer
                  │
                  ▼
       Canonical Reference Resolver
                  │
                  ▼
          Strict Decision Gate
        ┌─────────┴──────────┐
        ▼                    ▼
     ACCEPT                ABSTAIN
        │                    │
        ▼                    ▼
Reference Entity        Manual Review
        │                    │
        └──────────┬─────────┘
                   ▼
             Feedback / Labels
```

### 7.2 Identifier Role 不是 Matcher，而是 Gate

建议至少定义下面这些角色：

```text
BRAND_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
INTERNAL_ID
COMPATIBILITY_REFERENCE
NON_IDENTIFIER
UNKNOWN
```

其中：

- `BRAND_REFERENCE`：品牌官方 Reference / 型号；
- `PLATFORM_SKU`：雷小安/腕表之家/奢当家的平台货号；
- `SELLER_SKU`：店铺自己的内部编号；
- `SERIAL_NUMBER`：唯一序列号，不代表型号；
- `INTERNAL_ID`：数据库 ID、订单号、抓取 ID 等；
- `COMPATIBILITY_REFERENCE`：合法 Reference，但只表示“适用/兼容/关联”；
- `UNKNOWN`：任何不确定内容。

自动路径只允许：

```text
role == BRAND_REFERENCE
AND relation == SELF
```

继续向下。

---

## 8. Role Classifier 的建议实现

### 8.1 第一版不要直接上大模型

identifier 很短，真正有价值的信号主要是：

- 字符模式；
- 长度；
- 前缀/后缀；
- 数字/字母比例；
- 分隔符；
- 来源；
- 字段名；
- 左右上下文；
- 品牌。

这些信号用一个极小模型就足够。

原项目的 LSTM 可以作为 baseline，但生产上更推荐一个 **Char-CNN + Metadata Embedding**：

```text
identifier characters
      │
      ▼
Char Embedding 32d
      │
      ├─ Conv1D(k=2, 64)
      ├─ Conv1D(k=3, 64)
      ├─ Conv1D(k=4, 64)
      └─ Conv1D(k=5, 64)
              │
         GlobalMaxPool
              │
      ┌───────┴────────┐
      ▼                ▼
 source_emb        field_emb
 brand_emb         context features
      └───────┬────────┘
              ▼
          MLP 256 -> 128
              │
              ▼
       Role Softmax / logits
```

为什么这里比 LSTM 更推荐 Char-CNN：

- identifier 很短；
- SKU 模式大量由局部 n-gram、prefix、suffix 决定；
- CNN CPU 推理并行度高；
- 1000 万级离线批推更便宜；
- 更容易导出 ONNX；
- 模型足够小，可以常驻每个 worker。

如果希望最小改动，也可以继续使用：

```text
Embedding(32) -> BiLSTM(64) -> MLP
```

但必须加入 source / field / brand / context，而不能像原项目一样只输入字符串。

### 8.2 输入特征

建议模型输入至少包含：

```text
raw_token
normalized_surface
source_id
field_name
brand_id
product_category
left_context
right_context
position_in_title
has_letter
has_digit
digit_ratio
hyphen_count
dot_count
slash_count
length
known_platform_pattern_hit
known_reference_dictionary_hit
```

这里 `known_reference_dictionary_hit` 只作为辅助特征，最终决策仍由后面的 Resolver 决定。

### 8.3 不要让 softmax 强迫输出

模型可以输出 logits，但线上必须加 reject policy：

```python
if p_reference < T_reference:
    return UNKNOWN

if p_reference - second_best < T_margin:
    return UNKNOWN

if token_is_ood:
    return UNKNOWN
```

并且：

> 即使 `p_reference = 0.9999`，也不能仅凭模型概率自动建立最终匹配。

模型概率只代表“像 Reference”，不是“它就是当前商品唯一正确 Reference”。

---

## 9. 必须额外增加 Ownership / Relation 判定

Role Classifier 只能回答：

```text
126610LN 像不像一个品牌 Reference？
```

但当前系统还要回答：

```text
126610LN 是否描述当前售卖商品本身？
```

因此建议增加一个轻量 Relation 层。

### 9.1 第一阶段优先用规则

腕表/配件标题里很多危险模式是可解释的：

```text
适用 <REF>
兼容 <REF>
for <REF>
表带适配 <REF>
盒证 <REF>
附件 <REF>
同款 <REF>
参考 <REF>
```

只要 token 出现在明确的 compatibility/context pattern 里：

```text
relation = COMPATIBILITY
```

直接 veto。

### 9.2 第二阶段再训练 context classifier

对规则覆盖不到的情况，可以输入：

```text
[token 左 32 字符] + [TOKEN] + [token 右 32 字符]
```

训练一个小型 context classifier：

```text
SELF
COMPATIBILITY
RELATED
UNKNOWN
```

这里宁可把 SELF 判成 UNKNOWN，也不能把 COMPATIBILITY 判成 SELF。

---

## 10. Reference Normalization 必须“保守”，不能过度清洗

很多实体匹配系统会做：

```text
去空格
去横杠
去点号
去所有非字母数字
```

对 Reference 场景这很危险。

推荐同时保存三层值：

```text
raw_value
surface_normalized
brand_lookup_key
```

### 10.1 Surface Normalization

只做基本无损或近似无损处理：

```text
Unicode NFKC
全角 -> 半角
upper-case
trim
连续空格压缩
各种 dash 统一成 ASCII '-'
```

### 10.2 Brand-aware Lookup Key

只有在品牌规则明确证明分隔符只是展示格式时，才生成 lookup alias。

例如不要全局假设：

```text
ABC-123 == ABC123
```

而应该：

```text
brand_rule_version(role_brand, raw)
    -> one or more safe aliases
```

每次规范化都记录：

```text
normalizer_version
transform_steps
```

这样出现误合并时可追踪到底是哪条规范化规则导致。

---

## 11. Canonical Reference Dictionary：系统真正的中心

建议建立全局 Reference Entity 表，而不是把 Reference 当普通字符串。

### 11.1 reference_entity

```sql
CREATE TABLE reference_entity (
    ref_entity_id      BIGSERIAL PRIMARY KEY,
    brand_id           BIGINT NOT NULL,
    canonical_ref      TEXT NOT NULL,
    canonical_key      TEXT NOT NULL,
    status             TEXT NOT NULL,
    source_of_truth    TEXT,
    created_at         TIMESTAMP NOT NULL,
    updated_at         TIMESTAMP NOT NULL,
    UNIQUE (brand_id, canonical_key)
);
```

### 11.2 reference_alias

```sql
CREATE TABLE reference_alias (
    alias_id            BIGSERIAL PRIMARY KEY,
    ref_entity_id       BIGINT NOT NULL,
    alias_raw           TEXT NOT NULL,
    alias_key           TEXT NOT NULL,
    normalizer_version  TEXT NOT NULL,
    evidence_source     TEXT,
    UNIQUE (ref_entity_id, alias_key)
);

CREATE INDEX idx_reference_alias_lookup
ON reference_alias(alias_key);
```

### 11.3 product_reference_assignment

```sql
CREATE TABLE product_reference_assignment (
    product_id          BIGINT PRIMARY KEY,
    source_id           TEXT NOT NULL,
    ref_entity_id       BIGINT,
    decision            TEXT NOT NULL, -- ACCEPT / ABSTAIN / REVIEW / REJECT
    raw_reference       TEXT,
    normalized_key      TEXT,
    role                TEXT,
    relation            TEXT,
    role_score          DOUBLE PRECISION,
    evidence_json       JSONB NOT NULL,
    model_version       TEXT,
    rule_version        TEXT,
    normalizer_version  TEXT,
    decided_at          TIMESTAMP NOT NULL
);

CREATE INDEX idx_product_ref_entity
ON product_reference_assignment(ref_entity_id)
WHERE decision = 'ACCEPT';
```

这样跨来源匹配根本不需要再两两算相似度：

```sql
SELECT a.product_id, b.product_id
FROM product_reference_assignment a
JOIN product_reference_assignment b
  ON a.ref_entity_id = b.ref_entity_id
WHERE a.decision = 'ACCEPT'
  AND b.decision = 'ACCEPT'
  AND a.source_id <> b.source_id;
```

---

## 12. Strict Decision Gate：最终自动放行条件

当前需求最关键的组件不是模型，而是 **Decision Gate**。

建议证据按层级处理。

### 12.1 自动 ACCEPT 的强证据路径

#### 路径 A：可信结构化字段

```text
显式 reference 字段
+ role == BRAND_REFERENCE
+ relation == SELF
+ brand 一致
+ dictionary exact resolve 唯一
+ 无任何冲突 Reference
=> ACCEPT
```

#### 路径 B：多通道独立一致

当没有可信结构化字段时：

```text
Title 提取 REF X
+ OCR 提取 REF X
+ 二者独立
+ dictionary exact resolve 唯一
+ brand 一致
+ 无冲突
=> ACCEPT
```

也可以把两个独立文本字段一致作为类似强证据。

### 12.2 必须 ABSTAIN / REVIEW 的情况

以下情况一律不自动匹配：

```text
出现两个不同的高可信 Reference
只有 embedding/string similarity
只有图片视觉相似
只有一个低可信 title token
Role 模型置信度不够
Relation != SELF
品牌冲突
Reference 不在 dictionary 且来源不够权威
OCR 与结构化字段冲突
title 与 structured_ref 冲突
```

特别强调：

> **冲突时不要“投票选一个”，要拒识。**

因为当前业务允许漏匹配，不允许误匹配。

---

## 13. 一个可直接实现的决策伪代码

```python
def link_product(product):
    candidates = []

    # 1. 多通道候选提取
    candidates += extract_from_structured_fields(product)
    candidates += extract_from_title(product.title)
    candidates += extract_from_ocr(product.images)

    # 2. 角色识别 + ownership
    enriched = []
    for c in candidates:
        c.role, c.role_score = role_model.predict(
            token=c.raw,
            source=product.source,
            field=c.field,
            brand=product.brand,
            context=c.context,
        )

        c.relation = relation_verifier(c, product)

        if c.role != 'BRAND_REFERENCE':
            continue
        if c.relation != 'SELF':
            continue

        c.surface_norm = normalize_surface(c.raw)
        c.ref_candidates = reference_dict.resolve(
            brand=product.brand,
            value=c.surface_norm,
        )
        enriched.append(c)

    # 3. 没有可靠 Reference
    if not enriched:
        return ABSTAIN('NO_REFERENCE')

    # 4. 只允许唯一 Reference Entity
    resolved_ids = {
        rid
        for c in enriched
        for rid in c.ref_candidates
    }

    if len(resolved_ids) != 1:
        return REVIEW('REFERENCE_CONFLICT_OR_AMBIGUOUS')

    ref_entity_id = next(iter(resolved_ids))

    # 5. 强证据校验
    evidence = collect_evidence(enriched, ref_entity_id)

    if has_conflicting_hard_evidence(evidence):
        return REVIEW('HARD_CONFLICT')

    if trusted_structured_exact(evidence):
        return ACCEPT(ref_entity_id, evidence)

    if independent_title_and_ocr_exact(evidence):
        return ACCEPT(ref_entity_id, evidence)

    # 6. 默认拒识
    return ABSTAIN('EVIDENCE_NOT_STRONG_ENOUGH')
```

这段逻辑故意让 `ACCEPT` 很难，符合当前 precision-first 目标。

---

## 14. 图片应该怎么用

Spec 说明有图片。

图片最适合承担两种职责：

### 14.1 OCR 提取 Reference

从这些区域提取 identifier：

```text
表背刻字
保卡
吊牌
标签
盒证
证书
```

OCR token 继续走完全相同的：

```text
Role -> Relation -> Normalize -> Resolve -> Gate
```

不能把 OCR 文本直接 exact match。

### 14.2 冲突否决 / 辅助证据

例如：

```text
structured_ref = 126610LN
title_ref      = 126610LN
OCR_ref        = 126610LV
```

正确行为不是“2:1 多数投票”，而是：

```text
REVIEW / ABSTAIN
```

因为这里可能是真实字段错、OCR 错，也可能商品图与文本不一致。

视觉 embedding 可以用于：

- 人工复核排序；
- 候选召回；
- 发现异常；

但**绝不能越过 Reference mismatch 直接自动合并**。

---

## 15. 如何构造训练数据

用户允许人工标几百对黄金数据，这个预算应该优先用在最危险的边界，而不是随机样本。

### 15.1 Weak Supervision

可以自动生成大量弱标签：

#### Reference 正例候选

```text
可靠来源的显式 reference 字段
标题 token 与显式 reference exact 一致
同品牌 catalog 中已知 Reference
```

#### 平台 SKU / 内部 ID 负例

```text
sku 字段
product_id 字段
goods_id 字段
URL 中的平台 ID
抓取内部 ID
```

#### Compatibility Reference

```text
“适用 REF”
“兼容 REF”
“for REF”
配件品类 + 主表 Reference
```

弱标签不要直接进入最终评测集。

### 15.2 几百条人工标签应该重点覆盖 Hard Negatives

建议黄金集优先标：

```text
同系列只差 1 个字符的 Reference
126610LN vs 126610LV 这类近邻
平台 SKU 长得很像 Reference
Serial Number 长得很像 Reference
年份 / 尺寸 / 材质编码
配件标题里的兼容 Reference
OCR O/0, I/1, S/5, B/8 混淆
同一个 token 在不同字段中角色不同
品牌冲突
一个商品出现多个 Reference
新品牌 / 新格式
```

模型训练可以有几万、几十万弱标签，但真正决定 threshold 的 gold set 必须人工确认。

---

## 16. 评估指标必须改成 Precision-first

不要把：

```text
overall accuracy
macro F1
```

作为上线主指标。

至少要分层看：

### 16.1 Candidate Extraction

```text
Reference Candidate Recall
```

这一层可以偏 recall，因为漏掉只会少匹配。

### 16.2 Role Gate

核心指标：

```text
Precision(role = BRAND_REFERENCE)
False Acceptance Rate
```

特别统计：

```text
PLATFORM_SKU -> BRAND_REFERENCE
SERIAL -> BRAND_REFERENCE
COMPATIBILITY_REF -> SELF_REFERENCE
```

这些才是最危险错误。

### 16.3 Final Entity Assignment

核心指标：

```text
Auto-Accept Precision
Auto-Accept Coverage
Conflict Rate
Manual Review Rate
```

### 16.4 Cross-source Match

最终最重要：

```text
Pair Precision among ACCEPTED pairs
```

Recall/coverage 可以作为第二目标逐步提升。

---

## 17. 为什么“几百条黄金标签 + 模型 99.9%”仍然不能等价于“绝不误匹配”

如果业务要求极端 precision，不能只看一个小验证集上 0 error。

例如只有约 300 个自动接受样本且 0 个错误，从统计上并不能证明真实 precision 已经达到 99.9%。如果希望在“零观测错误”的情况下，让 95% 单侧置信下界达到约 99.9%，需要的无错误样本量接近 3000，而不是几百。

因此当前项目必须采用：

```text
模型
+ Reference Dictionary exact resolution
+ brand consistency
+ role gate
+ ownership gate
+ conflict veto
+ abstain
+ 可回溯 evidence
```

共同构成安全边界。

这也是为什么原项目的 98%～99% accuracy 虽然漂亮，但不能直接迁移成自动匹配器。

---

## 18. 数据切分策略

推荐至少维护下面几套 evaluation slice：

```text
random_holdout
source_holdout
brand_holdout
reference_holdout
time_holdout
ocr_noise_set
compatibility_set
one_char_neighbor_set
multi_reference_conflict_set
```

特别要防止 source leakage。

例如如果 Mouser SKU 有固定数字前缀，原项目随机切分时 train/val 都存在同样规则，模型自然很好。

而当前线上新增来源/新商家时，格式可能完全变化。

所以：

> **新来源、新品牌、新格式必须默认进入 ABSTAIN，直到完成抽样验证。**

---

## 19. 百万～千万级如何扩展

这个方案不需要做 1000 万商品之间的 pairwise matching。

### 19.1 CPU 级 identifier 模型就够

Role model 很小：

```text
字符 embedding
+ 4 个 Conv1D
+ 小 MLP
```

可批量 ONNX Runtime CPU 推理。

每条商品通常只有少量候选 token，因此实际推理量远小于标题 token 数量。

### 19.2 Reference Dictionary 用 Hash / B-Tree 即可

核心查询是：

```text
(brand_id, normalized_key) -> ref_entity_id
```

可以用：

- PostgreSQL unique index；
- RocksDB；
- Redis；
- worker 内存 hash table；

不需要为了 exact Reference lookup 引入向量数据库。

### 19.3 离线回填

历史 100 万～1000 万记录可以：

```text
分区读取
-> Candidate Extraction
-> 批量 Role Inference
-> 批量 Dictionary Resolve
-> 批量 UPSERT Assignment
```

Spark、Ray、Polars streaming、普通多进程 worker 都能实现；核心逻辑是 stateless 的。

### 19.4 增量更新

推荐事件模型：

```text
product.upsert
    │
    ▼
reference-linker worker
    │
    ▼
product_reference_assignment UPSERT
```

只在以下变化时重新计算：

```text
title_hash changed
reference_field changed
image_hash changed
brand changed
rule_version upgraded
model_version upgraded
normalizer_version upgraded
```

保证幂等即可。

---

## 20. 新 Reference 如何进入字典

这是容易被忽略但非常重要的问题。

如果一个 title 里出现“看起来很像”的新编号，不应该马上创建 Reference Entity。

建议新实体建立只允许以下路径：

### 强来源创建

```text
可信 catalog / 权威结构化字段
+ brand 明确
+ role 明确
+ 格式校验通过
=> 可创建 PENDING / VERIFIED Reference Entity
```

### 多来源独立确认

```text
至少两个独立来源
出现完全一致 canonical candidate
+ 无冲突
+ 人工/规则确认
=> 创建 Reference Entity
```

### 单一 title / OCR

```text
不自动建全局 Reference
=> PENDING_CANDIDATE
```

这样可以防止脏数据把错误编号永久污染 Reference Dictionary。

---

## 21. 审计与可逆性

当前业务不能容忍误匹配，所以不要做不可逆的 destructive merge。

每个自动决定必须保留完整证据：

```json
{
  "product_id": 123,
  "decision": "ACCEPT",
  "ref_entity_id": 987,
  "candidates": [
    {
      "raw": "126610LN",
      "channel": "structured_field",
      "field": "reference",
      "role": "BRAND_REFERENCE",
      "role_score": 0.9998,
      "relation": "SELF",
      "normalized_key": "126610LN",
      "dictionary_hit": true
    }
  ],
  "conflicts": [],
  "model_version": "role-v3",
  "rule_version": "gate-v5",
  "normalizer_version": "norm-v4"
}
```

如果后续发现一条规范化规则有问题，可以按 version 精确回滚/重算。

---

## 22. 一个现实可落地的迭代顺序

### P0：先不上 ML，也能获得很高价值

先实现：

```text
Source Adapter
显式字段映射
Reference Candidate Extractor
平台 SKU / ID 黑名单规则
Brand-aware Normalizer
Reference Dictionary
Strict Decision Gate
ABSTAIN
审计表
```

这一版很可能已经可以覆盖大量结构化 Reference 数据。

### P1：加入 Identifier Role Classifier

重点解决：

```text
标题中多个编号
字段脏标
新 SKU 格式
难以写完的正则
```

先做原项目同风格 Char-LSTM baseline，再上 Char-CNN + metadata，对比：

```text
Reference precision
false acceptance
latency
```

### P2：加入 Relation / Ownership

重点解决：

```text
适用型号
配件型号
相关型号
标题中的多个合法 Reference
```

### P3：加入 OCR

只把 OCR 作为独立证据来源和冲突信号。

### P4：主动学习

人工优先审最有价值的样本：

```text
高 role_score 但被规则 veto
多个 Reference 冲突
新格式 OOD
新品牌
模型与规则分歧
```

人工结果回流下一版 Role / Relation 模型。

---

## 23. 和已经分析过的 DeepBlocker 如何组合

前一篇 `DeepBlocker.md` 的核心结论是：Embedding 更适合负责候选召回，最终匹配必须由 Reference 强证据收口。

本项目正好补上一个 DeepBlocker 之前容易遗漏的安全层：

```text
DeepBlocker / fuzzy retrieval
        = “可能是哪一个 Reference？”

Identifier Role Gate
        = “这个字符串有没有资格被当成 Reference？”

Ownership Gate
        = “它是不是当前商品自己的 Reference？”

Canonical Resolver + Strict Gate
        = “是否允许自动链接到唯一 ref_entity_id？”
```

因此一个完整架构可以是：

```text
明确 Reference
    -> exact dictionary lookup
    -> strict gate

Reference 缺失 / OCR 噪声
    -> DeepBlocker / fuzzy candidate retrieval
    -> 得到少量 Reference candidates
    -> Role / Ownership / brand / exact evidence 复核
    -> strict gate
```

最关键的原则仍然是：

> **召回层可以模糊，决策层必须严格。**

---

## 24. 原项目哪些东西值得直接保留，哪些必须替换

### 可以保留

1. **字符级建模**：Identifier 不需要复杂语言模型也能学到强格式规律。
2. **Embedding + sequence encoder**：适合字母数字混合编号。
3. **按类别分析错误**：不能只看总体 accuracy。
4. **逐字符分析**：非常适合排查模型到底依赖了哪个 prefix/suffix。
5. **把模型、字符字典独立序列化**：生产部署要保持 preprocessing 和 model version 一致。

### 必须替换

1. `MPN = catch-all` 的三分类定义。
2. 强制 softmax 三选一，不支持 UNKNOWN。
3. 只输入 token、不输入 source/field/context/brand。
4. 只做随机 train/val。
5. 用 overall accuracy 判断能不能上线。
6. 让模型概率直接决定实体匹配。
7. 不区分合法 Reference 和“当前商品自身 Reference”。
8. 不做 canonical dictionary / conflict veto / audit。

---

## 25. 推荐最终方案

对于当前二奢/腕表三源数据，建议采用下面的主链路：

```text
1. Brand Canonicalization
2. Identifier Candidate Extraction
3. Identifier Role Gate
4. Ownership / Relation Gate
5. Conservative Reference Normalization
6. Canonical Reference Dictionary Resolve
7. Conflict Detection
8. Strict Auto-Accept Gate
9. Product -> Reference Entity Linking
10. Cross-source join by ref_entity_id
11. ABSTAIN / Manual Review / Feedback
```

其中模型真正负责的是 3、4 两个“辅助判断层”，而不是最终 match。

最终自动合并规则应该简单到可以被审计：

```text
只有两条商品都已经 ACCEPT 到同一个唯一 Reference Entity，
才算跨来源同一商品。
```

### 最终推荐原则

> **先判断“编号角色”，再判断“编号归属”，最后才做 Reference 链接。**
>
> **平台 SKU、商家 SKU、序列号、兼容型号宁可全部拒识，也不能让它们进入 Reference Entity。**
>
> **模型负责发现和否决风险，Canonical Reference + 冲突规则负责最终放行。**

这比直接训练一个“商品 A / 商品 B 是否相同”的通用 matcher 更符合当前 Spec，而且在 100 万～1000 万级数据上更便宜、更稳定、更容易解释和回滚。

---

## 参考资料

- parts-distributor-sku-classifier：<https://github.com/pcbng/parts-distributor-sku-classifier>
- README：<https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/README.md>
- Part 1：<https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/parts-distributor-sku-classifier-part-1.ipynb>
- Part 2：<https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/parts-distributor-sku-classifier-part-2-explore.ipynb>
- 当前需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
