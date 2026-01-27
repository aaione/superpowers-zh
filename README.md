# Superpowers

Superpowers 是一个完整的软件开发工作流，专为你的 coding agents 打造，建立在一套可组合的 "skills" 和一些确保你的 agent 使用它们的初始指令之上。

## 工作原理 (How it works)

它从你启动 coding agent 的那一刻就开始运作。只要它发现你正在构建某些东西，它*不会*直接跳进去尝试写代码。相反，它会退一步，询问你到底想要做什么。

一旦它从对话中梳理出规范，它会将其分成足够短的块展示给你，以便你阅读和消化。

在你签署设计方案后，你的 agent 会制定一个实施计划，这个计划足够清晰，即使是一个品味差、没有判断力、没有项目背景且不喜欢测试的热情初级工程师也能遵循。它强调真正的红/绿 TDD、YAGNI (You Aren't Gonna Need It) 和 DRY。

接下来，一旦你说 "go"，它就会启动一个 *subagent-driven-development* 流程，让 agents 完成每个工程任务，检查和审查他们的工作，然后继续前进。Claude 能够在遵循你制定的计划的情况下，自主工作几个小时而不偏离方向，这种情况并不少见。

还有很多其他功能，但这是系统的核心。而且因为 skills 会自动触发，你不需要做任何特殊的事情。你的 coding agent 直觉般地拥有了 Superpowers。


## 赞助 (Sponsorship)

如果 Superpowers 帮助你完成了赚钱的事情，并且你也愿意，如果你能考虑 [赞助我的开源工作](https://github.com/sponsors/obra)，我将不胜感激。

谢谢！

- Jesse


## 安装 (Installation)

**注意：** 不同平台的安装方式不同。Claude Code 有一个内置的插件系统。Codex 和 OpenCode 需要手动设置。

### Claude Code (通过 Plugin Marketplace)

在 Claude Code 中，首先注册 marketplace：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

然后从这个 marketplace 安装插件：

```bash
/plugin install superpowers@superpowers-marketplace
```

### 验证安装 (Verify Installation)

检查命令是否出现：

```bash
/help
```

```
# Should see:
# /superpowers:brainstorm - Interactive design refinement
# /superpowers:write-plan - Create implementation plan
# /superpowers:execute-plan - Execute plan in batches
```

### Codex

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

**详细文档:** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

告诉 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**详细文档:** [docs/README.opencode.md](docs/README.opencode.md)

## 基本工作流 (The Basic Workflow)

1. **brainstorming** - 在写代码之前激活。通过提问完善粗糙的想法，探索替代方案，通过分节展示设计进行验证。保存设计文档。

2. **using-git-worktrees** - 在设计获得批准后激活。在新分支上创建隔离的工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 在获得批准的设计后激活。将工作分解为原本大小的任务（每个 2-5 分钟）。每个任务都有确切的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 根据计划激活。为每个任务分派新的 subagent，并进行两阶段审查（先是规范合规性，然后是代码质量），或者分批执行并设有人工检查点。

5. **test-driven-development** - 在实施过程中激活。强制执行 RED-GREEN-REFACTOR：编写失败的测试，观察其失败，编写最少量的代码，观察其通过，提交。删除测试之前编写的代码。

6. **requesting-code-review** - 在任务之间激活。根据计划进行审查，按严重程度报告问题。关键问题会阻止进度。

7. **finishing-a-development-branch** - 在任务完成后激活。验证测试，提供选项（合并/PR/保留/丢弃），清理 worktree。

**Agent 在执行任何任务之前都会检查相关的 skills。** 强制性工作流，不仅仅是建议。

## 包含内容 (What's Inside)

### Skills 库 (Skills Library)

**测试 (Testing)**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包括测试反模式参考）

**调试 (Debugging)**
- **systematic-debugging** - 4 阶段根本原因查找流程（包括 root-cause-tracing, defense-in-depth, condition-based-waiting 技术）
- **verification-before-completion** - 确保问题确实已修复

**协作 (Collaboration)**
- **brainstorming** - 苏格拉底式设计完善
- **writing-plans** - 详细的实施计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发 subagent 工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 响应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带有两阶段审查（规范合规性，然后是代码质量）的快速迭代

**元 (Meta)**
- **writing-skills** - 遵循最佳实践创建新 skills（包括测试方法论）
- **using-superpowers** - skills 系统介绍

## 理念 (Philosophy)

- **Test-Driven Development** - 总是先写测试
- **Systematic over ad-hoc** - 流程优于猜测
- **Complexity reduction** - 简单性是首要目标
- **Evidence over claims** - 在宣布成功之前进行验证

阅读更多: [Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## 贡献 (Contributing)

Skills 直接位于此存储库中。要做出贡献：

1. Fork 存储库
2. 为你的 skill 创建一个分支
3. 遵循 `writing-skills` skill 来创建和测试新的 skills
4. 提交 PR

查看 `skills/writing-skills/SKILL.md` 获取完整指南。

## 更新 (Updating)

当你更新插件时，Skills 会自动更新：

```bash
/plugin update superpowers
```

## 许可证 (License)

MIT License - 查看 LICENSE 文件了解详情

## 支持 (Support)

- **Issues**: https://github.com/obra/superpowers/issues
- **Marketplace**: https://github.com/obra/superpowers-marketplace
