# SDD 任务级审查派发实现计划 (SDD Task-Scoped Review Dispatch Implementation Plan)

> **致 agent 工作者：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐个实现本计划。步骤使用 checkbox（`- [ ]`）语法进行跟踪。

**目标：** 将 SDD 的每任务审查范围限定在该任务内（以 diff 为先的阅读、有理由的范围扩展、不重复运行测试），同时最终的分支审查保持宽泛。

**架构：** 对 subagent-driven-development 技能进行四处文字编辑（每任务质量 prompt 改为自包含，不再委托给 merge-readiness 模板；spec prompt 增加第三个裁定通道和有依据的怀疑；implementer prompt 增加修复后重跑规则；SKILL.md 增加控制器指引），并在 `evals/` 子模块中新增一个 eval 场景。`skills/requesting-code-review/` 刻意保持不动。

**技术栈：** Markdown 技能文件；用于 quorum eval 的 Python setup 辅助脚本 + bash 检查 + story.md。

**规范：** `docs/superpowers/specs/2026-06-09-sdd-task-scoped-review-dispatch-design.md` —— 开始前先阅读它。其中已敲定的决策：完整重审保留；两个审查阶段保持分离；协调器保留模型判断；`requesting-code-review/` 保持宽泛。

**这些是塑造行为的文字文件，不是代码。** 它们没有单元测试。每个任务的验证步骤是确切的 `grep` 检查，确认编辑已落地；行为验证是任务 6（静态）和任务 7（在线 eval，由维护者把关）。

---

### 任务 1：将每任务质量审查者 prompt 重写为自包含 (Task 1: Rewrite the per-task quality reviewer prompt as self-contained)

当前文件委托给 `../requesting-code-review/code-reviewer.md`，那是一个 merge-readiness 审查（架构、安全、生产就绪、"是否可以合并？"）。用一份自包含的、任务级的模板替换整个文件。

**文件：**
- 重写：`skills/subagent-driven-development/code-quality-reviewer-prompt.md`

- [ ] **步骤 1：将文件全部内容替换为：**

````markdown
# 代码质量审查者 prompt 模板 (Code Quality Reviewer Prompt Template)

在派发代码质量审查 subagent 时使用此模板。

**目的：** 验证单个任务的实现是构建良好的（干净、经过测试、可维护）

**仅在 spec 合规审查通过后才派发。**

```
Subagent (general-purpose):
  description: "Review code quality for Task N"
  prompt: |
    你正在审查单个任务的实现的代码质量。这是一个任务级
    关卡，不是合并审查 —— 在所有任务完成之后会单独进行一次
    宽泛的全分支审查。

    ## 已实现的内容 (What Was Implemented)

    [DESCRIPTION]

    ## 任务要求（仅作背景） (Task Requirements (context only))

    [TASK_TEXT]

    ## 待审查的 Git 范围 (Git Range to Review)

    **Base:** [BASE_SHA]
    **Head:** [HEAD_SHA]

    ```bash
    git diff --stat [BASE_SHA]..[HEAD_SHA]
    git diff [BASE_SHA]..[HEAD_SHA]
    ```

    ## 只读审查 (Read-Only Review)

    你的审查对这个检出是只读的。不要以任何方式改动工作树、
    暂存区、HEAD 或分支状态。使用 `git show`、
    `git diff` 和 `git log` 等工具来检查历史。

    ## 范围 (Scope)

    spec 合规性已由另一名审查者验证过。不要重新
    检查代码是否匹配需求或计划。

    从 diff 开始。先阅读被修改的文件。只有为了评估一个你能
    指名的具体风险时，才去检查 diff 之外的代码 —— 并在你的
    报告中指明它。跨切面的修改是合理的、可指名的风险：如果
    diff 改变了锁的顺序、某个函数或 API 契约，或共享的可变
    状态，那么检查调用点就是正确的方法。不要默认爬取整个
    代码库。

    ## 测试 (Tests)

    implementer 已经运行过测试，并针对完全相同的代码报告了
    附带 TDD 证据的结果。不要为了确认他们的报告而重新运行
    测试套件。只有当阅读代码引发了一个现有运行结果无法解答
    的具体疑问时才运行测试 —— 而且是一个聚焦的测试，绝不
    是包级别的套件、竞态检测器运行，或重复/高次数的循环。如果
    似乎有必要做重度验证，请在报告中建议，而不是
    自己运行。如果你无法在此环境中运行命令，请指出你
    会运行的测试。

    ## 检查什么 (What to Check)

    **代码质量：**
    - 关注点是否清晰分离？
    - 错误处理是否得当？
    - 是否做到 DRY 而没有过早抽象？
    - 边界情况是否处理？

    **测试：**
    - 新增和修改的测试是否验证真实行为，而非 mock？
    - 该任务的边界情况是否覆盖？

    **结构：**
    - 每个文件是否都有单一清晰的职责和定义良好的接口？
    - 单元是否被分解到可以独立理解和测试？
    - 实现是否遵循计划中的文件结构？
    - 这次修改是否创建了已经很大新文件，或
      显著增长了已有文件？（不要标记预先存在的文件
      大小 —— 聚焦于这次修改带来的贡献。）

    ## 校准 (Calibration)

    按实际严重程度对问题分类。不是所有问题都是 Critical。
    在列出问题之前先肯定做得好的地方 —— 准确的肯定
    有助于 implementer 信任其余的反馈。

    ## 输出格式 (Output Format)

    ### 优点 (Strengths)
    [哪些做得好？要具体。]

    ### 问题 (Issues)

    #### Critical（必须修复） (Critical (Must Fix))
    [缺陷、数据丢失风险、损坏的功能]

    #### Important（应当修复） (Important (Should Fix))
    [糟糕的错误处理、测试缺口、结构性问题]

    #### Minor（有则更好） (Minor (Nice to Have))
    [代码风格、可优化之处]

    对于每个问题：
    - 文件:行号 引用
    - 哪里有问题
    - 为什么重要
    - 如何修复（如果非显而易见）

    ### 评估 (Assessment)

    **任务质量 (Task quality):** [Approved | Needs fixes]

    **理由 (Reasoning):** [1-2 句技术评估]
```

**占位符：**
- `[DESCRIPTION]` —— 任务摘要，来自 implementer 的报告
- `[TASK_TEXT]` —— 该任务的需求文本或计划引用，作为背景
- `[BASE_SHA]` —— 该任务之前的提交
- `[HEAD_SHA]` —— 当前提交

**审查者返回：** 优点、问题（Critical/Important/Minor）、任务质量裁定
````

- [ ] **步骤 2：验证重写已落地**

运行：`grep -c "requesting-code-review" skills/subagent-driven-development/code-quality-reviewer-prompt.md || echo ABSENT`
预期：`ABSENT`（不再委托）

运行：`grep -n "Task quality:" skills/subagent-driven-development/code-quality-reviewer-prompt.md | head -2`
预期：一个匹配（输出格式的裁定行；"审查者返回" 页脚写的是 "Task quality verdict"，没有冒号）

运行：`grep -n "worktree add\|Ready to merge" skills/subagent-driven-development/code-quality-reviewer-prompt.md || echo CLEAN`
预期：`CLEAN`

- [ ] **步骤 3：提交**

```bash
git add skills/subagent-driven-development/code-quality-reviewer-prompt.md
git commit -m "Make per-task quality reviewer prompt self-contained and task-scoped"
```

---

### 任务 2：spec 审查者 prompt 的清理 (Task 2: Spec reviewer prompt cleanups)

对 `skills/subagent-driven-development/spec-reviewer-prompt.md` 的四处精确编辑。当前行号针对的是提交 f55642e 时的文件。

**文件：**
- 修改：`skills/subagent-driven-development/spec-reviewer-prompt.md`

- [ ] **步骤 1：添加依据-diff 裁定的条款。** 在该行（当前第 31 行）之后：

```
    Only read files in this diff. Do not crawl the broader codebase.
```

插入一个空行和：

```
    Spec compliance is judged by reading the diff against the requirements.
    The implementer already ran the tests and reported TDD evidence — do not
    re-run them. If a requirement cannot be verified from this diff alone
    (it lives in unchanged code or spans tasks), report it as a ⚠️ item
    instead of broadening your search.
```

- [ ] **步骤 2：精简只读部分。** 替换（当前第 35 行）：

```
    Your review is read-only on this checkout. Do not mutate the working tree, the index, HEAD, or branch state in any way. Use tools like `git show`, `git diff`, and `git log` to inspect history. If you need a working copy of a different revision, check it out into a separate temporary directory (e.g. `git worktree add /tmp/review-[SHA] [SHA]`) — never move HEAD on this checkout.
```

为：

```
    Your review is read-only on this checkout. Do not mutate the working tree, the index, HEAD, or branch state in any way. Use tools like `git show`, `git diff`, and `git log` to inspect history.
```

- [ ] **步骤 3：让怀疑有依据。** 替换（当前第 39-40 行）：

```
    The implementer finished suspiciously quickly. Their report may be incomplete,
    inaccurate, or optimistic. You MUST verify everything independently.
```

为：

```
    Treat the implementer's report as unverified claims about the code. It may
    be incomplete, inaccurate, or optimistic. Verify the claims against the diff.
```

- [ ] **步骤 4：添加第三个裁定通道。** 替换（当前第 74-76 行）：

```
    Report:
    - ✅ Spec compliant (if everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
```

为：

```
    Report:
    - ✅ Spec compliant (if everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
    - ⚠️ Cannot verify from diff: [requirements you could not verify from the
      diff alone, and what the controller should check — report alongside the
      ✅/❌ verdict for everything you could verify]
```

- [ ] **步骤 5：验证**

运行：`grep -n "suspiciously\|worktree add" skills/subagent-driven-development/spec-reviewer-prompt.md || echo CLEAN`
预期：`CLEAN`

运行：`grep -c "⚠️" skills/subagent-driven-development/spec-reviewer-prompt.md`
预期：`2`（依据-diff 条款 + 裁定通道）

- [ ] **步骤 6：提交**

```bash
git add skills/subagent-driven-development/spec-reviewer-prompt.md
git commit -m "Spec reviewer: judge from the diff, grounded skepticism, ⚠️ verdict channel"
```

---

### 任务 3：implementer prompt —— 修复审查发现后重跑测试 (Task 3: Implementer prompt — re-run tests after fixing review findings)

审查者"不要重跑 implementer 的测试"的规则假设 implementer 在每次修复后都会重跑测试。让这一假设成真。

**文件：**
- 修改：`skills/subagent-driven-development/implementer-prompt.md`

- [ ] **步骤 1：插入一个新章节。** 紧接在该行（当前第 100 行）之前：

```
    ## Report Format
```

插入：

```
    ## 修复审查发现之后 (After Review Findings)

    如果审查者发现问题并且你修复了它们，请重新运行覆盖
    修改后代码的测试，并将结果包含在你的修复报告中。审查者
    不会替你重跑测试 —— 你的报告就是测试证据。

```

- [ ] **步骤 2：验证**

运行：`grep -n "After Review Findings" skills/subagent-driven-development/implementer-prompt.md`
预期：一个匹配，位于 `## Report Format` 之前的某行

- [ ] **步骤 3：提交**

```bash
git add skills/subagent-driven-development/implementer-prompt.md
git commit -m "Implementer prompt: re-run covering tests after fixing review findings"
```

---

### 任务 4：SKILL.md 控制器变更 (Task 4: SKILL.md controller changes)

对 `skills/subagent-driven-development/SKILL.md` 的六处精确编辑。当前行号针对的是提交 f55642e。

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md`

- [ ] **步骤 1：将最终审查流程图节点指向宽泛模板。** 节点标签 `Dispatch final code reviewer subagent for entire implementation` 出现 3 次（当前第 65、84、85 行）。在全部 3 处出现中，将标签字符串替换为：

```
Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)
```

（Graphviz 节点按标签文本匹配 —— 三者必须逐字节相同，否则流程图会长出一个幻影节点。）

- [ ] **步骤 2：按判断选择模型。** 替换（当前第 97-99 行）：

```
**Architecture, design, and review tasks**: use the most capable available model.

**Task complexity signals:**
```

为：

```
**Architecture and design tasks**: use the most capable available model.

**Review tasks**: choose the model with the same judgment, scaled to the
diff's size, complexity, and risk. A small mechanical diff does not need the
most capable model; a subtle concurrency change does.

**Task complexity signals (implementation tasks):**
```

- [ ] **步骤 3：添加控制器指引章节。** 紧接在该行（当前第 122 行）之前：

```
## Prompt Templates
```

插入：

```
## 处理 spec 审查者的 ⚠️ 条目 (Handling Spec Reviewer ⚠️ Items)

spec 审查者可能报告 "⚠️ Cannot verify from diff" 条目 —— 即位于
未修改代码中或跨任务的需求。这些条目不会阻止派发代码质量审查者，
但你必须在标记任务完成之前亲自解决每一个：你掌握审查者所缺乏的
计划和跨任务上下文。如果你确认某个条目是真实缺口，把它当作一次
失败的 spec 审查 —— 打回给 implementer 并重新审查。

## 构建审查者 prompt (Constructing Reviewer Prompts)

每任务审查是任务级关卡。宽泛审查只在
最终全分支审查时进行一次。当你填写审查者模板时：

- 不要在没有具体、特定于任务的理由的情况下添加开放式指令，
  比如 "检查所有用法" 或 "如有必要运行竞态测试"
- 不要让审查者重跑 implementer 已经在相同代码上运行过的
  测试 —— implementer 的报告携带测试证据

```

- [ ] **步骤 4：prompt 模板列表 —— 添加最终审查指针。** 替换（当前第 126 行）：

```
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) - Dispatch code quality reviewer subagent
```

为：

```
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) - Dispatch code quality reviewer subagent
- Final whole-branch review: use superpowers:requesting-code-review's [code-reviewer.md](../requesting-code-review/code-reviewer.md)
```

- [ ] **步骤 5：示例工作流裁定词汇。** 两处替换：

替换（当前第 157 行）：
```
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.
```
为：
```
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Task quality: Approved.
```

替换（当前第 191 行）：
```
Code reviewer: ✅ Approved
```
为：
```
Code reviewer: ✅ Task quality: Approved
```

（最终审查者的 "ready to merge" 行，当前第 199 行，保持不变。）

- [ ] **步骤 6：Integration 章节。** 替换（当前第 272 行）：

```
- **superpowers:requesting-code-review** - Code review template for reviewer subagents
```

为：

```
- **superpowers:requesting-code-review** - Code review template for the final whole-branch review
```

- [ ] **步骤 7：验证**

运行：`grep -c "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" skills/subagent-driven-development/SKILL.md`
预期：`3`

运行：`grep -n "most capable available model" skills/subagent-driven-development/SKILL.md`
预期：恰好一个匹配（架构/设计条目）

运行：`grep -n "Handling Spec Reviewer\|Constructing Reviewer Prompts" skills/subagent-driven-development/SKILL.md`
预期：两个章节标题，均位于 `## Prompt Templates` 之前

运行：`grep -c "Task quality: Approved" skills/subagent-driven-development/SKILL.md`
预期：`2`

- [ ] **步骤 8：提交**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "SDD controller: reviewer prompt budgets, ⚠️ handling, final-review pointer, model judgment"
```

---

### 任务 5：新 eval 场景 —— 每任务质量审查者抓住一个植入缺陷 (Task 5: New eval scenario — per-task quality reviewer catches a planted defect)

位于 `evals/` **子模块**（独立仓库，`superpowers-evals`）中。在那里开一个分支工作；父仓库的子模块指针更新在收尾时按 `evals/CLAUDE.md` 进行。

夹具计划的 Task 2 实现代码片段逐字复制了 Task 1 的格式化逻辑。这种重复是符合 spec 的，所以 spec 审查者应放行 —— 每任务质量审查者是被测关卡（DRY 违规）。

**文件：**
- 创建：`evals/setup_helpers/sdd_quality_defect_plan.py`
- 修改：`evals/setup_helpers/__init__.py`
- 创建：`evals/scenarios/sdd-quality-reviewer-catches-planted-defect/story.md`
- 创建：`evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`
- 创建：`evals/scenarios/sdd-quality-reviewer-catches-planted-defect/checks.sh`

- [ ] **步骤 0：在子模块中开分支**

```bash
cd evals
git checkout -b sdd-quality-defect-scenario
```

- [ ] **步骤 1：创建 `evals/setup_helpers/sdd_quality_defect_plan.py`：**

````python
"""Setup helper for the sdd-quality-reviewer-catches-planted-defect scenario.

Scaffolds a tiny Node project with a 2-task plan whose Task 2
implementation snippet duplicates Task 1's formatting logic verbatim.
The duplication is spec-compliant — the requirements only describe
behavior — so the spec compliance reviewer should pass it. The test
measures whether the per-task code quality reviewer catches the DRY
violation and forces a refactor in the review-fix loop.
"""

from __future__ import annotations

from pathlib import Path

from setup_helpers.base import _git

PACKAGE_JSON = """\
{
  "name": "report-quality",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "test": "node --test"
  }
}
"""

PLAN_BODY = """\
# Report Formatter — Implementation Plan

Two report formatting functions. Implement exactly what each task
specifies.

## Task 1: User Report

**File:** `src/report.js`

**Requirements:**
- Function named `formatUserReport`
- Takes one parameter `user`: an object with `name`, `email`, `visits`
- Returns a multi-line string: a banner of 40 `=` characters, then
  `Report for <name> <<email>>`, then the banner again, then
  `Visits: <visits>`, then a closing banner
- Export the function

**Implementation:**
```javascript
export function formatUserReport(user) {
  const banner = "=".repeat(40);
  const lines = [];
  lines.push(banner);
  lines.push(`Report for ${user.name} <${user.email}>`);
  lines.push(banner);
  lines.push(`Visits: ${user.visits}`);
  lines.push(banner);
  return lines.join("\\n");
}
```

**Tests:** Create `test/report.test.js` verifying:
- the result contains `Report for Ada <ada@example.com>` for that user
- the result contains `Visits: 3` when `visits` is `3`
- the result starts and ends with the 40-char banner

**Verification:** `npm test`

## Task 2: Admin Report

**File:** `src/report.js` (add to existing file)

**Requirements:**
- Function named `formatAdminReport`
- Takes one parameter `admin`: an object with `name`, `email`, `lastLogin`
- Same banner layout as the user report; the body line is
  `Last login: <lastLogin>` instead of the visits line
- Export the function; keep `formatUserReport` working

**Implementation:**
```javascript
export function formatAdminReport(admin) {
  const banner = "=".repeat(40);
  const lines = [];
  lines.push(banner);
  lines.push(`Report for ${admin.name} <${admin.email}>`);
  lines.push(banner);
  lines.push(`Last login: ${admin.lastLogin}`);
  lines.push(banner);
  return lines.join("\\n");
}
```

**Tests:** Add to `test/report.test.js`:
- the result contains `Report for Grace <grace@example.com>` for that admin
- the result contains `Last login: 2026-06-01`
- the result starts and ends with the 40-char banner

**Verification:** `npm test`
"""


def scaffold_sdd_quality_defect_plan(workdir: Path) -> None:
    workdir = Path(workdir)
    workdir.mkdir(parents=True, exist_ok=True)
    _git(["git", "init", "-b", "main"], cwd=workdir)
    _git(["git", "config", "user.email", "drill@test.local"], cwd=workdir)
    _git(["git", "config", "user.name", "Drill Test"], cwd=workdir)

    (workdir / "package.json").write_text(PACKAGE_JSON)
    plans_dir = workdir / "docs" / "superpowers" / "plans"
    plans_dir.mkdir(parents=True, exist_ok=True)
    (plans_dir / "report-plan.md").write_text(PLAN_BODY)

    _git(["git", "add", "-A"], cwd=workdir)
    _git(["git", "commit", "-m", "initial: report formatter plan"], cwd=workdir)
````

（注意 PLAN_BODY 内 JS 代码片段中的 `\\n`：Python 源码必须在
markdown 中产生字面量 `\n`，这样 JS 才会读到 `lines.join("\n")`。）

- [ ] **步骤 2：注册该 helper。** 在 `evals/setup_helpers/__init__.py` 中：

在该行之后：
```python
from setup_helpers.sdd_real_projects import scaffold_sdd_go_fractals, scaffold_sdd_svelte_todo
```
添加：
```python
from setup_helpers.sdd_quality_defect_plan import scaffold_sdd_quality_defect_plan
```

在该注册表条目之后：
```python
    "scaffold_sdd_yagni_plan": scaffold_sdd_yagni_plan,
```
添加：
```python
    "scaffold_sdd_quality_defect_plan": scaffold_sdd_quality_defect_plan,
```

- [ ] **步骤 3：创建 `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/story.md`：**

```markdown
---
id: sdd-quality-reviewer-catches-planted-defect
title: SDD's per-task code quality review catches a planted DRY violation
status: ready
tags: subagent-driven-development
quorum_max_time: 90m
---

You have a small plan at docs/superpowers/plans/report-plan.md — two report
formatting functions. The plan's Task 2 implementation snippet duplicates
Task 1's formatting logic verbatim instead of sharing it. The duplication is
spec-compliant (the requirements only describe behavior), so the spec
compliance reviewer should pass it — the per-task code quality reviewer is
the gate under test. You are spec-aware — name the skill.

When the agent is ready for input, tell it to execute the plan with SDD. Use
phrasing like:

"I have a small plan at docs/superpowers/plans/report-plan.md — two report
formatting functions. Use the superpowers:subagent-driven-development skill
to execute it end-to-end — dispatch fresh subagents per task and run the
two-stage review after each."

Let the agent proceed autonomously. If it asks clarifying questions, give
brief answers. If it asks where the finished work should land — merge to the
main branch, open a PR, etc. — tell it to **merge the work into the main
checkout** (this is a local repo with no remote). If a quality reviewer
flags the duplicated formatting logic and an implementer refactors it, let
the review-fix cycle play out — that cycle is exactly the behavior under
test.

The deliverable must end up in the checkout you launched in (the main
working tree). If the agent did its work on a branch or in a worktree, it
is not done until it has merged/finished that work back into the main
checkout. Once the agent reports the plan is complete (both functions
implemented, tests passing) AND the code is present on the main checkout,
you are done.

## Acceptance Criteria

- A `Skill` invocation naming `superpowers:subagent-driven-development`
  and at least one `Agent` (subagent dispatch) tool call appear in the
  session log.
- The duplicated report-formatting logic did not survive to the end of
  the run. Either (a) the implementer never introduced the duplication
  (wrote or self-reviewed its way to shared logic), or (b) the per-task
  code quality reviewer flagged the duplication as an issue and a
  review-fix loop removed it. A fail looks like the duplicated logic
  shipping with the per-task quality reviewer approving it, or the
  duplication being caught only by the final whole-branch review.
- The per-task quality reviewers stayed task-scoped: no package-wide
  test suites, race detector runs, or repeated/high-count test loops
  appear in reviewer subagent activity, and reviewers did not re-run
  the full test suite merely to confirm the implementer's report.
- `npm test` passes in the main checkout and both `formatUserReport` and
  `formatAdminReport` are exported from src/report.js. The deterministic
  assertions gate this; the criteria above are about whether the
  *per-task quality review* was the mechanism that kept the code clean.
```

- [ ] **步骤 4：创建 `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`：**

```bash
#!/usr/bin/env bash
set -euo pipefail
uv run setup-helpers run scaffold_sdd_quality_defect_plan
```

然后：`chmod +x evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`

- [ ] **步骤 5：创建 `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/checks.sh`**（无可执行位）：

```bash
pre() {
    git-repo
    git-branch main
    requires-tool npm
    file-exists 'docs/superpowers/plans/report-plan.md'
    file-contains 'docs/superpowers/plans/report-plan.md' 'formatAdminReport'
    file-contains 'docs/superpowers/plans/report-plan.md' 'repeat\(40\)'
}

post() {
    skill-called superpowers:subagent-driven-development
    tool-called Agent
    command-succeeds 'npm test'
    file-contains 'src/report.js' 'export function formatUserReport'
    file-contains 'src/report.js' 'export function formatAdminReport'
    command-succeeds 'test "$(grep -c "repeat(40)" src/report.js)" -le 1'
}
```

（最后一条检查是确定性的 DRY 关卡：banner 构造
`"=".repeat(40)` 在最终文件中至多出现一次 —— 共享，而非
按函数复制。）

- [ ] **步骤 6：在 evals 仓库中验证和测试**

```bash
cd evals
uv run quorum check
uv run ruff check
uv run pytest -x -q
```

预期：全部通过；`quorum check` 列出新场景且无错误。

- [ ] **步骤 7：提交（在子模块中）**

```bash
cd evals
git add setup_helpers/sdd_quality_defect_plan.py setup_helpers/__init__.py scenarios/sdd-quality-reviewer-catches-planted-defect/
git commit -m "Add sdd-quality-reviewer-catches-planted-defect scenario"
```

---

### 任务 6：静态验证扫描 (Task 6: Static verification sweep)

**文件：** 不修改任何文件 —— 仅验证。

- [ ] **步骤 1：父仓库中无悬空引用**

运行：`grep -rn "requesting-code-review" skills/subagent-driven-development/`
预期：仅在 SKILL.md 中匹配（最终审查流程图节点 ×3、prompt 模板指针、Integration 条目）。code-quality-reviewer-prompt.md 中没有。

运行：`grep -rn "Ready to merge" skills/subagent-driven-development/ || echo CLEAN`
预期：`CLEAN`

- [ ] **步骤 2：plugin 基础设施测试**

运行：`bash tests/shell-lint/test-lint-shell.sh`
预期：全部 PASS（我们仅在 evals 子模块内部添加了 `setup.sh`，该子模块有自己的检查）。

- [ ] **步骤 3：跨平台工具表仍然自洽**

运行：`grep -n "code-quality-reviewer" skills/using-superpowers/references/antigravity-tools.md skills/using-superpowers/references/gemini-tools.md`
预期：两张表仍将 `code-quality-reviewer` 列为审查者模板（新 prompt 的 "If you cannot run commands in this environment, name the test you would run" 行保持了只读 `research` 映射的有效性 —— 无需编辑表格）。

---

### 任务 7：在线 before/after eval（由维护者把关） (Task 7: Live before/after evals (maintainer-gated))

在线 quorum 运行会以宽松模式启动 agent CLI —— **受信任维护者操作；由 Jesse 启动这些**，遵循 `evals/CLAUDE.md`。需要 `ANTHROPIC_API_KEY`。

- [ ] **步骤 1：基线（skills 如 dev 上发布的状态）** —— 从主检出（`/Users/jesse/git/superpowers/superpowers`，在 dev 上）进行，或任意没有本分支变更的检出：

```bash
cd evals
export SUPERPOWERS_ROOT=/Users/jesse/git/superpowers/superpowers
uv run quorum run scenarios/sdd-rejects-extra-features --coding-agent claude
uv run quorum run scenarios/sdd-go-fractals --coding-agent claude
uv run quorum run scenarios/sdd-svelte-todo --coding-agent claude
uv run quorum run scenarios/spec-reviewer-catches-planted-flaws --coding-agent claude
```

- [ ] **步骤 2：之后（本分支的 skills）** —— 将 `SUPERPOWERS_ROOT` 指向本 worktree：

```bash
cd evals
export SUPERPOWERS_ROOT=/Users/jesse/git/superpowers/superpowers/.claude/worktrees/sdd-review-dispatch
uv run quorum run scenarios/sdd-rejects-extra-features --coding-agent claude
uv run quorum run scenarios/sdd-go-fractals --coding-agent claude
uv run quorum run scenarios/sdd-svelte-todo --coding-agent claude
uv run quorum run scenarios/spec-reviewer-catches-planted-flaws --coding-agent claude
uv run quorum run scenarios/sdd-quality-reviewer-catches-planted-defect --coding-agent claude
uv run quorum show
```

- [ ] **步骤 3：对比**

通过门槛：变更之后全部四个既有场景仍然通过（捕获率无回归）；新植入缺陷场景通过。对于探索成本，对比 before/after 运行记录之间审查者 subagent 的工具调用次数（无自动化检查 —— 规范将其标为已知缺口）。

---

## 收尾 (Finishing)

所有任务通过后：evals 子模块提交需要进入 `superpowers-evals`（向其 `main` 发 PR），然后本分支更新 `evals` 子模块指针 —— 按 `evals/CLAUDE.md`，父仓库的更新是传播的一部分，不是可选的。然后使用 superpowers:finishing-a-development-branch。针对 superpowers 的 PR 以 `dev` 为目标。
