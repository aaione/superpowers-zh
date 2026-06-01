# Svelte Todo List - 实施计划

使用 `superpowers:subagent-driven-development` skill 执行此计划。

## 上下文

使用 Svelte 构建 todo 列表应用。完整规范见 `design.md`。

## 任务

### 任务 1: 项目设置

创建带有 Vite 的 Svelte 项目。

**执行:**
- 运行 `npm create vite@latest . -- --template svelte-ts`
- 使用 `npm install` 安装依赖
- 验证 dev 服务器工作
- 从 App.svelte 清理默认 Vite 模板内容

**验证:**
- `npm run dev` 启动服务器
- 应用显示最小的 "Svelte Todos" 标题
- `npm run build` 成功

---

### 任务 2: Todo Store

创建用于 todo 状态管理的 Svelte store。

**执行:**
- 创建 `src/lib/store.ts`
- 定义带有 id、text、completed 的 `Todo` 接口
- 创建带有初始空数组的可写 store
- 导出函数: `addTodo(text)`, `toggleTodo(id)`, `deleteTodo(id)`, `clearCompleted()`
- 创建 `src/lib/store.test.ts`，测试每个函数

**验证:**
- 测试通过: `npm run test`（如需要安装 vitest）

---

### 任务 3: localStorage 持久性

为 todos 添加持久性层。

**执行:**
- 创建 `src/lib/storage.ts`
- 实现 `loadTodos(): Todo[]` 和 `saveTodos(todos: Todo[])`
- 优雅地处理 JSON 解析错误（返回空数组）
- 与 store 集成: 初始化时加载，更改时保存
- 添加 load/save/错误处理的测试

**验证:**
- 测试通过
- 手动测试: 添加 todo，刷新页面，todo 持久化

---

### 任务 4: TodoInput 组件

创建用于添加 todos 的输入组件。

**执行:**
- 创建 `src/lib/TodoInput.svelte`
- 绑定到本地状态的文本输入
- 添加按钮调用 `addTodo()` 并清除输入
- Enter 键也提交
- 当输入为空时禁用添加按钮
- 添加组件测试

**验证:**
- 测试通过
- 组件渲染输入和按钮

---

### 任务 5: TodoItem 组件

创建单个 todo 项目组件。

**执行:**
- 创建 `src/lib/TodoItem.svelte`
- Props: `todo: Todo`
- 复选框切换完成（调用 `toggleTodo`）
- 完成时文本带有删除线
- 删除按钮 (X) 调用 `deleteTodo`
- 添加组件测试

**验证:**
- 测试通过
- 组件渲染复选框、文本、删除按钮

---

### 任务 6: TodoList 组件

创建列表容器组件。

**执行:**
- 创建 `src/lib/TodoList.svelte`
- Props: `todos: Todo[]`
- 为每个 todo 渲染 TodoItem
- 为空时显示 "No todos yet"

**验证:**
- 测试通过
- 组件渲染 TodoItem 列表

---

### 任务 7: FilterBar 组件

创建筛选和状态栏组件。

**执行:**
- 创建 `src/lib/FilterBar.svelte`
- Props: `todos: Todo[]`, `filter: Filter`, `onFilterChange: (f: Filter) => void`
- 显示计数: "X items left"（未完成计数）
- 三个筛选按钮: 全部、活跃、已完成
- 活跃筛选按钮在视觉上突出显示
- "Clear completed" 按钮（无已完成 todos 时隐藏）
- 添加组件测试

**验证:**
- 测试通过
- 组件渲染计数、筛选、清除按钮

---

### 任务 8: 应用集成

在 App.svelte 中连接所有组件。

**执行:**
- 导入所有组件和 store
- 添加筛选状态（默认: 'all'）
- 根据筛选状态计算过滤的 todos
- 渲染: 标题、TodoInput、TodoList、FilterBar
- 将适当的 props 传递给每个组件

**验证:**
- 应用渲染所有组件
- 添加 todos 工作
- 切换工作
- 删除工作

---

### 任务 9: 筛选功能

确保端到端的筛选工作。

**执行:**
- 验证筛选按钮更改显示的 todos
- 'all' 显示所有 todos
- 'active' 仅显示未完成的 todos
- 'completed' 仅显示已完成的 todos
- 清除已完成删除已完成的 todos，并在需要时重置筛选
- 添加集成测试

**验证:**
- 筛选测试通过
- 所有筛选状态的手动验证

---

### 任务 10: 样式和完善

添加 CSS 样式以提高可用性。

**执行:**
- 样式化应用以匹配设计模型
- 已完成的 todos 具有删除线和柔和的颜色
- 活跃筛选按钮突出显示
- 输入具有焦点样式
- 鼠标悬停时出现删除按钮（或在移动端始终显示）
- 响应式布局

**验证:**
- 应用在视觉上可用
- 样式不破坏功能

---

### 任务 11: 端到端测试

添加完整用户流程的 Playwright 测试。

**执行:**
- 安装 Playwright: `npm init playwright@latest`
- 创建 `tests/todo.spec.ts`
- 测试流程:
  - 添加 todo
  - 完成 todo
  - 删除 todo
  - 筛选 todos
  - 清除已完成
  - 持久性（添加、重新加载、验证）

**验证:**
- `npx playwright test` 通过

---

### 任务 12: README

记录项目。

**执行:**
- 创建 `README.md`，包括:
  - 项目描述
  - 设置: `npm install`
  - 开发: `npm run dev`
  - 测试: `npm test` 和 `npx playwright test`
  - 构建: `npm run build`

**验证:**
- README 准确描述项目
- 说明工作
