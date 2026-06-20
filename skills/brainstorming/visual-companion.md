# 可视伴侣指南 (Visual Companion Guide)

基于浏览器的可视化头脑风暴伴侣，用于展示 mockup、图表和选项。

## 何时使用

按问题逐个决定，而不是按会话决定。判断标准是：**用户通过看它是否比读它更能理解？**

**当内容本身是可视的时候使用浏览器**：

- **UI mockup**——线框图、布局、导航结构、组件设计
- **架构图**——系统组件、数据流、关系图
- **并排的可视化对比**——对比两种布局、两种配色方案、两种设计方向
- **设计打磨**——当问题涉及外观与感觉、间距、视觉层级时
- **空间关系**——以图表形式渲染的状态机、流程图、实体关系

**当内容是文本或表格时使用终端**：

- **需求与范围问题**——"X 是什么意思？"、"哪些功能在范围内？"
- **概念性的 A/B/C 选择**——在用文字描述的方案之间做选择
- **权衡列表**——优缺点、对比表
- **技术决策**——API 设计、数据建模、架构方案选择
- **澄清问题**——任何答案是用文字而非视觉偏好来表达的情况

一个关于 UI 主题的问题不一定是视觉问题。"你想要哪种向导？"是概念性的——使用终端。"这些向导布局中哪一个感觉对？"是视觉性的——使用浏览器。

## 工作原理

服务器监视一个目录中的 HTML 文件，并把最新的一个提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到它，并可以点击选择选项。选择会被记录到 `state_dir/events`，你在下一轮读取它。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器会原样提供它（只是注入辅助脚本）。否则，服务器会自动将你的内容包装在框架模板中——添加页眉、CSS 主题、连接状态以及所有交互基础设施。**默认情况下编写内容片段。** 仅在你需要对页面完全控制时才编写完整文档。

## 启动一个会话

```bash
# 在用户批准该伴侣之后启动。--open 会在
# 第一屏时自动打开用户的浏览器；--project-dir 持久化 mockup 并支持同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回：{"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open` 时，当你推送第一屏时浏览器会自行打开——你不需要让用户去打开它，但仍应分享该 URL 作为后备（无头/远程环境不会自动打开）。

**该 URL 包含一个会话密钥（`?key=…`）。** 服务器会拒绝任何
没有它的请求，因此始终从 `url` 字段给用户**完整**的 URL——
绝不要去掉查询字符串，也绝不要交出一个光秃秃的 `http://host:port`。该
密钥管控 HTTP 和 WebSocket 访问，这样一个游离的浏览器标签页或网络上另一台机器
无法读取屏幕或注入事件。首次加载后，
浏览器会通过 cookie 记住密钥，因此重载和 `/files/*` 资源无需
重复它即可工作。

**查找连接信息：** 服务器会将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器且没有捕获 stdout，请读取该文件以获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 中的会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传入，以便 mockup 持久化在 `.superpowers/brainstorm/` 中并在服务器重启后保留。如果没有它，文件会进入 `/tmp` 并被清理。提醒用户如果 `.gitignore` 中还没有 `.superpowers/`，请将其添加进去。

**按平台启动服务器：**

**Claude Code:**
```bash
# 默认模式可用——脚本本身会让服务器在后台运行。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本会自动检测并切换到前台模式（这会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，以便服务器能在对话轮次之间存活，然后在下一轮读取 `$STATE_DIR/server-info` 以获取 URL 和端口。

**Codex:**
```bash
# Codex 会回收后台进程。脚本会自动检测 CODEX_CI 并
# 切换到前台模式。正常运行它即可——无需额外标志。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Gemini CLI:**
```bash
# 使用 --foreground 并在你的 shell 工具调用上设置 is_background: true
# 以便进程能跨轮次存活
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**Copilot CLI:**
```bash
# 使用 --foreground 并通过 bash 工具以 mode: "async" 启动服务器
# 以便进程能跨轮次存活。捕获返回的 shellId，
# 便于日后需要与之交互时用于 read_bash / stop_bash。
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在对话轮次之间保持后台运行。如果你的环境会回收分离的进程，请使用 `--foreground` 并通过你平台的后台执行机制启动该命令。

如果该 URL 无法从你的浏览器访问（在远程/容器化环境中很常见），绑定一个非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环

1. **检查服务器是否存活**，然后**编写 HTML** 到 `screen_dir` 中的新文件：
   - **必需：在引用该 URL 或推送屏幕之前，确认服务器存活。** 检查 `$STATE_DIR/server-info` 存在且 `$STATE_DIR/server-stopped` 不存在。如果它已关闭，使用**相同的 `--project-dir`** 通过 `start-server.sh` 重启它——它会复用同一端口，因此用户打开的标签页会自行重连（服务器关闭期间它显示一个"已暂停"的覆盖层），你不需要发送新 URL。服务器在空闲 4 小时后自动退出（可通过 `--idle-timeout-minutes` 配置）。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **绝不复用文件名**——每一屏都获得一个新文件
   - 使用你的文件创建工具——**绝不使用 cat/heredoc**（会把噪声倾倒进终端）
   - 服务器自动提供最新的文件

2. **告诉用户会有什么内容，然后结束你的轮次：**
   - 提醒他们 URL（每一步都要，不只是第一次）
   - 给出关于屏幕上内容的简短文字摘要（例如，"正在展示首页的 3 种布局选项"）
   - 请他们在终端中回应："看一下并告诉我你的想法。如果想的话，可以点击选择一个选项。"

3. **在你的下一轮**——在用户于终端回应之后：
   - 如果 `$STATE_DIR/events` 存在则读取它——其中包含用户在浏览器中的交互（点击、选择），格式为 JSON 行
   - 与用户的终端文字合并以获得完整画面
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据

4. **迭代或推进**——如果反馈改变了当前屏幕，编写一个新文件（例如 `layout-v2.html`）。仅当当前步骤被验证后才进入下一个问题。

5. **返回终端时卸载**——当下一步不需要浏览器时（例如一个澄清问题、一次权衡讨论），推送一个等待屏以清除陈旧内容：

   ```html
   <!-- 文件名：waiting.html（或 waiting-2.html 等） -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这能防止用户在对话已经继续时盯着一个已了结的选择。当下一个视觉问题出现时，像往常一样推送一个新的内容文件。

6. 重复直到完成。

## 编写内容片段

只编写要放入页面内部的内容。服务器会自动将它包装在框架模板中（页眉、主题 CSS、连接状态以及所有交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就这样。不需要 `<html>`、不需要 CSS、不需要 `<script>` 标签。服务器提供了所有这些。

## 可用的 CSS 类

框架模板为你的内容提供这些 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 为容器添加 `data-multiselect`，让用户可以选择多个选项。每次点击切换该项的选中样式。

```html
<div class="options" data-multiselect>
  <!-- 相同的 option 标记——用户可以选择/取消选择多个 -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### 分割视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### 优点/缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### 模拟元素（线框图构建块）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版与区块

- `h2`——页面标题
- `h3`——区块标题
- `.subtitle`——标题下方的次要文字
- `.section`——带底部边距的内容块
- `.label`——小号大写标签文字

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互会被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时该文件会被自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流展示了用户的探索路径——他们可能在确定之前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击的模式可能揭示值得追问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有与浏览器交互——只使用他们的终端文字。

## 设计提示

- **按问题缩放保真度**——布局用线框图，打磨问题用打磨级别的保真度
- **在每一页上解释问题**——"哪个布局感觉更专业？"而不只是"选一个"
- **推进前先迭代**——如果反馈改变了当前屏幕，编写一个新版本
- **每屏最多 2-4 个选项**
- **在重要的地方使用真实内容**——对于摄影作品集，使用真实图片（Unsplash）。占位内容会掩盖设计问题。
- **保持 mockup 简单**——专注于布局和结构，而非像素级完美的设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 绝不复用文件名——每一屏都必须是新文件
- 对于迭代：追加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果该会话使用了 `--project-dir`，mockup 文件会持久化在 `.superpowers/brainstorm/` 中以供日后参考。只有 `/tmp` 会话在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
