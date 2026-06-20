# Codex 工具映射 (Codex Tool Mapping)

Skills 以动作表述（"派发一个 subagent"、"创建一个 todo"、"读取一个文件"）。在 Codex 中，这些动作对应下面的工具。

| Skills 请求的动作 | Codex 等价物 |
|----------------------|------------------|
| 读取一个文件 | `shell`（例如 `cat`、`head`、`tail`）——Codex 通过 shell 读取文件 |
| 创建 / 编辑 / 删除一个文件 | `apply_patch`（用于创建、更新、删除的结构化 diff） |
| 运行一条 shell 命令 | `shell` |
| 搜索文件内容 | `shell`（例如 `grep`、`rg`） |
| 按名称查找文件 | `shell`（例如 `find`、`ls`） |
| 抓取一个 URL | `shell` 配合 `curl` / `wget`——Codex 没有原生的抓取工具 |
| 搜索网络 | `web_search`（默认启用；可在 `config.toml` 中通过顶层 `web_search` 设置配置——`live`、`cached` 或 `disabled`） |
| 调用一个 skill | Skills 原生加载——只需遵循指令 |
| 派发一个 subagent（`Subagent (general-purpose):` 模板） | `spawn_agent`（参见 [Subagent 派发需要多 agent 支持](#subagent-dispatch-requires-multi-agent-support)） |
| 多个并行派发 | 在一次响应中进行多次 `spawn_agent` 调用 |
| 等待 subagent 结果 | `wait_agent` |
| 完成后释放 subagent 槽位 | `close_agent` |
| 任务跟踪（"创建一个 todo"、"标记完成"） | `update_plan` |

## 指令文件

当某个 skill 提及"你的指令文件"时，在 Codex 中它是位于项目根目录的 **`AGENTS.md`**。Codex 还会读取 `~/.codex/AGENTS.md` 以获取全局上下文，并且在存在 `AGENTS.override.md`（在项目树中或 `~/.codex/` 中）时它具有优先权。Codex 从项目根目录向下遍历到当前工作目录，拼接沿途找到的 `AGENTS.md` 文件，上限为 `project_doc_max_bytes`（默认 32 KiB）。

## 个人 skills 目录

用户级 skills 位于 **`$CODEX_HOME/skills/`**（默认 `~/.codex/skills/`）。Codex 还会读取跨运行时路径 **`~/.agents/skills/`**（与 Copilot CLI 和 Gemini CLI 共享）。当两个目录在同一作用域同时存在时，Codex 会将两者作为独立的 skill 目录分别加载——Codex 的文档目前没有记录它们之间的优先级。每个 skill 是一个子目录，包含一个 `SKILL.md`（带有 `name` 和 `description` frontmatter）。

## Subagent 派发需要多 agent 支持

在你的 Codex 配置（`~/.codex/config.toml`）中添加：

```toml
[features]
multi_agent = true
```

这会为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skills 启用 `spawn_agent`、`wait_agent` 和 `close_agent`。

遗留说明：`rust-v0.115.0` 之前的 Codex 构建将派生 agent 的
等待暴露为 `wait`。当前的 Codex 对派生 agent 使用 `wait_agent`。`wait`
这个名称现在属于 code-mode 的 `exec/wait`，它按 `cell_id` 恢复一个让出的 exec
cell；它不是派生 agent 的结果工具。

## 环境检测

创建 worktree 或收尾分支的 skills 应该在继续之前用只读 git 命令检测其
环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已经在一个链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → 处于 detached HEAD（无法从 sandbox 分支/推送/提 PR）

关于每个 skill 如何使用这些信号，参见 `using-git-worktrees` 的 Step 0 和
`finishing-a-development-branch` 的 Step 1。

## Codex App 收尾

当 sandbox 阻止分支/推送操作（在外部管理的 worktree 中处于
detached HEAD）时，agent 会提交所有工作并通知
用户使用 App 的原生控件：

- **"Create branch"**——为分支命名，然后通过 App UI 进行提交/推送/提 PR
- **"Hand off to local"**——将工作转移到用户的本地检出

agent 仍可以运行测试、暂存文件，并输出建议的分支
名称、提交消息和 PR 描述供用户复制。
