# Superpowers 发布说明 (Superpowers Release Notes)

## v6.0.3 (2026-06-18)

### Subagent-Driven Development（subagent 驱动开发）

- **SDD 临时文件移出了 `.git/`。** Claude Code 把 `.git/` 当作受保护路径并拒绝 agent 在那里写入，因此一个实现者 subagent 在把报告写入 `.git/sdd/` 时会在运行中途被拦截。任务简报、实现者报告、审查 diff 和进度账本现在位于工作树中一个会自动忽略自身的 `.superpowers/sdd/` 目录中——既不出现在 `git status` 中，也不进入提交，并由一个共享的 `sdd-workspace` 辅助函数按 worktree 解析。一个注意事项：由于该工作区是被 git 忽略的工作树临时区，`git clean -fdx` 会删除进度账本；若发生这种情况，从 `git log` 恢复。(#1780)

## v6.0.2 (2026-06-16)

### 安装修复

- **我们不再发布 `evals` 子模块。** 它破坏了一些用户的插件安装，所以 eval 测试驱动程序现在位于自己的仓库中，与已发布的插件分开。(#1778, #1774)

## v6.0.1 (2026-06-16)

### Codex 修复

- **brainstorm 可视伴侣中的版本显示** —— 打包的 Codex 插件发布时没有根 `package.json`，所以可视化伴侣把版本报告为 "unknown"。`readSuperpowersVersion()` 现在在 `package.json` 缺失时回退到 `.codex-plugin/plugin.json`。
- **更干净的 Codex 插件同步** —— sync-to-codex 脚本现在排除了 `.gitmodules` 和 `.pre-commit-config.yaml`，使仓库元数据不会进入打包的 Codex 插件。

## v6.0.0 (2026-06-16)

Superpowers 6.0 是一个大版本。头条新闻是重写了 `subagent-driven-development` 审查每个任务的方式——更便宜、更严格、更难钻空子。

虽然这些数字不会在每个 harness 和每个工作负载上都成立，但在我们的 eval 中，Claude Code 和 Codex 产生类似高质量结果的速度大约快两倍，同时 token 花费减少了近 50%。

它还添加了三个新 harness（Kimi Code、Pi 和 Antigravity），给 brainstorming 可视伴侣一个更好的安全模型，并重写了若干 skill 的工具调用，使其显著更具厂商中立性。

### 可见的变化

- **每个任务的两个审查者 prompt 合并为一个。** `spec-reviewer-prompt.md` 和 `code-quality-reviewer-prompt.md` 已移除，替换为单个 `task-reviewer-prompt.md`。如果你直接派发旧文件，请切换到新的。
- **遗留的全局 worktree 目录已移除。** `using-git-worktrees` 和 `finishing-a-development-branch` 不再使用 `~/.config/superpowers/worktrees/`。worktree 现在落在项目里——你已有的 `.worktrees/` 或 `worktrees/`，否则是一个全新的 `.worktrees/`——除非你另作说明。

### 新 Harness 支持

Superpowers 现在在另外三个 harness 上运行。每个都有自己的 bootstrap、一个工具映射参考和测试，并在 README 中有各自的安装小节。

- **Kimi Code** —— 一个插件 manifest、安装文档和 manifest 测试；从 Kimi 的市场或直接从仓库安装。（初始 manifest 由 @qer 提供）
- **Pi** —— 一个会话开始扩展，注册 skills 并注入 `using-superpowers` bootstrap。Pi 有原生 skills，所以它不需要兼容垫片。
- **Antigravity（`agy`）** —— 直接安装插件并从第一条消息开始 bootstrap；已针对标准的 "make a react todo list" 验收测试端到端验证。

### Subagent-Driven Development（subagent 驱动开发）

在真实项目上进行了长期成本与质量实验，重塑了控制器如何审查每个任务。旧流程对每个任务运行两个审查者，并在模型选择和严重性判断上依赖控制器的判断，结果两者都代价高昂且容易钻空子。新流程对每个任务运行一个审查者，以文件而非粘贴文本的方式交接工作，并从控制器手中夺走若干判断决策。

- **每个任务一个审查者，两个裁决。** 单个 `task-reviewer-prompt.md` 一次性读取任务的 diff 并返回一个 spec 合规裁决和一个质量裁决，所以一次修复就能同时清掉两者。一个新的 "can't verify from the diff"（无法从 diff 验证）裁决标记出位于未被触碰代码中的需求，留给控制器自己检查。(#1538, #1543)
- **结尾一次宽泛审查。** 运行在最强模型上对整个分支做一次审查结束，而不是逐任务地重新审查一切。
- **计划获得一次起飞前阅读。** 在第一个任务之前，控制器检查计划是否存在内部冲突——以及计划所要求的、会被审查者标记为缺陷的任何东西——并一次性全部提出，而不是在运行中途撞上。
- **Diff 和任务文本以文件形式流转。** 一个粘贴的 diff 永久占据最昂贵的上下文，而一个没有 diff 的审查者会手工重建它——这是最大的审查者成本。两个新脚本，`task-brief` 和 `review-package`，把任务文本和审查 diff 写到文件中供 subagent 读取。
- **每次派发都声明其模型。** 任由选择时，控制器完全不再命名模型——而一个未命名的模型会悄悄继承会话中最贵的那个，所以有一次运行把全部 26 个审查者都放在了最高档。模板现在要求指定模型，并带有在条件允许时伸手拿更便宜档次的指导。
- **控制器不能告诉审查者忽略什么。** 真实运行中抓到过控制器指导审查者跳过某个发现或称之为 "Minor at most"，结果缺陷被发布。压制发现和预先评级严重性现在被彻底禁止，而一个计划本身要求的缺陷会被报告给你来决定，而非被放行。
- **审查者是只读的，并对合理化说辞持怀疑态度。** 审查不再触碰工作树或分支——一个运行 `git checkout` 的审查者曾让后续提交变成孤儿——而实现者的 "I left this unabstracted on purpose"（我故意没抽象它）不再能说服审查者放弃一个真实发现。
- **更强的证据和报告。** 审查者为每个答案附上文件和行号，实现者的报告移到文件中并在 TDD 适用时携带红/绿证据，而一个进度账本让丢失上下文的控制器得以恢复，而非重做已完成工作。(#994)

### 编写计划

计划现在携带控制器和审查者过去在每次派发时都要重新推导的结构。

- **一个全局约束块** 列出约束每个任务的规则——版本下限、依赖限制、命名和文案、精确值——逐字复制进来，这样它们才真正到达下游的实现者和审查者。
- **一个按任务的接口块** 精确命名每个任务消费和产出什么，这样只看到自己任务的实现者仍知道邻居们的契约。
- **大小合理的指导** 把一个任务保持在能获得自己测试周期和审查者通过的大小，把设置、配置和文档折叠到需要它们的任务中。在测试中，这样编写的计划需要一轮修复，而对照组需要两到四轮——而且对照组发布了一个真实 bug。

### Brainstorming 可视伴侣

可视伴侣是 agent 在会话旁边打开的一个小型 Web 服务器。它此前完全没有任何认证，所以在共享或远程机器上，任何能到达该端口的人都能读取你的 brainstorm——或注入 agent 当作你输入的事件。本次发布给它一个真正的安全模型，并让它能在重启和断连后存活。

- **一个每会话密钥现在守护一切。** agent 的 URL 携带一个一次性密钥，浏览器把它塞进一个 tab 作用域的 cookie，每个请求和 WebSocket 连接都必须出示它。这关闭了通往零散本地 tab 和可路由远程主机的门，包括源 allowlist 捕捉不到的 DNS 重绑定情形。（关闭 #1014）
- **文件服务器留在其 sandbox 内。** 它拒绝符号链接、dotfile 和任何爬出内容目录的路径，忽略 macOS 的 resource-fork 文件，并发送常规的 no-store 和 deny-framing 头部。持有会话密钥的文件以仅所有者权限写入。
- **该伴侣只在有帮助时才被提出。** skill 在第一次出现某个问题用展示比讲述更好时，作为自己的消息提出它，并允许拒绝成立。接受会打开你的浏览器到第一个屏幕。（关闭 #755）
- **它在重启和断连后存活。** 给定一个项目目录，服务器在重启间保持相同的端口和密钥，所以一个打开的 tab 简单地重新连接。页面自行重连，显示一个实时状态徽标，并在服务器宕机时升起一个 "paused" 覆盖层。
- **更长的空闲寿命、更安全的关闭。** 空闲超时从 30 分钟延长到 4 小时，`stop-server.sh` 现在在发信号前确认它拥有正确的进程，所以它在重启后绝不会杀掉一个无关的 `node`。(#1703)
- **Windows 启动加固** —— 整合了 shell 检测，Windows 现在依赖空闲超时来关闭，因为 Node 无法跨 MSYS2 追踪 POSIX 进程所有权。


### 现有 Harness 更新

- **Codex** 现在通过其自己的 SessionStart hook 进行 bootstrap，而非共享的接线，Codex App 增加了一个安装小节和更完整的工具文档（网络搜索、`AGENTS.md`、个人 skills）。(#1540)
- **OpenCode** 在其插件、安装文档和 README 中获得了基于动作的工具映射，外加一个 bootstrap 缓存测试。
- **Cursor** 的 manifest 删除了其 `agents` 和 `commands` 条目，因为这些目录已不再存在。

### 一套 Skills，每个 Harness

skills 过去说的是 Claude Code 的方言——"use the Task tool"、"put it in CLAUDE.md"。本次发布用你实际在做的事情（"派发一个 subagent"、"你的指令文件"）重写了那套词汇，并增加了一个每个 harness 的参考，把每个动作映射到正确的工具，并针对每个运行时校验。命名 "Claude" 的散文现在说 "your agent"。

- **每个 harness 一个工具参考**，位于 `skills/using-superpowers/references/`，覆盖 Claude Code、Codex、Copilot、Gemini、Pi 和 Antigravity。
- **`finishing-a-development-branch` 变成 forge 中立** —— 它不再硬编码 `gh pr create`，所以 agent 用它们手头的任何 forge 工具来 push。(#1609)
- **一次重命名：** "Claude Search Optimization" 现在是 "Skill Discovery Optimization"，因为该技术并非 Claude 专用。

### 编写 Skills

为 skill 作者新增两项。

- **匹配形态与失败** —— 一张用于挑选正确指导类型的小表。一句扁平的 "don't do X"（不要做 X）适用于纪律性失误，但当问题是一个输出的*形态*时就会适得其反，此时一个示例效果更好。这张表，以及对现有合理化小节更紧的范围控制，引导作者找到真正有效的形态。
- **Micro-Test 措辞** —— 在确定某个措辞前廉价检查它的方式：对着无指导对照组采样几次并逐一手动阅读每个结果，把运行间的方差当作警告信号。

### 测试

Skill 行为测试从 `tests/` 迁移到一个基于 "drill" 构建的新 `evals/` 子模块，它运行真实的 Claude Code、Codex 和 Gemini 会话并用一个 LLM 来评判。若干树内 bash 套件在更严格的 drill 场景覆盖后退役；少数没有等价物的保留了下来。从此处起，`tests/` 存放插件代码测试，`evals/` 存放 skill 行为测试，`docs/testing.md` 解释了这一拆分。新后端覆盖 Antigravity、Pi 和更多模型，新的 shell-lint 和 pre-commit 检查守护 harness。(#1541)

### 缺陷修复

- **systematic-debugging 不再强制每个会话进入扩展思考。** 一句要点恰好持有 Claude Code 扫描的那个精确关键字，悄悄地在每个加载了该 skill 的会话上触发了开关。一个连字符破坏了关键字；文字依然读得通。(#1283，由 @Nick Galatis 提供)
- **Windows 的 SessionStart hook 停止每个会话打印一次写入错误** —— 每个 `printf` 现在通过 `cat` 路由来吸收 broken pipe，输出在其他方面不变。(#1612，由 @silvertakana 报告)
- **Windows 前台模式** 追踪正确的进程并在 MSYS2 上清除其 owner PID。（由 @nestorluiscamachopaz 提供）
- **`using-superpowers` bootstrap** 不再把 "debugging" 列为一个不存在的 skill。（由 @mhat 报告）
- **TDD skill** 链接了测试反模式参考。(#1532, #1529；链接修复 #1474 由 @Stable Genius 提供)
- **`using-git-worktrees`** 修复了其步骤编号并删除了过时的 Cursor 引用。(#1522，以及由 @fuleinist 提供)
- **Codex 审查 skill** 把一段内部玩笑换成直白指导。(#1531)

### 文档与贡献者指南

- **一份将 Superpowers 移植到新 harness 的指南**（`docs/porting-to-a-new-harness.md`）列出了每个集成所需的三个部分以及决定成败的那条规则：在会话开始时加载 bootstrap。
- **每个 PR 和 issue 现在都披露它是如何制作的** —— 模型、harness、版本和已安装的插件，或一段说明它是手工书写的备注。我们根据它由什么产生来不同地权衡一项贡献。PR 也目标 `dev`，而非 `main`。PR 模板、全部三个 issue 模板，以及一个新的平台支持模板都承载了这一点。

### 贡献者

感谢 @mattvanhorn、@nawfal、@Nick Galatis、@silvertakana、@nestorluiscamachopaz、@qer、@mhat、@Stable Genius、@fuleinist、@dev_Hakaze、@robotsnh、Rahul 和 @arittr。

## v5.1.0 (2026-04-30)

### 移除项

- **遗留斜杠命令已移除** —— `/brainstorm`、`/execute-plan` 和 `/write-plan` 已移除。它们是已弃用的桩，除了告诉用户调用对应 skill 之外什么也不做。请改为直接调用 `superpowers:brainstorming`、`superpowers:executing-plans` 和 `superpowers:writing-plans`。(#1188)
- **`superpowers:code-reviewer` 命名 agent 已移除** —— 该 agent 是插件的唯一命名 agent，仅被两个 skill 使用，而仓库中所有其他审查者/实现者 subagent 都是带着一个 prompt 模板派发 `general-purpose` 外加其 skill。该 agent 的人格和检查清单已作为自包含的 Task 派发模板合并进 `skills/requesting-code-review/code-reviewer.md`。任何派发 `Task (superpowers:code-reviewer)` 的人都应改为用 prompt 模板派发 `Task (general-purpose)`。(PR #1299)
- **从 skills 中移除了集成小节** —— 它们是 agent 还没有原生 skills 系统那个时代的遗物，无助于引导。

### Worktree Skills 重写

`using-git-worktrees` 和 `finishing-a-development-branch` 现在检测 agent 是否已经在隔离的 worktree 内运行，并在回退到 `git worktree` 之前优先使用 harness 原生的 worktree 控制。行为已通过 TDD 验证并在五个 harness 上做了跨平台检查。(PRI-974, PR #1121)

- **环境检测** —— 两个 skill 在做任何事之前都检查 `GIT_DIR != GIT_COMMON`；如果已在一个链接的 worktree 中，则完全跳过创建。一个子模块守卫防止误检测。
- **创建 worktree 之前征得同意** —— `using-git-worktrees` 不再隐式创建 worktree；该 skill 先询问用户。修复 #991（subagent-driven-development 在未获同意时自动创建 worktree）。
- **原生工具优先（步骤 1a）** —— 当 harness 暴露其自己的 worktree 工具（例如 Codex）时，该 skill 让位于它。用户表达的偏好在被表达时得到尊重。
- **基于来源的清理** —— `finishing-a-development-branch` 只清理 `.worktrees/` 内的 worktree（由 superpowers 创建）；其外的任何东西保持不动。修复 #940（Option 2 错误地清理了 worktree）、#999（合并后移除的顺序）和 #238（`git worktree remove` 之前 `cd` 到仓库根）。
- **分离 HEAD 处理** —— 当没有可合并的分支时，完成菜单折叠为两个选项。
- **skill 示例中硬编码的 `/Users/jesse` 路径**替换为通用占位符。(#858, PR #1122)

### 面向 AI Agent 的贡献者指南

`CLAUDE.md`（符号链接到 `AGENTS.md`）顶部新增两节直接对 AI agent 说话。对本仓库最近 100 个已关闭 PR 的审计显示 94% 的拒绝率，由 AI 生成的垃圾内容驱动：agent 没读 PR 模板、开重复 PR、捏造问题描述，或把 fork 专用或领域专用的改动推到上游。

- **提交前检查清单** —— 阅读 PR 模板、搜索现有 PR、验证真实问题存在、确认改动属于核心，并在提交前向人类伙伴展示完整 diff。
- **我们不会接受的** —— 第三方依赖、对 skill 内容的"合规"重写、项目专用配置、批量 PR、投机性修复、领域专用 skill、fork 专用改动、捏造内容，以及捆绑的不相关改动。
- **新 harness PR 要求一份会话记录** —— 大多数过去的新 harness 集成是复制 skill 文件或用 `npx skills` 包装，而非在会话开始时加载 `using-superpowers` bootstrap。验收测试（"Let's make a react todo list" 必须在干净会话中自动触发 `brainstorming`）和一份完整会话记录现在成为必需。

### Codex 插件镜像工具

新的 `sync-to-codex-plugin` 脚本把 superpowers 镜像到 OpenAI Codex 插件市场，名为 `prime-radiant-inc/openai-codex-plugins`。与路径/用户无关，所以任何团队成员都能运行它。(PR #1165)

- 每次运行把 fork 全新克隆到临时目录，内联重新生成覆盖层，并开一个 PR；从脚本自身位置自动检测上游，并对 `rsync`/`git`/`gh auth`/`python3` 做预检。
- `--bootstrap` 标志用于首次设置；`EXCLUDES` 模式锚定到源根；`assets/` 被排除。
- 镜像 `CODE_OF_CONDUCT.md`；丢弃 `agents/openai.yaml` 覆盖层。
- 在镜像的 `plugin.json` 中播种 `interface.defaultPrompt`。(PR #1180 由 @arittr 提供)
- Codex 插件文件被提交到源仓库，这样同步脚本使用规范版本；Codex 市场元数据被保留。

### OpenCode

- **Bootstrap 内容缓存在模块级** —— `getBootstrapContent()` 此前在每个 agent 步骤都调用 `fs.existsSync` + `fs.readFileSync` + frontmatter 正则（`experimental.chat.messages.transform` hook 在 OpenCode 的 agent 循环中每步触发）。现在读取一次，在会话生命周期内缓存，并对文件缺失情况用一个 null 哨兵。15 个回归测试覆盖缓存行为、fs 调用计数、注入守卫、文件缺失哨兵和缓存重置。(修复 #1202)
- **集成测试现代化**。
- **README 中澄清了安装注意事项**。

### 代码审查整合

`requesting-code-review` 现在是自包含的：人格、检查清单和派发模板位于 `skills/requesting-code-review/code-reviewer.md`，skill 直接派发 `Task (general-purpose)`。(PR #1299)

- **单一事实来源** —— 此前同时存在于 `agents/code-reviewer.md` 和 skill 占位模板中（且独立漂移）的人格/检查清单，现在是一个文件。
- **`subagent-driven-development` 跟进** —— 它的 `code-quality-reviewer-prompt.md` 现在派发 `Task (general-purpose)` 而非命名 agent。
- **添加了行为测试** —— `tests/claude-code/test-requesting-code-review.sh` 在一个微型项目中植入真实 bug（SQL 注入、明文口令处理、凭据日志），并断言被派发的审查者把每个植入问题标记为 Critical/Important 严重性并拒绝批准 diff。

> 注意：`tests/claude-code/test-requesting-code-review.sh` 和 `tests/claude-code/test-document-review-system.sh`（在后文提及）于 2026-05-06 被提升为 drill 场景并从 `tests/` 移除。见 `evals/scenarios/code-review-catches-planted-bugs.yaml` 和 `evals/scenarios/spec-reviewer-catches-planted-flaws.yaml`。上面和下面的引用作为本节所述工作的注明工件保留。
- **Codex 和 Copilot 变通文档精简** —— `references/codex-tools.md` 和 `references/copilot-tools.md` 中的 "命名 agent 派发" 小节记录了如何把命名 agent 展平为通用派发。由于不再发布命名 agent，该变通方案已无必要；两节都被删除。

### Subagent-Driven Development（subagent 驱动开发）

- **不再每 3 个任务暂停一次** —— `requesting-code-review` 中（最初为 `executing-plans`）的"每批（3 个任务）后审查"节奏泄漏进了 `subagent-driven-development`。替换为"每个任务或在自然检查点"外加一条明确的连续执行指令。
- **SDD 集成测试现在真正运行其断言** —— 三个独立的 bug 导致该测试在打印任何验证结果前静默退出：工作目录路径中一个未解析的 `..` 段、`set -euo pipefail` 与 `find | sort | head -1` 的交互（生产者上的 SIGPIPE 杀死了脚本），以及 `claude -p` 调用上缺失的 `--plugin-dir`（导致测试加载已安装插件而非工作树）。三者全部修复；六个验证测试现在真正针对一次端到端 SDD 运行运行。

### Cursor

- **Windows 的 SessionStart hook** 通过 `run-hook.cmd` 路由，而非直接调用无扩展名 `session-start` 脚本。修复 Windows 用编辑器打开文件而非运行它的问题。还移除了 `hooks-cursor.json` 中一个意外的 UTF-8 BOM。

### Gemini CLI

- **Subagent 派发映射** —— Gemini 的 `Task` 派发现在映射到 `@agent-name` / `@generalist`，并为独立任务记录了并行 subagent 派发。

### Skills

- **跨 skill 内容的术语清理**。

### 文档与安装

- **Factory Droid 安装说明**加入 README。
- **README 中的快速开始安装链接**。(PR #1293 由 @arittr 提供)
- **Codex 插件安装指南更新**。(PR #1288 由 @arittr 提供)
- **Codex `wait` 映射更正**为工具参考中的 `wait_agent`。
- **安装顺序重组**；Codex 安装说明清理。
- **移除了残留的 `CHANGELOG.md`**，改用 `RELEASE-NOTES.md` 作为单一来源。(PR #1163 由 @shaanmajid 提供)
- **Discord 邀请链接修复**；发布宣告链接和详细的 Discord 描述加入 Community 小节。

### 社区

- @shaanmajid —— 残留 `CHANGELOG.md` 移除 (PR #1163)
- @arittr —— README 快速开始安装链接 (#1293)、Codex 插件安装指南 (#1288)、`sync-to-codex-plugin` `interface.defaultPrompt` 播种 (#1180)

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 支持

- **SessionStart 上下文注入** —— Copilot CLI v1.0.11 在 sessionStart hook 输出中增加了对 `additionalContext` 的支持。session-start hook 现在检测 `COPILOT_CLI` 环境变量并发出 SDK 标准的 `{ "additionalContext": "..." }` 格式，给 Copilot CLI 用户在会话开始时完整的 superpowers bootstrap。（原始修复由 @culinablaz 在 PR #910 提供）
- **工具映射** —— 添加了 `references/copilot-tools.md`，包含完整的 Claude Code 到 Copilot CLI 工具等价表
- **Skill 和 README 更新** —— 把 Copilot CLI 加入 `using-superpowers` skill 的平台说明和 README 安装小节

### OpenCode 修复

- **Skills 路径一致性** —— bootstrap 文本不再宣传一个与运行时路径不符的、具有误导性的 `configDir/skills/superpowers/` 路径。agent 应使用原生的 `skill` 工具，而非按路径导航到文件。测试现在使用从一个单一事实来源派生的一致路径。(#847, #916)
- **Bootstrap 作为 user 消息** —— 把 bootstrap 注入从 `experimental.chat.system.transform` 移到 `experimental.chat.messages.transform`，前置到第一条 user 消息而非添加一条 system 消息。避免每轮重复的 system 消息导致的 token 膨胀 (#750)，并修复与 Qwen 等在多条 system 消息上崩溃的模型的兼容性 (#894)。

## v5.0.6 (2026-03-24)

### 内联自我审查取代 Subagent 审查循环

subagent 审查循环（派发一个全新 agent 来审查 plan/spec）使执行时间翻倍（约 25 分钟开销），却没有可衡量地改善 plan 质量。跨 5 个版本的回归测试（每个 5 次试验）显示，无论审查循环是否运行，质量分数都相同。

- **brainstorming** —— 用内联 Spec 自我审查清单替换了 Spec 审查循环（subagent 派发 + 3 次迭代上限）：占位符扫描、内部一致性、范围检查、歧义检查
- **writing-plans** —— 用内联自我审查清单替换了 Plan 审查循环（subagent 派发 + 3 次迭代上限）：spec 覆盖、占位符扫描、类型一致性
- **writing-plans** —— 增加了明确的 "No Placeholders"（无占位符）小节，定义计划失败（TBD、模糊描述、未定义引用、"similar to Task N"）
- 自我审查在约 30 秒内每轮捕获 3-5 个真实 bug，而非约 25 分钟，缺陷率与 subagent 方式相当

### Brainstorm Server

- **会话目录重构** —— brainstorm server 会话目录现在包含两个对等子目录：`content/`（提供给浏览器的 HTML 文件）和 `state/`（事件、server-info、pid、log）。此前，服务器状态和用户交互数据与被服务内容并排存储，使其可通过 HTTP 访问。`screen_dir` 和 `state_dir` 路径都包含在 server-started JSON 中。（由 吉田仁 报告）

### 缺陷修复

- **Owner-PID 生命周期修复** —— brainstorm server 的 owner-PID 监控有两个 bug，导致 60 秒内误关机：(1) 跨用户 PID（Tailscale SSH 等）产生的 EPERM 被当作"进程已死"，(2) 在 WSL 上祖父 PID 解析为一个在第一次生命周期检查前就退出的短命子进程。通过把 EPERM 当作"活着"并在启动时验证 owner PID 来修复——如果它已经死了，则禁用监控，服务器依赖 30 分钟空闲超时。这也从 `start-server.sh` 中移除了 Windows/MSYS2 专用的特殊处理，因为服务器现在通用地处理它。(#879)
- **writing-skills** —— 纠正了 SKILL.md frontmatter 只支持"两个字段"的错误说法；现在说"两个必需字段"，并链接到 agentskills.io 规范以获取所有支持字段 (PR #882 由 @arittr 提供)

### Codex App 兼容性

- **codex-tools** —— 添加了命名 agent 派发映射，记录如何把 Claude Code 的命名 agent 类型翻译为带 worker 角色的 Codex `spawn_agent` (PR #647 由 @arittr 提供)
- **codex-tools** —— 为 worktree 感知 skill 添加了环境检测和 Codex App 完成小节（由 @arittr 提供）
- **设计 spec** —— 添加了 Codex App 兼容性设计 spec (PRI-823)，覆盖只读环境检测、worktree 安全的 skill 行为和 sandbox 后备模式（由 @arittr 提供）

## v5.0.5 (2026-03-17)

### 缺陷修复

- **Brainstorm server ESM 修复** —— 把 `server.js` 重命名为 `server.cjs`，使 brainstorming server 在 Node.js 22+ 上正确启动，那里根 `package.json` 的 `"type": "module"` 导致 `require()` 失败。(PR #784 由 @sarbojitrana，修复 #774、#780、#783)
- **Brainstorm owner-PID 在 Windows 上** —— 在 Windows/MSYS2 上跳过 PID 生命周期监控，那里 PID 命名空间对 Node.js 不可见，防止服务器在 60 秒后自终止。(#770，文档来自 PR #768 由 @lucasyhzlu-debug 提供)
- **stop-server.sh 可靠性** —— 在报告成功前验证服务器进程确实死了。SIGTERM + 2 秒等待 + SIGKILL 后备。(#723)

### 变更

- **执行交接** —— 在写完计划后，恢复用户在 subagent 驱动和内联执行之间的选择。推荐 subagent 驱动但不再强制。

## v5.0.4 (2026-03-16)

### 审查循环精炼

通过消除不必要的审查遍次和收紧审查者焦点，大幅降低 token 使用并加快 spec 和 plan 审查。

- **单次全计划审查** —— plan 审查者现在一次性审查完整计划，而非逐块。移除了所有块相关概念（`## Chunk N:` 标题、1000 行块限制、按块派发）。
- **提高了阻塞问题的门槛** —— spec 和 plan 审查者 prompt 现在都包含一个 "Calibration"（校准）小节：只标记会在实现期间造成真实问题的问题。次要的措辞、风格偏好和格式挑剔不应阻塞批准。
- **减少最大审查迭代次数** —— spec 和 plan 审查循环都从 5 次降到 3 次。如果审查者被正确校准，3 轮就足够了。
- **精简审查者检查清单** —— spec 审查者从 7 类精简到 5 类；plan 审查者从 7 类精简到 4 类。移除聚焦格式的检查（任务语法、块大小），转向实质（可构建性、spec 对齐）。

### OpenCode

- **一行插件安装** —— OpenCode 插件现在通过一个 `config` hook 自动注册 skills 目录。无需符号链接或 `skills.paths` 配置。安装只是向 `opencode.json` 添加一行。(PR #753)
- **添加了 `package.json`**，使 OpenCode 能从 git 把 superpowers 作为 npm 包安装。

### 缺陷修复

- **验证服务器确实停止了** —— `stop-server.sh` 现在在报告成功前确认进程已死。SIGTERM + 2 秒等待 + SIGKILL 后备。如果进程存活则报告失败。(PR #751)
- **通用 agent 措辞** —— brainstorm 伴侣等待页现在说 "the agent" 而非 "Claude"。

## v5.0.3 (2026-03-15)

### Cursor 支持

- **Cursor hooks** —— 添加了 `hooks/hooks-cursor.json`，使用 Cursor 的 camelCase 格式（`sessionStart`、`version: 1`），并更新了 `.cursor-plugin/plugin.json` 以引用它。修复了 `session-start` 中的平台检测，先检查 `CURSOR_PLUGIN_ROOT`（Cursor 也可能设置 `CLAUDE_PLUGIN_ROOT`）。(基于 PR #709)

### 缺陷修复

- **停止在 `--resume` 时触发 SessionStart hook** —— 启动 hook 此前在恢复的会话上重新注入上下文，而这些会话已经在对话历史中拥有该上下文。hook 现在只在 `startup`、`clear` 和 `compact` 上触发。
- **Bash 5.3+ hook 挂起** —— 用 `printf` 替换了 `hooks/session-start` 中的 heredoc（`cat <<EOF`）。修复了 macOS 上 Homebrew bash 5.3+ 因 heredoc 中大变量展开的 bash 回归导致的无限挂起。(#572, #571)
- **POSIX 安全的 hook 脚本** —— 在 `hooks/session-start` 中用 `$0` 替换了 `${BASH_SOURCE[0]:-$0}`。修复了 Ubuntu/Debian 上 `/bin/sh` 为 dash 时的 "Bad substitution" 错误。(#553)
- **可移植的 shebang** —— 在所有 shell 脚本中用 `#!/usr/bin/env bash` 替换了 `#!/bin/bash`。修复了 `/bin/bash` 过时或缺失时在 NixOS、FreeBSD 和 macOS 上使用 Homebrew bash 的执行问题。(#700)
- **Windows 上的 Brainstorm server** —— 自动检测 Windows/Git Bash（`OSTYPE=msys*`、`MSYSTEM`）并切换到前台模式，修复了 `nohup`/`disown` 进程回收导致的静默服务器失败。(#737)
- **Codex 文档修复** —— 在 Codex 文档中用已弃用的 `collab` 标志替换为 `multi_agent`。(PR #749)

## v5.0.2 (2026-03-11)

### 零依赖 Brainstorm Server

**移除了所有 vendored 的 node_modules —— server.js 现在完全自包含**

- 用使用内建 `http`、`fs` 和 `crypto` 模块的零依赖 Node.js 服务器替换了 Express/Chokidar/WebSocket 依赖
- 移除了约 1,200 行 vendored 的 `node_modules/`、`package.json` 和 `package-lock.json`
- 自定义 WebSocket 协议实现（RFC 6455 帧、ping/pong、正确的关闭握手）
- 原生 `fs.watch()` 文件监视取代 Chokidar
- 完整测试套件：HTTP 服务、WebSocket 协议、文件监视和集成测试

### Brainstorm Server 可靠性

- **空闲 30 分钟后自动退出** —— 服务器在没有客户端连接时关闭，防止孤儿进程
- **Owner 进程追踪** —— 服务器监控父 harness PID，并在拥有它的会话死亡时退出
- **存活检查** —— skill 在复用一个现有实例前验证服务器有响应
- **编码修复** —— 被服务的 HTML 页面上有正确的 `<meta charset="utf-8">`

### Subagent 上下文隔离

- 所有委派 skill（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）现在都包含上下文隔离原则
- subagent 只接收它们需要的上下文，防止上下文窗口污染

## v5.0.1 (2026-03-10)

### Agentskills 合规

**Brainstorm-server 移入 skill 目录**

- 按 [agentskills.io](https://agentskills.io) 规范，把 `lib/brainstorm-server/` 移到 `skills/brainstorming/scripts/`
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 引用替换为相对的 `scripts/` 路径
- Skills 现在完全跨平台可移植——无需平台专用环境变量来定位脚本
- `lib/` 目录被移除（是最后剩余的内容）

### 新特性

**Gemini CLI 扩展**

- 通过仓库根的 `gemini-extension.json` 和 `GEMINI.md` 提供原生 Gemini CLI 扩展支持
- `GEMINI.md` 在会话开始时 @import `using-superpowers` skill 和工具映射表
- Gemini CLI 工具映射参考（`skills/using-superpowers/references/gemini-tools.md`）——把 Claude Code 工具名（Read、Write、Edit、Bash 等）翻译为 Gemini CLI 等价物（read_file、write_file、replace 等）
- 记录 Gemini CLI 限制：无 subagent 支持，skill 回退到 `executing-plans`
- 扩展根位于仓库根，以实现跨平台兼容（避免 Windows 符号链接问题）
- 安装说明加入 README

### 改进

**多平台 brainstorm server 启动**

- visual-companion.md 中有按平台的启动说明：Claude Code（默认模式）、Codex（通过 `CODEX_CI` 自动前台）、Gemini CLI（带 `is_background` 的 `--foreground`），以及针对其他环境的后备
- 服务器现在把启动 JSON 写到 `$SCREEN_DIR/.server-info`，这样即使在后台执行隐藏了 stdout 时 agent 也能找到 URL 和端口

**Brainstorm server 依赖打包**

- `node_modules` 被 vendored 进仓库，使 brainstorm server 在全新插件安装上立即可用，无需在运行时 `npm`
- 从打包的依赖中移除了 `fsevents`（macOS 专用原生二进制；chokidar 在没有它时优雅回退）
- 如果 `node_modules` 缺失，通过 `npm install` 后备自动安装

**OpenCode 工具映射修复**

- `TodoWrite` → `todowrite`（此前错误映射为 `update_plan`）；已对照 OpenCode 源验证

### 缺陷修复

**Windows/Linux：单引号破坏 SessionStart hook** (#577, #529, #644, PR #585)

- hooks.json 中 `${CLAUDE_PLUGIN_ROOT}` 周围的单引号在 Windows 上失败（cmd.exe 不把单引号识别为路径分隔符），在 Linux 上也失败（单引号阻止变量展开）
- 修复：用转义双引号替换单引号——在 macOS bash、Windows cmd.exe、Windows Git Bash 和 Linux 上均工作，无论路径中有无空格
- 已在 Windows 11（NT 10.0.26200.0）上用 Claude Code 2.1.72 和 Git for Windows 验证

**Brainstorming spec 审查循环被跳过** (#677)

- spec 审查循环（派发 spec-document-reviewer subagent，迭代直至批准）存在于散文 "After the Design" 小节中，但缺失于检查清单和流程图
- 由于 agent 比散文更可靠地遵循流程图和检查清单，spec 审查步骤被完全跳过
- 把步骤 7（spec 审查循环）加到检查清单和相应节点加到 dot 图
- 用 `claude --plugin-dir` 和 `claude-session-driver` 测试：worker 现在正确派发审查者

**Cursor 安装命令** (PR #676)

- 修复了 README 中的 Cursor 安装命令：`/plugin-add` → `/add-plugin`（通过 Cursor 2.5 发布公告确认）

**brainstorming 中的用户审查门** (#565)

- 在 spec 完成和 writing-plans 交接之间添加了明确的用户审查步骤
- 用户必须在实现规划开始前批准 spec
- 检查清单、流程图和散文都用新门更新

**session-start hook 每个平台只发出一次上下文**

- hook 现在检测它是运行在 Claude Code 还是其他平台
- 为 Claude Code 发出 `hookSpecificOutput`，为其他平台发出 `additional_context` —— 防止双重上下文注入

**token 分析脚本中的 lint 修复**

- `tests/claude-code/analyze-token-usage.py` 中 `except:` → `except Exception:`

### 维护

**移除了死代码**

- 删除了 `lib/skills-core.js` 及其测试（`tests/opencode/test-skills-core.js`）——自 2026 年 2 月起未使用
- 从 `tests/opencode/test-plugin-loading.sh` 中移除了 skills-core 存在检查

### 社区

- @karuturi —— Claude Code 官方市场安装说明 (PR #610)
- @mvanhorn —— session-start hook 双重发出修复，OpenCode 工具映射修复
- @daniel-graham —— 裸 except 的 lint 修复
- PR #585 作者 —— Windows/Linux hooks 引号修复

---

## v5.0.0 (2026-03-09)

### 破坏性变更

**Spec 和 plan 目录重构**

- Spec（brainstorming 输出）现在保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- Plan（writing-plans 输出）现在保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- spec/plan 位置的用户偏好覆盖这些默认值
- 所有内部 skill 引用、测试文件和示例路径都更新以匹配
- 迁移：如需要，把现有文件从 `docs/plans/` 移到新位置

**在有能力 harness 上强制 subagent 驱动开发**

writing-plans 不再在 subagent 驱动和 executing-plans 之间提供选择。在有 subagent 支持的 harness（Claude Code、Codex）上，subagent-driven-development 是必需的。executing-plans 保留给没有 subagent 能力的 harness，并现在告诉用户 Superpowers 在有 subagent 能力的平台上工作得更好。

**executing-plans 不再批量执行**

移除了"执行 3 个任务后停下来审查"模式。计划现在连续执行，只在遇到阻塞时停止。

**斜杠命令弃用**

`/brainstorm`、`/write-plan` 和 `/execute-plan` 现在显示指向对应 skill 的弃用通知。命令将在下一个主版本中移除。

### 新特性

**可视化 brainstorming 伴侣**

用于 brainstorming 会话的可选浏览器伴侣。当一个主题能从可视化中受益时，brainstorming skill 提议在终端会话旁边，于一个浏览器窗口中展示 mockup、图表、对比和其他内容。

- `lib/brainstorm-server/` —— WebSocket 服务器，带浏览器辅助库、会话管理脚本，以及深色/浅色主题框架模板（"Superpowers Brainstorming"，带 GitHub 链接）
- `skills/brainstorming/visual-companion.md` —— 针对服务器工作流、屏幕编写和反馈收集的渐进式披露指南
- brainstorming skill 在其流程中增加一个可视化伴侣决策点：在探索项目上下文之后，skill 评估即将到来的问题是否涉及可视化内容，并在其自己的消息中提议该伴侣
- 按问题决策：即便在接受之后，每个问题都评估浏览器还是终端更合适
- `tests/brainstorm-server/` 中的集成测试

**文档审查系统**

使用 subagent 派发的 spec 和 plan 文档自动化审查循环：

- `skills/brainstorming/spec-document-reviewer-prompt.md` —— 审查者检查完整性、一致性、架构和 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md` —— 审查者检查 spec 对齐、任务分解、文件结构和文件大小
- brainstorming 在写设计文档后派发 spec 审查者
- writing-plans 在每节后包含按块的 plan 审查循环
- 审查循环重复直至批准或在 5 次迭代后升级
- `tests/claude-code/test-document-review-system.sh` 中的端到端测试
- `docs/superpowers/` 中的设计 spec 和实现计划

**跨 skill 流水线的架构指导**

design-for-isolation 和 file-size-awareness 指导加入 brainstorming、writing-plans 和 subagent-driven-development：

- **Brainstorming** —— 新小节："Design for isolation and clarity"（为隔离和清晰而设计——清晰的边界、定义良好的接口、可独立测试的单元）和 "Working in existing codebases"（在现有代码库中工作——遵循现有模式，只做有针对性的改进）
- **Writing-plans** —— 新 "File Structure" 小节：在定义任务之前规划文件和职责。新 "Scope Check" 后备：捕获本应在 brainstorming 期间分解的多子系统 spec
- **SDD 实现者** —— 新 "Code Organization" 小节（遵循计划的文件结构、报告对文件增长的担忧）和 "When You're in Over Your Head" 升级指导
- **SDD 代码质量审查者** —— 现在检查架构、单元分解、计划符合度和文件增长
- **Spec/plan 审查者** —— 架构和文件大小加入审查标准
- **范围评估** —— brainstorming 现在评估一个项目对一个 spec 是否太大。多子系统请求被及早标记并分解为子项目，每个有自己的 spec → plan → 实现周期

**Subagent 驱动开发改进**

- **模型选择** —— 按任务类型选择模型能力的指导：机械实现用便宜模型、集成用标准模型、架构和审查用高能力模型
- **实现者状态协议** —— subagent 现在报告 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。控制器适当处理每种状态：用更多上下文重新派发、升级模型能力、拆分任务，或升级到人类

### 改进

**指令优先级层次**

为 using-superpowers 添加了明确的优先级排序：

1. 用户的显式指令（CLAUDE.md、AGENTS.md、直接请求）——最高优先级
2. Superpowers skills——覆盖默认系统行为
3. 默认系统提示——最低优先级

如果 CLAUDE.md 或 AGENTS.md 说 "don't use TDD"，而一个 skill 说 "always use TDD"，用户指令获胜。

**SUBAGENT-STOP 门**

为 using-superpowers 添加 `<SUBAGENT-STOP>` 块。为特定任务派发的 subagent 现在跳过该 skill，而非触发 1% 规则并调用完整 skill 工作流。

**多平台改进**

- Codex 工具映射移到渐进式披露参考文件（`references/codex-tools.md`）
- 添加平台适配指针，使非 Claude Code 平台能找到工具等价物
- 计划头现在称呼 "agentic workers" 而非特指 "Claude"
- Collab 特性要求记录在 `docs/README.codex.md` 中

**writing-plans 模板更新**

- 计划步骤现在使用复选框语法（`- [ ] **Step N:**`）进行进度追踪
- 计划头引用 subagent-driven-development 和 executing-plans 两者，带平台感知路由

---

## v4.3.1 (2026-02-21)

### 新增

**Cursor 支持**

Superpowers 现在与 Cursor 的插件系统配合工作。包含一个 `.cursor-plugin/plugin.json` manifest 和 README 中 Cursor 专用的安装说明。SessionStart hook 输出现在在现有 `hookSpecificOutput.additionalContext` 旁边包含一个 `additional_context` 字段，以兼容 Cursor hook。

### 修复

**Windows：为可靠的 hook 执行恢复了 polyglot 包装器 (#518, #504, #491, #487, #466, #440)**

Claude Code 在 Windows 上的 `.sh` 自动检测此前对 hook 命令前置 `bash`，破坏了执行。修复：

- 把 `session-start.sh` 重命名为 `session-start`（无扩展名），使自动检测不干扰
- 恢复带多位置 bash 发现（标准 Git for Windows 路径，再 PATH 后备）的 `run-hook.cmd` polyglot 包装器
- 如果没找到 bash 则静默退出，而非报错
- 在 Unix 上，包装器通过 `exec bash` 直接运行脚本
- 使用 POSIX 安全的 `dirname "$0"` 路径解析（在 dash/sh 上工作，不只是 bash）

这修复了 Windows 上路径中有空格、缺少 WSL、MSYS 上 `set -euo pipefail` 脆弱性以及反斜杠破坏导致的 SessionStart 失败。

## v4.3.0 (2026-02-12)

这个修复应大幅改善 superpowers skills 合规性，并降低 Claude 非预期进入其原生 plan 模式的几率。

### 变更

**brainstorming skill 现在强制其工作流，而非描述它**

模型此前跳过设计阶段，直接跳到 frontend-design 等实现 skill，或把整个 brainstorming 过程折叠成一个文本块。该 skill 现在用硬门、强制检查清单和 graphviz 流程图来强制合规：

- `<HARD-GATE>`：在呈现设计并获用户批准之前，不出现实现 skill、代码或脚手架
- 显式检查清单（6 项），必须作为任务创建并按序完成
- 以 `writing-plans` 作为唯一有效终止状态的 graphviz 流程图
- 针对 "this is too simple to need a design"（这太简单不需要设计）的反模式提示——模型用来跳过流程的精确合理化
- 设计小节大小基于小节复杂度，而非项目复杂度

**using-superpowers 工作流图拦截 EnterPlanMode**

在 skill 流程图中增加一个 `EnterPlanMode` 拦截。当模型即将进入 Claude 原生 plan 模式时，它检查 brainstorming 是否已发生，并改为通过 brainstorming skill 路由。绝不进入 plan 模式。

### 修复

**SessionStart hook 现在同步运行**

把 hooks.json 中 `async: true` 改为 `async: false`。当异步时，hook 可能在模型第一轮之前无法完成，意味着 using-superpowers 指令不在第一条消息的上下文中。

## v4.2.0 (2026-02-05)

### 破坏性变更

**Codex：用原生 skill 发现替换了 bootstrap CLI**

`superpowers-codex` bootstrap CLI、Windows `.cmd` 包装器和相关 bootstrap 内容文件已被移除。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生 skill 发现，所以旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只是克隆 + 符号链接（记录在 INSTALL.md）。不需要 Node.js 依赖。旧的 `~/.codex/skills/` 路径已弃用。

### 修复

**Windows：修复了 Claude Code 2.1.x 的 hook 执行 (#331)**

Claude Code 2.1.x 改变了 hook 在 Windows 上的执行方式：它现在自动检测命令中的 `.sh` 文件并前置 `bash`。这破坏了 polyglot 包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 试图把 `.cmd` 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以对 shell 脚本强制 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

**Windows：SessionStart hook 异步运行以防止终端冻结 (#404, #413, #414, #419)**

同步的 SessionStart hook 阻止 TUI 在 Windows 上进入 raw 模式，冻结所有键盘输入。异步运行 hook 防止冻结，同时仍注入 superpowers 上下文。

**Windows：修复了 O(n^2) 的 `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环在 bash 中是 O(n^2)，因子字符串复制开销。在 Windows Git Bash 上这需要 60 多秒。替换为 bash 参数替换（`${s//old/new}`），它把每个模式作为单次 C 级遍历运行——macOS 上快 7 倍，Windows 上大幅更快。

**Codex：修复了 Windows/PowerShell 调用 (#285, #243)**

- Windows 不识别 shebang，所以直接调用无扩展名 `superpowers-codex` 脚本触发了一个"打开方式"对话框。所有调用现在前置 `node`。
- 修复了 Windows 上的 `~/` 路径展开——PowerShell 在把 `~` 作为参数传给 `node` 时不展开。改为 `$HOME`，它在 bash 和 PowerShell 中都正确展开。

**Codex：修复了安装器中的路径解析**

使用 `fileURLToPath()` 而非手工 URL pathname 解析，以正确处理所有平台上带空格和特殊字符的路径。

**Codex：修复了 writing-skills 中过时的 skills 路径**

把 `~/.codex/skills/` 引用（已弃用）更新为 `~/.agents/skills/`，用于原生发现。

### 改进

**实现之前现在要求 worktree 隔离**

把 `using-git-worktrees` 作为 `subagent-driven-development` 和 `executing-plans` 两者的必需 skill 添加。实现工作流现在明确要求在开始工作前设置一个隔离 worktree，防止在 main 上直接意外工作。

**main 分支保护软化为要求显式同意**

skill 现在不再完全禁止 main 分支工作，而是允许在用户显式同意下进行。更灵活，同时仍确保用户意识到其影响。

**简化了安装验证**

从验证步骤中移除了 `/help` 命令检查和特定斜杠命令列表。skill 主要通过描述你想做什么来调用，而非运行特定命令。

**Codex：在 bootstrap 中澄清了 subagent 工具映射**

改进了 Codex 工具如何映射到 Claude Code 等价物（用于 subagent 工作流）的文档。

### 测试

- 为 subagent-driven-development 添加了 worktree 要求测试
- 添加了 main 分支红旗警告测试
- 修复了 skill 识别测试断言中的大小写敏感性

---

## v4.1.1 (2026-01-23)

### 修复

**OpenCode：按官方文档标准化到 `plugins/` 目录 (#343)**

OpenCode 官方文档使用 `~/.config/opencode/plugins/`（复数）。我们的文档此前使用 `plugin/`（单数）。虽然 OpenCode 接受两种形式，但我们已标准化到官方约定以避免混淆。

变更：
- 在仓库结构中把 `.opencode/plugin/` 重命名为 `.opencode/plugins/`
- 更新了所有平台上的安装文档（INSTALL.md、README.opencode.md）
- 更新测试脚本以匹配

**OpenCode：修复了符号链接说明 (#339, #342)**

- 在 `ln -s` 之前添加显式 `rm`（修复重装时的"文件已存在"错误）
- 添加了 INSTALL.md 中缺失的 skills 符号链接步骤
- 从已弃用的 `use_skill`/`find_skills` 更新到原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### 破坏性变更

**OpenCode：切换到原生 skills 系统**

OpenCode 的 Superpowers 现在使用 OpenCode 原生的 `skill` 工具，而非自定义 `use_skill`/`find_skills` 工具。这是一个更干净的集成，与 OpenCode 内置的 skill 发现配合工作。

**需要迁移：** Skills 必须被符号链接到 `~/.config/opencode/skills/superpowers/`（见更新后的安装文档）。

### 修复

**OpenCode：修复了会话开始时的 agent 重置 (#226)**

此前使用 `session.prompt({ noReply: true })` 的 bootstrap 注入方式导致 OpenCode 在第一条消息时把选中的 agent 重置为 "build"。现在使用 `experimental.chat.system.transform` hook，它直接修改 system prompt，没有副作用。

**OpenCode：修复了 Windows 安装 (#232)**

- 移除了对 `skills-core.js` 的依赖（消除文件被复制而非符号链接时的破损相对导入）
- 添加了针对 cmd.exe、PowerShell 和 Git Bash 的全面 Windows 安装文档
- 记录了每个平台的正确符号链接 vs junction 用法

**Claude Code：修复了 Claude Code 2.1.x 的 Windows hook 执行**

Claude Code 2.1.x 改变了 hook 在 Windows 上的执行方式：它现在自动检测命令中的 `.sh` 文件并前置 `bash `。这破坏了 polyglot 包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 试图把 .cmd 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以对 shell 脚本强制 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

---

## v4.0.3 (2025-12-26)

### 改进

**为显式 skill 请求强化了 using-superpowers skill**

解决了一个失败模式：即使用户按名称显式请求（例如 "subagent-driven-development, please"），Claude 也跳过调用 skill。Claude 会想"我知道那是什么意思"并直接开始工作，而非加载 skill。

变更：
- 更新 "The Rule" 以说 "Invoke relevant or requested skills" 而非 "Check for skills" —— 强调主动调用而非被动检查
- 添加 "BEFORE any response or action" —— 原始措辞只提到 "response"，但 Claude 有时会在响应之前采取行动
- 添加保证：调用错误的 skill 没关系 —— 减少犹豫
- 添加新红旗："I know what that means"（我知道那是什么意思） → 知道概念 ≠ 使用 skill

**添加了显式 skill 请求测试**

`tests/explicit-skill-requests/` 中的新测试套件，验证当用户按名称请求时 Claude 正确调用 skill。包括单轮和多轮测试场景。

## v4.0.2 (2025-12-23)

### 修复

**斜杠命令现在仅限用户**

为全部三个斜杠命令（`/brainstorm`、`/execute-plan`、`/write-plan`）添加了 `disable-model-invocation: true`。Claude 不再能通过 Skill 工具调用这些命令——它们仅限手动用户调用。

底层 skill（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍可供 Claude 自主调用。此变更防止当 Claude 调用一个只是重定向到 skill 的命令时产生混淆。

## v4.0.1 (2025-12-23)

### 修复

**澄清了在 Claude Code 中如何访问 skills**

修复了一个令人困惑的模式：Claude 会通过 Skill 工具调用 skill，然后单独尝试 Read 该 skill 文件。`using-superpowers` skill 现在明确说明 Skill 工具直接加载 skill 内容——无需读取文件。

- 向 `using-superpowers` 添加 "How to Access Skills" 小节
- 把说明中的 "read the skill" 改为 "invoke the skill"
- 更新斜杠命令以使用完全限定 skill 名（例如 `superpowers:brainstorming`）

**向 receiving-code-review 添加了 GitHub 线程回复指导** (h/t @ralphbean)

添加了关于在原始线程中回复内联审查评论（而非作为顶层 PR 评论）的说明。

**向 writing-skills 添加了自动化优于文档的指导** (h/t @EthanJStark)

添加了机械约束应当被自动化、而非记录的指导——把 skill 留给判断性决策。

## v4.0.0 (2025-12-17)

### 新特性

**subagent-driven-development 中的两阶段代码审查**

Subagent 工作流现在在每个任务后使用两个独立的审查阶段：

1. **Spec 合规审查** —— 怀疑的审查者验证实现精确匹配 spec。捕获缺失需求以及过度构建。不信任实现者的报告——阅读真实代码。

2. **代码质量审查** —— 只在 spec 合规通过后运行。审查干净代码、测试覆盖、可维护性。

这捕获了代码写得很好但与请求不符这一常见失败模式。审查是循环，而非一次性：如果审查者发现问题，实现者修复，然后审查者再次检查。

其他 subagent 工作流改进：
- 控制器向 worker 提供完整任务文本（而非文件引用）
- worker 可以在工作之前和工作期间提出澄清问题
- 报告完成前的自我审查清单
- 计划在开始时读取一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新 prompt 模板：
- `implementer-prompt.md` —— 包含自我审查清单，鼓励提问
- `spec-reviewer-prompt.md` —— 针对需求的怀疑验证
- `code-quality-reviewer-prompt.md` —— 标准代码审查

**调试技术与工具整合**

`systematic-debugging` 现在捆绑支持技术和工具：
- `root-cause-tracing.md` —— 通过调用栈向后追踪 bug
- `defense-in-depth.md` —— 在多层添加验证
- `condition-based-waiting.md` —— 用条件轮询替换任意超时
- `find-polluter.sh` —— 二分脚本，找出哪个测试造成污染
- `condition-based-waiting-example.ts` —— 来自真实调试会话的完整实现

**测试反模式参考**

`test-driven-development` 现在包含 `testing-anti-patterns.md`，覆盖：
- 测试 mock 行为而非真实行为
- 向生产类添加仅测试方法
- 在不理解依赖的情况下 mock
- 隐藏结构假设的不完整 mock

**Skill 测试基础设施**

用于验证 skill 行为的三个新测试框架：

`tests/skill-triggering/` —— 验证 skill 从朴素 prompt 触发，无需显式命名。测试 6 个 skill 以确保仅凭描述就足够。

`tests/claude-code/` —— 使用 `claude -p` 的集成测试，用于无头测试。通过会话记录（JSONL）分析验证 skill 使用。包含用于成本追踪的 `analyze-token-usage.py`。

`tests/subagent-driven-dev/` —— 带两个完整测试项目的端到端工作流验证：
- `go-fractals/` —— 带 Sierpinski/Mandelbrot 的 CLI 工具（10 个任务）
- `svelte-todo/` —— 带 localStorage 和 Playwright 的 CRUD 应用（12 个任务）

### 主要变更

**DOT 流程图作为可执行规范**

用 DOT/GraphViz 流程图作为权威流程定义重写了关键 skill。散文成为支持内容。

**描述陷阱**（记录在 `writing-skills`）：发现当 skill 描述包含工作流摘要时，描述会覆盖流程图内容。Claude 遵循简短描述而非阅读详细流程图。修复：描述必须仅含触发条件（"Use when X"），不含流程细节。

**using-superpowers 中的 skill 优先级**

当多个 skill 适用时，流程 skill（brainstorming、debugging）现在明确排在实现 skill 之前。"Build X" 先触发 brainstorming，再触发领域 skill。

**brainstorming 触发强化**

描述改为祈使句："You MUST use this before any creative work—creating features, building components, adding functionality, or modifying behavior."（在任何创造性工作之前你必须使用它——创建功能、构建组件、添加功能或修改行为。）

### 破坏性变更

**Skill 整合** —— 六个独立 skill 合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 捆绑在 `systematic-debugging/` 中
- `testing-skills-with-subagents` → 捆绑在 `writing-skills/` 中
- `testing-anti-patterns` → 捆绑在 `test-driven-development/` 中
- `sharing-skills` 移除（过时）

### 其他改进

- **render-graphs.js** —— 从 skill 提取 DOT 图并渲染为 SVG 的工具
- **using-superpowers 中的合理化表** —— 可扫描格式，包含新条目："I need more context first"、"Let me explore first"、"This feels productive"
- **docs/testing.md** —— 用 Claude Code 集成测试测试 skill 的指南

---

## v3.6.2 (2025-12-03)

### 修复

- **Linux 兼容性**：修复了 polyglot hook 包装器（`run-hook.cmd`）以使用 POSIX 兼容语法
  - 在第 16 行把 bash 专用的 `${BASH_SOURCE[0]:-$0}` 替换为标准的 `$0`
  - 解决了 `/bin/sh` 为 dash 的 Ubuntu/Debian 系统上的 "Bad substitution" 错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### 变更

- **OpenCode Bootstrap 重构**：把 bootstrap 注入从 `chat.message` hook 切换到 `session.created` 事件
  - bootstrap 现在通过 `session.prompt()`（带 `noReply: true`）在会话创建时注入
  - 明确告诉模型 using-superpowers 已加载，以防止冗余的 skill 加载
  - 把 bootstrap 内容生成整合到共享的 `getBootstrapContent()` 辅助函数
  - 更干净的单实现方式（移除了后备模式）

---

## v3.5.0 (2025-11-23)

### 新增

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 用于跨上下文压缩保持 skill 持久化的消息插入模式
  - 通过 chat.message hook 的自动上下文注入
  - 在 session.compacted 事件上自动重新注入
  - 三级 skill 优先级：项目 > 个人 > superpowers
  - 项目本地 skill 支持（`.opencode/skills/`）
  - 共享核心模块（`lib/skills-core.js`），用于与 Codex 的代码复用
  - 带适当隔离的自动化测试套件（`tests/opencode/`）
  - 平台专用文档（`docs/README.opencode.md`、`docs/README.codex.md`）

### 变更

- **重构 Codex 实现**：现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - skill 发现和解析的单一事实来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进的文档**：重写 README 以清楚解释问题/解决方案
  - 移除重复小节和冲突信息
  - 添加完整工作流描述（brainstorm → plan → execute → finish）
  - 简化平台安装说明
  - 强调 skill 检查协议，而非自动激活声明

---

## v3.4.1 (2025-10-31)

### 改进

- 优化了 superpowers bootstrap 以消除冗余的 skill 执行。`using-superpowers` skill 内容现在直接在会话上下文中提供，并带有明确指导，仅对其他 skill 使用 Skill 工具。这减少了开销，并防止了 agent 尽管已在会话开始时拥有内容却仍手动执行 `using-superpowers` 这一令人困惑的循环。

## v3.4.0 (2025-10-30)

### 改进

- 简化了 `brainstorming` skill，以回归原始的对话式愿景。移除了带正式检查清单的重量级 6 阶段流程，转而采用自然对话：一次问一个问题，然后以 200-300 字小节呈现设计并验证。保留了文档和实现交接特性。

## v3.3.1 (2025-10-28)

### 改进

- 更新了 `brainstorming` skill，要求在提问前自主侦察，鼓励以推荐驱动的决策，并防止 agent 把优先级排序委派回人类。
- 按照 Strunk 的《风格的要素》原则，对 `brainstorming` skill 应用了写作清晰度改进（删除了不必要的词、把否定形式转为肯定、改进了平行结构）。

### 缺陷修复

- 澄清了 `writing-skills` 指导，使其指向正确的 agent 专用个人 skill 目录（Claude Code 用 `~/.claude/skills`，Codex 用 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### 新特性

**实验性 Codex 支持**
- 添加了统一的 `superpowers-codex` 脚本，带 bootstrap/use-skill/find-skills 命令
- 跨平台 Node.js 实现（在 Windows、macOS、Linux 上工作）
- 命名空间 skills：superpowers skill 用 `superpowers:skill-name`，个人 skill 用 `skill-name`
- 名称匹配时个人 skill 覆盖 superpowers skill
- 干净的 skill 显示：显示 name/description，不含原始 frontmatter
- 有用的上下文：为每个 skill 显示支持文件目录
- Codex 的工具映射：TodoWrite→update_plan、subagent→手动后备等
- 与用于自动启动的最小 AGENTS.md 的 bootstrap 集成
- Codex 专用的完整安装指南和 bootstrap 说明

**与 Claude Code 集成的关键区别：**
- 单一统一脚本，而非分离的工具
- 针对 Codex 专用等价物的工具替换系统
- 简化的 subagent 处理（手动工作而非委派）
- 更新的术语："Superpowers skills" 而非 "Core skills"

### 添加的文件
- `.codex/INSTALL.md` —— Codex 用户的安装指南
- `.codex/superpowers-bootstrap.md` —— 带 Codex 适配的 bootstrap 说明
- `.codex/superpowers-codex` —— 带全部功能的统一 Node.js 可执行文件

**注意：** Codex 支持是实验性的。该集成提供核心 superpowers 功能，但可能需要根据用户反馈精炼。

## v3.2.3 (2025-10-23)

### 改进

**更新了 using-superpowers skill 以使用 Skill 工具而非 Read 工具**
- 把 skill 调用说明从 Read 工具改为 Skill 工具
- 更新描述："using Read tool" → "using Skill tool"
- 更新步骤 3："Use the Read tool" → "Use the Skill tool to read and run"
- 更新合理化列表："Read the current version" → "Run the current version"

Skill 工具是在 Claude Code 中调用 skill 的正确机制。此更新更正了 bootstrap 说明，以引导 agent 使用正确工具。

### 变更的文件
- 更新：`skills/using-superpowers/SKILL.md` —— 把工具引用从 Read 改为 Skill

## v3.2.2 (2025-10-21)

### 改进

**强化 using-superpowers skill 以对抗 agent 合理化**
- 添加 EXTREMELY-IMPORTANT 块，带关于强制 skill 检查的绝对语言
  - "If even 1% chance a skill applies, you MUST read it"（即使只有 1% 的机会某 skill 适用，你绝对必须读取它）
  - "You do not have a choice. You cannot rationalize your way out."（你没有选择。你无法通过合理化逃避。）
- 添加 MANDATORY FIRST RESPONSE PROTOCOL 检查清单
  - agent 在任何响应之前必须完成的 5 步流程
  - 明确的"没有这个就响应 = 失败"后果
- 添加 Common Rationalizations 小节，带 8 种特定逃避模式
  - "This is just a simple question"（这只是个简单问题） → 错误
  - "I can check files quickly"（我可以快速检查文件） → 错误
  - "Let me gather information first"（让我先收集信息） → 错误
  - 外加在 agent 行为中观察到的 5 种常见模式

这些变更解决了观察到的 agent 行为：尽管有明确说明，它们仍围绕 skill 使用进行合理化。强硬的语言和预防性的反驳论据旨在让不合规变得更难。

### 变更的文件
- 更新：`skills/using-superpowers/SKILL.md` —— 添加三层强制以防止跳过 skill 的合理化

## v3.2.1 (2025-10-20)

### 新特性

**代码审查 agent 现已包含在插件中**
- 向插件的 `agents/` 目录添加 `superpowers:code-reviewer` agent
- agent 提供针对计划和编码标准的系统性代码审查
- 此前要求用户有个人 agent 配置
- 所有 skill 引用更新为使用命名空间的 `superpowers:code-reviewer`
- 修复 #55

### 变更的文件
- 新增：`agents/code-reviewer.md` —— 带审查检查清单和输出格式的 agent 定义
- 更新：`skills/requesting-code-review/SKILL.md` —— 对 `superpowers:code-reviewer` 的引用
- 更新：`skills/subagent-driven-development/SKILL.md` —— 对 `superpowers:code-reviewer` 的引用

## v3.2.0 (2025-10-18)

### 新特性

**brainstorming 工作流中的设计文档**
- 向 brainstorming skill 添加阶段 4：设计文档
- 设计文档现在在实现之前写到 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复了 skill 转换期间丢失的原始 brainstorming 命令功能
- 文档在 worktree 设置和实现规划之前编写
- 用 subagent 测试以验证在时间压力下的合规性

### 破坏性变更

**Skill 引用命名空间标准化**
- 所有内部 skill 引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（此前只是 `test-driven-development`）
- 影响所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与使用 Skill 工具调用 skill 的方式对齐
- 更新的文件：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### 改进

**设计 vs 实现 plan 命名**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实现计划继续使用现有 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，有清晰的命名区分

## v3.1.1 (2025-10-17)

### 缺陷修复

- **修复了 README 中的命令语法** (#44) —— 更新所有命令引用以使用正确的命名空间语法（`/superpowers:brainstorm` 而非 `/brainstorm`）。插件提供的命令会被 Claude Code 自动命名空间化，以避免插件之间冲突。

## v3.1.0 (2025-10-17)

### 破坏性变更

**Skill 名称标准化为小写**
- 所有 skill frontmatter 的 `name:` 字段现在使用与目录名匹配的小写 kebab-case
- 例如：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有 skill 公告和交叉引用都更新为小写格式
- 这确保了目录名、frontmatter 和文档间命名一致

### 新特性

**增强的 brainstorming skill**
- 添加 Quick Reference 表，显示阶段、活动和工具用法
- 添加可复制的工作流检查清单，用于追踪进度
- 添加关于何时重新访问更早阶段的决策流程图
- 添加全面的 AskUserQuestion 工具指导，带具体示例
- 添加 "Question Patterns" 小节，解释何时使用结构化 vs 开放式问题
- 重构 Key Principles 为可扫描表

**Anthropic 最佳实践整合**
- 添加 `skills/writing-skills/anthropic-best-practices.md` —— 官方 Anthropic skill 编写指南
- 在 writing-skills SKILL.md 中引用以提供全面指导
- 提供渐进式披露、工作流和评估的模式

### 改进

**Skill 交叉引用清晰度**
- 所有 skill 引用现在使用显式需求标记：
  - `**REQUIRED BACKGROUND:**` —— 你必须理解的先决条件
  - `**REQUIRED SUB-SKILL:**` —— 工作流中必须使用的 skill
  - `**Complementary skills:**` —— 可选但有帮助的相关 skill
- 移除旧的路径格式（`skills/collaboration/X` → 只是 `X`）
- 用分类关系（必需 vs 互补）更新集成小节
- 用最佳实践更新交叉引用文档

**与 Anthropic 最佳实践对齐**
- 修复描述语法和语态（完全第三人称）
- 为扫描添加 Quick Reference 表
- 添加 Claude 可复制并追踪的工作流检查清单
- 对非显而易见的决策点适当使用流程图
- 改进可扫描表格式
- 所有 skill 远低于 500 行建议

### 缺陷修复

- **重新添加缺失的命令重定向** —— 恢复在 v3.0 迁移中意外移除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复 `defense-in-depth` 名称不匹配（此前为 `Defense-in-Depth-Validation`）
- 修复 `receiving-code-review` 名称不匹配（此前为 `Code-Review-Reception`）
- 修复 `commands/brainstorm.md` 引用到正确的 skill 名
- 移除对不存在的相关 skill 的引用

### 文档

**writing-skills 改进**
- 用显式需求标记更新交叉引用指导
- 添加对 Anthropic 官方最佳实践的引用
- 改进展示正确 skill 引用格式的示例

## v3.0.1 (2025-10-16)

### 变更

我们现在使用 Anthropic 的第一方 skills 系统！

## v2.0.2 (2025-10-12)

### 缺陷修复

- **修复了当本地 skills 仓库领先于上游时的误警告** —— 初始化脚本此前在本地仓库有领先于上游的提交时，错误地警告 "New skills available from upstream"。逻辑现在正确区分三种 git 状态：本地落后（应更新）、本地领先（无警告）、已分叉（应警告）。

## v2.0.1 (2025-10-12)

### 缺陷修复

- **修复了插件上下文中的 session-start hook 执行** (#8, PR #9) —— hook 此前静默失败，报 "Plugin hook error"，阻止 skills 上下文加载。通过以下方式修复：
  - 当 BASH_SOURCE 在 Claude Code 执行上下文中未绑定时，使用 `${BASH_SOURCE[0]:-$0}` 后备
  - 添加 `|| true` 以在过滤状态标志时优雅处理空 grep 结果

---

# Superpowers v2.0.0 发布说明

## 概览

Superpowers v2.0 通过一次重大架构转变，使 skills 更易访问、更易维护、更社区驱动。

头条变更是 **skills 仓库分离**：所有 skills、脚本和文档已从插件移到一个专用仓库（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）。这把 superpowers 从一个单体插件转变为一个管理 skills 仓库本地克隆的轻量垫片。skills 在会话开始时自动更新。用户通过标准 git 工作流 fork 并贡献改进。skills 库独立于插件进行版本管理。

除基础设施外，本次发布添加了九个专注于问题解决、研究和架构的新 skill。我们用祈使语气和更清晰的结构重写了核心的 **using-skills** 文档，使 Claude 更容易理解何时以及如何使用 skills。**find-skills** 现在输出你可以直接粘贴进 Read 工具的路径，消除了 skill 发现工作流中的摩擦。

用户体验是无缝的：插件自动处理克隆、fork 和更新。贡献者发现新架构使改进和分享 skills 变得轻松。本次发布为 skills 作为社区资源快速演进奠定了基础。

## 破坏性变更

### Skills 仓库分离

**最大的变化：** Skills 不再位于插件中。它们已被移到一个独立仓库 [obra/superpowers-skills](https://github.com/obra/superpowers-skills)。

**这对你意味着什么：**

- **首次安装：** 插件自动把 skills 克隆到 `~/.config/superpowers/skills/`
- **Fork：** 在设置期间，你将被提供 fork skills 仓库的选项（如果安装了 `gh`）
- **更新：** skills 在会话开始时自动更新（尽可能快进）
- **贡献：** 在分支上工作，本地提交，向上游提交 PR
- **不再有遮蔽：** 旧的两级系统（个人/核心）被单仓库分支工作流取代

**迁移：**

如果你有现有安装：
1. 你的旧 `~/.config/superpowers/.git` 将被备份到 `~/.config/superpowers/.git.bak`
2. 旧 skills 将被备份到 `~/.config/superpowers/skills.bak`
3. obra/superpowers-skills 的新鲜克隆将在 `~/.config/superpowers/skills/` 创建

### 移除的特性

- **个人 superpowers 覆盖系统** —— 被 git 分支工作流取代
- **setup-personal-superpowers hook** —— 被 initialize-skills.sh 取代

## 新特性

### Skills 仓库基础设施

**自动克隆和设置**（`lib/initialize-skills.sh`）
- 首次运行时克隆 obra/superpowers-skills
- 如果安装了 GitHub CLI 则提供 fork 创建
- 正确设置 upstream/origin 远程
- 处理从旧安装的迁移

**自动更新**
- 每个会话开始时从追踪远程 fetch
- 尽可能自动以快进合并
- 当需要手动同步时通知（分支已分叉）
- 使用 pulling-updates-from-skills-repository skill 进行手动同步

### 新 Skills

**问题解决 Skills**（`skills/problem-solving/`）
- **collision-zone-thinking** —— 强制把不相关概念结合以获得涌现洞察
- **inversion-exercise** —— 翻转假设以揭示隐藏约束
- **meta-pattern-recognition** —— 跨领域发现通用原则
- **scale-game** —— 在极端情况下测试以暴露基本真相
- **simplification-cascades** —— 寻找能消除多个组件的洞察
- **when-stuck** —— 派发到正确的问题解决技术

**研究 Skills**（`skills/research/`）
- **tracing-knowledge-lineages** —— 理解思想如何随时间演进

**架构 Skills**（`skills/architecture/`）
- **preserving-productive-tensions** —— 保留多个有效方法，而非强制过早解决

### Skills 改进

**using-skills（原名 getting-started）**
- 从 getting-started 重命名为 using-skills
- 用祈使语气完全重写 (v4.0.0)
- 前置关键规则
- 为所有工作流添加 "Why" 解释
- 引用中始终包含 /SKILL.md 后缀
- 刚性规则和柔性模式之间更清晰的区分

**writing-skills**
- 交叉引用指导从 using-skills 移来
- 添加 token 效率小节（字数目标）
- 改进 CSO（Claude Search Optimization）指导

**sharing-skills**
- 为新的分支加 PR 工作流更新 (v2.0.0)
- 移除个人/核心拆分引用

**pulling-updates-from-skills-repository**（新）
- 与上游同步的完整工作流
- 替换旧的 "updating-skills" skill

### 工具改进

**find-skills**
- 现在输出带 /SKILL.md 后缀的完整路径
- 使路径可直接用于 Read 工具
- 更新帮助文本

**skill-run**
- 从 scripts/ 移到 skills/using-skills/
- 改进文档

### 插件基础设施

**Session Start Hook**
- 现在从 skills 仓库位置加载
- 在会话开始时显示完整 skills 列表
- 打印 skills 位置信息
- 显示更新状态（更新成功 / 落后上游）
- 把 "skills 落后" 警告移到输出末尾

**环境变量**
- `SUPERPOWERS_SKILLS_ROOT` 设置为 `~/.config/superpowers/skills`
- 在所有路径中一致使用

## 缺陷修复

- 修复了 fork 时重复添加 upstream 远程
- 修复了 find-skills 输出中重复的 "skills/" 前缀
- 从 session-start 中移除过时的 setup-personal-superpowers 调用
- 修复了 hook 和命令中的路径引用

## 文档

### README
- 为新的 skills 仓库架构更新
- 指向 superpowers-skills 仓库的醒目链接
- 更新自动更新描述
- 修复 skill 名称和引用
- 更新 Meta skills 列表

### 测试文档
- 添加全面的测试检查清单（`docs/TESTING-CHECKLIST.md`）
- 创建本地市场配置用于测试
- 记录手动测试场景

## 技术细节

### 文件变更

**新增：**
- `lib/initialize-skills.sh` —— skills 仓库初始化和自动更新
- `docs/TESTING-CHECKLIST.md` —— 手动测试场景
- `.claude-plugin/marketplace.json` —— 本地测试配置

**移除：**
- `skills/` 目录（82 个文件）—— 现在在 obra/superpowers-skills
- `scripts/` 目录 —— 现在在 obra/superpowers-skills/skills/using-skills/
- `hooks/setup-personal-superpowers.sh` —— 过时

**修改：**
- `hooks/session-start.sh` —— 使用 ~/.config/superpowers/skills 的 skills
- `commands/brainstorm.md` —— 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` —— 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` —— 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `README.md` —— 为新架构完全重写

### 提交历史

本次发布包括：
- 20+ 次 skills 仓库分离提交
- PR #1：受 Amplifier 启发的问题解决和研究 skills
- PR #2：个人 superpowers 覆盖系统（后被替换）
- 多次 skill 精炼和文档改进

## 升级说明

### 全新安装

```bash
# 在 Claude Code 中
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件自动处理一切。

### 从 v1.x 升级

1. **备份你的个人 skills**（如果有）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件：**
   ```bash
   /plugin update superpowers
   ```

3. **下次会话开始时：**
   - 旧安装将自动备份
   - 将克隆全新 skills 仓库
   - 如果你有 GitHub CLI，将被提供 fork 选项

4. **迁移个人 skills**（如果之前有）：
   - 在本地 skills 仓库创建分支
   - 从备份复制你的个人 skills
   - 提交并 push 到你的 fork
   - 考虑通过 PR 贡献回去

## 下一步

### 对用户

- 探索新的问题解决 skills
- 为 skill 改进尝试基于分支的工作流
- 把 skills 贡献回社区

### 对贡献者

- skills 仓库现在位于 https://github.com/obra/superpowers-skills
- Fork → 分支 → PR 工作流
- 见 skills/meta/writing-skills/SKILL.md 了解文档的 TDD 方法

## 已知问题

目前没有。

## 致谢

- 受 Amplifier 模式启发的问题解决 skills
- 社区贡献和反馈
- 对 skill 有效性的广泛测试和迭代

---

**完整变更日志：** https://github.com/obra/superpowers/compare/dd013f6...main
**Skills 仓库：** https://github.com/obra/superpowers-skills
**Issues：** https://github.com/obra/superpowers/issues
