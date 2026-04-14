---
section: skills/anti-patterns
description: 常见 skill 设计错误，避免让 skill 失效或产生负面效果
---

# Skill Anti-patterns

## 反模式列表

| Anti-pattern | 严重度 | 症状 |
|-------------|--------|------|
| [Skill 当内容容器用](skill-as-content-container.md) | 高 | SKILL.md > 500 行，每次加载浪费 context |
| [触发描述缺失](missing-trigger-description.md) | 高 | LLM 从不主动使用这个 skill |
| [重复造轮子](reinventing-existing-skills.md) | 中 | 写完才发现 skills.sh 已有同款 |
