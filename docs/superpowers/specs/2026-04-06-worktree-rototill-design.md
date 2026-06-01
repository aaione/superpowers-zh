# Worktree Rototill：检测与遵循

**日期：** 2026-04-06
**状态：** 草案
**工单：** PRI-974
**取代：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对 worktree 管理有自己的主张 -- 特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理（`git worktree remove`）。与此同时，Claude Code、Codex App、Gemini CLI 和 Cursor 都提供了原生的 worktree 支持，各有自己的路径、生命周期管理和清理。

这产生了三种故障模式：

1. **重复** -- 在 Claude Code 上，skill 做 `EnterWorktree`/`ExitWorktree` 已做的事
2. **冲突** -- 在 Codex App 上，skill 尝试在已管理的 worktree 内创建 worktree
3. **幽灵状态** -- skill 创建的 `.worktrees/` 下的 worktree 对 harness 不可见；harness 创建的 `.claude/worktrees/` 下的 worktree 对 skill 不可见

对于没有原生支持的 harness（Codex CLI、OpenCode、Copilot 独立版），superpowers 填补了真实的空白。Skill 不应该消失 -- 它应该在没有原生支持时让开。

## 目标

1. 当原生 harness worktree 系统存在时遵循它
2. 继续为缺少原生支持的 harness 提供 worktree 支持
3. 修复 finishing-a-development-branch 中的三个已知 bug（#940、#999、#238）
4. 使 worktree 创建为选择加入而非强制（#991）
5. 用平台中立的语言替换硬编码的 `CLAUDE.md` 引用（#1049）

## 非目标

- 每个 worktree 的环境约定（`.worktree-env.sh`、端口偏移）-- Phase 4
- 用于路径强制执行的 PreToolUse hooks -- Phase 4
- 多仓库 worktree 文档 -- Phase 4
- Brainstorming 清单的 worktree 更改 -- Phase 4
- `.superpowers-session.json` 元数据跟踪（有趣的 PR #997 想法，v1 不需要）
- Hooks 符号链接到 worktree 中（PR #965 想法，单独的关注点）

## 设计原则

### 检测状态，而非平台

使用 `GIT_DIR != GIT_COMMON` 来确定"我是否已在 worktree 中？"而不是通过嗅探环境变量来识别 harness。这是一个稳定的 git 原语（自 git 2.5，2015 年），在所有 harness 上通用工作，随着新 harness 出现需要零维护。

### 声明式意图，规定式回退

Skill 描述目标（"确保工作在隔离工作区中进行"）并在可用时遵循原生工具。它只在没有原生 worktree 支持的 harness 上规定特定的 git 命令作为回退。Step 1a 优先并明确命名原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；Step 1b 其次是 git 回退。原始规格保持 Step 1a 抽象（"你知道自己的工具集"），但 TDD 证明当 Step 1a 过于模糊时，agent 会锚定在 Step 1b 的具体命令上。显式工具命名和同意-授权桥接是使偏好可靠所需的。

### 基于来源的所有权

谁创建 worktree 谁拥有其清理。如果 harness 创建了它，superpowers 不碰它。如果 superpowers 创建了它（通过 git 回退），superpowers 清理它。启发式：如果 worktree 在 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下，superpowers 拥有它。其他任何位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`）属于 harness。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

Skill 在创建之前获得三个新步骤并简化创建流程。

#### Step 0：检测现有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 操作 |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 普通仓库检出 | 继续到 Step 0.5 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 已在 linked worktree 中 | 跳到 Step 3（项目设置）。报告："Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 外部管理的 worktree（例如 Codex App 沙箱） | 跳到 Step 3。报告："Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

Step 0 不关心谁创建了 worktree 或哪个 harness 在运行。Worktree 不论来源都是 worktree。

**Submodule 守卫：** `GIT_DIR != GIT_COMMON` 在 git submodule 内也为真。在得出"已在 worktree 中"的结论之前，检查我们不在 submodule 中：

```bash
# 如果这返回一个路径，你在一个 submodule 中，不是 worktree
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果在 submodule 中，视为 `GIT_DIR == GIT_COMMON`（继续到 Step 0.5）。

#### Step 0.5：同意

当 Step 0 发现没有现有隔离时（`GIT_DIR == GIT_COMMON`），创建前询问：

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

如果同意，继续到 Step 1。如果不同意，在原地工作 -- 跳到 Step 3，不创建 worktree。

当 Step 0 检测到现有隔离时，此步骤完全跳过（对已存在的东西询问没有意义）。

#### Step 1a：原生工具（首选）

> 用户已请求隔离工作区（Step 0 同意）。检查你的可用工具 -- 你有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志吗？如果是：用户创建 worktree 的同意就是你使用它的授权。现在使用它并跳到 Step 3。

使用原生工具后，跳到 Step 3（项目设置）。

**设计说明 -- TDD 修订：** 原始规格使用了刻意简短、抽象的 Step 1a（"你知道自己的工具集 -- skill 不需要列出具体工具"）。TDD 验证推翻了这一点：agent 锚定在 Step 1b 的具体 git 命令上，忽略了抽象指导（2/6 通过率）。三项更改修复了它（GREEN 和 PRESSURE 测试中 50/50 通过率）：

1. **显式工具命名** -- 按名称列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree` 将决策从解释（"我有原生工具吗？"）转变为事实查找（"`EnterWorktree` 在我的工具列表中吗？"）。在没有这些工具的平台上的 agent 只需检查、找不到并回退到 Step 1b。没有观察到误报。

2. **同意桥接** -- "用户创建 worktree 的同意就是你使用它的授权"直接解决了 `EnterWorktree` 的工具级别护栏（"仅在用户明确要求时"）。工具描述覆盖 skill 指令（Claude Code #29950），所以 skill 必须将用户同意表述为工具需要的授权。

3. **Red Flag 条目** -- 在 Red Flags 部分中命名特定的反模式（"当你有原生 worktree 工具时使用 `git worktree add` -- 这是 #1 错误"）。

文件拆分（Step 1b 在单独的 skill 中）已被测试并证明不必要。锚定问题通过 Step 1a 文本的质量解决，而非通过 git 命令的物理分离。使用完整 240 行 skill（所有 git 命令可见）的控制测试通过了 20/20。

#### Step 1b：Git Worktree 回退

当没有原生工具可用时，手动创建 worktree。

**目录选择**（优先级顺序）：
1. 检查现有的 `.worktrees/` 或 `worktrees/` 目录 -- 如果找到，使用它。如果两者都存在，`.worktrees/` 优先。
2. 检查现有的 `~/.config/superpowers/worktrees/<project>/` 目录 -- 如果找到，使用它（向后兼容旧的全局路径）。
3. 检查项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）中的 worktree 目录偏好。
4. 默认使用 `.worktrees/`。

没有交互式目录选择提示。全局路径（`~/.config/superpowers/worktrees/`）不再作为选择提供给新用户，但该位置的现有 worktree 会被检测并使用以实现向后兼容。

**安全验证**（仅限项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未被忽略，添加到 `.gitignore` 并在继续之前提交。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**Hooks 感知：** Git worktree 不继承父仓库的 hooks 目录。通过 1b 创建 worktree 后，如果主仓库中存在 hooks 目录，则符号链接：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这防止预提交检查、linters 和其他 hooks 在工作移到 worktree 时静默停止。（想法来自 PR #965。）

**沙箱回退：** 如果 `git worktree add` 因权限错误失败，将此视为受限环境。跳过创建，在当前目录中工作，继续到 Step 3。

**Step 编号说明：** 当前 skill 有 Steps 1-4 作为扁平列表。此重新设计使用 0、0.5、1a、1b、3、4。没有 Step 2 -- 它是旧的单一"Create Isolated Workspace"，现在拆分为 1a/1b 结构。实施应该干净地重新编号（例如 0 -> "Step 0: Detect"、0.5 -> 在 Step 0 的流程内、1a/1b -> "Step 1"、3 -> "Step 2"、4 -> "Step 3"）或保持当前编号并附注。实施者选择。

#### Steps 3-4：项目设置和基线测试（不变）

无论哪条路径创建了工作区（Step 0 检测到现有、Step 1a 原生工具、Step 1b git 回退、或根本没有 worktree），执行都会收敛：

- **Step 3：** 自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **Step 4：** 运行测试套件。如果测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

完成 skill 增加环境检测并修复三个 bug。

#### Step 1：验证测试（不变）

运行项目的测试套件。如果测试失败，停止。不提供完成选项。

#### Step 1.5：检测环境（新增）

重新运行与创建时 Step 0 相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 没有 worktree 需要清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源（见 Step 5） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 简化菜单：推送为新分支 + PR、保持现状、丢弃 | 没有合并选项（无法从 detached HEAD 合并） |

#### Step 2：确定 Base Branch（不变）

#### Step 3：展示选项

**普通仓库和命名分支 worktree：**

1. 本地合并回 `<base-branch>`
2. 推送并创建 Pull Request
3. 保持分支现状（我稍后处理）
4. 丢弃此工作

**Detached HEAD：**

1. 推送为新分支并创建 Pull Request
2. 保持现状（我稍后处理）
3. 丢弃此工作

#### Step 4：执行选择

**Option 1（本地合并）：**

```bash
# 获取主仓库根目录以确保 CWD 安全（Bug #238 修复）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并，在移除任何内容之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# 仅在合并成功后：移除 worktree，然后删除分支（Bug #999 修复）
git worktree remove "$WORKTREE_PATH"  # 仅当 superpowers 拥有时
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除 worktree → 删除分支。旧 skill 在移除 worktree 之前删除了分支（因为 worktree 仍引用分支而失败）。先移除 worktree 的天真修复也是错误的 -- 如果合并随后失败，工作目录就没了，更改就丢失了。

**Option 2（创建 PR）：**

推送分支，创建 PR。不要清理 worktree -- 用户需要它来迭代 PR 反馈。（Bug #940 修复：移除矛盾的"Then: Cleanup worktree"文本。）

**Option 3（保持现状）：** 无操作。

**Option 4（丢弃）：** 要求输入"discard"确认。然后移除 worktree（如果 superpowers 拥有），强制删除分支。

#### Step 5：清理（已更新）

```
if GIT_DIR == GIT_COMMON:
    # 普通仓库，没有 worktree 需要清理
    完成

if worktree 路径在 .worktrees/ 或 ~/.config/superpowers/worktrees/ 下：
    # Superpowers 创建了它 -- 我们负责清理
    cd 到主仓库根目录       # Bug #238 修复
    git worktree remove <path>

否则：
    # Harness 创建了它 -- 不碰
    # 如果平台提供 workspace-exit 工具，使用它
    # 否则，保留 worktree 不动
```

清理仅对 Option 1 和 4 运行。Option 2 和 3 始终保留 worktree。（Bug #940 修复。）

**过期 worktree 修剪：** 在任何 `git worktree remove` 之后，运行 `git worktree prune` 作为自我修复步骤。Worktree 目录可能被越界删除（例如由 harness 清理、手动 `rm` 或 `.claude/` 清理），留下导致混淆错误的过期注册。一行代码，防止静默腐烂。（想法来自 PR #1072。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者当前在集成部分中将 `using-git-worktrees` 列为 REQUIRED。更改为：

> `using-git-worktrees` -- Ensures isolated workspace (creates one or verifies existing)

Skill 本身现在处理同意（Step 0.5）和检测（Step 0），所以调用 skill 不需要门控或提示。

#### `writing-plans`

移除过期的声称"should be run in a dedicated worktree (created by brainstorming skill)。"Brainstorming 是一个设计 skill，不创建 worktree。Worktree 提示在执行时通过 `using-git-worktrees` 发生。

### 4. 平台中立的指令文件引用

所有 worktree 相关 skill 中硬编码的 `CLAUDE.md` 实例替换为：

> "项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）"

这适用于 Step 1b 中的目录偏好检查。

## Bug 修复（捆绑）

| Bug | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | Option 2 文本说"Then: Cleanup worktree (Step 5)"但快速参考说保留。Step 5 说"For Options 1, 2, 4"但 Common Mistakes 说"Options 1 and 4 only." | 从 Option 2 中移除清理。Step 5 仅适用于 Option 1 和 4。 | finishing SKILL.md |
| #999 | Option 1 在移除 worktree 之前删除分支。`git branch -d` 可能失败，因为 worktree 仍引用该分支。 | 重新排序为：合并 → 验证测试 → 移除 worktree → 删除分支。合并必须在任何东西被移除之前成功。 | finishing SKILL.md |
| #238 | `git worktree remove` 当 CWD 在被移除的 worktree 内时静默失败。 | 添加 CWD 守卫：在 `git worktree remove` 之前 `cd` 到主仓库根目录。 | finishing SKILL.md |

## 已解决的问题

| Issue | 解决方案 |
|-------|-----------|
| #940 | 直接修复（Bug #940） |
| #991 | Step 0.5 中选择加入的同意 |
| #918 | Step 0 检测 + Step 1.5 完成检测 |
| #1009 | 由 Step 1a 解决 -- agent 使用原生工具（例如 `EnterWorktree`），它们在 harness 原生路径创建。取决于 Step 1a 工作；见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立的指令文件引用 |
| #279 | 由检测与遵循解决 -- 原生路径被尊重，因为我们不覆盖它们 |
| #574 | **推迟。** 此规格没有触及 bug 所在的 brainstorming skill。完整修复（在 brainstorming 的清单中添加 worktree 步骤）是 Phase 4。 |

## 风险

### Step 1a 是承载假设 -- 已解决

Step 1a -- agent 偏好原生 worktree 工具而非 git 回退 -- 是整个设计的基础。如果 agent 忽略 Step 1a 并在拥有原生支持的 harness 上回退到 Step 1b，检测与遵循就完全失败。

**状态：** 此风险在实施期间已具体化。原始的抽象 Step 1a（"你知道自己的工具集"）在 Claude Code 上以 2/6 失败。TDD 门控按设计工作 -- 它在任何 skill 文件被修改之前捕获了失败，防止了损坏的发布。三次 REFACTOR 迭代识别了根本原因（agent 锚定具体命令、工具描述护栏覆盖 skill 指令）并产生了在 GREEN 和 PRESSURE 测试中以 50/50 验证的修复。详见上方 Step 1a 设计说明。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一具有 agent 可在会话中调用的 worktree 工具（`EnterWorktree`）的 harness。其他所有 harness 要么在 agent 启动之前创建 worktree（Codex App、Gemini CLI、Cursor），要么没有原生 worktree 支持（Codex CLI、OpenCode）。Step 1a 向前兼容：当其他 harness 添加 agent 可调用的 worktree 工具时，agent 会将它们与命名的示例匹配并使用它们而无需更改 skill。

| Harness | 当前 worktree 模型 | Skill 机制 | 已测试 |
|---------|----------------------|-----------------|--------|
| Claude Code | Agent 可调用 `EnterWorktree` | Step 1a | 50/50 (GREEN + PRESSURE) |
| Codex CLI | 无原生工具（仅 shell） | Step 1b git 回退 | 6/6 (`codex exec`) |
| Gemini CLI | 启动时 `--worktree` 标志，无 agent 工具 | 如果使用 flag 启动则 Step 0，否则 Step 1b | Step 0: 1/1, Step 1b: 1/1 (`gemini -p`) |
| Cursor Agent | 面向用户的 `/worktree`，无 agent 工具 | 如果用户激活则 Step 0，否则 Step 1b | Step 0: 1/1, Step 1b: 1/1 (`cursor-agent -p`) |
| Codex App | 平台管理，detached HEAD，无 agent 工具 | Step 0 检测现有 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无 agent 工具 | Step 1b git 回退 | 未测试（无 CLI 访问） |

**残余风险：**
1. 如果 Anthropic 将 `EnterWorktree` 的工具描述更改为更限制性（例如"不要基于 skill 指令使用"），同意桥接就会失效。值得提交 issue 请求工具描述适应 skill 驱动的调用。
2. 当其他 harness 添加 agent 可调用的 worktree 工具时，它们可能使用 Step 1a 列表中没有的名称。列表应随着新工具的出现而更新。通用措辞（"worktree 或 workspace-isolation 工具"）提供了一些前瞻覆盖。

### 来源启发式

`.worktrees/` 或 `~/.config/superpowers/worktrees/` = 我们的，其他一切 = 不碰 的启发式对每个当前 harness 都有效。如果未来的 harness 采用 `.worktrees/` 作为其约定，我们会有误报（superpowers 尝试清理 harness 拥有的 worktree）。同样，如果用户在没有 superpowers 的情况下手动运行 `git worktree add .worktrees/experiment`，我们会错误地声称所有权。两者都是低风险的 -- 每个 harness 使用品牌化路径，手动 `.worktrees/` 创建不太可能 -- 但值得注意。

### Detached HEAD 完成

Detached HEAD worktree 的简化菜单（没有合并选项）对 Codex App 的沙箱模型是正确的。如果用户因其他原因处于 detached HEAD，简化菜单仍然合理 -- 你确实无法从 detached HEAD 合并而不先创建分支。

## 实施说明

两个 skill 文件都包含在实施期间需要更新的核心步骤之外的部分：

- **Frontmatter**（`name`、`description`）：更新以反映检测与遵循行为
- **Quick Reference 表格**：重写以匹配新的步骤结构和 bug 修复
- **Common Mistakes 部分**：更新或移除引用旧行为的项目（例如"跳过 CLAUDE.md 检查"现在是错误的）
- **Red Flags 部分**：更新以反映新的优先级（例如"当 Step 0 检测到现有隔离时绝不创建 worktree"）
- **Integration 部分**：更新 skill 之间的交叉引用

规格描述了*更改什么*；实施计划将指定这些次要部分的确切编辑。

## 未来工作（不在此规格中）

- **Phase 3 剩余：** `$TMPDIR` 目录选项（#666）、缓存和环境继承的设置文档（#299）
- **Phase 4：** 用于路径强制执行的 PreToolUse hooks（#1040）、每个 worktree 的环境约定（#597）、brainstorming 清单 worktree 步骤（#574）、多仓库文档（#710）
