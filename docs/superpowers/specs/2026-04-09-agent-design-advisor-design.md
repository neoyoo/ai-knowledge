# Agent Design Advisor — Skill 设计规范

**文件路径**: `docs/superpowers/specs/2026-04-09-agent-design-advisor-design.md`
**状态**: 草稿
**日期**: 2026-04-09
**关联知识库**: `wiki/*.md`（12 个 L1 概念页）

---

## 1. 概述

### 1.1 这个 Skill 是什么

`agent-design-advisor` 是一个**场景驱动的 agent 架构决策引擎**。

它不是知识百科——不会把"Claude Code 这样设计，DeerFlow 那样设计，你自己选"扔给用户。它是一个**有立场的架构顾问**：通过引导式对话弄清楚用户在做什么，然后基于知识库中多项目的实战证据，给出"你这个场景应该用 X，理由是……"的具体建议。

类比：不是把医学书递给你，而是先问你症状，再告诉你该吃什么药、为什么、有什么副作用要注意。

### 1.2 定位与触发时机

**触发条件（满足任意一条）：**
- 用户说"我想做一个 agent"、"我在设计 agent 架构"、"帮我规划这个 agent 怎么做"
- 用户问具体 agent 架构问题但还没有清晰的系统需求（还不知道用户的完整约束）
- 用户在用 `create-agent-skill` 之前希望先想清楚架构方向

**不触发的情况：**
- 用户已经有明确的架构决策，只需要实现细节 → 直接查 L2 页面
- 用户在做模型选择、微调、量化 → 超出本 skill 范围
- 用户问的是通用软件设计（无 agent 成分）→ 用 brainstorming skill

### 1.3 输出物

本 skill 产出一份 **架构建议文档**（markdown 格式），内容包括：
- 用户需求画像（Phase 1 整理）
- 文字版架构概览
- 逐维度设计决策（每个维度：推荐方案 + 理由 + 备选方案 + 参考实现 + 避坑提醒）
- 概念间依赖关系
- 下一步行动建议（指向 L2 页面）

所有建议**必须可溯源**到知识库中的具体证据（哪个 L1 页面的哪个场景、哪个项目的哪个实现）。

### 1.4 与现有 Skill 的区别

| Skill | 定位 |
|-------|------|
| `superpowers:brainstorming` | 通用创意/功能设计探索，广度优先 |
| `agent-design-advisor` | Agent 架构专项决策，深度优先，有明确输出格式 |
| `create-agent-skill` | 具体 skill/组件实现，代码级 |

关系：brainstorming → **agent-design-advisor** → create-agent-skill

---

## 2. Skill 文件结构

### 2.1 文件位置

```
.claude/skills/agent-design-advisor.md
```

### 2.2 文件格式

Claude Code skill 文件由两部分组成：

```markdown
---
name: agent-design-advisor
description: >
  场景驱动的 agent 架构设计顾问。通过引导式问答了解你的具体场景，
  再基于知识库中多项目的实战证据，给出有立场、可溯源的架构建议。
  用于：设计新 agent、选型困惑、架构方向决策。
triggers:
  - "我想做一个 agent"
  - "agent 架构"
  - "帮我设计一个 agent"
  - "选什么框架"
  - "agent 怎么设计"
kb_root: wiki/
l1_concepts:
  - prompt-system
  - query-loop
  - runtime-state
  - context-management
  - tool-system
  - memory-system
  - multi-agent
  - hooks
  - mcp-skills
  - channel-remote
  - session-recovery
  - evaluation-observability
---

# Agent Design Advisor

## 你是谁

你是一个 agent 架构顾问，不是知识库搜索工具。
你的工作是：先听懂用户的具体情况，再给出有立场的架构建议。

（后续是完整的行为指令，见下文 Phase 1-3 详细设计）
```

### 2.3 Skill 文件是 LLM 的行为说明书

本 skill 文件**本身就是给 LLM 看的指令手册**，不是配置文件。文件内容决定 LLM 的行为方式：提问风格、信息整理方式、KB 查询策略、输出格式。

设计原则：
- 指令足够具体，LLM 不需要"自由发挥"问什么
- 但不是脚本——允许 LLM 根据上下文调整措辞和顺序
- KB 查询路径明确指定，避免 LLM 乱读文件

---

## 3. Phase 1：引导式需求发现（核心设计）

> 这是本 skill 的灵魂。用户往往无法一次性说清自己的需求——他们知道自己想做什么，但不一定知道这些需求对应哪些架构决策。本阶段的任务是**通过对话把隐性需求变成显性约束**。

### 3.1 基本原则

1. **每次只问一个问题**（向 brainstorming skill 学习），不做问题列表轰炸
2. **优先提供选项**（多选题降低认知负担），但保留"其他"出口
3. **问题分层**：先问大方向（使命/模式/规模），再问细节（记忆/工具/可靠性）
4. **每个答案在心里映射到概念重要度**（对用户不可见），但不说出来
5. **积累到足够信息**后，进入 Phase 2，不要无休止地问

### 3.2 需求发现问题流

问题分四个 Layer。每个 Layer 内的问题按顺序问，Layer 3 有条件跳转。

---

#### Layer 1 — 使命与模式（必问，3 个问题）

**Q1：Agent 的核心使命**

> "先从大方向开始——用一句话描述这个 agent 要做什么？不用太精准，描述核心用途就好。"

这是**唯一的开放题**，目的是建立初始语境。LLM 需要从回答中提取：
- 领域（编程/写作/分析/自动化/客服……）
- 是否涉及外部系统交互
- 是否有长时间运行的任务

---

**Q2：主要任务类型**

> "这个 agent 主要处理哪类任务？（选最接近的，可以选多个）
> 
> A. 编程辅助（写代码、调试、代码审查）
> B. 研究与分析（搜索、整理、生成报告）
> C. 客服与问答（回答用户问题、意图识别）
> D. 工作流自动化（按步骤执行固定流程）
> E. 创意内容（写作、头脑风暴、设计辅助）
> F. 数据处理（文件转换、批量处理、ETL）
> G. 其他（请描述）"

映射逻辑（静默）：
- A/B/F → tool-system 重要性 ↑，context-management 重要性 ↑
- C → channel-remote 重要性 ↑，evaluation-observability 重要性 ↑
- D → query-loop（图编排方向）↑，hooks 重要性 ↑
- E → prompt-system 重要性 ↑
- B/F → multi-agent 潜在需求

---

**Q3：交互模式**

> "这个 agent 怎么跟用户交互？
> 
> A. 用户驱动（用户发指令，agent 执行后等待下一条）
> B. Agent 自主（用户给目标，agent 自行规划和执行，偶尔汇报）
> C. 协作对话（用户和 agent 你来我往，共同推进）
> D. 无人值守（定时触发 / 事件触发，用户不在场）"

映射逻辑（静默）：
- A → query-loop 简单循环够用，runtime-state 要求低
- B → query-loop 复杂度 ↑，session-recovery 重要性 ↑，runtime-state ↑
- C → context-management 重要性 ↑，memory-system 潜在需求
- D → session-recovery 重要性 ↑，evaluation-observability ↑，hooks ↑，channel-remote ↑

---

#### Layer 2 — 规模与环境（必问，2 个问题）

**Q4：用户规模**

> "这个 agent 服务多少人？
> 
> A. 个人使用（就你自己）
> B. 小团队（< 20 人，内部工具）
> C. 中型组织（数十到数百人）
> D. 面向公众（不特定多数用户）"

映射逻辑（静默）：
- A/B → evaluation-observability 可以简单，session-recovery 不是优先级
- C/D → evaluation-observability ↑，session-recovery ↑，channel-remote ↑
- D → security/注入防护考虑，hooks 中等

---

**Q5：部署环境**

> "Agent 跑在哪里、用户怎么访问？
> 
> A. 命令行工具（CLI，用户在终端用）
> B. Web 应用（嵌入网页或独立 Web UI）
> C. 聊天平台（Slack/Discord/Telegram/企业微信等）
> D. API 服务（供其他程序调用）
> E. 多平台（同时支持多个上面的渠道）
> F. 嵌入式（在另一个工具/产品内作为模块存在）"

映射逻辑（静默）：
- A → channel-remote 不重要，可跳过
- B/C/E → channel-remote 重要性 ↑↑
- C → channel-remote（gateway 设计），session-recovery ↑
- D → runtime-state ↑，evaluation-observability ↑
- E → channel-remote 很重要，hooks 重要性 ↑

---

#### Layer 3 — 能力深潜（条件触发，按需问 2-5 个）

Layer 3 的问题**根据 Layer 1+2 的答案选择性提问**，不是每个都问。

---

**Q6：对话时长与连续性**（除 Q3=D 外必问）

> "你的 agent 对话是什么形态？
> 
> A. 单次问答（每次对话独立，无需记住上次）
> B. 短会话（一次任务 5-10 轮以内，完成即止）
> C. 长会话（一次任务可能持续 20+ 轮）
> D. 持续运行（跨多次启动，需要记住历史）"

条件跳转：
- 选 C 或 D → Q7（记忆需求）**必须问**
- 选 C → context-management 重要性 ↑↑
- 选 D → memory-system 重要性 ↑↑，session-recovery 重要性 ↑

---

**Q7：记忆需求**（仅当 Q6=C 或 D 时触发）

> "对记忆能力有什么期望？
> 
> A. 不需要记忆（每次对话从零开始）
> B. 会话内记住（同一次启动内有记忆，关掉就忘）
> C. 跨会话记住（下次启动后还记得用户偏好/项目上下文）
> D. 学习与自我更新（agent 能通过使用逐渐积累知识、修正错误理解）"

映射逻辑（静默）：
- A → memory-system 跳过
- B → memory-system 低，context-management 关注
- C → memory-system 重要性 ↑↑（需要选存储方案）
- D → memory-system 重要性 ↑↑↑（需要考虑 Auto-Dream 或类似更新机制）

---

**Q8：工具与外部系统**（仅当任务类型≠E，或 Q3=B/D 时触发）

> "Agent 需要跟哪些外部系统打交道？（可以多选）
> 
> A. 文件读写（本地文件系统、代码仓库）
> B. 代码执行（运行脚本、调用命令行）
> C. 网络/API（搜索、REST API、数据库查询）
> D. 动态工具加载（运行时按需引入新工具，而非固定工具集）
> E. MCP 生态（使用 Model Context Protocol 扩展工具）
> F. 不需要外部工具（纯 LLM 对话生成）"

映射逻辑（静默）：
- A/B/C → tool-system 重要性 ↑↑
- D → tool-system（动态发现方案）重要性 ↑↑↑
- E → mcp-skills 重要性 ↑↑↑
- F → tool-system 可简单/跳过

---

**Q9：多 Agent 协作**（仅当任务类型=B/D/F，或 Q3=B，或任务复杂度高时触发）

> "任务有没有需要多个 agent 配合的情况？
> 
> A. 不需要（单个 agent 搞定所有事）
> B. 简单委托（偶尔把子任务交给另一个 agent，但不需要协调）
> C. 主从分工（一个主 agent 拆任务，多个子 agent 并行执行）
> D. 动态编排（任务结构复杂，需要运行时动态决定用哪些 agent、怎么组合）"

映射逻辑（静默）：
- A → multi-agent 跳过或低优先级
- B/C → multi-agent 重要性 ↑↑
- D → multi-agent 重要性 ↑↑↑，runtime-state ↑

---

**Q10：可靠性与容错**（仅当 Q4=C/D，或 Q3=B/D，或目标非原型时触发）

> "对稳定性的期望是什么级别？
> 
> A. 原型验证（跑通就行，出错可以忍，重跑一次就好）
> B. 内部工具（团队日常用，偶尔出问题可以接受）
> C. 生产服务（有 SLA，出问题会有用户投诉或业务损失）
> D. 关键任务（长时间任务，中途中断代价极高）"

映射逻辑（静默）：
- A → session-recovery 低，evaluation-observability 低
- B → 两者中等
- C → evaluation-observability 重要性 ↑↑，hooks 中等
- D → session-recovery 重要性 ↑↑↑，evaluation-observability ↑↑

---

#### Layer 4 — 约束汇总（问 1 个问题）

**Q11：最重要的约束**（必问，作为收尾）

> "最后一个问题——什么是你最不能妥协的约束？（可以多选，但建议最多选 3 个）
> 
> A. 成本敏感（尽量少烧 token，优先低成本方案）
> B. 低延迟（用户等待时间必须短，不能用慢方案）
> C. 高可靠（稳定性优先，宁愿功能少也要不出错）
> D. 安全合规（数据不出境、权限最小化、有审计）
> E. 灵活扩展（现在是 MVP，但架构要能平滑演进）
> F. 快速迭代（先跑通，架构可以后面再优化）"

映射逻辑（静默）：
- A → 倾向轻量方案，反对重型框架
- B → 倾向静态方案，反对动态发现/复杂编排
- C → session-recovery ↑，evaluation-observability ↑，反对激进设计
- D → hooks（权限控制）↑，mcp-skills（安全边界）↑
- E → 架构建议偏向可扩展模式
- F → 所有维度倾向简单可用方案，不引入不必要的复杂度

---

### 3.3 概念重要度映射矩阵

以下矩阵是 Phase 1 的**分析骨干**——每个答案组合如何影响 12 个概念的重要度。

重要度级别：`critical` / `high` / `medium` / `low` / `skip`

> 说明：表格记录的是该答案**单独触发时的影响方向**。最终重要度需要叠加所有答案。

#### 任务类型（Q2）对概念重要度的影响

| 概念 | A-编程辅助 | B-研究分析 | C-客服问答 | D-工作流自动化 | E-创意内容 | F-数据处理 |
|------|-----------|-----------|-----------|--------------|-----------|-----------|
| prompt-system | high | medium | high | medium | critical | medium |
| query-loop | high | high | medium | critical | medium | high |
| runtime-state | medium | medium | low | high | low | medium |
| context-management | high | high | medium | medium | medium | high |
| tool-system | critical | high | medium | high | low | critical |
| memory-system | medium | high | high | medium | low | medium |
| multi-agent | medium | high | low | high | low | high |
| hooks | medium | low | medium | critical | low | medium |
| mcp-skills | high | medium | low | medium | low | medium |
| channel-remote | low | low | critical | medium | low | low |
| session-recovery | medium | medium | medium | high | low | high |
| evaluation-observability | medium | medium | high | high | low | high |

#### 交互模式（Q3）对概念重要度的影响

| 概念 | A-用户驱动 | B-Agent自主 | C-协作对话 | D-无人值守 |
|------|-----------|------------|-----------|-----------|
| prompt-system | medium | high | high | medium |
| query-loop | medium | critical | medium | high |
| runtime-state | low | high | medium | high |
| context-management | medium | high | critical | medium |
| tool-system | medium | high | medium | high |
| memory-system | low | medium | high | high |
| multi-agent | low | high | low | medium |
| hooks | low | medium | low | critical |
| mcp-skills | medium | medium | low | medium |
| channel-remote | low | low | medium | high |
| session-recovery | low | high | medium | critical |
| evaluation-observability | low | high | medium | critical |

#### 部署环境（Q5）对概念重要度的影响

| 概念 | A-CLI | B-Web应用 | C-聊天平台 | D-API服务 | E-多平台 |
|------|-------|----------|----------|---------|---------|
| prompt-system | medium | medium | high | medium | medium |
| query-loop | high | medium | medium | high | high |
| runtime-state | high | medium | medium | high | high |
| context-management | high | medium | medium | high | high |
| tool-system | high | medium | low | medium | medium |
| memory-system | high | medium | high | medium | high |
| multi-agent | medium | medium | low | medium | medium |
| hooks | medium | medium | high | medium | high |
| mcp-skills | high | medium | low | medium | medium |
| channel-remote | skip | medium | critical | medium | critical |
| session-recovery | high | medium | high | high | high |
| evaluation-observability | low | medium | high | high | high |

#### 对话时长（Q6）对概念重要度的影响

| 概念 | A-单次问答 | B-短会话 | C-长会话 | D-持续运行 |
|------|-----------|---------|---------|-----------|
| context-management | skip | low | critical | critical |
| memory-system | skip | low | high | critical |
| session-recovery | skip | low | medium | critical |
| prompt-system | low | medium | high | high |
| query-loop | medium | medium | high | high |

#### 可靠性要求（Q10）对概念重要度的影响

| 概念 | A-原型 | B-内部工具 | C-生产服务 | D-关键任务 |
|------|-------|----------|----------|-----------|
| session-recovery | skip | low | high | critical |
| evaluation-observability | skip | medium | high | critical |
| hooks | skip | low | medium | high |
| runtime-state | low | medium | high | high |
| query-loop（错误处理） | low | medium | high | critical |

#### 约束优先级（Q11）对方案选择的影响

| 约束 | 倾向方案 | 反对方案 |
|------|---------|---------|
| A-成本敏感 | 文件记忆、静态工具、简单循环、词法检索 | 向量数据库、多模型合成、重型框架 |
| B-低延迟 | 静态注册、短 context、简单循环 | 动态发现、实时压缩、多 agent 编排 |
| C-高可靠 | 对话快照恢复、三级权限控制、熔断机制 | 无 session-recovery、无观测、激进动态方案 |
| D-安全合规 | 工具权限分级、注入防护、审计日志 | 全开放权限、无 hooks 介入点 |
| E-灵活扩展 | MCP 生态、动态工具发现、双层记忆、模块化 hooks | 硬编码工具集、单文件逻辑堆叠 |
| F-快速迭代 | 最小可用方案、跳过非核心维度 | 过度工程化（过早引入多 agent、向量数据库）|

---

### 3.4 退出条件与需求画像

**何时停止提问，进入 Phase 2？**

满足以下两个条件时退出 Layer 3-4：
1. 所有标记为 `critical` 的概念都有了明确的约束输入
2. Layer 4（Q11）已完成

**需求画像（Requirement Profile）**

Phase 1 结束时，LLM 在内部形成一份需求画像，格式如下（不直接展示给用户，但在生成建议时引用）：

```
需求画像：
- 核心使命：[Q1 回答摘要]
- 任务类型：[Q2 选项]
- 交互模式：[Q3 选项]
- 用户规模：[Q4 选项]
- 部署环境：[Q5 选项]
- 对话时长：[Q6 选项]
- 记忆需求：[Q7 选项，如有]
- 工具需求：[Q8 选项，如有]
- 多 Agent：[Q9 选项，如有]
- 可靠性：[Q10 选项，如有]
- 核心约束：[Q11 选项]

概念优先级（叠加后）：
- critical: [概念列表]
- high: [概念列表]
- medium: [概念列表]
- skip: [概念列表]
```

**补充追问规则：**

如果某个 `critical` 概念的重要度来自多个矛盾信号（例如：交互模式=自主 → session-recovery 很高，但可靠性=原型 → session-recovery 低），需要在进入 Phase 2 前主动问一个澄清问题：

> "我注意到你的 agent 需要自主执行，但你说的是原型阶段——如果 agent 中途挂掉，你更希望 A) 直接重跑（简单但要重新开始），还是 B) 从断点恢复（开发成本高但长任务更安全）？"

---

## 4. Phase 2：KB 匹配协议

### 4.1 概述

Phase 1 结束后，LLM 进入内部匹配阶段。这个阶段**对用户透明**（用户看不到 LLM 在读哪些文件），但结果会体现在 Phase 3 的输出里。

### 4.2 匹配流程

**步骤 1：确定需要分析的概念集合**

从需求画像中取出所有重要度 `>= high` 的概念，加上所有 `critical` 概念。这是需要在输出中给出建议的维度集。

重要度为 `medium` 的概念：在输出中简要提及，不做详细分析。
重要度为 `skip` 的概念：完全不出现在输出中（避免噪音）。

---

**步骤 2：按概念读取 L1 页面，提取场景决策指南**

对每个需要分析的概念，读取对应的 L1 页面：

```
wiki/{concept-id}.md
```

重点提取以下两个 section：
- **`### 场景决策指南`** — 直接匹配用户场景
- **`### 常见陷阱`** — 生成"避坑提醒"

不要全量读取 L1 页面——关注场景决策指南和常见陷阱即可。方案对比表用于确认选项合法性。

---

**步骤 3：场景匹配逻辑**

对每个概念，遍历 L1 页面中的"场景决策指南"，找到与需求画像匹配的条件：

```
场景决策指南格式：
"如果你在做 [条件描述] → [推荐方案]"

匹配规则：
- 用户需求画像中的"任务类型 + 交互模式 + 约束"与条件描述对应 → 采用该推荐方案
- 多个条件同时匹配 → 优先匹配最具体的（包含更多约束词的）
- 找不到精确匹配 → 找最近似的，并在输出中标注"外推"（不是 KB 直接建议）
```

---

**步骤 4：歧义处理**

当出现以下情况时，暂停匹配，向用户提一个澄清问题：

1. **两个 `critical` 概念的建议相互冲突**：例如选了轻量 query-loop（简单循环），但 multi-agent 需求很高（简单循环不适合多 agent 编排）
2. **场景描述在多个方案之间平均分布**：KB 的场景指南里对这个需求组合没有明确倾向

澄清问题格式：
> "在 [概念A] 和 [概念B] 之间有个取舍要确认一下：[简单描述冲突]。你更看重 A) [选项A] 还是 B) [选项B]？"

---

**步骤 5：置信度评估**

每条建议都有一个内部置信度（输出时标注）：

| 置信度 | 条件 | 输出标注 |
|--------|------|---------|
| 高 | KB 场景决策指南有直接匹配 | 无标注（默认） |
| 中 | 接近匹配但有条件差异 | "（参考 [场景名]，需根据你的情况调整）" |
| 低 | 从 KB 外推，没有直接证据 | "（KB 中无直接案例，基于架构原则推断）" |

---

**步骤 6：L2 页面使用策略**

L2 页面（`wiki/_impl/{concept}--{source}.md`）**默认不读取**。

仅在以下情况读取：
- 用户在 Phase 3 输出后，针对某个建议追问"这个怎么实现"、"有没有代码例子"
- 需要给出特定项目的具体实现路径（如"参考 Hermes Agent 的 `delegate_task` 实现"）

L2 页面给出的是实现细节，不是决策依据。决策依据只来自 L1。

---

### 4.3 概念间依赖关系识别

某些维度的决策会影响其他维度，需要在输出中显式说明。

已知的重要依赖关系（来自 `schema/ontology.yaml` relations + KB 内容）：

| 如果选了... | 会影响... | 影响方式 |
|------------|----------|---------|
| memory-system = 向量数据库 | context-management | 可以不做大量 context 压缩，RAG 召回替代 |
| memory-system = 冻结快照（Hermes 模式） | context-management | 每 session prompt 前缀不变，压缩需求降低，但 prompt caching 收益极高 |
| multi-agent = 主从派发 | query-loop | 不能用最简单的单循环，需要支持子 agent 启动和结果汇总 |
| multi-agent = 主从派发 | runtime-state | 父子 agent 的状态/预算需要隔离管理 |
| channel-remote = 多平台 | hooks | 需要 hook 介入来处理不同渠道的消息格式差异 |
| channel-remote = 多平台 | session-recovery | 不同渠道的 session 生命周期不同，需要可配置的 reset policy |
| tool-system = MCP 动态发现 | query-loop | DeerFlow 的 `tool_search` 模式需要额外一轮工具调用，影响延迟设计 |
| tool-system = MCP 动态发现 | mcp-skills | 两者深度耦合，建议一起考虑 |
| evaluation-observability = 生产级 | hooks | 生产级观测通常需要 hook 点注入 trace/metrics |

---

## 5. Phase 3：架构建议输出

### 5.1 过渡提示

在进入输出前，给用户一个简短确认：

> "好，我已经了解了你的需求。我将基于知识库中 Claude Code、OpenHarness、DeerFlow、Hermes Agent 四个项目的实战分析，给出针对你这个场景的架构建议。"

（如果有信心不足的维度，在这里预告：）
> "其中 [概念名] 这个维度 KB 中没有完全对应你场景的案例，我会基于架构原则给出推断，会特别标注。"

---

### 5.2 输出文档格式

```markdown
# [Agent 名称（来自 Q1）] 架构建议

> 基于 AI Engineering Knowledge Base 分析
> 生成日期：YYYY-MM-DD

---

## 需求画像

| 维度 | 你的情况 |
|------|---------|
| 核心使命 | [Q1 摘要] |
| 任务类型 | [Q2 选项] |
| 交互模式 | [Q3] |
| 用户规模 | [Q4] |
| 部署环境 | [Q5] |
| 对话时长 | [Q6] |
| 记忆需求 | [Q7，如有] |
| 工具使用 | [Q8，如有] |
| 多 Agent | [Q9，如有] |
| 可靠性 | [Q10，如有] |
| 核心约束 | [Q11] |

---

## 架构概览

```
[Agent 名称]
├── Prompt System: [方案名]
├── Query Loop: [方案名]
├── Context Management: [方案名]
├── Tool System: [方案名]
├── Memory System: [方案名，或 N/A]
├── Multi-Agent: [方案名，或 N/A]
├── Hooks: [方案名，或 N/A]
├── MCP & Skills: [方案名，或 N/A]
├── Channel & Remote: [方案名，或 N/A]
├── Session Recovery: [方案名，或 N/A]
└── Eval & Observability: [方案名，或 N/A]
```

（用 ASCII 图或文字块，不用 Mermaid）

---

## 逐维度设计决策

### [概念名 1]

- **推荐方案**: [方案名]
- **理由**: [1-3 句话，引用 L1 场景决策指南的具体条件]
  - KB 依据：`wiki/[concept].md` → 场景决策指南 → "[匹配条件原文摘要]"
- **备选方案**: [方案名] — 当 [什么情况改变] 时考虑切换
- **参考实现**: [[wiki/_impl/[concept]--[source].md]] — [一句话说明这个项目的实现有什么可参考的]
- **避坑提醒**: [从 L1 常见陷阱中提取最相关的 1-2 条]

---

### [概念名 2]

（格式同上，repeat for each critical/high 概念）

---

## 概念间依赖

> 以下决策之间存在耦合关系，改变其中一个可能需要调整另一个：

- **[概念A] → [概念B]**: [你选的 A 方案] 意味着 [B 维度需要注意什么]
- **[概念C] ↔ [概念D]**: [描述相互影响]

（仅列出与你的推荐方案相关的依赖，不列所有可能依赖）

---

## 下一步

根据你的架构方向，建议按以下顺序深读：

1. **优先阅读**（核心路径，建议立刻看）：
   - [[wiki/_impl/[concept]--[source].md]] — [说明：这个实现最接近你的场景]
   - [[wiki/_impl/[concept2]--[source2].md]] — [说明]

2. **按需阅读**（遇到具体问题时再看）：
   - [[wiki/_impl/[concept3]--[source3].md]] — [说明：具体实现细节]

3. **可以跳过**（你的场景暂时不需要）：
   - [概念名] — [一句话说明为什么在你的场景里不重要]
```

---

### 5.3 输出质量要求

每条"理由"必须满足：
- **可溯源**：明确引用哪个 L1 页面的哪个场景条件
- **有立场**：不是"A 方案有优点，B 方案也有优点，你自己决定"，而是"你这个场景应该用 A，因为……"
- **有边界**：说清楚什么情况下这个建议不再适用（备选方案的切换条件）

每条"避坑提醒"必须：
- 来自 L1 页面的 `### 常见陷阱` section，不能自创
- 与用户选择的推荐方案相关（不相关的陷阱不列）

---

## 6. Non-Goals / 范围边界

以下内容**不在本 skill 范围内**，遇到时应主动告知用户并指引合适资源：

### 6.1 不生成代码

本 skill 产出的是**架构建议文档**，不是代码实现。

如果用户在输出后要求"帮我写这个 agent"，应该说：
> "架构建议完成了。接下来的实现部分，建议使用 `create-agent-skill` 来落地——那个 skill 专门负责把架构决策转化为实际代码。"

### 6.2 不替代 brainstorming

`superpowers:brainstorming` 处理的是**通用产品/功能探索**——用户连做什么都还没想清楚。

本 skill 假设用户已经知道要做什么（Q1 能回答），只是不确定怎么架构。如果 Q1 的回答也很模糊，应先建议：
> "你的 agent 目标还比较模糊，建议先用 brainstorming skill 梳理一下要解决的问题，再来做架构设计。"

### 6.3 不覆盖模型选型

本 skill 专注于 **agent 框架架构**，不涉及：
- 选哪个 LLM（GPT-4o / Claude / Llama）
- 是否需要微调、LoRA、量化
- embedding 模型选型

遇到这类问题，超出知识库范围，诚实说明。

### 6.4 不读取外部资源

本 skill 的全部证据来自当前项目的 wiki 目录。不访问互联网、不引用 arxiv、不查阅外部文档。如果有 KB 无法覆盖的场景，明确标注"外推，KB 无直接证据"，而不是凭印象编造建议。

### 6.5 KB 覆盖范围

当前 KB（截至 2026-04-08）分析了 4 个源项目：Claude Code、OpenHarness、DeerFlow、Hermes Agent。

当用户的场景在 KB 中没有充分对应（例如：专用于语音交互的 agent，或实时嵌入式系统中的 agent），应主动说明置信度限制：
> "你这个场景（[描述]）在知识库当前收录的项目中没有直接对应案例。以下建议基于架构原则推断，请谨慎参考。"

---

## 7. 未来扩展（简述）

### 7.1 新源项目 Ingest 后自动扩展

随着知识库 ingest 新的源项目（如 LangChain、AutoGen、CrewAI），各 L1 页面的"各家对比"和"场景决策指南"会自动丰富。本 skill 无需修改——因为它读的是 L1 页面而非硬编码的项目列表，新的对比数据会自然流入建议质量中。

惟一需要更新的地方：Q1-Q11 的映射矩阵（Section 3.3）。当新源项目引入了 KB 中尚未覆盖的新场景类型，矩阵应相应更新。

### 7.2 用户级 Skill 部署

目前本 skill 设计为项目级（`.claude/skills/`）。当用户的架构咨询需求跨多个 agent 项目时，可将其迁移到用户级 skill（`~/.claude/skills/`），配合知识库路径的绝对路径引用，实现跨项目可用。

### 7.3 与 create-agent-skill 的集成

架构建议输出的"下一步"section 目前是手动建议阅读 L2 页面。未来可以：
1. 直接输出一份 `create-agent-skill` 兼容的"架构配置文件"（JSON/YAML 格式）
2. 由 `create-agent-skill` 读取该配置，自动选择对应的 skill 模板和组件
3. 实现从"架构建议 → 可运行 agent"的完整自动化链路

这需要 `create-agent-skill` 侧添加"从架构配置初始化"的入口，是两个 skill 的协作演进方向。

---

## 附录：KB 路径速查

| 文件类型 | 路径模式 | 示例 |
|---------|---------|------|
| L1 概念页 | `wiki/{concept-id}.md` | `wiki/memory-system.md` |
| L2 实现详情 | `wiki/_impl/{concept}--{source}.md` | `wiki/_impl/memory-system--hermes-agent.md` |
| 知识索引 | `wiki/_index.md` | — |
| 本体定义 | `schema/ontology.yaml` | 包含所有 L1 概念 ID 和 relations |

**L1 概念 ID 完整列表**（来自 `schema/ontology.yaml`）：

| ID | 标题 | 一句话定义 |
|----|------|----------|
| `prompt-system` | Prompt System | 动态组装发给模型的 prompt |
| `query-loop` | Query Loop | Agent 主循环 — 发请求、拿结果、判断下一步 |
| `runtime-state` | Runtime State | 运行时状态容器与生命周期 |
| `context-management` | Context Management | 管理有限的 context window |
| `tool-system` | Tool System | 发现、选择、调用、管理外部工具 |
| `memory-system` | Memory System | 跨会话的持久记忆 |
| `multi-agent` | Multi-Agent | 多 agent 协作 — 拆任务、分发、汇总 |
| `hooks` | Hooks | 事件驱动的扩展点 |
| `mcp-skills` | MCP & Skills | 外部扩展协议 + 可复用能力包 |
| `channel-remote` | Channel & Remote | 多渠道接入与远程执行 |
| `session-recovery` | Session Recovery | 断点恢复与容错 |
| `evaluation-observability` | Evaluation & Observability | 效果评估 + 运行可观测 |
