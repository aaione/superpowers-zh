# Superpowers for Codex

使用 OpenAI Codex 的 Superpowers 完整指南。

## 快速安装 (Quick Install)

告诉 Codex:

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

## 手动安装 (Manual Installation)

### 先决条件 (Prerequisites)

- OpenAI Codex 访问权限
- 安装文件的 Shell 访问权限

### 安装步骤 (Installation Steps)

#### 1. Clone Superpowers

```bash
mkdir -p ~/.codex/superpowers
git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
```

#### 2. 安装 Bootstrap

Bootstrap 文件包含在仓库中的 `.codex/superpowers-bootstrap.md`。Codex 将自动从克隆的位置使用它。

#### 3. 验证安装 (Verify Installation)

告诉 Codex:

```
Run ~/.codex/superpowers/.codex/superpowers-codex find-skills to show available skills
```

你应该看到带有描述的可用 skills 列表。

## 用法 (Usage)

### 查找 Skills (Finding Skills)

```
Run ~/.codex/superpowers/.codex/superpowers-codex find-skills
```

### 加载 Skill (Loading a Skill)

```
Run ~/.codex/superpowers/.codex/superpowers-codex use-skill superpowers:brainstorming
```

### Bootstrap 所有 Skills

```
Run ~/.codex/superpowers/.codex/superpowers-codex bootstrap
```

这将加载包含所有 skill 信息的完整 bootstrap。

### 个人 Skills (Personal Skills)

在 `~/.codex/skills/` 中创建你自己的 skills:

```bash
mkdir -p ~/.codex/skills/my-skill
```

创建 `~/.codex/skills/my-skill/SKILL.md`:

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

Personal skills 会覆盖同名的 superpowers skills。

## 架构 (Architecture)

### Codex CLI Tool

**位置:** `~/.codex/superpowers/.codex/superpowers-codex`

一个提供三个命令的 Node.js CLI 脚本:
- `bootstrap` - 加载包含所有 skills 的完整 bootstrap
- `use-skill <name>` - 加载特定的 skill
- `find-skills` - 列出所有可用的 skills

### 共享核心模块 (Shared Core Module)

**位置:** `~/.codex/superpowers/lib/skills-core.js`

Codex 实现使用共享的 `skills-core` 模块 (ES module 格式) 进行 skill 发现和解析。这与 OpenCode 插件使用的模块相同，确保存跨平台的一致行为。

### 工具映射 (Tool Mapping)

为 Claude Code 编写的 Skills 针对 Codex 进行了以下映射适配:

- `TodoWrite` → `update_plan`
- `Task` with subagents → 告诉用户 subagents 不可用，直接做工作
- `Skill` tool → `~/.codex/superpowers/.codex/superpowers-codex use-skill`
- File operations → Native Codex tools

## 更新 (Updating)

```bash
cd ~/.codex/superpowers
git pull
```

## 故障排除 (Troubleshooting)

### Skills 未找到

1. 验证安装: `ls ~/.codex/superpowers/skills`
2. 检查 CLI 是否工作: `~/.codex/superpowers/.codex/superpowers-codex find-skills`
3. 验证 skills 是否有 SKILL.md 文件

### CLI 脚本不可执行

```bash
chmod +x ~/.codex/superpowers/.codex/superpowers-codex
```

### Node.js 错误

CLI 脚本需要 Node.js。验证:

```bash
node --version
```

应显示 v14 或更高版本 (推荐 v18+ 以支持 ES module)。

## 获取帮助 (Getting Help)

- 报告问题: https://github.com/obra/superpowers/issues
- 主要文档: https://github.com/obra/superpowers
- 博客文章: https://blog.fsck.com/2025/10/27/skills-for-openai-codex/

## 注意 (Note)

Codex 支持是实验性的，可能需要根据用户反馈进行改进。如果你遇到问题，请在 GitHub 上报告。
