# Tool Definitions

> 按功能分类的工具定义。每个页面包含可复制 schema + 跨项目对比 + 最佳实践。

## 工具类别

### 核心（几乎所有 Agent 都需要）
- [File Operations](file-operations.md) — 文件系统的六把钥匙：Read/Write/Edit/Glob/Grep/NotebookEdit，代码生成、文档处理、数据管道的基础设施
- [Shell Execution](shell-execution.md) — 让 agent 在宿主机或沙箱中执行任意 shell 命令，是 agent 与操作系统交互的最底层通道，也是权限风险最高的工具类别
- [Web Tools](web-tools.md) — 让 agent 能访问互联网——搜索最新信息、抓取网页内容，打破训练数据截止日期限制

### Agent 编排
- [Agent Orchestration](agent-orchestration.md) — 子 Agent 派发、任务生命周期管理、团队协作的工具集，多 Agent 系统的基础设施层
- [MCP Integration](mcp-integration.md) — 通过 Model Context Protocol 发现、加载、调用外部工具，解决"工具太多放不进 context"的工程问题

### 辅助
- [User Interaction](user-interaction.md) — 让 agent 在执行过程中主动向用户提问、请求确认或发送消息的工具集
- [Scheduling](scheduling.md) — 让 agent 创建、管理和触发定时任务，实现无人值守的周期性自动化
- [Code Intelligence](code-intelligence.md) — 通过 Language Server Protocol 实现精确的代码导航——跳转定义、查找引用、查看符号，比文本搜索更准确
- [Git Worktree](git-worktree.md) — 为 agent 创建隔离的工作目录，在不污染主工作区的情况下开发 feature、实验或并行任务
- [Skill System](skill-system.md) — 让 agent 加载、执行和管理 skill，可复用的能力模块，实现功能的动态扩展
