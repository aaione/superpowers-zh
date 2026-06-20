# Visual Companion 认证加固设计

**日期：** 2026-06-10
**状态：** 待 Drew 审查的草案

## 目标

修复在 PR #1720 的 brainstorming visual companion 中发现的安全性和可靠性缺口，而不改变 companion 的核心工作流，也不引入运行时依赖。

修复必须是测试优先的，并为以下各项留下清晰的自动化证据：

- 跨源浏览器标签页不能借由 cookies 注入 companion 事件
- 重启重连在不单纯依赖浏览器 cookie 行为的情况下工作
- bearer 密钥在 bootstrap 之后不残留在可见 URL 中
- `/files/*` 不能提供内容目录之外的文件
- 未来的同源 vendored UI 库仍能工作

## 威胁模型

该 companion 为单个 brainstorming 会话提供 agent 生成的本地 UI。重要资产是：

- 由 companion 提供的屏幕内容
- 会话密钥
- `state/events`，agent 将其读取为用户反馈
- companion 会话目录下的本地文件

范围内的攻击者：

- 另一个 `localhost` 端口上的恶意浏览器标签页
- 一个能向 companion 发起请求、但不应能以 companion UI 身份认证的浏览器页面
- 当服务器绑定到非环回接口时的直接远程客户端
- 通过 URL 历史、referrers 或已提交本地状态的意外泄漏
- 逃出 `/files/*` 的内容目录符号链接或路径伎俩

本次修复范围外：

- 恶意的 agent 编写的屏幕 HTML
- 由 companion 屏幕加载的恶意同源 vendored JavaScript

此范围外边界是有意为之的。companion 屏幕是 agent UI 面的一部分。它们今天可能使用内联脚本，未来某天可能使用诸如 Alpine 或 Three.js 之类的同源 vendored 库。防范恶意屏幕 HTML 需要一个更大的、带窄消息桥的沙箱化 iframe 架构；那不是本次 PR 加固轮的范围。

## 当前失败

自动化和有头浏览器测试在 PR 分支中发现以下失败：

1. 一个跨源 localhost 页面能够打开一个 cookie 认证的 WebSocket，并在真正的 companion 页面设置 cookie 之后，向 `state/events` 写入攻击者控制的选择。
2. `/files/*` 提供指向 `content/` 之外的符号链接，包括一个指向 `state/server-info`（含带密钥 URL）的符号链接。
3. 会话密钥残留在实际屏幕页面的 URL 中，因此同源屏幕 JavaScript 和意外的 referrers/历史都能看到它。
4. 辅助脚本用一个无密钥的 `ws://host` URL 重连。在有头 Chrome 中，在同端口/同令牌重启后，浏览器停止向重启后的服务器出示 cookie，因此打开的标签页一直卡在 tombstone 上，直到手工重新加载。
5. shell lint 和生命周期测试需要清理，以使测试通过在 Codex 中稳定。

## 设计

### 1. Bootstrap 带密钥加载

`GET /?key=<token>` 变成一个 bootstrap 响应，而非屏幕响应。

当密钥有效时，服务器：

1. 像今天一样设置 HttpOnly 会话 cookie
2. 返回一个小的 HTML bootstrap 页面
3. 该 bootstrap 页面把密钥存入标签页级 `sessionStorage`
4. 该 bootstrap 页面使用 `location.replace('/')` 导航到 `/`

此后，可见的屏幕 URL 是裸的 `/`，而非 `/?key=...`。

带有效 cookie 的 `GET /` 提供当前屏幕。无有效 cookie 的 `GET /` 仍返回友好的 403 页面。`GET /?key=<wrong>` 返回 403。

为何用 `sessionStorage`：辅助脚本需要一个能经受同端口重启、且不单纯依赖 cookie 行为的重连凭据。由于屏幕 HTML 是受信任的同源 UI，把密钥存入标签页级存储对这一威胁模型而言是可接受的。它在实质上优于把密钥留在地址栏、历史和 referrer 面中。

### 2. WebSocket 同源强制

WebSocket 升级必须通过两项检查：

1. 通过查询密钥或 cookie 的有效会话认证
2. 如果存在 `Origin` 头，它必须匹配请求目标源

源检查应当比较：

```text
Origin === "http://" + req.headers.host
```

浏览器攻击者页面示例：

```text
Origin: http://localhost:9999
Host: localhost:58088
```

即便浏览器发送了 companion cookie，这也必须被拒绝。

合法 companion 页面示例：

```text
Origin: http://localhost:58088
Host: localhost:58088
```

当密钥或 cookie 有效时，这应当被接受。

直接的非浏览器客户端可省略 `Origin`；它们仍需要会话密钥。

### 3. 辅助脚本重连凭据

`helper.js` 应当从 `sessionStorage` 读取标签页级密钥，并将其追加到 WebSocket URL：

```text
ws://<host>/?key=<stored-key>
```

如果不存在已存储的密钥，辅助脚本回退到当前仅 cookie 的 `ws://<host>` 行为。这为已加载的、确实拥有有效 cookie 但无存储条目的页面保留了兼容性。

### 4. `/files/*` 包含

文件服务器应当继续拒绝空名称和点文件。它还必须确保文件是 `CONTENT_DIR` 内的真实常规文件。

使用 realpath 包含作为边界：

- 计算 `realContentDir = fs.realpathSync(CONTENT_DIR)`
- 计算 `realFilePath = fs.realpathSync(filePath)`
- 仅当 `realFilePath` 等于 `realContentDir` 的后代时才提供
- 用 404 拒绝符号链接和内容目录之外的任何内容

服务器应继续使用 `path.basename`，以使嵌套路径仍不被支持。

### 5. 泄漏削减头

添加不会阻塞内联脚本或未来同源 vendored 库的保守头：

```text
Referrer-Policy: no-referrer
Cache-Control: no-store
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Cross-Origin-Resource-Policy: same-origin
```

本轮不要添加限制性的 `script-src` CSP。companion 当前注入内联辅助 JavaScript，未来的屏幕可能加载同源 vendored 库。

### 6. Gitignore 持久会话状态

把 `.superpowers/` 添加到仓库根的 `.gitignore`，以便在使用 `--project-dir` 时，持久化的 companion 状态和 `.last-token` 不会意外被提交。

### 7. 测试稳定性和 lint

清理所触及的启动/停止脚本中的 shell lint 告警。

更新调用 `start-server.sh --idle-timeout-minutes` 的生命周期测试，使其不会在 Codex 的 `CODEX_CI` 前台自动检测下挂起。该测试在期望脚本返回启动 JSON 时，应当用 `--background` 强制后台模式。

## 测试策略

所有行为变更都应是 TDD：

1. 编写失败的聚焦测试
2. 运行并确认它因预期原因失败
3. 实现最小修复
4. 重跑聚焦测试
5. 重跑完整 brainstorm-server 套件

要求的聚焦回归：

- 有效带密钥的 `/` 返回 bootstrap，而非屏幕内容
- bootstrap 把密钥存入 `sessionStorage` 并剥离 URL
- 仅 cookie 的 `/` 仍提供屏幕内容
- 辅助脚本对 WebSocket URL 使用 `sessionStorage` 密钥
- 同源 cookie WebSocket 能打开
- 跨源 cookie WebSocket 被拒绝且不写入事件
- 直接密钥 WebSocket 在无 `Origin` 下仍能打开
- `content/` 下指向 `state/server-info` 的符号链接返回 404
- 在常规 HTML、bootstrap、403 和文件响应上存在安全头
- 同端口/同令牌重启能用已存储密钥认证重连
- 所触及的 shell 脚本通过 shell lint
- 生命周期套件在 Codex 下不挂起

## 验收标准

- `cd tests/brainstorm-server && npm test` 反复通过且不挂起。
- 先前从另一个 localhost 源写入 `attacker-injected` 的安全探测，现在无法打开 WebSocket，并使 `state/events` 保持不变。
- 符号链接到 `server-info` 的探测返回 404。
- 一次有头或无头浏览器的带密钥加载，结束在裸的 `/` URL 上，且状态药丸达到 Connected。
- 同端口/同令牌重启自动重连，无需手工重新加载。
- `scripts/lint-shell.sh` 对所触及的 shell 脚本通过。

## 推迟的工作

如果项目日后需要把屏幕 HTML 当作不可信，则设计一个独立的沙箱化 iframe 架构。那应当把生成的屏幕隔离在独立的源或沙箱化 frame 上，并仅为用户选择暴露一个窄的 `postMessage` 桥。不要把它捆绑进本次修复。
