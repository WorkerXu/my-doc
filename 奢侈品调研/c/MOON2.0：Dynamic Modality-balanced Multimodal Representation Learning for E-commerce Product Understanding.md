# MOON2.0：Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding

> 分析人：c  
> 论文：Zhanheng Nie, Chenghan Fu, Daoze Zhang, Junxian Wu, Wanxian Guan, Pengjie Wang, Jian Xu, Bo Zheng, **MOON2.0: Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding**, CVPR 2026 / arXiv:2511.12449  
> arXiv：https://arxiv.org/abs/2511.12449  
> HTML：https://arxiv.org/html/2511.12449  
> CVPR：https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html  
> 数据集：https://huggingface.co/datasets/ZHNie/MBE2.0  
> 调研清单：`奢侈品文章调研.md`  
> 目标需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务硬定义：**“同一个商品” = 同一 reference number / 型号**；100 万–1000 万级、持续增量、字段稀疏、有图片；**precision 极端优先，绝不能误匹配，允许漏匹配**。

---

## 1. 选题与去重

执行前检查 `奢侈品调研/c/`，已存在的分析包括：

1. Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes
2. AnyMatch – Efficient Zero-Shot Entity Matching with a Small Language Model
3. Confidence Classifiers with Guaranteed Accuracy or Precision
4. Conformal Selective Prediction with General Risk Control
5. DeepBlocker
6. Efficient Model Repository for Entity Resolution：Construction, Search, and Integration
7. End-to-end multi-modal product matching in fashion e-commerce
8. Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration
9. GraLMatch：Matching Groups of Entities with Graphs and Language Models
10. How to Fix a Broken Confidence Estimator：Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks
11. Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes
12. Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification
13. PAM：Understanding Product Images in Cross Product Category Attribute Extraction
14. Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce
15. Query Brand Entity Linking in E-Commerce Search
16. Tailoring entity resolution for matching product offers
17. TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency
18. Using LLMs for the Extraction and Normalization of Product Attribute Values
19. parts-distributor-sku-classifier
20. pyJedAI

本次选择 `奢侈品文章调研.md` 中尚未分析的 **MOON2.0**。调研清单对它的推荐理由是：电商图文数据天然存在模态不平衡、图文噪声和信息缺失，MOON2.0 通过动态模态平衡、图文内部对齐和数据去噪，让模型在 text-only、image-only、multimodal 三类输入上都能得到可比较的统一表示。

它与腕表需求的关系很直接：三个来源的数据字段覆盖不一致，有的记录 reference 在结构化字段、有的埋在标题、有的只能从表背/吊牌/保卡图片获得，因此候选召回不能假设“每条记录都有完整文字”。

但它也有一个非常重要的边界：

> **MOON2.0 是“通用商品表征 / 检索模型”，不是“同一 reference 的高精度裁决器”。**

因此本文不会建议“相似度超过阈值就自动合并”，而是把 MOON2.0 的能力放在 **Candidate Retrieval、图文一致性验证、缺失模态补偿和人工复核排序**，最终自动 MATCH 仍由 canonical reference 的强证据 Gate 决定。

---

## 2. 先给结论：哪些东西值得直接借，哪些不能照搬

### 值得直接借的 4 个核心思想

1. **Multimodal Joint Learning**：同时训练 text-only、image-only、image+text 三种输入，适合三个来源字段稀疏不一致。
2. **Dual-level Alignment**：不仅学习“商品 A 和商品 B 是否相关”，还显式学习单条商品内部 image ↔ text 的一致性；这对表盘、表背、保卡、吊牌与标题 reference 交叉验证非常重要。
3. **Dynamic Sample Filtering**：弱标签/伪正负样本训练时，根据当前 embedding 的正负 margin 动态降权，适合海量自动构造训练对的噪声控制。
4. **模态自适应路由**：不是固定 50% 图像 + 50% 文本，而是根据“当前样本到底有什么信息、质量如何”选择更合适的专家/投影。

### 不建议直接照搬的 3 点

1. **不要把 MOON2.0 embedding similarity 作为 MATCH 终判。** 腕表同系列不同 reference 外观可能极度接近，模型很容易把“视觉同款感”误当成“同 reference”。
2. **不要在 reference OCR / 决策链路里使用生成式图片细节增强。** 论文的 visual expansion 会做 logo / fine-grained detail refinement；对普通商品理解是增强，对 reference number 来说却存在生成/改写数字字符的不可接受风险。
3. **不要第一版复刻论文的全量 MoE MLLM。** 论文使用内部 generative MLLM、内部商品实体抽取工具，训练集约 575 万、测试集约 63.6 万，训练约 18 小时、64 张 A100。当前需求只有几百对黄金标签，直接重造论文规模的基础模型性价比很低。

### 本需求最适合的改造版

> **Reference-first MOON-lite**：
>
> - reference 抽取 / 规范化 / 角色识别：决定能不能自动 MATCH；
> - MOON 风格 multimodal embedding：只负责召回、补模态、图文一致性与 Review ranking；
> - Dual-level Alignment + same-series hard negative：让 embedding 学会“外观很像但 reference 不同”的危险边界；
> - 动态融合先用小型 router，不需要把整个 LLM FFN 改成 MoE；
> - 所有无法证明 reference 一致的候选一律 ABSTAIN，而不是靠相似度冒险。

---

## 3. MOON2.0 到底解决什么问题

MOON2.0 面向电商通用商品理解，指出现有 MLLM / 多模态检索存在三类系统性问题。

### 3.1 固定模态混合比例会造成 modality imbalance

典型训练会按固定比例混合：

```text
image-only query
text-only query
image + text query
```

但线上任务的模态分布不是固定的。

比如当前三源腕表：

- 腕表之家可能文本字段丰富；
- 二手商家标题可能短但图片很多；
- 某条记录只有商品名 + 几张图；
- 另一些记录 reference 独立字段很完整。

如果训练集固定按某个比例喂数据，模型会偏向训练中占优势的模态。MOON2.0 的动机就是让模型根据输入模态和训练目标动态分配专家能力，而不是假设所有商品都有同等质量的文字和图片。

### 3.2 只做 inter-product alignment 浪费了单商品内部监督

很多商品检索训练只学：

```text
query product  <-> positive product
query product  <-> negative product
```

但一条电商商品自身的图片和标题已经是一对天然的弱监督：

```text
product image <-> product title
product image <-> product attributes
```

对腕表尤其有用：

```text
标题：劳力士 潜航者 126610LN 黑水鬼
图片：表盘 / 表背 / 保卡 / 吊牌
```

如果图片和标题对齐能力足够强，就能用来发现：

- 图片实际上是另一 reference；
- 标题里出现的是“兼容某型号”而不是本商品型号；
- 卖家把盒证/配件 reference 写进标题；
- OCR 出来的 reference 与标题 reference 冲突。

### 3.3 原始电商数据本身就是 noisy supervision

论文指出训练日志里会存在伪正、伪负、错误商品内容、噪声标题、背景杂乱图片等问题。

当前三源数据同样更严重：

- 抓取字段不完整；
- seller 标题不规范；
- reference 中存在连字符/空格/大小写差异；
- 可能混入平台 SKU、内部货号、库存编号；
- 配件/盒证标题会出现目标腕表型号；
- 图片可能是拼图、宣传图、保卡、表盒甚至其他商品。

所以训练数据不能“自动生成完就全信”，必须让训练过程具备降噪机制。

---

## 4. MOON2.0 技术架构深入拆解

## 4.1 统一 MLLM 表征：把 text / image / multimodal 放进同一空间

对每个 query / positive / negative，论文都会构造三种输入视图：

```text
x^t   = text only
x^i   = image only
x^mm  = image + text
```

文字经过 tokenizer 得到 text embedding；图片经过 vision encoder + projector 变成 visual tokens；随后两类 token 进入同一个 LLM backbone。

最后不是用生成 token 做分类，而是取最后一层 hidden states 后做 mean pooling：

```text
text tokens ----┐
                 ├-> LLM / MoE -> last hidden states -> mean pooling -> r ∈ R^D
visual tokens --┘
```

论文形式化为：

```text
h ∈ R^((2L + (nc+1)V) × D)
r = MeanPool(h)
```

这里 `r` 就是商品 embedding。

**对当前系统的价值**：一条记录缺文字或者缺图时，不需要换模型；text-only、image-only、multimodal 可以进入统一候选空间。

**但当前系统无需完全照搬 MLLM**。千万级商品每次都跑大 LLM 成本较高，生产上更合理的是：

- 训练阶段可使用较强 teacher；
- 推理阶段使用轻量 image encoder + text encoder + 小型融合/router；
- 或把强 MLLM embedding 预计算后蒸馏到 256/384 维 student embedding。

候选召回只需要稳定的 metric space，不需要在线生成文本。

---

## 4.2 Modality-driven MoE：不仅 token 路由，还让专家对“任务方向”产生偏好

MOON2.0 把 LLM 的 FFN 替换为多个 expert MLP。

普通 MoE 对 token 做：

```text
G = softmax(Wg h)
ĥ = Σ Gz · fz(h)
```

即每个 token 由 gating network 决定激活哪些专家。

MOON2.0 进一步认为，仅 token-level routing 不够，因为不同对齐任务对模态要求不同：

```text
text query  -> multimodal positive
image query -> multimodal positive
mm query    -> multimodal positive
image       -> text (同商品内部)
text        -> image (同商品内部)
```

所以论文为每个 expert 再维护一个对 alignment objective 的可学习偏好矩阵：

```text
W* ∈ R^(Z × M)
```

其中：

- `Z` = expert 数量；
- `M` = alignment objective 数量。

对每个 expert，其 objective preference 经过 softmax；再把“token 实际路由权重”和“expert 对某个 objective 的偏好”合起来，得到该 objective 的动态权重 `ωm`。

此外还加两个正则：

1. `L_aux`：常规 MoE load balancing，避免少数 expert 被打爆；
2. `L_sparsity`：最小化 expert 的 objective preference entropy，让 expert 真正形成专长。

### 为什么这个设计适合稀疏字段

可以把当前商品分成不同“证据形态”：

```text
A. reference 独立字段 + 文本完整 + 多图
B. 无 reference 字段，标题有 reference
C. 标题无 reference，图片/保卡 OCR 有 reference
D. 文本很弱，只能图片召回
E. 图片是噪声图，但文字可靠
```

固定融合层难以对这些形态都最优。动态 router 可以学到：

- C/D 更依赖 image expert；
- B/E 更依赖 text expert；
- A 依赖 multimodal consistency expert。

### 但 MVP 不需要全 LLM MoE

更实际的轻量版本：只在最终 projection/fusion 层做 3~4 个 expert。

```text
z_t  = TextEncoder(title + attrs)
z_i  = ImageEncoder(images)
z_ocr = Ref/OCR encoder(ocr_tokens)

quality = [has_text, has_image, has_ref_field,
           text_len, ocr_conf, image_count, image_quality]

g = softmax(Router(quality))

z = normalize(
      g_t   * P_t(z_t)
    + g_i   * P_i(z_i)
    + g_ocr * P_ocr(z_ocr)
)
```

这保留了 MOON2.0 的“按模态质量动态分工”思想，但训练/推理成本比修改整个 MLLM 小很多。

---

## 4.3 Dual-level Alignment：这篇论文最值得优先复用的部分

MOON2.0 同时做两层对齐。

### 第一层：Inter-product Alignment

训练三元组：

```text
(q, p, n)
```

`q` 为 query，`p` 为正商品，`n` 为负商品。

分别让：

```text
q_text  -> p_mm
q_image -> p_mm
q_mm    -> p_mm
```

都比 negative 更近。

论文用温度缩放的 contrastive loss；不同模态目标由前述 MoE 产生动态权重。

### 第二层：Intra-product Alignment

额外让同一商品自身：

```text
image(product) <-> text(product)
```

靠近，而与另一个商品的 text/image 拉开。

这层对当前需求非常重要，因为我们真正需要的不是“像不像腕表”，而是：

> **这张图上的细节是否支持这段标题/属性确实描述当前这块表。**

### 论文 ablation 表明 Alignment 是最大贡献项之一

MBE2.0 的消融中：

```text
完整 MOON2.0
text -> image R@10 = 73.12
image -> text R@10 = 64.91

去掉 Dual-level Alignment
text -> image R@10 = 31.45
image -> text R@10 = 23.35
```

下降非常大。

因此如果资源有限，我会把实现优先级排成：

```text
reference hard gate
    > reference-aware Dual-level Alignment
    > 轻量动态模态融合
    > 动态样本过滤
    > 生成式 co-augmentation
    > 全量 LLM MoE
```

---

## 4.4 Image-text Co-augmentation：思路可借，但必须“reference-preserving”

论文的数据增强分两部分。

### Textual Enrichment

原始标题 `T`、描述 `D`、图片 `I` 先经过内部实体抽取工具产生实体集合 `E`，再由 MLLM 生成 enriched title：

```text
T+ = MLLM_text(T, I, E)
```

目标是把短、噪声多的电商标题扩成语义更完整的描述。

### Visual Expansion

论文会：

1. 从原图抽出 main subject；
2. 改背景；
3. 改视角/光照；
4. 对 logo / fine-grained details 做 refinement；
5. 用 CLIP 做 image-title consistency filtering。

### 腕表 reference 场景必须做安全改造

**生成式增强绝不能进入 reference 证据链。**

原因很简单：

```text
126610LN
126610LV
116610LN
```

只差少数字符，但业务上是不同 reference。任何“细节增强”如果改了表背刻字、吊牌、保卡或 logo 周边像素，都可能制造假证据。

因此建议拆成两个数据域：

```text
Original Evidence Domain
- 原始标题
- 原始结构化字段
- 原始图片
- 原图 OCR
- 用于 MATCH Gate

Augmented Training Domain
- 背景移除
- 非语义 crop
- 亮度/对比度/压缩扰动
- 文本实体补全
- 仅用于 embedding 训练和候选召回
```

增强图从来不能生成 `reference_evidence`。

文本增强也必须加约束：

> MLLM 可以重组/补充语义，但如果输出新的 reference，只有当该字符串能在**原始字段或原图 OCR** 中找到可回溯证据时才允许写入候选 reference；否则只当普通语义文本，不能进入硬匹配。

这可以杜绝 hallucinated reference 进入数据库。

---

## 4.5 Dynamic Sample Filtering：非常适合海量弱监督对

论文为每个训练 triplet 计算：

```text
Δ = sim(q, p) - sim(q, n)
φ = sigmoid(κ(Δ - Δ_bar))
```

并使用阈值 `δ = 0.6`。`Δ_bar` 随训练过程衰减：

- 早期只信 margin 大的高置信样本；
- 后期逐渐引入更难样本。

`φ < δ` 的三元组会被降权，从而抑制 pseudo-positive / pseudo-negative。

### 当前业务可以更进一步：加入 reference 规则的先验过滤

训练时不要只看 embedding margin，还应先做 deterministic filtering：

```text
if high_conf_ref(q) != high_conf_ref(p):
    positive_weight = 0        # 绝不允许伪正

if high_conf_ref(q) == high_conf_ref(n):
    negative_weight = 0        # 绝不允许伪负
```

然后再对剩余弱标签使用 MOON2.0 的动态 reliability。

也就是说：

> **reference 规则负责“硬纠错”，Dynamic Sample Filtering 负责“软降噪”。**

---

## 5. 论文实验告诉我们什么

### 5.1 数据规模

论文从淘宝真实日志构造：

- 训练：5,751,594 条；
- 测试：636,241 条；
- 合计约 638 万；
- 包含 image query、text query、multimodal query；
- 正样本来自购买行为，负样本来自低相关曝光。

这个规模和当前 100 万–1000 万商品量级是同一数量级，所以论文的“多模态表征用于大规模检索”具有工程参考价值。

### 5.2 训练成本说明为什么不要直接复刻

论文使用内部 generative MLLM，单阶段 SFT：

- learning rate `1e-5`；
- cosine scheduler；
- batch size 4 / GPU；
- 64 × NVIDIA A100；
- 约 18 小时。

当前需求没有必要先承担这个成本。首先把 reference pipeline 做对，通常能解决绝大部分 precision 问题；多模态模型是解决“缺字段时如何召回 / 如何发现冲突”的第二层基础设施。

### 5.3 消融结果给出的工程优先级

论文 Table 4：

```text
                     t->mm R@10   i->mm R@10   mm->mm R@10  t->i R@10  i->t R@10
MOON2.0                 63.09         91.08         94.21       73.12      64.91
w/o MoE                 51.29         74.59         78.45       62.16      56.21
w/o Alignment           37.99         65.72         67.45       31.45      23.35
w/o Co-augmentation     59.69         78.17         80.62       64.79      58.68
w/o Filtering           60.63         83.40         80.00       70.40      63.21
```

最值得注意：

- Dual-level Alignment 的贡献非常大；
- MoE 也有明显价值；
- co-augmentation 有价值，但不是第一优先级；
- filtering 对原始电商噪声有稳定增益。

因此当前项目完全可以先做一个 **MOON-lite**，把大部分收益集中在“多视图对比学习 + 图文内部对齐 + hard negative”。

---

## 6. MOON2.0 与当前 Spec 的本质差异

论文的优化目标主要是：

```text
R@K / retrieval quality
classification accuracy / F1
attribute prediction
```

当前系统的业务目标却是：

```text
same product := same reference
false positive ≈ 0
允许大量 abstain / false negative
```

这是一个非常关键的目标错位。

### 为什么相似度阈值永远不能独立终判

腕表存在典型 hard negative：

```text
同品牌
同系列
同尺寸
同配色
几乎相同外观
reference 只差 1~2 个字符
```

多模态表示模型的目标恰恰会把这类商品映射得很近，因为从“商品语义”看它们确实接近。

而业务定义要求：

```text
视觉相似 0.999
文本相似 0.999
但 reference 不同
=> 必须 REJECT
```

因此本项目必须明确分离两个分数：

```text
semantic_similarity  # 用来召回
reference_evidence    # 用来裁决
```

这两个不能混成一个概率。

---

# 7. 建议直接落地的系统：Reference-first MOON-lite

## 7.1 总架构

```text
                         ┌──────────────────────────────┐
                         │ Raw Product Lake             │
                         │ 雷小安 / 腕表之家 / 奢当家   │
                         └──────────────┬───────────────┘
                                        │
                                        v
                         ┌──────────────────────────────┐
                         │ Normalization                │
                         │ schema / brand / text / img  │
                         └──────────────┬───────────────┘
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 │                                             │
                 v                                             v
      ┌─────────────────────────┐                  ┌─────────────────────────┐
      │ Reference Evidence      │                  │ MOON-lite Embedding     │
      │ - structured field      │                  │ - text                  │
      │ - title / desc          │                  │ - image                 │
      │ - original-image OCR    │                  │ - multimodal/router     │
      │ - role classifier       │                  │ - 256/384d vector       │
      └─────────────┬───────────┘                  └─────────────┬───────────┘
                    │                                            │
                    v                                            v
      ┌─────────────────────────┐                  ┌─────────────────────────┐
      │ Exact Ref Index         │                  │ ANN Index               │
      │ (brand, canonical_ref)  │                  │ brand/category shard    │
      └─────────────┬───────────┘                  └─────────────┬───────────┘
                    │                                            │
                    └──────────────────────┬─────────────────────┘
                                           v
                              ┌──────────────────────────┐
                              │ Candidate Union          │
                              │ exact + lexical + ANN    │
                              └────────────┬─────────────┘
                                           v
                              ┌──────────────────────────┐
                              │ Evidence / Conflict Gate │
                              │ reference is authority   │
                              └──────────┬───────┬───────┘
                                         │       │
                       MATCH <────────────┘       └──────> REJECT
                                         │
                                         v
                                      ABSTAIN
                                         │
                                         v
                                  Human Review Queue
```

核心原则：

> **MOON-lite 可以决定“看哪几个候选”，但不能决定“是不是同一个 reference”。**

---

## 7.2 Reference Evidence Service：真正的核心服务

每条商品不要只存一个 `reference` 字符串，而是存**证据集合**。

推荐结构：

```json
{
  "source": "watch_home",
  "item_id": "xxx",
  "brand_id": "rolex",
  "reference_candidates": [
    {
      "raw": "126610-LN",
      "canonical": "126610LN",
      "role": "manufacturer_reference",
      "channel": "title",
      "confidence": 0.998,
      "span": [12, 21],
      "verified_in_original": true
    },
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "role": "manufacturer_reference",
      "channel": "image_ocr",
      "image_id": "img_03",
      "bbox": [101, 86, 244, 132],
      "confidence": 0.991,
      "verified_in_original": true
    }
  ]
}
```

### reference canonicalization

建议按品牌做 grammar-aware canonicalization，而不是全局 fuzzy normalize。

基础步骤：

```text
Unicode NFKC
-> uppercase
-> 清理明显排版分隔符
-> 统一全/半角
-> 保留字母数字顺序
-> brand-specific pattern validate
-> 输出 canonical_ref
```

例如：

```text
126610 LN
126610-LN
126610LN
```

可以规范到：

```text
126610LN
```

但绝不能把：

```text
126610LN
126610LV
```

因为编辑距离近就归并。

### 编号角色分类必须在 reference 前

腕表数据里经常同时出现：

```text
品牌 reference
平台商品 ID
店铺 SKU
库存号
订单号
表带/配件编号
盒证编号
```

所以每个候选编号都要先分类：

```text
manufacturer_reference
platform_sku
seller_sku
inventory_id
accessory_reference
unknown_identifier
```

只有 `manufacturer_reference` 能进入自动匹配 Gate。

### MLLM 只能提候选，不能凭空制造 reference

如果用 MLLM 从长文本抽 reference，必须执行：

```text
LLM candidate
-> exact substring / normalized substring 回查原始文本
   OR original-image OCR bbox 回查
-> brand grammar validation
-> role classification
-> 才能成为 reference_evidence
```

LLM 输出但原文/原图找不到的 reference：

```text
verified_in_original = false
=> 永不用于 AUTO_MATCH
```

---

## 7.3 Candidate Generation：exact index 为主，MOON-lite ANN 为补充

### 通道 A：Exact Reference Index

Key：

```text
(brand_id, canonical_reference)
```

Value：

```text
[source:item_id ...]
```

只要两条记录都有高置信 canonical reference，这是最高价值、最低风险、最低成本的召回方式。

### 通道 B：Lexical / Character Retrieval

对疑似 reference、标题、OCR token 做：

- char n-gram；
- exact token；
- 品牌内 prefix / suffix；
- BM25 / inverted index。

用途是找格式不一致或 OCR 有轻微噪声的候选，**仍不直接 MATCH**。

### 通道 C：MOON-lite Multimodal ANN

只在以下情况启用或提高权重：

- 一侧 reference 缺失；
- OCR 只能得到模糊字符；
- 标题非常短；
- 需要给人工复核排候选。

ANN 一定先做 brand / category blocking：

```text
brand_id = Rolex
category = watch
-> ANN topK = 50 / 100
```

不要让全库所有品牌直接互搜。

---

## 7.4 Evidence / Conflict Gate：precision-first 的唯一终判层

推荐状态不是二分类，而是三态：

```text
MATCH
REJECT
ABSTAIN
```

### AUTO_MATCH 规则

最保守版本：

```python
def decide(a, b):
    if a.brand_id != b.brand_id:
        return "REJECT"

    a_refs = high_conf_main_refs(a)
    b_refs = high_conf_main_refs(b)

    if has_internal_ref_conflict(a) or has_internal_ref_conflict(b):
        return "ABSTAIN"

    if a_refs and b_refs:
        if a_refs.isdisjoint(b_refs):
            return "REJECT"

        if len(a_refs) == 1 and len(b_refs) == 1:
            # 同一 canonical manufacturer reference
            return "MATCH"

    return "ABSTAIN"
```

这里 `semantic_similarity` 完全没有 MATCH 权限。

### 明确的冲突优先级

只要出现以下任意一个高置信冲突，就算图片再像也不能自动合并：

```text
reference conflict
brand conflict
manufacturer_reference vs accessory_reference
同记录多 reference 无法确定主 reference
原始 OCR 与标题 reference 冲突
```

### 图片的正确权限

图片可以：

- 帮助 OCR 出 reference；
- 判断标题/图片是否明显不一致；
- 排人工候选顺序；
- 召回缺字段的潜在同 reference 商品。

图片不可以：

- 在 reference 明确冲突时覆盖冲突；
- 在 reference 缺失时单独触发 AUTO_MATCH。

---

# 8. MOON-lite 的训练设计

## 8.1 不需要几百万人工标注：自动弱监督 + 几百黄金 hard cases

只要 reference pipeline 已有一批高置信结果，就可以自动生成训练数据。

### 正样本

```text
same brand + same canonical reference
```

并优先取跨来源：

```text
雷小安 <-> 腕表之家
雷小安 <-> 奢当家
腕表之家 <-> 奢当家
```

### 最重要的 hard negative

```text
same brand
same series / family
high visual similarity
BUT different canonical reference
```

例如：

```text
126610LN vs 126610LV
116610LN vs 126610LN
```

这种负样本比随机不同品牌负样本有价值得多。

### 黄金标签几百对怎么花

不要平均随机抽。

建议黄金集主要覆盖：

1. 同 reference、格式差异极大；
2. 同系列相邻 reference；
3. 图片非常像但 reference 不同；
4. 标题包含配件兼容型号；
5. 平台 SKU 被误识别为 reference；
6. reference 只在图片/保卡/吊牌；
7. 一条商品内部标题 reference 与图片 OCR 冲突；
8. 新品牌 / 新来源增量。

黄金标签主要用于：

- 校准 reference extractor precision；
- 验证 hard-negative false accept；
- 调候选召回阈值；
- 评估人工队列排序；

而不是拿几百对强行训练一个通用大模型。

---

## 8.2 Reference-aware Dual-level Alignment

对 MOON2.0 做一个业务定制。

### Inter-product

```text
positive: same canonical reference
negative: same brand but different reference
```

损失仍使用 contrastive / InfoNCE。

### Intra-product

让：

```text
product image <-> original title
product image <-> OCR/entity text
```

对齐。

但如果本商品存在内部 reference 冲突，则这条样本不进入高权重 intra-product 正样本。

### 新增 Reference Margin Loss

针对危险 hard negative，单独增加：

```text
L_ref = max(0,
            margin
          + sim(q, hard_negative_diff_ref)
          - sim(q, positive_same_ref))
```

最终：

```text
L_total = L_inter
        + λ1 * L_intra
        + λ2 * L_ref
        + λ3 * L_router_balance
        + λ4 * L_filter
```

`L_ref` 的目的不是让所有不同 reference 都很远，而是明确压制“同系列近邻型号”这类 false-positive 高风险 pair。

---

## 8.3 轻量 Modality Router 代替全量 MoE

第一版建议：

```text
Text Encoder
Image Encoder
OCR / Ref Encoder
    │
    └-> three projected vectors

Quality Features
    │
    └-> Router -> [w_text, w_image, w_ocr]

weighted sum -> 384d normalized embedding
```

Quality features 可以包括：

```text
has_title
text_length
has_description
has_structured_ref
ref_confidence
image_count
image_resolution
ocr_char_count
ocr_confidence
image_text_consistency
```

这比只用 `modality_mask` 更适合真实抓取数据，因为“有图片”不代表图片质量高，“有文字”也不代表文字有用。

### 第二阶段才考虑真正 MoE

如果后续数据证明简单 router 有明显瓶颈，再升级：

- 4~8 个 projection experts；
- top-2 gating；
- expert load balancing；
- objective preference matrix；
- 对 text→mm、image→mm、mm→mm、image↔text 分配不同 expert preference。

不建议第一版就修改大 LLM 的所有 FFN。

---

# 9. 数据增强策略：只做不改变 reference 的增强

## 推荐

### 图像

- 主体 crop；
- 背景移除；
- 轻微亮度/对比度；
- JPEG 压缩；
- 轻微旋转；
- 多图随机采样；
- 保持表盘、表背、吊牌、保卡像素真实。

### 文本

- 标题 + 描述实体拼接；
- brand / series / material / size 等可验证属性补齐；
- 分隔符扰动；
- reference formatting augmentation，例如：

```text
126610LN
126610-LN
126610 LN
```

但是 label 始终绑定 canonical `126610LN`。

## 不推荐用于证据链

- 生成新的表背刻字；
- “高清修复” reference 数字；
- 生成式 logo/detail refinement；
- MLLM 自由补全一个原始记录不存在的型号。

---

# 10. 千万级部署架构

## 10.1 Feature 预计算，而不是 pairwise 大模型

每条商品只做一次：

```text
normalize
reference extract
OCR
text embedding
image embedding
multimodal embedding
```

之后跨源匹配只查 index。

千万级不应该做：

```text
10,000,000 × 10,000,000 pairwise cross-encoder
```

而应该做：

```text
new item
-> exact reference index
-> brand shard ANN topK
-> evidence gate
```

### embedding 存储量

若使用 384 维 FP16：

```text
10,000,000 × 384 × 2 bytes
≈ 7.7 GB raw / vector field
```

如果同时存 text/image/mm 三套，大约 23 GB raw，尚未包含 ANN 图结构开销。

因此生产上可选择：

- 只保留一个 fused 384d ANN 向量；
- text/image 原始向量落冷存储；
- 或用 int8 / product quantization；
- ANN 按 brand/category 分片。

---

## 10.2 增量更新

每条记录计算：

```text
content_hash = hash(normalized_text + image_hashes + structured_fields)
```

如果 hash 未变：

```text
skip feature recompute
```

如果变化：

```text
re-extract reference
re-run OCR if images changed
recompute embeddings
update exact index
update ANN
rerun affected candidate edges
```

### MATCH 结果必须可追溯版本

不要只存：

```text
A matched B = true
```

应存：

```json
{
  "a": "sourceA:item1",
  "b": "sourceB:item9",
  "decision": "MATCH",
  "decision_version": "ref_gate_v3",
  "brand_id": "rolex",
  "canonical_ref": "126610LN",
  "evidence_ids": ["ev_a_1", "ev_b_3"],
  "semantic_similarity": 0.982,
  "created_at": "..."
}
```

如果后续新 OCR 证据出现冲突，要能把 cluster 自动置为：

```text
QUARANTINED
```

而不是静默保留历史错边。

---

# 11. Cluster 层反而应该非常简单

既然业务定义就是同一 reference，那么最终 cluster key 不需要由图算法猜：

```text
cluster_key = (canonical_brand_id, canonical_reference)
```

只有通过高置信 evidence gate 的商品才进入这个 cluster。

缺 reference 的商品：

```text
unresolved
```

即使 ANN 和某 cluster 非常相似，也只是：

```text
candidate_cluster_id
```

不是正式成员。

这样可以避免典型的 transitive contamination：

```text
A ~ B
B ~ C
=> A,B,C 全并
```

而 C 实际是相邻 reference。

---

# 12. 人工复核应该看什么

Review 页面不要只显示一个“模型 98.7% 相似”。

应该展示证据矩阵：

```text
字段                    商品 A              商品 B
----------------------------------------------------------
brand                   Rolex               Rolex
canonical reference     126610LN            ?
结构化 ref              126610-LN           -
标题 ref                126610LN            -
图片 OCR                126610LN            126610LN ?
系列                     Submariner          Submariner
视觉相似度               0.982
图文一致性               0.94                0.91
冲突                     none                OCR low_conf
```

并直接给原图 OCR bbox。

人工只需要回答：

```text
B 的原始证据能否确认 manufacturer reference = 126610LN？
```

而不是主观判断“两块表看起来是不是一款”。

这样人工标签也能直接回流 Reference Evidence Service。

---

# 13. 评估指标必须改成 precision-first

论文主要看 R@K、Accuracy、F1；本项目上线指标应改成：

## 13.1 Reference Extraction

```text
ref_precision
ref_coverage
role_classification_precision
conflict_detection_recall
```

按：

```text
source × brand × extraction_channel
```

分别统计。

## 13.2 Candidate Retrieval

MOON-lite 只看候选层：

```text
candidate_recall@10
candidate_recall@50
candidate_recall@100
```

这层允许假阳性，目标是别漏掉潜在同 reference。

## 13.3 Final AUTO_MATCH

真正重要：

```text
auto_match_precision
false_accept_count
hard_negative_false_accept_rate
coverage / auto_match_rate
abstain_rate
```

上线 gate 应明确：

> 在专门构造的 same-brand / same-series / different-reference hard-negative 集合上，**不允许出现任何已知 false accept**；如果有一例，优先收紧 Gate，而不是为了 recall 放宽。

### 几百个黄金标签无法统计证明“绝对不会错”

这是必须直说的统计限制。

几百样本无法可靠证明一个 learned probability 达到 99.999% precision。

因此“绝不能误匹配”的主要保障不能来自：

```text
model score > 0.999
```

而必须来自：

```text
高置信原始 reference 证据 + deterministic conflict rules + abstention
```

模型只增加覆盖率和人工效率。

---

# 14. 建议的落地顺序

## P0：先做 reference truth layer

- canonical brand；
- reference grammar；
- 编号角色分类；
- structured/title/description/OCR 多通道抽取；
- evidence provenance；
- exact reference index；
- MATCH / REJECT / ABSTAIN Gate。

仅这一层就能产出第一批超高精度跨源 cluster，同时生成大量弱监督 pair。

## P1：加入 MOON-lite 候选召回

- text/image encoder；
- 384d fused embedding；
- brand/category ANN shard；
- topK candidate union；
- review ranking。

## P2：训练 Reference-aware Dual-level Alignment

- same reference positive；
- same-series different-reference hard negative；
- image-text intra alignment；
- dynamic sample filtering。

## P3：再决定是否需要真正 MoE

先观测不同模态质量桶：

```text
text-rich
image-rich
OCR-rich
multimodal-complete
```

如果简单 router 已足够，不必上大 MoE。

如果确实存在明显“某类输入始终被某模态拖累”，再引入 expert routing。

## P4：持续增量与主动采样

优先让人工标：

- 高 semantic similarity + reference conflict；
- 高 semantic similarity + reference missing；
- OCR 多候选；
- 新品牌 grammar 未覆盖；
- 新来源格式漂移。

这些样本的信息密度远高于随机抽样。

---

# 15. 一个可以直接实现的最小版本接口

## 15.1 `/extract_reference`

输入：

```json
{
  "brand": "Rolex",
  "title": "...",
  "description": "...",
  "structured_fields": {},
  "images": []
}
```

输出：

```json
{
  "brand_id": "rolex",
  "candidates": [
    {
      "canonical": "126610LN",
      "role": "manufacturer_reference",
      "channel": "title",
      "confidence": 0.998,
      "verified_in_original": true
    }
  ],
  "has_conflict": false
}
```

## 15.2 `/embed_product`

输出：

```json
{
  "vector": [0.1, -0.2, "..."],
  "dim": 384,
  "router_weights": {
    "text": 0.55,
    "image": 0.30,
    "ocr": 0.15
  },
  "quality": {
    "text": 0.92,
    "image": 0.77,
    "ocr": 0.44
  }
}
```

## 15.3 `/candidates`

```text
exact_ref candidates
UNION lexical candidates
UNION ANN topK within brand/category
```

## 15.4 `/decide_pair`

输出：

```json
{
  "decision": "MATCH",
  "reason": "same_high_conf_canonical_reference",
  "canonical_ref": "126610LN",
  "conflicts": [],
  "semantic_similarity": 0.982,
  "evidence_ids": ["ev1", "ev7"]
}
```

`semantic_similarity` 是审计字段，不是终判字段。

---

# 16. 与 MOON2.0 相比，这个改造为什么更适合当前需求

| 维度 | MOON2.0 原论文 | 当前建议 |
|---|---|---|
| 主目标 | 通用多模态表示 / 检索 | 同一 reference 的 precision-first matching |
| 输入 | text / image / mm | structured ref / raw text / OCR / image / mm |
| 模态融合 | LLM FFN 中 Modality-driven MoE | 先用轻量 quality-aware router，必要时再 MoE |
| Inter-product 正样本 | 用户行为相关商品 | same canonical reference |
| Hard negative | 日志负样本 | same brand/series + different reference |
| Intra-product | image ↔ text | original image ↔ title/OCR/reference evidence |
| 数据增强 | MLLM 图文共增强 | reference-preserving 增强；生成图永不做证据 |
| 动态过滤 | embedding margin | reference hard rule + embedding margin |
| 最终判断 | retrieval ranking | deterministic reference Gate + abstain |
| 缺 reference | 仍可检索 | 可召回/复核，但禁止仅凭相似度 AUTO_MATCH |
| Cluster | 检索关系 | `(brand_id, canonical_reference)` |

---

# 17. 风险清单

## 风险 1：reference 并非全局唯一

不同品牌可能复用同样数字，因此 cluster key 必须至少是：

```text
brand_id + canonical_reference
```

不能只用 reference。

## 风险 2：reference 字段本身可能写错

结构化字段也不能天然视为真值。

建议：

- 保存 provenance；
- 同记录多通道交叉验证；
- 若 structured/title/OCR 发生高置信冲突，直接 ABSTAIN / review。

## 风险 3：配件污染

标题：

```text
适用 Rolex 126610LN 表带
```

里面的 `126610LN` 不是当前商品 reference。

所以 role / product-type classifier 必须先识别：

```text
watch
strap
box
certificate
accessory
```

配件 reference 不进入 wristwatch cluster。

## 风险 4：视觉模型把相邻 reference 拉太近

这是预期行为，不是 bug。

解决：

- same-series hard negative；
- `L_ref`；
- semantic score 无终判权。

## 风险 5：OCR 数字混淆

典型：

```text
0/O
1/I
5/S
8/B
```

不要用激进 fuzzy correction 自动改 reference。

可以生成候选，但只有 brand grammar + 多通道支持后才升高置信度。

## 风险 6：生成式增强污染证据

所有生成图、增强标题必须和原始证据分库/分字段；审计界面永远优先显示 original source。

---

# 18. 最终建议

MOON2.0 对这个项目最有价值的地方，不是“我们也训练一个 64 A100 的多模态 MoE”，而是它给出了一个很清楚的工程思想：

> **面对 text-only、image-only、multimodal 混杂且带噪的数据，不应该固定相信某一种模态；应让表示模型动态利用当前可用的信息，同时显式学习单商品内部图文一致性。**

但是当前 Spec 的精度约束比论文严格得多，因此必须再加一层业务约束：

> **任何多模态表示都只是召回和证据辅助；同一商品的自动合并只能由可回溯到原始数据的 canonical manufacturer reference 一致性证明。**

综合起来，推荐落地为：

```text
Reference Truth Layer
    + Exact Ref Index
    + MOON-lite multimodal candidate retrieval
    + Reference-aware Dual-level Alignment
    + Same-series Hard Negative
    + Dynamic Sample Filtering
    + Deterministic Conflict Gate
    + ABSTAIN / Human Review
```

这套方案能同时满足：

- 千万级可扩展：embedding 和 reference 都预计算，ANN / inverted index 召回；
- 字段稀疏：text-only / image-only / mm 都可工作；
- 图片可利用：用于 OCR、图文一致性和候选召回；
- 几百黄金标签也有价值：集中用于 hard cases 和规则校准；
- 极端 precision-first：相似度无权越过 reference 冲突；
- 持续增量：新记录只重算自身特征和局部候选；
- 可审计：每个 MATCH 都能追溯到原始 reference evidence。

如果只实现一条最重要的原则，就是：

> **“模型负责找候选，reference 证据负责判生死。”**

---

## 参考资料

1. MOON2.0 arXiv v3（2026-07-29）：https://arxiv.org/abs/2511.12449
2. MOON2.0 HTML full text：https://arxiv.org/html/2511.12449
3. CVPR 2026 Open Access：https://openaccess.thecvf.com/content/CVPR2026/html/Nie_MOON2.0_Dynamic_Modality-balanced_Multimodal_Representation_Learning_for_E-commerce_Product_Understanding_CVPR_2026_paper.html
4. MBE2.0 Dataset：https://huggingface.co/datasets/ZHNie/MBE2.0
5. 当前调研清单：`/奢侈品文章调研.md`
6. 当前需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》
