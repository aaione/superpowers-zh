---
name: finishing-a-development-branch
description: 当实施完成，所有测试通过，你需要决定如何集成工作时使用 - 通过提供合并、PR 或清理的结构化选项来指导开发工作的完成
---

# Finishing a Development Branch

## 概述 (Overview)

通过提供清晰的选项和处理所选的工作流来指导开发工作的完成。

**核心原则:** 验证测试 → 展示选项 → 执行选择 → 清理。

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

### 第二步: 确定 Base Branch (基准分支)

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问: "This branch split from main - is that correct?"

### 第三步: 展示选项

确切地展示这 4 个选项:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**不要添加解释** - 保持选项简洁。

### 第四步: 执行选择

#### 选项 1: 本地 Merge

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

然后: Cleanup worktree (第五步)

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

然后: Cleanup worktree (第五步)

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
git checkout <base-branch>
git branch -D <feature-branch>
```

然后: Cleanup worktree (第五步)

### 第五步: Cleanup Worktree

**对于选项 1, 2, 4:**

检查是否在 worktree 中:
```bash
git worktree list | grep $(git branch --show-current)
```

如果是:
```bash
git worktree remove <worktree-path>
```

**对于选项 3:** 保留 worktree。

## 快速参考 (Quick Reference)

| 选项 | Merge | Push | Keep Worktree | Cleanup Branch |
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
- **问题:** "What should I do next?" → 模棱两可
- **修复:** 展示确切的 4 个结构化选项

**自动 worktree清理**
- **问题:** 在可能需要 worktree 时将其删除 (选项 2, 3)
- **修复:** 仅针对选项 1 和 4 进行清理

**丢弃无确认**
- **问题:** 意外删除工作
- **修复:** 需要输入 "discard" 确认

## 危险信号 (Red Flags)

**绝不 (Never):**
- 在测试失败的情况下继续
- 未在结果上验证测试即合并
- 未经确认删除工作
- 未经明确请求强制推送 (Force-push)

**总是 (Always):**
- 在提供选项之前验证测试
- 展示确切的 4 个选项
- 获得选项 4 的输入确认
- 仅为选项 1 和 4 清理 worktree

## 集成 (Integration)

**被调用方:**
- **subagent-driven-development** (Step 7) - 所有任务完成后
- **executing-plans** (Step 5) - 所有 batches 完成后

**搭配使用:**
- **using-git-worktrees** - 清理该 skill 创建的 worktree
