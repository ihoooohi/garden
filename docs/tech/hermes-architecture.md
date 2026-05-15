---
title: Hermes Agent 架构图解 —— 我的 AI 助手是怎么"住"在我电脑里的
description: 拆解 ~/.hermes/ 目录的 6 大类组件、记忆机制、技能系统、Cookie 抓取栈，全程用 mermaid 流程图。
date: 2026-05-14
tags:
  - hermes-agent
  - 架构
  - AI Agent
  - 技术笔记
---

# 🏛️ Hermes Agent 架构图解

<div style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); padding: 2rem 1.5rem; border-radius: 12px; color: white; margin: 1.5rem 0;">
<p style="font-size: 1.1rem; line-height: 1.7; margin: 0 0 1rem 0; font-weight: 500;">
Hermes 不是一个聊天框，<br>
它是一个有<b>持久记忆</b>、跨平台触手、可定时跑、能<b>自学技能</b>的"AI 个人秘书"——<br>
而它整个人就活在 <code style="background: rgba(255,255,255,0.2); padding: 2px 8px; border-radius: 4px; color: #fff;">~/.hermes/</code> 这一个目录里。
</p>
<div style="display: flex; flex-wrap: wrap; gap: 1.5rem; margin-top: 1.5rem; padding-top: 1.5rem; border-top: 1px solid rgba(255,255,255,0.2);">
  <div><div style="font-size: 1.6rem; font-weight: 700;">6</div><div style="font-size: 0.85rem; opacity: 0.9;">核心层</div></div>
  <div><div style="font-size: 1.6rem; font-weight: 700;">~50</div><div style="font-size: 0.85rem; opacity: 0.9;">Skills</div></div>
  <div><div style="font-size: 1.6rem; font-weight: 700;">2K</div><div style="font-size: 0.85rem; opacity: 0.9;">字符 MEMORY</div></div>
  <div><div style="font-size: 1.6rem; font-weight: 700;">15</div><div style="font-size: 0.85rem; opacity: 0.9;">子目录</div></div>
  <div><div style="font-size: 1.6rem; font-weight: 700;">∞</div><div style="font-size: 0.85rem; opacity: 0.9;">可搜索 sessions</div></div>
</div>
</div>

!!! abstract "为什么写这篇"
    用了 Hermes 一段时间后，我意识到它和 ChatGPT 那种"打开网页问几句"的工具是完全不同的物种——它有**持久记忆**、有**自动化定时任务**、能**调真实工具**（看我电脑、跑命令、调浏览器）、还会**自己积累技能**。我想搞清楚它每次回答我之前都"看到了什么"、"做了什么"，于是把整个 `~/.hermes/` 目录拆开分析了一遍，得到这篇图解。

---

## 📦 整体鸟瞰图

整个 Hermes 在我机器上就活在 `~/.hermes/` 这一个目录里，按职责拆成 **6 大类**——先用一张组件墙看清边界：

<div class="grid cards" markdown>

-   :material-cog-outline:{ .lg .middle } __配置层__

    ---

    `config.yaml` 决定模型/Provider/工具开关；`.env` 装秘钥。**两文件强分离**——一份能 git，一份永远私密。

    [:octicons-arrow-right-24: 跳到配置层](#1-hermes)

-   :material-brain:{ .lg .middle } __记忆层__

    ---

    `memories/MEMORY.md` 每 turn 全量注入 system prompt；`sessions/*.jsonl` 是可搜索的"长期外存"。

    [:octicons-arrow-right-24: 跳到记忆层](#2)

-   :material-tools:{ .lg .middle } __能力层__

    ---

    Skills = 程序性记忆。**用的时候按需加载，用完发现坑就自己 patch SKILL.md**——会越用越懂你。

    [:octicons-arrow-right-24: 跳到能力层](#3-skills)

-   :material-key-variant:{ .lg .middle } __凭据层__

    ---

    `cookies/{x,xiaohongshu}.json` chmod 600。绕开 OAuth 直接用浏览器登录态调真实平台。

    [:octicons-arrow-right-24: 跳到凭据层](#4-cookie)

-   :material-clock-outline:{ .lg .middle } __自动化层__

    ---

    `cron/jobs.json` + Gateway。每天 21:00 EDT 自动抓 Karpathy 推文整理日报推到飞书。

    [:octicons-arrow-right-24: 跳到自动化层](#5-cron)

-   :material-folder-multiple-outline:{ .lg .middle } __运行时层__

    ---

    `logs/` + `cache/`。所有 LLM 调用、tool call、错误堆栈都落盘——可 grep、可 git diff。

</div>

下面这张图把上面 6 块的协作关系和数据流画出来：

```mermaid
graph TB
    subgraph Hermes["🌳 ~/.hermes/"]
        A["📋 配置层<br/>config.yaml + .env"]
        B["🧠 记忆层<br/>memories/ + sessions/"]
        C["🛠️ 能力层<br/>skills/ + hermes-agent/"]
        D["🔐 凭据层<br/>cookies/ + auth.json"]
        E["⏰ 自动化<br/>cron/jobs.json"]
        F["📊 运行时<br/>logs/ + cache/"]
    end

    User[👤 我] -->|发消息| Gateway[🌐 Gateway<br/>Feishu / Telegram / CLI]
    Gateway --> Brain[🤖 Hermes Agent 主进程]
    Brain --> A
    Brain --> B
    Brain --> C
    Brain --> D
    Brain --> E
    Brain --> F

    Brain -->|调用| LLM[(🧠 LLM<br/>Claude Opus 4.7<br/>via Bedrock)]
    LLM -->|tool_calls| Brain

    classDef cfg fill:#e3f2fd,stroke:#1976d2
    classDef mem fill:#f3e5f5,stroke:#7b1fa2
    classDef skill fill:#e8f5e9,stroke:#388e3c
    classDef cred fill:#fff3e0,stroke:#f57c00
    classDef cron fill:#fce4ec,stroke:#c2185b
    classDef rt fill:#f5f5f5,stroke:#616161

    class A cfg
    class B mem
    class C skill
    class D cred
    class E cron
    class F rt
```

每次你给 Hermes 发一条消息，它都要"经过"上面这些层——**配置告诉它你是谁、用哪个 LLM；记忆告诉它你的习惯和过去聊过什么；能力告诉它能做什么；凭据让它能调你授权过的服务**。

---

## 🧬 一次对话的生命周期

先看一次对话从你发消息到看到回复，**Hermes 内部到底发生了什么**：

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 我
    participant G as 🌐 Gateway<br/>(Feishu)
    participant H as 🤖 Hermes Agent
    participant M as 🧠 Memory<br/>(MEMORY.md + USER.md)
    participant S as 📚 Skills<br/>(~/.hermes/skills/)
    participant L as 🔮 LLM<br/>(Claude Opus 4.7)
    participant T as 🛠️ Tools<br/>(terminal/file/web/...)

    U->>G: "帮我搜 karpathy 的推文"
    G->>H: 转发消息 + 用户身份
    H->>M: 加载 MEMORY.md + USER.md
    H->>S: 扫描 SKILL.md frontmatter
    H->>L: System prompt = 配置 + 记忆 + skill 索引<br/>+ 历史对话 + 当前消息
    L-->>H: tool_call: skill_view("x-cookie-scraping")
    H->>S: 加载完整 skill 内容
    H->>L: 把 skill 内容塞回 context
    L-->>H: tool_call: terminal("xsearch karpathy")
    H->>T: 真实跑 shell 命令
    T-->>H: 推文 JSON
    H->>L: 把 JSON 塞回 context
    L-->>H: 最终回复（Markdown）
    H->>G: 回复
    G->>U: 渲染并发送
```

!!! tip "关键洞察"
    **每次对话开始时，整个 `MEMORY.md` 和 `USER.md` 文件都被原文塞进 LLM 的 system prompt**——这就是为什么记忆容量很小（2K-2.2K 字符上限）但效果显著：它不是"我去查记忆"，而是 LLM 每次都"已经知道了"。所以记忆里写的东西要**精炼、稳定、跨会话有用**。

---

## 1️⃣ 配置层 —— Hermes 的"身份证"

```mermaid
graph LR
    subgraph CFG["📋 配置层"]
        Config[config.yaml<br/>11.6 KB]
        Env[.env<br/>24 KB]
        Auth[auth.json<br/>OAuth tokens]
        Channel[channel_directory.json<br/>已连平台清单]
        Backup[config.yaml.bak.*<br/>每次改自动备份]
    end

    Config --> |"模型路由"| ModelChoice{当前模型}
    ModelChoice --> Bedrock[Claude Opus 4.7<br/>via AWS Bedrock]
    ModelChoice --> Other[OR OpenRouter / DeepSeek<br/>/ Gemini / 本地模型]

    Env --> |"读取"| ApiKeys[各种 API Key<br/>OPENAI / ANTHROPIC<br/>BEDROCK / EXA / ...]

    Channel --> Feishu[Feishu]
    Channel --> Discord[Discord]
    Channel --> Telegram[Telegram]

    classDef file fill:#e3f2fd,stroke:#1976d2
    class Config,Env,Auth,Channel,Backup file
```

| 文件 | 是什么 | 改它要重启吗 |
|---|---|---|
| `config.yaml` | 主配置（模型、provider、工具、技能、压缩阈值） | 大部分要 `/reset` 或重启 |
| `.env` | 所有 API Key 和 secret | `/reload` 或重启 |
| `auth.json` | OAuth token（飞书 access_token 等） | 自动刷新 |
| `channel_directory.json` | 已连接的聊天平台清单 | gateway 重启 |
| `config.yaml.bak.*` | 每次改 config 都自动备份一份 | 用于回滚 |

!!! tip "配置和秘密分两个文件"
    是为了你能随便 `git push` config.yaml 同步设置，但 `.env` 永远在 `.gitignore` 里。

---

## 2️⃣ 记忆层 —— 跨会话的"共同体记忆"

这是 Hermes 和"普通 LLM 聊天"最大的区别。每次对话**都不是从零开始的**：

```mermaid
graph TB
    subgraph MEM["🧠 记忆层"]
        direction TB
        MMD["MEMORY.md<br/>2.2K 上限<br/>当前 66%"]
        UMD["USER.md<br/>1.4K 上限<br/>当前 70%"]
        SDIR["sessions/<br/>每次对话的完整 transcript<br/>共 45 个 JSON"]
    end

    NewMsg["📩 你发新消息"] --> Inject

    subgraph Inject["每个 turn 都做的事"]
        direction TB
        Step1["1. 读 MEMORY.md 全文"] --> Step2["2. 读 USER.md 全文"]
        Step2 --> Step3["3. 拼成 system prompt"]
        Step3 --> Step4["4. 发给 LLM"]
    end

    MMD -->|原文注入| Inject
    UMD -->|原文注入| Inject

    SDIR -->|按需检索| SS["session_search 工具<br/>SQLite FTS5 全文检索"]
    SS -->|找到相关历史| Inject

    classDef mem fill:#f3e5f5,stroke:#7b1fa2
    classDef inj fill:#fff8e1,stroke:#f57c00
    class MMD,UMD,SDIR mem
    class Step1,Step2,Step3,Step4,SS inj
```

### 两类记忆，分工明确

| 文件 | 内容 | 例子 |
|---|---|---|
| **MEMORY.md** | Hermes 自己的工作笔记 | 「主机是无图形界面 Linux」「web 抓取栈用 Exa + Jina」「cookie 都在 ~/.hermes/cookies/」 |
| **USER.md** | 关于用户（我）的画像 | 「飞书移动端不渲染 Markdown 表格」「中文交流」「讨厌方案对比让你选」 |

### 容量设计的取舍

```mermaid
graph LR
    A["容量小<br/>2K 字符"] --> B["每次都全量注入<br/>不需要检索"]
    B --> C["LLM 一上来就<br/>'已经知道了'"]
    C --> D["响应快<br/>不需要 RAG"]

    A -.- A2["⚠️ 必须精炼<br/>过时信息要清理"]
    A2 --> A3["Hermes 主动<br/>合并/淘汰旧条目"]
```

记忆容量小是**有意为之**——这样可以保证整段塞进每次的 system prompt，LLM 不需要"决定要不要查记忆"，记忆**永远是激活的**。代价是必须保持精简，所以我（Hermes）会主动建议合并冗余条目。

### Sessions 是"长期外存"

```mermaid
graph LR
    Sessions["sessions/<br/>📂 45 个对话 JSON<br/>共 7.8 MB"]
    FTS["SQLite FTS5<br/>全文索引"]
    Tool["session_search<br/>工具"]

    Sessions --> FTS
    FTS --> Tool

    Q["「上次我问过 X 没？」"] -->|关键词| Tool
    Tool -->|找到匹配| Result["相关历史片段<br/>注入当前 context"]
```

记忆是"现在还要用的事实"，sessions 是"以前聊过的所有内容"。当我说「上次咱们怎么解决 Y 的来着？」，Hermes 会调用 `session_search` 工具，从 SQLite 里全文检索，把找到的片段注入到当前对话的 context 里。

---

## 3️⃣ 能力层 —— Skills 系统（自学的"程序性记忆"）

如果说记忆是"陈述性知识"（事实），**Skills 就是"程序性知识"（怎么做某事）**。这是 Hermes 最有意思的部分：

```mermaid
graph TB
    subgraph SKILLS["📚 ~/.hermes/skills/ (20+ 分类)"]
        direction LR
        S1["autonomous-ai-agents/"]
        S2["creative/"]
        S3["github/"]
        S4["research/"]
        S5["social-media/"]
        S6["mlops/"]
        S7["..."]
    end

    subgraph S5_DETAIL["social-media/x-cookie-scraping/"]
        SkillMD["SKILL.md<br/>YAML frontmatter +<br/>步骤 + 踩坑 + 验证"]
        Refs["references/<br/>(可选) 详细文档"]
        Templates["templates/<br/>(可选) 代码模板"]
        Scripts["scripts/<br/>(可选) 辅助脚本"]
    end

    S5 --> S5_DETAIL

    Trigger["📩 用户问<br/>'帮我搜 X 上的 Y'"] --> Match{匹配 skill<br/>frontmatter}
    Match -->|hit| Load[skill_view]
    Load -->|加载完整 SKILL.md| Context["注入当前 context"]
    Context --> Execute["按步骤执行<br/>terminal/file/...等工具"]

    Execute --> Maintain{发现<br/>SKILL.md<br/>过时?}
    Maintain -->|是| Patch["skill_manage<br/>action='patch'<br/>立刻修正"]
    Maintain -->|否| Done["✅ 完成任务"]
    Patch --> Done

    classDef skill fill:#e8f5e9,stroke:#388e3c
    classDef trigger fill:#fff8e1,stroke:#f57c00
    class S1,S2,S3,S4,S5,S6,S7 skill
    class Trigger,Match,Load,Context,Execute trigger
```

### Skills 的关键创新：**自我维护**

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant H as Hermes
    participant Skill as SKILL.md
    participant Tool as 工具

    U->>H: 跑某个任务
    H->>Skill: 加载相关 skill
    Skill-->>H: 步骤 1 → 2 → 3
    H->>Tool: 执行步骤 1
    Tool-->>H: ✅ OK
    H->>Tool: 执行步骤 2
    Tool-->>H: ❌ 报错 X
    Note over H: 🤔 SKILL.md 没说会有这个错
    H->>H: 自己研究修复
    H->>Tool: 修复后重跑
    Tool-->>H: ✅ OK
    H->>Skill: skill_manage(patch)<br/>给步骤 2 加上「踩坑：X」
    H->>U: 任务完成
    Note over Skill: 下次别人遇到同样问题，<br/>SKILL.md 已经知道了
```

这就是为什么 Hermes 是 "self-improving"——**它每次踩坑都会修订 skill，让下次的自己（或别人）少走弯路**。

### Skills vs Memory 的边界

| | Memory | Skills |
|---|---|---|
| 内容类型 | 事实陈述 | 操作步骤 |
| 容量 | 受限（2K） | 不限 |
| 加载 | **每次自动注入** | **按需加载**（match 时） |
| 例子 | "用户用中文" | "怎么用 cookie 抓 X 的推文" |

---

## 4️⃣ 凭据层 —— Cookie 抓取栈（最近搭的）

最近我让 Hermes 能去 X、小红书帮我搜内容，整套机制就活在凭据层：

```mermaid
graph TB
    subgraph Mac["💻 我的 Mac"]
        Browser["🌐 Chrome 浏览器<br/>已登录目标网站"]
        Editor["🍪 Cookie-Editor<br/>扩展导出 JSON"]
    end

    subgraph Server["🖥️ Linux Server (~/.hermes/)"]
        subgraph Cookies["🔐 cookies/ (chmod 700)"]
            XJ["x.json<br/>(auth_token + ct0)"]
            XHS["xiaohongshu.json<br/>(web_session)"]
        end

        subgraph Tools["~/tools/ (venv 工具树)"]
            XBOT["x-bot/<br/>playwright + chromium"]
            XHSBOT["xhs-bot/<br/>xhs SDK + stealth.min.js"]
        end

        subgraph Bin["~/.local/bin/"]
            XSEARCH["xsearch CLI"]
        end

        subgraph Skill["skills/social-media/"]
            S1["x-cookie-scraping/SKILL.md"]
            S2["xhs cookie skill"]
        end
    end

    Browser --> Editor
    Editor -.->|"复制 JSON"| Server

    Cookies -.->|"权限 600<br/>永不进 LLM context"| XBOT
    Cookies -.-> XHSBOT

    XBOT --> XSEARCH

    User["📩 '搜 karpathy 的推文'"] --> Hermes[🤖 Hermes]
    Hermes --> Skill
    Skill -->|"按步骤执行"| XSEARCH
    XSEARCH -->|"playwright 启动"| Chromium["🦊 Chromium headless"]
    Chromium -->|"注入 cookie"| XCom["x.com 真页面"]
    XCom -->|"渲染好的 DOM"| Chromium
    Chromium -->|"抓 article[data-testid='tweet']"| Result["📰 推文 JSON"]
    Result --> Hermes
    Hermes --> User2["💬 回复用户"]

    classDef cred fill:#fff3e0,stroke:#f57c00
    classDef tool fill:#e8f5e9,stroke:#388e3c
    classDef cli fill:#e1f5fe,stroke:#0288d1
    class XJ,XHS,Cookies cred
    class XBOT,XHSBOT,Chromium tool
    class XSEARCH cli
```

### 为什么用 Cookie 而不是 API？

!!! tip "取舍"
    - **官方 API**（X API v2 / xurl）：稳，但要付费 + 注册 OAuth。
    - **Cookie + Playwright**：免费，立等可用，但 cookie 1-3 个月过期一次。

我的判断：如果只是日常给我抓信息，cookie 这条路**性价比更高**——失效时重导一次就行。重要的是 cookie 文件**永远不进 LLM 的 context**（Hermes 的 skill 里明确禁止读 cookie 内容回流），LLM 只知道"调 xsearch CLI"，cookie 在 OS 层面被它的工具脚本读取。

### 几个踩过的坑（已写进 skill）

```mermaid
graph LR
A1["❌ 直接调 X GraphQL API"] -->|"query_id 老变"| B1["404"]
A2["❌ twikit 库"] -->|"X 改了首页 JS"| B2["KEY_BYTE indices 错误"]
A3["❌ Nitter 镜像"] -->|"99% 已死"| B3["403 / 空 body"]
A4["✅ Playwright + 真 chromium"] -->|"和真浏览器一样"| B4["稳定可用"]

classDef bad fill:#ffebee,stroke:#c62828
classDef good fill:#e8f5e9,stroke:#388e3c
class A1,A2,A3,B1,B2,B3 bad
class A4,B4 good
```

---

## 5️⃣ 自动化层 —— Cron 定时任务

我每天 9:00（北京）会收到一份 AI/Agent 行业简报，整个流程是这样的：

```mermaid
sequenceDiagram
autonumber
participant Cron as ⏰ Cron 调度器
participant Job as jobs.json<br/>(任务定义)
participant Hermes as 🤖 Hermes Agent<br/>(独立 session)
participant Skill as 📚 Skills
participant CLI as 🛠️ xsearch / web_search
participant FS as 💬 Feishu

Note over Cron: 每分钟检查一次
Cron->>Job: 9:00 北京 = 21:00 EDT 触发
Job-->>Cron: 加载 prompt + skills 列表
Cron->>Hermes: 启动新 session<br/>注入 prompt
Hermes->>Skill: 加载 x-cookie-scraping
Hermes->>CLI: xsearch --user karpathy
CLI-->>Hermes: Karpathy 24h 内推文
Hermes->>CLI: web_extract OpenAI/Anthropic 博客
CLI-->>Hermes: 官方动态
Hermes->>CLI: web_search 论文/行业新闻
CLI-->>Hermes: 各类条目
Hermes->>Hermes: 综合总结成 Markdown
Hermes->>FS: 推送到飞书
FS->>User: 我看到日报
```

### Cron 任务的关键设计

```mermaid
graph TB
subgraph CronSystem["⏰ ~/.hermes/cron/"]
    JobsJSON["jobs.json<br/>所有任务定义"]
    Output["output/<br/>每次运行的输出归档"]
    Lock[".tick.lock<br/>防止重复触发"]
end

JobsJSON --> JobDef

subgraph JobDef["单个任务字段"]
    Schedule["schedule<br/>'0 9 * * *' / 'every 2h'"]
    Prompt["prompt<br/>给新 session 的指令"]
    Skills["skills<br/>预加载哪些"]
    Toolsets["enabled_toolsets<br/>限定 web/browser/terminal"]
    Deliver["deliver<br/>送到哪个平台"]
end

JobsJSON -.->|"3 分钟硬超时"| Limit["每次跑不超过 3 分钟"]
JobsJSON -.->|"skip_memory=True"| Pure["不污染主对话记忆"]

classDef cron fill:#fce4ec,stroke:#c2185b
class JobsJSON,Output,Lock,Schedule,Prompt,Skills,Toolsets,Deliver cron
```

!!! tip "Cron 任务的隔离性"
    Cron 任务和你的主聊天是**完全隔离的**——它跑在独立 session 里，看不到你正在聊的东西，也不会把它的工作记忆写到主 MEMORY.md（因为 `skip_memory=True`）。这避免了"日报跑了一次就把一堆抓取细节塞进我的长期记忆"的污染。

---

## 6️⃣ 把所有层串起来 —— 一张总览图

```mermaid
graph TB
subgraph Touch["🌐 接入层"]
    F[Feishu]
    T[Telegram]
    D[Discord]
    C[CLI]
end

subgraph Brain["🤖 Hermes Agent (核心)"]
    Loop[run_conversation 主循环]
    Compress[上下文压缩]
    Resolver[模型路由]
end

subgraph LLMs["🔮 LLM 后端"]
    Bedrock[Bedrock<br/>Claude Opus 4.7]
    Aux[Auxiliary<br/>压缩用小模型]
end

subgraph Mem2["🧠 记忆 (~/.hermes/)"]
    MMD2[MEMORY.md]
    UMD2[USER.md]
    Sess[sessions/]
end

subgraph Cap["🛠️ 能力 (~/.hermes/)"]
    Skills2[skills/<br/>20+ 分类]
    Tools[tools/<br/>terminal/file/web/<br/>browser/vision/...]
end

subgraph Cred["🔐 凭据"]
    ENV[.env<br/>API Keys]
    Cook[cookies/<br/>web_session]
    OAuth[auth.json<br/>OAuth tokens]
end

subgraph Auto["⏰ 自动化"]
    CronJ[cron/jobs.json]
    Hooks[hooks/]
    Webhooks[webhook 订阅]
end

F --> Brain
T --> Brain
D --> Brain
C --> Brain

Brain --> Mem2
Brain --> Cap
Brain --> Cred
Brain --> Auto

Loop --> Compress
Compress --> Aux
Loop --> Resolver
Resolver --> Bedrock

Tools --> Cred
CronJ -.->|定时启动新 session| Brain

classDef touch fill:#e1f5fe,stroke:#0288d1
classDef brain fill:#f3e5f5,stroke:#7b1fa2
classDef llm fill:#fff3e0,stroke:#f57c00
classDef mem fill:#f3e5f5,stroke:#7b1fa2
classDef cap fill:#e8f5e9,stroke:#388e3c
classDef cred fill:#fff3e0,stroke:#f57c00
classDef auto fill:#fce4ec,stroke:#c2185b

class F,T,D,C touch
class Loop,Compress,Resolver brain
class Bedrock,Aux llm
class MMD2,UMD2,Sess mem
class Skills2,Tools cap
class ENV,Cook,OAuth cred
class CronJ,Hooks,Webhooks auto
```

---

## 🤔 我的几点判断

!!! abstract "TL;DR"
1. **Hermes 不是"一个聊天机器人"，是一个有持久状态的"AI OS"**——文件系统就是它的世界。
2. **记忆 + 技能 + 工具的三层分离**很巧妙：事实级注入、操作级按需加载、能力级随时调用。
3. **本地化是核心优势**——所有数据在我自己电脑上，不依赖云端 RAG 服务，隐私和速度都有保障。

### 和 ChatGPT / Claude 网页版的本质区别

<div class="grid" markdown>

<div markdown>
:material-rocket-launch: **Hermes Agent**
{ .center }

- ✅ **MEMORY.md 跨会话**：上次说过的事下次还记得
- ✅ **真 shell + 真浏览器**：能 git push、能登小红书
- ✅ **Skills 系统**：程序性知识可沉淀
- ✅ **cron + gateway**：定时主动推消息
- ✅ **多平台触手**：飞书 / TG / Discord / 邮件
- ✅ **数据归属本地**：`~/.hermes/` 全在我电脑上
- ✅ **切换 LLM 一行配置**：Opus / GPT-5 / 自部署都行
</div>

<div markdown>
:material-web: **ChatGPT / Claude Web**
{ .center }

- ❌ 每次新对话从零开始
- 🟡 部分（Code Interpreter，沙箱有限制）
- ❌ 没有自定义"程序性知识"沉淀
- ❌ 永远是"我问它答"，不会主动找我
- ❌ 只在网页 / App 内
- 🟡 云端，受厂商 ToS 约束
- ❌ 锁定厂商
</div>

</div>

### 这套架构最让我觉得"做对了"的地方

1. **文件系统作为状态存储**。一切都是 Markdown / JSON / SQLite，可以 grep、可以 git diff、可以 backup——比"云端不可见的记忆服务"可控得多。
2. **Skills 的自我维护机制**。这不是简单的 RAG，而是"边用边修教程"——每次踩坑都让下一次更顺利。
3. **配置和秘密强分离**。`config.yaml` 可同步、`.env` 永远私密——这种"工程性的克制"在 AI 工具里很少见。
4. **Cron + Skills + Cookies 组合的可扩展性**。我搭 X 抓取栈花了 1 个晚上，下一个平台（知乎/微博/B站）只需要复用同样的模式。

### 不太满意的地方

1. **单 turn 输出 token 上限太低**——遇到长内容（比如这篇文章）容易被截断，需要主动拆段写。已经把 `model.max_tokens` 调到了 32K。
2. **MEMORY.md 容量受限**——2K 字符跨多个领域用很容易撑爆，必须经常合并精炼。某种程度上这逼迫记忆质量，但也确实是约束。
3. **Cookie 失效需要手动重导**——目前没有自动检测 + 提醒的机制（虽然脚本里加了 expiry warning）。

---

## 🔗 延伸阅读

- [Hermes Agent 官网](https://hermes-agent.nousresearch.com/) —— 文档和 quick start
- [Hermes GitHub 仓库](https://github.com/NousResearch/hermes-agent) —— 源代码
- [Anthropic: Building Agents with Claude](https://www.anthropic.com/engineering/building-effective-agents) —— Agent 设计的官方思考
- [Cognition: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) —— 关于 agent 架构取舍的另一种声音
- [Karpathy: LLM as OS](https://x.com/karpathy/status/1707437820045062561) —— "把 LLM 当成新一代操作系统"的视角，与 Hermes 的"AI OS"定位一脉相承

---

*这是 Garden 里的第 2 篇正式文章，也是把"我用的工具"系统性拆解的第一次尝试。如果对你也有帮助，欢迎转发；如果你有更好的 Agent 架构思路，欢迎来 GitHub 讨论。*




