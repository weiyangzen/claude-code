# MCPToolDetailView.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`MCPToolDetailView.tsx` 是 Claude Code CLI 中 **MCP 工具详细信息展示组件**。当用户在 `/mcp` 流程中浏览某服务器的工具列表并选择特定工具时，此组件负责展示该工具的完整元数据，包括名称、描述、参数模式等。

### 1.2 使用场景
- 用户在 `/mcp` → 选择服务器 → "View tools" → 选择特定工具
- 展示工具的完整信息供用户了解工具功能
- 显示工具的安全属性（read-only/destructive/open-world）
- 展示工具的 JSON Schema 参数定义

### 1.3 架构角色
```
MCPSettings (主控制器)
    ↓
MCPToolListView (工具列表)
    ↓ (用户选择工具)
MCPToolDetailView (本组件)
    ├── 标题区域 (显示名称 + 安全标签)
    ├── 工具名称 (原始名称)
    ├── 完整名称 (带 MCP 前缀)
    ├── 描述 (异步加载)
    └── 参数列表 (JSON Schema 解析)
```

---

## 2. 功能点目的

### 2.1 工具元数据展示
| 字段 | 来源 | 说明 |
|-----|------|------|
| `displayName` | `tool.userFacingName` 或 `getMcpDisplayName` | 用户友好的显示名称 |
| `toolName` | `getMcpDisplayName(tool.name, server.name)` | 不带前缀的工具名 |
| `fullName` | `tool.name` | 完整 MCP 工具名 (mcp__server__tool) |
| `description` | `tool.description()` | 异步加载的工具描述 |

### 2.2 安全属性标签
| 属性 | 条件 | 显示 |
|-----|------|------|
| `[read-only]` | `tool.isReadOnly?.({}) ?? false` | 绿色标签 |
| `[destructive]` | `tool.isDestructive?.({}) ?? false` | 红色标签 |
| `[open-world]` | `tool.isOpenWorld?.({}) ?? false` | 灰色标签 |

### 2.3 参数展示
从 `tool.inputJSONSchema` 解析并展示：
- 参数名称
- 是否必需 (检查 `required` 数组)
- 参数类型
- 参数描述

### 2.4 异步描述加载
```
组件挂载
    ↓
useEffect 触发
    ↓
tool.description() 调用
    ↓
setToolDescription(desc) 更新状态
```

---

## 3. 具体技术实现

### 3.1 核心数据结构

#### 3.1.1 Props 接口
```typescript
type Props = {
  tool: Tool                    // 工具定义对象
  server: ServerInfo            // 所属服务器信息
  onBack: () => void            // 返回回调
}
```

#### 3.1.2 Tool 类型 (简化)
```typescript
type Tool = {
  name: string                  // 完整 MCP 工具名
  userFacingName?: (input: unknown) => string  // 用户友好名称生成器
  description: (
    input: unknown,
    options: {
      isNonInteractiveSession: boolean
      toolPermissionContext: ToolPermissionContext
      tools: Tool[]
    }
  ) => Promise<string>         // 异步描述获取
  inputJSONSchema?: {          // JSON Schema 参数定义
    type: 'object'
    properties?: Record<string, unknown>
    required?: string[]
  }
  isReadOnly?: (input: unknown) => boolean
  isDestructive?: (input: unknown) => boolean
  isOpenWorld?: (input: unknown) => boolean
}
```

### 3.2 关键流程

#### 3.2.1 显示名称计算
```typescript
// 1. 获取工具名 (去掉 MCP 前缀)
const toolName = getMcpDisplayName(tool.name, server.name)

// 2. 获取完整显示名
const fullDisplayName = tool.userFacingName 
  ? tool.userFacingName({}) 
  : toolName

// 3. 提取纯工具显示名 (去掉服务器前缀和 "(MCP)" 后缀)
const displayName = extractMcpToolDisplayName(fullDisplayName)
```

#### 3.2.2 安全属性检测
```typescript
const isReadOnly = tool.isReadOnly?.({}) ?? false
const isDestructive = tool.isDestructive?.({}) ?? false
const isOpenWorld = tool.isOpenWorld?.({}) ?? false
```

#### 3.2.3 异步描述加载
```typescript
React.useEffect(() => {
  const loadDescription = async () => {
    try {
      const desc = await tool.description({}, {
        isNonInteractiveSession: false,
        toolPermissionContext: {
          mode: "default",
          additionalWorkingDirectories: new Map(),
          alwaysAllowRules: {},
          alwaysDenyRules: {},
          alwaysAskRules: {},
          isBypassPermissionsModeAvailable: false
        },
        tools: []
      })
      setToolDescription(desc)
    } catch {
      setToolDescription("Failed to load description")
    }
  }
  loadDescription()
}, [tool])
```

#### 3.2.4 参数渲染
```typescript
{tool.inputJSONSchema && tool.inputJSONSchema.properties && 
 Object.keys(tool.inputJSONSchema.properties).length > 0 && (
  <Box flexDirection="column" marginTop={1}>
    <Text bold>Parameters:</Text>
    <Box marginLeft={2} flexDirection="column">
      {Object.entries(tool.inputJSONSchema.properties).map(([key, value]) => {
        const required = tool.inputJSONSchema?.required as string[] | undefined
        const isRequired = required?.includes(key)
        return (
          <Text key={key}>
            • {key}
            {isRequired && <Text dimColor> (required)</Text>}:{" "}
            <Text dimColor>
              {typeof value === "object" && value && "type" in value 
                ? String(value.type) 
                : "unknown"}
            </Text>
            {typeof value === "object" && value && "description" in value && (
              <Text dimColor> - {String(value.description)}</Text>
            )}
          </Text>
        )
      })}
    </Box>
  </Box>
)}
```

---

## 4. 关键代码路径与文件引用

### 4.1 组件依赖图
```
MCPToolDetailView.tsx
├── Dialog (design-system)          # 对话框容器
├── ConfigurableShortcutHint        # 可配置快捷键提示
├── services/mcp/
│   └── mcpStringUtils.ts           # extractMcpToolDisplayName, getMcpDisplayName
├── Tool.ts                         # Tool 类型定义
└── types.js (同目录)               # ServerInfo 类型
```

### 4.2 关键函数引用
| 函数 | 来源 | 用途 |
|-----|------|------|
| `getMcpDisplayName` | `services/mcp/mcpStringUtils.ts` | 从完整 MCP 名中提取工具名 |
| `extractMcpToolDisplayName` | `services/mcp/mcpStringUtils.ts` | 从 userFacingName 中提取纯工具名 |

### 4.3 字符串工具函数详解

#### 4.3.1 getMcpDisplayName
```typescript
export function getMcpDisplayName(fullName: string, serverName: string): string {
  const prefix = `mcp__${normalizeNameForMCP(serverName)}__`
  return fullName.replace(prefix, '')
}
// 示例: "mcp__github__create_issue" + "github" → "create_issue"
```

#### 4.3.2 extractMcpToolDisplayName
```typescript
export function extractMcpToolDisplayName(userFacingName: string): string {
  // 1. 移除 "(MCP)" 后缀
  let withoutSuffix = userFacingName.replace(/\s*\(MCP\)\s*$/, '').trim()
  
  // 2. 移除服务器前缀 ("server - ")
  const dashIndex = withoutSuffix.indexOf(' - ')
  if (dashIndex !== -1) {
    return withoutSuffix.substring(dashIndex + 3).trim()
  }
  
  return withoutSuffix
}
// 示例: "github - Create Issue (MCP)" → "Create Issue"
```

---

## 5. 依赖与外部交互

### 5.1 导入依赖
```typescript
// React 核心 (使用 React Compiler)
import { c as _c } from "react/compiler-runtime"
import React from 'react'

// UI 组件
import { Box, Text } from '../../ink.js'
import { ConfigurableShortcutHint } from '../ConfigurableShortcutHint.js'
import { Dialog } from '../design-system/Dialog.js'

// MCP 字符串工具
import { extractMcpToolDisplayName, getMcpDisplayName } from '../../services/mcp/mcpStringUtils.js'

// 类型
import type { Tool } from '../../Tool.js'
import type { ServerInfo } from './types.js'
```

### 5.2 Tool 类型依赖
`Tool` 类型定义在 `src/Tool.ts`，包含以下关键字段：
- `name`: 工具唯一标识
- `userFacingName`: 用户友好名称生成函数
- `description`: 异步描述获取函数
- `inputJSONSchema`: JSON Schema 参数定义
- `isReadOnly`/`isDestructive`/`isOpenWorld`: 安全属性检查函数

### 5.3 权限上下文
描述加载时使用固定的权限上下文：
```typescript
{
  mode: "default",
  additionalWorkingDirectories: new Map(),
  alwaysAllowRules: {},
  alwaysDenyRules: {},
  alwaysAskRules: {},
  isBypassPermissionsModeAvailable: false
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 硬编码权限上下文
```typescript
toolPermissionContext: {
  mode: "default" as const,
  additionalWorkingDirectories: new Map(),
  // ...
}
```
- **风险**: 权限上下文硬编码，如果工具描述依赖于特定权限设置，可能显示不准确的描述
- **建议**: 考虑从父组件传入或从全局状态获取

#### 6.1.2 描述加载错误处理
```typescript
catch {
  setToolDescription("Failed to load description")
}
```
- **风险**: 错误被静默捕获，只显示通用错误信息，不记录具体错误原因
- **建议**: 添加日志记录或显示更详细的错误信息

#### 6.1.3 空输入调用
```typescript
const fullDisplayName = tool.userFacingName ? tool.userFacingName({}) : toolName
const isReadOnly = tool.isReadOnly?.({}) ?? false
```
- **风险**: 使用空对象 `{}` 调用可能不符合某些工具的预期输入格式
- **建议**: 如果可能，传入实际的工具参数或更合适的默认值

### 6.2 边界情况

#### 6.2.1 缺少 JSON Schema
如果 `tool.inputJSONSchema` 未定义或 `properties` 为空，参数区域不渲染：
```typescript
tool.inputJSONSchema && tool.inputJSONSchema.properties && 
Object.keys(tool.inputJSONSchema.properties).length > 0
```

#### 6.2.2 复杂的嵌套参数
当前实现只展示一级参数属性，不处理嵌套的 `object` 或 `array` 类型参数。

#### 6.2.3 长描述截断
工具描述可能很长，当前没有截断或折叠机制，可能导致 UI 过长。

### 6.3 改进建议

#### 6.3.1 参数类型美化
当前直接显示 JSON Schema 的 `type` 字段，建议添加类型映射：
```typescript
const typeLabels: Record<string, string> = {
  'string': 'Text',
  'number': 'Number',
  'integer': 'Integer',
  'boolean': 'Yes/No',
  'array': 'List',
  'object': 'Object'
}
```

#### 6.3.2 嵌套参数支持
对于 `object` 类型的参数，建议递归渲染嵌套属性：
```typescript
const renderProperty = (key: string, value: unknown, depth: number = 0) => {
  // 递归渲染嵌套属性
}
```

#### 6.3.3 描述折叠
对于长描述，建议添加 "展开/折叠" 功能：
```typescript
const [isDescriptionExpanded, setIsDescriptionExpanded] = useState(false)
const displayDescription = isDescriptionExpanded 
  ? toolDescription 
  : truncate(toolDescription, 200)
```

#### 6.3.4 参数示例
如果 JSON Schema 包含 `examples`，建议展示示例值：
```typescript
{"description" in value && value.examples && (
  <Text dimColor>Examples: {value.examples.join(', ')}</Text>
)}
```

### 6.4 性能考虑

#### 6.4.1 React Compiler 优化
组件使用 React Compiler (`_c` 函数) 进行自动记忆化，减少不必要的重渲染。

#### 6.4.2 描述加载优化
每次组件挂载都会重新加载描述，建议：
- 在父组件缓存描述
- 或使用 React Query/SWR 进行缓存

### 6.5 可访问性建议
- 为安全标签添加语义化标记
- 为参数列表添加键盘导航支持
- 考虑添加参数搜索/过滤功能（当参数较多时）
