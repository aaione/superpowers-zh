# 为 OpenCode 安装 Superpowers

## 先决条件 (Prerequisites)

- 已安装 [OpenCode.ai](https://opencode.ai)

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

使用 OpenCode 的原生 `skill` 工具:

```
use skill tool to list skills
use skill tool to load superpowers/brainstorming
```

## 更新 (Updating)

OpenCode 通过 git 支持的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会在 lockfile 或缓存中固定已解析的 git 依赖，因此重启可能无法获取最新的 Superpowers 提交。如果更新未出现，清除 OpenCode 的包缓存或重新安装插件。

要固定到特定版本:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 故障排除 (Troubleshooting)

### Plugin 未加载

1. 检查日志: `opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
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

1. 使用 `skill` 工具列出已发现的内容
2. 检查插件是否正在加载（见上文）

### 工具映射 (Tool mapping)

当 skills 引用 Claude Code 工具时:
- `TodoWrite` → `todowrite`
- `Task` with subagents → `@mention` syntax
- `Skill` tool → OpenCode's native `skill` tool
- File operations → your native tools

## 获取帮助 (Getting Help)

- 报告问题: https://github.com/obra/superpowers/issues
- 完整文档: https://github.com/obra/superpowers/blob/main/docs/README.opencode.md
