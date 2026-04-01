# tokenize.ts 研究报告

## 场景与职责

`tokenize.ts` 实现了 **ANSI 转义序列的分词器（Tokenizer）**。与 `parser.ts` 的语义解析不同，分词器只负责识别转义序列的边界，将输入分割为 `text`（纯文本）和 `sequence`（转义序列）两种 token。

该文件的核心职责：
1. **流式分词** —— 支持增量输入，维护解析状态
2. **序列边界检测** —— 准确识别 CSI、OSC、DCS、APC、SS3、ESC 序列的起止
3. **状态机实现** —— 使用有限状态机处理各种序列类型
4. **X10 鼠标支持** —— 可选的 X10 鼠标事件特殊处理

## 功能点目的

### 1. Token 类型

```typescript
export type Token =
  | { type: 'text'; value: string }
  | { type: 'sequence'; value: string }
```

### 2. 状态机状态

```typescript
type State =
  | 'ground'           // 基础状态，处理普通文本
  | 'escape'           // 收到 ESC，等待第二个字节
  | 'escapeIntermediate'  // ESC + 中间字节，等待最终字节
  | 'csi'              // CSI 序列（ESC [），等待终止字节
  | 'ss3'              // SS3 序列（ESC O），等待最终字节
  | 'osc'              // OSC 序列（ESC ]），等待终止符
  | 'dcs'              // DCS 序列（ESC P），等待终止符
  | 'apc'              // APC 序列（ESC _），等待终止符
```

### 3. Tokenizer 接口

```typescript
export type Tokenizer = {
  feed(input: string): Token[]    // 输入数据，返回 token 列表
  flush(): Token[]                // 刷新缓冲区，返回剩余 token
  reset(): void                   // 重置状态
  buffer(): string                // 获取当前缓冲的不完整序列
}
```

### 4. 创建函数

```typescript
type TokenizerOptions = {
  x10Mouse?: boolean  // 是否处理 X10 鼠标事件前缀
}

export function createTokenizer(options?: TokenizerOptions): Tokenizer
```

## 具体技术实现

### 状态机转换

```
[ground]
  ├─ 收到 ESC ──> [escape]
  └─ 其他字符 ──> 继续 ground

[escape]
  ├─ 收到 [ (0x5b) ──> [csi]
  ├─ 收到 ] (0x5d) ──> [osc]
  ├─ 收到 P (0x50) ──> [dcs]
  ├─ 收到 _ (0x5f) ──> [apc]
  ├─ 收到 O (0x4f) ──> [ss3]
  ├─ 中间字节 (0x20-0x2f) ──> [escapeIntermediate]
  ├─ 最终字节 (0x30-0x7e) ──> 完成序列，返回 ground
  ├─ 收到 ESC ──> 完成当前序列，开始新序列
  └─ 其他 ──> 无效，返回 ground

[escapeIntermediate]
  ├─ 中间字节 ──> 继续
  ├─ 最终字节 ──> 完成序列，返回 ground
  └─ 其他 ──> 无效，返回 ground

[csi]
  ├─ 最终字节 (0x40-0x7e) ──> 完成序列，返回 ground
  ├─ 参数字节 (0x30-0x3f) ──> 继续
  ├─ 中间字节 (0x20-0x2f) ──> 继续
  ├─ X10 鼠标特殊处理 ──> 消费 3 字节 payload
  └─ 其他 ──> 无效，返回 ground

[ss3]
  ├─ 最终字节 (0x40-0x7e) ──> 完成序列，返回 ground
  └─ 其他 ──> 无效，返回 ground

[osc/dcs/apc]
  ├─ 收到 BEL ──> 完成序列，返回 ground
  ├─ 收到 ESC \ ──> 完成序列，返回 ground
  └─ 其他 ──> 继续
```

### X10 鼠标特殊处理

```typescript
// X10 鼠标: CSI M + 3 原始 payload 字节 (Cb+32, Cx+32, Cy+32)
if (
  x10Mouse &&
  code === 0x4d /* M */ &&
  i - seqStart === 2 &&  // M 在 ESC [ 之后
  (i + 1 >= data.length || data.charCodeAt(i + 1) >= 0x20) &&
  (i + 2 >= data.length || data.charCodeAt(i + 2) >= 0x20) &&
  (i + 3 >= data.length || data.charCodeAt(i + 3) >= 0x20)
) {
  if (i + 4 <= data.length) {
    i += 4  // 消费 M + 3 payload 字节
    emitSequence(data.slice(seqStart, i))
  } else {
    // 不完整，等待更多数据
    i = data.length
  }
}
```

**注意**：X10 鼠标处理仅在 `x10Mouse: true` 时启用，且仅用于 stdin 输入。在输出流中，`\x1b[M` 也是 CSI DL（删除行），盲目启用会损坏输出。

### 核心分词循环

```typescript
function tokenize(
  input: string,
  initialState: State,
  initialBuffer: string,
  flush: boolean,
  x10Mouse: boolean,
): { tokens: Token[]; state: InternalState } {
  const tokens: Token[] = []
  const data = initialBuffer + input
  let i = 0
  let textStart = 0
  let seqStart = 0

  const flushText = (): void => {
    if (i > textStart) {
      const text = data.slice(textStart, i)
      if (text) tokens.push({ type: 'text', value: text })
    }
    textStart = i
  }

  const emitSequence = (seq: string): void => {
    if (seq) tokens.push({ type: 'sequence', value: seq })
    result.state = 'ground'
    textStart = i
  }

  // 状态机主循环...
}
```

### 结束处理

```typescript
// 输入结束时的处理
if (result.state === 'ground') {
  flushText()  // ground 状态：输出剩余文本
} else if (flush) {
  // flush=true：强制输出不完整序列
  const remaining = data.slice(seqStart)
  if (remaining) tokens.push({ type: 'sequence', value: remaining })
  result.state = 'ground'
} else {
  // flush=false：缓冲不完整序列等待更多数据
  result.buffer = data.slice(seqStart)
}
```

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `C0`, `ESC_TYPE`, `isEscFinal` | `./ansi.js` | 控制字符和 ESC 类型常量 |
| `isCSIFinal`, `isCSIIntermediate`, `isCSIParam` | `./csi.js` | CSI 字节分类 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `parser.ts` | `createTokenizer`, `Tokenizer` | Parser 内部使用 |
| `parse-keypress.ts` | `createTokenizer` | 键盘输入解析（启用 x10Mouse） |
| `tabstops.ts` | `createTokenizer` | Tab 扩展处理 |

### 导出内容

```typescript
export type Token = 
  | { type: 'text'; value: string }
  | { type: 'sequence'; value: string }

export type Tokenizer = {
  feed(input: string): Token[]
  flush(): Token[]
  reset(): void
  buffer(): string
}

export function createTokenizer(options?: { x10Mouse?: boolean }): Tokenizer
```

## 依赖与外部交互

### 内部依赖

- `ansi.ts`：`C0` 控制字符、`ESC_TYPE` 引入符类型、`isEscFinal()`
- `csi.ts`：`isCSIFinal()`、`isCSIIntermediate()`、`isCSIParam()`

### 外部使用场景

1. **Parser 内部使用**：`parser.ts` 创建 tokenizer 处理输入：
   ```typescript
   private tokenizer: Tokenizer = createTokenizer()
   
   feed(input: string): Action[] {
     const tokens = this.tokenizer.feed(input)
     // 处理 tokens...
   }
   ```

2. **键盘输入解析**：`parse-keypress.ts` 启用 `x10Mouse` 处理鼠标事件：
   ```typescript
   const tokenizer = prevState._tokenizer ?? createTokenizer({ x10Mouse: true })
   ```

3. **Tab 扩展**：`tabstops.ts` 使用 tokenizer 避免处理转义序列中的 Tab：
   ```typescript
   const tokenizer = createTokenizer()
   const tokens = tokenizer.feed(text)
   // 只在 text token 中扩展 Tab
   ```

## 风险、边界与改进建议

### 边界情况

1. **双 ESC**：`ESC ESC` 被视为两个独立序列
2. **无效序列**：CSI 中的非参数字节导致序列中止，ESC 后的文本被重新处理
3. **不完整序列**：默认缓冲等待更多数据，`flush()` 可强制输出
4. **X10 鼠标限制**：UTF-8 编码下，坐标字节可能合并为单个字符（162+ 列时）

### 风险

1. **状态机复杂性**：8 个状态，转换逻辑复杂，容易出错
2. **X10 鼠标与 CSI DL 冲突**：`\x1b[M` 既是 X10 鼠标也是删除行，需要上下文区分
3. **性能**：每个字符都进行状态检查和分支，高频输入可能有开销
4. **内存使用**：缓冲不完整序列可能累积数据（虽然通常很小）

### 改进建议

1. **添加更多序列类型支持**：
   - SOS (ESC X) 序列
   - PM (ESC ^) 序列
   - 更完整的 DCS 解析（目前只识别边界）

2. **优化 X10 鼠标处理**：
   - 考虑使用 latin1 编码处理 stdin 避免 UTF-8 合并问题
   - 添加 SGR 鼠标（1006 模式）支持

3. **性能优化**：
   - 使用查找表替代分支判断
   - 批量处理 ground 状态的文本
   - 使用 TypedArray 存储状态

4. **错误恢复**：
   - 添加上下文感知的错误恢复
   - 记录无效序列用于调试

5. **流式 API**：
   ```typescript
   export function* tokenizeStream(input: string): Generator<Token>
   ```

6. **文档改进**：
   - 添加状态机图
   - 添加每种序列类型的示例
   - 说明 X10 鼠标限制

### 测试建议

- 测试所有状态转换
- 测试各种序列类型的边界检测
- 测试不完整序列的缓冲和 flush
- 测试 X10 鼠标处理（启用和禁用）
- 测试无效序列的容错
- 测试双 ESC 处理
- 测试长文本的性能
- 验证与 `parser.ts` 和 `parse-keypress.ts` 的集成
