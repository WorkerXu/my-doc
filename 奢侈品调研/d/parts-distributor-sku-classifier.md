# parts-distributor-sku-classifier：把“编号角色识别”前置到腕表 reference 匹配之前

> 分析对象：[`pcbng/parts-distributor-sku-classifier`](https://github.com/pcbng/parts-distributor-sku-classifier)  
> 核心 Notebook：[`part 1`](https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/parts-distributor-sku-classifier-part-1.ipynb)、[`part 2`](https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/parts-distributor-sku-classifier-part-2-explore.ipynb)  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”定义为**同一 reference number / 型号**；字段稀疏；有图片；**precision 极端优先，允许漏匹配**。

---

## 1. 结论先行

这个项目真正值得当前需求借鉴的，不是“LSTM 能把 SKU 分到三个类别”这个模型本身，而是它揭示了一个在二奢/腕表数据里非常容易造成灾难性 false positive 的前置问题：

> **一个看起来像型号的字母数字串，到底是什么编号？**

原项目明确区分：

- manufacturer part number（制造商型号/零件号）
- Mouser SKU（渠道商自有货号）
- Digi-Key SKU（渠道商自有货号）

这和三源二奢数据高度同构。雷小安、腕表之家、奢当家里可能同时存在：

- 品牌 reference / 型号
- 平台商品 ID
- 店铺 SKU
- 库存货号
- 寄售编号
- 序列号 / serial number
- 保卡号
- 表壳号
- 配件适配型号
- 标题里提到的其他型号

如果不先判定“编号角色”，后面再强的字符串相似度、embedding、LLM 或图片模型，都可能把平台自有 SKU 当成 reference，从而直接违反“绝对不能误匹配”的约束。

因此建议把该项目的思想落成一个独立服务：

```text
Identifier Role Classifier（编号角色分类器）
```

它位于 reference 抽取之后、canonicalization 和实体匹配之前：

```text
原始商品
  -> 编号候选抽取
  -> 编号角色识别
  -> 仅保留高置信 BRAND_REFERENCE
  -> 品牌内 reference 规范化
  -> exact match
  -> 冲突检查
  -> AUTO_MATCH / ABSTAIN / REJECT
```

**最终 matcher 不应使用该模型的“相似度”直接判同；模型只负责决定某段字符串有没有资格进入 reference 硬匹配。**

---

## 2. 原项目解决的是什么问题

README 用一个非常典型的例子说明了“同一个商品可以有多个不同编号”：

```text
Texas Instruments manufacturer part number:
SN74LVC541APWR

Digi-Key SKU:
296-8521-1-ND

Mouser SKU:
595-SN74LVC541APWR
```

项目目标是给一个字符串，分为 3 类：

```text
0 = MPN
1 = Mouser SKU
2 = Digi-Key SKU
```

这件事表面上是字符串分类，实质上是：

> 从字符序列的前缀、后缀、分隔符、长度和局部模式中学习“编号命名制度”。

对腕表来说，可以把问题重写为：

```text
126610LN          -> BRAND_REFERENCE
LX-202608-88371   -> SOURCE_SKU
8R92XXXX          -> SERIAL_NUMBER
116610LN/97200    -> AMBIGUOUS_OR_COMPOSITE
适配 126610LN      -> ACCESSORY_COMPAT_REFERENCE
```

而当前 Spec 对最终身份的定义又恰好只接受 reference，因此这个角色分类器非常适合作为**高 precision 防误匹配门禁**。

---

## 3. 原项目的数据与训练流程

### 3.1 训练数据

Notebook 读取：

```python
df_raw = pd.read_csv('data/mpn_mouser_digikey.csv')
class_names = ['MPN', 'Mouser SKU', 'Digi-Key SKU']
```

数据集中三类不均衡，作者取最小类别样本数作为上限：

```python
limit_rows_per_class = int(df_raw.groupby('class').count().min())
```

实际最小类为 `4711`，然后每类截取相同数量并打乱：

```python
df = pd.concat(
    list(df_raw[df_raw['class'] == c][:limit_rows_per_class]
         for c in df_raw['class'].unique())
)
df = df.sample(frac=1, random_state=20181203)
```

再随机按约 80%/20% 分为 train / validation：

```python
np.random.seed(20181203)
df['dataset'] = np.random.choice(
    ['train', 'val'],
    size=len(df),
    replace=True,
    p=[0.80, 0.20]
)
```

训练日志显示：

```text
Train on 11344 samples
validate on 2789 samples
```

### 3.2 为什么这套数据处理不能直接照搬

当前腕表任务若采用随机行切分，很容易得到过于乐观的分数，因为：

- 同一 reference 的近似写法可能同时落到 train 和 validation；
- 同一来源的 SKU 命名模板会同时出现在两侧；
- 相邻型号可能共享大段字符；
- 新增来源、品牌和时间批次才是真正生产时的难点。

生产评测至少要额外做：

```text
1. reference-family holdout
2. brand holdout
3. source holdout
4. time-based holdout
5. hard-negative holdout
```

例如把：

```text
126610LN
126610LV
126611LN
116610LN
```

作为一个 reference family，尽量不要随机拆散后让模型“背答案”。

---

## 4. 字符级输入编码

原项目没有做 word/subword tokenization，而是直接对每个字符建字典：

```python
unique_chars = set()
for s in df['partnum'].values:
    unique_chars |= set(c for c in s)

partnum_dict = {c: i+1 for i, c in enumerate(unique_chars)}
```

然后把字符串转成整数序列：

```python
df['x'] = list(
    df['partnum'].map(
        lambda s: list(partnum_dict[c] for c in s)
    )
)
```

再做 post padding：

```python
maxlen = max(len(pn) for pn in df['partnum'].values)

df['x'] = list(
    sequence.pad_sequences(
        df['x'],
        maxlen=maxlen+1,
        padding='post'
    )
)
```

`0` 同时承担了序列结束/填充的角色。

这个思路非常适合 reference/SKU：编号中的语义通常就藏在字符形态里，例如：

- 是否纯数字
- 字母数字交替
- 连字符位置
- 固定前缀
- 固定后缀
- 长度
- `/`、`.`、`-` 的使用习惯
- 某来源是否有固定 wrapper

对于这种任务，字符级模型通常比把整个编号当“单词”更自然，也不会遇到长尾 reference 的词表爆炸。

---

## 5. 原模型架构

核心网络非常小：

```python
model = Sequential()
model.add(Embedding(len(partnum_dict)+1, 32))
model.add(LSTM(32, dropout=0.2, recurrent_dropout=0.2))
model.add(Dense(3, activation='softmax'))
```

训练配置：

```python
model.compile(
    loss='categorical_crossentropy',
    optimizer='adam',
    metrics=['accuracy']
)

batch_size = 32

epochs = 7
```

即：

```text
character id sequence
  -> Embedding(32)
  -> LSTM(32)
  -> Dense(3)
  -> softmax
```

仓库中保存的权重文件只有约 55 KB，说明整个模型很小。这个特征对于 100 万–1000 万级离线重跑和持续增量很有价值：编号角色识别并不需要大模型，可以用非常便宜的小模型完成。

---

## 6. 结果与最重要的失败模式

Part 1 训练后给出的总体验证 accuracy 约为：

```text
98.49%
```

按类别：

```text
MPN          96.19%
Mouser SKU   99.26%
Digi-Key SKU 100.00%
```

真正重要的不是 98%+ 的 accuracy，而是 Part 2 的错误分析。

作者画出的 confusion 关系显示，真实 MPN 会被错误识别为：

```text
MPN -> Mouser SKU    约 3.050%
MPN -> Digi-Key SKU  约 0.763%
```

作者自己指出：

> MPN 是一个“noisy catch-all”类别，模型对已知模式类别学得很好，但对“不属于已知模式的东西”缺乏可靠的 “I don't know” 能力。

这正是当前 Spec 的核心风险。

如果把标签换成：

```text
BRAND_REFERENCE
SOURCE_SKU
SERIAL_NUMBER
UNKNOWN
```

普通 softmax 很容易把真正未知的新编号格式**自信地塞进某个已知类别**。在 recall-first 任务里也许还能接受，但在“绝对不能误匹配”的任务里不能接受。

所以生产系统必须支持：

```text
ABSTAIN / UNKNOWN
```

而不是强迫每个字符串都属于某个角色。

---

## 7. Part 2 暴露出的实现问题

### 7.1 稀有字符会诱发错误高置信

作者逐字符喂入一个误判样本时观察到，模型对 `.`、`(` 等较少见字符会突然增强对某个错误类别的置信。

这说明模型学习到的不只是稳定“命名规则”，也会学习训练数据中的偶然相关性。

对腕表数据尤其危险，因为标题/OCR 中会出现：

```text
126610LN（全套）
126610-LN
126610/LN
Ref.126610LN
型号:126610LN
```

如果没有足够覆盖，标点可能变成错误强特征。

### 7.2 没有显式 start-of-sequence token

作者在结论里提出：模型是否真正理解“字符串开头”？

因为很多渠道 SKU 的决定性规则都在开头：

```text
595-...
296-...
```

腕表平台 SKU 也通常有固定前缀，所以生产版应该显式引入：

```text
<BOS> ... <EOS>
```

### 7.3 padding 可能形成偏差

原项目把所有字符串 pad 到全局最大长度，并用 0 表示结尾/填充。作者担心大量连续 padding 会产生偏差。

生产版应使用：

- 独立 PAD / BOS / EOS / UNK token；
- mask；
- 动态 batch padding；
- 或直接采用 Char-CNN / Transformer encoder。

### 7.4 训练/加载 loss 不一致

Part 1 训练使用：

```text
categorical_crossentropy
```

Part 2 重新加载模型后却 compile 为：

```text
binary_crossentropy
```

而且两个 Notebook 报告的 overall/per-class accuracy 也存在差异。

这对教学 Notebook 问题不大，但生产系统必须把以下内容一起版本化：

```text
model weights
model architecture
preprocessing vocabulary
normalization version
label schema
loss/config
calibration parameters
accept thresholds
```

否则同一权重在不同推理代码下会产生难以审计的结果。

---

## 8. 不能直接照搬的地方

### 8.1 它是“整串三分类”，而目标数据可能有嵌套编号

原项目最有意思的例子恰恰是：

```text
Mouser SKU = 595-SN74LVC541APWR
```

这个整体是渠道 SKU，但其中：

```text
SN74LVC541APWR
```

又是真正的 manufacturer part number。

二奢里也完全可能出现：

```text
SDJ-126610LN-202608
```

整体是奢当家库存号，但内部 `126610LN` 仍可能是真实品牌 reference。

因此生产版不能简单做：

```text
整个字符串 = SOURCE_SKU
=> 丢掉整串
```

而应当做**span-level / nested identifier parsing**：

```text
raw: SDJ-126610LN-202608

span 1: SDJ                 -> SOURCE_PREFIX
span 2: 126610LN            -> POSSIBLE_BRAND_REFERENCE
span 3: 202608              -> BATCH_OR_DATE
whole : SDJ-126610LN-202608 -> SOURCE_SKU
```

只有 `126610LN` 再通过品牌词典/角色模型/上下文校验后，才进入 canonical reference。

这是把原项目迁移到腕表场景时最重要的一处架构升级。

---

## 9. 推荐的目标标签体系

第一版不建议只做 `REFERENCE / NOT_REFERENCE` 两类，因为“不是 reference”的错误类型对于规则和监控很有价值。

建议至少：

```text
BRAND_REFERENCE
SOURCE_SKU
SHOP_SKU
SERIAL_NUMBER
ACCESSORY_COMPAT_REFERENCE
CERTIFICATE_OR_CARD_NUMBER
COMPOSITE_IDENTIFIER
UNKNOWN
```

其中真正允许进入自动匹配的只有：

```text
BRAND_REFERENCE
```

而且必须达到高置信接受门槛。

`ACCESSORY_COMPAT_REFERENCE` 必须作为明确负类，因为：

```text
“适配劳力士 126610LN 的表带”
```

标题中包含正确 reference，但商品本身并不是这只腕表。

---

## 10. 面向当前 Spec 的完整生产架构

推荐把系统拆成 6 层：

```text
                 ┌────────────────────┐
                 │  Source Ingestion  │
                 └─────────┬──────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │ Candidate Identifier    │
              │ Extraction              │
              └──────────┬──────────────┘
                         │
                         ▼
              ┌─────────────────────────┐
              │ Identifier Role         │
              │ Classification          │
              └──────────┬──────────────┘
                         │
                         ▼
              ┌─────────────────────────┐
              │ Reference Resolver &    │
              │ Canonicalizer           │
              └──────────┬──────────────┘
                         │
                         ▼
              ┌─────────────────────────┐
              │ Brand + Canonical Ref   │
              │ Exact Index             │
              └──────────┬──────────────┘
                         │
                         ▼
              ┌─────────────────────────┐
              │ Conflict / Safety Gate  │
              └───────┬─────────┬───────┘
                      │         │
                 AUTO_MATCH   ABSTAIN
```

### 10.1 Source Ingestion

统一三源数据为：

```json
{
  "source": "lei_xiao_an",
  "source_item_id": "...",
  "title": "...",
  "brand_raw": "...",
  "reference_raw": "...",
  "sku_raw": "...",
  "description": "...",
  "image_urls": ["..."]
}
```

原始字段永远保留，任何后续标准化结果都不能覆盖 raw value。

### 10.2 Candidate Identifier Extraction

从以下位置抽候选：

```text
1. 独立 reference/model 字段
2. SKU/货号字段
3. 标题
4. 描述
5. 图片 OCR
```

抽取器只负责尽量召回“像编号的 span”，不要在这里做最终身份判断。

输出：

```json
{
  "raw_token": "126610LN",
  "field": "title",
  "start": 8,
  "end": 16,
  "extractor": "regex_v3",
  "left_context": "劳力士 黑水鬼",
  "right_context": "全套 2022"
}
```

### 10.3 Identifier Role Classification

输入不应只有 token，还应拼上上下文：

```text
token chars
source
field_name
brand_candidate
left/right context
prefix/suffix features
```

模型输出：

```json
{
  "role": "BRAND_REFERENCE",
  "probabilities": {
    "BRAND_REFERENCE": 0.9997,
    "SOURCE_SKU": 0.0001,
    "SERIAL_NUMBER": 0.0001,
    "UNKNOWN": 0.0001
  },
  "model_version": "identifier-role-v4"
}
```

### 10.4 Reference Resolver / Canonicalizer

只处理被角色层放行的候选。

建议规范化顺序：

```text
Unicode NFKC
-> trim
-> uppercase
-> 标准化全角字符
-> 品牌特定 separator 规则
-> 品牌特定 prefix 规则
```

不要全局粗暴做：

```text
remove all '-' '/' '.'
```

因为不同品牌的 reference 规则不同，某些分隔符可能有语义。

建议维护：

```text
canonical_reference_v1(brand, raw_reference)
```

并把 normalization version 存到记录里。

### 10.5 Exact Index

由于业务定义是“同 reference 才同商品”，最终索引可以非常简单：

```text
key = (canonical_brand_id, canonical_reference)
```

例如：

```text
(ROLEX, 126610LN)
```

实体 ID 可以稳定生成：

```text
entity_key = SHA256(brand_id + "\0" + canonical_reference)
```

三个来源只需要查询这个 exact key，不需要做全表 pairwise 比较。

对于 100 万–1000 万级数据，主链路复杂度从：

```text
O(N²) pairwise matching
```

变成近似：

```text
O(N) parse + indexed exact lookup
```

这比先上通用 embedding matcher 更符合当前明确的业务定义。

---

## 11. 最终 Safety Gate：precision-first 的核心

建议把自动匹配规则写成显式决策表，而不是一个神秘总分。

### AUTO_MATCH

必须同时满足：

```text
1. 两侧 canonical_brand_id 相同
2. 两侧 canonical_reference 完全一致
3. 两侧 reference 角色均达到高置信门槛
4. 没有其他高置信 reference 与之冲突
5. 没有 accessory/compatibility 语义
6. 没有 serial/SKU 角色冲突
```

### REJECT

任一情况直接否决：

```text
品牌不同
canonical reference 不同
一侧明确被判为 SOURCE_SKU / SERIAL_NUMBER
标题明确是配件/表带/盒证且 reference 只是兼容对象
```

### ABSTAIN

以下情况不自动合并：

```text
reference 缺失
存在两个候选 reference
模型置信不足
structured reference 与 title/OCR reference 冲突
只凭图片相似
只凭标题语义相似
```

这正好符合 Spec：

> precision 优先到极致，允许漏匹配。

---

## 12. “拒识”不能只看 softmax 最大值

原项目暴露了典型 open-set 问题：模型没见过的东西仍会被 softmax 强行分类。

生产版建议至少使用三层接受条件：

```text
P(BRAND_REFERENCE) >= T_ref
AND
P(BRAND_REFERENCE) - max(P(other)) >= T_margin
AND
rule_conflict == false
```

阈值不要先拍脑袋写死，而是在黄金标注集上按目标 precision 反推。

还可以进一步加：

- temperature scaling / isotonic calibration
- conformal/selective prediction
- out-of-distribution score
- per-source / per-brand threshold

但最重要的是：

```text
低置信不是“判否”，而是 ABSTAIN。
```

---

## 13. 模型建议：保留字符级思想，不必执着 LSTM

原项目的 `Embedding + LSTM` 是很好的 baseline，但生产版可比较：

### A. 规则 + 字符 n-gram Logistic Regression

优点：

- 解释性极高
- 推理便宜
- 适合第一版
- 容易看到哪些前后缀导致判断

### B. Char-CNN

优点：

- 很适合固定前后缀和局部模式
- 比 RNN 更容易批量并行
- 对短 identifier 足够强

### C. 小型 Char Transformer

优点：

- 能结合较长的复合 identifier
- 可做 span-level 分类
- 容易融合 field/source/context token

第一版建议：

```text
规则引擎 + Char-CNN（或轻量 LSTM） + ABSTAIN
```

而不是 LLM。

LLM 可以用于离线分析新格式、生成候选规则、辅助标注，但不应位于 1000 万级主路径的最终自动放行点。

---

## 14. 训练数据如何从现有三源低成本构造

当前 Spec 允许人工标几百对，但角色分类数据可以大量利用已有结构字段弱监督生成。

### 高置信正样本

如果来源有明确：

```text
reference/model 字段
```

可作为 `BRAND_REFERENCE` 弱标签候选。

### 高置信负样本

明确字段：

```text
item_id
sku
stock_no
inventory_id
serial_no
```

可以产生：

```text
SOURCE_SKU / SHOP_SKU / SERIAL_NUMBER
```

### 人工标注重点

几百条人工不要随机抽，应集中在 hard cases：

```text
1. SKU 内嵌 reference
2. 同系列相邻 reference
3. reference vs serial 外形近似
4. 标题出现两个型号
5. 配件适配型号
6. OCR 读错 0/O、1/I、5/S
7. 新品牌、新来源编号格式
```

这样比随机标几百条更能降低 false positive。

---

## 15. 图片/OCR 应怎样接入

图片在当前系统里最好作为**独立 reference observation**，而不是直接以视觉相似度决定同款。

建议：

```text
图片
 -> OCR
 -> identifier candidate extraction
 -> role classifier
 -> canonicalizer
```

图片中常见的风险是：

- 表背既有 reference 又有 serial
- 保卡上有多个编号
- 吊牌有店铺/平台货号
- OCR 把 `0/O`、`1/I` 混淆

因此 OCR 结果同样必须经过“编号角色识别”。

如果文本 reference 与 OCR reference 一致，可以提高证据等级；如果冲突，默认：

```text
ABSTAIN
```

而不是挑一个分数更高的继续自动匹配。

---

## 16. 数据表建议

### `identifier_observation`

```sql
record_id
source
source_item_id
field_name
raw_text
raw_token
span_start
span_end
extraction_method
role
role_score
role_margin
brand_candidate
model_version
created_at
```

### `reference_observation`

```sql
record_id
raw_reference
canonical_reference
canonical_brand_id
normalizer_version
confidence_level
source_field
is_conflicted
```

### `reference_entity`

```sql
entity_id
canonical_brand_id
canonical_reference
first_seen_at
last_seen_at
```

唯一索引：

```sql
UNIQUE(canonical_brand_id, canonical_reference)
```

### `entity_membership`

```sql
entity_id
source
source_item_id
decision
reason_code
decision_version
created_at
```

这样每一次自动合并都能解释：

```text
为什么合并？
使用了哪个 reference？
来自哪个字段？
哪个模型版本判断它是 reference？
有没有冲突？
```

对 precision-first 系统，这种可审计性比一个黑盒相似度分数重要。

---

## 17. 推荐的 reason codes

不要只存 `matched=true/false`。

建议：

```text
MATCH_EXACT_VERIFIED_REFERENCE
MATCH_EXACT_EXTRACTED_REFERENCE
ABSTAIN_REFERENCE_MISSING
ABSTAIN_REFERENCE_AMBIGUOUS
ABSTAIN_ROLE_LOW_CONFIDENCE
ABSTAIN_REFERENCE_CONFLICT
ABSTAIN_OCR_TEXT_CONFLICT
REJECT_BRAND_CONFLICT
REJECT_REFERENCE_CONFLICT
REJECT_SOURCE_SKU
REJECT_ACCESSORY_COMPAT_REFERENCE
```

后续人工复核、指标统计和模型迭代都会非常方便。

---

## 18. 评测指标必须从 accuracy 改掉

原项目主要展示 overall accuracy，但当前业务不能把它作为核心指标。

真正要看：

### 编号角色层

```text
Precision(BRAND_REFERENCE)
False Positive Rate(BRAND_REFERENCE)
Coverage / Abstain Rate
```

### 最终匹配层

```text
Auto-match Precision
False Merge Count
Auto-match Coverage
Manual Review Rate
```

并额外维护 hard-negative suite：

```text
同品牌相邻 reference
同系列不同尺寸
钢款/金款不同 suffix
平台 SKU 与 reference 形似
serial 与 reference 形似
配件兼容型号
OCR 单字符错误
```

对这个 Spec，宁可 coverage 只有一部分，也不能为了 recall 把 false merge 放进自动链路。

---

## 19. 增量更新架构

数据会持续更新，所以不要每日全量 N² 重跑。

推荐事件化：

```text
商品新增/字段变化
  -> identifier extraction
  -> role classification
  -> reference resolve
  -> exact index lookup
  -> membership upsert
```

只有这些字段变化时需要重新解析：

```text
title
reference/model
sku
brand
description
images/OCR
```

如果 canonical reference 没变，实体关系无需重算。

模型/规则升级时可按 `decision_version` 做离线重放，而不是破坏旧结果的可追踪性。

---

## 20. 第一版可以直接落地的实现

不需要先做复杂的全栈 ER 平台。

### Step 1：品牌规范化

建立：

```text
raw brand -> canonical_brand_id
```

优先规则/词典，无法确定则 ABSTAIN。

### Step 2：编号候选抽取

优先级：

```text
structured reference > title span > description span > OCR
```

所有候选均保留来源和上下文。

### Step 3：编号角色规则层

先积累每个平台明显的自有 SKU 规则：

```text
known source prefix
known field semantics
known serial patterns
known accessory phrases
```

规则能明确否决的，直接不进入 reference。

### Step 4：字符级角色模型

把无法被规则明确判定的 token 输入小模型。

输入：

```text
<BOS>
<SOURCE=...>
<FIELD=...>
<BRAND=...>
字符序列
<EOS>
```

输出含 `UNKNOWN`。

### Step 5：canonical reference

只把高置信 `BRAND_REFERENCE` 规范化。

### Step 6：exact match

唯一自动合并条件：

```python
same_brand and
same_canonical_reference and
reference_evidence_high_confidence and
not conflict
```

### Step 7：其余全部 ABSTAIN

把人工复核数据回流为新的 hard examples。

---

## 21. 可直接实现的伪代码

```python
def resolve_record(record):
    brand = normalize_brand(record)
    if not brand.confident:
        return Abstain("ABSTAIN_BRAND_AMBIGUOUS")

    candidates = extract_identifier_candidates(record)

    ref_candidates = []

    for cand in candidates:
        # 先跑高精度规则
        rule_result = identifier_rules.classify(cand, record)

        if rule_result.role in {
            "SOURCE_SKU",
            "SHOP_SKU",
            "SERIAL_NUMBER",
            "ACCESSORY_COMPAT_REFERENCE",
        } and rule_result.confident:
            continue

        # 再跑字符级/上下文模型
        pred = role_model.predict(cand, record, brand)

        if not selective_accept_reference(pred):
            continue

        ref = canonicalize_reference(
            brand_id=brand.id,
            raw_reference=cand.raw_token,
        )

        if ref.valid:
            ref_candidates.append(ref)

    ref_candidates = dedupe(ref_candidates)

    if len(ref_candidates) == 0:
        return Abstain("ABSTAIN_REFERENCE_MISSING")

    if len(ref_candidates) > 1:
        return Abstain("ABSTAIN_REFERENCE_AMBIGUOUS")

    ref = ref_candidates[0]

    if has_reference_conflict(record, ref):
        return Abstain("ABSTAIN_REFERENCE_CONFLICT")

    entity = exact_index.get((brand.id, ref.canonical))

    if entity is None:
        entity = exact_index.create((brand.id, ref.canonical))

    return AutoMatch(
        entity_id=entity.id,
        reason="MATCH_EXACT_VERIFIED_REFERENCE",
    )
```

这段逻辑刻意没有：

```text
text similarity > 0.9 => match
image similarity > 0.9 => match
LLM says same => match
```

因为这些都不满足当前业务身份定义。

---

## 22. 必做的 hard-negative 测试

上线前建议构造最少一套专门打 false positive 的测试集：

```text
ROLEX 126610LN vs 126610LV
ROLEX 116610LN vs 126610LN
同一 reference 的空格/大小写/全角差异
平台 SKU 含 reference 子串
serial 恰好与 reference 长度相同
“适配 126610LN 表带” vs 真正 126610LN 腕表
标题含“同款 126610LN”但商品是另一个 reference
OCR: 126610LN -> 12661OLN
OCR: 116500LN -> 1165001N
```

系统必须证明：

```text
宁可 ABSTAIN，也不自动错合。
```

---

## 23. 与已调研方案的互补关系

该项目与 DeepBlocker / AMELI / selective entity matching 并不冲突，反而是更靠前的“语义门禁层”。

建议整体关系：

```text
编号候选抽取
  -> 编号角色识别（本项目启发）
  -> reference canonicalization
  -> exact index 高置信自动匹配
  -> 对 reference 缺失样本：
       DeepBlocker / multimodal retrieval 只做候选召回
  -> selective matcher / 人工复核
  -> graph/transitive consistency 做结果审计
```

也就是说：

- 本方案解决“这个字符串是不是 reference”；
- DeepBlocker 解决“reference 不好抽时先找谁像谁”；
- AMELI/多模态解决“文本稀疏时怎样增加证据”；
- selective matching 解决“证据不足时拒识”；
- TransClean/图一致性解决“如何发现已产生的可疑 false positive”。

对于当前 Spec，本方案应当位于非常靠前的位置，因为**错误理解编号角色会污染后面所有层**。

---

## 24. 最终建议

`parts-distributor-sku-classifier` 是一个老而小的教学项目，但它对当前需求有一个非常直接、而且容易被忽略的工程价值：

> **reference matching 之前，先确认“reference 真的是 reference”。**

推荐直接落地为：

```text
Reference Identity Firewall
```

即一层专门阻止错误编号进入匹配主链路的“身份防火墙”。

第一版的最优路径不是复刻原仓库，而是：

```text
1. 平台/字段规则
2. span 级 identifier extraction
3. 字符级编号角色模型
4. UNKNOWN / ABSTAIN
5. 品牌特定 canonicalizer
6. (brand, canonical_reference) exact match
7. 冲突硬否决
8. 人工 hard-case 回流
```

这样可以把项目里“MPN 被误认为渠道 SKU”的 open-set false-positive 问题，转化为当前系统里一个明确可控的安全边界。

在“同一商品 = 同一 reference”这一业务定义下，这层的收益很高：它不追求更聪明地猜，而是**减少不该进入 reference matcher 的字符串**。这比单纯继续堆更强的相似度模型，更符合“绝对不能误匹配”的核心约束。
