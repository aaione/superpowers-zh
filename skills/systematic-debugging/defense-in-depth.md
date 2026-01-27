# Defense-in-Depth Validation (纵深防御验证)

## 概述 (Overview)

当你修复由无效数据引起的 bug 时，在一个地方添加验证感觉就足够了。但那个单一检查可能会被不同的代码路径、重构或 mocks 绕过。

**核心原则:** 在数据经过的每一层进行验证。使 bug 在结构上变为不可能。

## 为什么多层 (Why Multiple Layers)

单一验证: "We fixed the bug"
多层验证: "We made the bug impossible"

不同的层捕获不同的情况:
- 入口验证捕获大多数 bugs
- 业务逻辑捕获边界情况
- 环境守卫防止特定上下文的危险
- 调试日志在其他层失败时提供帮助

## 四个层 (The Four Layers)

### Layer 1: 入口点验证 (Entry Point Validation)
**目的:** 在 API 边界拒绝明显无效的输入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... proceed
}
```

### Layer 2: 业务逻辑验证 (Business Logic Validation)
**目的:** 确保数据对此操作有意义

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### Layer 3: 环境守卫 (Environment Guards)
**目的:** 防止特定上下文中的危险操作

```typescript
async function gitInit(directory: string) {
  // In tests, refuse git init outside temp directories
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... proceed
}
```

### Layer 4: 调试仪器 (Debug Instrumentation)
**目的:** 捕获取证上下文

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... proceed
}
```

## 应用模式 (Applying the Pattern)

当你发现 bug 时:

1.  **追踪数据流** - 坏值源自哪里？哪里使用？
2.  **映射所有检查点** - 列出数据经过的每一点
3.  **在每一层添加验证** - 入口, 业务, 环境, 调试
4.  **测试每一层** - 尝试绕过 layer 1，验证 layer 2 捕获它

## 会话示例

Bug: 空 `projectDir` 导致源代码中 `git init`

**数据流:**
1. Test setup → empty string
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` runs in `process.cwd()`

**添加了四个层:**
- Layer 1: `Project.create()` 验证非空/存在/可写
- Layer 2: `WorkspaceManager` 验证 projectDir 非空
- Layer 3: `WorktreeManager` 在测试中拒绝 tmpdir 之外的 git init
- Layer 4: git init 之前的堆栈跟踪日志

**结果:** 所有 1847 个测试通过，bug 无法复现

## 关键见解 (Key Insight)

所有四个层都是必要的。在测试期间，每一层都捕获了其他层错过的 bugs:
- 不同的代码路径绕过了入口验证
- Mocks 绕过了业务逻辑检查
- 不同平台上的边界情况需要环境守卫
- 调试日志识别了结构性误用

**不要止步于一个验证点。** 在每一层添加检查。
