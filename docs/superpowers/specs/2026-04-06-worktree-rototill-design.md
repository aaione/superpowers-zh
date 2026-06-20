# Worktree 翻耕：检测并让位

**日期：** 2026-04-06
**状态：** 草案
**工单：** PRI-974
**吸收：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对 worktree 管理有自己的主张——特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理（`git worktree remove`）。与此同时，Claude Code、Codex App、Gemini CLI 和 Cursor 都提供原生 worktree 支持，各有自己的路径、生命周期管理和清理方式。

这造成三种失败模式：

1. **重复**——在 Claude Code 上，skill 做了 `EnterWorktree`/`ExitWorktree` 已经在做的事
2. **冲突**——在 Codex App 上，skill 试图在一个已被管理的 worktree 内部创建 worktree
3. **幽灵状态**——skill 在 `.worktrees/` 创建的 worktree 对 harness 不可见；harness 在 `.claude/worktrees/` 创建的 worktree 对 skill 不可见

对于没有原生支持的 harness（Codex CLI、OpenCode、Copilot 独立版），superpowers 填补了一个真实空白。该 skill 不应消失——它应当在存在原生支持时让位。

## 目标

1. 当存在原生 harness worktree 系统时让位给它
2. 继续为缺少它的 harness 提供 worktree 支持
3. 修复 finishing-a-development-branch 中的三个已知 bug（#940、#999、#238）
4. 使 worktree 创建变为可选而非强制（#991）
5. 用平台中立的措辞替换硬编码的 `CLAUDE.md` 引用（#1049）

## 非目标

- 每 worktree 的环境约定（`.worktree-env.sh`、端口偏移）——Phase 4
- 用于路径强制的 PreToolUse 钩子——Phase 4
- 多仓库 worktree 文档——Phase 4
- 针对 worktree 的 brainstorming 清单改动——Phase 4
- `.superpowers-session.json` 元数据跟踪（有趣的 PR #997 想法，v1 不需要）
- 钩子符号链接进 worktree（PR #965 的想法，独立的关注点）

## 设计原则

### 检测状态，而非平台

使用 `GIT_DIR != GIT_COMMON` 来判定"我是否已在一个 worktree 中"，而非靠嗅探环境变量来识别 harness。这是一个稳定的 git 原语（自 git 2.5，2015 年起），在所有 harness 上通用，且随着新 harness 出现无需维护。

### 声明式意图，规定式回退

该 skill 描述目标（"确保工作发生在隔离工作区中"），并在可用时让位给原生工具。它仅作为没有原生 worktree 支持的 harness 的回退来规定具体 git 命令。步骤 1a 居先并显式点名原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；步骤 1b 居后提供 git 回退。原始 spec 把步骤 1a 保持抽象（"you know your own toolkit"），但 TDD 证明当步骤 1a 过于含糊时，agent 会锚定在步骤 1b 的具体命令上。需要显式工具命名和一个同意-授权桥接才能让该偏好可靠。

### 基于来源的所有权

谁创建 worktree 谁拥有其清理。如果由 harness 创建，superpowers 不碰它。如果由 superpowers 创建（通过 git 回退），superpowers 清理它。启发式：如果 worktree 位于 `.worktrees/` 或 `worktrees/` 下，superpowers 拥有它。其他任何位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`，或旧的用户全局 Superpowers 路径）属于 harness 或用户，不予触碰。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

该 skill 在创建之前获得三个新步骤，并简化创建流程。

#### 步骤 0：检测既有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 动作 |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 普通仓库检出 | 继续到步骤 0.5 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 已在链接的 worktree 中 | 跳到步骤 3（项目设置）。报告："Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 外部管理的 worktree（例如 Codex App 沙箱） | 跳到步骤 3。报告："Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

步骤 0 不关心谁创建了 worktree 或哪个 harness 在运行。worktree 就是 worktree，无论来源如何。

**子模块护栏：**`GIT_DIR != GIT_COMMON` 在 git 子模块内部也成立。在得出"已在一个 worktree 中"之前，检查我们是否不在子模块中：

```bash
# 如果这返回一个路径，我们就在子模块中，而非 worktree
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果在子模块中，按 `GIT_DIR == GIT_COMMON` 处理（继续到步骤 0.5）。

#### 步骤 0.5：同意

当步骤 0 未找到既有隔离（`GIT_DIR == GIT_COMMON`）时，在创建之前询问：

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

如果同意，继续到步骤 1。如果不同意，就地工作——跳到步骤 3，不创建 worktree。

当步骤 0 检测到既有隔离时，此步骤完全跳过（没必要询问已存在的东西）。

#### 步骤 1a：原生工具（首选）

> 用户已要求一个隔离工作区（步骤 0 同意）。检查你的可用工具——你是否有 `EnterWorktree`、`WorktreeCreate`、一个 `/worktree` 命令，或一个 `--worktree` 标志？如果是：用户创建 worktree 的同意就是你对它使用的授权。现在使用它并跳到步骤 3。

使用原生工具之后，跳到步骤 3（项目设置）。

**设计注记——TDD 修订：**原始 spec 使用了一个故意简短、抽象的步骤 1a（"You know your own toolkit — the skill does not need to name specific tools"）。TDD 验证推翻了它：agent 锚定在步骤 1b 的具体 git 命令上而忽略抽象指导（2/6 通过率）。三项改动修复了它（跨 GREEN 和 PRESSURE 测试 50/50 通过率）：

1. **显式工具命名**——按名列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree` 把决策从解释（"我是否有原生工具？"）转变为事实查找（"`EnterWorktree` 是否在我的工具列表中？"）。没有这些工具的平台上的 agent 简单检查、什么也找不到、并落入步骤 1b。未观察到误报。
2. **同意桥接**——"the user's consent to create a worktree is your authorization to use it"直接回应了 `EnterWorktree` 工具级护栏（"ONLY when user explicitly asks"）。工具描述覆盖 skill 指令（Claude Code #29950），因此 skill 必须把用户同意框定为该工具要求的授权。
3. **Red Flag 条目**——在 Red Flags 章节中点名具体反模式（"Use `git worktree add` when you have a native worktree tool — this is the #1 mistake"）。

文件拆分（把步骤 1b 放进独立 skill）已被测试证明不必要。锚定问题由步骤 1a 文本的质量解决，而非通过 git 命令的物理分离。用完整 240 行 skill（所有 git 命令可见）的对照测试通过 20/20。

#### 步骤 1b：Git Worktree 回退

当没有原生工具可用时，手工创建 worktree。

**目录选择**（优先级顺序）：
1. 检查项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等价物）中的 worktree 目录偏好。
2. 检查既有的 `.worktrees/` 或 `worktrees/` 目录——如果找到则使用它。如果两者都存在，`.worktrees/` 优先。
3. 默认为 `.worktrees/`。

无交互式目录选择提示。旧的、用户全局的 Superpowers worktree 路径不被检测或提供；新的手工 worktree 是项目本地的，除非用户显式指定其他位置。

**安全验证**（仅项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未被忽略，则在继续之前添加到 `.gitignore` 并提交。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**钩子感知：**Git worktree 不继承父仓库的 hooks 目录。通过 1b 创建 worktree 之后，如果存在则从主仓库符号链接 hooks 目录：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这能防止 pre-commit 检查、linter 和其他钩子在 work 移到 worktree 时静默停止。（想法来自 PR #965。）

**沙箱回退：**如果 `git worktree add` 因权限错误失败，按受限环境处理。跳过创建，在当前目录工作，继续到步骤 3。

**步骤编号注记：**当前 skill 把步骤 1-4 作为扁平列表。本次重设计使用 0、0.5、1a、1b、3、4。没有步骤 2——它是旧的"创建隔离工作区"整块，现在拆成 1a/1b 结构。实现应当干净地重新编号（例如 0 → "Step 0: Detect"、0.5 → 在步骤 0 的流程内、1a/1b → "Step 1"、3 → "Step 2"、4 → "Step 3"）或保留当前编号并附注记。由实现者选择。

#### 步骤 3-4：项目设置和基线测试（不变）

无论哪条路径创建了工作区（步骤 0 检测到既有、步骤 1a 原生工具、步骤 1b git 回退，或完全不创建 worktree），执行都会汇聚：

- **步骤 3：**自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **步骤 4：**运行测试套件。如果测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

收尾 skill 获得环境检测并修复三个 bug。

#### 步骤 1：验证测试（不变）

运行项目测试套件。如果测试失败，停止。不提供完成选项。

#### 步骤 1.5：检测环境（新增）

重新运行与创建中步骤 0 相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 需清理 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 标准 4 选项 | 基于来源（见步骤 5） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 缩减菜单：作为新分支推送 + PR、原样保留、丢弃 | 无合并选项（无法从分离 HEAD 合并） |

#### 步骤 2：确定基础分支（不变）

#### 步骤 3：呈现选项

**普通仓库和具名分支 worktree：**

1. 在本地合并回 `<base-branch>`
2. 推送并创建 Pull Request
3. 原样保留分支（我稍后处理）
4. 丢弃此工作

**分离 HEAD：**

1. 作为新分支推送并创建 Pull Request
2. 原样保留（我稍后处理）
3. 丢弃此工作

#### 步骤 4：执行选择

**选项 1（本地合并）：**

```bash
# 获取主仓库根以保证 CWD 安全（Bug #238 修复）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并，在移除任何东西之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# 仅在合并成功之后：移除 worktree，然后删除分支（Bug #999 修复）
git worktree remove "$WORKTREE_PATH"  # 仅当 superpowers 拥有它时
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除 worktree → 删除分支。旧 skill 在移除 worktree 之前删除分支（这会失败，因为 worktree 仍引用该分支）。先移除 worktree 的天真修复同样错误——如果合并随后失败，工作目录已消失，改动丢失。

**选项 2（创建 PR）：**

推送分支，创建 PR。不要清理 worktree——用户在 PR 迭代时需要它。（Bug #940 修复：移除自相矛盾的"Then: Cleanup worktree"叙述。）

**选项 3（原样保留）：**无动作。

**选项 4（丢弃）：**要求键入 "discard" 确认。然后移除 worktree（如果 superpowers 拥有它），强制删除分支。

#### 步骤 5：清理（更新）

```
if GIT_DIR == GIT_COMMON:
    # 普通仓库，无 worktree 需清理
    done

if worktree path is under .worktrees/ or worktrees/:
    # Superpowers 创建了它——我们拥有清理
    cd to main repo root       # Bug #238 修复
    git worktree remove <path>

else:
    # harness 创建了它——别碰
    # 如果平台提供工作区退出工具，使用它
    # 否则，把 worktree 原样保留
```

清理仅对选项 1 和 4 运行。选项 2 和 3 总是保留 worktree。（Bug #940 修复。）

**陈旧 worktree 修剪：**在任意 `git worktree remove` 之后，作为自愈步骤运行 `git worktree prune`。worktree 目录可能被带外删除（例如被 harness 清理、手工 `rm` 或 `.claude/` 清理），留下导致混乱错误的陈旧注册。一行，防止静默腐烂。（想法来自 PR #1072。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者当前都在其集成章节中把 `using-git-worktrees` 列为 REQUIRED。改为：

> `using-git-worktrees` — 确保隔离工作区（创建一个或验证既有的）

该 skill 现在自行处理同意（步骤 0.5）和检测（步骤 0），因此调用方 skill 不需要把关或提示。

#### `writing-plans`

移除陈旧的主张"should be run in a dedicated worktree (created by brainstorming skill)"。Brainstorming 是一个设计 skill，不创建 worktree。worktree 提示发生在执行时，经由 `using-git-worktrees`。

### 4. 平台中立的指令文件引用

worktree 相关 skill 中所有硬编码 `CLAUDE.md` 的实例都被替换为：

> "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"

这适用于步骤 1b 中的目录偏好检查。

## Bug 修复（捆绑）

| Bug | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | 选项 2 叙述说"Then: Cleanup worktree (Step 5)"，但快速参考说保留它。步骤 5 说"For Options 1, 2, 4"，但 Common Mistakes 说"Options 1 and 4 only." | 从选项 2 移除清理。步骤 5 仅适用于选项 1 和 4。 | finishing SKILL.md |
| #999 | 选项 1 在移除 worktree 之前删除分支。`git branch -d` 可能失败，因为 worktree 仍引用该分支。 | 重新排序为：合并 → 验证测试 → 移除 worktree → 删除分支。合并必须在移除任何东西之前成功。 | finishing SKILL.md |
| #238 | 如果 CWD 在被移除的 worktree 内，`git worktree remove` 静默失败。 | 添加 CWD 护栏：在 `git worktree remove` 之前 `cd` 到主仓库根。 | finishing SKILL.md |

## 已解决的问题

| 问题 | 解决 |
|-------|-----------|
| #940 | 直接修复（Bug #940） |
| #991 | 步骤 0.5 中的可选同意 |
| #918 | 步骤 0 检测 + 步骤 1.5 收尾检测 |
| #1009 | 由步骤 1a 解决——agent 使用原生工具（例如 `EnterWorktree`），它们在 harness 原生路径下创建。依赖于步骤 1a 生效；见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立的指令文件引用 |
| #279 | 由检测并让位解决——原生路径被尊重，因为我们不覆盖它们 |
| #574 | **推迟。**本 spec 中没有任何东西触及 bug 所在的 brainstorming skill。完整修复（向 brainstorming 清单添加一个 worktree 步骤）是 Phase 4。 |

## 风险

### 步骤 1a 是承载假设——已解决

步骤 1a——agent 优先使用原生 worktree 工具而非 git 回退——是整个设计所依托的基础。如果 agent 在有原生支持的 harness 上忽略步骤 1a 并落入步骤 1b，检测并让位就完全失败。

**状态：**该风险在实现期间兑现。原始的抽象步骤 1a（"You know your own toolkit"）在 Claude Code 上以 2/6 失败。TDD 门槛按设计工作——它在任何 skill 文件被修改之前捕获了失败，防止了一次破损发布。三次 REFACTOR 迭代识别了根本原因（agent 锚定在具体命令上、工具描述护栏覆盖 skill 指令），并产生了一个在 GREEN 和 PRESSURE 测试上以 50/50 验证的修复。详见上方步骤 1a 设计注记。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一具有 agent 可在会话中途调用的 worktree 工具（`EnterWorktree`）的 harness。其余要么在 agent 启动之前创建 worktree（Codex App、Gemini CLI、Cursor），要么没有原生 worktree 支持（Codex CLI、OpenCode）。步骤 1a 是前向兼容的：当其他 harness 添加 agent 可调用的 worktree 工具时，agent 会把它们与具名示例匹配并使用，无需 skill 改动。

| Harness | 当前 worktree 模型 | skill 机制 | 已测试 |
|---------|----------------------|-----------------|--------|
| Claude Code | Agent 可调用 `EnterWorktree` | 步骤 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无原生工具（仅 shell） | 步骤 1b git 回退 | 6/6（`codex exec`） |
| Gemini CLI | 启动时 `--worktree` 标志，无 agent 工具 | 用标志启动走步骤 0，否则步骤 1b | 步骤 0：1/1，步骤 1b：1/1（`gemini -p`） |
| Cursor Agent | 面向用户的 `/worktree`，无 agent 工具 | 用户激活走步骤 0，否则步骤 1b | 步骤 0：1/1，步骤 1b：1/1（`cursor-agent -p`） |
| Codex App | 平台管理，分离 HEAD，无 agent 工具 | 步骤 0 检测既有 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无 agent 工具 | 步骤 1b git 回退 | 未测（无 CLI 访问） |

**残留风险：**
1. 如果 Anthropic 把 `EnterWorktree` 的工具描述改得更受限（例如"Do not use based on skill instructions"），同意桥接就会失效。值得提交一个 issue 请求让该工具描述容纳 skill 驱动的调用。
2. 当其他 harness 添加 agent 可调用的 worktree 工具时，它们可能使用不在步骤 1a 清单中的名称。随着新工具出现应当更新清单。泛化措辞（"a worktree or workspace-isolation tool"）提供一定前向覆盖。

### 来源启发式

`.worktrees/` 或 `worktrees/` = 我们的，其他任何位置 = 别碰这一启发式对每个当前 harness 都有效。如果未来某 harness 采用这些项目本地目录之一作为其约定，我们会有误报（superpowers 试图清理一个 harness 拥有的 worktree）。类似地，如果用户在没有 superpowers 的情况下手工运行 `git worktree add .worktrees/experiment`，我们会错误地声称所有权。两者都是低风险——每个 harness 都使用品牌化路径，而手工 `.worktrees/` 创建不太可能——但值得注意。

### 分离 HEAD 收尾

为分离 HEAD worktree 提供的缩减菜单（无合并选项）对 Codex App 的沙箱模型是正确的。如果用户因其他原因处于分离 HEAD，缩减菜单仍然合理——你确实无法在不先创建分支的情况下从分离 HEAD 合并。

## 实现注记

两个 skill 文件都包含核心步骤之外、需要在实现期间更新的章节：

- **Frontmatter**（`name`、`description`）：更新以反映检测并让位行为
- **快速参考表**：重写以匹配新步骤结构和 bug 修复
- **Common Mistakes 章节**：更新或移除引用旧行为的条目（例如"Skip CLAUDE.md check"现在是错的）
- **Red Flags 章节**：更新以反映新优先级（例如"Never create a worktree when Step 0 detects existing isolation"）
- **Integration 章节**：更新 skill 之间的交叉引用

本 spec 描述*改动什么*；实现计划将具体规定对这些次要章节的精确编辑。

## 未来工作（不在本 spec 内）

- **Phase 3 剩余：**`$TMPDIR` 目录选项（#666）、缓存和环境继承的设置文档（#299）
- **Phase 4：**用于路径强制的 PreToolUse 钩子（#1040）、每 worktree 环境约定（#597）、brainstorming 清单 worktree 步骤（#574）、多仓库文档（#710）
