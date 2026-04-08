---
title: MCP & Skills
aliases: [MCP, skills, 扩展协议, extension protocol]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[tool-system]]"
    type: extends
  - target: "[[hooks]]"
    type: alternative
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

外部扩展协议（MCP）+ 可复用的能力包（Skills），让 agent 能力可插拔。

## 核心问题

- MCP 和直接注册工具有什么区别？
- Skill 的粒度怎么定（一个 skill 包含多少能力）？
- 怎么发现和安装第三方扩展？
- 安全边界在哪？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | 按抽象层次拆成三种形态：MCP（协议接入层）、Skills（工作流模板层）、Plugins（生态封装层），三者分工明确互不混淆，MCP 解决"接什么工具"，Skills 解决"怎么做任务"，Plugins 解决"如何打包分发" | MCP 通过 Python `mcp` SDK 连接 stdio 服务器，每个工具包装为 `McpToolAdapter`（命名规则 `mcp__servername__toolname`）；Skills 是带可选 YAML frontmatter 的 `.md` 文件；两者通过统一 `plugin.json` 集成，兼容 Claude Code 插件生态 | `MultiServerMCPClient` 管理多服务器连接（支持 stdio/sse/http 三种 transport），引入延迟工具注册解决 MCP token 膨胀；Skills 用带 YAML frontmatter 的 Markdown 文件定义，支持 `skill_manage` 工具自动创建新 skill，实现自我进化 | **双向 MCP**：既作为 MCP Server 将自身会话暴露为 10 个标准工具（供 Claude Code/Cursor/Codex 调用），又作为 MCP Client 消费外部服务器工具；Skills 系统独立，带完整包管理器（Skills Hub）；三层完全解耦 |
| 关键特点 | MCP 工具包装为标准 Tool 接口，进入同一套权限/hook/transcript 流程；Skills 支持参数化（`arguments` + `substituteArguments`）和模型覆盖；子 agent 可声明专属 MCP server 集合实现工具级隔离 | `plugin.json` 将 skills、commands、hooks、MCP 配置聚合为单一 manifest，分发安装体验极简；`${CLAUDE_PLUGIN_ROOT}` 变量替换使插件路径可移植；Skills 纯 markdown 格式无代码门槛；复用 `.claude-plugin/plugin.json` 规范，Claude Code 插件可直接在 OpenHarness 中使用 | 延迟工具注册唯一性（分析过的所有项目中唯一将 MCP 工具按需检索的实现）；Skill 自进化（agent 完成复杂任务后自动创建/更新 skill 文件）；20 个内置 skill 覆盖多场景（PPT 生成/前端设计/数据分析）；内置 OAuth 支持 | MCP Server 使用 EventBridge（200ms 轮询 + mtime 双重检查，空闲 CPU 开销接近零）；MCP Client 专用后台 event loop + 每服务器独立 asyncio.Task + 5 次指数退避重连；Skills Hub 安全管道（隔离→正则扫描→信任分级→安装+锁文件）；SKILL.md frontmatter 支持平台过滤/条件注入/环境变量声明；77 个内置 + 45 个可选 skill |
| 局限 | MCP server 崩溃后无自动重连；Skills 和 Commands 同名按优先级静默覆盖，缺乏冲突检测；Plugin marketplace 是中心化信任模型 | 仅支持 stdio MCP 传输，不支持 HTTP/SSE/WebSocket 远程传输；Skills 无动态参数支持，每个 skill 只能执行固定逻辑；修改 `.md` 文件后需重启进程才能生效 | Skills 无结构化参数（无参数校验或自动补全）；Skill 自进化质量不可控（无评审或测试机制）；延迟工具注册增加调用轮次（需先调 `tool_search` 再调实际工具） | 审批响应是"尽力而为"（best-effort），无完整 IPC 回路；`TRUSTED_REPOS` 硬编码仅 2 个仓库；动态工具发现依赖 MCP SDK 版本特性，版本不符时退化为静态列表；Skills 无 FTS5 索引，百条以上 skill 时系统提示 token 开销显著 |

## 设计权衡

### 方案对比

| 方案 | 适用场景 | 优势 | 劣势 | 代表实现 |
|------|---------|------|------|---------|
| **A. 内置工具（Built-in only）** | 封闭系统、功能固定的专用 agent | 无外部依赖，行为完全可预测，无安全攻击面 | 每项新能力都需改代码发布，扩展成本高 | 早期 ChatGPT 插件前时代 |
| **B. 插件系统（Plugin/Extension）** | 开发者平台、需要生态的产品 | 社区贡献，能力快速扩展，无需修改核心 | 安全风险（任意代码执行），质量参差，兼容性维护成本高 | VS Code Extension、旧版 ChatGPT Plugins |
| **C. MCP 协议（标准化外部服务）** | 企业集成、跨语言工具共享 | 语言无关，进程隔离（安全），标准接口降低集成成本 | 网络/IPC 延迟，setup 复杂，协议开销，依赖外部服务可用性 | Claude Code MCP、OpenHarness MCP adapter |
| **D. Skills（运行时指令注入）** | 行为定制、工作流/风格扩展 | 零代码扩展，.md 文件即发布，可版本控制，门槛极低 | 依赖模型遵循指令，无编译期保证，复杂逻辑表达能力受限 | Claude Code Skills、OpenHarness `.md` Skills |
| **E. 双向 MCP（Server + Client 同时实现）** | 需要被其他 AI 客户端调用、同时又要调用外部工具的 agent | agent 本身成为生态节点，可被 Claude Code/Cursor/Codex 等工具直接接入；Server 侧无需额外协议适配 | 维护两条代码路径（Server 侧和 Client 侧），测试复杂度翻倍；Server 暴露的工具质量直接影响调用方 agent 体验 | Hermes Agent |
| **F. Skills Hub（去中心化 skill 包管理）** | 需要管理大量第三方 skill、有安全合规要求的团队 | 隔离-扫描-安装三段式保证第三方代码不污染主进程；信任分级（builtin/trusted/community/agent-created）可细粒度控制安装策略；lock 文件 + audit log 满足审计要求 | 扫描规则维护成本高（威胁模式库需持续更新）；信任仓库硬编码导致生态扩展受限；隔离目录引入额外的磁盘 I/O | Hermes Agent Skills Hub |

### 场景决策指南

- **封闭系统，功能稳定** → 内置工具。不引入插件/MCP 的额外复杂度，把精力放在工具实现质量上。
- **开发者平台，要建生态** → 插件 + MCP 组合：用 MCP 做标准化接入层（隔离第三方代码在独立进程），用插件打包格式（`plugin.json`）做分发体验——两者职责不混。
- **企业内部集成（数据库、内部 API、自有服务）** → MCP。进程隔离天然满足权限边界，协议标准化后不同语言写的服务都能接入，是生产环境的首选扩展方式。
- **行为定制（特定工作流、输出风格、领域规则）** → Skills。以 `.md` 文件描述工作流，按需 load，无代码门槛；但复杂逻辑（有状态、条件分支、循环）不适合用 skill，应升级为工具或 MCP。
- **子 agent 需要工具级隔离** → 为每个子 agent 声明独立 MCP server 集合，而非全局共享工具池——不同 agent 有不同的工具权限边界。
- **agent 需要被其他 AI 工具接入（Claude Code/Cursor/Codex 等）** → 实现 MCP Server 侧，将 agent 的核心能力（会话读写、事件轮询、审批响应）暴露为标准工具。EventBridge 轮询 + mtime 双检的模式是空闲时 CPU 开销最低的实现——轮询间隔 200ms 但实际 DB 读取只在文件变化时触发。
- **需要管理 skill 安全合规（团队/平台场景）** → 采用隔离-扫描-安装三段式管道：先写隔离目录，正则威胁扫描（凭证外泄/命令注入/反弹 shell/持久化植入/Unicode 注入）通过后再移入正式目录；信任分级（builtin > trusted > community > agent-created）决定对 `caution`/`dangerous` verdict 的处置策略。不要将所有来源平等对待。
- **Skills 数量增长后发现效率下降** → 引入条件式注入：`fallback_for_toolsets`（当更强工具存在时不显示）+ `requires_tools`（依赖工具不可用时不显示），保证系统提示中只出现当前上下文可用的 skill，避免 token 膨胀和模型幻觉调用。

### 常见陷阱

1. **插件无沙箱**：插件代码运行在 agent 进程内，恶意插件可以读取环境变量、窃取 API key。对策：用 MCP 替代插件做第三方集成，MCP server 跑在独立进程，主进程只通过协议通信。
2. **MCP server 挂掉 → agent 整体降级**：依赖 MCP 工具的能力在 server 不可用时全部失效，没有回退。对策：工具调用层做可用性检测，关键能力提供内置 fallback；设计时区分"核心能力"（内置）和"增强能力"（MCP），不把核心能力外包给 MCP。
3. **Skills 数量膨胀**：随着 skill 文件增多，skill 索引被注入 prompt 后占用大量上下文，挤压主任务空间。对策：按需加载（请求触发时才 load 对应 skill），而非在 system prompt 中预加载所有 skill 索引。
4. **Skills 和 Commands 同名冲突**：两者按优先级静默覆盖，开发者不易察觉。对策：在 plugin 安装时做冲突检测，命名规范中加 namespace 前缀（如 `myplugin:skill-name`）。
5. **MCP Sampling 引发工具调用递归**：MCP Server 发起 `sampling/createMessage` 时，若响应中携带工具调用，可能触发新一轮 sampling，形成无限递归。对策：在 SamplingHandler 层设置 `max_tool_rounds`（Hermes 默认 5），按实例计数强制中断，防止 token 耗尽。
6. **Skills 注入的上下文文件成为提示词注入攻击面**：`AGENTS.md`、`.cursorrules` 等被注入系统提示的文件可能携带恶意指令（Unicode 不可见字符、伪装成安全指令的越狱命令）。对策：注入前用静态威胁模式扫描（凭证外泄/命令注入/Unicode 不可见字符等），被识别为危险的文件替换为 `[BLOCKED: ...]` 占位符而非静默丢弃——保留可审计的拒绝记录。
7. **MCP Client 环境变量泄漏**：stdio 模式的 MCP Server 子进程默认继承父进程所有环境变量，可能泄漏 API Key 等机密。对策：维护安全变量白名单（仅透传 `PATH/HOME/USER/LANG` 等系统变量 + 用户显式配置的变量），任何凭证类变量都不应隐式继承。

## L2 详情

- [[mcp-skills--claude-code]]
- [[mcp-skills--openharness]]
- [[mcp-skills--deer-flow]]
- [[mcp-skills--hermes-agent]]
