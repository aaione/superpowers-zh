# 文档审查系统实施计划

> **面向 agentic worker：** 必须使用 superpowers:subagent-driven-development（如果有 subagent）或 superpowers:executing-plans 来执行此计划。

**目标：** 在 brainstorming 和 writing-plans skill 中添加 spec 和 plan 文档审查循环。

**架构：** 在每个 skill 目录中创建审查者 prompt 模板。修改 skill 文件以在文档创建后添加审查循环。使用 Task tool 和 general-purpose subagent 进行审查者调度。

**技术栈：** Markdown skill 文件，通过 Task tool 进行 subagent 调度

**规格文档：** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## Chunk 1：Spec 文档审查者

此 chunk 将 spec 文档审查者添加到 brainstorming skill。

### Task 1：创建 Spec 文档审查者 Prompt 模板

**文件：**
- 创建：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **Step 1:** 创建审查者 prompt 模板文件

```markdown
# Spec 文档审查者 Prompt 模板

在调度 spec 文档审查者 subagent 时使用此模板。

**目的：** 验证 spec 完整、一致，并准备好进行实施规划。

**调度时机：** Spec 文档写入 docs/superpowers/specs/ 之后

```
Task tool (general-purpose):
  description: "审查 spec 文档"
  prompt: |
    你是一个 spec 文档审查者。验证此 spec 是否完整并准备好进行规划。

    **要审查的 Spec：** [SPEC_FILE_PATH]

    ## 检查内容

    | 类别 | 要查找的内容 |
    |----------|------------------|
    | 完整性 | TODO、占位符、"TBD"、不完整的章节 |
    | 覆盖度 | 缺失的错误处理、边界情况、集成点 |
    | 一致性 | 内部矛盾、冲突的需求 |
    | 清晰度 | 模糊的需求 |
    | YAGNI | 未被要求的功能、过度工程 |

    ## 关键点

    特别注意查找：
    - 任何 TODO 标记或占位文本
    - 标注"待后续定义"或"将在 X 完成后 spec"的章节
    - 明显比其他章节细节少的章节

    ## 输出格式

    ## Spec 审查

    **状态：** 通过 | 发现问题

    **问题（如有）：**
    - [章节 X]: [具体问题] - [为什么重要]

    **建议（参考性质）：**
    - [不阻碍通过的建议]
```

**审查者返回：** 状态、问题（如有）、建议
```

- [ ] **Step 2:** 验证文件创建正确

运行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **Step 3:** 提交

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### Task 2：在 Brainstorming Skill 中添加审查循环

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **Step 1:** 读取当前 brainstorming skill

运行：`cat skills/brainstorming/SKILL.md`

- [ ] **Step 2:** 在"After the Design"之后添加审查循环部分

找到"After the Design"部分，在文档说明之后、实施之前添加新的"Spec 审查循环"部分：

```markdown
**Spec 审查循环：**
编写 spec 文档后：
1. 调度 spec-document-reviewer subagent（参见 spec-document-reviewer-prompt.md）
2. 如果发现问题：
   - 修复 spec 文档中的问题
   - 重新调度审查者
   - 重复直到通过
3. 如果通过：继续实施设置

**审查循环指导：**
- 编写 spec 的同一个 agent 负责修复（保留上下文）
- 如果循环超过 5 次迭代，提交给人类进行指导
- 审查者是参考性质 — 如果你认为反馈不正确，请解释原因
```

- [ ] **Step 3:** 验证更改

运行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
预期：显示新的审查循环部分

- [ ] **Step 4:** 提交

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## Chunk 2：Plan 文档审查者

此 chunk 将 plan 文档审查者添加到 writing-plans skill。

### Task 3：创建 Plan 文档审查者 Prompt 模板

**文件：**
- 创建：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **Step 1:** 创建审查者 prompt 模板文件

```markdown
# Plan 文档审查者 Prompt 模板

在调度 plan 文档审查者 subagent 时使用此模板。

**目的：** 验证 plan chunk 完整、与 spec 匹配，并具有适当的任务分解。

**调度时机：** 每个 plan chunk 编写完成后

```
Task tool (general-purpose):
  description: "审查 plan chunk N"
  prompt: |
    你是一个 plan 文档审查者。验证此 plan chunk 是否完整并准备好进行实施。

    **要审查的 Plan chunk：** [PLAN_FILE_PATH] - 仅 Chunk N
    **参考 Spec：** [SPEC_FILE_PATH]

    ## 检查内容

    | 类别 | 要查找的内容 |
    |----------|------------------|
    | 完整性 | TODO、占位符、不完整的任务、缺失的步骤 |
    | Spec 对齐 | Chunk 覆盖了相关 spec 需求，没有范围蔓延 |
    | 任务分解 | 任务原子化、边界清晰、步骤可操作 |
    | 任务语法 | 任务和步骤使用复选框语法（`- [ ]`） |
    | Chunk 大小 | 每个 chunk 不超过 1000 行 |

    ## 关键点

    特别注意查找：
    - 任何 TODO 标记或占位文本
    - 标注"类似 X"但没有实际内容的步骤
    - 不完整的任务定义
    - 缺少验证步骤或预期输出

    ## 输出格式

    ## Plan 审查 - Chunk N

    **状态：** 通过 | 发现问题

    **问题（如有）：**
    - [Task X, Step Y]: [具体问题] - [为什么重要]

    **建议（参考性质）：**
    - [不阻碍通过的建议]
```

**审查者返回：** 状态、问题（如有）、建议
```

- [ ] **Step 2:** 验证文件创建成功

运行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **Step 3:** 提交

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### Task 4：在 Writing-Plans Skill 中添加审查循环

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 读取当前 skill 文件

运行：`cat skills/writing-plans/SKILL.md`

- [ ] **Step 2:** 添加逐 chunk 审查部分

在"Execution Handoff"部分之前添加：

```markdown
## Plan 审查循环

完成 plan 的每个 chunk 后：

1. 为当前 chunk 调度 plan-document-reviewer subagent
   - 提供：chunk 内容、spec 文档路径
2. 如果发现问题：
   - 修复 chunk 中的问题
   - 为该 chunk 重新调度审查者
   - 重复直到通过
3. 如果通过：继续下一个 chunk（如果是最后一个 chunk 则进入执行交接）

**Chunk 边界：** 使用 `## Chunk N: <name>` 标题来划分 chunk。每个 chunk 应 <=1000 行且逻辑上自包含。
```

- [ ] **Step 3:** 更新任务语法示例以使用复选框

将 Task Structure 部分改为显示复选框语法：

```markdown
### Task N: [组件名称]

- [ ] **Step 1:** 编写失败的测试
  - 文件：`tests/path/test.py`
  ...
```

- [ ] **Step 4:** 验证审查循环部分已添加

运行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
预期：显示新的审查循环部分

- [ ] **Step 5:** 验证任务语法示例已更新

运行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
预期：显示复选框语法 `### Task N:`

- [ ] **Step 6:** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## Chunk 3：更新 Plan 文档头部

此 chunk 更新 plan 文档头部模板以引用新的复选框语法要求。

### Task 5：在 Writing-Plans Skill 中更新 Plan 头部模板

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 读取当前 plan 头部模板

运行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **Step 2:** 更新头部模板以引用复选框语法

Plan 头部应注明任务和步骤使用复选框语法。更新头部注释：

```markdown
> **面向 agentic worker：** 必须使用 superpowers:subagent-driven-development（如果有 subagent）或 superpowers:executing-plans 来执行此计划。任务和步骤使用复选框（`- [ ]`）语法进行跟踪。
```

- [ ] **Step 3:** 验证更改

运行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
预期：显示更新后的头部，包含复选框语法提及

- [ ] **Step 4:** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```
