# 视觉 Brainstorming 重构实施计划

> **面向 agentic worker：** 必须使用 superpowers:subagent-driven-development（如果有 subagent）或 superpowers:executing-plans 来执行此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将视觉 brainstorming 从阻塞式 TUI 反馈模型重构为非阻塞的"浏览器显示，终端命令"架构。

**架构：** 浏览器成为交互式显示；终端保持对话通道。服务器将用户事件写入每个屏幕的 `.events` 文件，Claude 在下一个轮次读取。消除 `wait-for-feedback.sh` 和所有 `TaskOutput` 阻塞。

**技术栈：** Node.js（Express、ws、chokidar）、原生 HTML/CSS/JS

**规格文档：** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 文件映射

| 文件 | 操作 | 职责 |
|------|--------|---------------|
| `lib/brainstorm-server/index.js` | 修改 | 服务器：添加 `.events` 文件写入、新屏幕时清除、替换 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | 修改 | 模板：移除反馈 footer、添加内容占位符 + 选择指示器 |
| `lib/brainstorm-server/helper.js` | 修改 | 客户端 JS：移除 send/feedback 函数，收窄为点击捕获 + 指示器更新 |
| `lib/brainstorm-server/wait-for-feedback.sh` | 删除 | 不再需要 |
| `skills/brainstorming/visual-companion.md` | 修改 | Skill 指令：将循环重写为非阻塞流程 |
| `tests/brainstorm-server/server.test.js` | 修改 | 测试：更新以适应新的模板结构和 helper.js API |

---

## Chunk 1：服务器、模板、客户端、测试、Skill

### Task 1：更新 `frame-template.html`

**文件：**
- 修改：`lib/brainstorm-server/frame-template.html`

- [ ] **Step 1：移除反馈 footer HTML**

用选择指示栏替换 feedback-footer div（第 227-233 行）：

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above, then return to the terminal</span>
  </div>
```

同时替换 `#claude-content` 内部的默认内容（第 220-223 行）为内容占位符：

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **Step 2：用指示栏 CSS 替换反馈 footer CSS**

移除 `.feedback-footer`、`.feedback-footer label`、`.feedback-row` 以及 `.feedback-footer` 内的 textarea/button 样式（第 82-112 行）。

添加指示栏 CSS：

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **Step 3：验证模板渲染**

运行测试套件检查模板仍然加载：
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：测试 1-5 应该仍然通过。测试 6-8 可能失败（预期 -- 它们断言旧结构）。

- [ ] **Step 4：提交**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### Task 2：更新 `index.js` -- 内容注入和 `.events` 文件

**文件：**
- 修改：`lib/brainstorm-server/index.js`

- [ ] **Step 1：为 `.events` 文件写入编写失败测试**

在 `tests/brainstorm-server/server.test.js` 的测试 4 区域之后添加 -- 一个新测试，发送带有 `choice` 字段的 WebSocket 事件并验证 `.events` 文件被写入：

```javascript
    // 测试：选择事件写入 .events 文件
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **Step 2：运行测试验证失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 -- `.events` 文件尚不存在。

- [ ] **Step 3：为新屏幕时清除 `.events` 编写失败测试**

添加另一个测试：

```javascript
    // 测试：新屏幕时清除 .events
    console.log('Test: .events cleared on new screen');
    // .events 文件应从上一个测试中存在
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **Step 4：运行测试验证失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 -- `.events` 在屏幕推送时未清除。

- [ ] **Step 5：在 `index.js` 中实现 `.events` 文件写入**

在 WebSocket `message` 处理程序（`index.js` 的第 74-77 行），在 `console.log` 之后添加：

```javascript
    // 将用户事件写入 .events 文件供 Claude 读取
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

在 chokidar `add` 处理程序（第 104-111 行），添加 `.events` 清除：

```javascript
    if (filePath.endsWith('.html')) {
      // 清除上一个屏幕的事件
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... 现有的 reload broadcast
    }
```

- [ ] **Step 6：用注释占位符注入替换 `wrapInFrame`**

替换 `wrapInFrame` 函数（`index.js` 的第 27-32 行）：

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **Step 7：运行所有测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新的 `.events` 测试通过。现有测试可能仍有旧断言的失败（在 Task 4 中修复）。

- [ ] **Step 8：提交**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### Task 3：简化 `helper.js`

**文件：**
- 修改：`lib/brainstorm-server/helper.js`

- [ ] **Step 1：移除 `sendToClaude` 函数**

删除 `sendToClaude` 函数（第 92-106 行）-- 函数体和页面接管 HTML。

- [ ] **Step 2：移除 `window.send` 函数**

删除 `window.send` 函数（第 120-129 行）-- 与已移除的发送按钮关联。

- [ ] **Step 3：移除表单提交和输入变更处理程序**

删除表单提交处理程序（第 57-71 行）和输入变更处理程序（第 73-89 行），包括 `inputTimeout` 变量。

- [ ] **Step 4：移除 `pageshow` 事件监听器**

删除我们之前添加的 `pageshow` 监听器（不再有 textarea 需要清除）。

- [ ] **Step 5：将点击处理程序收窄为仅 `[data-choice]`**

用更窄的版本替换点击处理程序（第 36-55 行）：

```javascript
  // 捕获选择元素上的点击
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **Step 6：在选择点击时添加指示栏更新**

在点击处理程序中的 `sendEvent` 调用之后添加：

```javascript
    // 更新指示栏
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **Step 7：从 `window.brainstorm` API 中移除 `sendToClaude`**

更新 `window.brainstorm` 对象（第 132-136 行）以移除 `sendToClaude`：

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **Step 8：运行测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **Step 9：提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### Task 4：为新结构更新测试

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`

**注意：** 下面的行号引用来自_原始_文件。Task 2 在文件更早的位置插入了新测试，所以实际行号会有偏移。通过测试的 `console.log` 标签（例如"Test 5:"、"Test 6:"）来查找测试。

- [ ] **Step 1：更新测试 5（完整文档断言）**

找到测试 5 的断言 `!fullRes.body.includes('feedback-footer')`。改为：完整文档也不应有指示栏（它们按原样提供）：

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **Step 2：更新测试 6（片段包装）**

第 125 行：将 `feedback-footer` 断言替换为指示栏断言：

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

同时验证内容占位符被替换（片段内容出现，占位注释不存在）：

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **Step 3：更新测试 7（helper.js API）**

第 140-142 行：更新断言以反映新的 API 表面：

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **Step 4：用指示栏测试替换测试 8（sendToClaude 主题）**

替换测试 8（第 145-149 行）-- `sendToClaude` 不再存在。改为测试指示栏：

```javascript
    // 测试 8：指示栏使用 CSS 变量（主题支持）
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **Step 5：运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **Step 6：提交**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### Task 5：删除 `wait-for-feedback.sh`

**文件：**
- 删除：`lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **Step 1：验证没有其他文件导入或引用 `wait-for-feedback.sh`**

搜索代码库：
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

预期引用：只有 `visual-companion.md`（在 Task 6 中重写）和可能的发布说明（历史性的，保持不变）。

- [ ] **Step 2：删除文件**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **Step 3：运行测试确认没有东西被破坏**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过（没有测试引用此文件）。

- [ ] **Step 4：提交**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### Task 6：重写 `visual-companion.md`

**文件：**
- 修改：`skills/brainstorming/visual-companion.md`

- [ ] **Step 1：更新"How It Works"描述（第 18 行）**

将关于接收反馈"as JSON"的句子替换为：

```markdown
服务器监视目录中的 HTML 文件，将最新的一个提供给浏览器。你编写 HTML 内容，用户在浏览器中看到并可以点击选择选项。选择被记录到 `.events` 文件中，你在下一个轮次读取。
```

- [ ] **Step 2：更新片段描述（第 20 行）**

从 frame template 提供的内容描述中移除"feedback footer"：

```markdown
**内容片段与完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供（仅注入 helper script）。否则，服务器自动将你的内容包装在 frame template 中 -- 添加 header、CSS 主题、选择指示器和所有交互基础设施。**默认编写内容片段。** 仅在需要完全控制页面时才编写完整文档。
```

- [ ] **Step 3：重写"The Loop"部分（第 36-61 行）**

将整个"The Loop"部分替换为：

```markdown
## The Loop

1. **编写 HTML** 到 `screen_dir` 中的新文件：
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **绝不重复使用文件名** -- 每个屏幕使用新文件
   - 使用 Write tool -- **绝不使用 cat/heredoc**（会在终端中输出噪音）
   - 服务器自动提供最新文件

2. **告诉用户期望什么并结束你的轮次：**
   - 每步都提醒 URL（不只是第一次）
   - 提供屏幕内容的简短文本摘要（例如，"正在显示主页的 3 种布局选项"）
   - 要求他们在终端中回复："看看并告诉我你的想法。如果你想选择一个选项，可以点击。"

3. **在你的下一个轮次** -- 用户在终端中回复后：
   - 如果 `$SCREEN_DIR/.events` 存在则读取 -- 其中包含用户的浏览器交互（点击、选择）作为 JSON 行
   - 与用户的终端文本合并以获得完整信息
   - 终端消息是主要反馈；`.events` 提供结构化的交互数据

4. **迭代或推进** -- 如果反馈改变了当前屏幕，编写新文件（例如 `layout-v2.html`）。仅当当前步骤被验证后才进入下一个问题。

5. 重复直到完成。
```

- [ ] **Step 4：替换"User Feedback Format"部分（第 165-174 行）**

替换为：

```markdown
## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互被记录到 `$SCREEN_DIR/.events`（每行一个 JSON 对象）。当你推送新屏幕时，文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流展示了用户的探索路径 -- 他们可能点击多个选项后才做决定。最后一个 `choice` 事件通常是最终选择，但点击模式可能揭示值得询问的犹豫或偏好。

如果 `.events` 不存在，用户没有与浏览器交互 -- 仅使用他们的终端文本。
```

- [ ] **Step 5：更新"Writing Content Fragments"描述（第 65 行）**

移除"feedback footer"引用：

```markdown
只编写放入页面内的内容。服务器自动将其包装在 frame template 中（header、主题 CSS、选择指示器和所有交互基础设施）。
```

- [ ] **Step 6：更新 Reference 部分（第 200-203 行）**

移除 helper.js 关于"JS API"的引用描述 -- API 现在已经极简。保留路径引用：

```markdown
## Reference

- Frame template（CSS 参考）：`${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script（客户端）：`${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **Step 7：提交**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### Task 7：最终验证

- [ ] **Step 1：运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **Step 2：手动冒烟测试**

手动启动服务器并验证流程端到端工作：

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

编写一个测试片段，在浏览器中打开，点击一个选项，验证 `.events` 文件被写入，验证指示栏更新。然后停止服务器：

```bash
lib/brainstorm-server/stop-server.sh <start output 中的 screen_dir>
```

- [ ] **Step 3：验证没有残留的过期引用**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

预期：除发布说明和 spec/plan 文档（历史性）外没有其他匹配。

- [ ] **Step 4：如需清理则最终提交**

```bash
git status
# 审查未跟踪/修改的文件，按需暂存特定文件，确认干净后提交
```
