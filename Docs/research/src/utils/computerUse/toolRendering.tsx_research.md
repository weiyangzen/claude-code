# toolRendering.tsx 研究文档

## 场景与职责

本文件实现 **Computer Use MCP 工具的渲染覆盖**，定义工具在 UI 中的显示方式。它提供用户友好的名称和参数格式化，使 Computer Use 操作对用户可见且可理解。

核心职责：
1. **用户友好名称**：为每个工具提供可读名称
2. **工具使用消息渲染**：格式化工具调用参数
3. **工具结果消息渲染**：简洁显示操作结果

## 功能点目的

### 1. 用户友好名称 (`userFacingName`)

格式：`Computer Use[{toolName}]`

示例：
- `Computer Use[screenshot]`
- `Computer Use[left_click]`
- `Computer Use[type]`

### 2. 工具使用消息渲染 (`renderToolUseMessage`)

根据工具类型格式化参数：

| 工具 | 显示格式 |
|------|----------|
| screenshot, left_mouse_down, etc. | '' (空，隐藏参数) |
| left_click, right_click, etc. | `(x, y)` |
| left_click_drag | `(x1, y1) → (x2, y2)` 或 `to (x, y)` |
| type | `"truncated text"` (最多40字符) |
| key, hold_key | `key name` |
| scroll | `direction ×amount at (x, y)` |
| zoom | `[x, y, w, h]` |
| wait | `duration s` |
| write_clipboard | `"truncated text"` |
| open_application | `bundle_id` |
| request_access | `App1, App2, ...` |
| computer_batch | `N actions` |

### 3. 工具结果消息渲染 (`renderToolResultMessage`)

非详细模式下的简洁摘要：

| 工具 | 摘要 |
|------|------|
| screenshot, zoom | Captured |
| request_access | Access updated |
| left_click, right_click, etc. | Clicked |
| type | Typed |
| key, hold_key | Pressed |
| scroll | Scrolled |
| left_click_drag | Dragged |
| open_application | Opened |

## 具体技术实现

### 类型定义

```typescript
type CuToolInput = Record<string, unknown> & {
  coordinate?: [number, number]
  start_coordinate?: [number, number]
  text?: string
  apps?: Array<{ displayName?: string }>
  region?: [number, number, number, number]
  direction?: string
  amount?: number
  duration?: number
}
```

### 坐标格式化

```typescript
function fmtCoord(c: [number, number] | undefined): string {
  return c ? `(${c[0]}, ${c[1]})` : ''
}
```

### 结果摘要映射

```typescript
const RESULT_SUMMARY: Readonly<Partial<Record<string, string>>> = {
  screenshot: 'Captured',
  zoom: 'Captured',
  request_access: 'Access updated',
  left_click: 'Clicked',
  // ... 其他工具
}
```

### 渲染函数实现

```typescript
export function getComputerUseMCPRenderingOverrides(toolName: string) {
  return {
    userFacingName() {
      return `Computer Use[${toolName}]`
    },
    
    renderToolUseMessage(input: CuToolInput) {
      switch (toolName) {
        case 'screenshot':
        case 'left_mouse_down':
          // ... 返回 '' 隐藏参数
        case 'left_click':
          return fmtCoord(input.coordinate)
        case 'type':
          return typeof input.text === 'string' 
            ? `"${truncateToWidth(input.text, 40)}"` 
            : ''
        // ... 其他工具
      }
    },
    
    renderToolResultMessage(output, _progress, { verbose }) {
      if (verbose || typeof output !== 'object' || output === null) 
        return null
      
      const summary = RESULT_SUMMARY[toolName]
      if (!summary) return null
      
      return (
        <MessageResponse height={1}>
          <Text dimColor>{summary}</Text>
        </MessageResponse>
      )
    }
  }
}
```

## 关键代码路径与文件引用

### 本文件导出
- `getComputerUseMCPRenderingOverrides(toolName)` - 获取工具的渲染覆盖

### 调用方
- `src/utils/computerUse/wrapper.tsx:284` - 合并到工具覆盖对象

### 依赖文件
- `src/components/MessageResponse.js` - `MessageResponse` 组件
- `src/ink.js` - `Text` 组件
- `src/utils/format.js` - `truncateToWidth`
- `src/utils/mcpValidation.js` - `MCPToolResult` 类型

### 外部包
- `react` - React 类型和 JSX

## 依赖与外部交互

### React/ink 集成
- 使用 React 组件定义渲染输出
- `MessageResponse` 和 `Text` 来自 ink（React for CLI）
- 返回 React 节点供 UI 渲染

### 工具系统集成
- 通过 `wrapper.tsx` 合并到 MCP 工具对象
- 在 `client.ts` 中传播到工具覆盖
- 遵循与 Chrome MCP 工具覆盖相同的模式

### 参数格式化
- 使用 `truncateToWidth` 限制文本长度
- 坐标格式化为 `(x, y)` 形式
- 数组参数格式化为 `[a, b, c, d]` 形式

## 风险、边界与改进建议

### 已知风险

1. **类型安全**：
   - `CuToolInput` 使用宽松类型
   - 运行时类型检查确保健壮性

2. **文本截断**：
   - 硬编码 40 字符限制
   - 可能在某些终端宽度下不合适

3. **React 依赖**：
   - 需要 React 运行时
   - 在非交互模式可能不可用

### 边界情况

1. **未知工具**：
   - switch 语句有 default 返回 ''
   - 新工具默认只显示名称

2. **无效参数**：
   - 每个格式化都有类型检查
   - 无效输入返回空字符串

3. **详细模式**：
   - `verbose` 为 true 时返回 null
   - 显示完整原始输出

4. **空参数**：
   - 坐标 undefined 返回 ''
   - 文本非字符串返回 ''

### 改进建议

1. **动态截断**：
   - 根据终端宽度动态调整截断长度
   - 使用 ink 的维度查询

2. **国际化**：
   - 摘要文本硬编码英文
   - 考虑添加 i18n 支持

3. ** richer 显示**：
   - 为更多工具添加自定义渲染
   - 考虑添加图标或颜色编码

4. **可配置性**：
   - 允许用户自定义显示格式
   - 添加紧凑/详细模式切换

5. **测试覆盖**：
   - 添加单元测试验证格式化
   - 测试边界输入

6. **文档完善**：
   - 添加 JSDoc 说明每个工具的显示格式
   - 提供视觉示例

7. **性能优化**：
   - 考虑缓存渲染结果
   - 避免重复计算
