---
anti-pattern: reinventing-existing-skills
category: skill-design
tags: [skill, reuse, skills.sh, discovery, maintenance]
severity: medium
related_patterns: [skill-discovery, community-skills, skill-composition]
---

# 重复造轮子

> 花时间写了一个 skill，事后发现社区早有质量更高、安装量更大的同类 skill，白费功夫还得额外维护。

## 症状

- 写完 skill 后，在 `skills.sh` 搜索时发现同名或同功能的热门 skill
- 自己维护的 skill 内容逐渐落后于社区版本
- 团队有多个人各自写了功能重叠的 skill
- skill 文件夹越来越大，但实际使用的内容大量重复

## 错误示例

自己花 3 小时写了 `git-commit-conventions.md`，内容基于 Conventional Commits 规范。

后来搜索发现：

```
$ npx skills find git commit
  
  github/awesome-copilot@git-commit
  ★ 23,900 installs | Updated 2 weeks ago
  Conventional Commits spec, commit types (feat/fix/docs/...), 
  breaking changes, scope conventions, examples for all types.
```

社区版本有 23.9K 安装，经过更多项目实战验证，内容比自己写的更全，还在持续更新。

类似情况还有：

| 自己写的 | 社区已有 |
|---------|---------|
| `python-type-hints.md` | `microsoft/pyright@python-types` 12K installs |
| `api-error-handling.md` | `stripe/api-patterns@errors` 8K installs |
| `dockerfile-best-practices.md` | `docker/official@dockerfile` 31K installs |

## 为什么有害

1. **重复维护负担**：社区 skill 在持续迭代，自己维护的版本会越来越落后。每次规范更新，你需要自己发现并手动同步。

2. **测试基础薄弱**：社区积累的 skill 经过数千个项目验证，覆盖了大量边缘情况。自己写的版本通常只经过少数场景测试，边角情况容易缺漏。

3. **时间成本**：写一个质量良好的 skill 需要 2-4 小时，如果社区有现成的，等于白花这些时间。

4. **生态割裂**：团队各自维护私有 skill，遇到问题无处参考，无法借助社区 issue 和 PR 改进质量。

## 正确做法

**创建任何新 skill 前，先执行搜索流程：**

```bash
# Step 1：用核心关键词搜索
npx skills find git commit

# Step 2：换近义词再搜一次
npx skills find conventional commits

# Step 3：浏览相关分类
# 打开 https://skills.sh/explore，按 category 筛选
```

**评估现有 skill 是否满足需求：**

| 判断维度 | 说明 |
|---------|------|
| 安装量 | > 1000 installs 通常质量有保障 |
| 更新频率 | 最近 3 个月内有更新说明在维护中 |
| 内容覆盖 | 快速阅读，核心场景是否覆盖？ |
| 语言匹配 | 英文 skill 能否满足中文流程需求？ |

**何时可以自己写：**

满足以下任一条件，才值得自己写：

- 搜索 3 个以上关键词，仍无相关结果
- 现有 skill 是英文，但团队流程需要中文描述
- 现有 skill 覆盖通用场景，但你的需求是高度特定的业务逻辑（如：公司内部代码审查规范）
- 现有 skill 与需求差异超过 40%（改造成本 > 重写成本）

**发现社区 skill 基本满足但有缺口时，优先考虑：**

1. 在本地用 `extends` 引用社区 skill，只补充差异部分
2. 向社区 skill 提 PR，贡献缺失内容
3. 最后才考虑 fork 并自维护

## 修复检查清单

- [ ] 创建前是否用至少 2 个不同关键词搜索过 `npx skills find`？
- [ ] 是否浏览了 `skills.sh` 相关分类页面？
- [ ] 是否评估了现有 skill 的安装量和更新频率？
- [ ] 是否确认需求与现有 skill 的差异足够大（> 40%），不满足才自己写？
- [ ] 如果选择自己写，是否记录了"为什么不用现有 skill"的决策原因？
