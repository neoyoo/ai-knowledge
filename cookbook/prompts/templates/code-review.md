---
template: code-review
scenario: AI 辅助代码审查
tags: [code-review, quality, security, best-practices]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_patterns: [chain-of-thought, structured-output, system-prompt-design]
---

# Code Review Template

> 让 LLM 扮演资深工程师，对提交的代码给出分级、结构化的审查意见，覆盖正确性、安全、性能、可维护性四个维度。

## 场景描述

代码审查是工程团队保障质量的关键环节，但人工 review 耗时且覆盖面参差不齐。这个模板把审查标准、严重度分级和输出格式固化进 system prompt，让 LLM 输出和有经验的工程师的 review 意见格式对齐，可直接对接 PR 评论或 CI 管道。适用于 PR review bot、IDE 插件、代码质量 CI 步骤等场景。

## 模板

### System Prompt

```text
你是一名资深软件工程师，专注于代码质量和安全审查。你的任务是对提交的代码进行系统性审查，给出可操作的改进建议。

## 审查维度

按以下四个维度审查代码，每个维度都必须覆盖：

1. **正确性（Correctness）** — 逻辑是否正确？边界条件处理是否完整？是否有 off-by-one 错误、空指针、竞态条件？
2. **安全性（Security）** — 是否有注入漏洞（SQL/XSS/命令注入）？敏感数据是否泄露？权限检查是否到位？依赖是否存在已知 CVE？
3. **性能（Performance）** — 是否有 N+1 查询、不必要的全量加载、低效算法、内存泄漏？
4. **可维护性（Maintainability）** — 命名是否清晰？函数是否单一职责？是否有重复代码？测试覆盖率是否足够？

## 严重度分级

- **CRITICAL** — 必须修复，否则不能合并。安全漏洞、数据丢失风险、逻辑错误导致功能不可用。
- **MAJOR** — 强烈建议修复。性能问题、可维护性严重受损、测试缺失的核心逻辑。
- **MINOR** — 建议修复，不阻塞合并。代码风格、命名改善、小优化。
- **NIT** — 可选改进，作者自行判断。格式、注释、变量名偏好等。

## 输出格式

必须严格按以下 JSON 格式输出，不要在 JSON 外添加任何文本：

\`\`\`json
{
  "summary": {
    "verdict": "APPROVE | REQUEST_CHANGES | COMMENT",
    "critical_count": 0,
    "major_count": 0,
    "minor_count": 0,
    "nit_count": 0,
    "overall_assessment": "一句话总结代码质量和主要问题"
  },
  "issues": [
    {
      "id": "ISSUE-001",
      "severity": "CRITICAL | MAJOR | MINOR | NIT",
      "dimension": "correctness | security | performance | maintainability",
      "location": "文件名:行号 或 函数名",
      "title": "问题标题（15字以内）",
      "description": "问题详细描述，说明为什么这是个问题",
      "suggestion": "具体修复建议，如有代码示例更好",
      "example": "可选：修复后的代码片段"
    }
  ],
  "positives": [
    "做得好的地方，至少列出 1 条"
  ]
}
\`\`\`

如果代码没有问题，issues 数组为空，verdict 为 APPROVE。
```

### User Prompt

#### 全文件模式

```text
请审查以下代码：

**语言/框架**：{{language_and_framework}}
**上下文**：{{context_description}}
**重点关注**：{{focus_areas}}（可选，如"重点看安全性"）

\`\`\`{{language}}
{{code_content}}
\`\`\`
```

#### Diff 模式（PR review）

```text
请审查以下 Pull Request 的变更：

**PR 标题**：{{pr_title}}
**变更描述**：{{pr_description}}
**语言/框架**：{{language_and_framework}}

变更 diff：

\`\`\`diff
{{git_diff_content}}
\`\`\`

注意：只审查 diff 中新增和修改的行（+ 号开头），不要对删除的行（- 号开头）提出问题。
```

## 自定义指南

**调整审查维度权重**：在 system prompt 的维度描述后添加权重说明，例如"这是金融系统，安全性权重最高，发现任何安全问题直接标记 CRITICAL"。

**添加项目规范**：在 system prompt 末尾追加一段"项目特定规范"，列出团队约定（如"本项目使用 snake_case 命名"、"所有数据库查询必须使用 ORM"），让审查意见与实际规范对齐。

**控制输出详细程度**：
- 快速审查：在 user prompt 加"只输出 CRITICAL 和 MAJOR 级别的问题"
- 教学模式：在 system prompt 加"对每个问题，除了建议外，解释背后的原理"

**多文件审查**：在 user prompt 中依次列出多个文件，每个文件用标题分隔，告诉模型"这些文件是同一个功能模块，跨文件的问题（如接口不一致）也要指出"。

**语言特定规则**：可以在 system prompt 末尾追加针对特定语言的规则，例如 Python 的"检查是否遗漏 type hint"、Go 的"检查 error 是否被正确处理"。

## 使用示例

**输入**（全文件模式）：

```
语言/框架：Python / FastAPI
上下文：用户登录接口
重点关注：安全性

def login(username: str, password: str, db: Session):
    user = db.execute(f"SELECT * FROM users WHERE username='{username}'").first()
    if user and user.password == password:
        return {"token": create_token(user.id)}
    return {"error": "Invalid credentials"}
```

**输出**（示意）：

```json
{
  "summary": {
    "verdict": "REQUEST_CHANGES",
    "critical_count": 2,
    "major_count": 1,
    "minor_count": 0,
    "nit_count": 0,
    "overall_assessment": "存在 SQL 注入和明文密码比较两个严重安全漏洞，必须修复后才能合并"
  },
  "issues": [
    {
      "id": "ISSUE-001",
      "severity": "CRITICAL",
      "dimension": "security",
      "location": "login:第2行",
      "title": "SQL 注入漏洞",
      "description": "直接拼接用户输入到 SQL 字符串，攻击者可通过 username=' OR 1=1 -- 绕过认证或执行任意 SQL",
      "suggestion": "使用参数化查询",
      "example": "db.execute('SELECT * FROM users WHERE username=:username', {'username': username}).first()"
    },
    {
      "id": "ISSUE-002",
      "severity": "CRITICAL",
      "dimension": "security",
      "location": "login:第3行",
      "title": "明文密码比较",
      "description": "直接比较明文密码意味着数据库中存储的是明文，一旦数据库泄露所有用户密码暴露",
      "suggestion": "使用 bcrypt 或 argon2 哈希后比较",
      "example": "if user and bcrypt.checkpw(password.encode(), user.password_hash):"
    }
  ],
  "positives": ["使用了 FastAPI 的依赖注入获取 db session，结构正确"]
}
```

## 适配建议

**diff 模式 vs 全文件模式的选择**：diff 模式聚焦变更，减少无关噪音，适合 PR review 自动化；全文件模式覆盖更全，适合初始代码质量评估或重要模块的深度审查。

**降低误报率**：在 user prompt 中提供足够的上下文（框架、业务背景、已知约束），避免模型对"看起来奇怪但实际合理"的代码报错。例如告知"这段代码运行在内网环境，不需要外部安全防护"。

**与 CI 集成**：输出是 JSON，可直接解析后通过 GitHub API 写入 PR review comments，每个 issue 对应一条 review comment。verdict 字段可以直接映射到 PR approve/request_changes 状态。

**处理大文件**：超过 500 行的文件，按模块分块送审，最后合并 issues 列表。避免单次 context 过大导致注意力分散，后半段代码被忽视。

## 关联

- 模式：[[cookbook/prompts/patterns/structured-output]] — 本模板的 JSON 输出格式基于 Structured Output pattern
- 模式：[[cookbook/prompts/patterns/chain-of-thought]] — 在 system prompt 中加入 CoT 指令可提升问题定位的准确性
- 模式：[[cookbook/prompts/patterns/system-prompt-design]] — 审查标准和维度定义是 system prompt 设计的典型案例
- Wiki：[[wiki/prompt-system]] — 理解 system/user prompt 分工
