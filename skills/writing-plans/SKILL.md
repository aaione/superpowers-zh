---
name: writing-plans
description: 当你有针对多步骤任务的 spec 或需求时使用，在动代码之前
---

# 编写计划 (Writing Plans)

## 概述 (Overview)

编写全面的实现计划，假设工程师对我们的代码库零上下文，且品味存疑。记录他们需要知道的一切：每个任务要触及哪些文件、代码、测试、他们可能需要查阅的文档、如何测试。以拆分好的小任务形式把整个计划给他们。DRY。YAGNI。TDD。频繁提交。

假设他们是有能力的开发者，但几乎不了解我们的工具集或问题域。假设他们不太懂好的测试设计。

**开始时宣布：** "我正在使用 writing-plans skill 来创建实现计划。"

**上下文：** 如果在隔离的 worktree 中工作，它应该在执行时通过 `superpowers:using-git-worktrees` skill 创建。

**计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的偏好优先于此默认值）

## 范围检查 (Scope Check)

如果 spec 涵盖多个独立子系统，它应该在 brainstorming 阶段被分解为子项目 spec。如果没有，建议把它拆成单独的计划 —— 每个子系统一个。每个计划应能独立产出可工作、可测试的软件。

## 文件结构 (File Structure)

在定义任务之前，规划哪些文件将被创建或修改，以及每个文件负责什么。这是分解决策被锁定的地方。

- 设计具有清晰边界和定义良好接口的单元。每个文件应有单一明确职责。
- 你能更好地推理可一次性放入上下文的代码，当文件聚焦时你的编辑更可靠。偏好更小、聚焦的文件，而非做太多的大文件。
- 一起改变的文件应放在一起。按职责拆分，而非按技术层。
- 在既有代码库中，遵循既有模式。如果代码库使用大文件，不要单方面重构 —— 但如果你正在修改的文件已变得难以驾驭，在计划中包含一次拆分是合理的。

此结构指导任务分解。每个任务应产生独立可理解的、自包含的改动。

## 任务定寸 (Task Right-Sizing)

一个任务是承载自己测试周期、且值得一个全新 reviewer 关卡的最小单元。在划定任务边界时：把设置、配置、脚手架和文档步骤折叠进需要它们的那个任务的交付物中；仅在一个 reviewer 能有意义地拒绝一个任务而批准其相邻任务时才拆分。每个任务以一个独立可测试的交付物结束。

## 拆分好的任务粒度 (Bite-Sized Task Granularity)

**每一步是一个动作（2-5 分钟）：**
- "编写失败的测试" —— 一步
- "运行它确保它失败" —— 一步
- "实现让测试通过的最小代码" —— 一步
- "运行测试确保它们通过" —— 一步
- "提交" —— 一步

## 计划文档头部 (Plan Document Header)

**每个计划必须以此头部开始：**

```markdown
# [功能名称] 实现计划

> **面向 agentic worker：** 必需的子 skill：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用 checkbox（`- [ ]`）语法进行跟踪。

**目标（Goal）：** [一句话描述这构建什么]

**架构（Architecture）：** [2-3 句关于方法的描述]

**技术栈（Tech Stack）：** [关键技术/库]

## 全局约束（Global Constraints）

[spec 的项目级需求 —— 版本下限、依赖限制、命名和文案规则、平台要求 —— 每条一行，值从 spec 中逐字复制。每个任务的需求隐式包含此小节。]

---
```

## 任务结构 (Task Structure)

````markdown
### 任务 N：[组件名称]

**文件：**
- 创建：`exact/path/to/file.py`
- 修改：`exact/path/to/existing.py:123-145`
- 测试：`tests/exact/path/to/test.py`

**接口：**
- 消费：[此任务从更早任务使用什么 —— 精确签名]
- 产出：[更晚任务依赖什么 —— 精确函数名、参数和返回类型。一个任务的 implementer 只看到自己的任务；这个块是他们了解相邻任务使用的名称和类型的方式。]

- [ ] **步骤 1：编写失败的测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **步骤 2：运行测试验证它失败**

运行：`pytest tests/path/test.py::test_name -v`
预期：FAIL，提示 "function not defined"

- [ ] **步骤 3：编写最小实现**

```python
def function(input):
    return expected
```

- [ ] **步骤 4：运行测试验证它通过**

运行：`pytest tests/path/test.py::test_name -v`
预期：PASS

- [ ] **步骤 5：提交**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 不要占位符 (No Placeholders)

每一步必须包含工程师需要的实际内容。这些是**计划失败** —— 绝不要写它们：
- "TBD"、"TODO"、"稍后实现"、"补充细节"
- "添加适当的错误处理" / "添加验证" / "处理边缘情况"
- "为上述编写测试"（没有实际测试代码）
- "类似于任务 N"（重复代码 —— 工程师可能乱序阅读任务）
- 描述做什么却不展示怎么做的步骤（代码步骤需要代码块）
- 引用任何任务中都未定义的类型、函数或方法

## 记住 (Remember)

- 总是精确的文件路径
- 每一步的完整代码 —— 如果某步改动代码，展示代码
- 精确命令及预期输出
- DRY、YAGNI、TDD、频繁提交

## 自审 (Self-Review)

写完完整计划后，以全新的眼光审视 spec，并对照它检查计划。这是你自己运行的检查清单 —— 不是 subagent 派发。

**1. Spec 覆盖：** 浏览 spec 中的每个小节/需求。你能指出实现它的任务吗？列出任何缺口。

**2. 占位符扫描：** 在你的计划中搜索危险信号 —— "不要占位符"小节中的任何模式。修复它们。

**3. 类型一致性：** 你在更晚任务中使用的类型、方法签名和属性名是否匹配更早任务中定义的？任务 3 中名为 `clearLayers()` 的函数在任务 7 中变成 `clearFullLayers()` 就是 bug。

如果发现问题，内联修复。无需重新审查 —— 修复后继续。如果发现没有任务的 spec 需求，添加该任务。

## 执行交接 (Execution Handoff)

保存计划后，提供执行选择：

**"计划完成并保存到 `docs/superpowers/plans/<filename>.md`。两种执行选项：**

**1. Subagent 驱动（推荐）** - 我为每个任务派发全新 subagent，任务之间 review，快速迭代

**2. 内联执行** - 使用 executing-plans 在本会话执行任务，带检查点的批量执行

**选哪种？"**

**如果选择 Subagent 驱动：**
- **必需的子 skill：** 使用 superpowers:subagent-driven-development
- 每个任务全新 subagent + 两阶段 review

**如果选择内联执行：**
- **必需的子 skill：** 使用 superpowers:executing-plans
- 带检查点的批量执行以供 review
