# OpenCode 支持实施计划

> **供 Claude 使用:** 必需子技能: 使用 superpowers:executing-plans 逐任务实施此计划。

**目标:** 使用原生 JavaScript 插件为 OpenCode.ai 添加完整的 superpowers 支持，该插件与现有 Codex 实现共享核心功能。

**架构:** 将通用 skill 发现/解析逻辑提取到 `lib/skills-core.js`，重构 Codex 以使用它，然后使用其原生插件 API、自定义工具和 session hooks 构建 OpenCode 插件。

**技术栈:** Node.js, JavaScript, OpenCode Plugin API, Git worktrees

---

## 阶段 1: 创建共享核心模块

### 任务 1: 提取 Frontmatter 解析

**文件:**
- 创建: `lib/skills-core.js`
- 参考: `.codex/superpowers-codex` (第 40-74 行)

**步骤 1: 创建 lib/skills-core.js 与 extractFrontmatter 函数**

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

/**
 * 从 skill 文件中提取 YAML frontmatter。
 * 当前格式:
 * ---
 * name: skill-name
 * description: Use when [condition] - [what it does]
 * ---
 *
 * @param {string} filePath - SKILL.md 文件路径
 * @returns {{name: string, description: string}}
 */
function extractFrontmatter(filePath) {
    try {
        const content = fs.readFileSync(filePath, 'utf8');
        const lines = content.split('\n');

        let inFrontmatter = false;
        let name = '';
        let description = '';

        for (const line of lines) {
            if (line.trim() === '---') {
                if (inFrontmatter) break;
                inFrontmatter = true;
                continue;
            }

            if (inFrontmatter) {
                const match = line.match(/^(\w+):\s*(.*)$/);
                if (match) {
                    const [, key, value] = match;
                    switch (key) {
                        case 'name':
                            name = value.trim();
                            break;
                        case 'description':
                            description = value.trim();
                            break;
                    }
                }
            }
        }

        return { name, description };
    } catch (error) {
        return { name: '', description: '' };
    }
}

module.exports = {
    extractFrontmatter
};
```

**步骤 2: 验证文件已创建**

运行: `ls -l lib/skills-core.js`
预期: 文件存在

**步骤 3: 提交**

```bash
git add lib/skills-core.js
git commit -m "feat: create shared skills core module with frontmatter parser"
```

---

### 任务 2: 提取 Skill 发现逻辑

**文件:**
- 修改: `lib/skills-core.js`
- 参考: `.codex/superpowers-codex` (第 97-136 行)

**步骤 1: 将 findSkillsInDir 函数添加到 skills-core.js**

在 `module.exports` 之前添加：

```javascript
/**
 * 在目录中递归查找所有 SKILL.md 文件。
 *
 * @param {string} dir - 要搜索的目录
 * @param {string} sourceType - 'personal' 或 'superpowers' 用于命名空间
 * @param {number} maxDepth - 最大递归深度（默认: 3）
 * @returns {Array<{path: string, name: string, description: string, sourceType: string}>}
 */
function findSkillsInDir(dir, sourceType, maxDepth = 3) {
    const skills = [];

    if (!fs.existsSync(dir)) return skills;

    function recurse(currentDir, depth) {
        if (depth > maxDepth) return;

        const entries = fs.readdirSync(currentDir, { withFileTypes: true });

        for (const entry of entries) {
            const fullPath = path.join(currentDir, entry.name);

            if (entry.isDirectory()) {
                // 检查此目录中的 SKILL.md
                const skillFile = path.join(fullPath, 'SKILL.md');
                if (fs.existsSync(skillFile)) {
                    const { name, description } = extractFrontmatter(skillFile);
                    skills.push({
                        path: fullPath,
                        skillFile: skillFile,
                        name: name || entry.name,
                        description: description || '',
                        sourceType: sourceType
                    });
                }

                // 递归进入子目录
                recurse(fullPath, depth + 1);
            }
        }
    }

    recurse(dir, 0);
    return skills;
}
```

**步骤 2: 更新 module.exports**

替换导出行为：

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir
};
```

**步骤 3: 验证语法**

运行: `node -c lib/skills-core.js`
预期: 无输出（成功）

**步骤 4: 提交**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill discovery function to core module"
```

---

### 任务 3: 提取 Skill 解析逻辑

**文件:**
- 修改: `lib/skills-core.js`
- 参考: `.codex/superpowers-codex` (第 212-280 行)

**步骤 1: 添加 resolveSkillPath 函数**

在 `module.exports` 之前添加：

```javascript
/**
 * 将 skill 名称解析为其文件路径，处理 shadowing
 *（个人 skills 覆盖 superpowers skills）。
 *
 * @param {string} skillName - 名称如 "superpowers:brainstorming" 或 "my-skill"
 * @param {string} superpowersDir - superpowers skills 目录路径
 * @param {string} personalDir - 个人 skills 目录路径
 * @returns {{skillFile: string, sourceType: string, skillPath: string} | null}
 */
function resolveSkillPath(skillName, superpowersDir, personalDir) {
    // 如果存在，去除 superpowers: 前缀
    const forceSuperpowers = skillName.startsWith('superpowers:');
    const actualSkillName = forceSuperpowers ? skillName.replace(/^superpowers:/, '') : skillName;

    // 首先尝试个人 skills（除非显式 superpowers:）
    if (!forceSuperpowers && personalDir) {
        const personalPath = path.join(personalDir, actualSkillName);
        const personalSkillFile = path.join(personalPath, 'SKILL.md');
        if (fs.existsSync(personalSkillFile)) {
            return {
                skillFile: personalSkillFile,
                sourceType: 'personal',
                skillPath: actualSkillName
            };
        }
    }

    // 尝试 superpowers skills
    if (superpowersDir) {
        const superpowersPath = path.join(superpowersDir, actualSkillName);
        const superpowersSkillFile = path.join(superpowersPath, 'SKILL.md');
        if (fs.existsSync(superpowersSkillFile)) {
            return {
                skillFile: superpowersSkillFile,
                sourceType: 'superpowers',
                skillPath: actualSkillName
            };
        }
    }

    return null;
}
```

**步骤 2: 更新 module.exports**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath
};
```

**步骤 3: 验证语法**

运行: `node -c lib/skills-core.js`
预期: 无输出

**步骤 4: 提交**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill path resolution with shadowing support"
```

---

### 任务 4: 提取更新检查逻辑

**文件:**
- 修改: `lib/skills-core.js`
- 参考: `.codex/superpowers-codex` (第 16-38 行)

**步骤 1: 添加 checkForUpdates 函数**

在 requires 之后的顶部添加：

```javascript
const { execSync } = require('child_process');
```

在 `module.exports` 之前添加：

```javascript
/**
 * 检查 git 仓库是否有可用更新。
 *
 * @param {string} repoDir - git 仓库路径
 * @returns {boolean} - 如果有可用更新则为 true
 */
function checkForUpdates(repoDir) {
    try {
        // 快速检查，3 秒超时以避免网络延迟时阻塞
        const output = execSync('git fetch origin && git status --porcelain=v1 --branch', {
            cwd: repoDir,
            timeout: 3000,
            encoding: 'utf8',
            stdio: 'pipe'
        });

        // 解析 git status 输出以查看我们是否落后
        const statusLines = output.split('\n');
        for (const line of statusLines) {
            if (line.startsWith('## ') && line.includes('[behind ')) {
                return true; // 我们落后于远程
            }
        }
        return false; // 最新
    } catch (error) {
        // 网络宕机、git 错误、超时等 - 不阻塞 bootstrap
        return false;
    }
}
```

**步骤 2: 更新 module.exports**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath,
    checkForUpdates
};
```

**步骤 3: 验证语法**

运行: `node -c lib/skills-core.js`
预期: 无输出

**步骤 4: 提交**

```bash
git add lib/skills-core.js
git commit -m "feat: add git update checking to core module"
```

---

## 阶段 2: 重构 Codex 以使用共享核心

### 任务 5: 更新 Codex 以导入共享核心

**文件:**
- 修改: `.codex/superpowers-codex`（在顶部添加导入）

**步骤 1: 添加导入语句**

在文件顶部现有的 requires 之后（约第 6 行），添加：

```javascript
const skillsCore = require('../lib/skills-core');
```

**步骤 2: 验证语法**

运行: `node -c .codex/superpowers-codex`
预期: 无输出

**步骤 3: 提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: import shared skills core in codex"
```

---

### 任务 6: 用核心版本替换 extractFrontmatter

**文件:**
- 修改: `.codex/superpowers-codex` (第 40-74 行)

**步骤 1: 删除本地 extractFrontmatter 函数**

删除第 40-74 行（整个 extractFrontmatter 函数定义）。

**步骤 2: 更新所有 extractFrontmatter 调用**

查找并替换所有调用从 `extractFrontmatter(` 到 `skillsCore.extractFrontmatter(`

影响的行大约: 90, 310

**步骤 3: 验证脚本仍然工作**

运行: `.codex/superpowers-codex find-skills | head -20`
预期: 显示 skills 列表

**步骤 4: 提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared extractFrontmatter in codex"
```

---

### 任务 7: 用核心版本替换 findSkillsInDir

**文件:**
- 修改: `.codex/superpowers-codex` (约第 97-136 行)

**步骤 1: 删除本地 findSkillsInDir 函数**

删除整个 `findSkillsInDir` 函数定义（约第 97-136 行）。

**步骤 2: 更新所有 findSkillsInDir 调用**

替换调用从 `findSkillsInDir(` 到 `skillsCore.findSkillsInDir(`

**步骤 3: 验证脚本仍然工作**

运行: `.codex/superpowers-codex find-skills | head -20`
预期: 显示 skills 列表

**步骤 4: 提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared findSkillsInDir in codex"
```

---

### 任务 8: 用核心版本替换 checkForUpdates

**文件:**
- 修改: `.codex/superpowers-codex` (约第 16-38 行)

**步骤 1: 删除本地 checkForUpdates 函数**

删除整个 `checkForUpdates` 函数定义。

**步骤 2: 更新所有 checkForUpdates 调用**

替换调用从 `checkForUpdates(` 到 `skillsCore.checkForUpdates(`

**步骤 3: 验证脚本仍然工作**

运行: `.codex/superpowers-codex bootstrap | head -50`
预期: 显示 bootstrap 内容

**步骤 4: 提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared checkForUpdates in codex"
```

---

## 阶段 3: 构建 OpenCode 插件

### 任务 9: 创建 OpenCode 插件目录结构

**文件:**
- 创建: `.opencode/plugin/superpowers.js`

**步骤 1: 创建目录**

运行: `mkdir -p .opencode/plugin`

**步骤 2: 创建基本插件文件**

```javascript
#!/usr/bin/env node

/**
 * Superpowers plugin for OpenCode.ai
 *
 * 提供用于加载和发现 skills 的自定义工具，
 * 并在会话启动时自动 bootstrap。
 */

const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const os = require('os');

const homeDir = os.homedir();
const superpowersSkillsDir = path.join(homeDir, '.config/opencode/superpowers/skills');
const personalSkillsDir = path.join(homeDir, '.config/opencode/skills');

/**
 * OpenCode 插件入口点
 */
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // 自定义工具和 hooks 将在这里
  };
};
```

**步骤 3: 验证文件已创建**

运行: `ls -l .opencode/plugin/superpowers.js`
预期: 文件存在

**步骤 4: 提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: create opencode plugin scaffold"
```

---

### 任务 10: 实现 use_skill 工具

**文件:**
- 修改: `.opencode/plugin/superpowers.js`

**步骤 1: 添加 use_skill 工具实现**

替换插件返回语句为：

```javascript
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  // 导入 zod 用于模式验证
  const { z } = await import('zod');

  return {
    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill to guide your work. Skills contain proven workflows, mandatory processes, and expert techniques.',
        schema: z.object({
          skill_name: z.string().describe('Name of the skill to load (e.g., "superpowers:brainstorming" or "my-custom-skill")')
        }),
        execute: async ({ skill_name }) => {
          // 解析 skill 路径（处理 shadowing: 个人 > superpowers）
          const resolved = skillsCore.resolveSkillPath(
            skill_name,
            superpowersSkillsDir,
            personalSkillsDir
          );

          if (!resolved) {
            return `Error: Skill "${skill_name}" not found.\n\nRun find_skills to see available skills.`;
          }

          // 读取 skill 内容
          const fullContent = fs.readFileSync(resolved.skillFile, 'utf8');
          const { name, description } = skillsCore.extractFrontmatter(resolved.skillFile);

          // 提取 frontmatter 之后的内容
          const lines = fullContent.split('\n');
          let inFrontmatter = false;
          let frontmatterEnded = false;
          const contentLines = [];

          for (const line of lines) {
            if (line.trim() === '---') {
              if (inFrontmatter) {
                frontmatterEnded = true;
                continue;
              }
              inFrontmatter = true;
              continue;
            }

            if (frontmatterEnded || !inFrontmatter) {
              contentLines.push(line);
            }
          }

          const content = contentLines.join('\n').trim();
          const skillDirectory = path.dirname(resolved.skillFile);

          // 格式化输出类似于 Claude Code 的 Skill 工具
          return `# ${name || skill_name}
# ${description || ''}
# Supporting tools and docs are in ${skillDirectory}
# ============================================

${content}`;
        }
      }
    ]
  };
};
```

**步骤 2: 验证语法**

运行: `node -c .opencode/plugin/superpowers.js`
预期: 无输出

**步骤 3: 提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement use_skill tool for opencode"
```

---

### 任务 11: 实现 find_skills 工具

**文件:**
- 修改: `.opencode/plugin/superpowers.js`

**步骤 1: 将 find_skills 工具添加到 tools 数组**

在 use_skill 工具定义之后、关闭 tools 数组之前添加：

```javascript
      {
        name: 'find_skills',
        description: 'List all available skills in the superpowers and personal skill libraries.',
        schema: z.object({}),
        execute: async () => {
          // 在两个目录中查找 skills
          const superpowersSkills = skillsCore.findSkillsInDir(
            superpowersSkillsDir,
            'superpowers',
            3
          );
          const personalSkills = skillsCore.findSkillsInDir(
            personalSkillsDir,
            'personal',
            3
          );

          // 组合并格式化 skills 列表
          const allSkills = [...personalSkills, ...superpowersSkills];

          if (allSkills.length === 0) {
            return 'No skills found. Install superpowers skills to ~/.config/opencode/superpowers/skills/';
          }

          let output = 'Available skills:\n\n';

          for (const skill of allSkills) {
            const namespace = skill.sourceType === 'personal' ? '' : 'superpowers:';
            const skillName = skill.name || path.basename(skill.path);

            output += `${namespace}${skillName}\n`;
            if (skill.description) {
              output += `  ${skill.description}\n`;
            }
            output += `  Directory: ${skill.path}\n\n`;
          }

          return output;
        }
      }
```

**步骤 2: 验证语法**

运行: `node -c .opencode/plugin/superpowers.js`
预期: 无输出

**步骤 3: 提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement find_skills tool for opencode"
```

---

### 任务 12: 实现会话启动 Hook

**文件:**
- 修改: `.opencode/plugin/superpowers.js`

**步骤 1: 添加 session.started hook**

在 tools 数组之后，添加：

```javascript
    'session.started': async () => {
      // 读取 using-superpowers skill 内容
      const usingSuperpowersPath = skillsCore.resolveSkillPath(
        'using-superpowers',
        superpowersSkillsDir,
        personalSkillsDir
      );

      let usingSuperpowersContent = '';
      if (usingSuperpowersPath) {
        const fullContent = fs.readFileSync(usingSuperpowersPath.skillFile, 'utf8');
        // 去除 frontmatter
        const lines = fullContent.split('\n');
        let inFrontmatter = false;
        let frontmatterEnded = false;
        const contentLines = [];

        for (const line of lines) {
          if (line.trim() === '---') {
            if (inFrontmatter) {
              frontmatterEnded = true;
              continue;
            }
            inFrontmatter = true;
            continue;
          }

          if (frontmatterEnded || !inFrontmatter) {
            contentLines.push(line);
          }
        }

        usingSuperpowersContent = contentLines.join('\n').trim();
      }

      // 工具映射说明
      const toolMapping = `
**Tool Mapping for OpenCode:**
When skills reference tools you don't have, substitute OpenCode equivalents:
- \`TodoWrite\` → \`update_plan\` (your planning/task tracking tool)
- \`Task\` tool with subagents → Use OpenCode's subagent system (@mention syntax or automatic dispatch)
- \`Skill\` tool → \`use_skill\` custom tool (already available)
- \`Read\`, \`Write\`, \`Edit\`, \`Bash\` → Use your native tools

**Skill directories contain supporting files:**
- Scripts you can run with bash tool
- Additional documentation you can read
- Utilities and helpers specific to that skill

**Skills naming:**
- Superpowers skills: \`superpowers:skill-name\` (from ~/.config/opencode/superpowers/skills/)
- Personal skills: \`skill-name\` (from ~/.config/opencode/skills/)
- Personal skills override superpowers skills when names match
`;

      // 检查更新（非阻塞）
      const hasUpdates = skillsCore.checkForUpdates(
        path.join(homeDir, '.config/opencode/superpowers')
      );

      const updateNotice = hasUpdates ?
        '\n\n⚠️ **Updates available!** Run `cd ~/.config/opencode/superpowers && git pull` to update superpowers.' :
        '';

      // 返回要注入到会话的上下文
      return {
        context: `<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'use_skill' tool:**

${usingSuperpowersContent}

${toolMapping}${updateNotice}
</EXTREMELY_IMPORTANT>`
      };
    }
```

**步骤 2: 验证语法**

运行: `node -c .opencode/plugin/superpowers.js`
预期: 无输出

**步骤 3: 提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement session.started hook for opencode"
```

---

## 阶段 4: 文档

### 任务 13: 创建 OpenCode 安装指南

**文件:**
- 创建: `.opencode/INSTALL.md`

**步骤 1: 创建安装指南**

```markdown
# 为 OpenCode 安装 Superpowers

## 先决条件

- [OpenCode.ai](https://opencode.ai) 已安装
- Node.js 已安装
- Git 已安装

## 安装步骤

### 1. 安装 Superpowers Skills

```bash
# 克隆 superpowers skills 到 OpenCode 配置目录
mkdir -p ~/.config/opencode/superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. 安装插件

插件包含在你刚刚克隆的 superpowers 仓库中。

OpenCode 将自动从以下位置发现它：
- `~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`

或者你可以将其链接到项目本地插件目录：

```bash
# 在你的 OpenCode 项目中
mkdir -p .opencode/plugin
ln -s ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js .opencode/plugin/superpowers.js
```

### 3. 重启 OpenCode

重启 OpenCode 以加载插件。在下一次会话时，你应该看到：

```
You have superpowers.
```

## 用法

### 查找 Skills

使用 `find_skills` 工具列出所有可用的 skills：

```
use find_skills tool
```

### 加载 Skill

使用 `use_skill` 工具加载特定的 skill：

```
use use_skill tool with skill_name: "superpowers:brainstorming"
```

### 个人 Skills

在 `~/.config/opencode/skills/` 中创建你自己的 skills：

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

个人 skills 覆盖同名的 superpowers skills。

## 更新

```bash
cd ~/.config/opencode/superpowers
git pull
```

## 故障排除

### 插件未加载

1. 检查插件文件是否存在: `ls ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`
2. 检查 OpenCode 日志中的错误
3. 验证 Node.js 已安装: `node --version`

### Skills 未找到

1. 验证 skills 目录是否存在: `ls ~/.config/opencode/superpowers/skills`
2. 使用 `find_skills` 工具查看发现了什么
3. 检查文件结构: 每个 skill 应该有一个 `SKILL.md` 文件

### 工具映射问题

当 skill 引用你没有的 Claude Code 工具时：
- `TodoWrite` → 使用 `update_plan`
- `Task` with subagents → 使用 `@mention` 语法调用 OpenCode subagents
- `Skill` → 使用 `use_skill` 工具
- 文件操作 → 使用你的原生工具

## 获取帮助

- 报告问题: https://github.com/obra/superpowers/issues
- 文档: https://github.com/obra/superpowers
```

**步骤 2: 验证文件已创建**

运行: `ls -l .opencode/INSTALL.md`
预期: 文件存在

**步骤 3: 提交**

```bash
git add .opencode/INSTALL.md
git commit -m "docs: add opencode installation guide"
```

---

### 任务 14: 更新主 README

**文件:**
- 修改: `README.md`

**步骤 1: 添加 OpenCode 部分**

查找有关支持平台的部分（在文件中搜索 "Codex"），并在其后添加：

```markdown
### OpenCode

Superpowers 通过原生 JavaScript 插件与 [OpenCode.ai](https://opencode.ai) 一起工作。

**安装:** 参见 [.opencode/INSTALL.md](.opencode/INSTALL.md)

**特性:**
- 自定义工具: `use_skill` 和 `find_skills`
- 自动会话 bootstrap
- 带有 shadowing 的个人 skills
- 支持文件和脚本访问
```

**步骤 2: 验证格式**

运行: `grep -A 10 "### OpenCode" README.md`
预期: 显示你添加的部分

**步骤 3: 提交**

```bash
git add README.md
git commit -m "docs: add opencode support to readme"
```

---

### 任务 15: 更新发布说明

**文件:**
- 修改: `RELEASE-NOTES.md`

**步骤 1: 添加 OpenCode 支持条目**

在文件顶部（在标题之后），添加：

```markdown
## [Unreleased]

### 新增

- **OpenCode 支持**: OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具: `use_skill` 和 `find_skills`
  - 自动会话 bootstrap 和工具映射说明
  - 共享核心模块 (`lib/skills-core.js`) 用于代码复用
  - `.opencode/INSTALL.md` 中的安装指南

### 更改

- **重构 Codex 实现**: 现在使用共享的 `lib/skills-core.js` 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - skill 发现和解析的唯一真实来源

---

```

**步骤 2: 验证格式**

运行: `head -30 RELEASE-NOTES.md`
预期: 显示你的新部分

**步骤 3: 提交**

```bash
git add RELEASE-NOTES.md
git commit -m "docs: add opencode support to release notes"
```

---

## 阶段 5: 最终验证

### 任务 16: 测试 Codex 仍然工作

**文件:**
- 测试: `.codex/superpowers-codex`

**步骤 1: 测试 find-skills 命令**

运行: `.codex/superpowers-codex find-skills | head -20`
预期: 显示带有名称和描述的 skills 列表

**步骤 2: 测试 use-skill 命令**

运行: `.codex/superpowers-codex use-skill superpowers:brainstorming | head -20`
预期: 显示 brainstorming skill 内容

**步骤 3: 测试 bootstrap 命令**

运行: `.codex/superpowers-codex bootstrap | head -30`
预期: 显示带有说明的 bootstrap 内容

**步骤 4: 如果所有测试通过，记录成功**

不需要提交 - 这只是验证。

---

### 任务 17: 验证文件结构

**文件:**
- 检查: 所有新文件存在

**步骤 1: 验证所有文件已创建**

运行:
```bash
ls -l lib/skills-core.js
ls -l .opencode/plugin/superpowers.js
ls -l .opencode/INSTALL.md
```
预期: 所有文件存在

**步骤 2: 验证目录结构**

运行: `tree -L 2 .opencode/`（或 `find .opencode -type f` 如果 tree 不可用）
预期:
```
.opencode/
├── INSTALL.md
└── plugin/
    └── superpowers.js
```

**步骤 3: 如果结构正确，继续**

不需要提交 - 这只是验证。

---

### 任务 18: 最终提交和摘要

**文件:**
- 检查: `git status`

**步骤 1: 检查 git 状态**

运行: `git status`
预期: 工作树干净，所有更改已提交

**步骤 2: 审查提交日志**

运行: `git log --oneline -20`
预期: 显示此实施的所有提交

**步骤 3: 创建摘要文档**

创建完成摘要，显示：
- 总提交数
- 创建的文件: `lib/skills-core.js`, `.opencode/plugin/superpowers.js`, `.opencode/INSTALL.md`
- 修改的文件: `.codex/superpowers-codex`, `README.md`, `RELEASE-NOTES.md`
- 执行的测试: Codex 命令已验证
- 准备进行: 使用真实 OpenCode 安装进行测试

**步骤 4: 报告完成**

向用户展示摘要并提供：
1. 推送到远程
2. 创建 pull request
3. 使用真实 OpenCode 安装进行测试（需要安装 OpenCode）

---

## 测试指南（手动 - 需要 OpenCode）

这些步骤需要安装 OpenCode，不属于自动实施的一部分：

1. **安装 skills**: 遵循 `.opencode/INSTALL.md`
2. **启动 OpenCode 会话**: 验证 bootstrap 出现
3. **测试 find_skills**: 应列出所有可用的 skills
4. **测试 use_skill**: 加载 skill 并验证内容出现
5. **测试支持文件**: 验证 skill 目录路径可访问
6. **测试个人 skills**: 创建个人 skill 并验证它覆盖核心
7. **测试工具映射**: 验证 TodoWrite → update_plan 映射工作

## 成功标准

- [ ] `lib/skills-core.js` 创建，包含所有核心函数
- [ ] `.codex/superpowers-codex` 重构为使用共享核心
- [ ] Codex 命令仍然工作（find-skills, use-skill, bootstrap）
- [ ] `.opencode/plugin/superpowers.js` 创建，包含工具和 hooks
- [ ] 安装指南已创建
- [ ] README 和 RELEASE-NOTES 已更新
- [ ] 所有更改已提交
- [ ] 工作树干净
