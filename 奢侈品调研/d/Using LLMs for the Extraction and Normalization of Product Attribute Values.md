# Using LLMs for the Extraction and Normalization of Product Attribute Values：把 LLM 属性抽取改造成 precision-first 的腕表 reference 证据流水线

> 分析对象：Alexander Brinkmann, Nick Baumann, Christian Bizer, **Using LLMs for the Extraction and Normalization of Product Attribute Values**（ADBIS 2024）  
> 论文：https://arxiv.org/abs/2403.02130  
> 官方实现：https://github.com/wbsg-uni-mannheim/wdc-pave  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”按 **reference number / 型号一致**定义；字段稀疏；reference 可能埋在标题/描述中；有图片；**precision 极端优先，允许漏匹配**。

---

## 1. 结论先行

这篇工作对当前 Spec 最值得借鉴的，不是“让 GPT 一次性抽出很多商品属性”，而是它已经把电商属性识别拆成了几个可以工程化的组件：

1. **先定义结构化属性 Schema，再让模型输出受约束 JSON**；
2. **按商品类别检索语义相近的 few-shot 示例**，而不是把固定示例塞给所有商品；
3. **把 extraction 和 normalization 都显式建模**，并为不同属性配置不同 normalization guideline；
4. 对 `Part Number` 这种 identifier，论文数据中已经使用 **String Wrangling / Delete Marks** 这类规范化；
5. 代码用 Pydantic 做结构校验、用精确值匹配计算 precision / recall / F1，并记录 token 与成本。

但是，当前腕表任务有一个比原论文严苛得多的目标：

> **不是“尽量正确地抽取属性”，而是“只有证据足够硬时，才允许两个跨源商品自动归为同一 reference；任何不确定都必须拒识”。**

因此不建议直接复用原论文的“LLM 抽取并同时规范化”作为最终 matcher，而建议把它改造成：

```text
Reference Evidence Pipeline

原始商品记录
  -> 多路 reference 候选发现
     - 结构化 reference 字段
     - 标题/描述规则抽取
     - 图片 OCR
     - LLM 受约束抽取（只补难例）
  -> 编号角色判定
     - BRAND_REFERENCE
     - SOURCE_SKU
     - SERIAL_NUMBER
     - ACCESSORY_COMPAT_REFERENCE
     - OTHER / UNKNOWN
  -> 原文证据校验
     - 必须能回指 source_span / OCR token
  -> 品牌级 deterministic canonicalizer
     - 只执行白名单、版本化规则
  -> reference registry 校验
  -> 多证据冲突检查
  -> Precision Gate
     - AUTO_ACCEPT
     - ABSTAIN
     - REJECT
  -> 仅 AUTO_ACCEPT 进入 exact-key matching
  -> entity_key = (canonical_brand_id, canonical_reference)
```

最核心的落地原则是：

> **LLM 可以发现 evidence，但不能创造 identity。**

最终自动归并只依赖“已验证的 canonical reference 严格相等”，而不是依赖 embedding 相似度、LLM 的 yes/no 判断或图片视觉相似度。

这与我之前在 `parts-distributor-sku-classifier.md` 中提出的“编号角色识别”是互补关系：那篇解决“这个字符串到底是不是品牌 reference”，本篇解决“reference 在哪里、如何抽出、如何安全规范化、如何留下可审计证据”。

---

## 2. 原论文到底解决了什么

论文研究的是电商商品 **Product Attribute Value Extraction + Normalization**。

商品标题和描述是非结构化文本，而下游希望得到结构化属性，例如：

```text
Title:
HP 6280-59-B21 600GB SAS ...

Desired structured values:
Manufacturer = Hewlett-Packard
Part Number = 628059B21
Capacity = 600 Gigabytes
...
```

其中不同属性需要不同的 normalization：

- Name Expansion
- Generalization
- Unit Normalization
- String Wrangling

论文特别适合当前任务，是因为 WDC-PAVE 数据里直接包含 `Part Number` 属性。

官方数据描述对 Part Number 的定义非常接近腕表 reference：

```text
Attribute: Part Number
Concept: identification

Normalization:
Delete Marks / Wrangling

Instruction:
remove hyphens, periods, spaces and other non-alphanumeric characters
```

项目给出的伪代码等价于：

```python
re.sub(r'[^a-zA-Z0-9]', '', id)
```

论文示例中：

```text
6280-59-B21 -> 628059B21
```

这说明论文已经明确面对了一个我们也会遇到的问题：

> 两个数据源可能写的是同一个 identifier，但因为空格、连字符、大小写或格式不同，不能直接对 raw string 做 exact match。

不过，腕表 reference 比通用 Part Number 更危险：**不能把“删标点”当成全品牌通用规则**。某些品牌的点号、斜线、后缀、尺寸/材质编码可能有语义。因此应该借论文的“按属性配置 normalization”，进一步升级成 **按品牌 + identifier role 配置 normalization policy**。

---

## 3. WDC-PAVE 数据集为什么值得借鉴

论文构建了 WDC-PAVE 作为评测集，规模虽然远小于我们的线上数据，但它的数据组织方法很有参考价值：

- 565 个商品 offer；
- 来自 59 个网站；
- 5 个商品类别；
- 4,687 个属性值标注；
- 大量属性为空，约 45% 为 `n/a`；
- 数据经过人工校验；
- 同时保留原始值和规范化目标值。

这里最重要的不是样本量，而是两个设计：

### 3.1 `n/a` 是一等输出，而不是错误

原项目明确要求：

```text
Unknown attribute values should be marked as 'n/a'.
```

这非常契合当前 precision-first 目标。

对腕表 reference 来说，必须把：

```text
没有可靠 reference
```

设计成完全正常的结果，而不能强迫模型“猜一个最像的型号”。

生产接口应该天然允许：

```json
{
  "status": "ABSTAIN",
  "reference": null,
  "reason": "NO_VERIFIABLE_REFERENCE"
}
```

召回下降没有问题；误匹配才是核心损失。

### 3.2 同时保存 raw 与 normalized value

对当前系统必须进一步保存：

```text
raw_reference
canonical_reference
normalization_rule_id
normalization_rule_version
source_field
source_span
extractor
role
confidence
```

这样一条 canonical reference 永远能追溯到：

> “原始页面哪个位置出现了什么字符串，经过哪一版规则变成了什么 canonical value”。

这比只保存一个 `reference = 126610LN` 更适合高风险实体归并。

---

## 4. 官方代码的技术架构

官方仓库 `wbsg-uni-mannheim/wdc-pave` 并不只是论文实验结果，而是包含完整的实验流水线：

```text
analysis/          结果分析
preprocessing/     数据预处理
prompts/           各类 prompt 实验
pieutils/          检索、规范化、评测、融合等公共组件
scripts/           批量实验脚本
src/               其他抽取/规范化实验代码
```

其中最值得当前需求参考的是：

```text
prompts/08_extraction_with_normalization/
pieutils/search.py
pieutils/normalization.py
pieutils/pydantic_models.py
pieutils/evaluation.py
```

下面按运行路径拆开。

---

## 5. Step 1：先动态建立属性 Schema，而不是直接问 LLM

在：

```text
prompts/08_extraction_with_normalization/
8_few_shot_extraction_with_normalization_description_with_example_correspondences.py
```

项目先为每个商品类别构造 `ProductCategory` / `Attribute` 元模型，再把它转换成 Pydantic Model 和 JSON Schema。

核心元模型类似：

```python
class Attribute(BaseModel):
    name: str
    description: str
    examples: Optional[list[str]]

class ProductCategory(BaseModel):
    name: str
    description: str
    attributes: list[Attribute]
```

流程是：

```text
已知 category + attributes + known values
  -> LLM 生成 attribute description
  -> Pydantic 验证
  -> 动态生成 Product schema
  -> 转成 JSON Schema
  -> 注入正式抽取 prompt
```

### 对当前腕表任务的改造

不要让 Schema 只表示：

```json
{
  "reference": "126610LN"
}
```

应该把 **证据链** 也变成 Schema 的一部分：

```json
{
  "brand": {
    "raw": "Rolex",
    "canonical_brand_id": "...",
    "source": "title"
  },
  "identifier_candidates": [
    {
      "raw": "126610LN",
      "source": "title",
      "source_span": [12, 20],
      "role": "BRAND_REFERENCE",
      "belongs_to_current_product": true,
      "normalization_hint": "126610LN",
      "evidence": "literal_text"
    }
  ],
  "status": "FOUND"
}
```

这里最重要的是 `source_span`。

**任何 LLM 输出的 reference，如果无法在原始 title/description/OCR token 中定位到原文证据，直接丢弃。**

这样可以把 LLM hallucination 从“可能制造错误 identity”降级为“无效候选”。

---

## 6. Step 2：论文的 prompt 不是简单问答，而是结构化执行协议

代码中的 prompt 大致由这些块组成：

```text
System:
  你是结构化信息抽取算法
  + JSON Schema

Task instruction:
  从 product title/description 抽取属性
  + normalization guideline
  + unknown = n/a
  + do not explain

Few-shot examples:
  input -> structured output

Actual input:
  title / description
```

项目把 normalization guideline 单独注入，而不是只靠模型隐式理解。

这个设计对 reference 很重要，因为我们可以把约束写成“不能越权”的协议：

```text
1. 只能报告原文中实际出现的 identifier 字符串；
2. 不允许根据品牌/系列知识猜测缺失 reference；
3. 如果字符串只表示兼容型号、对比型号、配件适配型号，不得标记为当前商品 reference；
4. 如果存在多个相互冲突的 candidate，必须返回 AMBIGUOUS；
5. normalization_hint 只允许执行指定的字符级变换；
6. 如果无法确定，返回 n/a / ABSTAIN。
```

和普通 prompt 最大区别是：

> **LLM 的任务从“回答型号是什么”改为“提交可验证的 reference evidence candidate”。**

---

## 7. Step 3：Category-aware Semantic Few-shot Retrieval

官方 `pieutils/search.py` 使用：

```text
OpenAIEmbeddings
FAISS
SemanticSimilarityExampleSelector
```

它先为每个商品 category 建独立向量库：

```text
category -> FAISS store
```

推理时：

```text
当前商品文本
  -> 在对应 category 的向量库检索 top-k 相似训练例
  -> 作为 few-shot demonstrations
```

代码甚至支持：

```text
force_from_different_website=True
```

此时先多取候选，再过滤掉与当前商品同网站的例子，从而减少“同站模板记忆”。

### 对腕表任务不应只按 category 检索

腕表几乎都属于同一大 category，真正危险的是：

```text
同品牌
同系列
相邻 reference
相似字符结构
同一种编号角色混淆
```

因此建议改成 **Brand + Confusion Family-aware Example Retrieval**：

```text
Query features:
- canonical_brand_id
- title embedding
- identifier lexical pattern
- source_id
- suspected series
- candidate role

Retrieve:
1. 同品牌 hard positive
2. 同品牌相邻 reference hard negative
3. SOURCE_SKU / SERIAL_NUMBER / COMPAT_REFERENCE 混淆例
4. 当前来源之外的近似例
```

few-shot 不应该只是“看起来相似的正例”，而应该故意加入最容易误匹配的反例。

例如：

```text
商品 A 标题出现 126610LN             -> BRAND_REFERENCE
商品 B 标题出现 126610LV             -> BRAND_REFERENCE，且是不同 reference
商品 C 标题“表带适配 126610LN”       -> ACCESSORY_COMPAT_REFERENCE
商品 D 库存号 126610LN-202608-001    -> SOURCE_SKU
```

这会比纯 embedding top-k 更符合 precision-first。

---

## 8. Step 4：Structured Output + Pydantic 验证

原代码对模型输出做了多层防御：

```text
LLM response
  -> json.loads
  -> Pydantic validation
  -> validation error handling
  -> 必要时修复输出格式
```

如果 JSON 不是合法格式，还会尝试解析修复；如果字段类型不对，还可以走额外的 error-handling chain。

### 当前系统应保留结构校验，但减少“二次 LLM 修复语义”

对研究任务来说，让第二次 LLM 把 list 改成 string 很合理；但对“绝不能误匹配”的系统，自动修复可能悄悄改变语义。

建议分两类：

#### A. 纯语法错误

例如：

```text
JSON 尾逗号
字符串转义错误
```

可以 deterministic repair。

#### B. 语义 / 类型错误

例如：

```text
reference 返回数组但 Schema 只接受单值
role 不在 enum
source_span 无法对齐原文
```

不要让另一个 LLM“猜着修”，而是：

```text
ABSTAIN + log
```

因为在当前任务中：

> 格式失败最多损失 recall；错误修复可能制造 false positive。

---

## 9. Step 5：Normalization 是这篇论文最能直接落地的部分

原项目把 normalization operation 显式配置化。

对 `Part Number`，WDC-PAVE 的 guideline 是：

```text
Delete Marks
```

即删除非字母数字符号。

论文结果也显示，identifier 类的 **String Wrangling** 是 LLM 相对擅长的 normalization 类型。

但这里应该做一个关键工程改写：

> **如果 normalization 可以用纯函数稳定表达，就不应该让 LLM 来执行。**

论文结论本身也提出：未来可让 LLM 调用专门的 normalization function。

当前 reference 系统正适合这样做。

### 9.1 Brand-specific Canonicalizer

建议定义：

```python
CanonicalizationPolicy(
    brand_id,
    accepted_raw_patterns,
    transforms,
    accepted_canonical_patterns,
    alias_rules,
    rule_version,
)
```

伪代码：

```python
def canonicalize_reference(brand_id: str, raw: str):
    policy = policy_registry.get(brand_id)
    if policy is None:
        return Abstain("NO_BRAND_POLICY")

    s = unicode_nfkc(raw).strip()

    if not policy.accept_raw(s):
        return Abstain("RAW_PATTERN_NOT_ALLOWED")

    canon = s
    for transform in policy.transforms:
        canon = transform.apply(canon)

    if not policy.accept_canonical(canon):
        return Abstain("CANONICAL_PATTERN_NOT_ALLOWED")

    return CanonicalReference(
        raw=raw,
        canonical=canon,
        rule_version=policy.version,
    )
```

### 9.2 规则必须版本化、可逆审计

保存：

```text
raw = "AB-12 34"
canonical = "AB1234"
rule_id = "brand_x_reference_v3"
transform_trace = ["NFKC", "UPPERCASE", "REMOVE_WHITESPACE", "REMOVE_HYPHEN"]
```

如果之后发现某个字符变换不安全，可以重新跑受影响记录，而不用猜当初为什么变成这个 canonical 值。

### 9.3 绝不能全局套 `remove non-alphanumeric`

原论文的 Part Number 规则适用于其特定标注语义，但当前腕表系统必须先确认：

```text
brand + role + raw pattern
```

再执行白名单 normalization。

否则可能把本来不同的 reference 折叠为同一个 canonical string。

这类 normalization collision 比抽不到型号更危险。

因此每个 policy 上线前必须跑：

```text
collision test:
不同 gold reference -> canonicalize 后是否发生相同值
```

要求：

```text
zero known collision
```

否则该规则不得用于 AUTO_MATCH。

---

## 10. 原论文实验结果给当前需求的启示

论文使用 exact-match 评测，而不是模糊相似度，这一点与 reference 很契合。

几个有代表性的结果：

### 10.1 属性抽取

GPT-4 在加入已知 example values 与语义检索 demonstrations 后，整体 F1 可以达到约 90.5。

但 90% 对当前系统远远不够。

因为：

```text
10% 错误的 attribute extractor
```

如果直接用作百万级 entity join key，会产生无法接受的 false merge。

### 10.2 Extraction + Normalization

论文中 GPT-4 加 example values / demonstrations 后，整体 F1 大约 91 左右。

对 normalization operation 分解后，不同类型差异明显，例如 String Wrangling 明显强于一些 Unit / Generalization 场景。

### 10.3 已给定正确 extracted value 后再做 normalization

当模型无需同时承担“在哪里抽”和“怎么规范化”两个任务时，normalization 结果显著更高，String Wrangling 可接近 99% 水平。

这给当前任务一个非常直接的结论：

> **不要把 reference extraction 与 canonicalization 做成一个不可解释的一步模型。**

更安全的是：

```text
原文 -> candidate extraction -> literal evidence validation -> deterministic normalization
```

而不是：

```text
原文 -> LLM 直接输出最终 canonical reference
```

后一种方案即使平均指标很好，也很难知道某个错误值到底是“抽错了”还是“规范化错了”。

---

## 11. 原项目评测代码对 precision-first 的启示

`pieutils/evaluation.py` 把结果拆成：

```text
NN: target none, prediction none
NV: target none, prediction value
VN: target value, prediction none
VC: target value, prediction correct
VW: target value, prediction wrong
```

然后：

```text
precision = VC / (NV + VC + VW)
recall    = VC / (VN + VC + VW)
```

这个拆法非常适合直接复用到 reference extraction。

但业务权重应改成：

```text
NV = catastrophic
VW = catastrophic
VN = acceptable
```

即：

```text
hallucinate reference when none exists -> 严重错误
extract wrong reference                -> 严重错误
miss a real reference                  -> 可以接受
```

因此发布门禁不应该是：

```text
maximize F1
```

而应该是：

```text
在 gold / hard-negative set 上先满足：
- FP = 0（或统计置信上界足够低）
- precision 达到极高目标

之后再在该约束下提高 coverage / recall
```

建议同时报告：

```text
Auto-Accept Precision
Auto-Accept Coverage
Abstain Rate
Wrong-Reference Rate
Hallucinated-Reference Rate
Normalization Collision Rate
Conflict-Veto Rate
```

而不是只看一个 F1。

---

## 12. 为什么不能直接照搬原项目

原仓库很适合研究，但生产上线必须做以下改造。

### 12.1 原项目更像实验脚本，不是在线/增量服务

主要流程仍是：

```text
读取 JSONL
批量调用 LLM
评测
落 task result
```

而我们有：

```text
1M–10M 存量
持续增量
三个来源
图片/OCR
需要幂等、重试、版本升级、增量回填
```

所以要服务化和事件化。

### 12.2 原项目默认 LLM 可以同时 extraction + normalization

当前任务必须强制拆开：

```text
Extraction = evidence discovery
Normalization = deterministic policy
Matching = exact identity key
```

### 12.3 原项目只关心“值是什么”，没有 identifier role

二奢标题中很可能出现：

```text
当前商品 reference
平台 SKU
serial
配件兼容 reference
对比 reference
套装里其他产品 reference
```

如果不先识别 role，值本身“抽对了”也可能形成错误 identity。

### 12.4 原项目的语义检索按 product category 分桶

腕表应按：

```text
brand + reference family + confusion type
```

构建 hard example memory。

### 12.5 原项目没有图片证据链

当前 Spec 有图片，所以应把 OCR 作为独立 evidence channel，而不是把图片只用作相似度 embedding。

---

# 13. 建议直接落地的生产架构

## 13.1 总体架构

```text
                 ┌────────────────────────────┐
                 │ 雷小安 / 腕表之家 / 奢当家 │
                 └──────────────┬─────────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ Raw Offer Ingestion │
                     │ source + raw fields │
                     └──────────┬──────────┘
                                │
          ┌─────────────────────┼──────────────────────┐
          │                     │                      │
          ▼                     ▼                      ▼
┌──────────────────┐  ┌───────────────────┐  ┌──────────────────┐
│ Structured Field │  │ Text Candidate    │  │ Image OCR        │
│ Extractor        │  │ Extractor         │  │ Candidate        │
└────────┬─────────┘  │ regex + LLM fallback│ └────────┬─────────┘
         │            └─────────┬─────────┘           │
         └──────────────────────┼─────────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Identifier Candidate DB│
                    │ raw + span + provenance│
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Identifier Role Gate   │
                    │ reference / sku / ...  │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Literal Evidence Gate  │
                    │ span / OCR token check │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Brand Canonicalizer    │
                    │ versioned whitelist    │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Reference Registry     │
                    │ validation / aliases   │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Evidence Fusion        │
                    │ agreement / conflict   │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Precision Gate         │
                    │ ACCEPT/ABSTAIN/REJECT  │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Exact Reference Matcher│
                    │ brand_id + canonical   │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ Entity Cluster / Index │
                    └────────────────────────┘
```

---

## 14. Stage A：多路 Candidate Extraction

不要一开始就让 LLM 扫全量 1000 万商品。

按成本和可信度分层：

### A0. 结构化字段优先

如果来源已有：

```text
reference
model_no
型号
款号
```

先进入 candidate 表，但仍保留 provenance：

```json
{
  "raw": "AB-12 34",
  "source_type": "STRUCTURED_FIELD",
  "source_field": "model_no"
}
```

**结构化字段不等于天然可信**。后面仍要做 role / pattern / conflict validation。

### A1. Regex / lexical candidate generator

标题中 identifier 通常具有：

```text
字母数字混合
长度范围
品牌特定前后缀
分隔符模式
上下文词：ref / 型号 / 款号 / model / reference ...
```

规则层负责高召回地产生候选，不负责最终判断。

输出例如：

```json
[
  {
    "raw": "AB-12 34",
    "span": [18, 26],
    "extractor": "REGEX_V5"
  }
]
```

### A2. OCR candidate

图片可分：

```text
表盘
表背
保卡
吊牌
盒标
其他
```

先做图片分类，再对高价值区域 OCR。

OCR token 也进入同一个 candidate schema：

```json
{
  "raw": "AB1234",
  "source_type": "IMAGE_OCR",
  "image_id": "...",
  "bbox": [x1, y1, x2, y2],
  "ocr_confidence": 0.98
}
```

### A3. LLM fallback

只有这些情况再进 LLM：

```text
规则抽出多个候选
reference 被自然语言包围
字段名不固定
需要判断“该 identifier 是否属于当前售卖商品”
来源模板首次出现
```

这样可以把 LLM 成本限制在少量 ambiguity queue，而不是全量推理。

---

## 15. Stage B：Identifier Role Gate

这一步与我之前分析的 `parts-distributor-sku-classifier` 直接衔接。

统一 enum：

```text
BRAND_REFERENCE
SOURCE_SKU
SELLER_SKU
SERIAL_NUMBER
ACCESSORY_COMPAT_REFERENCE
BUNDLE_COMPONENT_REFERENCE
OTHER_IDENTIFIER
UNKNOWN
```

只有：

```text
role == BRAND_REFERENCE
```

才有资格进入 canonicalization。

并且对：

```text
ACCESSORY_COMPAT_REFERENCE
```

必须明确做 hard reject，避免“适配某腕表的表带/配件”被当成那只腕表本身。

---

## 16. Stage C：Literal Evidence Gate —— 防 LLM 幻觉的关键

对文本模型输出：

```python
assert raw_reference == raw_text[start:end]
```

或者做最小的 Unicode 等价校验。

对 OCR：

```text
reference 必须来自 OCR token + bbox
```

禁止：

```text
模型根据品牌、系列、图片外观，直接“知识推断”出一个未出现的型号。
```

因为当前系统的 identity 定义就是 reference，**推断的 reference 不是证据**。

如果未来想用图片模型识别型号，可以作为人工复核提示，但不应该直接进入 AUTO_MATCH key。

---

## 17. Stage D：Deterministic Canonicalization

这是对原论文 normalization 架构的最重要生产化升级。

### 17.1 Policy Registry

可以用 YAML / DB 表维护：

```yaml
brand_x:
  version: 7
  reference:
    raw_regex: "..."
    transforms:
      - nfkc
      - uppercase
      - remove_whitespace
      - remove_hyphen_if_pattern_matches
    canonical_regex: "..."
```

### 17.2 不允许自由生成 canonical reference

LLM 可以输出：

```text
raw candidate
role
source span
```

但 `canonical_reference` 必须由 deterministic service 生成。

### 17.3 normalization collision 检测

对每次 policy 变更，CI 中执行：

```python
for ref_a, ref_b in known_distinct_references:
    assert canonicalize(ref_a) != canonicalize(ref_b)
```

同时统计：

```text
raw unique count
canonical unique count
collision groups
```

任何无法解释的 many-to-one 都阻止发布。

---

## 18. Stage E：Reference Registry

建议维护一个跨来源 canonical reference registry：

```text
reference_registry
------------------
brand_id
canonical_reference
raw_aliases[]
status
first_seen_at
last_seen_at
verified_source_count
policy_version
manual_verified
```

来源可以包括：

```text
历史高可信结构化字段
品牌目录/商品库
人工黄金集
跨来源一致观测
```

### Registry 的作用不是“相似搜索”，而是 validity check

例如候选 normalized 后得到一个从未见过的 reference：

```text
不要因为字符串格式合法就立刻 AUTO_ACCEPT
```

可以：

```text
NEW_REFERENCE -> ABSTAIN / 人工抽查 / 等待第二独立来源
```

这会牺牲新型号首发阶段的 recall，但非常符合当前业务约束。

---

## 19. Stage F：Evidence Fusion

建议给每个 offer 汇总为：

```json
{
  "offer_id": "...",
  "brand_id": "...",
  "reference_evidence": [
    {
      "raw": "AB-1234",
      "canonical": "AB1234",
      "channel": "STRUCTURED_FIELD",
      "role": "BRAND_REFERENCE",
      "valid": true
    },
    {
      "raw": "AB1234",
      "canonical": "AB1234",
      "channel": "TITLE",
      "role": "BRAND_REFERENCE",
      "valid": true
    },
    {
      "raw": "AB1234",
      "canonical": "AB1234",
      "channel": "IMAGE_OCR",
      "role": "BRAND_REFERENCE",
      "valid": true
    }
  ]
}
```

### 建议优先使用“证据一致性”而不是学一个总分

比如：

```text
structured field == title == OCR
  -> very strong

structured field == title, no OCR
  -> strong

only one structured field, registry verified, no conflict
  -> possibly accept depending source reliability

text candidate != structured field
  -> ABSTAIN

OCR != text
  -> ABSTAIN

multiple BRAND_REFERENCE candidates
  -> ABSTAIN unless role/context rule resolves
```

在 precision-first 系统中，**冲突是 veto，不是扣几分**。

---

## 20. Stage G：Precision Gate

最终决策不要只输出 binary match。

统一三态：

```text
AUTO_ACCEPT
ABSTAIN
REJECT
```

伪代码：

```python
def decide_reference(offer):
    refs = offer.valid_brand_reference_evidence

    if not refs:
        return ABSTAIN("NO_REFERENCE")

    canonical_values = set(r.canonical for r in refs)

    if len(canonical_values) != 1:
        return ABSTAIN("REFERENCE_CONFLICT")

    ref = next(iter(canonical_values))

    if not registry.is_valid(offer.brand_id, ref):
        return ABSTAIN("UNVERIFIED_REFERENCE")

    if any(e.hard_conflict for e in offer.evidence):
        return ABSTAIN("EVIDENCE_CONFLICT")

    if not satisfies_auto_accept_evidence_policy(offer, ref):
        return ABSTAIN("INSUFFICIENT_INDEPENDENT_EVIDENCE")

    return AUTO_ACCEPT(ref)
```

跨商品匹配则非常简单：

```python
def same_entity(a, b):
    if a.decision != AUTO_ACCEPT or b.decision != AUTO_ACCEPT:
        return ABSTAIN

    if a.brand_id != b.brand_id:
        return REJECT

    if a.canonical_reference != b.canonical_reference:
        return REJECT

    return AUTO_ACCEPT
```

在 Spec 已经定义“同一个商品 = 同一 reference”的前提下，**最终 matcher 不需要复杂神经网络**。

复杂度应该放在：

```text
reference 是否被正确识别和规范化
```

而不是放在“两个商品像不像”。

---

## 21. Stage H：Exact-key Entity Cluster

一旦 offer 得到可靠 canonical reference，实体聚类可以直接使用：

```text
entity_key = hash(canonical_brand_id, canonical_reference)
```

表结构例如：

```sql
CREATE TABLE canonical_entity (
    entity_id              BIGINT PRIMARY KEY,
    brand_id               BIGINT NOT NULL,
    canonical_reference    TEXT NOT NULL,
    canonicalizer_version  TEXT NOT NULL,
    created_at              TIMESTAMP NOT NULL,
    updated_at              TIMESTAMP NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);
```

offer 只需要 upsert mapping：

```sql
CREATE TABLE offer_entity_link (
    offer_id        BIGINT PRIMARY KEY,
    entity_id       BIGINT,
    decision        TEXT NOT NULL,
    reason_code     TEXT,
    evidence_version TEXT,
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);
```

这样跨三个来源的 join 从复杂 pairwise matching 变成：

```text
O(N) reference resolution + indexed key lookup
```

而不是：

```text
O(N²) candidate pair comparison
```

---

# 22. 建议的数据模型

## 22.1 raw_offer

```text
raw_offer
---------
offer_id
source_id
source_item_id
raw_title
raw_description
raw_structured_json
image_urls
crawl_timestamp
content_hash
```

## 22.2 identifier_candidate

```text
identifier_candidate
--------------------
candidate_id
offer_id
raw_value
source_type          # FIELD/TITLE/DESC/OCR
source_field
char_start
char_end
image_id
bbox
extractor_name
extractor_version
created_at
```

## 22.3 identifier_classification

```text
identifier_classification
-------------------------
candidate_id
role
role_model_version
role_confidence
belongs_to_current_product
hard_reject_reason
```

## 22.4 reference_normalization

```text
reference_normalization
-----------------------
candidate_id
brand_id
raw_value
canonical_value
policy_id
policy_version
transform_trace
valid_pattern
collision_safe
```

## 22.5 offer_reference_decision

```text
offer_reference_decision
------------------------
offer_id
brand_id
canonical_reference
decision
reason_code
evidence_count
independent_channel_count
decision_policy_version
```

这些表让任何误匹配都可定位到具体：

```text
谁抽的
从哪抽的
角色怎么判的
哪条 normalization 规则改的
哪个 gate 放行的
```

---

# 23. LLM 输出 Schema 建议

建议只让 LLM 做“证据抽取 + role/context 判断”，接口如下：

```json
{
  "status": "FOUND | NOT_FOUND | AMBIGUOUS",
  "candidates": [
    {
      "raw": "AB-1234",
      "source": "TITLE | DESCRIPTION | OCR",
      "source_span": {
        "start": 18,
        "end": 25
      },
      "role": "BRAND_REFERENCE | SOURCE_SKU | SELLER_SKU | SERIAL_NUMBER | ACCESSORY_COMPAT_REFERENCE | OTHER | UNKNOWN",
      "belongs_to_current_product": true,
      "context": "型号 AB-1234",
      "confidence_bucket": "HIGH | MEDIUM | LOW"
    }
  ],
  "abstain_reason": null
}
```

不让 LLM 返回：

```text
final_entity_id
same_product=true/false
最终 canonical_reference（作为权威值）
```

即把模型权限限制在最小必要范围。

---

# 24. Prompt 建议：从 WDC-PAVE prompt 改造

原 WDC-PAVE 的任务大致是：

```text
Extract valid attribute values.
Normalize according to guidelines.
Unknown -> n/a.
Return JSON.
```

腕表版本建议改为：

```text
你是 identifier evidence extractor，不是商品型号知识问答系统。

任务：
1. 仅抽取输入文本中逐字出现的 identifier candidate；
2. 为每个 candidate 返回精确字符位置；
3. 判断它是否是当前售卖商品本身的品牌 reference；
4. 兼容/适配/比较/赠品/配件中提到的型号不得归为当前商品 reference；
5. 平台 SKU、库存号、serial、保卡号不得归为 brand reference；
6. 不得根据常识补全、猜测或生成输入中没有出现的 reference；
7. 多个候选冲突时返回 AMBIGUOUS；
8. 没有可靠候选返回 NOT_FOUND；
9. 只返回符合 JSON Schema 的结果。
```

few-shot 中至少一半应是 hard negatives，而不是全部成功抽取示例。

---

# 25. 如何使用“几百对黄金标签”才最划算

Spec 允许人工标几百对。

如果只标：

```text
商品 A == 商品 B ?
```

信息密度不够高。

建议一条标注同时包含四层：

```json
{
  "offer_id": "...",
  "brand": "...",
  "reference_span": "...",
  "identifier_role": "BRAND_REFERENCE",
  "raw_reference": "...",
  "canonical_reference": "...",
  "match_group_id": "...",
  "hard_negative_reason": null
}
```

这样几百条标注可以同时训练/校准：

```text
candidate extractor
role classifier
canonicalizer test
final match decision
```

### 25.1 标注采样不要随机

应主动选这些边界样本：

```text
同品牌相邻 reference
仅一两个字符不同
带斜线/点号/连字符
标题有多个型号
配件“适配 XX 型号”
字段与标题冲突
字段与 OCR 冲突
平台 SKU 看起来像 reference
serial 看起来像 reference
新来源模板
新品牌
历史 normalization collision
```

这些样本对 precision 的价值远高于随机 easy positive。

---

# 26. 训练 / 校准集的拆分方式

不能只随机 split。

至少构造：

```text
1. random holdout
2. source holdout
3. time holdout
4. brand holdout（用于检验新品牌泛化，不一定用于自动放行）
5. reference-family holdout
6. hard-negative challenge set
```

最重要的是 `reference-family holdout`：

> 同系列的相邻 reference 不应同时泄漏到 train / test，否则看起来很高的准确率无法证明模型能抵抗真实 false positive。

---

# 27. Release Gate：如何把“绝对不能误匹配”变成工程约束

建议每个版本发布时跑固定 challenge set。

## 27.1 Reference extraction gate

```text
Wrong Reference = 0
Hallucinated Reference = 0
```

若 gold 规模较小，不应把“观测到 0 FP”理解为数学上的零风险，而应同时计算 precision 的置信下界 / error rate 上界，并把自动放行 coverage 控制在足够保守的区域。

## 27.2 Canonicalization gate

```text
Known Distinct Reference Collision = 0
Unknown Rule Application = 0
Non-reversible Transform Without Audit Trace = 0
```

## 27.3 Final matching gate

hard-negative set 必须覆盖：

```text
same brand + adjacent reference
same series + visually similar
same title except one reference char
compatibility mention
SKU/reference confusion
```

并要求自动接受区：

```text
false positive = 0 observed
```

否则降低 auto-accept policy，宁可增加 ABSTAIN。

---

# 28. 100 万–1000 万规模下如何增量运行

## 28.1 不做全量 pairwise

每条新 offer 先独立解析出：

```text
brand_id + canonical_reference
```

然后走数据库唯一键：

```text
(brand_id, canonical_reference)
```

查找已有 entity。

这样新数据处理复杂度近似：

```text
O(1) indexed lookup / offer
```

### 28.2 事件版本建议

每条商品保留：

```text
raw_version
extractor_version
role_model_version
canonicalizer_version
decision_policy_version
```

规则升级时，只回填受影响 partition：

```text
brand X
source Y
policy version < 7
```

不需要重跑整个 1000 万。

### 28.3 幂等 key

```text
(source_id, source_item_id, content_hash)
```

相同内容不重复做 OCR / LLM。

### 28.4 LLM cache

可以按：

```text
hash(model_version + prompt_version + normalized_input)
```

缓存推理结果。

这与原项目在 title / description 分开运行时缓存重复预测的思想一致，但生产环境必须做持久化 cache。

---

# 29. 推荐部署组件

一个足够直接的实现可以是：

```text
Ingestion / Orchestration:
  Kafka / Pulsar / cloud queue

Raw + audit metadata:
  PostgreSQL

大规模分析 / 回填：
  ClickHouse 或 Spark

图片：
  Object Storage

Candidate / reference registry：
  PostgreSQL + B-tree/Hash index

Few-shot / hard-case retrieval：
  FAISS / pgvector / Milvus

LLM extractor：
  API 模型或自托管模型

OCR：
  独立 OCR service

Rule Engine：
  Python/Go deterministic service
```

但技术选型不是重点。

最重要的是把责任边界固定：

```text
LLM = evidence extraction
Rule Engine = canonicalization
Registry = validity
Precision Gate = authority
Exact Key = final entity identity
```

---

# 30. 四个典型案例

## Case 1：结构化字段和标题一致

```text
structured model_no: AB-1234
title: 品牌 X ... AB1234 ...
```

处理：

```text
field -> AB-1234
text  -> AB1234
canonicalizer -> 都为 AB1234
registry -> valid
role -> BRAND_REFERENCE
no conflict
```

结果：

```text
AUTO_ACCEPT AB1234
```

---

## Case 2：标题是配件兼容型号

```text
title: 真皮表带 适配 品牌 X AB1234
```

规则/LLM 抽到：

```text
AB1234
```

但 role：

```text
ACCESSORY_COMPAT_REFERENCE
belongs_to_current_product = false
```

结果：

```text
REJECT candidate
offer reference = ABSTAIN
```

这是“抽取值本身正确，但实体语义完全错误”的典型 false positive 来源。

---

## Case 3：字段与 OCR 冲突

```text
structured field: AB1234
case-back OCR:   AB1235
```

即使 structured field source 通常可信，也不应做加权平均后放行。

结果：

```text
ABSTAIN: REFERENCE_CONFLICT
```

进入人工复核或下一轮证据收集。

---

## Case 4：LLM 识别出未在原文出现的“常识型号”

输入：

```text
品牌 + 系列名 + 年份，无明确 reference
```

LLM 输出：

```text
AB1234
```

`source_span` 无法回指原文。

结果：

```text
Literal Evidence Gate -> drop candidate
ABSTAIN: NO_VERIFIABLE_REFERENCE
```

这条 gate 可以直接消灭最危险的一类生成式 hallucination。

---

# 31. 与 d 目录此前几个方案如何组合

目前 `d` 已分析过的几项可以拼成一个完整体系：

```text
DeepBlocker
  -> 适合必要时做候选召回，但 reference exact key 成功时甚至不需要 pair blocking

parts-distributor-sku-classifier
  -> Identifier Role Gate：防 SKU / serial / compatibility identifier 冒充 reference

Using LLMs for Extraction & Normalization（本篇）
  -> Reference Evidence Extraction + Canonicalization Framework

Confidence Classifiers
  -> 对无法硬规则决定的边界候选做 selective acceptance / abstention

TransClean
  -> 对跨来源最终匹配图做传递一致性审计，找 false-positive edge

Ameli
  -> 图片/细粒度属性可用于辅助 evidence 和人工消歧
```

组合后，推荐总流程是：

```text
extract evidence
  -> classify identifier role
  -> deterministic canonicalize
  -> registry validate
  -> exact reference identity
  -> selective precision gate
  -> transitive conflict audit
```

而不是把这些模型都堆进一个 end-to-end classifier。

---

# 32. 最小可落地版本（P0）

如果只做第一版，我建议甚至不要先训练复杂匹配模型。

## P0-1：Reference Evidence 表

完成：

```text
structured field + title regex + OCR candidate
```

统一存 provenance。

## P0-2：Top Brands Canonicalization Policy

先针对数据量最大的品牌手工建立：

```text
raw pattern
canonical pattern
safe transforms
negative tests
collision tests
```

## P0-3：Role Gate

至少先支持：

```text
BRAND_REFERENCE
SOURCE_SKU
SERIAL
COMPAT_REFERENCE
UNKNOWN
```

## P0-4：Exact Match

只允许：

```text
same brand
+ canonical reference exact equal
+ no conflict
+ registry valid
```

## P0-5：LLM 只进入 ambiguity queue

专门处理：

```text
多 candidate
自然语言上下文判断
来源新模板
```

## P0-6：黄金集

先建立 300–500 个高价值 case，硬负例优先。

最终你会很快得到一个：

```text
coverage 不一定最高
但自动匹配区 precision 极高
```

的系统骨架。

这比一开始训练一个“商品 A / B 是否相同”的黑盒模型更符合 Spec。

---

# 33. 后续 P1 / P2

## P1

```text
- Brand-aware few-shot RAG
- LLM structured evidence extractor
- OCR image-type routing
- source reliability calibration
- manual review UI
- precision confidence bound / selective threshold
```

## P2

```text
- active learning：优先标注 disagreement / boundary cases
- reference registry 自动扩张，但新 reference 先处于 probation
- TransClean 式图冲突审计
- source/time drift monitor
- canonicalizer regression CI
```

---

# 34. 一个重要反直觉结论：最终“匹配模型”应尽量简单

这个 Spec 很容易让人先想到：

```text
Siamese model
Cross Encoder
Multimodal embedding
LLM pair classifier
Graph Neural Network
```

但根据业务定义：

> **同一个商品 = 同一个 reference**

那么最终匹配逻辑应该故意简单：

```text
canonical brand 相等
AND canonical reference 相等
```

真正值得投入模型能力的地方是：

```text
1. 从脏文本/图片里找到 reference evidence；
2. 判断这个 identifier 是否真的属于当前商品；
3. 避免错误 normalization；
4. 发现多源/多模态冲突并拒识。
```

这正是这篇论文比“直接商品 pair matching”更有价值的地方：它提醒我们先把非结构化商品记录转换为高质量、可审计、规范化的 identifier 属性，再做实体解析。

---

# 35. 最终推荐方案

建议把本论文的实现思想提炼为一个独立生产组件：

```text
Reference Resolver
```

它的 API 不返回“相似商品”，只返回：

```json
{
  "offer_id": "...",
  "decision": "AUTO_ACCEPT | ABSTAIN | REJECT",
  "brand_id": "...",
  "canonical_reference": "...",
  "evidence": [...],
  "policy_versions": {
    "extractor": "...",
    "role_gate": "...",
    "canonicalizer": "...",
    "decision": "..."
  },
  "reason_code": "..."
}
```

下游 Entity Resolver 只消费：

```text
AUTO_ACCEPT
```

并执行：

```text
(brand_id, canonical_reference) exact lookup/upsert
```

一句话总结：

> **把 WDC-PAVE 的“LLM 属性抽取 + normalization”改造成“可回指原文的 reference evidence extraction + 品牌级 deterministic canonicalization”，再用严格 exact key + abstention 做实体匹配，是当前 Spec 最稳、最容易审计、也最适合百万到千万级增量数据的落地路径。**

---

## 参考资料

- 论文：Using LLMs for the Extraction and Normalization of Product Attribute Values  
  https://arxiv.org/abs/2403.02130
- 官方实现：WDC-PAVE  
  https://github.com/wbsg-uni-mannheim/wdc-pave
- 关键 prompt 实现：  
  https://github.com/wbsg-uni-mannheim/wdc-pave/blob/main/prompts/08_extraction_with_normalization/8_few_shot_extraction_with_normalization_description_with_example_correspondences.py
- Category-aware semantic example retrieval：  
  https://github.com/wbsg-uni-mannheim/wdc-pave/blob/main/pieutils/search.py
- Pydantic schema：  
  https://github.com/wbsg-uni-mannheim/wdc-pave/blob/main/pieutils/pydantic_models.py
- Evaluation：  
  https://github.com/wbsg-uni-mannheim/wdc-pave/blob/main/pieutils/evaluation.py
- WDC-PAVE attribute descriptions / Part Number normalization：  
  https://github.com/wbsg-uni-mannheim/wdc-pave/blob/main/data/descriptions/wdc/descriptions.csv
