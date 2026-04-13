---
template: conversation-agent
scenario: 多轮对话 Agent
tags: [conversation, chat, multi-turn, memory, persona]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_patterns: [system-prompt-design, chain-of-thought, few-shot]
---

# Conversation Agent 模板

> 构建有角色、有记忆、能保持多轮上下文的对话 Agent，适配从客服机器人到个人助手的各类场景。

## 场景描述

多轮对话不等于把历史消息全量堆进 context。真正能用的对话 Agent 需要解决三个问题：

1. **角色一致性**：每轮回复都要符合预设角色、风格、边界
2. **上下文保持**：记住对话中提到的关键信息，不丢失重要细节
3. **长对话压缩**：历史消息超长时，不能无限扩张 context，需要摘要压缩

这个模板覆盖从基础对话到长对话记忆管理的完整实现路径。

典型场景：

- 客服 Agent（固定角色 + 话题边界限制 + 工单历史注入）
- 个人助手（自定义人格 + 用户偏好记忆 + 跨会话记忆）
- 教学辅导（学科专家角色 + 学生进度追踪 + 话题引导）

## 模板

### System Prompt（角色 + 风格 + 记忆管理指令）

```text
你是{{agent_name}}，{{agent_description}}。

**你的对话风格：**
{{conversation_style}}

**你的能力边界：**
你只负责处理以下范围内的问题：
{{scope_description}}

如果用户的问题超出你的能力范围，请友好地说明你无法帮助，并建议他们可以去哪里寻求帮助：{{fallback_suggestion}}。
不要尝试回答超出范围的问题，即使你知道答案。

**记忆管理：**
对话过程中，请注意记住以下类型的关键信息：
- 用户明确告诉你的个人信息（姓名、偏好、背景）
- 本次对话中已达成的结论或决定
- 用户反复提及的需求或痛点

{{#if memory_summary}}
**本次对话的已有记忆摘要（来自历史轮次的压缩）：**
{{memory_summary}}
请在回复中保持与这些已知信息的一致性。
{{/if}}

**回复原则：**
- 保持角色一致，不要在角色之外发表评论
- 如果不确定，优先提问澄清，而不是猜测
- 回复长度适中：简单问题简短回答，复杂问题详细说明
```

---

### 对话历史注入模板

```text
{{#if conversation_history}}
以下是本次对话的历史记录：

{{conversation_history}}

---
{{/if}}
用户：{{current_user_message}}
```

**conversation_history 的格式：**

```text
用户：我想了解 Python 异步编程
助手：Python 的异步编程主要通过 asyncio 模块实现……

用户：能举个实际例子吗？
助手：当然，以下是一个并发发送 HTTP 请求的示例……

用户：那 await 和 async def 有什么区别？
助手：这是一个很好的问题……
```

---

### 记忆摘要注入点（长对话压缩）

当对话历史超过阈值（建议 10 轮或 4000 tokens），触发摘要压缩。将历史摘要注入 system prompt 的 `{{memory_summary}}` 位置，只保留最近 N 轮原始对话。

**摘要压缩 Prompt：**

```text
以下是一段对话记录，请提取其中的关键信息，生成简洁的结构化摘要。

对话记录：
{{conversation_to_summarize}}

请按以下格式输出摘要（JSON）：
{
  "user_profile": {
    "name": "（如果提到）",
    "background": "（用户的相关背景）",
    "preferences": ["偏好1", "偏好2"]
  },
  "key_decisions": ["已确认的结论1", "已确认的结论2"],
  "open_questions": ["尚未解决的问题1"],
  "conversation_context": "一句话描述本次对话的主题和进展"
}

只输出 JSON，不要附加解释。
```

---

### Python 实现骨架

```python
import json
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT_TMPL = "..."        # 上方 System Prompt 模板
SUMMARY_PROMPT_TMPL = "..."       # 上方记忆摘要压缩 Prompt

COMPRESS_THRESHOLD_TURNS = 10     # 超过多少轮触发压缩
KEEP_RECENT_TURNS = 4             # 压缩后保留最近几轮原始对话


class ConversationAgent:
    def __init__(
        self,
        agent_name: str,
        agent_description: str,
        conversation_style: str,
        scope_description: str,
        fallback_suggestion: str,
    ):
        self.system_config = {
            "agent_name": agent_name,
            "agent_description": agent_description,
            "conversation_style": conversation_style,
            "scope_description": scope_description,
            "fallback_suggestion": fallback_suggestion,
        }
        self.history: list[dict] = []   # {"role": "user"/"assistant", "content": str}
        self.memory_summary: str = ""   # 历史摘要

    def _build_system_prompt(self) -> str:
        return SYSTEM_PROMPT_TMPL.format(
            **self.system_config,
            memory_summary=self.memory_summary,
        )

    def _maybe_compress(self):
        """超过阈值时，压缩旧历史为摘要"""
        if len(self.history) // 2 < COMPRESS_THRESHOLD_TURNS:
            return

        # 保留最近 N 轮，压缩其余
        keep_idx = len(self.history) - KEEP_RECENT_TURNS * 2
        to_compress = self.history[:keep_idx]
        self.history = self.history[keep_idx:]

        # 构造摘要输入
        history_text = "\n".join(
            f"{'用户' if m['role'] == 'user' else '助手'}：{m['content']}"
            for m in to_compress
        )
        summary_resp = client.messages.create(
            model="claude-haiku-4-5",   # 摘要任务用 Haiku 降低成本
            max_tokens=512,
            messages=[{
                "role": "user",
                "content": SUMMARY_PROMPT_TMPL.format(
                    conversation_to_summarize=history_text
                )
            }]
        )
        new_summary = json.loads(summary_resp.content[0].text)

        # 合并到已有摘要
        self.memory_summary = json.dumps(new_summary, ensure_ascii=False, indent=2)

    def chat(self, user_message: str) -> str:
        # 添加用户消息到历史
        self.history.append({"role": "user", "content": user_message})

        # 触发压缩（如有必要）
        self._maybe_compress()

        # 调用 API
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            system=self._build_system_prompt(),
            messages=self.history,
        )
        assistant_message = response.content[0].text

        # 保存助手回复到历史
        self.history.append({"role": "assistant", "content": assistant_message})

        return assistant_message
```

---

### 话题边界处理

在 system prompt 里设置明确的 scope，当用户越界时，Agent 应友好引导而不是拒绝：

```text
如果用户问题超出你的范围，回复模板：

"这个问题超出了我目前能帮你解决的范围。
[如果能判断相关] 对于这类问题，你可以试试 {{relevant_resource}}。
我可以继续帮你处理 {{agent_scope}} 相关的事情，有什么我能帮到你的吗？"
```

**检测越界的辅助 Prompt（可选，在路由层使用）：**

```text
判断以下用户消息是否属于「{{scope_description}}」范围内的问题。
只回答 "in_scope" 或 "out_of_scope"，不要解释。

用户消息：{{user_message}}
```

## 自定义指南

**替换要点：**

| 占位符 | 说明 | 示例 |
|--------|------|------|
| `{{agent_name}}` | Agent 的名称 / 称呼 | `小智`、`TechBot`、`CodeReview 助手` |
| `{{agent_description}}` | 一句话描述 Agent 的定位 | `专注 Python 学习辅导的编程助手` |
| `{{conversation_style}}` | 对话风格描述 | `简洁直接，多用代码示例，避免过于学术化` |
| `{{scope_description}}` | 能力边界的明确说明 | `Python 编程问题、算法与数据结构、代码调试` |
| `{{fallback_suggestion}}` | 越界时的引导建议 | `官方文档 docs.python.org 或 Stack Overflow` |
| `{{memory_summary}}` | 历史对话摘要（动态注入） | 由摘要压缩 Prompt 生成的 JSON 结构 |
| `{{conversation_history}}` | 近期对话历史（动态注入） | 最近 N 轮对话的文本 |

**压缩阈值调整建议：**

| 场景 | 推荐阈值 | 保留轮数 |
|------|----------|----------|
| 轻量客服对话 | 8 轮 | 3 轮 |
| 技术辅导 | 12 轮 | 5 轮 |
| 长期个人助手 | 20 轮 | 6 轮 |

## 使用示例

**场景：Python 编程辅导助手**

```python
agent = ConversationAgent(
    agent_name="CodeMentor",
    agent_description="专注 Python 学习辅导的编程助手，帮助初学者和中级开发者解决实际问题",
    conversation_style="耐心、具体，多用代码示例，对初学者友好，不使用术语堆砌",
    scope_description="Python 编程、算法与数据结构、常用库（requests/pandas/asyncio 等）、代码调试",
    fallback_suggestion="官方文档 docs.python.org、Stack Overflow、或请咨询专业老师",
)

# 多轮对话
print(agent.chat("我想学 async/await，从哪里入手？"))
print(agent.chat("能解释一下 event loop 是什么吗？"))
print(agent.chat("我写了一段代码，运行报错了：[贴上代码]"))
```

**场景：客服机器人（有工单历史）**

```python
# 在 system prompt 里注入用户工单历史作为 memory_summary
agent = ConversationAgent(
    agent_name="支持助手",
    agent_description="负责处理产品使用问题和订单查询的客服助手",
    conversation_style="礼貌、专业、简洁，首先确认用户问题再给出解决方案",
    scope_description="产品使用问题、订单状态查询、退换货政策说明",
    fallback_suggestion="人工客服（工作时间 9:00-18:00）",
)
# 将用户历史工单摘要注入
agent.memory_summary = load_user_ticket_history(user_id)
```

## 适配建议

**流式输出（改善用户体验）**

对于较长回复，使用 streaming 逐 token 输出，减少等待感：

```python
with client.messages.stream(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=self._build_system_prompt(),
    messages=self.history,
) as stream:
    full_text = ""
    for text in stream.text_stream:
        print(text, end="", flush=True)
        full_text += text
    print()  # 换行
self.history.append({"role": "assistant", "content": full_text})
```

**跨会话记忆持久化**

将 `memory_summary` 和最近 N 轮 `history` 序列化到数据库，下次会话时加载：

```python
import json

def save_session(agent: ConversationAgent, session_id: str):
    state = {
        "memory_summary": agent.memory_summary,
        "recent_history": agent.history[-KEEP_RECENT_TURNS * 2:],
    }
    db.set(f"session:{session_id}", json.dumps(state))

def load_session(agent: ConversationAgent, session_id: str):
    raw = db.get(f"session:{session_id}")
    if raw:
        state = json.loads(raw)
        agent.memory_summary = state["memory_summary"]
        agent.history = state["recent_history"]
```

**多语言支持**

在 system prompt 里加一条：`使用与用户相同的语言回复，如果用户用中文，则用中文；用英文，则用英文。`

## 关联

关联模板：[[multi-agent-orchestrator]]

关联 wiki：[[wiki/context-and-memory]] · [[wiki/prompt-system]] · [[wiki/agent-loop]]

## 来源

- **Anthropic Claude Docs** — Messages API 多轮对话设计
  [https://docs.anthropic.com/claude/docs/messages-overview](https://docs.anthropic.com/claude/docs/messages-overview)

- **mempalace** — raw verbatim 记忆范式、LongMemEval 96.6% 方案分析
  `raw/mempalace`

- **Prompt Engineering Guide** — System Prompt 设计与对话管理
  [https://www.promptingguide.ai/techniques/prompt_chaining](https://www.promptingguide.ai/techniques/prompt_chaining)
