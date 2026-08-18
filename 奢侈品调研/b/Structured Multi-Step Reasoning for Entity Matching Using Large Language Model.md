# Structured Multi-Step Reasoning for Entity Matching Using Large Language Model

## 0. 结论先行

论文 **Structured Multi-Step Reasoning for Entity Matching Using Large Language Model** 的核心价值，不是“让 LLM 直接判断两个商品是不是同一款”，而是把原本一次性输出 `match / non-match` 的黑盒判断，拆成可审计的中间步骤：

1. 先找两条记录中匹配与不匹配的 token；
2. 再判断哪些属性真正影响匹配；
3. 最后基于前两步证据做实体级判断。

论文还尝试了正反双方论证的 debate 方案，但实验反而显著变差，说明“让模型多说、多辩论”并不天然等于更可靠。

对当前 Notion Spec「雷小安 × 腕表之家 × 奢当家」而言，最值得借鉴的是 **显式证据分层 + 决策链可审计**，但论文原始方案不能直接作为生产匹配器。原因是当前业务定义远比通用 Entity Matching 更严格：

> 同一个商品 = **同一 reference number / 型号**；并且 **绝对不能误匹配，precision 优先到极致，允许漏匹配**。

因此建议把论文改造成一个 **ReferenceGuard（型号证据审计器）**：LLM 只负责把“为什么像 / 为什么不像 / 哪个字段是 reference / 是否存在冲突”结构化，最终自动合并仍由 deterministic gate 决定。

推荐落地架构：

```text
原始三源商品
   ↓
字段与图片 OCR 证据抽取
   ↓
identifier role classification
(reference / serial / platform SKU / store SKU / caliber / GTIN / unknown)
   ↓
保守 canonical reference 规范化
   ↓
品牌内 Blocking / 候选生成
   ↓
ReferenceGuard 三阶段证据审计
   ↓
Hard Precision Gate
   ├─ AUTO_MATCH：reference 强证据严格一致 + 无冲突
   ├─ REJECT：reference 明确冲突 / 编号角色冲突
   └─ ABSTAIN：证据不足、来源不可靠、模型不稳定 → 人工或等待更多数据
   ↓
可信边聚类 + 可回滚审计日志
```

关键原则：**LLM 可以否决、解释、补充抽取，但不能越权把 reference 不一致的记录判成同款。**

---

## 1. 论文来源与研究目标

论文：

- Rohan Bopardikar, Jin Wang, Jia Zou
- *Structured Multi-Step Reasoning for Entity Matching Using Large Language Model*
- arXiv:2511.22832, 2025-11-28
- https://arxiv.org/abs/2511.22832
- HTML: https://arxiv.org/html/2511.22832v1

论文关注的是 Entity Matching 的 **matching stage**。作者把传统流程概括为：

```text
Blocking → Labeling → Matching
```

其中本文不研究 blocking，而只研究给定候选 pair 后，怎样通过更结构化的 LLM reasoning 提高匹配效果。

论文指出大多数 LLM-based EM 仍然是单步 prompt：

```text
record A + record B
   ↓
LLM
   ↓
Match: Yes / No
```

问题是这种做法没有显式暴露 token、属性、冲突证据，因此难以解释模型究竟基于什么做判断，也容易被表面语义相似误导。

---

## 2. 论文方法拆解

## 2.1 Prompt 设计空间

作者先讨论了四个 prompt 维度：

### 2.1.1 General vs Domain-Specific

通用指令：

```text
Determine whether these two objects match.
```

领域化指令：

```text
Determine whether these two product listings represent the same consumer electronic device.
```

领域提示可以给模型额外先验，但也可能让模型过度使用领域语义，而忽略真正的主键证据。

在腕表场景中，这个风险尤其明显：

```text
Rolex Submariner Date 126610LN
Rolex Submariner Date 126610LV
```

两条记录在“品牌、系列、尺寸、外观、用途”上极其相似，但当前业务定义下必须判为不同商品，因为 reference 不同。

所以我们的领域 prompt 不能只是：

```text
判断是否为同一腕表款式
```

而应该明确约束：

```text
只有 canonical reference number 相同才允许判定为同一商品。
品牌、系列、图片相似度、标题语义相似度只能作为辅助或冲突证据，不能覆盖 reference 冲突。
```

### 2.1.2 Simple vs Complex Verbiage

论文比较简短与复杂说明式 prompt。复杂 prompt 能提供更多上下文，但增加 token 成本和模型认知负担。

对生产系统而言，不应该把所有规则都塞成自然语言长 prompt；稳定规则应代码化。例如：

```python
if canonical_ref_a and canonical_ref_b and canonical_ref_a != canonical_ref_b:
    return REJECT_REFERENCE_CONFLICT
```

而不是让模型阅读 1000 token 的规则后“自行遵守”。

### 2.1.3 Free vs Forced Response

论文讨论自由文本输出与强制格式输出。

本项目必须使用 **强结构输出**。推荐：

```json
{
  "reference_a": {
    "raw": "126610LN",
    "canonical": "126610LN",
    "role": "manufacturer_reference",
    "evidence": ["title"],
    "confidence": 0.99
  },
  "reference_b": {
    "raw": "126610LV",
    "canonical": "126610LV",
    "role": "manufacturer_reference",
    "evidence": ["title", "ocr_caseback"],
    "confidence": 0.99
  },
  "conflicts": ["canonical_reference_mismatch"],
  "recommended_action": "REJECT"
}
```

最终 action 仍由服务端重算，不能直接信任模型字段 `recommended_action`。

### 2.1.4 In-Context Learning

论文实验 zero-shot 与 2-shot。结果表明 few-shot 并非稳定提升 reasoning 方案。

这对本项目很重要：不要假设“塞两个例子进去”就能解决 precision 问题。更好的做法是把有限的几百对黄金标签用于：

- 建立 reference 角色分类器；
- 校准自动放行阈值；
- 构造 hard-negative 回归集；
- 验证不同品牌 / 来源 / OCR 模式下的 false positive；
- 做 prompt regression，而不是只做 prompt demonstration。

---

## 2.2 三阶段 Reasoning Framework

论文的核心方法为：

```text
Step 1: matched / unmatched tokens
Step 2: influential attributes
Step 3: entity-level decision
```

有两种执行方式。

### Multi-Prompt

三个步骤分别调用模型：

```text
Prompt 1 → Response 1
Response 1 + Prompt 2 → Response 2
Response 1 + Response 2 + Prompt 3 → Decision
```

优点：每步职责清晰、输出易检查。

缺点：成本高，而且早期错误会被串行放大。

### Single-Prompt

一个 prompt 要求一次性输出三阶段结果：

```text
Input pair
  ↓
LLM
  ↓
{
  step1_token_evidence,
  step2_attribute_importance,
  step3_decision
}
```

优点：调用便宜，延迟更低。

在生产场景中更适合用作在线审计器。

---

## 2.3 Debate-based Framework

论文还设计：

```text
支持 match 的理由
       +
反对 match 的理由
       ↓
最终综合判断
```

理念是减少过度自信。

但论文实际发现 debate 明显更差，例如 DBLP-ACM zero-shot 只有约 `0.835 F1`，因此作者没有把完整 debate 结果放入主表。

这说明一个很实用的工程结论：

> 对 identifier-sensitive 的实体匹配，不要用“多代理辩论”替代硬约束。

腕表 reference 的冲突不是观点问题：

```text
126610LN != 126610LV
```

不存在“支持方觉得外观很像，所以可以覆盖型号冲突”的空间。

因此本项目不建议采用 debate 作为生产判定模块。

---

## 3. 实验结果：为什么只能借鉴结构，不能照搬结论

论文使用 6 个公开 Entity Matching 数据集：

- Abt-Buy
- DBLP-ACM
- DBLP-Scholar
- Walmart-Amazon
- Amazon-Google
- WDC Products（hard version，包含大量 corner cases）

模型使用 GPT-5.1-mini API。

论文 zero-shot 主结果大致为：

| Dataset | Baseline F1 | Single-Prompt F1 | Multi-Prompt F1 |
|---|---:|---:|---:|
| Abt-Buy | 0.849 | 0.849 | 0.862 |
| DBLP-ACM | 0.949 | 0.929 | 0.948 |
| DBLP-Scholar | 0.899 | 0.899 | 0.918 |
| Walmart-Amazon | 0.790 | 0.743 | 0.756 |
| Amazon-Google | 0.581 | 0.628 | 0.652 |
| WDC | 0.798 | 0.831 | 0.815 |

可以看到：

1. Reasoning 并不是所有数据集都提升；
2. Multi-Prompt 也不是稳定优于 Single-Prompt；
3. WDC 上 Single-Prompt 反而高于 Multi-Prompt；
4. Walmart-Amazon 上 reasoning 明显低于 baseline。

更关键的是成本。

论文中 zero-shot token 使用量例如：

```text
Abt-Buy:
Baseline      183 tokens
Single        559 tokens
Multi        1499 tokens

WDC:
Baseline      326 tokens
Single        695 tokens
Multi        2124 tokens
```

也就是说 Multi-Prompt 常常是 baseline 的数倍成本。

这进一步支持当前系统采用：

```text
代码硬规则 > 单次结构化 LLM > 多轮 LLM
```

而不是默认把所有候选送进三次调用。

---

## 4. 论文方法与当前 Spec 的关键错位

## 4.1 论文优化 F1，我们真正要优化 false positive

论文主指标是 F1。

但当前 Spec 的损失函数非常不对称：

```text
False Positive：灾难性错误
False Negative：可接受
```

所以我们应关注：

```text
Precision / PPV
False Positive Count
False Positive Rate at Auto-Match
Coverage at target precision
Abstention Rate
```

例如目标不是：

```text
F1 > 0.90
```

而更像：

```text
在自动匹配覆盖率 30% 的记录上，验证集 0 个或近似 0 个 false positive；
剩余 70% 全部 abstain / 待补证据。
```

系统宁可不自动合并，也不能错合并。

---

## 4.2 论文是通用“实体同一性”，本项目是“reference 等价性”

通用 EM 允许根据多字段综合判断：

```text
品牌相同
型号名称相似
价格相近
描述一致
图片相似
→ 大概率是同一实体
```

本项目定义更窄：

```text
canonical manufacturer reference 相同
→ 才有资格进入 MATCH
```

所以 reference 必须是 **hard identity key**，其他字段只能用于：

- 发现 reference 可能抽错；
- 识别编号角色；
- 检测 accessory / compatible-with 文本；
- 冲突否决；
- 提供人工审核上下文。

不能把品牌、系列、图片相似度累加后“抵消 reference 不一致”。

---

## 4.3 Token-level reasoning 对腕表 reference 还不够细

论文 Step 1 关注 matched / unmatched token。

但腕表 reference 的错误经常发生在字符级：

```text
126610LN
126610LV

IW371604
IW371606

L2.320.0.87.6
L2.320.4.87.6
```

语义 tokenizer 可能把它们编码为若干子 token；对 LLM 来说整体非常像，但业务含义完全不同。

因此需要扩展成：

```text
token-level evidence
        +
character-level identifier diff
        +
reference grammar validation
```

推荐输出：

```json
{
  "identifier_diff": {
    "a": "126610LN",
    "b": "126610LV",
    "edit_distance": 1,
    "common_prefix_len": 7,
    "different_suffix": ["N", "V"],
    "hard_conflict": true
  }
}
```

这种逻辑应该由代码生成，再喂给 LLM，而不是让 LLM 自己数字符。

---

## 4.4 二分类缺乏 ABSTAIN

论文最终是：

```text
match ∈ {0, 1}
```

当前业务至少需要三态：

```text
MATCH
REJECT
ABSTAIN
```

建议再细分 reason code：

```text
AUTO_MATCH_EXACT_REFERENCE
REJECT_REFERENCE_CONFLICT
REJECT_BRAND_CONFLICT
REJECT_IDENTIFIER_ROLE_CONFLICT
ABSTAIN_MISSING_REFERENCE
ABSTAIN_AMBIGUOUS_REFERENCE
ABSTAIN_OCR_ONLY
ABSTAIN_ACCESSORY_CONTEXT
ABSTAIN_MODEL_DISAGREEMENT
```

这样才能真正支撑线上审计、回溯和迭代。

---

## 5. 可直接落地的 ReferenceGuard 架构

## 5.1 数据层：原始证据绝不覆盖

建议保存四类对象。

### `product_record`

```sql
product_record(
  record_id,
  source,               -- leixiaoan / xcar_watch / shedangjia
  source_item_id,
  title_raw,
  description_raw,
  raw_json,
  image_urls,
  fetched_at,
  parser_version
)
```

### `identifier_observation`

每次发现一个像编号的字符串，都先作为 observation，而不是直接写 reference：

```sql
identifier_observation(
  observation_id,
  record_id,
  raw_value,
  normalized_value,
  identifier_role,      -- manufacturer_reference / serial / sku / caliber / gtin / unknown
  evidence_type,        -- field / title / description / ocr_caseback / ocr_card
  evidence_location,
  extractor,
  extractor_version,
  role_confidence,
  created_at
)
```

### `reference_resolution`

```sql
reference_resolution(
  record_id,
  canonical_brand,
  canonical_reference,
  resolution_status,    -- resolved / ambiguous / missing / conflict
  evidence_count,
  strongest_evidence,
  resolver_version,
  updated_at
)
```

### `match_edge`

```sql
match_edge(
  left_record_id,
  right_record_id,
  decision,             -- AUTO_MATCH / REJECT / ABSTAIN
  reason_code,
  canonical_reference,
  rule_version,
  llm_audit_version,
  evidence_json,
  created_at
)
```

不要直接物理合并原记录。匹配应首先表现为可撤销的可信边。

---

## 5.2 Stage A：Reference Candidate Extraction

数据源可能有三种情况：

```text
1. reference 在独立字段
2. reference 埋在标题 / 描述
3. reference 只出现在图片：表背、吊牌、保卡
```

抽取顺序建议：

```text
结构化字段
  > 品牌 reference 词典 / regex
  > 文本 candidate extractor
  > OCR candidate extractor
  > LLM fallback extractor
```

所有候选都带 provenance。

例：

```json
[
  {
    "raw": "126610LN",
    "source": "title",
    "role": "manufacturer_reference",
    "confidence": 0.99
  },
  {
    "raw": "SKU-6610921",
    "source": "seller_field",
    "role": "store_sku",
    "confidence": 0.98
  }
]
```

---

## 5.3 Stage B：Identifier Role Classification

这是当前系统非常关键、但通用 EM 往往忽略的一层。

同一个页面里可能同时有：

```text
manufacturer reference
serial number
platform SKU
seller SKU
caliber
GTIN
compatible model reference
```

例如标题：

```text
劳力士原装表带 适配 126610LN / 126610LV，货号 STRAP-8892
```

如果只做正则，会抽到：

```text
126610LN
126610LV
STRAP-8892
```

但这里两个 Rolex reference 都是“兼容对象”，不是当前商品本身。

所以角色分类至少应综合：

```text
字段名
上下文窗口
品牌词典
编号格式
商品类目
是否出现“适配/兼容/for/compatible”等关系词
图片位置
```

LLM 在这里比在最终 match 判定更有价值，因为这是语义角色理解问题。

---

## 5.4 Stage C：保守 Canonicalization

只允许无损或可验证变换：

```text
trim
uppercase
Unicode normalize
统一明确的分隔符表现
品牌特定、经过验证的等价格式
```

不要默认：

```text
删除全部字母
删除所有尾缀
模糊编辑距离归一
把 0/O、1/I 自动互换
```

因为 reference 一位字符就可能代表不同型号。

推荐保留：

```json
{
  "raw": "L2 320 0 87 6",
  "canonical": "L2.320.0.87.6",
  "normalization_rule": "longines_separator_rule_v1",
  "lossless": true
}
```

任何非确定性修复都应产生候选集合而不是唯一值：

```json
{
  "status": "ambiguous",
  "candidates": ["IW371604", "IW371606"]
}
```

然后进入 `ABSTAIN`。

---

## 5.5 Stage D：Blocking

千万级数据不能全量 pairwise。

建议 blocking：

```text
第一层：canonical_brand
第二层：reference 已解析记录 → hash(canonical_reference)
第三层：reference 未解析记录 → series / lexical identifier / ANN 做弱召回
```

最安全的自动匹配路径其实无需复杂 ANN：

```text
brand + canonical_reference exact
```

就可以形成高置信候选簇。

ANN 的主要用途应是：

- 为 reference 缺失记录找到可能的同款记录，帮助补充证据；
- 构造 hard negatives；
- 发现同系列相邻 reference；
- 进入审计或人工复核。

ANN 不应成为自动 identity key。

---

## 6. 把论文三阶段改造成“证据审计”

建议把论文三步重新定义。

## Step 1：Identifier Evidence Diff

不是泛化地让模型列 matched words，而是服务端先生成确定性差异：

```json
{
  "exact_matches": ["ROLEX", "SUBMARINER", "41MM"],
  "identifier_candidates_a": ["126610LN"],
  "identifier_candidates_b": ["126610LV"],
  "identifier_exact_match": false,
  "char_diff": {
    "left": "126610LN",
    "right": "126610LV",
    "edit_distance": 1
  }
}
```

LLM 再解释：

```text
LN 与 LV 是 reference 的区分字符，不应被视为可忽略文本差异。
```

## Step 2：Attribute / Role Importance

固定优先级由代码给出：

```text
P0 manufacturer reference
P0 identifier role
P0 brand
P1 product type / accessory context
P1 GTIN
P2 series / model family
P3 visual similarity
P3 price / condition / seller text
```

LLM 只需回答：

```text
当前证据中哪个字段属于 P0？是否存在角色歧义？
```

不要让 LLM 自行决定“价格可能比 reference 更重要”。

## Step 3：Decision Proposal

模型可以给 proposal：

```text
MATCH_CANDIDATE
REJECT_CANDIDATE
ABSTAIN
```

但服务端最终 gate 必须重算：

```python
def precision_gate(a, b, audit):
    if a.brand and b.brand and a.brand != b.brand:
        return "REJECT", "BRAND_CONFLICT"

    if a.reference_status == "resolved" and b.reference_status == "resolved":
        if a.canonical_reference != b.canonical_reference:
            return "REJECT", "REFERENCE_CONFLICT"

        if a.strong_reference_evidence and b.strong_reference_evidence:
            if not audit.has_p0_conflict:
                return "AUTO_MATCH", "EXACT_REFERENCE"

        return "ABSTAIN", "WEAK_REFERENCE_EVIDENCE"

    return "ABSTAIN", "REFERENCE_NOT_RESOLVED"
```

这才是论文 reasoning 在 precision-first 系统里的正确位置。

---

## 7. 推荐的 LLM 单次结构化 Prompt

在线路径不建议三次 Multi-Prompt，而建议单次 JSON 输出。

示意：

```text
System:
你是腕表商品标识符审计器，不是相似度匹配器。
业务定义：只有 manufacturer reference number 相同，两个商品才可能是同一商品。
reference 不同必须判冲突；证据不足必须 abstain。
品牌、系列、图片、价格、描述相似不能覆盖 reference 冲突。

Task:
1. 检查两个商品中的 identifier observations，并判断每个 identifier 的角色。
2. 检查 canonical reference 是否有明确冲突。
3. 检查是否为表带、配件、兼容对象、包装等导致标题出现别的商品 reference。
4. 只输出 JSON，不要自由发挥。

Input:
<record_a>...</record_a>
<record_b>...</record_b>
<server_generated_identifier_diff>...</server_generated_identifier_diff>

Output schema:
{
  "identifier_role_checks": [...],
  "reference_evidence": {
    "left": {...},
    "right": {...}
  },
  "p0_conflicts": [...],
  "accessory_or_compatibility_risk": true|false,
  "decision_proposal": "MATCH_CANDIDATE|REJECT_CANDIDATE|ABSTAIN",
  "reason_codes": [...]
}
```

注意：不要求模型输出浮点 `match_probability` 作为最终阈值。LLM 的概率通常不可直接解释为校准后的业务 precision。

---

## 8. 图像如何接入

论文只处理文本记录，本 Spec 明确有图片，因此应把图片当作 **reference 证据来源**，而不是相似度投票器。

优先用途：

```text
表背 OCR
保卡 OCR
吊牌 OCR
表盒标签 OCR
证书 OCR
```

例如：

```text
标题：Rolex Submariner 41mm 黑盘
图片 OCR：126610LN
```

此时 OCR 能补齐 reference。

但如果：

```text
A OCR = 126610LN
B OCR = 126610LV
视觉 embedding cosine = 0.98
```

仍然必须 REJECT。

视觉相似度只可用于：

```text
候选召回
发现疑似 OCR 错误
人工审核排序
```

不能覆盖 identifier conflict。

---

## 9. Incremental Pipeline：适配 100 万–1000 万持续增量

建议事件驱动而非每天全量重算。

```text
新商品进入
   ↓
extract_identifiers(record)
   ↓
resolve_reference(record)
   ↓
if resolved:
    lookup brand + canonical_reference index
    ↓
    对命中的现有簇执行冲突检查
else:
    进入 weak candidate discovery / OCR / LLM audit queue
```

索引：

```text
Redis / RocksDB / PostgreSQL B-tree:
(brand_id, canonical_reference) -> entity_cluster_id(s)
```

如果 reference 能稳定解析，单条增量匹配接近 O(1) / O(logN)，无需对千万条记录重新跑大模型。

LLM 只处理长尾：

```text
reference missing
reference ambiguous
identifier role ambiguous
OCR conflict
accessory context
```

因此成本可控。

---

## 10. 可信实体簇设计

不要让 pairwise edge 传递性自动扩散。

危险例子：

```text
A --match--> B
B --match--> C
```

不能直接推出：

```text
A --match--> C
```

除非三者最终 resolved canonical reference 完全一致。

建议 cluster invariant：

```text
一个 AUTO_MATCH cluster 内只能存在一个 canonical manufacturer reference。
```

插入新边前：

```python
assert incoming_reference == cluster.canonical_reference
```

如果冲突：

```text
阻止合并
生成 cluster_conflict 事件
送人工复核
```

---

## 11. 黄金标签应该怎么标

Spec 允许人工标几百对，建议不要随机抽样。

应集中在 hard cases：

### 40% 相邻 reference

```text
126610LN vs 126610LV
116610LN vs 126610LN
```

### 20% identifier role 混淆

```text
reference vs platform SKU
reference vs serial
reference vs caliber
```

### 15% 配件兼容上下文

```text
表带适配多个 reference
盒证 / 零件页出现目标型号
```

### 15% OCR 难例

```text
0/O
1/I
5/S
8/B
模糊点号和连字符
```

### 10% 真正的同 reference 跨源表达

```text
空格 / 点号 / 大小写 / 品牌合法格式差异
```

黄金集需要记录 reason，不只记录 0/1：

```json
{
  "label": 0,
  "reason": "reference_suffix_conflict",
  "left_reference": "126610LN",
  "right_reference": "126610LV"
}
```

这样才能用于回归测试和错误定位。

---

## 12. 线上 Precision Gate 建议

可先采用极保守 V1。

### AUTO_MATCH 必须同时满足

```text
1. 两侧 canonical brand 一致；
2. 两侧 canonical manufacturer reference 均 resolved；
3. reference 严格相等；
4. 两侧 reference 至少各有一个 strong evidence；
5. identifier role 不是 serial / SKU / caliber / compatible-reference；
6. 没有任何独立来源产生相反 reference；
7. 商品类目不是明确 accessory / parts；
8. cluster invariant 未冲突。
```

### 直接 REJECT

```text
明确 reference 不一致
明确品牌冲突
明确 identifier role 冲突
明确 accessory-to-main-product 关系
```

### 其余全部 ABSTAIN

这会牺牲 recall，但完全符合当前业务目标。

---

## 13. 评估指标与上线门槛

不要只报 F1。

建议 dashboard：

```text
AUTO_MATCH count
AUTO_MATCH precision
AUTO_MATCH false-positive absolute count
ABSTAIN rate
reference resolution coverage
reference conflict rate
manual review overturn rate
per-source precision
per-brand precision
per-extractor precision
OCR-only auto-match count（初期应为 0）
```

推荐初始门槛：

```text
OCR-only：禁止自动匹配
LLM-only reference：禁止自动匹配
semantic-only：禁止自动匹配
image-only：禁止自动匹配

结构化字段 exact ref + role verified：允许
标题 exact ref + 品牌格式校验 + 无冲突：可灰度
多来源一致 ref：优先放行
```

随着黄金集积累，再逐步扩大 coverage。

---

## 14. 版本化与审计

每条自动匹配需要完整记录：

```json
{
  "decision": "AUTO_MATCH",
  "reason_code": "EXACT_REFERENCE_STRONG_EVIDENCE",
  "canonical_reference": "126610LN",
  "left_evidence": [...],
  "right_evidence": [...],
  "normalizer_version": "refnorm-1.3.0",
  "role_classifier_version": "idrole-0.4.2",
  "llm_audit_version": "referenceguard-prompt-7",
  "rule_version": "precision-gate-3",
  "created_at": "..."
}
```

模型、prompt、正则和品牌词典更新后，都应能回答：

```text
这条边当时为什么被自动放行？
用的是哪个版本？
如果今天重跑，结果是否变化？
```

这样才能支持真正的 production rollback。

---

## 15. 建议的最小可落地 V1

不需要一开始就训练复杂模型。

### V1-1：Identifier observation pipeline

先实现：

```text
结构化字段抽取
标题 regex / pattern
OCR 接口
identifier_observation 表
```

### V1-2：品牌级 reference grammar

先覆盖腕表量最大的品牌，例如：

```text
Rolex
Omega
Longines
IWC
Cartier
Patek Philippe
Audemars Piguet
```

每个品牌维护：

```text
reference regex
separator normalization
合法字符集
已知前缀 / 长度
高风险相邻格式
```

### V1-3：Precision Gate

只自动匹配最安全的一层：

```text
品牌一致 + canonical reference exact + strong evidence + 无冲突
```

### V1-4：ReferenceGuard LLM

只进入：

```text
标题里有多个 reference
疑似配件
编号角色不清楚
OCR 与文本冲突
```

模型输出结构化审计，不直接合并。

### V1-5：人工台

展示：

```text
左右原始记录
图片
所有 identifier observations
字符级 reference diff
LLM 审计结果
规则 decision reason
```

人工选择：

```text
confirm match
confirm non-match
resolve identifier role
resolve canonical reference
```

结果回流黄金集。

---

## 16. 与论文原方案的最终取舍

| 论文设计 | 当前项目建议 |
|---|---|
| 单步 / 三步 LLM 判断是否同实体 | 三步结构用于证据审计，不让 LLM 越权合并 |
| matched / unmatched tokens | 升级为 identifier observation + 字符级 diff |
| LLM 自行判断重要属性 | 代码固定 reference / role 为 P0，LLM 只检查语义角色 |
| 二分类 Match / Non-match | MATCH / REJECT / ABSTAIN |
| F1 为主 | Auto-match precision、FP 绝对数、coverage 为主 |
| Multi-Prompt 可提升部分任务 | 在线优先 Single structured prompt，Multi 只做离线疑难审计 |
| Debate 提升鲁棒性的假设 | 实验表现差，生产不采用 |
| 文本 Entity Matching | 加入图片 OCR 作为 reference 证据，不把视觉相似当身份键 |
| 通用实体同一性 | 严格限定为 canonical manufacturer reference 等价性 |

---

## 17. 最终推荐方案

本论文对 Spec 最有价值的启发是：**不要让匹配器只吐一个不可解释的分数，要强制它暴露证据层。**

但在“绝对不能误匹配”的约束下，真正可落地的系统不应是：

```text
LLM 看两条商品 → 猜是不是同款
```

而应该是：

```text
原始证据
  ↓
编号候选抽取
  ↓
编号角色识别
  ↓
canonical reference
  ↓
字符级严格比较
  ↓
LLM 结构化审计语义冲突
  ↓
deterministic precision gate
  ↓
AUTO_MATCH / REJECT / ABSTAIN
```

如果只做一个近期可上线版本，我建议优先完成：

> **`identifier_observation + canonical_reference + hard precision gate + ReferenceGuard structured audit + 可回滚 match_edge`**

这套方案比直接微调一个通用 Entity Matching 模型更符合当前“reference 是唯一身份定义、precision 极端优先、允许漏匹配、数据持续增量”的业务约束，也能自然利用图片 OCR 和几百对黄金标签。
