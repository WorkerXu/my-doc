# Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration：Selective EM + CMRL 在跨源腕表 Reference 匹配中的落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择此前 `a` 目录尚未分析的论文：

**Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration**

- 论文：<https://www.sciencedirect.com/science/article/pii/S1474034625004318>
- DOI：<https://doi.org/10.1016/j.aei.2025.103538>
- 官方代码：<https://github.com/quickhdsdc/LLM4EntityMatching>
- 需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

当前 Spec 的核心约束是：

1. 雷小安、腕表之家、奢当家三源，100 万～1000 万级商品，持续增量；
2. 业务定义的“同一个商品”不是同一块物理表，而是 **同一品牌 Reference Number / 型号**；
3. Reference 字段高度稀疏，可能来自结构化字段、标题、描述、图片 OCR；
4. **precision 极端优先，错误合并不可接受，允许大量拒识/漏匹配**；
5. 可以人工标注几百对黄金样本。

这篇论文最值得迁移的点不是“换成 Mistral 就更准”，而是把传统的：

```text
(query, candidate_1) -> match / non-match
(query, candidate_2) -> match / non-match
...
```

改成：

```text
query -> [candidate_1, candidate_2, ... candidate_K]
              ↓ 同时竞争
        只选最可能的候选
```

也就是 **Selective / Listwise Entity Matching**。它专门针对 blocking 后出现大量“非常像但实际上不同”的 hard negative，正好对应腕表场景中的：

```text
126610LN  vs  126610LV
116500LN  vs  126500LN
5711/1A   vs  5712/1A
平台 SKU  vs  品牌 Reference
主表型号  vs  “适配某型号”的表带/配件标题
OCR 中 0/O、1/I、8/B 混淆
```

论文重新编译 benchmark 后发现：即使是原 benchmark 上表现最好的 fine-tuned LLM pairwise matcher，面对更多 hard negative 时平均 precision 仍下降约 5.8%，相对 false-positive rate 上升约 67.6%。论文的 Mistral4SelectEM 则通过候选集竞争 + CMRL，把 MRRP 相比原始 blocker 平均提高约 9.6%，相比最佳 pairwise matcher 提高约 12.4%，同时论文报告单 query 推理约 1.2 秒。

**但这篇论文不能原样用于当前 Spec。** 原因是它本质上仍是“候选集里选一个最好的”，而我们的业务允许、且应该大量输出：

```text
ABSTAIN / UNKNOWN / NEED_REVIEW
```

所以我建议的生产方案是：

> **不要直接做商品记录 ↔ 商品记录的 Pairwise Merge，而是把每条商品记录链接到一个全局 Canonical Reference Entity。Selective EM 只负责在少量候选 Reference 中做竞争排序；最终自动落库必须同时经过 Reference 硬规则、冲突否决和可拒识阈值。**

最终关系应当是：

```text
雷小安商品 ─┐
腕表之家商品 ├──> Canonical Reference Entity: ROLEX / 126610LN
奢当家商品 ─┘
```

而不是：

```text
A商品 <-> B商品 <-> C商品
```

前一种设计天然避免一条错误 pairwise 边通过传递关系污染整个簇。

---

## 1. 论文到底解决了什么问题

## 1.1 Pairwise EM 的结构性缺陷

传统 Entity Matching 常见架构是：

```text
Source A Records
      |
      v
Blocking / Candidate Retrieval
      |
      v
(query, candidate_i)
      |
      v
Binary Matcher
      |
      +--> match
      +--> non-match
```

问题在于每一个 candidate 都是 **独立判断**。

假设 query 是：

```text
Rolex Submariner Date 126610LN 黑水鬼
```

Blocking 后得到：

```text
1. Rolex 126610LN
2. Rolex 126610LV
3. Rolex 116610LN
4. Rolex 126613LN
5. Rolex 124060
```

Pairwise classifier 在判断 `query vs 126610LV` 时看不到 `126610LN` 这个更合理的竞争项，因此可能因为品牌、系列、尺寸、材质、标题语义都高度相似而输出高 match probability。

对于普通商品搜索这可能只是排序误差；对本项目则是不可接受的错误合并。

Selective EM 的核心变化是：

```text
不是问：candidate_i 像不像 query？
而是问：在这一组相似候选里，谁最能解释 query？
```

这使模型的训练目标从“区分随机负例”变成“拉开真匹配与最难近邻之间的差距”。

---

## 1.2 论文如何重新制造 Hard Negative Benchmark

官方代码 `EM_data_convert.py` 的流程非常清楚：

```text
原始 EM 数据
   |
   +--> 保留 query 与已知真匹配
   |
   +--> 使用 Linq-Embed-Mistral 编码目标表
   |
   +--> FAISS IndexFlatIP
   |
   +--> 对每个 query 检索 Top-20
   |
   +--> 截取 Top-10 作为候选集
   |
   +--> 真匹配标 1，其余候选标 0
```

代码中使用：

```python
faiss.IndexFlatIP(...)
```

并且 embedding 在进入索引前做 L2 normalize，因此 inner product 实际对应 cosine similarity。

README 中说明最终 benchmark 每个 query 固定 10 个候选，大部分只有一个真匹配。

这一步非常重要，因为它不是随机采负例，而是让 blocker 主动找到“看起来最像”的候选。也就是把训练/评估压力集中在生产中最危险的区域。

### 一个必须注意的研究代码细节

`EM_data_convert.py` 中如果真实匹配没有进入 Top-10，会把最后一个候选强行替换成 ground truth：

```python
if 1 not in label_k10:
    candidates_k10[-1] = truth_list[0]
    label_k10[-1] = 1
```

这是为了构造“每个 query 都有正例”的实验 benchmark，**生产系统绝对不能照抄**。

在真实系统中：

```text
Top-K 没召回正确 Reference
```

必须被当作 blocker miss / abstain，而不能假装候选集中一定存在真值。

这也是当前项目需要额外设计 Reject Option 的原因之一。

---

## 2. Mistral4SelectEM 的真实代码架构

论文官方代码不是把 query + 10 candidates 拼成长 prompt 让生成式 LLM 输出编号，而是把 Mistral 当成 **共享权重的 embedding encoder**。

核心代码位于：

```text
tasks/SelectiveEntityMatching/selection_llm/
├── data_preprocessor.py
├── model_loader.py
├── model_finetuner.py
├── modelling_llama.py
└── evaluater.py
```

整体结构可以概括为：

```text
                   ┌──────────────────────┐
Query Text ───────>│                      │──> q embedding
                   │ Shared Mistral       │
Candidate 1 ──────>│ Encoder + LoRA       │──> c1 embedding
Candidate 2 ──────>│                      │──> c2 embedding
...                │                      │
Candidate K ──────>│                      │──> ck embedding
                   └──────────────────────┘
                              |
                              v
                     cosine / dot product
                              |
                [s1, s2, ... , sk]
                              |
                CE + CMRL hard-negative loss
```

所谓 Siamese 不是两套模型，而是 **Query 与 Candidate 共用同一套 Transformer 权重**。

---

## 2.1 Embedding：最后一个有效 Token + L2 Normalize

`modelling_llama.py` 中的 `EntityRetrieverMistral`：

1. Query 通过 `MistralModel`；
2. 根据 attention mask 取最后一个有效 token 的 hidden state；
3. 做 L2 normalize；
4. 所有 candidates 扁平化后批量通过同一个 Mistral；
5. 同样 last-token pooling + normalize；
6. Query 与所有 Candidate 做一次矩阵乘法得到相似度。

等价于：

```text
q = normalize(Encoder(query))
c_i = normalize(Encoder(candidate_i))
s_i = q · c_i
```

因为向量归一化，所以 `q · c_i` 即 cosine similarity。

代码随后：

```python
softmax_similarities = softmax(similarities, dim=1)
predicted_labels = argmax(softmax_similarities, dim=1)
```

这里已经体现出 listwise 的关键：一个 query 的候选会在同一个 score vector 内竞争。

---

## 2.2 CMRL：不只要求正例第一，还要求和最危险负例拉开 Margin

代码中 `loss_type == 'CMRL'` 时，损失由两部分组成：

```text
Loss = (1 - alpha) * CrossEntropy
     + alpha * ContrastiveMarginRankingLoss
```

默认实验参数在 `EM_training_contrastive.py` 中给出：

```text
num_neg = 9
margin = 0.5
alpha = 0.6
top_k = 1
LoRA r = 64
LoRA alpha = 64
learning rate = 2e-4
batch size = 32
train epochs = 10
```

其中 CE 的目标是让 true candidate 在整个 candidate set 中得分最高。

CMRL 的核心代码逻辑为：

```python
positive_scores = similarities[i, labels[i] == 1]
negative_scores = similarities[i, labels[i] == 0]
hard_negatives = topk(negative_scores, top_k)

difference = hard_negative - positive + margin
loss += weighted_relu(difference)
```

如果简化成一个正例、一个最难负例：

```text
L_margin = max(0, s_hard_negative - s_positive + margin)
```

也就是要求：

```text
s_positive >= s_hard_negative + margin
```

如果只让 positive 比 random negative 高，很容易学会“Rolex 和 Omega 不同”；CMRL 强迫模型学习的是：

```text
126610LN 为什么必须比 126610LV 更靠近 query
```

这正是当前腕表 Reference matching 的核心难点。

代码还对多个 hard negative 计算 softmax weight，让更违反 margin 的负例权重更高。

---

## 2.3 训练工程：4-bit Base Model + LoRA

`model_finetuner.py` 采用 PEFT/LoRA 路径：

```text
prepare_model_for_kbit_training
        ↓
LoraConfig(target_modules="all-linear")
        ↓
get_peft_model
        ↓
gradient checkpointing
        ↓
SFTTrainer
```

优化器配置为：

```text
paged_adamw_32bit
fp16
cosine lr scheduler
gradient accumulation = 8
warmup_ratio = 0.03
```

这个工程选择对当前项目很有价值：几百到几千条高质量 hard-negative 训练样本时，没有必要 full fine-tune 7B 模型，可以只训练 LoRA adapter。

更重要的是：生产推理阶段 `evaluater.py` 会把 LoRA merge 回 base model，再作为 feature extractor 运行，因此在线服务不需要额外维护“base + adapter 两次前向”。

---

## 2.4 论文评估指标 MRRP 为什么比 F1 更接近生产需求

论文提出 Mean Reciprocal Rank with Penalty（MRRP）。

基本思想：

```text
先看真匹配排在多前面；
再对 False Positive 和漏掉的 Positive 额外惩罚。
```

Penalty 形式是：

```text
Penalty = 1 / (1 + FP + FN)
```

这比只看 pairwise F1 更合理，因为 blocker 的真实输出本来就是一组候选，Matcher 的价值应该是：

```text
是否把正确候选推到第一，同时不错误提升干扰项
```

但当前 Spec 对 FP 的容忍比论文还低，所以 MRRP 只能作为模型研发指标，**不能作为生产发布的唯一指标**。

生产必须额外看：

```text
AUTO_MATCH Precision
AUTO_MATCH False Positive Count
Coverage / Abstain Rate
Hard-negative Slice Precision
```

---

## 2.5 官方代码有一个值得生产化时修正的实现点

论文叙述强调 query 与整个 candidate set 同时比较；`modelling_llama.py` 本身也支持一次输入完整 candidate tensor。

但 `evaluater.py` 的 retrieval 实现为了 rerank，会反复随机抽取 `num_neg + 1` 个 candidates，再迭代选择。

这更像研究实验脚本，不适合生产：

- 随机抽样会引入非确定性；
- 同一个 query 多次运行可能顺序不同；
- 不利于审计；
- 无法直接把 top1 / top2 margin 当成稳定安全信号。

当前项目应直接实现 **deterministic full-set scoring**：

```python
scores = selector(query, candidates)       # 一次得到 K 个分数
order = argsort(scores, descending=True)
top1 = order[0]
top2 = order[1]
margin = scores[top1] - scores[top2]
```

这比复用官方 evaluator 更适合线上。

---

# 3. 对当前 Spec 最重要的改造：不要匹配“商品”，要匹配“Reference Entity”

论文的 benchmark 真值是“两个记录是否同一个实体”。

我们的业务已经给出更强的定义：

> 同一个商品 = 同一个 Reference Number / 型号。

既然业务真值本身就是 Reference，最优架构就不应该把 Reference 当成普通文本特征，而应该把它提升成一等实体。

推荐建立：

```text
Canonical Reference Registry
```

例如：

```json
{
  "reference_entity_id": "rolex:126610ln",
  "brand": "Rolex",
  "reference_canonical": "126610LN",
  "reference_aliases": [
    "126610LN",
    "126610 LN",
    "M126610LN-0001"
  ],
  "family": "Submariner Date",
  "product_type": "watch"
}
```

所有三源记录最后都只产生：

```text
record_id -> reference_entity_id
```

是否“同商品”变成一个非常简单、可解释的下游判断：

```python
same_product = record_a.reference_entity_id == record_b.reference_entity_id
```

而不是再跑一次 pairwise model。

---

# 4. 推荐的生产架构

## 4.1 总体数据流

```text
┌─────────────────────────────────────────────────────────────┐
│  雷小安 / 腕表之家 / 奢当家 Raw Records                      │
└──────────────────────────┬──────────────────────────────────┘
                           v
┌─────────────────────────────────────────────────────────────┐
│  1. Normalize                                                │
│  品牌、标题、标点、全半角、大小写、商品类型、图片 URL          │
└──────────────────────────┬──────────────────────────────────┘
                           v
┌─────────────────────────────────────────────────────────────┐
│  2. Reference Evidence Extraction                            │
│  structured field / title / description / OCR                │
│  每个候选保留来源、span、role、confidence                      │
└──────────────────────────┬──────────────────────────────────┘
                           v
┌─────────────────────────────────────────────────────────────┐
│  3. Canonicalize + Role Classification                       │
│  品牌 reference / 平台 SKU / 店铺货号 / 配件适配型号 / unknown │
└──────────────────────────┬──────────────────────────────────┘
                           v
              ┌──────── exact unique ref? ────────┐
              │ yes                               │ no/ambiguous
              v                                   v
┌───────────────────────────────┐   ┌──────────────────────────────┐
│ 4A. Strict Reference Lookup   │   │ 4B. Candidate Blocking      │
│ brand + canonical_ref         │   │ brand/family/ref-string/ANN │
└──────────────┬────────────────┘   └──────────────┬───────────────┘
               │                                   v
               │                    ┌──────────────────────────────┐
               │                    │ 5. Selective Reference Ranker│
               │                    │ CMRL / listwise Top-K        │
               │                    └──────────────┬───────────────┘
               │                                   v
               └──────────────────────┬────────────────────────────┘
                                      v
┌─────────────────────────────────────────────────────────────┐
│  6. Precision Safety Gate                                   │
│  hard conflict veto + score threshold + top1/top2 margin     │
│  + evidence consensus + calibrated reject                    │
└──────────────────────────┬──────────────────────────────────┘
                           v
              ┌────────────┼────────────┐
              v            v            v
         AUTO_MATCH     REVIEW       REJECT
              |
              v
      product_reference_link
```

这里 **模型不是最终裁判**。最终裁判是 Safety Gate。

---

# 5. Reference Evidence 层：先把“型号字符串”变成可审计证据

不要只保存：

```text
extracted_reference = 126610LN
```

应该保存 evidence event：

```json
{
  "record_id": 123,
  "channel": "title",
  "raw_value": "M126610LN-0001",
  "canonical_value": "126610LN",
  "role": "brand_reference",
  "brand": "Rolex",
  "span": [18, 32],
  "extractor": "ref-regex-v3",
  "confidence": 0.997
}
```

如果来自图片：

```json
{
  "channel": "image_ocr",
  "image_type": "warranty_card",
  "raw_value": "126610L N",
  "canonical_value": "126610LN",
  "role": "brand_reference",
  "ocr_engine": "xxx",
  "confidence": 0.94
}
```

### 为什么 Evidence 必须一等存储

因为当前项目最关键的问题不是“模型输出了什么”，而是：

```text
为什么这条记录被链接到了这个 reference？
```

未来出现误匹配时，必须能回答：

```text
- 是标题 regex 抽错？
- OCR 把 LV 读成 LN？
- 把平台 SKU 当成 reference？
- 标题其实在说“适配 126610LN”？
- Candidate ranker 在两个 sibling refs 之间选错？
```

只有保存 evidence，才能回放、修规则和生成下一轮 hard negatives。

---

# 6. Canonicalization：Reference 不应该做普通 fuzzy string equality

建议 canonicalize 只做 **格式归一化**，不做语义猜测。

允许的操作例如：

```text
unicode normalize
uppercase
trim
统一破折号/空格
去品牌明确已知的包装前缀/后缀
按品牌规则生成 alias
```

例如：

```text
"126610 ln"          -> "126610LN"
"M126610LN-0001"     -> alias -> "126610LN"
```

但下面这种不能自动等价：

```text
126610LN != 126610LV
116500LN != 126500LN
```

哪怕编辑距离只有 1。

### 品牌级 Normalizer

Reference 形态高度品牌相关，建议：

```python
class ReferenceNormalizer:
    def normalize(self, brand, raw_ref) -> NormalizedReference:
        ...
```

内部按品牌路由：

```text
RolexNormalizer
OmegaNormalizer
PatekNormalizer
APNormalizer
CartierNormalizer
GenericNormalizer
```

Generic normalizer 要非常保守。

---

# 7. Blocking：在千万级数据下，不要做三源笛卡尔积

100 万～1000 万记录如果做 source A × source B pairwise comparison，复杂度完全没有必要。

Reference Entity 架构下，Blocking 的对象不是其他所有商品，而是一个相对小得多的 **Reference Registry**。

推荐 candidate 生成顺序：

## Tier 0：Exact Dictionary Lookup

```text
brand_id + canonical_reference
```

数据库唯一索引：

```sql
UNIQUE (brand_id, reference_canonical)
```

如果高可信结构化字段直接得到唯一 reference，根本不需要 ANN。

## Tier 1：Reference String Candidate

对于抽到疑似 reference、但字符有 OCR/格式不确定时，在 **同品牌** reference 库中做：

```text
prefix / ngram / edit distance / confusion-aware search
```

但这只是召回，不是自动匹配。

## Tier 2：Semantic ANN

只有没有稳定 reference token 时才使用 embedding：

```text
brand + product_type + family/block key
        ↓
reference profile embedding ANN Top-K
```

可以使用：

```text
FAISS / HNSW / Qdrant / pgvector
```

候选 K 建议初始实验 10～30，而不是全库。

### 一个关键原则

**Embedding 只负责 recall，不负责真值。**

Embedding 越相似，只代表值得比较，不代表 Reference 一致。

---

# 8. Selective Ranker 应该从“Record ↔ Record”改成“Record ↔ Reference Entity”

论文输入是：

```text
query record
candidate record 1
candidate record 2
...
```

当前项目建议改为：

```text
query = 当前商品的证据文本
candidate = Canonical Reference Profile
```

一个 Reference Profile 可以序列化为：

```text
brand: Rolex
reference: 126610LN
family: Submariner Date
aliases: M126610LN-0001 | 126610 LN
known tokens: submariner | date | black | 41mm
```

Query 可以序列化成结构化文本：

```text
source: leixiaoan
brand: Rolex
title: 劳力士潜航者型黑水鬼 41mm 全套 126610LN
structured_reference: null
title_reference_candidates: 126610LN
ocr_reference_candidates: 126610LN
product_type: watch
```

然后沿用论文的 shared encoder：

```text
q = Encoder(query evidence)
r_i = Encoder(reference profile_i)
s_i = cosine(q, r_i)
```

Reference profile embedding 可以预计算，所以在线只需要：

```text
1 次 query encode + K 次向量 dot product
```

如果 selector 本身仍需要 LoRA 后重新编码 reference，部署新版本时批量重建 reference embedding 即可。

这比每次拿 10 条商品记录都跑完整 Transformer 更便宜，也更稳定。

---

# 9. Hard Negative 设计：这是 CMRL 能否有价值的关键

不要随机采 negative。

训练集应该故意制造以下负例：

## HN1：同品牌、同系列、Reference 只差 1～2 个字符

```text
126610LN vs 126610LV
5711/1A vs 5712/1A
```

## HN2：上一代 / 下一代 Reference

```text
116610LN vs 126610LN
116500LN vs 126500LN
```

## HN3：视觉高度相似但 Reference 不同

尤其同系列不同尺寸、材质、表圈颜色。

## HN4：平台 SKU / 店铺编号伪装成型号

```text
DJ2025123408
A126610LN01
```

## HN5：配件兼容型号

标题：

```text
Rolex 126610LN 适配表带 / 表壳 / 贴膜 / 盒证
```

这里标题真实出现 `126610LN`，但当前售卖物不是该腕表。

## HN6：OCR Confusable

```text
0 / O
1 / I / l
8 / B
5 / S
LN / LV
```

## HN7：跨源脏字段冲突

```text
结构化字段 = 126610LV
标题 = 126610LN
OCR = 126610LV
```

这种样本不应该训练模型“猜谁对”，生产中应直接进入 conflict reject/review。

### 训练样本结构

```json
{
  "query_record": "...",
  "positive_reference": "rolex:126610ln",
  "negative_references": [
    "rolex:126610lv",
    "rolex:116610ln",
    "rolex:124060",
    "..."
  ]
}
```

几百条人工黄金标签最应该花在 **边界 hard cases**，而不是随机 easy pairs。

---

# 10. Production Safety Gate：论文最缺、当前项目最需要的一层

Mistral4SelectEM 官方实现最后是：

```python
predicted = argmax(scores)
```

这不适合“绝对不能误匹配”。

生产必须加：

```text
argmax != accept
```

推荐决策：

```python
best = scores[0]
second = scores[1]
margin = best - second

if hard_conflict:
    return REJECT

if not reference_evidence_valid:
    return REVIEW

if best < T_SCORE:
    return REVIEW

if margin < T_MARGIN:
    return REVIEW

if calibrated_fp_risk > EPSILON:
    return REVIEW

return AUTO_MATCH
```

其中 `T_SCORE`、`T_MARGIN` 不应该凭感觉写死，必须用黄金验证集按目标 precision 反推。

## 10.1 必须存在的 Hard Veto

下列任一条件满足，模型分再高也不能自动匹配：

```text
brand 冲突
明确 reference 冲突
商品类型是 accessory / strap / box / card 等非 watch 主体
同一记录存在两个互斥的高可信 reference
候选 reference 仅出现在“适配/兼容/for xxx”语境
reference role 被判定为 platform_sku / merchant_sku
```

### 推荐状态机

```text
AUTO_MATCH
  - 可直接写 product_reference_link

REVIEW
  - 有 plausible candidate，但证据不足或 margin 太小

REJECT
  - 明确冲突，或不是可匹配主商品

UNRESOLVED
  - 没有召回合理候选
```

REVIEW/UNRESOLVED 不是失败，而是 precision-first 系统设计的一部分。

---

# 11. 一套可以直接落库的数据模型

## 11.1 `product_record`

```sql
CREATE TABLE product_record (
    id                  BIGINT PRIMARY KEY,
    source              VARCHAR(32) NOT NULL,
    source_item_id      VARCHAR(128) NOT NULL,
    brand_id            BIGINT,
    title               TEXT,
    product_type        VARCHAR(64),
    raw_payload         JSONB,
    content_hash        VARCHAR(64),
    updated_at          TIMESTAMPTZ NOT NULL,
    UNIQUE(source, source_item_id)
);
```

## 11.2 `canonical_reference`

```sql
CREATE TABLE canonical_reference (
    id                  BIGSERIAL PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    reference_canonical VARCHAR(128) NOT NULL,
    family              VARCHAR(256),
    product_type        VARCHAR(64) DEFAULT 'watch',
    aliases             JSONB,
    metadata            JSONB,
    UNIQUE(brand_id, reference_canonical)
);
```

## 11.3 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    id                  BIGSERIAL PRIMARY KEY,
    record_id           BIGINT NOT NULL,
    channel             VARCHAR(32) NOT NULL,
    raw_value           VARCHAR(256),
    canonical_value     VARCHAR(256),
    role                VARCHAR(64),
    confidence          DOUBLE PRECISION,
    context             TEXT,
    extractor_version   VARCHAR(64),
    metadata            JSONB,
    created_at          TIMESTAMPTZ DEFAULT now()
);
```

## 11.4 `reference_candidate`

```sql
CREATE TABLE reference_candidate (
    record_id           BIGINT NOT NULL,
    reference_id        BIGINT NOT NULL,
    blocker             VARCHAR(64) NOT NULL,
    blocker_score       DOUBLE PRECISION,
    selector_score      DOUBLE PRECISION,
    selector_rank       INT,
    model_version       VARCHAR(128),
    PRIMARY KEY(record_id, reference_id, model_version)
);
```

## 11.5 `product_reference_link`

```sql
CREATE TABLE product_reference_link (
    record_id           BIGINT PRIMARY KEY,
    reference_id        BIGINT,
    status              VARCHAR(32) NOT NULL,
    decision_reason     VARCHAR(256),
    top1_score          DOUBLE PRECISION,
    top2_score          DOUBLE PRECISION,
    score_margin        DOUBLE PRECISION,
    rule_version        VARCHAR(64),
    model_version       VARCHAR(128),
    evidence_snapshot   JSONB,
    decided_at          TIMESTAMPTZ NOT NULL
);
```

关键点是把：

```text
rule_version
model_version
evidence_snapshot
```

都写进去，保证未来可重放。

---

# 12. 增量更新架构

三源会持续更新，因此所有阶段应该是幂等的。

推荐消息流：

```text
Crawler
  |
  v
record.upsert
  |
  v
normalize.record
  |
  v
extract.reference_evidence
  |
  v
resolve.reference_candidates
  |
  v
select.reference
  |
  v
safety_gate
  |
  +--> reference.link.auto
  +--> reference.review
  +--> reference.reject
```

可以使用 Kafka / Redpanda / Pulsar，也可以 MVP 先用数据库任务表；关键不是中间件，而是每一步都通过：

```text
(record_id, content_hash, extractor_version/model_version)
```

保证重复消费不会产生不同结果。

### 什么时候需要重新跑

```text
- 原始商品更新
- Reference normalizer 升级
- OCR 升级
- Canonical Reference Registry 新增 alias
- Selector model 升级
- Safety Gate threshold/rule 升级
```

不要原地覆盖历史 decision，建议 append decision log，再维护 current snapshot。

---

# 13. 模型训练如何直接落地

## 13.1 第一版完全可以先不训练 7B

先用规则 + Reference Registry 建立高 precision baseline：

```text
结构化 reference exact
标题 reference exact
品牌一致
role=brand_reference
无 conflict
```

这一步就能得到一批非常干净的自动链接，同时产生：

```text
- 未解析记录
- 多候选记录
- 冲突记录
```

这些才是后续 Selective EM 的真实训练数据。

## 13.2 黄金标注优先顺序

人工不要随机抽几百对，而应标：

```text
40% 同品牌同系列 sibling reference
20% OCR/格式混淆
15% accessory/compatibility reference
15% source field conflict
10% 完全未知/无 reference
```

比例可以根据线上分布再调整。

## 13.3 CMRL 微调方式

直接参考官方代码，但改训练对象：

```text
query = record evidence text
positive = correct canonical reference profile
negative = same brand hard references
```

训练仍可使用：

```text
Linq-Embed-Mistral / 其他强 embedding backbone
4-bit load
LoRA all-linear
CMRL
```

首轮可以沿用论文默认：

```text
K=10
Top hard negative = 1
```

`margin=0.5`、`alpha=0.6` 只能作为实验起点，不能直接当我们的最优参数。

## 13.4 每轮 Hard Negative Mining

训练一轮后：

```text
对所有训练 query 做 ANN + selector
        ↓
找模型得分最高的错误 reference
        ↓
加入下一轮 hard-negative pool
```

这会比无限增加 random negatives 更有价值。

---

# 14. 图片应该怎么用

Spec 明确“有图片可用”。

但由于业务真值是 Reference，图片最合适的角色不是“长得像就自动匹配”，而是：

```text
1. OCR Reference
2. Evidence Conflict Detection
3. 人工 Review 排序
```

优先 OCR：

```text
保卡
吊牌
表背刻字
盒证标签
机芯/壳号区域（若与 Reference 语义相关）
```

不建议第一版用 CLIP/image embedding 直接决定 same reference，因为：

```text
同系列不同 Reference 外观可以极其接近；
二手拍摄角度、灯光、表带、磨损差异很大；
视觉相似和业务 Reference 相同不是同一个目标函数。
```

图片 embedding 可以作为候选召回/Review 辅助，但不能越过 Reference Hard Veto。

---

# 15. 线上接口建议

可以把 resolver 做成一个清晰、可回放的服务：

```http
POST /v1/reference/resolve
```

输入：

```json
{
  "record_id": 123456,
  "source": "watchhome",
  "brand": "Rolex",
  "title": "劳力士潜航者 126610LN ...",
  "structured_fields": {},
  "images": []
}
```

输出不要只返回 reference：

```json
{
  "decision": "AUTO_MATCH",
  "reference_entity_id": "rolex:126610ln",
  "reference": "126610LN",
  "candidate_scores": [
    ["126610LN", 0.93],
    ["126610LV", 0.71]
  ],
  "margin": 0.22,
  "evidence": [
    {
      "channel": "title",
      "value": "126610LN",
      "role": "brand_reference"
    }
  ],
  "rule_version": "ref-gate-v4",
  "model_version": "select-ref-cmrl-v2"
}
```

所有自动 decision 必须可解释。

---

# 16. 发布指标：不要被普通 F1 欺骗

当前项目建议把指标分四层。

## 16.1 Extraction

```text
Reference Extraction Precision
Reference Role Classification Precision
Canonicalization Exact Accuracy
```

## 16.2 Blocking

```text
Candidate Recall@5
Candidate Recall@10
Candidate Recall@20
```

注意：Recall@K 低时，selector 再强也没有意义。

## 16.3 Selector

研究阶段看：

```text
MRR / MRRP
Top1 Accuracy
Top1-vs-Top2 Margin Distribution
Hard Negative Accuracy
```

## 16.4 Production Gate

真正发布时看：

```text
AUTO_MATCH Precision    << 第一优先级
AUTO_MATCH FP Count     << 必须单独展示
AUTO_MATCH Coverage
Review Rate
Unresolved Rate
```

还要按 slice 分开看：

```text
source
brand
reference pattern
structured/title/OCR channel
accessory vs watch
newly seen references
```

### “0 个 FP”也要看样本量

如果一个版本验证集上 0 个 false positive，不代表真实错误率绝对为 0。

粗略可用经典 rule-of-three：当观察到 0 个错误时，95% 置信上界大约为：

```text
3 / N
```

例如想证明 FP rate 大致低于 0.1%，需要至少几千条自动接受样本仍保持 0 FP，而不是只看几十条。

因此 release gate 应同时要求：

```text
0 observed FP
+ 足够大的 accepted validation sample
+ hard-negative 专项集 0 FP
```

---

# 17. 最推荐的 MVP 顺序

## Phase A：Reference Registry + Rule-first Resolver

先实现：

```text
品牌规范化
Reference normalizer
Reference role classifier
Evidence store
Canonical Reference Registry
Exact strict linking
Conflict reject
```

这一步不依赖大模型就能产生可靠价值。

## Phase B：Hard-negative Candidate Blocking

针对无法 exact 的记录加入：

```text
same-brand reference string search
family block
ANN Top-K
```

保留所有候选和 blocker score。

## Phase C：Selective EM / CMRL

把论文架构迁移成：

```text
record -> reference candidates
```

而不是：

```text
record -> other product records
```

LoRA 微调使用真实 hard-negative 标注。

## Phase D：Reject Calibration + Review Loop

使用黄金集确定：

```text
T_SCORE
T_MARGIN
各 evidence channel 的 hard rule
```

人工 Review 结果自动回流到：

```text
hard-negative pool
normalizer regression tests
reference aliases
selector training set
```

---

# 18. 可以直接实现的核心伪代码

```python
def resolve_record(record):
    normalized = normalize_record(record)

    evidences = extract_reference_evidence(normalized)
    evidences = [
        classify_reference_role(e)
        for e in evidences
    ]

    # 1. 强冲突直接拒绝自动路径
    conflict = detect_reference_conflict(evidences)
    if conflict:
        return Decision.review(
            reason="REFERENCE_CONFLICT",
            evidences=evidences,
        )

    # 2. 高可信 exact path
    exact_refs = get_high_precision_reference_values(evidences)
    exact_candidates = lookup_canonical_references(
        brand=normalized.brand,
        values=exact_refs,
    )

    if len(exact_candidates) == 1:
        candidate = exact_candidates[0]
        if strict_gate(normalized, evidences, candidate):
            return Decision.auto_match(candidate, evidences)

    # 3. ambiguous path -> blocking
    candidates = retrieve_reference_candidates(
        record=normalized,
        evidences=evidences,
        top_k=20,
    )

    if not candidates:
        return Decision.unresolved("NO_CANDIDATE")

    # 4. listwise selective ranking
    ranked = selector.score(normalized, evidences, candidates)
    top1 = ranked[0]
    top2 = ranked[1] if len(ranked) > 1 else None

    score_margin = (
        top1.score - top2.score
        if top2 is not None
        else float("inf")
    )

    # 5. precision-first reject option
    if not hard_reference_consistency(top1, evidences):
        return Decision.review("REFERENCE_NOT_CONFIRMED", ranked)

    if top1.score < thresholds.absolute_score:
        return Decision.review("LOW_SELECTOR_SCORE", ranked)

    if score_margin < thresholds.top_margin:
        return Decision.review("AMBIGUOUS_SIBLING_REFERENCE", ranked)

    if calibrated_fp_risk(top1, score_margin, evidences) > thresholds.max_fp_risk:
        return Decision.review("RISK_TOO_HIGH", ranked)

    return Decision.auto_match(top1.reference, evidences, ranked)
```

核心设计只有一句话：

```text
Selector 负责排序，Safety Gate 负责授权。
```

---

# 19. 为什么这套方案比直接“多模态 LLM 两两判断”更合适

如果直接让大模型回答：

```text
商品 A 和商品 B 是不是同一型号？
```

会遇到：

```text
O(N²) 候选爆炸
高成本
同一 pair 多次判断可能不稳定
很难维护全局一致性
一条误边会污染聚类
无法清晰利用业务定义的 Reference 真值
```

Reference Entity 方案把问题改成：

```text
每条记录只解析一次 Reference
```

随后跨源匹配基本退化成数据库 join：

```sql
SELECT ...
FROM source_a a
JOIN source_b b
  ON a.reference_entity_id = b.reference_entity_id;
```

ML 只处理“解析不到 / 有歧义”的长尾区域，而不是承担全部业务正确性。

这与当前“precision 优先到极致、允许漏匹配”的目标完全一致。

---

# 20. 最终建议

这篇论文值得采纳的核心是：

```text
Hard Negative Mining
        +
Candidate-set Competition
        +
Contrastive Margin Ranking
```

但当前项目必须额外加入：

```text
Canonical Reference Entity
        +
Reference Evidence Store
        +
Role Classification
        +
Hard Conflict Veto
        +
Abstain / Review Gate
```

因此推荐最终架构不是：

```text
Embedding -> Pairwise LLM -> Merge
```

而是：

```text
Raw Product
   ↓
Reference Evidence Extraction
   ↓
Canonicalization / Role Classification
   ↓
Reference Candidate Blocking
   ↓
Selective CMRL Ranker
   ↓
Precision Safety Gate
   ↓
Canonical Reference Entity Link
   ↓
Cross-source Same-Reference Join
```

其中自动接受的最强路径始终应当是：

```text
明确 Reference 硬证据一致
+ 无任何冲突
+ 候选唯一/显著领先
+ 校准后风险满足极低 FP 目标
```

其余全部允许拒识。

这既保留了 Mistral4SelectEM 对 hard negative 的核心优势，又避免了论文 closed-set `argmax` 设计与当前“绝不能误匹配”目标之间的冲突，适合直接作为跨源二奢/腕表 Reference 实体匹配系统的技术骨架。

---

## 参考实现与源码位置

论文：

- <https://www.sciencedirect.com/science/article/pii/S1474034625004318>
- <https://doi.org/10.1016/j.aei.2025.103538>

官方仓库：

- <https://github.com/quickhdsdc/LLM4EntityMatching>

关键源码：

- Candidate/FAISS 数据重编译：<https://github.com/quickhdsdc/LLM4EntityMatching/blob/main/EM_data_convert.py>
- Selective 训练入口：<https://github.com/quickhdsdc/LLM4EntityMatching/blob/main/EM_training_contrastive.py>
- 数据预处理：<https://github.com/quickhdsdc/LLM4EntityMatching/blob/main/tasks/SelectiveEntityMatching/selection_llm/data_preprocessor.py>
- Siamese + CMRL 核心：<https://github.com/quickhdsdc/LLM4EntityMatching/blob/main/tasks/SelectiveEntityMatching/selection_llm/modelling_llama.py>
- LoRA/训练参数：<https://github.com/quickhdsdc/LLM4EntityMatching/blob/main/tasks/SelectiveEntityMatching/selection_llm/model_finetuner.py>
- 推理与 MRRP：<https://github.com/quickhdsdc/LLM4EntityMatching/blob/main/tasks/SelectiveEntityMatching/selection_llm/evaluater.py>
