---
tool-category: web-tools
tags: [search, fetch, web, internet, information-retrieval, core]
sources: [claude-code, deer-flow, openharness]
---

# Web Tools

> 让 agent 能访问互联网——搜索最新信息、抓取网页内容、查找图片——是打破训练数据截止日期限制的核心能力。

## 本质

LLM 的训练集有截止日期，内部知识无法更新。Web 工具让 agent 能实时获取外部信息，从"博学但滞后的专家"变成"能随时查资料的分析师"。

三个工具的协作模式几乎是固定的：

1. **WebSearch** — 给关键词，得到 URL 列表 + 摘要（广撒网）
2. **WebFetch** — 给 URL，得到页面完整内容（深挖单页）
3. **ImageSearch**（DeerFlow 独有）— 给关键词，得到图片 URL 列表

三个项目对 WebSearch 和 WebFetch 都有实现，但 backend 可插拔性差异很大：DeerFlow 支持 4 种搜索 backend 和 3 种抓取 backend，Claude Code 内置固定实现，OpenHarness 支持配置 URL 但不换 backend。

## 工具清单

| 工具名 | 项目 | 核心能力 | 特有特性 |
|--------|------|----------|----------|
| `WebSearch` | Claude Code | 搜索网络，返回 URL+摘要 | 快速 model 处理结果，仅限美国 IP |
| `web_search` | DeerFlow | 搜索网络 | 4 种可插拔 backend |
| `web_search` | OpenHarness | 搜索网络 | 可配置 `search_url` |
| `WebFetch` | Claude Code | 抓取页面，HTML→Markdown | 快速 model 提取相关内容 |
| `web_fetch` | DeerFlow | 抓取页面 | 3 种可插拔 backend |
| `web_fetch` | OpenHarness | 抓取页面 | `max_chars` 限制 |
| `image_search` | DeerFlow | 搜索图片，返回图片 URL | DeerFlow 独有 |

## 可复制 Schema

### Claude Code — WebSearch

```python
{
    "name": "WebSearch",
    "description": "搜索互联网获取最新信息。注意：仅在美国 IP 可用，搜索结果由快速模型预处理。",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询词。使用英文效果更好，支持自然语言和关键词两种风格。"
            },
            "allowed_domains": {
                "type": "array",
                "items": {"type": "string"},
                "description": "限制搜索结果来源域名，例如 ['github.com', 'docs.python.org']。"
            },
            "blocked_domains": {
                "type": "array",
                "items": {"type": "string"},
                "description": "排除特定域名的搜索结果。"
            }
        },
        "required": ["query"]
    }
}
```

**关键行为：**
- 结果由内置快速 model 预处理，提取最相关部分，减少 token 消耗
- `allowed_domains` 适合在特定文档站（如 MDN、PyPI）内精准搜索
- 受地理限制（美国 IP），部分区域需要代理

### Claude Code — WebFetch

```python
{
    "name": "WebFetch",
    "description": "抓取指定 URL 的网页内容，转换为 Markdown 格式，并用快速模型提取与 prompt 相关的内容。",
    "input_schema": {
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "要抓取的完整 URL，包含协议（https://）。"
            },
            "prompt": {
                "type": "string",
                "description": "描述你想从页面中提取什么信息。快速模型会根据此 prompt 筛选内容。"
            }
        },
        "required": ["url", "prompt"]
    }
}
```

**关键行为：**
- HTML 自动转换为 Markdown，去除导航栏、广告等噪声
- `prompt` 参数驱动内容提取：填得越具体，返回越精准
- 底层用快速 model（如 Haiku）处理，成本低
- 不支持需要登录的页面或 JavaScript 渲染的单页应用

### DeerFlow — web_search

```python
{
    "name": "web_search",
    "description": "使用配置的 backend 搜索互联网。支持 DuckDuckGo、Tavily、Firecrawl、InfoQuest。",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询词。"
            },
            "max_results": {
                "type": "integer",
                "description": "最多返回多少条结果。默认 5，推荐范围 3–10。",
                "default": 5,
                "minimum": 1,
                "maximum": 20
            }
        },
        "required": ["query"]
    }
}
```

**Backend 选项（配置层决定，agent 不感知）：**

| Backend | 特点 | 适用场景 |
|---------|------|----------|
| DuckDuckGo | 免费，无需 API key，有速率限制 | 开发测试、低频搜索 |
| Tavily | 专为 AI 优化，结果质量高，需付费 | 生产环境推荐 |
| Firecrawl | 支持深度爬取，结果更完整 | 需要详细内容的场景 |
| InfoQuest | 企业级，支持私有数据源 | 企业内网搜索 |

### DeerFlow — web_fetch

```python
{
    "name": "web_fetch",
    "description": "抓取指定 URL 的网页内容，转换为干净的文本格式。",
    "input_schema": {
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "要抓取的完整 URL。"
            },
            "max_length": {
                "type": "integer",
                "description": "返回内容的最大字符数。默认 10000。",
                "default": 10000
            }
        },
        "required": ["url"]
    }
}
```

**Backend 选项：**

| Backend | 特点 | 适用场景 |
|---------|------|----------|
| Jina Reader | 免费，HTML→Markdown，速度快 | 通用网页抓取 |
| Tavily Extract | 结果质量高，与搜索 backend 配套 | 与 Tavily 搜索配合使用 |
| Firecrawl | 支持 JavaScript 渲染，绕过反爬 | 动态网页、单页应用 |

### DeerFlow — image_search

```python
{
    "name": "image_search",
    "description": "搜索互联网上的图片，返回图片 URL 列表及描述。",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "图片搜索关键词，英文效果更好。"
            },
            "max_results": {
                "type": "integer",
                "description": "最多返回多少张图片。默认 5。",
                "default": 5,
                "minimum": 1,
                "maximum": 10
            }
        },
        "required": ["query"]
    }
}
```

**Backend 选项：** DuckDuckGo Images（免费）、InfoQuest Images（企业级）。

### OpenHarness — web_search

```python
{
    "name": "web_search",
    "description": "使用配置的搜索服务搜索网络内容。",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询词。"
            },
            "max_results": {
                "type": "integer",
                "description": "最多返回多少条结果。默认 5。",
                "default": 5
            },
            "search_url": {
                "type": "string",
                "description": "覆盖默认搜索服务的 URL。可指向内部搜索服务或自定义 API。"
            }
        },
        "required": ["query"]
    }
}
```

### OpenHarness — web_fetch

```python
{
    "name": "web_fetch",
    "description": "抓取指定 URL 的网页内容。",
    "input_schema": {
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "要抓取的完整 URL。"
            },
            "max_chars": {
                "type": "integer",
                "description": "返回内容的最大字符数。默认 8000，用于控制 token 消耗。",
                "default": 8000
            },
            "start_index": {
                "type": "integer",
                "description": "从第几个字符开始返回，配合 max_chars 实现分页读取。",
                "default": 0
            }
        },
        "required": ["url"]
    }
}
```

**关键行为：**
- `start_index` + `max_chars` 组合支持分页读取长页面
- 无内置内容提取逻辑，返回原始 HTML 转换的文本
- `search_url` 可以指向企业内网搜索服务，适合私有部署

## 跨项目对比

### Backend 可插拔性

这是三个实现最核心的设计差异。

| 维度 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| **搜索 backend 数** | 1（固定，不可换） | 4（DuckDuckGo/Tavily/Firecrawl/InfoQuest） | 1 + 可配置 URL |
| **抓取 backend 数** | 1（固定，含模型处理） | 3（Jina/Tavily/Firecrawl） | 1（纯 HTTP） |
| **换 backend 方式** | 不支持 | 配置文件，agent 透明 | `search_url` 参数 |
| **JavaScript 渲染** | 不支持 | 支持（Firecrawl backend） | 不支持 |
| **私有数据源** | 不支持 | 支持（InfoQuest） | 支持（自定义 URL） |

DeerFlow 的 backend 切换对 agent 完全透明——同样的 `web_search` 调用，底层可以是 DuckDuckGo 或 Tavily，agent 的 prompt 完全不用改。这是最灵活的架构，代价是配置复杂度。

### 结果处理

拿到网页内容后，三个项目的处理方式差异显著：

| 项目 | HTML 处理 | 内容筛选 | 返回格式 |
|------|-----------|----------|----------|
| Claude Code | HTML→Markdown（内置） | 快速 model 按 prompt 提取 | 精炼 Markdown |
| DeerFlow | 依赖 backend（Jina/Firecrawl 各自处理） | 无额外筛选，返回完整内容 | Markdown 或文本 |
| OpenHarness | 基础 HTML strip | 无，返回原始文本 | 纯文本，支持分页 |

Claude Code 的 `prompt` 参数是独特设计：抓取时告诉工具"我要找什么"，快速 model 会帮你从整个页面里筛出相关段落。这在长文档中节省了大量 token。

OpenHarness 的分页方案（`start_index` + `max_chars`）是另一种思路：把长内容切片，让 agent 逐段读取，避免单次返回超过 context 限制。

### 搜索范围限制

| 项目 | 限制方式 | 粒度 |
|------|----------|------|
| Claude Code | `allowed_domains` / `blocked_domains` 参数 | 域名级，每次调用可不同 |
| DeerFlow | 配置文件 backend 选择 | 全局配置，切换 backend 改变数据源 |
| OpenHarness | `search_url` 参数 | 服务级，可指向不同搜索端点 |

### 地理限制

Claude Code 的 WebSearch 明确标注仅限美国 IP。DeerFlow 和 OpenHarness 无此限制（依赖配置的 backend/服务的可访问性）。在非美国环境部署 Claude Code 需要额外处理。

## 最佳实践

### 1. 两步走：先搜索，再深读

WebSearch 的返回是摘要 + URL 列表，不够深；WebFetch 的成本是每次一个 URL，搜一堆再逐个读太慢。正确用法是配合：

```python
# Step 1：搜索定位目标
search_result = web_search(
    query="Python asyncio timeout best practices 2024",
    max_results=5
)
# 得到 5 个 URL + 摘要，快速判断哪个值得深读

# Step 2：只深读最相关的 1-2 个
content = web_fetch(
    url="https://docs.python.org/3/library/asyncio-task.html#timeouts",
    prompt="asyncio timeout 的推荐用法和常见陷阱"
)
```

**规则：WebSearch 负责广度，WebFetch 负责深度。不要对每个搜索结果都 fetch。**

### 2. 用 max_results 控制 token 消耗

搜索结果越多，摘要文本越长，消耗 token 越多。实际上 3–5 条结果通常够用。

```python
# 探索性搜索：3 条够了
web_search(query="...", max_results=3)

# 需要比较多个来源：5–7 条
web_search(query="...", max_results=5)

# 超过 10 条几乎没有额外价值，不推荐
```

### 3. HTML 转 Markdown 减少噪声

直接处理 HTML 会带入大量无关标签、导航栏、广告文本。所有三个项目的 WebFetch 都会做 HTML→文本的转换，但转换质量有差异：

- Claude Code：转 Markdown，快速 model 二次筛选，噪声最少
- DeerFlow + Jina backend：Jina Reader 专门优化了阅读体验
- OpenHarness：基础 strip，可能仍有噪声

如果用 OpenHarness 抓取内容质量不稳定，可以在 prompt 里明确要求模型忽略导航和页脚。

### 4. 长页面用分页读取（OpenHarness）

```python
# 第一段：0–8000 字符
chunk1 = web_fetch(url="...", max_chars=8000, start_index=0)

# 如果需要继续
chunk2 = web_fetch(url="...", max_chars=8000, start_index=8000)
```

分页读取适合 API 文档、长教程等内容密集的页面。但注意：如果 Claude Code 的 `prompt` 参数能精准提取目标内容，通常不需要分页。

### 5. DeerFlow backend 选择建议

```yaml
# 开发 / 测试：免费不限额
search_engine: duckduckgo
web_loader: jina

# 生产环境：质量优先
search_engine: tavily
web_loader: tavily_extract

# 需要处理动态网页（SPA、需要登录的页面）
web_loader: firecrawl
```

切换 backend 只改配置文件，agent prompt 完全不变。

### 6. 图片搜索的使用场景

`image_search`（DeerFlow 独有）主要用于：
- 多模态 agent 需要收集视觉素材
- 验证某个概念的视觉呈现（如"确认这个 UI 元素的外观"）
- 新闻/报告类任务需要配图

注意 `image_search` 返回的是图片 URL，还需要额外的 fetch 或多模态处理才能分析图片内容。

## 关联

- [[cookbook/tools/patterns/search-then-read]] — WebSearch → WebFetch 两步读取模式（工具使用模式）
- [[wiki/tool-system]] — 工具系统架构，工具注册与 backend 切换机制
- [[wiki/context-memory]] — Web 内容的 token 消耗控制与 context 压缩
- [[cookbook/prompts/patterns/rag]] — Web 搜索结果作为动态 RAG 的知识来源

**来源：**
- Claude Code system prompt — WebSearch/WebFetch 描述、`prompt` 参数行为、地理限制说明
- DeerFlow `tools/search.py` — 4 种搜索 backend 注册逻辑和 `max_results` 配置
- DeerFlow `tools/crawl.py` — Jina/Tavily/Firecrawl 三种抓取 backend 实现
- DeerFlow `tools/image_search.py` — DuckDuckGo/InfoQuest 图片搜索实现
- OpenHarness `tools/web_tools.py` — `max_chars`、`start_index`、`search_url` 参数实现
