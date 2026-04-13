# Tool Definitions

> 按功能分类的工具定义。每个页面包含可复制 schema + 跨项目对比 + 最佳实践。

## 工具类别

### 核心（几乎所有 Agent 都需要）
- [File Operations](file-operations.md) — 文件读写：Read、Write、Edit、Glob、Grep、NotebookEdit
- [Shell Execution](shell-execution.md) — 命令执行：Bash、PowerShell
- [Web Tools](web-tools.md) — 网络交互：WebSearch、WebFetch

### Agent 编排
- [Agent Orchestration](agent-orchestration.md) — 子 Agent 管理：Agent、Task、SendMessage
- [MCP Integration](mcp-integration.md) — MCP 协议工具：ListResources、ReadResource、ToolSearch

### 辅助
- [User Interaction](user-interaction.md) — 用户交互：AskUserQuestion、Brief
- [Scheduling](scheduling.md) — 定时任务：CronCreate/List/Delete、RemoteTrigger
- [Code Intelligence](code-intelligence.md) — 代码理解：LSP（定义跳转、引用查找、符号搜索）
- [Git Worktree](git-worktree.md) — 工作区隔离：EnterWorktree、ExitWorktree
- [Skill System](skill-system.md) — 技能系统：Skill、SkillManage、ToolSearch
