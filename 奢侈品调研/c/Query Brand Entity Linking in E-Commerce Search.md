# Query Brand Entity Linking in E-Commerce Search：技术实现、架构拆解与跨源二奢 Reference 匹配落地方案

> 分析者：c  
> 目标需求：Notion「调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）」  
> 论文：Dong Liu, Sreyashi Nag, *Query Brand Entity Linking in E-Commerce Search*, arXiv:2502.01555（v2 2025-10-14）  
> 论文地址：https://arxiv.org/abs/2502.01555  
> PECOS 官方实现：https://github.com/amzn/pecos

## 1. 结论先行

这篇论文最值得借鉴的，不是“用一个模型直接判断两条商品是不是同一件”，而是它在生产系统里采用了一个非常适合当前 Spec 的原则：

1. **高精度的确定性路径优先**：NER 抽出品牌后，优先用 exact lexical match（精确词典）链接到 canonical entity。
2. **模型只负责补覆盖率**：对词典覆盖不到的长尾、别名、跨语言表达，使用 PECOS 这种极大类别空间分类（XMC）模型给出候选 entity。
3. **强约束做二次过滤**：论文用 Product Type 做候选过滤；映射仍不唯一时直接不预测，而不是“挑最像的一个”。
4. **Fusion 时显式偏向高 precision 路径**：当 exact lexical 与模型都有结果时，优先采用 lexical match 的结果。
5. **把拒识（abstain）当成正常输出**：论文明确为了 precision，在多候选仍无法消歧时不做 brand entity 预测。

这与当前需求“**同一商品=同一 reference number；绝对不能误匹配；允许漏匹配**”高度一致。

但不能原样照搬论文。论文的输出实体是 `brand_entity`，我们的最终输出实体是 `reference_entity`。最合理的迁移方式是：

> **先解决 canonical brand，再把 reference number 也建模成一个受品牌约束的 entity linking 问题。**

最终生产决策不能是“模型说这两个商品相似，所以合并”，而应该是：

> **模型只帮助“从脏文本/图片中找到可能对应的 canonical reference”；真正自动合并只在 canonical reference 的硬证据通过后发生。**

推荐的最终系统是一个 **Precision-first Reference Linker**：

```text
三源商品
  │
  ├─> 字段标准化 / 编号角色识别 / OCR
  │
  ├─> Canonical Brand Linker
  │      ├─ Exact Alias Dictionary  ──高置信──┐
  │      └─ PECOS/XMC Candidate      ──补召回──┤
  │                                          ▼
  ├─> Reference Mention Extractor
  │      ├─ 独立 reference 字段
  │      ├─ 标题/描述
  │      └─ 图片 OCR（表背/保卡/吊牌）
  │
  ├─> Reference Candidate Linker
  │      ├─ Exact canonical alias
  │      └─ PECOS top-k（只作为候选）
  │
  ├─> Hard Evidence Verifier
  │      ├─ brand 必须一致
  │      ├─ canonical reference 必须唯一
  │      ├─ 禁止冲突 reference
  │      ├─ 禁止把 SKU/序列号/兼容型号当 reference
  │      └─ 歧义则 ABSTAIN
  │
  └─> entity_id = brand_id + canonical_reference
         ├─ ACCEPT 自动归并
         ├─ REVIEW 人工复核
         └─ REJECT 不归并
```

这套方案可以先不用大模型，几百条黄金标注也足够启动；后续再用模型提升抽取与候选召回，但**不改变最终硬规则收口**。

---

## 2. 为什么选这篇论文

当前 Spec 的核心矛盾不是“如何找到更多相似商品”，而是：

- 三个平台字段不一致；
- reference 有时是独立字段，有时埋在标题；
- 品牌和型号都有别名、语言、空格、连字符等格式差异；
- 数据规模 100 万–1000 万，并持续增量；
- 最重要的是 false positive 几乎不可接受。

论文面对的是一个相似的生产问题：从短、噪声大的电商 query 中，把各种品牌 surface form 链到全局唯一 brand entity。论文指出品牌空间有数万级、长尾明显，字符串匹配 precision 高但 coverage 不够，而纯语义模型覆盖率高却更容易误报。

论文最终没有选择“全面模型化”，而是保留 exact lexical 作为高精度锚点，再用 XMC 补覆盖并做融合。这种架构比单纯用 embedding / LLM 做 pairwise matching 更适合作为当前项目的主干。

调研清单对该论文的推荐理由本身也非常准确：先把跨来源品牌统一到 canonical brand，再在品牌内解析 reference，可以显著减少不同品牌之间相似型号字符串造成的误匹配。

---

## 3. 论文技术方案拆解

## 3.1 任务建模：从 surface form 到唯一 entity

论文不是做普通 NER，而是完整 Entity Linking：

```text
raw query
  -> mention detection
  -> candidate generation / matching
  -> disambiguation
  -> unique brand entity
```

这和我们应做的 reference linking 完全同构：

```text
商品字段/标题/OCR
  -> reference mention extraction
  -> candidate canonical reference generation
  -> brand/品类/证据消歧
  -> unique reference entity
```

关键区别是：reference number 本身是强标识符，因此我们可以比论文更保守：**只有链接到唯一 canonical reference 时才自动聚合，语义相似度永远不能替代 reference 一致性。**

## 3.2 两阶段框架：NER + Matcher + Context Filter

论文的两阶段公式可以写成：

```text
m = f_NER(q)
E = g(m)
e = h(E, q, product_type)
```

其中：

- `f_NER`：从 query 中识别品牌 mention；
- `g`：把 mention 映射为一个或多个 brand entity 候选；
- `h`：结合 Product Type 等上下文做过滤和消歧。

值得注意的是：论文不是要求 `g` 一次就给出唯一答案，而是允许返回 candidate set，然后由更可靠的约束过滤。

对当前项目可改造成：

```text
mentions = extract_reference(product)
candidates = link_reference_candidates(brand_id, mentions)
reference_id = verify_and_select(candidates, product_evidence)
```

其中 `verify_and_select` 是整个系统最重要的 precision 防线。

## 3.3 Exact Lexical Match：论文里 precision 最可靠的基线

论文先构造一个 `brand-name -> brand-entity` 的静态词典，一个 brand entity 可以有多个合法 textual representation，然后对 NER 输出做 exact lookup。

这条路径有几个关键工程含义：

- canonical entity 与 alias 必须分开存；
- 一个 entity 可以有多个 alias；
- alias 甚至可能有 store-specific 版本；
- exact match 并不等于 raw string 完全一样，而是对“已经被明确收录为合法 alias”的字符串做精确映射。

对 reference number 也应如此：

```text
reference_entity
  id: ROLEX_126610LN
  brand_id: ROLEX
  canonical_reference: 126610LN

reference_alias
  126610LN
  126610-LN
  126610 LN
  M126610LN-0001   # 只有确认它确实代表同一 reference/变体关系后才能加入
```

重点是：**不要在线上临时使用“模糊编辑距离很近”把两个 reference 当成同一个；应该把确认过的等价写法沉淀成 alias，再做 exact alias lookup。**

`126610LN` 与 `126610LV` 只差一个字符，但必须永远是两个不同 entity。

## 3.4 PECOS：用极大类别空间分类补词典 coverage

论文把 mention->entity、query->entity 都建模成 Extreme Multi-Class Classification / Extreme Multi-Label Classification 问题。

其 entity label 空间约 6 万级，论文使用 PECOS。PECOS 的特点是用层次化 label tree 把巨大的输出空间组织起来，使推理从扫描全部 `L` 个标签，降低到与 `log L` 相关的搜索；论文给出的推理复杂度形式为：

```text
O(b * log L)
```

`b` 是 beam size，`L` 是 entity 数量。

PECOS 官方实现还提供 X-Linear / XR-Transformer 等组件，其中 X-Linear 可以构造层次化 label tree，并有 C++ 优化推理实现。这很适合百万到千万商品的离线/准实时链接。

对当前项目，我不建议让 PECOS 直接决定“是否同商品”，而建议让它输出：

```text
input: brand_id + normalized title + extracted reference-like strings
output: top-k canonical reference_id candidates + scores
```

然后交给硬规则验证器。

这样 PECOS 的职责是 **candidate retrieval / normalization**，不是最终 merge classifier。

## 3.5 Product Type Filter：论文里非常关键但容易被忽略的一层

论文发现同一个 brand surface form 可能链接到多个 brand entity，于是提前从 catalog 数据构建 `brand -> associated product types`，再用 query 的 Product Type 预测做过滤。

论文报告该过滤可以让超过一半的多候选案例收敛为单一 entity；如果仍然有多个 candidate，就为了高 precision 直接不预测。

迁移到腕表 reference 场景，可以把 Product Type Filter 升级为一组更强约束：

- `brand_id` 必须相同；
- `product_type` 必须兼容（腕表、表带、表盒、配件不能混）；
- `series/family` 若可靠则必须兼容；
- reference 的字符模式必须符合该品牌规则；
- 标题中出现的 reference 必须属于“当前出售商品”，不能只是“适用于 XXX 型号”的兼容描述；
- 独立字段、标题、OCR 多路证据如果冲突则拒识；
- 候选不唯一则拒识。

这比“统一一个模型阈值”更可控。

## 3.6 Q2E-PECOS：端到端模型的价值与限制

论文还训练了 Query-to-Entity PECOS，绕过 NER，直接从完整 query 预测 brand entity，并加入 `NIL` 类处理非品牌 query。

优势：

- 可以识别没有显式品牌词、只有隐式产品线信息的情况；
- coverage 更高；
- 一阶段部署简单。

但论文自己的结果也说明：纯 Q2E-PECOS precision 不如 exact lexical；exact lexical 的 false alarm 最低。论文最终使用 Fusion，而不是让端到端模型完全取代词典。

这正是本项目需要坚持的边界：

> 端到端模型适合提升“候选 reference 找得到”的概率，不适合成为自动归并的唯一依据。

## 3.7 Fusion：真正值得复制的生产策略

论文并行运行：

- NER + Exact Lexical Match
- Q2E-PECOS

当二者都有结果时，**优先 lexical match**，理由就是 precision 更高。

对于当前项目，应把 Fusion 进一步改成有层级的决策体系：

```text
Tier A: authoritative/exact evidence
Tier B: verified alias evidence
Tier C: model candidate + hard verification
Tier D: model-only candidate
```

只允许 A/B/C 中满足严格条件的进入 ACCEPT；D 永远只能 REVIEW。

---

## 4. 针对当前 Spec 的推荐总体架构

## 4.1 核心原则

### 原则 1：匹配对象不是商品 pair，而是 canonical reference entity

不要对三源做全量 pairwise：

```text
雷小安商品 A <-> 腕表之家商品 B ?
腕表之家商品 B <-> 奢当家商品 C ?
...
```

而是每条商品独立执行：

```text
product -> canonical_reference_entity
```

最后自然得到：

```text
canonical_reference_entity -> [雷小安商品, 腕表之家商品, 奢当家商品]
```

这样避免三源笛卡尔积，增量也非常简单。

### 原则 2：最终实体键必须是 brand + reference

建议全局主键：

```text
reference_entity_key = canonical_brand_id + ":" + canonical_reference
```

不能只使用 reference 字符串，因为不同品牌可能存在相同或高度相似编号。

### 原则 3：模型不能越权

即使 embedding / LLM / PECOS 给 0.999，也不允许把 `126610LN` 与 `126610LV` 合并。

模型可以做：

- reference mention extraction；
- alias candidate recall；
- brand linking；
- reference candidate ranking；
- 人工复核排序。

模型不能单独做：

- 最终自动 merge。

### 原则 4：ABSTAIN 是一等公民

最终状态必须至少有：

```text
ACCEPT   唯一且证据充分，自动归并
REVIEW   有候选但证据不足/冲突，人工复核
REJECT   明确不是可链接 reference / 配件 / SKU / 冲突
```

不要把所有记录强制分配到某个 reference。

---

## 5. 数据层设计

推荐至少维护以下 7 张逻辑表。

## 5.1 `product_raw`

```sql
product_raw(
  source              varchar,   -- leixiaoan / xcar_watch / shedangjia
  source_product_id   varchar,
  title               text,
  description         text,
  raw_brand           varchar,
  raw_reference       varchar,
  category            varchar,
  image_urls          jsonb,
  raw_payload          jsonb,
  crawled_at           timestamp,
  content_hash         varchar,
  primary key(source, source_product_id)
)
```

`content_hash` 用于增量判断：内容没变就不重复跑后续重模型。

## 5.2 `brand_entity`

```sql
brand_entity(
  brand_id             varchar primary key,
  canonical_name       varchar,
  status               varchar
)
```

## 5.3 `brand_alias`

```sql
brand_alias(
  normalized_alias     varchar,
  source_scope         varchar null,
  brand_id             varchar,
  provenance           varchar,
  confidence_tier      smallint,
  primary key(normalized_alias, source_scope, brand_id)
)
```

允许 source-specific alias，但如果同一 alias 指向多个 brand，不能直接自动链接，必须进入消歧。

## 5.4 `reference_entity`

```sql
reference_entity(
  reference_id         bigint primary key,
  brand_id             varchar,
  canonical_reference  varchar,
  family               varchar null,
  product_type         varchar null,
  status               varchar,
  unique(brand_id, canonical_reference)
)
```

## 5.5 `reference_alias`

```sql
reference_alias(
  brand_id             varchar,
  normalized_alias     varchar,
  reference_id         bigint,
  alias_type           varchar,   -- exact / punctuation_variant / catalog_variant
  provenance           varchar,
  is_auto_acceptable   boolean,
  primary key(brand_id, normalized_alias, reference_id)
)
```

`is_auto_acceptable` 很重要：不是所有历史 alias 都有资格自动归并。

## 5.6 `product_evidence`

```sql
product_evidence(
  source               varchar,
  source_product_id    varchar,
  evidence_type        varchar,   -- explicit_field/title/description/image_ocr
  raw_value            text,
  normalized_value     text,
  role                 varchar,   -- reference/sku/serial/compatible_reference/unknown
  extractor_version    varchar,
  score                double precision,
  metadata             jsonb
)
```

这张表让每个决策都可以追溯“为什么链接”。

## 5.7 `product_reference_link`

```sql
product_reference_link(
  source               varchar,
  source_product_id    varchar,
  reference_id         bigint null,
  decision             varchar,   -- ACCEPT/REVIEW/REJECT
  decision_tier        varchar,   -- A/B/C/D
  decision_reason      varchar,
  model_version        varchar null,
  rule_version         varchar,
  decided_at           timestamp,
  primary key(source, source_product_id)
)
```

任何自动合并都必须可复现、可回滚。

---

## 6. Reference 规范化：不要只做一个“去符号函数”

这里是最容易制造误匹配的地方。

建议保留三个层级的表示，而不是只存一个 normalized string。

## 6.1 `norm_safe`

只做几乎不会改变语义的转换：

- Unicode NFKC；
- Latin 大写；
- trim；
- 全角到半角；
- 标准化不同 dash；
- 连续空白压缩。

例：

```text
“126610－ln” -> “126610-LN”
```

## 6.2 `fingerprint_separatorless`

额外移除明确无语义的分隔符：空格、`-`、`_` 等。

```text
126610-LN -> 126610LN
```

但这个 fingerprint 只用来**找候选 alias**，不要直接把它当 canonical reference。

## 6.3 `brand_specific_canonicalizer`

品牌不同，reference 语法不同。应该给主要品牌配置 brand-specific parser：

```yaml
ROLEX:
  patterns:
    - '[A-Z]?\d{5,6}[A-Z]{0,3}'
  safe_separators: [' ', '-']

OMEGA:
  patterns:
    - '\d{3}\.\d{2}\.\d{2}\.\d{2}\.\d{2}\.\d{3}'
```

不要用一个全局规则把点号全部删除，否则某些品牌的层级编号可能被错误折叠。

---

## 7. 编号角色分类：避免把“长得像型号”的字符串都当 reference

三源数据中会出现：

- 品牌 reference / model number；
- 平台商品 ID；
- 商家 SKU；
- 序列号 serial；
- 机芯号；
- 配件兼容型号；
- 保卡/订单号；
- 年份、尺寸等数字。

所以在 Reference Linker 前要有一个轻量 `identifier_role_classifier`。

优先顺序建议：

1. **来源字段规则**：已知字段语义最可靠；
2. **邻近词规则**：`型号/腕表型号/ref/reference/model` 与 `货号/SKU/库存编号/序列号` 区分；
3. **品牌 regex / reference catalog membership**；
4. 小模型分类器作为补充；
5. 不确定时标为 `unknown`，不能自动 ACCEPT。

尤其要防：

```text
“适配 Rolex 126610LN 表带”
```

这里的 `126610LN` 是 `compatible_reference`，当前商品本身不是 Rolex 126610LN 腕表。

---

## 8. Canonical Brand Linker：直接落地论文方案

品牌层可以几乎原样使用论文架构。

## 8.1 Path A：Exact Alias Dictionary

```python
brand = exact_brand_alias_lookup(normalized_brand_or_title_mention)
```

如果唯一命中，记为 `brand_tier=A`。

## 8.2 Path B：PECOS brand candidate

当 exact 不命中时：

```python
candidates = pecos_brand_model.predict(text, topk=5)
```

然后用：

- source；
- product_type；
- catalog 中品牌对应品类；
- title 中其他证据

做 hard filter。

若过滤后唯一，可以作为 `brand_tier=B/C`；仍多候选则 REVIEW。

品牌链接错误会污染整个 reference 空间，因此 brand linker 也必须 precision-first。

---

## 9. Reference Candidate Linker：把论文的 Entity Linking 迁移到型号

## 9.1 Path A：显式字段 Exact

如果某平台有独立 reference 字段：

```python
mention = normalize(raw_reference)
candidates = reference_alias.lookup(brand_id, mention)
```

如果唯一且 alias 为 auto-acceptable：

```text
Tier A -> ACCEPT candidate
```

## 9.2 Path B：标题/描述抽取 + Exact Alias

通过规则/NER 抽出多个 reference-like span，再查 alias。

只有下面情况才允许自动继续：

```text
候选 reference 唯一
AND 没有第二个互相冲突的 product-reference 证据
AND identifier role == reference
```

## 9.3 Path C：PECOS XMC 候选

构造训练实例：

```text
X = [brand token] + title + selected attributes + extracted spans
Y = canonical reference_id
```

例如：

```text
[BRAND=ROLEX] 劳力士 潜航者 黑水鬼 41mm 126610 ln 全套
 -> ROLEX:126610LN
```

label 空间可以是全局 `reference_id`，但推理后必须做 brand hard-filter。

一种更容易运维的做法是：

- 一个 global PECOS model；
- 每个 label 带 `brand_id`；
- 先 brand linking；
- top-k 后丢弃所有非当前 brand label。

不建议一开始为每个 brand 维护独立模型，模型数量太多会增加版本和发布复杂度。

## 9.4 关键约束：PECOS candidate 不能直接 ACCEPT

模型候选必须回到证据层验证，例如：

```text
candidate = ROLEX:126610LN
```

至少满足下列一种才能自动通过：

- 标题/显式字段中存在 candidate 的已确认 exact alias；
- OCR 中存在高质量 exact alias，且文本侧没有冲突 reference；
- 两个相互独立证据源都映射到同一 candidate。

如果只是“语义很像 Submariner 黑水鬼”，但没有 reference 硬证据，最多 REVIEW。

---

## 10. 图片怎么用：作为证据，不作为身份定义

Spec 明确有图片可用。图片非常有价值，但应该放在正确的位置。

推荐只做两个任务：

### 10.1 OCR reference evidence

优先 OCR：

- 表背刻字；
- 保卡；
- 吊牌；
- 盒标；
- 证书。

OCR 输出也进入 `product_evidence`，经过同样的 `identifier_role_classifier + brand parser + alias lookup`。

### 10.2 视觉冲突否决

视觉模型可以识别：

- 腕表 vs 表带/盒子；
- 明显系列；
- 表盘关键外观；

但不应该仅凭“长得一样”把两个 reference 合并。

特别是同系列相邻 reference 往往外观极其接近，视觉更适合做 negative evidence / review routing，而不是唯一 positive evidence。

---

## 11. 最终 Decision Engine：建议写成确定性规则机

整个系统最值得工程化的部分不是模型，而是一个版本化、可审计的规则机。

示例：

```python
def decide(product, brand_result, evidences, ref_candidates):
    if brand_result.status != "UNIQUE":
        return REVIEW("brand_not_unique")

    refs = unique_verified_reference_ids(evidences)

    # 多条硬证据互相冲突
    if len(refs) > 1:
        return REVIEW("conflicting_reference_evidence")

    # 唯一精确硬证据
    if len(refs) == 1:
        ref = refs[0]
        if ref.brand_id != brand_result.brand_id:
            return REVIEW("brand_reference_conflict")
        if has_accessory_or_compatibility_signal(product, evidences):
            return REVIEW("possible_compatible_reference")
        return ACCEPT(ref, tier="A/B")

    # 没有硬证据，但模型有候选
    if ref_candidates:
        return REVIEW("model_candidate_without_hard_reference")

    return REJECT("no_reference_evidence")
```

第一版宁可非常保守，也不要为了 coverage 添加危险 fallback。

---

## 12. 如何把 PECOS 真正跑起来

PECOS 官方库可以直接安装：

```bash
pip install libpecos
```

对于 X-Linear，训练数据核心是：

- `X`: instance-feature CSR matrix
- `Y`: instance-label CSR matrix

简化伪代码：

```python
from pecos.xmc.xlinear.model import XLinearModel
from pecos.xmc import Indexer, LabelEmbeddingFactory

label_feat = LabelEmbeddingFactory.create(Y, X)
cluster_chain = Indexer.gen(label_feat)
model = XLinearModel.train(X, Y, C=cluster_chain)
model.save("reference_xmc")
```

推理：

```python
model = XLinearModel.load("reference_xmc", is_predict_only=True)
pred = model.predict(X_new)
```

### 第一版特征不必上 Transformer

推荐先用：

- word/char n-gram TF-IDF；
- brand token；
- source token；
- product_type token；
- reference-like span token；
- title token。

reference number 本质上字母数字模式很重要，char n-gram 往往比纯语义 embedding 更符合任务。

例如：

```text
126610LN
```

char 3-gram 会保留：

```text
126, 266, 661, 610, 10L, 0LN
```

而 `126610LV` 会在尾部特征明显不同。

这类表示对 hard negative 更友好。

### 什么时候再上 XR-Transformer

只有当发现大量真实 reference 无法通过字段、alias、char n-gram 候选召回时，再考虑 XR-Transformer / 小型 encoder。

当前目标是 precision-first，不要一开始把复杂语义模型放在主链路。

---

## 13. 黄金标注怎么花：不要随机标几百对

Spec 可接受人工标几百对。最优用法不是随机抽样，而是构造 **hard-case gold set**。

建议 400–800 条按下面分布采样：

1. 同品牌、同系列、reference 仅 1 个字符不同；
2. 一个有独立型号字段、另一个只有标题；
3. 标题出现多个 reference；
4. “适配/兼容某型号”的配件；
5. SKU 长得像 reference；
6. OCR 误识别 O/0、I/1、B/8；
7. 中文/英文/日文别名品牌；
8. 不同品牌但 reference 字符串相似；
9. 同 reference 不同年份/成色/套装；
10. 新品牌/新 reference 长尾。

标注内容不应只是 `pair match = yes/no`，最好同时标：

```text
canonical_brand
reference_mentions
identifier_role
canonical_reference
final_linkable
error_reason
```

这样一份标签可以同时训练/评估多个组件。

---

## 14. 评估指标：不要把 F1 当主指标

当前项目主指标应该改成：

### 14.1 Auto-link Precision

```text
precision_accept = 正确 ACCEPT / 全部 ACCEPT
```

这是第一优先级。

### 14.2 Coverage

```text
coverage = ACCEPT 数 / 全量商品数
```

在 precision 达标后再最大化 coverage。

### 14.3 Abstention Rate

```text
abstain = REVIEW / 全量
```

REVIEW 高并不是失败，只要后续人工成本可控。

### 14.4 False Alarm Rate

参考论文专门测 non-brand query 的 false alarm。我们也应该建立“没有真实 reference 的商品”集合，例如：

- 配件；
- 盒证；
- 模糊标题；
- 只有 SKU；

观察系统错误产生 canonical reference 的比例。

### 14.5 Candidate Recall@K

只评 PECOS 候选层：

```text
真实 canonical reference 是否在 top-k
```

candidate recall 可以追求高；最终 precision 由 verifier 保证。

---

## 15. 针对“绝不能误匹配”的上线门槛

建议把上线门槛拆到 Tier，而不是一个统一 score threshold。

### Tier A：可以最早上线

条件：

- canonical brand 唯一；
- 来源独立 reference 字段；
- exact/approved alias 唯一命中；
- 无冲突证据。

预期 coverage 可能不高，但风险最低。

### Tier B：第二阶段上线

条件：

- title/description 抽取 reference；
- approved exact alias 唯一命中；
- role classifier 判定为本商品 reference；
- 无 compatibility/accessory 信号。

### Tier C：谨慎灰度

条件：

- PECOS 召回 candidate；
- 另一路独立证据能够 exact 验证 candidate；
- hard-negative 回归集无新增 false positive。

### Tier D：永不自动归并

- 只有 embedding/LLM/PECOS score；
- 没有可验证 reference 字符串。

只进 REVIEW。

---

## 16. 100 万–1000 万规模下的工程部署

这里不需要做三源 N² 匹配，因此规模问题会大幅简化。

## 16.1 离线/增量流水线

推荐：

```text
Crawler
  -> Kafka/任务表（可选）
  -> Raw Storage (S3/MinIO + Parquet/Iceberg 或现有 DB)
  -> Normalize/Extract Workers
  -> Brand Linker
  -> Reference Linker
  -> Decision Engine
  -> Link Store
```

10M 商品的核心负载是每条商品一次线性处理，而不是 10M × 10M 比较。

## 16.2 在线存储建议

如果团队已有 PostgreSQL，可以先直接用：

- PostgreSQL：canonical entity、alias、decision、review queue；
- 对大体量 raw/evidence 历史使用对象存储 + Parquet；
- Redis/RocksDB：可选，用于高频 exact alias lookup；
- PECOS 模型常驻内存做 batch prediction。

没有必要第一版就引入复杂向量数据库。

## 16.3 增量更新

每次 crawl：

```python
if content_hash == previous_content_hash:
    skip()
else:
    rerun_extraction_and_linking()
```

Canonical KB 更新时，用版本号触发受影响记录重算：

```text
brand_alias_version
reference_alias_version
extractor_version
model_version
rule_version
```

这样所有历史 decision 都能 replay。

---

## 17. 人工复核工作台应该展示什么

人工不能只看两条商品标题。

Review UI 应展示：

```text
[商品]
source / source_product_id / title / images

[Brand]
raw brand
canonical brand
brand evidence

[Reference Evidence]
explicit_field: 126610-LN
headline: 126610LN
OCR: 126610LN

[Candidate]
ROLEX:126610LN
PECOS score
exact alias hit

[Conflict]
是否存在其他 reference-like token
是否出现“适配/兼容/表带/配件”

[Action]
Accept canonical alias
Reject candidate
Mark as SKU
Mark compatibility reference
Create new canonical reference
```

人工操作应直接回流 `brand_alias/reference_alias/identifier_role`，而不是只记录一个 pair label。

---

## 18. 推荐实施顺序

## Phase 0：先做可测数据

- 统一三源 raw schema；
- 抽 400–800 条 hard-case gold；
- 建立 canonical brand/reference 初始表；
- 建 false-positive regression set。

产物：离线 evaluator。

## Phase 1：无模型高精度 MVP

实现：

- brand alias exact linker；
- reference normalization；
- reference alias exact linker；
- identifier role rules；
- ACCEPT/REVIEW/REJECT rule engine；
- audit table。

这一步就能产生第一批可信跨源聚类。

## Phase 2：抽取增强

- 标题 reference regex/NER；
- OCR；
- 多证据冲突检测；
- 品牌特定 parser。

仍然保持 exact canonical reference 收口。

## Phase 3：PECOS Candidate Linker

- 构造 `(product text -> reference_id)` 训练数据；
- char n-gram + X-Linear；
- top-k candidate；
- brand hard filter；
- 只提升 REVIEW 命中和可验证 ACCEPT coverage。

## Phase 4：持续学习

- 人工 REVIEW 结果回流 alias 与训练集；
- 定期重训 PECOS；
- 按品牌/source 监控 precision/coverage 漂移；
- 新 alias 先影子运行，再升为 auto-acceptable。

---

## 19. 一个可直接开发的服务 API 草案

### `POST /v1/link-product`

请求：

```json
{
  "source": "leidangjia",
  "source_product_id": "123",
  "title": "劳力士潜航者型黑水鬼 126610LN 41mm 全套",
  "brand": "劳力士",
  "reference": null,
  "category": "腕表",
  "ocr_texts": ["126610LN"]
}
```

返回：

```json
{
  "decision": "ACCEPT",
  "decision_tier": "B",
  "brand": {
    "brand_id": "ROLEX",
    "method": "exact_alias"
  },
  "reference": {
    "reference_id": 1000123,
    "canonical_reference": "126610LN",
    "method": "title_exact_alias+ocr_exact_alias"
  },
  "reasons": [
    "unique_brand",
    "unique_reference",
    "two_independent_exact_evidences",
    "no_conflict"
  ],
  "versions": {
    "extractor": "ref-extractor-0.3",
    "model": null,
    "rules": "decision-0.2"
  }
}
```

对于只有模型候选的情况：

```json
{
  "decision": "REVIEW",
  "decision_tier": "D",
  "reference_candidates": [
    {"reference_id": 1000123, "score": 0.97},
    {"reference_id": 1000456, "score": 0.81}
  ],
  "reasons": ["model_candidate_without_hard_reference"]
}
```

---

## 20. 与常见替代方案的比较

| 方案 | 优点 | 对本 Spec 的问题 | 推荐定位 |
|---|---|---|---|
| 字符串 exact | precision 高、快、可解释 | coverage 低 | 主链路 |
| 编辑距离/fuzzy | 简单、召回高 | 相邻 reference 极易误合并 | 仅候选召回，禁止自动 merge |
| Embedding ANN | 长文本语义好 | 型号 1 字符差异可能被认为极相似 | 候选召回/Review |
| LLM pairwise | 开发快、能看复杂上下文 | 成本、稳定性、false positive 不易控 | 难例辅助，不做最终裁决 |
| PECOS XMC | 大 label 空间高效、适合 entity linking | 单独使用 precision 不如 exact | 候选归一化补召回 |
| 图片相似度 | 字段缺失时有补充 | 同系列不同 reference 外观很像 | OCR/冲突辅助 |
| 本文推荐 Fusion | precision-first、可拒识、可增量 | 需要维护 canonical KB / alias | 最适合作为生产主架构 |

---

## 21. 需要特别反驳的一个直觉：不要先做“商品相似度”再阈值切分

当前问题定义已经明确：**同一 reference 才是同一商品实体**。

因此下面这种方案逻辑上就是次优的：

```text
title embedding similarity
+ image similarity
+ brand similarity
+ price similarity
> threshold => same product
```

即使整体 F1 很高，也会把同系列、不同 reference 的腕表误合并。对于业务约束“绝对不能误匹配”，这类错误比漏匹配严重得多。

更合适的顺序是：

```text
先解析 identity key（brand + reference）
再用文本/图片帮助解析这个 key
```

而不是：

```text
先判断两个商品看起来像不像
再猜它们是不是同 identity
```

这是本文对当前需求最大的架构价值。

---

## 22. 最终建议

如果现在就开工，我建议把第一版范围压到下面 5 个模块：

1. `CanonicalBrandStore`
2. `ReferenceAliasStore`
3. `ReferenceExtractor`
4. `IdentifierRoleClassifier`
5. `PrecisionDecisionEngine`

先完全不依赖 PECOS，也能上线一个 coverage 较低但可控的高 precision 系统。

当 baseline 运行后，再增加：

6. `PECOSBrandCandidateModel`
7. `PECOSReferenceCandidateModel`
8. `ImageOCRExtractor`
9. `HumanReviewLoop`

最重要的是保持系统不变量：

> **任何学习模型都只能扩大“可验证候选”的覆盖率，不得改变“自动归并必须有唯一 canonical reference 硬证据”的规则。**

这比论文原始的 brand linking 还要更保守，但正好匹配本项目 precision 优先到极致的约束。

---

## 参考资料

- Dong Liu, Sreyashi Nag. Query Brand Entity Linking in E-Commerce Search. https://arxiv.org/abs/2502.01555
- PECOS - Predictions for Enormous and Correlated Output Spaces. https://github.com/amzn/pecos
- 当前需求：Notion「调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）」
