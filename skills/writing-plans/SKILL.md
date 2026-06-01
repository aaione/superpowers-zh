---
name: writing-plans
description: 当你有 spec 或多步骤任务的需求时使用，在接触代码之前
---

# Writing Plans

## 概述 (Overview)

编写全面的实施 plans，假设工程师对我们的 codebase 零背景且品味有问题。记录他们需要知道的一切：每个任务涉及哪些文件、代码、测试、他们可能需要检查的文档、如何测试它。以小节任务的形式给他们整个 plan。DRY. YAGNI. TDD. 频繁 commits。

假设他们是一位熟练的开发人员，但几乎不了解我们的工具集或问题领域。假设他们不太了解好的测试设计。

**开始时宣布:** "I'm using the writing-plans skill to create the implementation plan."

**上下文:** 如果在隔离的 worktree 中工作，它应该在执行时通过 `superpowers:using-git-worktrees` skill 创建。

**保存 plans 到:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (用户的 plan 位置偏好会覆盖此默认值)

## 范围检查 (Scope Check)

如果 spec 涵盖多个独立的子系统，它应该已经在 brainstorming 期间被分解为子项目 specs。如果没有，建议将其拆分为单独的 plans——每个子系统一个。每个 plan 应该能独立产出可工作的、可测试的软件。

## 文件结构 (File Structure)

在定义任务之前，先梳理哪些文件将被创建或修改，以及每个文件负责什么。这是分解决策被锁定的地方。

- 设计具有清晰边界和明确定义接口的单元。每个文件应该有一个明确的职责。
- 你对能一次性放在上下文中的代码推理得更好，当文件聚焦时你的编辑也更可靠。优先选择较小的、聚焦的文件，而不是做太多事情的大文件。
- 一起变更的文件应该放在一起。按职责拆分，而不是按技术层拆分。
- 在现有 codebase 中，遵循已建立的模式。如果 codebase 使用大文件，不要单方面重构——但如果你正在修改的文件已经变得笨拙，在 plan 中包含拆分是合理的。

此结构决定了任务分解。每个任务应该产出独立有意义的、自包含的变更。

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

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构 (Task Structure)

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 无占位符 (No Placeholders)

每一步必须包含工程师所需的实际内容。这些是 **plan 失败**——绝不要写这些：
- "TBD"、"TODO"、"implement later"、"fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (没有实际测试代码)
- "Similar to Task N" (重复代码——工程师可能不按顺序阅读任务)
- 描述做什么但不展示怎么做的步骤 (代码步骤需要代码块)
- 引用未在任何任务中定义的类型、函数或方法

## 记住 (Remember)

- 总是确切的文件路径
- 每个步骤中的完整代码——如果一个步骤更改代码，展示代码
- 确切的命令和预期的输出
- DRY, YAGNI, TDD, 频繁 commits

## 自我审查 (Self-Review)

写完完整的 plan 后，用全新的眼光查看 spec 并对照 plan 检查。这是你自己运行的检查清单——不是 subagent dispatch。

**1. Spec 覆盖率:** 浏览 spec 中的每个部分/需求。你能指出哪个任务实现了它吗？列出任何缺口。

**2. 占位符扫描:** 在你的 plan 中搜索危险信号——上面 "无占位符" 部分中的任何模式。修复它们。

**3. 类型一致性:** 你在后续任务中使用的类型、方法签名和属性名称是否与早期任务中定义的匹配？Task 3 中叫 `clearLayers()` 的函数在 Task 7 中变成了 `clearFullLayers()` 就是一个 bug。

如果发现问题，内联修复。无需重新审查——修复后继续。如果发现 spec 需求没有对应任务，添加任务。

## 执行交接 (Execution Handoff)

保存 plan 后，提供执行选择:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**如果选择 Subagent-Driven:**
- **REQUIRED SUB-SKILL:** 使用 superpowers:subagent-driven-development
- 每个任务新鲜 subagent + 两阶段审查

**如果选择 Inline Execution:**
- **REQUIRED SUB-SKILL:** 使用 superpowers:executing-plans
- 带有审查检查点的批量执行
