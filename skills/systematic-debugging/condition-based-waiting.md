# Condition-Based Waiting (基于条件的等待)

## 概述 (Overview)

不稳定的测试经常用任意延迟来猜测时间。这会产生 race conditions，测试在快速机器上通过，但在负载下或 CI 中失败。

**核心原则:** 等待你关心的实际条件，而不是猜测它需要多长时间。

## 何时使用 (When to Use)

```dot
digraph when_to_use {
    "Test uses setTimeout/sleep?" [shape=diamond];
    "Testing timing behavior?" [shape=diamond];
    "Document WHY timeout needed" [shape=box];
    "Use condition-based waiting" [shape=box];

    "Test uses setTimeout/sleep?" -> "Testing timing behavior?" [label="yes"];
    "Testing timing behavior?" -> "Document WHY timeout needed" [label="yes"];
    "Testing timing behavior?" -> "Use condition-based waiting" [label="no"];
}
```

**使用场景:**
- 测试有任意延迟 (`setTimeout`, `sleep`, `time.sleep()`)
- 测试不稳定 (有时通过，负载下失败)
- 并行运行时测试超时
- 等待异步操作完成

**不使用场景:**
- 测试实际的时间行为 (debounce, throttle intervals)
- 如果使用任意超时，总是记录**为什么**

## 核心模式 (Core Pattern)

```typescript
// ❌ BEFORE: Guessing at timing
await new Promise(r => setTimeout(r, 50));
const result = getResult();
expect(result).toBeDefined();

// ✅ AFTER: Waiting for condition
await waitFor(() => getResult() !== undefined);
const result = getResult();
expect(result).toBeDefined();
```

## 快速模式 (Quick Patterns)

| 场景 | 模式 |
|----------|---------|
| 等待事件 | `waitFor(() => events.find(e => e.type === 'DONE'))` |
| 等待状态 | `waitFor(() => machine.state === 'ready')` |
| 等待计数 | `waitFor(() => items.length >= 5)` |
| 等待文件 | `waitFor(() => fs.existsSync(path))` |
| 复杂条件 | `waitFor(() => obj.ready && obj.value > 10)` |

## 实施 (Implementation)

通用轮询函数:
```typescript
async function waitFor<T>(
  condition: () => T | undefined | null | false,
  description: string,
  timeoutMs = 5000
): Promise<T> {
  const startTime = Date.now();

  while (true) {
    const result = condition();
    if (result) return result;

    if (Date.now() - startTime > timeoutMs) {
      throw new Error(`Timeout waiting for ${description} after ${timeoutMs}ms`);
    }

    await new Promise(r => setTimeout(r, 10)); // Poll every 10ms
  }
}
```

参见此目录中的 `condition-based-waiting-example.ts` 获取带有特定领域辅助函数 (`waitForEvent`, `waitForEventCount`, `waitForEventMatch`) 的完整实现，来自实际调试会话。

## 常见错误 (Common Mistakes)

**❌ 轮询太快:** `setTimeout(check, 1)` - 浪费 CPU
**✅ 修复:** 每 10ms 轮询一次

**❌ 无超时:** 如果条件从未满足则永久循环
**✅ 修复:** 总是包含带有清晰错误的超时

**❌ 数据过时:** 循环前缓存状态
**✅ 修复:** 在循环内调用 getter 获取新鲜数据

## 何时任意超时是正确的

```typescript
// Tool ticks every 100ms - need 2 ticks to verify partial output
await waitForEvent(manager, 'TOOL_STARTED'); // First: wait for condition
await new Promise(r => setTimeout(r, 200));   // Then: wait for timed behavior
// 200ms = 2 ticks at 100ms intervals - documented and justified
```

**需求:**
1. 先等待触发条件
2. 基于已知时间 (不是猜测)
3. 注释解释**为什么**

## 真实世界影响

来自调试会话 (2025-10-03):
- 修复了 3 个文件中的 15 个不稳定测试
- 通过率: 60% → 100%
- 执行时间: 快 40%
- 不再有 race conditions
