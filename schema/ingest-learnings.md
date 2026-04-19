# Ingest 经验积累

> 每次 kb-ingest 完成后，Phase 7 自动追加经验条目到此文件。
> 每积累 5 条（或手动触发），启动策略蒸馏流程，提炼为 ingest-strategies.yaml。
> 蒸馏完成后，对应条目可归档（移至文件末尾的「已蒸馏」区块）。

## 活跃经验条目

### SimpleMem — 2026-04-15
- 覆盖率: 100%，补漏 0 轮
- 发现：SimpleMem 是记忆系统专项项目（非通用 agent 框架），9 个 ontology 概念中有 7 个高度相关（memory-system/context-management/query-loop/hooks/mcp-skills/runtime-state/channel-remote），2 个中等相关（multi-agent/evaluation-observability），2 个无对应实现（prompt-system/session-recovery）。项目分两个独立版本（文本版 core/ + 多模态版 OmniSimpleMem/），通过 Registry 路由器封装但接口不兼容，ingest 时需两套分别分析。cross/ 层是一个相对独立的"跨会话协调层"，与 core/ 层几乎零耦合，可视为独立子项目处理。
- 教训：（1）记忆系统项目的"多版本"特征（文本/多模态）要在 module-map 阶段识别，避免 L2 只分析一个版本；（2）双存储架构（SQLite+LanceDB）在多个概念维度（runtime-state/multi-agent/context-management）都有体现，ingest 时应识别「存储架构」作为跨概念的共性主题；（3）MCP 集成（api_mcp.py）同时涉及 mcp-skills 和 channel-remote 两个 L1，需要在两个 L2 页面分别从不同角度描述，避免重复；（4）专用记忆系统项目（vs 通用 agent 框架）的 query-loop 概念实际上是「检索循环」而非「agent 主循环」，在 L2 中要明确区分，避免与其他项目的 agent loop 概念混淆。
- L2 质量分布：high=6 medium=3 low=0
- 阻断问题：0
- 待确认问题：3（multi-agent/evaluation-observability 的 OmniSimpleMem 层实现细节；channel-remote 的 MCP transport 配置）

### agentscope — 2026-04-15
- 覆盖率: 90.5% (19/21)，补漏 0 轮
- 发现：AgentScope 是目前 KB 中唯一覆盖完整 finetuning pipeline（tune/tuner 模块：SFT/DPO/GRPO + DSPy prompt 优化 + 模型自动选择）的项目，且该能力完全不在现有 12 个 ontology 概念范围内。tune/ 已废弃（仅 ImportError），tuner/ 是真实实现，对接 Trinity-RFT 框架。此类"框架独有且无 ontology 对应"的模块是新 L1 概念的候选信号。另发现 AgentScope 的 multi-agent 实现与其他源有本质差异：不是经典 orchestrator/worker 模式，而是通过 pipeline 编排原语（sequential/fanout）+ MsgHub 广播总线 + A2A 协议三个正交抽象组合实现，这使 L1 对比表新增了一个有价值的非典型设计维度。
- 教训：（1）扫描有效模块时，若某模块完全超出 ontology 范围（如 finetuning），记录为"特别发现"而非阻断，ingest 完成后单独评估升 L1 资格；（2）对于"训练 vs 推理"的大类划分，AgentScope 的 tuner/ 属于训练系统，与推理架构 ontology 正交，可先跳过不影响整体 ingest 质量；（3）class-level hook 的继承共享 bug（AgentScope hooks 的已知问题，相关测试全被注释）是值得在 L1 「常见陷阱」中单独列出的实践教训；（4）12 个概念维度并行派发 subagent 效果良好，全部返回 high confidence，证明 Sonnet 模型在源码级分析上的可靠性。
- L2 质量分布：high=12 medium=0 low=0
- 阻断问题：0
- 待确认问题：0（自动修复 2 项：title 分隔符统一、H3 标题升 H2）

---

## 已蒸馏归档

（经策略蒸馏后的经验条目移至此处，仅作历史记录。）
