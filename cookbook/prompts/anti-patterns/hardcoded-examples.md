---
anti-pattern: hardcoded-examples
severity: warning
tags: [few-shot, overfitting, bias, examples]
related_patterns: [few-shot, zero-shot, active-prompt]
---

# Hardcoded Examples

> Few-shot 示例硬编码且从不更新，导致模型过度模仿示例的表面特征，在真实输入分布上系统性偏差。

## 本质

Few-shot 的威力在于"举例教学"，但这把双刃剑的反面是：**你给的示例决定了模型的"世界观"**。如果示例只覆盖了你数据分布的一小角，模型会把那一小角当成全部世界。

更隐蔽的问题是：硬编码示例一旦写下就很少被检查。系统上线六个月后，真实输入已经漂移，但示例还停留在开发时的想象里。模型忠实地模仿那些过时示例，输出开始系统性偏差，但因为"格式看起来对"，这种退化很难被及时发现。

三种典型失效模式：
- **类型偏差**：示例只覆盖了某几类输入，模型对其他类型强行套用示例模式
- **标签分布偏差**：某个标签的示例过多，模型在模糊边界上倾向于输出那个标签
- **格式过拟合**：模型学到了示例的表面格式而非任务逻辑，稍有偏差就"乱套"

## 症状

**边界案例一律归入最近的已知类别** — 模型遇到示例未覆盖的输入类型，不是输出"未知"或寻求澄清，而是强行匹配最相似的示例，产生错误但"格式正确"的输出。

**某个标签输出频率异常高** — 分析模型输出分布，发现某个标签占比远高于真实数据分布（如分类任务中"positive"占 80%，而真实数据只有 40%），与示例中该标签的比例相关。

**输出格式极度僵硬** — 模型对格式细节过分坚持：如果示例用双引号，模型绝不用单引号；示例中有 `→` 符号，模型在不该用的地方也加上。任何细微输入变化都会触发格式紊乱。

**生产效果远低于离线评估** — 离线评测集（通常和示例来自同一分布）表现很好，但生产环境效果明显下滑，因为真实用户输入的多样性远超示例覆盖范围。

## 错误示例

### 1. 标签分布不均

```python
# 客服工单情感分析
# 开发者手写了 6 个示例，但 5 个都是 Negative（因为"更有代表性"）

SENTIMENT_PROMPT = """分析客服工单的情感倾向。

工单：我的订单迟迟未到，已经等了两周了。
情感：Negative

工单：APP 经常崩溃，严重影响使用体验。
情感：Negative

工单：客服态度冷漠，问题没有得到解决。
情感：Negative

工单：退款流程太复杂，来回折腾好几次。
情感：Negative

工单：产品有些瑕疵但总体还可以。
情感：Negative  ← 本应是 Neutral，开发者简化处理了

工单：这款产品完全超出我的期望，非常满意！
情感：Positive

工单：{ticket}
情感："""
```

实际效果：模型对模糊工单（如"服务一般，价格还行"）几乎全部输出 `Negative`，因为负面示例的压倒性比例建立了强烈的先验偏向。

---

### 2. 示例只覆盖简单案例

```python
# 代码 bug 分类系统
# 开发时只收集了"教科书式"的 bug，真实 bug 更混乱

CODE_REVIEW_PROMPT = """分析以下代码问题并分类。

代码：for i in range(10): print(lst[i])
问题：列表越界
类别：index_error

代码：result = None; print(result.upper())
问题：None 值调用方法
类别：null_reference

代码：def get_user(id): return db.find(id)
问题：无异常处理
类别：missing_error_handling

代码：{code_snippet}
问题：
类别："""

# 真实代码中的复合问题：既有越界又有 null 引用
# 示例没有覆盖复合错误场景，模型只输出第一个匹配到的类别
# 整个错误分类系统的准确率在生产中比开发低 30%+
```

---

### 3. 示例格式过拟合

```python
# 示例全部是英文短句 + 单行输出
TRANSLATION_PROMPT = """翻译以下内容并标注语气。

Input: "Great job on the report!"
Output: 太棒了！(enthusiastic)

Input: "Please review the document."
Output: 请审阅文件。(neutral)

Input: "I'm not satisfied with this."
Output: 我对此不满意。(negative)

Input: {text}
Output: """

# 问题：示例全是短句，模型对长文本段落会截断输出
# 示例全是英文源文，遇到中文输入时模型会混淆输入输出方向
# 括号格式是示例带入的，遇到括号在文本中有含义的情况会冲突
```

## 为什么有害

**隐性偏见无法审计** — 硬编码示例的偏见是隐性的，没有文档、没有版本记录，不像代码 bug 那样可以被测试发现。随着时间推移，示例和真实分布的偏差越来越大，但没有任何系统会报警。

**分布漂移无感知** — 用户行为、产品功能、语言习惯都在随时间变化，但硬编码示例永远是六个月前的快照。模型用过时的"世界观"处理当下的输入，偏差是系统性的、持续的。

**调试极难** — 当你发现某类输入结果不对时，通常会先怀疑 prompt 逻辑、模型版本、系统 bug；很少有人第一时间想到"示例是不是选偏了"。示例偏差是最难定位的问题之一。

**掩盖了真实能力边界** — 过拟合的示例让模型"看起来能处理任何输入"，但只是在往示例套。这会让你对系统的真实鲁棒性产生错误判断，在关键时刻才暴露风险。

## 正确做法

### 原则 1：系统性示例选择

```python
from typing import List, Dict
import random

def select_balanced_examples(
    example_pool: List[Dict],
    n_per_label: int = 2,
    include_edge_cases: bool = True
) -> List[Dict]:
    """
    从示例池中系统选择，确保：
    1. 每个标签均等表示
    2. 覆盖边界案例
    3. 覆盖不同输入长度和风格
    """
    selected = []
    
    # 按标签分组
    by_label = {}
    for ex in example_pool:
        label = ex["label"]
        by_label.setdefault(label, []).append(ex)
    
    # 每个标签选 n 个，优先选"边界案例"
    for label, examples in by_label.items():
        edge_cases = [e for e in examples if e.get("is_edge_case")]
        clear_cases = [e for e in examples if not e.get("is_edge_case")]
        
        # 至少 1 个边界案例（如果有的话）
        n_edge = min(1, len(edge_cases)) if include_edge_cases else 0
        n_clear = n_per_label - n_edge
        
        selected.extend(random.sample(edge_cases, n_edge) if n_edge > 0 else [])
        selected.extend(random.sample(clear_cases, min(n_clear, len(clear_cases))))
    
    return selected

# 示例池应该包含
example_pool = [
    # 每个标签的典型正例
    {"input": "...", "label": "Positive", "is_edge_case": False},
    # 边界案例（标注时有争议的）
    {"input": "...", "label": "Neutral",  "is_edge_case": True},
    # 不同长度的输入
    {"input": "...", "label": "Negative", "is_edge_case": False, "length": "long"},
    # 不同风格的输入（正式/口语/技术术语）
    {"input": "...", "label": "Negative", "is_edge_case": False, "style": "casual"},
]
```

---

### 原则 2：动态示例选择（生产推荐）

```python
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

class DynamicFewShotSelector:
    """
    根据当前输入，从示例库中动态选择最相关的示例。
    适用于：示例库 50+ 个、输入分布变化大的生产系统。
    """
    
    def __init__(self, example_pool: List[Dict], embedding_model):
        self.pool = example_pool
        self.embedder = embedding_model
        # 预计算所有示例的向量（只做一次）
        self.embeddings = np.array([
            self.embedder.encode(ex["input"]) 
            for ex in example_pool
        ])
    
    def select(
        self, 
        query: str, 
        n: int = 4,
        ensure_label_diversity: bool = True
    ) -> List[Dict]:
        """选择与 query 最相关的 n 个示例，同时保证标签多样性"""
        
        query_emb = self.embedder.encode(query).reshape(1, -1)
        similarities = cosine_similarity(query_emb, self.embeddings)[0]
        
        if not ensure_label_diversity:
            # 直接取 top-n
            top_indices = np.argsort(similarities)[-n:][::-1]
            return [self.pool[i] for i in top_indices]
        
        # 保证每个标签至少一个示例
        by_label = {}
        for i, ex in enumerate(self.pool):
            label = ex["label"]
            if label not in by_label:
                by_label[label] = []
            by_label[label].append((i, similarities[i]))
        
        selected_indices = set()
        # 每个标签选相似度最高的一个
        for label, items in by_label.items():
            best_idx = max(items, key=lambda x: x[1])[0]
            selected_indices.add(best_idx)
        
        # 用剩余配额填充相似度最高的
        remaining = n - len(selected_indices)
        if remaining > 0:
            all_ranked = np.argsort(similarities)[-n*3:][::-1]
            for idx in all_ranked:
                if idx not in selected_indices:
                    selected_indices.add(idx)
                    remaining -= 1
                    if remaining == 0:
                        break
        
        # 按相似度排序（最相似的放最后——recency bias 利用）
        result = sorted(selected_indices, key=lambda i: similarities[i])
        return [self.pool[i] for i in result]
```

---

### 原则 3：示例质量监控

```python
# 定期（每月/每季度）检查示例是否仍然代表真实分布

def audit_examples(
    examples: List[Dict],
    recent_production_data: List[Dict],
    n_sample: int = 200
) -> Dict:
    """
    审计示例与近期生产数据的分布差异。
    返回：需要更新的信号。
    """
    sample = random.sample(recent_production_data, min(n_sample, len(recent_production_data)))
    
    # 检查标签分布
    example_label_dist = Counter(e["label"] for e in examples)
    production_label_dist = Counter(e["label"] for e in sample)
    
    # 检查输入长度分布
    example_avg_len = np.mean([len(e["input"]) for e in examples])
    production_avg_len = np.mean([len(e["input"]) for e in sample])
    
    alerts = []
    
    # 标签分布偏差 > 20%
    for label in set(list(example_label_dist.keys()) + list(production_label_dist.keys())):
        ex_ratio = example_label_dist.get(label, 0) / len(examples)
        prod_ratio = production_label_dist.get(label, 0) / len(sample)
        if abs(ex_ratio - prod_ratio) > 0.2:
            alerts.append(f"标签 '{label}' 分布偏差 {abs(ex_ratio - prod_ratio):.1%}")
    
    # 长度分布偏差 > 50%
    if abs(example_avg_len - production_avg_len) / production_avg_len > 0.5:
        alerts.append(f"输入长度偏差：示例均长 {example_avg_len:.0f}，生产均长 {production_avg_len:.0f}")
    
    return {
        "needs_update": len(alerts) > 0,
        "alerts": alerts,
        "recommendation": "更新示例库" if alerts else "示例仍具代表性"
    }
```

## 修复检查清单

- [ ] 示例覆盖所有目标标签/类型，各标签数量大致均衡（除非刻意反映真实不均衡分布）
- [ ] 示例中包含至少 1-2 个边界案例（标注时有争议的输入）
- [ ] 示例覆盖不同输入长度（短句、中段、长文本）
- [ ] 示例覆盖不同输入风格（正式、口语、专业术语）
- [ ] 有示例版本记录（什么时候加的，为什么加）
- [ ] 生产系统使用动态示例选择，或有定期（季度）的示例库更新机制
- [ ] 有离线评测集，且评测集不与示例来自同一批数据
- [ ] 定期比对示例分布与近期生产数据分布，差异 > 20% 时触发更新

## 关联

- [[few-shot]] — Few-shot 的正确使用方式，包括示例顺序、数量等
- [[zero-shot]] — 当示例反而引入偏差时，zero-shot 可能更稳健
- [[active-prompt]] — 自动化示例选择和优化的 advanced 方案

## 来源

- Min, S. et al. (2022). *Rethinking the Role of Demonstrations: What Makes In-Context Learning Work?* EMNLP 2022. https://arxiv.org/abs/2202.12837
- Lu, Y. et al. (2022). *Fantastically Ordered Prompts and Where to Find Them: Overcoming Few-Shot Prompt Order Sensitivity*. ACL 2022. https://arxiv.org/abs/2104.08786
- Zhao, Z. et al. (2021). *Calibrate Before Use: Improving Few-Shot Performance of Language Models*. ICML 2021. https://arxiv.org/abs/2102.09690
- Prompt Engineering Guide (DAIR.AI). Few-Shot Prompting — Common Pitfalls. https://www.promptingguide.ai/techniques/fewshot
