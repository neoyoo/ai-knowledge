---
pattern: graph-prompting
category: reasoning
tags: [graph, structured-reasoning, relationship, knowledge-graph]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_wiki: [wiki/prompt-system]
related_patterns: [chain-of-thought, tree-of-thoughts]
---

# Graph Prompting

> 用图结构（节点 + 边）组织 prompt 中的信息和推理路径，适合处理实体关系和网络结构问题。

## 本质

大多数 prompt 是线性的：背景 → 问题 → 答案。但现实世界的很多信息不是线性的——人际关系、知识依赖、系统组件、工作流节点之间是网状的。当你把网状信息强行写成线性文本，模型很容易丢失关系结构。

Graph Prompting 的核心思路：**显式地用图语言（节点 + 边）描述信息结构**，让模型在推理时能沿着关系边"遍历"，而不是在平铺文本里猜关系。

Liu et al., 2023 的 GraphPrompt 框架在图学习任务（节点分类、链接预测）上验证了这个方向：把图结构编码进 prompt，借助 LLM 的 in-context learning 能力处理图任务，在下游任务上超越了专门的图神经网络微调方法。

实践中不需要训练新模型，只需要在 prompt 里用清晰的格式表达图结构即可。

## 什么时候用

**实体关系推理** — "A 和 B 什么关系"、"C 依赖哪些组件"、"这个人的权限链路是什么"。关系清楚地写成图，模型推理准确率显著提升。

**依赖分析** — 代码模块依赖、任务前置条件、知识点依赖链。线性描述容易遗漏，图结构强制完整性。

**影响链分析** — "修改 X 会影响哪些下游节点"。适合架构审查、风险评估、变更分析。

**知识图谱问答** — 当回答问题需要在知识网络里多跳推理时（"A 的老师的学生中谁也在 B 机构工作过"）。

**工作流/流程分析** — 流程有分支、汇合、并行时，图比线性文本更准确地表达结构。

## 什么时候不用

**线性推理任务** — 数学推导、文本摘要等本质是顺序处理的任务，图结构引入不必要的复杂度。

**关系数量极少时** — 只有 2-3 个实体、1-2 条关系，直接说清楚比画图高效。

**动态关系频繁变化时** — 每次调用关系结构都不同，维护图的成本超过收益，考虑用程序动态生成。

**模型上下文窗口紧张时** — 完整的图文本描述比线性叙述占用更多 token，长图会撑满 context。

## 模板

### 基础版：邻接表格式

```text
以下是系统组件关系图（格式：节点 → [依赖节点]）：

节点列表:
- API Gateway
- Auth Service
- User Service
- Database

边（依赖关系）:
- API Gateway → [Auth Service, User Service]
- Auth Service → [User Service, Database]
- User Service → [Database]

问题：如果 Database 出现故障，哪些服务会受影响？请沿着依赖链分析。
```

---

### 结构化节点版：带属性的图

```text
知识图谱:

节点:
- [A] 张三 {角色: 工程师, 团队: 后端}
- [B] 李四 {角色: 工程师, 团队: 前端}
- [C] 王五 {角色: TL, 团队: 后端}
- [D] 项目X {状态: 进行中}

关系:
- A --[汇报给]--> C
- B --[协作]--> A
- A --[负责]--> D
- C --[审批]--> D

问题：{{question}}
```

---

### Agent 集成版：动态图构建

```python
def build_graph_prompt(nodes: dict, edges: list, question: str) -> str:
    node_text = "\n".join(
        f"- [{k}] {v['name']} {{{', '.join(f'{p}: {q}' for p, q in v.get('attrs', {}).items())}}}"
        for k, v in nodes.items()
    )
    edge_text = "\n".join(
        f"- {e['from']} --[{e['rel']}]--> {e['to']}"
        for e in edges
    )
    return f"""关系图:

节点:
{node_text}

关系:
{edge_text}

沿图结构推理回答：{question}"""
```

---

### Markdown 表格版（适合简单关系）

```text
| 实体 | 关系 | 目标实体 |
|------|------|---------|
| 模块A | 依赖 | 模块B |
| 模块A | 依赖 | 模块C |
| 模块B | 依赖 | 数据库 |

{{question}}
```

## 组合与选择

**Graph Prompting + CoT** → 图结构提供关系框架，CoT 在图上做多跳推理。让模型先"加载图"，再一步步沿着关系边推理。适合复杂的多跳问答。

**Graph Prompting + ReAct** → 图是静态背景，ReAct 动态扩展图信息（通过工具查询补充节点/边）。适合图信息不完整、需要实时补全的场景。

**Graph Prompting vs Tree-of-Thoughts** — ToT 是推理路径的树形展开（一个问题有多条思路）；Graph Prompting 是输入数据的图形表达（问题本身是图结构）。两者不互斥：数据是图时用 Graph Prompting，推理有多路径时再加 ToT。

**Graph Prompting vs Prompt Chaining** — 关系简单（线性依赖链）用 Prompt Chaining 逐步处理；关系复杂（多对多、环形依赖）用 Graph Prompting 一次性表达完整结构。

## 模型差异

| 模型 | 表现 | 注意事项 |
|------|------|---------|
| Claude 4 | 理解图结构能力强，能准确识别邻接表和 Markdown 表格格式，多跳推理稳定 | 超过 20 个节点后推理质量下降，建议分拆子图 |
| GPT-4 | 图结构解析稳定，支持多种格式；o1/o3 对图推理有额外提升 | 节点 ID 命名要清晰，避免用单字母（易混淆） |
| Gemini 2 | 处理图结构能力一般，建议用最简单的邻接表格式 | 复杂图容易"漏边"，建议在问题末尾要求模型先复述图结构再推理 |
| 开源模型 | 7B-13B 模型对图结构的理解能力有限 | 建议用极简格式（A->B, B->C），避免复杂属性描述 |

## 常见踩坑

**图结构描述和自然语言混在一起** — 把图定义夹在段落叙述里，模型很难提取结构信息。解决：图结构单独用代码块或列表格式，与任务描述明确分隔。

**节点命名不一致** — 同一个节点在不同地方叫"数据库"、"DB"、"database"，模型会以为是三个节点。解决：全局统一节点名称，建议在节点列表里定义标准名。

**忘记声明关系方向** — "A 依赖 B" 和 "B 依赖 A" 完全相反，但不加箭头方向时模型可能猜错。解决：始终用 `A --[rel]--> B` 格式，箭头方向必须明确。

**图太大一次性放入** — 50+ 节点的完整图塞进 prompt，模型会"迷失"。解决：按问题范围剪枝，只放相关子图；或用 Prompt Chaining 分多次处理子图。

**期待模型发现图中隐含的规律** — 模型会沿已知边推理，但不会"发明"新的关系。如果问题需要推断隐含关系，需要显式告诉模型"基于以下规则推断新关系"。

## 来源

- Liu, Z. et al. (2023). *GraphPrompt: Unifying Pre-Training and Downstream Tasks for Graph Neural Networks*. WWW 2023. https://arxiv.org/abs/2302.08043
- Prompt Engineering Guide (DAIR.AI). Graph Prompting. https://www.promptingguide.ai/techniques/graph
- 关联 wiki: [[wiki/prompt-system]]
