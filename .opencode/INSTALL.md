# 为 OpenCode 安装 Superpowers

## 先决条件 (Prerequisites)

- 已安装 [OpenCode.ai](https://opencode.ai)
- 已安装 Git

## 安装步骤 (Installation Steps)

### 1. Clone Superpowers

```bash
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. 注册 Plugin

创建一个 symlink 以便 OpenCode 发现 plugin:

```bash
mkdir -p ~/.config/opencode/plugins
rm -f ~/.config/opencode/plugins/superpowers.js
ln -s ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js ~/.config/opencode/plugins/superpowers.js
```

### 3. Symlink Skills

创建一个 symlink 以便 OpenCode 的原生 skill 工具发现 superpowers skills:

```bash
mkdir -p ~/.config/opencode/skills
rm -rf ~/.config/opencode/skills/superpowers
ln -s ~/.config/opencode/superpowers/skills ~/.config/opencode/skills/superpowers
```

### 4. 重启 OpenCode

重启 OpenCode。Plugin 将自动注入 superpowers 上下文。

通过询问验证: "do you have superpowers?"

## 用法 (Usage)

### 查找 Skills

使用 OpenCode 的原生 `skill` 工具列出可用的 skills:

```
use skill tool to list skills
```

### 加载 Skill

使用 OpenCode 的原生 `skill` 工具加载特定的 skill:

```
use skill tool to load superpowers/brainstorming
```

### 个人 Skills (Personal Skills)

在 `~/.config/opencode/skills/` 中创建你自己的 skills:

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`:

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目 Skills (Project Skills)

在你项目内的 `.opencode/skills/` 中创建项目特定的 skills。

**Skill 优先级:** Project skills > Personal skills > Superpowers skills

## 更新 (Updating)

```bash
cd ~/.config/opencode/superpowers
git pull
```

## 故障排除 (Troubleshooting)

### Plugin 未加载

1. 检查 plugin symlink: `ls -l ~/.config/opencode/plugins/superpowers.js`
2. 检查源是否存在: `ls ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`
3. 检查 OpenCode 日志以查找错误

### Skills 未找到

1. 检查 skills symlink: `ls -l ~/.config/opencode/skills/superpowers`
2. 验证它指向: `~/.config/opencode/superpowers/skills`
3. 使用 `skill` 工具列出已发现的内容

### 工具映射 (Tool mapping)

当 skills 引用 Claude Code 工具时:
- `TodoWrite` → `update_plan`
- `Task` with subagents → `@mention` syntax
- `Skill` tool → OpenCode's native `skill` tool
- File operations → your native tools

## 获取帮助 (Getting Help)

- 报告问题: https://github.com/obra/superpowers/issues
- 完整文档: https://github.com/obra/superpowers/blob/main/docs/README.opencode.md
