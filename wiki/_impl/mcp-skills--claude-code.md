---
title: "MCP、Skills 与 Plugins — Claude Code"
category: L2
parent: "[[mcp-skills]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的扩展层不是单一插件机制，而是按抽象层次拆成三种不同形态：MCP（协议接入层）、Skills（工作流模板层）、Plugins（生态封装层）。三者分工明确、互不混淆——MCP 解决"能接什么工具"，Skills 解决"怎么做某类任务"，Plugins 解决"如何打包分发整套能力"。这种分层设计使扩展体系在保持灵活性的同时具备清晰的边界。

## 架构分析

### MCP — 统一协议接入层

`src/services/mcp/client.ts` 是 MCP 层的核心，负责连接外部能力并将其桥接到 Claude Code 的工具层。

**传输层多路支持：**
MCP client 支持三种传输协议，在连接时自动选择：
- `StdioClientTransport` — 通过标准输入/输出与本地 MCP server 进程通信（最常见）
- `SSEClientTransport` — Server-Sent Events，适用于 HTTP 流式场景
- `StreamableHTTPClientTransport` — 用于支持流式 HTTP 的 server
- `WebSocketTransport` — WebSocket 传输（自定义实现，非 SDK 内置）

**工具封装机制：**
MCP client 从 server 获取工具列表后，将每个远程工具包装成 `MCPTool` 对象，使其与 Claude Code 内置工具（BashTool、FileReadTool 等）在接口层面完全一致。`MCPTool` 继承 `Tool` 接口，进入同一套权限检查、hook 触发、transcript 记录流程。

**大输出处理：**
`mcpValidation.ts` 中的 `mcpContentNeedsTruncation` / `truncateMcpContentIfNeeded` 防止 MCP 工具返回超大内容撑爆上下文。超出阈值的二进制内容通过 `persistBinaryContent` 持久化到磁盘，向 agent 返回引用路径而非原始内容。

**MCP Resources 作为一等公民：**
除工具外，client 还暴露 MCP Resources 和 Prompts：
- `ListMcpResourcesTool` / `ReadMcpResourceTool` 让 agent 可以发现和读取 MCP server 提供的资源
- `ListPromptsResult` 支持获取 server 定义的 prompt 模板

**OAuth 与认证：**
通过 `checkAndRefreshOAuthTokenIfNeeded` / `handleOAuth401Error` 管理 OAuth token 生命周期，`getSessionIngressAuthToken` 处理 ingress 认证场景。

**子 Agent 专属 MCP：**
`runAgent.ts` 中的 `initializeAgentMcpServers` 允许子 agent 在 frontmatter 中声明自己的 MCP server 列表，这些 server 独立于父 agent 初始化，在子 agent 退出时清理。这意味着不同 agent 可以连接不同的工具集。

### Skills — 高层工作流模板

`src/skills/loadSkillsDir.ts` 是 Skills 系统的加载引擎。Skill 的本质是带结构化 frontmatter 的 Markdown 文件，它不是"工具"，而是"经验模块"——封装了特定任务的执行模式。

**Frontmatter 字段体系：**
Skill 的 frontmatter 控制着它的所有行为特征：
- `allowed-tools` — 限定 skill 可调用的工具集（安全边界）
- `whenToUse` / `description` — 触发条件，用于自动/推荐时机判断
- `hooks` — skill 专属的 hook 配置（通过 `registerFrontmatterHooks` 注册到 hook 系统）
- `arguments` — 支持参数化（`parseArgumentNames` / `substituteArguments`），skill 可接受调用时传入参数
- `model` — 可覆盖默认模型（`parseUserSpecifiedModel`）
- `effort` — 可指定任务复杂度级别（`parseEffortValue`，对应 EFFORT_LEVELS 枚举）
- `shell` — 可指定执行 shell 类型（`parseShellFrontmatter`）

**多目录扫描：**
`loadSkillsDir` 扫描多个来源目录，优先级由高到低：
- 用户本地（`~/.claude/skills/`）
- 项目级（`.claude/skills/`）
- 插件提供（`plugin/skills/`）
- 托管/bundled skills

同名 skill 按来源优先级覆盖，保证用户本地配置最高优先。

**MCP Skill Builder：**
`registerMCPSkillBuilders` 函数允许通过 MCP 动态注册 skill builder，这是 MCP 与 Skills 层的交汇点——外部 MCP server 可以动态贡献新的 skill 能力，而不需要本地文件。

**Skill 与 Agent 的关系：**
`getSkillToolCommands` 在 `runAgent.ts` 中被调用，说明子 agent 可以使用 skill 作为可调用命令。Skill 是 agent 的"可复用工作流积木"。

### Plugins — 生态封装与分发

`src/utils/plugins/pluginLoader.ts` 是插件系统的发现、加载和管理引擎。

**Plugin 目录结构：**
```
my-plugin/
├── plugin.json          # 可选 manifest（名称、版本、描述）
├── commands/            # 自定义 slash commands（.md 文件）
├── agents/              # 自定义 AI agents（.md 文件）
└── hooks/               # Hook 配置
    └── hooks.json       # Hook 定义
```

**发现来源（优先级顺序）：**
1. **Marketplace-based plugins**：`enabledPlugins` 中 `plugin@marketplace` 格式，从注册的 marketplace 获取
2. **Session-only plugins**：通过 `--plugin-dir` CLI flag 或 SDK 的 `plugins` 选项传入，仅在当前 session 生效

**Plugin 封装的能力范围：**
- `commands/` — Slash commands，与用户本地 skills 合并（`useMergedCommands`）
- `agents/` — 自定义 agent 定义，通过 `loadAgentsDir` 加载
- `hooks/hooks.json` — Hook 配置，通过 `loadPluginHooks` 注册到全局 hook 系统

**Marketplace 管理：**
- `strictKnownMarketplaces`：策略层（企业管理）可限制只允许特定 marketplace 的插件
- `blockedMarketplaces`：可屏蔽整个 marketplace
- `ALLOWED_OFFICIAL_MARKETPLACE_NAMES`：内置官方 marketplace 白名单
- Plugin 版本缓存机制：避免每次 session 都重新从 marketplace 拉取

**启用/禁用状态管理：**
`clearPluginSettingsBase` / `getPluginSettingsBase` 管理插件的启用状态持久化。`pruneRemovedPluginHooks` 在插件被禁用/卸载时立即从运行时移除其 hook，不等待下次完整 reload。

### 三者的架构关系

```
Plugin
  ├── 包含 commands（= skills 的分发形式）
  ├── 包含 agents（= 特定任务的 agent 定义）
  └── 包含 hooks（= 注入到全局 hook 系统）
         ↓
     经由 loadPluginHooks 注册
         ↓
Skills（frontmatter-based 工作流模板）
  ├── 由 loadSkillsDir 扫描（含 plugin 来源）
  ├── 可包含 hooks（注入到 hook 系统）
  └── 可被子 agent 调用（getSkillToolCommands）
         ↓
MCP（协议层）
  ├── 由 mcp/client.ts 管理连接
  ├── tools/resources 包装为标准 Tool 接口
  └── 可通过 MCP Skill Builder 动态贡献 skills
```

### 关键代码路径

- `src/services/mcp/client.ts` — MCP 连接管理，传输层，工具/资源/prompt 获取
- `src/skills/loadSkillsDir.ts` — Skills 扫描、frontmatter 解析、参数化、hook 注册
- `src/utils/plugins/pluginLoader.ts` — Plugin 发现、manifest 验证、marketplace 管理
- `src/utils/plugins/loadPluginHooks.ts` — Plugin hooks 加载、注册、热重载
- `src/tools/MCPTool/MCPTool.ts` — MCP 工具的 Tool 接口包装
- `src/tools/AgentTool/runAgent.ts` — 子 agent 专属 MCP server 初始化

## 设计亮点

- **三层抽象不越界**：MCP 只管协议接入，Skill 只管工作流模板，Plugin 只管打包分发——三者有明确的单一职责，避免了"万物皆插件"的混乱
- **Plugin 能力的原子性管理**：Plugin 被禁用时通过 `pruneRemovedPluginHooks` 立即移除 hook，commands/agents 通过 `useMergedCommands`/`useMergedTools` 动态合并——系统总是反映当前的真实状态
- **子 Agent 专属 MCP**：每个子 agent 可以声明自己的 MCP server 集合，实现工具级别的 agent 隔离，而不需要在父 agent 层面预先注册所有可能用到的工具
- **Skill 参数化**：`arguments` 字段 + `substituteArguments` 使 skill 成为真正可复用的模板，而不是只能硬编码的脚本

## 局限性

- **MCP server 的连接状态缺乏弹性**：当 MCP server 崩溃或断连时，需要重启 session 或手动 `/reload-plugins` 才能重连，没有自动重连机制
- **Skills 和 Commands 的命名空间合并有冲突风险**：来自不同来源（用户/项目/插件）的同名 skill 按优先级静默覆盖，缺乏冲突检测和警告
- **Plugin marketplace 是中心化信任模型**：`strictKnownMarketplaces` 的存在暗示未来可能有更多 marketplace，但当前信任链仍依赖 Anthropic 的官方 marketplace
- **MCP 工具的大输出截断是单向的**：`truncateMcpContentIfNeeded` 会截断超大输出，但 agent 无法请求获取被截断的部分（只能通过持久化路径间接访问）

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
