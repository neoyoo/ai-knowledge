---
tool-category: git-worktree
tags: [git, worktree, isolation, branch, parallel-development]
sources: [claude-code, openharness]
difficulty: intermediate
related_wiki: [wiki/tool-system, wiki/multi-agent]
related_tools: [code-intelligence]
---

# Git Worktree Tools

> 为 agent 创建隔离的工作目录——在不污染主工作区的情况下开发 feature、实验或并行任务。

## 工具列表

| 工具名 | 来源 | 用途 |
|--------|------|------|
| `EnterWorktree` | Claude Code | 创建并进入隔离 worktree，绑定新分支 |
| `ExitWorktree` | Claude Code | 退出 worktree，自动清理无变更的工作区 |
| `enter_worktree` | OpenHarness | 创建 worktree，支持 branch/path/create_branch/base_ref 参数 |
| `exit_worktree` | OpenHarness | 退出并清理 worktree |

---

## Schema

### Claude Code — `EnterWorktree`

```json
{
  "name": "EnterWorktree",
  "description": "创建一个 git worktree 隔离工作区并进入。需要用户显式请求，agent 不应自行决定创建。",
  "input_schema": {
    "type": "object",
    "properties": {
      "branch": {
        "type": "string",
        "description": "要创建或切换到的分支名"
      },
      "path": {
        "type": "string",
        "description": "worktree 目录路径，默认自动生成"
      },
      "base_branch": {
        "type": "string",
        "description": "基础分支，新分支从此分支创建，默认为 main/master"
      }
    },
    "required": ["branch"]
  }
}
```

**示例调用：**

```json
{
  "branch": "feature/add-caching",
  "base_branch": "main"
}
```

---

### Claude Code — `ExitWorktree`

```json
{
  "name": "ExitWorktree",
  "description": "退出当前 worktree。如果 worktree 中没有未提交变更，自动清理目录和分支。",
  "input_schema": {
    "type": "object",
    "properties": {
      "force_cleanup": {
        "type": "boolean",
        "description": "是否强制清理（即使有未提交变更），默认 false",
        "default": false
      }
    }
  }
}
```

---

### OpenHarness — `enter_worktree`

```json
{
  "name": "enter_worktree",
  "description": "创建并进入 git worktree 隔离工作区。",
  "input_schema": {
    "type": "object",
    "properties": {
      "branch": {
        "type": "string",
        "description": "目标分支名"
      },
      "path": {
        "type": "string",
        "description": "worktree 路径，不指定则自动生成临时目录"
      },
      "create_branch": {
        "type": "boolean",
        "description": "是否创建新分支（branch 不存在时），默认 true",
        "default": true
      },
      "base_ref": {
        "type": "string",
        "description": "新分支的起点 ref（commit/branch/tag），默认 HEAD"
      }
    },
    "required": ["branch"]
  }
}
```

**示例调用：**

```json
{
  "branch": "experiment/new-retrieval",
  "create_branch": true,
  "base_ref": "main"
}
```

---

### OpenHarness — `exit_worktree`

```json
{
  "name": "exit_worktree",
  "description": "退出当前 worktree 并可选地清理。",
  "input_schema": {
    "type": "object",
    "properties": {
      "cleanup": {
        "type": "boolean",
        "description": "退出后是否清理 worktree 目录，默认 true",
        "default": true
      }
    }
  }
}
```

---

## 跨项目对比

| 维度 | Claude Code | OpenHarness |
|------|-------------|-------------|
| 自主创建策略 | 需要用户显式请求 | agent 可自行判断 |
| 自动清理条件 | 无变更自动清理 | 由 `cleanup` 参数控制 |
| base_ref 支持 | base_branch（分支名） | base_ref（任意 ref） |
| 路径自定义 | 支持 path 参数 | 支持 path 参数 |
| 强制清理 | force_cleanup 标志 | cleanup=true 覆盖 |

**关键差异：自主性策略。** Claude Code 要求用户显式请求才能创建 worktree，强调人在循环；OpenHarness 允许 agent 自主决策。这反映了两个框架在 agent 自主权上的不同哲学。

---

## 最佳实践

**feature work 用 worktree 隔离。** 当任务涉及多个文件的修改、或有实验性质时，worktree 让主工作区保持干净。出错了直接丢弃 worktree，不影响主分支。

**完成后务必清理。** 每个 worktree 是独立的 git 工作目录，会占用磁盘空间。用 `ExitWorktree` 而不是直接删除目录，确保 git 的 worktree 注册表也被清理。

**parallel agents 的天然隔离机制。** 多个 agent 并行工作时，每个 agent 使用独立 worktree，避免文件系统冲突。这是 multi-agent 并行开发的标准模式。

**不要在 worktree 里做长期工作后忘记 commit。** worktree 的变更不会自动合并回主分支。完成工作后要 commit，然后在主工作区 merge 或 rebase。

**base_ref 指向稳定点。** 创建 worktree 时的 base_ref 选已经过测试的 commit 或发布 tag，而不是正在开发中的功能分支。

---

## 关联

- [[cookbook/tools/definitions/code-intelligence]] — LSP 工具在 worktree 隔离环境下同样可用
- [[wiki/multi-agent]] — worktree 是 multi-agent 并行执行的推荐隔离机制
- [[wiki/tool-system]] — worktree 工具属于环境管理工具类别
