---
title: "SQLite+LanceDB 双存储记忆架构"
category: insight
source: simplemem
concept: memory-system
created: 2026-04-15
tags: [memory-system, dual-storage, sqlite, lancedb, vector-search]
---

## 核心洞察

用 SQLite 管用户身份与元数据（精确匹配），用 LanceDB 存记忆向量（语义检索），职责边界清晰，两者不交叉。

## 设计方案

SimpleMem MCP 服务器将存储层拆成两个物理数据库，分工明确：

**SQLite 路径（精确匹配）**
- 存储用户注册信息：`user_id`、加密 API key、`table_name`（用于映射到 LanceDB 表）、时间戳
- 场景：用户认证、查找用户对应的向量表名、更新活跃时间
- 查询方式：主键精确查找（`WHERE user_id = ?`）、索引查找（`WHERE table_name = ?`）

**LanceDB 路径（语义检索）**
- 每个用户独占一张 LanceDB 表（`table_name` 字段由 SQLite 维护）
- 存储压缩后的记忆条目 + 嵌入向量（2560 维 float32）+ 结构化元数据（persons/location/entities/timestamp）
- 支持三路并行检索：语义向量搜索、关键词评分匹配、结构化元数据过滤

**关键绑定点**
SQLite 的 `table_name` 字段是两层存储的桥梁。认证流程先查 SQLite 得到 `table_name`，再用 `table_name` 定位 LanceDB 的具体表。两层解耦，彼此不感知对方的内部结构。

## 关键代码证据

```python
# SQLite 层：user_store.py — 用户元数据表
conn.execute("""
    CREATE TABLE IF NOT EXISTS users (
        user_id TEXT PRIMARY KEY,
        openrouter_api_key_encrypted TEXT NOT NULL,
        table_name TEXT NOT NULL UNIQUE,  # ← 绑定到 LanceDB 表
        created_at TEXT NOT NULL,
        last_active TEXT NOT NULL
    )
""")

# 精确查找：通过 user_id 取出 table_name
def get_user(self, user_id: str) -> Optional[User]:
    row = conn.execute("SELECT * FROM users WHERE user_id = ?", (user_id,)).fetchone()
    ...

# LanceDB 层：vector_store.py — 记忆向量表
class MultiTenantVectorStore:
    def _get_table(self, table_name: str) -> Any:
        """Get or create a user's table"""
        if table_name not in self._tables:
            if table_name in self.db.table_names():
                self._tables[table_name] = self.db.open_table(table_name)
            else:
                schema = get_memory_schema(self.embedding_dimension)
                self._tables[table_name] = self.db.create_table(table_name, schema=schema)
        return self._tables[table_name]

    async def semantic_search(self, table_name: str, query_embedding: List[float], top_k: int = 25):
        table = self._get_table(table_name)
        results = table.search(query_embedding).limit(top_k).to_pandas()
        ...
```

## 适用场景

- **多租户 SaaS 服务**：用户身份管理需要精确查找（认证/鉴权），记忆内容需要语义检索，两种需求天然分属不同数据库类型
- **需要数据隔离的 Agent 服务**：每个用户/会话独立一张 LanceDB 表，SQLite 做目录索引，避免跨用户数据污染
- **查询路径分叉明确的系统**：已知查询类型时（精确 vs 语义）可直接路由，不需要混合查询

## 代价与局限

1. **关键词搜索退化为全表扫描**：SimpleMem 的"keyword_search"实际上是把 LanceDB 表全量加载到 pandas DataFrame 再逐行打分，数据量大时性能急剧下降，没有真正利用 BM25 索引
2. **结构化过滤同样全表扫描**：`structured_search` 用 Python 逐行遍历过滤，LanceDB 的 SQL-like filter API 没被用上
3. **两层存储需要事务协调**：删除用户时须同时清理 SQLite 记录和 LanceDB 表，SimpleMem 没有事务保障，存在数据残留风险
4. **SQLite 单点并发瓶颈**：高并发写入时 SQLite 的写锁成为瓶颈，适合小规模场景
5. **LanceDB 无淘汰策略**：记忆只增不减，长期运行后检索噪声累积，SimpleMem 文本版未实现 TTL 或重要性衰减

## 来源
- 项目：simplemem（实验级，仅作参考）
- 完整分析：[[shelf/simplemem/wiki/_impl/memory-system--simplemem]]
- 核心文件：`MCP/server/database/user_store.py`、`MCP/server/database/vector_store.py`
