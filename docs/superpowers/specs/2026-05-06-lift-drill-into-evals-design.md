# 将 drill 提升进 superpowers 作为 `evals/` — 设计

## 背景

Drill 是一个 Python skill 合规性基准测试，存放在其独立仓库 `obra/drill` 中。它驱动真实的 tmux 会话，以一个 LLM actor 充当模拟用户运行，再以一个 LLM verifier 对生成的 transcript 进行校验，并按场景报告通过/失败。它支持 Claude Code、Codex、Gemini CLI，以及（根据近期提交）OpenCode 和 Copilot CLI。

Drill 早已是 superpowers *事实上的* eval harness。drill 仓库中的 PRI-1397 提交系列把约 22 个 superpowers bash 测试提升为 drill 场景，而最近的 superpowers 提交（`a2292c5`）明确删除了一个冗余的 bash 测试，提交信息为 *"replaced by drill behavioral coverage"*。迁移势头已经存在；本 spec 完成这一迁移。

本次工作将 drill 移入 superpowers 下的 `evals/`，在逐文件确认 drill 场景覆盖后删除冗余的 bash 测试，并更新文档，使贡献者能落到新结构上。

## 目标

1. `evals/` 成为 superpowers 中权威的 eval harness——包含完整的 drill 源码、场景、fixtures、prompts、后端配置和测试。
2. `superpowers/tests/` 中已被逐一确认由 drill 场景 100% 覆盖的 bash 测试将被删除；其余保留。
3. `tests/`（plugin 基础设施：bash + node + python 集成测试）与 `evals/`（含 actor + verifier 的 LLM 行为测试）之间的分工是明确且有文档说明的。
4. 顶层文档（`README.md`、`CLAUDE.md`、`docs/testing.md`）把贡献者引导到正确的位置。
5. 独立仓库 `obra/drill` 继续存在（本 PR 不动它），并在本 PR 合并后作为单独的手工步骤归档。

## 非目标

- **CI 集成。** 本次仅限手工运行。自然的后续是"分层"方案：每个 PR 跑快速子集，夜间 + 按需跑完整扫描。这需要 API 预算决策、GitHub Actions secrets，以及一个预装了 `tmux` + `node` + `python` + `claude` / `codex` / `gemini` CLI 的 runner 镜像。超出范围。
- **场景与 skill 同目录共存。** 场景仍集中放在 `evals/scenarios/`。如果我们日后决定每个 skill 应拥有自己的场景，那是一次路径查找并替换的操作；YAML 格式不变。
- **重命名内部 Python 包**（`drill` → `evals`）。目录是 `evals/`（面向用户）；Python 包保留 `drill` 名以缩小 diff。`evals/README.md` 中有简短说明。
- **Drill 仓库归档。** 本 PR 不动 `obra/drill`。合并后，drill 仓库由手工归档（在 GitHub 上设为只读，README 指向 `obra/superpowers/evals/`）。
- **将 `tests/claude-code/analyze-token-usage.py` 提升进 `evals/bin/`。** 有用的工具，不是测试代码。可以日后移动；本 PR 不要求。

## 分支

从 `dev` 切出分支 `f/evals-lift`。本工作独立于正在进行的 `f/cross-platform` PR——除可能的 `README.md` 外没有共享文件改动，而 README 即便冲突也足够小，可在合并时解决。

## 移动后的目录结构

```
superpowers/
  evals/                              ← 新增（完整 drill 副本）
    pyproject.toml                    (Python 3.11，由 uv 管理)
    uv.lock
    .gitignore                        (drill 自带；results/、.venv/、.env)
    README.md                         (原为 drill 的 README；安装说明已更新)
    CLAUDE.md                         (原为 drill 的 CLAUDE.md；路径已更新)
    docs/
      design.md                       (drill 的设计文档——逐字保留，与本 spec 交叉链接)
      manual-testing.md
      pressure-and-red-testing.md
    drill/                            (Python 包；名称保留；cli、engine、actor、verifier 等)
    backends/                         (claude-*.yaml、codex.yaml、gemini.yaml)
    scenarios/                        (32+ 个 YAML 场景)
    setup_helpers/                    (15 个 Python helper；create_base_repo、sdd_*、spec_*、worktree 等)
    fixtures/                         (template-repo、sdd-go-fractals、sdd-svelte-todo)
    prompts/                          (actor.md、verifier.md)
    bin/                              (断言辅助脚本：tool-called、tool-count 等)
    tests/                            (drill 自带的 pytest 套件)

  tests/                              ← 默认保留的 bash 测试
    brainstorm-server/                ← 保留（brainstorm-server JS 代码的 node 测试）
    opencode/                         ← 保留（plugin 加载测试）
    codex-plugin-sync/                ← 保留（同步校验）
    claude-code/                      ← 基本保留——见删除门槛
    explicit-skill-requests/          ← 除非确认已被替代，否则保留
    skill-triggering/                 ← 除非确认已被替代，否则保留
    subagent-driven-dev/              ← 除非确认已被替代，否则保留

  docs/
    testing.md                        ← 已更新（拆分为"Plugin tests"+ "Skill behavior evals"）
    superpowers/
      specs/
        2026-05-06-lift-drill-into-evals-design.md   ← 本 spec

  README.md                           ← 在 Contributing 章节加一行指向 evals/
  CLAUDE.md                           ← 一行"Eval harness 位于 evals/"的指针
```

本 PR 之后，`tests/` 和 `evals/` 目录承担清晰不同的角色：

- **`tests/`** — plugin 的非 LLM 代码能正常工作吗？针对 brainstorm-server JS 代码、OpenCode plugin 加载、codex-plugin-sync 同步校验的单元和集成测试。bash + node + python。
- **`evals/`** — agent 在真实 LLM 会话上行为正确吗？含 actor + verifier 的 drill 场景。仅 Python，运行真实 tmux 会话。

## 删除门槛（按 bash 测试）

仅当某个 drill 场景可验证地覆盖某 bash 测试的全部断言时，该 bash 测试才被删除。实现计划按文件记录此验证：阅读 bash 测试、列出其检查项、找到对应的 drill 场景、确认每一项检查都有匹配的 `verify.assertions` 或 `verify.criteria` 条目。即便只有一项检查缺失，选择也是要么扩展 drill 场景，要么保留该 bash 测试。默认保留。

**暂定的覆盖映射表**（基于提交信息；任何删除前需逐文件验证）：

| Bash 测试 | 声称的 drill 替代 | 覆盖状态 |
|-----------|---------------------------|-----------------|
| `tests/skill-triggering/prompts/*`（6 个 prompt 文件） | `triggering-*.yaml`（6 个场景） | 候选——删除前逐 prompt 验证 |
| `tests/skill-triggering/run-test.sh`、`run-all.sh` | 无（是运行器，不是测试） | **保留**——运行器脚本 |
| `tests/explicit-skill-requests/prompts/please-use-brainstorming.txt` | 需验证——drill 暂无明显的对应项 | 可能**保留**，除非新增 drill 场景 |
| `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` | 需验证——drill 暂无明显的对应项 | 可能**保留**，除非新增 drill 场景 |
| `tests/explicit-skill-requests/run-claude-describes-sdd.sh` | 部分对应 → `mid-conversation-skill-invocation.yaml` | 候选——逐脚本验证 |
| `tests/explicit-skill-requests/run-haiku-test.sh` | 没有 drill 场景覆盖 Haiku 特有行为 | **保留** |
| `tests/explicit-skill-requests/run-multiturn-test.sh`、`run-extended-multiturn-test.sh` | 没有 drill 场景覆盖多轮累积 | **保留**，除非新增 drill 场景 |
| `tests/explicit-skill-requests/run-test.sh`、`run-all.sh` | 无（是运行器） | **保留** |
| `tests/subagent-driven-dev/go-fractals/`、`tests/subagent-driven-dev/svelte-todo/` | `sdd-go-fractals.yaml`、`sdd-svelte-todo.yaml` | 候选——删除前验证（这些包含关于测试套件通过的真实断言） |
| `tests/claude-code/test-document-review-system.sh` | `spec-reviewer-catches-planted-flaws.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-requesting-code-review.sh` | `code-review-catches-planted-bugs.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | `sdd-rejects-extra-features.yaml`（YAGNI 子集） | **部分**——bash 测试还断言 ≥3 次提交 / `npm test` 通过 / 运行 `analyze-token-usage.py`。drill 场景断言 forbidden-exports + reviewer-as-gate。两者基本不相交——几乎肯定**保留 + 扩展 drill 场景**。 |
| `tests/claude-code/test-subagent-driven-development.sh` | 元/文档测试（要求 agent *描述* SDD）；没有 drill 场景覆盖描述类测试 | **保留**，除非新增 drill 场景 |
| `tests/claude-code/test-worktree-native-preference.sh` | `worktree-creation-under-pressure.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-helpers.sh`、`run-skill-tests.sh`、`analyze-token-usage.py` | 无（是工具，不是测试） | **保留**——库/工具 |

## 验证协议（subagent 把关）

实现计划中的每一项改动在提交前都由一个独立 subagent 交叉检查。

| 改动类别 | Subagent 验证 |
|----------------|----------------------|
| 每个 bash 测试删除 | 派发一个 subagent，提供：(a) bash 测试文件内容、(b) 候选 drill 场景 YAML、(c) 提示词：*"列出 bash 测试的每一个断言。列出 drill 场景中的每一个 verify 条目。对每个 bash 断言，找到匹配的 drill 检查项，或报告为无匹配。输出一张逐断言的表。"* 该 subagent 的输出即是门槛——只有当每个 bash 断言都有匹配项时才删除。 |
| 初始 `evals/` 复制 | Subagent 验证：(a) 提升提交信息中记录了被复制的 drill SHA，使来源可审计；(b) 每个文件的**逐文件 SHA-256 校验和**与 drill 仓库匹配（不仅仅是文件数）；(c) 被排除的路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、任何 `.private-journal/`）在 `evals/` 中不存在；(d) 所有后端 YAML 引用的路径在移动后仍然存在；(e) `pyproject.toml`、`uv.lock`、`.gitignore` 完整。 |
| drill 自带的 pytest 套件 | Subagent 在路径默认值改动后运行 `cd evals && uv run pytest`。drill 在 `evals/tests/` 附带自己的 pytest 套件，包括 `test_backend.py`，它会测试 `SUPERPOWERS_ROOT` 环境变量行为——这些测试必须更新以匹配 helper 并继续通过。 |
| 删除后的引用清理 | Subagent 在整个 superpowers 树（排除 `node_modules/`、`.venv/` 和 `evals/`）中 grep 已删除 bash 测试路径的引用。搜索目标：`docs/`、`docs/superpowers/plans/`、`RELEASE-NOTES.md`、`CLAUDE.md`、`GEMINI.md`、`AGENTS.md`、`README.md`、`.github/`、`scripts/`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`lefthook.yml`。任何命中要么被更新，要么暴露一个被遗漏的依赖。 |
| 路径默认值改动（`SUPERPOWERS_ROOT` 默认值） | Subagent 在路径改动后至少运行一个便宜的 drill 场景（例如 `triggering-test-driven-development`）并确认其仍然通过。是真实校验，而非仅代码审查。 |
| PR 前最终对抗式审查 | 两个 subagent 并行，"谁找到最多合理问题就得 5 分"的话术——与 cross-platform PR 所用协议相同。同时校验源代码和行为。 |

实现计划中每个 subagent 任务都有独立的条目，带有明确的输入和通过标准。subagent 的输出被摘录进相应提交信息（"Subagent verification: …"）中，使审计链可追溯。

## 具体的路径/配置编辑

**在编写本 spec 前已验证。** `drill/cli.py` 定义 `PROJECT_ROOT = Path(__file__).parent.parent`。移动后 `cli.py` 位于 `evals/drill/cli.py`，因此 `PROJECT_ROOT` 解析为 `evals/`，而 `PROJECT_ROOT.parent` 解析为 superpowers 仓库根目录。这正是 `SUPERPOWERS_ROOT` 默认应取的值。

**YAML 替换审计。** 只有四个 `claude*.yaml` 后端配置在 `args` 中（用于 `--plugin-dir` 标志）插值 `${SUPERPOWERS_ROOT}`；`codex.yaml` 和 `gemini.yaml` 仅在 `required_env` 中列出 `SUPERPOWERS_ROOT`（由 `engine.py:233` / `setup.py:25` 在前/后运行钩子中的 `os.environ["SUPERPOWERS_ROOT"]` 查找消费）。helper 的 `os.environ` 改动同时覆盖两条代码路径。

| 文件 | 当前 | 之后 |
|------|---------|-------|
| `drill/cli.py` | 模块导入时 `load_dotenv(PROJECT_ROOT / ".env")`；关于 `SUPERPOWERS_ROOT` 无任何内容 | 在 `load_dotenv` 之后，调用新 helper `_set_superpowers_root_default()`，当且仅当尚未设置时，将 `os.environ["SUPERPOWERS_ROOT"]` 设为 `str(PROJECT_ROOT.parent)`。顺序：`load_dotenv` → 设置默认 → click group 定义。 |
| `drill/engine.py:233`、`drill/setup.py:25` | 直接 `os.environ["SUPERPOWERS_ROOT"]` 访问（未设置时 KeyError） | 不变。CLI 启动钩子保证在 engine/setup 执行时该环境变量已设置。 |
| `backends/claude*.yaml`（5 个文件） | `${SUPERPOWERS_ROOT}` 在 `args` 中被替换以用于 `--plugin-dir` | 不变。YAML 替换在后端加载时读取 `os.environ`，这在 CLI 启动之后。 |
| `backends/codex.yaml`、`backends/gemini.yaml` | `SUPERPOWERS_ROOT` 仅在 `required_env` 中 | 从 `required_env` 中移除（由 helper 提供）。`claude*.yaml` 为向后兼容保留 `required_env`（环境变量仍可作为覆盖）。 |
| `evals/tests/test_backend.py` | 测试断言 `SUPERPOWERS_ROOT` 在 `required_env` 列表中，外加路径解析测试 | 更新测试以匹配新契约：helper 提供默认值、环境覆盖仍然有效、codex/gemini 不再要求 `required_env`。 |
| `evals/README.md` | "export SUPERPOWERS_ROOT=/path/to/superpowers" | 删除该 export 行；说明该环境变量自动默认为 `evals/` 的父目录；指出唯一要求的设置是 `ANTHROPIC_API_KEY`（或 `OPENAI_API_KEY` / Gemini 认证）。 |
| `evals/CLAUDE.md` | 同上 | 同上 |
| `evals/.gitignore` | drill 现有模式（`results/`、`.venv/`、`__pycache__/`、`.env`、`*.pyc`、`*.egg-info/`、`dist/`、`build/`、`.claude/`） | 逐字复制。模式相对于文件位置，因此在 `evals/` 下仍正确生效。 |
| `evals/lefthook.yml` | drill 附带 `lefthook.yml`，定义 `pre-commit: uv run ruff check && uv run ty check` | 移至 `evals/lefthook.yml`。要么 (a) 在 superpowers 根安装 lefthook 并让它联邦到 `evals/lefthook.yml`，要么 (b) 文档说明贡献者手工运行 `cd evals && lefthook run pre-commit`。**实现中的决定：为简洁选择选项 (b)**——superpowers 顶层工作流不变。 |

`.env` 放置：保留 `evals/.env`（已被 gitignore）。贡献者从那里 source 它，或在 shell 环境中设置 `ANTHROPIC_API_KEY`。

**需要小幅增补的顶层 superpowers 文件：**

- `superpowers/.gitignore`：添加 `evals/results/`、`evals/.venv/`、`evals/.env`（双保险；evals/.gitignore 已在本地覆盖这些）。
- `superpowers/CLAUDE.md`：添加一行指针"Eval harness 位于 `evals/`——见 `evals/README.md`"，以便 agent 发现它。
- `superpowers/docs/testing.md`：拆分为"## Plugin tests"（既有 tests/ 内容，并裁剪已删除测试的引用）和"## Skill behavior evals"（一段摘要 + 指向 `evals/` 的指针）。
- `superpowers/README.md`：在 Contributing 章节添加一行，指向 `evals/` 用于 skill 行为测试。

## 迁移顺序

每一步都是一个独立提交（或一小组提交）。第 2 步是最大的单次提交（逐字 drill 复制）；后续步骤都很小且原子。

```
1. 从 dev 切出分支（f/evals-lift）

2. 把 drill 仓库复制进 evals/（单次提交，便于回退）
   ├─ 复制时记录 drill SHA → 写入提交信息
   ├─ 使用 `rsync -a --exclude=.git --exclude=.venv --exclude=results
   │  --exclude=.env --exclude=__pycache__ --exclude='*.egg-info'
   │  --exclude=.private-journal /path/to/drill/ evals/`
   │  （选择 rsync 而非 `cp -r` 是为了显式排除；用
   │  `find evals -name '.git' -type d` 验证返回为空）
   ├─ Subagent 把关：每个非排除文件的 SHA-256 校验和与 drill 仓库匹配；
   │  被排除路径在 evals/ 中不存在
   └─ 冒烟检查：`cd evals && uv sync` 成功（仅证明可安装；
      不是行为测试）

3. 更新路径默认值
   ├─ 向 drill/cli.py 添加 _set_superpowers_root_default() helper
   ├─ 在 load_dotenv 之后、click group 定义之前接入
   ├─ 更新 evals/README.md 和 evals/CLAUDE.md（删除 SUPERPOWERS_ROOT 安装步骤）
   ├─ 从 codex.yaml/gemini.yaml 的 required_env 中删除 SUPERPOWERS_ROOT
   │  （在 claude*.yaml 中保留作为覆盖）
   └─ 更新 evals/tests/test_backend.py 以匹配新契约

4. 从新位置验证（两项检查）
   ├─ 运行 drill 自带的 pytest：`cd evals && uv run pytest` ——必须通过
   └─ 运行便宜的 drill 场景：`cd evals && uv run drill run
      triggering-test-driven-development -b claude` ——必须通过。
      是真实的行为校验，而非仅代码审查。

5. Bash 测试删除阶段——逐文件并配 subagent 把关
   对候选删除清单中的每个文件：
   a. Subagent 比对 bash 测试断言与 drill 场景 verify 块
   b. 通过标准：每个 bash 断言都有匹配的 drill 检查
   c. 若通过 → 删除该 bash 测试文件（每个文件或每组连贯文件一次提交）
   d. 若失败 → 要么扩展 drill 场景（单独提交 + 验证），要么
      保留该 bash 测试（不提交）

6. 过时引用清理
   ├─ Subagent 在 superpowers 树（排除 node_modules/、.venv/、
   │  evals/）中 grep 已删除文件路径
   ├─ 搜索目标：docs/、docs/superpowers/plans/、RELEASE-NOTES.md、
   │  CLAUDE.md、GEMINI.md、AGENTS.md、README.md、.github/、scripts/、
   │  .opencode/INSTALL.md、.codex-plugin/INSTALL.md、lefthook.yml
   ├─ 更新活动引用（例如 docs/testing.md、README.md 安装）
   └─ docs/superpowers/plans/*.md 和 RELEASE-NOTES.md 中的历史引用
      被保留并附简短注释
      （"（test removed; behavior covered by drill scenario X）"），
      而非重写——这些是带日期的产物，不是活文档。

7. 顶层文档
   ├─ docs/testing.md 拆分
   ├─ CLAUDE.md 指针
   └─ README.md Contributing 章节

8. 重跑冒烟检查（回归门槛）
   ├─ `cd evals && uv run pytest`
   └─ `cd evals && uv run drill run triggering-test-driven-development -b claude`

9. 最终对抗式审查
   └─ 两个并行 subagent，完整 diff，"谁找到最多合理问题就得 5 分"话术。
      在推送前处理发现的问题。

10. 推送分支 + 向 dev 开 PR
    └─ PR 描述包括：复制时锁定的 drill SHA、归档待办事项
       （"after merge: archive obra/drill, add README pointer to
       obra/superpowers/evals/"）、每个已删除文件的覆盖凭据。
```

## 验证（实现之后）

实现计划必须展示：

- 第 2 步后所有非排除的 drill 源文件都存在于 `evals/`（subagent **逐文件 SHA-256 校验和比对** vs `obra/drill@<recorded-sha>`）。
- 被排除路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、`.private-journal/`）在 `evals/` 中不存在。
- 第 2 步提交信息记录了 drill 源 SHA。
- 在未设置 `SUPERPOWERS_ROOT` 的情况下 `cd evals && uv sync` 成功。
- `cd evals && uv run pytest` 通过（drill 自带 pytest 套件）。
- `cd evals && uv run drill list` 返回与独立 drill 仓库在记录 SHA 处相同的场景数。
- `cd evals && uv run drill run triggering-test-driven-development -b claude` 通过（证明路径默认值端到端工作）。
- 对每个已删除的 bash 测试：提交信息中有 subagent 验证表，展示每个断言如何映射到某个 drill 检查。
- grep 已删除文件路径在活 superpowers 文档中返回零命中（第 6 步之后）；`docs/superpowers/plans/*.md` 和 `RELEASE-NOTES.md` 中的历史引用被注释而非重写。
- `docs/testing.md` 同时具有 "Plugin tests" 和 "Skill behavior evals" 章节。
- drill 仓库的历史未被触动；`obra/drill` 不受本 PR 影响。
- PR 描述列出合并后归档 `obra/drill` 的待办事项。

## 待澄清问题

无。所有澄清性决策都已作出：

| 问题 | 决定 |
|----------|----------|
| drill 在 superpowers 中位于何处？ | `evals/`（从 drill 改名）；独立仓库作为单独步骤归档 |
| 冗余 bash 测试的命运？ | 逐文件删除并配 subagent 覆盖验证；默认保留 |
| 场景布局？ | 集中于 `evals/scenarios/` |
| Python 工具链放置？ | 自包含于 `evals/` |
| CI 集成？ | 本 PR 仅手工；文档化未来路径 |
| 迁移机制？ | 普通复制；drill 仓库历史保留在被归档仓库中，不进 in-tree |
| 内部 Python 包名？ | 保留为 `drill`（目录为 `evals/`） |
| 分支策略？ | 独立于 `dev` 切出（不叠在 `f/cross-platform` 之上） |
