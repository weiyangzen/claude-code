# stream.ts 研究文档

## 场景与职责

`stream.ts` 提供了一个轻量级的异步流实现 `Stream<T>` 类，用于在 Claude Code 内部处理异步数据流。该实现是一个自定义的异步迭代器，支持生产者-消费者模式，适用于需要流式处理数据的场景。

## 功能点目的

### 异步流抽象
- **目的**: 提供比原生 Node.js Stream 更轻量的异步数据流
- **特点**: 
  - 纯 TypeScript 实现，无外部依赖
  - 支持背压控制（通过队列）
  - 支持错误传播
  - 支持完成回调

### 核心功能
1. **数据入队** (`enqueue`): 生产者向流中添加数据
2. **完成标记** (`done`): 标记流结束
3. **错误传播** (`error`): 向消费者传播错误
4. **资源清理** (`return`): 支持迭代器 return 协议

## 具体技术实现

### 类结构

```typescript
export class Stream<T> implements AsyncIterator<T> {
  private readonly queue: T[] = []
  private readResolve?: (value: IteratorResult<T>) => void
  private readReject?: (error: unknown) => void
  private isDone: boolean = false
  private hasError: unknown | undefined
  private started = false

  constructor(private readonly returned?: () => void) {}
}
```

### 核心机制

#### 1. 队列缓冲
- 使用数组 `queue` 存储待消费的数据
- 消费者通过 `next()` 获取数据
- 当队列为空时，消费者等待 Promise 解决

#### 2. 异步等待模式
```typescript
next(): Promise<IteratorResult<T, unknown>> {
  if (this.queue.length > 0) {
    // 队列有数据，立即返回
    return Promise.resolve({ done: false, value: this.queue.shift()! })
  }
  if (this.isDone) {
    // 流已结束
    return Promise.resolve({ done: true, value: undefined })
  }
  if (this.hasError) {
    // 流有错误
    return Promise.reject(this.hasError)
  }
  // 等待新数据、结束或错误
  return new Promise<IteratorResult<T>>((resolve, reject) => {
    this.readResolve = resolve
    this.readReject = reject
  })
}
```

#### 3. 单次迭代保护
```typescript
[Symbol.asyncIterator](): AsyncIterableIterator<T> {
  if (this.started) {
    throw new Error('Stream can only be iterated once')
  }
  this.started = true
  return this
}
```

### 状态转换

```
初始状态 -> started = true (首次迭代)
         -> 数据入队 -> 消费者获取
         -> done() -> isDone = true -> 消费者收到 done: true
         -> error(e) -> hasError = e -> 消费者收到 reject
```

## 关键代码路径与文件引用

### 本文件导出
- `Stream<T>`: 异步流类

### 依赖模块
无外部依赖，纯 TypeScript 实现。

### 调用方

通过 Grep 搜索未发现直接的 `import` 引用。这是一个基础工具类，可能用于：
- SDK 消息流处理
- 内部异步管道
- 测试模拟

## 依赖与外部交互

### 异步迭代器协议
实现标准的 `AsyncIterator<T>` 接口：
- `[Symbol.asyncIterator]()`: 返回异步可迭代对象
- `next()`: 返回 `Promise<IteratorResult<T>>`
- `return?()`: 可选的提前终止处理

### Promise 状态管理
使用两个可选的回调函数管理异步状态：
- `readResolve`: 解决等待中的 `next()` 调用
- `readReject`: 拒绝等待中的 `next()` 调用

## 风险、边界与改进建议

### 潜在风险

1. **内存泄漏**: 如果生产者入队速度快于消费者，队列可能无限增长
2. **竞态条件**: 需要确保 `readResolve`/`readReject` 的赋值和调用是原子的
3. **错误处理**: 错误发生后，队列中未消费的数据被丢弃

### 边界情况

1. **空流**: `done()` 在没有任何 `enqueue` 时调用，消费者立即收到 `done: true`
2. **错误后入队**: 错误发生后调用 `enqueue`，数据被忽略
3. **多次结束**: 多次调用 `done()`，只有第一次有效

### 改进建议

1. **背压控制**: 添加队列大小限制
```typescript
constructor(
  private readonly returned?: () => void,
  private readonly maxQueueSize: number = Infinity
) {}

enqueue(value: T): boolean {
  if (this.queue.length >= this.maxQueueSize) {
    return false // 背压信号
  }
  // ...
}
```

2. **可读流转换**: 提供与 Node.js ReadableStream 的互操作
```typescript
static fromReadableStream<T>(stream: ReadableStream<T>): Stream<T>
toReadableStream(): ReadableStream<T>
```

3. **组合操作**: 添加 map/filter/reduce 等流式操作
```typescript
map<U>(fn: (value: T) => U): Stream<U>
filter(predicate: (value: T) => boolean): Stream<T>
```

4. **超时处理**: 为 `next()` 添加超时选项
```typescript
next(timeoutMs?: number): Promise<IteratorResult<T>>
```

5. **调试支持**: 添加当前队列大小、等待状态等诊断信息
```typescript
get debugInfo(): { queueSize: number; isDone: boolean; hasError: boolean }
```
