# withResolvers.ts 研究文档

## 场景与职责

`withResolvers.ts` 提供 `Promise.withResolvers()` 的 polyfill 实现。这是 ES2024 新增的标准 API，但 Claude Code 需要支持 Node.js 18+（package.json 中声明的 engines 字段）。

**核心使用场景：**
- 需要手动控制 Promise 的 resolve/reject 时
- 异步操作的回调模式转换为 Promise 模式
- 与标准 `Promise.withResolvers()` API 保持兼容，便于未来迁移

## 功能点目的

### Promise.withResolvers Polyfill
- **目的**：在 Node.js 18-21 环境中提供 ES2024 标准 API 的兼容实现
- **标准行为**：返回 `{ promise, resolve, reject }` 对象，允许外部代码控制 Promise 状态

## 具体技术实现

### 关键代码

```typescript
export function withResolvers<T>(): PromiseWithResolvers<T> {
  let resolve!: (value: T | PromiseLike<T>) => void
  let reject!: (reason?: unknown) => void
  const promise = new Promise<T>((res, rej) => {
    resolve = res
    reject = rej
  })
  return { promise, resolve, reject }
}
```

### 类型定义

```typescript
// TypeScript 内置类型（ES2024）
interface PromiseWithResolvers<T> {
  promise: Promise<T>
  resolve: (value: T | PromiseLike<T>) => void
  reject: (reason?: unknown) => void
}
```

### 使用模式

```typescript
// 典型使用场景
const { promise, resolve, reject } = withResolvers<string>()

// 在某个异步回调中 resolve
someAsyncOperation((result, error) => {
  if (error) {
    reject(error)
  } else {
    resolve(result)
  }
})

// 等待结果
const value = await promise
```

## 关键代码路径与文件引用

### 导出函数
- `src/utils/withResolvers.ts:5` - `withResolvers<T>()`

### 调用方（通过代码搜索）
该模块被设计为通用工具函数，可能在以下场景使用：
- 异步锁实现
- 事件到 Promise 的转换
- 需要外部控制 Promise 状态的场景

## 依赖与外部交互

### 外部依赖
无外部依赖，纯 TypeScript/JavaScript 实现。

### 内部依赖
无内部依赖。

## 风险、边界与改进建议

### 已知风险

1. **未来兼容性**
   - 当项目升级到 Node.js 22+ 时，此 polyfill 可能与原生实现并存
   - 需要评估是否需要移除或保留为兼容层

2. **非标准扩展**
   - 当前实现严格遵循 ES2024 规范
   - 如果标准有更新，需要同步修改

### 边界情况

1. **多次调用 resolve/reject**
   - 遵循 Promise 规范：只有第一次调用有效
   - 后续调用静默忽略

2. **resolve 自身 Promise**
   - 遵循 Promise 规范：如果 resolve 的值是 promise 本身，会抛出 TypeError

### 改进建议

1. **条件导出**
   ```typescript
   export const withResolvers = 
     typeof Promise.withResolvers === 'function'
       ? Promise.withResolvers.bind(Promise)
       : withResolversPolyfill
   ```
   这样可以在支持原生 API 的环境中直接使用原生实现。

2. **文档完善**
   - 添加使用示例到 JSDoc
   - 说明与原生 API 的兼容性保证

3. **迁移计划**
   - 记录当项目最低 Node.js 版本升级到 22+ 时的移除计划
