# UserResourceUpdateMessage.tsx 研究文档

## 场景与职责

`UserResourceUpdateMessage` 是一个 React 组件，用于渲染 MCP（Model Context Protocol）资源更新和轮询更新的消息。该组件解析资源更新和轮询更新的 XML 标签，以统一的格式显示服务器资源和工具状态的变更。

**核心职责：**
- 解析 `<mcp-resource-update>` 和 `<mcp-polling-update>` XML 标签
- 提取服务器名称、目标（URI 或工具名）和原因
- 格式化并显示更新信息（带刷新箭头图标）
- 对 file:// URI 进行简化显示（只显示文件名）

## 功能点目的

1. **资源更新解析**：提取 MCP 资源更新的服务器、URI 和原因
2. **轮询更新解析**：提取 MCP 轮询更新的服务器、工具名和原因
3. **URI 格式化**：简化 file:// URI 显示，只显示文件名
4. **视觉反馈**：使用刷新箭头图标和颜色区分更新消息

## 具体技术实现

### 关键流程

```
输入: addMargin (boolean), param (TextBlockParam)
  ↓
解析 text 中的更新
  ↓
匹配 <mcp-resource-update server="..." uri="...">（可选 <reason>）
匹配 <mcp-polling-update type="tool" server="..." tool="...">（可选 <reason>）
  ↓
如果无更新：返回 null
  ↓
对每个更新渲染一行：
  刷新箭头 + 服务器名 + 目标（格式化后的 URI 或工具名）+ 原因（如果有）
```

### 数据结构

**Props 接口：**
```typescript
{
  addMargin: boolean      // 是否在顶部添加边距
  param: TextBlockParam   // 包含更新标签的文本块
}

// TextBlockParam 结构
{
  text: string  // 包含 mcp-resource-update 或 mcp-polling-update 标签
}
```

**ParsedUpdate 类型：**
```typescript
type ParsedUpdate = {
  kind: 'resource' | 'polling'  // 更新类型
  server: string                // 服务器名称
  target: string                // URI（资源）或工具名（轮询）
  reason?: string               // 更新原因（可选）
}
```

### 关键代码路径

**文件位置：** `src/components/messages/UserResourceUpdateMessage.tsx`

**正则表达式定义：**
```typescript
// 匹配 <mcp-resource-update server="..." uri="...">
const resourceRegex = 
  /<mcp-resource-update\s+server="([^"]+)"\s+uri="([^"]+)"[^>]*>(?:[\s\S]*?<reason>([^<]+)<\/reason>)?/g

// 匹配 <mcp-polling-update type="..." server="..." tool="...">
const pollingRegex = 
  /<mcp-polling-update\s+type="([^"]+)"\s+server="([^"]+)"\s+tool="([^"]+)"[^>]*>(?:[\s\S]*?<reason>([^<]+)<\/reason>)?/g
```

**解析函数：**
```typescript
function parseUpdates(text: string): ParsedUpdate[] {
  const updates: ParsedUpdate[] = []

  // 解析资源更新
  let match
  while ((match = resourceRegex.exec(text)) !== null) {
    updates.push({
      kind: 'resource',
      server: match[1] ?? '',
      target: match[2] ?? '',
      reason: match[3]
    })
  }

  // 解析轮询更新
  while ((match = pollingRegex.exec(text)) !== null) {
    updates.push({
      kind: 'polling',
      server: match[2] ?? '',
      target: match[3] ?? '',
      reason: match[4]
    })
  }

  return updates
}
```

**URI 格式化函数：**
```typescript
function formatUri(uri: string): string {
  // 对于 file:// URI，只显示文件名
  if (uri.startsWith('file://')) {
    const path = uri.slice(7)
    const parts = path.split('/')
    return parts[parts.length - 1] || path
  }
  // 其他 URI 超过 40 字符截断
  if (uri.length > 40) {
    return uri.slice(0, 39) + '…'
  }
  return uri
}
```

**渲染结构：**
```tsx
<Box flexDirection="column" marginTop={addMargin ? 1 : 0}>
  {updates.map((update, i) => (
    <Box key={i}>
      <Text>
        <Text color="success">{REFRESH_ARROW}</Text>{' '}
        <Text dimColor>{update.server}:</Text>{' '}
        <Text color="suggestion">
          {update.kind === 'resource' ? formatUri(update.target) : update.target}
        </Text>
        {update.reason && <Text dimColor> · {update.reason}</Text>}
      </Text>
    </Box>
  ))}
</Box>
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| TextBlockParam | '@anthropic-ai/sdk/resources/index.mjs' | SDK 类型 |
| React | 'react' | UI 框架 |
| REFRESH_ARROW | '../../constants/figures.js' | 刷新箭头图标 (↻) |
| Box, Text | '../../ink.js' | 终端 UI 组件 |

### 相关常量

**figures.js：**
```typescript
export const REFRESH_ARROW = '\u21bb' // ↻ - used for resource update indicator
```

### MCP 相关

**MCP（Model Context Protocol）：**
- 资源更新：当 MCP 服务器上的资源发生变化时通知客户端
- 轮询更新：定期轮询工具状态的变化
- 服务器名称：标识提供资源/工具的 MCP 服务器

### 主题颜色

- `success` - 刷新箭头颜色
- `suggestion` - 目标（URI/工具名）颜色
- `dimColor` - 服务器名和原因的颜色

## 风险、边界与改进建议

### 潜在风险

1. **正则表达式贪婪匹配**：`[\s\S]*?` 在大文本上可能有性能问题
2. **XML 解析局限**：正则无法处理嵌套标签或复杂 XML 结构
3. **URI 格式假设**：`formatUri` 假设 file:// 后紧跟路径，可能有 edge cases

### 边界情况

1. **无匹配**：如果 text 不包含更新标签，返回 null
2. **空属性**：server、uri、tool 为空字符串时正常显示
3. **超长 URI**：非 file:// URI 超过 40 字符会被截断
4. **多个更新**：支持一个消息中包含多个更新
5. **无原因**：reason 未提供时不显示 `· reason` 部分

### 改进建议

1. **XML 解析器**：考虑使用轻量级 XML 解析器替代正则
2. **URI 验证**：添加 URI 格式验证
3. **点击交互**：支持点击服务器名或 URI 查看详情
4. **更新分组**：相同服务器的更新可以分组显示
5. **时间戳**：显示更新发生的时间
6. **状态图标**：根据更新类型显示不同图标（新增/修改/删除）
7. **折叠长列表**：多个更新时可以折叠显示

### 测试建议

1. 各种 XML 格式变体测试
2. 多个更新同时存在测试
3. 特殊 URI 格式测试（file://, http://, 自定义协议）
4. 超长 URI 截断测试
5. 空属性边界测试
6. 大文本性能测试
