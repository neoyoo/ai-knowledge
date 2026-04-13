---
pattern: tool-permission-model
category: tool-design
tags: [permission, safety, sandbox, approval, read-only, destructive]
sources: [claude-code, deer-flow, openharness]
related_wiki: [wiki/tool-system]
related_patterns: [tool-design-principles, error-handling]
---

# Tool Permission Model

> 工具权限分级设计——哪些工具可以自动执行、哪些需要用户确认、哪些绝对禁止，保障 agent 安全运行。

## 本质

给 agent 工具就是给它权力。给对了，agent 能干事；给错了，agent 能删库、泄露数据、绕过安全限制。

权限模型的核心问题是：**agent 应该被允许在没有人监督的情况下做什么？**

这不是技术问题，是信任问题——你对这个 agent 的信任程度，决定了给它多少自主权。好的权限模型让你能精确表达这个信任边界，而不是"全开放"或"全锁死"二选一。

## 权限三级分类

| 级别 | 操作类型 | 执行方式 | 典型工具 |
|------|---------|---------|---------|
| **L1 自动执行** | 只读，不改变任何状态 | 直接执行，无需确认 | read_file、search_web、list_directory、get_file_info |
| **L2 需确认** | 写操作，有可逆性 | 显示预览，用户确认后执行 | write_file、edit_file、create_directory |
| **L3 高风险确认** | 不可逆或影响范围大 | 高亮风险，明确用户意图后执行 | delete_file、execute_bash、network_request（外部） |
| **禁止** | 绝对不执行 | 直接拒绝，返回错误 | 修改权限配置、自我提权、访问禁止路径 |

**分级依据不是"危险程度"，而是"可逆性"和"影响范围"。**

文件修改可能被 git 恢复，所以是 L2 而不是 L3；删除文件难以恢复，所以是 L3；修改 agent 自己的权限配置，影响的是整个安全模型，所以直接禁止。

## Claude Code 的权限模型

Claude Code 是权限系统设计最完整的参考实现，值得深入理解。

### 三种运行模式

`src/utils/permissions/PermissionMode.ts` 定义了五种权限模式，日常使用主要是三种：

```typescript
enum PermissionMode {
    Plan = "plan",              // 只能查看文件、制定计划，不执行任何写操作
    Default = "default",        // 读操作自动执行，写操作需要用户确认（日常推荐）
    AcceptEdits = "acceptEdits", // 读写自动执行，高风险操作需确认（进阶模式）
    BypassPermissions = "bypass", // 全部自动执行，用于脚本/CI（危险，慎用）
    Auto = "auto"               // 内置分类器决策，低风险自动放行，高风险走确认
}
```

**使用建议：**
- 交互式开发 → `Default`（有提示，但不烦人）
- 代码审查、探索 → `Plan`（只读，零风险）
- 批量自动化任务 → `AcceptEdits`（写操作不再阻塞，但高风险仍确认）
- CI/CD 流水线 → `BypassPermissions`（全自动，配合沙箱隔离使用）
- 不确定时 → `Auto`（让分类器判断）

### checkPermissions 流程

每次工具调用前，`useCanUseTool()` 执行权限检查：

```typescript
// 简化版流程（实际在 src/hooks/useCanUseTool.tsx）
async function checkPermissions(
    tool: Tool,
    args: ToolInput,
    context: ToolUseContext
): Promise<PermissionResult> {

    // 只读工具 + Default 模式 → 直接放行
    if (tool.isReadOnly() && context.mode !== "plan") {
        return { approved: true, source: "auto" };
    }

    // 高风险工具（bash、网络、文件删除）→ 无论什么模式都提示
    if (tool.isHighRisk(args)) {
        return await context.requestUserApproval({
            preview: tool.getPreview(args),
            riskDescription: tool.getRiskDescription(args)
        });
    }

    // 根据运行模式决策
    switch (context.mode) {
        case "plan":          return { approved: false, reason: "Plan 模式不允许写操作" };
        case "acceptEdits":   return { approved: true, source: "mode" };
        case "bypass":        return { approved: true, source: "bypass" };
        case "auto":          return await classifierApproval(tool, args);
        default:              return await requestUserApproval(tool, args);
    }
}
```

### allowedTools 白名单

`settings.json` 支持配置工具白名单，只有白名单内的工具才能执行：

```json
{
    "permissions": {
        "allowedTools": ["read_file", "list_directory", "search_web"],
        "deniedTools": ["delete_file", "execute_bash"]
    }
}
```

白名单模式适合限制 agent 的能力范围，例如"这个 agent 只能查阅，不能修改"。

### OS 级沙箱

`src/utils/sandbox/sandbox-adapter.ts` 在权限审批之外增加一层 OS 级防线：

```typescript
// 沙箱配置示例
const sandboxPolicy = {
    filesystem: {
        readPaths: ["/home/user/project"],   // 只允许读这些路径
        writePaths: ["/home/user/project"],  // 只允许写这些路径
        deniedPaths: ["/etc", "/usr", "~/.ssh"]  // 绝对禁止
    },
    network: {
        allowedDomains: ["api.github.com", "registry.npmjs.org"],
        deniedDomains: ["*"]  // 默认拒绝所有，白名单放行
    }
};
```

**关键保护项目：**
- `.claude/settings.json` 写保护——防止 agent 通过修改设置文件给自己提权
- `.claude/skills` 目录写保护——技能目录是高权限扩展面，不能被工具修改
- git bare repo 逃逸防御——防止通过 git 操作绕出沙箱路径限制

这展示了一个重要原则：**安全不是"检查 agent 的意图"，而是在 OS 层面硬限制它能触达的资源范围**。即使 agent 被 prompt injection 操控，沙箱也能阻止实际的系统损害。

## DeerFlow 的权限模型

DeerFlow 的权限控制基于 `GuardrailMiddleware`，与工具调度完全解耦，是更接近中间件模式的设计。

### 护栏分类

```python
# 工具按权限分组（config.yaml）
tools:
    bash:
        enabled: true
        sandbox: true         # bash 工具强制走沙箱
        allowed_commands:     # 命令白名单
            - "ls"
            - "cat"
            - "grep"
            - "python"

    file_read:
        enabled: true
        sandbox: false        # 读文件不需要沙箱

    file_write:
        enabled: true
        sandbox: false
        allowed_paths:        # 只允许写这些路径
            - "/tmp"
            - "./output"

    network:
        enabled: true
        allowed_domains:      # 域名白名单
            - "arxiv.org"
            - "github.com"
```

### GuardrailMiddleware

```python
# 每次工具调用前拦截（deerflow/agents/middlewares/guardrail.py）
class GuardrailMiddleware:
    def __init__(self, guardrail_provider: GuardrailProvider):
        self.provider = guardrail_provider

    async def before_tool_call(
        self,
        tool_name: str,
        arguments: dict,
        context: AgentContext
    ) -> GuardrailResult:
        result = await self.provider.evaluate(
            tool_name=tool_name,
            arguments=arguments,
            agent_context=context
        )
        return result  # allow / deny + reason

    async def after_tool_call(
        self,
        tool_name: str,
        tool_result: str,
        context: AgentContext
    ) -> str:
        # 可以对工具返回内容做脱敏或过滤
        return self.provider.filter_output(tool_result)
```

DeerFlow 护栏的最大优点是**正交性**：安全逻辑与调度逻辑完全分离，可以独立替换护栏实现（从规则型换成 LLM 语义评估型）而不影响工具注册和执行逻辑。

### 沙箱隔离

```python
# is_host_bash_allowed 控制 bash 工具在宿主还是沙箱执行
def is_host_bash_allowed() -> bool:
    return os.getenv("DEERFLOW_ALLOW_HOST_BASH", "false") == "true"

# 默认走沙箱（容器/虚拟路径隔离）
if not is_host_bash_allowed():
    return await execute_in_sandbox(command, virtual_path=working_dir)
else:
    return await execute_on_host(command)
```

默认策略是 fail-safe：bash 命令默认走沙箱，需要明确配置才能在宿主执行。

## OpenHarness 的权限模型

OpenHarness 的权限设计最轻量，以 `is_read_only()` 为核心接口：

```python
# tools/base.py
class BaseTool:
    def is_read_only(self, arguments: dict) -> bool:
        """
        声明此次调用是否只读。
        子类必须覆盖此方法，默认返回 False（保守策略）。
        """
        return False  # fail-closed 默认

# 具体工具声明自己的读写语义
class ReadFileTool(BaseTool):
    def is_read_only(self, arguments: dict) -> bool:
        return True  # 纯读操作

class WriteFileTool(BaseTool):
    def is_read_only(self, arguments: dict) -> bool:
        return False  # 写操作

class EditFileTool(BaseTool):
    def is_read_only(self, arguments: dict) -> bool:
        # 参数级判断：如果是 dry_run 模式，视为只读
        return arguments.get("dry_run", False)
```

权限检查在 `_execute_tool_call()` 中集中处理：

```python
async def _execute_tool_call(self, tool_name: str, arguments: dict):
    tool = self.registry.get(tool_name)

    # 验证参数
    validated = tool.input_model.model_validate(arguments)

    # 权限检查
    if not tool.is_read_only(arguments) and self.mode == "read_only":
        return ToolResult(
            content="当前模式只允许只读操作。",
            is_error=True
        )

    return await tool.execute(validated)
```

OpenHarness 的权限系统虽然简单，但体现了一个核心原则：**权限语义下沉到工具层**，每个工具自己声明读写属性，调度层无需了解工具语义。

## Subagent 权限继承

多 Agent 系统里，权限应该**向下单调递减**，绝不向上扩展。

```text
用户设定：工作目录 = /home/user/project，模式 = default

父 Agent 权限：
  - 读：/home/user/project/**
  - 写：/home/user/project/**（需确认）

Subagent 应该获得的权限（最小权限原则）：
  - 读：/home/user/project/src/**（更窄的范围）
  - 写：/home/user/project/src/**（需确认）
  - 禁止：/home/user/project/.git/**（不暴露 git 操作）
```

**绝不允许 Subagent 的权限超过父 Agent**，即使 prompt 里要求了也不行。Subagent 可以请求父 Agent 代为执行高权限操作，但不能直接获取高于自身等级的权限。

Claude Code 的 multi-agent 设计体现了这个原则：Subagent 的工具集是父 Agent 工具集的子集，由父 Agent 在派发时显式指定。

## 审计日志

所有工具调用都应该可追溯：

```python
# 最小审计日志
@dataclass
class ToolCallLog:
    timestamp: datetime
    agent_id: str
    tool_name: str
    arguments: dict
    result_status: str   # "success" | "error" | "denied"
    result_summary: str  # 不记录完整输出（可能包含敏感数据）
    permission_mode: str
    approved_by: str     # "auto" | "user" | "classifier"

# 权限拒绝必须记录（安全审计的核心需求）
async def log_permission_denial(tool_name, arguments, reason):
    await audit_logger.write(ToolCallLog(
        tool_name=tool_name,
        arguments=sanitize(arguments),  # 脱敏
        result_status="denied",
        result_summary=reason,
        ...
    ))
```

**审计日志的价值：**
- 安全事故后复盘：agent 做了什么、什么时候做的
- 权限调优：哪些操作频繁需要确认 → 考虑提升该工具的信任级别
- 异常检测：非正常时间的高权限操作、异常频率的工具调用

## 三个项目权限模型对比

| 维度 | Claude Code | OpenHarness | DeerFlow |
|------|------------|-------------|----------|
| 核心抽象 | PermissionMode 枚举 + checkPermissions() | is_read_only() 方法 | GuardrailMiddleware |
| 权限粒度 | 工具级 + 参数级（bash 命令白名单） | 工具级 | 工具类型级 + 路径级 |
| 运行模式 | 5 种（plan/default/acceptEdits/bypass/auto） | 2 种（read_only/full） | 2 种（sandboxed/host） |
| 沙箱隔离 | OS 级（文件路径 + 网络域名 + 命令）| 无 | 容器/虚拟路径 |
| 默认策略 | Fail-closed（未声明 → 需确认） | Fail-closed（未声明 → 非只读） | Fail-safe（bash 默认沙箱） |
| 审计日志 | 有（工具调用记录） | 无 | 有（GuardrailMiddleware 记录） |
| 用户确认 UI | 交互式弹窗 | 返回 is_error | 返回 deny + reason |

## 最小权限原则 Checklist

设计工具权限时的自查清单：

```text
□ 只读工具是否标记了 is_read_only / isConcurrencySafe?
□ 写操作是否需要用户确认（至少在 default 模式下）?
□ 破坏性操作（删除、覆盖、外部请求）是否有高风险标记?
□ 禁止操作列表是否明确（配置文件修改、提权操作）?
□ Subagent 的权限是否 ≤ 父 Agent?
□ 是否有沙箱限制文件系统访问范围（即使只是路径过滤）?
□ 是否有审计日志记录所有工具调用和拒绝记录?
□ CI/CD 全自动模式是否配合了沙箱隔离（不要裸 bypass）?
□ settings/config 文件是否有写保护（防止 agent 自我提权）?
```

## 常见踩坑

**1. 开发时全开放，上线忘了收回权限**

"本地测试嘛，bypass 模式方便点"——上线时没改回 default，agent 在生产环境里全自动执行，没有人工确认，一次 prompt injection 就出大事。权限模式应该是环境配置，不是代码里的常量。

**2. 只读/写的分类过于粗糙**

`bash` 工具统一视为"写操作"——但 `ls`、`cat`、`grep` 是只读的，`rm`、`chmod` 才是危险的。粗糙分类导致所有 bash 调用都需要确认，用起来极度繁琐。Claude Code 的 bash 分类器就是解决这个问题的。

**3. 沙箱只限制路径，忘了网络**

沙箱配置了文件系统访问范围，但 agent 可以通过 `curl` 或 `requests` 访问任意外部地址，绕过文件系统限制直接泄露数据。完整的沙箱必须同时限制网络访问。

**4. Subagent 权限继承没有显式约束**

父 Agent 把全部工具列表传给 Subagent，Subagent 拿到和父 Agent 相同的权限。当 Subagent 被恶意 prompt 操控时，危害范围等同于父 Agent。

**5. 没有审计日志就去排查安全事故**

"agent 好像删了什么东西"——没有工具调用日志，完全没法知道发生了什么。审计日志不是可选的，是发现问题时唯一的调查线索。

**6. 权限绕过通过配置文件修改**

Agent 有写文件权限，写了 `.claude/settings.json` 修改自己的权限模式。Claude Code 专门对 settings 文件加了写保护就是防这个。不要认为"agent 不会想到这样做"——通过 prompt injection 输入的攻击向量经常走这条路。

## 来源

- **Claude Code 2.1.88** — `src/utils/permissions/PermissionMode.ts`：五种权限模式枚举；`src/hooks/useCanUseTool.tsx`：三路审批分发（interactive/coordinator/swarm）；`src/utils/sandbox/sandbox-adapter.ts`：OS 级沙箱策略，settings 文件保护、git bare repo 逃逸防御
- **OpenHarness 0.1.0** — `tools/base.py`：`is_read_only()` 工具自声明权限接口，fail-closed 默认策略，`_execute_tool_call()` 权限前置检查路径
- **DeerFlow 2.0** — `deerflow/guardrails/builtin.py`：GuardrailProvider 正交安全层；`config.yaml`：工具分组 + 路径白名单 + 命令白名单配置；`is_host_bash_allowed()` 沙箱开关

关联 wiki：[[wiki/tool-system]] · [[wiki/multi-agent]]
关联模式：[[tool-design-principles]] · [[error-handling]]
