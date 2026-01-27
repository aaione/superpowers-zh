---
name: writing-plans
description: 当你有 specs 或多步骤任务的需求时使用，在接触代码之前
---

# Writing Plans

## 概述 (Overview)

编写全面的实施 plans，假设工程师对我们的 codebase 零背景且品味有问题。记录他们需要知道的一切：每个任务涉及哪些文件、代码、测试、他们可能需要检查的文档、如何测试它。以小节任务的形式给他们整个 plan。DRY. YAGNI. TDD. 频繁 commits。

假设他们是一位熟练的开发人员，但几乎不了解我们的工具集或问题领域。假设他们不太了解好的测试设计。

**开始时宣布:** "I'm using the writing-plans skill to create the implementation plan."

**上下文:** 这应该在一个专门的 worktree (由 brainstorming skill 创建) 中运行。

**保存 plans 到:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## 细粒度任务粒度 (Bite-Sized Task Granularity)

**每一步是一个动作 (2-5 分钟):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**每个 plan 必须以这个 header 开头:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构 (Task Structure)

```markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
```

## 记住 (Remember)

- 总是确切的文件路径
- Plan 中包含完整代码 (不仅仅是 "add validation")
- 确切的命令和预期的输出
- 用 @ 语法引用相关的 skills
- DRY, YAGNI, TDD, 频繁 commits

## 执行交接 (Execution Handoff)

保存 plan 后，提供执行选择:

**"Plan complete and saved to `docs/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?"**

**如果选择 Subagent-Driven:**
- **REQUIRED SUB-SKILL:** 使用 superpowers:subagent-driven-development
- 留在此会话中
- 每个任务新鲜 subagent + code review

**如果选择 Parallel Session:**
- 引导他们在 worktree 中打开新会话
- **REQUIRED SUB-SKILL:** 新会话使用 superpowers:executing-plans
