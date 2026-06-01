# Worktree Rototill 实施计划

> **面向 agentic worker：** 必须使用的子 Skill：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务执行此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 使 superpowers 在有原生 harness worktree 系统时遵循它，在没有时回退到手动 git worktree，并修复三个已知的完成 bug。

**架构：** 两个 skill 文件被重写（`using-git-worktrees`、`finishing-a-development-branch`），三个文件各进行一行的集成更新（`executing-plans`、`subagent-driven-development`、`writing-plans`）。核心变更是添加检测（`GIT_DIR != GIT_COMMON`）和原生工具优先的创建路径。这些是 markdown skill 指令文件，不是应用代码 -- "测试"是使用 testing-skills-with-subagents TDD 框架的 agent 行为测试。

**技术栈：** Markdown（skill 文件）、bash（测试脚本）、Claude Code CLI（`claude -p` 用于无头测试）

**规格文档：** `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md`

---

### Task 1：门控 -- Step 1a（原生工具偏好）的 TDD 验证

Step 1a 是整个设计的承载假设。如果 agent 不偏好原生 worktree 工具而使用 `git worktree add`，规格就失败了。在触碰任何 skill 文件之前首先验证这一点。

**文件：**
- 创建：`tests/claude-code/test-worktree-native-preference.sh`
- 读取：`skills/using-git-worktrees/SKILL.md`（当前版本，用于 RED 基线）
- 读取：`tests/claude-code/test-helpers.sh`（用于 `run_claude`、`assert_contains` 等）
- 读取：`skills/writing-skills/testing-skills-with-subagents.md`（TDD 框架）

**此任务是门控。** 如果 GREEN 阶段在 2 次 REFACTOR 迭代后仍失败，停止。不要继续到 Task 2。报告回来 -- 创建方法需要重新设计。

- [ ] **Step 1：编写 RED 基线测试脚本**

创建测试脚本，在不含和含更新 skill 文本的情况下都运行场景。RED 阶段针对当前 skill 运行（没有 Step 1a）。

```bash
#!/usr/bin/env bash
# 测试：agent 是否偏好原生 worktree 工具（EnterWorktree）而非 git worktree add？
# 框架：RED-GREEN-REFACTOR 按照 testing-skills-with-subagents.md
#
# RED:   当前 skill 没有原生工具偏好。Agent 应使用 git worktree add。
# GREEN: 更新的 skill 有 Step 1a。Agent 应在 Claude Code 上使用 EnterWorktree。

set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# 压力场景：agent 需要隔离的现实实施任务
SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to implement a small feature (add a "version" field to package.json).
This should be done in an isolated workspace to protect the main branch.

You have the using-git-worktrees skill available. Set up the isolated workspace now.
Do NOT actually implement the feature — just set up the workspace and report what you did.

Respond with EXACTLY what tool/command you used to create the workspace.'

echo "=== Worktree Native Preference Test ==="
echo ""

# 阶段选择
PHASE="${1:-red}"

if [ "$PHASE" = "red" ]; then
    echo "--- RED 阶段：在不含 Step 1a 的情况下运行（当前 skill）---"
    echo "预期：Agent 使用 'git worktree add'（没有原生工具意识）"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent 输出："
    echo "$output"
    echo ""

    # RED 预期：agent 使用 git worktree add（当前行为）
    if echo "$output" | grep -qi "EnterWorktree"; then
        echo "[意外] Agent 在不含 Step 1a 的情况下使用了 EnterWorktree -- skill 可能不需要"
        echo "调查：Claude Code 的默认行为是否已经正确？"
    else
        echo "[RED 确认] Agent 没有使用 EnterWorktree"
        assert_contains "$output" "git worktree" "Agent 使用了 git worktree（RED 中预期）"
    fi

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "green" ]; then
    echo "--- GREEN 阶段：在含 Step 1a 的情况下运行（更新 skill）---"
    echo "预期：Agent 使用 EnterWorktree 而非 git worktree add"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent 输出："
    echo "$output"
    echo ""

    # GREEN 预期：agent 使用 EnterWorktree
    assert_contains "$output" "EnterWorktree" "Agent 使用了原生 EnterWorktree 工具"
    assert_not_contains "$output" "git worktree add" "Agent 没有回退到 git worktree add"

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "pressure" ]; then
    echo "--- PRESSURE 阶段：多种压力跳过原生工具 ---"
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

    echo "Agent 输出："
    echo "$output"
    echo ""

    # 即使在压力下也应使用 EnterWorktree
    assert_contains "$output" "EnterWorktree" "Agent 在时间压力下仍使用原生工具"
    assert_not_contains "$output" "git worktree add" "Agent 抵抗了压力下回退到 git 的诱惑"

    cleanup_test_project "$test_dir"
fi

echo ""
echo "=== 测试完成 ==="
```

- [ ] **Step 2：运行 RED 阶段 -- 确认 agent 当前使用 git worktree add**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh red`

预期：`[RED 确认] Agent 没有使用 EnterWorktree` -- agent 使用 `git worktree add`，因为当前 skill 没有原生工具偏好。

逐字记录 agent 的确切输出和任何合理化解释。这是 skill 必须修复的基线失败。

- [ ] **Step 3：如果 RED 确认，继续。编写 Step 1a skill 文本。**

创建 skill 的临时测试版本，仅添加 Step 1a（最小变更以隔离变量）。在 skill 创建指令的顶部，现有目录选择流程之前，添加此部分：

```markdown
## Step 1: Create Isolated Workspace

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

如果你的平台提供 worktree 或 workspace-isolation 工具，使用它。你知道自己的工具集 -- skill 不需要列出具体工具。原生工具自动处理目录放置、分支创建和清理。

使用原生工具后，跳到 Step 3（Project Setup）。

### 1b. Git Worktree 回退

如果没有原生工具可用，使用 git 手动创建 worktree。
```

- [ ] **Step 4：运行 GREEN 阶段 -- 确认 agent 现在使用 EnterWorktree**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh green`

预期：`[通过] Agent 使用了原生 EnterWorktree 工具`

如果失败：逐字记录 agent 的确切输出和合理化解释。这是 REFACTOR 信号 -- Step 1a 文本需要修改。最多尝试 2 次 REFACTOR 迭代。如果 2 次迭代后仍失败，停止并报告。

- [ ] **Step 5：运行 PRESSURE 阶段 -- 确认 agent 在压力下抵抗回退**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh pressure`

预期：`[通过] Agent 在时间压力下仍使用原生工具`

如果失败：逐字记录合理化解释。向 Step 1a 文本添加显式反例（例如，一个 Red Flag 条目："当你的平台提供原生 worktree 工具时，绝不使用 git worktree add"）。重新运行。

- [ ] **Step 6：提交测试脚本**

```bash
git add tests/claude-code/test-worktree-native-preference.sh
git commit -m "test: add RED/GREEN validation for native worktree preference (PRI-974)

Gate test for Step 1a — validates agents prefer EnterWorktree over
git worktree add on Claude Code. Must pass before skill rewrite."
```

---

### Task 2：重写 `using-git-worktrees` SKILL.md

创建 skill 的完整重写。完全替换现有文件。

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md`（完整重写，219 行 → 约 210 行）

**依赖：** Task 1 GREEN 通过。

- [ ] **Step 1：编写完整的新 SKILL.md**

将 `skills/using-git-worktrees/SKILL.md` 的全部内容替换为：

```markdown
---
name: using-git-worktrees
description: 在开始需要与当前工作区隔离的功能工作或执行实施计划之前使用 - 通过原生工具或 git worktree 回退确保隔离工作区存在
---

# Using Git Worktrees

## Overview

确保工作在隔离的工作区中进行。优先使用平台的原生 worktree 工具。仅在没有原生工具可用时回退到手动 git worktree。

**核心原则：** 首先检测现有隔离。然后使用原生工具。然后回退到 git。绝不与 harness 对抗。

**开始时宣布：** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Step 0: 检测现有隔离

**在创建任何内容之前，检查你是否已在隔离工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Submodule 守卫：** `GIT_DIR != GIT_COMMON` 在 git submodule 内也为真。在得出"已在 worktree 中"的结论之前，验证你不在 submodule 中：

```bash
# 如果这返回一个路径，你在一个 submodule 中，不是 worktree -- 继续到 Step 1
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是 submodule）：** 你已在 linked worktree 中。跳到 Step 3（Project Setup）。不要创建另一个 worktree。

报告分支状态：
- 在分支上："Already in isolated workspace at `<path>` on branch `<name>`."
- Detached HEAD："Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**如果 `GIT_DIR == GIT_COMMON`（或在 submodule 中）：** 你在普通仓库检出中。

用户是否已在你的指令中表明了 worktree 偏好？如果没有，在创建 worktree 之前征得同意：

> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

尊重任何已有的声明偏好，不再询问。如果用户拒绝同意，在原地工作并跳到 Step 3。

## Step 1: 创建隔离工作区

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

如果你的平台提供 worktree 或 workspace-isolation 工具，使用它。你知道自己的工具集 -- skill 不需要列出具体工具。原生工具自动处理目录放置、分支创建和清理。

使用原生工具后，跳到 Step 3（Project Setup）。

### 1b. Git Worktree 回退

如果没有原生工具可用，使用 git 手动创建 worktree。

#### 目录选择

按以下优先级顺序：

1. **检查现有目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 替代
   ```
   如果找到，使用该目录。如果两者都存在，`.worktrees` 优先。

2. **检查现有全局目录：**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ls -d ~/.config/superpowers/worktrees/$project 2>/dev/null
   ```
   如果找到，使用它（向后兼容旧的全局路径）。

3. **检查你的指令中的 worktree 目录偏好。** 如果指定，使用它而不再询问。

4. **默认使用 `.worktrees/`。**

#### 安全验证（仅限项目本地目录）

**必须在创建 worktree 之前验证目录已被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，提交更改，然后继续。

**为什么关键：** 防止意外将 worktree 内容提交到仓库。

全局目录（`~/.config/superpowers/worktrees/`）无需验证。

#### 创建 Worktree

```bash
project=$(basename "$(git rev-parse --show-toplevel)")

# 根据选择的位置确定路径
# 项目本地：path="$LOCATION/$BRANCH_NAME"
# 全局：path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

#### Hooks 感知

Git worktree 不继承父仓库的 hooks 目录。创建 worktree 后，如果主仓库中存在 hooks，则符号链接：

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这防止预提交检查、linters 和其他 hooks 在工作移到 worktree 时静默停止。

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），将此视为受限环境。跳过创建，在当前目录中运行设置和基线测试，相应报告。

## Step 3: 项目设置

自动检测并运行适当的设置：

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

## Step 4: 验证干净基线

运行测试以确保工作区从干净状态开始：

```bash
# 使用适合项目的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是否继续或调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| 情况 | 操作 |
|-----------|--------|
| 已在 linked worktree 中 | 跳过创建（Step 0） |
| 在 submodule 中 | 视为普通仓库（Step 0 守卫） |
| 原生 worktree 工具可用 | 使用它（Step 1a） |
| 没有原生工具 | Git worktree 回退（Step 1b） |
| `.worktrees/` 存在 | 使用它（验证已忽略） |
| `worktrees/` 存在 | 使用它（验证已忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 检查指令文件，然后默认 `.worktrees/` |
| 全局路径存在 | 使用它（向后兼容） |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，在原地工作 |
| 基线时测试失败 | 报告失败 + 询问 |
| 没有 package.json/Cargo.toml | 跳过依赖安装 |

## Common Mistakes

### 与 harness 对抗

- **问题：** 当平台已提供隔离时使用 `git worktree add`
- **修复：** Step 0 检测现有隔离。Step 1a 遵循原生工具。

### 跳过检测

- **问题：** 在现有 worktree 内创建嵌套 worktree
- **修复：** 在创建任何内容之前始终运行 Step 0

### 跳过忽略验证

- **问题：** Worktree 内容被跟踪，污染 git status
- **修复：** 创建项目本地 worktree 之前始终使用 `git check-ignore`

### 假设目录位置

- **问题：** 创建不一致性，违反项目约定
- **修复：** 遵循优先级：现有 > 指令文件 > 默认

### 在测试失败时继续

- **问题：** 无法区分新 bug 和预先存在的问题
- **修复：** 报告失败，获得明确的继续许可

## Red Flags

**绝不：**
- 在 Step 0 检测到现有隔离时创建 worktree
- 在原生 worktree 工具可用时使用 git 命令
- 创建 worktree 而不验证已忽略（项目本地）
- 跳过基线测试验证
- 在测试失败时未经询问就继续

**始终：**
- 首先运行 Step 0 检测
- 优先使用原生工具而非 git 回退
- 遵循目录优先级：现有 > 指令文件 > 默认
- 验证项目本地的目录已被忽略
- 自动检测并运行项目设置
- 验证干净的测试基线
- 通过 1b 创建 worktree 后符号链接 hooks

## Integration

**被以下调用：**
- **subagent-driven-development** - Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - Ensures isolated workspace (creates one or verifies existing)
- 任何需要隔离工作区的 skill

**配对：**
- **finishing-a-development-branch** - 工作完成后清理所必需
```

- [ ] **Step 2：验证文件可正确读取**

运行：`wc -l skills/using-git-worktrees/SKILL.md`

预期：约 200-220 行。扫描任何 markdown 格式问题。

- [ ] **Step 3：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat: rewrite using-git-worktrees with detect-and-defer (PRI-974)

Step 0: GIT_DIR != GIT_COMMON detection (skip if already isolated)
Step 0 consent: opt-in prompt before creating worktree (#991)
Step 1a: native tool preference (short, first, declarative)
Step 1b: git worktree fallback with hooks symlink and legacy path compat
Submodule guard prevents false detection
Platform-neutral instruction file references (#1049)"
```

---

### Task 3：重写 `finishing-a-development-branch` SKILL.md

完成 skill 的完整重写。添加环境检测、修复三个 bug、添加基于来源的清理。

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（完整重写，201 行 → 约 220 行）

- [ ] **Step 1：编写完整的新 SKILL.md**

将 `skills/finishing-a-development-branch/SKILL.md` 的全部内容替换为：

```markdown
---
name: finishing-a-development-branch
description: 在实施完成、所有测试通过时使用，需要决定如何整合工作 - 通过结构化选项指导开发工作的完成，用于合并、PR 或清理
---

# Finishing a Development Branch

## Overview

通过提供清晰选项和处理选定的工作流来指导开发工作的完成。

**核心原则：** 验证测试 → 检测环境 → 展示选项 → 执行选择 → 清理。

**开始时宣布：** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: 验证测试

**在展示选项之前，验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
Tests failing (<N> failures). Must fix before completing:

[显示失败]

Cannot proceed with merge/PR until tests pass.
```

停止。不要继续到 Step 2。

**如果测试通过：** 继续到 Step 2。

### Step 2: 检测环境

**在展示选项之前确定工作区状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定展示哪个菜单以及清理如何工作：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 没有 worktree 需要清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源（见 Step 6） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 简化 3 选项（无合并） | 无需清理（外部管理） |

### Step 3: 确定 Base Branch

```bash
# 尝试常见的 base branch
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："This branch split from main - is that correct?"

### Step 4: 展示选项

**普通仓库和命名分支 worktree -- 精确展示这 4 个选项：**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD -- 精确展示这 3 个选项：**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**不要添加解释** - 保持选项简洁。

### Step 5: 执行选择

#### Option 1：本地合并

```bash
# 获取主仓库根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并 -- 在移除任何内容之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 仅在合并成功后：移除 worktree，然后删除分支
# （见 Step 6 的 worktree 清理）
git branch -d <feature-branch>
```

然后：清理 worktree（Step 6）

#### Option 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>

# 创建 PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**不要清理 worktree** -- 用户需要它来迭代 PR 反馈。

#### Option 3：保持现状

报告："Keeping branch <name>. Worktree preserved at <path>."

**不要清理 worktree。**

#### Option 4：丢弃

**先确认：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待精确确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（Step 6），然后强制删除分支：
```bash
git branch -D <feature-branch>
```

### Step 6: 清理工作区

**仅对 Option 1 和 4 运行。** Option 2 和 3 始终保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，没有 worktree 需要清理。完成。

**如果 worktree 路径在 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下：** Superpowers 创建了此 worktree -- 我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自我修复：清理任何过期的注册
```

**否则：** 宿主环境（harness）拥有此工作区。不要移除它。如果你的平台提供 workspace-exit 工具，使用它。否则，保留工作区不动。

## Quick Reference

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持现状 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## Common Mistakes

**跳过测试验证**
- **问题：** 合并损坏的代码，创建失败的 PR
- **修复：** 在提供选项之前始终验证测试

**开放式问题**
- **问题：** "接下来我应该做什么？" 是模糊的
- **修复：** 精确展示 4 个结构化选项（或 detached HEAD 的 3 个）

**为 Option 2 清理 worktree**
- **问题：** 移除用户需要的 worktree 用于 PR 迭代
- **修复：** 仅对 Option 1 和 4 清理

**在移除 worktree 之前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍引用该分支
- **修复：** 先合并，移除 worktree，然后删除分支

**从 worktree 内部运行 git worktree remove**
- **问题：** 当 CWD 在被移除的 worktree 内时命令静默失败
- **修复：** 在 `git worktree remove` 之前始终 `cd` 到主仓库根目录

**清理 harness 拥有的 worktree**
- **问题：** 移除 harness 创建的 worktree 会导致幽灵状态
- **修复：** 仅清理 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下的 worktree

**丢弃时没有确认**
- **问题：** 意外删除工作
- **修复：** 要求输入"discard"确认

## Red Flags

**绝不：**
- 在测试失败时继续
- 不在合并结果上验证测试就合并
- 不确认就删除工作
- 未经明确请求就强制推送
- 在确认合并成功之前移除 worktree
- 清理不是你创建的 worktree（来源检查）
- 从 worktree 内部运行 `git worktree remove`

**始终：**
- 在提供选项之前验证测试
- 在展示菜单之前检测环境
- 精确展示 4 个选项（或 detached HEAD 的 3 个）
- 为 Option 4 获得输入确认
- 仅对 Option 1 和 4 清理 worktree
- 在 worktree 移除之前 `cd` 到主仓库根目录
- 移除后运行 `git worktree prune`

## Integration

**被以下调用：**
- **subagent-driven-development** (Step 7) - 所有任务完成后
- **executing-plans** (Step 5) - 所有批次完成后

**配对：**
- **using-git-worktrees** - 清理该 skill 创建的 worktree 所必需
```

- [ ] **Step 2：验证文件可正确读取**

运行：`wc -l skills/finishing-a-development-branch/SKILL.md`

预期：约 210-230 行。

- [ ] **Step 3：提交**

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

### Task 4：集成更新

对引用 `using-git-worktrees` 的三个文件进行一行的更改。

**文件：**
- 修改：`skills/executing-plans/SKILL.md:68`
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/writing-plans/SKILL.md:16`

- [ ] **Step 1：更新 executing-plans 集成行**

在 `skills/executing-plans/SKILL.md` 中，将第 68 行从：

```markdown
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

改为：

```markdown
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2：更新 subagent-driven-development 集成行**

在 `skills/subagent-driven-development/SKILL.md` 中，将第 268 行从：

```markdown
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

改为：

```markdown
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 3：更新 writing-plans 上下文行**

在 `skills/writing-plans/SKILL.md` 中，将第 16 行从：

```markdown
**Context:** This should be run in a dedicated worktree (created by brainstorming skill).
```

改为：

```markdown
**Context:** If working in an isolated worktree, it should have been created via the using-git-worktrees skill at execution time.
```

- [ ] **Step 4：提交所有三个更改**

```bash
git add skills/executing-plans/SKILL.md skills/subagent-driven-development/SKILL.md skills/writing-plans/SKILL.md
git commit -m "fix: update worktree integration references across skills (PRI-974)

Remove REQUIRED language from executing-plans and subagent-driven-development.
Consent and detection now live inside using-git-worktrees itself.
Fix stale 'created by brainstorming' claim in writing-plans."
```

---

### Task 5：端到端验证

验证完整重写的 skill 协同工作。运行现有测试套件加上手动验证。

**文件：**
- 读取：`tests/claude-code/run-skill-tests.sh`
- 读取：`skills/using-git-worktrees/SKILL.md`（验证最终状态）
- 读取：`skills/finishing-a-development-branch/SKILL.md`（验证最终状态）

- [ ] **Step 1：运行现有测试套件**

运行：`cd tests/claude-code && bash run-skill-tests.sh`

预期：所有现有测试通过。如果有任何失败，调查 -- 集成更改（Task 4）可能破坏了内容断言。

- [ ] **Step 2：重新运行 Step 1a GREEN 测试**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh green`

预期：通过 -- agent 仍使用 EnterWorktree 配合最终 skill 文本（不仅仅是 Task 1 中的最小 Step 1a 添加）。

- [ ] **Step 3：手动验证 -- 从头到尾阅读两个重写的 skill**

阅读 `skills/using-git-worktrees/SKILL.md` 和 `skills/finishing-a-development-branch/SKILL.md` 全文。检查：

1. 没有对旧行为的引用（硬编码的 `CLAUDE.md`、交互式目录提示、"REQUIRED"语言）
2. Step 编号在每个文件内一致
3. Quick Reference 表格与正文匹配
4. Integration 部分交叉引用正确
5. 没有 markdown 格式问题

- [ ] **Step 4：验证 git status 干净**

运行：`git status`

预期：干净的工作树。所有更改已在 Task 1-4 中提交。

- [ ] **Step 5：如需修复则最终提交**

如果手动验证发现问题，修复并提交：

```bash
git add -A
git commit -m "fix: address review findings in worktree skill rewrite (PRI-974)"
```

如果未发现问题，跳过此步骤。
