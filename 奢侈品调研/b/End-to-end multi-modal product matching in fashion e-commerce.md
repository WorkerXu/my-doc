# End-to-end multi-modal product matching in fashion e-commerce：把 fashionID 改造成腕表 Reference 安全候选层与 HITL 精排层

- 分析人：b
- 调研文章：End-to-end multi-modal product matching in fashion e-commerce
- 作者：Sándor Tóth, Stephen Wilson, Alexia Tsoukara, Enric Moreu, Anton Masalovich, Lars Roemheld（Zalando）
- 论文地址：https://arxiv.org/abs/2403.11593
- HTML 全文：https://arxiv.org/html/2403.11593
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么这次选择这篇，而不是继续做一个普通 Entity Matching 模型

当前 Spec 的约束非常特殊：

1. 三个来源：雷小安、腕表之家、奢当家；
2. 规模 100 万～1000 万，并持续增量；
3. “同一个商品”被严格定义为 **同一 reference number / 型号**；
4. reference 可能在结构化字段里，也可能埋在标题、描述甚至图片中；
5. 图片可用；
6. **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
7. 可以人工标注几百对黄金标签。

此前 `b` 已分析过：

- `parts-distributor-sku-classifier.md`：解决“像型号的字符串到底是不是 manufacturer reference”；
- `Using LLMs for the Extraction and Normalization of Product Attribute Values.md`：解决 reference 抽取与规范化；
- `DeepBlocker.md`：解决千万级候选生成；
- `Ditto.md`：解决文本 pairwise entity matching；
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`：解决 selective prediction / 拒识；
- `TransClean.md` / `GraLMatch.md`：解决图级 false-positive 污染与多源实体组一致性。

但还缺一块非常工程化的拼图：

> **如何把多模态表示、候选检索、阈值拒绝和人工复核组织成一个真正能跑在生产环境里的端到端 matching pipeline。**

这正是 Zalando 这篇论文最值得借鉴的地方。

它的结论并不是“再训练一个更大的模型”，而是：

- 图像、文本、数值特征分别编码；
- 大预训练 encoder 尽量冻结；
- 只训练一个很小的 projection；
- 用对比学习建立统一 metric space；
- 大规模数据先 blocking，再 NN retrieval；
- 高置信结果直接过，边界结果交给 HITL；
- HITL 不是兜底客服，而是被定量建模成生产系统的一部分。

对于腕表项目，最重要的迁移不是照搬 fashionID 的“相似就是同款”，而是把它改造成：

> **reference 硬证据负责决定能不能自动合并；多模态模型只负责找候选、排序和发现冲突；人工负责处理仍然无法由 reference 硬规则闭合的黄色区域。**

这是本文下面方案的核心。

---

## 2. 原论文解决的生产问题

论文场景是多个电商 domain / seller 之间识别“同一商品”。每条 offer 包含：

- 一组商品图片；
- brand；
- title；
- price；
- 可售尺码数量；
- 部分商品代码。

不同 domain 对同一商品的图片风格、标题表达方式、拍摄角度和光照存在明显 distribution shift。

论文把问题拆成两步：

```text
offer -> encoder -> embedding
embedding -> blocking + nearest-neighbor retrieval -> match candidate
```

而不是对所有商品做笛卡尔积 pairwise classifier。

这个结构对当前 100 万～1000 万数据同样正确：

```text
10,000,000 x 10,000,000
```

任何全量两两比较都不可接受，必须把任务拆成：

```text
高召回候选生成 -> 高精度判定 / 拒识
```

但腕表场景比论文更严格：

```text
论文：视觉 + 文本高度相似，可以作为 match 的主要依据
当前需求：只有同 reference number 才算 match
```

因此我们只能迁移论文的“工程框架”，不能原样迁移它的“业务判定语义”。

---

## 3. fashionID 原始技术架构拆解

## 3.1 输入特征

论文对每个 offer 构建三类输入。

### 图像

一个 offer 平均有多张图片。每张图片分别过预训练 image encoder，然后对同一 offer 的图像 embedding 做平均池化：

```text
img_1 -> image encoder -> e1
img_2 -> image encoder -> e2
...
img_n -> image encoder -> en

image_embedding = mean(e1 ... en)
```

这使得一个 offer 不论有 1 张还是 8 张图，最终都变成固定维度表示。

论文比较了 CLIP 与 DINOv2，最终使用 CLIP ViT-bigG-14 体系的 image/text encoder 作为强基座。

### 文本

原论文只做很轻的预处理：

```text
normalize_unicode(brand + title)
```

然后进入 text encoder。

这点非常有启发：为了适配新 domain，不必一开始就给每个来源手写大量复杂文本规则；复杂规则应该集中在真正的硬字段，例如我们这里的 reference。

### 数值特征

论文额外使用：

```text
[number_of_sizes, log(number_of_sizes), log(price)]
```

得到 3 维 numeric vector。

对腕表不应照搬 size count，但可以替换成：

```text
log(price)
year_confidence
condition_bucket
case_size_mm
reference_source_reliability
ocr_reference_confidence
```

这些只作为辅助特征，不能推翻 reference 硬证据。

---

## 3.2 Late Fusion + 小型线性投影

fashionID 最有工程价值的选择，是**冻结大 encoder，只训练 projection**。

原结构：

```text
                   +-------------------+
images ----------> | frozen CLIP image | -- pooled image emb --+
                   +-------------------+                       |
                                                               |
brand + title ---> | frozen CLIP text  | ---- text emb --------+--> concat --> Linear --> 192-d
                                                               |
numeric features ----------------------------------------------+
```

论文最佳配置把拼接后的高维向量投影到 192 维 metric embedding。

线性层只有约 0.6M 参数，而大 encoder 不参与反向传播。

这样做有四个现实优势：

1. 可以提前离线算好 image/text embedding；
2. 训练 projection 时 GPU 显存主要花在小 embedding 和 batch 上；
3. 可以把 batch 做得非常大；
4. 新 encoder 替换时，工程成本较小。

论文报告：预计算输入 embedding 后，projection 在单张 NVIDIA A100 上 50 epoch 的典型训练时间约 10 分钟。

这与当前场景非常匹配：几百条高质量标签不适合从头训练大 VLM，但足够训练 / 校准一个小型融合层或二阶段打分器。

---

## 3.3 对比学习

论文使用 supervised contrastive loss。

直观目标：

```text
同商品 offer embedding -> 拉近
不同商品 offer embedding -> 推远
```

归一化 embedding 后，用 batch 内其他样本作为负例。

论文特别强调两件事。

### A. batch 采样围绕 product id，而不是纯随机

随机抽样时，batch 中 positive pair 太少。

所以它先抽 product id，再把同一个 product id 的所有 offer 放进 batch，最大化正对数量。

对腕表项目可直接迁移成：

```text
抽 canonical (brand, reference)
-> 放入三个来源中属于该 reference 的多条记录
-> 再加入“同品牌、同系列、近似 reference”的困难负例
```

甚至应该比论文更激进地构造 hard negatives，例如：

```text
Rolex 126334 vs 126300
Rolex 126610LN vs 126610LV
Omega 210.30.42.20.01.001 vs 210.30.42.20.03.001
同 reference 的表 vs 该 reference 的表带 / 盒证 / 配件
```

随机负例几乎没有训练价值；真正决定 precision 的是这种“长得非常像、字符串也非常像，但 reference 不同”的边界样本。

### B. 大 batch 替代复杂 hard-negative mining 的一部分

论文研究到 32k batch，最终固定使用 16k；大 batch 明显改善对比学习表现。

它的关键工程前提就是“大 encoder 冻结 + embedding 预计算”，否则很难用这么大的 batch。

对我们而言，不一定需要追求 16k，但应继承这个思想：

> **把重计算变成离线特征，把训练阶段压缩成小 embedding 上的轻量学习，从而把预算用于更多高价值 pair。**

---

## 3.4 Retrieval：blocking + NN + threshold

原论文的 retrieval 架构是：

```text
query offer
    |
    v
brand fuzzy blocking
    |
    v
candidate brand subset
    |
    v
nearest-neighbor search by cosine similarity
    |
    v
score threshold
    |
    +--> accept candidate
```

论文明确指出，直接在大 index 上做全量 NN 不现实，因此先做 brand blocking。

这个结构可以迁移，但腕表需要把 blocking 做得更“硬”。

原论文：

```text
brand fuzzy similarity > threshold
```

腕表应至少是：

```text
canonical brand 一致
AND
候选 reference compatible
AND
product_type 不冲突
```

向量 ANN 只进入 reference 不确定的 fallback 路径。

---

## 3.5 人工闭环不是补丁，而是一个有指标的分类器

论文最值得当前 Spec 借鉴的第二点，是把 HITL 认真做成了一个系统组件。

最终 UI：

```text
左侧：query product
右侧：top-3 nearest neighbors
```

验证者从 top-3 中选正确 match，或者全部拒绝。

论文迭代得到几个重要经验：

1. 不要把所有特征都塞给人；color、price、size 等信息不一定帮助人工；
2. top-3 比只展示 top-1 更容易找回正确候选；
3. 高 precision 场景，多张图、尤其 close-up 图有帮助；
4. 专门训练的 validator 明显优于无训练 crowd；
5. validator 需要持续 error review 和反馈训练。

论文把人工验证也建模成二分类器，并用：

```text
LR+ = TPR / FPR
```

描述 validator 的增益。

最终输出 precision 依赖于输入模型 precision，而不是“人工一看就 100%”。

论文给出：

```text
P_hitl = [1 + (1 / P_model - 1) / LR+]^-1
```

这个公式非常值得当前项目保留，因为它告诉我们：

> **如果送给人工的候选池本身质量太差，再认真审核也不一定达到目标 precision。人工不是无限能力的 oracle。**

所以真正合理的系统是：先把黄色队列做到“高质量但不够自动放行”，再交给专家审核。

---

# 4. 这篇论文不能直接照搬的地方

这是本次方案最重要的边界。

## 4.1 视觉相似 ≠ 同 reference

时尚商品很多时候没有强唯一编码，视觉是主要识别信号。

但腕表不同。

两个不同 reference 可能：

- 表盘几乎一样；
- 表壳几乎一样；
- 同系列只差材质 / 盘色 / 表圈 / 机芯；
- 图片甚至直接复用官网图；
- 卖家可能上传错误图片。

因此：

```text
image similarity = 0.99
```

绝不能推出：

```text
same reference = true
```

图片只能：

- 帮 reference 缺失的记录找候选；
- 帮 OCR 找刻字 / 吊牌 / 保卡；
- 发现文本 reference 与图片视觉明显冲突；
- 给人工复核排序。

不能成为自动 merge 的最终证据。

---

## 4.2 原论文 threshold 是业务 precision threshold；我们需要硬逻辑 + threshold 双保险

原论文可以用 embedding distance threshold 接受同商品预测。

当前系统应该反过来：

```text
hard reference gate
    先决定“是否具备自动匹配资格”

model score
    再决定“候选排序 / 是否值得人工看”
```

也就是说模型不是裁判，而是搜索助手。

---

## 4.3 几百条黄金标签不足以统计证明“99.9% precision”

这是必须提前写进验收标准的现实问题。

假设黄金测试集中 300 个自动接受样本全部正确，即 0 false positive。

在独立同分布的理想假设下，单侧 95% Clopper-Pearson 下界约为：

```text
0.05^(1/300) ≈ 99.006%
```

也就是说：

> 300 个全对，只足够把 precision 的 95% 下界推到大约 99.0%，不能严格证明 99.9%。

如果仍然要求 95% 置信下证明：

```text
precision >= 99.9%
```

且测试里仍然 0 错误，需要大约：

```text
n >= log(0.05) / log(0.999) ≈ 2995
```

因此“几百对黄金标签”更适合：

- 训练 hard-negative / reference extractor；
- 初步选阈值；
- 构造 error taxonomy；
- 做第一阶段离线验证。

上线后仍需要持续 audit sampling 才能对极高 precision 做统计背书。

---

# 5. 为当前 Spec 设计的最终架构

下面给出我认为可以直接落地的版本。

```text
                     ┌──────────────────────────┐
                     │ 雷小安 / 腕表之家 / 奢当家 │
                     └────────────┬─────────────┘
                                  │
                                  v
                        [Raw Ingestion Layer]
                                  │
                                  v
                    [字段标准化 + 商品类型识别]
                                  │
                  ┌───────────────┴────────────────┐
                  │                                │
                  v                                v
        [Reference Evidence Pipeline]      [Multimodal Feature Pipeline]
        - structured ref                  - title embedding
        - title ref candidates            - image embedding
        - OCR ref candidates              - OCR text embedding
        - role classification              - numeric/meta features
        - canonicalization                 - frozen encoders
                  │                                │
                  └───────────────┬────────────────┘
                                  v
                         [Candidate Generator]
                  ┌───────────────┼────────────────┐
                  │               │                │
                  v               v                v
         exact ref index     ref-compatible     ANN fallback
                             blocker            within brand
                  │               │                │
                  └───────────────┬────────────────┘
                                  v
                         [Evidence Verifier]
                    hard rules + model score
                                  │
                 ┌────────────────┼─────────────────┐
                 │                │                 │
                 v                v                 v
              GREEN            YELLOW              RED
            auto-link        human review        reject
                 │                │
                 └────────────┬───┘
                              v
                      [Entity Graph / Cluster]
                              │
                              v
                    [Transitive Safety Audit]
                              │
                              v
                       [Canonical Entity]
```

核心原则：

> **GREEN 的自动合并必须能被 reference 证据解释；embedding 永远不能单独把记录送进 GREEN。**

---

# 6. 第一层：Reference Evidence Pipeline

这是整个系统最关键的一层。

建议每条记录都不要只保留一个 `reference` 字符串，而是保留**多个候选 + 来源 + 置信度 + 角色**。

推荐数据结构：

```json
{
  "record_id": "ldj_123",
  "brand_canonical": "ROLEX",
  "reference_candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "source": "structured_field",
      "role": "brand_reference",
      "confidence": 0.999
    },
    {
      "raw": "126610 LN",
      "canonical": "126610LN",
      "source": "title_parser",
      "role": "brand_reference",
      "confidence": 0.97
    }
  ]
}
```

而不是：

```json
{"reference": "126610LN"}
```

原因是 precision-first 系统必须知道：

```text
这个 reference 是谁说的？
来自平台结构化字段？
标题 regex？
OCR？
LLM？
是否可能其实是店铺 SKU？
```

## 6.1 reference source reliability 建议分级

例如：

```text
S0  品牌官方 / 已验证 reference 字典
S1  来源平台明确的品牌型号结构化字段
S2  标题中高精度规则抽取 + 品牌格式校验
S3  OCR 中抽取 + 与文字证据一致
S4  LLM / VLM 预测但无独立证据
```

自动 merge 至少要求 S0/S1/S2 级证据，或者两个独立较弱证据互相一致。

具体阈值必须用黄金标签校准，不能直接把上面的等级数字当概率。

---

# 7. Reference canonicalization：宁可拆开，不要过度归一化

reference 规范化是最容易“好心办坏事”的地方。

建议做两层 canonicalization。

## 7.1 安全规范化

允许：

```text
Unicode normalize
uppercase
去首尾空白
统一全角/半角
统一明确无语义的分隔符展示
```

例如：

```text
"126610 ln" -> "126610LN"
```

前提是该品牌已验证空格没有型号语义。

## 7.2 品牌特定 canonicalizer

不同品牌 reference 格式完全不同，所以不要使用一个全局正则暴力删除：

```text
- . / whitespace
```

例如某些品牌的小数点、斜线或后缀可能有业务含义。

建议：

```python
canonicalize(brand, raw_reference)
```

而不是：

```python
canonicalize(raw_reference)
```

并保留：

```text
raw_reference
canonical_reference
canonicalizer_version
```

以后规则升级时可以重放，不会失去原始证据。

---

# 8. 第二层：Candidate Generator

候选生成分三条路径。

## 8.1 Path A：Exact Reference Index

这是主路径。

索引 key：

```text
(brand_canonical, reference_canonical)
```

如果新记录得到高可信 reference：

```text
new record
  -> exact lookup (brand, reference)
  -> 返回其他来源同 key 记录
```

复杂度近似从两两比较降为 key lookup。

在当前业务定义下，这是最应该优先投入的路径。

---

## 8.2 Path B：Reference-compatible blocker

有些记录可能抽出多个候选：

```text
126334
126300
```

或 OCR 有字符混淆：

```text
O / 0
I / 1
S / 5
```

这类不能直接 canonicalize 成同一个 reference。

正确做法是生成小候选集：

```text
brand exact
series compatible
reference edit structure compatible
```

然后进入 verifier / 人工，而不是把 edit distance 当 match。

---

## 8.3 Path C：fashionID-style ANN fallback

仅当：

```text
reference 缺失
OR
reference 证据弱
OR
多个 reference 冲突
```

才进入多模态 ANN。

建议输入：

```text
image embedding
text embedding(brand + title + description)
OCR text embedding
numeric/meta features
```

融合结构沿用论文思路：

```text
concat -> Linear projection -> 128/192-d normalized vector
```

再按 brand / product_type 分区 ANN：

```text
query embedding
  -> brand partition
  -> topK = 20
  -> evidence verifier
```

这层目标只有一个：

> **不要漏掉值得检查的候选。**

而不是直接决定 same reference。

---

# 9. 腕表版 multimodal encoder：建议只训练小 projection

可以直接复刻论文的低成本思路。

## 9.1 图像 encoder

使用冻结的通用图像 / vision-language encoder。

同一商品多图：

```text
front
back
clasp
certificate
box
wrist shot
```

不要简单只做全局 mean，建议第一版同时保留：

```text
mean_pool_embedding
max_similarity_image
```

因为 reference 可能只出现在某一张表背 / 保卡 / 吊牌图中。

后续如果数据量足够，再研究 attention pooling。

## 9.2 文本 encoder

输入建议拆两路：

```text
semantic_text = brand + series + title + description
identifier_text = extracted reference candidates + OCR identifier tokens
```

不能把 identifier 完全混在自然语言里让大模型自己“领会”。

reference 是离散 identifier，应单独编码并进入 hard feature。

## 9.3 numeric / categorical features

例如：

```text
log_price
case_size
material
movement_type
product_type
source_id
reference_evidence_count
reference_conflict_flag
```

`source_id` 可以帮助模型学习不同来源的表现差异，但必须防止模型过拟合来源捷径。

## 9.4 projection

第一版坚持：

```text
frozen encoders + small linear projection
```

不要一开始 fine-tune 整个 VLM。

理由和论文一致：

- 数据少；
- 域不断变化；
- 需要快速迭代；
- 大 encoder embedding 可缓存复用；
- 线上 inference 成本更可控。

---

# 10. 训练集：几百对应该怎么标才最值钱

如果预算只有几百对，不能随机抽。

建议至少 70% 用于 hard negatives。

## 10.1 Positive

```text
同品牌
同 reference
不同来源
标题表达差异大
图片风格差异大
字段缺失不同
```

## 10.2 Hard Negative

优先：

```text
同品牌 + 同系列 + reference 只差 1~3 个字符
同外观 + 不同 reference
同 reference 字符串出现在配件标题里，但售卖主体不是表
平台 SKU 很像品牌 reference
标题 ref 与结构化 ref 冲突
OCR 把 0/O、1/I 读错
同一系列不同尺寸
同一系列不同材质
同一系列不同表盘颜色
```

## 10.3 标注 unit 不是只要 pair label

建议每个黄金 pair 同时标：

```json
{
  "same_reference": false,
  "left_reference": "126610LN",
  "right_reference": "126610LV",
  "left_reference_source": "title",
  "right_reference_source": "structured",
  "product_type_left": "watch",
  "product_type_right": "watch",
  "error_category": "same_series_different_variant"
}
```

这样几百对不仅能训练 matcher，还能训练 extractor、role classifier 和 error analyzer。

---

# 11. Evidence Verifier：不要训练一个“黑盒最终分类器”

建议最终判定输出一组解释性 feature：

```text
brand_exact
reference_exact
reference_source_strength_left
reference_source_strength_right
reference_conflict_left
reference_conflict_right
product_type_match
series_match
text_cosine
image_cosine
ocr_reference_agreement
price_ratio
ann_rank
```

然后使用：

```text
hard rule gate + calibrated lightweight model
```

而不是直接：

```text
VLM(pair) -> Match / NoMatch
```

---

# 12. GREEN / YELLOW / RED 三段式决策

## 12.1 GREEN：允许自动 link

第一版建议非常保守。

示例：

```text
brand canonical exact
AND
reference canonical exact
AND
双方至少有一个高可信 reference evidence
AND
双方无其他高可信 conflicting reference
AND
product_type compatible
AND
无已知 blacklisted parsing pattern
```

注意：

```text
image_similarity
text_similarity
```

不需要成为 GREEN 的必要条件，因为同 reference 跨来源图片文本可能差异很大。

但它们可以成为**冲突否决信号**：

如果 reference exact，但图片 / title 极端异常，可降级 YELLOW。

例如：

```text
结构化字段写 126610LN
但标题明显写“表带”
```

这时不能因为 reference 一致就自动合并成腕表实体。

---

## 12.2 YELLOW：人工复核

进入条件：

```text
reference 只有弱证据
多个 reference 冲突
OCR / title 只差疑似字符误识别
ANN 找到非常强候选但没有 reference
同 reference 但 product_type / title / image 有冲突
```

HITL UI 直接借鉴论文 top-3 设计。

建议界面：

```text
┌──────── query ────────┐    ┌──── candidate 1 ────┐
│ 多张图                │    │ 多张图               │
│ brand                 │    │ brand                │
│ raw title             │    │ raw title            │
│ ref candidates        │    │ ref candidates       │
│ evidence provenance   │    │ evidence provenance  │
└───────────────────────┘    └──────────────────────┘

                           candidate 2
                           candidate 3

[同 reference] [不同 reference] [无法判断]
```

与论文不同的是，腕表 validator 最需要看到的不是 price，而是：

- reference 原始文本；
- reference 出现位置；
- OCR crop；
- 表背 / 保卡 / 吊牌 close-up；
- 品牌 / 系列；
- 商品主体类型。

---

## 12.3 RED：直接拒绝

硬冲突直接 RED：

```text
brand conflict
两个高可信 reference 不同
watch vs strap/accessory
型号后缀明确不同且品牌规则确认有语义
同来源自有 SKU 被误识别为 reference
```

precision-first 的系统必须接受大量 RED / abstain。

---

# 13. 将 HITL 做成可测量组件

论文最大的工程经验之一，是验证者本身也需要评估 TPR / FPR。

建议每周 / 每批做 blind gold injection：

```text
真实人工队列中混入已知 gold pair
```

估计：

```text
validator TPR
validator FPR
LR+
```

并按 validator / 品牌 / error category 分桶。

不要只统计：

```text
每天审核了多少条
```

要统计：

```text
哪个品牌最容易误判？
哪类 reference 格式最容易错？
哪个 validator 在 same-series hard negative 上 FPR 高？
```

人工反馈可以回流：

```text
reference parser rules
brand-specific canonicalizer
projection training
model calibration
```

但不能把人工结果未经审计直接当绝对真值长期污染训练集。

---

# 14. 数据与存储设计

建议至少拆成五张逻辑表。

## 14.1 source_record

```sql
source_record(
  record_id,
  source,
  source_item_id,
  raw_title,
  raw_description,
  raw_brand,
  raw_reference,
  raw_payload,
  updated_at
)
```

## 14.2 normalized_record

```sql
normalized_record(
  record_id,
  brand_canonical,
  product_type,
  series,
  price,
  case_size,
  normalizer_version
)
```

## 14.3 reference_evidence

```sql
reference_evidence(
  record_id,
  raw_reference,
  canonical_reference,
  evidence_source,
  evidence_location,
  role,
  confidence,
  extractor_version
)
```

一条 record 可以有多行。

## 14.4 match_edge

```sql
match_edge(
  left_record_id,
  right_record_id,
  decision,
  decision_reason,
  model_score,
  reference_rule_version,
  model_version,
  reviewed_by,
  created_at
)
```

`decision`：

```text
AUTO_ACCEPT
HUMAN_ACCEPT
REJECT
ABSTAIN
```

## 14.5 entity_membership

```sql
entity_membership(
  entity_id,
  record_id,
  canonical_brand,
  canonical_reference,
  membership_source,
  created_at
)
```

实体主键优先直接映射：

```text
hash(brand_canonical + canonical_reference)
```

只要业务定义长期保持“同 reference 即同商品”，就不必让聚类模型自行发明 entity id。

---

# 15. 在线 / 增量处理流程

每来一条新记录：

```python
def process(record):
    n = normalize(record)

    ref_evidence = extract_reference_evidence(n)

    strong_refs = select_strong_brand_references(ref_evidence)

    if has_conflicting_strong_refs(strong_refs):
        return enqueue_yellow(record, reason="reference_conflict")

    if len(strong_refs) == 1:
        ref = strong_refs[0]
        candidates = exact_ref_lookup(n.brand, ref.canonical)

        safe = [c for c in candidates if hard_compatibility(n, c)]

        if safe:
            return auto_link(safe, reason="exact_reference")

    # reference 不足：才使用多模态候选
    emb = fashion_id_style_encoder(n)
    candidates = ann_search(
        brand=n.brand,
        product_type=n.product_type,
        embedding=emb,
        top_k=20,
    )

    ranked = evidence_verifier(n, candidates)

    if has_hard_conflict(ranked):
        return reject_or_abstain(ranked)

    return enqueue_yellow(top3(ranked))
```

这里的关键是：

```text
ANN 永远在 exact-ref path 之后
```

而不是把全部商品先向量搜索。

这能同时提高准确率和成本效率。

---

# 16. 规模化架构建议（100 万～1000 万）

一套不复杂但足够生产化的架构可以是：

```text
                    Kafka / Queue
                         |
        +----------------+----------------+
        |                                 |
        v                                 v
Reference Extractor                Feature Workers
(CPU / small LLM)                  (GPU batch)
        |                                 |
        v                                 v
PostgreSQL exact index           Object Store + Vector DB
        |                                 |
        +----------------+----------------+
                         v
                  Candidate Service
                         |
                         v
                  Decision Engine
               hard rules + calibration
                         |
              +----------+-----------+
              |                      |
              v                      v
        Entity Store             Review Queue
                                      |
                                      v
                                  HITL UI
```

## 推荐组件职责

### PostgreSQL / 等价关系数据库

存：

- `(brand, canonical_reference)` exact index；
- reference evidence；
- match decision；
- audit trail。

10M 级记录对数据库本身并不离谱，真正需要避免的是 10M² pair 表。

### Object Storage

存：

- 原始图片；
- OCR crop；
- frozen encoder embedding 文件 / 特征归档。

### Vector DB / FAISS service

只服务：

```text
reference 缺失 / 冲突的 fallback candidate retrieval
```

并按 brand / product type 分区。

### Queue

增量抓取后异步：

```text
normalize -> reference extraction -> feature extraction -> decision
```

不需要所有步骤同步阻塞采集。

---

# 17. 与 b 之前调研结果拼起来后的完整系统

这篇论文不是孤立使用，和之前结果可以形成完整链路。

```text
1. parts-distributor-sku-classifier
   ↓
   先判断一个编号是不是“品牌 reference”，避免 SKU 污染

2. Using LLMs for Extraction and Normalization
   ↓
   从标题/描述抽 reference，并做品牌感知 normalization

3. DeepBlocker / fashionID-style retrieval
   ↓
   reference 不足时，低成本生成候选

4. Ditto / lightweight verifier
   ↓
   对候选做文本/属性复核

5. Confidence Classifier
   ↓
   只自动接受满足极高 precision calibration 的部分，其他 abstain

6. 本文 fashionID + HITL
   ↓
   用多模态 retrieval 提高黄色队列质量，并用专家 top-3 UI 收口

7. TransClean / GraLMatch
   ↓
   在实体图层审计 false-positive 传递污染
```

最后的判定层永远保留：

```text
canonical reference exactness
```

作为业务定义的最高优先级约束。

---

# 18. 第一版可以直接落地的 MVP

不建议第一版就训练复杂 VLM。

## Phase 0：先建立黄金评测集

标 500～1000 对，优先 hard negatives。

产物：

```text
gold_pairs.jsonl
reference_error_taxonomy.md
brand_reference_rules.yaml
```

如果当前人工预算严格只有几百对，先从 300～500 对开始，但必须保留持续线上 audit 机制。

## Phase 1：只做 reference hard pipeline

实现：

```text
brand normalization
reference candidate extraction
role classification
brand-specific canonicalization
exact index
GREEN / YELLOW / RED
```

这一步很可能已经覆盖大部分结构化 reference 数据，而且风险最低。

## Phase 2：OCR reference evidence

只对：

```text
reference missing
reference conflict
high-value yellow cases
```

跑 OCR / VLM，不要全量无脑跑。

重点 crop：

```text
表背
保卡
吊牌
证书
盒贴
```

## Phase 3：fashionID-style multimodal fallback

冻结 image/text encoder，离线预计算 embedding。

训练一个：

```text
Linear(concat(image, text, numeric, ref_features)) -> 192-d
```

用 hard-negative oriented supervised contrastive training。

服务 top-20 ANN，仅供 YELLOW 候选。

## Phase 4：HITL UI

一次给专家：

```text
query + top3 candidates + 多张图 + reference provenance
```

输出：

```text
same reference
different reference
cannot determine
```

## Phase 5：图级安全审计

对已接受 edge 构成的 entity graph 跑：

```text
canonical reference consistency
source uniqueness check
TransClean-style transitive conflict check
```

任何 component 出现两个高可信 canonical reference：

```text
立即 quarantine
```

---

# 19. 关键 API 设计

## 19.1 reference extraction

```http
POST /v1/reference/extract
```

返回：

```json
{
  "brand": "ROLEX",
  "candidates": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "role": "brand_reference",
      "source": "title",
      "confidence": 0.982
    }
  ]
}
```

## 19.2 candidate retrieval

```http
POST /v1/match/candidates
```

优先 exact reference；没有才 ANN。

返回时强制标记候选来源：

```json
{
  "candidate_id": "...",
  "retrieval_reason": "EXACT_REFERENCE | ANN | OCR_COMPATIBLE",
  "ann_score": 0.91
}
```

## 19.3 decision

```http
POST /v1/match/decide
```

返回：

```json
{
  "decision": "AUTO_ACCEPT",
  "reason_codes": [
    "BRAND_EXACT",
    "REFERENCE_EXACT",
    "NO_REFERENCE_CONFLICT",
    "PRODUCT_TYPE_COMPATIBLE"
  ],
  "model_version": "fashionid-watch-v1",
  "ruleset_version": "ref-rules-2026-08-18"
}
```

不要只返回一个：

```json
{"score": 0.998}
```

生产审计必须能解释为什么合并。

---

# 20. 监控指标

不要用 overall F1 做主指标。

至少分四类。

## 20.1 自动接受质量

```text
AUTO_ACCEPT precision
AUTO_ACCEPT false positive count
precision lower confidence bound
```

这是最高优先级。

## 20.2 候选层

```text
Recall@1
Recall@3
Recall@20
```

只在有真实 match 的 query 上计算。

候选层允许 recall 高、precision 低，因为它不直接 merge。

## 20.3 拒识 / 人工

```text
abstention rate
review queue size
review precision
validator TPR / FPR
median review latency
```

## 20.4 分桶监控

按：

```text
source
brand
product_type
reference format
extractor version
model version
```

分别统计。

一个总体 99.9% 可能掩盖某个新品牌只有 92%。

---

# 21. 增量与分布漂移

论文专门设计了 out-domain / time-split 测试，说明生产 matching 不能只随机拆训练集。

当前项目应模仿：

## 21.1 时间外测试

```text
训练：过去数据
测试：未来新抓数据
```

检查标题风格、图片风格、卖家模板变化。

## 21.2 来源外测试

例如未来新增第四来源：

```text
训练完全不含第四来源
测试只用第四来源 query
```

这比随机 split 更能评估上线风险。

## 21.3 新品牌冷启动

新品牌没有品牌特定 reference 规则时：

```text
默认不自动 merge 弱 reference
-> YELLOW
-> 收集几十条 hard cases
-> 建 brand canonicalizer / parser
-> 再逐步开放 GREEN
```

precision-first 系统应采用“逐品牌解锁”，而不是一次性全局放开。

---

# 22. 哪些地方应该使用图片

图片不是最终 match key，但仍然很有价值。

## A. reference OCR

最高价值。

```text
表背刻字
保卡
吊牌
盒贴
```

## B. ANN fallback

reference 缺失时召回同款候选。

## C. 矛盾检测

例如：

```text
文本：Rolex Submariner 126610LN
图片：明显是 Cartier Tank
```

把原本可能 GREEN 的记录降成 YELLOW。

## D. 人工界面

展示 close-up，提高 validator 识别 reference / variant 的能力。

不建议：

```text
image cosine > 0.98 -> 自动 same reference
```

---

# 23. 一个非常实用的“安全分”而不是“相似分”

传统 matcher 输出：

```text
match_similarity = 0.997
```

建议改成：

```text
safety_score = P(no_false_positive | evidence)
```

输入包括：

```text
reference_exact
reference_provenance
reference_conflict
role_classifier_confidence
product_type_match
brand_match
ann_score
image_conflict_score
text_conflict_score
```

但即使 `safety_score` 很高，也必须先过 hard reference gate。

更准确地说：

```text
AUTO_ACCEPT = hard_gate AND calibrated_safety_score >= threshold
```

而不是：

```text
AUTO_ACCEPT = safety_score >= threshold
```

---

# 24. 失败模式清单

上线前必须专门造这些测试。

### 24.1 配件污染

标题：

```text
适用 Rolex 126610LN 表带
```

抽到 126610LN，但商品不是腕表。

预防：

```text
product_type gate
reference role/context classifier
```

### 24.2 多型号堆砌

标题：

```text
适配 126610LN / 126610LV / 116610LN
```

不能任选一个 reference。

### 24.3 平台 SKU 冒充 reference

长数字 / 字母数字串被 regex 误识别。

预防：

```text
role classifier
source-specific field semantics
```

### 24.4 字符过度规范化

把有意义后缀删掉：

```text
126610LN
126610LV
```

如果只保留数字就会灾难性误并。

### 24.5 图片复用

不同 reference 使用相同官网图。

所以视觉不能自动 merge。

### 24.6 标题与结构化字段冲突

不能“相信分数更高的那个然后继续”。

应该：

```text
conflict -> YELLOW
```

### 24.7 同 reference 不代表同一物理单品

当前 Spec 明确定义“同一个商品 = 同 reference”，因此系统实体是“型号实体”，不是“物理二手单品”。

这个语义必须写进数据模型，否则后续团队很容易误以为 entity_id 代表某一只具体手表。

---

# 25. 推荐的验收标准

第一阶段不要定“F1 > 0.9”这种目标。

建议：

```text
A. AUTO_ACCEPT
   1. 黄金集 false positive = 0
   2. 给出 precision 的置信下界
   3. 每个主要品牌单独报告

B. Candidate Retrieval
   Recall@20 >= 目标值
   但不能牺牲 hard blocking 安全规则

C. YELLOW
   人工队列占比可控
   validator FPR 持续监控

D. Graph
   一个 entity component 内不得存在两个不同的高可信 canonical reference

E. Auditability
   每次 AUTO_ACCEPT 都能重建：
   - 输入原始数据
   - reference evidence
   - 规则版本
   - 模型版本
   - decision reason
```

---

# 26. 最终建议

如果现在要从这篇论文里只拿三件东西，我会拿：

## 1. 冻结大模型，只训练轻量 projection

这使多模态能力可以快速、低成本地进入系统，而且非常适合小标注预算和持续增量场景。

## 2. Blocking -> ANN -> Threshold / Abstain -> HITL 的生产流水线

不要做全量 pairwise；不要幻想一个 matcher 单独解决所有问题。

## 3. 把人工验证量化

专家审核也有 FPR，也受输入候选 precision 影响。把 HITL 当成一个可测量分类器，而不是“最后让人看一下”。

但对当前腕表 Spec 必须做一个关键改造：

> **论文中的 embedding similarity 是主判定信号；我们这里必须降级为候选信号。真正的自动合并主键是经过来源审计、角色识别和品牌感知 canonicalization 的 `(brand, reference)`。**

因此我建议直接落地成：

```text
Reference-first Entity Matching
+
Multimodal Retrieval Fallback
+
Selective Auto-Accept
+
Expert HITL
+
Graph Safety Audit
```

这套设计既保留了论文在大规模、多模态、分布漂移和人机协同上的生产经验，又不会违反当前最重要的业务规则：

> **可以漏，但不能把不同 reference 合在一起。**

---

# 27. 可以直接交给工程实现的优先级

```text
P0
- brand normalization
- reference evidence schema
- brand-aware canonicalizer
- exact (brand, reference) index
- conflict -> abstain
- full audit log

P1
- product_type classifier
- OCR reference extraction
- HITL top-3 review UI
- gold hard-negative dataset

P2
- frozen multimodal embeddings
- lightweight linear projection
- brand-partitioned ANN retrieval
- calibrated safety score

P3
- continuous validator quality estimation
- graph transitive audit
- time/out-domain drift evaluation
```

如果资源有限，先完成 P0 + P1 就能形成一个符合 precision-first 原则的可用系统；P2 的多模态能力应该用于提高“reference 缺失时找到正确候选”的效率，而不是替代 reference。

---

## 参考

1. Tóth, S. et al. *End-to-end multi-modal product matching in fashion e-commerce*. arXiv:2403.11593, 2024.  
   https://arxiv.org/abs/2403.11593
2. HTML full text:  
   https://arxiv.org/html/2403.11593
3. 当前需求 Spec：  
   https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31
