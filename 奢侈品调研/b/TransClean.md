# TransClean：把“传递一致性”改造成 precision-first 腕表 Reference 匹配系统的图级安全审计层

- 分析人：b
- 调研文章：TransClean: Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency
- 论文地址：https://arxiv.org/abs/2506.04006
- HTML 全文：https://arxiv.org/html/2506.04006v1
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么这次选择 TransClean

当前 Spec 的核心约束是：

1. 三个来源：雷小安、腕表之家、奢当家；
2. 数据规模 100 万～1000 万，并持续增量；
3. “同一个商品”被严格定义为 **同一 reference number / 型号**；
4. reference 有时是结构化字段，有时埋在标题，图片也可用；
5. **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
6. 可以人工标注几百对黄金样本。

此前 `b/parts-distributor-sku-classifier.md` 解决的是前置问题：

> 一个“长得像型号”的字符串，到底是品牌 reference、平台 SKU、店铺货号、序列号，还是别的编号？

`b/DeepBlocker.md` 解决的是规模问题：

> 当 reference 证据不足时，如何从百万/千万级数据中先找出少量“值得进一步验证”的候选，而不是做全量笛卡尔积？

但还有一个非常危险、而且 pairwise 模型经常忽略的问题：

> **即使每一条 pairwise 边单独看起来都很可信，只要其中混入一条 false positive，传递闭包就可能把两个完全不同的 reference 簇合并，导致错误成倍扩散。**

TransClean 正是在解决这个问题。它不是单纯继续提高某个 pair classifier 的 F1，而是把 pairwise 预测形成图之后，利用图中“隐式产生的传递匹配”反过来审计显式边。

对于当前需求，它最值得迁移的并不是论文中的 DistilBERT/CLER，而是一个安全架构原则：

> **自动匹配不能只检查“这两条记录像不像”，还必须检查“如果接受这条边，整个实体组是否仍然自洽”。**

这正好可以变成当前系统最后一道“误匹配保险丝”。

---

## 2. TransClean 的原始问题建模

论文把 pairwise entity matching 的结果表示为无向图：

```text
G_f = (V, E)

V: records
E: pairwise matcher 预测为 Match 的记录对
```

如果：

```text
A --match-- B --match-- C
```

那么即使模型从没直接比较 `A` 和 `C`，最终实体解析结果已经隐含：

```text
A == C
```

这就是论文定义的 **Transitive Match（传递匹配）**。

因此，一个 connected component 实际上代表一个真实世界实体组，最终业务语义相当于把 component 做 clique closure：同一 component 中任意两点都被认为是同一实体。

这也是为什么 false positive 边非常危险。

例如：

```text
Reference 126334 簇               Reference 126300 簇
 A1 -- A2 -- A3        B1 -- B2 -- B3
          \             /
           \--- X -----/
```

如果 `X` 是一条错误匹配边，那么系统最后不是只错一对，而是把左右两个簇整体合并。假设左右各 100 条记录，错误隐式关系可能从 1 条扩散到上千甚至上万条。

所以论文指出：只看 pairwise precision 会低估真实实体组上的风险。

---

## 3. 核心概念：Transitive Consistency

论文定义：对于匹配图 `G_f` 中同一 connected component 内所有没有显式边连接的记录对，如果 pairwise matcher 对这些“传递对”也全部预测为 Match，则 matcher 在这个 component 上满足 **Transitive Consistency**。

可以把它理解成：

```text
显式边说：A=B、B=C
系统因此隐式声称：A=C

现在重新问 matcher：A 和 C 真的是 Match 吗？

如果 matcher 回答 NoMatch：
说明当前图内部自相矛盾。
```

论文把传递对的重新预测分成两类：

- Positive Transitive Prediction：模型同意这个隐式关系；
- Negative Transitive Prediction：模型否认这个隐式关系。

Negative Transitive Prediction 越多，说明某个 component 中越可能存在 false positive 边。

论文实验中，这两个数量还能作为没有完整 ground truth 时的质量代理指标：

- positive transitive predictions 与 true positives 有相关性；
- negative transitive predictions 与 false positives 有相关性。

这点对当前项目尤其重要，因为 100 万～1000 万记录不可能人工验全量，而我们只允许几百对黄金标签。

---

## 4. TransClean 原始算法拆解

TransClean 不是一个 matcher，而是包在 matcher 后面的 **graph cleanup procedure**。它可以接任何 pairwise matching model。论文实验接了两个模型：

- DistilBERT；
- CLER（blocker + matcher 的端到端实体匹配方法）。

整体分三阶段。

### 4.1 阶段一：Initial Steps with Finetuning

目标：优先找到“最能污染整个 component”的 false positive 边。

论文先按每个 component 中 **negative transitive predictions 数量**排序，优先处理最不一致的 component。

然后重点抽两类边去标注：

#### A. Minimum Edge Cut

Minimum Edge Cut 是：删除最少数量的边就能把 component 切开的边集合。

如果一个 component 内存在大量 negative transitive predictions，负责把两个大簇“桥接”起来的最小割边非常可疑。

示意：

```text
[一大组 126334]
      |
   suspect edge  <- minimum cut 很可能命中这里
      |
[一大组 126300]
```

对 precision-first 系统而言，这比“找最低相似度边”更有意义，因为它直接衡量 **一条边对整簇污染的结构影响**。

#### B. Negative transitive pair 之间的 shortest path

假设系统图中隐含 `A == C`，但 matcher 重新比较 `A,C` 得到 NoMatch。

则从 A 到 C 的图路径中至少有一条边是错误的，否则一串真正的同实体关系不应连接一个 true negative pair。

因此论文选择这类负传递对之间 shortest path 上的边作为高价值标注对象。

这实际上是在用“矛盾的两端”倒推“哪条中间桥可能错了”。

#### C. 用新标签继续 fine-tune matcher

TransClean 会把人工标签或伪标签加入训练，再 fine-tune pairwise matcher，然后重新计算传递一致性。

论文重复这个过程若干轮；实验中 initial finetuning steps 设为 5，component size threshold `S=50`。

对于大 component，论文不会无限制地计算全部传递对，因为传递对数量按 component size 近似二次增长。

---

### 4.2 阶段二：Post Finetuning Cleanup & Checks

在前面已经清掉一批显著 false positives 后，继续做更强的收口。

论文正文描述的关键逻辑是：

1. 如果一个 component 的 negative transitive predictions 多于 positive transitive predictions，就继续移除 minimum edge cut，直到这种明显不一致的 component 消失；
2. 对尺寸仍然过大的 component 做检查；
3. 如果某个当前 transitive match 曾经被人工标为 false，则该 component 中至少还存在错误边，需要继续检查；
4. 对仍然存在 negative transitive predictions 的 component，最终检查其边，直到图达到 transitive consistency。

这里有一个实现时必须注意的细节：论文 HTML 版 Algorithm 2 的条件行与正文语义看起来存在符号方向不一致；正文明确写的是“negative 多于 positive 时继续剪枝”。落地代码不能机械照抄伪代码，必须用单元测试固定期望行为，最好再与论文 TeX/作者实现核对。

---

### 4.3 阶段三：Edge Recovery

前两阶段为了安全会主动剪边，因此一定可能误删 true positive。

TransClean 最后会尝试恢复之前移除的边：

```text
对 removed edge (u, v):
  如果 u、v 已经在同一 component -> 无需恢复
  否则模拟把它加回去
  计算加回后新产生的 transitive matches

  如果所有新 transitive matches 都被模型判断 Match:
      add back
  否则：
      有人工预算 -> 人工检查
      没预算 -> 不恢复
```

这个机制非常符合当前需求的风险偏好：

> **证据不足时保持拆开，而不是为了 recall 强行恢复。**

---

## 5. 论文实验里最值得当前项目吸收的结论

### 5.1 pairwise 指标可能严重掩盖实体组污染

论文 Synthetic Companies 上，DistilBERT 的 pairwise F1 约 81.54，但把传递关系也算进去后，Pre-TransClean precision 下降到极低水平。

这说明：

> 一个看起来“不差”的 pairwise matcher，只要有少量 bridge false positives，也可能产生极差的最终聚类结果。

这对腕表特别危险，因为同品牌同系列不同 reference：

- 文本非常相似；
- 图片非常相似；
- 价格区间重叠；
- 型号只差一两个字符。

错误边往往不是随机噪声，而是会发生在相邻 reference family 之间，恰好最容易把两个大簇接起来。

### 5.2 TransClean 更像“false-positive detector”，不是万能 matcher

论文结论也明确指出：TransClean 的效果依赖底层 pairwise matcher 本身足够有效。

因此我们不能把它理解为：

```text
先随便用模型连边 -> TransClean 会帮我修好
```

正确理解应该是：

```text
先用非常保守的高质量边建图
-> 再用 Transitive Consistency 找隐藏冲突
-> 只把图审计作为额外安全层
```

### 5.3 LLM 伪标签不是自动合并授权

论文用 DeepSeek 7B 给部分候选对做 Yes/No 伪标签，但也观察到：

- LLM 推理明显更慢；
- 伪标签会出错；
- 某些数据集上会损失大量 true positives。

对当前“绝对不能误匹配”的需求，LLM 只能用来：

- 帮助人工排序；
- 生成解释；
- 给训练候选提供弱标签；

不能直接成为“允许 merge”的最终授权者。

---

## 6. 不应该原样照搬 TransClean 的地方

当前腕表需求比论文的一般 Entity Matching 更特殊：**业务已经给出了实体等价的强定义——同一 reference。**

因此最重要的改造是：

> 不要把一个通用 pairwise semantic matcher 放在最高权力位置；最高权力应该是 `(canonical_brand, canonical_reference)` 的确定性一致性。

换句话说，当前系统没有必要回答泛化问题：

```text
“这两条商品描述是不是同一个真实商品？”
```

而应该回答更可控的问题：

```text
“这两条记录是否都能被高置信地解析为同一个品牌 reference？”
```

TransClean 在这里应该从“修复 pairwise matcher 图”改造成：

> **Reference Graph Safety Auditor（reference 图安全审计器）**。

它负责发现错误合并、证据冲突和异常大簇，而不负责创造模糊匹配边。

---

## 7. 建议直接落地的生产架构

### 7.1 总体架构

```text
雷小安 / 腕表之家 / 奢当家
          |
          v
+-------------------------+
| Raw Record Store        |
| 原始字段 + 图片 + source |
+-------------------------+
          |
          v
+-------------------------+
| Evidence Extraction     |
| 1. structured reference |
| 2. title/desc candidate |
| 3. image OCR candidate  |
+-------------------------+
          |
          v
+-------------------------+
| Identifier Role Gate    |
| BRAND_REFERENCE         |
| PLATFORM_SKU            |
| SERIAL                  |
| ACCESSORY_COMPAT_REF    |
| UNKNOWN                 |
+-------------------------+
          |
          v
+-------------------------+
| Canonicalization        |
| brand_id + canonical_ref|
+-------------------------+
          |
          v
+-------------------------+
| Evidence Safety Gate    |
| A/A+ -> 可自动进入主索引 |
| B/C  -> 只做候选/复核    |
+-------------------------+
          |
          v
+------------------------------------+
| Deterministic Reference Index      |
| key=(brand_id, canonical_reference)|
+------------------------------------+
          |
          v
+-------------------------+
| Entity / Component      |
| Registry                |
+-------------------------+
          |
          v
+-------------------------+
| Reference Graph Auditor |
| Transitive Consistency  |
| Conflict / Mincut / Path|
+-------------------------+
       |             |
       v             v
    PASS          QUARANTINE
       |             |
       v             v
  对外实体结果      人工复核队列
                     |
                     v
              Gold Labels / Feedback
                     |
                     v
       抽取器 / Role Gate / 校准器迭代
```

这个架构把三个 `b` 的调研组合起来：

- `parts-distributor-sku-classifier`：编号角色闸门；
- `DeepBlocker`：低证据记录的高召回候选层；
- `TransClean`：最终实体组的图级冲突审计层。

---

## 8. Reference 抽取与证据分级

真正决定误匹配率的不是图算法本身，而是 reference evidence 的可信度。

建议每条记录不要只保存一个 `reference` 字符串，而是保存 **证据集合**。

示例：

```json
{
  "record_id": "sdj:12345",
  "brand_id": "rolex",
  "reference_candidates": [
    {
      "raw": "126334-0001",
      "canonical": "126334",
      "source": "structured_field",
      "role": "BRAND_REFERENCE",
      "extractor": "source_rule_v3",
      "confidence": 0.9998
    },
    {
      "raw": "126334",
      "canonical": "126334",
      "source": "title",
      "role": "BRAND_REFERENCE",
      "extractor": "title_ref_v5",
      "confidence": 0.9971
    }
  ]
}
```

建议至少分成：

### A+：可以自动聚合

满足两个独立高可信通道一致，例如：

```text
结构化 reference == 标题抽取 reference
结构化 reference == 图片 OCR reference
标题 reference == OCR reference，并且品牌规则通过
```

### A：可以自动聚合，但必须经过图审计

例如：

- 平台明确的 reference 字段；
- 通过 source-specific mapping；
- 通过编号角色分类；
- 通过品牌 reference grammar；
- 没有任何强冲突证据。

### B：只允许成为候选，不允许自动 merge

例如：

- 只在标题出现一次；
- OCR 单独识别；
- 有多个 reference candidate；
- reference 可能是配件兼容型号。

### C：永远不能作为自动匹配依据

例如：

- 向量相似；
- LLM 猜测；
- 图片外观相似；
- 编辑距离很近；
- 同系列/同机芯/同尺寸。

这些只能用于召回或人工复核排序。

---

## 9. Canonical Reference 不能用“模糊归一化”

precision-first 系统里最危险的操作之一，是为了提高 recall 进行过度 normalize。

安全 normalization 可以做：

```text
trim
Unicode normalize
统一大小写
已知无语义分隔符处理
品牌明确规则下的展示后缀拆解
```

但不能默认：

```text
去掉所有非数字字符
前缀模糊匹配
编辑距离 <= 1 视为相同
把 126334 / 126300 当同系列合并
```

建议 canonicalization 版本化：

```text
canonical_ref
canonicalizer_version
brand_rule_version
raw_reference
```

这样某条规则回滚时可以完整重放。

---

## 10. 主实体键：优先使用确定性 key，而不是全局图聚类

因为 Spec 已经规定“同 reference 即同商品”，所以主路径可以非常简单：

```text
entity_key = hash(brand_id + "\x1f" + canonical_reference)
```

数据库中：

```sql
CREATE TABLE entity (
    entity_id BIGSERIAL PRIMARY KEY,
    brand_id TEXT NOT NULL,
    canonical_reference TEXT NOT NULL,
    status TEXT NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);

CREATE TABLE record_entity_link (
    record_id TEXT PRIMARY KEY,
    entity_id BIGINT NOT NULL,
    decision TEXT NOT NULL,          -- AUTO / REVIEWED / QUARANTINED
    evidence_level TEXT NOT NULL,    -- A+ / A / B / C
    rule_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);
```

100 万～1000 万级别完全没有必要先做一个巨大的通用 graph clustering 才能得到 entity。

**Reference exact index 才是主干，图是审计结构。**

这能极大降低复杂度和误合并面。

---

## 11. 把 Transitive Consistency 改造成 Reference Graph Auditor

### 11.1 节点与边

节点仍然是一条 source record。

边建议分类型：

```text
EXACT_REF_EDGE
  两条记录都通过 A/A+ Gate，brand + canonical_ref 完全一致

REVIEWED_EDGE
  人工确认同 reference

CANDIDATE_EDGE
  DeepBlocker/文本/视觉只召回，不进入主 component

CONFLICT_EDGE
  发现强冲突，仅用于审计
```

只有前两类可以影响正式 entity membership。

### 11.2 对每个 component 强制维护不变量

一个正式 component 必须满足：

```text
Invariant 1:
所有已确认 BRAND_REFERENCE 的 canonical value 必须只有一个。

Invariant 2:
所有高可信品牌证据必须相同。

Invariant 3:
不存在“当前商品是配件，但 reference 属于被适配主体”的强证据。

Invariant 4:
不存在明确平台字段指向另一个 reference。

Invariant 5:
如果图像 OCR 得到不同 reference 且 OCR 质量高，则 component 不能自动 PASS。
```

只要破坏任何一个不变量，整个 component 进入：

```text
QUARANTINED
```

而不是继续对外发布。

这一点比论文原始 TransClean 更保守，符合“不能误匹配”。

---

## 12. 腕表版 Negative Transitive Prediction 应该怎么定义

论文用 pairwise ML model 对 transitive pair 做 Match/NoMatch。

当前需求可以把 verifier 改成更强、更可解释的组合：

```python
def transitive_verifier(a, b):
    if strong_brand(a) != strong_brand(b):
        return NO_MATCH

    if strong_ref(a) and strong_ref(b):
        return MATCH if canonical_ref(a) == canonical_ref(b) else NO_MATCH

    if high_quality_ocr_ref(a) and high_quality_ocr_ref(b):
        if ocr_ref(a) != ocr_ref(b):
            return NO_MATCH

    if accessory_compatibility_conflict(a, b):
        return NO_MATCH

    return ABSTAIN
```

这里故意增加 `ABSTAIN`，不强迫模型二分类。

图审计状态可以定义为：

```text
PASS:
  无 NO_MATCH，且关键节点拥有足够强 reference evidence

QUARANTINE:
  存在任意强 NO_MATCH / invariant violation

REVIEW:
  没有硬冲突，但存在大量 ABSTAIN
```

这比论文直接要求 matcher 对所有 transitive pairs 输出 Match 更适合 zero-false-positive 业务。

---

## 13. Minimum Cut 在本项目中的正确用途：定位嫌疑边，不自动授权

假设某 component：

```text
A1 -- A2 -- A3 -- X -- B1 -- B2 -- B3
```

审计发现：

```text
A1 reference = 126334
B3 reference = 126300
```

则 A1 和 B3 构成强 `NO_MATCH` transitive pair。

系统可以：

1. 找 A1 到 B3 的 shortest path；
2. 找 component 的 minimum edge cut；
3. 计算两者交集；
4. 优先把交集边推入人工复核。

例如 `X` 同时是 shortest path 关键边和 minimum cut，那么它的 review priority 应最高。

推荐 risk score：

```text
risk(edge) =
    w1 * negative_transitive_paths_through_edge
  + w2 * is_min_cut_edge
  + w3 * component_size_impact
  + w4 * ref_conflict_strength
  + w5 * source_rule_risk
```

但需要强调：

> risk score 只用于决定“先审哪条边”，不用于自动 merge。

---

## 14. 大 component 是必须立刻报警的异常，而不是普通计算负担

论文为了复杂度给大 component 设置 `S=50` 并限制传递对计算。

腕表场景中，异常大 component 还有更强的业务含义：

如果一个 reference 突然聚合了几千/几万商品，很可能发生了：

- `N/A` / `0000` / `UNKNOWN` 被误当 reference；
- 平台公共 SKU pattern 被误分类；
- 机芯型号被当成表款 reference；
- 配件标题中的兼容型号被当当前商品型号；
- canonicalizer 把多个真实 reference 过度折叠。

因此建议：

```text
component_size > brand/reference-specific threshold
=> 直接 QUARANTINE + alarm
```

而不是继续默认它是合法实体组。

还应该维护高频 reference denylist / anomaly list：

```text
count(reference) 突然跳变
source entropy 异常
brand 内唯一 reference 数量异常下降
某 canonicalizer 版本产生 giant component
```

全部应阻断发布。

---

## 15. 百万～千万级增量实现，不需要图数据库作为第一选择

建议存储拆分：

### Raw / Feature 层

对象存储 + Parquet：

- 原始抓取结果；
- 图片 OCR 输出；
- extractor intermediate result；
- 可重放、可版本化。

### 在线/权威状态层

PostgreSQL：

- canonical brand/reference；
- entity；
- record -> entity；
- evidence；
- review decision；
- rule/model version。

核心 `(brand_id, canonical_reference)` 走 B-tree/hash 等值索引即可。

### 审计计算

不建议把 1000 万节点整体塞给 NetworkX。

真实情况下，exact-reference component 应该很小；只需要对被标记的 component 运行局部图算法。

可选：

- 小规模/原型：NetworkX；
- 高性能局部图：igraph / rustworkx；
- 全局连接维护：Disjoint Set Union（Union-Find）或数据库 entity_key；
- 审计边表：PostgreSQL / ClickHouse 做离线统计。

**不要因为用了“图方法”就默认必须上 Neo4j。** 当前的核心查询不是任意深度图遍历，而是 exact key 查找 + 异常 component 局部分析。

---

## 16. 增量记录的 precision-first 决策流程

对每条新记录：

```python
def process_record(record):
    evidence = extract_reference_evidence(record)

    role_checked = classify_identifier_roles(evidence)
    canonical = canonicalize_brand_reference(role_checked)

    level = evidence_gate(canonical)

    if level not in {"A", "A+"}:
        save_as_unresolved(record)
        generate_candidates_if_needed(record)  # 只召回
        return "REVIEW_OR_UNRESOLVED"

    entity = lookup_by_exact_key(
        canonical.brand_id,
        canonical.reference,
    )

    if entity is None:
        entity = create_entity_exact_key(...)

    tentative_link(record, entity)

    audit = audit_component(entity)

    if audit.has_hard_conflict:
        quarantine_component(entity)
        rollback_publish(record)
        enqueue_review(audit.suspect_edges)
        return "QUARANTINED"

    publish_link(record, entity)
    return "AUTO_MATCHED"
```

这里最重要的是：

> 先 exact key，再审计；没有 exact reference 的记录绝不能因为 embedding/LLM 很像就自动加入实体。

---

## 17. 人工标注几百对应该标什么

不要随机抽几百对。

应该用 TransClean 的思想把有限预算集中到 **最可能减少 false positive 的结构难例**。

优先级：

### 第一类：同品牌、同系列、相邻 reference

例如只差一位数字/字母的 hard negative。

这是最重要的防误合并数据。

### 第二类：编号角色冲突

```text
BRAND_REFERENCE vs PLATFORM_SKU
BRAND_REFERENCE vs SERIAL
BRAND_REFERENCE vs ACCESSORY_COMPAT_REF
```

### 第三类：negative transitive path 上的 bridge edge

优先标会把两个大 component 连起来的边。

### 第四类：多通道证据冲突

```text
structured ref = A
OCR ref = B
标题 ref = A / B / 多值
```

### 第五类：新增来源/品牌的分布漂移样本

每次新 source、新抓取模板、新 brand rule 上线，都要从高风险区域补标签。

这些黄金标签主要用于：

- reference extractor；
- identifier role classifier；
- evidence gate 校准；
- conflict verifier；
- 阈值评估。

不是训练一个大而全的“同商品分类器”。

---

## 18. 评估指标必须改成安全指标，而不是只看 F1

当前需求不应该以普通 F1 为主 KPI。

建议至少有：

```text
1. Auto-Match Precision
   自动发布的匹配里真正同 reference 的比例

2. False Merge Count
   错误把不同 canonical reference 合并的实体数

3. Quarantine Recall on Known Conflicts
   已知冲突能否被安全审计层拦住

4. Coverage
   全量记录中有多少能自动确定 entity

5. Review Rate
   多少记录/边进入人工队列

6. Giant Component Alarm Recall
   注入公共假 reference 时是否必然触发报警

7. Negative Transitive Count
   正式发布图中强 NO_MATCH 传递对数量
```

生产验收应该明确：

```text
False Merge Count = 0
```

优先级高于 coverage。

---

## 19. 必须专门构造的压力测试

### Case 1：一条 bridge false positive 污染两个大簇

验证 Graph Auditor 能立即 quarantine。

### Case 2：不同 reference 外观极像

保证 CLIP/图片 embedding 不能越权 merge。

### Case 3：标题写“适配 126334”但商品其实是表带

保证 compatibility reference 不进入 BRAND_REFERENCE 主索引。

### Case 4：平台 SKU 与品牌 reference 形态类似

保证 role gate 优先阻断。

### Case 5：OCR 把 `8` 看成 `B`、`0` 看成 `O`

保证单 OCR 证据最多 B 级。

### Case 6：canonicalizer 发布错误规则

一次把多个 reference 折叠成同字符串，必须触发 giant component / source entropy / conflict alarm。

### Case 7：新 source schema 漂移

structured 字段语义变化时不能静默污染线上实体。

---

## 20. 推荐的数据表结构

### `reference_evidence`

```text
id
record_id
source
field_path
raw_value
normalized_value
canonical_reference
brand_id
identifier_role
extractor_version
confidence
evidence_level
created_at
```

### `entity`

```text
entity_id
brand_id
canonical_reference
status               # ACTIVE / QUARANTINED / REVIEW
canonicalizer_version
created_at
updated_at
UNIQUE(brand_id, canonical_reference)
```

### `match_edge`

```text
left_record_id
right_record_id
edge_type            # EXACT_REF / REVIEWED / CANDIDATE
status               # ACTIVE / REMOVED / QUARANTINED
reason
model_or_rule_version
created_at
```

### `component_audit`

```text
entity_id
node_count
active_edge_count
negative_transitive_count
abstain_transitive_count
hard_conflict_count
giant_component_flag
audit_version
result                # PASS / REVIEW / QUARANTINE
audited_at
```

### `review_task`

```text
review_id
entity_id
left_record_id
right_record_id
priority
reason_codes
mincut_flag
negative_path_count
status
human_label
reviewer
created_at
```

所有决策都要保存 `rule_version/model_version/audit_version`，保证可回溯。

---

## 21. 一个更适合当前业务的“TransClean-lite”算法

论文完整 TransClean 包含反复 fine-tune、剪边、恢复边。

当前需求第一版可以先实现一个更安全、更容易审计的 `TransClean-lite`：

```python
def audit_entity_component(component):
    # 1. 硬 reference 不变量
    refs = strong_canonical_refs(component)
    if len(refs) > 1:
        return quarantine("MULTIPLE_STRONG_REFERENCES")

    brands = strong_brands(component)
    if len(brands) > 1:
        return quarantine("MULTIPLE_STRONG_BRANDS")

    # 2. giant component
    if component.size > size_threshold(component.brand, component.ref):
        return quarantine("GIANT_COMPONENT")

    # 3. 只检查高风险 transitive pairs
    pairs = select_transitive_pairs(
        component,
        prioritize=[
            "cross_source",
            "different_extractor",
            "ocr_vs_text",
            "low_confidence_bridge",
        ]
    )

    negatives = []
    abstains = []

    for a, b in pairs:
        result = transitive_verifier(a, b)
        if result == NO_MATCH:
            negatives.append((a, b))
        elif result == ABSTAIN:
            abstains.append((a, b))

    if negatives:
        suspects = rank_edges_by_min_cut_and_paths(component, negatives)
        enqueue_review(suspects)
        return quarantine("NEGATIVE_TRANSITIVE_PREDICTION")

    if too_many_abstains(abstains):
        return review("INSUFFICIENT_EVIDENCE")

    return pass_audit()
```

第一版甚至不需要自动删除边。

**发现冲突 -> 整簇 quarantine -> 人工确认后再拆。**

这种策略会牺牲一点可用性，但更符合当前极端 precision 目标，也比论文为了 F1 主动做 heuristic pruning 更安全。

---

## 22. 后续再逐步引入论文完整版能力

当 `TransClean-lite` 有稳定的审计数据后，再逐步加入：

1. minimum cut 自动定位嫌疑 edge；
2. negative transitive shortest path 统计；
3. hard case active learning；
4. 用人工标签 fine-tune reference verifier；
5. 对“误剪但高置信”的边做 edge recovery；
6. 按 source/brand 做漂移监控。

但建议始终坚持：

```text
模型可以建议删边
模型可以建议复核
模型可以帮助找 candidate

模型不能绕过 canonical reference 的硬定义直接授权 merge
```

---

## 23. 与 DeepBlocker 的边界必须非常清楚

可以把整个系统理解为三个层次：

```text
Layer 1: Candidate Recall
DeepBlocker / lexical / ANN
目标：不要漏掉值得看的候选
允许很多 false positives
绝不直接 merge

Layer 2: Reference Evidence & Exact Decision
extractor + role gate + canonicalizer + exact index
目标：给出极高 precision 的正式 entity key

Layer 3: Group Safety Audit
Transitive Consistency / conflict / mincut / path
目标：发现 Layer 2 中仍然残留的错误合并和规则事故
```

如果把三层混为一个“综合相似度分数”，会重新回到不可解释、难保证 zero false positive 的状态。

---

## 24. 这篇论文对当前 Spec 最重要的架构启示

TransClean 最有价值的一句话可以重新表述成：

> **不要只验证边，要验证接受这些边之后形成的整个实体组。**

在当前腕表场景中，应进一步强化为：

> **一个 candidate pair 即使局部证据很强，只要加入后让 component 出现 reference 冲突，它就不能进入线上正式匹配结果。**

因此我建议最终采用：

```text
reference-first
+ exact-match-first
+ abstention-first
+ graph-consistency-audit
+ quarantine-on-conflict
+ human-review-on-bridge-risk
```

而不是：

```text
text/image similarity
-> pair classifier
-> threshold
-> connected components
```

后者在百万级多源数据上只需要少量 false positive bridge edge，就可能产生大规模错误传递。

---

## 25. 最终落地建议

### 可以直接做的最小闭环

1. 建 `reference_evidence`，不要只存一个 reference 字段；
2. 引入编号角色 `BRAND_REFERENCE / SKU / SERIAL / ACCESSORY_COMPAT_REF / UNKNOWN`；
3. brand-specific canonicalizer 版本化；
4. 只有 A/A+ reference 进入 `(brand_id, canonical_reference)` exact index；
5. entity 主键由 exact key 产生，不由 embedding/LLM 聚类产生；
6. 每个正式 entity 跑 component invariant + transitive verifier；
7. 任意强冲突立即 `QUARANTINE`；
8. 用 minimum cut + negative shortest paths 给 review queue 排序；
9. 几百条人工标签集中标 hard negative 和 bridge edge；
10. DeepBlocker/图片/LLM 只服务于 unresolved records 的候选召回和复核，不授予自动 merge 权限。

### 预期得到的系统性质

- 主路径近似 O(N) reference extraction + O(1)/O(logN) exact index；
- 不需要千万级全 pair 比较；
- graph algorithm 只跑局部异常 component；
- 可以持续增量；
- 每一次自动合并都有可追踪 reference evidence；
- 一条错误边即使进入图，也有 component-level 冲突审计继续兜底；
- 发生 canonicalizer/source schema 事故时，可以通过 giant component 和 negative transitive signal 快速阻断；
- 系统天然允许 abstain，因此可以用 coverage 换 precision。

---

## 26. 结论

TransClean 不应该在这个项目中被理解为“再上一个更复杂的实体匹配算法”，而应该被提炼为 **图级安全审计思想**。

当前 Spec 已经给出最强业务先验：“同一商品 = 同一 reference”。因此最安全的技术路线是：

```text
先把 reference 识别正确
-> 严格 canonical exact match
-> 再对匹配结果做 transitive consistency 审计
-> 有任何冲突就 quarantine
-> 人工只处理最有结构影响的 bridge edge
```

如果前两篇调研分别解决“编号是不是 reference”和“千万级如何召回候选”，TransClean 正好补上最后一块：

> **防止局部 false positive 通过传递关系演化成全局错误实体组。**

这是当前“绝对不能误匹配”约束下非常值得直接落地的一层。

---

## 参考

1. Fernando De Meer Pardo, Branka Hadji Misheva, Martin Braschler, Kurt Stockinger. *TransClean: Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency*. arXiv:2506.04006, 2025. https://arxiv.org/abs/2506.04006
2. 论文 HTML：https://arxiv.org/html/2506.04006v1
3. 当前仓库已有互补调研：`奢侈品调研/b/parts-distributor-sku-classifier.md`
4. 当前仓库已有互补调研：`奢侈品调研/b/DeepBlocker.md`
