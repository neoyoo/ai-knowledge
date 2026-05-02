---
title: Session metadata storage — asyncpg + Postgres (not SQLAlchemy, not in-memory)
date: 2026-04-19
status: accepted
decider: neo
related-wiki: [[session-recovery]], [[runtime-state]]
related-patterns: [[_patterns/async-native-style]]
---

## 背景

P1a 将 trip-os 从命令行工具升级为 Web 服务（FastAPIChannel）。Web 服务需要跨 HTTP 请求持久化会话元数据：id、slug、status、input、result_path、node_id、timestamps。

需要一个存储层满足以下条件：
- (a) 跨请求持久化
- (b) 可查询（按 status 列表、按 id 获取）
- (c) P2 多节点分布式就绪
- (d) 1 表 schema 低仪式感

## 候选方案

| 方案 | 优点 | 缺点 | 推荐 |
|------|------|------|------|
| **in-memory dict** | 零依赖，启动快 | ❌ 重启丢失；多节点无法共享（P2 阻塞） | ❌ |
| **SQLite** | 简单文件存储 | ❌ 单写者限制并发；多节点困难（P2 阻塞） | ❌ |
| **SQLAlchemy 2.0 async + Alembic** | 良好 IDE 支持、迁移工具成熟 | ORM 开销；Alembic 对 1 表过度设计；学习成本 | ⚠️ 备选 |
| **asyncpg + 原始 SQL** | 零 ORM 开销、SQL 全控制、与 neoagent 异步风格匹配 | 迁移自己写、无 IDE 自动补全 | ✅ |

## 决策

**采用 asyncpg + 原始 SQL**。

### 实现细节

- **迁移工具**：`infra/postgres/migrate.py`（~50 行 psql 包装，顺序执行 `*.sql` 文件，跟踪状态于 `migrations_log` 表）
- **SessionRepo**：`explorer/storage/session_repo.py`（~125 行，方法：create、get、update_status、update_result、list_by_status）
- **Pool 初始化**：FastAPI lifespan hook，执行 `SET search_path=trip_os` 初始化函数
- **表 schema**：见 `infra/postgres/migrations/001_sessions.sql`

### 为什么选择

1. **零 ORM 开销**——1 表 schema，SQL 手写更清晰
2. **neoagent 异步匹配**——SDK 是异步原生，asyncpg 风格一致
3. **透明升级路径**——当 schema 扩展到 3+ 表或团队规模突破 1 人时，重新考虑 SQLAlchemy 2.0 + Alembic，但不是现在
4. **Postgres 优先**——多节点数据共享；相比 SQLite 单写者有本质优势

## 后果与约束

### 正面

- 新表遵循同样模式：编号 SQL 迁移文件，薄 repo 包装
- asyncpg 连接池与 FastAPI lifespan 无缝集成

### 需要评估

- 若无故新增 3+ 表（agent_registry、task_queue 等）且不重构，**必须重新评估 SQLAlchemy**
- psql 迁移器故意设计简陋——无回滚能力，回滚 = 手动 SQL；对早期 schema 变动可接受

### 开发摩擦

- asyncpg JSONB 返回类型怪癖（见 2026-04-19 KB friction 条目）——需要 `dict(row)` 转换或自定义类型

## 相关文档

- **执行计划**：`../../../trip-os/docs/superpowers/plans/2026-04-19-phase1a-web-channel.md`
- **路线图**：`../../../trip-os/docs/superpowers/plans/2026-04-19-distributed-agent-roadmap.md`（P2/P3 会分层加入 agent_registry + task_queue 表）
- **实现**：`../../../trip-os/explorer/storage/session_repo.py`、`../../../trip-os/infra/postgres/migrate.py`

## 未来触发点

- P2 引入多节点时——验证 Postgres 连接池在分布式下的一致性需求
- Schema 超过 3 表时——重评 SQLAlchemy + Alembic
