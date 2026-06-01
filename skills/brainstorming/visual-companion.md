# Visual Companion 指南 (视觉化伴生指南)

基于浏览器的 visual brainstorming（视觉化头脑风暴）伴生工具，用于展示 mockups、图示和选项。

## 何时使用 (When to Use)

按问题决策，而不是按会话决策。测试标准：**用户通过看到它而不是阅读它能更好地理解吗？**

**使用浏览器** 当内容本身是视觉化的：

- **UI mockups** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉比较** — 比较两个布局、两种配色方案、两个设计方向
- **设计打磨** — 当问题关于外观、间距、视觉层次时
- **空间关系** — 渲染为图示的状态机、流程图、实体关系

**使用终端** 当内容是文本或表格：

- **需求和范围问题** — "X 是什么意思？"、"哪些功能在范围内？"
- **概念性 A/B/C 选择** — 从文字描述的方法中选择
- **权衡列表** — 优缺点、比较表
- **技术决策** — API 设计、数据建模、架构方法选择
- **澄清问题** — 任何答案是文字而不是视觉偏好

关于 UI 主题的问题不自动是视觉问题。"你想要什么样的向导？"是概念性的 — 使用终端。"这些向导布局中哪个感觉合适？"是视觉性的 — 使用浏览器。

## 工作原理 (How It Works)

服务器监听目录中的 HTML 文件，并将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到它并点击选择选项。选择记录到 `state_dir/events`，你在下一轮读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供（只注入辅助脚本）。否则，服务器自动将你的内容包装在 frame 模板中 — 添加 header、CSS 主题、选择指示器和所有交互基础设施。**默认编写内容片段。** 仅在需要完全控制页面时才编写完整文档。

## 启动会话 (Starting a Session)

```bash
# 启动带持久化的服务器（mockups 保存到项目）
scripts/start-server.sh --project-dir /path/to/project

# 返回: {"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。告诉用户打开 URL。

**查找连接信息：** 服务器将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动服务器且没有捕获 stdout，请读取该文件以获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 获取会话目录。

**注意：** 传递项目根目录作为 `--project-dir`，这样 mockups 会持久保存在 `.superpowers/brainstorm/` 中并在服务器重启后保留。没有它，文件会进入 `/tmp` 并被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore`（如果还没有）。

**按平台启动服务器：**

**Claude Code (macOS / Linux):**
```bash
# 默认模式有效 — 脚本自行将服务器置于后台
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows):**
```bash
# Windows 自动检测并使用前台模式，这会阻塞工具调用。
# 在 Bash 工具调用上使用 run_in_background: true，这样服务器可以跨对话轮次存活
scripts/start-server.sh --project-dir /path/to/project
```
通过 Bash 工具调用时，设置 `run_in_background: true`。然后在下一轮读取 `$STATE_DIR/server-info` 以获取 URL 和端口。

**Codex:**
```bash
# Codex 会收割后台进程。脚本自动检测 CODEX_CI 并
# 切换到前台模式。正常运行 — 无需额外标志。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI:**
```bash
# 使用 --foreground 并在你的 shell 工具调用上设置 is_background: true
# 这样进程可以跨轮次存活
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务器必须在对话轮次之间在后台持续运行。如果你的环境会收割分离的进程，使用 `--foreground` 并使用你平台的后台执行机制启动命令。

如果 URL 无法从浏览器访问（在远程/容器化设置中很常见），绑定非环回主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环流程 (The Loop)

1. **检查服务器是否存活**，然后**将 HTML 写入** `screen_dir` 中的新文件：
   - 每次写入前，检查 `$STATE_DIR/server-info` 是否存在。如果不存在（或 `$STATE_DIR/server-stopped` 存在），服务器已关闭 — 在继续之前用 `start-server.sh` 重新启动。服务器在 30 分钟不活动后自动退出。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **绝不要重用文件名** — 每个 screen 都是一个新文件
   - 使用 Write 工具 — **绝不要使用 cat/heredoc**（会向终端倾倒噪音）
   - 服务器自动提供最新的文件

2. **告诉用户期望什么并结束你的轮次：**
   - 提醒他们 URL（每一步，不只是第一步）
   - 给出屏幕内容的简要文字摘要（例如，"显示主页的 3 种布局选项"）
   - 要求他们在终端中响应："看看并告诉我你的想法。如果想选择选项，点击它。"

3. **在你的下一轮** — 用户在终端中响应后：
   - 如果 `$STATE_DIR/events` 存在则读取它 — 这包含用户的浏览器交互（点击、选择）作为 JSON 行
   - 与用户的终端文本合并以获得完整图片
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据

4. **迭代或前进** — 如果反馈改变了当前屏幕，编写新文件（例如 `layout-v2.html`）。仅当当前步骤验证后才能进入下一个问题。

5. **返回终端时卸载** — 当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送等待屏幕以清除陈旧内容：

   ```html
   <!-- filename: waiting.html (或 waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">在终端中继续...</p>
   </div>
   ```

   这可以防止用户在对话已经转移时盯着已解决的选择。当下一个视觉问题出现时，照常推送新内容文件。

6. 重复直到完成。

## 编写内容片段 (Writing Content Fragments)

只编写进入页面内部的内容。服务器自动将其包装在 frame 模板中（header、主题 CSS、选择指示器和所有交互基础设施）。

**最小示例：**

```html
<h2>哪种布局效果更好？</h2>
<p class="subtitle">考虑可读性和视觉层次</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单栏</h3>
      <p>简洁、专注的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双栏</h3>
      <p>侧边栏导航与主内容</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`，不需要 CSS，不需要 `<script>` 标签。服务器提供所有这些。

## 可用的 CSS 类 (CSS Classes Available)

frame 模板为你的内容提供这些 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>标题</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

**多选：** 向容器添加 `data-multiselect` 以让用户选择多个选项。每次点击切换项目。指示栏显示计数。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记 — 用户可以多选/取消选择多个 -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup 内容 --></div>
    <div class="card-body">
      <h3>名称</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">预览：仪表板布局</div>
  <div class="mockup-body"><!-- 你的 mockup HTML --></div>
</div>
```

### 分屏视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- 左边 --></div>
  <div class="mockup"><!-- 右边 --></div>
</div>
```

### 优缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>好处</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>坏处</li></ul></div>
</div>
```

### Mock 元素（线框构建块）

```html
<div class="mock-nav">Logo | 主页 | 关于 | 联系</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航</div>
  <div class="mock-content">主内容区域</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入字段">
<div class="placeholder">占位符区域</div>
```

### 排版和章节

- `h2` — 页面标题
- `h3` — 章节标题
- `.subtitle` — 标题下方的次要文字
- `.section` — 带有底部边距的内容块
- `.label` — 小写大写标签文字

## 浏览器事件格式 (Browser Events Format)

当用户在浏览器中点击选项时，他们的交互被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时，文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流显示用户的探索路径 — 他们可能在确定之前点击多个选项。最后的 `choice` 事件通常是最终选择，但点击模式可以揭示值得询问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，用户没有与浏览器交互 — 只使用他们的终端文本。

## 设计技巧 (Design Tips)

- **按问题调整保真度** — 布局用线框图，打磨问题用精细图
- **在每个页面上解释问题** — "哪种布局看起来更专业？" 而不是 "选一个"
- **前进前迭代** — 如果反馈改变了当前屏幕，编写新版本
- **每个屏幕最多 2-4 个选项**
- **重要时使用真实内容** — 对于摄影作品集，使用实际图片（Unsplash）。占位符内容会掩盖设计问题。
- **保持 mockup 简单** — 专注于布局和结构，而不是像素级设计

## 文件命名 (File Naming)

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 绝不要重用文件名 — 每个 screen 必须是一个新文件
- 对于迭代：追加版本后缀如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理 (Cleaning Up)

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，mockup 文件会持久保存在 `.superpowers/brainstorm/` 中供以后参考。只有 `/tmp` 会话在停止时被删除。

## 参考 (Reference)

- Frame 模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
