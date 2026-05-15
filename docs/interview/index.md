---
title: 面试题集 · 大模型 / Agent 岗
description: 收集 AI Agent / 大模型相关的真实面试题，重在背后的考察意图与系统化答法，不是题库搬运。
icon: material/briefcase-search
---

# 💼 面试题集

!!! abstract "这个专栏在收集什么"
    AI Agent / 大模型岗的**真实面试题**——不是简单的题目+标答，而是**面试官真正在测什么、生产级答法该怎么搭骨架**。

    每篇按"题目原文 → 考察意图 → 我的整理 → 配套清单"四段式落笔，方便临阵翻看。

## 为什么开这个专栏

刷八股答题没意义——大模型/Agent 这个赛道的面试题流动性极快（半年前问 LangChain，现在问 MCP / Skill / Memory），**单题答法死记会过时，但每题背后那条「面试官在两条线上来回晃」的隐线不会**：

- **线 1：这东西在线上会不会出事？** —— 故障模式、降级、可观测性、幂等
- **线 2：你知不知道自己在用什么、没用上什么？** —— 框架取舍、为什么不直接用 Cursor、是不是过度设计

把题目按这两条线重新组织，下次进面试间脑子里走一遍就能成体系，不用再背"五大组件三大模块"。

## 当前收录

<div class="grid cards" markdown>

-   :material-format-list-bulleted:{ .lg .middle } **AI Agent 面试题合集（TrustZone 版）**

    ---

    **来源**：知乎 @TrustZone（海思员工，176 赞）
    **形态**：碎片合集 → 复盘整理
    **覆盖**：项目讲法 · LangGraph · 记忆 · Agent 形态 · 工具/Skill/MCP · RAG · 后端基础

    [:octicons-arrow-right-24: 进入](ai-agent-interview-tour.md)

-   :material-shield-check:{ .lg .middle } **字节二面：Agent 服务的高可用与稳健性**

    ---

    **来源**：知乎 @IT杨秀才（"大模型面试题"专栏，39 篇）
    **形态**：单题深度解析
    **覆盖**：四层防御（LLM/工具/链路/语义）+ 可观测性 + 标准答案模板

    [:octicons-arrow-right-24: 进入](agent-service-reliability.md)

</div>

## 阅读建议

如果你也在面 Agent 岗，**先读第一篇**——它是面试官提问思路的全景图，让你知道"哪类问题在测哪条线"；**再读第二篇**——它是把第一篇里"线上会不会出事"那条线给你打满的标杆答案。两篇互为骨架与血肉。

---

*持续收录中。看到值得收藏的真实面试题（不要题库式的"100 道八股"），会陆续补进来。*
