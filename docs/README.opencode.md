# 适用于 OpenCode 的 Superpowers (Superpowers for OpenCode)

使用 [OpenCode.ai](https://opencode.ai) 配合 Superpowers 的完整指南。

## 安装

将 superpowers 添加到你的 `opencode.json`（全局或项目级）的 `plugin` 数组中：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。插件通过 OpenCode 的插件管理器安装并注册所有 skills。

通过提问来验证："Tell me about your superpowers"

OpenCode 使用其自身的插件安装。如果你同时使用 Claude Code、Codex 或其他 harness，请为每一个分别安装 Superpowers。

### 从旧的基于符号链接的安装迁移

如果你此前使用 `git clone` 和符号链接安装了 superpowers，请移除旧的设置：

```bash
# 移除旧的符号链接
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选：移除克隆的仓库
rm -rf ~/.config/opencode/superpowers

# 如果你曾为 superpowers 在 opencode.json 中添加过 skills.paths，请移除它
```

然后按照上面的安装步骤操作。

## 用法

### 查找 Skills

使用 OpenCode 原生的 `skill` 工具列出所有可用 skills：

```
use skill tool to list skills
```

### 加载一个 Skill

```
use skill tool to load brainstorming
```

### 个人 Skills

在 `~/.config/opencode/skills/` 中创建你自己的 skills：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目 Skills

在项目内的 `.opencode/skills/` 中创建项目专用 skills。

**Skill 优先级：** 项目 skills > 个人 skills > Superpowers skills

## 更新

OpenCode 通过基于 git 的包规格安装 Superpowers。某些 OpenCode 和 Bun 版本会将解析后的 git 依赖锁定在 lockfile 或缓存中，因此重启可能无法获取最新的 Superpowers 提交。如果更新没有出现，请清除 OpenCode 的包缓存或重新安装插件。

要锁定特定版本，使用分支或标签：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 工作原理

该插件做两件事：

1. **注入 bootstrap 上下文**：通过 `experimental.chat.messages.transform` hook，为每个会话添加 superpowers 感知。
2. **注册 skills 目录**：通过 `config` hook，使 OpenCode 能发现所有 superpowers skills，无需符号链接或手动配置。

### 工具映射

Skills 以动作表达，而非命名某个运行时的工具。在 OpenCode 上它们解析为：

- "Create a todo"（创建 todo）/ "mark complete in todo list"（在 todo 列表中标记完成） → `todowrite`
- `Subagent (general-purpose):` 模板 → OpenCode 的 `task` 工具，使用 `subagent_type: "general"`（或用 `"explore"` 进行代码库探索）
- "Invoke a skill"（调用一个 skill） → OpenCode 原生的 `skill` 工具
- "Read a file"（读文件） → `read`
- "Create a file"（创建文件）/ "edit a file"（编辑文件）/ "delete a file"（删除文件） → `apply_patch`
- "Run a shell command"（运行 shell 命令） → `bash`
- "Search file contents"（搜索文件内容）/ "find files by name"（按名称查找文件） → `grep`、`glob`
- "Fetch a URL"（获取 URL） → `webfetch`

（已对照已安装的 OpenCode CLI 工具清单验证。）

## 故障排查

### 插件未加载

1. 检查 OpenCode 日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证 `opencode.json` 中的插件行是否正确
3. 确保你运行的是较新版本的 OpenCode

### Windows 安装问题

某些 Windows 版本的 OpenCode 构建在基于 git 的插件规格上存在上游安装器问题，包括 `git+https` URL 的缓存路径问题，以及 Bun 在普通终端中能正常工作时却找不到 `git.exe` 的问题。如果 OpenCode 无法安装该插件，请尝试用系统 npm 安装，并让 OpenCode 指向本地包：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中使用已安装的包路径：

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### 找不到 Skills

1. 使用 OpenCode 的 `skill` 工具列出可用 skills
2. 检查插件是否正在加载（见上文）
3. 每个 skill 都需要一个带有效 YAML frontmatter 的 `SKILL.md` 文件

### Bootstrap 未出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.messages.transform` hook
2. 配置变更后重启 OpenCode

## 获取帮助

- 反馈问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode 文档：https://opencode.ai/docs/
