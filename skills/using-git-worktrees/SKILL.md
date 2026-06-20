---
name: using-git-worktrees
description: 当开始需要与当前工作区隔离的功能工作，或在执行实现计划之前使用 - 通过原生工具或 git worktree 回退确保存在隔离工作区
---

# 使用 Git Worktrees (Using Git Worktrees)

## 概述 (Overview)

确保工作发生在隔离的工作区中。优先使用你平台的原生 worktree 工具。仅在没有原生工具可用时回退到手动 git worktree。

**核心原则：** 先检测既有隔离。然后使用原生工具。然后回退到 git。绝不与 harness 对抗。

**开始时宣布：** "我正在使用 using-git-worktrees skill 来设置隔离工作区。"

## 步骤 0：检测既有隔离

**在创建任何东西之前，检查你是否已在一个隔离工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块守卫：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为真。在下结论"已在 worktree 中"之前，验证你不在子模块中：

```bash
# 如果这返回一个路径，你在子模块中，不是 worktree —— 当作普通仓库处理
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已在一个 linked worktree 中。跳到步骤 2（项目设置）。不要创建另一个 worktree。

按分支状态报告：
- 在分支上："已在 `<path>` 的隔离工作区中，分支 `<name>`。"
- Detached HEAD："已在 `<path>` 的隔离工作区中（detached HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你在普通仓库检出中。

用户是否已在你的指令中表明 worktree 偏好？如果没有，在创建 worktree 前请求同意：

> "你想让我设置一个隔离的 worktree 吗？它会保护你当前的分支免受改动。"

尊重任何已声明的偏好，无需询问。如果用户拒绝同意，就地工作并跳到步骤 2。

## 步骤 1：创建隔离工作区

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已要求隔离工作区（步骤 0 同意）。你是否已有创建 worktree 的方式？它可能是一个名为 `EnterWorktree`、`WorktreeCreate` 的工具，一个 `/worktree` 命令，或一个 `--worktree` 标志。如果有，使用它并跳到步骤 2。

原生工具自动处理目录放置、分支创建和清理。当你有原生工具时使用 `git worktree add` 会创建你的 harness 无法看到或管理的幽灵状态。

仅在没有原生 worktree 工具可用时才继续步骤 1b。

### 1b. Git Worktree 回退

**仅当步骤 1a 不适用时使用** —— 你没有可用的原生 worktree 工具。使用 git 手动创建 worktree。

#### 目录选择

遵循此优先顺序。显式用户偏好总是优先于观察到的文件系统状态。

1. **检查你的指令中是否有已声明的 worktree 目录偏好。** 如果用户已指定一个，无需询问直接使用。

2. **检查是否存在项目本地 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，使用它。如果两者都存在，`.worktrees` 优先。

3. **如果没有其他指导可用**，默认在项目根目录使用 `.worktrees/`。

#### 安全验证（仅项目本地目录）

**创建 worktree 前必须验证目录被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，提交改动，然后继续。

**为何关键：** 防止意外把 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# 根据所选位置确定路径
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，你改为在当前目录工作。然后就地运行设置和基线测试。

## 步骤 2：项目设置

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

## 步骤 3：验证干净基线

运行测试以确保工作区以干净状态开始：

```bash
# 使用适合项目的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree 就绪于 <full-path>
测试通过（<N> 个测试，0 个失败）
准备好实现 <feature-name>
```

## 快速参考 (Quick Reference)

| 情况 | 动作 |
|-----------|--------|
| 已在 linked worktree 中 | 跳过创建（步骤 0） |
| 在子模块中 | 当作普通仓库处理（步骤 0 守卫） |
| 有原生 worktree 工具 | 使用它（步骤 1a） |
| 无原生工具 | Git worktree 回退（步骤 1b） |
| `.worktrees/` 存在 | 使用它（验证已忽略） |
| `worktrees/` 存在 | 使用它（验证已忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 检查指令文件，然后默认 `.worktrees/` |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，就地工作 |
| 基线时测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误 (Common Mistakes)

### 与 harness 对抗

- **问题：** 当平台已提供隔离时使用 `git worktree add`
- **修复：** 步骤 0 检测既有隔离。步骤 1a 委托给原生工具。

### 跳过检测

- **问题：** 在既有 worktree 内部创建嵌套 worktree
- **修复：** 创建任何东西之前总是运行步骤 0

### 跳过忽略验证

- **问题：** Worktree 内容被追踪，污染 git status
- **修复：** 创建项目本地 worktree 之前总是使用 `git check-ignore`

### 假设目录位置

- **问题：** 造成不一致，违反项目约定
- **修复：** 遵循优先级：显式指令 > 既有项目本地目录 > 默认值

### 带着失败的测试继续

- **问题：** 无法区分新 bug 和既有问题
- **修复：** 报告失败，获得明确许可再继续

## 危险信号 (Red Flags)

**绝不：**
- 当步骤 0 检测到既有隔离时创建 worktree
- 当你有原生 worktree 工具（如 `EnterWorktree`）时使用 `git worktree add`。这是头号错误 —— 如果你有它，就用它。
- 通过直接跳到步骤 1b 的 git 命令来跳过步骤 1a
- 不验证被忽略就创建 worktree（项目本地）
- 跳过基线测试验证
- 不询问就带着失败的测试继续

**总是：**
- 先运行步骤 0 检测
- 优先原生工具而非 git 回退
- 遵循目录优先级：显式指令 > 既有项目本地目录 > 默认值
- 对项目本地验证目录被忽略
- 自动检测并运行项目设置
- 验证干净的测试基线
