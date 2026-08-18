# Intermediate Fusion for Multimodal Product Matching：中间融合多模态商品匹配及其对跨源腕表 reference 解析的落地方案

> 分析人：c  
> 论文：Jacob Pollack, Hanna Köpcke, Erhard Rahm, **Intermediate Fusion for Multimodal Product Matching**, GvDB 2024  
> 论文：https://ceur-ws.org/Vol-3710/paper14.pdf  
> 官方代码：https://git.informatik.uni-leipzig.de/jp31zusu/intermediate-fusion-for-multimodal-product-matching  
> 调研清单：`奢侈品文章调研.md`  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务硬约束：**“同一个商品” = 同一 reference number / 型号；precision 极端优先，允许漏匹配；100 万–1000 万级并持续增量。**

---

## 0. 去重检查与选题结论

执行前先检查了 `奢侈品调研/c/` 中已有结果，并对本次目标标题做了文件存在性检查：`Intermediate Fusion for Multimodal Product Matching.md` 不存在，因此不是 c 已经分析过的文章。

同时已经确认 c 此前做过 `parts-distributor-sku-classifier`、`End-to-end multi-modal product matching in fashion e-commerce` 等同主题项目/文章，本次不重复这些内容。

本次选择 `Intermediate Fusion for Multimodal Product Matching`，原因是调研清单对它的推荐点和当前需求高度同构：

1. 它直接研究跨商店商品的**文本 + 图片**实体匹配；
2. 数据集的 ground truth 本身使用了 **MPN（manufacturer part number）+ 人工检查**，与当前“reference 才是实体定义”的业务口径非常接近；
3. 它专门构造了“视觉/文本很像，但实际不是同商品”的 hard negative；
4. 它的核心技术不是简单 late fusion，而是先分别训练文本分支和图像 Siamese 分支，再把中间层表征融合；
5. 论文在更真实的 Zalando 数据上 precision 只有 0.7144，这反过来非常清楚地证明：**多模态模型可以提高判别能力，但绝不能直接承担当前需求的最终自动合并权限。**

本文最终结论是：

> **保留论文的 hard-negative 构造、分支式中间训练、图文互补融合思想；但把模型从“最终 MATCH 分类器”降级成“reference 证据验证器 / 未解析商品候选排序器 / 人工审核辅助器”。真正自动合并仍必须由 `verified brand + canonical reference exact equality + no conflict` 的确定性 Gate 决定。**

---

# 1. 论文到底解决什么问题

论文面对的是典型 Web Product Matching：不同电商站点中的两个 offer 是否代表同一个真实商品。

传统文本实体匹配的问题是：

- 标题字段不完整；
- 不同站点写法不同；
- 商品名称可能高度相似；
- 文本里缺少能区分变体的属性。

而图片能够提供颜色、形状、纹理、款式等额外信息，所以作者研究如何把文本和图片融合起来。

论文把多模态融合分为四类：

```text
Early Fusion
  原始/低层模态很早就混合

Intermediate Fusion
  各模态先独立提取高层特征
  -> 合并特征
  -> 合并后继续经过可学习网络
  -> 最终分类

Late Fusion
  各模态先独立决策/打分
  -> 合并后直接阈值/加权得到结论

Hybrid Fusion
  在多个阶段重复进行模态融合
```

作者认为 early fusion 容易引入高维冗余，late fusion 又可能错过复杂的跨模态关系，因此选择 **Intermediate Fusion**。

对当前腕表需求，这个划分非常重要：

- `reference exact match` 是**硬证据层**；
- 文本、图片、OCR、属性冲突是**辅助证据层**；
- 如果把所有东西简单加权后直接出 MATCH，就相当于让弱证据覆盖硬规则，风险很高；
- 更合理的是让各类证据先独立形成高层表示，再做“是否支持/反对某个 reference 候选”的融合，但最终实体合并仍由硬规则收口。

---

# 2. 数据集与 hard negative：这篇论文最值得迁移的部分之一

## 2.1 WDC Shoes

论文沿用了 WDC 商品匹配数据的鞋类子集，并使用带图片的多模态版本。

数据规模：

| 数据集 | Products | Matches | Non-matches |
|---|---:|---:|---:|
| WDC Shoes train | 950 | 1350 | 6286 |
| WDC Shoes test | 813 | 206 | 586 |

论文指出，WDC Shoes 的 train/test 存在部分商品重叠，因此结果可能比真实线上分布乐观。

## 2.2 Zalando / Tommy Hilfiger / Gerry Weber

作者另外构建了更现实的数据集：

- Zalando；
- Tommy Hilfiger 官方站；
- Gerry Weber 官方站。

ground truth 采用：

```text
MPN matching
  +
manual inspection
```

数据规模：

| 数据集 | Products | Matches | Non-matches |
|---|---:|---:|---:|
| Zalando train | 6676 | 945 | 3843 |
| Zalando test | 1431 | 178 | 785 |

而且 train/test 中的 Zalando 商品互斥，避免同一商品泄漏到训练和测试两侧。

这比随机 pair split 更接近当前持续增量场景。

## 2.3 Hard negative 的构造方式

论文没有随机抽一堆很容易区分的负样本，而是：

```text
商品文本 -> 预训练 RoBERTa embedding
商品图片 -> 预训练 Swin-Transformer embedding
              │
              ▼
          多模态 embedding
              │
              ▼
       Approximate Nearest Neighbor
              │
              ▼
对每个商品找“最像但 ground truth 不匹配”的商品
```

因此负样本专门包含：

- 图片非常相似；
- 标题非常相似；
- 只有小变体不同；
- 人眼第一眼容易误判，但实际不是同商品。

**这正是腕表 reference 匹配最应该采用的负样本构造方式。**

当前系统不能主要训练：

```text
Rolex Submariner
vs
Hermès handbag
```

这种 trivial negative。

真正有价值的 hard negative 应该是：

```text
同品牌
+ 同系列
+ 标题很像
+ 图片很像
+ 但 reference 不同
```

例如：

```text
brand = same
series = same
reference_A != reference_B
image_similarity 很高
```

还要额外加入当前二奢特有的负样本：

- 表带/表盒/保卡标题里出现目标手表 reference；
- 平台 SKU 被误抽成 reference；
- 序列号被误抽成 reference；
- OCR 从保卡或吊牌里读到另一个编号；
- 标题复制粘贴了其他型号；
- 同系列不同尺寸、材质、盘面、机芯的 reference 近邻。

在 precision-first 任务里，**hard negative 的质量通常比“再换一个更大的模型”更重要。**

---

# 3. 原论文 Intermediate Fusion 架构逐层拆解

论文 Figure 3 给出了完整结构。它不是“文本分数 + 图片分数加权”，而是三阶段结构：

```text
Text Branch intermediate training
             ┐
             │
Image Siamese Branch intermediate training
             │
             ▼
取两个分支的高层中间表示
             │
             ▼
       Concatenate 1x768
             │
             ▼
        Fusion MLP
             │
             ▼
         binary match
```

## 3.1 Text Branch

输入两个商品标题/文本，论文使用 pair 格式：

```text
[CLS] text_1 [SEP] text_2 [SEP]
```

随后：

```text
RoBERTa
  │
  ▼
Bidirectional layer
（Figure 3 标注 64x128）
  │
  ▼
Hybrid Pooling
max pooling + mean pooling
  │
  ▼
1 x 256 vector
  │
  ▼
Dense 512
  │
  ▼
Dense 256
  │
  ├─ standalone classifier head（中间训练时使用）
  │
  └─ 256-d high-level text representation（融合阶段使用）
```

论文文字说明使用 BiLSTM 对 RoBERTa 表示继续做双向建模，然后通过 max pooling 和 mean pooling 的 hybrid pooling 获取固定长度向量。

Hybrid pooling 的意义是：

- max pooling 强调最显著的局部匹配信号；
- mean pooling 保留整体语义平均状态；
- 二者拼接后比只拿一个 `[CLS]` 或只做平均更丰富。

对腕表标题，这个思路有实际价值，因为标题里往往同时存在：

```text
品牌 / 系列 / reference / 尺寸 / 材质 / 年份 / 成色 / 营销词
```

其中 reference、尺寸之类是局部高价值 token，而品牌系列属于整体上下文。

但生产迁移时不建议直接把整段原始标题喂进去就结束，后文会改成 **reference-aware evidence text**。

## 3.2 Image Branch：Swin-Transformer Siamese Network

图像分支是两个共享参数的子网络：

```text
image_1                         image_2
  │                               │
Swin-Transformer               Swin-Transformer
  │        shared weights          │
  ▼                               ▼
Pooling 1x2048                 Pooling 1x2048
  │                               │
Dense 512                      Dense 512
  │                               │
Dense 256                      Dense 256
  │                               │
Dense 256                      Dense 256
  │                               │
  └──────── Euclidean distance ────┘
                  │
                  ▼
        standalone image classifier
```

Siamese 结构的价值在于：

- 两边用完全相同的特征提取器；
- 目标不是记住某个平台图片风格；
- 而是学习“两个商品图片之间哪些差异对匹配最关键”。

论文没有把最终 Euclidean distance 直接与文本 embedding 拼起来；它指出过去方案仅拼一个距离标量会损失大量信息。

作者真正保留下来的是 Siamese 两边各自已经经过匹配任务训练的 **高层 256-d image embedding**。

这个设计非常值得借鉴：

> 不要只把 `image_similarity=0.93` 这样一个标量交给最终模型。相似度标量压缩掉了“为什么相似/为什么不同”的结构信息；保留高层表示再融合，最终网络有机会学习更细粒度的交互关系。

## 3.3 Intermediate Fusion

Text Branch 输出 256 维，两个 Image Branch 各输出 256 维：

```text
text_pair_embedding: 256
image_1_embedding:    256
image_2_embedding:    256
                      ---
concat:                768
```

然后：

```text
Concatenation 768
  │
Dense 512
  │
Dense 512
  │
Dense 256
  │
Dense 1
  │
match / non-match
```

关键点是：**各分支先单独经历匹配任务的 intermediate training，再抽其中间层表示做融合。**

这和简单 late fusion 的区别是：

```text
late fusion:
text_score + image_score -> weighted average -> decision

paper intermediate fusion:
text_high_level_feature
image1_high_level_feature
image2_high_level_feature
        -> joint MLP
        -> decision
```

联合 MLP 可以学习：

- 文本强、图片弱时如何处理；
- 图片强、文本弱时如何处理；
- 文本与图片互相冲突时如何处理；
- 某一模态缺信息时另一模态如何补偿。

## 3.4 论文没有说清楚的部分，不应该擅自补全

论文正文和 Figure 3 能确认网络主结构，但没有完整披露所有生产实现细节，例如：

- 最终 fusion 阶段是否冻结所有 branch 参数；
- 每层具体 activation / dropout；
- optimizer、learning rate、batch size；
- 所有训练阶段的 early stopping 策略。

因此落地时不应该“照猜一个参数表”。更稳妥的训练策略是：

```text
Stage A：训练/微调 text head
Stage B：训练 image Siamese head
Stage C：冻结大 backbone，只训练 fusion MLP
Stage D：如果验证集稳定，再小学习率解冻各分支顶部层
```

这样既保留预训练模型泛化，又避免几百条黄金标签把大模型过拟合。

---

# 4. 一个容易被忽略的架构问题：pair 顺序对称性

实体匹配理论上应满足：

```text
MATCH(A, B) == MATCH(B, A)
```

但论文 Figure 3 的融合方式是：

```text
[text_pair_256, image_A_256, image_B_256]
```

直接 concat 两边图像 embedding，会让 MLP 天然具有位置敏感性。

Text Branch 的 pair 输入：

```text
[CLS] text_A [SEP] text_B [SEP]
```

同样也可能对 A/B 顺序敏感。

论文没有把“严格 permutation invariance”作为重点，但生产实体解析最好补上。

建议至少使用以下一种：

### 方案 1：训练时双向增强

```text
(A, B, label)
(B, A, label)
```

推理时：

```text
score = min(score(A,B), score(B,A))
```

在 precision-first 场景用 `min` 比平均更保守。

### 方案 2：对图像 embedding 使用对称交互特征

```text
abs(i1 - i2)
i1 * i2
cosine(i1, i2)
```

而不是直接：

```text
concat(i1, i2)
```

### 方案 3：把 pair model 只用于候选排序

即便存在轻微顺序敏感，也不让它直接造成实体合并，最终自动合并仍由 reference exact rule 决定。

当前需求最推荐方案 2 + 3。

---

# 5. 论文实验结果说明了什么

## 5.1 WDC Shoes

| Model | F1 | Precision | Recall |
|---|---:|---:|---:|
| Text Branch (RoBERTa) | 0.8477 | 0.8102 | 0.8900 |
| Image Branch (Swin) | 0.7593 | 0.7058 | 0.8223 |
| Intermediate Fusion | **0.8829** | **0.8704** | 0.8968 |
| DeepMatcher title+image | 0.8560 | 0.7920 | **0.9300** |

Intermediate Fusion 的 precision 从 text-only 的 0.8102 提升到 0.8704，说明图片确实提供了有效补充信息。

## 5.2 Zalando

| Model | F1 | Precision | Recall |
|---|---:|---:|---:|
| Text Branch (RoBERTa) | 0.6230 | 0.4982 | **0.8464** |
| Image Branch (Swin) | 0.6178 | 0.6206 | 0.6161 |
| Intermediate Fusion | **0.7401** | **0.7144** | 0.7697 |
| Reimpl. ImageBERT | 0.6048 | 0.4624 | 0.8858 |

Zalando 更难，原因包括：

- train/test 更干净地分离；
- 不同站点的文字写法差异更大；
- 同商品图片可能差异很大；
- 非同商品的小变体图片又可能几乎一样。

**这里对当前需求最重要的不是 F1=0.7401，而是 precision=0.7144。**

对于普通商品匹配研究，这可能已经是明显提升；但对“绝对不能误匹配”的三源腕表实体系统，这个水平完全不能直接自动上线做 merge。

因此论文应该被解读为：

> Intermediate Fusion 是很好的“困难候选判别/排序能力”，但不是“reference 身份证明”。

---

# 6. 为什么原论文不能原样用于当前腕表 Spec

当前 Spec 有一个比普通 product matching 更强的定义：

```text
same entity <=> same reference number
```

这会改变整个系统架构。

## 6.1 外观近似不等于 reference 相同

腕表里最危险的 false positive 恰好是：

```text
同品牌
同系列
同设计语言
图片极像
但 reference 不同
```

差异可能只是：

- 36mm vs 41mm；
- 钢 vs 间金；
- 不同盘面；
- 不同圈口；
- 不同机芯代际；
- 同壳型不同配置；
- 某个后缀代表不同材质/版本。

模型越擅长“找长得像的”，如果没有 reference 硬约束，越可能在最危险的边界上产生误合并。

## 6.2 图片里可能根本不是商品主体

二奢商品图片常包含：

- 表本体；
- 表背；
- 保卡；
- 包装盒；
- 说明书；
- 吊牌；
- 配件；
- 鉴定报告。

这些图片的价值不同。

原论文每个商品用单一产品图片，当前系统则应该先做 **image role classification**：

```text
WATCH_FRONT
WATCH_BACK
CARD
BOX
ACCESSORY
DOCUMENT
OTHER
```

然后不同角色进入不同证据通路：

- 表盘/表壳图片 -> 视觉相似性与类别冲突；
- 保卡/吊牌/表背 -> OCR reference 证据；
- 盒子/说明书 -> 不应直接拿来做主体视觉匹配。

## 6.3 Pairwise classifier 不适合 100 万–1000 万全量匹配

如果三个来源总量 1000 万，直接把所有记录做两两模型判定是不可接受的。

论文关注 pair 分类精度，但当前系统首先应该解决**搜索空间缩减**：

```text
绝大多数已解析记录：
O(N) reference resolution
+ hash/index join

只有 unresolved 小部分：
brand/category blocking
+ ANN top-K
+ multimodal verifier
```

## 6.4 原论文是强制二分类，没有 ABSTAIN

当前系统必须允许：

```text
MATCH
NO_MATCH
ABSTAIN / NEEDS_REVIEW
```

而且最好不是一个三分类 softmax，而是业务状态机：

```text
VERIFIED_REFERENCE
CONFLICT
UNKNOWN
NEEDS_REVIEW
```

因为“模型不确定”不是错误，而是 precision-first 系统的正常输出。

---

# 7. 推荐把问题从“pair matching”重写为“reference resolution”

## 7.1 核心原则

不要把主问题定义成：

```text
两个商品是否相似？
```

而应定义成：

```text
每条商品记录的 canonical brand 是什么？
每条商品记录自身的 canonical reference 是什么？
这个 reference 的证据是否足够可靠？
```

得到稳定结果后：

```text
entity_key = hash(brand_id + "\x1f" + canonical_reference)
```

自动匹配甚至不需要神经网络：

```python
def auto_match(a, b):
    if a.brand_id != b.brand_id:
        return "NO_MATCH"

    if a.reference_state == "VERIFIED" and b.reference_state == "VERIFIED":
        if a.canonical_reference != b.canonical_reference:
            return "NO_MATCH"

        if a.has_hard_conflict or b.has_hard_conflict:
            return "ABSTAIN"

        return "MATCH"

    return "ABSTAIN"
```

**模型没有权限把不同 verified reference 的商品“纠正”为 MATCH。**

---

# 8. 直接可落地的总体架构

```text
雷小安 ─┐
腕表之家 ├──────────────┐
奢当家 ─┘              │
                       ▼
                Raw Product Store
                       │
                       ▼
             Source Field Normalizer
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
  Structured Fields  Title/Desc    Images
          │            │            │
          │            │      ┌─────┴────────┐
          │            │      ▼              ▼
          │            │  Image Role      OCR Worker
          │            │  Classifier         │
          └────────────┴────────────┬─────────┘
                                    ▼
                         Identifier Candidate Miner
                                    │
                                    ▼
                        Identifier Role / Ownership
                                    │
                                    ▼
                     Conservative Reference Canonicalizer
                                    │
                                    ▼
                         Brand + Reference Verifier
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
                VERIFIED_REFERENCE          UNRESOLVED
                         │                     │
                         ▼                     ▼
                  Deterministic Join     Brand/Series Block
                         │                     │
                         ▼                     ▼
                     Entity Key          ANN Candidate Retrieval
                                               │
                                               ▼
                                  Intermediate Fusion Verifier
                                               │
                                      ┌────────┴────────┐
                                      ▼                 ▼
                                 Review Queue        NO CANDIDATE
                                      │
                                      ▼
                              Human confirms reference
                                      │
                                      ▼
                               VERIFIED_REFERENCE
```

注意这里 **Intermediate Fusion 被放在 unresolved 支路，而不是主合并路径。**

这才符合“图片可用，但 reference 是唯一实体定义”的业务语义。

---

# 9. 第一层：Source Field Normalizer

三个来源不要直接 flatten 成统一大文本。

先对每个来源定义字段语义：

```yaml
source: lei_xiao_an
fields:
  product_id: ...
  title: ...
  brand: ...
  explicit_model: ...
  explicit_reference: ...
  source_sku: ...
  seller_inventory_id: ...
  description: ...
  images: ...
```

核心目标是保留字段 provenance：

```text
“126334”来自结构化 model 字段
```

和：

```text
“126334”来自标题里“适配 126334 表带”
```

在语义上完全不是同一个证据等级。

每个来源都应维护字段可信度配置，例如：

```text
explicit_reference > explicit_model > title-labelled-reference > free-text-token > OCR-token
```

但**权重高也不代表可以覆盖冲突**。如果结构化字段和高可信 OCR 明确给出两个不同 reference，应进入 `CONFLICT`，而不是简单取更高权重那个。

---

# 10. Identifier Candidate Miner：先保留所有候选，不要急着选一个

输出建议格式：

```json
{
  "product_id": "lx-123",
  "raw_value": "126334",
  "normalized_value": "126334",
  "origin": "title",
  "field": "title",
  "span": [18, 24],
  "left_context": "型号",
  "right_context": "蓝盘",
  "image_id": null,
  "ocr_box": null,
  "extractor_version": "ref-miner-v3"
}
```

从以下位置全部抽候选：

1. 结构化 reference/model 字段；
2. 标题；
3. 描述；
4. 属性表；
5. 图片 OCR；
6. 可选：图片 caption / VLM 生成的结构化属性。

但不要自由生成 reference。

对于 LLM/VLM，推荐限制为：

```text
从给定文本/OCR token 中选择候选
```

而不是：

```text
请猜这块表是什么型号
```

后者会引入 hallucinated reference，precision-first 场景非常危险。

---

# 11. Identifier Role / Ownership：论文前面还需要增加的一层

一个“长得像 reference 的字符串”不等于“当前商品自身的 reference”。

建议标签至少包含：

```text
SELF_REFERENCE
PLATFORM_SKU
SELLER_INVENTORY_ID
SERIAL_NUMBER
ACCESSORY_TARGET_REFERENCE
COMPATIBLE_REFERENCE
DOCUMENT_ID
OTHER_IDENTIFIER
UNKNOWN
```

输入不只看字符串形态，还看：

```text
raw token
+ 左右上下文
+ 来源字段名
+ source
+ product category
+ image role
```

例如：

```text
“原装表带 适配 126334 / 126300”
```

`126334` 可能是一个完全合法的 Rolex reference，但角色应是：

```text
ACCESSORY_TARGET_REFERENCE
```

而不是：

```text
SELF_REFERENCE
```

这一层可以先用规则 + 小模型，效果通常比一上来训练大多模态 matcher 更可控。

---

# 12. Conservative Reference Canonicalizer

reference 规范化必须保守。

安全的通用操作：

```text
Unicode NFKC
trim
uppercase
统一明确等价的全角/半角字符
移除字段标签前缀（如 Ref., 型号:）
```

危险的通用操作：

```text
删除所有 '-' '/' '.'
删除所有前导 0
模糊编辑距离自动合并
把 O/0、I/1 全局互换
```

这些操作可能把原本不同的 reference 合并。

正确做法是：

```text
Global conservative normalization
        +
Brand-specific rewrite rules
        +
Validated alias table
```

例如维护：

```text
reference_alias(
  brand_id,
  raw_normalized,
  canonical_reference,
  rule_source,
  validated,
  version
)
```

所有自动规则都必须版本化，可以回滚。

---

# 13. Reference Resolution 状态机

每条记录不要只有一个 `reference` 字符串，至少要有：

```text
reference_state =
  VERIFIED
  CANDIDATE
  CONFLICT
  UNKNOWN
  REJECTED
```

建议状态逻辑：

## VERIFIED

满足：

```text
brand 已 canonicalize
SELF_REFERENCE 候选唯一
canonicalization 无歧义
来源/上下文证据足够
无高置信冲突
```

## CONFLICT

例如：

```text
结构化 reference = 126334
OCR 保卡 reference = 126300
```

或者：

```text
标题强标注一个 reference
属性表又强标注另一个 reference
```

CONFLICT 必须拒绝自动实体合并。

## UNKNOWN

没有找到 reference，或者只有模糊视觉猜测。

## CANDIDATE

找到了可能 reference，但证据不足以进入 VERIFIED。

---

# 14. 论文 Intermediate Fusion 在当前系统中的正确位置

原论文：

```text
text + image -> fusion -> MATCH
```

当前系统应改为：

```text
unresolved product
+ candidate reference/entity profile
        │
        ▼
reference-aware text evidence
+ multi-view image evidence
+ OCR/attribute conflict features
        │
        ▼
Intermediate Fusion Verifier
        │
        ▼
ranking / review confidence
        │
        ▼
人工或 reference resolver 确认
        │
        ▼
VERIFIED reference
```

即：**模型回答“这个候选 reference 值得不值得继续看”，而不是直接回答“可以把两个实体合并”。**

---

# 15. 对论文 Text Branch 的生产改造：Reference-aware Evidence Text

不要只输入原始标题：

```text
A_title [SEP] B_title
```

建议构造受控 evidence packet：

```text
[SOURCE] lei_xiao_an
[BRAND] ROLEX
[CATEGORY] watch
[TITLE] 劳力士日志型蓝盘...
[EXPLICIT_REF] 126334
[REF_CANDIDATES] 126334
[REF_CONTEXT] 型号 126334 蓝盘
[OCR_REF] none
```

候选另一侧可以是：

```text
[SOURCE] watch_home
[BRAND] ROLEX
[CATEGORY] watch
[TITLE] ...
[EXPLICIT_REF] none
[REF_CANDIDATES] 126334, 126300
[REF_CONTEXT] ...
[OCR_REF] 126334
```

这样 RoBERTa/Transformer 学到的不是营销词相似度，而是**证据结构是否一致**。

对中文三源数据可以替换为适合中文/多语言的预训练 encoder，但没有必要为“模型更大”而改架构；真正重要的是输入语义是否与 reference resolution 对齐。

---

# 16. 对论文 Image Branch 的生产改造：多图、多角色、OCR 优先

原论文每个商品基本是一张标准商品图；当前二奢通常有多张图。

不要简单：

```text
所有图片 embedding 平均
```

因为保卡、盒子和手表正面平均到一起会污染表示。

建议：

```text
images
  │
  ▼
Image Role Classifier
  │
  ├─ WATCH_FRONT -> visual encoder
  ├─ WATCH_BACK  -> visual encoder + OCR
  ├─ CARD        -> OCR
  ├─ TAG         -> OCR
  ├─ BOX         -> weak visual evidence
  └─ OTHER       -> ignored / low weight
```

每个角色内部保留 top-K 表征，例如：

```text
front_embeddings: top 2
back_embeddings: top 2
ocr_reference_candidates: all high confidence
```

Pair verifier 的图像特征可以采用对称聚合：

```text
max cosine across valid image pairs
mean top-3 cosine
min top-3 cosine
abs(proto_A - proto_B)
proto_A * proto_B
```

但这些仍只是辅助证据。

### OCR 应独立于视觉相似分支

reference 如果出现在：

- 表背；
- 保卡；
- 吊牌；
- 鉴定单；

OCR 得到的是**符号级 identifier 证据**，价值远高于“图片看起来像”。

因此 OCR 不应该只隐藏在 Swin/CLIP embedding 中，而要显式进入 `identifier_evidence` 表。

---

# 17. 推荐的 Multimodal Verifier 结构

在需要模型复核 unresolved candidate 时，可以保留论文 intermediate fusion 思想，但改成安全的对称结构：

```text
A evidence text ─┐
                 ├─ Text Pair Encoder -> text_pair_256
B evidence text ─┘

A watch images -> Image Encoder -> image_A_256
B watch images -> Image Encoder -> image_B_256

symmetric_image_features = [
    abs(image_A_256 - image_B_256),
    image_A_256 * image_B_256,
    cosine(image_A_256, image_B_256)
]

structured_features = [
    brand_equal,
    category_equal,
    explicit_ref_equal,
    explicit_ref_conflict,
    ocr_ref_equal,
    ocr_ref_conflict,
    accessory_flag,
    source_pair,
    ref_role_confidence,
    ref_parser_confidence
]

fusion = concat(
    text_pair_256,
    symmetric_image_features,
    structured_features
)

fusion -> MLP -> candidate_score
```

注意：

```text
candidate_score
```

只用于：

- 排序；
- 决定是否进入人工复核；
- 帮助 reference resolver 选择候选；
- 数据质量冲突报警。

不要写：

```python
if candidate_score > 0.99:
    merge()
```

---

# 18. “同 reference 自动合并”的硬 Gate

推荐的自动合并规则：

```python
def deterministic_entity_key(r):
    if r.reference_state != "VERIFIED":
        return None
    if r.brand_id is None:
        return None
    if r.canonical_reference is None:
        return None
    if r.has_hard_conflict:
        return None
    if r.product_role != "MAIN_PRODUCT":
        return None

    return hash_key(r.brand_id, r.canonical_reference)
```

跨源 join：

```sql
SELECT
  a.product_id,
  b.product_id,
  a.brand_id,
  a.canonical_reference
FROM reference_resolution a
JOIN reference_resolution b
  ON a.brand_id = b.brand_id
 AND a.canonical_reference = b.canonical_reference
WHERE a.reference_state = 'VERIFIED'
  AND b.reference_state = 'VERIFIED'
  AND a.has_hard_conflict = false
  AND b.has_hard_conflict = false
  AND a.source <> b.source;
```

索引：

```sql
CREATE INDEX idx_ref_verified
ON reference_resolution (
  brand_id,
  canonical_reference,
  source
)
WHERE reference_state = 'VERIFIED'
  AND has_hard_conflict = false;
```

这样主匹配路径是数据库索引/hash join，不需要把千万商品送进 pair classifier。

---

# 19. 建议的数据表

## 19.1 `raw_product`

```text
id
source
source_product_id
raw_payload
content_hash
source_updated_at
ingested_at
```

`(source, source_product_id)` 唯一。

## 19.2 `identifier_evidence`

```text
id
product_id
raw_value
normalized_value
origin
field_name
image_id
ocr_box
left_context
right_context
role
role_confidence
extractor_version
created_at
```

## 19.3 `reference_resolution`

```text
product_id
brand_id
canonical_reference
reference_state
primary_evidence_id
supporting_evidence_ids
conflict_codes
has_hard_conflict
resolver_version
updated_at
```

## 19.4 `product_embedding`

```text
product_id
text_embedding
image_role
image_id
image_embedding
embedding_version
```

只给 unresolved / review 路径使用 ANN；不需要让全部已解析商品都参与 pairwise search。

## 19.5 `entity`

```text
entity_key
brand_id
canonical_reference
created_at
```

## 19.6 `entity_member`

```text
entity_key
product_id
source
resolver_version
```

### 不建议物理 merge 原始记录

保留原始 `product_id`，实体关系只做派生 membership。

这样 reference parser 更新后可以：

```text
recompute resolution
-> recompute entity membership
```

而不需要恢复被破坏的原始数据。

这是 precision-first 系统非常重要的可回滚设计。

---

# 20. 大规模 100 万–1000 万的候选检索策略

## 20.1 已解析商品

```text
record
 -> reference resolver
 -> entity_key
 -> index lookup/upsert
```

复杂度接近：

```text
O(N)
```

不做 O(N²)。

## 20.2 Unresolved 商品

只有 `UNKNOWN/CANDIDATE` 进入检索：

```text
brand_id block
  + product category block
  + optional series block
        │
        ▼
ANN top-K（例如 K=20）
        │
        ▼
Intermediate Fusion rerank
        │
        ▼
Top-3 / Top-5 Review
```

向量索引可以按品牌/品类分区，避免：

```text
Rolex watch
```

去和所有包、首饰、鞋子搜索。

## 20.3 更进一步：检索 reference profile，而不是历史 offer

可以为每个：

```text
(brand_id, canonical_reference)
```

建立 `reference_profile`：

```text
canonical title aliases
structured attributes
verified text prototypes
verified image prototypes
known reference forms
known negative neighbors
```

未解析商品先检索 reference profile，而不是百万历史 listing。

这会把搜索目标从：

```text
offer -> offer
```

变成：

```text
offer -> canonical reference entity
```

和业务定义更加一致，也更容易人工解释。

---

# 21. 训练集应该怎么构造

## 21.1 Positive

优先使用高可靠样本：

```text
两个来源
brand 一致
verified canonical reference 一致
无冲突
```

## 21.2 Hard Negative

按以下顺序采：

### 第一类：同 brand、同 series、不同 reference

这是最重要的。

### 第二类：ANN 视觉近邻、不同 reference

复用论文的思路。

### 第三类：文本近邻、不同 reference

尤其是卖家复制模板造成的标题高度相似。

### 第四类：Accessory leakage

```text
表带/表盒/保卡
标题里出现目标腕表 reference
```

### 第五类：identifier role confusion

```text
source SKU
serial number
inventory id
```

与品牌 reference 形态相似。

### 第六类：OCR confusion

OCR 把：

```text
O / 0
I / 1
B / 8
S / 5
```

读错的 near-reference。

## 21.3 Train/Test split

不要随机 pair split。

至少做：

```text
reference-level group split
```

更严格再做：

```text
brand holdout
reference-family holdout
source-pair holdout
time holdout
```

最终最重要的是：

```text
未来新增批次/新 reference 上的 false positive
```

而不是训练集邻居上的 F1。

---

# 22. 几百对黄金标签怎么用

Spec 允许人工标几百对，这足以：

- 校准 candidate ranking；
- 构建高价值 hard-negative 集；
- 训练轻量 role classifier；
- 发现规则盲区；
- 做初始阈值选择。

但**几百对不能证明“99.99% precision”这类极高统计保证。**

举一个量级说明：如果自动接受样本中观察到 0 个 false positive，使用一侧 95% 精确二项置信下界，全部成功时下界约为：

```text
p_lower = 0.05^(1/n)
```

所以：

```text
n = 300  -> 下界约 99.006%
n = 500  -> 下界约 99.403%
n = 1000 -> 下界约 99.701%
```

若想仅靠“0 个错误的 IID 审计样本”把一侧 95% 下界推到 99.99%，量级约需要：

```text
n ≈ 29,956
```

这只是统计量级说明，而且实际数据并非严格 IID。

因此当前系统安全性不能寄希望于“几百标签训练一个 0.99 阈值模型”，而应该主要来自：

```text
reference identity hard rule
+ conservative canonicalization
+ conflict abstention
+ continuous audit
```

模型用于扩大 coverage，而不是定义安全边界。

---

# 23. 评估指标：不要再以 F1 为主

论文报告 F1/Precision/Recall 是合理的研究指标，但当前业务应该增加生产指标。

推荐 Dashboard：

```text
AutoMatch Precision
AutoMatch Coverage
False Positive Count
Abstain Rate
Reference Resolution Rate
Reference Conflict Rate
Review Queue Precision
Review Queue Yield
New/Unknown Reference Rate
OCR Reference Accuracy
Role Classification Precision(SELF_REFERENCE)
```

其中最核心：

```text
AutoMatch Precision
False Positive Count
```

并且要按以下维度切片：

```text
source pair
brand
reference family
product category
parser version
model version
time window
```

整体 precision 很高并不能掩盖某个新来源突然出现大量 source SKU 误判。

---

# 24. 人工审核 UI 应展示“证据”，而不是只展示模型分数

审核卡片建议：

```text
左：商品 A
右：候选 reference profile / 商品 B

[Brand]
A: ROLEX
B: ROLEX

[Reference Evidence]
A structured: 126334
A title:       126334 (highlighted)
A OCR:         none

B structured: none
B title:       126334 (highlighted)
B OCR card:    126334 (crop shown)

[Conflicts]
none

[Images]
front / back / card 分角色展示

[Model]
multimodal candidate score: 0.94
image nearest-neighbor rank: 2
text nearest-neighbor rank: 1
```

人工操作不要只提供：

```text
Match / No Match
```

还应提供：

```text
Confirm reference
Wrong reference extraction
Accessory / compatible reference
Platform SKU
OCR error
Brand error
Need more evidence
```

这样反馈可以直接训练最上游的 reference resolver，而不是只累积不容易复用的 pair 标签。

---

# 25. 冲突证据 Gate

对于 precision-first，负证据通常比弱正证据更重要。

建议 hard conflict：

```text
verified reference mismatch
verified brand mismatch
product role = ACCESSORY vs MAIN_PRODUCT
explicit model conflict
multiple high-confidence SELF_REFERENCE candidates
reference catalog says invalid for brand
```

soft conflict：

```text
image category mismatch
size mismatch
material mismatch
series mismatch
OCR weak mismatch
```

决策规则：

```text
hard conflict -> ABSTAIN / NO_MATCH
soft conflict -> lower candidate rank / review
```

不要让高图片相似度覆盖 `verified reference mismatch`。

---

# 26. 论文组件到当前方案的映射

| 论文组件 | 是否保留 | 当前方案中的角色 |
|---|---|---|
| RoBERTa pair encoder | 保留思想，改输入 | reference-aware evidence text 编码 |
| BiLSTM + hybrid pooling | 可保留做 baseline | 强调局部 reference token 与整体语义 |
| Swin-Transformer | 保留/可替换同类视觉 encoder | 主体图片视觉表征 |
| Siamese image branch | 保留 | 学习“近似变体”的细粒度差异 |
| Euclidean distance | 仅辅助 | 不直接决定匹配 |
| 两个 256-d image high-level embedding | 保留思想 | 融合前保留高层细节 |
| Intermediate Fusion MLP | 保留 | unresolved candidate verifier/ranker |
| 直接 binary MATCH 输出 | **不保留为自动合并依据** | 仅 review score |
| MPN + manual ground truth | 强烈保留 | reference 黄金数据构建 |
| ANN hard negative mining | 强烈保留 | 同系列不同 reference 难负样本 |
| F1 优化 | 降级 | 主要看 auto-match precision / FP |

---

# 27. 最小可行版本（MVP）建议

如果现在要直接开始落地，不建议第一步训练论文里的全量多模态网络。

## Phase 1：先建立 deterministic reference pipeline

实现：

```text
source schema mapping
brand canonicalization
identifier candidate miner
conservative reference canonicalizer
reference_resolution table
entity_key exact join
conflict / abstain
```

这一阶段就能覆盖所有“有干净 reference”的记录，并且 precision 最可控。

## Phase 2：补 identifier role + OCR

实现：

```text
SELF_REFERENCE vs SKU/SERIAL/ACCESSORY_REFERENCE
image role classification
OCR on card/back/tag/document
reference evidence provenance
```

这会显著提高可解析率。

## Phase 3：构建 hard-negative 数据集

针对：

```text
same brand
same series
visual nearest neighbor
text nearest neighbor
reference different
```

构建专用评测集。

## Phase 4：上 Intermediate Fusion Verifier

先只做：

```text
unresolved top-K rerank
人工 review 辅助
```

观察真实线上 false-positive pattern 后，再考虑是否给某些非常受限的品牌/来源规则开放更多自动化。

---

# 28. 推荐服务拆分

不需要一开始微服务化过度，但逻辑边界建议明确：

```text
1. ingest-worker
   三源增量同步、幂等

2. evidence-extractor
   字段解析、候选 identifier、上下文

3. image-worker
   image role、OCR、visual embedding

4. reference-resolver
   role classification、canonicalization、conflict state

5. entity-indexer
   verified entity_key upsert / membership

6. candidate-retriever
   unresolved blocking + ANN top-K

7. multimodal-verifier
   intermediate fusion score

8. review-service
   人工审核与反馈闭环

9. audit-job
   采样已自动匹配结果，持续统计 precision / drift
```

对于最初版本，1–5 可以在一个 Python 服务 + PostgreSQL 中完成；ANN 和图像 worker 再独立。

---

# 29. 增量与版本管理

持续增量场景一定要版本化：

```text
extractor_version
ocr_version
brand_normalizer_version
reference_parser_version
role_model_version
embedding_version
fusion_model_version
```

每次规则/模型升级不要覆盖历史解释，而是保留：

```text
old resolution
new resolution
change reason
```

特别关注：

```text
VERIFIED -> CONFLICT
VERIFIED reference A -> reference B
```

这两种变化应该触发实体 membership 重算和告警。

因为实体关系是派生的 `entity_key`，而不是物理 merge，所以回滚成本很低。

---

# 30. 缓存与计算成本

论文中 RoBERTa/Swin 是 pair 判别网络的一部分；但当前 1000 万规模不能每次 candidate 比较都重复编码相同商品。

应该缓存 record-level 特征：

```text
normalized text hash -> text embedding
image content hash -> image embedding
OCR image hash -> OCR result
```

更新时：

```text
content_hash unchanged -> reuse
text changed -> only recompute text branch
new image -> only compute new image/OCR
reference parser changed -> reuse embeddings, rerun resolver
fusion MLP changed -> reuse all embeddings
```

这样模型升级不会导致全量重新跑昂贵视觉 backbone。

---

# 31. 对“图片可用”这条需求的最终定位

图片有三种价值，优先级不同。

## 价值 1：从图片中读取 reference / identifier

最高价值。

```text
OCR(card/back/tag)
```

因为它可以直接产生实体定义所需的 reference 证据。

## 价值 2：排除明显错误

例如：

```text
标题说 wristwatch
图片主体是 strap/box
```

或者同一候选 pair 的图片显示完全不同类别。

适合作为 conflict / data-quality gate。

## 价值 3：在 unresolved 中召回相似候选

适合论文的 Swin Siamese / Intermediate Fusion。

但优先级最低，且不能直接替代 reference。

因此实现顺序建议：

```text
OCR identifier evidence
>
image role / contradiction
>
visual similarity candidate ranking
```

而不是先做一个大视觉 matching 模型。

---

# 32. 一个完整决策例子

假设雷小安：

```text
brand = Rolex
title = “劳力士日志型 126334 蓝盘 全套”
structured_model = “126334”
images = 正面 + 表背 + 保卡
```

腕表之家：

```text
brand = 劳力士
title = “日志型 41 蓝色盘面”
structured_model = null
OCR(card) = “126334”
```

处理：

```text
brand canonicalize -> ROLEX / ROLEX

雷小安：
  structured 126334 -> SELF_REFERENCE
  title 126334      -> SELF_REFERENCE support
  state -> VERIFIED

腕表之家：
  title 无明确 ref
  OCR card 126334 -> candidate
  image_role=CARD
  如果 OCR + context/catalog validation 足够 -> VERIFIED

最终：
  brand equal
  canonical_ref equal
  no hard conflict
  -> deterministic MATCH
```

这里 Intermediate Fusion 完全可以不运行。

再看危险例子：

```text
奢当家：
“原装劳力士表带 适配 126334”
```

Identifier Role：

```text
126334 -> ACCESSORY_TARGET_REFERENCE
product_role -> ACCESSORY
```

即使图片/文字与 126334 腕表相关，也不能生成：

```text
entity_key = ROLEX + 126334
```

这类 false positive 正是通用 pair matcher 最容易犯的错误。

---

# 33. 最终落地建议

这篇论文最值得当前项目复制的不是“RoBERTa + Swin”这两个模型名字，而是三个架构思想：

### 1. Hard negative 必须来自“最像但实际不同”的候选

腕表里就是：

```text
同品牌 / 同系列 / 不同 reference
```

### 2. 不要过早压缩多模态信息成一个 similarity 标量

图像和文本应先形成任务相关的高层表示，再融合；尤其图像分支应保留高层 embedding，而不是只传 Euclidean distance。

### 3. 多模态模型适合解决“模糊证据”，不适合覆盖“确定性 identifier”

结合当前 Spec，推荐最终职责划分：

```text
Reference Extractor / Resolver
    负责安全地产生 canonical reference

Deterministic Entity Join
    负责真正的自动 MATCH

Intermediate Fusion Verifier
    负责 unresolved 的 candidate ranking、冲突发现和人工复核提效
```

最重要的生产规则可以压缩成一句话：

> **能证明 reference 一致时，用确定性 join；不能证明时，用多模态模型帮忙找证据，但模型本身不能把“相似”升级成“同一 reference”。**

这套方案既吸收了论文 intermediate fusion 在困难商品匹配上的优势，又把其 0.71–0.87 级 precision 无法满足“绝不能误匹配”的风险隔离在自动实体合并之外，适合当前三源、千万级、持续增量的二奢/腕表数据场景。