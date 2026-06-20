# Visual Companion 最终加固修复设计

**日期：** 2026-06-11
**状态：** 待 Drew 审查的草案

## 目标

完成 PR #1720 visual companion 加固轮，使分支准备好接受 Jesse 审查，具备干净的安全行为、确定性的测试，以及一个仅包含 companion 工作的 PR diff。

这是在既有认证加固设计之上的修复。它不应当重新设计 companion 或扩大功能面。

## 背景

先前的加固轮添加了带密钥会话、同源 WebSocket 检查、URL 密钥剥离、`/files/*` 包含、泄漏削减头、IPv6 URL 格式化、Windows 生命周期覆盖，以及 PR 证据更新。

最终审查轮发现五个剩余问题：

1. 根 `GET /` 屏幕选择路径仍可能提供 `content/` 下指向内容目录之外的符号链接或硬链接。
2. 当首选端口被占用时，回退服务器可能复用一个持久化的 `.last-token`，从而创建两个带有相同 bearer 密钥的、同项目的活动 companion 服务器。
3. `stop-server.sh` 在强所有权证明不可用时，可能向一个无关的 `node server.cjs` 进程发信号。
4. 某些测试可能针对错误的回退进程通过、在失败时泄漏后台进程，或在类 Windows 主机上假设支持符号链接。
5. 该 PR 当前处于冲突状态，因为分支包含一个较早的、已单独处理的 `evals` 子模块版本提升。

## 非目标

- 本轮不添加 HTTPS 隧道或 `wss://` 源语义。
- 不实现退出、自由文本或对比辅助等 companion 功能。
- 不 vendor Alpine、Three.js 或任何其他 JavaScript 库。
- 不尝试沙箱化恶意的 agent 编写的屏幕 HTML。
- 不为陈旧的 stop-server PID 文件添加向后兼容，除非 Drew 明确批准该权衡。

## 继承的安全不变量

本修复保留已设计并实现的认证加固：

- `.last-token` 和 `state/server-info` 仍是敏感的仅所有者状态。
- 回退令牌可能出现在启动 JSON 和 `state/server-info` 中，但绝不能写入 `.last-token`。
- cookie 仍是按端口命名、`HttpOnly`、`SameSite=Strict`，并限定于 `/`。
- WebSocket 升级仍要求有效密钥或 cookie。
- 当浏览器提供 `Origin` 头时，仍强制执行 WebSocket `Origin` 检查。
- 直接的无 `Origin` 客户端仅在携带会话密钥时仍被允许。
- 生成的同源屏幕 JavaScript 和未来同源 vendored 库是受信任的。沙箱化恶意屏幕 HTML 仍推迟。

## 设计

### 1. 重新变基到当前 `dev`

在实现工作之前把 `brainstorming-companion` 重新变基到当前 `origin/dev`。通过取 `dev` 来解决 `evals` 子模块冲突。

变基之后：

- `evals` 不得出现在 PR diff 中。
- PR #1720 仍可提到在别处运行的 eval 证据，但必须包含精确的外部证据：eval 仓库提交、场景路径、命令、结果产物路径或 id，以及 RED/GREEN 结果。
- PR 正文不得暗示 evals 子模块版本提升是本 PR 的一部分。
- 任何更早的、暗示包含子模块版本提升的 PR 正文文本或评论，都必须被最终的 PR 正文证据取代。

### 2. 根屏幕包含

根屏幕路由必须使用与 `/files/*` 相同的包含边界。

`getNewestScreen()` 应当忽略任何未通过常规文件在内容目录内这一护栏的 `.html` 候选。该护栏必须解析真实路径并确保被提供的文件位于 `CONTENT_DIR` 内。当平台报告链接计数时，它还必须通过拒绝链接计数不正好为 1 的文件来保留既有的硬链接保护。

预期行为：

- `content/` 下指向 `content/` 之外的符号链接被忽略。
- 当 `fs.linkSync` 成功且 `lstat.nlink > 1` 时，`content/` 下指向 `state/server-info` 的硬链接被忽略。
- 如果没有安全的屏幕文件剩余，则提供等待页面。
- 既有的 `/files/*` 包含行为保持不变：空名称、点文件、符号链接、硬链接和目录仍返回 404。

### 3. 回退令牌隔离

端口回退绝不能复用从持久化的 `.last-token` 加载的令牌。

令牌来源在代码中应当显式：

- 来自环境的 `BRAINSTORM_TOKEN` 是有意的运维者/测试覆盖。如果在设置了显式环境令牌的同时首选端口被占用，服务器必须失败关闭而非回退，因为被占用的服务器可能正在使用同一个显式令牌。
- `.last-token` 是为同端口重连便利而持久化的状态。如果服务器因首选端口被占用而回退，丢弃该已加载令牌并为回退进程生成一个新的、未持久化的令牌。
- 一个新生成、非从 `.last-token` 加载的令牌可在同一进程内复用，因为不知道有其他活动进程持有它。

回退服务器必须继续避免覆盖 `.last-port` 和 `.last-token`。

### 4. stop-server 所有权证明

`start-server.sh` 应当创建一个按启动的服务器实例 id，并作为惰性命令行参数传给 Node，例如：

```text
node server.cjs --brainstorm-server-id=<id>
```

该 id 不是认证凭据。它仅是本地生命周期脚本的进程所有权证据。`server.cjs` 可忽略该参数。

该 id 必须使用 shell/MSYS 安全的字母表，例如 `^[A-Za-z0-9_-]{32,64}$`。以仅所有者权限把它存入 `state/server-instance-id`。

`stop-server.sh` 应当从状态读取预期 id，且仅当目标进程 argv 包含作为完整 argv token（而非松散子串）的精确参数 `--brainstorm-server-id=<id>` 时才向 PID 发信号。可用时优先使用 `/proc/<pid>/cmdline`，然后回退到宽泛的 `ps` 输出。即便 `server-info` 缺失或 `lsof` 不可用，一个匹配的实例 id 也是充分证明。既有的端口到 PID 检查可作为附加证据保留。

当所有权无法证明时失败关闭：

- 缺失 PID 文件
- 缺失或格式错误的 server id
- 目标命令行不可用
- 目标命令行不包含预期 id
- 无新 id 的旧/陈旧会话元数据

这有意优先于让陈旧进程继续运行，而非杀死一个无关进程。

对运维者可见的结果应当显式：

- 缺失 PID 文件返回 `not_running`
- 缺失或格式错误的 server id 返回 `stale_pid`
- 不可用的命令行返回 `stale_pid`
- 错误或缺失的 argv id 返回 `stale_pid`
- 成功停止返回 `stopped`

在 `stale_pid` 和 `stopped` 结果上，移除 `server.pid` 和 `server-instance-id`，以便未来的停止尝试不会持续瞄准同一个模糊进程。不要移除持久会话内容。

### 5. 测试加固

测试通过应当在 macOS 和用于验证的 Windows Git Bash 主机上都具确定性。

要求的改动：

- 固定端口套件必须在服务器报告回退端口时要么快速失败，要么从报告的启动端口驱动所有客户端。
- `stop-server.test.sh` 需要在启动任何后台进程之前有一个顶层清理 trap。
- 符号链接专属断言应当探测符号链接能力，并仅当主机无法创建可用测试符号链接时才跳过该断言。
- 创建冒名顶替进程的测试必须断言，当生命周期元数据缺失或不足时，冒名顶替者存活。
- Windows/MSYS 的 start-server 测试必须断言，类 Windows 检测仍清除 `BRAINSTORM_OWNER_PID`、在适当时仍自动前台化，并仍精确传入 instance-id argv。

### 6. 文档和 PR 一致性

在 Jesse 审查之前，协调审查者可见的文档和 PR 元数据：

- 更新问题清单，使处置与本 PR 实际交付的内容匹配。
- 保持 auto-open 文档与已实现的 `--open` 行为一致。
- 在所有地方保持记录在案的默认空闲超时为 4 小时。
- 变基之后对照模板审查 PR 正文。
- 在 PR 正文中以具体命令和结果记录 macOS、Windows、浏览器/手工以及外部 eval 证据。

## 测试策略

对每个行为变更使用 TDD：

1. 添加或收紧一个聚焦回归测试。
2. 运行并确认它因预期原因失败。
3. 实现最小修复。
4. 重跑聚焦测试。
5. 重跑完整 brainstorm-server 套件。

要求的聚焦回归：

| 行为 | 测试文件 | 聚焦命令 | 预期 RED | 预期 GREEN |
| --- | --- | --- | --- | --- |
| 根路由忽略符号链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已认证的 `GET /` 提供链接到内容之外的内容 | 响应提供等待页面或安全屏幕 |
| 根路由忽略支持的硬链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已认证的 `GET /` 提供硬链接的 `server-info` | 当 `nlink > 1` 时硬链接候选被忽略 |
| `/files/*` 包含保持不变 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 既有包含测试回归 | 空、点文件、目录、符号链接、硬链接用例仍为 404 |
| 持久化令牌回退轮换令牌 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 URL 密钥等于持久化的首选端口密钥 | 回退 URL 密钥不同且不写入 `.last-token` |
| 显式令牌回退失败关闭 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 服务器在设置了 `BRAINSTORM_TOKEN` 时回退 | 进程以非零退出且不启动回退 |
| 回退密钥无法向原服务器认证 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退密钥从原端口收到 200 | 原端口拒绝回退密钥 |
| 正确实例 id 允许停止 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 真实由 start-server 启动的服务器存活 | 停止返回 `stopped` 且进程退出 |
| 错误、缺失、格式错误或陈旧的 id 是安全的 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 冒名顶替者被发信号 | 停止返回 `stale_pid` 且冒名顶替者存活 |
| 固定端口套件不能通过回退通过 | `tests/brainstorm-server/server.test.js`、`tests/brainstorm-server/auth.test.js` | 各自的 `node` 命令 | 测试静默地与回退端口通信 | 测试清晰失败或有意图地使用报告端口 |
| shell 清理 trap 在失败时运行 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 失败留下子进程 | trap 回收后台子进程 |
| Windows/MSYS 启动行为保持生命周期不变量 | `tests/brainstorm-server/start-server.test.sh`、`tests/brainstorm-server/windows-lifecycle.test.sh` | 在 macOS 和 `ballmer` 上的 `bash` 测试命令 | 所有者 PID 或 argv 处理回归 | 所有者 PID 被清除、前台检测成立、id argv 存在 |

每个 RED/GREEN 循环都应为 PR 正文留下一条简短证据笔记：聚焦命令、修复前的失败断言、修复后的通过断言，以及证据是在 macOS 还是 Windows 上采集。

## 验证

在宣告修复完成之前，运行：

- `git fetch origin dev && git rebase origin/dev`
- `git diff --quiet origin/dev...HEAD -- evals`
- `gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid`
- `cd tests/brainstorm-server && npm test`
- TDD 期间使用的相关聚焦测试命令
- `git diff --check`
- 对所触及的 JavaScript 文件做 Node 语法检查
- 对所触及的 shell 文件做 shell lint
- 在 `ballmer` 上的 Windows 验证：完整可运行的 brainstorm-server 套件加上独立的 Windows 生命周期探测

手工/浏览器测试仅在自动化通过之后进行。

## 验收标准

- PR #1720 干净地重新变基到当前 `dev`。
- `evals` 在 PR diff 中不存在。
- 根屏幕提供不能通过符号链接或支持的硬链接逃逸读取 `content/` 之外。
- `/files/*` 包含保护保持不变。
- 没有回退服务器运行一个可能被占用首选端口服务器共享的令牌。
- 当所有权证明缺失或模糊时 `stop-server.sh` 不向无关进程发信号。
- 当 `server-info` 或 `lsof` 不可用时，`stop-server.sh` 仍能用匹配的实例 id 停止一个合法服务器。
- 每个回归都记录了聚焦的 RED/GREEN 证据。
- macOS 和 Windows 验证证据记录在 PR 正文中。
- PR 正文准确描述了分支中的内容以及在外部采集了什么证据。
