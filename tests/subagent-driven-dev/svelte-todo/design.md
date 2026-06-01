# Svelte Todo List - 设计

## 概述

使用 Svelte 构建的简单 todo 列表应用程序。支持创建、完成和删除 todos，并具有 localStorage 持久性。

## 功能

- 添加新 todos
- 将 todos 标记为完成/未完成
- 删除 todos
- 按筛选: 全部 / 活跃 / 已完成
- 清除所有已完成的 todos
- 持久化到 localStorage
- 显示剩余项目计数

## 用户界面

```
┌─────────────────────────────────────────┐
│  Svelte Todos                           │
├─────────────────────────────────────────┤
│  [________________________] [Add]       │
├─────────────────────────────────────────┤
│  [ ] Buy groceries                  [x] │
│  [✓] Walk the dog                   [x] │
│  [ ] Write code                     [x] │
├─────────────────────────────────────────┤
│  2 items left                           │
│  [All] [Active] [Completed]  [Clear ✓]  │
└─────────────────────────────────────────┘
```

## 组件

```
src/
  App.svelte           # 主应用，状态管理
  lib/
    TodoInput.svelte   # 文本输入 + 添加按钮
    TodoList.svelte    # 列表容器
    TodoItem.svelte    # 单个 todo 带复选框、文本、删除
    FilterBar.svelte   # 筛选按钮 + 清除已完成
    store.ts           # Svelte store 用于 todos
    storage.ts         # localStorage 持久性
```

## 数据模型

```typescript
interface Todo {
  id: string;        // UUID
  text: string;      // Todo 文本
  completed: boolean;
}

type Filter = 'all' | 'active' | 'completed';
```

## 验收标准

1. 可以通过输入并按 Enter 或点击添加来添加 todo
2. 可以通过点击复选框来切换 todo 完成
3. 可以通过点击 X 按钮来删除 todo
4. 筛选按钮显示 todos 的正确子集
5. "X items left" 显示未完成 todos 的计数
6. "Clear completed" 删除所有已完成的 todos
7. Todos 跨页面刷新持久化（localStorage）
8. 空状态显示有用的消息
9. 所有测试通过
