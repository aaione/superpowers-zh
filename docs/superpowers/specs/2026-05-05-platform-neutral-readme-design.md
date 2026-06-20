# 平台中立的 README 排序 — Phase C 设计

## 背景

Phase A 和 Phase B（见 `2026-05-05-platform-neutral-prose-design.md` 和 `2026-05-05-platform-neutral-config-refs-design.md`）已经将 README 中的泛化 Claude 叙述和配置文件引用中立化。剩余带有平台倾向的信号在于布局：README 的两个平台列表把 Claude Code 放在首位，其他位置的排序也不严格遵循字母顺序。

本阶段修复排序问题。不涉及叙述文本改动。

## 范围内

1. **Quickstart 平台列表**（`README.md:7`）— 受支持 harness 的内联链接列表
2. **安装章节排序**（`README.md:35–152`）— 按 harness 划分的安装子章节

## 范围外

- 叙述文本、marketplace 名称、plugin ID、URL — 这些都事实正确，保持原样。
- Claude Code 章节的视觉权重（它有两个子章节——官方 Anthropic marketplace 和 Superpowers marketplace）。两者都是真实的安装路径；合并它们会隐藏准确的信息。
- 各安装块内部的章节标题和内容 — 仅改变块的顺序。

## 替换

两个列表都重新排序为严格的字母序：

| 旧顺序 | 新顺序 |
|-----------|-----------|
| Claude Code | Claude Code |
| Codex CLI | Codex App |
| Codex App | Codex CLI |
| Factory Droid | Cursor |
| Gemini CLI | Factory Droid |
| OpenCode | Gemini CLI |
| Cursor | GitHub Copilot CLI |
| GitHub Copilot CLI | OpenCode |

三次移动：Codex App 与 Codex CLI 互换；Cursor 上移两位；GitHub Copilot CLI 上移一位。

Claude Code 因字母序巧合仍排在首位（`Cl…` 排在 `Co…` 之前）。

## 提交计划

一个原子提交涵盖两个列表，因为只改其中一个会在 quickstart 和安装章节之间造成不一致。

## 验证

- Quickstart 锚点（`#claude-code`、`#codex-app` 等）仍然解析到既有的 `### …` 标题——没有标题被重命名。
- 每个安装子章节的正文在改动前后逐字节相同；仅位置发生变化。
- `git diff README.md` 仅显示章节移动，无内容编辑。
