# Codex App 兼容性实施计划

> **面向 agentic worker：** 必须使用的子 Skill：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务执行此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 使 `using-git-worktrees`、`finishing-a-development-branch` 及相关 skill 在 Codex App 的沙箱 worktree 环境中工作，不破坏现有行为。

**架构：** 在两个 skill 的开始处添加只读环境检测（`git-dir` vs `git-common-dir`）。如果已在 linked worktree 中，跳过创建。如果在 detached HEAD 上，发出交接 payload 而非 4 选项菜单。沙箱回退捕获 worktree 创建期间的权限错误。

**技术栈：** Git、Markdown（skill 文件是指令文档，不是可执行代码）

**规格文档：** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 文件结构

| 文件 | 职责 | 操作 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | Worktree 创建 + 隔离 | 添加 Step 0 检测 + 沙箱回退 |
| `skills/finishing-a-development-branch/SKILL.md` | 分支完成工作流 | 添加 Step 1.5 检测 + 清理守卫 |
| `skills/subagent-driven-development/SKILL.md` | 使用 subagent 执行计划 | 更新 Integration 描述 |
| `skills/executing-plans/SKILL.md` | 内联执行计划 | 更新 Integration 描述 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 平台参考 | 添加检测 + 完成文档 |

---

### Task 1：为 `using-git-worktrees` 添加 Step 0

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:14-15`（在 Overview 之后、Directory Selection Process 之前插入）

- [ ] **Step 1：读取当前 skill 文件**

读取 `skills/using-git-worktrees/SKILL.md` 全文。确认确切的插入位置：在"Announce at start"行（第 14 行）之后，"## Directory Selection Process"（第 16 行）之前。

- [ ] **Step 2：插入 Step 0 部分**

在 Overview 部分和"## Directory Selection Process"之间插入以下内容：

```markdown
## Step 0: 检查是否已在隔离工作区中

在创建 worktree 之前，检查是否已存在一个：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**如果 `GIT_DIR` 与 `GIT_COMMON` 不同：** 你已经在 linked worktree 中（由 Codex App、Claude Code 的 Agent tool、之前的 skill 运行或用户创建）。不要创建另一个 worktree。而是：

1. 运行项目设置（按下面"Run Project Setup"中的说明自动检测包管理器）
2. 验证干净基线（按"Verify Clean Baseline"中的说明运行测试）
3. 报告分支状态：
   - 在分支上："Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD："Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

报告后，停止。不要继续到 Directory Selection 或 Creation Steps。

**如果 `GIT_DIR` 等于 `GIT_COMMON`：** 继续下面的完整 worktree 创建流程。

**沙箱回退：** 如果你继续到 Creation Steps 但 `git worktree add -b` 因权限错误失败（例如"Operation not permitted"），将此视为延迟检测到的受限环境。回退到上述行为 -- 在当前目录中运行设置和基线测试，相应报告，然后停止。
```

- [ ] **Step 3：验证插入**

再次读取文件。确认：
- Step 0 出现在 Overview 和 Directory Selection Process 之间
- 文件的其余部分（Directory Selection、Safety Verification、Creation Steps 等）未改变
- 没有重复部分或损坏的 markdown

- [ ] **Step 4：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection (PRI-823)

Skip worktree creation when already in a linked worktree. Includes
sandbox fallback for permission errors on git worktree add."
```

---

### Task 2：更新 `using-git-worktrees` Integration 部分

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:211-215`（Integration > Called by）

- [ ] **Step 1：更新三个"Called by"条目**

将第 212-214 行从：

```markdown
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

改为：

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2：验证 Integration 部分**

读取 Integration 部分。确认所有三个条目已更新，"Pairs with"未改变。

- [ ] **Step 3：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions (PRI-823)

Clarify that skill ensures a workspace exists, not that it always creates one."
```

---

### Task 3：为 `finishing-a-development-branch` 添加 Step 1.5

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md:38`（在 Step 1 之后、Step 2 之前插入）

- [ ] **Step 1：读取当前 skill 文件**

读取 `skills/finishing-a-development-branch/SKILL.md` 全文。确认插入位置：在"**If tests pass:** Continue to Step 2."（第 38 行）之后，"### Step 2: Determine Base Branch"（第 40 行）之前。

- [ ] **Step 2：插入 Step 1.5 部分**

在 Step 1 和 Step 2 之间插入以下内容：

```markdown
### Step 1.5: 检测环境

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**路径 A -- `GIT_DIR` 与 `GIT_COMMON` 不同 且 `BRANCH` 为空（外部管理的 worktree，detached HEAD）：**

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。

然后向用户展示以下内容（不要展示 4 选项菜单）：

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

分支名称：如有 ticket ID 则使用（例如 `pri-823/codex-compat`），否则将 plan 标题的前 5 个词 slugify，否则省略。避免在分支名称中使用敏感内容。

跳到 Step 5（清理是空操作 -- 见下方守卫）。

**路径 B -- `GIT_DIR` 与 `GIT_COMMON` 不同 且 `BRANCH` 存在（外部管理的 worktree，命名分支）：**

继续到 Step 2 并正常展示 4 选项菜单。

**路径 C -- `GIT_DIR` 等于 `GIT_COMMON`（正常环境）：**

继续到 Step 2 并正常展示 4 选项菜单。
```

- [ ] **Step 3：验证插入**

再次读取文件。确认：
- Step 1.5 出现在 Step 1 和 Step 2 之间
- Steps 2-5 未改变
- 路径 A 交接包含 commit SHA 和数据丢失警告
- 路径 B 和 C 正常继续到 Step 2

- [ ] **Step 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection (PRI-823)

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu. Includes commit SHA and data loss warning."
```

---

### Task 4：为 `finishing-a-development-branch` 添加 Step 5 清理守卫

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（Step 5: Cleanup Worktree -- 按章节标题查找，Task 3 插入后行号会偏移）

- [ ] **Step 1：读取当前 Step 5 部分**

在 `skills/finishing-a-development-branch/SKILL.md` 中找到"### Step 5: Cleanup Worktree"部分（Task 3 插入后行号已偏移）。当前 Step 5 为：

```markdown
### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

- [ ] **Step 2：在现有逻辑之前添加清理守卫**

将 Step 5 部分替换为：

```markdown
### Step 5: Cleanup Worktree

**首先，检查 worktree 是否为外部管理：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

如果 `GIT_DIR` 与 `GIT_COMMON` 不同：跳过 worktree 移除 -- 宿主环境拥有此工作区。

**否则，对于 Option 1 和 4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于 Option 3：** 保留 worktree。
```

注意：原文说"For Options 1, 2, 4"但 Quick Reference 表和 Common Mistakes 部分说"Options 1 & 4 only."此编辑使 Step 5 与这些部分一致。

- [ ] **Step 3：验证替换**

读取 Step 5。确认：
- 清理守卫（重新检测）首先出现
- 现有移除逻辑为非外部管理的 worktree 保留
- "Options 1 and 4"（不是"1, 2, 4"）与 Quick Reference 和 Common Mistakes 一致

- [ ] **Step 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard (PRI-823)

Re-detect externally managed worktree at cleanup time and skip removal.
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only, matching Quick Reference and Common Mistakes."
```

---

### Task 5：更新 `subagent-driven-development` 和 `executing-plans` 中的 Integration 行

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/executing-plans/SKILL.md:68`

- [ ] **Step 1：更新 `subagent-driven-development`**

将第 268 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2：更新 `executing-plans`**

将第 68 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 3：验证两个文件**

读取 `skills/subagent-driven-development/SKILL.md` 的第 268 行和 `skills/executing-plans/SKILL.md` 的第 68 行。确认两者都说"Ensures isolated workspace (creates one or verifies existing)"。

- [ ] **Step 4：提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): update worktree Integration descriptions (PRI-823)

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one."
```

---

### Task 6：向 `codex-tools.md` 添加环境检测文档

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md:25`（在末尾追加）

- [ ] **Step 1：读取当前文件**

读取 `skills/using-superpowers/references/codex-tools.md` 全文。确认它在 multi_agent 部分之后的第 25-26 行结束。

- [ ] **Step 2：追加两个新部分**

在文件末尾添加：

```markdown

## 环境检测

创建 worktree 或完成分支的 skill 应在继续之前使用只读 git 命令检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在 linked worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法在沙箱中 branch/push/PR）

参见 `using-git-worktrees` Step 0 和 `finishing-a-development-branch`
Step 1.5 了解每个 skill 如何使用这些信号。

## Codex App 完成

当沙箱阻止 branch/push 操作（外部管理的 worktree 中的 detached HEAD）时，agent 提交所有工作并通知用户使用 App 的原生控件：

- **"Create branch"** -- 命名分支，然后通过 App UI 提交/push/PR
- **"Hand off to local"** -- 将工作转移到用户的本地检出

Agent 仍可运行测试、暂存文件，并为用户输出建议的分支名称、提交消息和 PR 描述供复制。
```

- [ ] **Step 3：验证添加内容**

读取完整文件。确认：
- 两个新部分出现在现有内容之后
- Bash 代码块正确渲染（未转义）
- 存在对 Step 0 和 Step 1.5 的交叉引用

- [ ] **Step 4：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs (PRI-823)

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt."
```

---

### Task 7：自动化测试 -- 环境检测

**文件：**
- 创建：`tests/codex-app-compat/test-environment-detection.sh`

- [ ] **Step 1：创建测试目录**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **Step 2：编写检测测试脚本**

创建 `tests/codex-app-compat/test-environment-detection.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 测试 PRI-823 的环境检测逻辑
# 测试 using-git-worktrees Step 0 和 finishing-a-development-branch Step 1.5 使用的
# git-dir vs git-common-dir 比较

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# 辅助函数：运行检测并返回 "linked" 或 "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== 测试 1：普通仓库检测 ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "普通仓库检测为 normal"
else
  log_fail "普通仓库检测为 '$result'（预期 'normal'）"
fi

echo "=== 测试 2：Linked worktree 检测 ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree 检测为 linked"
else
  log_fail "Linked worktree 检测为 '$result'（预期 'linked'）"
fi

echo "=== 测试 3：Detached HEAD 检测 ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch 为空"
else
  log_fail "Detached HEAD: branch 为 '$branch'（预期空）"
fi

echo "=== 测试 4：Linked worktree + detached HEAD（Codex App 模拟）==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App 模拟: linked + detached HEAD"
else
  log_fail "Codex App 模拟: result='$result', branch='$branch'"
fi

echo "=== 测试 5：清理守卫 -- linked worktree 不应移除 ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "清理守卫: linked worktree 正确检测（将跳过移除）"
else
  log_fail "清理守卫: 预期 'linked'，得到 '$result'"
fi

echo "=== 测试 6：清理守卫 -- 主仓库应移除 ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "清理守卫: 主仓库正确检测（将继续移除）"
else
  log_fail "清理守卫: 预期 'normal'，得到 '$result'"
fi

# 临时目录删除前清理 worktree
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== 结果: $PASS 通过, $FAIL 失败 ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **Step 3：设置可执行权限并运行**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

预期输出：6 通过，0 失败。

- [ ] **Step 4：提交**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat (PRI-823)

Tests git-dir vs git-common-dir comparison in normal repo, linked
worktree, detached HEAD, and cleanup guard scenarios."
```

---

### Task 8：最终验证

**文件：**
- 读取：所有 5 个修改的 skill 文件

- [ ] **Step 1：运行自动化检测测试**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

预期：6 通过，0 失败。

- [ ] **Step 2：读取每个修改的文件并验证更改**

从头到尾读取每个文件：
- `skills/using-git-worktrees/SKILL.md` -- Step 0 存在，其余未改变
- `skills/finishing-a-development-branch/SKILL.md` -- Step 1.5 存在，清理守卫存在，其余未改变
- `skills/subagent-driven-development/SKILL.md` -- 第 268 行已更新
- `skills/executing-plans/SKILL.md` -- 第 68 行已更新
- `skills/using-superpowers/references/codex-tools.md` -- 末尾有两个新部分

- [ ] **Step 3：验证没有意外更改**

```bash
git diff --stat HEAD~7
```

应显示恰好 6 个文件更改（5 个 skill 文件 + 1 个测试文件）。没有其他文件被修改。

- [ ] **Step 4：运行现有测试套件**

如果有测试运行器：
```bash
# 运行 skill 触发测试
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "此环境中 Skill 触发测试不可用"

# 运行 SDD 集成测试
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "此环境中 SDD 集成测试不可用"
```

注意：这些测试需要带有 `--dangerously-skip-permissions` 的 Claude Code。如果不可用，记录应手动运行回归测试。
