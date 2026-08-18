# product-matcher：把通用商品加权匹配器改造成“Reference Gate”的三源腕表实体匹配系统

- 分析人：b
- 调研项目：product-matcher
- 项目地址：https://github.com/Zarenk/product-matcher
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么本次选择 product-matcher

本次执行前先检查了 `奢侈品调研/b` 的已有结果，已经分析过 Ameli、AnyMatch、Confidence Classifiers、Conformal Selective Prediction、DeepBlocker、Ditto、GraLMatch、TransClean、pyJedAI、catalog-forge、parts-distributor-sku-classifier 等项目/论文，当前目录中没有 `product-matcher.md`，因此本次选择它继续分析。

当前 Spec 的约束是：

1. 数据来自雷小安、腕表之家、奢当家三个来源；
2. 总规模约 100 万～1000 万，并持续增量；
3. “同一个商品”被严格定义为 **同一 reference number / 型号**；
4. reference 可能在结构化字段，也可能埋在标题里，图片也可用；
5. **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
6. 可以人工标注几百对黄金样本。

product-matcher 很适合拿来做这次分析，因为它不是纯论文，而是一个很小、很容易读透的 TypeScript 商品匹配核心库，README 说明它来自一个 12 家商店、约 8k listings 的价格比较平台。项目把匹配过程明确拆成：

```text
title normalization
  -> spec canonicalization
  -> identifier exact match
  -> model-number match
  -> blocking
  -> fuzzy/spec scoring
  -> weighted composite score
  -> auto_match / review / new_product
```

其模块边界和当前需求非常接近，但它的默认决策逻辑是为普通电商“尽量匹配”设计的，**不能原样用于“绝不能误匹配”的腕表 reference 场景**。

本次最重要的结论是：

> **可以直接参考 product-matcher 的模块拆分、候选生成和可解释 score breakdown，但必须把最终身份判定从“加权相似度达到阈值”改成不可被其它特征补偿的 `Reference Gate`：只有 `canonical brand + verified canonical reference` 严格一致且无冲突时才允许自动合并。**

图片、标题相似度、系列、尺寸、材质、价格、模型分数都只能用于“找 reference、发现冲突、排序人工复核”，不能拥有自动合并权。

---

## 2. product-matcher 的技术实现与架构

项目核心源码集中在 `src/`：

```text
src/
  text-normalizer.ts
  spec-normalizer.ts
  identifier-matcher.ts
  model-number-matcher.ts
  blocking.ts
  fuzzy-matcher.ts
  weights.ts
  composite-matcher.ts
  index.ts
  *.spec.ts
```

高层入口是 `src/index.ts` 的 `matchProduct()`。

### 2.1 text-normalizer：先把标题变成适合模糊比较的文本

`normalizeProductTitle()` 依次执行：

```text
Unicode NFKD
-> 去重音符号
-> lowercase
-> / \ , ; ( ) [ ] - 等分隔符替换为空格
-> 缩写展开
-> compound token 拆分
-> 单位规范化
-> stop words 删除
-> whitespace collapse
```

例如：

```text
RTX3060 -> RTX 3060
i712700H -> i7 12700H
16 GB -> 16GB
```

这对于笔记本标题模糊匹配很合理，因为它想提高不同商家标题的文本可比性。

但是对腕表 reference，**绝不能直接复用这个 normalizer**。原因是 reference 是 identifier，不是自然语言。对 identifier 做“去符号、拆 token、模糊归一”会把本来需要区分的串压得过于相似。

正确做法应是将两种 normalization 完全拆开：

```text
normalize_title_for_retrieval()
normalize_reference_for_identity()
```

前者可以激进，后者必须保守且可追溯。

### 2.2 identifier-matcher：UPC/EAN/ASIN/MPN 任一精确相等就直接 auto match

`src/identifier-matcher.ts` 的实现非常直接：

```text
for product in products:
  for key in [upc, ean, asin, mpn]:
    if lower(trim(listing[key])) == lower(trim(product[key])):
      return product
```

`src/index.ts` 又进一步规定：

```text
exact identifier hit
-> score = 1
-> decision = auto_match
-> reason = identifier
-> skip all fuzzy scoring
```

这是项目最值得借鉴、同时也是当前需求里最需要重写的一点。

值得借鉴的是：

> **强 identifier 应该优先于模糊模型，并能够 short-circuit。**

不能照搬的是：

> **它默认“字段名叫 MPN”就足够可信，且精确相同就直接合并。**

对三源二奢数据来说，这不安全，因为一个“像 reference 的字符串”至少有以下角色：

```text
manufacturer reference
platform SKU
shop SKU
internal listing id
series/model family
accessory-compatible model
OCR noise
seller-entered typo
```

因此当前项目必须先做 identifier role classification 和 provenance validation，再谈 exact match。

### 2.3 model-number-matcher：brand gated，但仍允许 base model 产生高分

`src/model-number-matcher.ts` 的规则是：

```text
exact full model      = 1.00
exact base model      = 0.85
base model contained  = 0.80
```

如果 listing 与 product 都有 brand，则品牌不同直接跳过该候选。

这个设计在普通商品匹配里有价值：完整型号最强，产品线/基础型号稍弱。

但腕表需求对“同一个商品”的定义就是同一 reference，因此：

```text
Rolex Submariner
Rolex 126610
Rolex 126610LN
Rolex 126610LV
```

这些层级不能被压进一个连续的“model similarity score”。

尤其 `126610LN` 与 `126610LV` 可能只差两个字符，视觉上又极其接近，但按需求必须判为不同商品。

因此需要将 product-matcher 的 `modelBase` / `modelFull` 语义改成：

```text
series/model_family     -> 只用于 blocking / review
reference_candidate     -> 进入严格校验
canonical_reference     -> 才拥有 identity 权限
```

### 2.4 blocking：三层候选缩减，最多保留 50 条

`src/blocking.ts` 使用三层策略：

```text
Level 1: exact brand + category
Level 2: same brand + model prefix in normalized name
Fallback: normalized title token overlap
MAX_CANDIDATES = 50
```

只有前两层合计不足 5 个候选时才启用 token overlap fallback。

README 说明生产系统把 Level 1/2 下推到 SQL，内存版本主要用于测试和小目录。

这个思路非常适合复用到“疑难候选人工审核”，但不适合作为有 reference 时的主链路。

如果已经拿到 canonical reference，1000 万规模也不应该做 fuzzy blocking，而应直接：

```sql
SELECT entity_id
FROM product_entity
WHERE brand_id = ?
  AND canonical_reference = ?;
```

或走等价的复合索引/hash lookup。

另外，product-matcher 当前内存 blocking 达到 50 条以后直接停止，候选没有严格 ranking；在大目录上如果把它当召回主链路，真实候选可能因为遍历顺序被截掉。

### 2.5 fuzzy-matcher：标题 token_set_ratio + 规格 overlap

标题相似使用 `fuzzball.token_set_ratio`；规格相似对 CPU、RAM、storage、GPU、screen size 等字段加权。

这个设计说明它的目标是让不同弱证据“合起来把分数抬高”。

对当前腕表需求，这套能力仍然有用，但用途应该反过来：

```text
不是：弱证据累加 -> 允许 match
而是：弱证据发现 -> 提供候选 / 找冲突 / 排 review 优先级
```

比如图片非常像、标题相似、系列相同，但 reference 不同，应当让这些相似度帮助我们构造 hard negative，而不是帮助自动合并。

### 2.6 composite-matcher：当前设计允许“关键冲突被其它维度补偿”

`src/composite-matcher.ts` 对所有分数做加权平均：

```text
score = sum(score_i * weight_i) / sum(weight_i)
```

`weights.ts` 按品类提供权重，例如 laptops：

```text
model    0.35
brand    0.15
specs    0.25
title    0.20
category 0.05
```

`matchProduct()` 默认：

```text
score >= 0.85 -> auto_match
score >= 0.60 -> review
else          -> new_product
```

这是当前 Spec 下最危险的结构性问题：

> **加权平均意味着一个真正决定身份的字段即使缺失/冲突，也可能被标题、规格、图片等其它相似度补偿。**

而业务定义明确要求：reference 不同就是不同商品。因此 reference 不是一个“高权重 feature”，而是一个 **hard gate / constraint**。

还有一个值得注意的实现细节：`src/index.ts` 中 `categoryScore` 的计算是：

```ts
categoryScore: listing.categorySlug ? 1 : 0
```

它没有比较 candidate 的 category，只要 listing 自己带 category 就给 1 分。因此这个维度实际上可能给每个候选无条件加正分。普通项目里影响不一定大，但也说明“最终决策不能依赖多个可补偿 soft score”对于 precision-first 场景尤为重要。

---

## 3. product-matcher 原样用于当前需求会出现哪些 False Positive

### 3.1 平台 SKU 被误当成 MPN/reference

假设两个来源都存在：

```text
source A: sku = 126610LN
source B: mpn = 126610LN
```

如果 source A 的字段映射错误，把平台 sku 填入 `mpn`，product-matcher 会直接 identifier short-circuit，score=1 自动合并。

当前需求需要的是：

```text
raw value
-> identifier role
-> source field provenance
-> brand validation
-> reference syntax validation
-> canonical reference
-> identity gate
```

### 3.2 同系列不同 reference 被模糊分数拉成同款

例如：

```text
Rolex Submariner Date 126610LN
Rolex Submariner Date 126610LV
```

品牌相同、标题绝大多数 token 相同、外观很接近、尺寸和机芯可能也相同。

任何 title/spec/image composite 都有把它们推向高分的风险。

正确规则必须是：

```text
canonical_reference A != canonical_reference B
=> hard reject
```

其它 feature 不允许 override。

### 3.3 reference 缺失时被“高相似”自动合并

当前项目可以在没有 exact identifier 的情况下靠 model/title/spec 达到 `auto_match`。

当前需求不应允许：

```text
reference missing + title/image highly similar -> auto_match
```

最多只能：

```text
reference missing + highly similar -> REVIEW_CANDIDATE
```

### 3.4 低分不等于 new product

product-matcher 把低于 review threshold 的结果叫 `new_product`。

对持续增量二奢数据，这个语义过强：

```text
低分
```

可能只是：

```text
reference 没抽到
图片不可用
标题太短
来源字段缺失
OCR 失败
新品牌规则没覆盖
```

并不能证明它真的是“一个从未存在的新商品实体”。

因此应把决策状态改成：

```text
AUTO_MATCH
UNRESOLVED
REVIEW
CONFLICT
```

是否新建实体应由“可信 reference 是否已建立 canonical entity”决定，而不是相似度低不低。

### 3.5 缺少证据血缘，难以审计错误来源

product-matcher 返回 `breakdown`，这是优点，但对于本项目还不够。

我们需要知道：

```text
reference 来自哪个字段？
来自标题的哪段 span？
来自哪张图片的 OCR？
是哪版 extractor 抽的？
是否命中官方/reference 字典？
是否存在另一个冲突 reference？
为什么自动放行？
人工是否改过？
```

precision-first 系统不能只有“最终 score”，还必须有 **evidence ledger**。

---

## 4. 建议直接落地：Reference Gate Matcher

建议保留 product-matcher 的“模块化 matcher”思想，但把核心管线改造成：

```text
Raw Listing
  |
  v
Source Adapter
  |
  +--> canonical brand
  +--> structured identifier candidates
  +--> title reference candidates
  +--> image OCR reference candidates
  |
  v
Reference Evidence Ledger
  |
  v
Identifier Role Classifier
  |
  v
Reference Validator / Canonicalizer
  |
  +--> conflict? --------------------> CONFLICT / REVIEW
  |
  +--> no verified reference? ------> UNRESOLVED / REVIEW
  |
  v
REFERENCE GATE
  |
  |  brand exact
  |  canonical reference exact
  |  provenance acceptable
  |  no contradicting evidence
  v
Exact Entity Lookup
  |
  +--> existing key -> AUTO_MATCH
  |
  +--> no key -> create/propose canonical entity
  |
  v
Audit Log + Membership
```

核心原则：

> **soft model 永远在 gate 外面；只有 verified reference 能穿过 gate。**

---

## 5. 数据模型：不要只存 `reference` 一个字符串，要存 Reference Assertion

建议至少建立下面几张表。

### 5.1 raw_listing

```sql
CREATE TABLE raw_listing (
  id                BIGSERIAL PRIMARY KEY,
  source            TEXT NOT NULL,
  source_listing_id TEXT NOT NULL,
  title             TEXT,
  raw_payload       JSONB NOT NULL,
  image_urls        JSONB,
  observed_at       TIMESTAMPTZ NOT NULL,
  content_hash      TEXT NOT NULL,
  UNIQUE(source, source_listing_id, content_hash)
);
```

保留 raw payload 的意义是：以后 extraction / normalizer 升级可以重跑，不会丢原始证据。

### 5.2 reference_assertion

```sql
CREATE TABLE reference_assertion (
  id                 BIGSERIAL PRIMARY KEY,
  listing_id         BIGINT NOT NULL REFERENCES raw_listing(id),
  brand_id           BIGINT,
  raw_value          TEXT NOT NULL,
  normalized_value   TEXT,
  canonical_value    TEXT,
  evidence_type      TEXT NOT NULL,
  evidence_location  TEXT,
  identifier_role    TEXT NOT NULL,
  role_confidence    DOUBLE PRECISION,
  extraction_score   DOUBLE PRECISION,
  validation_status  TEXT NOT NULL,
  extractor_version  TEXT NOT NULL,
  normalizer_version TEXT NOT NULL,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`evidence_type` 建议枚举：

```text
STRUCTURED_REFERENCE
STRUCTURED_UNKNOWN_ID
TITLE_EXACT_SPAN
DESCRIPTION_EXACT_SPAN
IMAGE_OCR
MANUAL
```

`identifier_role` 建议枚举：

```text
MANUFACTURER_REFERENCE
PLATFORM_SKU
SHOP_SKU
INTERNAL_ID
MODEL_FAMILY
ACCESSORY_TARGET_MODEL
UNKNOWN
```

`validation_status`：

```text
VERIFIED
PLAUSIBLE
INVALID
CONFLICTING
```

这样以后发现误匹配时，可以直接追溯是哪类证据造成的。

### 5.3 canonical_reference

```sql
CREATE TABLE canonical_reference (
  id                  BIGSERIAL PRIMARY KEY,
  brand_id            BIGINT NOT NULL,
  canonical_reference TEXT NOT NULL,
  reference_hash      BYTEA NOT NULL,
  status              TEXT NOT NULL,
  validation_source   TEXT,
  created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(brand_id, canonical_reference)
);
```

这里非常关键：

> canonical identity key 不是裸 reference，而是 `(brand_id, canonical_reference)`。

因为不同品牌可能存在相同/近似型号字符串。

### 5.4 product_entity

```sql
CREATE TABLE product_entity (
  id                    BIGSERIAL PRIMARY KEY,
  brand_id              BIGINT NOT NULL,
  canonical_reference_id BIGINT NOT NULL REFERENCES canonical_reference(id),
  status                TEXT NOT NULL,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(brand_id, canonical_reference_id)
);
```

### 5.5 entity_membership

```sql
CREATE TABLE entity_membership (
  listing_id      BIGINT PRIMARY KEY REFERENCES raw_listing(id),
  entity_id       BIGINT NOT NULL REFERENCES product_entity(id),
  decision        TEXT NOT NULL,
  decision_reason TEXT NOT NULL,
  decision_version TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.6 match_audit

```sql
CREATE TABLE match_audit (
  id               BIGSERIAL PRIMARY KEY,
  listing_id       BIGINT NOT NULL,
  entity_id        BIGINT,
  decision         TEXT NOT NULL,
  reason_codes     JSONB NOT NULL,
  evidence_ids     JSONB NOT NULL,
  conflict_ids     JSONB,
  matcher_version  TEXT NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

这张表负责回答“为什么这条记录会被自动合并”。

---

## 6. Reference Normalization：必须与 title normalization 分离

建议新建一个专用 `reference-normalizer`，规则必须遵守“最小变换原则”。

### 6.1 通用层只做确定不会改变语义的操作

例如：

```text
Unicode NFKC
trim
uppercase
统一全角 ASCII
统一明确的 separator 字符编码
```

不要默认做：

```text
edit distance
删除任意字母数字
同形字符猜测
前后缀模糊合并
substring match
```

### 6.2 品牌层采用 whitelist canonicalization

如果某品牌官方资料/历史数据已经证明：

```text
AB-1234
AB1234
AB 1234
```

三种只是展示格式不同，才能加入品牌级 alias rule：

```text
(brand, raw_pattern) -> canonical_reference
```

而不是全局地“把所有连字符都删掉”。

推荐函数签名：

```ts
interface NormalizedReference {
  raw: string;
  strict: string;
  canonical: string | null;
  ruleId: string | null;
  verified: boolean;
}

normalizeReference(brandId, raw): NormalizedReference
```

每次 canonicalization 都返回 `ruleId`，便于审计和回滚。

---

## 7. Reference Gate：最终自动匹配规则

建议把自动放行写成显式布尔规则，不要把它藏进一个 ML score。

伪代码：

```ts
function decide(listing: ListingEvidence): Decision {
  if (!listing.brand?.verified) {
    return unresolved('BRAND_NOT_VERIFIED');
  }

  const refs = listing.references.filter(
    r => r.role === 'MANUFACTURER_REFERENCE' &&
         r.validationStatus === 'VERIFIED'
  );

  if (refs.length === 0) {
    return unresolved('NO_VERIFIED_REFERENCE');
  }

  const uniqueRefs = uniq(refs.map(r => r.canonicalValue));

  if (uniqueRefs.length > 1) {
    return conflict('MULTIPLE_VERIFIED_REFERENCES');
  }

  if (hasStrongContradictoryEvidence(listing)) {
    return conflict('REFERENCE_EVIDENCE_CONFLICT');
  }

  const key = {
    brandId: listing.brand.id,
    reference: uniqueRefs[0],
  };

  const entity = exactEntityLookup(key);

  if (entity) {
    return autoMatch(entity.id, 'BRAND_REFERENCE_EXACT');
  }

  return proposeEntity(key, 'NEW_VERIFIED_REFERENCE');
}
```

这里没有 `titleSimilarity > 0.9`，也没有 `imageSimilarity > 0.95`。

这正是与 product-matcher 最大的架构差异。

---

## 8. 什么条件才允许 `VERIFIED_REFERENCE`

建议把 reference 证据分成不同等级。

### A 级：最强

```text
可信结构化 reference 字段
+ identifier role = manufacturer reference
+ brand 明确
+ 通过品牌 reference 规则/字典验证
```

可以直接进入 gate。

### B 级：标题中抽取

```text
标题 exact span
+ 品牌明确
+ reference pattern 合法
+ 候选在 canonical reference 字典中
```

可以进入 gate。

### C 级：图片 OCR

OCR 很重要，但不应单独拥有身份权。

建议：

```text
IMAGE_OCR reference
+ TITLE/STRUCTURED 中另一个独立证据一致
```

才升级为 verified。

如果只有 OCR：

```text
-> PLAUSIBLE
-> REVIEW
```

### D 级：模型推断/LLM 生成

LLM/VLM 从图文“猜”出的 reference 必须只算候选：

```text
-> PLAUSIBLE
-> 必须回到原始文本 span、OCR 或 reference 字典做 grounding
```

禁止直接 auto match。

---

## 9. 图片应该怎么用：从“正向相似度”改成“证据提取 + 冲突否决”

当前 Spec 明确有图片可用。product-matcher 预留了 `embedScore`，但默认是 0。

腕表场景不建议简单把 CLIP/VLM 图片相似度接进 composite score，因为：

```text
同系列不同 reference 外观可能极像
官方图可能被多个 listing 重复使用
卖家可能上传盒证/配件/手腕照
图片可能来自网络而非实物
```

更安全的图片链路：

```text
image quality / utility classifier
  -> OCR 表背、保卡、吊牌、铭牌
  -> 提取 alphanumeric reference candidate
  -> brand-specific regex
  -> 与结构化字段/标题 reference 交叉验证
  -> 冲突则 REJECT/REVIEW
```

视觉 embedding 仍可使用，但仅用于：

```text
1. reference 缺失记录的候选召回
2. hard negative mining
3. 人工 review 排序
4. 发现“标题说 A，图片极像 B”这种异常
```

不用于最终 identity equality。

---

## 10. 百万到千万级规模：有 reference 时根本不需要 pairwise EM

如果最终身份键是：

```text
brand_id + canonical_reference
```

主链路的复杂度会从 pairwise matching 变成 key lookup。

### 10.1 在线增量

建立索引：

```sql
CREATE UNIQUE INDEX uq_entity_brand_ref
ON canonical_reference(brand_id, canonical_reference);
```

新 listing 处理：

```text
extract verified reference
-> exact indexed lookup
-> O(log N) / hash-like lookup
-> attach entity or propose entity
```

这比对 1000 万记录做向量 top-k 再 cross-encoder 更便宜、更稳定、更可解释。

### 10.2 历史全量 backfill

可以用 Polars / Spark / DuckDB / ClickHouse 中任一方案先做：

```text
source rows
-> canonical brand
-> reference assertions
-> verified canonical reference
-> group by brand_id, canonical_reference
```

真正需要模型的只剩 reference 缺失或冲突的尾部数据。

### 10.3 推荐生产形态

不用一开始就上复杂图数据库。一个足够直接的组合是：

```text
Raw data / image: Object Storage
Canonical/Audit/Review: PostgreSQL
Batch backfill: Polars or Spark
Extraction workers: Python
OCR/VLM: 独立异步 worker
API: FastAPI / NestJS 均可
```

如果现有团队是 TypeScript，也可以直接 fork product-matcher 的模块组织，在 NestJS 中实现 Reference Gate；数据处理量大的历史回填再用 Python/SQL。

---

## 11. 增量架构：必须可重算，而不是一次匹配后永久固化

三源数据会持续变化：

```text
标题被商家修改
reference 字段后来补齐
图片新增
规则更新
OCR 模型升级
品牌 alias 更新
人工修正
```

因此所有关键结果都应该带版本：

```text
extractor_version
normalizer_version
matcher_version
```

推荐增量流程：

```text
1. source adapter 产生 content_hash
2. 新 hash 才进入 extraction
3. 产生 immutable reference_assertion
4. 决策器读取当前有效 assertions
5. 生成 match_audit
6. entity_membership 更新为新 decision
7. 若规则版本变化，只重算受影响 brand/reference/listing
```

不要直接覆盖原始 reference；新版本 assertion 追加写，旧证据保留。

这样出现一次错误规则时，可以知道它影响了哪些记录，并局部回滚。

---

## 12. 三源聚类：不要依赖 pairwise 传递闭包

因为“同一商品”已经定义成同一 reference，最稳的 cluster key 就是：

```text
entity_key = brand_id + canonical_reference
```

这比：

```text
A ~= B
B ~= C
=> A = C
```

安全得多。

三源记录只需 attach 到同一个 canonical entity：

```text
Entity: Rolex | 126610LN
  - 雷小安 listing #...
  - 腕表之家 listing #...
  - 奢当家 listing #...
```

如果某条 listing 同时产生两个 verified reference assertion：

```text
126610LN
126610LV
```

它不是“选分数最高的一个”，而应进入：

```text
CONFLICT
```

并禁止污染任何 entity。

---

## 13. product-matcher 哪些模块可以直接参考，哪些必须重写

### 可以保留/参考

#### `text-normalizer.ts`

保留用于：

```text
标题候选召回
reference extractor 前的文本清洗辅助
人工 review 搜索
```

但不能用于 reference identity normalization。

#### `blocking.ts`

保留“多层候选、逐步放宽”的思想，用于：

```text
NO_VERIFIED_REFERENCE 的人工复核候选
hard negative mining
```

#### `ScoreBreakdown`

保留可解释 breakdown 的思想，但改成 review score，不再决定自动合并。

#### `*.spec.ts`

项目测试都和模块放在一起，这个习惯值得直接照搬。reference rule 尤其需要大量 regression test。

### 必须重写

#### `identifier-matcher.ts`

从：

```text
same string -> match
```

改成：

```text
same canonical string
+ same canonical brand
+ role verified
+ source/provenance accepted
+ no conflict
-> match
```

#### `model-number-matcher.ts`

不能让 base model/substring 拥有 identity 分数。

改成：

```text
model family -> retrieval only
canonical reference -> identity only
```

#### `composite-matcher.ts`

不能作为 auto-match 决策器。

改为：

```text
reviewPriorityScore
```

例如帮助人工优先看“几乎相同但 reference 冲突”的高风险记录。

#### `matchProduct()`

应改成显式 gate，而不是 weighted threshold。

---

## 14. 如果直接 fork product-matcher，建议新增这些文件

```text
src/
  reference-normalizer.ts
  reference-extractor.ts
  identifier-role.ts
  reference-validator.ts
  reference-evidence.ts
  reference-gate.ts
  entity-index.ts
  conflict-detector.ts
  audit.ts
  review-scorer.ts
```

建议核心类型：

```ts
type EvidenceType =
  | 'STRUCTURED_REFERENCE'
  | 'TITLE_EXACT_SPAN'
  | 'IMAGE_OCR'
  | 'MANUAL';

type IdentifierRole =
  | 'MANUFACTURER_REFERENCE'
  | 'PLATFORM_SKU'
  | 'SHOP_SKU'
  | 'MODEL_FAMILY'
  | 'UNKNOWN';

interface ReferenceAssertion {
  rawValue: string;
  canonicalValue: string | null;
  evidenceType: EvidenceType;
  role: IdentifierRole;
  verified: boolean;
  ruleId?: string;
  extractorVersion: string;
}

type Decision =
  | { type: 'AUTO_MATCH'; entityId: string }
  | { type: 'UNRESOLVED'; reason: string }
  | { type: 'REVIEW'; reason: string }
  | { type: 'CONFLICT'; reason: string };
```

Reference Gate：

```ts
export function referenceGate(
  brandId: string | null,
  refs: ReferenceAssertion[],
  lookup: (brandId: string, ref: string) => string | null,
): Decision {
  if (!brandId) {
    return { type: 'UNRESOLVED', reason: 'NO_VERIFIED_BRAND' };
  }

  const verified = refs.filter(
    r => r.verified && r.role === 'MANUFACTURER_REFERENCE' && r.canonicalValue
  );

  const values = [...new Set(verified.map(r => r.canonicalValue!))];

  if (values.length === 0) {
    return { type: 'UNRESOLVED', reason: 'NO_VERIFIED_REFERENCE' };
  }

  if (values.length > 1) {
    return { type: 'CONFLICT', reason: 'MULTIPLE_VERIFIED_REFERENCES' };
  }

  const entityId = lookup(brandId, values[0]);

  if (!entityId) {
    return { type: 'REVIEW', reason: 'VERIFIED_REFERENCE_NOT_IN_ENTITY_INDEX' };
  }

  return { type: 'AUTO_MATCH', entityId };
}
```

生产版可以进一步区分“可信结构化 reference 首次出现”是否允许自动创建 entity；为了极致 precision，初期建议新 reference 先进入 `REVIEW`，人工确认后再建立 canonical entity。

---

## 15. 几百对黄金标签应该怎么花

当前约束允许人工标几百对，建议不要随机抽样，也不要以普通 F1 为唯一目标。

### 15.1 正例

覆盖：

```text
同 reference，不同来源
reference 格式有空格/连字符变化
reference 只在标题
reference 只在结构化字段
图片 OCR 与文本一致
```

### 15.2 最重要的是 hard negatives

重点标：

```text
同品牌 + 同系列 + reference 只差 1 个字符
同品牌 + 同系列 + 不同后缀
外观极像但 reference 不同
标题含“适用于 XXX reference”的配件
平台 SKU 看起来像 reference
来源字段把 SKU 错标为型号
OCR 把 0/O、1/I、5/S 混淆
不同品牌出现相同型号串
```

这些样本才能真实检验“绝不能误匹配”。

### 15.3 指标

主指标应改为：

```text
AUTO_MATCH precision
AUTO_MATCH false-positive count
verified reference extraction precision
conflict detection recall
coverage / abstention rate
```

最重要的 acceptance test：

```text
黄金集 hard negatives 上 AUTO_MATCH FP = 0
```

在这个约束满足以后，再讨论 coverage。

---

## 16. 必须写进回归测试的案例

### Case 1：同 reference，格式差异已被品牌规则确认等价

```text
A: AB-1234
B: AB1234
```

如果 brand-specific alias rule 明确证明等价：

```text
AUTO_MATCH
```

### Case 2：同系列不同后缀

```text
A: 126610LN
B: 126610LV
```

必须：

```text
NO_MATCH / CONFLICT
```

即使 title/image similarity = 0.99 也不能改变结果。

### Case 3：reference 缺失，标题与图片高度相似

必须：

```text
UNRESOLVED / REVIEW
```

不能 AUTO_MATCH。

### Case 4：同 reference 字符串，不同品牌

```text
brand A: 12345
brand B: 12345
```

必须是不同 entity。

### Case 5：平台 SKU 相同，但 manufacturer reference 不同

必须：

```text
CONFLICT / NO_MATCH
```

### Case 6：结构化 reference 与 OCR 冲突

```text
structured: 126610LN
OCR:        126610LV
```

必须：

```text
CONFLICT
```

不能“选 structured 分数更高”后继续自动合并。

### Case 7：LLM/VLM 猜出型号但找不到原文/OCR grounding

必须：

```text
PLAUSIBLE only
UNRESOLVED / REVIEW
```

---

## 17. 建议的上线顺序

### Phase A：先把确定性 reference 主链路跑起来

```text
source adapter
canonical brand
structured/title reference extraction
reference role
strict canonicalization
exact entity key
```

先覆盖最容易、最安全的一部分数据。

### Phase B：加入证据账本和冲突检测

确保每个自动结果都能回答“为什么”。

### Phase C：加入图片 OCR

只解决 reference 缺失和文本不完整，不提升视觉相似度的自动合并权限。

### Phase D：加入 review candidate scoring

这时才复用 product-matcher 的：

```text
title similarity
spec similarity
blocking
weights
```

但它们只负责人工队列排序和 hard-negative mining。

### Phase E：用黄金标签校准覆盖率

在 FP=0 的门槛下，逐步放宽哪些 evidence combination 可以从 REVIEW 升级为 AUTO_MATCH。

---

## 18. 与 product-matcher 相比，最终决策语义应这样变化

product-matcher：

```text
identifier exact -> auto_match
else composite score >= 0.85 -> auto_match
else >= 0.6 -> review
else -> new_product
```

当前项目应改为：

```text
verified brand
AND exactly one verified manufacturer reference
AND canonical reference exact
AND no strong conflict
AND canonical entity exists / has approved creation policy
-> AUTO_MATCH

verified reference conflict
-> CONFLICT

reference insufficient
-> UNRESOLVED

soft evidence highly suggestive
-> REVIEW
```

这不是单纯“把阈值从 0.85 调到 0.99”，而是把决策范式从：

```text
score-based matching
```

换成：

```text
constraint-based identity + soft-evidence review
```

---

## 19. 最终建议

product-matcher 最值得参考的是它清晰的小模块设计：normalizer、identifier matcher、model matcher、blocking、scoring、decision 都能独立测试和替换。对于当前雷小安 × 腕表之家 × 奢当家的需求，可以直接沿用这种工程边界，甚至可以 fork 其 TypeScript 代码快速搭原型。

但不要沿用它的最终决策哲学。

当前需求已经给了一个极强的领域定义：

> **同一个商品 = 同一 reference number。**

因此最优系统不应该努力训练一个“更聪明的同款相似度模型”去覆盖这个定义，而应把大部分工程投入到：

```text
reference 找得准
reference 角色判得准
reference 规范化可审计
冲突能被发现
图片只提供独立证据
canonical key 查询足够快
增量规则可回放
```

最终自动合并的唯一硬钥匙建议是：

```text
verified canonical brand
+
verified canonical reference
```

所有其它信息都只能辅助，不能越权。

这套改造可以把 product-matcher 从一个普通的“多特征相似度商品 matcher”，变成真正符合本 Spec 的 **Reference-First、precision-first、可拒识、可审计、百万到千万级可增量运行的三源腕表实体匹配系统**。
