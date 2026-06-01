---
name: requesting-code-review
description: 当完成任务、实施主要功能或在合并之前验证工作是否符合要求时使用
---

# Requesting Code Review

Dispatch superpowers:code-reviewer subagent 以在问题级联之前捕获它们。

**核心原则:** 尽早审查，经常审查。

## 何时请求审查

**强制:**
- 在 subagent-driven development 中的每个任务之后
- 完成主要功能后
- 在合并到 main 之前

**可选但有价值:**
- 受阻时 (全新的视角)
- 重构前 (基线检查)
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHAs:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. Dispatch code-reviewer subagent:**

使用 Task 工具和 superpowers:code-reviewer 类型，填写 `code-reviewer.md` 中的模板

**占位符:**
- `{WHAT_WAS_IMPLEMENTED}` - 你刚刚构建了什么
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始 commit
- `{HEAD_SHA}` - 结束 commit
- `{DESCRIPTION}` - 简要总结

**3. 根据反馈行动:**
- 立即修复 Critical 问题
- 在继续之前修复 Important 问题
- 记录 Minor 问题以供稍后处理
- 如果 reviewer 错了，进行反驳 (带有理由)

## 示例

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch superpowers:code-reviewer subagent]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## 与工作流集成

**Subagent-Driven Development:**
- 每个任务后审查
- 在问题复合之前捕获它们
- 在移动到下一个任务之前修复

**Executing Plans:**
- 每个 batch (3 个任务) 后审查
- 获取反馈，应用，继续

**Ad-Hoc Development:**
- 合并前审查
- 受阻时审查

## 危险信号 (Red Flags)

**绝不 (Never):**
- 因为 "它很简单" 而跳过审查
- 忽略 Critical 问题
- 在未修复 Important 问题的情况下继续
- 与有效的技术反馈争论

**如果 reviewer 错了:**
- 用技术推理反驳
- 展示证明其有效的代码/测试
- 请求澄清

参见模板: requesting-code-review/code-reviewer.md
