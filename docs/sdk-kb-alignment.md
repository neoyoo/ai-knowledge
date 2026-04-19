# SDK ↔ 知识库版本对齐记录

> 知识库的内容决定了 SDK 设计时能参考的「视野范围」。
> 每次知识库有重大新增，都应评估对 SDK 下一版本的设计影响。

## neoagent SDK 各版本的 KB 覆盖状态

| SDK 版本 | 发布时间 | KB 中已有的源 | KB 中缺失的源 | 备注 |
|---------|---------|------------|------------|------|
| v1–v3.2d | 2026-04-10 ~ 2026-04-13 | claude-code, openharness, deer-flow, hermes, mempalace | **agentscope** | 当前已发布版本，设计中无 agentscope 参考 |
| v3.3（规划中） | — | + agentscope（2026-04-15 ingest 完成） | — | 首个可参考 agentscope 设计的版本 |

## agentscope 对 SDK 下一版本的设计影响

以下是 agentscope ingest 后发现的、值得在 v3.3+ 评估引入的设计模式：

### query-loop
- **元类透明 hook 注入**（`_AgentMeta`）：hook 系统在类定义时自动植入，子类零侵入。对比 neoagent 当前的显式注册方式，可评估是否更优雅。
- **`__call__` / `reply` 职责分离**：`__call__` 专职中断处理和广播，`reply` 只管生成逻辑。

### multi-agent
- **MsgHub 广播总线**：`async with MsgHub(participants=[...])` 上下文管理器，自动管理订阅生命周期，比手动订阅/取消订阅更安全。
- **pipeline 三原语**：`sequential_pipeline` / `fanout_pipeline` / `ChatRoom`，轻量但覆盖主要编排模式。
- **A2A 协议**：Agent-to-Agent 标准化跨进程调用，是 neoagent 目前缺失的能力。

### tool-system
- **ToolGroup 动态启用/停用**：LLM 通过元工具 `reset_equipped_tools` 在运行时自主切换工具集，是目前 neoagent 不支持的动态工具管理模式。
- **`preset_kwargs` 参数隐藏**：比 closure wrapper 更声明式的参数隐藏机制。

### context-management
- **三层正交架构**：Token 计数层（5 种后端）/ Formatter 截断层（tool 配对安全）/ Memory 压缩层（独立压缩模型）。neoagent 当前只有单层压缩。

### hooks
- **双作用域 hook**（实例级 + 类级）：实例级先执行，类级影响所有实例。返回值决定是否修改语义（None = 观测，有返回值 = 修改）。

### finetuning-system（新 L1）
- AgentScope 揭示了 agent 框架集成 finetuning pipeline 的可能性（SFT/GRPO + prompt 自动优化）。v3.3 不一定实现，但值得在 roadmap 中记录。

---

## 使用方式

每次 KB 有新源 ingest 完成，在此文件追加一行版本对齐记录，并更新「设计影响」章节。
评估下一 SDK 版本设计时，先查此文件了解 KB 当前覆盖状态。
