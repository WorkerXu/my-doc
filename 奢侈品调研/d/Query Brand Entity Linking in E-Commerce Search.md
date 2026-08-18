# Query Brand Entity Linking in E-Commerce Search：把“精确词典优先 + XMC 候选 + 拒识”迁移为腕表 reference linking

> 分析对象：[Query Brand Entity Linking in E-Commerce Search](https://arxiv.org/abs/2502.01555)（Dong Liu, Sreyashi Nag，Amazon）  
> 参考实现：[Amazon PECOS](https://github.com/amzn/pecos)  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”定义为**同一 reference number / 型号**；字段稀疏；有图片；**precision 极端优先，允许漏匹配**。

---

## 1. 结论先行

这篇论文非常适合当前需求，但**不能原样照搬**。

它解决的是电商搜索中的“品牌 mention -> 全局唯一 brand entity”问题，核心架构是：

```text
高精度规则路径：
NER -> Exact Lexical Match -> 上下文过滤 -> 唯一实体 / 拒识

高覆盖模型路径：
原始 query 或 NER mention -> PECOS 极端多分类 -> Top-K entity -> 上下文过滤

融合：
Exact 命中优先；模型负责补 coverage
```

对腕表 reference matching，最值得复制的是 4 个设计原则：

1. **把字符串抽取和实体归一分开**：先找到可能的 reference mention，再链接到 canonical reference entity。
2. **高精度路径拥有最高优先级**：结构化 reference 或可靠抽取后 exact lookup 成功时，不让 embedding/LLM 覆盖它。
3. **模型只负责在巨大输出空间中缩小候选，不直接决定“同一个商品”**。
4. **一对多、证据冲突、候选不唯一时直接 abstain**，而不是“选分最高的那个”。

但论文有一个对当前 Spec 极其重要的警告：它的语义模型虽然显著提高 coverage，却会增加 false alarm。论文对非品牌 query 的测试中：

- NER + Exact Lexical Match false alarm：**1.177%**
- NER + M2E-PECOS：**3.267%**
- Q2E-PECOS（无弱标签）：**4.037%**
- Q2E-PECOS（有弱标签）：**6.550%**

对于普通搜索理解，这可能是可接受的；但对于当前“**绝对不能误匹配**”的商品实体合并，这个错误量级完全不可接受。

因此最终建议不是“用 PECOS 直接判相同 reference”，而是：

> **PECOS / embedding / LLM 只能成为 candidate proposer；真正 AUTO_MATCH 的最终门槛仍然必须是可审计的 canonical reference 硬证据。**

建议直接落地为：

```text
Reference Linking Gateway

商品原始字段 / 标题 / 描述 / OCR
       |
       v
品牌归一 + 编号候选抽取
       |
       v
编号角色分类（reference / SKU / serial / accessory-compatible-ref ...）
       |
       +----------------------+
       |                      |
       v                      v
Exact Resolver          XMC Candidate Proposer
高精度主路径             仅召回候选
       |                      |
       +----------+-----------+
                  v
          Evidence Verifier
        + Conflict Gate
                  |
       +----------+----------+
       |                     |
   AUTO_MATCH             ABSTAIN
```

其中 `AUTO_MATCH` 只输出 `canonical_reference_id`，跨来源商品是否同一实体则变成一个非常简单、安全的等值判断：

```text
same_product(a, b)
    = a.canonical_reference_id IS NOT NULL
      AND a.canonical_reference_id = b.canonical_reference_id
```

---

## 2. 为什么这次选择这篇论文

`d` 目录里已经分析过：TransClean、GraLMatch、Ditto/Deep Entity Matching、DeepBlocker、pyJedAI、AnyMatch、Ameli、PAM、MOON2.0、选择性预测、Conformal Risk Control、SKU role classifier 等，因此本次排除了这些已有结果。

这篇论文在当前未分析条目里有三个特别直接的价值：

### 2.1 它研究的不是普通相似度，而是“mention -> canonical entity”

当前问题本质也不是：

```text
商品 A 和商品 B 看起来像不像？
```

而是：

```text
商品 A 的文本/图片里出现的型号字符串，究竟对应哪个 canonical reference？
商品 B 又对应哪个 canonical reference？
两个 canonical reference 是否完全相同？
```

把 pairwise matching 转成 entity linking 后，系统复杂度和安全性都会更好。

### 2.2 它原生处理巨大、长尾、持续增长的实体空间

论文中的 brand entity 数约 6.1 万；PECOS 本身就是为 millions/billions 级输出空间设计的。当前腕表 reference 数量虽然通常低于整个商品记录数，但随着品牌、历史型号、珠宝、箱包等品类扩展，canonical entity 也会变成长尾大标签空间。

### 2.3 它明确使用“无法唯一确定就不预测”的策略

论文在 lexical matcher 一对多时，会用 product type 辅助消歧；如果过滤后仍对应多个实体，则**不输出 brand entity**，以保证 precision。

这是当前 Spec 最应该复制的行为，而不是复制某个具体模型。

---

## 3. 原论文的问题定义与数据结构

论文目标是：输入一个非常短、噪声大的电商搜索 query，输出其对应的全局唯一 brand entity。

例如：

```text
query: "nike running shoes"
mention: "nike"
brand_entity_id: BRAND_12345
```

挑战包括：

- query 极短，平均约 2.4 个词；
- 同品牌跨语言、跨商店有不同 surface form；
- 缩写、全称、别名并存；
- parent brand / sub-brand 有关系；
- brand entity 空间巨大且长尾；
- 新品牌持续加入。

这与腕表 reference linking 的映射关系几乎一一对应：

| 原论文 | 当前腕表需求 |
|---|---|
| query | 商品标题 + 描述 + OCR + 结构化字段 |
| brand mention | reference mention / 型号候选 |
| brand entity | canonical reference entity |
| brand aliases | reference 的连字符、空格、大小写、地区写法、旧称 |
| product type | 品牌 + 系列 + 商品类型 + 配件/整表角色 |
| NIL | 未发现可信 reference |
| multi-brand ambiguity | 一个字符串对应多个型号 / 多个 reference 同时出现 |
| lexical matching | canonicalized exact reference lookup |
| PECOS semantic matching | 超大 reference 空间中的候选召回 |

最关键的变化是：

> 品牌可以有一定语义容错；reference 是身份主键，必须把“语义相似”降级为候选生成，不允许它直接变成身份判定。

---

## 4. 原论文技术架构

## 4.1 路径 A：NER + Exact Lexical Match

论文第一条路径是典型两阶段 entity linking：

```text
query
  -> MetaTS-NER
  -> brand mention
  -> brand_name -> entity dictionary
  -> product-type filtering
  -> brand entity / abstain
```

形式化为：

```text
m = f_NER(q)
E = g(m)
e = h(E, q, PT_q)
```

其中：

- `f_NER`：抽取 brand mention；
- `g`：把 mention 映射成候选 entity 集合；
- `h`：结合 query 和 product type 做消歧。

论文的 exact matcher 并不是“一个字符串只能对应一个 entity”。同一个 surface form 可能映射到多个 brand entity；作者会使用 product type 过滤。

更重要的是：

```text
过滤后仍有多个候选 -> 不预测
```

这个拒识规则对当前方案的价值远高于“NER 用了哪个模型”。

### 迁移到腕表

```text
商品 title / description / OCR
  -> reference mention extractor
  -> canonicalize
  -> reference alias dictionary lookup
  -> brand/series/product-role filter
  -> unique canonical_reference_id / ABSTAIN
```

---

## 4.2 路径 B：NER + PECOS Semantic Matching

Exact dictionary 的问题是 coverage：

```text
126610 LN
126610-LN
126610LN
劳力士 126610 黑水鬼
126610LN-0002
```

如果词典 alias 没收全，就可能漏掉。

论文因此把 mention-to-entity 改成 eXtreme Multi-Class Classification：

```text
brand mention
  -> PECOS
  -> Top-K brand entities + scores
  -> context filter
```

论文使用 PECOS 的原因是：

- label/entity 数量大；
- label 分布长尾；
- 需要低延迟；
- 不能对全部 label 做线性扫描。

PECOS 的基本思想是将巨大 label 空间组织成层次结构，先从树上快速筛出少量候选簇，再在叶节点内做精排。

官方 PECOS 的 X-Linear 可以理解为：

```text
百万级 label
  -> hierarchical label tree
  -> learned matcher 逐层下钻
  -> 小量 leaf candidates
  -> rank top-k labels
```

而不是：

```text
input embedding x 100万 label embedding 全量比较
```

这对 100 万–1000 万商品的增量系统非常重要，因为真正需要扩展的不是 pair 数，而是 canonical reference 词表和 candidate lookup。

---

## 4.3 路径 C：Query-to-Entity End-to-End PECOS

论文还提出 Q2E-PECOS：直接把原始 query 输入 PECOS，输出 brand entity：

```text
query -> PECOS -> brand_entity
```

并增加一个 `NIL` 类，表示 query 没有品牌意图。

它的优点：

- 不被 NER recall 卡住；
- 能利用隐式上下文；
- 部署链路更短。

但它的缺点对当前任务是致命的：

- 端到端模型更容易“猜”；
- 很难证明输出 reference 确实出现在商品证据里；
- false alarm 明显高于 lexical path。

所以在腕表方案中，不建议做：

```text
商品文本 -> 模型 -> reference -> 自动合并
```

而建议做：

```text
商品文本 -> 模型 -> reference candidate
                   |
                   v
              证据回查验证
                   |
              通过才可使用
```

即模型必须提供一个**可证伪的候选**，后续系统再回到原始 title/OCR/字段里验证。

---

## 4.4 Fusion：高精度 lexical path 永远优先

论文最值得直接照抄的工程思想是 fusion priority：

> Exact lexical matcher 与模型同时运行；两者都有结果时，优先 lexical result。

这实际上是一个规则优先的专家混合：

```text
if lexical_result is unique:
    use lexical_result
else:
    consider model_result
```

论文结果也证明了：semantic matching 虽然 coverage 更高，但 precision 会下降。

在两个商店组上，NER + Exact Lexical Match 的 precision 分别约为：

```text
97.22%
99.15%
```

而仅使用更激进的 Q2E-PECOS 时 precision 明显更低；加入 fusion 后才能恢复到更高水平。

对当前需求，这还不够严格，因此需要再向前走一步：

```text
Exact path = AUTO_MATCH candidate
Model path = REVIEW/verification candidate
```

而不是两条路径都拥有自动合并权。

---

## 5. 原论文训练数据设计：强标签 + 弱标签 + 词典

论文把训练数据拆成三类：

### 5.1 Brand2Entity dictionary

```text
surface_form -> brand_entity_id
```

既用于 exact lookup，也把 surface form 当 pseudo-query 训练模型。

映射到当前任务：

```text
reference_alias -> canonical_reference_id
```

例如：

```text
"126610LN"       -> rolex:126610ln
"126610 LN"      -> rolex:126610ln
"126610-LN"      -> rolex:126610ln
"m126610ln-0001" -> rolex:126610ln   # 只有经过品牌规则确认后才建立 alias
```

### 5.2 Strongly-labeled data

论文使用人工标注 query。

当前已有“可接受人工标注几百对”的条件，可以把它放在最有价值的位置：

- 同品牌同系列相邻 reference；
- 只差 1 个字符的型号；
- 表带/配件标题包含主表 reference；
- serial / SKU 与 reference 混淆；
- OCR 中 `O/0`、`I/1`、`S/5` 混淆；
- 一个商品同时出现多个 reference；
- structured reference 与 title/OCR 冲突。

不要随机标几百对；要标**最可能产生 false positive 的 hard cases**。

### 5.3 Weakly-labeled data

论文从历史 query-product engagement 构造 130 万级弱标签。

当前可直接利用已经存在的结构化 reference 字段生成海量弱监督数据：

```text
如果某平台 structured_reference 非空且通过格式校验：
    label = canonical_reference_id
    input = title + description + OCR
```

这会自动形成“文本/OCR -> reference entity”的大规模训练集。

但必须注意：

> 弱标签可以训练 candidate proposer，但不能反过来作为 AUTO_MATCH 的真值来源。

否则会把原始平台中的脏 reference 错误复制到模型中。

---

## 6. 面向当前 Spec 的推荐总体架构

建议把系统从“跨源 pair matching”改造成“各源独立 reference linking + canonical id join”。

```text
                 +----------------------+
雷小安 ---------->|                      |
腕表之家 -------->|  Ingestion / Normalize |----+
奢当家 ---------->|                      |    |
                 +----------------------+    |
                                             v
                                   +-------------------+
                                   | Brand Canonicalizer|
                                   +-------------------+
                                             |
                                             v
                                   +-------------------+
                                   | Ref Candidate     |
                                   | Extraction        |
                                   | field/text/OCR    |
                                   +-------------------+
                                             |
                                             v
                                   +-------------------+
                                   | Identifier Role   |
                                   | Classifier        |
                                   +-------------------+
                                             |
                                             v
          +--------------------+   +-------------------+   +------------------+
          | Reference Alias DB |-->| Exact Resolver    |<--| Brand/Series KB  |
          +--------------------+   +-------------------+   +------------------+
                                             |
                             unresolved/ambiguous only |
                                             v
                                   +-------------------+
                                   | PECOS Top-K       |
                                   | Candidate Proposer|
                                   +-------------------+
                                             |
                                             v
                                   +-------------------+
                                   | Evidence Verifier |
                                   +-------------------+
                                             |
                                             v
                                   +-------------------+
                                   | Conflict / Abstain|
                                   | Gate              |
                                   +-------------------+
                                      |             |
                                      v             v
                                canonical_ref     REVIEW
                                      |
                                      v
                              cross-source GROUP BY
                              canonical_reference_id
```

核心原则：

```text
先 link，再 match。
```

只要每条商品记录被安全地链接到同一个 canonical reference，跨源匹配就不再需要复杂模型：

```sql
SELECT canonical_reference_id,
       array_agg(product_id)
FROM product_reference_link
WHERE decision = 'AUTO'
GROUP BY canonical_reference_id;
```

---

## 7. Canonical Reference 数据模型

建议建立一个明确的 reference knowledge base，而不是只存字符串。

### 7.1 `reference_entity`

```sql
CREATE TABLE reference_entity (
    canonical_reference_id VARCHAR PRIMARY KEY,
    brand_id               VARCHAR NOT NULL,
    canonical_reference    VARCHAR NOT NULL,
    series_id              VARCHAR NULL,
    product_type           VARCHAR NULL,
    status                 VARCHAR NOT NULL,
    created_at             TIMESTAMP NOT NULL,
    updated_at             TIMESTAMP NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);
```

例：

```text
canonical_reference_id = rolex:126610ln
brand_id                = rolex
canonical_reference     = 126610LN
series_id               = submariner
product_type            = watch
```

### 7.2 `reference_alias`

```sql
CREATE TABLE reference_alias (
    brand_id               VARCHAR NOT NULL,
    alias                  VARCHAR NOT NULL,
    alias_norm             VARCHAR NOT NULL,
    canonical_reference_id VARCHAR NOT NULL,
    alias_type             VARCHAR NOT NULL,
    confidence_tier        VARCHAR NOT NULL,
    source                 VARCHAR NOT NULL,
    PRIMARY KEY (brand_id, alias_norm, canonical_reference_id)
);
```

`alias_type` 可包括：

```text
EXACT_OFFICIAL
FORMAT_VARIANT
MARKET_VARIANT
LEGACY_NAME
MODEL_PLUS_SUFFIX
OCR_CONFUSION_ALIAS   # 默认不得直接 AUTO，需要二次证据
```

这里必须保留“一 alias -> 多 canonical reference”的可能性，不能在数据结构层强行唯一；否则会把歧义隐藏掉。

### 7.3 `reference_extraction`

每次抽取都保留 provenance：

```sql
CREATE TABLE reference_extraction (
    product_id              VARCHAR,
    candidate_raw           VARCHAR,
    candidate_norm          VARCHAR,
    source_field            VARCHAR,
    extractor               VARCHAR,
    role                    VARCHAR,
    role_score              DOUBLE,
    char_start              INT,
    char_end                INT,
    image_id                VARCHAR NULL,
    ocr_bbox                JSON NULL,
    created_at              TIMESTAMP
);
```

这使每个自动匹配结果都可以回答：

```text
“为什么这条商品被判成 126610LN？”
```

而不是只有一个不可解释的 similarity score。

---

## 8. Reference Canonicalization：只允许可逆、可审计规则

不要做激进 fuzzy normalization。

建议分层：

### Tier 0：字符标准化

安全操作：

```text
Unicode NFKC
全角 -> 半角
英文统一 uppercase
首尾 whitespace 去除
连续空白折叠
标准化 dash 字符
```

### Tier 1：品牌内格式规则

例如某品牌官方 reference 允许：

```text
126610 LN
126610-LN
126610LN
```

只有确定是**同一品牌编号语法**时，才把空格/连字符折叠。

### Tier 2：别名映射

通过知识库显式记录：

```text
alias -> canonical_reference
```

### 禁止的 normalization

不要默认做：

```text
任意删除全部标点
编辑距离 <= 1 就相同
O 自动改 0
I 自动改 1
去掉所有前后缀
只保留数字
```

例如：

```text
116610LN
126610LN
```

只差一个字符，但绝不能合并。

因此 canonicalization 的目标不是“让字符串尽量像”，而是：

> **只消除已知、可证明不改变 reference 身份的表面差异。**

---

## 9. 抽取层：structured field、文本、OCR 三路并行

建议不要把所有信息先拼成一句文本再交给一个 LLM。

应保留三路独立 evidence：

```text
A. structured_reference
B. title / description extraction
C. OCR extraction
```

### 9.1 Structured reference

如果来源有独立型号字段，优先级最高，但仍需：

- 编号角色校验；
- 品牌格式校验；
- alias/canonical lookup；
- 与标题/OCR 的冲突检查。

### 9.2 文本抽取

优先采用：

```text
品牌规则 / regex
+ 字符级 token classifier
+ 小模型/LLM 只处理难例
```

LLM 输出必须是 span，而不是自由生成：

```json
{
  "reference_spans": [
    {"text": "126610LN", "start": 18, "end": 26}
  ]
}
```

禁止：

```json
{"reference": "我推测应该是126610LN"}
```

### 9.3 OCR

OCR 不直接改写成 canonical ref；先保留原 token、bbox、图像来源。

例如：

```text
OCR: 12661OLN
candidate proposer: 126610LN
```

只能进入：

```text
POSSIBLE_OCR_ALIAS
```

不能因为编辑距离 1 自动合并，除非另一路 evidence 也支持同一 canonical reference。

---

## 10. 编号角色分类必须放在 linking 前

`d` 目录此前已分析过 `parts-distributor-sku-classifier`，这里应与本文架构组合，而不是重复造一个 matcher。

每个抽到的字母数字串先分类：

```text
BRAND_REFERENCE
SOURCE_SKU
SHOP_SKU
SERIAL_NUMBER
WARRANTY_NUMBER
ACCESSORY_COMPAT_REFERENCE
OTHER_IDENTIFIER
AMBIGUOUS
```

只有高置信：

```text
BRAND_REFERENCE
```

才允许进入 exact resolver。

尤其要拦截：

```text
“适配 Rolex 126610LN 表带”
```

这里的 `126610LN` 是被适配商品 reference，不是当前售卖商品 reference。

这类错误非常危险，因为字符串完全 exact，普通 exact matcher 反而会 100% 自信地错。

---

## 11. Exact Resolver：真正的主生产路径

输入：

```text
brand_id
candidate_norm
product_role
source
```

输出不是单一值，而应是集合：

```text
[]
[ref_1]
[ref_1, ref_2]
```

决策：

```python
if len(candidates) == 0:
    return UNRESOLVED

if len(candidates) > 1:
    return AMBIGUOUS

return UNIQUE(candidates[0])
```

然后继续做冲突检查，不能一 unique 就直接 AUTO。

建议自动放行条件：

```text
1. brand 已确定；
2. candidate 角色是 BRAND_REFERENCE；
3. alias lookup 唯一；
4. 没有另一个高可信 evidence 指向不同 reference；
5. 当前商品不是配件/表带/盒证等非主体商品；
6. reference 与品牌规则兼容；
7. 来源字段不存在强冲突。
```

满足后才：

```text
AUTO_LINK(canonical_reference_id)
```

---

## 12. PECOS 在本项目中的正确角色：Candidate Proposer

PECOS 不应该回答：

```text
“这条商品就是 126610LN”
```

它应该回答：

```text
“如果必须去 reference KB 里找，最值得核查的是这 5 个：
126610LN
126610LV
116610LN
116610LV
124060
”
```

然后 `Evidence Verifier` 再回查原始证据。

### 12.1 Label 定义

```text
label = canonical_reference_id
```

例如：

```text
rolex:126610ln
rolex:126610lv
rolex:116610ln
```

### 12.2 Input 文本

建议显式分字段：

```text
[SOURCE] she-dang-jia
[BRAND] rolex
[TITLE] 劳力士潜航者型黑水鬼 41mm 126610LN 全套
[DESC] ...
[OCR] 126610LN ORIG ROLEX ...
[PT] watch
[SERIES] submariner
```

但不要把 source SKU 等 identifier 无脑拼进去，可以先 mask：

```text
[SOURCE_SKU]
[SERIAL]
```

减少模型学习错误捷径。

### 12.3 Feature 选择

第一版不需要上巨大 Transformer。

对于 reference 这类强字符串任务，可以先用：

```text
字符 n-gram TF-IDF
+ 品牌/系列 one-hot
+ 来源 one-hot
+ 商品类型
+ 抽取候选模式特征
```

结合 PECOS `XLinearModel`。

原因：

- reference 的信息主要在局部字符模式；
- character n-gram 对连字符/空格/局部噪声有效；
- 推理便宜；
- 输出空间可扩展；
- 可解释性比端到端 LLM 好。

如果后续证实复杂上下文确实有帮助，再试 XR-Transformer。

### 12.4 训练数据

```text
Positive：
高可信 structured reference -> canonical entity
人工确认样本
多源已经严格 exact 对齐的记录

Hard negative：
同品牌同系列相邻 reference
编辑距离 1 的 reference
同前缀不同后缀
同一商品文本里出现的配件兼容 reference
SKU/serial 形似 reference
```

尤其不要用随机 negative 为主，因为随机不同品牌太容易，不能训练出防 false positive 的边界。

---

## 13. PECOS 可落地的最小代码骨架

官方 PECOS 的 X-Linear API 允许用稀疏矩阵训练。

概念代码：

```python
from pecos.xmc.xlinear.model import XLinearModel
from pecos.xmc import Indexer, LabelEmbeddingFactory

# X: N x D，商品字符 ngram/结构化稀疏特征
# Y: N x L，canonical_reference_id label

label_feat = LabelEmbeddingFactory.create(Y, X)
cluster_chain = Indexer.gen(label_feat)

model = XLinearModel.train(
    X,
    Y,
    C=cluster_chain,
)

model.save("reference_xmc")

# inference
P = model.predict(X_query)
# 只取 top-k candidate，不在此处 AUTO_MATCH
```

工程上建议 `topk=5~20`，具体由 recall/成本曲线决定。

最终服务接口不要返回：

```json
{"reference": "126610LN", "confidence": 0.93}
```

而应该返回：

```json
{
  "candidates": [
    {"canonical_reference_id": "rolex:126610ln", "score": 0.93},
    {"canonical_reference_id": "rolex:126610lv", "score": 0.51}
  ],
  "decision": "CANDIDATE_ONLY"
}
```

这样可以从 API 契约层防止下游把模型分数误当身份结论。

---

## 14. Evidence Verifier：模型候选必须“回到证据”

这是相比原论文最重要的加强。

假设 PECOS 给出：

```text
rolex:126610ln
```

Verifier 必须检查：

### 14.1 是否有 candidate surface form 出现在原始数据

允许：

```text
126610LN
126610 LN
126610-LN
```

但这些变体必须来自 alias KB，不是动态 fuzzy 猜测。

### 14.2 是否由多个独立 evidence 支持

例如：

```text
title: 126610LN
OCR:   126610LN
```

比：

```text
title: 黑水鬼
model guess: 126610LN
```

安全得多。

### 14.3 是否存在反证

任何以下情况都应阻断自动放行：

```text
structured_ref = 126610LV
但 title 提取到 126610LN

标题同时出现：126610LN / 116610LN

商品类型 = strap/accessory
标题 = 适配 126610LN

OCR 中出现 serial，但没有 reference 证据
```

对 precision-first 系统：

```text
negative evidence > positive score
```

即一个明确冲突可以否决多个弱支持。

---

## 15. 推荐决策矩阵

| 场景 | 决策 |
|---|---|
| 结构化 reference 通过品牌规则，唯一映射，且无冲突 | AUTO_LINK |
| 标题明确 reference，唯一 alias，role=BRAND_REFERENCE，且无冲突 | AUTO_LINK |
| 标题 + OCR 独立命中同一 canonical reference | AUTO_LINK |
| OCR 单独命中 reference | 默认 REVIEW/ABSTAIN |
| PECOS 高分但原文/OCR 无 reference 证据 | ABSTAIN |
| PECOS 候选可通过官方 alias 回查，且另一路证据支持 | 可进入高门槛 AUTO tier |
| 两个高可信字段指向不同 reference | CONFLICT_REVIEW |
| 一个 alias 对应多个 reference | ABSTAIN |
| 配件/表带出现兼容型号 | REJECT_AS_PRODUCT_REFERENCE |
| 只有图片外观相似 | 不允许 AUTO_LINK |

这里最重要的一行是：

> **模型很确信，但没有硬证据，不等于可以自动合并。**

---

## 16. 跨源匹配不再做 pairwise classifier

三个来源共 100 万–1000 万记录，如果直接做跨表笛卡尔积：

```text
N x M
```

一定不可接受。

而 reference linking 后：

```text
source product
  -> canonical_reference_id
```

跨源匹配只需 hash join / group by：

```sql
SELECT a.product_id AS a_id,
       b.product_id AS b_id,
       a.canonical_reference_id
FROM product_link a
JOIN product_link b
  ON a.canonical_reference_id = b.canonical_reference_id
WHERE a.source <> b.source
  AND a.decision = 'AUTO'
  AND b.decision = 'AUTO';
```

复杂模型只发生在每条商品的 linking 阶段，不发生在所有商品 pair 上。

从系统复杂度上，可以从接近平方级比较问题变成接近线性处理 + 索引查找。

---

## 17. 增量架构

当前数据会持续更新，建议把 reference linker 做成幂等流水线。

### 17.1 事件格式

```json
{
  "source": "leixiaoan",
  "product_id": "123",
  "version": 17,
  "title": "...",
  "description": "...",
  "structured_reference": "...",
  "images": ["..."],
  "updated_at": "..."
}
```

### 17.2 处理顺序

```text
upsert raw product
-> normalize
-> extract
-> exact resolve
-> if unresolved: candidate propose
-> evidence verify
-> decision
-> upsert product_reference_link
```

每步都应以：

```text
(source, product_id, version)
```

作为幂等键。

### 17.3 Reference KB 更新

当 alias KB 或 canonical reference KB 更新时，不必重跑全部原始抓取。

维护：

```text
normalization_version
extractor_version
alias_kb_version
model_version
policy_version
```

只重算受影响记录。

---

## 18. 建议的服务拆分

第一版不必微服务过度设计，可以逻辑模块化、物理上少服务。

### `reference-kb`

负责：

```text
canonical reference
brand / series metadata
alias
reference syntax rules
```

### `reference-extractor`

负责：

```text
structured field
text spans
OCR spans
identifier role
```

### `reference-resolver`

负责：

```text
exact lookup
ambiguity detection
context filtering
```

### `reference-candidate`

负责：

```text
PECOS top-k
```

### `reference-policy`

负责：

```text
AUTO / REVIEW / REJECT / ABSTAIN
conflict gates
```

即使物理上都部署在一个 Python/FastAPI 服务里，也建议代码结构保持这个边界，防止未来“模型 score 直接写 match_result”。

---

## 19. 推荐结果表：不要只有 matched=true/false

```sql
CREATE TABLE product_reference_link (
    source                    VARCHAR NOT NULL,
    product_id                VARCHAR NOT NULL,
    canonical_reference_id    VARCHAR NULL,
    decision                  VARCHAR NOT NULL,
    decision_tier             VARCHAR NULL,
    primary_evidence_type     VARCHAR NULL,
    primary_evidence_value    VARCHAR NULL,
    resolver                  VARCHAR NULL,
    resolver_score            DOUBLE NULL,
    conflict_codes            JSON NULL,
    normalization_version     VARCHAR NOT NULL,
    extractor_version         VARCHAR NOT NULL,
    alias_kb_version          VARCHAR NOT NULL,
    model_version             VARCHAR NULL,
    policy_version            VARCHAR NOT NULL,
    updated_at                TIMESTAMP NOT NULL,
    PRIMARY KEY (source, product_id)
);
```

`decision`：

```text
AUTO
REVIEW
ABSTAIN
REJECT
```

`decision_tier`：

```text
A_STRUCTURED_EXACT
B_TITLE_EXACT
C_MULTI_EVIDENCE_EXACT
D_MODEL_VERIFIED
```

以后 precision 下降时，可以精确定位哪个 tier 出问题，而不是重新猜整个模型。

---

## 20. 评测指标必须从 F1 改为 Precision / False Positive First

论文同时看 precision、recall、coverage、F1；当前需求应重新排序：

```text
P0: False Positive count / rate
P1: Precision lower confidence bound
P2: Coverage
P3: Recall
P4: F1
```

甚至可以不把 F1 放入发布门槛。

### 20.1 分 tier 评测

分别测：

```text
structured exact precision
text exact precision
OCR exact precision
model-verified precision
```

不能只报总 precision，否则高质量大量简单样本会掩盖某个危险 tier。

### 20.2 Hard-case benchmark

黄金集应大量包含：

```text
相邻 reference：126610LN vs 126610LV
新旧款：116610LN vs 126610LN
一字符差异
相同系列不同尺寸
相同外观不同材质
配件适配型号
SKU/serial 假型号
标题多个 reference
OCR O/0 I/1 S/5
字段冲突
品牌缺失
错误品牌
```

### 20.3 “零误匹配”需要统计意义

如果一个 AUTO tier 在测试中 0 个 false positive，也不意味着真实错误率是 0。

若希望在 95% 置信水平下把真实 error rate 上界压到约 `1e-4`，零错误样本量需要大约 3 万条量级：

```text
(1 - 1e-4)^n <= 0.05
n ≈ 29956
```

因此几百条人工标注适合：

```text
训练/校准 hard cases
```

而大规模 precision 验证应使用：

```text
高质量自动黄金标签 + 分层人工抽检 + 线上持续审计
```

---

## 21. 论文结果给当前系统的一个反直觉启示

论文中，语义模型加入更多弱标签后，整体 recall/coverage 会提高，但在“非品牌 query”上 false alarm 也会上升。

这说明：

> **更多训练数据、更强模型、更高 recall，不等于更安全。**

当前系统最危险的优化方向就是：

```text
“这个模型 recall 又高了 5%，所以替换 exact rule。”
```

正确方式应该是：

```text
旧 AUTO tier 原样保留
新模型只扩大 candidate/review coverage
只有当新 tier 单独证明 precision 足够高，才获得 AUTO 权限
```

这是一种单调扩张式发布：不会为了 recall 牺牲已有高精度区间。

---

## 22. 与论文不同：我们应把 product-type filter 升级成多重硬约束

论文用 product type 做 brand disambiguation。

腕表 reference 可用的约束更多：

```text
brand
series
product_type
watch/accessory role
case diameter
movement
material
gender/collection
source-specific field semantics
```

但这些属性的使用方式应主要是：

```text
否决 / 缩小候选
```

而不是：

```text
推断 reference
```

例如：

```text
候选 reference = 126610LN
已知该 ref 是 Rolex Submariner 41mm
商品明确写 36mm Datejust
=> 直接 REJECT candidate
```

但不能反过来：

```text
商品是 Rolex + Submariner + 41mm
=> 猜它一定是 126610LN
```

因为同规格仍可能有多个 reference。

---

## 23. 图片的正确角色

当前 Spec 明确“有图片可用”。

图片不要直接作为“同款”主键，而应该提供三类辅助信息：

### 23.1 OCR reference evidence

优先识别：

```text
保卡
吊牌
表背
标签
证书
盒贴
```

### 23.2 冲突否决

例如：

```text
文字候选是黑盘 reference
图片高置信识别为绿盘/不同表圈
```

可以把它作为 `CONFLICT_VISUAL`，阻止 AUTO。

### 23.3 人工 review 排序

图片相似度可以帮助 review queue 找最值得核对的候选，但不要越过 canonical reference 规则。

---

## 24. 直接可落地的 Phase 方案

## Phase A：先做“零模型”高精度主干

只做：

```text
brand canonicalization
reference syntax rules
identifier role rules/model
alias KB
exact resolver
conflict gate
provenance logging
```

先回答：

```text
在完全不用 PECOS/LLM 的情况下，可以安全覆盖多少商品？
```

这会成为后续所有模型的 precision 基线。

## Phase B：建立 hard-case 黄金集

从真实 unresolved/conflict 数据中采样：

```text
同系列近邻 ref
多个 ref
OCR 冲突
SKU/serial
配件
品牌歧义
```

人工只标最难部分。

## Phase C：PECOS Candidate Proposer

只对：

```text
UNRESOLVED
AMBIGUOUS
```

运行 top-k candidate。

初期全部进入：

```text
REVIEW
```

不改变 AUTO coverage。

## Phase D：建立 Model-Verified tier

只有当模型候选满足可审计硬验证：

```text
alias 回查
多证据一致
无冲突
候选唯一
```

并且独立测试达到目标 precision，才创建新的：

```text
D_MODEL_VERIFIED
```

AUTO tier。

---

## 25. 一份可直接实现的 policy 伪代码

```python
def link_reference(product):
    brand = resolve_brand(product)
    if not brand.is_unique:
        return abstain("BRAND_AMBIGUOUS")

    evidences = extract_reference_candidates(product)

    valid = []
    for ev in evidences:
        role = classify_identifier_role(ev, product)
        if role != "BRAND_REFERENCE":
            continue

        norm = normalize_reference(
            brand_id=brand.id,
            raw=ev.text,
        )

        refs = exact_lookup(
            brand_id=brand.id,
            alias_norm=norm,
        )

        if len(refs) == 1:
            valid.append((refs[0], ev))

    # 高可信 evidence 出现两个不同 canonical ref，直接冲突
    unique_refs = {r.id for r, _ in valid}
    if len(unique_refs) > 1:
        return review("REFERENCE_CONFLICT", valid)

    if len(unique_refs) == 1:
        ref = next(iter(unique_refs))
        if has_negative_conflict(product, ref, valid):
            return review("NEGATIVE_CONFLICT", valid)
        if evidence_tier_allows_auto(valid):
            return auto_link(ref, valid)

    # Exact 主路径没有安全结果，模型只能提候选
    candidates = pecos_topk(product, brand_id=brand.id, k=10)

    verified = []
    for ref in candidates:
        result = verify_candidate_against_evidence(
            product=product,
            canonical_reference=ref,
        )
        if result.pass_hard_checks:
            verified.append((ref, result))

    if len(verified) != 1:
        return abstain("MODEL_NOT_UNIQUELY_VERIFIED", verified)

    ref, verification = verified[0]

    if not model_verified_tier_enabled():
        return review("MODEL_VERIFIED_BUT_NOT_AUTO_ENABLED", verified)

    return auto_link(ref.id, verification)
```

最关键的是最后两段：

> 即使模型候选被 verifier 选成唯一，也可以通过 feature flag 暂时只进 review，直到离线 precision 验证通过。

---

## 26. 数据与模型版本治理

生产匹配结果必须能复现。

建议每条结果写入：

```text
raw_data_version
normalization_version
brand_kb_version
reference_kb_version
role_model_version
extractor_version
pecos_model_version
policy_version
```

因为未来最常见的问题不是：

```text
“模型坏了”
```

而是：

```text
“昨天新加的 alias 把两个 reference 意外合并了”
```

KB/规则版本同样属于模型治理对象。

---

## 27. 线上监控

建议至少监控：

```text
AUTO coverage by source
AUTO precision audit by tier
ABSTAIN rate
CONFLICT rate
multi-reference rate
unknown reference rate
brand-specific false positive
source-specific false positive
OCR-only auto rate（理想情况下接近 0，除非单独验证）
model candidate -> verified conversion rate
```

出现以下情况自动降级相关 tier：

```text
某品牌 conflict 激增
某来源字段 schema 改变
某 extractor 新版本输出多个 reference 激增
某 alias 一对多比例上升
人工 audit 出现 false positive
```

系统要支持：

```text
D_MODEL_VERIFIED AUTO -> REVIEW
```

的即时策略切换，而不需要重新训练模型。

---

## 28. 与已有 d 调研结果的组合方式

这篇论文不是替代已有研究，而是可以把多个结果串成一条完整 pipeline：

```text
DeepBlocker / pyJedAI
    -> 更偏大规模候选/ER 工具箱

parts-distributor-sku-classifier
    -> 编号角色门禁

PAM / MOON / 多模态属性抽取
    -> OCR/图像辅助 evidence

Query Brand Entity Linking（本文）
    -> mention -> canonical entity 的主 linking 架构

TransClean / GraLMatch
    -> 链接完成后的跨源图一致性审计

Confidence Classifier / Conformal Risk Control
    -> AUTO tier 的统计校准与拒识
```

其中本文最适合充当“主骨架”：

```text
抽取 -> canonical linking -> conflict/abstain -> canonical id
```

其它研究作为抽取、审计、风险控制模块插进去。

---

## 29. 不建议做的方案

### 29.1 直接 pairwise embedding top-1

```text
A 商品 embedding
B 商品 embedding
cosine > 0.95 -> same
```

腕表同系列不同 reference 外观与文字都可能极近，风险太高。

### 29.2 LLM 直接回答“是否同款”

LLM 可以抽取/解释/辅助 review，但不能拥有自动合并权。

### 29.3 编辑距离近就归一

reference 是 identifier，不是自然语言。

### 29.4 模型 top-1 必选

必须存在：

```text
NIL / ABSTAIN
```

### 29.5 为追 recall 把模糊 alias 写入 exact KB

Alias KB 一旦进入 exact path，就相当于获得最高信任等级；模糊映射必须放在低 tier。

---

## 30. 最终推荐方案

如果只采纳本文一个核心思想，我建议是：

> **把当前任务从“跨来源商品相似度匹配”重新定义成“每条商品独立链接到 canonical reference entity”，并采用 exact-first、model-as-candidate、ambiguity-abstain 的架构。**

推荐生产决策链：

```text
1. canonical brand
2. extract identifier spans
3. identifier role gate
4. brand-aware canonicalization
5. exact alias lookup
6. uniqueness + conflict checks
7. AUTO if hard evidence passes
8. otherwise PECOS top-k candidate
9. evidence back-verification
10. still ambiguous -> ABSTAIN/REVIEW
11. cross-source match = canonical_reference_id exact equality
```

这样做有四个直接收益：

1. **满足 precision-first**：模型不能绕过硬规则。
2. **可扩展**：避免千万级数据做跨源笛卡尔 pair 比较。
3. **可审计**：每个 AUTO 结果都能追溯到具体 reference span/字段/OCR。
4. **可持续增量**：新增来源、新品牌、新 alias、新模型都只更新 linking 层，不破坏最终实体定义。

论文真正给我们的启发不是“PECOS 很强”，而是：

> **在超大实体空间中，规则与模型应该有不同权限：规则负责做高精度确定性决策，模型负责扩大候选覆盖；当系统无法唯一证明实体身份时，拒识本身就是正确结果。**

这与 Notion Spec 的“绝对不能误匹配、允许漏匹配”高度一致。

---

## 31. 参考资料

- Dong Liu, Sreyashi Nag. [Query Brand Entity Linking in E-Commerce Search](https://arxiv.org/abs/2502.01555), arXiv:2502.01555v2, 2025.
- Amazon. [PECOS - Prediction for Enormous and Correlated Spaces](https://github.com/amzn/pecos).
- Hsiang-Fu Yu et al. [PECOS: Prediction for Enormous and Correlated Output Spaces](https://arxiv.org/abs/2010.05878).
