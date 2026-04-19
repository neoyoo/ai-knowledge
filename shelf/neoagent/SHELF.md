---
归档时间: 2026-04-19
归档原因: 项目尚未成熟（非 PyPI 公开、个人项目阶段），设计仍在演化，不合并 wiki
状态: reference
---

# neoagent — 归档说明

**归档原因：**
- 尚未 PyPI 公开，仅作者本机 editable 安装使用，API 仍在频繁演化（例如 `auto_free_after` 协议为近期新增）
- 生产验证面窄（单项目 trip-os 在内部使用）
- 和主 wiki 里的成熟参考系（claude-code / deer-flow / openharness / agentscope）还不是一个量级

**保留价值（值得从源抽取成独立 insight，不作为 L1 概念页的首选参考）：**

三个 neoagent 独创且同类源未见的设计，建议各写成 `wiki/_insights/` 洞察卡：

1. **`BaseTool.auto_free_after` 协议驱动的 freed/recall 循环**
   - 工具自声明"我返回的内容多少轮后可以折叠"
   - loop 每轮扫描，到龄自动将 tool_result 替换为 `[freed: id=…, preview=…]` 占位符
   - 同时往 system prompt 注入可恢复清单
   - LLM 可用 `recall_tool_result(id)` 本轮重激活
   - 突破了"压缩 vs 丢弃"的二元对立，提供了第三种"折叠可召回"的形态

2. **Session-scoped MCP `promoted_tools` + ContextVar 并发隔离**
   - MCP 工具默认 deferred（隐藏），`tool_search` 按需提升
   - 提升可见性是 **per-session** 不是 global——多个并发 HTTP session 互不污染
   - 通过 ContextVar 把 session 传给 tool 实现，零锁、零全局

3. **Hook + Event 双轨正交分离**
   - Hook：pre/post 拦截点，可返回 allow/block/modify 改变运行流
   - Event：纯观察，不阻塞，通过 EventBus 多订阅者广播
   - 两条路互不依赖，Observer 订阅 Event 做日志，Hook 做权限/审计
   - 对比 Claude Code 混合 Hook+Event 的做法更干净

**保留内容：**
- 完整 L2 分析：`shelf/neoagent/wiki/_impl/`
- L1 语义补丁（未合入）：`shelf/neoagent/wiki/*.md.patch`
- 模块映射：`shelf/neoagent/analysis/module-map.md`
- ingest meta：`shelf/neoagent/meta.yaml`（标记 `READY_TO_MERGE`，但主观决定不合入）

**注意：** 不要将此项目内容合并到 wiki/。如果 neoagent 未来公开并被多项目采用，重新评估。
