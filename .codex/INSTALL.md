# 为 Codex 安装 Superpowers

快速设置以在 Codex 中启用 superpowers skills。

## 安装 (Installation)

1. **Clone superpowers repository**:
   ```bash
   mkdir -p ~/.codex/superpowers
   cd ~/.codex/superpowers
   git clone https://github.com/obra/superpowers.git .
   ```

2. **创建个人 skills 目录**:
   ```bash
   mkdir -p ~/.codex/skills
   ```

3. **更新 ~/.codex/AGENTS.md** 以包含此 superpowers 部分:
   ```markdown
   ## Superpowers System

   <EXTREMELY_IMPORTANT>
   You have superpowers. Superpowers teach you new skills and capabilities. RIGHT NOW run: `~/.codex/superpowers/.codex/superpowers-codex bootstrap` and follow the instructions it returns.
   </EXTREMELY_IMPORTANT>
   ```

## 验证 (Verification)

测试安装:
```bash
~/.codex/superpowers/.codex/superpowers-codex bootstrap
```

你应该看到 skill 列表和引导说明。系统现在已准备好使用。