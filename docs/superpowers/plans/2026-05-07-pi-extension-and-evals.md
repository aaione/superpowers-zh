# Pi 扩展与 evals 实现计划 (Pi Extension and Evals Implementation Plan)

> **致 agent 工作者：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐个实现本计划。步骤使用 checkbox（`- [ ]`）语法进行跟踪。

**目标：** 为 Superpowers 添加一等公民的 Pi 包支持，并把 Pi 添加为 Drill 的 eval 后端。

**架构：** Pi 包声明在根 `package.json` 中，加载既有的 `skills/` 加上一个小的 Pi 扩展。该扩展在会话启动时和压缩之后，以 user 角色消息的形式把 `using-superpowers` bootstrap 注入到 provider 上下文中，并带有 Pi 专用的工具映射。Drill 获得一个 `pi` 后端、Pi 会话日志规范化以及测试。

**技术栈：** Pi TypeScript 扩展 API、Node 内置测试运行器、Drill Python eval harness、pytest。

---

### 任务 1：Pi 包清单与扩展测试 (Task 1: Pi package manifest and extension tests)

**文件：**
- 修改：`package.json`
- 创建：`tests/pi/test-pi-extension.mjs`

- [ ] **步骤 1：编写失败的包/扩展测试**

创建 `tests/pi/test-pi-extension.mjs`，其中测试导入 `extensions/superpowers.ts`，注册假的 Pi 处理器，并断言：
- 根 `package.json` 的 `keywords` 包含 `pi-package`
- 根 `package.json` 有 `pi.skills: ["./skills"]`
- 根 `package.json` 有 `pi.extensions: ["./extensions/superpowers.ts"]`
- 该扩展注册了 `resources_discover`、`session_start`、`session_compact`、`context` 和 `agent_end`
- 启动 `context` 注入恰好一条 user 角色 bootstrap 消息
- `agent_end` 清除启动注入
- `session_compact` 重新启用注入
- 该扩展不注册 `session_before_compact`

- [ ] **步骤 2：运行测试并验证 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：FAIL，因为 `extensions/superpowers.ts` 不存在且 `package.json` 缺少 `pi` 清单。

- [ ] **步骤 3：实现清单字段**

更新 `package.json`，加入 `description`、`keywords`、`pi.extensions` 和 `pi.skills`，同时保留既有的 `name`、`version`、`type` 和 `main`。

- [ ] **步骤 4：实现 `extensions/superpowers.ts`**

创建一个零运行时依赖的扩展，它：
- 从 `import.meta.url` 定位包根
- 读取 `skills/using-superpowers/SKILL.md`
- 剥离 YAML frontmatter
- 追加 Pi 专用的工具映射
- 暴露带有 skills 路径的 `resources_discover`
- 在 `session_start` 和 `session_compact` 上把 bootstrap 标记为待处理
- 在 `context` 中注入一条 user 角色 bootstrap 消息
- 在开头的 `compactionSummary` 消息之后插入压缩后 bootstrap
- 在 `agent_end` 上清除待处理的 bootstrap

- [ ] **步骤 5：运行测试并验证 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：PASS。

### 任务 2：Pi 工具映射参考 (Task 2: Pi tool mapping reference)

**文件：**
- 创建：`skills/using-superpowers/references/pi-tools.md`
- 修改：`tests/pi/test-pi-extension.mjs`

- [ ] **步骤 1：为 Pi 参考文档编写失败测试**

添加断言：`skills/using-superpowers/references/pi-tools.md` 存在，并文档化了 `Skill`、`Task`、`TodoWrite` 以及内置工具名称的映射。

- [ ] **步骤 2：运行测试并验证 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：FAIL，因为 `pi-tools.md` 不存在。

- [ ] **步骤 3：添加 Pi 参考文档**

创建 `skills/using-superpowers/references/pi-tools.md`，解释 Pi 原生 skills、可选的 `pi-subagents`、没有规范的 todo/tasklist 插件，以及内置的小写工具。

- [ ] **步骤 4：运行测试并验证 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：PASS。

### 任务 3：Drill Pi 后端与会话日志规范化 (Task 3: Drill Pi backend and session log normalization)

**文件：**
- 创建：`evals/backends/pi.yaml`
- 修改：`evals/drill/backend.py`
- 修改：`evals/drill/engine.py`
- 修改：`evals/drill/normalizer.py`
- 修改：`evals/tests/test_backend.py`
- 修改：`evals/tests/test_normalizer.py`

- [ ] **步骤 1：编写失败的后端/规范化器测试**

为以下内容添加 pytest 覆盖：
- `load_backend("pi")` 返回 `family == "pi"`
- Pi 后端命令以 `pi` 开头并包含 `-e ${SUPERPOWERS_ROOT}`
- `_resolve_log_dir()` 对于 Pi 指向 `~/.pi/agent/sessions` 之下
- `filter_pi_logs_by_cwd()` 只保留头部 `cwd` 匹配场景工作目录的会话文件
- `normalize_pi_logs()` 从 Pi 助手会话条目中提取 `toolCall` 块，并把内置的小写工具映射为规范名称

- [ ] **步骤 2：运行测试并验证 RED**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：FAIL，因为 Pi 后端和规范化器不存在。

- [ ] **步骤 3：添加 `evals/backends/pi.yaml`**

把后端配置为运行 `pi -e ${SUPERPOWERS_ROOT}`，使用宽松的 TUI 就绪检测、`/quit` 关闭以及 Pi 会话日志位置。

- [ ] **步骤 4：实现 Pi family 支持**

用 Pi 日志过滤与规范化更新 `Backend.family`、`Engine._resolve_log_dir`、`Engine._collect_tool_calls` 和 `normalizer.py`。

- [ ] **步骤 5：运行测试并验证 GREEN**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：PASS。

### 任务 4：文档与完整验证 (Task 4: Documentation and full verification)

**文件：**
- 修改：`README.md`
- 修改：`evals/README.md`

- [ ] **步骤 1：文档化 Pi 安装与 eval 后端**

把 Pi 添加到 README 快速开始/安装列表，并把后端条目/用法添加到 `evals/README.md`。

- [ ] **步骤 2：运行验证**

运行：
```bash
node --experimental-strip-types --test tests/pi/test-pi-extension.mjs
uv run pytest evals/tests/test_backend.py evals/tests/test_setup.py evals/tests/test_normalizer.py -q
```

预期：全部测试通过。
