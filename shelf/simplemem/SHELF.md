---
归档时间: 2026-04-15
归档原因: 代码质量不高，架构不稳定，不合并 wiki
状态: reference
---

# SimpleMem — 归档说明

**归档原因：**
- 双版本（文本/多模态）接口不兼容，设计本身不稳定
- cross/ 层与 core/ 层几乎零耦合，像两个独立项目强行拼在一起
- L2 中 multi-agent/evaluation-observability/channel-remote 为 medium confidence（该框架核心不在此）

**保留价值：**
- SQLite+LanceDB 双存储思路 → 已提炼为 `wiki/_insights/simplemem--dual-storage.md`
- 完整 L2 分析在 `shelf/simplemem/wiki/_impl/` 可查阅

**注意：** 不要将此项目内容合并到 wiki/。
