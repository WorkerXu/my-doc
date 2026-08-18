# Deep Entity Matching with Pre-Trained Language Models

> 分析人：c  
> 项目/论文：Yuliang Li, Jinfeng Li, Yoshihiko Suhara, AnHai Doan, Wang-Chiew Tan, **Deep Entity Matching with Pre-Trained Language Models（Ditto）**, PVLDB 2021  
> 论文：https://arxiv.org/abs/2004.00584  
> 论文 HTML：https://arxiv.org/html/2004.00584v3  
> 官方代码：https://github.com/megagonlabs/ditto  
> 调研清单：`奢侈品文章调研.md`  
> 目标需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务定义：**“同一个商品” = 同一 reference number / 型号**；数据规模 100 万–1000 万、持续增量；字段稀疏；reference 可能埋在标题；图片可用；**precision 极端优先，绝不能误匹配，允许漏匹配**；可人工标注几百对黄金标签。

---

## 1. 选题与去重

执行前检查 `奢侈品调研/c/`，当前已有 23 个分析结果，包括 Ameli、AnyMatch、Confidence Classifiers、Conformal Selective Prediction、DeepBlocker、GraLMatch、MOON2.0、TransClean、pyJedAI 等，但没有 `Deep Entity Matching with Pre-Trained Language Models.md`，因此本次选择 Ditto，不重复既有分析。

`奢侈品文章调研.md` 中对应条目推荐它的理由是：

> 将实体匹配建模为预训练语言模型的序列对分类，并支持注入领域知识、难例增强和少量标注训练；可用于 reference 抽取后的候选复核或打分，处理字段稀疏与脏文本，但自动合并仍应要求规范化 reference 一致。

这篇论文很适合当前需求，因为它同时覆盖了：

1. **异构 schema**：三个来源字段不一致；
2. **脏数据与缺字段**：Ditto 明确在 Dirty benchmark 上有优势；
3. **少量标注**：论文强调 label efficiency；
4. **大规模 blocking + matcher 两阶段架构**；
5. **商品域**：WDC benchmark 直接包含 Watches；
6. **领域知识注入**：可以把 reference、品牌、serial、平台 SKU 等“编号角色”显式标给模型。

但对本需求最关键的结论不是“直接部署 Ditto 做二分类”，而是：

> **Ditto 很适合作为 reference 实体解析中的候选消歧器，却不应该成为最终自动合并的唯一裁判。**

原因是官方实现默认通过验证集寻找 **F1 最优阈值**，而当前业务要求是 **precision-first / zero-false-positive-first**。如果原样上线，目标函数就已经与需求错位。

---

## 2. Ditto 实际解决的技术问题

传统 Entity Matching 的输入是两个记录：

```text
record_a + record_b
```

输出：

```text
MATCH / NO_MATCH
```

Ditto 的核心思想非常简单：

```text
结构化/半结构化记录
        ↓ 序列化
Transformer sequence-pair classification
        ↓
match probability
```

论文把完整系统明确拆成两部分：

```text
              ┌─────────────┐
Dataset A ───▶│             │
              │   Blocker   │── candidate pairs ──▶ Ditto Matcher
Dataset B ───▶│             │                         │
              └─────────────┘                         ▼
                                                MATCH / NO_MATCH
```

Blocking 追求高 recall，把笛卡尔积压缩成候选对；Ditto 只负责候选对判别。

这与 100 万–1000 万规模非常重要：即使只有 100 万 × 100 万，两源全量 pairwise 比较也是不可接受的。论文真实案例中两张公司表分别约 78.9 万与 41.2 万条，先通过规则/TF-IDF blocking 生成约 1065 万候选，再用 Sentence-BERT top-k 做高级 blocking，把最终 matcher 的比较量继续压缩。

---

## 3. Ditto 的模型架构

### 3.1 记录序列化

Ditto 不要求左右记录 schema 完全一致。每条记录按属性序列化：

```text
COL title VAL ...
COL manufacturer VAL ...
COL modelno VAL ...
COL price VAL ...
```

论文形式为：

```text
[COL] attr_1 [VAL] value_1 ... [COL] attr_k [VAL] value_k
```

候选对再组成：

```text
[CLS] serialize(left) [SEP] serialize(right) [SEP]
```

`[CLS]` 的 contextual embedding 进入一个简单的线性分类层：

```text
Transformer
   ↓
CLS embedding
   ↓
Linear(hidden_size, 2)
   ↓
Softmax
   ↓
P(match)
```

官方 `ditto_light/ditto.py` 也完全对应这一结构：

```python
self.bert = AutoModel.from_pretrained(...)
self.fc = torch.nn.Linear(hidden_size, 2)
...
enc = self.bert(x1)[0][:, 0, :]
return self.fc(enc)
```

换句话说，Ditto 没有复杂的属性对齐网络；它把“属性怎么对应、哪些 token 重要、哪些文本是同义表达”大部分交给预训练 Transformer 的 self-attention 学习。

### 3.2 对当前三源数据的意义

雷小安可能有：

```text
brand / title / model / price / images
```

腕表之家可能有：

```text
品牌 / 系列 / 型号 / 腕径 / 机芯 / 标题
```

奢当家可能又是：

```text
product_name / item_no / goods_code / description / images
```

不必先把三个来源强行映射成完全相同的 schema 才能送模型。可以保留来源原字段名，同时增加统一字段：

```text
COL source VAL leixiaoan
COL brand_norm VAL ROLEX
COL ref_struct VAL 126610LN
COL title VAL 劳力士潜航者黑水鬼...
COL source_sku VAL ...
```

这正是 Ditto 的强项。

---

## 4. 领域知识注入：最值得迁移的部分

论文把 Domain Knowledge 分两类：

1. **Span Typing**：告诉模型某段文本是什么角色；
2. **Span Normalization**：把等价表达规范成一致形式。

官方 ProductDKInjector 会：

- 用 spaCy NER 标记 PRODUCT、NUM 等；
- 对长数字/字母数字 token 粗略加 `ID`；
- 规范化部分数值格式。

论文的直觉是：如果模型知道“这是 Product ID”“这是 Brand”“这是配置数字”，就更容易把同角色信息互相对齐，而不是被无关相似文本带偏。

### 4.1 原版 ProductDKInjector 不能直接用于腕表

当前官方轻量代码中存在这种逻辑：

```python
elif len(token.text) >= 7 and any(ch.isdigit() for ch in token.text):
    res += 'ID ' + token.text + ' '
```

对一般商品是合理 heuristic，但对二奢/腕表有风险，因为“长且包含数字”可能是：

- 品牌 reference；
- 序列号 serial number；
- 平台 SKU；
- 店铺货号；
- 抓取内部 ID；
- 订单号；
- 保卡编号。

而当前需求只把 **reference number** 当作“同一个商品”的业务定义。

因此必须把 `ID` 拆成更细的角色：

```text
[REF]       品牌 reference / 型号
[SERIAL]    单只腕表序列号
[SKU]       来源平台 SKU
[ITEM_NO]   商家货号
[ORDER_ID]  交易/订单编号
[SIZE]      腕径等规格
[BRAND]     品牌
[SERIES]    系列
```

推荐新建：

```text
WatchReferenceDKInjector
```

而不是使用原版 `ProductDKInjector`。

### 4.2 Reference Normalization 必须按品牌规则化

不能做简单的：

```python
ref.replace('-', '').replace('.', '').replace('/', '')
```

然后全部 upper-case 就结束。

原因是 reference 的后缀、段结构、字符位本身可能是型号语义的一部分。比如：

```text
Rolex 126610LN
Rolex 126610LV
```

仅差两个后缀字符，但代表不同 reference，视觉又高度相似。

因此建议：

```text
normalize_reference(brand, raw_ref)
```

做成品牌级 grammar：

```text
RolexNormalizer
OmegaNormalizer
PatekNormalizer
APNormalizer
CartierNormalizer
GenericNormalizer
```

输出不仅是一个字符串，而是：

```json
{
  "raw": "126610 ln",
  "canonical": "126610LN",
  "brand": "ROLEX",
  "segments": ["126610", "LN"],
  "rule_version": "rolex-v3",
  "confidence": 0.999,
  "warnings": []
}
```

必须保存 `rule_version`，因为持续增量系统后续一定会升级 normalization 规则，历史结果要能回溯和重算。

---

## 5. Summarization：论文有效，但原版策略要改

Ditto 为长文本设计 TF-IDF summarization，而不是简单截断前 N token。论文在 Company 长文本数据上，summary 将 baseline F1 从约 41% 提升到 93% 以上。

这是合理的，因为商品描述真正有判别力的 token 可能在后半段，例如：

```text
标题：劳力士 潜航者 黑水鬼 全套附件...
描述：...
附件：...
型号：126610LN
```

如果固定截断，最关键的 reference 可能直接丢掉。

### 5.1 官方实现值得注意的代码细节

`ditto_light/summarize.py` 会建立 TF-IDF index，再保留高 IDF token。

但生产落地不建议原样复制，至少有三个原因：

1. 代码当前用 train/valid/test 一起 build index；工程上应只使用训练/生产参考语料，避免评测污染；
2. reference 是业务硬证据，**不能交给通用 TF-IDF 决定是否保留**；
3. 对中文标题/中英混合文本，原版英文 stopwords 与空格 tokenization 不适合。

建议改成 **Pinned-Identifier Summarization**：

```text
永远保留：
  brand
  reference candidates
  serial / sku role tags
  series
  size
  material
  OCR reference evidence

剩余 token：
  再按 BM25 / TF-IDF / attention score 做压缩
```

即：

```text
hard evidence first + semantic summary second
```

而不是纯统计摘要。

---

## 6. MixDA / 难例增强：非常适合字段稀疏，但要做“受控增强”

论文的 Data Augmentation 包含：

```text
span_del
span_shuffle
attr_del
attr_shuffle
entry_swap
```

官方轻量版本进一步提供：

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

训练时可产生增强样本，然后在表示空间使用 MixDA：

```python
enc1 = LM(original)
enc2 = LM(augmented)
lambda ~ Beta(alpha, alpha)
enc = lambda * enc1 + (1-lambda) * enc2
```

最后再分类。

论文在 WDC 商品数据上发现 `span_del` 很有效，说明模型通过随机缺失部分 token 学会对脏字段和信息缺失更稳健。

### 6.1 当前需求应如何改造增强逻辑

腕表场景不能盲目删除 reference，因为 label 定义本身就是 reference equality。

推荐：

#### 正样本增强

允许删除：

```text
price
seller
condition
marketing words
some description
some accessory information
```

谨慎删除：

```text
series
size
material
```

默认禁止删除：

```text
all trusted reference evidence
brand
```

#### 负样本增强

重点制造 **hard negatives**：

```text
同品牌
同系列
高文本相似
高图像相似
reference 只差 1~2 个字符/后缀
```

这比随机负样本重要得多。

示意：

```text
A: Rolex Submariner 126610LN
B: Rolex Submariner 126610LV
label = 0
```

模型必须学会：

```text
title 99% 相似 + 图片很像 != same reference
```

这才符合业务的误匹配风险。

### 6.2 官方轻量代码的一个实现风险

当前 `augment.py` 的 `del` 分支存在疑似代码问题：

```python
new_labels = tokens[:pos1] + labels[pos2+1:]
```

这里前半段看起来应当是 `labels[:pos1]`，却写成了 `tokens[:pos1]`。

因此这个 repo 更适合作为论文实现参考，不建议直接原封不动作为生产依赖。应重新实现并补 unit test / property test。

---

## 7. 官方代码默认阈值与当前需求冲突

这是本次分析中最需要强调的一点。

官方 `evaluate()` 在 validation set 上：

```python
for th in np.arange(0.0, 1.0, 0.05):
    pred = [1 if p > th else 0 for p in all_probs]
    new_f1 = metrics.f1_score(all_y, pred)
    if new_f1 > f1:
        best_th = th
```

即：

```text
选择 F1 最大的 threshold
```

但是需求写的是：

```text
绝对不能误匹配
precision 优先到极致
允许漏匹配
```

所以正确目标不是：

```text
max F1
```

而应是：

```text
max coverage
subject to precision >= P_target
```

甚至第一阶段直接规定：

```text
只要存在任何强 reference 冲突 => 永不 AUTO_MATCH
```

模型高分也不能越过这个 veto。

---

## 8. 最重要的架构重构：不要把问题做成 N×N 商品 pair matching

根据 Notion Spec，业务已经把“同一个商品”严格定义成：

```text
same product <=> same reference number
```

这意味着实际问题比通用 Entity Matching 更简单，也更适合做高精度系统。

### 8.1 推荐把问题改写成 Reference Entity Linking

不要主要做：

```text
雷小安 record A
   vs
腕表之家 record B
   -> same / not same
```

而应该做：

```text
record
  ↓
解析/链接到 canonical reference entity
  ↓
reference_entity_id
```

最终：

```text
record A.reference_entity_id == record B.reference_entity_id
=> same product
```

这样可以彻底避免大规模 pairwise 全比较，也避免 pairwise 模型的传递污染。

### 8.2 Canonical Reference Registry

维护一个中心表：

```text
reference_entity
```

核心唯一键：

```text
(brand_id, canonical_reference)
```

例如：

```text
ROLEX | 126610LN
OMEGA | 210.30.42.20.01.001
...
```

每条来源商品记录最终只是“链接”到这个 reference entity。

因此系统真正的 ML 子任务是：

```text
给定一个稀疏、脏、跨 schema 的商品记录
从该品牌候选 reference 中找出唯一 reference
或者拒绝判断
```

这比“直接判断两个商品是不是同一个”更符合业务定义，也更容易做到高 precision。

---

## 9. 建议的生产架构

```text
┌──────────────────────────────────────────────────────────┐
│                  Source Ingestion                        │
│ 雷小安 / 腕表之家 / 奢当家                               │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────┐
│ 1. Canonicalization                                      │
│ - brand normalize                                        │
│ - field role mapping                                     │
│ - source SKU / serial / reference role classification    │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────┐
│ 2. Reference Evidence Extraction                         │
│ - structured reference field                             │
│ - title regex/parser                                     │
│ - description parser                                     │
│ - image OCR                                               │
│ - provenance + confidence                                │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────┐
│ 3. Canonical Reference Registry                          │
│ exact index: (brand_id, canonical_ref)                    │
└───────────────┬───────────────────────────────┬──────────┘
                │                               │
      exact/high-confidence                 uncertain
                │                               │
                ▼                               ▼
┌─────────────────────────┐       ┌─────────────────────────┐
│ 4A. Deterministic Link  │       │ 4B. Candidate Retrieval│
│ exact same ref          │       │ brand + lexical + ANN   │
└──────────────┬──────────┘       └──────────────┬──────────┘
               │                                  │
               │                                  ▼
               │                       ┌──────────────────────┐
               │                       │ 5. Ditto Disambiguator│
               │                       │ custom DK + hard neg  │
               │                       └──────────┬───────────┘
               │                                  │
               └────────────────────┬─────────────┘
                                    ▼
                         ┌─────────────────────────┐
                         │ 6. Precision Policy     │
                         │ AUTO_LINK / REVIEW /    │
                         │ UNRESOLVED / REJECT     │
                         └──────────┬──────────────┘
                                    │
                                    ▼
                         ┌─────────────────────────┐
                         │ reference_entity_id     │
                         └──────────┬──────────────┘
                                    │
                                    ▼
                       Cross-source exact join/group
```

---

## 10. 数据模型建议

### 10.1 原始商品表

```sql
CREATE TABLE product_record (
    source              varchar(32) NOT NULL,
    source_record_id    varchar(128) NOT NULL,
    source_version      varchar(64),
    raw_json            jsonb NOT NULL,
    title               text,
    brand_raw           text,
    brand_id            bigint,
    image_urls          jsonb,
    content_hash        varchar(64),
    updated_at          timestamptz,
    PRIMARY KEY (source, source_record_id)
);
```

### 10.2 reference entity

```sql
CREATE TABLE reference_entity (
    id                  bigserial PRIMARY KEY,
    brand_id            bigint NOT NULL,
    canonical_ref       varchar(128) NOT NULL,
    ref_segments        jsonb,
    series_norm         text,
    profile_json        jsonb,
    status              varchar(32) NOT NULL,
    created_at          timestamptz NOT NULL,
    updated_at          timestamptz NOT NULL,
    UNIQUE (brand_id, canonical_ref)
);
```

### 10.3 证据表

```sql
CREATE TABLE reference_evidence (
    source              varchar(32) NOT NULL,
    source_record_id    varchar(128) NOT NULL,
    evidence_type       varchar(32) NOT NULL,
    raw_value           text,
    canonical_candidate varchar(128),
    confidence          double precision,
    extractor_version   varchar(64),
    image_id            text,
    metadata            jsonb,
    created_at          timestamptz NOT NULL
);
```

`evidence_type` 建议枚举：

```text
STRUCTURED_REF
TITLE_REF
DESCRIPTION_REF
OCR_REF
SERIAL
SOURCE_SKU
ITEM_NO
```

### 10.4 最终解析表

```sql
CREATE TABLE reference_resolution (
    source                 varchar(32) NOT NULL,
    source_record_id       varchar(128) NOT NULL,
    reference_entity_id    bigint,
    decision               varchar(32) NOT NULL,
    rule_score             double precision,
    ditto_score            double precision,
    threshold_version      varchar(64),
    policy_version         varchar(64) NOT NULL,
    reason_codes           jsonb,
    resolved_at            timestamptz NOT NULL,
    PRIMARY KEY (source, source_record_id)
);
```

`decision`：

```text
AUTO_LINK
REVIEW
UNRESOLVED
REJECT_CONFLICT
```

不要只存 boolean match。**三态/四态决策是 precision-first 系统的基本能力。**

---

## 11. Reference Evidence Extraction

### 11.1 优先级

建议证据可信度分层：

```text
Tier 0: 来源明确的 reference 独立字段
Tier 1: 标题中品牌 grammar 高置信抽取
Tier 2: 描述中高置信抽取
Tier 3: 图片 OCR 中符合品牌 reference grammar 的字符串
Tier 4: 模型/模糊检索猜测候选
```

不能让 Tier 4 覆盖 Tier 0/1 的冲突。

### 11.2 提取器不是“找所有字母数字串”

推荐：

```python
def extract_reference_candidates(record):
    brand = normalize_brand(record)
    candidates = []

    candidates += from_structured_fields(record, brand)
    candidates += from_title(record.title, brand)
    candidates += from_description(record.description, brand)
    candidates += from_ocr(record.images, brand)

    return classify_identifier_roles(candidates)
```

每个 candidate 必须携带：

```text
value
canonical_value
evidence_type
source_position
extractor_version
confidence
identifier_role
```

重点是 `identifier_role`。

---

## 12. 图片应该怎么用

当前 Spec 有图片，这是优势，但不建议“图片相似 => same reference”。

腕表同系列不同 reference 经常外观极度接近，甚至只改机芯、尺寸、材质、年份或细小表盘元素。

推荐图片用途按可靠性排序：

### 12.1 OCR 证据

优先识别：

```text
保卡
吊牌
标签
表背刻字
包装标签
商品信息卡
```

OCR 得到的字母数字串再进入：

```text
identifier role classifier
        ↓
reference grammar validator
```

不是 OCR 到数字就当 reference。

### 12.2 视觉 embedding

CLIP / SigLIP 类 embedding 只适合：

```text
candidate retrieval
candidate ranking
异常检测
```

不作为 positive proof。

### 12.3 冲突否决

如果文本高置信 reference 与 OCR 高置信 reference 冲突：

```text
AUTO_LINK = 禁止
=> REVIEW / REJECT_CONFLICT
```

图片更适合作为第二独立证据或否决信号，而不是越权替代 reference。

---

## 13. Ditto 在新架构中的正确位置

原论文是：

```text
record A vs record B -> match
```

推荐改成：

```text
product record vs canonical reference profile -> link / not link
```

右侧不是另一条来源商品，而是 `reference_entity` 的稳定 profile。

例如左侧：

```text
COL source VAL she-dang-jia
COL brand VAL ROLEX
COL title VAL 劳力士潜航者黑水鬼 全套 41mm
COL ref_title VAL REF 126610LN
COL source_sku VAL SKU SDJ123...
COL ocr VAL ...
```

右侧：

```text
COL brand VAL ROLEX
COL canonical_ref VAL REF 126610LN
COL series VAL Submariner
COL aliases VAL 126610 LN | 126610-LN
COL size VAL 41mm
```

模型任务：

```text
这个商品记录是否应链接到这个 reference entity？
```

好处：

1. 避免任意来源两两比较；
2. 右侧 entity 稳定，训练更容易；
3. reference 候选空间受品牌约束；
4. 一旦链接完成，跨源“同款”只需按 `reference_entity_id` join；
5. 不会因为 A-B、B-C 两条模型边产生错误传递，把 A-C 隐式合并。

---

## 14. Candidate Retrieval / Blocking

### 14.1 第一层：exact index

如果有高置信 canonical ref：

```sql
SELECT id
FROM reference_entity
WHERE brand_id = :brand_id
  AND canonical_ref = :canonical_ref;
```

这是最安全、最快的主路径。

### 14.2 第二层：同品牌 reference 近邻

只在 reference 不完整或 OCR 有噪声时：

```text
brand exact
AND
reference trigram / edit-distance / prefix candidate
```

例如最多返回 Top-10。

### 14.3 第三层：语义候选

reference 彻底缺失时才使用：

```text
brand + series + title text embedding + attribute retrieval
```

同样只用于候选生成，不直接 auto-link。

### 14.4 为什么不需要全局 ANN

由于业务已经有 `brand` 和 reference 语义，绝大多数候选检索应先缩到品牌内。千万级商品不等于千万级 reference entity；reference registry 的实体数通常远小于商品记录数。

因此架构成本主要在：

```text
reference extraction
OCR
少量 ambiguous candidate inference
```

而不是全量 pair inference。

---

## 15. 高精度决策策略

推荐规则优先于模型。

### 15.1 硬否决规则

```python
if high_conf_brand_conflict:
    return REJECT_CONFLICT

if high_conf_ref_a and high_conf_ref_b and ref_a != ref_b:
    return REJECT_CONFLICT

if trusted_structured_ref conflicts with OCR/title ref:
    return REVIEW
```

### 15.2 初期 AUTO_LINK 规则

第一版建议只允许以下路径自动链接：

```text
Path A
trusted structured reference
+ brand canonical exact
+ registry exact hit
+ no conflict

Path B
两个独立证据通道得到同一 canonical reference
例如 title parser + OCR
+ brand exact
+ no conflict
```

Ditto 只做：

```text
candidate ranking
review prioritization
ambiguous disambiguation
```

而**不独立创造 AUTO_LINK**。

这是最符合“绝不能误匹配”的上线方式。

### 15.3 后续开放 model-assisted AUTO_LINK

只有积累足够线上验证数据后，才可以增加：

```text
high-confidence extracted ref candidate
+ Ditto score >= t_high
+ candidate score margin >= delta
+ all hard veto = false
+ calibrated precision lower bound meets target
```

模型永远不能覆盖 reference 冲突。

---

## 16. 阈值校准：不能再用 F1

定义：

```text
accepted(t) = {x | p_match(x) >= t}
```

生产阈值应求：

```text
maximize coverage(t)
subject to precision_lower_bound(t) >= target
```

可以对每个阈值计算：

```text
TP_t
FP_t
precision_t = TP_t / (TP_t + FP_t)
```

并使用 Clopper-Pearson / Wilson 下界，而不是只看点估计。

### 16.1 几百条黄金标签的现实限制

如果自动接受样本里观察到 0 个 false positive，常用“rule of three”近似：

```text
95% 置信上界 error_rate ≈ 3 / n
```

所以：

```text
n = 300，0 FP
=> 只能大致支持 error < 1%
=> precision > 99%
```

要用纯统计样本证明 99.9% precision，约需要 3000 个 zero-FP 自动接受样本。

因此 Spec 所说“几百对黄金标签”足够：

```text
训练/微调
hard-negative 选择
模型排序
初期阈值探索
```

但不足以单独证明极端 precision。

所以**高精度保证必须主要来自 deterministic reference identity + abstention，而不是只靠模型概率。**

---

## 17. 黄金标签如何花最值

不要随机抽几百对。

建议初期 500–800 对，重点放在决策边界：

### 正样本

```text
同 reference
不同来源
字段缺失
大小写/分隔符差异
中英文标题差异
OCR 有少量噪声
```

### Hard negatives

至少占一半：

```text
同品牌同系列但 reference 不同
reference 编辑距离极小
同外观不同材质/尺寸/后缀
标题高度相似
图片高度相似
平台 SKU 被误识别为 reference
serial 被误识别为 reference
配件标题包含适配腕表 reference
```

这比普通随机 negatives 有价值得多。

### 数据切分

不要简单随机 pair split。

应该按 reference entity 切分：

```text
train refs
valid refs
test refs
```

尽量不重叠，才能测试新 reference / 长尾 reference 的泛化能力，避免同一 reference 的近重复样本泄漏到 train 和 test。

---

## 18. Watch-Ditto 输入设计

推荐统一序列：

```text
COL source VAL SOURCE lei_xiao_an
COL brand VAL BRAND ROLEX
COL title VAL 劳力士 潜航者 黑水鬼 41mm
COL ref_struct VAL REF 126610LN
COL ref_title VAL REF 126610LN
COL serial VAL SERIAL ********
COL source_sku VAL SKU LXA-****
COL series VAL SERIES SUBMARINER
COL size VAL SIZE 41MM
COL material VAL STEEL
COL ocr_ref VAL REF 126610LN
```

右侧 reference profile：

```text
COL brand VAL BRAND ROLEX
COL canonical_ref VAL REF 126610LN
COL series VAL SERIES SUBMARINER
COL aliases VAL 126610 LN | 126610-LN
COL size VAL SIZE 41MM
COL material VAL STEEL
```

注意：

```text
REF
SERIAL
SKU
```

必须让模型看到不同 tag。

这是对 Ditto Domain Knowledge 的直接迁移，而且比原版 ProductDKInjector 更贴合当前 domain。

---

## 19. 模型实现建议

不建议直接依赖 2020 年代码环境：

```text
Python 3.7
旧 transformers
apex fp16
```

建议用当前 HuggingFace/PyTorch 重写同一思想：

```python
class WatchDitto(nn.Module):
    def __init__(self, encoder_name):
        self.encoder = AutoModel.from_pretrained(encoder_name)
        self.classifier = nn.Linear(hidden_size, 2)

    def forward(self, input_ids, attention_mask, token_type_ids=None):
        out = self.encoder(
            input_ids=input_ids,
            attention_mask=attention_mask,
            token_type_ids=token_type_ids,
        )
        cls = out.last_hidden_state[:, 0]
        return self.classifier(cls)
```

### 19.1 不要照搬官方 light 版 padding

官方轻量 `dataset.py` 自己补 `0`，而模型调用没有显式 attention mask：

```python
x12 = [xi + [0]*(maxlen-len(xi)) ...]
enc = self.bert(x1)[0][:, 0, :]
```

生产实现应使用 tokenizer / DataCollator 的 `pad_token_id` 和 `attention_mask`，避免不同 backbone 的 padding token 语义问题。

### 19.2 训练 loss

初期可以仍用：

```text
CrossEntropyLoss
```

但采样策略应强制 hard negatives。

如果后续正负极不平衡，可考虑：

```text
class weighting
focal loss
pairwise ranking loss
```

不过优先级低于“样本构造正确”。

---

## 20. Inference Policy 伪代码

```python
def resolve_record(record):
    brand = brand_normalizer(record)
    if brand.confidence < BRAND_MIN:
        return UNRESOLVED("brand_uncertain")

    evidence = extract_reference_evidence(record, brand)
    evidence = classify_identifier_roles(evidence)

    strong_refs = unique_high_conf_reference(evidence)

    if strong_refs.has_conflict:
        return REVIEW("reference_conflict", evidence)

    if strong_refs.has_unique_ref:
        entity = registry.exact_lookup(brand.id, strong_refs.ref)
        if entity is not None:
            return AUTO_LINK(
                entity.id,
                reason="trusted_exact_reference",
                evidence=evidence,
            )

    candidates = registry.retrieve_candidates(
        brand_id=brand.id,
        reference_hints=evidence,
        title=record.title,
        topk=10,
    )

    if not candidates:
        return UNRESOLVED("no_candidate")

    scored = watch_ditto.score(record, candidates)
    best, second = top2(scored)

    if violates_hard_rule(record, best.entity, evidence):
        return REJECT_CONFLICT("hard_veto")

    # 第一阶段不让模型单独自动链接
    return REVIEW(
        reason="model_assisted_candidate",
        candidate=best.entity.id,
        ditto_score=best.score,
        margin=best.score - second.score if second else best.score,
    )
```

成熟后才考虑把最后一段改为：

```python
if calibrated_safe_accept(best, second, evidence):
    return AUTO_LINK(...)
```

---

## 21. 增量更新设计

需求是持续增量，不应每次全量重跑。

### 21.1 Record-level idempotency

定义：

```text
content_hash = hash(
    brand + title + ref_fields + description + image fingerprints
)
```

只有相关字段变化才重新解析。

### 21.2 Registry versioning

以下任何变化应触发 affected records 重算：

```text
brand normalization version
reference normalization version
extractor version
OCR version
registry alias version
policy version
model version
```

每个 resolution 记录都保存版本。

### 21.3 新 reference

如果记录抽出高置信 reference，但 registry 不存在：

```text
进入 reference onboarding queue
```

人工确认或从可信来源自动建立 reference entity。

不要因为 registry 不存在就模糊匹配到最近 reference。

---

## 22. 存储与计算栈建议

一个足够直接落地的组合：

```text
对象存储 / Parquet      原始抓取与历史快照
PostgreSQL              brand + canonical reference registry
ClickHouse              千万级 product/evidence/resolution 分析表（可选）
OpenSearch              同品牌 title/reference 模糊候选（可选）
PyTorch + HF            Watch-Ditto
PaddleOCR/其他 OCR      图片文字证据
Airflow/Dagster         增量批处理编排
```

如果当前数据工程简单，也可以先：

```text
PostgreSQL + Parquet + Python/Polars
```

等吞吐成为瓶颈再拆。

核心架构不依赖某个特定数据库。

---

## 23. 为什么这个方案能撑 100 万–1000 万

假设 1000 万商品记录。

错误做法：

```text
跨源 pairwise all-to-all
```

复杂度近似：

```text
O(N^2)
```

推荐架构：

```text
每条记录：
1 次 reference extraction
1 次 exact registry lookup
只有少量 ambiguous record 做 top-k candidate + Ditto
```

近似：

```text
O(N) + O(N_ambiguous * K)
```

且：

```text
K <= 10/20
N_ambiguous << N
```

最终跨来源结果通过：

```sql
JOIN reference_resolution r1
JOIN reference_resolution r2
ON r1.reference_entity_id = r2.reference_entity_id
```

不是再跑 matcher。

---

## 24. 线上监控指标

不要只看 F1。

### 一级指标

```text
AUTO_LINK audited false positives = 0
AUTO_LINK precision
AUTO_LINK coverage
REVIEW rate
UNRESOLVED rate
```

### reference extraction

```text
structured_ref_precision
 title_ref_precision
ocr_ref_precision
identifier_role_accuracy
reference_conflict_rate
```

### matcher

```text
precision@accepted
coverage@target_precision
hard-negative precision
candidate recall@K
calibration error
score margin distribution
```

### drift

按：

```text
source
brand
series
time window
extractor_version
```

分别看。

如果某品牌 reference grammar 改变，不能让整体均值掩盖局部事故。

---

## 25. 测试集必须专门包含的反例

上线前建议建立 regression suite：

### 编号角色错误

```text
平台 SKU == 另一个 reference 格式
serial 看起来像 reference
订单号出现在标题
```

### 近型号

```text
同品牌 + 同系列 + reference 差 1 位
同主体号 + 不同后缀
```

### 配件污染

```text
“适配 XXX reference 的表带”
“原装盒适用于 XXX”
```

不能把配件链接到腕表 reference。

### 多 reference 文本

```text
“老款 116xxx 升级新款 126xxx”
```

必须判断当前售卖主体，不是见到任意 reference 就选。

### 图片冲突

```text
标题 ref=A
OCR 保卡 ref=B
```

必须拒绝自动合并。

---

## 26. 与 Ditto 论文逐项映射

| Ditto 原组件 | 论文目的 | 当前需求改造 |
|---|---|---|
| Blocking | 减少 candidate pairs | 优先 exact `(brand, ref)` registry lookup，模糊情况才 top-k |
| Record Serialization | 兼容异构 schema | 保留来源字段并加统一 `brand/ref/sku/serial/ocr` 角色 |
| Transformer Matcher | 判断 pair 是否同实体 | 改成 record → canonical reference entity 消歧 |
| Domain Knowledge | 强调关键 span | 自定义 `[REF] [SERIAL] [SKU] [BRAND] [SIZE]` |
| Span Normalization | 合并等价表达 | 品牌级 reference grammar，不做激进全局字符串清洗 |
| Summarization | 长文本保留关键信息 | reference/brand 永久 pin，剩余文本再摘要 |
| Data Augmentation | 抗缺失/脏数据 | 只删除非关键属性；hard-negative 重点做近 reference |
| MixDA | 减少 augmentation 失真 | 可保留，但不是 MVP 必需 |
| Validation Threshold | F1 最优 | 改成 precision lower bound + abstention |
| Binary Output | match/no-match | 改为 AUTO_LINK / REVIEW / UNRESOLVED / REJECT |

---

## 27. MVP 落地顺序

### Phase 0：数据审计

抽样三源各 1000 条：

```text
reference 独立字段覆盖率
标题 reference 覆盖率
品牌覆盖率
serial/SKU 混淆率
图片可 OCR 比例
```

### Phase 1：不需要模型的高精度主路径

实现：

```text
brand normalization
reference grammar
identifier role classifier
Canonical Reference Registry
exact auto-link
conflict veto
```

先获得一条真正可用的 zero/near-zero false positive 主路径。

### Phase 2：候选检索

对 unresolved：

```text
同品牌 reference lexical top-k
title/series candidate retrieval
```

### Phase 3：Watch-Ditto

使用几百对高价值标签：

```text
record vs reference profile
hard-negative-heavy training
custom DK
```

第一阶段只辅助人工 review。

### Phase 4：校准与选择性自动化

积累足够审核数据后：

```text
score calibration
precision lower bound threshold
model-assisted AUTO_LINK
```

### Phase 5：图片增强

```text
OCR reference evidence
visual candidate ranking
visual conflict detection
```

不要一开始就上复杂 multimodal end-to-end matcher。

---

## 28. 可直接落地的目录结构

```text
matching/
  brand/
    normalize.py
    aliases.yaml

  reference/
    extract.py
    role_classifier.py
    normalize.py
    grammars/
      rolex.py
      omega.py
      patek.py
      ap.py
      cartier.py
      generic.py

  registry/
    models.py
    repository.py
    candidate_search.py

  ocr/
    pipeline.py
    reference_postprocess.py

  ditto/
    serialize.py
    knowledge.py
    dataset.py
    model.py
    train.py
    calibrate.py
    infer.py

  policy/
    decision.py
    veto.py
    thresholds.py

  jobs/
    ingest_incremental.py
    resolve_incremental.py
    backfill.py

  evaluation/
    hard_negative_suite.py
    precision_report.py
    drift_report.py
```

---

## 29. 最终建议

Ditto 值得借鉴，但不应以“一个 Transformer 二分类器”理解它。

最值得吸收的是四件事：

1. **schema-agnostic serialization**：三源字段不一致时仍能统一送模型；
2. **domain knowledge injection**：把 reference / serial / SKU 的编号角色显式告诉模型；
3. **hard-example augmentation**：对缺字段和近型号难例进行定向训练；
4. **blocking + matcher 分层**：大规模系统绝不能全量 pairwise。

但为了满足 Notion Spec，必须进一步做两个关键改造：

### 改造一：从 pairwise entity matching 变成 canonical reference entity linking

因为业务已经定义：

```text
same product = same reference
```

因此最安全的中心实体就是：

```text
(brand, canonical_reference)
```

三源商品只需解析到同一个 reference entity。

### 改造二：从 F1 二分类变成 precision-first selective decision

官方 Ditto 的 F1 最优 threshold 不符合需求。

生产决策必须是：

```text
硬 reference 一致 -> 自动
硬冲突 -> 否决
证据不足 -> abstain/review
模型 -> 只在模糊区提高覆盖率
```

如果只做一个最优先版本，我建议先落：

```text
Brand Normalizer
+ Reference Grammar / Canonicalizer
+ Identifier Role Classifier
+ Canonical Reference Registry
+ Exact Auto-Link
+ Conflict Veto
+ Review Queue
```

然后再把 Ditto 接在 `UNRESOLVED` 分支上。

这比“直接训练一个跨来源商品 matcher”更便宜、更可解释、更容易增量维护，也更符合“绝不能误匹配”的核心约束。

---

## 30. 参考资料

- 调研清单：`奢侈品文章调研.md`
- 需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》
- Ditto 论文：https://arxiv.org/abs/2004.00584
- Ditto HTML：https://arxiv.org/html/2004.00584v3
- Ditto 官方代码：https://github.com/megagonlabs/ditto
- 官方核心实现：`ditto_light/ditto.py`
- 数据集与 tokenizer：`ditto_light/dataset.py`
- Domain Knowledge：`ditto_light/knowledge.py`
- Summarization：`ditto_light/summarize.py`
- Augmentation：`ditto_light/augment.py`

### 与 c 既有调研的组合关系

如果后续要做完整系统，可以把已有几篇结论组合起来：

```text
DeepBlocker / retrieval     -> candidate generation
Ditto                       -> difficult candidate disambiguation
Confidence / Conformal      -> precision-first selective acceptance
TransClean                  -> post-match graph consistency audit
Ameli / multimodal papers   -> OCR/视觉辅助证据
```

但对于当前 Spec，最终合并主键仍应收口到：

```text
brand_id + canonical_reference
```

而不是任意模型相似度。
