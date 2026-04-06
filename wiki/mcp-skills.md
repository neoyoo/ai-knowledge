---
title: MCP & Skills
aliases: [MCP, skills, 扩展协议, extension protocol]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[tool-system]]"
    type: extends
  - target: "[[hooks]]"
    type: alternative
sources: [claude-code, openharness]
---

## 一句话定义

外部扩展协议（MCP）+ 可复用的能力包（Skills），让 agent 能力可插拔。

## 核心问题

- MCP 和直接注册工具有什么区别？
- Skill 的粒度怎么定（一个 skill 包含多少能力）？
- 怎么发现和安装第三方扩展？
- 安全边界在哪？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 按抽象层次拆成三种形态：MCP（协议接入层）、Skills（工作流模板层）、Plugins（生态封装层），三者分工明确互不混淆，MCP 解决"接什么工具"，Skills 解决"怎么做任务"，Plugins 解决"如何打包分发" | MCP 通过 Python `mcp` SDK 连接 stdio 服务器，每个工具包装为 `McpToolAdapter`（命名规则 `mcp__servername__toolname`）；Skills 是带可选 YAML frontmatter 的 `.md` 文件；两者通过统一 `plugin.json` 集成，兼容 Claude Code 插件生态 |
| 关键特点 | MCP 工具包装为标准 Tool 接口，进入同一套权限/hook/transcript 流程；Skills 支持参数化（`arguments` + `substituteArguments`）和模型覆盖；子 agent 可声明专属 MCP server 集合实现工具级隔离 | `plugin.json` 将 skills、commands、hooks、MCP 配置聚合为单一 manifest，分发安装体验极简；`${CLAUDE_PLUGIN_ROOT}` 变量替换使插件路径可移植；Skills 纯 markdown 格式无代码门槛；复用 `.claude-plugin/plugin.json` 规范，Claude Code 插件可直接在 OpenHarness 中使用 |
| 局限 | MCP server 崩溃后无自动重连；Skills 和 Commands 同名按优先级静默覆盖，缺乏冲突检测；Plugin marketplace 是中心化信任模型 | 仅支持 stdio MCP 传输，不支持 HTTP/SSE/WebSocket 远程传输；Skills 无动态参数支持，每个 skill 只能执行固定逻辑；修改 `.md` 文件后需重启进程才能生效 |

## 设计权衡

- **三层分离 vs 单一插件机制**：Claude Code 选择了三层分离（MCP/Skills/Plugins 各有单一职责），避免了"万物皆插件"的混乱，但代价是理解成本高，开发者需要判断自己的扩展属于哪一层。
- **子 Agent 专属 MCP vs 全局共享工具池**：允许子 agent 声明独立 MCP server 集合，实现了工具级隔离——不同 agent 可以连接不同工具集，但增加了连接管理的复杂度。

## L2 详情

- [[mcp-skills--claude-code]]
- [[mcp-skills--openharness]]
