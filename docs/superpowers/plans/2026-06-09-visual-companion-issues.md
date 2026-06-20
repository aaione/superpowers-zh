# 可视化头脑风暴伴侣 —— 议题与变更目录 (Visual Brainstorming Companion — Issue & Change Catalog)

**日期：** 2026-06-09
**状态：** 分析 / 分诊。我们自己实现这些；所引用的
社区 PR 是证据和参考资料，**不是**我们打算合并的代码。

## 目的 (Purpose)

一个统一的地方，捕获每一个涉及可视化头脑风暴伴侣（位于
`skills/brainstorming/scripts/` 的本地服务器）的开放议题和 PR，提炼到底层的问题以及我们会做的变更。每个条目都
对照当前代码核对，而不是 PR 作者的描述。

## 范围决策（Jesse，2026-06-09） (Scope decisions (Jesse, 2026-06-09))

- **不内嵌 Alpine.js。** PR #1639（通过内嵌
  Alpine 构建实现交互式 mockup）被**放弃**。见 E3。
- **E1（终端对比 HTML 的硬性门控）是一个 workshop 议题。** 我们会一起设计它；它在此处没有成文。
- **E2（存储位置，#975/#977）暂缓**处理。
- **远程服务是一等场景。** Superpowers 是通用的；
  用户从远程连接（SSH 隧道、Tailscale、`--host 0.0.0.0`）。安全修复
  必须保护这些用户，而不仅是回环。**决策：每会话密钥**，而不是 Host 白名单。Host 白名单只能防御回环浏览器的 confused deputy；一个直接的远程客户端只需
  发送预期的 `Host` 即可，所以白名单对远程暴露而言只是表演。密钥是在回环、隧道和直接远程之间统一认证客户端的唯一手段，
  而且它也能击败 DNS 重绑定。见 A1。

## 组件映射 (Component map)

| 文件 | 角色 |
|------|------|
| `skills/brainstorming/scripts/server.cjs` | 零依赖 HTTP + WebSocket 服务器（RFC 6455 手工实现）。提供最新屏幕服务，监视 `content/`，把事件记录到 `state/events`。 |
| `skills/brainstorming/scripts/helper.js` | 注入到每个页面。WebSocket 客户端、点击捕获、`window.brainstorm` API。 |
| `skills/brainstorming/scripts/frame-template.html` | 包裹内容片段的框架（页眉、主题 CSS、状态点、指示条）。 |
| `skills/brainstorming/scripts/start-server.sh` | 启动包装器。会话目录、host/url-host、owner-PID 解析、平台后台化。 |
| `skills/brainstorming/scripts/stop-server.sh` | 按 PID 文件杀死服务器，清理 `/tmp` 会话。 |
| `skills/brainstorming/visual-companion.md` | agent 在接受该伴侣时阅读的操作指南。 |
| `skills/brainstorming/SKILL.md` | 提供该伴侣以及每问题决策所在之处。 |

## 处置摘要 (Disposition summary)

| ID | 条目 | 来源 | 处置 |
|----|------|--------|-------------|
| A1 | `/`、`/files/*` 和 WS 上的每会话密钥（取代 Host 白名单） | issues #1014, PRs #1110/#1553 | **做** —— 选定方案 |
| A2 | Host 白名单；浏览器 WS Origin 检查 | PRs #1110/#1553 | Host 白名单放弃；WS Origin 检查在认证之后保留，作为浏览器 confused deputy 防御 |
| A3 | 在 `null` / 非对象 WS payload 上崩溃 | PR #1504 | 做 |
| A4 | `decodeFrame` 中的帧长度上界 | issue #1446 | 已修复 —— 验证/关闭 |
| B1 | 点文件屏幕被当作内容服务（`._*.html`） | PR #950 | 做 |
| B2 | `stop-server.sh` 杀死被复用/陈旧的 PID | PR #1703 | 做 |
| B3 | WS 客户端重连退避 + 状态指示器 | PR #856 | 做 |
| C1 | 空闲超时过短 / 不可配置；关闭时不关闭 WS | issue #1237 (PR #1689) | 做 |
| C2 | 服务器死亡对用户/agent 不可见 | issue #1237（残留） | 做 |
| D1 | 永久退出该伴侣 | issue #892 | 暂缓 —— 不在 PR #1720 中 |
| D2 | 来自浏览器的自由文本反馈 | issue #957 | 暂缓 —— 不在 PR #1720 中 |
| D3 | 自动打开伴侣 URL | PR #759 (#755) | 已通过 PR #1720 的 `--open` 完成 |
| D4 | 框架中的浅色/深色对比度辅助类 | PR #1683 | 暂缓 —— 不在 PR #1720 中 |
| E1 | 按问题硬性门控终端对比 HTML | PR #1037 | **Workshop** |
| E2 | 把会话状态移出工作树 | issue #975 (PR #977) | **暂缓** |
| E3 | 内嵌 Alpine.js 用于交互式 mockup | PR #1639 | **放弃** |
| E4 | 启动/停止脚本中的 shell-lint 警告 | PR #1677 | 仅机会性处理 |

---

## A. 服务器安全加固（`server.cjs`） (A. Server security hardening (`server.cjs`))

### A1 —— 每会话密钥（选定方案） (A1 — Per-session secret key (chosen approach))

**威胁模型。** 两项资产：所服务屏幕（`/`）和
文件（`/files/*`）的机密性，以及 `state/events` 的完整性 —— 一个带有
truthy `choice` 的 WebSocket 客户端会写入那里（`server.cjs:243-246`），而 agent 在下一轮
把它读作用户的选择，即**对带有完整工具访问权限的在线会话进行 prompt 注入**。可达者：在默认 `127.0.0.1` 绑定下，用户浏览器中的恶意
页面（一个 confused deputy —— 同时运行攻击者 JS *且*能到达
回环）；在远程绑定（`--host 0.0.0.0`，tailnet/LAN）下，任何能
路由到该端口的主机，直接可达，中间没有同源策略。今天
`handleUpgrade`（`server.cjs:176`）只检查 `Sec-WebSocket-Key`，而
`handleRequest`（`server.cjs:138`）什么也不检查 —— 两者都是敞开的。

**为什么用密钥而不是 Host 白名单。** Host 白名单只能防御
回环浏览器 deputy。一个直接的远程客户端只需发送预期的 `Host`
并伪造/省略 `Origin`，所以白名单对我们必须保护的远程
场景而言只是表演。每会话密钥在回环、SSH 隧道和直接远程之间统一认证客户端，
而且它也杀死 DNS 重绑定
（被重绑定的页面既不知道密钥，也收不到 host 作用域的 cookie）。
所以该密钥**完全取代**了 A1/A2 的 Host 白名单 —— 没有 `BRAINSTORM_ALLOWED_HOSTS`。

**设计。** 随机令牌（`crypto.randomBytes(32)` 的十六进制），在
启动时于 `server.cjs` 中生成（可通过 `BRAINSTORM_TOKEN` 覆盖以用于确定性
测试）：

1. **URL 携带它**，形式为 `?key=<token>`。服务器已经在它的
   `server-started` JSON（`server.cjs:351`）中构建 `url` 并写入 `state/server-info`
   —— 在那里追加 `?key=` 意味着 `start-server.sh`（grep 并打印那个
   JSON）和技能（把那个 URL 交给用户）都**无需更改**。
2. **Cookie 引导。** `/` 上一个有效的 `?key` 会设置
   `brainstorm-key-<port>=<token>; HttpOnly; SameSite=Strict; Path=/`。然后
   浏览器会自动把它附加到同源子资源（`/files/*`）和
   WebSocket 握手，所以 agent 可以写任何 URL 风格都能工作，
   且 `helper.js` 无需更改。Cookie 名称**按端口区分**，以避免
   Jupyter 多服务器冲突（cookie 不按端口作用域化）。
   `SameSite=Strict` 对 CDN/Unsplash 内容是安全的 —— 那个 cookie 是 host
   作用域的，所以出站 CDN 请求从不携带它；SameSite 只约束
   回到我们源的请求，那些都是同站的。
3. **认证门控** = 有效的 `?key` **或** 有效的 cookie（用
   `crypto.timingSafeEqual` 比较），作用于 `/`、`/files/*` 和 WS 升级。缺失/错误
   密钥 → 友好的 **403 HTML 页面**（"此页面需要你的编码
   agent 给你的完整 URL，包括 `?key=…`" —— 通用的"编码 agent"，而非"Claude"，
   因为它也发布在 Codex/Gemini/Copilot 上）。WS 升级 → 销毁 socket。

查询令牌是事实来源；cookie 是一种便利，从不
承担初始认证负载。

**影响范围。** `server.cjs`（全部逻辑）。`helper.js` 可选一行
（从 `location.search` 向 WS URL 追加 `?key=` 作为 cookie 被阻断时的
回退）。`start-server.sh` 无。`visual-companion.md` 文档说明（URL 现在带
`?key=`；不要剥离它）。更新测试以传递令牌。

### A2 —— Host 白名单放弃；浏览器 WS Origin 保留 (A2 — Host allowlist dropped; browser WS Origin retained)

被 A1 吞并。密钥用一个机制关闭了 WS 注入向量（#1014）、
HTTP/WS DNS 重绑定读取向量（PR #1553）和跨源 WS 向量
（PR #1110），而且与白名单不同，它真正保护
远程绑定场景。没有 `BRAINSTORM_ALLOWED_HOSTS`，也没有 Host 白名单。最终
实现仍在会话认证之后检查浏览器 WebSocket `Origin`，这样跨源
的 localhost 标签页就无法搭车使用伴侣 cookie。

### A3 —— 服务器在 `null` / 原始类型 WS payload 上崩溃 (A3 — Server crashes on `null` / primitive WS payload)

**问题。** `handleMessage`（`server.cjs:233`）执行 `JSON.parse(text)`，然后
在 `server.cjs:243` 处 `if (event.choice)`。一个发送 4 字节文本
帧 `null` 的客户端会产生 `event === null`，而 `null.choice` 抛出。该抛出
**未被**捕获 —— `handleMessage` 是从 `socket.on('data')` 处理器
（`server.cjs:207`）中调用的，在 `try/catch` 之外，而该 try/catch 只包裹 `decodeFrame`。结果是一个未捕获异常和进程退出。任何本地客户端都能杀死
服务器。

**变更。** 保护这次访问：`if (event && event.choice)`。最小且精确 ——
`JSON.parse` 无法产生 `undefined`，原始类型对
`.choice` 返回 `undefined` 而不抛出，所以只有 `null` 是真正的危险。（避免
更宽泛的修复 —— 顶层 `try/catch` 或 `process.on('uncaughtException')`
会掩盖其他缺陷。）

### A4 —— `decodeFrame` 中的帧长度上界（相邻） (A4 — Frame-length bound in `decodeFrame` (adjacent))

被 PR #1504 引用为 #1446。当前代码**已经**对扩展
帧长度设了上界：`MAX_FRAME_PAYLOAD_BYTES = 10MB`（`server.cjs:10`）在
`server.cjs:58-67` 处、任何 `Buffer.alloc` 之前被强制执行。行动：对照
当前 `dev` 验证 #1446，如果已解决则关闭，而不是重新实现。

---

## B. 服务器健壮性 / 正确性 (B. Server robustness / correctness)

### B1 —— macOS resource-fork 点文件被当作屏幕内容服务 (B1 — macOS resource-fork dotfiles served as screen content)

**问题。** 最新屏幕选择器只按 `f.endsWith('.html')` 过滤
（`server.cjs:127-128`）。在 macOS/ExFAT 上，`._screen.html` resource-fork 文件会通过
该过滤器，而且因为与真实文件一同写入，可能排序为最新 —— 所以
浏览器收到二进制元数据而非 mockup。四个读取点共享这个
弱过滤器：`getNewestScreen`（`server.cjs:127`）、`knownFiles` 初始化
（`server.cjs:279`）、`fs.watch` 处理器（`server.cjs:286`）和 `/files/`
端点（`server.cjs:154-156`）。

**变更。** 在所有四个点拒绝点文件（`!f.startsWith('.')`）。覆盖
`._*`、`.DS_Store` 等。

### B2 —— `stop-server.sh` 可能杀死一个被复用的 PID (B2 — `stop-server.sh` can kill a reused PID)

**问题。** `stop-server.sh` 从 `state/server.pid`
（`stop-server.sh:20`）读取 PID 并 `kill` 它（`:23`，在 `:35` 升级为 `-9`），
而不确认该 PID 仍属于我们的服务器。重启或 PID
回绕之后，该文件可能指向一个无关进程，而我们会随后 SIGKILL 它。

**变更。** 在发信号之前，验证所有权 —— 该 PID 的命令是
运行我们 `server.cjs` 的 `node`，理想情况下匹配本会话。如果所有权无法
证明，则失败关闭（报告 `stale_pid`，不杀死）。对真实
场景保留既有 `stopped` / `not_running` 输出。

### B3 —— WebSocket 客户端：静默重连、陈旧的"已连接" (B3 — WebSocket client: silent reconnect, stale "Connected")

**问题。** `helper.js` 按固定 1 秒计时器重连（`helper.js:21-23`），
没有 `onerror` 处理器，关闭时从不把 `ws` 置空，也从不清除待处理
重连计时器。框架的状态元素被硬编码为"已连接"，圆点
钉死在 `var(--success)`（`frame-template.html:77,200`）。当笔记本
休眠或服务器重启时，页面在一个死掉的 socket 上显示"已连接"，并
无反馈地排队事件。

**变更。**
- `helper.js`：指数退避（500ms → ×2 → 上限 30s，打开时重置）；
  `onerror` 委托给 `onclose`；关闭时 `ws = null`；重连前
  `clearTimeout`。
- `frame-template.html`：由一个 `--status-color` 自定义
  属性驱动状态圆点，这样 JS 可以切换 已连接（绿色）/ 重连中（黄色）/
  已断开（红色）。

---

## C. 生命周期 / 超时（issue #1237） (C. Lifecycle / timeout (issue #1237))

### C1 —— 空闲超时过短、不可配置、WS 使进程保持存活 (C1 — Idle timeout too short, not configurable, WS keeps process alive)

**问题。** `IDLE_TIMEOUT_MS` 硬编码为 30 分钟（`server.cjs:258`），
由 60 秒生命周期检查（`server.cjs:329-332`）强制执行。单个头脑风暴
问题可能超过 30 分钟（用户思考或离开），所以
服务器在会话中途死亡。另外，`shutdown()`（`server.cjs:310-321`）调用
`server.close()` 但从不关闭 `clients`（`server.cjs:174`）中已升级的 socket，所以一个打开的浏览器连接能使 Node 进程
在关闭之后保持存活。

**变更。**
- 把默认值提高到 4 小时并使其可配置：
  `start-server.sh` 中的 `--idle-timeout-minutes` → 一个环境变量 → `IDLE_TIMEOUT_MS`，
  并针对 Node 计时器溢出做验证。
- 在启动 JSON / `state/server-info` 中暴露有效超时。
- 在 `shutdown()` 中，关闭 `clients` 中的每个 socket，这样进程才能真正
  退出。

### C2 —— 服务器死亡不可见 (C2 — Server death is invisible)

**问题。** 当服务器退出时，它写入 `state/server-stopped` 并移除
`state/server-info`（`server.cjs:312-317`），并且技能被*告知*去检查
这些文件（`visual-companion.md:108`）—— 但那是模型会跳过的软性指引，
而浏览器只显示一个通用的"无法到达"。用户手动诊断；agent 持续引用一个死掉的 URL。

**变更（两部分，独立于 C1）：**
- **面向浏览器的墓碑。** 在最后服务的 URL 处留下点东西，说
  "此伴侣已过期 —— 让 Claude 重启它"，而不是连接
  错误。待权衡的选项：`helper.js` 在 socket 超过退避仍
  无法连接时渲染一个 banner（仅当页面已加载时有效），对比一种更复杂的
  方法，保持一个最小响应器存活以服务一个墓碑页面。
- **更硬的技能检查。** 收紧 `visual-companion.md` / `SKILL.md`，使
  "在引用 URL 或推送
  屏幕之前检查 `server-info`/`server-stopped`" 成为一个必需步骤，而非一条说明。保持轻量 —— 可能是 agent 总是运行的一行辅助。

---

## D. 功能 (D. Features)

### D1 —— 永久退出可视化伴侣（issue #892） (D1 — Permanent opt-out of the visual companion (issue #892))

**问题。** 该伴侣在每个会话都作为自己的消息被提供
（`SKILL.md:25,151-152`）。一个永不想要它的用户每次都要支付那次往返 —— 以及
HTML 生成。没有办法说"永不提供这个"。

**变更。** 在提供步骤之前，技能检查一个用户级设置，并在
设置了退出时完全跳过提供。

**待定的设计选择。** 机制尚未敲定：
- 环境变量（例如 `SUPERPOWERS_VISUAL_COMPANION=off`），告知技能去读取 ——
  最简单，匹配议题所要求的，放在 `.zshrc` 中。
- 一个 plugin-settings 文件（`.claude/superpowers.local.md` frontmatter）—— 更
  结构化，能按项目生效，但更重且按项目作用域。
- 来自议题的可靠性告诫：一个单独的"无伴侣"技能在
  触发词上竞争且不可靠 —— 已否决。

选定机制，然后这就是一个小的 `SKILL.md` 变更加一个有文档记录的旋钮。

### D2 —— 来自浏览器的自由文本反馈（issue #957） (D2 — Free-text feedback from the browser (issue #957))

**问题。** 客户端只捕获对 `[data-choice]`
（`helper.js:36-62`）的点击。一个想为 mockup 做注释（"蓝色色号不对"）
的用户必须切换到终端，打断了视觉流程。

**变更。** 添加一个反馈 `<textarea>`，其提交通过既有的
`window.brainstorm.send` 路径（`helper.js:82-85`）发出
`{"type":"feedback","text":...,"timestamp":...}`。

**横切 —— 需要服务器变更。** `handleMessage` 仅在
`event.choice` 为 truthy（`server.cjs:243`）时持久化事件。一个 `feedback` 事件没有
`choice`，所以今天它会被记录但**从不写入 `state/events`**，
而 agent 看不到它。持久化条件也必须接受
`feedback` 事件。在 `visual-companion.md`
（浏览器事件格式，`:247-259`）中文档化新的事件形状。决定提交触发器（按钮对比失焦
对比两者）以及 textarea 渲染位置（框架级对比按屏幕可选）。

### D3 —— 自动打开伴侣 URL（PR #759，issue #755） (D3 — Auto-open the companion URL (PR #759, issue #755))

**问题。** `start-server.sh` 只打印 URL；用户手动打开它。
尤其在 WSL2 中，人们期望浏览器会打开。

**变更。** 在 `server-started` JSON 被解析之后尽力打开：
Windows/WSL → `rundll32.exe url.dll,FileProtocolHandler <url>`，macOS → `open`，
Linux → 仅当设置了 `DISPLAY`/`WAYLAND_DISPLAY` 时 `xdg-open`。吞掉
失败，绝不阻塞启动，持续回显 URL。在
`visual-companion.md` 中文档化。（考虑为无头/远程运行提供退出选项，在那种场景弹出浏览器是错误的 —— 与 D1 的配置机制挂钩。）

### D4 —— 浅色/深色对比度辅助类（PR #1683） (D4 — Light/dark contrast helpers (PR #1683))

**问题。** 内容片段被包裹在感知操作系统的框架
（`frame-template.html`）中。在深色模式下，快速 mockup 常使用白色内联
背景，同时继承低对比度的框架文本，使卡片/面板难以
阅读。

**变更。** 添加 `.light-surface` / `.dark-surface` 辅助类，以及
针对常见内联浅色背景的保守回退，并在
`visual-companion.md` 的 CSS 参考中文档化它们。纯 CSS 在 `frame-template.html` 中。

---

## E. Workshop / 暂缓 / 放弃 (E. Workshop / deferred / dropped)

### E1 —— 按问题硬性门控终端对比 HTML（PR #1037）—— WORKSHOP (E1 — Hard-gate terminal-vs-HTML per question (PR #1037) — WORKSHOP)

软性指引已存在："按问题决策"，带有浏览器对比终端
测试在 `SKILL.md:156-161` 和 `visual-companion.md:5-25`。抱怨是
模型为纯文本内容（A/B 列表、澄清
问题）渲染 HTML，浪费 token 和一个回合。PR #1037 把这个决策包裹在
`<HARD-GATE>` 中。**按 Jesse，我们会一起打磨措辞/机制** ——
这是塑造行为的技能内容，不在此处成文。

### E2 —— 把会话状态移出工作树（issue #975 / PR #977）—— 暂缓 (E2 — Move session state out of the working tree (issue #975 / PR #977) — DEFERRED)

今天 `--project-dir` 把会话状态写入 `<project>/.superpowers/brainstorm/`
（`start-server.sh:80-84`），而技能告诉用户将其 gitignore
（`visual-companion.md:58`）。请求是一个位于
仓库之外（XDG）的 `--state-dir` / `SUPERPOWERS_STATE_DIR`
默认值，保留 `--project-dir` 作为别名。
**按 Jesse 暂缓。** 记录下来以免丢失。

### E3 —— 内嵌 Alpine.js 用于交互式 mockup（PR #1639）—— 放弃 (E3 — Vendor Alpine.js for interactive mockups (PR #1639) — DROPPED)

添加一个内嵌 Alpine 构建以便 mockup 可以是交互式的（标签页、手风琴、
表单）而无需手工实现 JS。**按 Jesse 放弃** —— 我们不在伴侣运行时中引入
内嵌的第三方依赖。底层需求
（交互式 mockup）不通过这条路推进。

### E4 —— Shell-lint 警告（PR #1677）—— 机会性 (E4 — Shell-lint warnings (PR #1677) — OPPORTUNISTIC)

`start-server.sh` / `stop-server.sh` 中的 SC2034（及其同类）。琐碎；当
我们已经在编辑那些脚本时（B2/C1/D3）顺手折入，而不是作为它自己的
独立变更。

---

## 建议的实现分组 (Suggested grouping for implementation)

这些聚类为几个连贯的轮次（每轮都可独立针对
`tests/brainstorm-server/` 测试）：

1. **安全轮**（进行中，分支 `brainstorm-companion-session-key`）——
   A1 每会话密钥（取代 A2）+ A3 null 崩溃防护。验证/关闭 A4。
   *最高优先级。*
2. **生命周期轮** —— C1 + C2 一起（两者都触及 `shutdown()` 和
   服务器死亡叙事）。
3. **健壮性轮** —— B1、B2、B3（独立，小）。
4. **暂缓功能轮** —— D1、D2、D4 不是 PR #1720 的一部分。D3
   通过 `--open` 流程交付。

E1 是一个单独的 workshop 会话。E2/E3 超出本轮范围。
