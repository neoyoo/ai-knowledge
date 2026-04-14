---
pattern: agent-dispatch
category: agent-dispatch
tags: [agent, subagent, parallel, orchestration, delegation]
difficulty: advanced
examples: [subagent-driven-development, dispatching-parallel-agents, memory-management]
related_patterns: [multi-doc-navigation, hook-triggered]
---

# Agent Dispatch 型 Skill

> 一句话定义：Skill 不自己执行任务，而是负责"判断时机 + 准备上下文 + 派发 subagent"，把真正的工作交给独立 agent 去做。

## 本质

Skill 在这里扮演**调度员**角色，而不是执行者。它回答三个问题：

1. **什么时候该派**：用决策树描述触发条件（任务数量、独立性、执行环境）
2. **派什么**：指向 agent prompt 文件，这是 skill 的核心载荷
3. **派完怎么收**：如何整合结果、验证质量、决定下一步

这个模式的价值在于把"判断逻辑"和"执行 prompt"分离。SKILL.md 负责判断，`xxx-prompt.md` 负责执行质量。

## 什么时候用

- 任务超过 5 步，或预计执行时间超过 1 分钟
- 有多个**相互独立**的子任务，可以并行处理
- 需要**角色分离**：实现者、审查者、验证者各自独立
- 子任务需要**全新上下文**，避免主线程的状态污染
- 执行过程中需要**质量门禁**（self-review + spec compliance + code quality）

## 什么时候不用

- 任务紧密耦合，前一步的输出是后一步的输入（用 sequential pipeline 而非 parallel）
- 需要实时交互和方向调整（subagent 不适合频繁 mid-task 决策）
- 任务太简单，dispatch 开销大于收益（3 步以内直接执行）
- Agents 会操作**同一个文件**（写冲突风险，shared state 禁止并行）

## 目录结构

两种子模式：

### 子模式一：单 agent 派发（只有 SKILL.md）

适合逻辑简单、一次性派发、无需角色分离的场景。

```
skills/my-skill/
└── SKILL.md          # 包含触发条件 + 内嵌 agent prompt 模板
```

`dispatching-parallel-agents` 就是这种结构：SKILL.md 内直接给出 agent prompt 示例，不需要独立文件。

### 子模式二：多 agent 并行（SKILL.md + agent prompt 子文件）

适合流程复杂、需要多角色、prompt 内容需要复用的场景。

```
skills/my-skill/
├── SKILL.md                        # 触发条件、流程决策树、引用下方文件
├── implementer-prompt.md           # 实现 agent 的完整 prompt 模板
├── spec-reviewer-prompt.md         # Spec 合规审查 agent 的 prompt 模板
└── code-quality-reviewer-prompt.md # 代码质量审查 agent 的 prompt 模板
```

`subagent-driven-development` 采用这种结构：三个角色各有独立 prompt 文件，SKILL.md 只负责流程编排。

## Agent Prompt 模板

这是 agent-dispatch skill 最核心的部分。一个好的 agent prompt 模板必须具备：

- **角色声明**：让 agent 知道自己是谁、在做什么
- **完整上下文**：不让 agent 自己去读文件，由 caller 直接注入
- **明确边界**：做什么、不做什么都要说清楚
- **问题升级机制**：遇到不确定的情况，先问再动手
- **报告格式**：输出结构化，方便 caller 整合

### 实现类 agent prompt 模板

```
Task tool:
  description: "Implement {{TASK_NAME}}"
  prompt: |
    你正在实现：{{TASK_NAME}}

    ## 任务描述

    {{TASK_DESCRIPTION}}
    （完整任务文本，由 caller 直接粘贴，不要让 agent 自己读文件）

    ## 上下文

    {{CONTEXT}}
    （场景说明：这个任务在整体计划中的位置、依赖关系、架构背景）

    ## 约束条件

    {{CONSTRAINTS}}
    （例如：只改 src/xxx 目录、不动测试文件、遵循现有代码风格）

    ## 工作目录

    {{WORKING_DIRECTORY}}

    ## 开始前

    如果你对以下任何一点有疑问：
    - 需求或验收标准
    - 实现方式或技术选型
    - 依赖关系或前提假设
    - 任务描述中不清楚的地方

    **先问清楚再动手。** 工作过程中遇到意外也可以随时暂停提问，不要猜测。

    ## 你的工作

    确认需求后：
    1. 按任务描述精确实现，不多不少
    2. 写测试（如任务要求 TDD，则先写测试）
    3. 验证实现正确
    4. Commit 你的工作
    5. Self-review（见下方清单）
    6. 报告结果

    ## Self-Review 清单

    报告前，用新鲜视角审查你的工作：

    **完整性：**
    - 是否实现了 spec 要求的全部内容？
    - 是否遗漏了任何需求？
    - 边界情况是否处理？

    **质量：**
    - 代码是否清晰可维护？
    - 命名是否准确反映实际行为？
    - 是否遵循了代码库现有模式？

    **克制性：**
    - 是否避免了过度设计（YAGNI）？
    - 是否只构建了被要求的内容？

    **测试：**
    - 测试是否真正验证了行为，而非只是 mock？
    - 覆盖率是否充分？

    发现问题就地修复，再报告。

    ## 报告格式

    完成后报告：
    - 实现了什么
    - 测试结果（通过/失败数量）
    - 改动的文件列表
    - Self-review 发现（如有）
    - 未解决的问题或顾虑
```

### 审查类 agent prompt 模板

```
Task tool:
  description: "Review spec compliance for {{TASK_NAME}}"
  prompt: |
    你正在审查一个实现是否符合其规格说明。

    ## 原始需求

    {{TASK_REQUIREMENTS}}
    （完整的任务需求文本）

    ## 实现者声称完成了什么

    {{IMPLEMENTER_REPORT}}
    （来自实现 agent 的报告）

    ## 重要：不要相信报告

    实现者的报告可能不完整、不准确或过于乐观。
    你必须独立验证所有内容。

    **不要：**
    - 相信他们说实现了什么
    - 信任他们对需求的解读
    - 接受"差不多"的结果

    **必须：**
    - 直接读实际代码
    - 逐条对比需求与实现
    - 检查他们声称实现但实际缺失的部分

    ## 你的工作

    读代码，验证：

    **缺失项：** 是否有需求未被实现？
    **多余项：** 是否有未被要求的功能被添加？
    **误解项：** 是否对需求有错误解读？

    报告格式：
    - ✅ Spec 合规（代码检查后，所有内容匹配）
    - ❌ 发现问题：[具体列出缺失或多余的内容，附 file:line 引用]
```

## SKILL.md 模板

SKILL.md 负责"何时派发"和"派发什么"，不负责执行细节。

```markdown
---
name: {{SKILL_NAME}}
description: {{TRIGGER_DESCRIPTION}}
---

# {{SKILL_TITLE}}

## 何时使用

（决策树：用 dot 图或 bullet 列表描述触发条件）

**使用条件：**
- 条件一
- 条件二

**不用条件：**
- 反例一
- 反例二

## 流程

（step-by-step 流程，或 dot 流程图）

1. 分析任务，识别独立子任务
2. 为每个子任务准备上下文（从计划文件提取，不让 agent 自己读）
3. 派发 agent（引用 `./implementer-prompt.md`）
4. 等待结果，处理质量问题
5. 整合所有结果

## Prompt 模板

- `./implementer-prompt.md` — 实现 agent
- `./reviewer-prompt.md` — 审查 agent

## 反模式

- 让 agent 自己读计划文件（caller 应该提供完整文本）
- 并行派发有写冲突风险的 agent
- 跳过审查环节
```

## 变体

### 单 agent 顺序派发

最简单的形式。一个任务对应一个 agent，串行执行。每个 agent 完成后才派下一个。

```
任务 1 → Agent A → 完成
任务 2 → Agent B → 完成
任务 3 → Agent C → 完成
```

适用：任务有先后依赖，或需要前一步结果决定下一步策略。

### 并行多 agent

多个独立任务同时派发，充分利用并发。

```
任务 A ──→ Agent 1 ─┐
任务 B ──→ Agent 2 ─┤→ 整合结果
任务 C ──→ Agent 3 ─┘
```

适用：任务相互独立，无共享状态，无写冲突。`dispatching-parallel-agents` 是标准实现。

### Sequential pipeline（多角色流水线）

同一任务经过多个专职 agent，每个 agent 负责一个环节。

```
任务 → Implementer → Spec Reviewer → Quality Reviewer → 完成
```

适用：需要严格质量门禁，不同角色关注点不同。`subagent-driven-development` 是标准实现。

## 真实案例分析

### subagent-driven-development 的三文件结构

这个 skill 展示了 agent-dispatch 最成熟的形式：

**SKILL.md** 只做三件事：
1. 决策树（有计划？任务独立？同一 session？）
2. 流程图（implementer → spec reviewer → quality reviewer）
3. 引用子文件（`./implementer-prompt.md` 等）

**implementer-prompt.md** 是给实现 agent 的完整指令，包含：
- `[FULL TEXT of task from plan]` 占位符，明确要求 caller 直接粘贴内容
- "开始前先问问题"机制，防止 agent 基于错误假设动工
- Self-review 清单，内建质量检查
- 结构化报告格式，方便 caller 整合

**spec-reviewer-prompt.md** 是给审查 agent 的指令，核心设计是：
- 明确写出 "Do Not Trust the Report"
- 要求直接读代码，而非相信实现者的汇报
- 输出格式用 ✅/❌ 结构化，方便自动化处理

**为什么要把 agent prompt 独立成文件，而不是嵌入 SKILL.md：**

1. **复用性**：implementer-prompt.md 可以被多个 skill 引用，不需要复制粘贴
2. **可测试性**：可以独立测试单个 prompt 的效果，不需要触发整个 skill
3. **可读性**：SKILL.md 保持简洁（决策和流程），prompt 文件专注于执行细节，两者不相互污染
4. **可维护性**：修改某个角色的 prompt，不需要改动 SKILL.md 的流程逻辑
5. **长度管理**：agent prompt 往往很长（50-100 行），嵌入 SKILL.md 会让文件难以阅读

### dispatching-parallel-agents 的内嵌 prompt 结构

这个 skill 选择把 prompt 示例**直接嵌入** SKILL.md，而不是独立文件。原因：

- Prompt 示例短（约 10 行），不值得独立成文件
- 是**示例**而非**模板**，每次使用都需要根据具体场景定制
- 强调的是"好 prompt 的特征"（focused、self-contained、specific output），而非固定格式

这说明：**独立 prompt 文件适合稳定的、可复用的模板；内嵌 prompt 适合示范性的、高度定制化的片段。**

## 常见踩坑

**1. Skill 只说"派发一个 agent"但没有 agent prompt 内容**

最常见的错误。SKILL.md 写了"dispatch implementer"，但没有 agent 应该收到什么内容。结果每次使用都要临时想 prompt，失去了 skill 的复用价值。

修复：始终提供完整 agent prompt 模板，用 `{{PLACEHOLDER}}` 标出需要填充的部分。

**2. Agent prompt 太短，缺乏足够上下文**

给 agent 发一句 "fix the bug in auth module" 这类 prompt，agent 会花大量时间探索代码库，产生错误假设，或者做出范围不对的修改。

修复：Agent prompt 必须包含：具体范围、完整需求文本、约束条件、预期输出格式。如果 agent 需要读文件才能理解任务，caller 应该提前读好直接注入 prompt。

**3. 并行 agent 之间有隐藏依赖**

两个 agent 同时修改同一文件，或 Agent B 的正确执行依赖 Agent A 先完成某个状态变更，但并行执行时顺序不确定。

修复：派发前做独立性检查。判断标准：如果把两个任务互换执行顺序，结果是否完全相同？如果不是，就不能并行。

**4. 让 agent 自己去读计划文件**

浪费 token，更重要的是 agent 会读到比它需要的更多内容，干扰专注度，还可能读到错误版本。

修复：Caller 预先读好计划，把相关任务的**完整文本**直接注入 agent prompt。

**5. 跳过 re-review 循环**

审查 agent 发现问题后，实现 agent 修复了，但没有再次触发审查。"修复"本身也可能引入新问题。

修复：审查 → 修复 → 再审查，直到明确通过。不接受"应该修好了"。

## 来源

- `superpowers/4.3.1/skills/subagent-driven-development/SKILL.md`
- `superpowers/4.3.1/skills/subagent-driven-development/implementer-prompt.md`
- `superpowers/4.3.1/skills/subagent-driven-development/spec-reviewer-prompt.md`
- `superpowers/4.3.1/skills/dispatching-parallel-agents/SKILL.md`
