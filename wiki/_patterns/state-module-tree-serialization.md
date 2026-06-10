---
title: State Module Tree Serialization
aliases: [状态树序列化, state_dict protocol, module state tree]
kind: pattern
created: 2026-05-02
updated: 2026-06-10
concepts_involved: [[runtime-state]], [[memory-system]], [[session-recovery]]
reference_implementations: [agentscope <=1.x]
status: historical
---

## 一句话定义

把 agent、memory、toolkit 等运行时对象组织成可递归序列化的状态树，用统一 `state_dict()` / `load_state_dict()` 协议完成保存、恢复和跨后端持久化。

> 状态：历史模式。AgentScope Python 2.x 当前源码已迁移到 `AgentState + SessionRecord + StorageBase + MessageBus`，不再使用旧版 `StateModule / SessionBase / MemoryBase` 路径。本页保留作为“状态树快照”方案的历史参考，不应再作为 AgentScope 2.x 当前事实引用。

## 触发问题

复杂 agent 的状态不只是一份 messages：还有工作记忆、压缩摘要、工具激活状态、运行配置、计划、临时标记。若每个模块各自实现保存逻辑，session recovery 会变成一组脆弱的手写拼接。

## 参与的概念

- [[runtime-state]] — 定义哪些对象属于可恢复运行时状态
- [[memory-system]] — memory 本身成为状态树中的子模块
- [[session-recovery]] — 存储层只保存统一 state dict，不理解业务对象

## 核心设计

### 1. 所有有状态模块继承同一基类

`StateModule` 提供两个能力：注册标量字段、自动发现子模块。普通属性通过 `register_state("field")` 声明，子模块在赋值时被 `__setattr__` 识别并加入 `_module_dict`。

### 2. 状态协议与存储后端解耦

业务对象只负责输出 JSON-like `state_dict()`。JSON 文件、Redis、Tablestore 或数据库后端只保存这份 dict，不依赖 agent / memory 的内部类。

### 3. 恢复路径递归向下分发

`load_state_dict()` 从根 agent 开始，标量字段直接恢复，子模块 state 递归交给对应子模块。新增模块时只要注册到树上，就自然进入保存/恢复路径。

## 适用场景

- 长会话 agent，需要断点恢复
- multi-agent session 中同时保存多个 agent 状态
- memory / toolkit / plan 等模块会独立演进
- 希望同一套状态协议适配本地文件和远程存储

## 不适用场景

- 完全 stateless API endpoint
- 只需保存聊天历史的简单 bot
- 状态量巨大且需要高频增量写入；此时要引入 event log 或 diff 机制

## 参考实现

历史 AgentScope 的 `StateModule` / memory / session 路线：

- `wiki/_impl/runtime-state--agentscope.md` — 当前 2.x 已改为 `AgentState + SessionRecord`
- `wiki/_impl/memory-system--agentscope.md` — 当前 2.x 无内置长期 memory
- `wiki/_impl/session-recovery--agentscope.md` — 当前 2.x 已改为 `StorageBase + ChatService + MessageBus`

## 迁移 checklist

- 明确哪些字段必须参与恢复，哪些只是运行时缓存
- 给不可 JSON 序列化对象提供自定义 to/from JSON
- 保存前打印一次 state tree，确认关键子模块已注册
- 版本升级时为新增字段设计默认值或 migration
- 大 memory 场景避免每步全量保存

## 常见陷阱

- 只注册 agent 根对象，忘记 memory / toolkit 子模块
- `strict=True` 直接加载旧快照，新增字段导致恢复失败
- 把文件句柄、socket、子进程对象直接放进 state
- 全量快照频率过高，长对话下序列化成本线性膨胀

## 相关概念

- [[runtime-state]]
- [[memory-system]]
- [[session-recovery]]
