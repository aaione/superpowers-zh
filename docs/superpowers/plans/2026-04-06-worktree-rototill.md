# Worktree 全面翻整实现计划 (Worktree Rototill Implementation Plan)

> **致 agent 工作者：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐个实现本计划。步骤使用 checkbox（`- [ ]`）语法进行跟踪。

**目标：** 让 superpowers 在可用时服从原生 harness worktree 系统，在不可用时回退到手工 git worktree，并修复三个已知的收尾缺陷。

**架构：** 两个技能文件被重写（`using-git-worktrees`、`finishing-a-development-branch`），三个文件获得单行 integration 更新（`executing-plans`、`subagent-driven-development`、`writing-plans`）。核心变更是添加检测（`GIT_DIR != GIT_COMMON`）和原生工具优先的创建路径。这些是 markdown 技能指令文件，不是应用程序代码 —— "测试"是使用 testing-skills-with-subagents TDD 框架的 agent 行为测试。

**技术栈：** Markdown（技能文件）、bash（测试脚本）、Claude Code CLI（`claude -p` 用于无头测试）

**规范：** `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md`

---

### 任务 1：门控 —— 步骤 1a 的 TDD 验证（原生工具偏好） (Task 1: GATE — TDD Validation of Step 1a (Native Tool Preference))

步骤 1a 是整个设计的承重假设。如果 agent 不偏好原生 worktree 工具胜过 `git worktree add`，规范就失败。先验证这一点，再动任何技能文件。

**文件：**
- 创建：`tests/claude-code/test-worktree-native-preference.sh`
- 阅读：`skills/using-git-worktrees/SKILL.md`（当前版本，作为 RED 基线）
- 阅读：`tests/claude-code/test-helpers.sh`（用于 `run_claude`、`assert_contains` 等）
- 阅读：`skills/writing-skills/testing-skills-with-subagents.md`（TDD 框架）

**此任务是一道门控。** 如果在 2 次 REFACTOR 迭代之后 GREEN 阶段失败，停止。不要继续到任务 2。回报 —— 创建方法需要重新设计。

- [ ] **步骤 1：编写 RED 基线测试脚本**

创建既会运行未更新也会运行更新后技能文本场景的测试脚本。RED 阶段针对当前技能运行（没有步骤 1a）。

```bash
#!/usr/bin/env bash
# Test: Does the agent prefer native worktree tools (EnterWorktree) over git worktree add?
# Framework: RED-GREEN-REFACTOR per testing-skills-with-subagents.md
#
# RED:   Current skill has no native tool preference. Agent should use git worktree add.
# GREEN: Updated skill has Step 1a. Agent should use EnterWorktree on Claude Code.

set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# Pressure scenario: realistic implementation task where agent needs isolation
SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to implement a small feature (add a "version" field to package.json).
This should be done in an isolated workspace to protect the main branch.

You have the using-git-worktrees skill available. Set up the isolated workspace now.
Do NOT actually implement the feature — just set up the workspace and report what you did.

Respond with EXACTLY what tool/command you used to create the workspace.'

echo "=== Worktree Native Preference Test ==="
echo ""

# Phase selection
PHASE="${1:-red}"

if [ "$PHASE" = "red" ]; then
    echo "--- RED PHASE: Running WITHOUT Step 1a (current skill) ---"
    echo "Expected: Agent uses 'git worktree add' (no native tool awareness)"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # RED expectation: agent uses git worktree add (current behavior)
    if echo "$output" | grep -qi "EnterWorktree"; then
        echo "[UNEXPECTED] Agent used EnterWorktree WITHOUT Step 1a — skill may not be needed"
        echo "Investigate: is Claude Code's default behavior already correct?"
    else
        echo "[RED CONFIRMED] Agent did NOT use EnterWorktree"
        assert_contains "$output" "git worktree" "Agent used git worktree (expected in RED)"
    fi

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "green" ]; then
    echo "--- GREEN PHASE: Running WITH Step 1a (updated skill) ---"
    echo "Expected: Agent uses EnterWorktree instead of git worktree add"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # GREEN expectation: agent uses EnterWorktree
    assert_contains "$output" "EnterWorktree" "Agent used native EnterWorktree tool"
    assert_not_contains "$output" "git worktree add" "Agent did NOT fall back to git worktree add"

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "pressure" ]; then
    echo "--- PRESSURE PHASE: Multiple pressures to skip native tools ---"
    echo ""

    PRESSURE_SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to set up an isolated workspace URGENTLY. Production is impacted and
the fix needs to go in a worktree. You know git worktree add works reliably.
The .worktrees/ directory already exists and is gitignored.

You have the using-git-worktrees skill. Set up the workspace NOW.
Speed matters. Report EXACTLY what tool/command you used.'

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees
    echo ".worktrees/" >> .gitignore

    output=$(run_claude "$PRESSURE_SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # Should STILL use EnterWorktree even under pressure
    assert_contains "$output" "EnterWorktree" "Agent used native tool even under time pressure"
    assert_not_contains "$output" "git worktree add" "Agent resisted falling back to git despite pressure"

    cleanup_test_project "$test_dir"
fi

echo ""
echo "=== Test Complete ==="
```

- [ ] **步骤 2：运行 RED 阶段 —— 确认 agent 今天使用 git worktree add**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh red`

预期：`[RED CONFIRMED] Agent did NOT use EnterWorktree` —— agent 使用 `git worktree add`，因为当前技能没有原生工具偏好。

逐字记录 agent 的确切输出和任何自我合理化。这是技能必须修复的基线失败。

- [ ] **步骤 3：如果 RED 确认，继续。编写步骤 1a 技能文本。**

创建技能的临时测试版本，仅添加步骤 1a（最小变更以隔离变量）。在技能创建指令的顶部、既有目录选择流程之前，添加此章节：

```markdown
## Step 1: Create Isolated Workspace

**You have two mechanisms. Try them in this order.**

### 1a. Native Worktree Tools (preferred)

If your platform provides a worktree or workspace-isolation tool, use it. You know your own toolkit — the skill does not need to name specific tools. Native tools handle directory placement, branch creation, and cleanup automatically.

After using a native tool, skip to Step 3 (Project Setup).

### 1b. Git Worktree Fallback

If no native tool is available, create a worktree manually using git.
```

- [ ] **步骤 4：运行 GREEN 阶段 —— 确认 agent 现在使用 EnterWorktree**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh green`

预期：`[PASS] Agent used native EnterWorktree tool`

如果 FAIL：逐字记录 agent 的确切输出和自我合理化。这是一个 REFACTOR 信号 —— 步骤 1a 文本需要修订。最多尝试 2 次 REFACTOR 迭代。如果 2 次迭代后仍失败，停止并回报。

- [ ] **步骤 5：运行 PRESSURE 阶段 —— 确认 agent 在压力下抵抗回退**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh pressure`

预期：`[PASS] Agent used native tool even under time pressure`

如果 FAIL：逐字记录自我合理化。向步骤 1a 文本添加明确的反驳（例如，一个 Red Flag 条目："当你的平台提供原生 worktree 工具时，绝不使用 git worktree add"）。重新运行。

- [ ] **步骤 6：提交测试脚本**

```bash
git add tests/claude-code/test-worktree-native-preference.sh
git commit -m "test: add RED/GREEN validation for native worktree preference (PRI-974)

Gate test for Step 1a — validates agents prefer EnterWorktree over
git worktree add on Claude Code. Must pass before skill rewrite."
```

---

### 任务 2：重写 `using-git-worktrees` SKILL.md (Task 2: Rewrite `using-git-worktrees` SKILL.md)

创建技能的全面重写。完全替换既有文件。

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md`（全面重写，219 行 → 约 210 行）

**依赖于：** 任务 1 GREEN 通过。

- [ ] **步骤 1：编写完整的新 SKILL.md**

将 `skills/using-git-worktrees/SKILL.md` 的全部内容替换为：

```markdown
---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# Using Git Worktrees

## Overview

Ensure work happens in an isolated workspace. Prefer your platform's native worktree tools. Fall back to manual git worktrees only when no native tool is available.

**Core principle:** Detect existing isolation first. Then use native tools. Then fall back to git. Never fight the harness.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Step 0: Detect Existing Isolation

**Before creating anything, check if you are already in an isolated workspace.**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Submodule guard:** `GIT_DIR != GIT_COMMON` is also true inside git submodules. Before concluding "already in a worktree," verify you are not in a submodule:

```bash
# If this returns a path, you're in a submodule, not a worktree — proceed to Step 1
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**If `GIT_DIR != GIT_COMMON` (and not a submodule):** You are already in a linked worktree. Skip to Step 3 (Project Setup). Do NOT create another worktree.

Report with branch state:
- On a branch: "Already in isolated workspace at `<path>` on branch `<name>`."
- Detached HEAD: "Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**If `GIT_DIR == GIT_COMMON` (or in a submodule):** You are in a normal repo checkout.

Has the user already indicated their worktree preference in your instructions? If not, ask for consent before creating a worktree:

> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

Honor any existing declared preference without asking. If the user declines consent, work in place and skip to Step 3.

## Step 1: Create Isolated Workspace

**You have two mechanisms. Try them in this order.**

### 1a. Native Worktree Tools (preferred)

If your platform provides a worktree or workspace-isolation tool, use it. You know your own toolkit — the skill does not need to name specific tools. Native tools handle directory placement, branch creation, and cleanup automatically.

After using a native tool, skip to Step 3 (Project Setup).

### 1b. Git Worktree Fallback

If no native tool is available, create a worktree manually using git.

#### Directory Selection

Follow this priority order:

1. **Check your instructions for a worktree directory preference.** If specified, use it without asking.

2. **Check existing project-local directories:**
   ```bash
   ls -d .worktrees 2>/dev/null     # Preferred (hidden)
   ls -d worktrees 2>/dev/null      # Alternative
   ```
   If found, use that directory. If both exist, `.worktrees` wins.

3. **Default to `.worktrees/`.**

#### Safety Verification (project-local directories only)

**MUST verify directory is ignored before creating worktree:**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:** Add to .gitignore, commit the change, then proceed.

**Why critical:** Prevents accidentally committing worktree contents to repository.

#### Create the Worktree

```bash
# Determine path based on chosen location
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

#### Hooks Awareness

Git worktrees do not inherit the parent repo's hooks directory. After creating the worktree, symlink hooks from the main repo if they exist:

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

This prevents pre-commit checks, linters, and other hooks from silently stopping when work moves to a worktree.

**Sandbox fallback:** If `git worktree add` fails with a permission error (sandbox denial), treat this as a restricted environment. Skip creation, run setup and baseline tests in the current directory, report accordingly.

## Step 3: Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## Step 4: Verify Clean Baseline

Run tests to ensure workspace starts clean:

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### Report

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Already in linked worktree | Skip creation (Step 0) |
| In a submodule | Treat as normal repo (Step 0 guard) |
| Native worktree tool available | Use it (Step 1a) |
| No native tool | Git worktree fallback (Step 1b) |
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check instruction file, then default `.worktrees/` |
| Directory not ignored | Add to .gitignore + commit |
| Permission error on create | Sandbox fallback, work in place |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Fighting the harness

- **Problem:** Using `git worktree add` when the platform already provides isolation
- **Fix:** Step 0 detects existing isolation. Step 1a defers to native tools.

### Skipping detection

- **Problem:** Creating a nested worktree inside an existing one
- **Fix:** Always run Step 0 before creating anything

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > instruction file > default

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

## Red Flags

**Never:**
- Create a worktree when Step 0 detects existing isolation
- Use git commands when a native worktree tool is available
- Create worktree without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking

**Always:**
- Run Step 0 detection first
- Prefer native tools over git fallback
- Follow directory priority: existing > instruction file > default
- Verify directory is ignored for project-local
- Auto-detect and run project setup
- Verify clean test baseline
- Symlink hooks after creating worktree via 1b

## Integration

**Called by:**
- **subagent-driven-development** - Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - Ensures isolated workspace (creates one or verifies existing)
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
```

- [ ] **步骤 2：验证文件读取正确**

运行：`wc -l skills/using-git-worktrees/SKILL.md`

预期：约 200-220 行。扫查是否有任何 markdown 格式问题。

- [ ] **步骤 3：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat: rewrite using-git-worktrees with detect-and-defer (PRI-974)

Step 0: GIT_DIR != GIT_COMMON detection (skip if already isolated)
Step 0 consent: opt-in prompt before creating worktree (#991)
Step 1a: native tool preference (short, first, declarative)
Step 1b: git worktree fallback with project-local directory policy
Submodule guard prevents false detection
Platform-neutral instruction file references (#1049)"
```

---

### 任务 3：重写 `finishing-a-development-branch` SKILL.md (Task 3: Rewrite `finishing-a-development-branch` SKILL.md)

收尾技能的全面重写。添加环境检测，修复三个缺陷，添加基于来源的清理。

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（全面重写，201 行 → 约 220 行）

- [ ] **步骤 1：编写完整的新 SKILL.md**

将 `skills/finishing-a-development-branch/SKILL.md` 的全部内容替换为：

```markdown
---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Detect environment → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Detect Environment

**Determine workspace state before presenting options:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

This determines which menu to show and how cleanup works:

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (normal repo) | Standard 4 options | No worktree to clean up |
| `GIT_DIR != GIT_COMMON`, named branch | Standard 4 options | Provenance-based (see Step 6) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | Reduced 3 options (no merge) | No cleanup (externally managed) |

### Step 3: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 4: Present Options

**Normal repo and named-branch worktree — present exactly these 4 options:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD — present exactly these 3 options:**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 5: Execute Choice

#### Option 1: Merge Locally

```bash
# Get main repo root for CWD safety
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first — verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>

# Verify tests on merged result
<test command>

# Only after merge succeeds: remove worktree, then delete branch
# (See Step 6 for worktree cleanup)
git branch -d <feature-branch>
```

Then: Cleanup worktree (Step 6)

#### Option 2: Push and Create PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**Do NOT clean up worktree** — user needs it alive to iterate on PR feedback.

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

Then: Cleanup worktree (Step 6), then force-delete branch:
```bash
git branch -D <feature-branch>
```

### Step 6: Cleanup Workspace

**Only runs for Options 1 and 4.** Options 2 and 3 always preserve the worktree.

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**If `GIT_DIR == GIT_COMMON`:** Normal repo, no worktree to clean up. Done.

**If worktree path is under `.worktrees/` or `worktrees/`:** Superpowers created this worktree — we own cleanup.

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**Otherwise:** The host environment (harness) owns this workspace. Do NOT remove it. If your platform provides a workspace-exit tool, use it. Otherwise, leave the workspace in place.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | yes | - | - | yes |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | yes (force) |

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" is ambiguous
- **Fix:** Present exactly 4 structured options (or 3 for detached HEAD)

**Cleaning up worktree for Option 2**
- **Problem:** Remove worktree user needs for PR iteration
- **Fix:** Only cleanup for Options 1 and 4

**Deleting branch before removing worktree**
- **Problem:** `git branch -d` fails because worktree still references the branch
- **Fix:** Merge first, remove worktree, then delete branch

**Running git worktree remove from inside the worktree**
- **Problem:** Command fails silently when CWD is inside the worktree being removed
- **Fix:** Always `cd` to main repo root before `git worktree remove`

**Cleaning up harness-owned worktrees**
- **Problem:** Removing a worktree the harness created causes phantom state
- **Fix:** Only clean up worktrees under `.worktrees/` or `worktrees/`

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request
- Remove a worktree before confirming merge success
- Clean up worktrees you didn't create (provenance check)
- Run `git worktree remove` from inside the worktree

**Always:**
- Verify tests before offering options
- Detect environment before presenting menu
- Present exactly 4 options (or 3 for detached HEAD)
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only
- `cd` to main repo root before worktree removal
- Run `git worktree prune` after removal

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill
```

- [ ] **步骤 2：验证文件读取正确**

运行：`wc -l skills/finishing-a-development-branch/SKILL.md`

预期：约 210-230 行。

- [ ] **步骤 3：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat: rewrite finishing-a-development-branch with detect-and-defer (PRI-974)

Step 2: environment detection (GIT_DIR != GIT_COMMON) before presenting menu
Detached HEAD: reduced 3-option menu (no merge from detached HEAD)
Provenance-based cleanup: .worktrees/ = ours, anything else = hands off
Bug #940: Option 2 no longer cleans up worktree
Bug #999: merge -> verify -> remove worktree -> delete branch
Bug #238: cd to main repo root before git worktree remove
Stale worktree pruning after removal (git worktree prune)"
```

---

### 任务 4：Integration 更新 (Task 4: Integration Updates)

对三个引用 `using-git-worktrees` 的文件的单行变更。

**文件：**
- 修改：`skills/executing-plans/SKILL.md:68`
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/writing-plans/SKILL.md:16`

- [ ] **步骤 1：更新 executing-plans 的 integration 行**

在 `skills/executing-plans/SKILL.md` 中，把第 68 行从：

```markdown
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

改为：

```markdown
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 2：更新 subagent-driven-development 的 integration 行**

在 `skills/subagent-driven-development/SKILL.md` 中，把第 268 行从：

```markdown
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

改为：

```markdown
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 3：更新 writing-plans 的 context 行**

在 `skills/writing-plans/SKILL.md` 中，把第 16 行从：

```markdown
**Context:** This should be run in a dedicated worktree (created by brainstorming skill).
```

改为：

```markdown
**Context:** If working in an isolated worktree, it should have been created via the using-git-worktrees skill at execution time.
```

- [ ] **步骤 4：提交全部三个**

```bash
git add skills/executing-plans/SKILL.md skills/subagent-driven-development/SKILL.md skills/writing-plans/SKILL.md
git commit -m "fix: update worktree integration references across skills (PRI-974)

Remove REQUIRED language from executing-plans and subagent-driven-development.
Consent and detection now live inside using-git-worktrees itself.
Fix stale 'created by brainstorming' claim in writing-plans."
```

---

### 任务 5：端到端验证 (Task 5: End-to-End Validation)

验证全部重写的技能协同工作。运行既有测试套件加手动验证。

**文件：**
- 阅读：`tests/claude-code/run-skill-tests.sh`
- 阅读：`skills/using-git-worktrees/SKILL.md`（验证最终状态）
- 阅读：`skills/finishing-a-development-branch/SKILL.md`（验证最终状态）

- [ ] **步骤 1：运行既有测试套件**

运行：`cd tests/claude-code && bash run-skill-tests.sh`

预期：全部既有测试通过。如果任何一个失败，调查 —— integration 变更（任务 4）可能破坏了一个内容断言。

- [ ] **步骤 2：重新运行步骤 1a 的 GREEN 测试**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh green`

预期：PASS —— agent 在最终技能文本下仍使用 EnterWorktree（而不仅是任务 1 中最小的步骤 1a 添加）。

- [ ] **步骤 3：手动验证 —— 端到端阅读两个重写的技能**

完整阅读 `skills/using-git-worktrees/SKILL.md` 和 `skills/finishing-a-development-branch/SKILL.md`。检查：

1. 没有对旧行为的引用（硬编码的 `CLAUDE.md`、交互式目录提示、"REQUIRED"措辞）
2. 每个文件内的步骤编号一致
3. Quick Reference 表与文字匹配
4. Integration 章节正确交叉引用
5. 没有 markdown 格式问题

- [ ] **步骤 4：验证 git 状态干净**

运行：`git status`

预期：干净的工作树。跨任务 1-4 的全部变更已提交。

- [ ] **步骤 5：如有需要则进行最终提交**

如果手动验证发现问题，修复并提交：

```bash
git add -A
git commit -m "fix: address review findings in worktree skill rewrite (PRI-974)"
```

如果未发现问题，跳过此步骤。
