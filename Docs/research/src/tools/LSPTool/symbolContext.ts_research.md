# symbolContext.ts 研究文档

## 场景与职责

symbolContext.ts 是 LSPTool 的符号提取工具模块，负责从源代码文件中提取指定位置的符号（标识符）名称。它主要用于增强 UI 显示，在工具使用消息中显示目标符号名称而非仅仅是位置坐标。

**主要职责：**
- 从文件指定位置提取符号/单词
- 支持多种编程语言的标识符模式
- 提供同步文件读取接口（适配 React 渲染）
- 优雅处理各种错误情况

**使用场景：**
- 用户调用 goToDefinition 时，UI 显示 "symbol: 'foo' in 'src/bar.ts'" 而非 "position: 10:5"
- 增强工具使用消息的可读性和上下文理解

---

## 功能点目的

### 1. 符号提取（getSymbolAtPosition）

```typescript
export function getSymbolAtPosition(
  filePath: string,
  line: number,      // 0-indexed
  character: number  // 0-indexed
): string | null
```

**核心功能：**
- 读取文件内容（限制 64KB）
- 定位到指定行
- 使用正则表达式匹配符号
- 返回包含指定位置的符号

### 2. 多语言符号支持

正则表达式模式：`/[\w$'!]+|[+\-*/%&|^~<>=]+/g`

支持：
- **标准标识符**：字母、数字、下划线、美元符（`\w$`）
- **Rust 生命周期**：`'a`, `'static`（`'!`）
- **Rust 宏**：`macro_name!`
- **运算符**：`+`, `-`, `*`, `/`, `%`, `&`, `|`, `^`, `~`, `<`, `>`, `=`

### 3. 性能优化

- **有限读取**：只读取前 64KB（约 1000 行典型代码）
- **快速失败**：目标行超出范围时立即返回 null
- **符号截断**：限制符号长度最大 30 字符

### 4. 错误处理

- 文件不存在：返回 null
- 权限问题：返回 null
- 编码问题：返回 null
- 行/列超出范围：返回 null

---

## 具体技术实现

### 核心算法

```typescript
export function getSymbolAtPosition(
  filePath: string,
  line: number,      // 0-indexed
  character: number  // 0-indexed
): string | null {
  try {
    const fs = getFsImplementation()
    const absolutePath = expandPath(filePath)

    // 1. 同步读取前 64KB
    const { buffer, bytesRead } = fs.readSync(absolutePath, {
      length: MAX_READ_BYTES,  // 64 * 1024
    })
    const content = buffer.toString('utf-8', 0, bytesRead)
    const lines = content.split('\n')

    // 2. 边界检查
    if (line < 0 || line >= lines.length) {
      return null
    }
    
    // 如果读取了完整 64KB 且目标行是最后一行，
    // 可能行内容被截断，放弃提取
    if (bytesRead === MAX_READ_BYTES && line === lines.length - 1) {
      return null
    }

    const lineContent = lines[line]
    if (!lineContent || character < 0 || character >= lineContent.length) {
      return null
    }

    // 3. 使用正则匹配符号
    const symbolPattern = /[\w$'!]+|[+\-*/%&|^~<>=]+/g
    let match: RegExpExecArray | null

    while ((match = symbolPattern.exec(lineContent)) !== null) {
      const start = match.index
      const end = start + match[0].length

      // 检查位置是否落在此匹配内
      if (character >= start && character < end) {
        const symbol = match[0]
        return truncate(symbol, 30)  // 截断过长符号
      }
    }

    return null
  } catch (error) {
    // 记录调试日志，但返回 null 保持优雅
    if (error instanceof Error) {
      logForDebugging(
        `Symbol extraction failed for ${filePath}:${line}:${character}: ${error.message}`,
        { level: 'warn' }
      )
    }
    return null
  }
}
```

### 同步读取说明

```typescript
// eslint-disable-next-line custom-rules/no-sync-fs -- 
// called from sync React render (renderToolUseMessage)
const { buffer, bytesRead } = fs.readSync(absolutePath, {
  length: MAX_READ_BYTES,
})
```

**为什么使用同步读取？**
- 调用方 `renderToolUseMessage` 是同步 React 渲染函数
- React 渲染必须是同步的，不能使用 async/await
- 通过限制读取大小（64KB）控制阻塞时间

### 符号匹配详解

```typescript
const symbolPattern = /[\w$'!]+|[+\-*/%&|^~<>=]+/g
```

**模式分解：**
- `[\w$'!]+`：标识符类
  - `\w`：字母、数字、下划线
  - `$`：JavaScript 允许的标识符字符
  - `'`：Rust 生命周期前缀
  - `!`：Rust 宏后缀
- `|`：或
- `[+\-*/%&|^~<>=]+`：运算符类

**匹配优先级：**
- 先匹配标识符（更长更具体）
- 再匹配运算符

### 截断处理

```typescript
import { truncate } from '../../utils/format.js'

return truncate(symbol, 30)
```

避免超长符号（如压缩后的代码）影响 UI 显示。

---

## 关键代码路径与文件引用

### 内部依赖

```typescript
import { logForDebugging } from '../../utils/debug.js'
import { truncate } from '../../utils/format.js'
import { getFsImplementation } from '../../utils/fsOperations.js'
import { expandPath } from '../../utils/path.js'
```

### 常量定义

```typescript
const MAX_READ_BYTES = 64 * 1024  // 64KB
```

### 导出函数

| 函数名 | 参数 | 返回值 | 用途 |
|--------|------|--------|------|
| `getSymbolAtPosition` | `filePath, line, character` | `string \| null` | 提取指定位置的符号 |

### 被引用位置

| 文件 | 引用方式 | 用途 |
|------|----------|------|
| `UI.tsx` | `import { getSymbolAtPosition } from './symbolContext.js'` | `renderToolUseMessage` 中显示符号名称 |

### 在 UI.tsx 中的使用

```typescript
export function renderToolUseMessage(input: Partial<Input>, { verbose }) {
  if ((input.operation === 'goToDefinition' || 
       input.operation === 'findReferences' || 
       input.operation === 'hover' || 
       input.operation === 'goToImplementation') && 
      input.filePath && 
      input.line !== undefined && 
      input.character !== undefined) {
    
    // 注意：转换为 0-indexed
    const symbol = getSymbolAtPosition(
      input.filePath, 
      input.line - 1, 
      input.character - 1
    )
    
    if (symbol) {
      return `operation: "${input.operation}", symbol: "${symbol}", in: "${displayPath}"`
    }
  }
  // ...
}
```

---

## 依赖与外部交互

### 文件系统

```typescript
import { getFsImplementation } from '../../utils/fsOperations.js'
```

使用抽象的文件系统实现，支持：
- 真实文件系统（生产环境）
- 模拟文件系统（测试环境）

### 路径处理

```typescript
import { expandPath } from '../../utils/path.js'
```

将相对路径扩展为绝对路径。

### 工具函数

| 函数 | 来源 | 用途 |
|------|------|------|
| `truncate` | `../../utils/format.js` | 截断过长符号 |
| `logForDebugging` | `../../utils/debug.js` | 调试日志记录 |

---

## 风险、边界与改进建议

### 已知风险

1. **同步 I/O 阻塞**
   - 风险：`readSync` 会阻塞事件循环
   - 缓解：限制 64KB，典型读取 < 1ms
   - 极端情况：网络文件系统可能较慢

2. **64KB 限制**
   - 风险：目标位置在 64KB 之后时无法提取
   - 缓解：大多数代码文件 < 64KB，且 LSP 操作通常针对近期编辑位置
   - 回退：返回 null，UI 显示位置坐标

3. **编码问题**
   - 风险：非 UTF-8 编码文件可能乱码
   - 缓解：`toString('utf-8')`，异常被 try-catch 捕获

4. **截断行检测**
   - 逻辑：如果读取了完整 64KB 且目标行是最后一行，认为可能截断
   - 风险：恰好 64KB 的文件最后一行可能误判
   - 影响：低，只是放弃提取，不影响功能

### 边界情况

| 场景 | 行为 |
|------|------|
| 文件不存在 | 返回 null |
| 无读取权限 | 返回 null |
| 行号超出范围 | 返回 null |
| 列号超出范围 | 返回 null |
| 位置不在符号上 | 返回 null |
| 符号 > 30 字符 | 截断返回 |
| 目标行在 64KB 后 | 返回 null |
| 二进制文件 | 尝试读取，可能返回乱码或 null |

### 改进建议

1. **异步支持**
   - 当前：仅同步接口
   - 建议：添加异步版本供非 React 场景使用

   ```typescript
   export async function getSymbolAtPositionAsync(
     filePath: string,
     line: number,
     character: number
   ): Promise<string | null>
   ```

2. **更大文件支持**
   - 当前：固定 64KB
   - 建议：可配置或智能调整（如读取目标行附近区域）

3. **缓存机制**
   - 当前：每次调用都读取文件
   - 建议：对最近读取的文件进行缓存

   ```typescript
   // 简单 LRU 缓存
   const contentCache = new Map<string, { content: string; mtime: number }>()
   ```

4. **更多语言支持**
   - 当前：基本标识符 + Rust 特性
   - 建议：添加语言特定的模式

   ```typescript
   const patterns = {
     default: /[\w$]+/g,
     rust: /[\w$']+|[!]/g,
     haskell: /[\w']+/g,  // Haskell 允许 ' 在标识符中
     lisp: /[\w\-?]+/g,   // Lisp 允许 - 和 ?
   }
   ```

5. **语义分析**
   - 当前：纯文本正则匹配
   - 建议：考虑使用 Tree-sitter 等解析器获取更准确结果

6. **性能监控**
   - 建议：添加读取时间指标收集

   ```typescript
   const start = performance.now()
   const result = fs.readSync(...)
   logForDebugging(`Symbol extraction took ${performance.now() - start}ms`)
   ```

### 测试建议

- 测试各种编程语言的标识符提取
- 测试边界位置（行首、行尾）
- 测试大文件（> 64KB）
- 测试无效输入（负数、超出范围）
- 测试特殊字符和编码
- 测试权限问题
- 测试并发调用

### 代码注释说明

文件中的关键注释：

```typescript
/**
 * Note: This uses synchronous file I/O because it is called from
 * renderToolUseMessage (a synchronous React render function). The read is
 * wrapped in try/catch so ENOENT and other errors fall back gracefully.
 */
```

解释同步 I/O 的必要性和错误处理策略。

```typescript
// Read only the first 64KB instead of the whole file. Most LSP hover/goto
// targets are near recent edits; 64KB covers ~1000 lines of typical code.
// If the target line is past this window we fall back to null (the UI
// already handles that by showing `position: line:char`).
```

解释 64KB 限制的设计决策和回退机制。
