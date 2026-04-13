---
pattern: pal
category: reasoning
tags: [code-generation, program-aided, math, computation, interpreter]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/tool-system, wiki/prompt-system]
related_patterns: [chain-of-thought, react, structured-output]
---

# PAL (Program-Aided Language Models)

> 让模型把推理过程写成代码而非自然语言，然后执行代码得到精确结果，彻底避免计算错误。

## 本质

LLM 做算术会犯错，但写代码很靠谱。这两件事的差距不是偶然的——模型在训练时见过大量正确的代码，却几乎没有"验证中间计算步骤"的训练信号。

PAL 的洞察很直接：**让模型写代码来算，而不是自己算**。模型负责把自然语言问题翻译成程序逻辑（它擅长），Python 解释器负责执行计算（它零错误）。两者各司其职，最终结果可靠。

与 CoT 的根本区别：CoT 用自然语言推理，每一步都可能有数值错误；PAL 用代码推理，执行结果由解释器保证精确。计算越复杂，差距越大。

来自 Gao et al., (2022)，论文 [PAL: Program-aided Language Models](https://arxiv.org/abs/2211.10435)。

## 什么时候用

**数学计算与定量推理** — 多步算术、概率计算、代数。任何"算出一个数"的任务都适合。模型写 `(15 * 0.8) + 7` 比在自然语言里心算要可靠得多。

**日期与时间运算** — 涉及日期加减、工作日计算、时区转换的问题。Python 有 `datetime` 和 `dateutil`，一行代码解决，语言推理容易出错。

**数据处理与统计** — 给定一组数字，求均值/中位数/百分位；列表排序、过滤、聚合。这类任务语言描述步骤冗长且易出错，代码表达简洁且精确。

**逻辑推理可编程化** — 规则推导、组合计数、约束满足。只要推理步骤可以用 `if/for/while` 表达，就值得用 PAL。

**需要精确结果的场景** — 金融计算、科学计量、工程参数。容不得计算误差时，必须用代码执行而非语言估算。

## 什么时候不用

**纯语言理解任务** — 情感分析、摘要、翻译。这类任务的"答案"本身就是自然语言，没什么好编程的。

**没有代码执行环境** — PAL 的前提是有解释器能跑生成的代码。如果只是纯 prompt 场景，没有 `exec` 或 code interpreter，生成的代码就是一段看起来正确的文本，没有实际价值。

**推理无法程序化** — 有些推理天然是语言性的，比如道德判断、主观评估、创意构思。强行用代码表达反而增加复杂度，不如直接用 CoT。

**代码 bug 风险高于计算误差风险** — 如果问题复杂到生成的代码可能有逻辑 bug，而 bug 又难以检测，那代码执行的"精确性"反而是个陷阱——你得到了精确错误的答案。

## 模板

### 基础版

**Zero-shot PAL** — 直接要求模型生成 Python 代码作为解题步骤，最后一行赋值变量作为答案：

```text
用 Python 代码一步步解决以下问题。将最终答案赋值给变量 `answer`，每一步用注释说明含义。

问题：{{problem_statement}}
```

示例输入：
```text
用 Python 代码一步步解决以下问题。将最终答案赋值给变量 `answer`，每一步用注释说明含义。

问题：我去市场买了 10 个苹果，给了邻居 2 个，给了修理工 2 个，又买了 5 个，最后吃了 1 个。我还剩多少个苹果？
```

模型输出（可直接 `exec`）：

```python
# 初始苹果数
apples = 10
# 给邻居
apples -= 2
# 给修理工
apples -= 2
# 又买了
apples += 5
# 吃了
apples -= 1
# 最终剩余
answer = apples
```

执行后 `answer = 10`。

---

**Few-shot PAL** — 提供带代码推理的示例，引导模型使用特定的代码风格和注释格式：

```text
按以下示例格式，用 Python 代码解决问题。每一步用注释说明，最终答案存入 `answer`。

# Q: 果园里有 15 棵树，工人今天种了一些树，种完后共有 21 棵。他们今天种了几棵？
initial_trees = 15       # 初始树数
final_trees = 21         # 最终树数
answer = final_trees - initial_trees  # 种的数量 = 差值
# answer = 6

# Q: 停车场有 3 辆车，又来了 2 辆，现在有几辆？
initial_cars = 3         # 初始车数
arrived = 2              # 新来的
answer = initial_cars + arrived
# answer = 5

# Q: {{new_question}}
```

---

**日期计算示例**（来自 Gao et al. 原始论文）：

```text
# Q: 2015 is coming in 36 hours. What is the date one week from today in MM/DD/YYYY?
from datetime import datetime
from dateutil.relativedelta import relativedelta
# If 2015 is coming in 36 hours, then today is 36 hours before.
today = datetime(2015, 1, 1) - relativedelta(hours=36)
# One week from today
one_week_from_today = today + relativedelta(weeks=1)
answer = one_week_from_today.strftime('%m/%d/%Y')

# Q: {{date_question}}
```

---

### Agent 集成版

在有 code interpreter 工具的 agent 中，把 PAL 作为工具调用策略内嵌到 system prompt：

```text
你是一个解题助手，配备了 Python 代码执行工具。

解题策略：
- 遇到数学计算、日期运算、数据统计类问题，优先使用 execute_python 工具
- 不要在推理过程中直接心算，把所有数值操作写成代码
- 代码中每一步用注释说明含义
- 将最终答案存入变量 `answer` 并打印

只有纯语言理解类问题（如摘要、翻译、情感分析）才直接回答，不使用代码工具。
```

工具调用流程：

```
用户问题
  → LLM 判断是否需要计算
  → 生成 Python 代码（带注释）
  → execute_python 工具执行
  → 读取 answer 变量值
  → 用自然语言组织最终回答
```

## PAL vs CoT

| 维度 | CoT | PAL |
|------|-----|-----|
| 推理载体 | 自然语言步骤 | Python 代码 |
| 计算准确性 | 受模型能力限制，可能出错 | 解释器执行，精确 |
| 可读性 | 人直接可读 | 需要懂代码才能审查 |
| 执行依赖 | 无，纯文本 | 需要代码解释器 |
| 适用任务 | 广泛，含语言推理 | 聚焦可编程化推理 |
| 调试方式 | 重新审查文字推理链 | 运行代码、检查中间变量 |
| 延迟 | 较低（无执行开销） | 略高（含代码执行时间） |

**选择原则**：任务涉及计算，优先 PAL；任务是语言性推理（因果分析、概念解释、道德判断），优先 CoT；两者可以组合——用 CoT 做任务分解，用 PAL 处理计算子任务。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | 代码生成质量高，注释习惯好；原生支持 extended thinking，可在生成代码前用思维链做任务分析 | 生成的代码默认会用 f-string 等现代 Python 语法，执行环境需支持 Python 3.8+ |
| GPT-4 | few-shot PAL 效果稳定，代码风格一致；o1/o3 系列会在内部推理链里自发写伪代码，不一定是可执行 Python | o1/o3 内置推理不可控，如果需要可执行代码，要在 prompt 中明确要求输出 Python 代码块 |
| Gemini 2 | 代码生成能力强，尤其是数学库的使用；支持原生 code interpreter | Flash 系列在复杂推理任务上代码可能过于简化，建议用 few-shot 示例校准代码粒度 |
| 开源模型（CodeLlama、DeepSeek-Coder、Qwen2.5-Coder） | 代码专精模型做 PAL 比通用模型更稳定；DeepSeek-Coder-V2 在数学代码上表现突出 | 开源通用模型（非代码专精）做 PAL 效果不稳定，建议优先选代码专精模型；few-shot 示例可显著提升稳定性 |

## 常见踩坑

**没有安全沙箱就执行生成代码** — 现象：`exec(llm_out)` 直接在主进程运行，恶意或意外代码可以读写文件、网络请求、退出进程。根因：把代码执行当成"就是跑个 Python"而忽略安全边界。避免方式：使用隔离的 subprocess、Docker 容器、或专门的 code sandbox 服务（如 E2B、Modal），限制可用模块和执行时间。

**生成的代码有逻辑 bug** — 现象：代码语法正确、能执行、但逻辑错误，输出了精确的错误答案。根因：模型错误理解了问题或把条件翻译成了错误的代码逻辑。避免方式：在 prompt 中要求模型在代码前加一句自然语言说明解题思路；对高风险计算加断言检查；或用 self-consistency（多次采样比较代码逻辑）。

**代码引用了未定义的变量或库** — 现象：生成的代码 `import` 了标准库之外的包，或引用了上下文里没有的变量。根因：few-shot 示例包含了特定库（如 `dateutil`），模型泛化时继续使用。避免方式：在 prompt 中明确声明可用的库；few-shot 示例使用尽量少的外部依赖；在执行前做 AST 静态检查。

**对不适合编程的问题强行生成代码** — 现象：模型生成了一段"代码"，但里面全是字符串拼接和 `print`，实际上只是把自然语言推理翻译成了代码外壳，没有任何计算价值。根因：prompt 要求"用代码回答所有问题"，但任务本身没有可编程的逻辑。避免方式：在 system prompt 中明确 PAL 的触发条件（涉及数值计算、日期运算、逻辑枚举时才用代码），其他任务直接用自然语言。

**忽略代码执行超时和无限循环** — 现象：模型生成了带循环的代码，某些输入导致死循环或超长运行时间。避免方式：sandbox 设置执行超时（建议 5-10 秒）；在 prompt 中强调避免循环或限制循环次数；对递归代码设置最大深度。

## 来源

- Gao, L. et al. (2022). *PAL: Program-aided Language Models*. ICML 2023. https://arxiv.org/abs/2211.10435
- Prompt Engineering Guide (DAIR.AI). PAL. https://www.promptingguide.ai/techniques/pal
- PAL 官方代码库: https://github.com/reasoning-machines/pal
- 关联 wiki: [[wiki/tool-system]] [[wiki/prompt-system]]
