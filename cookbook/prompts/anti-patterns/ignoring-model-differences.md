---
anti-pattern: ignoring-model-differences
severity: warning
tags: [model-specific, portability, testing, cross-model]
related_patterns: [system-prompt-design, structured-output, zero-shot]
---

# Ignoring Model Differences

> 在一个模型上调好的 prompt 直接用于另一个模型，忽视不同模型的训练偏好、指令遵循风格和能力边界，导致效果悄然下降却难以排查。

## 本质

不同模型的行为差异，不是参数不同那么简单，而是**不同的训练目标、不同的 RLHF/RLAIF 偏好数据、不同的系统设计决策**共同形塑的不同"人格"。

一个在 GPT-4 上完美运行的 prompt，在 Claude 上可能因为指令风格差异表现平平；在开源模型（LLaMA、Qwen）上可能因为 instruction-following 能力不同而完全失效。更糟糕的是：失效往往不是"报错崩溃"，而是"输出看起来差不多但系统性偏差"——很难在没有对比的情况下发现。

两类最常见的迁移失效：
- **格式偏好差异**：Claude 更喜欢 XML 标签，GPT-4 对 JSON 结构更原生，开源模型对特殊格式的解读各不相同
- **指令遵循度差异**：闭源大模型对"不要做 X"的遵循度远高于小型开源模型，某些限制性指令在开源模型上形同虚设

## 症状

**跨模型迁移后效果不可解释地下降** — 换了模型，其他条件不变，但效果指标下降 15-30%，找不到明显原因。排查了代码、系统配置，最后发现是 prompt 格式不适配。

**结构化输出变得不稳定** — 原来在模型 A 上稳定输出 JSON 的 prompt，在模型 B 上开始输出带 markdown 围栏的 JSON（```json ... ```），或字段顺序变化，或偶尔忽略某些字段。

**同一 prompt 在开源模型上指令被部分忽略** — System prompt 里的限制性指令（"只回答 X 主题"、"不要输出 Y"）在闭源模型上生效，在 7B-13B 量级的开源模型上被忽略或遵循度很低。

**Token 计算和成本预估失准** — 同一段文本在不同模型的 tokenizer 下长度差异可达 15-25%，导致 context 管理、批处理大小、成本计算全都偏差。

## 错误示例

### 1. 直接迁移 GPT-style prompt 到 Claude

```python
# 在 GPT-4 上调好的 prompt，直接用于 Claude
gpt_prompt = """You are a helpful assistant. Follow these rules:
1. Always respond in JSON format
2. Never include markdown
3. Keep responses concise

User query: {query}

Respond with: {"answer": "...", "confidence": 0.0-1.0}"""

# 同样的 prompt 发给 Claude
# 问题：
# - Claude 对编号列表的理解和 GPT 稍有不同
# - Claude 看到 JSON 示例时倾向于用 XML 标签说明而非直接输出
# - Claude 有时会在 JSON 前加解释性文字，而 GPT 通常不会
# - Claude 对"Never include markdown"的解读可能包含它自己的结构化表达习惯

response = claude.complete(gpt_prompt.format(query="what is 2+2"))
# 实际输出可能：
# Here's my response:
# {"answer": "4", "confidence": 1.0}
# 注意：多了"Here's my response:"这行，破坏了 JSON 解析
```

---

### 2. 在开源模型上使用依赖强指令遵循的 prompt

```python
# 依赖严格指令遵循的 prompt（在 GPT-4/Claude 上有效）
STRICT_BOUNDARY_PROMPT = """You are a customer service agent for AcmeCorp.

STRICT RULES:
- ONLY answer questions about AcmeCorp products
- NEVER discuss competitors
- NEVER answer questions outside your domain
- If asked anything else, say: "I can only help with AcmeCorp questions"

User: {query}"""

# 在 GPT-4 上：边界遵循度 ~95%
response_gpt4 = gpt4.complete(STRICT_BOUNDARY_PROMPT.format(
    query="What's the weather like today?"
))
# 输出："I can only help with AcmeCorp questions"  ✓

# 在 LLaMA-3 8B 上：边界遵循度 ~40-60%
response_llama = llama.complete(STRICT_BOUNDARY_PROMPT.format(
    query="What's the weather like today?"
))
# 可能输出："The weather today depends on your location. Generally..."  ✗
# 模型"想要"回答，指令约束力不足以阻止

# 开发者没有测试开源模型，切换时才发现这个系统性问题
```

---

### 3. 忽略 tokenizer 差异导致 context 溢出

```python
# 针对 GPT-4 (cl100k_base tokenizer) 设计的 context 管理
def build_prompt_gpt4(documents: List[str], query: str) -> str:
    max_tokens = 6000  # 为 system prompt 和输出预留 2000
    
    included_docs = []
    total_tokens = 0
    
    for doc in documents:
        # 用 GPT-4 的 tokenizer 计算
        doc_tokens = len(gpt4_tokenizer.encode(doc))
        if total_tokens + doc_tokens < max_tokens:
            included_docs.append(doc)
            total_tokens += doc_tokens
    
    return f"Based on these documents: {included_docs}\nAnswer: {query}"

# 同样的函数用于 Claude (不同 tokenizer)
# 问题：相同文本在 Claude tokenizer 下可能多 10-20%
# 结果：实际 token 数超出预期，context 被截断，模型看不到完整信息
# 而且不会报错，只是悄悄截断

# 更严重：Mistral、Qwen、LLaMA 各有不同 tokenizer
# 多模型路由场景下，用 GPT4 tokenizer 估算所有模型的 token 数，
# 系统性低估导致频繁的隐式截断
```

---

### 4. XML 标签 vs 普通格式的偏好差异

```python
# 为 Claude 优化的 prompt（使用 XML 标签）
claude_optimized = """<instructions>
You are a data analyst. Analyze the data and provide insights.
</instructions>

<data>
{data}
</data>

<output_format>
Provide a JSON object with keys: summary, trends, anomalies
</output_format>"""

# 直接用于 GPT-4
# GPT-4 能理解 XML，但不像 Claude 那样被 RLHF 专门训练来优先处理 XML 结构
# 实际差异：
# - GPT-4 可能把 <output_format> 的内容当作示例而非指令
# - 对于开源模型（Mistral、LLaMA），XML 标签可能被当作普通文字处理
# - 输出的 JSON 可能包含 XML 标签本身的文字

# 反过来，GPT-4 的 function calling / JSON mode 迁移到 Claude 也有问题
# GPT-4 有原生 JSON mode，Claude 没有同等机制（要用 tool use 或 prompt 控制）
```

## 为什么有害

**失效是静默的** — 代码不报错，输出"看起来差不多"，但准确率、遵循度、格式稳定性都在悄悄下降。没有对照实验，几乎不可能发现问题来自 prompt 不适配。

**多模型路由系统风险放大** — 现代生产系统越来越多地用多模型路由（如用便宜小模型处理简单请求，用强模型处理复杂请求），单一 prompt 面对不同模型，等于默默接受各模型的系统性偏差叠加。

**开源模型的指令能力边界被忽视** — 工程师常把开源模型当作"便宜版 GPT-4"，沿用同一套 prompt 期待类似效果。但 7B-13B 开源模型的 instruction-following 能力在复杂约束场景下有本质性差距，不是换个 prompt 能弥补的——需要重新评估任务分配。

**迁移成本被低估** — "换个模型"被当作一行配置的改动，实际上需要完整的 prompt 适配和效果评测。低估迁移成本导致团队在没有足够验证的情况下上线，线上质量下降才发现代价。

## 正确做法

### 原则 1：了解各模型的核心差异

```python
# 不同模型的关键行为差异速查

MODEL_CHARACTERISTICS = {
    "claude-3-opus / claude-4": {
        "preferred_structure": "XML 标签（<instructions>, <context>, <format>）",
        "instruction_following": "极强，对复杂约束和边界条件遵循度高",
        "json_output": "需要 prompt 明确要求，无原生 JSON mode；tool use 方式更稳定",
        "system_prompt": "强烈遵循，可作为硬约束",
        "verbosity": "倾向于给出解释，需要明确要求'只输出结果'",
        "context_window": "200K tokens，tokenizer 效率与 GPT-4 相近",
        "notes": "对'请直接输出，不要解释'响应好；不要在 system prompt 和 user prompt 中给出矛盾指令"
    },
    "gpt-4 / gpt-4o": {
        "preferred_structure": "Markdown、编号列表、普通段落均可",
        "instruction_following": "极强，特别对系统级约束",
        "json_output": "支持原生 JSON mode（response_format: json_object），格式稳定",
        "system_prompt": "强力遵循",
        "verbosity": "相对平衡，不过度解释",
        "context_window": "128K tokens (gpt-4-turbo)，cl100k_base tokenizer",
        "notes": "function calling 在结构化输出上比 few-shot 更稳定；o1/o3 不支持 system prompt 同等权重"
    },
    "llama-3 70B / qwen-2.5 72B": {
        "preferred_structure": "ChatML 格式（<|im_start|>system...），或模型特定模板",
        "instruction_following": "强，接近闭源模型，但复杂约束有时漂移",
        "json_output": "需要明确 prompt 控制，部分支持 grammar-based 解码（llama.cpp）",
        "system_prompt": "遵循，但不如闭源模型稳定",
        "verbosity": "变化较大，模型不同差异明显",
        "context_window": "8K-128K 不等，tokenizer 与 GPT-4 有差异（需各自计算）",
        "notes": "使用模型官方推荐的 chat template；不要直接用 GPT-4 的 prompt 格式"
    },
    "llama-3 7B / mistral 7B": {
        "preferred_structure": "简单直接的指令，避免复杂结构",
        "instruction_following": "中等，简单约束（1-2条）遵循好；超过 3 条限制开始漂移",
        "json_output": "不稳定，grammar-based 解码（llama.cpp/vllm）是唯一可靠方案",
        "system_prompt": "部分遵循，不可作为硬约束",
        "verbosity": "变化大，需要测试",
        "context_window": "4K-32K，tokenizer 效率较低（同文本 token 数更多）",
        "notes": "复杂任务换用 30B+ 模型；依赖 grammar-based 解码确保格式"
    }
}
```

---

### 原则 2：抽象 Prompt 适配层

```python
from abc import ABC, abstractmethod
from typing import Dict, Any

class PromptAdapter(ABC):
    """为不同模型适配同一任务的 prompt"""
    
    @abstractmethod
    def format_system_prompt(self, instructions: str) -> str:
        pass
    
    @abstractmethod
    def format_user_input(self, content: str, label: str = "user_input") -> str:
        pass
    
    @abstractmethod
    def format_output_spec(self, spec: str) -> str:
        pass

class ClaudeAdapter(PromptAdapter):
    def format_system_prompt(self, instructions: str) -> str:
        return f"<instructions>\n{instructions}\n</instructions>"
    
    def format_user_input(self, content: str, label: str = "user_input") -> str:
        return f"<{label}>\n{content}\n</{label}>"
    
    def format_output_spec(self, spec: str) -> str:
        return f"<output_format>\n{spec}\n</output_format>"

class GPT4Adapter(PromptAdapter):
    def format_system_prompt(self, instructions: str) -> str:
        return instructions  # GPT-4 system prompt 不需要特殊包装
    
    def format_user_input(self, content: str, label: str = "user_input") -> str:
        return f"---\n{content}\n---"
    
    def format_output_spec(self, spec: str) -> str:
        return f"Output format: {spec}"

class OpenSourceAdapter(PromptAdapter):
    """适配开源模型（7B-13B），使用简化指令"""
    
    def format_system_prompt(self, instructions: str) -> str:
        # 开源小模型对长 system prompt 的遵循度差，提炼核心约束
        lines = instructions.strip().split('\n')
        # 最多保留 3 条最重要的约束
        key_lines = [l for l in lines if l.strip() and not l.startswith('#')][:3]
        return '\n'.join(key_lines)
    
    def format_user_input(self, content: str, label: str = "user_input") -> str:
        return f"Input:\n{content}"
    
    def format_output_spec(self, spec: str) -> str:
        return f"Respond with: {spec}"


def get_adapter(model_name: str) -> PromptAdapter:
    if "claude" in model_name.lower():
        return ClaudeAdapter()
    elif "gpt" in model_name.lower():
        return GPT4Adapter()
    else:
        return OpenSourceAdapter()

# 使用示例
def analyze_sentiment(text: str, model: str) -> str:
    adapter = get_adapter(model)
    
    instructions = "You are a sentiment analyzer. Classify the sentiment as Positive, Negative, or Neutral."
    user_input = adapter.format_user_input(text)
    output_spec = adapter.format_output_spec('{"sentiment": "Positive|Negative|Neutral", "confidence": 0.0-1.0}')
    
    prompt = f"{adapter.format_system_prompt(instructions)}\n\n{user_input}\n\n{output_spec}"
    return llm.complete(prompt, model=model)
```

---

### 原则 3：多模型评测流水线

```python
# 核心原则：新 prompt 上线前，必须在所有目标模型上测试

def evaluate_prompt_cross_model(
    prompt_template: str,
    test_cases: List[Dict],
    target_models: List[str],
    metrics: List[str] = ["format_compliance", "accuracy", "instruction_following"]
) -> Dict:
    """
    在多个模型上评测同一 prompt 模板的效果。
    任何模型的评分低于阈值，触发该模型的专项适配。
    """
    results = {}
    
    for model in target_models:
        model_results = []
        adapter = get_adapter(model)
        adapted_prompt = adapt_prompt(prompt_template, adapter)
        
        for case in test_cases:
            response = llm.complete(adapted_prompt.format(**case["input"]), model=model)
            score = evaluate_response(response, case["expected"], metrics)
            model_results.append(score)
        
        avg_scores = {
            metric: np.mean([r[metric] for r in model_results])
            for metric in metrics
        }
        results[model] = avg_scores
        
        # 发现低分项，触发警告
        for metric, score in avg_scores.items():
            if score < 0.8:
                print(f"警告：模型 {model} 在 {metric} 上得分 {score:.2f}，需要专项适配")
    
    return results

# 推荐：在 CI/CD 中加入跨模型回归测试
# 每次修改 prompt 模板时自动运行，防止"在 Claude 上优化时偷偷降低 GPT-4 效果"
```

---

### 原则 4：Tokenizer 独立的 context 管理

```python
from typing import Optional

def count_tokens_for_model(text: str, model: str) -> int:
    """为指定模型准确计算 token 数"""
    if "gpt" in model or "o1" in model or "o3" in model:
        import tiktoken
        enc = tiktoken.encoding_for_model(model)
        return len(enc.encode(text))
    elif "claude" in model:
        # Claude 官方建议：字符数 / 4 作为估算，或用 API 的 token counting endpoint
        # 精确方案：调用 Anthropic token counting API
        return anthropic_client.count_tokens(text, model=model)
    else:
        # 开源模型：加载对应 tokenizer
        from transformers import AutoTokenizer
        tokenizer = AutoTokenizer.from_pretrained(get_hf_model_id(model))
        return len(tokenizer.encode(text))

def build_context_safe(
    documents: List[str],
    query: str,
    model: str,
    max_context_tokens: int,
    reserve_for_output: int = 1000
) -> List[str]:
    """根据目标模型的 tokenizer 安全构建上下文"""
    available = max_context_tokens - reserve_for_output
    query_tokens = count_tokens_for_model(query, model)
    remaining = available - query_tokens
    
    included = []
    for doc in documents:
        doc_tokens = count_tokens_for_model(doc, model)
        if doc_tokens <= remaining:
            included.append(doc)
            remaining -= doc_tokens
        else:
            break
    
    return included
```

## 修复检查清单

- [ ] 了解生产中使用的每个模型的关键差异（指令格式偏好、JSON 处理方式、context 计算）
- [ ] 有 Prompt 适配层，不同模型使用各自优化的格式，而非共享一套 prompt
- [ ] Token 计算使用目标模型的 tokenizer，不用 GPT-4 tokenizer 估算所有模型
- [ ] 新 prompt 上线前在所有目标模型上运行评测，有基线指标对比
- [ ] 多模型路由系统有各模型独立的效果监控，而非混合指标
- [ ] 在开源小模型（<30B）上测试了指令遵循的边界（超过几条约束开始漂移）
- [ ] 结构化输出场景在小模型上使用 grammar-based 解码而非纯 prompt 控制
- [ ] 有跨模型回归测试，防止优化一个模型时悄悄降低另一个模型的效果

## 关联

- [[system-prompt-design]] — 各模型 system prompt 的设计最佳实践差异
- [[structured-output]] — JSON/结构化输出在不同模型上的可靠性差异
- [[zero-shot]] — 当模型差异导致 few-shot 示例不适用时，zero-shot 的替代方案

## 来源

- Anthropic. (2024). *Prompt engineering overview*. https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview
- OpenAI. (2024). *GPT-4 Technical Report*. https://arxiv.org/abs/2303.08774
- Touvron, H. et al. (2023). *Llama 2: Open Foundation and Fine-Tuned Chat Models*. https://arxiv.org/abs/2307.09288
- Wei, J. et al. (2022). *Emergent Abilities of Large Language Models*. TMLR 2022. https://arxiv.org/abs/2206.07682
- Prompt Engineering Guide (DAIR.AI). Model-specific considerations. https://www.promptingguide.ai
