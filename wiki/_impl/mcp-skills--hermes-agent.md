---
title: "MCP & Skills — Hermes Agent"
category: L2
parent: "[[mcp-skills]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的 MCP 与 Skills 系统由两个完全独立的方向构成：一端是 **MCP Server**（`mcp_serve.py`），将 Hermes 自身的消息对话暴露为标准 MCP 工具，让其他 AI 客户端（Claude Code、Cursor、Codex）可以接入；另一端是 **MCP Client**（`tools/mcp_tool.py`），让 Hermes 自己消费外部 MCP Server 的工具。Skills 系统则是独立的工作流模板层（`tools/skills_tool.py` + `tools/skills_hub.py`），从 `~/.hermes/skills/` 加载 SKILL.md 文件，注入系统提示，并提供一套完整的安全安装管道（隔离 → 扫描 → 安装 → 锁文件记录）。

这套设计区别于 Claude Code 的关键点：Hermes 将"作为 MCP Server 被调用"和"作为 MCP Client 调用外部工具"都实现了，而 Skills Hub 还加入了 GitHub API 多源聚合、OSV 威胁扫描、信任分级安装策略——是一个小型的去中心化 skill 包管理器。

---

## 架构分析

### MCP Server — 将 Hermes 会话桥接为 MCP 工具

**`mcp_serve.py`** 使用 `FastMCP`（`mcp.server.fastmcp`）创建 stdio MCP Server，对外暴露 10 个工具：

| 工具名 | 功能 |
|--------|------|
| `conversations_list` | 列出所有平台的活跃对话（支持 platform/search 过滤） |
| `conversation_get` | 按 session_key 获取对话详情（token 统计、创建时间等） |
| `messages_read` | 读取对话历史（最近 N 条，过滤 user/assistant 角色） |
| `attachments_fetch` | 提取消息中的非文本附件（图片、媒体文件） |
| `events_poll` | 基于 cursor 的无阻塞事件轮询 |
| `events_wait` | 长轮询等待下一个事件（最长 5 分钟，cap 在 `wait_for_event` 中实施） |
| `messages_send` | 向 `platform:chat_id` 目标发送消息 |
| `channels_list` | 列出所有可发送消息的 channel |
| `permissions_list_open` | 列出待审批请求 |
| `permissions_respond` | 响应审批（allow-once / allow-always / deny） |

这 10 个工具与 OpenClaw 的 9 工具 MCP bridge 对齐，额外增加了 `channels_list`（Hermes-specific）。

**EventBridge — 低开销实时事件桥：**

`EventBridge` 类是 MCP Server 的核心运行时，在后台守护线程中以 200ms 间隔轮询 SQLite（`SessionDB`）。关键优化是 **mtime 双重检查**：每次 `_poll_once()` 先比对 `sessions.json` 和 `state.db` 的文件修改时间（约 1μs），只要两者都没变，整个 DB 查询直接跳过，使 200ms 轮询的实际 CPU 开销接近零。

```python
# EventBridge._poll_once — mtime 跳过逻辑
if db_mtime == self._state_db_mtime and sj_mtime == self._sessions_json_mtime:
    return  # Nothing changed since last poll — skip entirely
```

事件队列上限 1000 条（`QUEUE_LIMIT`），以 `cursor` 整数单调递增标记位置。`wait_for_event()` 通过 `threading.Event` 实现阻塞等待，避免了 busy-wait。

**MCP Client 配置样例（`claude_desktop_config.json`）：**
```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

---

### MCP Client — 消费外部 MCP Server 工具

**`tools/mcp_tool.py`** 实现了一个线程安全的 MCP Client，架构核心是 **专用后台 Event Loop**：

```
主线程（同步）
  └── _run_on_mcp_loop(coro) → run_coroutine_threadsafe → MCP background loop (daemon thread)
                                                              └── MCPServerTask (asyncio.Task per server)
                                                                    ├── _run_stdio()  → StdioClientTransport
                                                                    └── _run_http()   → StreamableHTTPClientTransport
```

**传输层支持：**
- **stdio**：`StdioServerParameters` + `stdio_client`，命令 + args + 过滤后的安全环境变量
- **HTTP/StreamableHTTP**：`streamablehttp_client` / `streamable_http_client`（SDK 版本自适应，mcp >= 1.24.0 走新 API）
- **OAuth 2.1 PKCE**：`auth: oauth` 配置项 → `tools/mcp_oauth.build_oauth_auth()`

**安全环境变量过滤（`_build_safe_env`）：**
只透传 `_SAFE_ENV_KEYS`（PATH/HOME/USER/LANG/LC_ALL/TERM/SHELL/TMPDIR）和 `XDG_*` 变量，加上用户 config 中显式指定的变量，防止向 MCP 子进程泄漏 API Key 等机密。

**Sampling 支持（`SamplingHandler`）：**
MCP Server 可发起 `sampling/createMessage` 请求，让 LLM 处理后返回结果。`SamplingHandler` 实现了：
- 滑动窗口速率限制（`max_rpm`，默认 10 次/分钟）
- 模型白名单（`allowed_models`）
- 工具循环防御（`max_tool_rounds`，默认 5 轮）
- MCP 消息格式 → OpenAI format 转换（`_convert_messages`）
- 错误响应中自动脱敏凭证（`_sanitize_error`）

**自动重连：**
`MCPServerTask.run()` 实现指数退避重连，最多 5 次（`_MAX_RECONNECT_RETRIES`），最大退避 60 秒（`_MAX_BACKOFF_SECONDS`）。

**动态工具发现（`tools/list_changed` 通知）：**
`_make_message_handler()` 监听 `ToolListChangedNotification`，触发 `_refresh_tools()`，重新向 server 获取工具列表并原子更新 registry，无需重启即可感知 server 端工具变化。

**Config 加载（`~/.hermes/config.yaml`）：**
```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    timeout: 120
    connect_timeout: 60
  remote_api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer ${MY_TOKEN}"   # ${VAR} 占位符由 _interpolate_env_vars 展开
    sampling:
      enabled: true
      max_rpm: 10
      max_tool_rounds: 5
```

---

### OSV 恶意软件扫描：MCP 包的启动前检查

`tools/osv_check.py` 在每次通过 `npx` 或 `uvx` 启动 MCP server 前，调用 Google OSV API 检查目标包是否存在已知恶意软件通告（MAL-* advisory）。这一机制作为 skills_guard.py 静态扫描的补充，专门面向 MCP 包的**供应链安全**，而不是 skill 内容安全。

**触发时机：**

`check_package_for_malware(command, args)` 在 `tools/mcp_tool.py` 启动 MCP 子进程前调用：只对 `npx`/`npx.cmd`（npm 生态）和 `uvx`/`uvx.cmd`/`pipx`（PyPI 生态）触发，其余命令（如直接调用 Go/Rust 二进制）直接跳过，不访问网络。

**OSV API 查询流程：**

```python
# 1. 从命令和 args 提取包名 + 版本
ecosystem = _infer_ecosystem(command)         # "npm" 或 "PyPI"
package, version = _parse_package_from_args(args, ecosystem)

# 2. 向 OSV API 发 POST 请求
payload = {"package": {"name": package, "ecosystem": ecosystem}}
# 有 version 时加入 "version" 字段，精确匹配该版本的 advisory

# 3. 只保留 MAL-* advisory（忽略普通 CVE）
return [v for v in vulns if v.get("id", "").startswith("MAL-")]
```

MAL-* 是 OSV 数据库中专门标识已确认恶意软件的 advisory 类型，与常规漏洞（CVE）完全分开，避免把"有安全漏洞"误判为"是恶意软件"。

**包名解析：**

| 生态 | 典型格式 | 解析逻辑 |
|------|---------|---------|
| npm | `@scope/name@version` | 正则 `^(@[^/]+/[^@]+)(?:@(.+))?$` |
| npm | `name@version` | `rsplit("@", 1)`，`latest` 版本号忽略 |
| PyPI | `name==version` | 正则，支持 extras `name[extra]==version` |

解析失败（格式无法识别）时返回 `(None, None)`，视为无法判断，放行。

**Fail-open 模型：**

网络错误、超时（默认 10 秒）、JSON 解析失败、命令格式无法识别——所有异常情况一律放行（返回 None），不阻断 MCP server 启动。设计哲学与 tirith 一致：宁可放行未知风险，也不因网络抖动阻断正常工作流。

**被拦截时的返回格式：**

```python
f"BLOCKED: Package '{package}' ({ecosystem}) has known malware "
f"advisories: {ids}. Details: {summaries}"
# ids: 最多展示 3 个 advisory ID（如 MAL-2024-1234）
# summaries: 每条最多 100 字符
```

**与 tirith 扫描的关系：**

| 机制 | 作用目标 | 检测方式 | 触发时机 |
|------|---------|---------|---------|
| OSV 恶意软件扫描 | MCP 包（npm/PyPI） | 在线查询 OSV 数据库 MAL-* advisory | 每次 MCP server 启动前 |
| Tirith 安全扫描 | 终端命令内容 | 本地二进制语义分析 | 每次 terminal 工具执行前 |

两者正交互补：OSV 检查包的**已知恶意软件历史**（供应链视角），tirith 检查命令的**当前执行内容**（运行时视角）。一个 MCP server 包可能通过 OSV 检查（无已知 advisory）但仍被 tirith 拦截（动态生成的危险命令），反之亦然。

---

### Skills 格式 — SKILL.md frontmatter schema

每个 skill 是一个目录，必须包含 `SKILL.md`，可选包含 `references/`、`templates/`、`assets/` 子目录。SKILL.md 的 YAML frontmatter 定义了 skill 的全部元数据：

```yaml
---
name: apple-reminders                    # 必填，最长 64 字符
description: Manage Apple Reminders...  # 必填，最长 1024 字符
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [macos]                       # 平台过滤：macos / linux / windows，缺省 = 全平台
prerequisites:
  commands: [remindctl]                  # 命令可用性检查（advisory）
  env_vars: [API_KEY]                    # 遗留字段，自动归一化为 required_environment_variables
required_environment_variables:          # 新式格式
  - name: OPENAI_API_KEY
    prompt: "Enter your OpenAI key"
    help: "https://platform.openai.com/api-keys"
setup:
  collect_secrets:
    - env_var: OPENAI_API_KEY
      prompt: "OpenAI API key"
      secret: true
metadata:
  hermes:
    tags: [llm, fine-tuning]
    related_skills: [peft, lora]
    fallback_for_toolsets: [browser]     # 当 browser toolset 可用时隐藏此 skill
    requires_tools: [terminal]           # 当 terminal 工具不可用时隐藏
    config:
      - key: wiki.path
        description: "Wiki directory"
        default: "~/wiki"
---
```

**平台匹配（`skill_utils.skill_matches_platform`）：**
`PLATFORM_MAP = {"macos": "darwin", "linux": "linux", "windows": "win32"}`，通过 `sys.platform.startswith()` 匹配。缺省（`platforms` 字段缺失或为空）视为全平台兼容。

---

### Skill 加载与系统提示注入

**`agent/prompt_builder.py` 的 `build_skills_system_prompt()`** 负责将 skill 目录扫描结果注入系统提示，采用两层缓存设计：

1. **In-process LRU**：`_SKILLS_PROMPT_CACHE`（上限 8 条），key 为 `(skills_dir, external_dirs, available_tools, available_toolsets, platform_hint)` 的元组
2. **磁盘快照**：`~/.hermes/.skills_prompt_snapshot.json`，通过 `_build_skills_manifest()` 对比所有 SKILL.md/DESCRIPTION.md 的 mtime_ns + size 做缓存有效性验证，进程重启后快速复用

**条件过滤（`_skill_should_show`）：**
- `fallback_for_toolsets/fallback_for_tools`：当指定的 toolset/tool 已加载时，此 skill 不显示（fallback 语义）
- `requires_toolsets/requires_tools`：当指定的 toolset/tool 未加载时，此 skill 不显示（依赖检查）

**注入格式（系统提示片段）：**
```
## Skills (mandatory)
Before replying, scan the skills below. If one clearly matches your task,
load it with skill_view(name) and follow its instructions. ...

<available_skills>
  apple: macOS native integrations
    - apple-reminders: Manage Apple Reminders via remindctl CLI...
    - imessage: Send and read iMessages via applescript...
  mlops:
    - axolotl: Fine-tune LLMs with axolotl...
</available_skills>
```

**`SKILLS_GUIDANCE` 常量（`agent/prompt_builder.py` 第 164 行）：**
```python
SKILLS_GUIDANCE = (
    "After completing a complex task (5+ tool calls), fixing a tricky error, "
    "or discovering a non-trivial workflow, save the approach as a "
    "skill with skill_manage so you can reuse it next time.\n"
    "When using a skill and finding it outdated, incomplete, or wrong, "
    "patch it immediately with skill_manage(action='patch') — don't wait to be asked. "
    "Skills that aren't maintained become liabilities."
)
```
该常量指导模型在完成复杂任务后主动创建/更新 skill，形成自我进化闭环。

**外部 skill 目录（`skills.external_dirs`）：**
`agent/skill_utils.get_external_skills_dirs()` 从 `~/.hermes/config.yaml` 读取额外的 skills 目录，支持 `~` 和 `${VAR}` 扩展。外部目录与本地目录合并扫描，本地优先（重名 skill 本地覆盖外部）。

---

### Skills Hub — GitHub 多源聚合与安全安装

**`tools/skills_hub.py`** 是 skill 的包管理器，核心数据流：

```
源适配器（SkillSource）
  ├── OptionalSkillSource  — 随 Hermes 附带的官方可选 skills（非默认激活）
  ├── GitHubSource         — GitHub Contents API / Git Trees API
  ├── WellKnownSkillSource — /.well-known/skills/index.json 端点
  └── SkillsShSource       — skills.sh 聚合平台
         ↓ fetch(identifier) → SkillBundle
  quarantine_bundle()      — 写入隔离目录 ~/.hermes/skills/.hub/quarantine/
         ↓
  scan_skill()             — tools/skills_guard.py 静态扫描
         ↓ ScanResult
  should_allow_install()   — 信任分级策略决定
         ↓ 通过
  install_from_quarantine() — 移入 ~/.hermes/skills/{category}/{name}/
         ↓
  HubLockFile.record_install()  — 写入 lock.json
  append_audit_log()            — 追加 audit.log
```

**`GitHubSource` — GitHub API 下载优化：**
`_download_directory_via_tree()` 优先使用 Git Trees API（一次请求获取整个目录树），避免 Contents API 的逐目录请求导致的速率限制；Trees API 响应被截断时才回退到 `_download_directory_recursive()`（Contents API 递归）。索引结果缓存在 `~/.hermes/skills/.hub/index-cache/` 下，TTL 1 小时（`INDEX_CACHE_TTL = 3600`）。

**`GitHubAuth` — 多级认证：**
优先级顺序：`GITHUB_TOKEN`/`GH_TOKEN` 环境变量 → `gh auth token` CLI → GitHub App JWT（`GITHUB_APP_ID` + `GITHUB_APP_PRIVATE_KEY_PATH` + `GITHUB_APP_INSTALLATION_ID`）→ 匿名（60 req/hr）。GitHub App token 有效期缓存 3500 秒（即将满 1 小时时刷新）。

**信任分级（`tools/skills_guard.py`）：**

| 信任级别 | 来源 | safe | caution | dangerous |
|---------|------|------|---------|-----------|
| builtin | Hermes 内置 | allow | allow | allow |
| trusted | openai/skills, anthropics/skills | allow | allow | block |
| community | 其他所有来源 | allow | block | block |
| agent-created | 模型自创 | allow | allow | ask |

`TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}` — 目前仅硬编码 2 个信任仓库。

**静态威胁扫描（`scan_skill()`）：**
`tools/skills_guard.py` 对隔离目录内所有文件执行：
1. 结构检查（文件数量、总大小、二进制文件、符号链接）
2. 正则威胁模式匹配（`THREAT_PATTERNS`），覆盖：
   - 凭证外泄（curl/wget/fetch/httpx 携带 secret 变量）
   - 读取 SSH/AWS/GPG/kubeconfig/Docker/Hermes .env
   - 命令注入（反引号、`$(...)` 拼接外部输入）
   - 破坏性操作（`rm -rf /`、`dd if=/dev/urandom`）
   - 持久化植入（crontab、SSH authorized_keys、systemd unit）
   - 网络回连（ngrok、netcat 反弹 shell）
   - Unicode 不可见字符（用于提示词注入）
3. 可选 LLM 审计（`llm_audit_skill()`）：静态扫描后可接入 LLM 二次审查，LLM verdict 只能提升严重度，不能降低

**`HubLockFile`（`lock.json`）记录的字段：**
```json
{
  "installed": {
    "axolotl": {
      "source": "github",
      "identifier": "openai/skills/skills/axolotl",
      "trust_level": "trusted",
      "scan_verdict": "safe",
      "content_hash": "sha256:abc123...",
      "install_path": "mlops/axolotl",
      "files": ["SKILL.md", "references/config.md"],
      "installed_at": "2026-04-08T00:00:00+00:00",
      "updated_at": "2026-04-08T00:00:00+00:00"
    }
  }
}
```

**`TapsManager`（`taps.json`）：**
用户可通过 `hermes skills hub tap` 添加自定义 GitHub 仓库作为 skill 源，格式 `{"repo": "owner/repo", "path": "skills/"}`。

---

### 内置与可选 Skills 分布

| 类型 | 目录 | 数量 |
|------|------|------|
| 内置 built-in | `skills/`（26 个分类） | 77 个 SKILL.md |
| 可选 optional | `optional-skills/`（14 个分类） | 45 个 SKILL.md |

内置分类：apple、autonomous-ai-agents、creative、data-science、devops、diagramming、dogfood、domain、email、feeds、gaming、gifs、github、inference-sh、leisure、mcp、media、mlops、note-taking、productivity、red-teaming、research、smart-home、social-media、software-development 等。

可选分类：autonomous-ai-agents、blockchain、communication、creative、devops、email、health、mcp、migration、mlops、productivity、research、security 等，默认不激活，需用户手动启用。

---

## 关键代码路径

- `mcp_serve.py` — MCP Server 入口，`create_mcp_server()` 注册 10 个工具，`EventBridge` 负责 SQLite 轮询与事件队列
- `tools/mcp_tool.py` — MCP Client，`MCPServerTask`（单 Server 连接 Task），`SamplingHandler`（sampling 回调），`_ensure_mcp_loop()`（后台事件循环管理），`_build_safe_env()`（环境变量安全过滤）
- `agent/skill_utils.py` — skill 元数据工具库：`parse_frontmatter()`、`skill_matches_platform()`、`extract_skill_conditions()`、`get_all_skills_dirs()`、`iter_skill_index_files()`
- `agent/prompt_builder.py` — 系统提示组装：`build_skills_system_prompt()`（两层缓存 + conditional filtering），`SKILLS_GUIDANCE` / `MEMORY_GUIDANCE` 常量
- `tools/skills_tool.py` — 对话工具层：`skills_list()`、`skill_view()`、`_find_all_skills()`，前向代理 `agent.skill_utils` 的实现
- `tools/skills_hub.py` — Hub 管理：`GitHubSource`（GitHub API 适配）、`WellKnownSkillSource`（标准端点）、`HubLockFile`、`TapsManager`、`quarantine_bundle()`、`install_from_quarantine()`、`append_audit_log()`
- `tools/skills_guard.py` — 安全扫描：`scan_skill()`、`should_allow_install()`、`INSTALL_POLICY`、`TRUSTED_REPOS`

---

## 设计亮点

**EventBridge 的 mtime 优化：** 200ms 轮询不做任何 DB 读写——只比对两个文件的修改时间（约 2 次 `stat()` 调用，耗时 ~2μs）。只有文件确实变化时才进入完整的 DB 查询路径。这使得 MCP Server 在空闲时的 CPU 开销可以忽略不计，是轮询架构下的最优实现。

**Skills 注入的双层缓存 + 差异感知：** `build_skills_system_prompt()` 的磁盘快照通过 mtime_ns + size 的 manifest 精确感知任何 SKILL.md 变化，变化后重新扫描并更新快照。In-process LRU 则覆盖短期内的多次调用（尤其是 gateway 模式下同时服务多个 platform 会话）。两层结合保证了既不过期、又不重复扫描文件系统。

**条件式 Skill 注入（`_skill_should_show`）：** `fallback_for_*` 和 `requires_*` 两套条件规则允许 skill 作者声明"当更强的工具存在时我不显示"（fallback 语义）和"当依赖工具不存在时我不显示"（前提检查）。系统提示中永远不会出现当前上下文不可用的 skill，避免模型幻觉调用。

**Sampling Handler 的工具循环防御：** MCP Server 的 `sampling/createMessage` 可能携带工具调用响应，引发递归工具循环。`SamplingHandler` 用 `max_tool_rounds`（默认 5）在实例级别计数并强制中断，防止无限递归消耗 token。

**隔离-扫描-安装三段式管道：** 外部 skill 先写入隔离目录（`quarantine_bundle()`），扫描通过后才移入实际 skills 目录（`install_from_quarantine()`），两步之间有完整的路径遍历校验（`_normalize_bundle_path()`）。这保证了即使恶意 skill bundle 在扫描阶段幸存，也无法通过路径穿越污染 skills 目录之外的文件。

**上下文文件的提示词注入防护（`prompt_builder._scan_context_content`）：** `AGENTS.md`、`.cursorrules`、`HERMES.md` 等被注入系统提示的上下文文件在注入前经过 14 种威胁模式（`_CONTEXT_THREAT_PATTERNS`）和不可见 Unicode 字符检测，被识别为危险的文件替换为 `[BLOCKED: ...]` 占位符而不是直接丢弃，保留了可审计的拒绝记录。

---

## 局限性

**MCP Server 的审批响应是"尽力而为"（best-effort）：** `EventBridge.respond_to_approval()` 将审批决定写入内存队列，但并没有实际的 IPC 通道将决定传回 Hermes 的权限管理系统。文档注释明确标注 `best-effort without gateway IPC`，意味着这个功能在当前版本是架构占位，而非完整实现。

**TRUSTED_REPOS 硬编码且范围极窄：** `tools/skills_guard.py` 中 `TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}` 仅有 2 个信任仓库，且路径写死在源码中。社区知名仓库（如 VoltAgent/awesome-agent-skills 已在 `DEFAULT_TAPS` 中但不在信任列表）仍然是 community 级别，`caution` 以上 verdict 一律阻止安装。

**动态工具发现依赖 SDK 版本特性：** `_MCP_MESSAGE_HANDLER_SUPPORTED` 通过检查 `ClientSession` 的构造函数签名确定 `message_handler` 是否可用，SDK 版本不支持时动态 `tools/list_changed` 通知完全失效，退化为静态工具列表（连接时获取一次，之后不刷新）。

**Skills 系统没有 FTS5 索引：** skill 的发现完全依赖系统提示中注入的全量索引（`build_skills_system_prompt` 注入所有 skill 的 name+description），依靠 LLM 语义匹配触发 `skill_view()` 加载全文。对于 skill 数量大幅增长（如 100+）的场景，注入的 token 开销会显著增大。相比之下，FTS5 被用在了 session 历史搜索（`tools/session_search_tool.py`）和记忆检索（`plugins/memory/holographic/retrieval.py`），但 skill 发现没有享受到同等的索引支持。

**Sampling 中的 LLM 调用路径是同步阻塞：** `SamplingHandler.__call__()` 用 `asyncio.to_thread(_sync_call)` 将同步 LLM 调用卸载到线程池，实际的 LLM 请求来自 `agent.auxiliary_client.call_llm()`，这条路径在并发多个 MCP Server 同时触发 sampling 时可能在线程池层面形成排队。

---

## 来源

- 源码版本：Hermes Agent 0.16.0（`pyproject.toml`）
- 分析深度：源码级
- 关键文件：`mcp_serve.py`、`tools/mcp_tool.py`、`tools/skills_hub.py`、`tools/skills_tool.py`、`tools/skills_guard.py`、`agent/skill_utils.py`、`agent/prompt_builder.py`
