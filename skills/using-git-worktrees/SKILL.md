---
name: using-git-worktrees
description: 当开始需要与当前 workspace 隔离的功能工作或在执行实施 plans 之前使用 - 通过智能目录选择和安全验证创建隔离的 git worktrees
---

# Using Git Worktrees

## 概述 (Overview)

Git worktrees 创建共享同一存储库的隔离 workspaces，允许同时在多个分支上工作而无需切换。

**核心原则:** 系统目录选择 + 安全验证 = 可靠隔离。

**开始时宣布:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## 目录选择流程

遵循此优先顺序:

### 1. 检查现有目录

```bash
# Check in priority order
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**如果找到:** 使用该目录。如果两者都存在，`.worktrees` 胜出。

### 2. Check CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**如果指定了首选项:** 不经询问直接使用它。

### 3. Ask User

如果目录不存在且无 CLAUDE.md 首选项:

```
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## 安全验证 (Safety Verification)

### 对于 Project-Local 目录 (.worktrees or worktrees)

**创建 worktree 之前必须验证目录被忽略:**

```bash
# Check if directory is ignored (respects local, global, and system gitignore)
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果没有被忽略:**

根据 Jesse 的规则 "Fix broken things immediately":
1. 将适当的行添加到 .gitignore
2. Commit 更改
3. 继续创建 worktree

**为什么关键:** 防止意外将 worktree 内容 commit 到存储库。

### 对于 Global 目录 (~/.config/superpowers/worktrees)

不需要 .gitignore 验证 - 完全在项目之外。

## 创建步骤 (Creation Steps)

### 1. 检测项目名称

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. 创建 Worktree

```bash
# Determine full path
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# Create worktree with new branch
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. 运行项目设置

自动检测并运行适当的设置:

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

### 4. 验证干净基线

运行测试以确保 worktree 启动干净:

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**如果测试失败:** 报告失败，询问是继续还是调查。

**如果测试通过:** 报告就绪。

### 5. 报告位置

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速参考 (Quick Reference)

| 情况 | 行动 |
|-----------|--------|
| `.worktrees/` 存在 | 使用它 (验证忽略) |
| `worktrees/` 存在 | 使用它 (验证忽略) |
| 两者都存在 | 使用 `.worktrees/` |
| 都不存在 | 检查 CLAUDE.md → Ask user |
| 目录未忽略 | 添加到 .gitignore + commit |
| 基线期间测试失败 | 报告失败 + ask |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误 (Common Mistakes)

### 跳过忽略验证

- **问题:** Worktree 内容被跟踪，污染 git status
- **修复:** 在创建 project-local worktree 之前总是使用 `git check-ignore`

### 假设目录位置

- **问题:** 造成不一致，违反项目惯例
- **修复:** 遵循优先级: existing > CLAUDE.md > ask

### 在测试失败的情况下继续

- **问题:** 无法区分新 bugs 和原有问题
- **修复:** 报告失败，获得明确许可继续

###硬编码设置命令

- **问题:** 在使用不同工具的项目上中断
- **修复:** 从项目文件 (package.json 等) 自动检测

## 示例工作流

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## 危险信号 (Red Flags)

**绝不 (Never):**
- 在未验证其被忽略的情况下创建 worktree (project-local)
- 跳过基线测试验证
- 在未询问的情况下继续失败的测试
- 当模棱两可时假设目录位置
- 跳过 CLAUDE.md 检查

**总是 (Always):**
- 遵循目录优先级: existing > CLAUDE.md > ask
- 验证 project-local 目录被忽略
- 自动检测并运行项目设置
- 验证干净的测试基线

## 集成 (Integration)

**被调用方:**
- **brainstorming** (Phase 4) - 必需，当设计被批准且实施随之而来时
- 任何需要隔离 workspace 的 skill

**搭配使用:**
- **finishing-a-development-branch** - 必需，用于工作完成后的清理
- **executing-plans** 或 **subagent-driven-development** - 工作在这个 worktree 中发生
