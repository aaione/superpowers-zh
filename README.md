![](/superpowers.png)

# Superpowers

Superpowers 是一套面向你 coding agents 的完整软件开发方法论，建立在一组可组合的 skills 和一些初始指令之上，这些指令确保你的 agent 会使用它们。

## 我们在招人！

我们正在全职招聘一位帮助 Superpowers 社区和代码工作的人。你可以在 https://primeradiant.com/jobs/superpowers-community-engineer/ 了解这份工作。如果你觉得这听起来像你认识的人，务必把我们的方式告诉他们。

## 快速开始

赋予你的 agent Superpowers：[Claude Code](#claude-code)、[Antigravity](#antigravity)、[Codex App](#codex-app)、[Codex CLI](#codex-cli)、[Cursor](#cursor)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[GitHub Copilot CLI](#github-copilot-cli)、[Kimi Code](#kimi-code)、[OpenCode](#opencode)、[Pi](#pi)。

## 工作原理

它从你启动 coding agent 的那一刻就开始运作。一旦它发现你正在构建某些东西，它*不会*直接跳进去尝试写代码。相反，它会退一步，询问你到底想要做什么。

一旦它从对话中梳理出规范说明（spec），它会将其分成足够短的块展示给你，以便你真正阅读和消化。

在你对设计签字认可后，你的 agent 会组装一份实现计划，清晰到连一个热情但品味差、没有判断力、没有项目上下文、还抗拒测试的初级工程师都能照做。它强调真正的红/绿 TDD、YAGNI（You Aren't Gonna Need It，你不会需要它）和 DRY。

接下来，一旦你说 "go"，它会启动一个 *subagent-driven-development*（子代理驱动开发）流程，让 agent 逐个完成每项工程任务，检查和审查它们的工作，并继续推进。你的 agent 一次自主工作几个小时而不偏离你制定的计划，这并不罕见。

还有更多内容，但这就是系统的核心。而且因为 skills 会自动触发，你不需要做任何特殊操作。你的 coding agent 天生就拥有 Superpowers。

## 商业服务

如果你在企业中使用 Superpowers，并能从商业支持、额外工具或托管支出中受益，请不要犹豫，通过 sales@primeradiant.com 联系我们。

## 安装

安装方式因 harness 而异。如果你使用不止一种，请为每一种分别安装 Superpowers。

### Claude Code

Superpowers 可通过[官方 Claude 插件市场](https://claude.com/plugins/superpowers)获取。

#### 官方市场

- 从 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场为 Claude Code 提供 Superpowers 和其他一些相关插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从这个市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

从这个仓库以插件形式安装 Superpowers：

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity 会运行插件的 session-start hook，所以 Superpowers 从第一条消息起就生效。用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 在 Codex app 中，点击侧边栏的 Plugins。
- 你应该在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按提示操作。

### Codex CLI

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或在插件市场中搜索 "superpowers"。

### Factory Droid

- 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 安装扩展：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 之后更新：

  ```bash
  gemini extensions update superpowers
  ```

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Kimi Code

Superpowers 可在 Kimi Code 的插件市场中获取。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 进入 `Marketplace` > `Superpowers` 并安装。

- 或直接从这个仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他 harness 中使用，也要单独安装 Superpowers。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

从这个仓库以 Pi 包形式安装 Superpowers：

```bash
pi install git:github.com/obra/superpowers
```

对于本地开发，将这个检出作为临时包加载来运行 Pi：

```bash
pi -e /path/to/superpowers
```

Pi 包会加载 Superpowers skills 和一个小扩展，该扩展在会话启动时以及压缩（compaction）之后注入 `using-superpowers` bootstrap。Pi 拥有原生 skills，因此不需要兼容性的 `Skill` 工具。Subagent 和任务列表工具仍是可选的 Pi 配套包。

## 基本工作流

1. **brainstorming** - 在编写代码前激活。通过提问细化粗略想法，探索替代方案，分节呈现设计以供验证。保存设计文档。

2. **using-git-worktrees** - 设计批准后激活。在新分支上创建隔离工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 携批准的设计激活。把工作拆成小任务（每个 2-5 分钟）。每个任务都有精确的文件路径、完整代码、验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 携计划激活。为每个任务派发全新 subagent 并进行两阶段 review（规范合规，然后代码质量），或带人工检查点批量执行。

5. **test-driven-development** - 实现期间激活。强制执行 RED-GREEN-REFACTOR：编写失败的测试，看它失败，编写最小代码，看它通过，提交。删除在测试之前编写的代码。

6. **requesting-code-review** - 任务之间激活。对照计划 review，按严重度报告问题。Critical 问题阻塞推进。

7. **finishing-a-development-branch** - 任务完成时激活。验证测试，呈现选项（合并/PR/保留/丢弃），清理 worktree。

**agent 在任何任务之前都会检查相关 skills。** 这是强制工作流，不是建议。

## 内容概览

### Skills 库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因流程（含根因追踪、纵深防御、条件等待等技术）
- **verification-before-completion** - 确保它真的修复了

**协作**
- **brainstorming** - 苏格拉底式设计细化
- **writing-plans** - 详细实现计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发 subagent 工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带两阶段 review（规范合规，然后代码质量）的快速迭代

**元**
- **writing-skills** - 按最佳实践创建新 skills（含测试方法论）
- **using-superpowers** - skills 系统入门

## 哲学

- **测试驱动开发** - 总是先写测试
- **系统化优于随意** - 流程优于猜测
- **降低复杂度** - 以简洁为首要目标
- **证据优于声明** - 宣告成功前先验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

Superpowers 的一般贡献流程如下。请记住，我们通常不接受新 skills 的贡献，且对 skills 的任何更新都必须在我们支持的所有 coding agents 上工作。

1. Fork 仓库
2. 切换到 'dev' 分支
3. 为你的工作创建分支
4. 遵循 `writing-skills` skill 来创建和测试新的及修改的 skills
5. 提交 PR，务必填写 pull request 模板。

Skill 行为测试使用来自 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 的 drill eval 框架，克隆到 `evals/` —— 设置见 `evals/README.md`。插件基础设施测试位于 `tests/`，通过相关的 `run-*.sh` 或 `npm test` 运行。

完整指南见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新在一定程度上取决于 coding agent，但通常是自动的。

## 许可证

MIT License - 详情见 LICENSE 文件

## 视觉伴侣遥测

因为 skills 和插件不会向创建者提供任何反馈，我们不知道你们中有多少人在使用 Superpowers。默认情况下，brainstorming 可选视觉伴侣功能上的 Prime Radiant logo 会从我们的网站加载。它包含正在使用的 Superpowers 版本。它不包含关于你的项目、prompt 或 coding agent 的任何细节。我们看不到你的点击或你正在构建的任何内容。这帮助我们大致了解有多少人在使用 Superpowers 以及他们使用的是哪个版本。它是 100% 可选的。要禁用它，请将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设置为任何真值。Superpowers 也会遵循 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 选择退出选项。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他人构建。

- **Discord**: [加入我们](https://discord.gg/35wsABTejz) 获取社区支持、提问，并分享你用 Superpowers 构建的内容
- **Issues**: https://github.com/obra/superpowers/issues
- **发布公告**: [注册](https://primeradiant.com/superpowers/) 以在新版本发布时收到通知
