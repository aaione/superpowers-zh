# Codex App 兼容性：Worktree 和完成 Skill 适配

使 superpowers skill 在 Codex App 的沙箱 worktree 环境中工作，不破坏现有的 Claude Code 或 Codex CLI 行为。

**工单：** PRI-823

## 动机

Codex App 在它管理的 git worktree 内运行 agent -- detached HEAD，位于 `$CODEX_HOME/worktrees/` 下，带有 Seatbelt 沙箱，阻止 `git checkout -b`、`git push` 和网络访问。三个 superpowers skill 假设不受限制的 git 访问：`using-git-worktrees` 用命名分支创建手动 worktree，`finishing-a-development-branch` 按分支名 merge/push/PR，`subagent-driven-development` 需要两者。

Codex CLI（开源终端工具）没有此冲突 -- 它没有内置的 worktree 管理。我们的手动 worktree 方法在那里填补了隔离空缺。问题具体出在 Codex App 上。

## 经验发现

2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙箱 | 完全访问沙箱 |
|---|---|---|
| `git add` | 可用 | 可用 |
| `git commit` | 可用 | 可用 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 可用 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 可用 |
| `gh pr create` | **被阻止**（网络） | 可用 |
| `git status/diff/log` | 可用 | 可用 |

额外发现：
- `spawn_agent` subagent **共享**父线程的文件系统（通过标记文件测试确认）
- "Create branch"按钮在 App header 中始终显示，无论 worktree 从哪个分支启动
- App 的原生完成流程：Create branch → Commit 模态框 → Commit and push / Commit and create PR
- `network_access = true` 配置在 macOS 上静默失效（issue #10390）

## 设计：只读环境检测

三个只读 git 命令检测环境，无副作用：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

衍生两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON` -- agent 在其他人创建的 worktree 中（Codex App、Claude Code Agent tool、之前的 skill 运行、或用户）
- **ON_DETACHED_HEAD：** `BRANCH` 为空 -- 没有命名分支存在

为什么使用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在普通仓库中，两者解析到相同的 `.git` 目录
- 在 linked worktree 中，`git-dir` 是 `.git/worktrees/<name>` 而 `git-common-dir` 是 `.git`
- 在 submodule 中，两者相等 -- 避免了 `show-toplevel` 会产生的误报
- 通过 `cd && pwd -P` 解析处理了相对路径问题（`git-common-dir` 在普通仓库中返回相对的 `.git` 但在 worktree 中返回绝对路径）和符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| Linked Worktree? | Detached HEAD? | 环境 | 操作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整 skill 行为（不变） |
| 是 | 是 | Codex App worktree（workspace-write） | 跳过 worktree 创建；完成时交接 payload |
| 是 | 否 | Codex App（完全访问）或手动 worktree | 跳过 worktree 创建；完整完成流程 |
| 否 | 是 | 不寻常（手动 detached HEAD） | 正常创建 worktree；完成时警告 |

## 更改

### 1. `using-git-worktrees/SKILL.md` -- 添加 Step 0（约 12 行）

在"Overview"和"Directory Selection Process"之间新增部分：

**Step 0: 检查是否已在隔离工作区中**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过 worktree 创建。而是：
1. 跳到 Creation Steps 下的"Run Project Setup"子部分 -- `npm install` 等操作是幂等的，为了安全值得运行
2. 然后"Verify Clean Baseline" -- 运行测试
3. 报告分支状态：
   - 在分支上："Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD："Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

如果 `GIT_DIR == GIT_COMMON`，继续完整的 worktree 创建流程（不变）。

安全验证（.gitignore 检查）在 Step 0 触发时跳过 -- 对外部创建的 worktree 不相关。

更新 Integration 部分的"Called by"条目。将每个条目的描述从特定上下文的文本改为："Ensures isolated workspace (creates one or verifies existing)"。例如，`subagent-driven-development` 条目从"REQUIRED: Set up isolated workspace before starting"改为"REQUIRED: Ensures isolated workspace (creates one or verifies existing)"。

**沙箱回退：** 如果 `GIT_DIR == GIT_COMMON` 且 skill 继续到 Creation Steps，但 `git worktree add -b` 因权限错误失败（例如 Seatbelt 沙箱拒绝），将此视为延迟检测到的受限环境。回退到 Step 0 的"已在工作区中"行为 -- 跳过创建，在当前目录中运行设置和基线测试，相应报告。

在 Step 0 报告后，停止。不要继续到 Directory Selection 或 Creation Steps。

**其他一切不变：** Directory Selection、Safety Verification、Creation Steps、Project Setup、Baseline Tests、Quick Reference、Common Mistakes、Red Flags。

### 2. `finishing-a-development-branch/SKILL.md` -- 添加 Step 1.5 + 清理守卫（约 20 行）

**Step 1.5: 检测环境**（在 Step 1 "Verify Tests"之后，Step 2 "Determine Base Branch"之前）

运行检测命令。三条路径：

- **路径 A** 完全跳过 Steps 2 和 3（不需要 base branch 或选项）。
- **路径 B 和 C** 正常继续通过 Step 2（Determine Base Branch）和 Step 3（Present Options）。

**路径 A -- 外部管理的 worktree + detached HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的完成控件操作已提交的工作。

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

分支名称推导：如有 ticket ID 则使用（例如 `pri-823/codex-compat`），否则将 plan 标题的前 5 个词 slugify，否则省略建议。避免包含敏感内容（漏洞描述、客户名称）在分支名称中。

跳到 Step 5（对外部管理的 worktree 清理是空操作）。

**路径 B -- 外部管理的 worktree + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

正常展示 4 选项菜单。（Step 5 清理守卫将独立重新检测外部管理的状态。）

**路径 C -- 正常环境**（`GIT_DIR == GIT_COMMON`）：

按原样展示 4 选项菜单（不变）。

**Step 5 清理守卫：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 检测（不要依赖之前的 skill 输出 -- 完成 skill 可能在不同的会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove` -- 宿主环境拥有此工作区。

否则，按原样检查和移除。注意：现有 Step 5 文本说"For Options 1, 2, 4"但 Quick Reference 表和 Common Mistakes 部分说"Options 1 & 4 only."新守卫添加在此现有逻辑之前，不改变哪些选项触发清理。

**其他一切不变：** Options 1-4 逻辑、Quick Reference、Common Mistakes、Red Flags。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` -- 各 1 行编辑

两个 skill 有相同的 Integration 部分行。从：
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**其他一切不变：** Dispatch/review 循环、prompt 模板、模型选择、状态处理、red flags。

### 4. `codex-tools.md` -- 添加环境检测文档（约 15 行）

末尾两个新部分：

**环境检测：**

```markdown
## 环境检测

创建 worktree 或完成分支的 skill 应在继续之前使用只读 git 命令检测环境：

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → 已在 linked worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法在沙箱中 branch/push/PR）

参见 `using-git-worktrees` Step 0 和 `finishing-a-development-branch`
Step 1.5 了解每个 skill 如何使用这些信号。
```

**Codex App 完成：**

```markdown
## Codex App 完成

当沙箱阻止 branch/push 操作（外部管理的 worktree 中的 detached HEAD）时，agent 提交所有工作并通知用户使用 App 的原生控件：

- **"Create branch"** -- 命名分支，然后通过 App UI 提交/push/PR
- **"Hand off to local"** -- 将工作转移到用户的本地检出

Agent 仍可运行测试、暂存文件，并为用户输出建议的分支名称、提交消息和 PR 描述供复制。
```

## 不改变的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` -- subagent prompt 未触动
- `executing-plans/SKILL.md` -- 仅更改 1 行 Integration 描述（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` -- 没有 worktree 或完成操作
- `.codex/INSTALL.md` -- 安装过程不变
- 4 选项完成菜单 -- 为 Claude Code 和 Codex CLI 精确保留
- 完整 worktree 创建流程 -- 为非 worktree 环境精确保留
- Subagent dispatch/review/iterate 循环 -- 不变（文件系统共享已确认）

## 范围总结

| 文件 | 更改 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + 清理守卫） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

跨 5 个文件添加/更改约 50 行。零个新文件。零个破坏性更改。

## 未来考量

如果第三个 skill 需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件中（方案 B）。目前不需要 -- 只有 2 个 skill 使用它。

## 测试计划

### 自动化（实施后在 Claude Code 中运行）

1. 普通仓库检测 -- 断言 IN_LINKED_WORKTREE=false
2. Linked worktree 检测 -- `git worktree add` 测试 worktree，断言 IN_LINKED_WORKTREE=true
3. Detached HEAD 检测 -- `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 完成 skill 交接输出 -- 验证受限环境中的交接消息（非 4 选项菜单）
5. **Step 5 清理守卫** -- 创建 linked worktree（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入其中，运行 Step 5 清理检测（`GIT_DIR` vs `GIT_COMMON`），断言它不会调用 `git worktree remove`。然后 `cd` 回主仓库，运行相同的检测，断言它会调用 `git worktree remove`。之后清理测试 worktree。

### 手动 Codex App 测试（5 项测试）

1. Worktree 线程中的检测（workspace-write）-- 验证 GIT_DIR != GIT_COMMON，空 branch
2. Worktree 线程中的检测（完全访问）-- 相同检测，不同沙箱行为
3. 完成 skill 交接格式 -- 验证 agent 发出交接 payload，而非 4 选项菜单
4. 完整生命周期 -- 检测 → 提交 → 完成检测 → 正确行为 → 清理
5. **Local 线程中的沙箱回退** -- 启动 Codex App 的 **Local 线程**（workspace-write 沙箱）。提示："Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change."预检：`git checkout -b test-sandbox-check` 应该因 `Operation not permitted` 失败。预期：skill 检测到 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到 Step 0 的"已在工作区中"行为 -- 运行设置、基线测试，从当前目录报告就绪。通过：agent 优雅恢复，无神秘错误消息。失败：agent 打印原始 Seatbelt 错误、重试或以令人困惑的输出放弃。

### 回归

- 现有 Claude Code skill 触发测试仍通过
- 现有 subagent-driven-development 集成测试仍通过
- 普通 Claude Code 会话：完整 worktree 创建 + 4 选项完成仍可用
