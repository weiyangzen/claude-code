# hyperlink.ts 研究文档

## 场景与职责

`hyperlink.ts` 是 Claude Code CLI 的**终端超链接生成工具**，实现了 OSC 8 转义序列标准，允许在支持的终端中创建可点击的超链接。该模块为 CLI 输出提供富文本交互能力，使用户能够直接在终端中点击链接跳转到相关资源。

### 核心使用场景

1. **Markdown 链接渲染**：在 `markdown.ts` 中将 URL 转换为终端可点击链接
2. **Shell 输出行**：在 `OutputLine.tsx` 中为命令输出中的 URL 添加点击支持
3. **MCP 工具 UI**：在 `MCPTool/UI.tsx` 中为工具相关的链接提供交互

### OSC 8 标准简介

OSC 8（Operating System Command 8）是终端超链接的开放标准，格式为：
```
\e]8;;URL\e\\TEXT\e]8;;\e\\
```

或使用 BEL（\x07）作为终止符（更广泛支持）：
```
\x1b]8;;URL\x07TEXT\x1b]8;;\x07
```

---

## 功能点目的

### 1. 超链接创建 (`createHyperlink`)

**设计目标**：在支持 OSC 8 的终端中创建可点击链接，在不支持的终端中优雅降级为纯文本。

**功能特性**：
- **终端能力检测**：通过 `supportsHyperlinks()` 检测终端是否支持 OSC 8
- **显示文本定制**：支持自定义链接显示文本（而非直接显示 URL）
- **ANSI 颜色支持**：使用 `chalk.blue` 为链接添加蓝色样式
- **测试覆盖**：支持通过 `options.supportsHyperlinks` 覆盖检测逻辑

### 2. 终端支持检测

**检测逻辑**（位于 `src/ink/supports-hyperlinks.ts`）：
- 使用 `supports-hyperlinks` 库进行基础检测
- 扩展支持 Ghostty、Hyper、Kitty、Alacritty、iTerm2 等终端
- 检测 `TERM_PROGRAM`、`LC_TERMINAL`、`TERM` 环境变量

---

## 具体技术实现

### OSC 8 转义序列常量

```typescript
// 使用 \x07 (BEL) 作为终止符，更广泛支持
export const OSC8_START = '\x1b]8;;'  // ESC ] 8 ; ;
export const OSC8_END = '\x07'        // BEL 字符
```

### 超链接创建函数

```typescript
export function createHyperlink(
  url: string,
  content?: string,        // 可选的显示文本
  options?: HyperlinkOptions  // 测试覆盖选项
): string {
  const hasSupport = options?.supportsHyperlinks ?? supportsHyperlinks()
  
  if (!hasSupport) {
    return url  // 降级：返回纯 URL
  }

  const displayText = content ?? url
  const coloredText = chalk.blue(displayText)
  
  // 格式：OSC8_START + URL + OSC8_END + 显示文本 + OSC8_START + OSC8_END
  return `${OSC8_START}${url}${OSC8_END}${coloredText}${OSC8_START}${OSC8_END}`
}
```

### 类型定义

```typescript
type HyperlinkOptions = {
  supportsHyperlinks?: boolean  // 用于测试覆盖终端检测
}
```

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/hyperlink.ts`
- **导出常量**：
  - `OSC8_START` - OSC 8 序列起始标记
  - `OSC8_END` - OSC 8 序列结束标记（BEL 字符）
- **导出函数**：
  - `createHyperlink(url, content?, options?)` - 创建超链接

### 调用方分布

| 文件路径 | 使用函数 | 使用场景 |
|---------|---------|---------|
| `src/utils/markdown.ts` | `createHyperlink` | Markdown 链接渲染，行 10 |
| `src/components/shell/OutputLine.tsx` | `createHyperlink` | Shell 输出行链接 |
| `src/tools/MCPTool/UI.tsx` | `createHyperlink` | MCP 工具 UI 链接 |

### 调用示例（markdown.ts）

```typescript
import { createHyperlink } from './hyperlink.js'

// 在 Markdown 渲染中使用
function formatLink(token: Tokens.Link): string {
  const text = token.tokens?.map(t => formatToken(t, theme)).join('') ?? token.text
  return createHyperlink(token.href, text)
}
```

### 依赖导入

```typescript
import chalk from 'chalk'                                    // ANSI 颜色
import { supportsHyperlinks } from '../ink/supports-hyperlinks.js'  // 终端能力检测
```

---

## 依赖与外部交互

### 外部依赖

| 包名 | 用途 |
|------|------|
| `chalk` | 为链接文本添加蓝色 ANSI 颜色 |

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `ink/supports-hyperlinks.ts` | `supportsHyperlinks` | 检测终端是否支持 OSC 8 |

### 终端能力检测详情

**`src/ink/supports-hyperlinks.ts` 实现**：

```typescript
import supportsHyperlinksLib from 'supports-hyperlinks'

export const ADDITIONAL_HYPERLINK_TERMINALS = [
  'ghostty', 'Hyper', 'kitty', 'alacritty', 'iTerm.app', 'iTerm2'
]

export function supportsHyperlinks(options?: SupportsHyperlinksOptions): boolean {
  const stdoutSupported = options?.stdoutSupported ?? supportsHyperlinksLib.stdout
  if (stdoutSupported) return true

  const env = options?.env ?? process.env
  
  // 检查 TERM_PROGRAM
  const termProgram = env['TERM_PROGRAM']
  if (termProgram && ADDITIONAL_HYPERLINK_TERMINALS.includes(termProgram)) {
    return true
  }
  
  // 检查 LC_TERMINAL（tmux 中保留）
  const lcTerminal = env['LC_TERMINAL']
  if (lcTerminal && ADDITIONAL_HYPERLINK_TERMINALS.includes(lcTerminal)) {
    return true
  }
  
  // 检查 TERM（Kitty 设置 xterm-kitty）
  const term = env['TERM']
  if (term?.includes('kitty')) return true
  
  return false
}
```

---

## 风险、边界与改进建议

### 已知限制

1. **RGB 颜色不支持**
   - 代码注释明确说明：`wrap-ansi` 不保留 RGB 颜色（如主题色）
   - 仅支持基本 ANSI 颜色（如 `chalk.blue`）
   - 如果使用 RGB 颜色，换行后样式可能丢失

2. **终端兼容性**
   - 不支持 OSC 8 的终端会降级为纯 URL 显示
   - 但某些终端可能部分支持，导致显示异常
   - 无运行时检测机制验证链接是否真正可点击

3. **URL 长度限制**
   - 某些终端对转义序列长度有限制
   - 超长 URL 可能被截断或导致显示问题

### 边界情况

| 场景 | 处理 |
|------|------|
| 终端不支持 OSC 8 | 返回纯 URL，无颜色 |
| content 未提供 | 使用 URL 作为显示文本 |
| URL 为空字符串 | 生成空链接（潜在问题） |
| 测试覆盖 | 通过 `options.supportsHyperlinks` 强制启用/禁用 |

### 改进建议

1. **URL 验证和清理**
   ```typescript
   export function createHyperlink(url: string, content?: string): string {
     // 验证 URL 格式
     if (!isValidUrl(url)) {
       return content ?? url  // 无效 URL 降级为纯文本
     }
     // ...
   }
   ```

2. **支持更多链接属性**
   OSC 8 标准支持 `id` 参数用于链接分组：
   ```typescript
   export function createHyperlink(
     url: string,
     content?: string,
     id?: string  // 链接分组 ID
   ): string {
     const params = id ? `id=${id};` : ''
     return `${OSC8_START}${params}${url}${OSC8_END}${coloredText}${OSC8_START}${OSC8_END}`
   }
   ```

3. **添加链接点击统计**
   ```typescript
   // 通过特定参数追踪链接点击（需终端支持）
   const trackingUrl = addTrackingParams(url, { source: 'claude-cli' })
   ```

4. **支持文件路径链接**
   ```typescript
   // 自动将文件路径转换为 file:// URL
   export function createFileLink(filePath: string, content?: string): string {
     const url = path.isAbsolute(filePath) 
       ? `file://${filePath}` 
       : `file://${process.cwd()}/${filePath}`
     return createHyperlink(url, content ?? filePath)
   }
   ```

5. **单元测试覆盖**
   - 终端支持/不支持的降级测试
   - URL 编码处理测试
   - 特殊字符转义测试
   - 颜色样式验证测试

6. **性能优化**
   - `supportsHyperlinks()` 结果被多次调用时重复计算
   - 考虑添加记忆化（memoization）缓存结果
   - 注意环境变量可能在运行时变化（罕见）
