---
归档时间: 2026-04-15
更新: 2026-06-09
归档原因: 整体架构仍不适合 wholesale 合并 wiki；允许窄主题提升
状态: reference-with-promoted-insights
---

# SimpleMem — 归档说明

**归档原因：**
- v0.3.0 已用 `SimpleMem = AutoMemory`、`create(mode=...)` 和 text/omni backend router 缓解旧版“双版本接口不兼容”问题，但 AutoMemory 首次调用后 backend 不可切换，cross/ 层仍是独立顶层包
- cross/ 层与 core/ 层几乎零耦合，像两个独立项目强行拼在一起
- L2 中 multi-agent/evaluation-observability/channel-remote 为 medium confidence（该框架核心不在此）

**保留价值：**
- SQLite+LanceDB 双存储思路 → 已提炼为 `wiki/_insights/simplemem--dual-storage.md`
- EvolveMem 离线检索策略优化 → 已提炼为 `wiki/_insights/simplemem--evolvemem-retrieval-optimizer.md`
- 完整 L2 分析在 `shelf/simplemem/wiki/_impl/` 可查阅

**注意：** 不要将 SimpleMem 整体合并到 wiki/；只提升源码证据充分、可迁移的窄主题 insight/pattern。
