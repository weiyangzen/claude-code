# xml.ts 研究文档

## 场景与职责

`xml.ts` 提供 XML/HTML 特殊字符的转义功能，用于安全地将不受信任的字符串插入到 XML/HTML 内容中。这是防止 XSS（跨站脚本攻击）和 XML 注入的基础安全措施。

**核心使用场景：**
- 将进程输出（stdout）插入到 XML/HTML 标签内容中
- 将用户输入插入到 XML/HTML 文档中
- 将外部数据插入到 XML/HTML 模板中

## 功能点目的

### 1. XML 内容转义 (`escapeXml`)
- **目的**：转义用于元素文本内容的特殊字符
- **转义字符**：`&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`
- **使用场景**：`<tag>${content}</tag>`

### 2. XML 属性转义 (`escapeXmlAttr`)
- **目的**：转义用于属性值的特殊字符
- **额外转义**：`"` → `&quot;`, `'` → `&apos;`
- **使用场景**：`<tag attr="${value}">` 或 `<tag attr='${value}'>`

## 具体技术实现

### 核心代码

```typescript
/**
 * Escape XML/HTML special characters for safe interpolation into element
 * text content (between tags). Use when untrusted strings (process stdout,
 * user input, external data) go inside `<tag>${here}</tag>`.
 */
export function escapeXml(s: string): string {
  return s.replace(/&/g, '&amp;')
          .replace(/</g, '&lt;')
          .replace(/>/g, '&gt;')
}

/**
 * Escape for interpolation into a double- or single-quoted attribute value:
 * `<tag attr="${here}">`. Escapes quotes in addition to `& < >`.
 */
export function escapeXmlAttr(s: string): string {
  return escapeXml(s)
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')
}
```

### 转义映射表

| 原始字符 | 转义序列 | 转义原因 |
|----------|----------|----------|
| `&` | `&amp;` | 避免被解析为实体引用开始 |
| `<` | `&lt;` | 避免被解析为标签开始 |
| `>` | `&gt;` | 避免被解析为标签结束 |
| `"` | `&quot;` | 避免在双引号属性中结束 |
| `'` | `&apos;` | 避免在单引号属性中结束 |

### 使用示例

```typescript
// 元素内容转义
const userInput = '<script>alert("xss")</script>'
const safeContent = escapeXml(userInput)
// 结果: '&lt;script&gt;alert("xss")&lt;/script&gt;'
const html = `<div>${safeContent}</div>`
// 结果: '<div>&lt;script&gt;alert("xss")&lt;/script&gt;</div>'

// 属性值转义
const attrValue = 'value" onclick="evil()'
const safeAttr = escapeXmlAttr(attrValue)
// 结果: 'value&quot; onclick=&quot;evil()'
const html = `<button data-value="${safeAttr}">Click</button>`
```

## 关键代码路径与文件引用

### 导出函数
- `src/utils/xml.ts:6` - `escapeXml(s: string): string`
- `src/utils/xml.ts:14` - `escapeXmlAttr(s: string): string`

### 依赖
无依赖，纯字符串操作。

## 依赖与外部交互

### 外部依赖
无外部依赖。

### 内部依赖
无内部依赖。

## 风险、边界与改进建议

### 已知风险

1. **不完全的 XSS 防护**
   - 仅转义基本 XML/HTML 特殊字符
   - 不处理 JavaScript 上下文（如 `<script>${content}</script>`）
   - 不处理 CSS 上下文（如 `<style>${content}</style>`）
   - 不处理 URL 上下文（如 `<a href="${url}">`）

2. **字符编码问题**
   - 假设输入是有效的 Unicode 字符串
   - 不处理非 UTF-8 编码的输入

3. **性能考虑**
   - 使用正则表达式进行替换，对于大字符串可能较慢
   - 多次 `replace` 调用会创建多个中间字符串

### 边界情况

1. **空字符串**
   - 返回空字符串（正确行为）

2. **无特殊字符**
   - 返回原字符串（正确行为）

3. **已转义字符**
   - 会再次转义（`&amp;` → `&amp;amp;`）
   - 这是设计行为，适用于单次转义场景
   - 如果需要避免双重转义，调用方需要自行处理

4. **控制字符**
   - 不处理控制字符（如 `\x00` - `\x1f`）
   - XML 1.0 不允许某些控制字符

5. **Unicode 和 HTML 实体**
   - 不转换 Unicode 字符到 HTML 实体
   - 例如 `©` 保持为 `©`，不会变成 `&copy;`

### 改进建议

1. **添加 HTML 特定转义**
   ```typescript
   export function escapeHtml(s: string): string {
     return escapeXml(s)
       .replace(/\//g, '&#x2F;')  // 可选：转义斜杠（针对 </script>）
   }
   ```

2. **性能优化**
   ```typescript
   export function escapeXml(s: string): string {
     let result = ''
     for (let i = 0; i < s.length; i++) {
       const ch = s[i]
       switch (ch) {
         case '&': result += '&amp;'; break
         case '<': result += '&lt;'; break
         case '>': result += '&gt;'; break
         default: result += ch
       }
     }
     return result
   }
   ```

3. **上下文感知转义**
   ```typescript
   export type EscapeContext = 'text' | 'attr' | 'url' | 'js' | 'css'
   export function escapeForContext(s: string, context: EscapeContext): string {
     // 根据上下文选择适当的转义策略
   }
   ```

4. **验证输入**
   - 添加对 `null` 和 `undefined` 的处理
   - 添加对非字符串输入的类型检查

5. **添加测试**
   - 边界情况测试（空字符串、特殊字符组合）
   - XSS 攻击向量测试
   - 性能基准测试

6. **文档完善**
   - 明确说明此函数不适用于 JavaScript/CSS/URL 上下文
   - 提供安全使用指南
