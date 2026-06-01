# 零依赖 Brainstorm 服务器

将 brainstorm 伴侣服务器的 vendored node_modules（express、ws、chokidar -- 714 个跟踪文件）替换为一个仅使用 Node.js 内置模块的零依赖 `server.js`。

## 动机

将 node_modules vendor 到 git 仓库中会产生供应链风险：冻结的依赖得不到安全补丁，714 个第三方代码文件未经审计就提交，而且对 vendor 代码的修改看起来像普通提交。虽然实际风险很低（仅限本地开发服务器），但消除它是直接的。

## 架构

一个 `server.js` 文件（约 250-300 行），使用 `http`、`crypto`、`fs` 和 `path`。该文件承担两个角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被 require 时**（`require('./server.js')`）：导出 WebSocket 协议函数用于单元测试

### WebSocket 协议

仅实现 RFC 6455 的文本帧：

**握手：** 使用 SHA-1 + RFC 6455 magic GUID 从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种屏蔽长度编码：
- 小：payload < 126 字节
- 中：126-65535 字节（16 位扩展）
- 大：> 65535 字节（64 位扩展）

使用 4 字节掩码密钥进行 XOR 解屏蔽。返回 `{ opcode, payload, bytesConsumed }`，缓冲区不完整时返回 `null`。拒绝未屏蔽帧。

**帧编码（服务器到客户端）：** 未屏蔽帧，使用相同的三种长度编码。

**处理的操作码：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。未识别的操作码返回状态 1003（不支持的数据）的关闭帧。

**刻意跳过的：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。这些对于本地客户端之间的小型 JSON 文本消息是不必要的。扩展和子协议在握手中协商 -- 不广播它们，就不会被激活。

**缓冲区累积：** 每个连接维护一个缓冲区。在 `data` 事件时，追加并循环调用 `decodeFrame` 直到返回 null 或缓冲区为空。

### HTTP 服务器

三个路由：

1. **`GET /`** -- 按 mtime 从屏幕目录提供最新的 `.html` 文件。检测完整文档与片段，将片段包装在 frame template 中，注入 helper.js。返回 `text/html`。当没有 `.html` 文件时，提供硬编码的等待页面（"Waiting for Claude to push a screen..."）并注入 helper.js。
2. **`GET /files/*`** -- 使用硬编码的扩展名映射（html、css、js、png、jpg、gif、svg、json）从屏幕目录提供静态文件，带 MIME 类型查找。找不到则返回 404。
3. **其他所有请求** -- 404。

WebSocket 升级通过 HTTP 服务器上的 `'upgrade'` 事件处理，与请求处理程序分离。

### 配置

环境变量（全部可选）：

- `BRAINSTORM_PORT` -- 绑定端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` -- 绑定接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` -- 启动 JSON 中 URL 的主机名（默认：host 为 `127.0.0.1` 时使用 `localhost`，否则与 host 相同）
- `BRAINSTORM_DIR` -- 屏幕目录路径（默认：`/tmp/brainstorm`）

### 启动序列

1. 如果 `SCREEN_DIR` 不存在则创建（`mkdirSync` recursive）
2. 从 `__dirname` 加载 frame template 和 helper.js
3. 在配置的 host/port 上启动 HTTP 服务器
4. 在 `SCREEN_DIR` 上启动 `fs.watch`
5. 成功监听后，将 `server-started` JSON 记录到 stdout：`{ type, port, host, url_host, url, screen_dir }`
6. 将相同的 JSON 写入 `SCREEN_DIR/.server-info`，以便当 stdout 被隐藏时（后台执行）agent 可以找到连接详情

### 应用层 WebSocket 消息

当从客户端收到 TEXT 帧时：

1. 解析为 JSON。如果解析失败，记录到 stderr 并继续。
2. 以 `{ source: 'user-event', ...event }` 格式记录到 stdout。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每个事件一行）。

### 文件监视

`fs.watch(SCREEN_DIR)` 替代 chokidar。对 HTML 文件事件：

- 新文件（`rename` 事件且文件存在）：如果存在 `.events` 文件则删除（`unlinkSync`），将 `screen-added` 以 JSON 格式记录到 stdout
- 文件更改（`change` 事件）：将 `screen-updated` 以 JSON 格式记录到 stdout（不清除 `.events`）
- 两种事件：向所有已连接的 WebSocket 客户端发送 `{ type: 'reload' }`

对每个文件名使用约 100ms 超时的防抖，防止重复事件（在 macOS 和 Linux 上常见）。

### 错误处理

- WebSocket 客户端的格式错误 JSON：记录到 stderr，继续
- 未处理的操作码：以状态 1003 关闭
- 客户端断开连接：从广播集合中移除
- `fs.watch` 错误：记录到 stderr，继续
- 没有优雅关闭逻辑 -- shell 脚本通过 SIGTERM 管理进程生命周期

## 更改内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单文件） |
| express、ws、chokidar 依赖 | 无 |
| 没有静态文件服务 | `/files/*` 从屏幕目录提供服务 |

## 保持不变

- `helper.js` -- 无更改
- `frame-template.html` -- 无更改
- `start-server.sh` -- 一行更新：`index.js` 改为 `server.js`
- `stop-server.sh` -- 无更改
- `visual-companion.md` -- 无更改
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台的 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上的单个扁平目录中可靠
- shell 脚本需要 bash（Windows 上的 Git Bash，Claude Code 需要它）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require `server.js` 的导出直接测试 WebSocket 帧编码/解码、握手计算和协议边界情况。

**集成测试**（`server.test.js`）：测试完整服务器行为 -- HTTP 服务、WebSocket 通信、文件监视、brainstorming 工作流。使用 `ws` npm 包作为仅用于测试的客户端依赖（不分发给终端用户）。
