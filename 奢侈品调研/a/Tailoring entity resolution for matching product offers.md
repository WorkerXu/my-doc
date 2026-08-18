# Tailoring entity resolution for matching product offers：从 Product Code 抽取到二奢腕表 Reference Entity Linking 的落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取论文 **Tailoring entity resolution for matching product offers**（Köpcke, Thor, Thomas, Rahm，EDBT 2012）进行深入分析。

- 论文介绍：<https://dbs.uni-leipzig.de/research/publications/tailoring-entity-resolution-for-matching-product-offers>
- 论文 PDF：<https://dbs.uni-leipzig.de/files/research/publications/2012-3/pdf/edbt2012_final.pdf>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

这篇论文虽然较早，但它击中了当前需求里最容易造成灾难性 false positive 的问题：

> **标题里出现一个“看起来像型号”的字符串，不代表它就是当前售卖商品自己的型号。**

论文举的典型例子是配件商品：一个电池标题里可能同时出现“电池自己的 product code”和“它兼容的相机 product code”。如果系统只要看到型号就匹配，就会把配件错误链接到主商品。

二奢腕表完全有同构问题：

- “适用 Rolex 126610LN 的表带”里出现 `126610LN`，但商品不是 Rolex 126610LN 腕表；
- “AP 15500ST 原装盒证”里出现 `15500ST`，商品可能只是附件；
- “劳力士黑水鬼 126610LN / 116610LN 对比”可能同时出现两个 reference；
- 店铺内部 SKU、平台货号、鉴定编号也可能长得像 reference。

因此，本论文对当前 Spec 最值得直接迁移的并不是它最后的 SVM matcher，而是两条原则：

1. **先把“型号候选”从脏文本中抽出来，再验证这个编号是否真的属于当前商品；**
2. **Product Code / Reference 应是高权重的结构化证据，而不是普通文本相似度的一部分。**

但论文原始实现不能直接上线：其 product-code 抽取平均 precision 只有约 79%，而当前需求要求 precision 极端优先，基本不能接受任何自动误合并。

所以推荐把论文方案重构成：

```text
三源商品记录
   │
   ▼
来源字段标准化
   │
   ▼
品牌归一化 + 商品角色识别
   │
   ▼
Reference 候选抽取
   │
   ▼
候选角色判定：这是“当前商品型号”还是“兼容/提及/平台编号”？
   │
   ▼
品牌内 Reference 规范化 + 可信目录验证
   │
   ▼
多证据一致性校验
   │
   ├── 证据充分且无冲突 → Link 到 Reference Entity
   └── 任一冲突/不确定 → ABSTAIN，不自动匹配

商品 A -> Reference Entity X
商品 B -> Reference Entity X
=> A、B 为同一 reference
```

最终不要训练一个“商品 A 和商品 B 是否相同”的黑盒分类器作为放行器，而是建立一个全局 **Reference Entity Registry**，每条商品独立链接到 `(brand, canonical_reference)`。这样自动合并的业务规则可以保持非常简单、可审计：

> **只有两条记录都被高置信度链接到同一个品牌下的同一个 canonical reference，才自动认为是同商品。**

---

## 1. 论文解决的原始问题

论文研究的是跨电商站点的 Product Offer Matching：判断不同商家发布的 offer 是否指向同一个真实商品。

它指出普通 Entity Resolution 在商品场景特别困难，因为：

- 商品品类跨度大；
- 同系列不同 variant 极其相似；
- 字段经常缺失或错误；
- title / description 非结构化；
- 同一个商品在不同商户里命名差异很大；
- accessory 商品会在标题中大量提及其兼容主商品型号。

这些问题与当前雷小安、腕表之家、奢当家三源数据高度一致。

论文把一个 offer 抽象成少量字段：

```text
Offer {
  title,
  description,
  manufacturer,
  price,
  optional UPC
}
```

当前二奢场景可自然映射为：

```text
LuxuryOffer {
  source,
  source_item_id,
  title,
  description,
  brand_raw,
  reference_raw?,
  category?,
  images[],
  price?,
  source_specific_ids...
}
```

最大的共同点是：真正有判别力的 identifier/reference 经常没有独立字段，而是埋在 title / description 里。

---

## 2. 论文整体系统架构

论文 Figure 4 的系统可以概括为三阶段：

```text
                 ┌─────────────────────┐
Raw Product  ──> │ 1. Pre-processing   │
Offers           │                     │
                 │ - manufacturer clean│
                 │ - categorization    │
                 │ - product code ext. │
                 └─────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │ 2. Training         │
                 │                     │
                 │ labeled pairs       │
                 │ matcher features    │
                 │ category SVM        │
                 └─────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │ 3. Application      │
                 │                     │
                 │ blocking            │
                 │ feature calculation │
                 │ match / non-match   │
                 └─────────────────────┘
```

### 2.1 Manufacturer cleaning

论文先清洗 manufacturer：

- 字符串相似度；
- synonym list；
- 既有 manufacturer dictionary；
- 当 manufacturer 字段缺失时，从 title / description 里反查品牌词典。

这是一个非常重要但常被忽略的设计：**型号只能在品牌语境内解释。**

例如 `5711`、`15500`、`126610` 单独看只是数字串；只有和 Patek Philippe、Audemars Piguet、Rolex 等品牌绑定后，才能成为可控的实体键。

因此当前系统的真正主键不是 `reference`，而应是：

```text
(brand_id, canonical_reference)
```

而不是：

```text
canonical_reference
```

### 2.2 Product categorization

论文用 Modified Naive Bayes 做商品分类，目的是：

1. 只在同品类里比较，降低 candidate pair 数量；
2. 为不同品类训练不同 match strategy。

对于当前腕表为主的需求，不一定需要复杂的全品类分类器，但至少应该做 **商品角色分类**：

```text
WATCH
WATCH_ACCESSORY
BOX_PAPERS
STRAP_BRACELET
PART
SERVICE_ITEM
UNKNOWN
```

原因不是为了提升 recall，而是为了保护 precision。

只要识别为 `WATCH_ACCESSORY / BOX_PAPERS / STRAP_BRACELET / PART`，标题中出现的腕表 reference 默认不能作为当前商品 reference 自动放行。

### 2.3 Learning-based matcher

论文把匹配转成二分类问题，主要特征包括：

- title TF/IDF similarity；
- title trigram Jaccard；
- description TF/IDF similarity；
- description trigram Jaccard；
- product-code matcher；
- SVM 输出 match / non-match。

训练时还不是随机采一堆 pair，而是尽量选择“相似度达到一定门槛”的 pair，让训练集里有足够困难负例，而不是被海量 trivial non-match 淹没。

这个 hard-pair 思想可以直接迁移到当前黄金标签构建：

> 几百对人工标签不要浪费在“Rolex vs Cartier”这种显然不匹配的 pair 上，而要集中标注同品牌、同系列、reference 只差 1～2 个字符、或标题里同时出现多个 reference 的 hard cases。

### 2.4 Blocking

论文在线应用时先用：

```text
same cleaned manufacturer
AND
same category
```

做 blocking，再运行 matcher。

当前系统应进一步简化为：

```text
brand blocking
    ↓
reference candidate lookup
    ↓
exact canonical reference entity
```

当 Reference Entity Registry 建起来后，甚至不需要商品两两生成 pair。

---

## 3. Product Code Extraction：论文最值得迁移的部分

论文 Algorithm 1 的核心流程如下：

```text
getProductCode(offer):
    offer = removeFeatures(offer)
    tokens = tokenize(offer)
    tokens = removeFrequentTokens(tokens)
    candidates = generateCandidates(tokens)
    candidates = regexFilter(candidates)
    code = webVerification(candidates)
    return code
```

它不是“对 title 跑一个正则就完了”，而是分层过滤。

### 3.1 Step 1：先移除常见数值特征

论文会先去掉：

- dimensions；
- weight；
- voltage；
- capacity；
- color 等。

这是为了避免把 `7.2V`、`680mAh` 这种字母数字混合 token 错当型号。

对应腕表领域，应先识别并降权/移除：

```text
尺寸：40mm / 41 mm / 42毫米
年份：2021 / 2022年
价格：¥86800 / 8.68w
材质：18K / 750 / 904L
防水：100m / 300m
机芯：Cal.3235 / 3235
重量：155g
表径：41
认证/库存号：平台自定义数字串
```

尤其注意：**机芯号不能和 reference 混淆。**

`3235` 对 Rolex 来说很有业务意义，但它不是 `126610LN` 的 reference。

### 3.2 Step 2：Tokenization + stop/frequent token filtering

论文引入一个很实用的 manufacturer-based frequency：

```text
N(t, m) = 品牌 m 的 offer 中包含 token t 的记录数
N(t)    = 全部 offer 中包含 token t 的记录数
score(t,m) = N(t,m) / N(t)
```

只有 token 对某 manufacturer 足够专有时，才继续被当作 code 候选。

这个思想在腕表上非常适合做 **弱先验**：

- `126610LN` 高度 Rolex-specific；
- `15500ST` 高度 AP-specific；
- `WSSA0018` 高度 Cartier-specific；
- `41MM`、`AUTOMATIC`、`2022` 则不是 reference-specific。

但它只能做候选排序，不能做最终放行，因为跨品牌也可能出现格式相似或碰撞。

### 3.3 Step 3：生成最多 3 个连续 token 的 code candidate

论文考虑的 product code 可能被空格拆开，例如：

```text
HF S10
```

所以会生成最多 3 个连续 token 的组合，再用 pattern 筛选。

腕表 reference 也不能只看单 token：

```text
210.30.42.20.01.001
IW 371605
Ref. 126610 LN
15500ST.OO.1220ST.01
```

因此建议 Candidate Generator 同时生成：

- 单 token；
- 相邻 2-token；
- 相邻 3-token；
- 去掉 `Ref / Reference / 型号 / 编号` 前缀后的片段；
- 品牌特定 pattern 命中的 span。

### 3.4 Step 4：Pattern filtering

论文手工维护 regex pattern，典型特点是：

- 同时含字母和数字；
- 符合 manufacturer 常见 code 结构。

当前系统应把 pattern 变成 **brand-specific configuration**，而不是全局一个大正则。

例如配置可以长这样：

```yaml
brands:
  rolex:
    aliases: [Rolex, 劳力士]
    reference_patterns:
      - '(?<![0-9A-Z])\d{5,6}[A-Z]{0,3}(?![0-9A-Z])'
    safe_normalization:
      uppercase: true
      remove_spaces: true
      remove_hyphen: false

  omega:
    aliases: [Omega, 欧米茄]
    reference_patterns:
      - '\d{3}[.]\d{2}[.]\d{2}[.]\d{2}[.]\d{2}[.]\d{3}'
    safe_normalization:
      uppercase: true
      remove_spaces: true
      remove_dots: false
```

关键点是 **safe normalization 必须品牌化**。

不要全局粗暴执行：

```text
删掉所有点、横杠、字母后缀
```

否则会把本来不同的 variant/reference 折叠到一起。

### 3.5 Step 5：Web Verification

论文最有意思的一步是 external verification。

对每个 code candidate 去搜索引擎检索，然后看搜索结果里 manufacturer 出现比例。比如：

```text
候选 HL-XF51
搜索结果基本都和 Hahnel 一起出现
=> 更可能是 Hahnel 自己的 code

候选 NP-FF51
搜索结果都和 Sony 一起出现
=> 虽然它出现在 Hahnel 电池标题里，但不是 Hahnel 电池自己的 code
```

这一步实际上解决的是：

> **编号角色归属（ownership / role validation）**

而不只是“这个字符串是不是一个合法编号”。

这正是当前系统最需要单独实现的一层。

---

## 4. 为什么论文原方案不能直接用于当前 Spec

论文原始数据规模约 10 万 offer、71 个品类；product-code 抽取在非配件类覆盖较好，但整体平均 precision 大约 79%，并且 accessory 类更难。

论文里 web verification 能显著提升 extraction quality，但即便如此仍不是“几乎 0 false positive”。

当前 Notion Spec 的约束更苛刻：

```text
同一个商品 = 同一个 reference number
绝对不能误匹配
可以漏匹配
允许人工标注几百对
100万～1000万条
持续增量
有图片
```

因此有三个必须重构的地方。

### 4.1 SVM 不能作为最终放行器

普通二分类器优化的通常是 F1 / accuracy / cross entropy。

即使总体 precision 很高，也不能保证高风险区域没有 false positive。

当前业务更适合：

```text
candidate extraction / model score
             ↓
       deterministic gate
             ↓
  MATCH / ABSTAIN / CONFLICT
```

而不是：

```text
model probability > 0.5 => MATCH
```

### 4.2 “搜索结果验证”应改成“可信 Reference Registry 验证”

搜索引擎结果是动态、不可审计、容易受 SEO 和噪声影响的。

生产系统应维护自己的：

```text
Reference Entity Registry
```

每个实体至少包含：

```text
brand_id
canonical_reference
reference_aliases
collection / model_family (optional)
source_evidence
first_seen_at
last_seen_at
support_count
trusted_status
```

搜索/LLM 可以帮助发现新 reference，但不能直接把新结果升格为自动匹配真值。

### 4.3 Pairwise Matching 应改成 Entity Linking

如果三源各几百万记录，做 pairwise matching 会产生巨大的 candidate pair 管理复杂度。

但当前“同一个商品”的定义已经是同 reference，因此最自然的建模是：

```text
Offer -> Reference Entity
```

最终聚类只是：

```sql
GROUP BY brand_id, canonical_reference
```

这个架构比 pairwise graph 更简单，也更容易做到高 precision。

---

## 5. 推荐的生产架构

### 5.1 总体架构

```text
┌──────────────────────────────────────────┐
│ 雷小安 / 腕表之家 / 奢当家 Raw Data      │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ A. Source Adapter / Schema Normalization │
│ - 保留 raw 字段                          │
│ - 映射 title/brand/ref/images            │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ B. Brand Resolver                        │
│ raw brand -> canonical brand_id          │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ C. Product Role Classifier               │
│ WATCH / ACCESSORY / BOX / STRAP / PART   │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ D. Reference Candidate Extractor         │
│ explicit field + title + desc + OCR      │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ E. Reference Role Validator              │
│ OWN / COMPATIBLE_WITH / MENTIONED / SKU  │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ F. Brand-aware Canonicalizer             │
│ 只做可证明安全的 normalization           │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ G. Evidence & Conflict Gate              │
│ 多证据一致才 PASS；冲突立即 ABSTAIN      │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ H. Reference Entity Registry             │
│ UNIQUE(brand_id, canonical_reference)    │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│ I. Offer -> Reference Link               │
│ PASS / ABSTAIN / CONFLICT                │
└──────────────────────┬───────────────────┘
                       │
                       ▼
        同 Reference Entity 的跨源聚合
```

---

## 6. 数据模型

建议不要只保存一个 `reference` 字段，而要完整记录“证据链”。

### 6.1 offer 表

```sql
CREATE TABLE offer (
    id                  BIGSERIAL PRIMARY KEY,
    source              TEXT NOT NULL,
    source_item_id      TEXT NOT NULL,
    title_raw           TEXT,
    description_raw     TEXT,
    brand_raw           TEXT,
    explicit_ref_raw    TEXT,
    image_urls          JSONB,
    raw_payload         JSONB NOT NULL,
    ingested_at         TIMESTAMPTZ NOT NULL,
    UNIQUE(source, source_item_id)
);
```

### 6.2 normalized_offer 表

```sql
CREATE TABLE normalized_offer (
    offer_id             BIGINT PRIMARY KEY REFERENCES offer(id),
    brand_id             BIGINT,
    product_role         TEXT,
    product_role_score   DOUBLE PRECISION,
    canonical_reference  TEXT,
    link_status          TEXT NOT NULL,
    link_reason          TEXT,
    extractor_version    TEXT NOT NULL,
    updated_at           TIMESTAMPTZ NOT NULL
);
```

`link_status` 建议只允许：

```text
MATCHED
ABSTAIN_NO_REFERENCE
ABSTAIN_WEAK_EVIDENCE
CONFLICT_MULTIPLE_REFERENCE
CONFLICT_BRAND
CONFLICT_ROLE
MANUAL_REVIEW
```

不要用一个模糊的 `confidence = 0.87` 来掩盖业务语义。

### 6.3 reference_candidate 表

```sql
CREATE TABLE reference_candidate (
    id                    BIGSERIAL PRIMARY KEY,
    offer_id              BIGINT NOT NULL,
    raw_value             TEXT NOT NULL,
    canonical_value       TEXT,
    extraction_channel    TEXT NOT NULL,
    text_field             TEXT,
    span_start             INT,
    span_end               INT,
    context_before         TEXT,
    context_after          TEXT,
    candidate_role         TEXT,
    candidate_role_score   DOUBLE PRECISION,
    pattern_id             TEXT,
    ocr_confidence         DOUBLE PRECISION,
    registry_hit           BOOLEAN,
    evidence_json          JSONB
);
```

`extraction_channel`：

```text
EXPLICIT_FIELD
TITLE_REGEX
DESCRIPTION_REGEX
IMAGE_OCR
MANUAL
```

`candidate_role`：

```text
OWN_REFERENCE
COMPATIBLE_TARGET_REFERENCE
MENTIONED_REFERENCE
PLATFORM_SKU
MOVEMENT_NUMBER
SERIAL_NUMBER
UNKNOWN
```

### 6.4 reference_entity 表

```sql
CREATE TABLE reference_entity (
    id                   BIGSERIAL PRIMARY KEY,
    brand_id             BIGINT NOT NULL,
    canonical_reference  TEXT NOT NULL,
    model_family         TEXT,
    collection           TEXT,
    trust_level          TEXT NOT NULL,
    support_count        BIGINT NOT NULL DEFAULT 0,
    evidence_json        JSONB,
    created_at           TIMESTAMPTZ NOT NULL,
    updated_at           TIMESTAMPTZ NOT NULL,
    UNIQUE(brand_id, canonical_reference)
);
```

---

## 7. Reference 抽取：直接可实现的策略

## 7.1 证据优先级

建议按下面顺序处理：

```text
P0  人工确认 reference
P1  来源独立 reference 字段
P2  title 中品牌 pattern 命中
P3  图片 OCR 中命中 reference pattern
P4  description 中命中
P5  通用 NER / LLM 猜测
```

但优先级高不代表自动放行。

例如来源独立 `reference` 字段如果历史上实际混入“库存号”，仍需要先做字段质量审计。

### 7.2 Brand-specific Pattern Registry

维护版本化配置：

```text
reference_rules/
  rolex.yaml
  omega.yaml
  cartier.yaml
  audemars_piguet.yaml
  patek_philippe.yaml
  iwc.yaml
  ...
```

每个品牌至少包含：

- 品牌 aliases；
- reference 正则；
- 常见 reference 长度；
- 合法字符集；
- 安全 normalization；
- movement-number patterns；
- serial-number patterns；
- 明确不能当 reference 的 patterns。

这样比一个通用 LLM prompt 更可控、可审计、可回滚。

### 7.3 Context Window

每次抽到 reference candidate，必须保留其前后至少 20～40 字符上下文。

例如：

```text
“适用劳力士 126610LN 的橡胶表带”
```

不要只存：

```text
126610LN
```

要存：

```text
before = “适用劳力士 ”
after  = “ 的橡胶表带”
```

因为“适用 / for / compatible / replacement / 代用 / 同款适配”等 context 是识别 false positive 的核心证据。

---

## 8. Reference Role Validator：本次最建议新增的模块

这是从论文 Web Verification 思想迁移出来、但更适合当前业务的模块。

目标不是判断：

```text
“126610LN 是不是一个 Rolex reference？”
```

而是判断：

```text
“126610LN 是不是当前这条 listing 所售商品自己的 reference？”
```

### 8.1 规则优先

先做高 precision 的上下文规则：

```text
if candidate 前后出现：
  适用 / 兼容 / for / compatible / replacement /
  配 / 可用于 / 支持 / 对应 / 替换
then role = COMPATIBLE_TARGET_REFERENCE
```

```text
if 商品角色是 STRAP_BRACELET / BOX_PAPERS / PART
and candidate 是腕表 reference
then 默认不允许 OWN_REFERENCE
```

```text
if 同一个 title 出现两个或以上不同合法 reference
and 没有清楚的主 reference 标记
then CONFLICT_MULTIPLE_REFERENCE
```

### 8.2 小模型只处理规则无法覆盖的 case

可用几百条黄金标签训练一个轻量 classifier：

输入：

```text
brand
product_role
candidate
candidate context window
title
field/source
```

输出：

```text
OWN_REFERENCE
COMPATIBLE_TARGET_REFERENCE
OTHER_ID
UNKNOWN
```

模型可选：

- Logistic Regression + char/token n-gram；
- LightGBM；
- 小型中文 encoder；
- LLM 仅离线辅助标注。

因为 precision 优先，推荐先从可解释模型开始，不需要第一版就上大模型。

自动放行只允许：

```text
OWN_REFERENCE
```

其余全部 abstain。

---

## 9. Reference Canonicalization：必须保守

最危险的实现之一是：

```python
ref = re.sub(r'[^A-Z0-9]', '', ref.upper())
```

这对搜索召回可以，对最终 identity key 不安全。

推荐区分两个值：

```text
retrieval_key
canonical_reference
```

### 9.1 retrieval_key

允许更激进：

- uppercase；
- 去空格；
- Unicode NFKC；
- 可选去部分标点。

它只用于“找候选”。

### 9.2 canonical_reference

必须按品牌规则生成，只做确认不会改变 reference 语义的转换。

示例：

```text
raw: "126610 ln"
brand: Rolex
canonical: "126610LN"
```

但对于：

```text
15500ST.OO.1220ST.01
```

不能随意只保留：

```text
15500ST
```

因为 suffix 可能表达重要 variant 信息。

当前 Spec 明确定义“同 reference 才同商品”，所以 canonicalization 宁可少归一，也不能过归一。

---

## 10. Reference Entity Registry 如何冷启动

第一版不需要先购买一个完美的全球腕表目录。

可以从三源数据自身构建 **高精度种子集**。

### 10.1 Seed 规则

只有满足以下条件之一才创建 `TRUSTED` reference entity：

#### Rule A：两个独立来源的结构化字段一致

```text
同一 brand
雷小安 explicit_ref = X
腕表之家 explicit_ref = X
```

经过安全 canonicalization 后一致，可作为高质量 seed。

#### Rule B：结构化字段 + title 双证据一致

```text
explicit_ref = X
title regex 也抽到 X
无其他 reference 冲突
商品角色 = WATCH
```

#### Rule C：人工确认

黄金标签/人工审核直接进入 registry。

### 10.2 不应该自动建实体的情况

```text
仅 description 出现一次
仅 OCR 出现一次
仅 LLM 推断
仅视觉相似
品牌不确定
标题同时出现多个 reference
accessory listing
```

这些可以进入 `PROVISIONAL`，但不能被下游自动匹配当作真值。

---

## 11. 最终自动 MATCH 的 Hard Gate

建议第一版极度保守。

### 11.1 PASS 条件

一条 offer 自动链接到 Reference Entity，必须同时满足：

```text
1. brand_id 已高置信度确定；
2. product_role = WATCH；
3. 只存在一个 OWN_REFERENCE 候选；
4. candidate 通过 brand-specific pattern；
5. canonical_reference 能命中 TRUSTED registry；
6. 没有任何冲突 reference；
7. 没有 accessory / compatible context；
8. 至少有一个强证据，或两个独立中等证据；
9. extractor/version 在当前品牌 precision 审计中达到门槛。
```

强证据可定义为：

```text
- 可信来源的独立 reference 字段；
- 人工确认；
- 官方/高可信目录导入。
```

中等证据：

```text
- title pattern；
- 高质量 OCR；
- 第二来源一致记录；
```

### 11.2 推荐的 evidence policy

```text
AUTO_MATCH if:

  strong_evidence >= 1
  AND registry_hit = true
  AND conflicts = 0

OR

  independent_medium_evidence >= 2
  AND registry_hit = true
  AND conflicts = 0
```

### 11.3 否决优先于支持

任何一个强冲突都直接覆盖多个正向相似证据：

```text
candidate brand != resolved brand        => CONFLICT
multiple own refs                         => CONFLICT
accessory context                         => CONFLICT
explicit ref != title own ref             => CONFLICT
OCR high-conf ref != text ref             => CONFLICT
registry says different variant key       => CONFLICT
```

原则：

> **Negative evidence has veto power.**

这比把所有 feature 丢进一个加权平均分更符合“绝不能误匹配”。

---

## 12. 图片如何使用

Spec 说明“有图片可用”。

推荐图片只做 **证据增强与冲突否决**，不做 identity 正向主键。

### 12.1 OCR 优先于视觉 embedding

对腕表图片，优先 OCR：

- 表背刻字；
- 保卡；
- 吊牌；
- 证书；
- 盒贴；
- 发票/标签。

OCR 抽到的 reference 进入同一 Candidate Pipeline。

```text
image
  ↓
OCR
  ↓
reference pattern
  ↓
role / source type
  ↓
canonicalization
  ↓
evidence gate
```

### 12.2 Visual embedding 只做人工复核排序

同系列不同 reference 的腕表外观可能极其接近。

因此禁止：

```text
CLIP similarity > 0.95 => same reference
```

可以做：

```text
视觉很像 + reference 冲突
=> 提高人工复核优先级
```

或：

```text
text reference 缺失
=> 用图像召回可能的 model family
=> 再做 OCR / reference evidence 验证
```

但视觉不能越权成为自动 MATCH 的唯一依据。

---

## 13. 直接可落地的 Python 决策骨架

```python
from dataclasses import dataclass
from enum import Enum


class LinkStatus(str, Enum):
    MATCHED = "MATCHED"
    ABSTAIN = "ABSTAIN"
    CONFLICT = "CONFLICT"


@dataclass
class RefCandidate:
    raw: str
    canonical: str | None
    channel: str
    role: str
    registry_hit: bool
    strong: bool
    medium: bool


@dataclass
class LinkDecision:
    status: LinkStatus
    canonical_reference: str | None
    reason: str


def decide_reference_link(*, brand_id, product_role, candidates):
    if not brand_id:
        return LinkDecision(LinkStatus.ABSTAIN, None, "brand_unknown")

    if product_role != "WATCH":
        return LinkDecision(LinkStatus.ABSTAIN, None, "not_watch")

    own = [c for c in candidates if c.role == "OWN_REFERENCE"]

    canonical_values = {
        c.canonical for c in own
        if c.canonical is not None
    }

    if len(canonical_values) == 0:
        return LinkDecision(LinkStatus.ABSTAIN, None, "no_own_reference")

    if len(canonical_values) > 1:
        return LinkDecision(LinkStatus.CONFLICT, None, "multiple_reference_conflict")

    ref = next(iter(canonical_values))
    same = [c for c in own if c.canonical == ref]

    if not any(c.registry_hit for c in same):
        return LinkDecision(LinkStatus.ABSTAIN, None, "reference_not_trusted")

    strong_count = sum(c.strong for c in same)
    medium_channels = {c.channel for c in same if c.medium}

    if strong_count >= 1:
        return LinkDecision(LinkStatus.MATCHED, ref, "trusted_strong_evidence")

    if len(medium_channels) >= 2:
        return LinkDecision(LinkStatus.MATCHED, ref, "two_independent_medium_evidence")

    return LinkDecision(LinkStatus.ABSTAIN, None, "insufficient_evidence")
```

注意：这里故意没有输出 `0.93 probability`。

线上需要的是可审计 reason code，而不是难解释的单一概率。

---

## 14. 跨源匹配如何变成 O(N)

如果仍采用 pairwise：

```text
雷小安 N1
腕表之家 N2
奢当家 N3
```

理论 pair 数：

```text
N1*N2 + N1*N3 + N2*N3
```

百万级不可直接算。

Reference Entity Linking 后，每条记录只做一次：

```text
Offer -> (brand_id, canonical_reference)
```

复杂度接近：

```text
O(N * extraction_cost)
```

最后用数据库索引：

```sql
CREATE UNIQUE INDEX ux_reference_entity
ON reference_entity(brand_id, canonical_reference);

CREATE INDEX ix_offer_ref
ON normalized_offer(brand_id, canonical_reference)
WHERE link_status = 'MATCHED';
```

跨源查询：

```sql
SELECT brand_id,
       canonical_reference,
       array_agg(offer_id) AS offers
FROM normalized_offer
WHERE link_status = 'MATCHED'
GROUP BY brand_id, canonical_reference;
```

不需要构建千万/百亿级 pair 表。

---

## 15. 100 万～1000 万规模的工程实现

这个规模不需要一开始就做复杂微服务。

### 15.1 MVP 推荐栈

```text
Python
PostgreSQL
对象存储（raw payload / image）
批处理 worker
OCR worker
版本化 YAML rules
```

离线第一次全量：

```text
分片读取 -> normalize -> extract -> validate -> upsert
```

增量：

```text
source crawler
   ↓
new/changed offer queue
   ↓
reference pipeline
   ↓
upsert normalized_offer
```

当吞吐量需要提升再换：

```text
Kafka / Pulsar
+ Spark / Flink / Ray
```

不要因为目标是千万级，就第一天把业务逻辑拆成十几个服务。

### 15.2 计算缓存

以下内容都可缓存：

- `(brand_id, raw_reference) -> canonical_reference`；
- regex compiled rules；
- OCR image hash -> OCR result；
- registry lookup；
- brand aliases；
- role-classifier result（按 title hash）。

### 15.3 版本化

每次结果必须记录：

```text
brand_rules_version
reference_extractor_version
role_classifier_version
ocr_version
registry_snapshot_version
```

否则规则升级后出现错链，无法追查是哪版逻辑导致。

---

## 16. 黄金标签如何用才最值

Spec 允许标注几百对。

不建议全部拿去训练普通 pairwise matcher。

建议拆成 3 个标注集。

### 16.1 Set A：Reference Extraction Gold

标注：

```text
title / description
正确 reference span
reference role
```

重点覆盖：

- 中文/英文混排；
- 点号/横杠/空格变体；
- OCR 错字；
- 多 reference；
- 型号 + 机芯号；
- 型号 + 库存号。

### 16.2 Set B：Hard Negative Role Gold

专门标：

```text
适用某型号
兼容某型号
盒证
表带
零件
比较文章式标题
回收/求购描述
```

这些样本对 precision 的价值远高于随机负例。

### 16.3 Set C：Final Link Audit Gold

每个品牌抽取最终 `MATCHED` 结果做人审，估计：

```text
precision_brand
precision_channel
precision_rule_version
```

只有达到目标 precision 的 brand/rule 才开自动放行。

---

## 17. 评测指标：不要以 F1 为主

当前业务应把指标排序改成：

```text
1. Auto-match Precision
2. False Positive Count
3. Conflict Detection Recall
4. Auto-match Coverage
5. Reference Extraction Recall
```

核心 dashboard：

```text
自动匹配数
人工抽检数
误匹配数
按品牌 precision
按来源 precision
按 evidence policy precision
ABSTAIN 比例
CONFLICT 比例
新增未知 reference 数
```

### 17.1 建议上线门槛

例如：

```text
自动 MATCH precision >= 99.9% 才允许某规则上线
```

如果样本量不足以证明这一点，则继续 abstain。

对于 Spec 的“绝不能误匹配”，工程上应接受：

```text
Coverage 30% + Precision 99.99%
```

优于：

```text
Coverage 90% + Precision 98%
```

---

## 18. 必须专门建设的 Hard-Negative 测试集

至少包含以下 case：

### 18.1 同系列相邻 reference

```text
126610LN vs 116610LN
```

字符高度相似但必须不同实体。

### 18.2 reference prefix 相同

```text
15500ST...
15510ST...
```

不能做模糊等值。

### 18.3 accessory mentions

```text
“适用 126610LN 表带”
```

不能链接 Rolex 126610LN。

### 18.4 多 reference title

```text
“126610LN / 116610LN 通用...”
```

自动冲突或 accessory。

### 18.5 movement number

```text
“Cal. 3235”
```

不能当 Rolex reference。

### 18.6 serial / inventory id

平台号看起来像字母数字串，但必须识别为来源内部 ID。

### 18.7 OCR confusion

```text
0 / O
1 / I / l
5 / S
8 / B
```

OCR 推断不能直接自动纠错后匹配，除非还有独立强证据。

---

## 19. 人工审核界面应该展示什么

审核员不应该只看到：

```text
模型分数：0.94
```

应该看到证据：

```text
品牌：Rolex
候选 reference：126610LN

来源字段：126610LN
Title 命中：126610LN
OCR 命中：126610LN
Registry：TRUSTED

上下文：
“劳力士潜航者 126610LN 全套...”

冲突：无
商品角色：WATCH

决策原因：trusted_strong_evidence
```

如果是危险 case：

```text
Title：适用 Rolex 126610LN 橡胶表带
候选：126610LN
role：COMPATIBLE_TARGET_REFERENCE
product_role：STRAP_BRACELET

=> ABSTAIN / NOT OWN REFERENCE
```

这比“AI 说不是同一商品”更适合运营落地。

---

## 20. Reference Registry 的增量治理

每天增量会不断出现新 reference。

建议状态机：

```text
UNKNOWN
   │
   ├─ 单弱证据 ─────────> PROVISIONAL
   │
   ├─ 多独立证据一致 ───> CANDIDATE_TRUSTED
   │
   └─ 人工确认 ─────────> TRUSTED

TRUSTED
   │
   └─ 新证据强冲突 ─────> QUARANTINED
```

新 reference 不要因为第一次出现就立即自动用于跨源聚合。

可以设置：

```text
至少两个独立来源支持
或
一个可信结构化来源 + title 一致
或
人工审核
```

才晋级为 `TRUSTED`。

---

## 21. 对论文 Web Verification 的现代化替代

论文用搜索引擎验证：

```text
candidate code
   ↓ search web
结果是否与 manufacturer 共现
```

当前建议替换成多层 deterministic evidence：

```text
Level 1  Internal trusted reference registry
Level 2  Cross-source independent agreement
Level 3  Trusted external catalog / official source
Level 4  OCR evidence
Level 5  Search / LLM discovery only
```

其中 Level 5 只用于发现，不用于自动放行。

这保留了论文“外部知识验证编号归属”的核心思想，同时避免实时搜索引擎的不稳定和不可审计问题。

---

## 22. 是否还需要 Pairwise Matcher

需要，但只放在边缘路径。

### 22.1 不需要 matcher 的主路径

```text
A -> Rolex / 126610LN / TRUSTED
B -> Rolex / 126610LN / TRUSTED
```

直接同实体。

### 22.2 可以用 matcher 的疑难路径

```text
A reference 缺失
B 已知 reference
```

可用：

- title embedding；
- BM25；
- image embedding；
- small cross encoder；

只做 candidate retrieval：

```text
A 可能是 {126610LN, 116610LN, 126610LV}
```

然后仍需要 reference evidence 才能自动链接。

如果没有证据：

```text
ABSTAIN
```

而不是让 matcher 猜一个最像的。

---

## 23. 第一阶段可以在 1 个服务里完成

建议最小代码结构：

```text
luxury_reference_linker/
  adapters/
    leixiaoan.py
    xcar_watch.py
    shedangjia.py

  brands/
    resolver.py
    rules/
      rolex.yaml
      omega.yaml
      cartier.yaml
      ap.yaml
      patek.yaml

  extraction/
    features.py
    candidate_generator.py
    regex_extractor.py
    ocr_extractor.py

  role/
    product_role.py
    reference_role.py

  canonicalization/
    canonicalizer.py

  registry/
    repository.py
    bootstrap.py
    governance.py

  decision/
    evidence.py
    conflict.py
    gate.py

  jobs/
    full_backfill.py
    incremental.py
    registry_refresh.py

  evaluation/
    gold.py
    precision_report.py
    hard_negative_suite.py
```

不必马上做复杂微服务。

当吞吐出现瓶颈，再把 OCR、批处理、registry lookup 拆出去。

---

## 24. 推荐 MVP 执行顺序

### Phase 1：数据审计

先对三个来源各抽样 500～1000 条：

- reference 独立字段覆盖率；
- title 包含 reference 比例；
- 品牌字段质量；
- accessory 比例；
- 多 reference 比例；
- 内部 SKU 形态。

输出字段可靠性矩阵。

### Phase 2：Top 品牌规则

先覆盖数据量最大的 5～10 个腕表品牌。

每个品牌：

- reference regex；
- canonicalization；
- movement / serial 排除规则；
- 50 个 hard negative case。

### Phase 3：Reference Registry Seed

从强证据构建 trusted registry。

### Phase 4：只开放最安全自动 MATCH

例如第一版仅：

```text
WATCH
+ canonical brand
+ explicit ref
+ title 同 reference
+ registry hit
+ no conflict
```

覆盖率可能不高，但 precision 应非常高。

### Phase 5：加入 OCR 与 role classifier

逐步扩大 coverage。

### Phase 6：才考虑 embedding / LLM 辅助

只用于：

- candidate discovery；
- unknown reference triage；
- 人工审核排序；
- rule suggestion。

不改变 hard gate。

---

## 25. 与论文原架构的对应关系

| 论文模块 | 当前系统对应模块 | 是否直接沿用 |
|---|---|---|
| Manufacturer Cleaning | Brand Resolver | 是，强化为全局 brand_id |
| Product Categorization | Product Role Classifier | 是，但目标改为 false-positive 防护 |
| Feature Removal | 非 reference 数字/单位/机芯号清洗 | 是 |
| Token Filtering | 品牌专有 token / pattern ranking | 是，作为候选层 |
| Product Code Regex | Brand-specific Reference Extractor | 是，重点增强 |
| Web Verification | Registry + Cross-source Evidence | 思想沿用，实现替换 |
| Category SVM Matcher | 可选疑难候选 matcher | 不作为最终决策 |
| Manufacturer+Category Blocking | Brand+Reference Registry Lookup | 简化 |
| Pairwise Match | Offer -> Reference Entity Linking | 重构 |

---

## 26. 最关键的设计结论

这篇论文对当前项目最大的价值，不是“它有一个 SVM，可以拿来做商品匹配”，而是它把 product code 从普通文本相似度中提升成一个需要：

```text
抽取
→ 角色判断
→ 归属验证
→ 独立结构化匹配
```

的核心实体属性。

结合当前 Spec，我建议进一步把这个思想推到底：

> **不要做“商品对商品”的主匹配系统，而做“商品到 Reference Entity”的链接系统。**

最终自动匹配规则保持成一句话：

```text
只有高置信度链接到同一 (brand_id, canonical_reference) 的记录，才认为相同。
```

任何一条记录如果：

- reference 缺失；
- reference 只有弱证据；
- 出现多个 reference；
- reference 只出现在兼容/附件语境；
- 图片 OCR 和文本冲突；
- 品牌与 reference registry 冲突；

则：

```text
ABSTAIN
```

这会牺牲 recall，但与需求“precision 优先到极致，允许漏匹配”完全一致。

如果要从本次分析直接选一个最先实现的模块，我会优先做：

> **Brand-aware Reference Candidate Extractor + Reference Role Validator + Trusted Reference Registry + Hard Decision Gate**

而不是先训练通用 pairwise matcher。

这四个模块一旦跑通，就已经能对三源中相当一部分明确 reference 的商品做高精度跨源合并，并且整个判定过程可解释、可审计、可版本化、可回滚，也天然适合后续持续增量。