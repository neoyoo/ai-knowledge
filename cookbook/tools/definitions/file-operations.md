---
tool-category: file-operations
tags: [file-read, file-write, edit, glob, grep, notebook, core]
sources: [claude-code, deer-flow, openharness]
related_wiki: [wiki/tool-system, wiki/context-memory]
---

# File Operations

> 文件系统的六把钥匙：Read/Write/Edit/Glob/Grep/NotebookEdit——几乎所有 Agent 都离不开这组工具，是代码生成、文档处理、数据管道的基础设施。

## 工具一览

| 工具名 | 作用 | Claude Code | DeerFlow | OpenHarness |
|--------|------|:-----------:|:--------:|:-----------:|
| Read / read_file | 读文件内容，支持部分读取 | Read | read_file | read_file |
| Write / write_file | 写入或覆盖文件 | Write | write_file | write_file |
| Edit / str_replace / edit_file | 精确字符串替换 | Edit | str_replace | edit_file |
| Glob / glob | 按 glob 模式列出文件路径 | Glob | glob | glob |
| Grep / grep | 按正则搜索文件内容 | Grep | grep | grep |
| NotebookEdit / notebook_edit | 编辑 Jupyter notebook 单个 cell | NotebookEdit | — | notebook_edit |

---

## 1. Read / read_file

读取文本文件内容，返回带行号的输出。支持 offset/limit 参数做部分读取，避免一次性加载超大文件消耗 token。

**可复制 schema（Anthropic tool_use 格式）：**

```json
{
  "name": "read_file",
  "description": "Read a text file from the local filesystem. Returns content with line numbers. Use offset/limit to read large files in chunks.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute path to the file"
      },
      "offset": {
        "type": "integer",
        "description": "Zero-based line number to start reading from. Use with limit to read a specific range."
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of lines to read. Default 200, max 2000."
      }
    },
    "required": ["path"]
  }
}
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| path | string | 是 | 文件绝对路径 |
| offset | integer | 否 | 从第几行开始读（0-based）。不传则从头开始 |
| limit | integer | 否 | 最多读多少行。默认 200，最大 2000 |

**三个项目的差异：**

- **Claude Code `Read`**：支持图片、PDF（需 pages 参数）、Jupyter notebook（自动渲染 cell 输出）；PDF 超过 10 页必须指定 pages 范围
- **DeerFlow `read_file`**：参数名用 `start_line`/`end_line`（1-indexed，inclusive），路径使用虚拟路径系统（`/mnt/workspace/`），额外必填 `description` 参数用于审计
- **OpenHarness `read_file`**：参数名用 `offset`/`limit`（0-based start，行数 limit），默认 200 行，最大 2000 行，纯文本只读

---

## 2. Write / write_file

创建新文件或覆盖现有文件的完整内容。写入前建议先用 Read 了解现有内容；要局部修改请用 Edit 而非 Write。

**可复制 schema：**

```json
{
  "name": "write_file",
  "description": "Create or completely overwrite a text file. For partial edits, use edit_file instead. Parent directories are created automatically.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute path to the file to write"
      },
      "content": {
        "type": "string",
        "description": "Full file contents to write"
      },
      "create_directories": {
        "type": "boolean",
        "description": "Automatically create parent directories if they don't exist. Default true.",
        "default": true
      }
    },
    "required": ["path", "content"]
  }
}
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| path | string | 是 | 文件绝对路径 |
| content | string | 是 | 要写入的完整内容 |
| create_directories | boolean | 否 | 是否自动创建父目录，默认 true |

**三个项目的差异：**

- **Claude Code `Write`**：要求写入前必须先 Read（否则工具报错）；不支持 append 模式
- **DeerFlow `write_file`**：支持 `append: bool` 参数（默认 false），可追加写入；额外必填 `description` 参数；路径走虚拟路径系统
- **OpenHarness `write_file`**：有 `create_directories` 参数（默认 true），自动创建多级目录；不支持 append

---

## 3. Edit / str_replace / edit_file

在已有文件中做精确字符串替换。比 Write 安全——只改你指定的部分，不会意外覆盖其他内容。`old_string` 在文件中必须唯一（或用 `replace_all` 替换所有出现）。

**可复制 schema：**

```json
{
  "name": "edit_file",
  "description": "Replace a specific string in an existing file. The old_string must appear exactly once unless replace_all is true. Prefer this over write_file for modifications.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute path to the file to edit"
      },
      "old_string": {
        "type": "string",
        "description": "The exact text to replace. Must be unique in the file (or use replace_all)."
      },
      "new_string": {
        "type": "string",
        "description": "The replacement text"
      },
      "replace_all": {
        "type": "boolean",
        "description": "If true, replace every occurrence of old_string. Default false.",
        "default": false
      }
    },
    "required": ["path", "old_string", "new_string"]
  }
}
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| path | string | 是 | 文件绝对路径 |
| old_string | string | 是 | 要被替换的原始文本，必须在文件中能精确匹配 |
| new_string | string | 是 | 替换后的新文本 |
| replace_all | boolean | 否 | 是否替换所有出现，默认只替换第一个 |

**三个项目的差异：**

- **Claude Code `Edit`**：参数名 `old_string`/`new_string`；如果 `old_string` 在文件中不唯一会报错，要求你提供更多上下文使其唯一；支持 `replace_all`
- **DeerFlow `str_replace`**：参数名 `old_str`/`new_str`；默认 `replace_all=False` 时要求 old_str 在文件中**恰好出现一次**（否则报错）；额外必填 `description` 参数
- **OpenHarness `edit_file`**：参数名 `old_str`/`new_str`，支持 `replace_all`；不严格要求唯一性（默认只替第一次）

---

## 4. Glob

按 glob 模式列出匹配的文件路径。用于在动手搜索内容之前，先把候选文件范围缩小到可控规模。

**可复制 schema：**

```json
{
  "name": "glob",
  "description": "List files matching a glob pattern. Use before grep to narrow the search scope. Returns file paths sorted by modification time.",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": {
        "type": "string",
        "description": "Glob pattern to match, e.g. '**/*.py' or 'src/**/*.ts'"
      },
      "path": {
        "type": "string",
        "description": "Root directory to search under. Defaults to working directory."
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of results to return. Default 200, max 5000.",
        "default": 200
      }
    },
    "required": ["pattern"]
  }
}
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pattern | string | 是 | glob 模式，如 `**/*.py`、`src/**/*.ts` |
| path | string | 否 | 搜索根目录，不填则用当前工作目录 |
| limit | integer | 否 | 最多返回多少条结果 |

**三个项目的差异：**

- **Claude Code `Glob`**：结果按**修改时间倒序**排列（最近改动的在前）；参数名 `path`（可选）
- **DeerFlow `glob`**：额外支持 `include_dirs: bool`（是否包含目录，默认 false）；额外必填 `description` 参数；路径走虚拟路径系统
- **OpenHarness `glob`**：参数名 `root`（可选搜索根）；结果按字母序排列（非修改时间）

---

## 5. Grep

在文件内容中按正则表达式搜索，返回匹配的行及行号。基于 ripgrep（Claude Code）或纯 Python 实现，支持多种过滤维度。

**可复制 schema：**

```json
{
  "name": "grep",
  "description": "Search file contents with a regular expression. Returns matching lines with file path and line number. Use glob parameter to restrict which files to search.",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": {
        "type": "string",
        "description": "Regular expression pattern to search for"
      },
      "path": {
        "type": "string",
        "description": "Root directory to search under"
      },
      "glob": {
        "type": "string",
        "description": "Glob filter for candidate files, e.g. '**/*.py'. Limits which files are searched."
      },
      "case_sensitive": {
        "type": "boolean",
        "description": "Whether matching is case-sensitive. Default false (case-insensitive).",
        "default": false
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of matching lines to return. Default 100.",
        "default": 100
      }
    },
    "required": ["pattern", "path"]
  }
}
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pattern | string | 是 | 正则表达式（DeerFlow 可选 `literal: true` 当普通字符串用） |
| path | string | 是（DeerFlow/OH）| 搜索根目录 |
| glob | string | 否 | 限制搜索哪些文件，如 `**/*.ts` |
| case_sensitive | boolean | 否 | 大小写是否敏感，各项目默认不同 |
| limit | integer | 否 | 最大返回行数 |

**三个项目的差异：**

- **Claude Code `Grep`**：基于 ripgrep，性能最强；支持 `output_mode`（content/files_with_matches/count）、上下文行数（`-A`/`-B`/`-C`）、`type` 过滤（如 `type: "py"`）、`multiline` 跨行匹配；`case_sensitive` 默认 **true**
- **DeerFlow `grep`**：额外支持 `literal: bool`（关闭正则，按字面量搜索）；`case_sensitive` 默认 **false**；额外必填 `description` 参数；最大 500 条结果
- **OpenHarness `grep`**：纯 Python 实现（无 ripgrep 依赖）；参数名 `file_glob`（而非 `glob`）；`case_sensitive` 默认 **true**；最大 2000 行

---

## 6. NotebookEdit / notebook_edit

直接编辑 Jupyter notebook（`.ipynb`）的单个 cell，无需序列化整个 notebook。支持替换或追加 cell 内容，也可创建新 cell。

**DeerFlow 没有此工具。**

**可复制 schema：**

```json
{
  "name": "notebook_edit",
  "description": "Create or edit a single cell in a Jupyter notebook (.ipynb). Can replace or append content, and will create the notebook if it doesn't exist.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute path to the .ipynb file"
      },
      "cell_index": {
        "type": "integer",
        "description": "Zero-based index of the cell to edit or create",
        "minimum": 0
      },
      "new_source": {
        "type": "string",
        "description": "New source code or markdown text for the cell"
      },
      "cell_type": {
        "type": "string",
        "enum": ["code", "markdown"],
        "description": "Cell type. Default 'code'.",
        "default": "code"
      },
      "mode": {
        "type": "string",
        "enum": ["replace", "append"],
        "description": "Whether to replace the cell content or append to it. Default 'replace'.",
        "default": "replace"
      },
      "create_if_missing": {
        "type": "boolean",
        "description": "Create the notebook if it doesn't exist. Default true.",
        "default": true
      }
    },
    "required": ["path", "cell_index", "new_source"]
  }
}
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| path | string | 是 | .ipynb 文件的绝对路径 |
| cell_index | integer | 是 | 0-based cell 下标，超出范围会自动追加新 cell |
| new_source | string | 是 | 新的 cell 内容 |
| cell_type | "code"\|"markdown" | 否 | Cell 类型，默认 code |
| mode | "replace"\|"append" | 否 | replace 覆盖原内容，append 在末尾追加 |
| create_if_missing | boolean | 否 | 文件不存在时自动创建，默认 true |

**两个项目的差异：**

- **Claude Code `NotebookEdit`**：专注于替换 cell 的 `new_source`；读取 notebook 用专门的 `Read` 工具（支持渲染 cell 输出）
- **OpenHarness `notebook_edit`**：额外支持 `mode`（replace/append）和 `create_if_missing`；无需依赖 nbformat 库，直接操作 JSON

---

## 跨项目设计对比

| 维度 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| **路径策略** | 宿主机绝对路径 | 虚拟路径（`/mnt/workspace/`），沙箱内隔离 | 宿主机绝对路径（相对路径自动转换） |
| **安全机制** | Write 要求先 Read；Edit 要求 old_string 唯一 | 沙箱隔离 + 路径白名单 + 审计日志（description 参数） | read_only 标记；Edit 检查文件存在；无强制先读约束 |
| **部分读取** | offset(0-based 行号) + limit(行数) | start_line/end_line(1-indexed，inclusive) | offset(0-based 行号) + limit(行数，max 2000) |
| **追加写入** | 不支持 | write_file 有 append 参数 | 不支持（Edit/Write 均全量） |
| **编辑机制** | 精确字符串匹配，old_string 不唯一报错 | 同精确匹配，old_str 不唯一报错 | 精确匹配，replace_all 可批量替换 |
| **Glob 排序** | 按修改时间倒序 | 按路径字母序 | 按路径字母序 |
| **Grep 实现** | 基于 ripgrep（外部依赖） | 沙箱内执行（本地或 Docker） | 纯 Python（无外部依赖） |
| **审计追踪** | 无 | 每个工具调用必填 description，便于日志审计 | 无 |
| **多媒体读取** | Read 支持图片/PDF/Notebook | 不支持 | 不支持（二进制文件报错） |

---

## 最佳实践

**先 Glob 再 Grep，先缩范围再搜内容。** 直接对整个代码库 Grep 会返回大量噪声并消耗不必要的 token。用 `Glob` 定位到相关目录或文件类型，再对这些文件跑 `Grep`，精度和效率都更好。

```json
// 第一步：找到所有配置文件
{"name": "glob", "input": {"pattern": "**/*.config.ts", "path": "/project/src"}}

// 第二步：在这些文件里搜具体内容
{"name": "grep", "input": {"pattern": "timeout", "path": "/project/src", "glob": "**/*.config.ts"}}
```

**Edit 优于 Write，Write 用于新建。** 修改现有文件时用 Edit——它只改你指定的部分，不会意外覆盖其他内容。只有创建全新文件，或需要整体重写，才用 Write。

**大文件用 offset/limit 分段读取。** 一次读完 5000 行会大量消耗 context window。先读文件头部了解结构（`limit: 50`），再用 Grep 定位目标行号，最后用 `offset`/`limit` 精准读取所需段落。

```json
// 错误做法：一次读全部
{"name": "read_file", "input": {"path": "/large-file.py"}}

// 正确做法：先定位再精读
{"name": "grep", "input": {"pattern": "class MyAgent", "path": "/", "glob": "**/large-file.py"}}
// 得到行号后：
{"name": "read_file", "input": {"path": "/large-file.py", "offset": 120, "limit": 80}}
```

**Edit 前先 Read，确认 old_string 唯一。** 如果你要替换的字符串在文件中出现多次，Edit 会报错（Claude Code/DeerFlow）或替换所有（OpenHarness 默认第一个）。Read 一下，确认上下文足够唯一，或适当扩大 old_string 的匹配范围。

**DeerFlow 的 description 参数要认真填。** DeerFlow 每个文件工具都有必填的 `description` 参数，不是形式主义——这些描述会写入审计日志，用于操作溯源。填 "读取配置文件以检查超时设置" 比填 "read config" 有用得多。

**Grep pattern 记得转义特殊字符。** 点号（`.`）、括号、星号在正则中有特殊含义。搜索类名 `MyAgent.run()` 时记得转义成 `MyAgent\.run\(\)`，或使用 DeerFlow 的 `literal: true` 参数跳过正则解析。

---

## 关联

- [[wiki/tool-system]] — 工具系统的架构设计：工具注册、调度、权限控制
- [[wiki/context-memory]] — 上下文管理：文件操作与 Agent 记忆系统的关系
- [[cookbook/tools/definitions/shell-execution]] — Shell 工具：Bash 执行文件操作的另一种方式（更灵活，安全边界更宽）
- [[cookbook/prompts/patterns/react]] — ReAct 模式：文件操作工具在 Agent 循环中的典型用法
- [[cookbook/prompts/patterns/structured-output]] — 工具定义本身就是 Structured Output 的应用
