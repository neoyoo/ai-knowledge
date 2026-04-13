---
tool-category: agent-orchestration
tags: [agent, subagent, task, delegation, multi-agent, coordination]
sources: [claude-code, deer-flow, openharness]
---

# Agent Orchestration Tools

> 子 Agent 派发、任务生命周期管理、团队协作的工具集——多 Agent 系统的基础设施层。

## 本质

多 Agent 系统不是"让模型自由聊天"，而是在正式的任务系统上运行一套**有生命周期、可追踪、可恢复**的派发协议。

这一类工具解决的核心问题：Orchestrator 怎么把任务交出去、怎么追踪进展、怎么在任务完成后拿回结果。每个工具对应任务生命周期的一个操作节点。

## 工具总览

| 工具名 | 来源 | 核心作用 |
|--------|------|----------|
| `Agent` / `agent` / `task` | Claude Code / OpenHarness / DeerFlow | 派发子 Agent 任务 |
| `TaskCreate` / `task_create` | Claude Code / OpenHarness | 创建后台任务 |
| `TaskList` / `task_list` | Claude Code / OpenHarness | 列出所有任务 |
| `TaskGet` / `task_get` | Claude Code / OpenHarness | 获取任务详情 |
| `TaskOutput` / `task_output` | Claude Code / OpenHarness | 读取任务输出 |
| `TaskStop` / `task_stop` | Claude Code / OpenHarness | 停止任务 |
| `TaskUpdate` / `task_update` | Claude Code / OpenHarness | 更新任务状态（OpenHarness 独有 progress 字段） |
| `SendMessage` / `send_message` | Claude Code / OpenHarness | 向运行中的 Agent 发消息 |
| `TeamCreate` / `team_create` | Claude Code / OpenHarness | 创建 Agent 团队 |
| `TeamDelete` / `team_delete` | Claude Code / OpenHarness | 删除团队 |

---

## 工具 Schema（Anthropic tool_use 格式）

以下 schema 可直接复制到你的 agent 工具定义数组中。

### Agent — 派发子 Agent（Claude Code）

最完整的子 Agent 派发接口：支持模型覆盖、隔离模式、后台运行。

```json
{
  "name": "Agent",
  "description": "Launch a subagent to handle an independent subtask. The subagent runs in an isolated context with its own tools and transcript. Use for tasks that can be completed independently or in parallel.",
  "input_schema": {
    "type": "object",
    "properties": {
      "prompt": {
        "type": "string",
        "description": "The task description and instructions for the subagent. Be specific about deliverables and constraints."
      },
      "description": {
        "type": "string",
        "description": "Short display label for this agent task (shown in UI)."
      },
      "subagent_type": {
        "type": "string",
        "description": "The type of subagent to launch. Determines the system prompt and tool set.",
        "enum": ["general", "code", "research", "coordinator"]
      },
      "model": {
        "type": "string",
        "description": "Override the default model for this subagent. e.g. 'claude-opus-4-5' for complex reasoning, 'claude-haiku-4-5' for simple tasks."
      },
      "run_in_background": {
        "type": "boolean",
        "description": "If true, returns a task_id immediately and runs asynchronously. If false, blocks until completion.",
        "default": false
      },
      "cwd": {
        "type": "string",
        "description": "Working directory for the subagent. Defaults to current directory. Use a worktree path for full isolation."
      }
    },
    "required": ["prompt", "description"]
  }
}
```

---

### agent — 派发子 Agent（OpenHarness）

OpenHarness 的轻量版本：以进程为隔离单元，支持 team 归属和执行模式选择。

```json
{
  "name": "agent",
  "description": "Spawn a subprocess agent to handle an independent task. Each agent runs as a separate process with full isolation.",
  "input_schema": {
    "type": "object",
    "properties": {
      "prompt": {
        "type": "string",
        "description": "Task description and instructions sent to the subagent via stdin."
      },
      "description": {
        "type": "string",
        "description": "Human-readable label for this agent task."
      },
      "model": {
        "type": "string",
        "description": "Model to use for this subagent. Defaults to the global model setting."
      },
      "team": {
        "type": "string",
        "description": "Team name to assign this agent to. Creates the team if it doesn't exist."
      },
      "mode": {
        "type": "string",
        "description": "Execution mode for the agent.",
        "enum": ["local_agent", "remote_agent", "in_process_teammate"],
        "default": "local_agent"
      },
      "command": {
        "type": "string",
        "description": "Optional override for the agent entrypoint command."
      }
    },
    "required": ["prompt", "description"]
  }
}
```

---

### task — 派发子 Agent（DeerFlow）

DeerFlow 的子 Agent 接口：通过线程池真并行，内置并发上限（最多 3 个并发）。

```json
{
  "name": "task",
  "description": "Dispatch a subtask to a subagent executor. Subagents run in parallel via a thread pool. Maximum 3 concurrent task calls per model response.",
  "input_schema": {
    "type": "object",
    "properties": {
      "description": {
        "type": "string",
        "description": "Brief description of the subtask (used for progress tracking)."
      },
      "prompt": {
        "type": "string",
        "description": "Detailed instructions for the subagent."
      },
      "subagent_type": {
        "type": "string",
        "description": "Subagent configuration to use.",
        "enum": ["general-purpose", "bash"],
        "default": "general-purpose"
      },
      "max_turns": {
        "type": "integer",
        "description": "Maximum number of reasoning turns for the subagent. Prevents runaway loops.",
        "default": 10
      }
    },
    "required": ["description", "prompt"]
  }
}
```

---

### TaskCreate / task_create — 创建后台任务

```json
{
  "name": "task_create",
  "description": "Create a background task from a prompt. Returns a task_id for tracking. Equivalent to Agent with run_in_background=true.",
  "input_schema": {
    "type": "object",
    "properties": {
      "prompt": {
        "type": "string",
        "description": "Task description and instructions."
      },
      "description": {
        "type": "string",
        "description": "Human-readable label for the task."
      },
      "model": {
        "type": "string",
        "description": "Model to use. Omit to use default."
      }
    },
    "required": ["prompt", "description"]
  }
}
```

---

### TaskList / task_list — 列出任务

```json
{
  "name": "task_list",
  "description": "List all active and recent background tasks with their status.",
  "input_schema": {
    "type": "object",
    "properties": {
      "status_filter": {
        "type": "string",
        "description": "Filter tasks by status.",
        "enum": ["running", "completed", "failed", "all"],
        "default": "all"
      }
    },
    "required": []
  }
}
```

---

### TaskGet / task_get — 获取任务详情

```json
{
  "name": "task_get",
  "description": "Get detailed status and metadata for a specific task by ID.",
  "input_schema": {
    "type": "object",
    "properties": {
      "task_id": {
        "type": "string",
        "description": "The task ID returned by task_create or Agent with run_in_background=true."
      }
    },
    "required": ["task_id"]
  }
}
```

---

### TaskOutput / task_output — 读取任务输出

```json
{
  "name": "task_output",
  "description": "Read the output produced by a task. For completed tasks, returns full output. For running tasks, returns output so far.",
  "input_schema": {
    "type": "object",
    "properties": {
      "task_id": {
        "type": "string",
        "description": "The task ID to read output from."
      },
      "offset": {
        "type": "integer",
        "description": "Byte offset to read from (for incremental polling). Default 0 reads from beginning.",
        "default": 0
      }
    },
    "required": ["task_id"]
  }
}
```

---

### TaskStop / task_stop — 停止任务

```json
{
  "name": "task_stop",
  "description": "Stop a running background task. The task's partial output remains readable via task_output.",
  "input_schema": {
    "type": "object",
    "properties": {
      "task_id": {
        "type": "string",
        "description": "The task ID to stop."
      },
      "reason": {
        "type": "string",
        "description": "Optional reason for stopping (logged for observability)."
      }
    },
    "required": ["task_id"]
  }
}
```

---

### TaskUpdate / task_update — 更新任务（OpenHarness 独有 progress 字段）

```json
{
  "name": "task_update",
  "description": "Update task metadata. OpenHarness extends this with a progress field for numeric progress tracking (0-100).",
  "input_schema": {
    "type": "object",
    "properties": {
      "task_id": {
        "type": "string",
        "description": "The task ID to update."
      },
      "status": {
        "type": "string",
        "description": "New status for the task.",
        "enum": ["running", "paused", "completed", "failed"]
      },
      "progress": {
        "type": "integer",
        "description": "[OpenHarness only] Numeric progress percentage (0-100).",
        "minimum": 0,
        "maximum": 100
      },
      "metadata": {
        "type": "object",
        "description": "Arbitrary key-value metadata to attach to the task.",
        "additionalProperties": true
      }
    },
    "required": ["task_id"]
  }
}
```

---

### SendMessage / send_message — 向 Agent 发消息

```json
{
  "name": "send_message",
  "description": "Send a message to a running background agent. Used to provide feedback, inject new information, or redirect the agent mid-task.",
  "input_schema": {
    "type": "object",
    "properties": {
      "task_id": {
        "type": "string",
        "description": "The target task/agent ID."
      },
      "message": {
        "type": "string",
        "description": "Message content to send. The agent receives this on its next turn."
      }
    },
    "required": ["task_id", "message"]
  }
}
```

---

### TeamCreate / team_create — 创建团队

```json
{
  "name": "team_create",
  "description": "Create a named team to group related agents. Teams allow batch operations and coordinated messaging.",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string",
        "description": "Team name. Must be unique within the session."
      },
      "description": {
        "type": "string",
        "description": "Purpose and coordination strategy for this team."
      }
    },
    "required": ["name"]
  }
}
```

---

### TeamDelete / team_delete — 删除团队

```json
{
  "name": "team_delete",
  "description": "Delete a team. Running agents in the team continue running but lose team affiliation.",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string",
        "description": "Team name to delete."
      }
    },
    "required": ["name"]
  }
}
```

---

## 跨项目对比

### 隔离策略

| 维度 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| **隔离单元** | worktree（文件系统级）或同进程 | 线程池（asyncio 独立事件循环） | 独立子进程（OS 级） |
| **上下文继承** | 可选继承父 agent context | 继承 sandbox 和 thread_data | 无，只传 prompt 字符串 |
| **工具集** | 子 agent 可声明独立 MCP servers | 不含 task 工具（禁止嵌套） | 完整工具集，无限制 |
| **隔离强度** | 中（可共享也可隔离） | 低（线程共享内存空间） | 高（进程边界，无共享内存） |

### 模型选择能力

| 项目 | 模型覆盖 | 粒度 |
|------|----------|------|
| Claude Code | 支持，`model` 参数 | 每个子 Agent 独立指定 |
| DeerFlow | 受限，由 `subagent_type` 的预设配置决定 | 配置级 |
| OpenHarness | 支持，`model` 参数 | 每个子 Agent 独立指定 |

### 后台执行 vs 同步

| 项目 | 默认模式 | 后台执行 | 同步执行 |
|------|----------|----------|----------|
| Claude Code | 同步（`run_in_background: false`） | 支持，返回 task_id | 支持，阻塞到完成 |
| DeerFlow | 异步（总是后台） | 总是后台，主 agent 通过 stream 事件接收进度 | 不支持 |
| OpenHarness | 后台（subprocess） | 支持，返回 task_id | 不支持，所有子 agent 都是后台 |

### 任务生命周期管理

| 能力 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| 创建 | Agent（blocking）/ TaskCreate（async） | task | agent |
| 列出 | TaskList | 通过 stream 事件感知 | task_list |
| 获取详情 | TaskGet | 无显式 API | task_get |
| 读输出 | TaskOutput | stream 事件 | task_output |
| 停止 | TaskStop | 不支持 | task_stop |
| 更新状态 | 有限 | 无 | TaskUpdate（含 progress%） |
| 发消息 | SendMessage（mailbox 机制） | 无 | send_message（stdin 写入） |
| 持久化 | 支持（磁盘 task 状态） | 内存，无持久化 | 内存，无持久化 |

---

## 最佳实践

### 1. 独立任务才并行，有依赖的串行

并行的前提是任务之间**没有共享状态、没有顺序依赖**。

```python
# 正确：三个独立分析任务并行
task_ids = []
for topic in ["performance", "security", "architecture"]:
    result = agent_tool.call({
        "prompt": f"Analyze {topic} aspects of the codebase",
        "description": f"{topic} analysis",
        "run_in_background": True
    })
    task_ids.append(result["task_id"])

# 等待所有完成后再汇总
outputs = [task_output.call({"task_id": tid}) for tid in task_ids]

# 错误：有依赖的任务不能并行
# Step 2 依赖 Step 1 的输出，必须串行
```

### 2. 给子 Agent 明确的 scope 和约束

模糊的任务描述会让子 Agent 做出超出预期的操作。Prompt 里要明确说明：可以做什么、不能做什么、产出格式是什么。

```text
# 好的 prompt
"分析 src/auth/ 目录下的认证实现，重点关注：
1. token 验证逻辑（只读，不要修改代码）
2. 潜在的安全漏洞
3. 与 OWASP Top 10 的对照

输出格式：JSON，包含 findings[] 数组，每项有 severity/location/description 字段"

# 差的 prompt
"看一下认证模块，说说有什么问题"
```

### 3. 设置 max_turns 防止失控

对于 DeerFlow 的 task 工具，始终显式设置 `max_turns`。对于 Claude Code 和 OpenHarness，用 TaskStop 作为超时保障。

```python
# DeerFlow：明确限制推理轮次
task_tool.call({
    "description": "Generate test cases",
    "prompt": "...",
    "max_turns": 5  # 简单任务不超过 5 轮
})

# Claude Code：后台任务配合超时停止
task_id = task_create.call({"prompt": "...", "description": "..."})["task_id"]
time.sleep(timeout)
status = task_get.call({"task_id": task_id})["status"]
if status == "running":
    task_stop.call({"task_id": task_id, "reason": "timeout"})
```

### 4. 用 TaskOutput 的 offset 做增量读取，而非轮询全量

长时间运行的任务输出会累积，每次全量读取浪费 token。用 offset 增量拉取。

```python
offset = 0
while True:
    result = task_output.call({"task_id": task_id, "offset": offset})
    new_content = result["content"]
    if new_content:
        process(new_content)
        offset += len(new_content.encode())
    
    if result["status"] in ("completed", "failed"):
        break
    
    time.sleep(2)  # 避免过于频繁
```

### 5. Team 用于批量协调，不用于状态共享

团队是组织工具，不是通信通道。如果需要 Agent 间通信，用 SendMessage 逐个发，而不是靠团队机制。

```python
# 用 team 做批量停止
team_name = "research-agents"
team_create.call({"name": team_name, "description": "Parallel research workers"})

# 创建时绑定 team（OpenHarness）
for topic in topics:
    agent_tool.call({"prompt": f"Research {topic}", "team": team_name, ...})

# 批量停止（通过 TaskList 过滤 team，再逐个 stop）
tasks = task_list.call({"status_filter": "running"})
team_tasks = [t for t in tasks if t.get("team") == team_name]
for t in team_tasks:
    task_stop.call({"task_id": t["task_id"]})
```

---

## 常见踩坑

**1. 并行子 Agent 共享文件系统导致冲突**

多个 Agent 同时写同一文件，最后写入的覆盖之前的结果。

解决：Claude Code 用 worktree（`cwd` 指向独立目录）隔离文件系统；OpenHarness 进程天然隔离；约定不同 Agent 写不同文件路径。

**2. DeerFlow 超过 3 个并发被静默拒绝**

DeerFlow 的 `SubagentLimitMiddleware` 在单次 model response 中超过 3 个 task 调用时拒绝执行，但不报错，任务只是没被创建。

解决：单次 response 最多发 3 个 task 调用；更多任务分成多轮 response 发出。

**3. OpenHarness 团队注册不持久**

`TeamRecord` 只存在内存中，进程重启后团队信息消失，之前绑定的任务 ID 也无法通过团队名找回。

解决：不依赖团队做跨会话状态追踪；重要关联关系存到外部（如文件或数据库）。

**4. SendMessage 不保证送达时机**

Claude Code 的 `writeToMailbox` 是异步写文件，OpenHarness 是 stdin 追加。子 Agent 什么时候读到消息取决于它的推理节奏，可能在几轮之后才处理。

解决：SendMessage 用于"下一步方向调整"类信息，不用于需要立即响应的紧急控制信号。

---

## 关联

关联 wiki：[[wiki/multi-agent]] · [[wiki/tool-system]] · [[wiki/query-loop]]

关联 cookbook：[[cookbook/tools/patterns/parallel-tool-calls]] · [[cookbook/prompts/patterns/prompt-chaining]]
