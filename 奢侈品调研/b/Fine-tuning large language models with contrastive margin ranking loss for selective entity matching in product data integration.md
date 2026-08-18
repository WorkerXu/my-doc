# Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration

## 0. 结论先行

这篇论文最值得借鉴的不是“用 7B LLM 做实体匹配”，而是它把传统的 **pairwise entity matching（逐对二分类）** 改造成 **selective/listwise matching（候选集内竞争选择）**：一个 query 不再分别问“它是否和 A 匹配、是否和 B 匹配”，而是把同一个 blocking 召回出来的多个相似候选放到同一竞争空间，让模型学习“真正匹配项必须显著优于最像的 hard negative”。

对当前 Spec「雷小安 × 腕表之家 × 奢当家」而言，这个思想非常契合最危险的错误类型：

- 同品牌、同系列、外观高度相似，但 reference 只差 1~2 个字符；
- 标题同时出现商品自身 reference、兼容型号、配件适用型号、平台 SKU；
- 图片看起来几乎一样，但实际是不同尺寸、材质、机芯或年份的相邻 reference；
- 普通 pairwise matcher 对每个候选独立打分，可能给多个相似候选都打出“match”，产生 false positive。

但论文原始方案 **不能原样上线**。当前需求明确规定：

> “同一个商品”定义为同一 reference number，且绝对不能误匹配，precision 优先，允许漏匹配。

因此推荐把论文方法降级为 **reference-first 架构中的候选集判别器/冲突检测器**，而不是最终真值来源。生产系统的最终自动合并必须由“可审计的 reference 硬证据 + listwise margin + 拒识机制”共同放行。

推荐落地形态：

> **Reference 抽取/角色识别 → 保守规范化 → 品牌内 Blocking → Hard-negative Candidate Set → SelectEM 双塔判别 → Precision Gate → 可信边聚类**

其中 SelectEM 负责“在非常像的一组候选里分辨谁最像”，**Precision Gate 才负责是否允许自动合并**。

---

## 1. 论文与代码来源

论文：

- Qian Ruan, Dachuan Shi, Thomas Bauernhansl, *Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration*，Advanced Engineering Informatics 67 (2025), 103538。
- DOI: https://doi.org/10.1016/j.aei.2025.103538
- ScienceDirect: https://www.sciencedirect.com/science/article/pii/S1474034625004318

官方代码：

- https://github.com/quickhdsdc/LLM4EntityMatching

重点实现文件：

- `EM_data_convert.py`：用 embedding + FAISS 重编 benchmark，生成 Top-K hard-negative 候选集。
- `EM_training_contrastive.py`：实验入口、模型/损失/LoRA 超参数。
- `tasks/SelectiveEntityMatching/selection_llm/data_preprocessor.py`：候选集输入组织与负样本抽样。
- `tasks/SelectiveEntityMatching/selection_llm/modelling_llama.py`：Siamese 编码、相似度、CMRL/InfoNCE/Focal 类损失实现。
- `tasks/SelectiveEntityMatching/selection_llm/model_loader.py`：4-bit NF4 量化加载。
- `tasks/SelectiveEntityMatching/selection_llm/model_finetuner.py`：LoRA/QLoRA 微调。
- `tasks/SelectiveEntityMatching/selection_llm/evaluater.py`：候选集重排和 MRR 风格评估。

---

## 2. 论文真正解决的核心问题

### 2.1 Pairwise EM 的结构性缺陷

传统流程通常是：

1. Blocking 先从大库中召回 K 个候选；
2. 对 `(query, candidate_i)` 分别做二分类；
3. 每个 pair 独立输出 match / non-match。

问题是 Blocking 本身就会把“最相似”的记录放在一起，所以候选集天然包含大量 hard negatives。

腕表示例：

```text
query:
Rolex Submariner Date 41mm 黑盘 126610LN

候选:
A. Rolex Submariner Date 41mm 126610LN   <- 真匹配
B. Rolex Submariner Date 41mm 126610LV   <- 绿圈，相邻 reference
C. Rolex Submariner Date 40mm 116610LN   <- 上一代，相似度极高
D. Rolex Sea-Dweller 126600
```

如果分别判断：

```text
P(match(query, A)) = 0.96
P(match(query, B)) = 0.94
P(match(query, C)) = 0.91
```

阈值 0.9 会得到 3 个“正例”。

而 listwise 目标强调的是：

```text
score(A) 必须显著 > score(B), score(C), score(D)
```

这比单独把 A 推到 0.9 以上更符合真实候选选择问题。

### 2.2 论文的关键变化

论文做了三件事：

1. **重编 benchmark**：故意给每个 query 加入语义非常相似的 hard negative；
2. **把 EM 改成候选集选择任务**：query 一次面对整个候选集合；
3. **用 CMRL（Contrastive Margin Ranking Loss）**：重点惩罚分数最接近正例的 hard negatives。

对本项目最重要的是第 2、3 点。

---

## 3. 官方实现架构拆解

## 3.1 候选集生成：Embedding + FAISS

`EM_data_convert.py` 中先将 target corpus 编码成向量，再建立：

```python
faiss.IndexFlatIP(d)
```

因为向量在前面已经 L2 normalize，所以 Inner Product 等价于 cosine similarity。

针对每个 query：

```python
scores, indices = faiss_index.search(query_embedding, 20)
```

然后保留 Top-10 形成最终候选集。

论文代码使用过：

- `Linq-AI-Research/Linq-Embed-Mistral`
- `Salesforce/SFR-Embedding-Mistral`
- SentenceTransformer
- Azure embedding

官方 README 的 benchmark 重编方式主要采用 `Linq-Embed-Mistral + FAISS`。

### 对本项目的意义

千万级数据不可能做三源全笛卡尔积。

假设三源总计 10,000,000 条记录，pairwise 全比较量是灾难性的；必须先 blocking，把每条商品压缩到几十个候选以内。

但是本项目不应只做纯 semantic ANN。推荐 blocking 先分层：

```text
brand hard partition
  ↓
reference token / series / family lexical block
  ↓
semantic ANN / text embedding
  ↓
Top-K hard candidate set
```

这样能明显降低跨品牌相似文本带来的无意义候选，也更符合“reference 是最终身份”的约束。

---

## 3.2 数据形态：从 pair 变成 query + candidates + label vector

官方重编数据不是：

```text
(query, candidate, label)
```

而是：

```text
query
candidates = [c0, c1, ..., c9]
labels     = [0, 0, 1, ..., 0]
```

`data_preprocessor.py` 会：

1. 取正样本；
2. 从负样本中抽 `num_neg`；
3. 正负候选混合并 shuffle；
4. query 和 candidates 分别 tokenize。

默认实验：

```python
num_neg = 9
```

因此大部分训练样本是 `1 positive + 9 hard negatives`。

这非常适合本项目训练“相邻 reference 排雷器”。

建议腕表训练样本不要随机负采样，而要按错误风险构造：

```text
正例：同 canonical reference

Hard Negative 优先级：
P0: 同品牌 + 同系列 + reference 编辑距离 1~2
P0: 同品牌 + 同系列 + 相同前缀/主体，仅尾缀不同
P1: 同品牌 + 同系列 + 不同尺寸/材质/机芯
P1: 标题出现兼容 reference，但商品本身不是该 reference
P1: 平台 SKU / 店铺货号长得像品牌 reference
P2: 同品牌相似产品名但不同 reference
P3: 普通语义相似负样本
```

训练价值最大的不是 easy negative，而是 P0/P1。

---

## 3.3 Siamese / 双塔编码

`modelling_llama.py` 的主体是一个共享参数的 `MistralModel`。

Query：

```python
hidden_states_query = model(query)
query_emb = last_token_embedding(hidden_states_query)
query_emb = normalize(query_emb)
```

Candidates：

```python
[B, K, L] -> [B*K, L]
     ↓ shared Mistral
candidate_emb
     ↓ normalize
[B, K, D]
```

然后：

```python
similarities = query_emb @ candidate_emb.T
```

也就是说：

```text
               ┌──────── shared Mistral encoder ────────┐
Query ─────────┤                                         ├─ q ∈ R^d
               │                                         │
Cand 1 ────────┤                                         ├─ c1 ∈ R^d
Cand 2 ────────┤       同一组模型权重                    ├─ c2 ∈ R^d
...            │                                         │
Cand K ────────┤                                         ├─ ck ∈ R^d
               └─────────────────────────────────────────┘

score_i = cosine(q, c_i)
```

共享编码器带来两个工程优势：

1. 候选向量可以离线预计算；
2. 推理不是 Cross Encoder 的 `K` 次完整交互计算，规模更容易扩展。

对于百万到千万级库，这是比直接 GPT pairwise 判定更现实的结构。

---

## 3.4 Last-token pooling

官方代码取最后一个有效 token 的 embedding：

```python
sequence_lengths = attention_mask.sum(dim=1) - 1
last_token_embeddings = last_hidden_state.gather(...)
```

然后：

```python
F.normalize(..., p=2, dim=1)
```

这是 Mistral embedding 类模型常见做法。

对腕表项目，输入不要简单把所有字段拼成自然语言，建议显式保留字段语义：

```text
[BRAND] ROLEX
[TITLE] 劳力士 潜航者 日历型 41 黑盘
[REFERENCE_CANDIDATES] 126610LN
[REFERENCE_SOURCE] title_regex
[REFERENCE_ROLE] brand_reference
[SERIES] Submariner
[SIZE] 41mm
[MATERIAL] Oystersteel
[PLATFORM_SKU] LXA-839281
```

候选也用同样 schema。

这样模型更容易学到“reference 角色”而不是被营销词淹没。

---

## 3.5 CMRL：最值得迁移的部分

官方实现先计算普通 Cross Entropy：

```python
labels_ce = argmax(labels)
loss_ce = CrossEntropy(similarities, labels_ce)
```

然后从所有 negative score 中选最难的一部分：

```python
hard_negatives = topk(negative_scores, top_k)
```

对每个 hard negative 与 positive 计算：

```text
difference = score_negative - score_positive + margin
```

只有：

```text
score_positive < score_negative + margin
```

才产生 ranking penalty。

可写成近似形式：

```text
L_margin = Σ w_j · max(0, s_neg_j - s_pos + m)
```

其中 hard-negative 权重由：

```text
w_j ∝ exp(s_neg_j - s_pos + m)
```

得到。越接近或超过正例的负样本，权重越大。

最终：

```text
L = (1 - α) · L_CE + α · L_margin
```

官方默认实验参数：

```python
loss_type = "CMRL"
margin = 0.5
alpha = 0.6
top_k = 1
```

### 为什么非常适合 reference 场景

如果：

```text
正例：126610LN
难负例：126610LV
普通负例：SBGX261
```

普通 CE 可能主要关心正例是不是 Top-1；CMRL 会专门盯住 `126610LV` 这种高分难负例，并要求：

```text
s(126610LN) >= s(126610LV) + margin
```

这正是当前业务最需要的优化方向：**不是提升平均 F1，而是把“极像但不相同”的 reference 拉开。**

---

## 3.6 QLoRA / 训练资源控制

`model_loader.py` 使用：

```python
load_in_4bit=True
bnb_4bit_use_double_quant=True
bnb_4bit_quant_type="nf4"
bnb_4bit_compute_dtype=torch.bfloat16
```

`model_finetuner.py` 再使用 LoRA：

```python
r = 64
lora_alpha = 64
lora_dropout = 0.1
target_modules = "all-linear"
```

训练参数示例：

```python
learning_rate = 2e-4
batch_size = 32
gradient_accumulation_steps = 8
train_epochs = 10
scheduler = cosine
optimizer = paged_adamw_32bit
```

这意味着不需要全参数微调 7B 模型。

对只有几百对黄金标注的当前需求，LoRA 很合适；但建议先训练一个更小、更便宜的 encoder baseline，再判断是否真的需要 7B。

推荐顺序：

```text
规则 baseline
  ↓
BGE/E5/MPNet 小双塔 + CMRL
  ↓ 如果 hard-negative precision 不够
7B Mistral embedding + QLoRA + CMRL
```

不要一开始就把 7B 当必要条件。

---

## 4. 官方实现中不适合直接生产的地方

这是本次分析最重要的部分之一。

## 4.1 Benchmark 会“强行保证 Top-10 有真匹配”

`EM_data_convert.py` 中：

```python
if 1 not in label_k10:
    candidates_k10[-1] = truth_list[0]
    label_k10[-1] = 1
```

也就是说，如果 blocking 没召回真值，代码会把真值直接塞到第 10 位。

因此论文的 selective matcher 评估实际上把两个问题拆开了：

- Blocking recall；
- Candidate set 内选择。

但它没有真实模拟“候选集合根本没有正确答案”的生产情况。

### 本项目必须修改

线上必须允许：

```text
NONE / ABSTAIN / NO_MATCH
```

否则模型被迫从错误候选里挑一个，precision-first 需求会直接被破坏。

建议训练候选集合明确加入：

```text
[NO_MATCH]
```

并构造至少 30% 的训练/验证 query 为“候选集中没有真匹配”。

---

## 4.2 原论文问题定义不是“reference 强约束”

它解决的是一般 product entity matching，而当前 Spec 已经给了一个非常强的业务定义：

```text
same product ⇔ same reference number
```

这意味着模型语义相似度绝对不能覆盖 reference 冲突。

例如：

```text
A: 126610LN
B: 126610LV
```

不管图像、标题、系列多像，只要两个 reference 都被高置信识别为“当前商品品牌 reference”，就应该：

```text
HARD_REJECT
```

而不是让模型决定。

---

## 4.3 原代码没有真正的“拒识阈值”

`modelling_llama.py` 推理本质：

```python
predicted_labels = argmax(softmax(similarities))
```

也就是必选一个 Top-1。

当前需求必须额外加：

```text
absolute score threshold
+ Top1/Top2 margin threshold
+ hard-rule consistency
+ optional evidence count
```

例如：

```text
accept only if:
  canonical_ref(q) == canonical_ref(c)
  AND reference_role(q) == brand_reference
  AND reference_role(c) == brand_reference
  AND no conflicting reference evidence
  AND score_top1 >= τ_score
  AND score_top1 - score_top2 >= τ_margin
  AND blocker_contains_no_same-brand-conflict
otherwise abstain
```

---

## 4.4 原评估指标不适合“绝不能误匹配”

官方主要使用 MRR 风格的候选排序指标，并实现了一个对 false positive / missed positive 施加 penalty 的 variant。

但当前系统上线应把核心指标改成：

```text
Auto-Match Precision
False Match Count
False Match Rate
Coverage / Auto-Match Rate
Recall under fixed precision
Precision@AutoAccept
Abstention Rate
```

关键曲线应该是：

```text
coverage vs precision
```

而不是只看 F1 或 MRR。

---

## 4.5 多模态不能让图片成为越权证据

论文主体是文本/属性 embedding。

当前项目有图片，可以用于：

- OCR 表背、保卡、吊牌 reference；
- 判断标题 reference 是否属于当前主商品还是配件；
- 发现文字信息冲突；
- 人工复核排序。

但不建议：

```text
视觉很像 -> 自动认定 same reference
```

同系列腕表视觉差异可能非常小，图片应是 **辅助证据/否决证据**，不是 final identity key。

---

# 5. 面向当前 Spec 的直接落地架构

## 5.1 总体架构

```text
                ┌──────────────────────────┐
                │ 雷小安 / 腕表之家 / 奢当家 │
                └─────────────┬────────────┘
                              │ incremental ingest
                              ▼
                   ┌─────────────────────┐
                   │ Raw Record Store     │
                   │ 原始字段/图片/来源ID  │
                   └──────────┬──────────┘
                              ▼
               ┌──────────────────────────────┐
               │ Normalization & Extraction    │
               │ brand / series / reference    │
               │ identifier role / OCR evidence│
               └──────────────┬───────────────┘
                              ▼
               ┌──────────────────────────────┐
               │ Conservative Ref Canonicalizer│
               │ 生成 canonical_ref + evidence│
               └──────────────┬───────────────┘
                              ▼
                  ┌────────────────────────┐
                  │ Hierarchical Blocking   │
                  │ brand → lexical → ANN   │
                  └────────────┬───────────┘
                               ▼
             ┌──────────────────────────────────┐
             │ Hard-Negative Candidate Set K=10 │
             └────────────────┬─────────────────┘
                              ▼
               ┌─────────────────────────────┐
               │ SelectEM / CMRL Siamese      │
               │ score + top1/top2 margin     │
               └──────────────┬──────────────┘
                              ▼
              ┌────────────────────────────────┐
              │ Precision Gate / Rule Engine    │
              │ ACCEPT / REJECT / ABSTAIN       │
              └───────────────┬────────────────┘
                              ▼
                  ┌────────────────────────┐
                  │ Trusted Match Graph      │
                  │ reference invariant      │
                  └────────────┬───────────┘
                               ▼
                        Global Product ID
```

---

## 5.2 第一级：Identifier Extraction 不能只抽字符串，还要抽“角色”

这是 reference 系统能否高 precision 的关键。

标题里出现的编号至少分成：

```text
brand_reference      品牌官方 reference / model number
platform_sku         平台自有商品号
seller_sku           商家内部 SKU
compatible_reference “适用 126610LN”中的被适配型号
accessory_reference  表带/表盒/零件 reference
serial_number        序列号
unknown_identifier   未确认身份编号
```

建议中间结果结构：

```json
{
  "raw": "126610LN",
  "canonical": "126610LN",
  "role": "brand_reference",
  "brand": "ROLEX",
  "source": "title",
  "extractor": "brand_regex_v3",
  "confidence": 0.997,
  "span": [18, 26]
}
```

只有：

```text
role == brand_reference
```

才可以参与自动 same-reference 合并。

这一层能直接避免大量“把平台货号当型号”的灾难性误匹配。

---

## 5.3 第二级：Reference Canonicalization 必须保守、可逆

不要为了提升 recall 过度删除字符。

建议 canonicalization：

安全操作：

```text
Unicode NFKC
英文统一大写
全角转半角
首尾空白删除
常见分隔符标准化
品牌明确规则下的格式标准化
```

危险操作：

```text
无条件删除所有 - / . 空格
模糊纠正 O ↔ 0
模糊纠正 I ↔ 1
截断后缀
只保留数字
```

例如某些品牌后缀就代表材质/盘面/地区或版本，不应随意删除。

推荐存：

```text
reference_raw
reference_normalized
reference_canonical
normalization_rule_version
```

每一步都能追溯。

---

## 5.4 第三级：Blocking 不应让 ANN 独自决定

百万/千万级建议：

### Block A：品牌硬分区

```text
canonical_brand 必须相同
```

除非品牌自身尚未识别，否则不跨品牌比较。

### Block B：Reference lexical block

若已抽到候选 reference：

```text
exact canonical token
prefix/suffix family
character n-gram
edit distance neighborhood
```

这里故意召回相邻 reference，给 hard-negative selector 做排雷。

### Block C：Semantic ANN

文本 embedding 只补足：

- reference 缺失；
- 字段稀疏；
- 标题描述方式差异大。

ANN 建议按品牌分 shard，避免全局索引。

候选集：

```text
K = 10~30
```

真正昂贵的 SelectEM 只跑这 K 个。

---

## 5.5 第四级：腕表版 SelectEM 输入设计

建议输入不是纯 title，而是结构化文本：

```text
[BRAND] ROLEX
[SERIES] SUBMARINER
[TITLE] 劳力士潜航者型日历腕表 黑盘
[REFERENCE] 126610LN
[REFERENCE_ROLE] brand_reference
[REF_EVIDENCE] title:126610LN; ocr_caseback:126610LN
[SIZE] 41mm
[MATERIAL] oystersteel
[MOVEMENT] automatic
[COLOR] black
[SOURCE] leixiaoan
```

候选同结构。

训练：

```text
1 positive + N hard negatives + optional NO_MATCH
```

推荐先：

```text
N = 9
```

与论文保持一致，便于复现实验。

---

## 5.6 第五级：把论文的 CMRL 变成 Reference-aware CMRL

基础：

```text
L = (1-α)L_ce + αL_margin
```

可以增加业务权重：

```text
w_neg =
  3.0  if same_brand & same_series & ref_edit_distance <= 2
  2.5  if same_brand & conflicting_high_conf_ref
  2.0  if compatible_reference trap
  1.5  if platform_sku trap
  1.0  otherwise
```

最终：

```text
L_ref_margin = Σ w_neg · softmax_weight · max(0, s_neg - s_pos + m)
```

这样训练会显著偏向“业务最不能错”的 hard negatives。

但注意：即便模型把这些负样本学得很好，**最终 hard reference conflict 仍然应该规则拒绝**。

---

# 6. Precision Gate：真正决定“能不能自动合并”的模块

建议结果不输出 bool，而是三态：

```text
ACCEPT
REJECT
ABSTAIN
```

## 6.1 ACCEPT 条件

第一版可以极保守：

```python
def can_auto_match(a, b, select_score, second_score):
    if a.brand != b.brand:
        return False

    if not a.ref.is_high_conf_brand_reference:
        return False
    if not b.ref.is_high_conf_brand_reference:
        return False

    if a.ref.canonical != b.ref.canonical:
        return False

    if a.has_reference_conflict or b.has_reference_conflict:
        return False

    if a.is_accessory_or_compatible_item or b.is_accessory_or_compatible_item:
        return False

    if select_score < TAU_SCORE:
        return False

    if select_score - second_score < TAU_MARGIN:
        return False

    return True
```

这时模型不是证明“它们相同”，而是在 reference 已经强一致之后确认：

```text
这个候选确实是候选集中唯一合理解释
```

## 6.2 HARD REJECT 条件

```text
高置信 brand_reference 冲突
品牌冲突
一个是腕表主体、一个明显是配件
同一条记录出现两个互相冲突的高置信 reference
```

## 6.3 ABSTAIN 条件

```text
reference 缺失
reference 仅 OCR 单证据且质量差
reference role 不确定
Top1/Top2 margin 太小
候选集里多个同 reference 但来源字段冲突
模型分数高但规则证据不足
```

ABSTAIN 进入人工池或等待后续更多证据。

这正符合“允许漏匹配”。

---

# 7. 必须加入 NO_MATCH 候选

论文原实现最大的上线风险，是默认候选里一定有正确答案。

当前系统必须显式训练：

```text
Query → [cand1, ..., candK, NO_MATCH]
```

训练数据必须覆盖：

```text
case A: 真匹配在候选中
case B: 真匹配不在候选中
case C: 当前三个源里压根没有同 reference 商品
case D: 候选都是相邻 reference
```

可以两种实现：

### 方案 1：显式 Null Embedding

学习一个 `[NO_MATCH]` embedding，与 K 个候选共同 softmax。

### 方案 2：阈值拒识

```text
Top1 score < τ_score -> NO_MATCH
Top1 - Top2 < τ_margin -> ABSTAIN
```

推荐生产上同时做，两道保险。

---

# 8. 几百对黄金标签怎么花最值

不要随机标几百对。

建议 400 对黄金集：

```text
160 对：同品牌同系列相邻 reference hard negative
80 对：compatibility / accessory reference trap
60 对：platform SKU / seller SKU 与 brand reference 混淆
40 对：reference 格式差异但实际相同
30 对：OCR reference 与文本冲突
30 对：字段极稀疏 / 无 reference / 无正确候选
```

正负比例可以偏负，因为 precision-first 更关心 false positive。

### 切分必须按 reference group

不能随机 record split，否则同一 reference 的近重复记录可能同时出现在 train/test，指标会虚高。

建议：

```text
GroupKFold by canonical_reference
```

甚至保留“未见品牌/未见系列”测试集，模拟增量来源漂移。

---

# 9. 推荐的数据库中间模型

## product_record

```text
record_id
source
source_record_id
raw_title
brand_raw
brand_canonical
series_raw
images[]
raw_payload
created_at
updated_at
```

## extracted_identifier

```text
identifier_id
record_id
raw_value
canonical_value
role
brand
source_field
extractor
confidence
rule_version
span_or_image_id
```

## candidate_edge

```text
left_record_id
right_record_id
block_reason
ann_score
lexical_score
select_score
select_margin
reference_equal
reference_conflict
status
```

## match_decision

```text
decision_id
left_record_id
right_record_id
decision = ACCEPT | REJECT | ABSTAIN
reason_codes[]
model_version
rule_version
evidence_snapshot
created_at
```

## entity_cluster

```text
global_product_id
canonical_brand
canonical_reference
cluster_version
```

这样每个自动合并都能完整审计。

---

# 10. 聚类不能简单做“相似边传递闭包”

三源最终很容易做：

```text
A match B
B match C
=> A/B/C 一个 cluster
```

但 precision-first 系统必须维护 invariant：

```text
一个 cluster 中所有高置信 canonical brand_reference 必须完全一致
```

伪代码：

```python
def union(cluster_a, cluster_b):
    refs = high_conf_refs(cluster_a) | high_conf_refs(cluster_b)
    if len(refs) > 1:
        raise Conflict("reference invariant violated")
    do_union(cluster_a, cluster_b)
```

因此即便某条 ML 边错误，也不能污染整个 connected component。

---

# 11. 线上增量架构

需求是 100 万~1000 万级并持续更新，推荐采用离线主索引 + 增量索引。

```text
新记录
  ↓
抽取 reference / brand / series
  ↓
查询 canonical reference index
  ↓
若存在高置信 exact ref：进入 precision gate
  ↓
若没有：进入 brand shard ANN / lexical blocker
  ↓
形成 K candidates
  ↓
SelectEM + rules
  ↓
ACCEPT / REJECT / ABSTAIN
```

### 存储建议

第一版无需复杂大数据栈：

- PostgreSQL：记录、证据、规则结果、聚类主数据；
- 对象存储：图片；
- FAISS/HNSW：向量候选；
- Python/FastAPI worker：抽取和匹配服务；
- Celery/Kafka/消息队列任选一个：增量异步任务。

数据继续扩大后再替换为 Milvus/Qdrant/Elastic/OpenSearch 等服务化索引。

重点不是选什么中间件，而是把：

```text
blocking / extraction / matching / decision / clustering
```

分成可独立版本化的阶段。

---

# 12. 训练与上线流程

## Phase 0：纯规则基线

先做：

```text
brand canonicalization
reference extraction
identifier role classifier
reference canonicalization
exact reference match
```

只有双方高置信 brand_reference 完全一致才自动匹配。

这一步本身可能已经解决大量高 precision case。

## Phase 1：建立 hard-negative benchmark

从真实三源数据挖：

```text
same brand + same series + different ref
```

以及：

```text
标题包含对方 reference 但商品不是对方型号
```

形成内部 benchmark。

## Phase 2：复现 SelectEM baseline

先直接复现论文：

```text
K=10
CMRL
margin=0.5
alpha=0.6
top_k=1
```

但加入 NO_MATCH。

## Phase 3：Reference-aware CMRL

增加 hard-negative 类型权重；输入加入 reference role/evidence。

## Phase 4：Shadow Mode

线上只打分，不自动合并；与规则和人工结果对比。

重点记录：

```text
false positive type
margin distribution
score calibration
brand/reference bucket
```

## Phase 5：只开放最安全 bucket

例如第一批仅自动开放：

```text
brand 相同
双方高置信 reference 相同
双方 reference role=brand_reference
没有任何冲突 reference
select margin > τ_margin
```

其余仍 abstain。

之后按 bucket 逐步放量。

---

# 13. 阈值怎么定：不要围绕 F1 调

假设验证集得到每条 candidate：

```text
score_top1
score_top2
margin = top1 - top2
rule evidence
```

按：

```text
τ_score × τ_margin × evidence bucket
```

网格搜索。

目标不是：

```text
max F1
```

而是：

```text
在满足目标 precision 的所有阈值中，coverage 最大
```

例如：

```text
maximize auto_match_coverage
subject to precision >= target_precision
```

几百个标签无法统计上证明“绝对零错误”，所以生产还需要：

- 规则硬约束；
- shadow validation；
- 持续抽样人工审计；
- 新品牌/新来源默认回到 abstain；
- 模型/规则版本可回滚。

---

# 14. 图片应该怎么接入

不建议一开始把图片 embedding 直接和文本 embedding 融成一个最终 match score。

推荐先做独立 evidence channel：

```text
Image
  ↓
OCR
  ↓
reference candidate
  ↓
identifier role / evidence confidence
```

例如：

```text
标题 reference = 126610LN
保卡 OCR = 126610LN
表背 OCR = 无
```

这比“两个表的图片 embedding 很像”更可靠。

第二阶段才考虑视觉模型用于：

```text
same series / dial / bezel / case family
```

并且只做 conflict detection / manual ranking。

---

# 15. 推荐的最小代码模块

```text
luxury_matcher/
├── ingest/
│   └── normalize_record.py
├── reference/
│   ├── extract.py
│   ├── role_classifier.py
│   ├── canonicalize.py
│   └── brand_rules/
├── blocking/
│   ├── lexical.py
│   ├── ann.py
│   └── candidate_builder.py
├── selectem/
│   ├── dataset.py
│   ├── encoder.py
│   ├── cmrl_loss.py
│   ├── train.py
│   └── inference.py
├── decision/
│   ├── precision_gate.py
│   └── reason_codes.py
├── clustering/
│   └── constrained_union_find.py
└── evaluation/
    ├── hard_negative_benchmark.py
    ├── coverage_precision.py
    └── audit_report.py
```

`cmrl_loss.py` 可以直接参考论文代码重写成清晰版本：

```python
def cmrl_loss(scores, labels, margin=0.5, alpha=0.6, hard_k=1):
    # scores: [B, K]
    pos_idx = labels.argmax(dim=1)
    ce = F.cross_entropy(scores, pos_idx)

    ranking_losses = []
    for row_scores, row_labels in zip(scores, labels):
        pos = row_scores[row_labels.bool()]
        neg = row_scores[~row_labels.bool()]

        hard_neg = torch.topk(neg, k=min(hard_k, len(neg))).values
        diff = hard_neg[:, None] - pos[None, :] + margin
        weights = torch.softmax(diff, dim=0)
        ranking_losses.append((weights * F.relu(diff)).sum())

    ranking = torch.stack(ranking_losses).mean()
    return (1 - alpha) * ce + alpha * ranking
```

再增加业务 hard-negative weight 即可。

---

# 16. 与普通 Pairwise / Ditto / DeepBlocker 的关系

这个方法不是替代所有已有组件，而是处于不同层：

```text
DeepBlocker / ANN
负责：候选召回

Ditto / Pairwise Matcher
负责：逐对判定

Mistral4SelectEM / CMRL
负责：候选集内 hard-negative 竞争

Reference Rule Engine
负责：最终身份约束
```

当前 Spec 的推荐组合不是“只选一个模型”，而是：

```text
Blocking 高召回
+ SelectEM 消 hard negative
+ Reference Rule 高 precision 收口
+ Abstention 承接不确定性
```

---

# 17. 最推荐的最终方案

如果现在就要开始实现，我会选下面这条路线：

### Step 1

先建立跨三源统一 `brand_reference` 抽取层，所有抽出的编号必须同时判断 identifier role。

### Step 2

做保守 canonicalization，建立：

```text
(brand, canonical_reference) -> global_product_id
```

的高可信索引。

### Step 3

对 reference 缺失/冲突/多候选的记录，通过 brand shard + lexical + ANN 生成 K=10 候选。

### Step 4

把候选构造成 listwise hard-negative 数据，使用论文 Mistral4SelectEM 的 Siamese + CMRL 思路训练候选集判别器。

### Step 5

增加 `[NO_MATCH]`，不允许模型被迫选择。

### Step 6

最终自动匹配必须通过 Precision Gate：

```text
高置信 same canonical reference
+ identifier role 正确
+ 无 reference 冲突
+ SelectEM Top1 score 足够高
+ Top1/Top2 margin 足够大
```

否则一律 ABSTAIN。

### Step 7

聚类层维护“一个 cluster 只能有一个高置信 canonical reference”的 invariant。

这比单纯复制论文模型更适合当前“absolute precision-first”的业务目标。

---

# 18. 一句话评价这篇论文对当前需求的价值

**最有价值的是把“是不是匹配”改成“在一堆极像的候选里，正确 reference 必须赢过最危险的 hard negative”，并用 margin loss 专门优化这个差距；但当前项目必须在它外面再加 reference 硬约束和 NO_MATCH/ABSTAIN，才能真正满足“宁可漏匹配，也绝不能误匹配”。**
