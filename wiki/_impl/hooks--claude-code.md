---
title: "Hooks System — Claude Code"
category: L2
parent: "[[hooks]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的 Hooks 系统允许用户通过 shell 命令在 agent 生命周期的特定事件点介入执行流程。Hook 不只是简单的 callback，它拥有完整的类型化事件系统（28+ 事件类型）、同步/异步执行模式、权限决策能力、以及通过 JSON 协议与 agent 双向通信的机制。Hook 来源可以是用户配置（settings.json）、技能（skills frontmatter）或插件（plugins），三者通过统一的注册系统合并管理。

## 架构分析

### 事件类型系统 — 完整的生命周期覆盖

`src/types/hooks.ts` 和 `src/utils/plugins/loadPluginHooks.ts` 中定义了完整的 `HookEvent` 枚举，涵盖 28+ 事件：

**工具调用生命周期**
- `PreToolUse` — 工具调用前，可修改输入、批准或拒绝执行
- `PostToolUse` — 工具调用成功后，可修改 MCP 工具输出
- `PostToolUseFailure` — 工具调用失败后

**会话生命周期**
- `Setup` — 会话初始化阶段（早于 SessionStart）
- `SessionStart` — 会话开始（含 resume/clear/compact 触发）
- `SessionEnd` — 会话结束
- `Stop` / `StopFailure` — agent 正常/异常停止

**用户交互**
- `UserPromptSubmit` — 用户提交 prompt 时，可追加额外上下文
- `Notification` — agent 发送通知时
- `Elicitation` / `ElicitationResult` — 结构化信息收集的请求和结果

**权限控制**
- `PermissionRequest` — 权限请求时，hook 可做 allow/deny 决策
- `PermissionDenied` — 权限被拒绝后，支持 retry 标志

**多智能体**
- `SubagentStart` / `SubagentStop` — 子 agent 的启动和停止
- `TeammateIdle` — Teammate 进入空闲状态
- `TaskCreated` / `TaskCompleted` — 任务创建和完成

**上下文压缩**
- `PreCompact` / `PostCompact` — 上下文压缩前后

**环境变化**
- `CwdChanged` — 工作目录变更
- `FileChanged` — 监听文件变更（配合 `watchPaths` 使用）
- `WorktreeCreate` / `WorktreeRemove` — Git worktree 管理
- `InstructionsLoaded` — 指令文件加载完成
- `ConfigChange` — 配置变更

### Hook 通信协议 — 双向 JSON 交互

Hook 通过 stdout JSON 与 agent 通信，支持两种模式（在 `src/types/hooks.ts` 中用 Zod schema 严格定义）：

**同步模式**（`SyncHookJSONOutput`）
- `continue: boolean` — 是否允许 agent 继续执行
- `suppressOutput: boolean` — 是否隐藏 hook 的 stdout 输出
- `stopReason: string` — 当 `continue=false` 时展示的消息
- `decision: "approve" | "block"` — 对工具调用的显式决策
- `systemMessage: string` — 向用户展示的警告消息
- `hookSpecificOutput` — 事件特定的输出（按 `hookEventName` 区分）

**异步模式**（`AsyncHookJSONOutput`）
- `async: true` + 可选 `asyncTimeout` — 告知 runtime 这是异步 hook，等待后续结果

**事件特定能力举例：**
- `PreToolUse`：`permissionDecision`（allow/deny/ask）、`updatedInput`（修改工具输入）、`additionalContext`
- `SessionStart`：`initialUserMessage`（注入初始消息）、`watchPaths`（注册文件监听）
- `PostToolUse`：`updatedMCPToolOutput`（修改 MCP 工具的输出）
- `PermissionRequest`：`behavior: "allow"` 带 `updatedPermissions` 或 `behavior: "deny"` 带 interrupt

### 执行引擎 — utils/hooks.ts

`src/utils/hooks.ts` 是 hook 执行的核心，注释第一行即说明：

> "Hooks are user-defined shell commands that can be executed at various points in Claude Code's lifecycle."

执行机制：
- 通过 `spawn` 启动 shell 子进程（支持 bash/zsh/powershell，通过 `DEFAULT_HOOK_SHELL` 配置）
- Hook 进程通过环境变量接收上下文（`CLAUDE_HOOK_*` 系列），通过 `getHookEnvFilePath` 指向 env 文件
- 通过 `subprocessEnv` 传递安全的环境变量子集
- 支持 `asyncTimeout`：通过 `registerPendingAsyncHook` 注册异步 hook，等待结果
- 执行过程通过 `emitHookStarted` / `emitHookResponse` / `startHookProgressInterval` 发出 UI 进度事件

**toolHooks.ts 中的工具 hook 执行流程：**
- `runPreToolUseHooks` — 异步生成器，依次执行所有匹配的 PreToolUse hook，yield 权限决策、输入修改、或停止信号
- `runPostToolUseHooks` — 执行 PostToolUse hook，处理 blocking error、continuation 控制、MCP 输出替换
- `resolveHookPermissionDecision` — 将 hook 的权限结果与 settings.json 规则合并（hook allow 不能绕过 deny 规则）

### 权限决策的分层设计

在 `src/services/tools/toolHooks.ts` 的 `resolveHookPermissionDecision` 中体现了一个重要设计原则：

**hook allow 不能绕过 settings.json deny 规则**。即：
1. Hook 返回 `allow` → 跳过用户交互弹窗 → 但仍执行 `checkRuleBasedPermissions`
2. 如果 settings.json 中有 deny 规则 → deny 规则胜出
3. 如果 settings.json 中有 ask 规则 → 仍需用户确认

这防止了通过恶意 hook 绕过安全策略的攻击面。

### Hook 注册系统 — 三来源合并

Hook 注册通过 `src/bootstrap/state.ts` 中的 `registerHookCallbacks` / `getRegisteredHooks` 统一管理，来源有三：

1. **用户配置（settings.json）**：通过 `getHooksConfigFromSnapshot` 读取，支持 `matcher` 匹配特定工具名
2. **Skills frontmatter**：`loadSkillsDir.ts` 解析 skill 文件中的 hooks 字段，通过 `registerFrontmatterHooks` 注册
3. **Plugins**：`loadPluginHooks.ts` 扫描启用插件的 hooks 目录（`hooks/hooks.json`），通过 `registerHookCallbacks` 注入

Plugin hooks 支持热重载（`setupPluginHookHotReload`）：监听 policy settings 变化，当 marketplace 配置或 enabledPlugins 变化时原子性地 clear + re-register。

### 托管 Hook 与安全策略

存在 `shouldAllowManagedHooksOnly()` 和 `shouldDisableAllHooksIncludingManaged()` 两个策略控制：
- `allowManagedHooksOnly`：只允许经过认证的 "managed" hook（来自官方 marketplace 的插件）运行，用户自定义 hook 被跳过
- `disableAllHooks`：完全禁用所有 hook，包括 managed hook

这是面向企业/受管设备场景的 hook 执行控制。

### 关键代码路径

- `src/utils/hooks.ts` — Hook 执行引擎主文件（shell 进程 spawn，协议解析，超时管理）
- `src/types/hooks.ts` — Hook 类型定义（事件枚举、响应 schema、HookResult 类型）
- `src/services/tools/toolHooks.ts` — 工具级 hook runner（PreToolUse/PostToolUse 生成器，权限决策）
- `src/utils/plugins/loadPluginHooks.ts` — Plugin hook 加载与热重载
- `src/utils/sessionStart.ts` — SessionStart hook 的触发时机与执行
- `src/entrypoints/agentSdkTypes.ts` — 公开的 SDK hook 类型（HookEvent、HookInput 等）
- `src/utils/hooks/hookEvents.ts` — Hook 执行过程的 UI 进度事件发射

## 设计亮点

- **Zod schema 双向验证**：Hook 响应通过 `hookJSONOutputSchema` 严格校验，compile-time 断言保证 SDK 类型与 Zod schema 同步（`_assertSDKTypesMatch`），确保协议定义单一来源
- **异步 hook 不阻塞主循环**：`async: true` 模式允许 hook 通知 agent "我在异步处理"，agent 可继续其他工作，通过 `AsyncHookRegistry` 追踪待完成的异步 hook
- **原子性插件 hook 交换**：clear-then-register 作为原子对，解决了 plugin uninstall 后 Stop hook 沉默触发的 bug（gh-29767）
- **watchPaths 文件监听**：SessionStart hook 可返回 `watchPaths`，触发 `FileChanged` 事件链，将文件系统变化纳入 agent 事件循环

## 局限性

- **Hook 隔离性依赖 shell**：Hook 本质是 shell 子进程，恶意 hook 仍可访问文件系统和环境变量（`allowManagedHooksOnly` 是软性防护）
- **异步模式的超时机制较粗糙**：`asyncTimeout` 是可选的单一超时值，没有细粒度的重试或 backoff 机制
- **Hook 输出的 append-only**：`additionalContext` 只能追加内容到上下文，不能替换已有内容（PostToolUse 的 `updatedMCPToolOutput` 是例外）
- **Matcher 只匹配工具名字符串**：`PreToolUse`/`PostToolUse` 的 matcher 只能匹配工具名，不支持对工具参数的条件匹配

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
