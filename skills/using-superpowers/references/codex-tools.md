# Codex 工具映射

Skills 使用 Claude Code 工具名称。当你在 skill 中遇到这些时，使用你的平台等价物：

| Skill 引用 | Codex 等价物 |
|-----------------|------------------|
| `Task` 工具（dispatch subagent） | `spawn_agent`（见 [Subagent dispatch 需要多代理支持](#subagent-dispatch-requires-multi-agent-support)） |
| 多个 `Task` 调用（并行） | 多个 `spawn_agent` 调用 |
| Task 返回结果 | `wait_agent` |
| Task 自动完成 | `close_agent` 释放槽位 |
| `TodoWrite`（任务跟踪） | `update_plan` |
| `Skill` 工具（调用 skill） | Skills 原生加载 — 只需遵循说明 |
| `Read`、`Write`、`Edit`（文件） | 使用你的原生文件工具 |
| `Bash`（运行命令） | 使用你的原生 shell 工具 |

## Subagent dispatch 需要多代理支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skills 启用 `spawn_agent`、`wait_agent` 和 `close_agent`。

遗留说明：`rust-v0.115.0` 之前的 Codex 构建将 spawned-agent 等待暴露为 `wait`。当前 Codex 对 spawned agents 使用 `wait_agent`。`wait` 名称现在属于 code-mode `exec/wait`，它通过 `cell_id` 恢复一个 yield 的 exec 单元；它不是 spawned-agent 结果工具。

## 环境检测

创建 worktree 或完成分支的 skills 应在继续之前使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已经在 linked worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从 sandbox 分支/推送/创建 PR）

见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1 了解每个 skill 如何使用这些信号。

## Codex App 完成

当 sandbox 阻止分支/推送操作时（在外部管理的 worktree 中的 detached HEAD），agent 会提交所有工作并通知用户使用 App 的原生控件：

- **"创建分支"** — 命名分支，然后通过 App UI commit/push/创建 PR
- **"移交给本地"** — 将工作转移到用户的本地 checkout

Agent 仍然可以运行测试、暂存文件，并输出建议的分支名称、commit 消息和 PR 描述供用户复制。
