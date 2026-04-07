# Module Registry — Agent 模块热插拔系统设计

> 日期：2026-04-07
> 状态：已确认，待实施
> 参与者：Neo + Claude
> 项目类型：独立新项目（纯 Python 库）

## 1. 定位

让 agent 具备运行时按需加载/卸载能力模块的能力。核心 agent loop 保持不变，通过动态 Module Registry 实现热插拔。

**核心理念**：agent 从"被配置"变成"自适应" — 用到什么能力就加载什么模块，用完可以卸掉。

**不是什么**：
- 不是 agent 框架（不提供 loop、不提供 model 调用）
- 不绑定任何框架（LangChain、LangGraph、自研都能用）
- 不管模块内部实现（模块可大可小，用户自定义粒度）

## 2. 核心组件

| 组件 | 职责 |
|------|------|
| `Module` | 抽象基类，定义模块接口，用户实现具体能力 |
| `Registry` | 管理模块生命周期（注册/卸载/查询）、依赖检查、权限控制 |
| `manage_modules` tool | 暴露给 LLM 的工具，让模型自主加载/卸载模块 |
| `permissions.yaml` | 人类可编辑的权限配置文件 |

## 3. Module 接口

```python
from abc import ABC, abstractmethod
from typing import Any

class Module(ABC):
    """所有模块的基类"""

    # --- 元数据（子类必须定义）---
    name: str           # 唯一标识，如 "search"
    version: str        # 版本号，如 "1.0.0"
    description: str    # 一句话描述，注入到模型的能力目录
    deps: list[str]     # 依赖的其他模块 name 列表
    tools: list[str]    # 本模块提供的工具名列表（注册后暴露给模型）

    # --- 生命周期 ---
    async def on_load(self, ctx: dict) -> None:
        """加载时调用，初始化资源"""
        pass

    async def on_unload(self) -> None:
        """卸载时调用，清理资源"""
        pass

    # --- 执行钩子（可选覆写）---
    async def before_model(self, state: dict) -> None:
        """模型调用前"""
        pass

    async def after_model(self, state: dict, response: Any) -> None:
        """模型调用后"""
        pass

    async def wrap_tool_call(self, tool_name: str, args: dict, call_next) -> Any:
        """包装工具调用（类似中间件）"""
        return await call_next(tool_name, args)
```

**生命周期流程**：
```
register() → on_load(ctx) → [active 状态]
  → before_model() / after_model() / wrap_tool_call() 被 loop 调用
  → on_unload() → 从 registry 移除
```

## 4. Registry API

```python
from enum import Enum

class Permission(Enum):
    AUTO = "auto"   # agent 自主加载
    ASK = "ask"     # 需要人确认
    DENY = "deny"   # 禁止加载

class Registry:
    def __init__(self, permissions: dict[str, Permission] = None):
        """
        permissions: {"module_name": Permission, "*": Permission}
        支持通配符 "*" 作为默认规则
        """

    # --- 基本操作 ---
    def register(self, module: Module) -> None:
        """
        注册模块。
        1. 检查依赖是否满足（deps 中的模块必须已注册）
        2. 检查循环依赖
        3. 调用 module.on_load(ctx)
        4. 加入 active 列表
        """

    def unregister(self, name: str) -> None:
        """
        卸载模块。
        1. 检查是否有其他模块依赖它 → 有则拒绝
        2. 调用 module.on_unload()
        3. 从 active 列表移除
        """

    def get_active(self) -> list[Module]:
        """返回当前所有 active 模块，按注册顺序"""

    def get_module(self, name: str) -> Module | None:
        """按 name 获取已注册模块"""

    # --- Agent 自主加载 ---
    async def request_load(
        self,
        name: str,
        requester: str = "agent",
        reason: str = "",
        on_ask: callable = None,
    ) -> bool:
        """
        请求加载模块（受权限控制）。
        - AUTO → 直接加载，返回 True
        - ASK → 调用 on_ask(name, reason) 回调等人确认
        - DENY → 直接返回 False
        - 模块不在已知列表 → 查通配符规则

        on_ask: 异步回调函数，返回 bool（人是否同意）
        """

    async def request_unload(self, name: str, requester: str = "agent") -> bool:
        """请求卸载模块（同样受权限控制）"""

    # --- Prompt 注入 ---
    def get_module_catalog(self) -> str:
        """
        生成可注入 system prompt 的模块目录。
        格式：
        | 模块 | 能力 | 状态 |
        |------|------|------|
        | search | 网页搜索 | 已加载 |
        | code_exec | 执行代码 | 可加载(需确认) |
        | data_analysis | 数据分析 | 可加载 |
        """

    def get_active_tools(self) -> list[dict]:
        """返回所有已加载模块提供的工具定义（用于模型的 tools 参数）"""
```

## 5. manage_modules Tool

暴露给 LLM 的工具定义：

```python
def manage_modules_tool():
    return {
        "name": "manage_modules",
        "description": (
            "管理可用能力模块。"
            "list: 查看所有可用模块及状态。"
            "load: 加载一个模块以获得新能力。"
            "unload: 卸载不再需要的模块以释放资源。"
        ),
        "parameters": {
            "action": {"type": "string", "enum": ["list", "load", "unload"]},
            "module_name": {"type": "string", "description": "模块名（load/unload 时必填）"},
            "reason": {"type": "string", "description": "为什么要加载/卸载（方便人审批）"},
        }
    }
```

**System Prompt 注入模板**：

```markdown
# 可扩展能力

你可以通过 manage_modules 工具按需加载能力模块。

{{module_catalog}}

当前已加载：{{active_modules}}

规则：
- 需要某个能力但没加载时，用 manage_modules("load", "模块名", "原因")
- 任务完成后不再需要的模块，用 manage_modules("unload", "模块名", "原因") 释放
- 标记为"需确认"的模块会等待用户批准
```

## 6. 权限配置

```yaml
# permissions.yaml

# 每个模块的权限等级
modules:
  search: auto
  memory: auto
  http_client: auto
  data_analysis: auto
  code_exec: ask
  file_write: ask
  shell_exec: ask
  system_admin: deny

# 默认规则（未列出的模块）
default: ask
```

**加载时的决策流程**：
```
request_load("X")
  → modules 表里有 X？ → 用对应权限
  → 没有？ → 用 default 权限
  → AUTO → 直接加载
  → ASK → 回调 on_ask，等人确认
  → DENY → 拒绝
```

## 7. 依赖管理

**三条硬规则**：

1. **注册时依赖必须先在** — `register(A)` 时，A.deps 中的每个模块必须已在 registry 中。不满足则：尝试自动加载依赖（递归 request_load），或报错。

2. **卸载时被依赖的不能先卸** — `unregister(B)` 时，如果有模块 A 的 deps 包含 B，拒绝卸载并返回错误消息。

3. **禁止循环依赖** — 注册时用拓扑排序检测，发现环则拒绝注册。

## 8. 错误处理

| 场景 | 处理方式 |
|------|---------|
| on_load 抛异常 | 模块不进入 active，返回错误给调用方 |
| before_model 抛异常 | 跳过该模块，记录错误，继续执行 |
| after_model 抛异常 | 同上 |
| on_unload 抛异常 | 强制移除，记录错误 |
| 模块加载超时 | 可配置超时（默认 30s），超时则 on_load 失败 |

## 9. 文件结构

```
module-registry/
├── src/
│   └── module_registry/
│       ├── __init__.py          # 导出 Module, Registry, Permission
│       ├── module.py            # Module 抽象基类
│       ├── registry.py          # Registry 核心实现
│       ├── permissions.py       # 权限加载和检查
│       ├── dependency.py        # 依赖检查（拓扑排序）
│       ├── catalog.py           # 模块目录生成（prompt 注入）
│       └── tool.py              # manage_modules tool 定义
├── tests/
│   ├── test_registry.py         # 注册/卸载/查询
│   ├── test_permissions.py      # 权限检查
│   ├── test_dependencies.py     # 依赖管理
│   └── test_integration.py      # 和 agent loop 集成
├── examples/
│   ├── simple_agent.py          # 单 agent 按需加载示例
│   └── multi_agent_discussion.py # 集成 multi-agent MVP
├── permissions.yaml             # 示例权限配置
├── pyproject.toml
└── README.md
```

## 10. 测试计划

### Phase 1：单 agent 按需加载
- 创建 3 个示例模块（search、data_analysis、file_write）
- 单 agent + manage_modules tool + Registry
- 验证：模型能根据任务自主加载需要的模块
- 验证：ASK 权限的模块需要人确认
- 验证：卸载后工具从模型可用列表消失

### Phase 2：集成 multi-agent discussion MVP
- 将 Supervisor/Compressor/Coordinator 改造为 Module
- Registry 管理它们的动态加载
- 验证：讨论开始时按需加载各角色模块
- 验证：讨论结束后卸载释放

### Phase 3：自愈验证
- 模拟模块 on_load 失败 → 验证降级处理
- 模拟模块运行时异常 → 验证跳过不崩溃
- 模拟依赖被卸载 → 验证拒绝并报错

## 11. 与知识库的关系

这个项目验证后将产出：
- 知识库新 L1 概念候选："Agent Self-Evolution"（或作为多个 L1 的横切关注点）
- 实践验证的设计模式写入 L1 设计权衡
- 成熟后抽取为 create-agent-like-claude 的核心模块
