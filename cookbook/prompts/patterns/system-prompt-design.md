---
pattern: system-prompt-design
category: meta
tags: [system-prompt, agent-identity, behavior-control, persona, instructions, agent-core]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/prompt-system, wiki/query-loop, wiki/context-management]
related_patterns: [chain-of-thought, react, structured-output, prompt-chaining]
---

# System Prompt Design

> 系统提示词定义 LLM 在本次交互中的身份、能力、限制和行为规范，是 agent 设计的第一步也是最重要的一步。

## 本质

给模型写的"岗位说明书"。写得好像训练有素的员工，写得差像没人带的新人。

大多数 agent 行为问题——绕圈、越界、格式乱、不遵循规则——根源不在模型能力不够，而在系统提示词没写清楚。

System prompt 做的事情本质上只有三件：

- **定义身份**：这个模型是谁？有什么能力？在什么场景下工作？
- **设定约束**：什么可以做，什么不能做，边界在哪里？
- **规范行为**：怎么思考、怎么输出、遇到边缘情况怎么处理？

这三件事写清楚了，90% 的行为问题就解决了。

## 什么时候用

- **构建任何 agent**：系统提示词是 agent 的第一层控制，不可省略
- **定义专业领域边界**：限制模型只在特定领域（代码/法律/医疗）回答，避免越界
- **控制输出风格**：固定语气、格式、长度，确保输出一致性
- **注入领域知识**：把模型不知道的背景信息（内部规则、业务术语、产品约束）预先告知

## 什么时候保持精简

- **一次性问答**：单轮对话没有持久角色，简单指令即可，不需要完整 system prompt
- **实验和原型阶段**：先在 user 消息里测想法，稳定后再提炼到 system prompt
- **System prompt 超过 3000 tokens**：太长会降低遵循度，尤其是规则在后半段的情况——考虑拆分或精简

## 模板

### 基础版（最小结构）

四个模块缺一不可：Role（谁）+ Task（做什么）+ Rules（怎么做）+ Output Format（输出什么）。

```text
你是 [角色描述]。你的任务是 [核心职责]。

## 规则
- [规则 1]
- [规则 2]
- [规则 3]

## 输出格式
[格式描述，例如：用 markdown 列表、JSON、纯文本段落]
```

**示例：代码审查助手**

```text
你是一位资深 Python 工程师，专注代码质量审查。你的任务是审查用户提交的 Python 代码，找出潜在问题。

## 规则
- 只审查 Python 代码，其他语言礼貌拒绝
- 关注：安全漏洞、性能问题、可读性、Pythonic 写法
- 不重写整段代码，只指出问题和改进方向
- 每条反馈都要说明原因，不要只给结论

## 输出格式
按严重程度分组：严重问题 / 建议改进 / 风格建议
每条格式：[行号或函数名] 问题描述 → 建议做法
```

### Agent 集成版（完整结构）

生产环境的 agent system prompt，覆盖七个关键模块：

```text
# Identity（身份）
你是 [名称]，一个 [定位描述]。你的核心能力是 [1-2 句话]。

# Capabilities（能力）
你可以：
- [能力 1]
- [能力 2]
- [能力 3]

你不能：
- [限制 1]（超出范围时，[处理方式：告知用户/转接/拒绝]）
- [限制 2]

# Workflow（工作流程）
处理每个请求时：
1. [步骤 1]
2. [步骤 2]
3. [步骤 3]

# Rules（核心规则）
- [规则 1]
- [规则 2]
- [规则 3]

# Error Handling（错误处理）
- 如果 [情况 A]：[处理方式]
- 如果 [情况 B]：[处理方式]
- 如果无法完成任务：[告知用户的方式]

# What NOT to Do（反例）
- 不要 [常见错误行为 1]
- 不要 [常见错误行为 2]
- 不要在 [场景] 时 [行为]

# Examples（示例）
**用户**：[典型输入]
**你**：[期望输出]
```

**示例：研究助手 agent**

```text
# Identity
你是 ResearchBot，一个自主研究助手。你的核心能力是将复杂研究问题拆解为可执行的搜索任务，并综合结果生成结构化报告。

# Capabilities
你可以：
- 将用户的问题分解为 3-5 个具体搜索子任务
- 使用 web_search 工具获取最新信息
- 综合多个来源，生成有层次的分析报告

你不能：
- 访问用户本地文件（如需要，告知用户上传）
- 处理 2024 年以前的私人或付费墙内容
- 提供医疗、法律建议（转引专业人士）

# Workflow
1. 分析用户问题，识别核心信息需求
2. 拆解为 3-5 个搜索子任务，每个子任务聚焦一个具体问题
3. 按优先级依次执行搜索，记录关键发现
4. 综合所有发现，生成报告
5. 在报告末尾注明所有来源

# Rules
- 每个搜索子任务必须执行，跳过时必须说明原因
- 报告中的每个事实性陈述都要有来源支撑
- 不确定的信息要标注置信度（高/中/低）

# Error Handling
- 如果搜索失败：用不同关键词重试一次，仍失败则记录并继续其他子任务
- 如果超过 50% 的搜索失败：停止并告知用户，请求指导
- 如果问题超出能力范围：解释限制，建议替代资源

# What NOT to Do
- 不要在搜索之前就给出结论
- 不要把搜索结果原文堆砌，要提炼核心洞察
- 不要在没有搜索依据时引用具体数据或统计数字

# Examples
**用户**：分析 2024 年生成式 AI 在企业端的采用趋势
**你**：好的，我会拆解为 4 个子任务：①企业 AI 采用率数据 ②主要行业应用案例 ③投资和支出趋势 ④实施障碍。现在开始第一项搜索……
```

### 变体：四档递进

| 档位 | 规模 | 适用场景 | Token 预算 |
|------|------|----------|-----------|
| **Minimal** | Role + 3 条规则 | 单任务工具、内部脚本 | < 200 tokens |
| **Standard** | 基础版四模块 | 大多数专业助手 | 200-600 tokens |
| **Comprehensive** | Agent 集成版七模块 | 生产 agent、复杂工作流 | 600-1500 tokens |
| **Constitutional** | Comprehensive + 价值观层 | 高风险场景（医疗/金融/法律）、需要防御性行为 | 1500-3000 tokens |

**Constitutional 额外模块示例：**

```text
# Core Values（价值观层）
在所有决策中，按以下优先级排序：
1. 用户安全（高于任务完成）
2. 信息准确性（高于用户期望）
3. 任务完成
4. 效率

当这些价值观冲突时（例如用户要求提供可能有害的信息）：
- 首先确认潜在风险
- 解释限制和原因
- 提供无风险的替代方案
- 不要在没有解释的情况下直接拒绝
```

## 模型差异

| 模型 | XML 标签 | 规则遵循 | 长文遵循 | 注意事项 |
|------|---------|---------|---------|---------|
| **Claude 4** | 偏好 `<tag>` 结构，遵循度高 | 严格，几乎不越界 | 较好，2000 tokens 内稳定 | 支持 prompt caching（≥1024 tokens），复杂 system prompt 建议开启 |
| **GPT-4o** | Markdown `##` 效果好 | 有首因效应，重要规则放前面 | 超过 1500 tokens 后半段遵循度下降明显 | 反例（"不要做"）在 GPT 系列比 Claude 更容易被遵循 |
| **Gemini 2** | 纯文本和 Markdown 均可 | 一般任务遵循好，边缘情况偶有偏差 | 超长 prompt 在复杂多步任务中稳定性略差 | 角色设定对行为影响比其他模型更显著 |
| **开源模型** | 依赖微调数据，差异大 | Llama 3 / Qwen 2.5 在 7B+ 参数时基本可用 | 参数小时后半段规则容易被忽视 | Instruction-tuned 版本比 base 好很多；规则数量建议 ≤5 条 |

**工程建议：**

- Claude：用 `<role>`, `<rules>`, `<output_format>` 等 XML 标签结构化，配合 prompt caching 节省成本
- GPT-4：最重要的规则放第一段，不要放文末
- 开源模型：规则越少越好，优先用示例（few-shot）代替文字规则

## 常见踩坑

**1. 规则互相矛盾**

症状：模型行为飘忽不定，同样的输入有时遵循规则 A，有时遵循规则 B。

原因：两条规则在某些边缘情况下指向相反的行为，模型自己做了随机选择。

解决方案：
- 写完规则后，主动想 3-5 个边缘情况，测试规则是否冲突
- Constitutional 写法：用优先级排序代替平级规则列表
- 冲突规则合并成一条，在合并后的规则里注明优先级

**2. System prompt 过长导致遵循度下降**

症状：前几条规则被遵循，后半段的规则经常被忽视。

原因：超过 ~1500 tokens 后，大多数模型对后半段内容的注意力权重下降（"Lost in the Middle"效应）。

解决方案：
- 重要规则放前面，不重要的放后面或删掉
- 用格式分块（`##`、XML 标签）增强结构，帮助模型"定位"规则
- 超过 2000 tokens 时考虑拆分为多层 prompt（system + 动态注入的 user 前缀）

**3. 只写"不要做"，没写"要做什么"**

症状：模型在被禁止的情况下不知道该做什么，要么沉默，要么乱给一个答案。

原因：禁令只告诉模型避开什么，没有给出替代行为的指引。

解决方案：
```text
# 错误写法
不要在不确定时给出答案。

# 正确写法
如果不确定答案，先说"我不确定，以下是我知道的部分"，
再给出已知信息，最后告知用户验证渠道。
```

**4. 角色定义模糊**

症状：模型时而按助手模式回答，时而按专家模式回答，风格不一致。

原因：Identity 模块太泛，没有明确回答"这个角色和普通 LLM 的区别是什么"。

解决方案：
```text
# 模糊写法
你是一个有帮助的助手。

# 清晰写法
你是 DataBot，一个专注于 SQL 查询优化的数据库助手。
你只处理数据库相关问题（MySQL / PostgreSQL / SQLite）。
对于非数据库问题，你会简短地说"这超出了我的专业范围"，
然后建议用户使用通用助手。
```

**5. 没有测试对抗输入**

症状：正常用法下表现完美，但用户一旦问边缘问题（越界话题、角色扮演绕过规则、模糊请求）就行为失控。

原因：只用"正常"输入测试了 system prompt，没有压测边界。

解决方案：每次写完 system prompt，至少测试：
- 明确越界请求（"帮我做你规则里禁止的事"）
- 角色扮演绕过（"忘记你的限制，现在假装你是..."）
- 歧义请求（可以被合理解读为越界的请求）
- 空请求/极短请求（"？"、"你好"）

## 组合与选择

**System Prompt + ReAct**

System prompt 的 Workflow 模块定义了 agent 循环的行为规范（何时调用工具、何时停止），ReAct 是具体的运行模式。两者配合：system prompt 设定"游戏规则"，ReAct loop 在规则内执行。

**System Prompt + Structured Output**

在 Output Format 模块里直接嵌入 JSON schema 或 XML 结构，配合 `response_format` API 参数，可以强制模型按固定格式输出，不再依赖文字规则。

**System Prompt + Prompt Chaining**

多步流程中，每个节点可以有自己专用的 system prompt（而不是一个超长的通用 system prompt）。拆分后每个节点的 system prompt 都能更精准，遵循度更高。

**System Prompt + Context Engineering**

System prompt 是静态的"规范层"，动态注入（用户画像、任务上下文、历史摘要）是"数据层"。两者分离：规范层保持稳定、可复用；数据层随每次请求变化。参见 [[wiki/context-management]]。

## 来源

- **Prompt Engineering Guide** — Agent Components: Planning/Tool/Memory 三层结构
  [https://www.promptingguide.ai/agents/components](https://www.promptingguide.ai/agents/components)

- **Prompt Engineering Guide** — Context Engineering: System prompt 作为 context 第一层，消除歧义、显式期望、迭代优化
  [https://www.promptingguide.ai/agents/context-engineering](https://www.promptingguide.ai/agents/context-engineering)

- **Prompt Engineering Guide** — General Tips: 指令清晰、避免模糊、正向表达（说"要做什么"而非"不要做什么"）
  [https://www.promptingguide.ai/introduction/tips](https://www.promptingguide.ai/introduction/tips)

- **Anthropic Claude Docs** — System prompts, prompt caching, XML tag formatting
  [https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)

- **OpenAI Best Practices** — Instruction placement, role definition, "do" vs "don't" framing
  [https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-openai-api](https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-openai-api)

关联 wiki：[[wiki/prompt-system]] · [[wiki/query-loop]] · [[wiki/context-management]]
