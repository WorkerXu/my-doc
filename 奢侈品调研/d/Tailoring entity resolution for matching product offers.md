# Tailoring entity resolution for matching product offers

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **Tailoring entity resolution for matching product offers**（Hanna Köpcke, Andreas Thor, Stefan Thomas, Erhard Rahm，EDBT 2012）进行深入分析。

- 论文 PDF：<https://dbs.uni-leipzig.de/files/research/publications/2012-3/pdf/edbt2012_final.pdf>
- 论文页面：<https://dbs.uni-leipzig.de/node/10584>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

这篇论文对当前需求最有价值的不是它最后的 SVM matcher，而是它提出了一个非常接近当前真实瓶颈的思想：

> **先从标题/描述中抽取“manufacturer-specific product code”，再把这个强标识作为匹配依据，而不是直接拿整段商品标题做相似度。**

当前 Spec 已经把业务定义钉死：

> **“同一个商品” = 同一个 reference number / 型号；precision 极端优先，允许漏匹配。**

因此，当前系统根本不应该以“pairwise 商品相似度分类器”为核心，而应该以 **Reference Resolution（reference 解析）** 为核心：

```text
原始商品记录
  -> 品牌规范化
  -> reference 候选抽取
  -> reference 角色判定（是不是当前商品自己的型号）
  -> 保守 canonicalization
  -> 强证据验证 / 冲突否决
  -> (brand_id, canonical_reference) 精确键
  -> 跨源实体组
  -> 一致性审计
```

论文原始方案不能直接照搬，因为它在真实数据上的 product-code 抽取平均 precision 约 **79%**。对于普通商品匹配这可能有价值，但对当前“绝不能误匹配”的需求完全不够。

因此本方案的关键改造是：

> **论文的 product-code extractor 只负责“生成候选证据”，绝不直接负责“确认 reference”；只有通过品牌约束、角色判定、来源可信度、局部上下文、known-reference registry 和冲突检查后的 reference，才能进入 VERIFIED。**

最终建议直接落地一套 **Reference Evidence Pipeline**。匹配层不做模糊判断，只做：

```text
same canonical brand
AND same VERIFIED canonical reference
AND no trusted conflicting reference
=> MATCH

otherwise
=> UNKNOWN / CONFLICT
```

这比训练一个“同商品/不同商品”二分类模型更贴合业务定义，也更容易审计、回滚和持续增量。

---

## 1. 为什么选这篇论文

当前三源数据与论文问题高度同构：

1. 商品来自不同平台，标题、描述、字段结构差异大；
2. 关键型号往往不是独立字段，而是埋在标题或描述中；
3. 大量相似但不同的商品，整段文本相似度很容易误判；
4. 可能出现多个“看起来像型号”的字符串；
5. 配件、表带、盒证、说明书等记录可能在标题中写“适用于某型号”，但这个型号并不是当前售卖商品自己的 reference；
6. 数据量百万到千万，不能做全量笛卡尔积；
7. precision 比 recall 重要得多。

论文里一个非常重要的观察是：

> accessory 商品标题可能包含“它所适配的主商品”的 product code，因此“标题中出现了型号”并不等于“这个型号属于当前商品”。

这点在腕表场景尤其危险。例如以下都是典型误匹配来源：

```text
“适配 XXXXX 表带”
“for XXXXX bracelet”
“XXXXX 原装盒”
“XXXXX 说明书”
“适用于 XXXXX 的表扣”
```

如果只做正则抽取 + exact match，就会把配件错误挂到腕表 reference 上。

所以真正的任务不是：

```text
extract_reference(text) -> string
```

而是：

```text
extract_reference_candidates(text)
    -> classify_reference_role(candidate, context)
    -> verify_reference(candidate, brand, provenance)
    -> accept / abstain / conflict
```

这个“候选 + 角色 + 验证”的拆分，是本轮最值得落地的结论。

---

## 2. 原论文的技术实现

## 2.1 整体架构

论文把商品匹配分成三阶段：

```text
Pre-processing
    ├── manufacturer cleaning
    ├── product categorization
    └── product code extraction

Training
    ├── 构造 match / non-match 训练对
    ├── title / description 多种字符串 matcher
    ├── product-code matcher
    └── SVM 学习组合策略

Application
    ├── manufacturer + category blocking
    ├── 计算 matcher features
    └── SVM 输出 match / non-match
```

其中论文真正新颖、且与当前需求最相关的是 **product code extraction**。

## 2.2 Product Code Extraction 算法

原论文算法可以简化为：

```text
getProductCode(offer):
    offer = removeFeatures(offer)
    tokens = tokenize(offer)
    tokens = removeFrequentTokens(tokens)
    candidates = generateCandidates(tokens)   # 最多连续 3 token
    candidates = filterCandidates(regexes)
    code = webVerification(candidates)
    return code
```

### 第一步：先去掉高频但不是型号的结构化特征

论文先通过正则去掉诸如：

- 尺寸；
- 重量；
- 电压；
- 容量；
- 颜色等。

原因是这类字符串经常同时包含数字、字母和符号，非常像 product code。

迁移到腕表领域，需要扩充为：

```text
尺寸：40mm / 41 mm / 36毫米
年份：2023 / 2024
成色：95新 / 99新
价格：¥ / RMB / 元
机芯：caliber / calibre / cal.
材质：18K / 750 / steel 等
序列号提示词：serial / SN / 编号
店铺 SKU / 平台货号
```

注意：不能仅靠字符串形状判断 reference。

### 第二步：tokenize + 去掉 stopword / 高频 token

论文不是简单做全局词频，而是使用 manufacturer-aware frequency。

它定义：

```text
N(t, m) = manufacturer m 的商品中包含 token t 的 offer 数
N(t)    = 所有 manufacturer 中包含 token t 的 offer 数
```

只保留：

```text
N(t, m) / N(t) > threshold
```

论文实验阈值为 50%。

直觉是：

> 真正的 manufacturer-specific product code 往往强烈依附某个 manufacturer；“for”“black”“new”“camera”等通用词会跨 manufacturer 大量出现。

迁移到当前系统，可以把 manufacturer 替换为 canonical `brand_id`，形成 **brand-affinity feature**：

```text
brand_affinity(candidate, brand)
  = count(candidate, brand) / count(candidate, all_brands)
```

但在当前 precision-first 设计中，这个值只能作为候选过滤特征，不能作为自动匹配的充分条件。

### 第三步：生成最多 3-token 的候选

论文允许 product code 跨多个 token，例如：

```text
HF S10
```

因此会生成连续 1~3 token 的 n-gram，并使用正则保留“像 code”的形式，比如：

- 字母 + 数字；
- 特定品牌常见 pattern；
- 某些连字符结构。

这点对腕表 reference 很有用，因为抓取来源可能把同一个 reference 写成：

```text
AB-1234
AB 1234
AB1234
AB / 1234
```

但这里有一个风险：

> **不能全局把所有空格、斜杠、点号、连字符都删除后比较。**

某些品牌的后缀、分隔符、材质码、表带码可能参与 reference 语义。全局“只保留字母数字”会制造 false positive。

因此必须使用 **brand-specific canonicalization rule**，并且规则版本化。

### 第四步：Web Verification

论文最有启发的一步，是它不把“像型号的字符串”直接当成产品码，而是再利用 Web 搜索结果做验证。

对候选 code `c` 发搜索查询，查看搜索结果是否与当前 manufacturer 高度共现：

```text
candidate = HL-XF51
manufacturer = Hahnel

搜索结果大量出现 Hahnel
=> candidate 更可能是 Hahnel 自己的 product code
```

而如果：

```text
candidate = NP-FF51
manufacturer = Hahnel

搜索结果主要出现 Sony
=> 这是配件标题里“被适配产品”的 code
=> 不应把 NP-FF51 当成当前 Hahnel 商品自己的 code
```

这其实已经隐含了一个非常重要的概念：

> **型号不仅需要 extraction，还需要 ownership / role verification。**

当前系统应该把这个思想显式化，而不是继续依赖在线搜索。

---

## 3. 原论文的 Matching 架构

论文不是只靠 product code，它还组合了：

- title TF/IDF；
- title Trigram/Jaccard 类相似度；
- description 相似度；
- product-code equality；
- manufacturer；
- category。

最终使用 SVM 学习 match / non-match。

在线应用时先使用：

```text
same cleaned manufacturer
AND same category
```

做 blocking，再只比较 block 内的商品对。

论文还强调 category-specific matcher 通常比全品类统一模型更好，因为不同品类的数据分布、错误类型、属性重要性不同。

迁移到腕表需求，可以保留这个“局部规则”的思想，但**不建议保留 SVM 作为最终 match 决策器**。

原因很简单：

当前业务不是“语义上同一个商品”，而是已经明确规定：

```text
same product := same reference number
```

既然业务 identity key 已经明确，就没有必要让一个黑盒模型决定最终 identity。

模型应该解决的是：

```text
“这个字符串到底是不是当前商品自己的 reference？”
```

而不是：

```text
“这两条商品记录看起来是不是同一个商品？”
```

这是架构方向上的核心区别。

---

## 4. 论文实验结果对当前需求的真正含义

论文在 102,182 条真实商品 offer、71 个品类上做实验。

### 4.1 Product code 的覆盖率差异很大

论文报告：

- 非配件类商品平均约 85% 能抽到 product code；
- 配件类平均只有约 34%。

这说明：

> 不应该把“抽不到 reference”视为异常，更不能为了 recall 强行猜一个 reference。

当前系统天然应该支持：

```text
VERIFIED
UNKNOWN
CONFLICT
```

而不是只有：

```text
MATCH / NO_MATCH
```

### 4.2 Web Verification 很重要，但仍远远不够安全

论文中 web verification 能显著改善抽取效果，但最终 product-code extraction 的平均 precision 仍约为 79%。

这对于当前需求意味着：

```text
79% precision
=> 每 100 个自动确认 reference 里可能约 21 个错误
=> 完全不能直接用于实体合并
```

所以本方案绝不能复制：

```text
regex candidate
  -> web verify
  -> accept code
```

而必须升级成：

```text
candidate
  -> role
  -> provenance
  -> brand-specific normalization
  -> trusted reference registry
  -> contradiction check
  -> calibrated accept policy
  -> VERIFIED / UNKNOWN / CONFLICT
```

### 4.3 Product code 对相似变体区分非常有效

论文发现 product code matcher 能明显改善商品匹配，尤其是非配件产品。

这与腕表高度一致：

```text
同品牌
同系列
同尺寸
同材质
视觉几乎一样
```

仍然可能是不同 reference。

因此当前系统必须坚持：

> **语义相似、图像相似、系列相同都不能越权替代 reference。**

### 4.4 “看似唯一”的其他 ID 也可能不可靠

论文还观察到 UPC 存在：

- 同商品不同 UPC；
- 不同商品相同 UPC。

对当前系统的启示是：

> 平台自己的 `item_id / sku / goods_id / spu / product_id` 只能作为 source-local identity，绝不能跨源直接当 global identity。

跨源 global identity 仍应该由：

```text
brand_id + canonical_reference
```

定义。

---

# 5. 当前需求应该如何改造论文方案

## 5.1 核心原则：Reference 是实体，不只是字段

不要把 reference 只存成商品表中的一个字符串字段：

```text
product.reference = "XXXXX"
```

应该显式建模：

```text
Product Record
    |
    | 产生多个 observations
    v
Reference Evidence
    |
    | resolve
    v
Reference Entity
```

一个商品记录可能同时出现：

- 结构化 reference 字段；
- 标题里的 reference；
- 描述里的 reference；
- 图片 OCR 里的 reference；
- 被适配商品的 reference；
- 机芯编号；
- 序列号；
- 平台 SKU。

这些证据不能提前压成一个字符串。

只有证据解析完成后，才能建立：

```text
record -> VERIFIED Reference Entity
```

---

## 5.2 推荐的三值状态

### VERIFIED

表示有足够强、可审计的证据确认：

```text
candidate 是当前商品自己的 reference
```

### UNKNOWN

表示：

- 没找到 reference；
- 找到候选但证据不够；
- 只有低可信 OCR；
- 上下文不清楚；
- 多个候选无法消歧。

`UNKNOWN` 不能进入自动匹配。

### CONFLICT

表示强证据互相矛盾，例如：

```text
结构化 reference = R1
标题高置信 OWN_REFERENCE = R2
R1 != R2
```

这种记录要直接进入隔离区，不参与自动 entity grouping。

---

# 6. 可直接落地的 Reference Evidence Pipeline

## 6.1 Stage A：Source Adapter

三个来源先各自做 adapter，统一成最小 schema：

```text
record_id
source
source_item_id
url
raw_brand
raw_title
raw_description
raw_reference_field
category
images[]
raw_json
updated_at
```

原则：

> 永远保留 raw，不要只保留清洗后的值。

因为后续每次修改 extractor / normalizer，都需要可重放。

---

## 6.2 Stage B：Brand Canonicalization

建立：

```text
brand_alias
-----------
alias
brand_id
canonical_name
confidence
source
```

例如一个品牌可能存在：

```text
中文名
英文名
缩写
大小写变体
空格变体
历史别名
```

最终所有 reference evidence 必须绑定一个 `brand_id`。

为什么 reference 一定要带 brand：

```text
reference 本质上是 manufacturer / brand namespace 内的 identifier
```

所以全局 key 不能只是：

```text
canonical_reference
```

而应该至少是：

```text
(brand_id, canonical_reference)
```

品牌不确定时，不自动匹配。

---

## 6.3 Stage C：Reference Candidate Extraction

不要写一个大正则，而是输出多个候选和 provenance。

```text
reference_candidate
-------------------
record_id
candidate_id
raw_value
normalized_surface
origin              # STRUCTURED_FIELD / TITLE / DESCRIPTION / OCR
span_start
span_end
context_before
context_after
pattern_id
brand_id
extractor_version
```

候选来源按可信度大致排序：

```text
1. 平台明确 reference / model 字段
2. source-specific JSON 中明确的型号字段
3. 标题中显式 “型号 / reference / ref” 附近
4. 标题普通 pattern candidate
5. 描述 candidate
6. 图片 OCR candidate
```

这只是 provenance prior，不等于最终确认。

### Candidate generation 建议

使用两层：

```text
Layer 1: brand-specific regex
Layer 2: generic code-like n-gram generator
```

generic generator 借鉴论文：

- 1~3 个连续 token；
- 至少包含数字；
- 可包含字母；
- 可包含有限集合分隔符；
- 先排除尺寸、年份、价格、序列号提示、机芯提示等。

品牌规则优先于通用规则。

---

# 7. 最关键的新模块：Reference Role Classification

论文通过“候选是否和 manufacturer 在 Web 上共现”间接判断 ownership。

当前系统应该显式建立角色：

```text
OWN_REFERENCE
COMPATIBLE_REFERENCE
ACCESSORY_TARGET_REFERENCE
PLATFORM_SKU
SHOP_SKU
SERIAL_NUMBER
CALIBER
DIMENSION
YEAR
UNKNOWN_CODE
```

只有：

```text
OWN_REFERENCE
```

才有资格进入后续 VERIFIED。

## 7.1 第一版先用规则，不急着上大模型

高 precision 场景下，第一版推荐 rules-first：

### 强正向上下文

```text
型号
reference
reference no
ref.
ref no
款号
```

### 强负向上下文

```text
适用
适配
兼容
for
compatible
表带
表扣
盒
包装盒
说明书
配件
```

### 其他重要 feature

```text
candidate 在标题中的位置
candidate 左右各 N token
是否与 canonical brand 靠近
同一记录中候选数量
是否与 structured field 一致
是否匹配 brand-specific pattern
candidate 的 brand affinity
来源平台
商品 category
```

## 7.2 第二版再训练轻量 role classifier

如果有几百条人工标注，可以标注：

```text
(candidate, local context) -> role
```

优先建议：

- Logistic Regression；
- LightGBM / XGBoost；
- 小型文本 encoder + 线性头。

不建议一开始让大 LLM 直接输出最终 reference，因为：

- 不稳定；
- 很难做强约束；
- 可能自由生成不存在的 reference；
- 难以保证长期增量的一致性；
- 成本高。

LLM 可以用于：

```text
UNKNOWN case 的离线建议
规则生成辅助
人工 review explanation
```

但不能直接成为 auto-merge authority。

---

# 8. Conservative Canonicalization

这是整个系统最容易制造“批量 false positive”的地方。

错误做法：

```python
key = re.sub(r'[^A-Z0-9]', '', raw.upper())
```

这种“删除所有符号”的全局规则风险极高。

推荐两层值：

```text
normalized_surface
canonical_reference
```

## 8.1 normalized_surface

只做无语义风险的操作：

```text
Unicode NFKC
trim
多空格 -> 单空格
英文字母 uppercase
Unicode dash -> ASCII dash
全角标点 -> 半角对应标点
```

## 8.2 canonical_reference

只允许通过：

```text
brand-specific rule
```

产生。

例如：

```text
brand = B1
rule_version = B1_REF_V3
raw = "AB - 1234"
canonical = "AB-1234"
```

如果不能确认空格/符号是否无语义，则：

```text
abstain
```

而不是激进归一化。

每次 normalizer 变更都必须 version 化：

```text
normalizer_version
```

并跑完整回归集后才能 promotion。

---

# 9. 替代论文 Web Verification：本地 Known Reference Registry

原论文在 2012 年用搜索引擎做实时验证很聪明，但不适合作为当前生产系统的核心：

- 外部搜索结果不稳定；
- 查询成本和延迟不可控；
- 搜索排序会变化；
- 无法保证可重放；
- 搜索结果本身可能包含大量二手平台噪声。

建议建立本地：

```text
known_reference
---------------
brand_id
canonical_reference
status              # VERIFIED / CANDIDATE / RETIRED / QUARANTINED
pattern_id
first_seen_at
last_seen_at
trusted_source_count
independent_evidence_count
verification_method
verification_note
normalizer_version
```

### Registry seed 的安全来源

可以从以下强证据逐步构建：

```text
1. 平台明确 reference 字段 + 人工确认
2. 多个独立来源都出现 OWN_REFERENCE，且无冲突
3. 官方/可信 catalog 的离线导入
4. 人工 review 通过
```

注意：

> “多个网页都出现同一个字符串”并不足以证明它是当前商品自己的 reference。

必须先过 role / ownership。

### Registry 的用途

```text
candidate 已在 registry VERIFIED
+ 当前 brand 一致
+ role = OWN_REFERENCE
=> evidence 强度明显提升
```

Registry 是验证器，不是召回器。

---

# 10. 最终自动 Match Policy

## 10.1 不再做 pairwise fuzzy match

一旦每条 record 有 VERIFIED reference，跨源匹配就是数据库等值 join：

```text
entity_key = (brand_id, canonical_reference)
```

因此百万到千万级不需要：

```text
O(N²) pair comparison
```

而是：

```text
O(N) extraction
+ hash / B-tree index
+ group by entity_key
```

这比 embedding blocking + pair classifier 更简单、更可解释，也更符合业务定义。

## 10.2 推荐的放行规则

### Tier A：最安全

```text
same brand
AND both have trusted structured reference
AND canonical references equal
AND neither has trusted conflicting evidence
=> AUTO_MATCH
```

### Tier B：结构化字段 + 标题抽取

```text
same brand
AND A.structured_reference == B.verified_title_reference
AND B.role == OWN_REFERENCE
AND candidate exists in known_reference registry
AND no conflict
=> AUTO_MATCH
```

### Tier C：双方都来自非结构化文本

初期建议：

```text
MANUAL / UNKNOWN
```

只有积累足够黄金标签、brand-specific extractor 成熟、离线评估达到目标后，才逐品牌开放：

```text
same brand
AND same canonical reference
AND both role == OWN_REFERENCE
AND both registry verified
AND no conflict
AND rule_version approved
=> AUTO_MATCH
```

### 永不自动放行

```text
brand unknown
multiple OWN_REFERENCE candidates
trusted sources disagree
reference role = COMPATIBLE / TARGET
only image visual similarity
only embedding similarity
only title semantic similarity
only same series
only same model name without reference
```

---

# 11. Conflict-first 设计

在 precision-first 系统里，**负证据比正相似度更重要**。

建议每条 record 都执行 conflict detector：

```text
if structured_ref != verified_title_ref:
    CONFLICT

if two high-trust OWN_REFERENCE candidates disagree:
    CONFLICT

if brand(candidate) conflicts with record brand:
    CONFLICT

if candidate appears only in accessory-target context:
    do not verify

if OCR reference conflicts with structured reference:
    keep OCR as conflict evidence, do not overwrite
```

自动匹配必须要求：

```text
verified positive evidence
AND no trusted contradiction
```

而不是：

```text
positive_score > threshold
```

---

# 12. 图片应该怎么用

Spec 说图片可用，但业务 identity 又明确等于 reference。

因此图片不应该做：

```text
image_similarity > 0.95
=> same product
```

因为同系列不同 reference 的外观可能极近。

图片建议只承担两类任务：

## 12.1 OCR 型号证据

从：

- 表背；
- 吊牌；
- 保卡；
- 盒标；
- 标签；
- 说明书封面

提取 code candidate。

OCR 输出仍走同一条：

```text
candidate
-> role
-> canonicalize
-> verify
```

## 12.2 冲突否决

图片模型可以判断：

```text
候选记录在视觉上明显不属于同一大类
```

这可以作为 `CONFLICT` 或 review signal。

但图片不能覆盖 reference hard rule。

---

# 13. 数据模型建议

## 13.1 product_record

```sql
CREATE TABLE product_record (
    record_id           BIGINT PRIMARY KEY,
    source              TEXT NOT NULL,
    source_item_id      TEXT NOT NULL,
    url                 TEXT,
    raw_brand           TEXT,
    brand_id            BIGINT,
    raw_title           TEXT,
    raw_description     TEXT,
    raw_reference_field TEXT,
    category            TEXT,
    raw_json             JSONB,
    updated_at          TIMESTAMPTZ NOT NULL
);
```

## 13.2 reference_evidence

```sql
CREATE TABLE reference_evidence (
    evidence_id          BIGSERIAL PRIMARY KEY,
    record_id            BIGINT NOT NULL,
    brand_id             BIGINT,
    raw_value            TEXT NOT NULL,
    normalized_surface   TEXT NOT NULL,
    canonical_reference  TEXT,
    origin               TEXT NOT NULL,
    role                 TEXT NOT NULL,
    role_score           DOUBLE PRECISION,
    pattern_id           TEXT,
    context_before       TEXT,
    context_after        TEXT,
    extractor_version    TEXT NOT NULL,
    normalizer_version   TEXT,
    registry_status      TEXT,
    decision             TEXT NOT NULL,
    conflict_reason      TEXT,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`decision`：

```text
VERIFIED
UNKNOWN
REJECTED
CONFLICT
```

## 13.3 record_reference_link

一条 record 最终只能有 0 或 1 个 active VERIFIED reference：

```sql
CREATE TABLE record_reference_link (
    record_id            BIGINT PRIMARY KEY,
    brand_id             BIGINT NOT NULL,
    canonical_reference  TEXT NOT NULL,
    evidence_id          BIGINT NOT NULL,
    confidence_tier      TEXT NOT NULL,
    policy_version       TEXT NOT NULL,
    verified_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_record_ref_entity
ON record_reference_link (brand_id, canonical_reference);
```

实体组直接：

```sql
SELECT
    brand_id,
    canonical_reference,
    array_agg(record_id)
FROM record_reference_link
GROUP BY brand_id, canonical_reference;
```

不需要保存海量 pairwise match edges。

---

# 14. 服务架构建议

为了 100 万~1000 万历史数据 + 持续增量，不建议一开始堆过重的分布式系统。

可以采用：

```text
                ┌────────────────────┐
                │ Raw source storage │
                │ JSON / Parquet     │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Source Adapters    │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Brand Resolver     │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Candidate Extractor│
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Role Classifier    │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Canonicalizer      │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Registry Verifier  │
                └─────────┬──────────┘
                          │
                          v
                ┌────────────────────┐
                │ Conflict Guard     │
                └──────┬───────┬─────┘
                       │       │
               VERIFIED│       │UNKNOWN/CONFLICT
                       v       v
                ┌──────────┐  ┌────────────┐
                │Entity Key│  │Review Queue│
                └────┬─────┘  └────────────┘
                     │
                     v
          (brand_id, canonical_reference)
```

### 技术栈建议

#### 历史 backfill

```text
Parquet + Polars
```

原因：

- 1000 万行对列式扫描完全合理；
- reference extraction 主要是字符串转换；
- 可以并行按 source / brand 分区；
- 不需要先上 Spark。

数据继续增长到更大规模时再迁移 Spark/Flink。

#### 在线增量

```text
Python worker / FastAPI service
+ PostgreSQL
```

如果现有抓取已经有 Kafka，则接 Kafka；没有的话不要为了这个项目单独引入。

#### 状态与证据

```text
PostgreSQL
```

保存：

- reference registry；
- evidence；
- rule version；
- match decision；
- audit trail；
- manual review。

### 为什么不用向量库作为主存储

因为最终 key 是精确 reference，不是 embedding。

向量召回可以用于：

```text
UNKNOWN case 的候选参考
人工 review
规则发现
```

但不应该参与自动 identity key。

---

# 15. Incremental Matching 流程

每来一条增量商品：

```python
def resolve_record(record):
    brand = resolve_brand(record)
    if not brand.verified:
        return UNKNOWN("brand_unknown")

    candidates = extract_candidates(record, brand)
    evidences = []

    for c in candidates:
        role = classify_role(c, record, brand)
        normalized = normalize_surface(c.raw)
        canonical = canonicalize_by_brand(normalized, brand)

        evidences.append(
            verify_evidence(
                record=record,
                candidate=c,
                role=role,
                canonical=canonical,
                brand=brand,
            )
        )

    conflict = detect_conflict(evidences)
    if conflict:
        return CONFLICT(conflict)

    verified = [e for e in evidences if e.decision == "VERIFIED"]

    if len(verified) != 1:
        return UNKNOWN("reference_not_uniquely_verified")

    ref = verified[0]
    return VERIFIED(
        entity_key=(brand.id, ref.canonical_reference),
        evidence_id=ref.id,
    )
```

真正的“匹配”阶段只有：

```python
def find_entity(resolution):
    if resolution.status != "VERIFIED":
        return None

    return db.query(
        """
        SELECT *
        FROM record_reference_link
        WHERE brand_id = %s
          AND canonical_reference = %s
        """,
        resolution.brand_id,
        resolution.canonical_reference,
    )
```

这就是 current Spec 最适合的架构：

> 把复杂度放在“reference 是否可信”的解析上，而不是放在“两个商品是否相似”的 pairwise 比较上。

---

# 16. 如何使用“几百对黄金标签”

几百对标签不能浪费在随机正负样本上。

随机 negative 太简单：

```text
不同品牌
完全不同系列
标题差异巨大
```

模型学不到最危险的边界。

应专门构造 hard cases：

## 16.1 Hard Negative

```text
同品牌 + 同系列 + 相邻 reference
同品牌 + 同款名 + 不同 reference
同 reference-like token 但一个是配件 target
平台 SKU 与真实 reference 长得很像
structured field 与 title 候选冲突
OCR O/0、I/1、S/5 混淆
同系列不同尺寸/材质对应不同 reference
```

## 16.2 Hard Positive

```text
同 reference 不同分隔符
中英文标题差异巨大
标题缺失但 structured field 有 reference
一边结构化、一边标题抽取
OCR 与结构化字段一致
```

## 16.3 标签应该拆成两层

不要只标 pair：

```text
record A == record B ?
```

还应尽量顺手标：

```text
candidate 是不是 OWN_REFERENCE？
正确 canonical_reference 是什么？
```

这样同一批人工成本能同时训练：

- extractor validation；
- role classifier；
- canonicalization regression；
- final match policy。

---

# 17. Precision-first 评估体系

不要只看 overall F1。

至少拆成：

```text
1. Candidate extraction precision / recall
2. OWN_REFERENCE role precision
3. Canonicalization collision rate
4. VERIFIED reference precision
5. Final auto-match precision
6. Coverage / abstain rate
7. Conflict detection rate
```

其中最重要的是：

```text
Final auto-match precision
```

### 推荐 release gate

每一个品牌规则版本上线前：

```text
- hard-negative regression 必须 0 false positive
- structured-vs-title conflict 测试必须全部阻断
- accessory-target 测试必须全部拒识
- canonicalization collision 测试必须 0 collision
- shadow 数据人工抽查通过
```

重点不是追求一个“99.x% overall F1”，而是保证：

> 自动放行的 Tier 只覆盖我们真正证明过安全的区域。

不安全区域全部 abstain。

---

# 18. 规则版本化与可回滚

这是生产系统必须有、论文没有展开的部分。

每个 evidence 保存：

```text
extractor_version
normalizer_version
role_model_version
registry_version / registry snapshot
policy_version
```

例如：

```text
ROLEX_EXTRACTOR_V4
OMEGA_NORMALIZER_V2
ROLE_MODEL_V3
MATCH_POLICY_V5
```

任何规则升级都先：

```text
old_version vs new_version diff
```

重点检查：

```text
新版本新增了哪些 VERIFIED？
哪些 entity_key 被合并？
是否出现两个历史 entity 被合并成一个？
是否出现 reference collision？
```

如果一个 normalizer 版本把两个原来不同 reference 合并到同一个 canonical key，默认应直接阻止 promotion。

---

# 19. 与当前已有几类调研的组合方式

当前目录里已经有 DeepBlocker、SKU classifier、Ameli、多模态/置信度以及 TransClean 等方向。本篇更适合作为它们的“主干层”。

建议组合关系：

```text
                         ┌──────────────────┐
                         │ DeepBlocker 等   │
                         │ 仅用于 UNKNOWN   │
                         │ 候选发现         │
                         └────────┬─────────┘
                                  │
                                  v
Raw Record -> Reference Evidence Pipeline -> VERIFIED entity key
                    │                    │
                    │                    └-> TransClean-style consistency audit
                    │
                    ├-> SKU / code role classifier
                    │
                    └-> Ameli / OCR / image evidence
```

关键优先级是：

```text
Reference Evidence > fuzzy matching
Conflict veto > similarity score
Abstain > 猜测
```

DeepBlocker / embedding 的价值主要在：

```text
帮 UNKNOWN 记录找到“值得人工看”的候选
```

而不是让其自动越过 reference 规则。

---

# 20. 可直接实施的 MVP

## Phase 0：数据审计（1 个规则版本）

对三个来源分别统计：

```text
reference 独立字段覆盖率
brand 覆盖率
title 中 code-like token 数量分布
一条记录多 candidate 的比例
structured ref 与 title candidate 冲突率
配件/盒证/表带类比例
```

输出按品牌的 risk profile。

## Phase 1：只做最高 precision Tier

上线：

```text
brand canonicalizer
structured reference adapter
conservative normalizer
reference evidence table
entity_key exact join
conflict logging
```

只自动匹配：

```text
两边都有可信 structured reference
```

先建立最安全 baseline。

## Phase 2：标题 reference extraction

实现：

```text
brand-specific regex
通用 1~3 token candidate generator
feature remover
role rules
known-reference registry
```

标题抽取先 shadow，不直接 auto-match。

## Phase 3：按品牌开放 Tier B

对黄金标签 + 历史 hard-negative regression 达标的品牌，开放：

```text
structured ref <-> verified title ref
```

## Phase 4：OCR 和人工回流

增加图片 OCR：

```text
OCR candidate -> same evidence pipeline
```

人工 review 的结果回流：

```text
registry
role labels
brand rule tests
```

## Phase 5：UNKNOWN 候选发现

最后才加入：

```text
embedding / multimodal / DeepBlocker
```

用途仅为：

```text
review candidate retrieval
```

默认不 auto-merge。

---

# 21. 建议的目录与代码结构

```text
reference_resolution/
├── adapters/
│   ├── leixiaoan.py
│   ├── xxxxx_watch.py
│   └── shedangjia.py
├── brand/
│   ├── aliases.yaml
│   └── resolver.py
├── extraction/
│   ├── generic_candidate.py
│   ├── feature_remover.py
│   ├── rolex.py
│   ├── omega.py
│   └── ...
├── role/
│   ├── rules.py
│   ├── features.py
│   └── model.py
├── normalization/
│   ├── surface.py
│   ├── brand_rules/
│   └── registry.py
├── verification/
│   ├── evidence_policy.py
│   ├── conflict.py
│   └── known_reference.py
├── matching/
│   └── entity_key.py
├── evaluation/
│   ├── hard_negative.jsonl
│   ├── hard_positive.jsonl
│   └── regression.py
└── jobs/
    ├── backfill.py
    └── incremental.py
```

所有品牌规则必须同时提交 regression cases。

例如：

```yaml
- name: accessory_target_must_not_be_own_ref
  brand: B1
  title: "原装表带 适配 B1 AB-1234"
  expected:
    candidate: "AB-1234"
    role: "ACCESSORY_TARGET_REFERENCE"
    decision: "REJECTED"
```

这样每次规则改动都能阻止 precision 回退。

---

# 22. 一个更安全的 Match Decision 数据结构

最终不要只存：

```text
A matches B
```

建议存 decision explanation：

```json
{
  "record_id": 123,
  "status": "VERIFIED",
  "entity_key": {
    "brand_id": 17,
    "canonical_reference": "AB-1234"
  },
  "policy_version": "MATCH_POLICY_V3",
  "evidence": [
    {
      "origin": "STRUCTURED_FIELD",
      "raw": "AB 1234",
      "canonical": "AB-1234",
      "role": "OWN_REFERENCE",
      "registry": "VERIFIED"
    }
  ],
  "conflicts": []
}
```

这样出现误匹配时可以直接回答：

```text
为什么合并？
由哪条 evidence 触发？
用了哪个 normalizer？
用了哪个 rule version？
```

这对 precision-first 系统非常重要。

---

# 23. 关键失败模式清单

必须专门测试：

## 23.1 配件引用主商品 reference

```text
“适配 R1 的表带”
```

风险：把表带合并进 R1 腕表。

解决：role = ACCESSORY_TARGET_REFERENCE。

## 23.2 一个标题包含多个 reference

```text
“适配 R1 / R2 / R3”
```

解决：多个 target refs，全拒识。

## 23.3 structured field 写错

不能迷信结构化字段。

如果：

```text
structured = R1
high-trust title OWN = R2
```

必须 CONFLICT，而不是 structured 覆盖 title。

## 23.4 过度 canonicalization

```text
R1-A
R1A
```

如果品牌规则没有证明它们等价，就不能合并。

## 23.5 OCR 字符混淆

```text
O / 0
I / 1
S / 5
B / 8
```

OCR candidate 默认低于结构化和文本原始字符串。

不能自动做模糊字符修复后放行。

## 23.6 品牌别名错误

reference namespace 依赖 brand。

brand 错一次可能把大量 reference 放进错误 namespace。

品牌 unresolved 时必须 abstain。

## 23.7 平台 SKU 冒充 reference

一些平台内部编码也满足：

```text
[A-Z]+[0-9]+
```

必须结合字段名、上下文、跨品牌频率和 source-specific 规则分类。

## 23.8 同系列名称被误当 reference

系列名可以做候选辅助，但不能作为 global identity。

---

# 24. 这篇论文哪些部分应该保留，哪些必须放弃

## 应该保留

### 1. 预处理优先于 matcher

先把脏文本中的强标识抽出来。

### 2. manufacturer / brand-aware processing

reference 是品牌 namespace 内标识。

### 3. pattern-based candidate generation

高 precision 场景下，brand-specific pattern 很有价值。

### 4. accessory-aware ownership verification

这是腕表场景必须有的安全机制。

### 5. category / domain-specific strategy

不要指望一个全品牌统一大模型解决所有 reference pattern。

## 必须改造或放弃

### 1. Web 搜索在线验证

替换为：

```text
local known-reference registry + offline enrichment
```

### 2. 79% precision 的 candidate 直接作为 code

必须拆成：

```text
candidate -> evidence -> VERIFIED
```

### 3. SVM 作为最终同商品判定

当前业务 definition 已经是 reference equality，因此最终判定应是 deterministic exact key。

### 4. 二值 match / non-match

改成：

```text
VERIFIED / UNKNOWN / CONFLICT
```

### 5. 追求 overall F1

改成：

```text
auto-match precision first
coverage second
```

---

# 25. 最终推荐架构

综合论文思想和当前 Spec，我建议最终系统采用：

```text
                    ┌──────────────────────┐
                    │  雷小安 / 腕表之家   │
                    │  / 奢当家 Raw Data   │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │   Source Adapter     │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │ Canonical Brand ID   │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │ Reference Candidate  │
                    │ Extraction           │
                    │ field/title/desc/OCR │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │ Reference Role       │
                    │ OWN/TARGET/SKU/...   │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │ Brand-specific       │
                    │ Canonicalization     │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │ Known Ref Registry   │
                    │ + Evidence Policy    │
                    └──────────┬───────────┘
                               │
                               v
                    ┌──────────────────────┐
                    │ Conflict Guard       │
                    └───────┬───────┬──────┘
                            │       │
                     VERIFIED      UNKNOWN / CONFLICT
                            │       │
                            v       v
             ┌──────────────────┐  Review / abstain
             │ Exact Entity Key │
             │ brand + ref      │
             └────────┬─────────┘
                      │
                      v
             ┌──────────────────┐
             │ Cross-source     │
             │ Entity Group     │
             └────────┬─────────┘
                      │
                      v
             ┌──────────────────┐
             │ Consistency Audit│
             └──────────────────┘
```

最终最重要的系统不变量：

```text
Invariant 1:
任何 AUTO_MATCH 都必须能追溯到 VERIFIED reference evidence。

Invariant 2:
任何 trusted reference 冲突都必须阻断 AUTO_MATCH。

Invariant 3:
图片/embedding/LLM 不得单独创建 entity identity。

Invariant 4:
canonicalization 规则必须 brand-specific 且 versioned。

Invariant 5:
UNKNOWN 永远不因为“看起来很像”而被自动升级为 MATCH。
```

---

# 26. 最终结论

这篇 2012 年的论文虽然模型部分已经不新，但它抓住了当前需求中一个比“大模型 matcher”更基础的问题：

> **商品匹配的关键不是先判断两段脏文本有多相似，而是先把 manufacturer-specific identifier 从脏文本中可靠地解析出来。**

对当前腕表/二奢场景，可以把论文中的 `product code` 直接类比为 `reference number`，但必须做更严格的工程升级：

```text
论文：
pattern extraction
-> web verification
-> product code
-> SVM matcher

当前推荐：
pattern extraction
-> reference candidate
-> role / ownership classification
-> brand-specific canonicalization
-> known-reference registry
-> conflict veto
-> VERIFIED reference
-> exact entity key
-> multi-source entity group
```

最值得直接落地的一句话是：

> **不要做“商品相似度匹配系统”，要做“Reference Evidence Resolution System”。**

一旦 `reference` 被高精度解析并验证，后面的千万级跨源匹配反而非常简单：

```text
GROUP BY brand_id, canonical_reference
```

真正需要投入研发和黄金标签的地方，是：

1. reference candidate extraction；
2. candidate role / ownership；
3. conservative canonicalization；
4. conflict detection；
5. per-brand release gate；
6. human review feedback loop。

这种架构天然符合当前 Spec 的目标：

> **宁可 UNKNOWN，绝不靠模糊相似度把两个不同 reference 的商品合并。**
