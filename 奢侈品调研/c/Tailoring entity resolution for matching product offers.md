# Tailoring entity resolution for matching product offers：从 product code 抽取到 precision-first 腕表 reference 实体解析

> 分析人：c  
> 原论文：Hanna Köpcke, Andreas Thor, Stefan Thomas, Erhard Rahm, *Tailoring entity resolution for matching product offers*, EDBT 2012  
> 论文：https://dbs.uni-leipzig.de/files/research/publications/2012-3/pdf/edbt2012_final.pdf  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 核心约束：**同一商品 = 同一 reference number / 型号；precision 极端优先，宁可漏匹配，绝不能误匹配。**

## 1. 为什么选这篇论文

这篇 2012 年论文虽然年代较早，但它研究的问题和当前 Spec 的“真正难点”高度同构：

- 商品数据来自不同商家/来源；
- 字段稀疏，关键标识常常不在独立字段，而埋在 title / description；
- 商品标题很脏，营销词、规格、配件适配对象会混在一起；
- 相邻型号外观和文本非常相似；
- 一个标题里可能出现多个“像型号”的字符串；
- 直接依赖一个看似唯一的标识（论文中的 UPC）也可能因为脏数据产生错误匹配。

论文提出：不要一上来就对整条商品记录做相似度匹配，而应先进行强预处理，从非结构化标题中抽取 manufacturer-specific **product code**，并验证这个 code 是否真的属于当前商品。它甚至专门举了配件商品的例子：

```text
Hahnel HL-XF51 7.2V 680mAh for Sony NP-FF51
```

标题同时出现：

- `HL-XF51`：当前 Hahnel 商品自己的 product code；
- `NP-FF51`：它所适配的 Sony 商品型号。

如果只用“像不像型号”的正则或 NER，两个都可能被抽出来；如果再直接拿抽到的字符串跨源 exact join，就会把配件错误并入 Sony NP-FF51 对应实体。

这恰好对应二奢/腕表的高风险场景：

```text
劳力士原装表带 适配 116500LN / 126500LN
欧米茄海马 210.30.42... 原装盒证
适用于 Cartier WSSA00xx 的第三方表带
Rolex 126610LN 同款展示盒 / 保卡套 / 配件
```

因此，这篇论文对当前项目最大的价值是：

> **reference extraction 不是“找到一个字母数字串”就结束，而必须包含 reference ownership verification：证明该 reference 属于当前售卖商品本体，而不是标题中提到的兼容对象、配件对象、平台 SKU 或其他编号。**

这比再增加一个通用 pairwise matching 模型更符合当前 Spec 的 precision-first 目标。

---

## 2. 原论文的系统架构

论文的总体流程是三阶段：

```text
                   ┌──────────────────────────┐
                   │      Raw product offers  │
                   └────────────┬─────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 1: Pre-processing                                 │
│                                                         │
│  manufacturer cleaning                                 │
│  product categorization                                │
│  product-code extraction + verification                │
└───────────────────────────────┬─────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 2: Training                                       │
│                                                         │
│  build labeled match/non-match pairs                   │
│  per-category matcher features                         │
│  TF/IDF + trigram/Jaccard + product-code matcher       │
│  SVM                                                    │
└───────────────────────────────┬─────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 3: Application                                    │
│                                                         │
│  blocking by cleaned manufacturer + category           │
│  calculate matcher similarities                        │
│  SVM => match / non-match                              │
└─────────────────────────────────────────────────────────┘
```

其中最重要的是预处理层。论文明确认为商品匹配不能只靠 title 的整体字符串相似度，因为真实商品 title 长短差异巨大，配件 title 还可能包含一长串兼容型号。

### 2.1 Manufacturer cleaning

论文先规范 manufacturer：

- 通过字符串相似度聚合同一厂商的不同写法；
- 使用同义词表和厂商字典；
- 如果 manufacturer 字段缺失，则回到 title / description 中用字典找品牌。

这是后续 product code verification 的基础，因为 paper 的验证逻辑依赖“候选 code 与 manufacturer 是否一致”。

对当前项目对应的是：

```text
Rolex / ROLEX / 劳力士 / 劳 / 劳服
Omega / OMEGA / 欧米茄
Patek Philippe / PP / 百达翡丽
Audemars Piguet / AP / 爱彼
```

必须先映射到 canonical brand entity，否则相同 reference 字符串在不同品牌下可能产生碰撞，或者同一品牌的中文/英文写法无法共享规则。

### 2.2 Product categorization

原论文用 Modified Naive Bayes 将 offer 分到 category，并强调 category-specific match strategy 通常优于统一策略。

其核心价值不在朴素贝叶斯本身，而是：**先判断商品属于什么对象，再决定如何解释其中的编号。**

对腕表应该至少区分：

```text
WATCH                 手表本体
STRAP                 表带
BRACELET              钢带/链带
BOX                    表盒
PAPER                  保卡/证书
ACCESSORY              其他配件
SERVICE_PART           零件
UNKNOWN
```

如果 `item_type != WATCH`，标题中出现腕表 reference 时默认应解释为“适配/关联对象 reference”，而不是当前商品 reference。只有在有强证据证明配件自身也存在独立品牌 reference 时，才允许生成 `OWN_REFERENCE`。

### 2.3 Product-code extraction

论文 Algorithm 1 可以整理成：

```text
offer
  │
  ├─ 1. removeFeatures()
  │      去掉尺寸、重量、电压、容量、颜色等常见规格
  │
  ├─ 2. tokenize()
  │      按空格、标点切 token
  │
  ├─ 3. removeFrequentTokens()
  │      去 stop words
  │      去跨 manufacturer 高频 token
  │
  ├─ 4. generateCandidates()
  │      生成最多连续 3 token 的候选串
  │
  ├─ 5. filterCandidates(regex)
  │      只保留具有 product-code 形态的串
  │      例如同时有字母 + 数字
  │
  └─ 6. webVerification()
         去外部搜索候选 code
         检查搜索结果是否与当前 manufacturer 一致
         选通过阈值的 code
```

一个非常值得迁移的细节是论文的 **manufacturer-based token frequency**。

对于 token `t` 和 manufacturer `m`，定义：

```text
N(t,m) = manufacturer m 的商品中包含 token t 的 offer 数
N(t)   = 全部 manufacturer 中包含 token t 的 offer 数
```

只保留：

```text
N(t,m) / N(t) > threshold
```

论文实验 threshold 取 50%。

这相当于用数据自动发现“某 token 是否对某品牌有特异性”。例如 `Submariner` 对 Rolex 很有品牌特异性，但它仍不是 reference；而类似 `116500LN` 这种 token 的特异性会更强。

### 2.4 Web verification

论文真正解决 `HL-XF51` 与 `NP-FF51` 归属冲突的是最后一步：

- 搜索 `HL-XF51`，结果大量和 Hahnel 共现 => 认为是 Hahnel product code；
- 搜索 `NP-FF51`，结果和 Sony 共现而非 Hahnel => 拒绝把它当 Hahnel 当前商品 code。

这实际上不是普通“格式验证”，而是一个非常早期的 **entity-to-identifier ownership verification**。

对于当前项目，可把问题写成：

```text
P(reference r belongs_to brand b and current item x | all evidence)
```

而不是：

```text
P(string r looks_like reference)
```

二者是完全不同的任务。

---

## 3. 原论文实验结果，以及为什么不能直接照搬

论文使用 102,182 条真实电商 offer，覆盖 71 个 category。

几个关键结果：

1. 非配件商品中，平均约 85% 的 offer 能抽出 product code；配件中只有约 34%。
2. web verification 能显著提高 code extraction 质量，部分场景提升可达约 20%。
3. 最终抽出的 product code 平均 precision 约 79%。
4. product code 对非配件商品匹配帮助最大；手机类的匹配 F-measure 最好达到约 73%。
5. 在人工构造的 reference mapping 上，camera 类加入 product code 后 F-measure 可到约 81%。
6. 论文还发现 UPC 不是绝对可靠：同一商品可能多个 UPC，不同商品也可能错误共享 UPC。

### 3.1 79% extraction precision 对当前项目远远不够

这篇论文的平均 product-code precision 约 79%，在普通 recommendation / price-comparison 场景已经有价值，但对当前“绝不能误匹配”的约束完全不可直接使用。

假设有 500 万条商品，哪怕只有 100 万条进入自动 reference extraction：

```text
99.0% precision => 约 10,000 个错误 reference
99.9% precision => 约 1,000 个错误 reference
99.99% precision => 约 100 个错误 reference
```

而错误 reference 一旦成为实体主键，会产生级联污染：

```text
错抽 reference
   ↓
跨源 exact join
   ↓
错误实体簇
   ↓
价格 / 图片 / 卖家 / 历史记录全部并错
   ↓
后续模型甚至会把这个错误簇继续当训练数据
```

因此原论文的“product code 是一个 matcher feature”在当前项目中要改成更严格的设计：

> **只有 reference 被高置信度解析并验证通过后，才生成可参与自动实体合并的 `reference_key`。模型相似度、图片相似度、标题相似度都不能越权创造 MATCH。**

### 3.2 原论文的 SVM 不是当前需求的核心

论文最终把 title/description 的字符串特征和 product-code matcher 输入 SVM，输出 match/non-match。

当前 Spec 已明确“同一个商品 = 同一个 reference number”。因此不应让 SVM 学到：

```text
标题很像 + 图片很像 + reference 不一致
=> 仍然 MATCH
```

对于腕表，同系列不同 reference 往往就是：

- 外观极其相似；
- 标题只有一两个字符差异；
- 市场价格接近；
- 图片 embedding 很近。

所以应反过来：

```text
reference exact equal + reference ownership verified
=> MATCH

reference different
=> NON_MATCH（硬否决）

reference missing / conflicting / uncertain
=> ABSTAIN，不自动匹配
```

图片/文本模型只用于：

- 提高 reference 抽取或 OCR 成功率；
- 发现冲突并否决；
- 给人工复核排序；
- 不能把两个不同 reference “相似地”合并。

---

## 4. 对当前 Spec 的核心改造：Reference Resolver，而不是通用 Matcher

建议将项目的中心模块定义为：

```text
Reference Resolver
```

输入一条商品记录，输出：

```json
{
  "brand_id": "rolex",
  "reference_raw": "126610 LN",
  "reference_canonical": "126610LN",
  "ownership": "OWN_PRODUCT",
  "item_type": "WATCH",
  "confidence_tier": "CERTIFIED",
  "evidence": [
    "structured_field",
    "title_pattern",
    "brand_pattern",
    "cross_source_dictionary"
  ],
  "conflicts": []
}
```

或者明确拒识：

```json
{
  "reference_canonical": null,
  "confidence_tier": "ABSTAIN",
  "reason": "MULTIPLE_CONFLICTING_REFERENCES"
}
```

这样最终实体解析会非常简单：

```text
entity_key = (canonical_brand_id, certified_reference)
```

只有 `confidence_tier = CERTIFIED` 的记录才能自动进入实体簇。

---

## 5. 推荐的生产架构

针对 100 万～1000 万级、持续增量三来源数据，建议采用下面的分层架构：

```text
雷小安 crawler ─┐
腕表之家 crawler ├──> Raw Offer Store
奢当家 crawler ─┘        │
                         ▼
               Normalize / Parse Layer
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
       Brand Resolver  Item Type   Text/OCR
                         │           │
                         └─────┬─────┘
                               ▼
                     Reference Candidate
                         Extraction
                               │
                               ▼
                    Candidate Role / Owner
                         Verification
                               │
             ┌─────────────────┼─────────────────┐
             ▼                 ▼                 ▼
        CERTIFIED          CONFLICT          ABSTAIN
             │
             ▼
      Canonical Reference
             │
             ▼
    Exact Reference Entity Join
             │
             ▼
      Entity / Offer Mapping
             │
             ├──> downstream search / analytics
             └──> review + feedback loop
```

### 5.1 Raw Offer Store

先保存不可变原始记录，不要在抓取后直接覆盖字段：

```sql
raw_offer(
    offer_id            bigint primary key,
    source              varchar,
    source_offer_id     varchar,
    fetched_at          timestamp,
    title_raw           text,
    description_raw     text,
    brand_raw           text,
    reference_raw_field text,
    category_raw        text,
    price_raw           text,
    image_urls          jsonb,
    payload_raw         jsonb,
    payload_hash        char(64)
)
```

`payload_hash` 用来支持增量幂等：同一来源商品内容没变化时无需重复解析。

### 5.2 Brand Resolver

输出 canonical brand：

```sql
brand_alias(
    alias_norm     varchar primary key,
    brand_id       bigint,
    confidence     numeric,
    source         varchar
)
```

品牌解析可以先用字典 + 规则，因为品牌集合相对小。不能解析出唯一品牌时，原则上不允许自动构造 reference entity key。

原因是很多 reference 只在品牌内部唯一，不能假设全球唯一。

### 5.3 Item Type Resolver

这是从原论文“category-specific matching”迁移出的关键防线。

规则优先判断：

```text
WATCH
ACCESSORY / STRAP / BOX / PAPER / PART
UNKNOWN
```

高风险否定词/关系词包括：

```text
适配
兼容
for
compatible with
表带
盒
证书
保卡
配件
零件
替换
replacement
```

如果一条记录被判为配件，则 title 中出现的腕表 reference 默认标记为：

```text
TARGET_PRODUCT_REFERENCE
```

而不是：

```text
OWN_PRODUCT_REFERENCE
```

这一层能够消灭一类非常难靠 pairwise similarity 修复的 false positive。

---

## 6. Reference Candidate Extraction：如何直接落地

建议采用“多通道候选生成”，不要让单一 LLM 直接输出一个最终型号。

### 6.1 通道 A：结构化 reference 字段

如果来源本身有 reference/model 字段：

```text
candidate.source = STRUCTURED_FIELD
```

但仍不能直接信任，需要经过：

- 字符规范化；
- 品牌 pattern 验证；
- 与 title/OCR 冲突检测；
- 判断字段是否其实是平台货号。

### 6.2 通道 B：标题正则 / 词法候选

参考论文 `generateCandidates + filterCandidates`，先大量召回“像 reference”的 token span。

例如：

```python
GENERIC_PATTERNS = [
    r'(?<![A-Z0-9])[A-Z]{1,5}[-./ ]?[0-9]{2,8}[A-Z0-9./-]{0,10}(?![A-Z0-9])',
    r'(?<![A-Z0-9])[0-9]{3,8}[-./][A-Z0-9.-]{1,12}(?![A-Z0-9])',
]
```

但 generic regex 只能用于**候选召回**，不能用于最终认定。

必须再叠加 brand-specific pattern，例如：

```text
Rolex: 某些 reference 常为 5/6 位数字 + 可选字母后缀
Omega: 常见 xxx.xx.xx.xx.xx.xxx 形式
Cartier: 常见 W 开头字母数字串
...
```

pattern 应版本化：

```sql
brand_reference_pattern(
    brand_id,
    pattern_version,
    regex,
    valid_from,
    valid_to,
    enabled
)
```

### 6.3 通道 C：description

description 召回率高但风险更高，因为商品描述更容易包含：

- “适配某某型号”；
- 对比型号；
- 品牌历史型号列表；
- SEO keyword；
- 维修/保养涉及的其他 reference。

因此 description 候选默认 evidence tier 低于 title/structured field。

### 6.4 通道 D：OCR

图片可用时建议 OCR 只做候选生成：

优先 OCR：

- 表背刻字；
- 保卡；
- 吊牌；
- 原厂标签；
- 证书。

OCR 结果必须保留：

```text
image_id
bbox
ocr_text
ocr_confidence
image_type
```

不要只保存最终字符串，否则后续无法解释它到底从哪张图、哪个区域来的。

### 6.5 通道 E：受控 LLM extraction

LLM 可以作为 fallback，但输出必须是候选集合而非最终决策：

```json
{
  "candidates": [
    {
      "text": "126610LN",
      "role": "possible_own_reference",
      "evidence_span": "劳力士黑水鬼126610LN",
      "reason": "..."
    }
  ]
}
```

LLM 不允许直接输出：

```text
entity_id = xxx
```

最终决策必须由可审计 gate 完成。

---

## 7. Reference Canonicalization：只能做“被证明安全”的规范化

这是非常容易制造静默误匹配的地方。

### 7.1 安全的通用规范化

通常可以：

```text
Unicode NFKC
全角 -> 半角
trim
连续空白压缩
英文字母 upper-case
OCR 常见不可见字符清理
```

### 7.2 不要全局无脑删除所有符号

例如：

```text
210.30.42.20.01.001
21030422001001
```

对 Omega 可能可以建立确定性等价规则；但不能因此全局写：

```python
canonical = re.sub(r'[^A-Z0-9]', '', raw)
```

因为 `/`、`.`、`-`、字母后缀有时就是型号边界或变体信息。

### 7.3 品牌级 canonicalization rule

正确做法：

```python
def canonicalize_reference(brand_id, raw):
    s = unicode_nfkc(raw).strip().upper()
    rule = load_brand_rule(brand_id)
    return rule.canonicalize(s)
```

每个 rule 必须配：

- positive examples；
- dangerous near-neighbor negative examples；
- regression tests；
- version。

特别需要把以下 pair 固定加入 hard-negative test：

```text
同系列、只差 1 位数字
同 base reference、不同字母后缀
新旧代 reference
不同尺寸 reference
不同材质 reference
不同地区后缀
```

如果业务定义“reference 必须完全相同”，则任何不能被官方/人工确认等价的变换都不能折叠。

---

## 8. Reference Ownership Verification：当前系统最应该新增的模块

借鉴论文 web verification，但生产实现建议改为多证据 ownership verifier。

对于候选 `(offer x, brand b, reference r)`，生成以下 evidence：

### 8.1 来源字段证据

```text
E_structured_ref
E_title_ref
E_description_ref
E_ocr_ref
```

结构化字段通常权重最高，但如果来源历史上该字段混入 SKU，则要降级。

### 8.2 品牌模式证据

```text
E_brand_pattern = reference 是否符合 brand b 的已知 reference pattern
```

如果 `r` 明显符合另一个品牌而非当前品牌，是强冲突。

### 8.3 上下文 ownership 证据

在候选周围保留窗口，例如 ±30 字符：

```text
劳力士 126610LN 全套
        ^^^^^^^^^
```

对比：

```text
原装表带 适配劳力士 126610LN
               ^^^^^^^^^
```

需要抽取 relation：

```text
OWN_PRODUCT
TARGET_OF_ACCESSORY
COMPATIBLE_WITH
COMPARE_WITH
UNKNOWN
```

这一步可以规则 + 小模型完成，且应该有 `ABSTAIN`。

### 8.4 跨来源共现字典

有三个来源后，可以自己构建比 2012 年“web search”更稳定的验证知识库。

定义：

```sql
reference_brand_stats(
    candidate_ref,
    brand_id,
    source,
    offer_count,
    own_context_count,
    accessory_context_count,
    first_seen,
    last_seen
)
```

计算：

```text
brand_specificity(r,b)
  = own_context_count(r,b) / Σ_b' own_context_count(r,b')
```

如果一个候选长期只在 Rolex 本体商品中出现，可信度会很高。

### 8.5 外部 reference catalog / 搜索验证

如果可获得品牌官网、腕表之家型号库或其他可信 catalog，可将原论文 web verification 改成：

```text
candidate r
   │
   ├─ official / trusted reference catalog exact lookup
   ├─ brand + reference co-occurrence search
   └─ known variant / suffix table lookup
```

这里要注意：

- 外部搜索只能增强/否决，不应单独成为自动 MATCH 依据；
- 搜索结果会变化，必须缓存 query、top results、timestamp；
- 第三方页面可能复制错误数据；
- 生产吞吐下不可能对 1000 万条逐条实时搜索。

因此建议只有 `uncertain but promising` 候选才使用外部验证，并把验证结果回灌到内部 reference dictionary。

---

## 9. 不用“概率大于 0.9 就 MATCH”，而用分层 Gate

最终状态建议不是二分类，而是：

```text
CERTIFIED
PROBABLE
CONFLICT
ABSTAIN
REJECTED
```

其中只有 `CERTIFIED` 可以自动实体合并。

一个可落地的第一版 CERTIFIED gate：

```python
def certify_reference(x):
    if x.brand_id is None:
        return False

    if x.item_type != "WATCH":
        return False

    if x.reference_candidate_count != 1:
        return False

    if not x.brand_pattern_valid:
        return False

    if x.ownership != "OWN_PRODUCT":
        return False

    if x.has_reference_conflict:
        return False

    if x.from_structured_field:
        return x.structured_field_reliability >= SOURCE_CERT_THRESHOLD

    return (
        x.from_title
        and x.cross_source_brand_specificity >= 0.999
        and x.reference_dictionary_verified
    )
```

这段逻辑故意保守。

更重要的是：**第一版千万不要追求 coverage。**

可以先让：

```text
自动匹配覆盖率只有 20%～40%
```

只要这部分接近零 false positive，就已经能建立可信的实体主键和高质量自举数据。之后再逐步扩大 CERTIFIED 集合。

---

## 10. 最终 Entity Resolution：退化成确定性 Join

当一条记录得到：

```text
brand_id = rolex
reference = 126610LN
status = CERTIFIED
```

则实体键：

```text
reference_entity_key = SHA256("rolex|126610LN")
```

建议表：

```sql
reference_entity(
    entity_id             bigint primary key,
    brand_id              bigint not null,
    reference_canonical   varchar not null,
    created_at            timestamp,
    updated_at            timestamp,
    unique(brand_id, reference_canonical)
)
```

offer 映射：

```sql
offer_entity_map(
    offer_id              bigint primary key,
    entity_id             bigint,
    resolution_status     varchar,
    resolver_version      varchar,
    evidence_json         jsonb,
    resolved_at           timestamp
)
```

自动合并逻辑：

```sql
INSERT INTO reference_entity(brand_id, reference_canonical)
VALUES (:brand, :ref)
ON CONFLICT (brand_id, reference_canonical)
DO UPDATE SET updated_at = now()
RETURNING entity_id;
```

这比千万级记录两两 pairwise comparison 简单几个数量级。

理论上如果 reference resolver 做对了，复杂度从候选 pair 的近似 `O(N * k)` 进一步变成基于 key 的 `O(N)` upsert / hash join。

---

## 11. 图片在这个架构里怎么用

当前 Spec 明确“有图片可用”，但 precision-first 下图片不能成为身份主键。

### 11.1 图片适合做 reference OCR

优先级：

```text
保卡 / 吊牌 / 表背刻字 > 表盘整体图
```

流程：

```text
image classifier
    ↓
定位 card / caseback / tag
    ↓
OCR
    ↓
reference candidate
    ↓
brand pattern + ownership verification
```

### 11.2 图片适合做冲突否决

例如文本抽到：

```text
126610LN
```

但 OCR 在保卡读到另一个明确 reference：

```text
124060
```

应产生：

```text
CONFLICT
```

而不是让某个多模态模型“综合概率后仍然放行”。

### 11.3 图片相似不能覆盖 reference 冲突

硬规则：

```text
if certified_ref_A != certified_ref_B:
    NON_MATCH
```

无论 CLIP / DINO / VLM 图片相似度多高都不能推翻。

这是腕表尤其需要坚持的，因为同系列不同 reference 本来就可能视觉上极其接近。

---

## 12. 几百对人工黄金标签应该怎么花

Spec 允许人工标注几百对。不要把几百对平均随机采样后只训练一个 pair classifier；应该全部围绕“会产生 false positive 的边界”使用。

建议标注对象分两类。

### 12.1 Offer-level reference labels

每条记录标：

```json
{
  "brand": "Rolex",
  "item_type": "WATCH",
  "own_reference": "126610LN",
  "mentioned_other_references": [],
  "reference_status": "CERTAIN"
}
```

或：

```json
{
  "item_type": "STRAP",
  "own_reference": null,
  "mentioned_other_references": ["126610LN"],
  "relation": "COMPATIBLE_WITH"
}
```

这比简单 pair label 更能直接训练/评估最关键的 ownership 问题。

### 12.2 Hard-negative pair labels

重点采：

- 同品牌同系列、reference 只差 1 个字符；
- base reference 相同、后缀不同；
- 表带/盒证标题含腕表 reference；
- 两来源 title 高度相似但 reference 不同；
- OCR 易混字符：`0/O`, `1/I`, `5/S`, `8/B`；
- 平台 SKU 与品牌 reference 形态相似；
- structured field 与 title reference 冲突；
- 同图重复转载但 reference 字段不同。

### 12.3 标注配额建议

第一轮 400 条可以：

```text
100  条：明确正样本 / 易样本，验证 pipeline 基本正确
200  条：hard negative / near-reference
50   条：配件 ownership
50   条：OCR / 字段冲突
```

后续 active learning 只送人工看：

```text
最接近 CERTIFIED gate、但缺一个条件的样本
```

这样人工成本会更有效。

---

## 13. Precision-first 的指标设计

不能只报整体 F1。

至少分四层指标：

### 13.1 Reference extraction precision

```text
抽出的 candidate 中，字符串本身是否是真实 reference
```

### 13.2 Ownership precision

```text
抽出的 reference 是否属于当前商品本体
```

这是论文给当前项目最重要的启发。

### 13.3 Certification precision

```text
CERTIFIED 的 reference 是否正确
```

这个指标应极高。

### 13.4 Final merge precision

```text
自动进入同一 entity 的 offer pair 是否真的满足相同 reference
```

这是最终业务指标。

同时单独看 coverage：

```text
auto_resolution_coverage
```

但 coverage 只是第二目标。

建议上线门槛不是“F1 最大”，而是：

```text
Final auto-merge precision >= 99.99%（或样本中 0 FP）
在此约束下最大化 coverage
```

如果黄金集太小，99.99% 无法靠统计直接证明，因此上线时应采用更保守策略：

- hard gate；
- 0 false-positive regression set；
- 人工抽检；
- 新规则先 shadow；
- 新品牌/新来源默认降级到 ABSTAIN。

---

## 14. 增量更新怎么做

三来源持续更新时，建议 pipeline 事件化：

```text
RAW_OFFER_UPSERT
    ↓
NORMALIZE
    ↓
REFERENCE_RESOLVE
    ↓
ENTITY_ATTACH
```

每个阶段记录版本：

```text
normalizer_version
brand_resolver_version
item_type_model_version
reference_extractor_version
reference_rule_version
resolver_gate_version
```

### 14.1 幂等

相同 `payload_hash` 不重复解析。

### 14.2 可重算

规则升级后不要改历史 raw data，只针对受影响 brand/source 回放。

例如 Rolex canonicalization v3 发布：

```sql
select offer_id
from resolved_reference
where brand_id = :rolex
  and rule_version < 3;
```

批量重算。

### 14.3 实体簇不要“不可逆 merge”

即使 precision 很高，也建议实体关系可回溯。

不要只有：

```text
offer.entity_id = 123
```

还要保存：

```text
为什么进 123
用哪个 reference
哪个 resolver 版本
哪组证据
```

这样发现错误规则时，可以有选择地撤销，而不是全库重建。

---

## 15. 数据表建议

### 15.1 候选表

```sql
reference_candidate(
    candidate_id          bigint primary key,
    offer_id              bigint not null,
    raw_text              varchar not null,
    normalized_text       varchar,
    candidate_source      varchar, -- STRUCTURED/TITLE/DESC/OCR/LLM
    source_location       jsonb,   -- span/bbox/image_id
    brand_id              bigint,
    brand_pattern_valid   boolean,
    context_role          varchar,
    extraction_score      numeric,
    extractor_version     varchar,
    created_at            timestamp
)
```

### 15.2 解析结果表

```sql
resolved_reference(
    offer_id                 bigint primary key,
    brand_id                 bigint,
    reference_canonical      varchar,
    resolution_status        varchar, -- CERTIFIED/PROBABLE/CONFLICT/ABSTAIN
    ownership                varchar,
    evidence_json            jsonb,
    conflict_json            jsonb,
    resolver_version         varchar,
    updated_at               timestamp
)
```

### 15.3 Reference dictionary

```sql
reference_dictionary(
    brand_id                 bigint,
    reference_canonical      varchar,
    verification_status      varchar,
    evidence_count           bigint,
    independent_source_count int,
    accessory_context_count  bigint,
    first_seen               timestamp,
    last_seen                timestamp,
    metadata                 jsonb,
    primary key(brand_id, reference_canonical)
)
```

这个 dictionary 是把原论文依赖的“外部 Web 知识”逐渐变成系统自己的可控知识库。

---

## 16. 一个更适合当前需求的 scoring / gate 组合

内部可以计算分数帮助排序，但最终仍用 gate。

例如：

```text
S =
  + 4.0 * structured_reference_field
  + 3.0 * title_exact_candidate
  + 2.5 * trusted_catalog_exact
  + 2.0 * brand_pattern_valid
  + 2.0 * cross_source_independent_support
  + 1.5 * OCR_card_or_caseback_support
  - 6.0 * accessory_target_context
  - 6.0 * conflicting_reference
  - 5.0 * brand_mismatch
  - 4.0 * platform_sku_pattern
```

分数只用于：

```text
人工复核排序
PROBABLE / ABSTAIN 分层
active learning 采样
```

CERTIFIED 仍要求必要条件同时成立，而不是 `S > threshold`。

这样可以避免两个弱正证据叠加“抵消”一个强冲突。

例如：

```text
图片很像 + title 很像 + 价格很像
```

绝不能抵消：

```text
reference 明确不同
```

---

## 17. 对论文方案最关键的现代化改造

可以把原论文每个模块映射为 2026 可直接落地版本：

| 原论文 | 当前项目改造 |
|---|---|
| manufacturer cleaning | canonical brand resolver |
| product categorization | WATCH / ACCESSORY / BOX / PAPER 等 item type resolver |
| regex product-code candidate | structured field + regex + NER/LLM + OCR 多通道候选 |
| manufacturer token frequency | brand/reference 共现统计 + reference dictionary |
| web verification | trusted catalog + 三来源内部知识库 + 缓存外部 search |
| product-code matcher feature | certified reference hard key |
| title/desc TF-IDF / trigram | 仅辅助抽取、review、冲突检测 |
| SVM match classifier | 可选，不具有越权 MATCH 权限 |
| match / non-match | CERTIFIED / PROBABLE / CONFLICT / ABSTAIN / REJECTED |
| category-specific model | brand-specific reference grammar + source-specific reliability |

其中最重要的结构变化是：

```text
论文：
product code 是“帮助模型判断 match”的特征

当前项目：
reference 是“定义 match”的业务主键；
模型只负责把这个主键从脏数据中安全解析出来。
```

---

## 18. 第一版可直接上线的最小实现（MVP）

不需要先训练大模型。

### Phase 1：只做高精度规则，快速建立可信实体层

实现：

1. 三来源 raw offer 统一入库；
2. brand alias dictionary；
3. `WATCH vs ACCESSORY` 高精度规则；
4. structured reference 字段解析；
5. title regex 候选；
6. 10～20 个主要品牌的 reference regex；
7. conservative canonicalization；
8. 如果 reference 唯一、品牌明确、无配件上下文、无冲突 => CERTIFIED；
9. 用 `(brand_id, ref)` exact join。

预期：coverage 不一定高，但 precision 容易做得极高。

### Phase 2：加内部 reference dictionary

从 Phase 1 的 CERTIFIED 数据统计：

```text
brand-reference 共现
source 数
title 上下文
reference grammar
```

用它验证 Phase 1 没覆盖的候选。

### Phase 3：加 OCR

只处理：

```text
文本缺 reference
但有高价值图片
```

不要全图片 OCR，先做 card/caseback/tag 检测，控制成本。

### Phase 4：加 ownership classifier / LLM fallback

只处理：

```text
存在多个候选 reference
或存在“适配/配件/兼容”复杂句式
```

模型必须允许 abstain。

### Phase 5：active learning 扩 coverage

从未通过 CERTIFIED 的高价值候选中抽样标注，逐步扩 brand rule / ownership model。

---

## 19. 推荐的在线判定伪代码

```python
def resolve_offer(offer):
    brand = resolve_brand(offer)
    if not brand.certified:
        return Abstain("BRAND_UNCERTAIN")

    item_type = resolve_item_type(offer)

    candidates = []
    candidates += from_structured_fields(offer, brand)
    candidates += from_title_regex(offer, brand)
    candidates += from_description(offer, brand)

    if not candidates and has_high_value_images(offer):
        candidates += from_ocr(offer, brand)

    candidates = canonicalize_candidates(candidates, brand)
    candidates = classify_identifier_role(candidates, offer, brand)
    candidates = verify_ownership(candidates, offer, brand, item_type)
    candidates = attach_dictionary_evidence(candidates, brand)

    own_refs = [
        c for c in candidates
        if c.role == "BRAND_REFERENCE"
        and c.ownership == "OWN_PRODUCT"
    ]

    if has_conflict(own_refs):
        return Conflict(own_refs)

    unique = unique_canonical_refs(own_refs)
    if len(unique) != 1:
        return Abstain("NO_UNIQUE_REFERENCE")

    r = unique[0]

    if item_type != "WATCH":
        return Abstain("NON_WATCH_ITEM")

    if not passes_certification_gate(r, offer, brand):
        return Probable(r)

    return CertifiedReference(
        brand_id=brand.id,
        reference=r.canonical,
        evidence=r.evidence,
    )
```

实体解析：

```python
def attach_entity(offer, resolved):
    if resolved.status != "CERTIFIED":
        return None

    entity = get_or_create_entity(
        resolved.brand_id,
        resolved.reference,
    )

    save_offer_entity_mapping(
        offer_id=offer.id,
        entity_id=entity.id,
        evidence=resolved.evidence,
        resolver_version=RESOLVER_VERSION,
    )

    return entity.id
```

---

## 20. 必须做的 false-positive regression suite

上线前建立固定测试集，每次规则/模型升级必须 0 回归。

建议至少包含：

```text
1. 表带标题含多个腕表 reference
2. 表盒/保卡/配件含目标手表 reference
3. 同系列相邻 reference
4. 只有 suffix 不同的 reference
5. OCR 0/O、1/I 混淆
6. 平台 SKU 看起来像品牌 reference
7. title 与 structured reference 冲突
8. OCR 与 title reference 冲突
9. brand 缺失但 reference 很像
10. 同 reference string 出现在不同 brand
11. 卖家写“对标/同款/类似 XXX”
12. 历史型号列表出现在 description
13. 一条商品同时出现旧 reference / 新 reference
14. reference 被截断
15. reference 中分隔符的危险归一化
```

任何一条从 `ABSTAIN/NON_MATCH` 变成 `CERTIFIED MATCH` 都必须人工审查。

---

## 21. 这个方案和“直接上 embedding / 向量库”相比为什么更适合

千万级商品很容易想到：

```text
title embedding + image embedding
       ↓
vector DB top-k
       ↓
cross encoder / VLM
       ↓
match
```

这个方案召回会很好，但不适合当前核心定义。

因为：

- 向量相似回答的是“像不像”；
- Spec 要回答的是“reference 是否完全相同”；
- 同系列相邻腕表恰恰是最像、却最不能合并的对象。

向量库可以作为：

```text
reference 缺失时的人工候选召回
OCR/抽取失败时的 review helper
发现疑似重复抓取
```

但不能成为最终 automatic merge authority。

这个项目的关键资产应该是：

```text
高精度 reference resolver
+ 可追溯 evidence ledger
+ brand-specific canonicalization rules
+ reference ownership model
+ 高质量 reference dictionary
```

而不是一个“整体相似度更高”的 matcher。

---

## 22. 最终建议

这篇论文最值得直接落地的一句话可以概括为：

> **先提取并验证产品标识的归属，再做实体匹配；不要用整条商品的相似度去猜标识。**

针对当前雷小安 × 腕表之家 × 奢当家三源数据，我建议将系统明确设计成：

```text
Raw Offer
   ↓
Brand Resolve
   ↓
Item Type / Accessory Guard
   ↓
Multi-channel Reference Candidate Extraction
   ↓
Identifier Role Classification
   ↓
Reference Ownership Verification
   ↓
Conservative Brand-specific Canonicalization
   ↓
CERTIFIED / CONFLICT / ABSTAIN
   ↓
只对 CERTIFIED 执行 (brand, reference) EXACT JOIN
   ↓
Entity Cluster
```

最关键的工程原则：

1. **reference 不确定就不匹配。**
2. **出现多个冲突 reference 就不匹配。**
3. **配件语境中的腕表 reference 默认不是当前商品 reference。**
4. **品牌不确定时不生成实体主键。**
5. **canonicalization 只能做经过品牌规则证明安全的变换。**
6. **图片和文本相似度只能辅助、排序或否决，不能推翻 reference 冲突。**
7. **模型必须允许 abstain。**
8. **所有自动实体合并都必须有可回溯证据和 resolver version。**
9. **先用低 coverage 建立几乎零 FP 的 CERTIFIED 数据，再利用这批数据自举 reference dictionary 和模型。**
10. **最终优化目标是在极高 precision 约束下逐步提高 coverage，而不是最大化 F1。**

从实现成本看，第一版甚至不需要复杂深度模型：PostgreSQL/ClickHouse + 规则服务 + OCR + 少量异步 worker 就能先跑出安全的 `reference_entity` 层。后续再将 ownership classifier、LLM extraction、图像证据和 active learning 逐步接入，但不改变“只有 certified reference exact equality 才能自动合并”的系统不变量。

这比原论文直接把 product code 作为 SVM 特征更严格，却正好满足当前需求对 false positive 的极端敏感性，也能把 100 万～1000 万级跨源匹配问题从昂贵的 pairwise similarity 判断，转化为可扩展、可解释、可增量、可审计的 reference key resolution + exact join。