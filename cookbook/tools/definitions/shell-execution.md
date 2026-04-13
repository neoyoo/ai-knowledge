---
tool-category: shell-execution
tags: [bash, shell, command, sandbox, core]
sources: [claude-code, deer-flow, openharness]
---

# Shell Execution

> 让 agent 在宿主机或沙箱中执行任意 shell 命令，是 agent 与操作系统交互的最底层通道，也是权限风险最高的工具类别。

## 本质

Agent 读文件、编辑文件、运行测试、部署服务——这些操作底层都是 shell 命令。Shell 工具就是那把"万能钥匙"：几乎什么都能做，也因此必须格外谨慎。

三个项目都实现了 Bash 工具，但在安全约束、超时机制、后台执行上的设计选择截然不同。核心分歧在一个问题：**谁来决定命令能不能跑？**

- Claude Code：用户信任 + 沙箱模式（可 bypass），给开发者充分自主权
- DeerFlow：严格沙箱优先，`allow_host_bash` 是明确的高风险开关
- OpenHarness：配置驱动，`cwd` 和 `timeout` 可精细控制，但没有内置沙箱

## 工具清单

| 工具名 | 项目 | 核心能力 | 特有特性 |
|--------|------|----------|----------|
| `Bash` | Claude Code | 执行 shell 命令 | `run_in_background`、`rerun` alias、沙箱 bypass 模式 |
| `bash` | DeerFlow | 执行 shell 命令（沙箱内） | 仅在 `sandbox` 或 `allow_host_bash=true` 时可用 |
| `bash` | OpenHarness | 执行 shell 命令 | `cwd` override、timeout 范围 1–600s |

## 可复制 Schema

### Claude Code — Bash

```python
{
    "name": "Bash",
    "description": "执行 shell 命令。优先使用专用工具（Read/Edit/Glob）而非 Bash。",
    "input_schema": {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "要执行的 bash 命令。含空格的路径用双引号包裹。"
            },
            "timeout": {
                "type": "integer",
                "description": "超时时间（毫秒），最大 600000（10 分钟）。默认 120000（2 分钟）。",
                "default": 120000
            },
            "run_in_background": {
                "type": "boolean",
                "description": "设为 true 时在后台运行，立即返回。适合长任务，完成后会收到通知。",
                "default": false
            },
            "description": {
                "type": "string",
                "description": "命令用途的简短描述（主动语态），便于日志和审查。"
            },
            "rerun": {
                "type": "string",
                "description": "重新执行之前的命令，填入先前结果中的 alias（如 'b3'）。与 command 互斥。"
            }
        },
        "required": ["command"]
    }
}
```

**关键行为：**
- 沙箱模式下（默认启用）会拦截潜在危险命令，弹出用户确认
- `dangerouslyDisableSandbox: true` 可 bypass 沙箱，仅限明确信任的环境
- 每次 bash 调用 cwd 会重置，跨调用需用绝对路径
- `rerun` alias 来自前一次结果的 `[rerun: bN]` 尾注，精确重复执行

### DeerFlow — bash

```python
{
    "name": "bash",
    "description": "在沙箱环境中执行 shell 命令。仅在沙箱已配置或 allow_host_bash=true 时可用。",
    "input_schema": {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "要执行的 bash 命令。"
            },
            "timeout": {
                "type": "integer",
                "description": "超时时间（秒）。默认 30，最大 300。",
                "default": 30
            }
        },
        "required": ["command"]
    }
}
```

**关键行为：**
- 工具本身不暴露，除非满足：沙箱环境已就绪 OR 配置 `allow_host_bash=true`
- 沙箱内运行：文件系统隔离，网络访问受限
- `allow_host_bash=true` 等同于直接操作宿主机，DeerFlow 文档将其标记为高风险配置

### OpenHarness — bash

```python
{
    "name": "bash",
    "description": "执行 shell 命令，支持工作目录覆盖和精细超时控制。",
    "input_schema": {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "要执行的 bash 命令。"
            },
            "cwd": {
                "type": "string",
                "description": "命令执行的工作目录（绝对路径）。覆盖默认工作目录。"
            },
            "timeout": {
                "type": "integer",
                "description": "超时时间（秒），范围 1–600。默认 60。",
                "default": 60,
                "minimum": 1,
                "maximum": 600
            }
        },
        "required": ["command"]
    }
}
```

**关键行为：**
- `cwd` 是 OpenHarness 特有参数，方便在多项目目录间切换而不用 `cd`
- 无内置沙箱，依赖运行环境的 OS 级隔离（容器、VM）
- timeout 范围比 Claude Code 更保守（上限 600s vs 600000ms，两者等价）

## 跨项目对比

### 沙箱策略

这是三个实现分歧最大的维度。

| 维度 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| **默认行为** | 沙箱模式，危险命令需用户确认 | 沙箱优先，无沙箱则工具不可用 | 无内置沙箱，依赖外部隔离 |
| **bypass 方式** | `dangerouslyDisableSandbox: true` | `allow_host_bash: true` | 无需 bypass（本来就没沙箱） |
| **沙箱实现** | 命令拦截 + 用户确认提示 | 隔离容器/进程 | — |
| **适用场景** | 开发者本地环境，高度可信 | 多租户、生产环境、代码执行服务 | 容器化部署场景 |

**设计哲学差异：**
- Claude Code 的"沙箱"更像"安全确认"——不阻止操作，而是让用户知情同意
- DeerFlow 的"沙箱"是真正的运行时隔离——文件系统和网络都与宿主隔离
- OpenHarness 把隔离责任交给部署层，工具本身不管

### 超时机制

| 项目 | 默认超时 | 最大超时 | 单位 | 后台执行 |
|------|---------|---------|------|----------|
| Claude Code | 120s | 600s | 毫秒 | 支持（`run_in_background`） |
| DeerFlow | 30s | 300s | 秒 | 不支持 |
| OpenHarness | 60s | 600s | 秒 | 不支持 |

Claude Code 默认超时最长（120s），适合本地开发的长编译任务。DeerFlow 最保守（30s），符合沙箱多租户场景的资源控制需求。

### 后台执行

只有 Claude Code 原生支持后台执行（`run_in_background: true`）。

后台执行的价值：
- 启动服务器后继续其他操作，不阻塞 agent 主流程
- 触发长时间构建/测试任务后，立即去做其他独立工作
- 任务完成后会收到通知，无需轮询

DeerFlow 和 OpenHarness 如需后台执行，需要在命令本身加 `&` 或用 `nohup`，但这会失去结构化的完成通知。

### 安全约束（破坏性命令处理）

三个项目都没有内置的命令黑名单，但约束方式不同：

| 项目 | `rm -rf /` 的处理 | 机制 |
|------|-----------------|------|
| Claude Code | 沙箱模式下触发用户确认 | 规则拦截 + prompt |
| DeerFlow | 沙箱内执行，影响沙箱内的文件系统，宿主安全 | 运行时隔离 |
| OpenHarness | 直接执行，无内置保护 | 依赖 prompt engineering 和用户审查 |

实践中，Claude Code 的 system prompt 里会明确禁止某些命令模式（如 `rm -rf`、`git push --force`）：这是 prompt 层面的约束，不是工具层面的硬限制。

## 最佳实践

### 1. 优先用专用工具

Shell 是万能的，但万能意味着可观察性差、重试成本高。

```
# 不推荐：用 Bash 读文件
Bash("cat /path/to/file.py")

# 推荐：用专用工具
Read("/path/to/file.py")
```

专用工具（Read/Edit/Glob/Grep）的优势：
- 结构化输出，不受 stdout 格式影响
- 工具本身记录操作意图，方便审查
- 错误处理更清晰（文件不存在 vs 权限错误 vs 解析错误）

**规则：能用专用工具的，不用 Bash。Bash 留给"专用工具做不到的事"。**

### 2. 设置合理 timeout

```python
# 短命令（ls、cat 小文件）
{"command": "ls -la", "timeout": 10000}  # 10s 够了

# 编译/测试（可能较慢）
{"command": "npm run build", "timeout": 180000}  # 3 分钟

# 长任务用后台执行
{"command": "npm run test:full", "run_in_background": true}
```

timeout 设太小会误判为失败；设太大会让 agent 长时间卡住。经验值：
- 文件操作 / 简单命令：10–30s
- 编译 / 安装依赖：60–180s
- 测试套件：用后台执行

### 3. 生产环境必须有沙箱

无沙箱的 shell 执行等同于给 agent 完整的 root 权限（取决于运行用户）。

生产/多租户部署的最低要求：
- 使用 DeerFlow 的沙箱容器模式，或在容器/VM 内运行 OpenHarness
- 限制文件系统挂载范围
- 用非特权用户运行 agent 进程
- 启用网络出站白名单

### 4. 后台执行适合长任务

```python
# 启动开发服务器，不阻塞 agent 继续工作
{"command": "python -m http.server 8080", "run_in_background": true}

# 然后 agent 可以继续做其他事
# 服务器完成（或崩溃）后会收到通知
```

不要用 `sleep` 等待后台任务——设置 `run_in_background: true`，框架会在任务结束时主动通知。

### 5. 路径处理

```bash
# 含空格的路径必须加引号
cd "/path with spaces/project"

# 跨调用保持 cwd 一致（各项目行为不同）
# Claude Code：每次调用 cwd 重置，用绝对路径
# OpenHarness：用 cwd 参数显式指定
```

## 关联

- [[cookbook/tools/definitions/file-operations]] — 文件读写专用工具，优先于 Bash
- [[wiki/tool-system]] — 工具系统架构，工具注册与执行机制
- [[wiki/reliability]] — 工具失败处理、超时熔断、重试策略

**来源：**
- Claude Code system prompt — Bash 工具 description 和 `dangerouslyDisableSandbox` 行为
- DeerFlow `tools/bash.py` — 沙箱检测逻辑和 `allow_host_bash` 配置
- OpenHarness `tools/bash_tool.py` — `cwd`、`timeout` 参数实现
