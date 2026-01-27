# Superpowers for OpenCode

使用 [OpenCode.ai](https://opencode.ai) 的 Superpowers 完整指南。

## 快速安装 (Quick Install)

告诉 OpenCode:

```
Clone https://github.com/obra/superpowers to ~/.config/opencode/superpowers, then create directory ~/.config/opencode/plugins, then symlink ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js to ~/.config/opencode/plugins/superpowers.js, then symlink ~/.config/opencode/superpowers/skills to ~/.config/opencode/skills/superpowers, then restart opencode.
```

## 手动安装 (Manual Installation)

### 先决条件 (Prerequisites)

- 已安装 [OpenCode.ai](https://opencode.ai)
- 已安装 Git

### macOS / Linux

```bash
# 1. Install Superpowers (or update existing)
if [ -d ~/.config/opencode/superpowers ]; then
  cd ~/.config/opencode/superpowers && git pull
else
  git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
fi

# 2. Create directories
mkdir -p ~/.config/opencode/plugins ~/.config/opencode/skills

# 3. Remove old symlinks/directories if they exist
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 4. Create symlinks
ln -s ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js ~/.config/opencode/plugins/superpowers.js
ln -s ~/.config/opencode/superpowers/skills ~/.config/opencode/skills/superpowers

# 5. Restart OpenCode
```

#### 验证安装 (Verify Installation)

```bash
ls -l ~/.config/opencode/plugins/superpowers.js
ls -l ~/.config/opencode/skills/superpowers
```

两者都应显示指向 superpowers 目录的 symlinks。

### Windows

**先决条件:**
- 已安装 Git
- 启用 **Developer Mode** 或拥有 **Administrator privileges**
  - Windows 10: Settings → Update & Security → For developers
  - Windows 11: Settings → System → For developers

选择你的 shell: [Command Prompt](#command-prompt) | [PowerShell](#powershell) | [Git Bash](#git-bash)

#### Command Prompt

作为管理员运行，或启用 Developer Mode:

```cmd
:: 1. Install Superpowers
git clone https://github.com/obra/superpowers.git "%USERPROFILE%\.config\opencode\superpowers"

:: 2. Create directories
mkdir "%USERPROFILE%\.config\opencode\plugins" 2>nul
mkdir "%USERPROFILE%\.config\opencode\skills" 2>nul

:: 3. Remove existing links (safe for reinstalls)
del "%USERPROFILE%\.config\opencode\plugins\superpowers.js" 2>nul
rmdir "%USERPROFILE%\.config\opencode\skills\superpowers" 2>nul

:: 4. Create plugin symlink (requires Developer Mode or Admin)
mklink "%USERPROFILE%\.config\opencode\plugins\superpowers.js" "%USERPROFILE%\.config\opencode\superpowers\.opencode\plugins\superpowers.js"

:: 5. Create skills junction (works without special privileges)
mklink /J "%USERPROFILE%\.config\opencode\skills\superpowers" "%USERPROFILE%\.config\opencode\superpowers\skills"

:: 6. Restart OpenCode
```

#### PowerShell

作为管理员运行，或启用 Developer Mode:

```powershell
# 1. Install Superpowers
git clone https://github.com/obra/superpowers.git "$env:USERPROFILE\.config\opencode\superpowers"

# 2. Create directories
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\plugins"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\skills"

# 3. Remove existing links (safe for reinstalls)
Remove-Item "$env:USERPROFILE\.config\opencode\plugins\superpowers.js" -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\.config\opencode\skills\superpowers" -Force -ErrorAction SilentlyContinue

# 4. Create plugin symlink (requires Developer Mode or Admin)
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\plugins\superpowers.js" -Target "$env:USERPROFILE\.config\opencode\superpowers\.opencode\plugins\superpowers.js"

# 5. Create skills junction (works without special privileges)
New-Item -ItemType Junction -Path "$env:USERPROFILE\.config\opencode\skills\superpowers" -Target "$env:USERPROFILE\.config\opencode\superpowers\skills"

# 6. Restart OpenCode
```

#### Git Bash

注意: Git Bash 的原生 `ln` 命令复制文件而不是创建 symlinks。请改用 `cmd //c mklink` (`//c` 是 `/c` 的 Git Bash 语法).

```bash
# 1. Install Superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers

# 2. Create directories
mkdir -p ~/.config/opencode/plugins ~/.config/opencode/skills

# 3. Remove existing links (safe for reinstalls)
rm -f ~/.config/opencode/plugins/superpowers.js 2>/dev/null
rm -rf ~/.config/opencode/skills/superpowers 2>/dev/null

# 4. Create plugin symlink (requires Developer Mode or Admin)
cmd //c "mklink \"$(cygpath -w ~/.config/opencode/plugins/superpowers.js)\" \"$(cygpath -w ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js)\""

# 5. Create skills junction (works without special privileges)
cmd //c "mklink /J \"$(cygpath -w ~/.config/opencode/skills/superpowers)\" \"$(cygpath -w ~/.config/opencode/superpowers/skills)\""

# 6. Restart OpenCode
```

#### WSL 用户

如果在 WSL 内运行 OpenCode，请改用 [macOS / Linux](#macos--linux) 说明。

#### 验证安装 (Verify Installation)

**Command Prompt:**
```cmd
dir /AL "%USERPROFILE%\.config\opencode\plugins"
dir /AL "%USERPROFILE%\.config\opencode\skills"
```

**PowerShell:**
```powershell
Get-ChildItem "$env:USERPROFILE\.config\opencode\plugins" | Where-Object { $_.LinkType }
Get-ChildItem "$env:USERPROFILE\.config\opencode\skills" | Where-Object { $_.LinkType }
```

在输出中查找 `<SYMLINK>` 或 `<JUNCTION>`。

#### Windows 故障排除

**"You do not have sufficient privilege" error:**
- 在 Windows Settings 中启用 Developer Mode, 或
- 右键点击你的终端 → "Run as Administrator"

**"Cannot create a file when that file already exists":**
- 先运行移除命令 (step 3)，然后重试

**Symlinks not working after git clone:**
- 运行 `git config --global core.symlinks true` 并重新 clone

## 用法 (Usage)

### 查找 Skills

使用 OpenCode 的原生 `skill` 工具列出所有可用的 skills:

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

在你的 OpenCode 项目中创建项目特定的 skills:

```bash
# In your OpenCode project
mkdir -p .opencode/skills/my-project-skill
```

创建 `.opencode/skills/my-project-skill/SKILL.md`:

```markdown
---
name: my-project-skill
description: Use when [condition] - [what it does]
---

# My Project Skill

[Your skill content here]
```

## Skill 位置

OpenCode 从以下位置发现 skills:

1. **项目 skills** (`.opencode/skills/`) - 最高优先级
2. **个人 skills** (`~/.config/opencode/skills/`)
3. **Superpowers skills** (`~/.config/opencode/skills/superpowers/`) - via symlink

## 特性 (Features)

### 自动上下文注入 (Automatic Context Injection)

Plugin 通过 `experimental.chat.system.transform` hook 自动注入 superpowers 上下文。这会在每个请求的 system prompt 中添加 "using-superpowers" skill 内容。

### 原生 Skills 集成

Superpowers 使用 OpenCode 的原生 `skill` 工具进行 skill 发现和加载。Skills 被 symlinked 到 `~/.config/opencode/skills/superpowers/`，因此它们与你的个人和项目 skills 一起出现。

### 工具映射 (Tool Mapping)

为 Claude Code 编写的 Skills 会自动适配 OpenCode。Bootstrap 提供了映射说明:

- `TodoWrite` → `update_plan`
- `Task` with subagents → OpenCode's `@mention` system
- `Skill` tool → OpenCode's native `skill` tool
- File operations → Native OpenCode tools

## 架构 (Architecture)

### Plugin 结构

**位置:** `~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`

**组件:**
- `experimental.chat.system.transform` hook 用于 bootstrap 注入
- 读取并注入 "using-superpowers" skill 内容

### Skills

**位置:** `~/.config/opencode/skills/superpowers/` (symlink to `~/.config/opencode/superpowers/skills/`)

Skills 由 OpenCode 的原生 skill 系统发现。每个 skill 都有一个带有 YAML frontmatter 的 `SKILL.md` 文件。

## 更新 (Updating)

```bash
cd ~/.config/opencode/superpowers
git pull
```

重启 OpenCode 以加载更新。

## 故障排除 (Troubleshooting)

### Plugin 未加载

1. 检查 plugin 是否存在: `ls ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`
2. 检查 symlink/junction: `ls -l ~/.config/opencode/plugins/` (macOS/Linux) or `dir /AL %USERPROFILE%\.config\opencode\plugins` (Windows)
3. 检查 OpenCode 日志: `opencode run "test" --print-logs --log-level DEBUG`
4. 在日志中查找 plugin 加载消息

### Skills 未找到

1. 验证 skills symlink: `ls -l ~/.config/opencode/skills/superpowers` (should point to superpowers/skills/)
2. 使用 OpenCode 的 `skill` 工具列出可用的 skills
3. 检查 skill 结构: 每个 skill 需要一个具有有效 frontmatter 的 `SKILL.md` 文件

### Windows: Module not found error

如果你在 Windows 上看到 `Cannot find module` 错误:
- **原因:** Git Bash `ln -sf` 复制文件而不是创建 symlinks
- **修复:** 改用 `mklink /J` directory junctions (见 Windows 安装步骤)

### Bootstrap 未出现

1. 验证 using-superpowers skill 是否存在: `ls ~/.config/opencode/superpowers/skills/using-superpowers/SKILL.md`
2. 检查 OpenCode 版本是否支持 `experimental.chat.system.transform` hook
3. Plugin 更改后重启 OpenCode

## 获取帮助 (Getting Help)

- 报告问题: https://github.com/obra/superpowers/issues
- 主要文档: https://github.com/obra/superpowers
- OpenCode 文档: https://opencode.ai/docs/

## 测试 (Testing)

验证你的安装:

```bash
# Check plugin loads
opencode run --print-logs "hello" 2>&1 | grep -i superpowers

# Check skills are discoverable
opencode run "use skill tool to list all skills" 2>&1 | grep -i superpowers

# Check bootstrap injection
opencode run "what superpowers do you have?"
```

Agent 应该提到拥有 superpowers 并且能够从 `superpowers/` 列出 skills。
