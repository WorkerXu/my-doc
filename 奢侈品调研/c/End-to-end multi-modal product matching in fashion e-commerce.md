# End-to-end multi-modal product matching in fashion e-commerce

> 分析人：c  
> 论文：Sándor Tóth, Stephen Wilson, Alexia Tsoukara, Enric Moreu, Anton Masalovich, Lars Roemheld, **End-to-end multi-modal product matching in fashion e-commerce**, arXiv:2403.11593, 2024  
> 论文：https://arxiv.org/abs/2403.11593  
> PDF：https://arxiv.org/pdf/2403.11593  
> 调研清单：`奢侈品文章调研.md`  
> 目标需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务定义：**“同一个商品” = 同一 reference number / 型号**；数据规模 100 万–1000 万、持续增量；字段稀疏；图片可用；**precision 极端优先，绝不能误匹配，允许漏匹配**。

---

## 1. 选题与去重

执行前检查 `奢侈品调研/c/`，c 已经分析过以下 8 个文章/项目：

- `Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes.md`
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
- `DeepBlocker.md`
- `Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration.md`
- `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
- `TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency.md`
- `parts-distributor-sku-classifier.md`
- `pyJedAI.md`

本次选择调研清单中的 **End-to-end multi-modal product matching in fashion e-commerce**（arXiv:2403.11593），目录中尚无该文章，因此不重复。

选择它的原因不是“多模态”三个字本身，而是它给出了一个真实工业环境下相当完整的 **大规模商品匹配生产链路**：

```text
多源商品
  -> 预计算图像/文本 embedding
  -> 低成本 late fusion
  -> blocking
  -> KNN retrieval
  -> 阈值筛选
  -> Human-in-the-loop
```

论文处理 200 万级 offer、跨 domain 分布漂移和大量无匹配商品，并且明确讨论高 precision 区间和人工复核，因此和当前 100 万–1000 万规模、持续增量的三源腕表需求非常接近。

不过当前业务的关键定义比论文更严格：**视觉/文本上“像同款”不算同一个商品，必须是同一 reference**。因此这篇论文不能原样照搬。最值得落地的是它的“低成本多模态召回基础设施”，而不是让 embedding 相似度直接决定 MATCH。

本文最终建议是：

> **把 fashionID 风格的多模态模型降级为 Candidate Generator / Review Ranker；把 canonical reference 的强证据与冲突检测提升为唯一自动合并 Gate。**

这样既能利用图片解决字段稀疏和人工复核效率问题，又不会让“长得很像但 reference 不同”的腕表被错误合并。

---

## 2. 论文解决了什么问题

论文场景是多个 seller/domain 都在销售商品，需要识别不同来源的 offer 是否代表同一个真实商品。

每个 offer 可能包含：

- 多张图片；
- brand；
- title；
- price；
- size 数量等数值特征。

不同 domain 之间存在明显分布漂移：

- 标题写法不同；
- 图片角度不同；
- 模特/棚拍风格不同；
- 灯光和背景不同。

论文数据约 200 万 offer、5 个 domain；每个 offer 平均约 4.5 张图片。测试集还特意包含一个训练阶段完全未见的 domain，并且 index 中超过 99% 的商品在 query 侧没有匹配对象，这一点与真实二奢数据很接近：绝大多数候选本来就不应该被配对。

论文把问题拆成两个阶段：

```text
1. Encoding
   把 offer 编码到同一 metric space

2. Retrieval
   query 在 index 中找最近邻，足够相似时才产生匹配候选
```

这比全量 pairwise cross-encoder 更适合百万级以上数据，因为可以把昂贵的模型计算前置成一次性 embedding，然后用 ANN/KNN 解决查询。

---

## 3. fashionID 的技术实现

### 3.1 Late Fusion Encoder

论文提出的多模态 encoder 叫 **fashionID**。

结构非常简单：

```text
images[1..n]
   │
   ├─ image encoder -> embedding_1
   ├─ image encoder -> embedding_2
   └─ image encoder -> embedding_n
              │
          average pooling
              │
        image_embedding
              │
              ├──────────────────┐
text -> text encoder -> text_embedding          │
numeric features -------------------------------┤
                                                 │
                           concat(image, text, numeric)
                                                 │
                                         Linear Projection
                                                 │
                                          normalized vector
```

论文的关键工程选择是：

1. 图像 encoder 和文本 encoder 都使用大规模 pretrained model；
2. encoder 权重冻结；
3. 只训练最后一层线性 projection；
4. 多图先平均 pooling；
5. 图像、文本、数值特征最后拼接，做 late fusion。

其最佳实验使用 CLIP ViT-bigG-14 同时作为图像和文本 encoder，最终只训练一个约 0.6M 参数、输出 192 维的线性层。

这个结构最大的优点不是模型多新，而是**训练和迭代成本极低**：原始 CLIP embedding 可以预计算并重复利用；训练时 GPU 只需要处理小矩阵，不需要重复跑大模型前向和反向传播。

论文报告，在输入 embedding 已预计算的情况下，单张 A100 上约 10 分钟可以跑完 50 epoch。

### 3.2 Contrastive Learning

论文不是把两个商品拼起来做二分类，而是学习一个 metric space。

对于 batch 中的 anchor `i`，同商品 offer 是正样本，其余样本构成负样本，通过 supervised contrastive loss 让同商品向量靠近、不同商品向量远离。

形式上：

```text
sim(i, j) = vi · vj
```

在 normalized embedding 下，等价于 cosine similarity。

训练有两个重要技巧。

#### 技巧一：按 product id 采样

随机采 batch 时，正样本 pair 太少；论文按 product id 采样，把同一个 product 的多个 offer 放进同一 batch，从而提高 batch 内 positive pair 密度。

#### 技巧二：超大 batch 代替复杂 hard-negative mining

因为大 encoder 已冻结，论文可以预计算所有 embedding，并使用很大的 batch。实验发现 batch size 增大到 16K 时效果明显提升，之后过大又会让训练不稳定，因此最终固定 16K。

论文由此减少了专门离线 hard-negative mining 的工程复杂度。

### 3.3 线性层比 MLP 更稳健

论文还用 1-hidden-layer MLP 替换线性 projection 做实验：

- MLP 在 validation 上更好；
- 但线性 projection 在 in-domain 和 out-domain test 上略好。

作者认为简单线性映射更能保留大 pretrained encoder 原本的泛化能力。

这个结论对当前需求非常重要：三源数据会持续增量，未来还可能加入新平台/新品牌。此时追求 validation 榜单分数，不如保留预训练表征的跨域泛化。

### 3.4 Retrieval：Blocking + KNN + Threshold

论文的 retrieval 结构是：

```text
query
  -> Encoder
  -> Blocker
  -> KNN(index embeddings)
  -> distance threshold
  -> candidate match
```

直接在全量 index 做最近邻虽然比笛卡尔积好，但百万级仍然需要限制搜索空间。论文先根据 brand 名称做 fuzzy string similarity blocking，只在相似品牌分区中找 nearest neighbors。

距离采用 cosine：

```text
d(i,j) = 1 - vi · vj
```

然后用测试集上的 Precision-Recall 曲线选择 distance threshold。

这本质上是三层架构：

```text
高召回 blocker
    ↓
语义向量召回
    ↓
阈值 discriminator
```

### 3.5 为什么论文不用 F1 作为核心指标

论文明确认为 product matching 同时有两个重要工作区间：

- high precision / low recall：可以自动接受；
- lower precision / higher recall：交给人工复核。

因此论文重点看 PR curve/AUCPR，而不是只看一个 F1 点。

这与当前项目的方向完全一致，但当前项目应该再进一步：**自动区间的目标不是普通 high precision，而应近似零 false positive；模型不确定时宁可拒识。**

---

## 4. 论文结果中最值得迁移的发现

### 4.1 Frozen pretrained encoder + 小 projection 已经很强

论文完整多模态版本（CLIP image + CLIP text + numeric + linear projection）在测试中明显优于直接使用 raw pretrained embedding。

核心启示是：

> 不需要一开始就在三源腕表数据上 fine-tune 一个巨大视觉语言模型；先缓存通用 embedding，再训练一个很小的 domain projection，就能快速迭代。

这非常适合当前“可人工标几百对”的资源条件。

### 4.2 图片在时尚商品上比文本强，但腕表不能照搬结论

论文中 image-only 明显强于 text-only。原因是时尚服装标题通常缺乏唯一标识，而图片包含款式、版型、花纹等大量信息。

腕表也有相似点：

- 商品标题可能非常短；
- 卖家命名不一致；
- 图片通常更丰富；
- 同型号跨平台图像风格不同。

但腕表和服装存在一个致命差异：

> 同系列不同 reference 的腕表可以在视觉上几乎一样，只差尺寸、材质、机芯、盘面细节、年份或细小刻字。

所以图片适合做**召回和否决辅助**，不适合成为最终自动匹配依据。

### 4.3 模型 embedding 可预计算，特别适合增量

论文只训练小 projection 的设计天然适合持续增量：

```text
新商品到达
 -> 运行一次 frozen image/text encoder
 -> 保存 raw embeddings
 -> projection 得到 retrieval embedding
 -> upsert ANN index
```

历史图片不需要因每次小模型重训重新跑视觉 encoder；只要 projection 版本变化，就可以用已保存的 raw embedding 快速重算 192 维输出。

这是当前 100 万–1000 万规模最有价值的工程点之一。

### 4.4 HITL 不是“最后兜底”，而是一条独立生产路径

论文最终给人工审核者展示：

- 左边 query 商品；
- 右边 top-3 最近邻候选；
- 多张商品图片；
- 部分文本信息；
- 由人工选择匹配对象或 No Match。

论文发现：

- top-3 比只展示 top-1 更有利于 recall；
- 高 precision 场景下，多张图片、尤其 close-up 很有价值；
- 专职/训练过的审核员比无训练 crowd worker 更可靠；
- 对审核员进行错误回顾和反馈训练会明显改善效果。

这非常适合当前“允许漏匹配”的业务：模型负责把需要人工看的集合压缩到极小，人工只做 precision 提升，不承担全库搜索。

---

## 5. 为什么不能把论文阈值匹配原样用于本需求

这是本次分析最重要的部分。

论文中的“同商品”是 marketplace 语义上的 identical product；当前 Spec 则明确规定：

```text
MATCH <=> canonical reference number 相同
```

因此以下做法在当前系统中都不安全：

```text
cosine_similarity > 0.95 -> MATCH      # 不安全
image_similarity > threshold -> MATCH  # 不安全
LLM says same model -> MATCH            # 不安全
brand + series + visual close -> MATCH  # 不安全
```

模型只能证明“看起来非常像”或“语义上高度接近”，不能证明 reference 相等。

对腕表，false positive 的典型来源包括：

- 同系列不同尺寸：例如 36 / 41；
- 同壳型不同材质；
- 同盘面不同机芯或代际；
- reference 只有一个字符不同；
- 标题写的是兼容配件对应型号；
- 商品图里拍到盒证/保卡上别的 reference；
- 卖家复制模板导致标题包含多个 reference；
- 平台 SKU / 店铺 SKU 被误当品牌 reference。

所以最终架构必须是 **Reference-first, Multimodal-second**。

---

## 6. 建议直接落地的系统架构

建议将论文架构改造成下面这一版：

```text
┌──────────────────────────────────────────────────────────┐
│  雷小安 / 腕表之家 / 奢当家 增量商品流                    │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
                 [1] Source Normalizer
                           │
           ┌───────────────┼────────────────┐
           │               │                │
           ▼               ▼                ▼
     structured fields    title/desc       images
           │               │                │
           └───────────────┼────────────────┘
                           ▼
                [2] Reference Evidence Extractor
                   - structured ref
                   - title regex/parser
                   - OCR: dial/caseback/card/tag
                   - identifier role classifier
                           │
                           ▼
                [3] Canonical Reference Service
                   - brand canonicalization
                   - punctuation normalization
                   - brand-specific grammar
                   - reference dictionary / aliases
                   - provenance + confidence
                           │
                ┌──────────┴───────────┐
                │                      │
        explicit trusted ref      missing/ambiguous ref
                │                      │
                ▼                      ▼
       [4] HARD MATCH GATE       [5] Multimodal Retrieval
      exact canonical ref           frozen image/text encoder
      + brand compatible            + small projection
      + no contradiction            + brand/series blocking
                │                      │
                │                      ▼
                │                Top-K candidates
                │                      │
                │                      ▼
                │              [6] Reference Verifier
                │              extract/compare evidence
                │              contradiction => reject
                │                      │
                └──────────┬───────────┘
                           ▼
                   [7] Decision Router
                  AUTO_MATCH / REVIEW / REJECT
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
          entity/reference graph     HITL review queue
                │                     │
                └──────────┬──────────┘
                           ▼
                    audit + label feedback
```

这套结构中，多模态模型没有权限直接产生 `AUTO_MATCH`。

---

## 7. Stage 1：Reference Evidence Extractor

先不要做“两个商品是不是同一个”，先把每条商品记录变成一组可审计的 reference 证据。

建议数据结构：

```json
{
  "record_id": "sdj:12345",
  "brand": "Rolex",
  "reference_evidence": [
    {
      "value_raw": "126334-0001",
      "value_canonical": "1263340001",
      "source": "structured_field",
      "location": "model_no",
      "role": "PRODUCT_REFERENCE",
      "confidence": 0.999
    },
    {
      "value_raw": "126334",
      "value_canonical": "126334",
      "source": "image_ocr",
      "location": "warranty_card",
      "role": "PRODUCT_REFERENCE",
      "confidence": 0.97
    }
  ]
}
```

证据来源优先级可以先定为：

```text
P0 可信结构化 reference 字段
P1 标题/详情中符合品牌 grammar 且唯一的 reference
P1 OCR 在保卡/吊牌/表背区域识别出的 reference
P2 OCR/LLM 自由抽取但有品牌字典候选约束
P3 仅语义模型猜测的 reference
```

只允许 P0/P1 进入自动匹配 Gate；P2/P3 只参与召回和人工提示。

### 7.1 必须记录 provenance

不能只存：

```text
reference = 126334
```

必须存：

```text
reference = 126334
source = title
span = "劳力士日志型 m126334-0014"
parser_version = ref-parser-v7
confidence = ...
```

这样一旦发现某个 parser 规则有系统性误抽，可以快速回滚受影响的匹配边。

---

## 8. Stage 2：Canonical Reference Service

核心原则：**normalize 是为了消除格式差异，不是为了模糊匹配。**

可安全统一的例子：

```text
"126334-0014" -> "1263340014"
"126334 0014" -> "1263340014"
"Ref. 126334" -> "126334"
大小写统一
全角半角统一
Unicode normalization
```

但不能无条件做：

```text
去掉末位字符
编辑距离 <= 1
只比较数字前缀
把 126334 和 126300 当同款
```

因为这些操作恰好会把“相邻 reference”合并。

建议 canonicalization 做成 **brand-specific plugin**：

```python
canonicalize(brand, raw_reference) -> CanonicalReference
```

每个品牌可有：

- regex grammar；
- 长度规则；
- 合法前后缀；
- 是否允许 `/`、`.`、`-`；
- 父 reference / variant suffix 的业务语义；
- alias table。

如果业务最终确认“同一个商品”只看完整 reference，就必须把父型号和完整型号区分开。例如：

```text
126334       # family / base ref
126334-0014  # exact variant ref
```

若两个平台粒度不同，系统应返回 `REVIEW/UNKNOWN`，而不是擅自截断后 MATCH。

---

## 9. Stage 3：自动匹配 Hard Gate

建议把自动合并规则写成显式可审计策略，而不是一个模型阈值。

第一版可以非常保守：

```python
def auto_match(a, b):
    if not brand_compatible(a, b):
        return False

    ra = trusted_product_references(a)
    rb = trusted_product_references(b)

    # 任一侧没有可信 reference，禁止自动匹配
    if not ra or not rb:
        return False

    # 有明确 reference 冲突，直接否决
    if ra.isdisjoint(rb):
        return False

    # 多个 reference 且无法确定哪一个属于当前商品，拒识
    if is_ambiguous(a) or is_ambiguous(b):
        return False

    # 只有 canonical ref 明确交集，且无其它冲突，才允许自动匹配
    return True
```

最终决策状态不要只有二值：

```text
AUTO_MATCH
REVIEW
REJECT
UNKNOWN
```

`UNKNOWN` 是 precision-first 系统必须存在的一等公民。

---

## 10. Stage 4：借鉴 fashionID 的多模态召回层

对于 reference 缺失、埋在图片里、OCR 失败或出现多个候选的商品，再进入多模态 retrieval。

### 10.1 第一版 encoder

不建议直接上论文的 ViT-bigG-14，因为成本较高。先做可验证 MVP：

```text
Image: OpenCLIP ViT-L/14 或 SigLIP 类通用 encoder
Text : 同一多模态模型的 text tower，或中文/多语 embedding model
OCR  : 单独保留文本特征
Meta : brand / series / year / diameter / material（有则用）
```

输出：

```text
raw_image_embedding
raw_text_embedding
raw_ocr_embedding
numeric/meta vector
```

第一阶段甚至可以不训练 projection，先测 raw embedding 的 Recall@K。

如果 top-K recall 不够，再训练论文式小 projection：

```text
concat(image_pool, text, OCR, meta)
      -> Linear(?, 192/256)
      -> L2 normalize
```

### 10.2 腕表不要简单 average pooling 所有图片

论文的多图平均 pooling 对服装有效，但腕表场景可能把最关键的局部证据稀释掉。

建议先把图片分角色：

```text
HERO / DIAL / CASEBACK / CLASP / WARRANTY_CARD / TAG / BOX / OTHER
```

分别保留：

```text
hero_embedding
closeup_embedding
caseback_embedding
card_embedding
```

候选召回可以做加权：

```text
score =
  0.45 * visual_global
+ 0.20 * dial_similarity
+ 0.10 * caseback_similarity
+ 0.15 * text_similarity
+ 0.10 * metadata_similarity
```

权重只是示意，不能作为自动 MATCH 条件。

对 reference 证据而言，`warranty_card / tag / caseback OCR` 的价值远高于商品主图的视觉相似。

### 10.3 Blocking

论文用 fuzzy brand blocking；当前项目应做得更严格：

```text
L0: canonical brand
L1: product type = watch / strap / accessory / box / card
L2: series/family（若可信）
L3: reference prefix / grammar bucket（若有）
L4: ANN retrieval
```

重要的是：

> blocking 只负责减少搜索空间，不负责产生 MATCH。

如果 reference 已经可信，甚至无需 ANN，直接 inverted index：

```text
(brand, canonical_reference) -> record_ids
```

这是百万级数据中最便宜、最准确的主路径。

---

## 11. 多模态模型真正应该解决的三个任务

### 任务 A：没有 reference 时，找最可能的候选簇

例如某条奢当家记录只有：

```text
劳力士 日志型 蓝盘 41mm
+ 8 张图
```

模型可以从同品牌商品中返回 top-20 可能 reference，对后续 OCR/人工确认做候选压缩。

### 任务 B：发现 reference 抽取错误

如果标题 parser 抽到 `126334`，但视觉上 top-K 全部落在另一系列，不能直接改 reference，但可以产生：

```text
REFERENCE_SUSPECT
```

交给二次抽取或人工复核。

### 任务 C：人工审核排序

人工不应该在几百万商品里搜索，而是看：

```text
query
candidate #1
candidate #2
candidate #3
```

并且界面直接突出 reference 证据来源。

---

## 12. 建议的人审 UI：从论文 top-3 方案进一步强化 reference 证据

论文 UI 的思想值得直接复用，但腕表 UI 应改成：

```text
┌──────── Query ────────┐    ┌──── Candidate 1 ────┐
│ source                │    │ source               │
│ brand                 │    │ brand                │
│ title                 │    │ title                │
│ REF evidence          │    │ REF evidence         │
│ [title span]          │    │ [structured field]   │
│ [caseback OCR crop]   │    │ [card OCR crop]      │
│ images: dial/back/... │    │ images: dial/back/...│
└───────────────────────┘    └──────────────────────┘

Candidate 2 / Candidate 3 ...

[Same reference] [Different reference] [Unable to determine]
```

必须保留 `Unable to determine`。

不建议像论文一样仅做“这个商品是不是 match”的视觉判断，因为当前业务的 ground truth 是 reference。

审核员操作规范应该是：

1. 先看双方可信 reference；
2. 再看 reference 所在原始 span / OCR crop；
3. 只有 reference 证据一致才选 Match；
4. 图片相似只作为定位证据；
5. 任一字符无法确认 -> Unable to determine。

高风险候选可以采用双人独立复核，而不是简单 2/3 多数投票。

---

## 13. 只有几百人工标签，怎么训练

论文有大量高质量 product id 标签；当前只有几百对人工黄金标签，不能照搬其 supervised contrastive 数据规模。

但本项目有一个论文没有的巨大优势：**reference 本身就能生成训练监督。**

### 13.1 Silver Positive

从可信结构化 reference 中自动构造：

```text
source A: Rolex + 126334
source B: Rolex + 126334
=> positive silver pair
```

只取：

- brand 一致；
- reference 来自高可信字段；
- 唯一且无冲突；
- 排除配件/盒证等非腕表商品。

### 13.2 Hard Negative

最有价值的负样本不是随机其他品牌，而是：

```text
same brand
same series
very similar title/image
DIFFERENT canonical reference
```

例如：

```text
126334 vs 126300
126610LN vs 126610LV
```

这种 hard negative 才真正训练模型认识“视觉很像但 reference 不同”的危险边界。

### 13.3 几百黄金标签应该花在哪里

不要随机标。

优先标：

```text
1. parser 有多个 reference 的记录
2. OCR 与标题冲突的记录
3. embedding Top-1 很近但 reference 不同的 pair
4. 新品牌/新来源
5. 低频 reference
6. 同系列相邻 reference
7. 配件/盒证/表带等容易误识别的商品
```

这几百对主要用于：

- 验证抽取器 precision；
- 校准 REVIEW/AUTO_MATCH gate；
- 评估 hard-negative false positive；
- 训练轻量 ranker，而不是端到端 fine-tune 大模型。

---

## 14. 可直接实现的训练流程

### Phase 0：无需训练的 baseline

```text
brand canonicalization
+ reference extraction
+ exact canonical reference inverted index
+ strict hard gate
```

先算出自动匹配覆盖率和人工复核量。

### Phase 1：Frozen embedding retrieval

```text
precompute image/text embeddings
build brand-partitioned ANN index
measure Recall@10 / Recall@20
```

模型只做召回，不参与自动合并。

### Phase 2：小 projection

使用 silver positive + hard negative：

```python
z = normalize(W @ concat(image_emb, text_emb, ocr_emb, meta))
```

用 supervised contrastive loss 训练 `W`。

推荐先从 192 或 256 维做实验；论文发现 192 维附近表现较好，但腕表必须重新验证，不能机械复用。

### Phase 3：Review ranker / calibrator

对于 top-K candidate，用轻量模型输入：

```text
image cosine
text cosine
OCR cosine
brand exact
series exact
reference evidence overlap
reference conflict
diameter conflict
material conflict
year conflict
source pair
```

输出：

```text
review_priority
```

注意不是自动 MATCH probability。

---

## 15. 100 万–1000 万规模的工程实现

### 15.1 建议的存储拆分

#### Raw Record Store

保存三源原始数据，建议对象存储 + OLAP/关系库元数据。

#### Canonical Record Table

```sql
record_id
source
source_item_id
brand_id
title
product_type
canonical_reference
reference_confidence
reference_provenance
parser_version
updated_at
```

#### Reference Inverted Index

最关键的在线索引：

```text
(brand_id, canonical_reference) -> [record_id...]
```

可直接放 PostgreSQL/MySQL 唯一/组合索引，或 RocksDB/Redis 等 KV 层。

#### Embedding Store

保存：

```text
image_raw_embedding
text_raw_embedding
ocr_embedding
projection_embedding
model_version
```

raw embedding 和 projection embedding 分开保存，便于更换小 projection 而无需重新跑大 encoder。

#### ANN Index

可选：

- FAISS IVF/HNSW；
- Milvus；
- Qdrant；
- pgvector（规模和 QPS 不高时）。

第一版如果主要做离线批匹配，FAISS 最简单。

### 15.2 增量 pipeline

```text
Kafka / Queue
   -> Normalize Worker
   -> Reference Extract Worker
   -> OCR Worker（按需）
   -> Embedding Worker
   -> Match Gate
   -> ANN Upsert
   -> Review Queue
```

关键点是每一步都版本化：

```text
normalizer_version
ref_parser_version
ocr_version
image_encoder_version
projection_version
decision_policy_version
```

否则线上出了 false positive 无法追踪是哪一层造成的。

### 15.3 不必每次三源全量重跑

增量记录到达时：

```text
1. 先查 exact reference inverted index
2. 如果 trusted ref 命中，执行 hard gate
3. 否则只在对应 brand partition 做 ANN
4. 只对 top-K 跑昂贵 verifier/OCR/人工流程
```

从复杂度上，把：

```text
O(N × M)
```

压缩为：

```text
exact hash lookup + O(log N) / ANN top-K
```

---

## 16. Precision-first 决策矩阵

第一版建议：

| 情况 | 决策 |
|---|---|
| 双方可信完整 canonical reference 完全一致，品牌一致，无冲突 | `AUTO_MATCH` |
| 双方可信 reference 明确不同 | `REJECT` |
| 一侧有可信 ref，一侧无 ref，但多模态高度相似 | `REVIEW` |
| 双方都无可信 ref，多模态高度相似 | `REVIEW`，绝不自动合并 |
| 标题 ref 与图片/OCR ref 冲突 | `REVIEW` 或 `REJECT` |
| 同系列、图片几乎一致，但 reference 差 1 字符 | `REJECT` |
| 只能得到 base reference，另一侧是完整 variant reference | `UNKNOWN/REVIEW` |
| 商品被识别为表带/盒证/配件 | 不进入腕表主实体自动匹配 |

自动规则宁可保守。

如果后续真的要允许“模型自动接受无 reference 候选”，必须把它当另一个受风险控制的产品能力单独评审，而不是在当前 Spec 中隐式开启。

---

## 17. 评测体系：不要用总体 F1 掩盖误匹配

建议至少分四组指标。

### 17.1 Reference Extraction

```text
exact reference precision
exact reference recall
ambiguous rate
conflict rate
```

其中 precision 远重要于 recall。

### 17.2 Candidate Retrieval

```text
Recall@1
Recall@3
Recall@10
Recall@20
```

这里可以追求高 recall，因为 retrieval 还不会自动合并。

### 17.3 Auto-Match Gate

核心指标：

```text
Auto-Match Precision
False Matches / 100k accepted pairs
Coverage = Auto-Match / all records
```

不建议只报 F1。

### 17.4 Review Queue

```text
review precision
review yield
avg seconds / decision
unable-to-determine rate
false-positive rate by source pair / brand
```

并且必须按：

```text
雷小安 x 腕表之家
雷小安 x 奢当家
腕表之家 x 奢当家
brand
reference family
new/old data window
```

分别统计，防止总体指标掩盖某一个来源的系统性问题。

---

## 18. 对“绝对不能误匹配”的统计现实

产品要求可以是“绝不能误匹配”，但离线测试永远不能数学证明线上永远零错误。

工程上应把它翻译成三条：

1. **决策逻辑设计为只有强 reference 证据才自动放行；**
2. **所有低置信度样本默认拒识；**
3. **持续测量 false-positive 上界，而不是一次性报一个 100% precision。**

例如测试集上“几百个自动匹配样本 0 错”只能说明当前样本没看到错例，不能证明百万级线上错误率足够低。

因此几百人工黄金标签应该更多用于发现边界和规则错误；当系统准备大规模自动合并时，需要持续扩大高风险抽样审计集，尤其抽：

```text
新品牌
新 source schema
parser 新版本
OCR 新版本
reference 极相似 family
模型高相似但硬规则拒绝的 pair
```

---

## 19. 与 c 已有调研结果的组合方式

这篇论文不是孤立落地，和 c 前面的调研可以组合成一条完整链路。

### 与 parts-distributor-sku-classifier 的组合

先判断一个编号究竟是：

```text
BRAND_REFERENCE
PLATFORM_SKU
SELLER_SKU
ACCESSORY_COMPATIBILITY_CODE
UNKNOWN
```

再允许进入 reference gate。

### 与 Multi-Value-Product RAG 的组合

当标题中的 reference 不完整或是长尾格式时：

```text
brand/series -> retrieve reference dictionary candidates
             -> constrained extraction
```

禁止自由生成一个不存在的型号。

### 与 DeepBlocker 的组合

如果后续发现简单 brand partition + ANN 不够，可以用 DeepBlocker 思路增强 candidate generation；但仍然保持 blocker/retrieval 与 final decision 解耦。

### 与 Confidence Classifier 的组合

如果以后确实需要让某些模型路径自动放行，应在独立 calibration 层做 selective prediction / abstention，而不是直接使用 cosine 阈值。

### 与 TransClean 的组合

自动/人工 MATCH 最终会形成跨源实体图。对图做 transitive consistency 审计，可以发现一条错误边是否把两个 reference cluster 粘在一起。

因此可以形成：

```text
Reference Extract
 -> Exact Gate
 -> Multimodal Retrieval
 -> Human Review
 -> Entity Graph
 -> Transitive Audit
```

---

## 20. 建议的最小可落地版本（MVP）

### Sprint 1：reference-first baseline

目标：不训练模型也能上线一个高 precision 主路径。

实现：

1. 三源字段 mapping；
2. brand canonicalization；
3. reference parser；
4. identifier role classifier 的第一版规则；
5. canonical reference service；
6. `(brand, reference)` inverted index；
7. AUTO_MATCH / REVIEW / REJECT 状态机；
8. 生成首批 300–500 个黄金审计 pair。

输出：

```text
auto-match coverage
precision audit
missing-reference rate
ambiguous-reference rate
```

### Sprint 2：多模态召回

1. 预计算图片 embedding；
2. 预计算 title embedding；
3. brand-partitioned FAISS；
4. 对缺 ref 记录跑 top-20 retrieval；
5. 建立 Recall@K 数据集；
6. 做 top-3 人审页面。

此阶段仍不让 embedding 产生 AUTO_MATCH。

### Sprint 3：OCR + 小 projection

1. 图片角色分类；
2. warranty card / tag / caseback OCR；
3. silver positive / hard negative 生成；
4. 训练 192/256 维 linear projection；
5. A/B 对比 raw embedding 与 projection 的 Recall@K；
6. 上线 REVIEW 排序模型。

### Sprint 4：安全审计

1. reference cluster graph；
2. 冲突边检测；
3. transitive consistency audit；
4. 按品牌/来源做 false positive dashboard；
5. parser/model version 回溯能力。

---

## 21. 推荐的服务接口

### Reference Extract API

```http
POST /reference/extract
```

返回：

```json
{
  "brand_id": "rolex",
  "evidence": [...],
  "canonical_candidates": [
    {
      "reference": "1263340014",
      "confidence": 0.997,
      "role": "PRODUCT_REFERENCE"
    }
  ],
  "status": "UNAMBIGUOUS"
}
```

### Match API

```http
POST /match/decide
```

```json
{
  "left_id": "lxa:1",
  "right_id": "sdj:9",
  "decision": "AUTO_MATCH",
  "reason_codes": [
    "BRAND_EXACT",
    "TRUSTED_REFERENCE_EXACT",
    "NO_REFERENCE_CONFLICT"
  ],
  "policy_version": "match-policy-v1"
}
```

### Candidate Retrieval API

```http
GET /candidates/{record_id}?k=20
```

返回 top-K 和解释字段：

```text
visual_sim
text_sim
ocr_sim
brand
series
reference evidence
conflicts
```

这样 matcher 的“为什么”可以被业务和人审系统直接消费。

---

## 22. 本文对原论文的“采用 / 修改 / 不采用”清单

### 直接采用

- 大 encoder embedding 预计算；
- frozen pretrained encoder；
- late fusion + 小 linear projection；
- ANN/KNN 两阶段 retrieval；
- blocking 先缩小候选空间；
- PR curve 而非只看 F1；
- 高精度自动区间 + 低精度 HITL；
- top-3 人工候选；
- 多图用于人审；
- out-domain/time-split 评估分布漂移。

### 修改后采用

- `brand fuzzy blocking` -> canonical brand + product type + series/ref grammar blocking；
- `average image pooling` -> 腕表图片角色感知 pooling；
- `distance threshold => match` -> distance threshold 只用于 candidate/review routing；
- `human judges same product` -> human judges same canonical reference；
- 普通 negatives -> 重点训练同系列不同 reference hard negatives。

### 明确不采用

- 视觉/多模态 similarity 直接自动判 MATCH；
- 为提高 recall 放宽 canonical reference 比较；
- 编辑距离近就认为 reference 相同；
- 用 embedding cluster 替代 reference entity key；
- 没有 reference 证据时仅凭模型高分自动合并。

---

## 23. 最终建议

如果只从这篇论文拿一个可直接落地的技术决策，我建议是：

> **把昂贵多模态大模型做成可缓存的 frozen feature factory，把业务学习压到一个很小的 projection/ranker；同时彻底把“相似度召回”和“是否同 reference 的安全决策”解耦。**

当前系统的主路径应该非常朴素：

```text
可信 reference 抽取
 -> canonicalization
 -> exact inverted index
 -> contradiction check
 -> AUTO_MATCH
```

多模态路径则解决主路径覆盖不了的长尾：

```text
缺失/歧义 reference
 -> frozen multimodal embeddings
 -> ANN top-K
 -> OCR/reference verifier
 -> 人工审核
```

这比直接训练一个“同款/不同款”多模态 classifier 更符合 Spec，因为它把最容易产生 false positive 的模型限制在**候选生成层**，最终自动决策仍由可解释、可审计、可回滚的 reference 硬证据控制。

对于 100 万–1000 万且持续增量的数据，论文的预计算 embedding + 小 projection 设计也非常适合工程化：大模型推理只做一次，历史 raw embedding 可复用；每次模型迭代只重算小 projection 和 ANN index，不需要重跑所有图片。

因此推荐的总体架构不是“fashionID 替代 reference matching”，而是：

```text
Reference-first Safety Core
        +
FashionID-style Multimodal Retrieval Layer
        +
Reference-aware HITL
        +
Graph Consistency Audit
```

这四层组合后，既能满足百万级扩展和字段稀疏，又能把“绝不能误匹配”放在架构层而不是寄希望于某一个模型分数。

---

## 24. 参考资料

1. Sándor Tóth et al., **End-to-end multi-modal product matching in fashion e-commerce**, arXiv:2403.11593, 2024.  
   https://arxiv.org/abs/2403.11593
2. 本仓库：`奢侈品文章调研.md`。
3. Notion Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）。
