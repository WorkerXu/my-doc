# Dual-Objective Fine-Tuning of BERT for Entity Matching

## 1. 调研对象与结论先行

- 论文：**Dual-Objective Fine-Tuning of BERT for Entity Matching**
- 作者：Ralph Peeters, Christian Bizer
- 会议/期刊：PVLDB 14(10), 2021
- 论文：https://www.vldb.org/pvldb/vol14/p1913-peeters.pdf
- 官方代码：https://github.com/wbsg-uni-mannheim/jointbert
- 对应需求：Notion「跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）」

### 核心判断

这篇论文最值得借鉴的不是“BERT 做二分类匹配”，而是它处理了一个与当前需求高度同构的问题：**一部分商品有共享标识符，可以确定同一实体；另一部分商品没有共享标识符，需要利用前一部分数据训练模型。**

但对于当前 Spec，不能直接照搬 JointBERT 做最终自动匹配，原因有三点：

1. 当前业务已经把“同一个商品”定义得非常严格：**同一 reference number / 型号**，因此 reference 本身是业务主键，而模型相似度不是业务真值。
2. 当前要求“绝对不能误匹配”，precision 优先到极致。任何纯概率二分类器都不应该拥有越过 reference 冲突的权限。
3. JointBERT 的辅助任务是“预测左右记录所属的实体 ID”，这是典型闭集多分类；在 100 万–1000 万条持续增量商品、reference 不断新增的场景下，直接维护一个巨大的实体类别头不现实，也不利于未见 reference 泛化。

因此建议落地为一个 **Reference-First / Precision-First 的多任务系统**：

> **规则与证据决定是否能自动合并；模型只负责从脏文本/图片中找到 reference、判断这个编号是否真的属于当前商品、发现冲突和给人工复核排序。**

最终自动合并条件保持简单且可审计：

```text
canonical_brand 相同
AND canonical_reference 相同
AND reference_role = OWN_REFERENCE
AND 无强冲突证据
AND 证据等级达到自动放行级别
=> 自动归入同一 reference entity
```

模型分数不能覆盖 `canonical_reference` 不一致。

---

## 2. JointBERT 原论文解决了什么问题

论文的实际出发点是 Web 商品实体匹配：大量商品 offer 来自不同站点，部分记录存在 GTIN、MPN 等共享 identifier，可以用这些 identifier 建立高质量匹配训练数据；但真实线上数据并不是每条记录都有 identifier，因此还需要一个 matcher 去判断无 identifier 记录。

这与当前三源腕表数据非常相似：

- 一部分记录有独立 reference 字段；
- 一部分 reference 埋在标题/描述；
- 一部分只有图片中的表背、保卡、吊牌文字；
- 同一个 reference 会出现很多不同描述；
- 同系列相邻 reference 的文字和图片又非常相似。

JointBERT 的关键思路是：如果训练数据中已经知道每条描述属于哪个真实商品实体，就不要只训练“pair 是否 match”一个任务，而应额外要求 BERT 学会“这条描述在说哪个实体”。这样 encoder 会更倾向捕捉具有实体区分力的 token，例如 model number，而不是只学习宽泛的语义相似度。

论文还专门分析了模型关注的词。其结论对当前腕表任务很重要：BERT 类模型会明显依赖 model number 等强区分属性；在 model number 缺失时，也会退回 model name 等次级线索。这说明“共享 identifier 作为弱监督，训练模型学习如何从自然语言描述里识别 identifier 相关证据”是可行路线。

---

## 3. 官方代码的技术实现拆解

### 3.1 输入结构

官方 `BertDatasetJoint` 读取的数据字段为：

```text
pair_id
sequence_left
sequence_right
label
label_multi1
label_multi2
```

其中：

- `label`：左右记录是否属于同一实体；
- `label_multi1`：左记录所属实体类别；
- `label_multi2`：右记录所属实体类别。

代码直接调用 HuggingFace tokenizer 对左右文本做 pair encoding，并使用 `truncation='longest_first'`。因此最终结构本质上就是：

```text
[CLS] left sequence [SEP] right sequence [SEP]
```

这一点非常适合实体匹配，因为 self-attention 可以直接做跨记录 token 交互，而不是先独立编码两个文本再计算一个粗粒度向量相似度。

### 3.2 模型结构

官方 `JointBertModelLogit` 的结构非常简单：

```text
BERT-base encoder
     |
     +--> binary head: 768 -> 1
     |
     +--> left entity head: 768 -> N_entity
     |
     +--> right entity head: 768 -> N_entity
```

三个 head 都直接读取同一个 `pooler_output`。

代码中的核心逻辑等价于：

```python
_, pooler_output = bert(input_ids, attention_mask, token_type_ids)

logits_binary = binary_head(pooler_output)
logits_left_entity = left_entity_head(pooler_output)
logits_right_entity = right_entity_head(pooler_output)
```

也就是说，JointBERT 没有引入复杂图网络或额外 attention 模块，价值主要来自**训练目标设计**，而不是模型堆叠。

### 3.3 损失函数

官方 TrainerJoint 把三个任务损失直接相加：

```text
L = L_binary + L_left_entity + L_right_entity
```

其中：

- pair match 使用 `binary_cross_entropy_with_logits`；
- 左右实体分类使用 NLL loss；
- 多分类任务还会按类别频率计算反比 class weight，缓解不同实体描述数量不平衡。

因此它真正实现的是：

> pair-level objective 负责“两个记录是不是一个实体”；entity-level objective 逼迫 encoder 对每条记录学到更强的实体身份特征。

### 3.4 训练工程

官方实现还包含：

- AMP 混合精度训练；
- gradient accumulation；
- gradient clipping；
- class imbalance 权重；
- BERT、RoBERTa、DistilBERT 等对照；
- 多个 WDC 商品数据集，包括 watches；
- LIME / word-class 分析用于解释模型为什么做出判断。

README 也明确提示部分实验需要 64GB+ RAM 和 GPU。这说明官方工程主要是科研复现代码，不应该原样搬到生产环境，但模型思想本身很轻，完全可以重写成现代 Transformers + PyTorch/Lightning 或纯 Trainer 服务。

---

## 4. JointBERT 对当前腕表需求真正有价值的地方

### 4.1 用显式 reference 数据自动生成大规模弱监督

当前只有几百对人工标注预算，但其实不必把几百对标签用于从零训练 matcher。

只要某批数据里 reference 是结构化且可信的，就天然得到实体 ID：

```text
entity_id = canonical_brand + ':' + canonical_reference
```

例如：

```text
ROLEX:126610LN
OMEGA:21030422001001
CARTIER:WSSA0018
```

同一 `entity_id` 下的不同来源记录就是正样本，不同 reference 就是负样本。

因此可以从三源中自动产生大量训练对：

```text
雷小安/126610LN  <-> 腕表之家/126610LN => positive
雷小安/126610LN  <-> 奢当家/126610LV => hard negative
```

几百条人工标签应主要用于：

- 检查 reference 抽取是否把平台 SKU 当成型号；
- 检查“兼容/适配某型号”的配件文本；
- 校准自动放行证据等级；
- 标注同系列邻近 reference hard negative；
- 建立最终 precision 验收集。

### 4.2 让模型学习“什么 token 真正决定 reference”

传统文本 embedding 很容易把：

```text
劳力士 潜航者 41 黑盘 黑圈 126610LN
劳力士 潜航者 41 绿圈 126610LV
```

编码得非常接近。

但业务上两者必须判为不同实体。

JointBERT 的辅助实体目标能强迫模型关注 `126610LN` 与 `126610LV` 这种决定身份的细粒度 token，而不是被“劳力士/潜航者/41”这些大量相同词主导。

这正是当前系统需要的 inductive bias。

### 4.3 对缺字段与脏标题有帮助

对于：

```text
ROLEX 126610 LN
Rolex 126610-LN
劳力士 126610ln
劳力士潜航者126610LN全套
```

规则规范化已经能解决大部分问题，但当标题更脏、夹杂平台货号、配件型号、系列名称时，模型可以做“编号角色校验”，判断候选字符串到底是不是当前商品自己的 reference。

---

## 5. 为什么不能直接把原版 JointBERT 当最终 matcher

### 5.1 闭集实体分类头无法覆盖持续新增 reference

原版辅助任务需要：

```text
Linear(768, num_entities)
```

如果实体数从几百扩展到几十万甚至百万：

- 输出层参数巨大；
- 新 reference 出现就要扩类别；
- 旧模型无法自然预测 unseen entity；
- 新旧类别分布持续漂移；
- 重新训练和部署成本高。

更重要的是，论文自己的结果也显示 JointBERT 对 **seen products** 很强，但在 **unseen products** 上不一定优于普通 BERT/RoBERTa。这恰好意味着它不能承担“长尾新 reference 自动判定”的唯一职责。

### 5.2 Pairwise 二分类不应覆盖 reference 硬冲突

假设模型给：

```text
P(match | 126610LN, 126610LV) = 0.999
```

只要 canonical reference 不一致，业务结论仍必须是：

```text
NOT MATCH
```

这不是阈值问题，而是业务定义问题。

### 5.3 1000 万级不能做全量 pairwise

如果每个来源上百万记录，任意两源直接笛卡尔积会不可接受。

当前定义反而给了更简单的工程路径：reference 一旦可靠抽取，只需建立倒排索引：

```text
(brand_id, canonical_reference) -> entity_id
```

主链路应是 O(N) / O(N log N) 的 key lookup，而不是 O(N²) matcher。

模型只处理非常小的 uncertain tail。

---

# 6. 推荐落地架构：Reference-First JointBERT

## 6.1 总体结构

```text
                 雷小安 / 腕表之家 / 奢当家
                           |
                           v
                  [1] Raw Offer Store
                           |
                           v
                 [2] Brand Normalizer
                           |
                           v
              [3] Reference Candidate Extractor
                 /          |             \
        structured field   title/text      OCR
                 \          |             /
                           v
              [4] Reference Role Validator
             规则 + 多任务 Transformer
                           |
                           v
               [5] Canonical Reference
                           |
               +-----------+-----------+
               |                       |
          evidence strong          uncertain/conflict
               |                       |
               v                       v
       [6] Exact-key Entity Index   Review / Audit Model
       (brand, canonical_ref)          |
               |                       |
               +-----------+-----------+
                           v
                  [7] Entity Link Store
```

**最重要的架构原则：模型永远在 Exact-key gate 之前做“提取和验证”，而不是在 gate 之后覆盖 key。**

---

## 6.2 数据表建议

### `raw_offer`

```sql
source              -- leixiaoan / xxxxx / shedangjia
source_offer_id
brand_raw
title_raw
description_raw
reference_raw
image_urls
crawl_time
payload_json
```

### `reference_evidence`

每一条“我为什么认为它是这个 reference”的证据都单独保存：

```sql
offer_id
candidate_raw
candidate_canonical
candidate_role       -- OWN_REFERENCE / COMPATIBLE_REFERENCE / PLATFORM_SKU / UNKNOWN
source_type          -- STRUCTURED / TITLE / DESCRIPTION / OCR / MODEL
span_start
span_end
extractor_version
confidence
rule_flags
created_at
```

### `reference_entity`

```sql
entity_id
brand_id
canonical_reference
status
created_at
updated_at
```

建议唯一约束：

```sql
UNIQUE(brand_id, canonical_reference)
```

### `offer_entity_link`

```sql
offer_id
entity_id
link_status          -- AUTO_ACCEPT / REVIEW_ACCEPT / REJECT / PENDING
link_reason
evidence_grade
model_version
rule_version
created_at
```

这种设计能让任何一次自动合并都可回溯：**是哪条原始文本、哪个 span、哪条规则、哪个模型版本导致的。** 对“不能误匹配”的系统，这是比单个 probability 更重要的生产能力。

---

# 7. Reference 规范化层必须先于模型

## 7.1 品牌级 canonicalizer

不能做一个全局暴力“去掉所有符号”的 normalize，因为不同品牌 reference 的点号、斜杠、后缀有时有语义。

建议：

```python
def canonicalize_reference(brand, raw):
    s = unicode_nfkc(raw).upper().strip()
    s = normalize_fullwidth(s)
    s = normalize_hyphens(s)
    return BRAND_RULES[brand].normalize(s)
```

品牌规则要显式版本化：

```text
ROLEX v3
OMEGA v2
CARTIER v5
...
```

并保留：

```text
raw = "126610-LN"
canonical = "126610LN"
rule_version = "rolex_ref_v3"
```

不要只存 canonical 结果，否则规则升级后无法审计。

## 7.2 reference dictionary

利用已经可信的结构化字段、品牌官网/型号库或人工确认数据维护：

```text
brand_id
canonical_reference
aliases
series
valid_pattern
active
```

抽取出的字符串如果能映射到已知 canonical reference，证据等级显著提高。

对于新 reference，可以先进入 `UNKNOWN_NEW_REFERENCE` 状态，不应因为模型相似度高就自动挂到已有 reference。

---

# 8. 把 JointBERT 改造成适合本需求的多任务模型

建议不再使用“百万实体 softmax”，而把辅助目标改成能泛化到新 reference 的任务。

## 8.1 Task A：Reference Span Extraction

对单条 title/description 做 token classification：

```text
B-REF / I-REF / O
```

示例：

```text
劳力士 潜航者 41 黑盘 126610LN 全套
                     B-REF
```

训练数据可以低成本生成：只要结构化 `reference_raw` 在标题中出现，就自动对齐 span 形成弱标签。

人工只需要处理：

- 标题中 reference 被切分；
- OCR 错字；
- 多个候选编号；
- 兼容型号；
- 平台 SKU 与品牌 reference 共存。

## 8.2 Task B：Reference Role Classification

这是当前需求里非常关键、但普通 matcher 很容易忽略的一层。

候选编号分类：

```text
OWN_REFERENCE
COMPATIBLE_REFERENCE
ACCESSORY_TARGET_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
UNKNOWN
```

例如：

```text
适配 Rolex 126610LN 表带
```

这里 `126610LN` 虽然是真实 reference，但它不是“当前售卖商品”的 reference。

如果没有 role classifier，exact match 反而可能把配件错误并入腕表实体。

## 8.3 Task C：Pairwise Same-Reference Head（只做辅助审计）

保留 JointBERT 原来的 binary pair head：

```text
P(same_reference | record_a, record_b)
```

但作用改为：

1. hard-negative 训练，强化 encoder 对相邻型号的敏感性；
2. 对 deterministic gate 已经判断的结果做异常审计；
3. 给人工 review queue 排序。

**它不直接产生 AUTO_ACCEPT。**

## 8.4 Task D：Brand / Series 辅助头

可选增加：

```text
brand classification
series classification
```

其意义不是直接决定同款，而是提高 encoder 对品牌内部 reference 结构的理解，并提供冲突信号。

## 8.5 推荐损失

可以沿用 JointBERT 的“共享 encoder + 多目标”思想：

```text
L = λ1 * L_span
  + λ2 * L_role
  + λ3 * L_pair
  + λ4 * L_brand_series
```

初始可设：

```text
λ_span = 1.0
λ_role = 1.0
λ_pair = 0.5
λ_brand_series = 0.2
```

但 production 里不要凭经验固定，应该围绕 **reference extraction precision / false-positive count** 调参，而不是只看总体 F1。

---

# 9. Precision-First 自动放行门控

建议定义明确证据等级。

## Grade A：直接自动放行

满足：

```text
1. canonical_brand 一致
2. 双方均有可信 STRUCTURED reference
3. canonical_reference 完全一致
4. reference 通过品牌格式/字典校验
5. 无冲突 candidate
```

这是最安全的一层。

## Grade B：可自动放行，但要求独立证据交叉确认

例如一方没有结构化 reference，但：

```text
title 抽取 = 126610LN
AND OCR/图片文字 = 126610LN
AND brand = ROLEX
AND role = OWN_REFERENCE
AND dictionary 中存在
AND 无其他冲突 reference
```

此时模型参与“抽取/role 判定”，但最后仍是 reference exact equality。

## Grade C：不自动合并

例如：

```text
只有标题模型抽取到一个 reference
没有独立字段/OCR/字典支持
```

即使模型置信度 0.9999，也建议进入 review 或等待后续数据补全。

## Grade D：强制拒绝/冲突

只要出现：

```text
structured_ref != title_ref
structured_ref != OCR_ref
左右 canonical_reference 不同
候选 role = COMPATIBLE_REFERENCE
出现两个互斥 OWN_REFERENCE
```

则不自动合并。

这种 evidence gate 比简单的：

```text
if model_score > 0.99: match
```

更符合 Spec 的安全目标。

---

# 10. Hard Negative 是这个项目成败的关键

随机负样本太简单，无法压低 false positive。

应该主动构造以下 hard negative：

### 10.1 同品牌同系列邻近 reference

```text
126610LN vs 126610LV
116610LN vs 126610LN
210.30.42.20.01.001 vs 210.30.42.20.03.001
```

### 10.2 规范化后只差 1–2 个字符

```text
edit_distance <= 2
same prefix
same length
```

### 10.3 配件/兼容场景

```text
“适配 126610LN 表带” vs “Rolex 126610LN 腕表”
```

### 10.4 平台 SKU 长得像 reference

从三个来源统计高频数字模式，专门生成：

```text
platform_sku / seller_sku / inventory_id
```

和真实 reference 的对比训练样本。

### 10.5 图像外观极近但 reference 不同

图片 embedding 可以用于找“视觉最像但 reference 不同”的负样本。

这是图片最有价值的用途之一：**帮助挖最危险的误匹配候选，而不是直接证明相同。**

---

# 11. 图片在架构里的正确位置

Spec 明确说有图片，但在腕表场景，视觉相似不能代替 reference。

建议图片只做三件事：

1. **OCR**：读表背刻字、保卡、吊牌上的 reference；
2. **hard-negative mining**：找外观极相似但 reference 不同的变体；
3. **conflict detection**：文字说 A 系列但图片明显属于另一个系列时，降低证据等级并转人工。

不建议：

```text
CLIP cosine > 0.98 => same product
```

因为同系列不同 reference 的外观可能非常接近，这与“绝不能误匹配”的目标冲突。

---

# 12. 千万级增量处理流程

## 12.1 在线增量主链路

每条新 offer 到达：

```text
1. normalize brand
2. collect structured reference candidates
3. regex / parser extract title candidates
4. optional OCR extract image candidates
5. canonicalize all candidates
6. role classification
7. conflict check
8. calculate evidence grade
9. if Grade A/B:
       upsert entity by (brand_id, canonical_reference)
       create offer_entity_link
   else:
       enqueue review / unresolved store
```

这个主流程不需要与历史 1000 万条记录逐对比较。

## 12.2 Entity Index

Redis / RocksDB / PostgreSQL / ClickHouse 都可以承担索引，核心 key 很简单：

```text
brand_id + canonical_reference
```

例如：

```text
ROLEX|126610LN -> entity_00012345
```

批量计算场景可以用 Spark/Flink；实时可用 Kafka + stream processor + transactional store。

推荐先从简单架构落地：

```text
Kafka/DB CDC
  -> Python/Go normalization service
  -> reference extraction service
  -> PostgreSQL entity index
  -> review queue
```

量级真的上到千万并且增量吞吐压力明显后，再拆成 Flink/Spark 流批架构，不必第一天过度设计。

---

# 13. 训练数据如何在几百人工标签约束下构造

## 13.1 自动正样本

可信显式 reference 相同：

```text
same brand + same canonical ref => positive
```

## 13.2 自动强负样本

可信显式 reference 不同：

```text
same brand + different canonical ref => negative
```

优先采样相似 reference，而不是随机跨品牌负样本。

## 13.3 人工标签怎么花

建议首批 300–500 条不要平均随机抽样，而分配为：

```text
35% 同系列邻近 reference hard negatives
25% 一个标题出现多个编号的 role 判断
15% 配件/表带/盒证兼容型号
10% OCR 与标题冲突
10% 新 reference / 字典外 reference
 5% 普通随机样本用于估计总体分布
```

目标不是让人工替代弱监督，而是覆盖**最危险的 false-positive 模式**。

---

# 14. 评测方式必须改成 Precision-first

不能把总体 F1 作为发布门槛。

至少分层统计：

```text
Auto-accept precision
False positive absolute count
Coverage / auto-link rate
Review rate
Reference extraction precision
Reference role precision
Conflict detection recall
Unseen-reference precision
```

尤其要单独维护以下切片：

```text
same-series-neighbor
single-char-reference-diff
accessory-compatible
missing-reference-field
ocr-only
new-reference
multi-reference-title
source-pair: A-B / A-C / B-C
```

发布规则建议：

```text
只要 golden hard-negative set 出现新增 FP，阻断发布。
```

由于业务允许漏匹配，因此可以宁可降低 coverage，也不要为提升 recall 放松 deterministic gate。

---

# 15. 可直接落地的 MVP

## Phase 1：不训练大模型也能上线的版本

1. 建 `brand_alias` 与 `reference_dictionary`；
2. 实现品牌级 reference normalizer；
3. 三源结构化 reference 字段统一；
4. 标题 regex/parser 抽候选；
5. 保存 evidence lineage；
6. `(brand, canonical_reference)` exact-key entity index；
7. structured/title 冲突一律转人工；
8. 建 hard-negative golden set。

这一阶段已经能完成大量高精度匹配。

## Phase 2：引入 JointBERT 思想

训练多任务模型：

```text
span extraction + role classification + pair audit
```

训练数据主要来自 Phase 1 的高置信 reference 弱标签。

模型只提升：

- 标题埋 reference 的抽取率；
- 多编号文本 role 判断；
- hard negative 识别；
- 人工 review 排序。

## Phase 3：图片与增量反馈

加入：

- OCR；
- 图片 hard-negative mining；
- review 结果回流；
- 按品牌/来源做 drift 监控；
- extractor / canonicalizer 版本化回放。

---

# 16. 推荐服务接口

## `/extract-reference`

输入：

```json
{
  "brand": "劳力士",
  "title": "劳力士潜航者41黑盘126610LN全套",
  "description": "...",
  "structured_reference": null,
  "ocr_texts": ["ROLEX 126610LN"]
}
```

输出：

```json
{
  "brand_id": "ROLEX",
  "candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "role": "OWN_REFERENCE",
      "sources": ["TITLE", "OCR"],
      "evidence_grade": "B"
    }
  ],
  "conflicts": []
}
```

## `/resolve-offer`

输出不能只给一个 score，而应给可审计决策：

```json
{
  "decision": "AUTO_ACCEPT",
  "entity_key": "ROLEX|126610LN",
  "reason_codes": [
    "BRAND_EXACT",
    "REFERENCE_EXACT",
    "TITLE_OCR_AGREE",
    "ROLE_OWN_REFERENCE",
    "NO_CONFLICT"
  ]
}
```

这比黑盒：

```json
{"match_probability": 0.9974}
```

更适合生产和事故追责。

---

# 17. 与原始 JointBERT 的映射关系

| JointBERT 原设计 | 当前需求改造 |
|---|---|
| BERT pair encoder | 保留，可用于疑难 pair 审计 |
| binary match head | 保留但不拥有自动合并权限 |
| left entity-ID head | 改为 reference span / role / brand-series auxiliary task |
| right entity-ID head | 同上 |
| entity ID 来自共享 identifier | 直接使用 canonical reference 生成弱监督 |
| 多分类帮助 encoder 学实体特征 | 多任务帮助 encoder 学 reference 身份 token 与角色 |
| 预测 pair 是否 match | 最终由 canonical reference exact gate 收口 |
| 适合 seen entities | 对 seen reference 用于增强鲁棒性 |
| unseen entity 较弱 | 新 reference 依赖抽取+规范化，不依赖闭集 softmax |

---

# 18. 最终推荐方案

如果只选一个从这篇论文演化出的可落地方案，我建议：

> **不要构建“千万级 Pairwise 商品匹配器”，而要构建“Reference Resolution Platform”。**

系统的核心对象不是 pair score，而是：

```text
Reference Evidence
      -> Canonical Reference
      -> Reference Entity
      -> Offer Link
```

JointBERT 则成为 Reference Resolution Platform 内部的一个可替换组件：

```text
脏文本理解器 + reference role validator + hard-case auditor
```

这样做同时满足：

- **规模**：主链路是 key-based lookup，不做千万级笛卡尔积；
- **字段稀疏**：模型帮助从 title / description / OCR 中补 reference；
- **高 precision**：模型不能覆盖 reference 冲突，自动合并有硬规则；
- **图片可用**：OCR/反证，而非视觉相似直接判同款；
- **少量人工标签**：显式 reference 自动生成弱监督，人工集中标最危险边界；
- **持续增量**：新 reference 不需要扩百万类 softmax，只需新增 dictionary/index key；
- **可审计**：每条 link 都能还原原始 evidence 和决策理由。

对当前 Spec 来说，这是比“直接 fine-tune BERT 做 match/no-match”更稳、更简单，也更接近可以直接上线的架构。

---

## 19. 实施优先级

按投入产出比，推荐顺序：

```text
P0  brand/reference canonicalization
P0  evidence lineage + exact entity index
P0  conflict gate
P1  hard-negative golden set
P1  title reference span extractor
P1  reference role classifier
P2  OCR cross evidence
P2  JointBERT-style pair audit head
P2  active learning / review feedback
P3  image embedding hard-negative mining
```

**不要先做：** 全量向量库商品相似搜索 + 大模型 pair 判定 + 单阈值自动合并。它们可以做候选探索，但不适合作为该 Spec 的主判定链路。

---

## 20. 参考源码位置

官方仓库中最值得直接阅读的文件：

```text
src/productbert/model/model.py
src/productbert/model/loss.py
src/productbert/dataset/datasets.py
src/productbert/trainer/trainer.py
src/productbert/data_loader/data_collators.py
```

其中 `JointBertModelLogit` 展示了三头共享 BERT encoder 的最小实现；`TrainerJoint` 明确实现了 binary loss + 两个 entity classification loss 的联合训练；`BertDatasetJoint` 则展示了 pair text 与左右 entity label 的数据组织方式。
