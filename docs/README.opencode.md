# Superpowers for OpenCode

使用 [OpenCode.ai](https://opencode.ai) 的 Superpowers 完整指南。

## 安装 (Installation)

在你的 `opencode.json`（全局或项目级）的 `plugin` 数组中添加 superpowers:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。插件通过 OpenCode 的插件管理器安装并注册所有 skills。

通过询问验证: "Tell me about your superpowers"

OpenCode 使用自己的插件安装方式。如果你也使用 Claude Code、Codex 或其他 harness，请分别为每个平台单独安装 Superpowers。

### 从旧的基于 symlink 的安装迁移

如果你之前使用 `git clone` 和 symlinks 安装了 superpowers，请移除旧的设置:

```bash
# 移除旧的 symlinks
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选地移除克隆的仓库
rm -rf ~/.config/opencode/superpowers

# 如果你为 superpowers 添加了 skills.paths，从 opencode.json 中移除
```

然后按照上面的安装步骤操作。

## 用法 (Usage)

### 查找 Skills (Finding Skills)

使用 OpenCode 的原生 `skill` 工具列出所有可用的 skills:

```
use skill tool to list skills
```

### 加载 Skill (Loading a Skill)

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

在你的项目中的 `.opencode/skills/` 目录中创建项目特定的 skills。

**Skill 优先级:** Project skills > Personal skills > Superpowers skills

## 更新 (Updating)

OpenCode 通过 git 支持的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会在 lockfile 或缓存中固定已解析的 git 依赖，因此重启可能无法获取最新的 Superpowers 提交。如果更新未出现，清除 OpenCode 的包缓存或重新安装插件。

要固定到特定版本，使用分支或标签:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 工作原理 (How It Works)

插件做两件事:

1. **注入 bootstrap 上下文** 通过 `experimental.chat.system.transform` hook，为每个对话添加 superpowers 感知。
2. **注册 skills 目录** 通过 `config` hook，使 OpenCode 无需 symlinks 或手动配置即可发现所有 superpowers skills。

### 工具映射 (Tool Mapping)

为 Claude Code 编写的 Skills 会自动适配 OpenCode:

- `TodoWrite` → `todowrite`
- `Task` with subagents → OpenCode's `@mention` system
- `Skill` tool → OpenCode's native `skill` tool
- File operations → Native OpenCode tools

## 故障排除 (Troubleshooting)

### Plugin 未加载

1. 检查 OpenCode 日志: `opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证你的 `opencode.json` 中的插件行是否正确
3. 确保你运行的是最新版本的 OpenCode

### Windows 安装问题

某些 Windows OpenCode 构建在 git 支持的插件规范方面存在上游安装器问题，包括 `git+https` URL 的缓存路径和 Bun 无法找到 `git.exe`（即使在正常终端中可以工作）。如果 OpenCode 无法安装插件，尝试使用系统 npm 安装并指向本地包:

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中使用已安装的包路径:

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### Skills 未找到

1. 使用 OpenCode 的 `skill` 工具列出可用的 skills
2. 检查插件是否正在加载（见上文）
3. 每个 skill 需要一个具有有效 YAML frontmatter 的 `SKILL.md` 文件

### Bootstrap 未出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.system.transform` hook
2. 配置更改后重启 OpenCode

## 获取帮助 (Getting Help)

- 报告问题: https://github.com/obra/superpowers/issues
- 主要文档: https://github.com/obra/superpowers
- OpenCode 文档: https://opencode.ai/docs/
