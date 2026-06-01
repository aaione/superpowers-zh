---
name: finishing-a-development-branch
description: 当实施完成，所有测试通过，你需要决定如何集成工作时使用 - 通过提供合并、PR 或清理的结构化选项来指导开发工作的完成
---

# Finishing a Development Branch

## 概述 (Overview)

通过提供清晰的选项和处理所选的工作流来指导开发工作的完成。

**核心原则:** 验证测试 → 检测环境 → 展示选项 → 执行选择 → 清理。

**开始时宣布:** "I'm using the finishing-a-development-branch skill to complete this work."

## 流程 (The Process)

### 第一步: 验证测试

**在展示选项之前，验证测试通过:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**如果测试失败:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停止。不要继续到第二步。

**如果测试通过:** 继续到第二步。

### 第二步: 检测环境 (Detect Environment)

**在展示选项之前确定工作区状态:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定了展示哪个菜单以及清理如何工作:

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON` (普通 repo) | 标准 4 个选项 | 无 worktree 需要清理 |
| `GIT_DIR != GIT_COMMON`, 命名分支 | 标准 4 个选项 | 基于来源的 (见第六步) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | 简化 3 个选项 (无 merge) | 无清理 (外部管理) |

### 第三步: 确定 Base Branch (基准分支)

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问: "This branch split from main - is that correct?"

### 第四步: 展示选项

**普通 repo 和命名分支 worktree — 确切地展示这 4 个选项:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD — 确切地展示这 3 个选项:**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**不要添加解释** - 保持选项简洁。

### 第五步: 执行选择

#### 选项 1: 本地 Merge

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

# Only after merge succeeds: cleanup worktree (Step 6), then delete branch
```

然后: 清理 worktree (第六步)，然后删除分支:

```bash
git branch -d <feature-branch>
```

#### 选项 2: Push 并创建 PR

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

**不要清理 worktree** — 用户需要它来迭代 PR 反馈。

#### 选项 3: 保持原样

报告: "Keeping branch <name>. Worktree preserved at <path>."

**不要 cleanup worktree。**

#### 选项 4: 丢弃

**首先确认:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待确切的确认。

如果已确认:
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后: 清理 worktree (第六步)，然后强制删除分支:
```bash
git branch -D <feature-branch>
```

### 第六步: 清理工作区 (Cleanup Workspace)

**仅在选项 1 和 4 时运行。** 选项 2 和 3 总是保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`:** 普通 repo，无 worktree 需要清理。完成。

**如果 worktree 路径在 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下:** Superpowers 创建了此 worktree — 我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自修复: 清理任何过时的注册
```

**否则:** 宿主环境 (harness) 拥有此工作区。不要移除它。如果你的平台提供了 workspace-exit 工具，使用它。否则，保留工作区不动。

## 快速参考 (Quick Reference)

| 选项 | Merge | Push | 保留 Worktree | 清理 Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ | ✓ | - |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

## 常见错误 (Common Mistakes)

**跳过测试验证**
- **问题:** 合并损坏的代码，创建失败的 PR
- **修复:** 在提供选项之前始终验证测试

**开放式问题**
- **问题:** "What should I do next?" is ambiguous → 模棱两可
- **修复:** 确切地展示 4 个结构化选项 (或 detached HEAD 时的 3 个)

**为选项 2 清理 worktree**
- **问题:** 移除用户迭代 PR 所需的 worktree
- **修复:** 仅对选项 1 和 4 进行清理

**在移除 worktree 之前删除分支**
- **问题:** `git branch -d` 失败，因为 worktree 仍然引用该分支
- **修复:** 先 merge，移除 worktree，然后删除分支

**从 worktree 内部运行 git worktree remove**
- **问题:** 当 CWD 在正在被移除的 worktree 内时，命令静默失败
- **修复:** 在 `git worktree remove` 之前总是 `cd` 到主 repo root

**清理 harness 拥有的 worktrees**
- **问题:** 移除 harness 创建的 worktree 会导致幻影状态
- **修复:** 只清理 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下的 worktrees

**丢弃无确认**
- **问题:** 意外删除工作
- **修复:** 需要输入 "discard" 确认

## 危险信号 (Red Flags)

**绝不 (Never):**
- 在测试失败的情况下继续
- 未在结果上验证测试即合并
- 未经确认删除工作
- 未经明确请求强制推送 (Force-push)
- 在确认 merge 成功之前移除 worktree
- 清理你没有创建的 worktrees (来源检查)
- 从 worktree 内部运行 `git worktree remove`

**总是 (Always):**
- 在提供选项之前验证测试
- 在展示菜单之前检测环境
- 确切地展示 4 个选项 (或 detached HEAD 时的 3 个)
- 获得选项 4 的输入确认
- 仅为选项 1 和 4 清理 worktree
- 在移除 worktree 之前 `cd` 到主 repo root
- 在移除后运行 `git worktree prune`

## 集成 (Integration)

**必需的工作流 skills:**
- **superpowers:using-git-worktrees** - 确保隔离的工作区 (创建或验证现有的)
- **superpowers:writing-plans** - 创建此 skill 完成的 plan
- **superpowers:finishing-a-development-branch** - 所有任务后的完成开发

**被调用方:**
- **subagent-driven-development** - 所有任务完成后
- **executing-plans** - 所有 batches 完成后
