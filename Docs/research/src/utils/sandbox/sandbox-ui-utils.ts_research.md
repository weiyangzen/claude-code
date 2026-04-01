# sandbox-ui-utils.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位

`sandbox-ui-utils.ts` 是一个轻量级的 UI 工具函数模块，专门用于处理沙箱违规（sandbox violations）相关的文本清理和格式化。它位于沙箱适配层和 UI 组件之间，为错误消息的展示提供后处理功能。

### 1.2 主要职责

| 职责 | 说明 |
|------|------|
| **标签清理** | 从文本中移除 `<sandbox_violations>` XML 标签及其内容 |
| **错误消息格式化** | 为 UI 展示准备清理后的错误消息 |

### 1.3 使用场景

1. **错误消息展示**：当沙箱执行失败时，错误输出可能包含 `<sandbox_violations>` 标签，需要在展示给用户前清理
2. **日志记录**：清理后的消息更适合记录到日志或遥测系统
3. **测试输出**：测试断言中需要比较清理后的错误消息

---

## 2. 功能点目的

### 2.1 `removeSandboxViolationTags`

**目的**：移除文本中的沙箱违规标签，使错误消息更适合人类阅读。

**输入示例**：
```
Command failed with exit code 1
<sandbox_violations>
  File: /etc/passwd
  Operation: write
  Reason: Path outside allowed directories
</sandbox_violations>
Error details here
```

**输出示例**：
```
Command failed with exit code 1
Error details here
```

**实现细节**：
- 使用正则表达式 `/<sandbox_violations>[\s\S]*?<\/sandbox_violations>/g`
- `[\s\S]*?` 匹配任意字符（包括换行），非贪婪模式
- `g` 标志全局匹配，处理多个标签

---

## 3. 具体技术实现

### 3.1 代码实现

```typescript
/**
 * UI utilities for sandbox violations
 * These utilities are used for displaying sandbox-related information in the UI
 */

/**
 * Remove <sandbox_violations> tags from text
 * Used to clean up error messages for display purposes
 */
export function removeSandboxViolationTags(text: string): string {
  return text.replace(/<sandbox_violations>[\s\S]*?<\/sandbox_violations>/g, '')
}
```

### 3.2 正则表达式分析

| 模式 | 含义 |
|------|------|
| `<sandbox_violations>` | 匹配开始标签（字面量） |
| `[\s\S]*?` | 匹配任意字符（包括换行），非贪婪模式 |
| `<\/sandbox_violations>` | 匹配结束标签（`/` 需要转义） |
| `g` | 全局标志，匹配所有出现 |

**为什么选择 `[\s\S]` 而不是 `.`**：
- `.` 在默认模式下不匹配换行符
- `[\s\S]` 匹配任何空白和非空白字符，包括换行
- 适用于多行 XML 标签内容

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置

```
src/utils/sandbox/sandbox-ui-utils.ts (12 行)
```

### 4.2 调用方文件

| 文件 | 使用方式 |
|------|----------|
| `src/components/FallbackToolUseErrorMessage.tsx` | 清理工具使用错误消息 |
| `src/tools/BashTool/BashToolResultMessage.tsx` | 清理 Bash 工具结果消息 |

### 4.3 代码行号

```
行 1- 4: 文件头注释
行 6-12: removeSandboxViolationTags 函数实现
```

---

## 5. 依赖与外部交互

### 5.1 依赖分析

**零依赖**：本文件不导入任何内部或外部模块。

**原因**：
- 功能简单，仅需字符串操作
- 保持轻量，避免循环依赖风险
- 可被任何模块安全导入

### 5.2 被依赖情况

```typescript
// FallbackToolUseErrorMessage.tsx
import { removeSandboxViolationTags } from '../../utils/sandbox/sandbox-ui-utils.js'

// BashToolResultMessage.tsx
import { removeSandboxViolationTags } from '../../utils/sandbox/sandbox-ui-utils.js'
```

---

## 6. 风险、边界与改进建议

### 6.1 潜在风险

| 风险 | 说明 | 可能性 |
|------|------|--------|
| **正则表达式灾难性回溯** | 如果输入包含大量嵌套或损坏的标签，可能导致性能问题 | 低 |
| **标签嵌套** | 如果存在嵌套的 `<sandbox_violations>` 标签，非贪婪模式可能处理不正确 | 极低 |
| **HTML 实体编码** | 如果标签使用 HTML 实体编码（`&lt;`），正则无法匹配 | 低 |

### 6.2 边界情况

1. **空字符串输入**：返回空字符串（符合预期）
2. **无标签文本**：返回原字符串（符合预期）
3. **多个标签**：全部移除（`g` 标志）
4. **标签跨行**：正确处理（`[\s\S]` 包含换行）
5. **标签属性**：`<sandbox_violations attr="value">` 无法匹配（当前实现假设无属性）

### 6.3 改进建议

#### 6.3.1 增强健壮性

**建议**：处理带属性的标签

```typescript
// 当前
/<sandbox_violations>[\s\S]*?<\/sandbox_violations>/g

// 改进
/<sandbox_violations[^>]*>[\s\S]*?<\/sandbox_violations>/gi
```

- `[^>]*` 匹配任何属性
- `i` 标志处理大小写变化（虽然 XML 标签通常大小写敏感）

#### 6.3.2 性能优化

**建议**：对于大文本，考虑使用字符串扫描替代正则

```typescript
export function removeSandboxViolationTags(text: string): string {
  const startTag = '<sandbox_violations>'
  const endTag = '</sandbox_violations>'
  let result = text
  let startIndex = result.indexOf(startTag)
  
  while (startIndex !== -1) {
    const endIndex = result.indexOf(endTag, startIndex)
    if (endIndex === -1) break
    result = result.slice(0, startIndex) + result.slice(endIndex + endTag.length)
    startIndex = result.indexOf(startTag)
  }
  
  return result
}
```

**适用场景**：处理 MB 级别的日志输出时更可控。

#### 6.3.3 功能扩展

**建议**：添加更多沙箱相关的文本处理函数

```typescript
// 提取违规信息用于结构化日志
export function extractSandboxViolations(text: string): Array<{
  file?: string
  operation?: string
  reason?: string
}> 

// 检查是否包含违规标签
export function hasSandboxViolations(text: string): boolean

// 格式化违规信息为 Markdown
export function formatSandboxViolationsForDisplay(text: string): string
```

#### 6.3.4 单元测试

**建议**：添加边界情况测试

```typescript
describe('removeSandboxViolationTags', () => {
  it('removes single tag', () => { ... })
  it('removes multiple tags', () => { ... })
  it('handles empty string', () => { ... })
  it('handles text without tags', () => { ... })
  it('handles nested-like content', () => { ... })
  it('handles multiline content', () => { ... })
})
```

### 6.4 代码组织建议

当前文件非常简单（12 行），考虑以下选项：

1. **保持独立**：优点是不污染其他模块，缺点是文件数量多
2. **合并到 sandbox-adapter.ts**：作为导出的一部分，减少文件数量
3. **合并到通用 UI 工具**：创建 `src/utils/ui/textCleaning.ts` 包含所有文本清理函数

**推荐**：保持独立，因为：
- 职责单一清晰
- 避免 sandbox-adapter.ts 过于庞大
- 便于测试和 tree-shaking
