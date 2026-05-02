---
title: "channel-remote——agentscope"
category: L2
parent: "[[channel-remote]]"
source: agentscope
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: channel-remote
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

AgentScope 通过 `realtime/` 和 `tts/` 两个模块实现多模态实时通道。`realtime/` 提供基于 WebSocket 的双向流式对话抽象，支持 OpenAI、DashScope（阿里云）、Gemini 三家 Realtime API，并定义了一套跨提供商统一的事件体系（ModelEvents / ServerEvents / ClientEvents）；`tts/` 提供独立的语音合成通道，支持流式输出与实时流式输入两种模式。整体采用异步 asyncio + Queue 驱动，前后端通过结构化事件对象解耦。

---

## 架构分析

### 三层事件模型

AgentScope 在 Realtime 通道中设计了三层语义明确的事件体系：

| 事件层 | 命名空间 | 方向 | 职责 |
|---|---|---|---|
| `ModelEvents` | `model_*` | API → Agent | 抹平不同 Realtime API 的原生事件差异，统一为 AgentScope 内部格式 |
| `ServerEvents` | `agent_*` / `server_*` | Backend → Frontend | 在 `ModelEvents` 基础上增加 `agent_id` / `agent_name`，向 Web 前端或其他 Agent 广播 |
| `ClientEvents` | `client_*` | Frontend → Backend | 接收用户操作（音频流、文本、图像、工具结果），路由到对应 Agent |

`ServerEvents.from_model_event()` 实现了 ModelEvents → ServerEvents 的机械转换：通过类名替换（`Model` → `Agent`）+ `model_dump()` + `model_validate()` 完成，无需逐类手写。

### RealtimeModelBase 抽象层

`RealtimeModelBase`（`realtime/_base.py`）是所有 Realtime 模型的基类，核心契约：

- `connect(outgoing_queue, instructions, tools)` — 建立 WebSocket 连接，发送 session config，启动内部接收任务
- `send(data: AudioBlock | TextBlock | ImageBlock | ToolResultBlock)` — 向模型发送多模态输入（各子类实现平台格式转换）
- `parse_api_message(message)` — 将平台原生 JSON 解析为统一 `ModelEvents.EventBase`
- `_receive_model_event_loop(outgoing_queue)` — asyncio Task，持续从 WebSocket 读取消息，调用 `parse_api_message`，将结果放入 `outgoing_queue`

三家实现（OpenAI / DashScope / Gemini）各自重写 `send` 和 `parse_api_message`，差异点见下文。

### RealtimeAgent：事件路由中枢

`RealtimeAgent`（`agent/_realtime_agent.py`）连接前后端与模型，管理两个异步循环：

1. `_forward_loop` — 从 `_incoming_queue` 取事件（ClientEvents 或来自其他 Agent 的 ServerEvents），转为多模态 Block 发送给模型
2. `_model_response_loop` — 从 `_model_response_queue` 取 ModelEvent，映射为 ServerEvent 放入 `outgoing_queue`；`ToolUseDone` 事件会额外异步执行工具调用并将结果回馈给模型

### TTS 通道：独立的语音合成层

`TTSModelBase`（`tts/_tts_base.py`）抽象两种模式：

- **非流式** (`supports_streaming_input=False`)：`synthesize(msg)` 一次完成，等待完整音频返回
- **流式输入** (`supports_streaming_input=True`)：通过 async context manager 管理生命周期，`push(msg)` 接受增量文本并非阻塞返回当前已合成的音频，`synthesize()` 等待完成并返回剩余部分

`DashScopeCosyVoiceRealtimeTTSModel` 是唯一实现流式输入的 TTS 模型，通过 DashScope SDK 的 `SpeechSynthesizer.streaming_call()` 驱动，内部使用 `threading.Event` 做线程边界（SDK 回调在独立线程）。

---

## 关键代码路径

### WebSocket 连接建立与事件循环

```
RealtimeModelBase.connect(outgoing_queue, instructions, tools)
  ├── websockets.connect(self.websocket_url, additional_headers)  → self._websocket
  ├── asyncio.create_task(_receive_model_event_loop(outgoing_queue))
  └── self._websocket.send(json.dumps(_build_session_config(instructions, tools)))

_receive_model_event_loop(outgoing_queue):
  async for message in self._websocket:
    events = await self.parse_api_message(message)
    for event in events:
      await outgoing_queue.put(event)
```

文件：`realtime/_base.py:68-157`

### DashScope session config 构建

```python
DashScopeRealtimeModel._build_session_config(instructions, tools) -> dict
  # 固定 turn_detection.type = "server_vad"
  # input_audio_format = "pcm16" / "pcm24" 根据 input_sample_rate
  # output_audio_format 同理
  # enable_input_audio_transcription 时注入 input_audio_transcription.model = "gummy-realtime-v1"
  return {"type": "session.update", "session": session_config}
```

文件：`realtime/_dashscope_realtime_model.py:96-131`

### OpenAI 工具调用参数累积

OpenAI Realtime API 以流式 delta 下发 function call arguments，需在客户端手动累积：

```python
# parse_api_message 内部
case "response.function_call_arguments.delta":
    self._tool_args_accumulator[call_id] += arguments_delta
    # 返回累积值而不是仅返回 delta
    model_event = ModelResponseToolUseDeltaEvent(
        tool_use=ToolUseBlock(..., raw_input=self._tool_args_accumulator[call_id])
    )

case "response.function_call_arguments.done":
    model_event = ModelResponseToolUseDoneEvent(
        tool_use=ToolUseBlock(
            input=_json_loads_with_repair(current_input),
            raw_input=current_input,
        )
    )
    del self._tool_args_accumulator[call_id]
```

文件：`realtime/_openai_realtime_model.py:310-352`

### Gemini 事件解析（无原生 response.created）

Gemini Live API 不发送 `response.created` 事件，AgentScope 在收到第一个 audio/text chunk 时自行生成 response_id：

```python
def _ensure_response_id(self) -> str:
    if not self._response_id:
        self._response_id = f"resp_{shortuuid.uuid()}"
    return self._response_id

# parse_api_message 路由
if "setupComplete" in data:
    → ModelSessionCreatedEvent(session_id="gemini_session")
elif "serverContent" in data:
    → _parse_server_content(data["serverContent"])
        ├── "modelTurn"          → _parse_model_turn → audio/transcript delta
        ├── "generationComplete" → ModelResponseDoneEvent (清空 _response_id)
        ├── "turnComplete"       → 若有 _response_id 则发 ModelResponseDoneEvent
        └── "interrupted"        → 忽略（log debug）
elif "toolCall" in data:
    → 批量 ModelResponseToolUseDoneEvent（Gemini 不分 delta/done，直接全量）
```

文件：`realtime/_gemini_realtime_model.py:251-516`

### ModelEvents → ServerEvents 自动映射

```python
ServerEvents.from_model_event(model_event, agent_id, agent_name):
    cls_name = model_event.__class__.__name__.replace("Model", "Agent")
    # e.g. "ModelResponseAudioDeltaEvent" → "AgentResponseAudioDeltaEvent"
    agent_event_cls = getattr(ServerEvents, cls_name)
    model_event_dict = model_event.model_dump()
    model_event_dict["type"] = model_event_dict["type"].replace("model_", "agent_")
    model_event_dict["agent_id"] = agent_id
    model_event_dict["agent_name"] = agent_name
    return agent_event_cls.model_validate(model_event_dict)
```

文件：`realtime/_events/_server_event.py:463-524`

### RealtimeAgent 工具调用链路

```
_model_response_loop 接收 ModelResponseToolUseDoneEvent
  ├── 立即 put AgentResponseToolUseDoneEvent 到 outgoing_queue（通知前端）
  └── asyncio.create_task(_acting(tool_use, outgoing_queue))
        ├── toolkit.call_tool_function(tool_use) → 异步执行
        ├── model.send(ToolResultBlock)           → 发回给 Realtime API
        └── put AgentResponseToolResultEvent 到 outgoing_queue（通知前端结果）
```

文件：`agent/_realtime_agent.py:275-360`

### DashScope CosyVoice 实时 TTS 流式输入路径

```
DashScopeCosyVoiceRealtimeTTSModel.push(msg) → TTSResponse(非阻塞):
  synthesizer.streaming_call(delta_to_send)
  res = await _dashscope_callback.get_audio_data(block=False)
  return res  # 可能为空，音频还没合成完

DashScopeCosyVoiceRealtimeTTSModel.synthesize(msg=None) → TTSResponse(阻塞):
  synthesizer.streaming_complete()  # 通知 SDK 输入结束
  if self.stream:
    return _dashscope_callback.get_audio_chunk()  # AsyncGenerator
  else:
    return await _dashscope_callback.get_audio_data(block=True)
```

文件：`tts/_dashscope_cosyvoice_realtime_tts_model.py:144-279`

### PCM/Base64 对齐算法（CosyVoice 回调）

```python
# on_data(data: bytes) 内：
# 以 6 字节为单位编码，保证 PCM（2字节/样本）和 base64（3字节/组）双对齐
aligned_len = (len(self._audio_bytes) // 6) * 6
if aligned_len > self._last_encoded_pos:
    new_chunk = self._audio_bytes[self._last_encoded_pos : aligned_len]
    self._audio_base64 += base64.b64encode(new_chunk).decode()
    self._last_encoded_pos = aligned_len
```

文件：`tts/_utils.py:63-77`

---

## 设计亮点

**统一事件体系 + 自动映射**：三层事件（ModelEvents / ServerEvents / ClientEvents）通过命名约定（`model_` ↔ `agent_`）实现机械转换，新增事件类型只需在模型层定义，无需手动维护映射表。Pydantic v2 的 `model_dump()` + `model_validate()` 做序列化中转，类型安全且零手写胶水代码。

**多提供商统一 API 但差异隔离**：三家 Realtime API 的协议差异（Gemini 无 `response.created`、OpenAI 需累积 tool args delta、DashScope 不支持 tools）全部封装在各自的 `parse_api_message` 和 `_build_session_config` 内，`RealtimeAgent` 完全不感知提供商差异。

**工具调用的异步并行执行**：`_acting` 以 `asyncio.create_task` 异步派发，不阻塞模型响应处理循环；工具结果既发回给 Realtime API（继续对话），也广播给 outgoing_queue（让前端显示结果），双路通知。

**PCM + base64 双对齐**：CosyVoice 回调中以 LCM(2,3)=6 字节为边界编码，保证实时流式切片时既不破坏 PCM 样本边界，也不产生 base64 padding 碎片，可直接用于流式播放。

**VAD 服务端检测**：DashScope 和 OpenAI 均默认启用 `server_vad`（语音活动检测在 API 服务端），AgentScope 层不做本地 VAD，降低客户端复杂度，由云端决定说话结束时机。

**Cold Start 门槛控制**：`DashScopeCosyVoiceRealtimeTTSModel` 的 `cold_start_length` 和 `cold_start_words` 参数允许设置首批 TTS 请求的最低文本量，避免因输入太短造成合成停顿，对流式 LLM 输出接驳 TTS 场景实用。

---

## 局限性

**DashScope Realtime 不支持 Tools**：`DashScopeRealtimeModel.support_tools = False`，工具调用功能仅 OpenAI 和 Gemini 可用。DashScope 的 `text` 输入模式在源码中也有 `TODO: 尚未可用` 注释，功能不完整。

**CosyVoice 单请求限制**：`DashScopeCosyVoiceRealtimeTTSModel` 每次只能处理一个流式输入序列，不支持并发多路 TTS 合成。注释明确："不能处理 `[msg_1_chunk0, msg_2_chunk0]` 这类交叉消息"。

**OpenAI 并行工具调用不完整**：`parse_api_message` 中 `TODO` 注释指出，工具调用累积器 `_tool_args_accumulator` 设计对并行工具调用（parallel function calls）处理有缺陷，当前一次只能可靠处理一个工具调用。

**Session 生命周期管理薄弱**：`disconnect()` 中有 `TODO: session ended` 注释，WebSocket 断开后没有会话状态持久化或恢复机制，重连需要从头建立 session。

**PCM 重采样依赖第三方**：`_forward_loop` 中 Agent 间音频路由需要 PCM 重采样（`_resample_pcm_delta`），实现在 `_utils/_common.py`，但没有声明依赖的重采样库，生产部署可能存在隐性依赖。

**Gemini 无 token 计数**：Gemini 的 `ModelResponseDoneEvent` 中 `input_tokens=0, output_tokens=0` 为硬编码，Gemini Live API 当前版本不返回 token 用量，成本追踪不可用。

**TTS 模块与 Realtime Agent 未集成**：`tts/` 和 `realtime/` 是独立模块，`RealtimeAgent` 没有内置 TTS 支持。若需要 LLM 文字回复 → TTS 语音输出，需要用户自己编写桥接逻辑。

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 覆盖文件：
  - `realtime/_base.py`
  - `realtime/_dashscope_realtime_model.py`
  - `realtime/_openai_realtime_model.py`
  - `realtime/_gemini_realtime_model.py`
  - `realtime/_events/_model_event.py`
  - `realtime/_events/_server_event.py`
  - `realtime/_events/_client_event.py`
  - `realtime/_events/_utils.py`
  - `agent/_realtime_agent.py`
  - `tts/_tts_base.py`
  - `tts/_tts_response.py`
  - `tts/_dashscope_cosyvoice_realtime_tts_model.py`
  - `tts/_openai_tts_model.py`
  - `tts/_utils.py`
