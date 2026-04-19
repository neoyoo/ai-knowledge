---
title: "prompt-system — neoagent"
category: L2
parent: "[[prompt-system]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: prompt-system
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的 prompt 系统由 `PromptBuilder`（`core/prompt.py`，仅 57 行）+ `QueryLoop` 动态段注入组成。`PromptBuilder` 是**纯有序容器 + 三态 skill 管理器**：`add_section`/`remove_section` 管理常驻段，`register_skill`（储存但不激活）→ `activate_skill`（加入活跃列表）→ `deactivate_skill`（从活跃列表移除）三步流程支持运行时动态开关 skill。每次 `build()` 按 `(is_static, priority)` 两级排序拼接——静态段先按 priority 升序，动态段（`is_static=False`）追加在后面。真正"动态注入"在 `QueryLoop.run()` 内发生：deferred 工具名列表和 freed tool result 清单每轮按 session 状态追加到 `system = self._prompt_builder.build()` 返回值的末尾。

## 架构分析

### PromptSection 数据模型

```python
@dataclass
class PromptSection:
    name: str
    content: str | Callable[[], str]   # 静态字符串 or 延迟计算
    priority: int                       # 越小越先
    is_static: bool = True              # False = 动态段，排序时置后
```

- **identity section**（`priority=0, is_static=True`）：在 `NeoAgent.__init__` 调用 `add_section(PromptSection(name="identity", content=config.system_prompt or "You are neoagent, a helpful AI assistant.", priority=0))` 建立。这是唯一默认段。
- **memory section**（`priority=5, is_static=False`）：在 `NeoAgent.enable_memory()` 里通过 `add_section(PromptSection(name="memory", content=lambda: memory_manager.build_prompt_section(), priority=5, is_static=False))` 注入——`content` 是 lambda，每次 build 时重新从 MemoryStore 读取 MEMORY.md 索引。

`PromptBuilder.build()` 输出形如：

```
# identity
You are neoagent, a helpful AI assistant.

# memory
# Memory
- [用户偏好](user_prefs.md)
- [已完成任务列表](completed_tasks.md)
```

### 循环级动态段（在 builder 外追加）

`PromptBuilder.build()` 返回后，`QueryLoop.run()` 按 session 状态追加两种段：

1. **deferred-tools 段**：
   ```
   <deferred-tools>
   github__create_issue
   github__list_prs
   playwright__click
   ...
   </deferred-tools>
   ```
2. **freed tool results 段**（由 `_render_freed_section` 生成）：
   ```
   ## Freed Tool Results (recoverable)
   - `toolu_01A...` · run_python · 4321B · 'First 80 chars preview...'
   
   Call `recall_tool_result(tool_use_id)` to view full content for the current turn.
   ```

这两段刻意不进 `PromptBuilder`——因为它们每轮变化且依赖 session state，进 builder 反而要破坏 "builder 只知道声明式段" 的 clean 模型。

### Skill 激活的三态流转

```python
builder.register_skill("xiumi-pattern", section)   # 存在 _skills dict，不进 _sections
builder.activate_skill("xiumi-pattern")             # 进 _sections
builder.deactivate_skill("xiumi-pattern")           # 从 _sections 移除，仍在 _skills
builder.is_skill_active("xiumi-pattern")            # -> bool
```

这和 `skill_load` 工具的 "progressive disclosure" 设计互补：LLM 可以在 list 阶段只看到 name+description，调 `load_skill` 时拿到全文，然后由外部代码决定是否把该 skill 的 procedure 作为 PromptSection 注入（通过 `register_skill + activate_skill`）。目前 `QueryLoop` 自身未自动调用这三个 API——激活动作需要调用方（应用代码）在 load_skill 结果返回后手动触发，框架只提供基础设施。

### 关键代码路径

- `neoagent/core/prompt.py:1-57` — `PromptBuilder` 全部实现
- `neoagent/agent.py:51-55` — identity section 默认注入
- `neoagent/agent.py:197-204` — `enable_memory()` 注入 memory section（lambda content）
- `neoagent/core/loop.py:123-157` — deferred-tools 段拼接
- `neoagent/core/loop.py:159-169, 387-397` — freed tool results 段拼接（`_render_freed_section`）
- `neoagent/events.py:89-96` — `SkillChangeEvent` 已定义但源码注释 `TODO v3.2: wire into PromptBuilder.activate_skill() / deactivate_skill() once PromptBuilder receives EventBus access` ——尚未接入 event bus

## 设计亮点

- **三态 skill 管理**（register / activate / deactivate），语义清晰——"存在但未激活"和"完全未知"是不同状态，支持 LLM 自主选择要不要加载一项 skill 到下一轮 prompt；相比 AgentScope 的 `reset_equipped_tools` 元工具方案是更细粒度的：tool 集切换 + prompt section 切换分离。
- **lambda content 让动态段（如 memory）可延迟计算**：memory section 每次 build 时重新读 MEMORY.md，自然实现"会话过程中记忆库被写入新文件，下一轮 prompt 自动反映"——无需显式触发 prompt 刷新。
- **priority + is_static 两级排序**：静态段之间按 priority 稳定排序，动态段（如 memory）永远置后，确保"身份指令永远在最前"这个约束不需要手动维护 priority 数值。
- **循环级段与 builder 段分离**：`<deferred-tools>` 和 `## Freed Tool Results` 不进 builder，体现了"builder 管声明式常驻段，loop 管每轮动态段"的职责划分——这让 builder 保持纯粹的"无状态渲染器"属性，易测。

## 局限性

- **SkillChangeEvent 未接入**：代码自带 TODO 注释，说明 skill 激活/停用当前无事件追踪，Observer 看不到。`agent.on("skill_change", ...)` 接口不存在，要做 skill 审计必须包装 PromptBuilder。
- **PromptBuilder 不持有 EventBus 引用**：`NeoAgent.__init__` 没有把 `_event_bus` 传进 PromptBuilder，导致 skill 事件无法发射；这是 TODO 的根因。重构方案应在 `PromptBuilder.__init__(event_bus)` 注入。
- **没有段级 token 预算**：`PromptBuilder.build()` 拼完就返回，不知道总长度；当 memory section 返回很大的内容（或 deferred tools 很多），system prompt 可能悄悄膨胀到数千 token，compressor 只能事后检测、不能事前截断单段。Claude Code 和 Hermes 的 system prompt token 估算没进入 neoagent。
- **段名重复检查是 O(n)**：`add_section` 里 `any(s.name == section.name for s in self._sections)`，在上百段时有性能问题——不过实际场景 section 数很少（5-10 个），不算真问题。
- **Skill section 激活后无法按优先级动态调整**：`activate_skill` 只追加到 `_sections` 末尾，如果 skill 的 priority 应该在 identity 之后但 memory 之前，需要自己管 priority——没有"insert at position"接口。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/core/prompt.py`, `neoagent/agent.py:51-55, 197-204`, `neoagent/core/loop.py:123-169`
