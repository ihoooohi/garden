---
title: Hermes 的全息记忆 — 当 AI 的记忆不再是关键词检索
description: Hermes Agent 的 fact_store 用的是 1995 年的向量符号架构（HRR）配相位编码，能做 embedding 数据库做不到的事 —— 多实体代数 JOIN、自动找记忆矛盾。讲清它的工作原理和为什么这选择是对的。
date: 2026-05-17
tags:
  - Hermes
  - 记忆系统
  - HRR
  - 向量符号架构
  - 检索
---

# Hermes 的全息记忆 — 当 AI 的记忆不再是关键词检索

!!! quote "源代码出处"
    `plugins/memory/holographic/holographic.py` · `retrieval.py` · `store.py`
    Plate (1995) — *Holographic Reduced Representations*
    Gayler (2004) — *Vector Symbolic Architectures answer Jackendoff's challenges*

!!! abstract "一句话定位"
    **AI 长期记忆的主流做法是 embedding + 向量检索；Hermes 用了一个更老更怪、但能做组合推理的方案 —— 全息归约表示（HRR）配相位编码。它能算"既关于 A 又关于 B 的事实"、能自动找记忆矛盾，这是 embedding 数据库结构上做不到的。**

---

## 问题：AI 的记忆为什么这么烂

ChatGPT 类产品的"记忆"基本是这套：

- 每条 memory 是一段文本
- 写入时算 embedding 存进向量库
- 检索时把当前问题也算成 embedding，找最近邻

这套在**召回单一话题**上够用。但有两件事它结构上做不到：

1. **多实体交集**：「关于 *飞书* 且 *cron* 的所有事实」—— 向量检索没有 AND 语义，只有"相似"。两次查询取交集勉强模拟，但相似度阈值难调，会漏会多。
2. **自动找矛盾**：「我之前是不是存过冲突的说法」—— 向量距离能告诉你两条 memory 像不像，但「像但不一样」（共享主语、不同断言）就是个二阶判断，embedding 给不出来。

Hermes 选了一条非主流路：**Holographic Reduced Representations (HRR)**，1995 年 Tony Plate 提出的向量符号架构（VSA）。它本来是给认知科学搞「神经网络如何表示符号结构」的，三十年后被一个 AI agent 框架捡起来做记忆。


## 它的本质：把"事实"编码成相位向量

普通向量（embedding）是"这段文本在语义空间的位置"，HRR 不是。HRR 是**用代数操作把结构编码进单一固定长度向量**——而且这些代数操作是可逆的。

Hermes 用的是相位编码版本：**每个概念是一个 1024 维向量，每一维是 [0, 2π) 上的一个角度**。三个核心操作：

| 操作 | 数学 | 语义 |
|---|---|---|
| **bind**（绑定） | 相位逐元素相加 mod 2π | 把"角色"和"值"关联起来 |
| **unbind**（解绑） | 相位逐元素相减 mod 2π | 已知角色，反推出值 |
| **bundle**（叠加） | 复指数循环平均 | 把多个概念合成一个 |

关键性质：

- `unbind(bind(A, B), A) ≈ B` —— 用 A 解开 (A 绑 B)，能近似还原 B
- `bind(A, B)` 跟 A、B 单独对比都几乎正交（点积近 0）—— 复合体看上去和原料完全不像
- 每个 atom（概念）是用 SHA-256 计数器块**确定性生成**的相位向量 —— 跨机器、跨进程、跨语言版本的 "peppi" 都映射到同一个向量

代码长这样（直接抄自 `holographic.py`）：

```python
def bind(a, b):    return (a + b) % _TWO_PI
def unbind(m, k):  return (m - k) % _TWO_PI
def bundle(*vs):   return np.angle(np.sum([np.exp(1j*v) for v in vs], axis=0)) % _TWO_PI
def similarity(a, b): return float(np.mean(np.cos(a - b)))   # phase cosine, [-1, 1]
```

四行代码，承载了整个系统的代数能力。

---

## 一条事实怎么被存下来

当我说 *"NewsBot 是飞书 app cli_xxx，每天 9:00 跑 cron"*，加上 entities = `["飞书", "cron", "NewsBot"]`，存储时干这几件事：

```python
# 来自 holographic.py 的 encode_fact —— 简化版伪码
ROLE_CONTENT = encode_atom("__hrr_role_content__")   # 内容角色
ROLE_ENTITY  = encode_atom("__hrr_role_entity__")    # 实体角色

# 1. 内容向量（bag-of-words bundle）和"内容角色"绑定
content_part = bind(encode_text("NewsBot 是飞书 app..."), ROLE_CONTENT)

# 2. 每个 entity 和"实体角色"绑定
entity_parts = [
    bind(encode_atom("飞书"),     ROLE_ENTITY),
    bind(encode_atom("cron"),     ROLE_ENTITY),
    bind(encode_atom("newsbot"),  ROLE_ENTITY),
]

# 3. 全部 bundle 成一个 1024 维相位向量
fact_vector = bundle(content_part, *entity_parts)
```

这个 `fact_vector` 是个 8KB 的 float64 数组（1024 × 8 字节），跟事实绑定存进 SQLite。它**同时包含了内容和所有实体的结构信息**，但你直接看它什么都看不出来 —— 只是一堆角度。

魔法在解码时发生。

---

## probe：用代数提问 "谁关于 X？"

我刚才演示时跟你说 probe 是"entity 倒排索引"——那是错的。我看了源码后纠正：probe 是**真正的代数解绑操作**。

`retrieval.py` 的 `probe("飞书")` 干的是：

```python
# 构造"飞书在实体角色上"的探针
probe_key = bind(encode_atom("飞书"), ROLE_ENTITY)

# 对每条 fact 的向量做 unbind
for fact in all_facts:
    residual = unbind(fact.vector, probe_key)
    # 如果"飞书"真的以 entity 角色出现在这条 fact 里，
    # residual 应该接近 ROLE_CONTENT 绑定的那部分
    sim = similarity(residual, role_content_signal)
    fact.score = (sim + 1) / 2 * fact.trust_score
```

为什么这能工作？因为 `bundle` 是有损叠加，但**对单个 binding 项做 unbind 仍能近似还原它的"伴侣"**。如果飞书确实在这条 fact 里以实体身份出现，那 fact_vector 里就有一项是 `bind(飞书, ROLE_ENTITY)`，用 probe_key 去 unbind 这一项就会得到一个接近 ROLE_ENTITY 的"自反射"信号；而其它无关项 unbind 后是噪声。**信号-噪声比 ≈ √(dim / n_items)**，1024 维大约能稳定容纳 250+ items 而不严重退化。

---

## reason：embedding 数据库做不到的事

更关键的是多实体复合查询。问 "*关于飞书 AND 关于 cron 的事实*"，`retrieval.py` 的 `reason(["飞书", "cron"])` 这么干：

```python
# 1. 给每个实体构造 probe key
probe_keys = [bind(encode_atom(e), ROLE_ENTITY) for e in entities]

# 2. 对每条 fact，分别 unbind 每个 entity，看每个 entity 是否都有结构存在
for fact in all_facts:
    sims = []
    for probe_key in probe_keys:
        residual = unbind(fact.vector, probe_key)
        sim = similarity(residual, role_content_signal)
        sims.append(sim)

    # 3. 关键一行：取最小值，AND 语义
    fact.score = min(sims) * fact.trust_score
```

`min(sims)` 这一行就是代数 AND —— 任何一个 entity 不强存在，整条事实都低分。换 `mean` 是 OR，换 `max` 是 ANY。**同一套 fact 向量，不同聚合函数=不同布尔语义**，全部不需要重新建索引。

这是 embedding 数据库结构上做不到的：你无法对两个 query embedding 做"代数交集"，只能取近似（两次查询取交集集合，但召回质量取决于阈值）。HRR 是**真的代数**。

---

## contradict：自动找记忆矛盾

最妙的是 `contradict`。这是其它任何记忆系统都没有的能力：

```python
# 简化的伪码（真实在 retrieval.py:338）
for (fact_a, fact_b) in pairs(all_facts):
    shared_entities = fact_a.entities & fact_b.entities
    if not shared_entities:
        continue                              # 不相关，跳过

    content_a = extract_content_signal(fact_a.vector)
    content_b = extract_content_signal(fact_b.vector)
    content_sim = similarity(content_a, content_b)

    if content_sim < threshold:               # 共享主语 + 内容分歧
        yield (fact_a, fact_b, divergence_score)
```

「**共享 entity 但内容向量分歧**」就是矛盾的代数定义。比如：

- Fact A：「Sonnet 4.6 是 1M context」
- Fact B：「Sonnet 4.6 是 200K context」

两条都有 entity = `["sonnet"]`，但内容向量距离很远 —— `contradict()` 直接拎出来给你一个 cleanup 候选清单。这个能力对 LLM agent 长期使用至关重要：**人会一直更新自己说过的话，记忆系统必须能识别"我之前可能存错了"**。

---

## 完整的检索栈：HRR + FTS5 + Jaccard + Trust

读完 `retrieval.py` 我才发现一个工程巧思：HRR 不是单飞，它和经典的关键词检索做了**加权融合**：

```python
# FactRetriever.search() 的核心
relevance = (
    fts_weight   * fts_rank      # SQLite FTS5 全文检索分
    + jaccard_weight * jaccard   # token 集合 Jaccard 相似度
    + hrr_weight * hrr_sim       # HRR 相位余弦相似度
)
final_score = relevance * trust_score          # 信任度加权

if half_life > 0:
    final_score *= 0.5 ** (age_days / half_life)  # 可选时间衰减
```

默认权重 `0.4 / 0.3 / 0.3`。这是一个非常 sober 的工程决策：

- **FTS5** 解决"用户精确记得关键词"的情况
- **Jaccard** 解决"关键词没全打但有 overlap"
- **HRR** 解决"语义/结构相关但词面不重合"
- **trust_score** 解决"用得多的浮上来，错的沉下去"

主流 RAG 是只用第三种（向量），Hermes 是 4 套打分串联。结果就是**键入精确名词时 FTS5 强；说话模糊时 HRR 强；时间长了 trust 自动排序**。

---

## trust score 的训练：用一次升一点

每次召回完一条事实，agent 可以打 `helpful` 或 `unhelpful`：

| 反馈 | trust 变化 |
|---|---|
| helpful | +0.05 |
| unhelpful | **-0.10**（不对称） |

不对称的设计很关键 —— **错误事实的成本比正确事实的收益高得多**。一条错事实留在记忆里会反复被召回、反复污染推理。所以惩罚加重，让烂事实快速沉底。

trust 起点 0.5、上限 1.0、下限 0。一条 fact 打 `unhelpful` 5 次就跌到 0，下次再 `min_trust=0.3` 过滤就直接看不见了 —— 软删除而不需要手动 `remove`。

---

## 容量边界：HRR 能装多少

文件里直接写了：

```python
# holographic.py:179
def snr_estimate(dim: int, n_items: int) -> float:
    """SNR = sqrt(dim / n_items)"""
    if snr < 2.0:
        logger.warning("HRR storage near capacity ...")
```

经验上 SNR < 2 检索就开始退化。dim=1024 时安全装 256 条 item，理论极限附近能塞 500 条。Hermes 的解法：

- **每条事实独立存一个 1024 维向量**（不是塞进一个全局 bank）
- **加一个 `cat:<category>` 的合成 bank** 给 probe 用，但只是优化提速，不是依赖
- 检索是 O(n) 扫描所有事实向量做 unbind + cos sim —— 1000 条事实级别完全不是瓶颈

也就是说**容量天花板是 SQLite 而不是 HRR**。HRR 在这里的角色不是"压缩到一个向量"，是"每条事实有结构化的代数表示，能被组合查询"。

---

## 和 embedding 检索的本质差异

写到这里可以做个总结对比了：

**Embedding RAG：**

- 查询是"语义相似"
- 不能做代数（AND/OR/NOT 都是后处理近似）
- 没有结构（"X 关于 Y" 和 "Y 关于 X" embedding 一样）
- 矛盾检测要外挂

**Hermes Holographic Memory：**

- 查询是"代数解绑 + 相似度"
- 原生 AND（min）、OR（mean）、ANY（max）
- 有结构（content vs entity 是不同 ROLE，能反向解出来）
- 矛盾检测是内建操作

代价：

- HRR 不能算"两段长文本语义有多像"（这是 embedding 的强项）
- 编码靠 SHA-256 atom + bag-of-words，**不会处理同义词**（"飞书" 和 "Lark" 是两个不同 atom）
- 1024 维向量算起来比 384 维 embedding 慢一点（但 numpy 矢量化下完全够用）

所以 Hermes 不是"用 HRR 替换 embedding"，是**给 agent 的记忆选了一个更适合"事实+实体+组合查询"语义的工具**。RAG 文档检索 embedding 还是更好；个人记忆/agent 状态 HRR 更对路。

---

## 对我自己用的启示

读完代码我对怎么用 fact_store 有了几个新认知：

1. **entities 是核心**，不是装饰。写 fact 时认真挑 entity 列表 —— 这些 token 决定了之后 probe/reason/contradict 能不能命中。命名要稳定（永远叫"飞书"还是有时叫"Lark"会算成两个 atom）。
2. **content 写得越具体，HRR 命中越好**。bag-of-words 编码下，"配置好了" 和 "配置了 cron 飞书 app NewsBot" 的检索质量差几倍。
3. **trust 是会衰减的工具，要主动用 fact_feedback**。不打分=trust 永远 0.5，整个排序系统就废了一半。
4. **看到老 fact 错了，update 比 remove 好**。trust 衰减 + 内容修正会自然让旧版本沉底，新版本浮上来 —— 比强删保留了"曾经记错过什么"的痕迹。

---

## 延伸阅读

- [Hermes 架构总览](hermes-architecture.md) —— fact_store 是 memory plugin 的一员
- [Hermes session_search 的工作原理](hermes-session-search.md) —— 跨会话记忆的另一条路（FTS5 + LLM 摘要）
- Plate, T. A. (1995). *Holographic Reduced Representations*. IEEE Transactions on Neural Networks. —— 论文原文
- 代码：`plugins/memory/holographic/{holographic,retrieval,store}.py` —— 总共 1782 行，读两小时能读透

*HRR 是 1995 年的论文，2026 年被一个 AI agent 框架捡起来做记忆 —— 旧的好工具不会过期，只是在等对的应用场景。*
