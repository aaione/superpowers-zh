# 测试 Superpowers (Testing Superpowers)

Superpowers 有两类截然不同的测试，分别位于各自的目录中：

- **`tests/`** —— 插件的非 LLM 代码是否工作正常？针对 brainstorm-server 的 JS 代码、OpenCode 插件加载、codex-plugin 同步和分析工具，提供 Bash + node + python 集成测试。
- **`evals/`** —— 在真实 LLM 会话中，agent 行为是否正确？由 Python 测试驱动程序驱动 Claude Code / Codex / Gemini CLI 的真实 tmux 会话，并由一个 LLM actor 和验证器来判定 skill 合规性。

## 插件测试

位于 `tests/`。当前包括：

- `tests/brainstorm-server/` —— brainstorm server JS 代码的 node 测试套件。
- `tests/opencode/` —— 针对 OpenCode 插件加载、bootstrap 缓存和工具注册的 bash 测试。
- `tests/codex-plugin-sync/` —— bash 同步验证。
- `tests/kimi/` —— 针对 Kimi 插件 manifest 接线的 bash/Python 检查。
- `tests/claude-code/test-helpers.sh`、`analyze-token-usage.py` —— 其余 bash 测试使用的工具。
- `tests/claude-code/test-subagent-driven-development.sh` —— agent 能否描述 SDD 的测试（没有对应的 drill 场景；测试的是描述回忆能力，而非行为）。
- `tests/claude-code/test-subagent-driven-development-integration.sh` —— 带 token 分析的扩展 SDD 集成测试（drill 覆盖 YAGNI 子集；bash 额外覆盖提交数、Claude Code 任务追踪和 token 遥测断言）。
- `tests/claude-code/test-worktree-native-preference.sh` —— worktree skill 的 RED-GREEN-REFACTOR 验证（drill 覆盖 PRESSURE 阶段；bash 还覆盖 RED/GREEN 基线）。
- `tests/explicit-skill-requests/` —— 针对 Haiku 的、多轮的、以及通过 skill 名称提示的测试，覆盖 drill 未涵盖的部分。

通过相应目录的 `run-*.sh` 或 `npm test` 运行插件测试。

## Skill 行为 eval

位于 `evals/`。Drill 是测试驱动程序；场景位于 `evals/scenarios/*.yaml`。设置步骤见 `evals/README.md`。快速开始：

```bash
cd evals
uv sync --extra dev
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill 场景运行较慢（每个 3-30 分钟以上），且会运行真实 LLM 会话。它们目前不属于 CI；自然的后续是采用分层模型（PR 上跑快速子集，夜间和按需跑全量扫描）。
