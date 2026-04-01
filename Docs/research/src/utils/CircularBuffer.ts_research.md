# CircularBuffer.ts 研究文档

## 1. 场景与职责

`CircularBuffer<T>`（位于 `src/utils/CircularBuffer.ts`）是一个泛型定长循环缓冲区实现，核心职责是**在内存中维护一个固定容量的滚动数据窗口**。当写入数据超过容量上限时，自动驱逐最旧的数据项，从而保证内存占用始终有界。该组件属于底层工具类，不依赖任何外部框架或平台 API，可在 Node.js 与浏览器环境中复用。

当前仓库中，它主要服务于两类场景：

- **Shell 任务输出截流**：在 `src/utils/task/TaskOutput.ts` 第 40 行，以 `new CircularBuffer<string>(1000)` 的形式实例化，用于保存后台 Shell 命令输出的最近 1000 行文本（`#recentLines`）。当 UI 需要展示“进度/尾部输出”时，可直接从该缓冲区提取最新行，而无需加载完整的磁盘日志文件。
- **WebSocket 消息重放缓冲**：在 `src/cli/transports/WebSocketTransport.ts` 第 106、132 行，以 `new CircularBuffer(DEFAULT_MAX_BUFFER_SIZE)`（默认值同样为 1000）实例化，用于在断线重连后向服务端重放最近的标准输出消息（`StdoutMessage`），确保连接恢复后客户端状态能够同步。

## 2. 功能点目的

| 方法 | 行号 | 功能目的 |
|------|------|----------|
| `constructor(capacity)` | 10–12 | 初始化定长底层数组 `buffer`，并设定容量上限。 |
| `add(item)` | 18–24 | 单条写入。若缓冲区已满，则覆盖最旧位置的数据，实现 O(1) 时间复杂度的循环写入。 |
| `addAll(items)` | 29–33 | 批量写入，内部循环调用 `add`，方便调用方一次性追加数组或流式数据块。 |
| `getRecent(count)` | 39–50 | 按“从新到旧”的顺序提取最近 `count` 条数据，但返回结果按**旧到新**排列（因为实现上从 `size - available` 处开始顺序读取）。实际返回数量受 `Math.min(count, this.size)` 限制。 |
| `toArray()` | 55–67 | 导出当前缓冲区中全部有效元素，顺序为**最旧到最新**。空缓冲区时直接返回 `[]`。 |
| `clear()` | 72–76 | 清空缓冲区。除了重置 `head` 与 `size` 外，还将底层数组 `buffer.length` 设为 0，显式释放数组槽位引用。 |
| `length()` | 81–83 | 返回当前有效元素个数（非容量上限）。 |

整体而言，该类的设计目标是**以最小的运行时开销（避免数组移位、避免动态扩容）实现有界队列**，同时对外提供“最近 N 条”与“全部导出”两种读取视角。

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 核心数据结构

```typescript
private buffer: T[]    // 定长数组，长度 = capacity
private head = 0       // 下一个写入位置的索引
private size = 0       // 当前有效元素数量（0 ~ capacity）
```

- **未写满阶段**：`size < capacity`，`head` 指向下一个空闲槽位，最旧数据始终位于索引 `0`。
- **写满阶段**：`size === capacity`，`head` 同时指向“最旧数据”所在位置（因为下一次 `add` 会覆盖它）。

### 3.2 `add` 的循环覆盖逻辑（第 18–24 行）

```typescript
add(item: T): void {
  this.buffer[this.head] = item
  this.head = (this.head + 1) % this.capacity
  if (this.size < this.capacity) {
    this.size++
  }
}
```

流程说明：
1. 将新元素写入 `head` 指向的槽位。
2. `head` 循环右移（取模 `capacity`）。
3. 若尚未写满，`size` 自增；否则 `size` 保持为 `capacity`。

该实现保证了写入操作始终是 **O(1)**，无需像普通数组 `push` + `shift` 那样进行整体元素迁移。

### 3.3 `toArray` 的读取策略（第 55–67 行）

```typescript
const start = this.size < this.capacity ? 0 : this.head
for (let i = 0; i < this.size; i++) {
  const index = (start + i) % this.capacity
  result.push(this.buffer[index]!)
}
```

- 若缓冲区未满，`start = 0`，顺序读取 `0 ~ size-1`。
- 若缓冲区已满，`start = head`，此时 `head` 指向最旧元素，顺序读取 `head ~ head+capacity-1`（取模），即可得到“旧 → 新”的完整序列。

### 3.4 `getRecent` 的读取策略（第 39–50 行）

```typescript
const start = this.size < this.capacity ? 0 : this.head
const available = Math.min(count, this.size)
for (let i = 0; i < available; i++) {
  const index = (start + this.size - available + i) % this.capacity
  result.push(this.buffer[index]!)
}
```

计算 `start + this.size - available` 的本质是：从“有效数据段”中跳过前 `size - available` 个较旧的元素，只取最后 `available` 个较新的元素。返回数组内部仍然保持“旧 → 新”顺序。

### 3.5 `clear()` 的特殊处理（第 72–76 行）

```typescript
clear(): void {
  this.buffer.length = 0
  this.head = 0
  this.size = 0
}
```

与常见的“仅重置指针”不同，这里显式将 `this.buffer.length` 设为 0。这意味着：
- 底层数组的所有槽位引用被释放，有助于 GC 回收其中存放的对象（尤其是 `T` 为引用类型时）。
- 但副作用是**再次 `add` 时会触发数组重新分配与扩容**（JavaScript 引擎内部行为），因为 `buffer` 已从定长数组退化为空数组，后续写入会走 V8 等引擎的动态数组路径，直到再次达到 `capacity` 附近。这与构造函数中 `new Array(capacity)` 的预分配意图略有背离。

## 4. 关键代码路径与文件引用

### 4.1 定义文件

- **`src/utils/CircularBuffer.ts`**（共 84 行）
  - 第 5 行：`export class CircularBuffer<T>` 泛型类定义。
  - 第 18 行：`add(item)` 核心写入方法。
  - 第 39 行：`getRecent(count)` 获取最近 N 条。
  - 第 55 行：`toArray()` 导出全部。
  - 第 72 行：`clear()` 清空方法。
  - 第 81 行：`length()` 获取当前长度。

### 4.2 调用方文件

- **`src/utils/task/TaskOutput.ts`**
  - 第 2 行：`import { CircularBuffer } from '../CircularBuffer.js'`
  - 第 40 行：`#recentLines = new CircularBuffer<string>(1000)` — 固定容量 1000 行，用于保存 Shell 输出的最近文本行。
  - 第 204 行注释：提及 `CircularBuffer / progress`，说明 `#recentLines` 在进度回调场景下被使用。
  - 第 280 行注释：说明在 pipe 模式下返回内存中的 CircularBuffer 尾部数据。

- **`src/cli/transports/WebSocketTransport.ts`**
  - 第 4 行：`import { CircularBuffer } from '../../utils/CircularBuffer.js'`
  - 第 22 行：`const DEFAULT_MAX_BUFFER_SIZE = 1000`
  - 第 106 行：`private messageBuffer: CircularBuffer<StdoutMessage>`
  - 第 132 行：`this.messageBuffer = new CircularBuffer(DEFAULT_MAX_BUFFER_SIZE)` — 用于断线重连后的消息重放。

## 5. 依赖与外部交互

`CircularBuffer.ts` 是**零依赖**的纯 TypeScript 工具类：

- 无第三方 npm 包依赖。
- 无 Node.js 内置模块（如 `fs`、`path`）依赖。
- 无浏览器 DOM API 依赖。
- 仅使用 ECMAScript 标准特性：`Array`、`%` 取模运算符、泛型类语法。

外部交互表现为单向数据流：调用方通过 `add` / `addAll` 注入数据，通过 `getRecent` / `toArray` / `length` 读取数据。该类本身不发起任何 I/O、不触发事件、不持有外部回调。

## 6. 风险、边界与改进建议

### 6.1 非空断言（`!`）的潜在 undefined 风险

在 `getRecent`（第 46 行）与 `toArray`（第 63 行）中，均使用了非空断言：

```typescript
result.push(this.buffer[index]!)
```

**风险分析**：
- 从逻辑上看，只要 `index` 计算正确且 `size` 与 `head` 保持一致，`this.buffer[index]` 必然存在有效值。
- 但 `buffer` 是通过 `new Array(capacity)` 创建的稀疏数组，初始值为 `empty × capacity`（即 `undefined`）。如果在实例化后、首次写满前就读取，只要 `index < size`，读到的都是已写入位置，不会触及未初始化槽位。
- **真正的问题在于防御性编程不足**：若未来有开发者误改 `size` 或 `head`（例如并发修改、手动篡改私有字段），可能导致读取到 `undefined` 却未被 TypeScript 类型系统捕获，进而在调用方引发运行时异常（如调用字符串方法时 `Cannot read properties of undefined`）。

**建议**：
- 在读取处增加运行时断言或降级处理：
  ```typescript
  const value = this.buffer[index]
  if (value !== undefined) {
    result.push(value)
  }
  ```
- 或者在严格模式下使用 `noUncheckedIndexedAccess`（若项目 TypeScript 配置允许），让 `this.buffer[index]` 的类型自动变为 `T | undefined`，从而强制处理空值。

### 6.2 `clear()` 中 `buffer.length = 0` 的副作用

如第 3.5 节所述，`clear()` 将 `buffer.length` 置 0 会清空底层数组。虽然有助于释放对象引用，但会导致后续 `add` 操作时：

- 数组需要重新从 0 开始分配内存。
- 失去构造函数中 `new Array(capacity)` 预分配的优化效果。
- 在写入密集场景（如高频日志行追加）中，可能引入微小的 GC 与内存分配开销。

**建议**：
- 若调用方对性能敏感且 `T` 为值类型（如 `string`、`number`），可考虑将 `clear()` 改为仅重置指针，保留数组容量：
  ```typescript
  clear(): void {
    this.head = 0
    this.size = 0
    // 可选：显式填充 undefined 以释放引用，但保留长度
    // this.buffer.fill(undefined as any)
  }
  ```
- 若当前行为（释放引用）是刻意为之（例如 `T` 为大对象，需要尽快 GC），则应在代码注释中明确说明这一设计权衡。

### 6.3 容量为 0 或负数的边界

构造函数（第 10 行）直接接受 `capacity: number`，未做校验。若传入 `0` 或负数：

- `new Array(0)` 会产生空数组，`add` 时写入 `this.buffer[0]` 实际为数组越界赋值（JavaScript 允许，但数组长度会异常增长）。
- `% this.capacity` 在 `capacity = 0` 时会产生 `NaN`，导致 `head` 与后续索引计算全部失效，最终陷入不可恢复状态。

**建议**：在构造函数中增加防御性校验：

```typescript
if (capacity <= 0) {
  throw new RangeError('Capacity must be a positive integer')
}
```

### 6.4 `getRecent` 与 `toArray` 的重复遍历逻辑

两个方法内部均包含几乎相同的索引计算与 `for` 循环。若未来需要增加“过滤”、“映射”等读取变体，代码重复度会进一步上升。

**建议**：提取一个私有迭代器或生成器方法，供 `getRecent`、`toArray` 复用：

```typescript
private *items(): Generator<T> {
  if (this.size === 0) return
  const start = this.size < this.capacity ? 0 : this.head
  for (let i = 0; i < this.size; i++) {
    yield this.buffer[(start + i) % this.capacity]!
  }
}
```

### 6.5 缺少 `peek()` 或 `peekOldest()` 等随机访问能力

当前 API 仅支持“全部导出”或“最近 N 条”，不支持 O(1) 地查看最新或最旧元素。对于 `TaskOutput` 这类只需要看最后一行的场景，当前必须调用 `getRecent(1)` 并构造新数组，存在不必要的数组分配。

**建议**：可扩展以下方法（若调用方有需求）：

```typescript
peek(): T | undefined      // 查看最新元素
peekOldest(): T | undefined // 查看最旧元素
```

### 6.6 类型安全与文档

- 当前 JSDoc 注释已覆盖主要方法，但缺少对 `@throws` 或 `@param` 取值范围的说明。
- 建议补充 `@param capacity - 缓冲区最大容量，必须为正整数` 等约束说明，以提升 IDE 提示体验。

---

**总结**：`CircularBuffer<T>` 是一个轻量、高效的定长循环缓冲区实现，广泛应用于日志截流与消息重放场景。其时间复杂度表现优异（写入 O(1)、读取 O(n)），但在 `clear()` 的内存策略、非空断言的防御性、以及容量边界校验方面仍有改进空间。
