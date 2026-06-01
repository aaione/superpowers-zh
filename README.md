# Superpowers

Superpowers 是一个完整的软件开发方法论，专为你的 coding agents 打造，建立在一套可组合的 skills 和一些确保你的 agent 使用它们的初始指令之上。

## 快速开始

赋予你的 agent Superpowers：[Claude Code](#claude-code)、[Codex CLI](#codex-cli)、[Codex App](#codex-app)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[OpenCode](#opencode)、[Cursor](#cursor)、[GitHub Copilot CLI](#github-copilot-cli)。

## 工作原理

它从你启动 coding agent 的那一刻就开始运作。只要它发现你正在构建某些东西，它*不会*直接跳进去尝试写代码。相反，它会退一步，询问你到底想要做什么。

一旦它从对话中梳理出规范说明，它会将其分成足够短的块展示给你，以便你真正阅读和消化。

在你批准设计方案后，你的 agent 会制定一个实施计划，这个计划足够清晰，即使是一个热情但品味差、没有判断力、没有项目背景且不喜欢测试的初级工程师也能遵循。它强调真正的红/绿 TDD、YAGNI（You Aren't Gonna Need It）和 DRY。

接下来，一旦你说 "go"，它就会启动一个 *subagent-driven-development* 流程，让 agents 逐个完成工程任务，检查和审查他们的工作，然后继续前进。Claude 能够在遵循你制定的计划的情况下，自主工作数小时而不偏离方向，这并不罕见。

还有很多其他功能，但这是系统的核心。而且因为 skills 会自动触发，你不需要做任何特殊操作。你的 coding agent 自然就拥有了 Superpowers。

## 安装

安装方式因 harness 而异。如果你使用多个平台，请分别为每个平台安装 Superpowers。

### Claude Code

Superpowers 可通过 [Claude 官方插件市场](https://claude.com/plugins/superpowers) 获取。

#### 官方市场

- 从 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场为 Claude Code 提供 Superpowers 及其他相关插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从该市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Codex CLI

Superpowers 可通过 [Codex 官方插件市场](https://github.com/openai/plugins) 获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Codex App

Superpowers 可通过 [Codex 官方插件市场](https://github.com/openai/plugins) 获取。

- 在 Codex App 中，点击侧边栏中的 Plugins。
- 你应该能在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。

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

- 后续更新：

  ```bash
  gemini extensions update superpowers
  ```

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他 harness 中使用，也需要单独为 OpenCode 安装 Superpowers。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或者在插件市场中搜索 "superpowers"。

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

## 基本工作流

1. **brainstorming** - 在编写代码之前激活。通过提问完善粗略想法，探索替代方案，分段展示设计以供验证。保存设计文档。

2. **using-git-worktrees** - 在设计获批后激活。在新分支上创建隔离工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 在设计获批后激活。将工作分解为小粒度任务（每个 2-5 分钟）。每个任务都有精确的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 在计划获批后激活。为每个任务 dispatch 新的 subagent 进行两阶段审查（先是规范合规性，然后是代码质量），或分批执行并设有人工检查点。

5. **test-driven-development** - 在实施过程中激活。强制执行 RED-GREEN-REFACTOR：编写失败测试，观察其失败，编写最少量的代码，观察其通过，提交。删除在测试之前编写的代码。

6. **requesting-code-review** - 在任务之间激活。对照计划进行审查，按严重程度报告问题。关键问题会阻止进度。

7. **finishing-a-development-branch** - 在任务完成后激活。验证测试，提供选项（合并/PR/保留/丢弃），清理 worktree。

**Agent 在执行任何任务之前都会检查相关的 skills。** 这是强制性工作流，不是建议。

## 包含内容

### Skills 库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包括测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根本原因查找流程（包括 root-cause-tracing、defense-in-depth、condition-based-waiting 技术）
- **verification-before-completion** - 确保问题确实已修复

**协作**
- **brainstorming** - 苏格拉底式设计细化
- **writing-plans** - 详细的实施计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发 subagent 工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 响应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带两阶段审查（规范合规性，然后代码质量）的快速迭代

**元**
- **writing-skills** - 遵循最佳实践创建新 skills（包括测试方法论）
- **using-superpowers** - skills 系统入门

## 理念

- **Test-Driven Development** - 总是先写测试
- **Systematic over ad-hoc** - 流程优于猜测
- **Complexity reduction** - 简洁是首要目标
- **Evidence over claims** - 在宣布成功之前先验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

Superpowers 的一般贡献流程如下。请注意，我们通常不接受新 skills 的贡献，任何对 skills 的更新都必须在我们支持的所有 coding agents 上正常工作。

1. Fork 本仓库
2. 切换到 'dev' 分支
3. 为你的工作创建一个分支
4. 遵循 `writing-skills` skill 来创建和测试新的和修改的 skills
5. 提交 PR，确保填写 pull request 模板

参见 `skills/writing-skills/SKILL.md` 获取完整指南。

## 更新

Superpowers 的更新在某种程度上取决于 coding agent，但通常是自动的。

## 许可证

MIT License - 详情请参见 LICENSE 文件

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他成员共同构建。

- **Discord**: [加入我们](https://discord.gg/35wsABTejz) 获取社区支持、提问和分享你用 Superpowers 构建的内容
- **Issues**: https://github.com/obra/superpowers/issues
- **版本发布公告**: [注册](https://primeradiant.com/superpowers/) 以获取新版本通知
