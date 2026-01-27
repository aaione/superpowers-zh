# Claude Code 的跨平台多语言 Hooks (Cross-Platform Polyglot Hooks)

Claude Code 插件需要能在 Windows、macOS 和 Linux 上工作的 hooks。本文档解释了实现这一目标的多语言 wrapper（包装器）技术。

## 问题 (The Problem)

Claude Code 通过系统的默认 shell 运行 hook 命令：
- **Windows**: CMD.exe
- **macOS/Linux**: bash 或 sh

这带来了以下挑战：

1. **脚本执行**: Windows CMD 不能直接执行 `.sh` 文件——它会尝试在文本编辑器中打开它们。
2. **路径格式**: Windows 使用反斜杠 (`C:\path`)，Unix 使用正斜杠 (`/path`)。
3. **环境变量**: `$VAR` 语法在 CMD 中不起作用。
4. **PATH 中没有 `bash`**: 即使安装了 Git Bash，当 CMD 运行时，`bash` 也不在 PATH 中。

## 解决方案：多语言 `.cmd` Wrapper

多语言脚本同时在多种语言中都是有效的语法。我们的 wrapper 在 CMD 和 bash 中都有效：

```cmd
: << 'CMDBLOCK'
@echo off
"C:\Program Files\Git\bin\bash.exe" -l -c "\"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")/hooks/session-start.sh\""
exit /b
CMDBLOCK

# Unix shell 从这里开始运行
"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
```

### 工作原理 (How It Works)

#### 在 Windows 上 (CMD.exe)

1. `: << 'CMDBLOCK'` - CMD 将 `:` 视为标签（如 `:label`），并忽略 `<< 'CMDBLOCK'`。
2. `@echo off` - 关闭命令回显。
3. 运行 bash.exe 命令：
   - `-l` (login shell) 以获取带有 Unix 实用程序的正确 PATH。
   - `cygpath -u` 将 Windows 路径转换为 Unix 格式 (`C:\foo` → `/c/foo`)。
4. `exit /b` - 退出批处理脚本，在此停止 CMD。
5. CMD 永远不会到达 `CMDBLOCK` 之后的任何内容。

#### 在 Unix 上 (bash/sh)

1. `: << 'CMDBLOCK'` - `:` 是一个空操作 (no-op)，`<< 'CMDBLOCK'` 开始一个 heredoc。
2. `CMDBLOCK` 之前的所有内容都被 heredoc 消费（被忽略）。
3. `# Unix shell 从这里开始运行` - 注释。
4. 脚本直接使用 Unix 路径运行。

## 文件结构 (File Structure)

```
hooks/
├── hooks.json           # 指向 .cmd wrapper
├── session-start.cmd    # 多语言 wrapper (跨平台入口点)
└── session-start.sh     # 实际的 hook 逻辑 (bash 脚本)
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.cmd\""
          }
        ]
      }
    ]
  }
}
```

注意：路径必须用引号括起来，因为在 Windows 上 `${CLAUDE_PLUGIN_ROOT}` 可能包含空格（例如 `C:\Program Files\...`）。

## 要求 (Requirements)

### Windows
- 必须安装 **Git for Windows** (提供 `bash.exe` 和 `cygpath`)
- 默认安装路径：`C:\Program Files\Git\bin\bash.exe`
- 如果 Git 安装在其他地方，则需要修改 wrapper。

### Unix (macOS/Linux)
- 标准的 bash 或 sh shell。
- `.cmd` 文件必须具有执行权限 (`chmod +x`)。

## 编写跨平台 Hook 脚本

你的实际 hook 逻辑放在 `.sh` 文件中。为了确保它在 Windows 上（通过 Git Bash）工作：

### 应该 (Do):
- 尽可能使用纯 bash 内置函数。
- 使用 `$(command)` 而不是反引号。
- 为所有变量扩展加上引号：`"$VAR"`。
- 使用 `printf` 或 here-docs 进行输出。

### 避免 (Avoid):
- 可能不在 PATH 中的外部命令 (sed, awk, grep)。
- 如果你必须使用它们，它们在 Git Bash 中是可用的，但要确保设置了 PATH（使用 `bash -l`）。

### 示例：不使用 sed/awk 的 JSON 转义

不要使用：
```bash
escaped=$(echo "$content" | sed 's/\\/\\\\/g' | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
```

使用纯 bash：
```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 可重用的 Wrapper 模式

对于具有多个 hooks 的插件，你可以创建一个通用的 wrapper，将脚本名称作为参数：

### run-hook.cmd
```cmd
: << 'CMDBLOCK'
@echo off
set "SCRIPT_DIR=%~dp0"
set "SCRIPT_NAME=%~1"
"C:\Program Files\Git\bin\bash.exe" -l -c "cd \"$(cygpath -u \"%SCRIPT_DIR%\")\" && \"./%SCRIPT_NAME%\""
exit /b
CMDBLOCK

# Unix shell 从这里开始运行
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]:-$0}")" && pwd)"
SCRIPT_NAME="$1"
shift
"${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### 使用可重用 wrapper 的 hooks.json
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

## 故障排除 (Troubleshooting)

### "bash is not recognized"
CMD 找不到 bash。wrapper 使用完整路径 `C:\Program Files\Git\bin\bash.exe`。如果 Git 安装在其他地方，请更新路径。

### "cygpath: command not found" 或 "dirname: command not found"
Bash 未作为 login shell 运行。确保使用了 `-l` 标志。

### 路径中包含奇怪的 `\/`
`${CLAUDE_PLUGIN_ROOT}` 被展开为以反斜杠结尾的 Windows 路径，然后附加了 `/hooks/...`。使用 `cygpath` 转换整个路径。

### 脚本在文本编辑器中打开而不是运行
hooks.json 指向了 `.sh` 文件。请改为指向 `.cmd` wrapper。

### 在终端中工作但作为 hook 不工作
Claude Code 运行 hooks 的方式可能不同。通过模拟 hook 环境进行测试：
```powershell
$env:CLAUDE_PLUGIN_ROOT = "C:\path\to\plugin"
cmd /c "C:\path\to\plugin\hooks\session-start.cmd"
```

## 相关问题 (Related Issues)

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) - .sh 脚本在 Windows 上在编辑器中打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) - Hooks 在 Windows 上不起作用
- [anthropics/claude-code#6023](https://github.com/anthropics/claude-code/issues/6023) - 找不到 CLAUDE_PROJECT_DIR
