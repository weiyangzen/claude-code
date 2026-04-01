# use-terminal-title.ts 深入研究

## 场景与职责

`useTerminalTitle` 是 Ink 终端 UI 框架中用于设置终端标签页/窗口标题的 Hook。它提供了一个声明式的方式来更新终端标题，自动处理 ANSI 转义序列剥离和跨平台兼容性。

## 功能点目的

### 1. 声明式标题设置
- 通过 Hook 声明式设置终端标题
- 自动处理标题变化
- 支持条件性设置（传入 `null` 跳过）

### 2. 跨平台兼容
- Windows 使用 `process.title`
- 其他平台使用 OSC 0 序列
- 自动检测平台并选择合适的方法

### 3. ANSI 处理
- 自动剥离 ANSI 转义序列
- 调用者无需关心终端编码
- 避免标题显示乱码

## 具体技术实现

### 接口定义

```typescript
export function useTerminalTitle(title: string | null): void
```

- `title`: 要设置的标题字符串，或 `null` 表示不设置

### 核心实现逻辑

```typescript
import { useContext, useEffect } from 'react'
import stripAnsi from 'strip-ansi'
import { OSC, osc } from '../termio/osc.js'
import { TerminalWriteContext } from '../useTerminalNotification.js'

export function useTerminalTitle(title: string | null): void {
  const writeRaw = useContext(TerminalWriteContext)

  useEffect(() => {
    if (title === null || !writeRaw) return

    const clean = stripAnsi(title)

    if (process.platform === 'win32') {
      process.title = clean
    } else {
      writeRaw(osc(OSC.SET_TITLE_AND_ICON, clean))
    }
  }, [title, writeRaw])
}
```

### 关键技术点

1. **ANSI 剥离**：
   ```typescript
   const clean = stripAnsi(title)
   ```
   使用 `strip-ansi` 库移除所有 ANSI 转义序列

2. **平台检测**：
   ```typescript
   if (process.platform === 'win32') {
     process.title = clean
   } else {
     writeRaw(osc(OSC.SET_TITLE_AND_ICON, clean))
   }
   ```
   - Windows: 经典 conhost 不支持 OSC，使用 `process.title`
   - 其他: 使用 OSC 0（设置标题+图标）

3. **条件执行**：
   - `title === null` 时跳过
   - `writeRaw` 未提供时跳过

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/termio/osc.ts` | 提供 OSC 序列生成 |
| `src/ink/useTerminalNotification.ts` | 提供 TerminalWriteContext |
| `strip-ansi` | 第三方库，用于剥离 ANSI 序列 |

### OSC 相关定义

```typescript
// src/ink/termio/osc.ts

export const OSC = {
  SET_TITLE_AND_ICON: 0,
  SET_ICON: 1,
  SET_TITLE: 2,
  // ...
}

export function osc(...parts: (string | number)[]): string {
  const terminator = env.terminal === 'kitty' ? ST : BEL
  return `${OSC_PREFIX}${parts.join(SEP)}${terminator}`
}
```

### OSC 0 序列格式

```
ESC ] 0 ; <title> BEL   // 设置标题和图标
ESC ] 2 ; <title> BEL   // 仅设置标题
```

### 使用示例

```typescript
import { useTerminalTitle } from 'ink'

const App = ({ currentFile }) => {
  // 动态设置标题
  useTerminalTitle(currentFile ? `Editing: ${currentFile}` : 'My App')
  
  return <Text>Content</Text>
}

// 条件性设置
const OptionalTitle = ({ showTitle, title }) => {
  useTerminalTitle(showTitle ? title : null)
  
  return <Text>Content</Text>
}

// 带 ANSI 的标题（自动剥离）
const StyledTitle = () => {
  useTerminalTitle('\x1b[32mGreen Title\x1b[0m')  // 输出: Green Title
  
  return <Text>Content</Text>
}
```

## 依赖与外部交互

### 与 TerminalWriteContext 的交互

```typescript
const writeRaw = useContext(TerminalWriteContext)
```

- `writeRaw` 用于向终端写入原始 OSC 序列
- 由 Ink 主组件通过 Provider 提供
- Windows 平台不使用（使用 `process.title` 代替）

### 与终端的交互

1. **序列发送**：
   - 非 Windows: 通过 `writeRaw` 发送 OSC 0 序列
   - 序列包含标题文本和终止符（BEL 或 ST）

2. **终端支持**：
   - 大多数现代终端支持 OSC 0
   - 不支持的终端静默丢弃序列

3. **清理**：
   - 组件卸载时不自动恢复原标题
   - 应用退出时可发送空标题恢复

### 与 strip-ansi 的交互

```typescript
import stripAnsi from 'strip-ansi'

// 示例
stripAnsi('\u001B[4mUnicorn\u001B[0m')  // => 'Unicorn'
stripAnsi('\u001B]8;;https://example.com\u0007Click\u001B]8;;\u0007')  // => 'Click'
```

确保标题中不包含可能干扰终端解析的转义序列。

## 风险、边界与改进建议

### 潜在风险

1. **标题长度限制**：
   - 某些终端对标题长度有限制
   - 超长标题可能被截断或导致问题

2. **特殊字符**：
   - 标题中的 BEL (\x07) 可能提前终止序列
   - 需要额外的转义处理

3. **并发设置**：
   - 多个组件同时设置标题可能产生竞争
   - 最后的设置生效

### 边界情况

1. **空字符串标题**：
   - 设置空标题可能清除标题或显示默认标题
   - 取决于终端实现

2. **null 值**：
   - Hook 成为无操作
   - 终端标题保持不变

3. **writeRaw 未提供**：
   - 静默返回
   - 不抛出错误

4. **Windows 限制**：
   - `process.title` 可能有长度限制
   - Unicode 支持取决于系统区域设置

### 改进建议

1. **添加标题历史**：
   ```typescript
   const { setTitle, restorePrevious } = useTerminalTitle({ history: true })
   ```

2. **支持标题模板**：
   ```typescript
   useTerminalTitle('App - {filename}', { 
     vars: { filename: 'test.txt' }
   })
   ```

3. **添加长度限制**：
   ```typescript
   useTerminalTitle(longTitle, { maxLength: 100 })
   ```

4. **支持标题前缀**：
   ```typescript
   useTerminalTitle('Subpage', { prefix: 'MyApp - ' })
   // 最终标题: MyApp - Subpage
   ```

5. **添加清理恢复**：
   ```typescript
   useEffect(() => {
     const originalTitle = process.title
     return () => {
       process.title = originalTitle  // 恢复原标题
     }
   }, [])
   ```

6. **支持动态平台检测**：
   ```typescript
   useTerminalTitle(title, { 
     method: process.env.WT_SESSION ? 'osc' : 'auto' 
   })
   ```

### 测试建议

1. 测试普通标题设置
2. 测试带 ANSI 的标题（验证剥离）
3. 测试 null 值（验证无操作）
4. 测试快速标题切换
5. 测试超长标题
6. 测试特殊字符（BEL、ESC 等）
7. 测试多组件同时设置
