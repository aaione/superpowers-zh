---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过、且你需要决定如何集成工作时使用 - 通过为合并、PR 或清理呈现结构化选项来指导开发工作的完成
---

# 完成开发分支 (Finishing a Development Branch)

## 概述 (Overview)

通过呈现清晰的选项并处理所选工作流，指导开发工作的完成。

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开始时宣布：** "我正在使用 finishing-a-development-branch skill 来完成这项工作。"

## 流程 (The Process)

### 步骤 1：验证测试

**在呈现选项之前，验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。完成前必须修复：

[展示失败]

测试通过之前无法继续合并/PR。
```

停下。不要继续步骤 2。

**如果测试通过：** 继续步骤 2。

### 步骤 2：检测环境

**在呈现选项之前确定工作区状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定显示哪个菜单以及清理如何工作：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 需清理 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 标准 4 选项 | 基于来源（见步骤 6） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 精简 3 选项（无合并） | 无清理（外部管理） |

### 步骤 3：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："这个分支从 main 分出 —— 对吗？"

### 步骤 4：呈现选项

**普通仓库和具名分支 worktree —— 精确呈现这 4 个选项：**

```
实现完成。你想怎么做？

1. 本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支原样（我稍后处理）
4. 丢弃这项工作

选哪个？
```

**Detached HEAD —— 精确呈现这 3 个选项：**

```
实现完成。你处于 detached HEAD（外部管理的工作区）。

1. 作为新分支推送并创建 Pull Request
2. 保持原样（我稍后处理）
3. 丢弃这项工作

选哪个？
```

**不要添加解释** —— 保持选项简洁。

### 步骤 5：执行选择

#### 选项 1：本地合并

```bash
# 获取主仓库根目录以保证 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并 —— 在移除任何东西之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 仅在合并成功后：清理 worktree（步骤 6），然后删除分支
```

然后：清理 worktree（步骤 6），然后删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>
```

**不要清理 worktree** —— 用户需要它存活以便根据 PR 反馈迭代。

#### 选项 3：保持原样

报告："保留分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
这将永久删除：
- 分支 <name>
- 所有 commits：<commit-list>
- <path> 处的 worktree

输入 'discard' 确认。
```

等待精确确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（步骤 6），然后强制删除分支：
```bash
git branch -D <feature-branch>
```

### 步骤 6：清理工作区

**仅对选项 1 和 4 运行。** 选项 2 和 3 总是保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无 worktree 需清理。完成。

**如果 worktree 路径在 `.worktrees/` 或 `worktrees/` 下：** Superpowers 创建了这个 worktree —— 清理由我们负责。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自愈：清理任何陈旧的注册
```

**否则：** 宿主环境（harness）拥有这个工作区。不要移除它。如果你的平台提供工作区退出工具，使用它。否则，保留工作区原样。

## 快速参考 (Quick Reference)

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误 (Common Mistakes)

**跳过测试验证**
- **问题：** 合并坏代码，创建失败的 PR
- **修复：** 提供选项前总是验证测试

**开放式问题**
- **问题：** "我接下来该做什么？"是模糊的
- **修复：** 精确呈现 4 个结构化选项（detached HEAD 为 3 个）

**为选项 2 清理 worktree**
- **问题：** 移除了用户进行 PR 迭代所需的 worktree
- **修复：** 仅对选项 1 和 4 清理

**在移除 worktree 之前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍引用该分支
- **修复：** 先合并，移除 worktree，然后删除分支

**从 worktree 内部运行 git worktree remove**
- **问题：** 当 CWD 在被移除的 worktree 内时命令静默失败
- **修复：** 总是在 `git worktree remove` 之前 `cd` 到主仓库根

**清理 harness 拥有的 worktree**
- **问题：** 移除 harness 创建的 worktree 会导致幽灵状态
- **修复：** 仅清理 `.worktrees/` 或 `worktrees/` 下的 worktree

**丢弃时不确认**
- **问题：** 意外删除工作
- **修复：** 要求键入 "discard" 确认

## 危险信号 (Red Flags)

**绝不：**
- 带着失败的测试继续
- 不在结果上验证测试就合并
- 不确认就删除工作
- 没有明确请求就强制推送
- 在确认合并成功之前移除 worktree
- 清理你没创建的 worktree（来源检查）
- 从 worktree 内部运行 `git worktree remove`

**总是：**
- 提供选项前验证测试
- 呈现菜单前检测环境
- 精确呈现 4 个选项（detached HEAD 为 3 个）
- 对选项 4 获取键入确认
- 仅对选项 1 和 4 清理 worktree
- 在 worktree 移除前 `cd` 到主仓库根
- 移除后运行 `git worktree prune`
