---
tool-category: code-intelligence
tags: [lsp, language-server, go-to-definition, references, symbols]
sources: [claude-code, openharness]
difficulty: intermediate
related_wiki: [wiki/tool-system]
related_tools: [git-worktree]
---

# Code Intelligence Tools (LSP)

> 通过 Language Server Protocol 实现精确的代码导航——跳转定义、查找引用、查看符号，比文本搜索更准确、更快。

## 工具列表

| 工具名 | 来源 | 用途 |
|--------|------|------|
| `LSP` | Claude Code | 全功能 LSP 客户端，支持 definition/references/hover/symbols/call hierarchy |
| `lsp` | OpenHarness | LSP 操作，支持 document_symbol/workspace_symbol/go_to_definition/find_references/hover |

---

## Schema

### Claude Code — `LSP`

```json
{
  "name": "LSP",
  "description": "通过 Language Server Protocol 执行代码智能操作，需要对应语言的语言服务器已在运行。",
  "input_schema": {
    "type": "object",
    "properties": {
      "operation": {
        "type": "string",
        "enum": [
          "go-to-definition",
          "find-references",
          "hover",
          "document-symbols",
          "workspace-symbols",
          "call-hierarchy-incoming",
          "call-hierarchy-outgoing"
        ],
        "description": "要执行的 LSP 操作"
      },
      "file_path": {
        "type": "string",
        "description": "目标文件的绝对路径"
      },
      "line": {
        "type": "integer",
        "description": "光标所在行号（0-based），go-to-definition/find-references/hover/call-hierarchy 必填"
      },
      "character": {
        "type": "integer",
        "description": "光标所在列号（0-based）"
      },
      "query": {
        "type": "string",
        "description": "符号查询字符串，workspace-symbols 必填"
      }
    },
    "required": ["operation", "file_path"]
  }
}
```

**示例调用：**

```json
{
  "operation": "go-to-definition",
  "file_path": "/project/src/agent/executor.py",
  "line": 42,
  "character": 15
}
```

```json
{
  "operation": "workspace-symbols",
  "file_path": "/project/src",
  "query": "ToolExecutor"
}
```

```json
{
  "operation": "call-hierarchy-incoming",
  "file_path": "/project/src/agent/executor.py",
  "line": 10,
  "character": 8
}
```

---

### OpenHarness — `lsp`

```json
{
  "name": "lsp",
  "description": "执行 LSP 代码智能操作。",
  "input_schema": {
    "type": "object",
    "properties": {
      "operation": {
        "type": "string",
        "enum": [
          "document_symbol",
          "workspace_symbol",
          "go_to_definition",
          "find_references",
          "hover"
        ],
        "description": "LSP 操作类型"
      },
      "path": {
        "type": "string",
        "description": "文件路径（document_symbol/go_to_definition/find_references/hover）或目录路径（workspace_symbol）"
      },
      "line": {
        "type": "integer",
        "description": "行号（1-based），go_to_definition/find_references/hover 必填"
      },
      "column": {
        "type": "integer",
        "description": "列号（1-based）"
      },
      "query": {
        "type": "string",
        "description": "符号查询字符串，workspace_symbol 必填"
      },
      "include_declaration": {
        "type": "boolean",
        "description": "find_references 时是否包含声明位置，默认 true"
      }
    },
    "required": ["operation", "path"]
  }
}
```

**示例调用：**

```json
{
  "operation": "find_references",
  "path": "/project/src/executor.py",
  "line": 10,
  "column": 5,
  "include_declaration": false
}
```

---

## 跨项目对比

| 维度 | Claude Code `LSP` | OpenHarness `lsp` |
|------|-------------------|-------------------|
| 行号基准 | 0-based | 1-based |
| Call hierarchy | 支持（incoming/outgoing） | 不支持 |
| 符号查询 | workspace-symbols | workspace_symbol |
| 文件符号 | document-symbols | document_symbol |
| hover 支持 | 支持 | 支持 |
| 命名风格 | kebab-case | snake_case |

**注意行号基准差异。** Claude Code 用 0-based 行号，OpenHarness 用 1-based。从 Read 工具拿到的行号（1-based）直接传给 OpenHarness，传给 Claude Code 需要减 1。

---

## 最佳实践

**先 Grep 粗搜，再 LSP 精定位。** Grep 找到候选文件和大概行号，然后用 LSP 的 go-to-definition 或 find-references 获得精确的跨文件跳转。单纯文本搜索会漏掉通过接口/继承调用的路径。

**LSP 需要语言服务器运行。** 不是开箱即用：Python 需要 `pylsp`/`pyright`，TypeScript 需要 `typescript-language-server`，Go 需要 `gopls`。在 CI 或无头环境中使用前确认语言服务器已启动。

**call-hierarchy 是理解复杂调用链的利器。** 想知道"谁调用了这个函数"用 incoming，想知道"这个函数调用了哪些"用 outgoing。比逐层 find-references 效率高得多。

**workspace-symbols 替代全局 grep 找类/函数。** 搜索类名、函数名时，`workspace-symbols` 比 `grep -r "class Foo"` 更准确，不会误匹配注释或字符串。

**document-symbols 快速获取文件结构。** 打开一个陌生文件时，先用 document-symbols 列出所有类和方法，比从头读文件效率高。

---

## 关联

- [[cookbook/tools/definitions/git-worktree]] — 在 worktree 隔离环境中做代码修改时，LSP 仍然适用
- [[wiki/tool-system]] — LSP 工具属于 code intelligence 工具分类的典型实现
