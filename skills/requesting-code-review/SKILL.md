---
name: requesting-code-review
description: 当完成任务、实现主要功能或合并之前使用，以验证工作满足需求
---

# 请求代码审查 (Requesting Code Review)

派发一个 code reviewer subagent，在问题级联之前捕获它们。reviewer 获得精确构建的用于评估的上下文——绝不是你会话的历史。这使 reviewer 聚焦于工作产物，而非你的思考过程，并为你继续工作保留自己的上下文。

**核心原则：** 尽早 review，频繁 review。

## 何时请求审查 (When to Request Review)

**强制：**
- subagent-driven development 中每个任务之后
- 完成主要功能之后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（新鲜视角）
- 重构之前（基线检查）
- 修复复杂 bug 之后

## 如何请求 (How to Request)

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派发 code reviewer subagent：**

派发一个 `general-purpose` subagent，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` - 你构建内容的简要摘要
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始 commit
- `{HEAD_SHA}` - 结束 commit

**3. 根据反馈行动：**
- 立即修复 Critical 问题
- 在继续之前修复 Important 问题
- Minor 问题记下留待以后
- 如果 reviewer 错了，提出反驳（附理由）

## 示例 (Example)

```
[刚完成任务 2：添加验证函数]

你：继续之前，让我请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派发 code reviewer subagent]
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，含 4 种问题类型
  PLAN_OR_REQUIREMENTS: docs/superpowers/plans/deployment-plan.md 中的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent 返回]：
  优点：架构清晰，测试真实
  问题：
    Important: 缺少进度指示器
    Minor: 报告间隔的魔法数字（100）
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流集成 (Integration with Workflows)

**Subagent-Driven Development：**
- 每个任务之后 review
- 在问题累积之前捕获它们
- 进入下一个任务之前修复

**Executing Plans：**
- 每个任务或自然检查点之后 review
- 获取反馈，应用，继续

**临时开发（Ad-Hoc Development）：**
- 合并前 review
- 卡住时 review

## 危险信号 (Red Flags)

**绝不：**
- 因为"很简单"而跳过 review
- 忽略 Critical 问题
- 带着未修复的 Important 问题继续
- 与有效的技术反馈争论

**如果 reviewer 错了：**
- 以技术理由反驳
- 展示证明它有效的代码/测试
- 请求澄清

见模板：[code-reviewer.md](code-reviewer.md)
