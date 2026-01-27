# Testing Anti-Patterns

**加载此参考当:** 编写或更改测试，添加 mocks，或试图在生产代码中添加仅测试用方法时。

## 概述 (Overview)

测试必须验证真实行为，而不是 mock 行为。Mocks 是隔离的手段，而不是被测试的对象。

**核心原则:** 测试代码做什么，而不是 mocks 做什么。

**遵循严格的 TDD 可以防止这些反模式。**

## 铁律 (The Iron Laws)
```
1. 绝不测试 mock 行为
2. 绝不在生产类中添加仅测试用方法
3. 绝不在不了解依赖关系的情况下 mock
```

## Anti-Pattern 1: 测试 Mock 行为

**违规:**
```typescript
// ❌ BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错的:**
- 你正在验证 mock 工作，而不是组件工作
- 当 mock 存在时测试通过，不存在时失败
- 没有告诉你关于真实行为的任何信息

**你的人类伙伴的纠正:** "Are we testing the behavior of a mock?"

**修复:**
```typescript
// ✅ GOOD: Test real component or don't mock it
test('renders sidebar', () => {
  render(<Page />);  // Don't mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### Gate Function (把关函数)

```
在对任何 mock 元素进行断言之前:
  问: "Am I testing real component behavior or just mock existence?"

  如果是测试 mock 存在:
    停止 - 删除断言或取消 mock 组件

  测试真实行为代替
```

## Anti-Pattern 2: 生产中的仅测试用方法

**违规:**
```typescript
// ❌ BAD: destroy() only used in tests
class Session {
  async destroy() {  // Looks like production API!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// In tests
afterEach(() => session.destroy());
```

**为什么这是错的:**
- 生产类被仅测试用代码污染
- 如果在生产中意外调用很危险
- 违反 YAGNI 和关注点分离
- 混淆对象生命周期与实体生命周期

**修复:**
```typescript
// ✅ GOOD: Test utilities handle test cleanup
// Session has no destroy() - it's stateless in production

// In test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// In tests
afterEach(() => cleanupSession(session));
```

### Gate Function

```
在向生产类添加任何方法之前:
  问: "Is this only used by tests?"

  如果是:
    停止 - 不要添加它
    把它放在测试工具中代替

  问: "Does this class own this resource's lifecycle?"

  如果否:
    停止 - 此方法的类错误
```

## Anti-Pattern 3: 不理解的情况下 Mocking

**违规:**
```typescript
// ❌ BAD: Mock breaks test logic
test('detects duplicate server', () => {
  // Mock prevents config write that test depends on!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // Should throw - but won't!
});
```

**为什么这是错的:**
- Mocked 方法具有测试依赖的副作用 (写配置)
- 过度 mocking 以 "be safe" 破坏了实际行为
- 测试因错误原因通过或莫名其妙地失败

**修复:**
```typescript
// ✅ GOOD: Mock at correct level
test('detects duplicate server', () => {
  // Mock the slow part, preserve behavior test needs
  vi.mock('MCPServerManager'); // Just mock slow server startup

  await addServer(config);  // Config written
  await addServer(config);  // Duplicate detected ✓
});
```

### Gate Function

```
在 mock 任何方法之前:
  停止 - 暂时不要 mock

  1. 问: "What side effects does the real method have?"
  2. 问: "Does this test depend on any of those side effects?"
  3. 问: "Do I fully understand what this test needs?"

  如果依赖副作用:
    Mock 在较低级别 (实际的慢/外部操作)
    或使用保留必要行为的测试替身
    而不是测试依赖的高级方法

  如果不确定测试依赖什么:
    首先用真实实施运行测试
    观察实际需要发生什么
    然后在正确级别添加最小 mocking

  危险信号:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

## Anti-Pattern 4: 不完整的 Mocks

**违规:**
```typescript
// ❌ BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**为什么这是错的:**
- **部分 mocks 隐藏结构假设** - 你只 mocked 你知道的字段
- **下游代码可能依赖你没包含的字段** - 沉默的失败
- **测试通过但集成失败** - Mock 不完整，真实 API 完整
- **虚假的信心** - 测试未证明关于真实行为的任何事情

**铁律:** Mock **完整**的数据结构，正如现实中存在的那样，而不仅仅是你当前测试使用的字段。

**修复:**
```typescript
// ✅ GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

### Gate Function

```
在创建 mock 响应之前:
  检查: "What fields does the real API response contain?"

  行动:
    1. 从 docs/examples 检查实际 API 响应
    2. 包含系统下游可能消费的所有字段
    3. 验证 mock 完全匹配真实响应 schema

  关键:
    如果你正在创建一个 mock，你必须理解完整的结构
    当代码依赖省略的字段时，部分 mocks 会静默失败

  如果不确定: 包含所有记录的字段
```

## Anti-Pattern 5: 集成测试作为事后想法

**违规:**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**为什么这是错的:**
- 测试是实施的一部分，不是可选的后续
- TDD 会抓住这一点
- 没有测试不能声称完成

**修复:**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## 当 Mocks 变得太复杂

**警告信号:**
- Mock 设置比测试逻辑长
- Mocking 一切以使测试通过
- Mocks 缺少真实组件拥有的方法
- 当 mock 更改时测试中断

**你的人类伙伴的问题:** "Do we need to be using a mock here?"

**考虑:** 带有真实组件的集成测试通常比复杂的 mocks 更简单

## TDD 防止这些反模式

**为什么 TDD 有帮助:**
1.  **先写测试** → 迫使你思考你实际上在测试什么
2.  **看着它失败** → 确认测试测试真实行为，而不是 mocks
3.  **最小实施** → 没有仅测试用方法潜入
4.  **真实依赖** → 在 mocking 之前你看到测试实际上需要什么

**如果你正在测试 mock 行为，你违反了 TDD** - 你在没有看着测试针对真实代码先失败的情况下添加了 mocks。

## 快速参考 (Quick Reference)

| Anti-Pattern | Fix |
|--------------|-----|
| Assert on mock elements | 测试真实组件或取消 mock |
| Test-only methods in production | 转移到测试工具 |
| Mock without understanding | 首先理解依赖关系，mock 最小化 |
| Incomplete mocks | 完整镜像真实 API |
| Tests as afterthought | TDD - tests first |
| Over-complex mocks | 考虑集成测试 |

## 危险信号 (Red Flags)

- 断言检查 `*-mock` test IDs
- 方法仅在测试文件中调用
- Mock 设置 > 50% 的测试
- 当你移除 mock 时测试失败
- 无法解释为什么需要 mock
- Mocking "just to be safe"

## 总结 (The Bottom Line)

**Mocks 是隔离的工具，而不是要测试的东西。**

如果 TDD 揭示你正在测试 mock 行为，你就错了。

修复: 测试真实行为或质疑为什么你根本需要在 mock。
