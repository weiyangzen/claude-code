# toolValidationConfig.ts 研究文档

## 场景与职责

`toolValidationConfig.ts` 定义了权限规则验证的工具特定配置。该模块集中管理：

1. **文件模式工具** - 接受 glob 模式的工具（如 `*.ts`, `src/**`）
2. **Bash 前缀工具** - 接受 bash 通配符和 `:*` 前缀语法的工具
3. **自定义验证规则** - 特定工具的额外验证逻辑

这是权限验证系统的配置层，被 `permissionValidation.ts` 使用。

## 功能点目的

### 1. 文件模式工具配置
- **工具列表**: Read, Write, Edit, Glob, NotebookRead, NotebookEdit
- **模式类型**: Glob 模式（`*.ts`, `src/**`, `**/*.test.ts`）
- **验证**: 检查 `:*` 语法误用（这是 Bash 特有的）

### 2. Bash 前缀工具配置
- **工具列表**: Bash
- **模式类型**: 
  - 通配符 `*`（可在任意位置）
  - 传统 `:*` 前缀语法（必须在末尾）
- **验证**: 检查 `:*` 位置，禁止单独使用 `:*`

### 3. 自定义验证规则
- **WebSearch**: 禁止通配符（`*`, `?`）
- **WebFetch**: 必须使用 `domain:` 前缀，禁止 URL 格式

## 具体技术实现

### 配置结构

```typescript
export type ToolValidationConfig = {
  filePatternTools: string[]
  bashPrefixTools: string[]
  customValidation: {
    [toolName: string]: (content: string) => {
      valid: boolean
      error?: string
      suggestion?: string
      examples?: string[]
    }
  }
}

export const TOOL_VALIDATION_CONFIG: ToolValidationConfig = {
  filePatternTools: [
    'Read',
    'Write', 
    'Edit',
    'Glob',
    'NotebookRead',
    'NotebookEdit',
  ],
  bashPrefixTools: ['Bash'],
  customValidation: {
    WebSearch: (content) => { /* ... */ },
    WebFetch: (content) => { /* ... */ },
  },
}
```

### 自定义验证详情

#### WebSearch 验证
```typescript
WebSearch: content => {
  if (content.includes('*') || content.includes('?')) {
    return {
      valid: false,
      error: 'WebSearch does not support wildcards',
      suggestion: 'Use exact search terms without * or ?',
      examples: ['WebSearch(claude ai)', 'WebSearch(typescript tutorial)'],
    }
  }
  return { valid: true }
}
```

#### WebFetch 验证
```typescript
WebFetch: content => {
  // 检查 URL 格式误用
  if (content.includes('://') || content.startsWith('http')) {
    return {
      valid: false,
      error: 'WebFetch permissions use domain format, not URLs',
      suggestion: 'Use "domain:hostname" format',
      examples: ['WebFetch(domain:example.com)'],
    }
  }

  // 检查 domain: 前缀
  if (!content.startsWith('domain:')) {
    return {
      valid: false,
      error: 'WebFetch permissions must use "domain:" prefix',
      suggestion: 'Use "domain:hostname" format',
      examples: ['WebFetch(domain:example.com)', 'WebFetch(domain:*.google.com)'],
    }
  }

  return { valid: true }
}
```

### 辅助函数

```typescript
// 检查工具是否使用文件模式
export function isFilePatternTool(toolName: string): boolean

// 检查工具是否使用 Bash 前缀模式
export function isBashPrefixTool(toolName: string): boolean

// 获取工具的自定义验证函数
export function getCustomValidation(toolName: string) 
  => (content: string) => { valid: boolean; error?: string; suggestion?: string; examples?: string[] } | undefined
```

### 关键代码路径

| 函数/常量 | 行号 | 说明 |
|-----------|------|------|
| `TOOL_VALIDATION_CONFIG` | 26-88 | 主配置对象 |
| `filePatternTools` | 28-36 | 文件模式工具列表 |
| `bashPrefixTools` | 38-39 | Bash 前缀工具列表 |
| `customValidation.WebSearch` | 42-53 | WebSearch 验证 |
| `customValidation.WebFetch` | 55-87 | WebFetch 验证 |
| `isFilePatternTool` | 91-93 | 文件模式检查辅助函数 |
| `isBashPrefixTool` | 96-98 | Bash 前缀检查辅助函数 |
| `getCustomValidation` | 101-103 | 自定义验证获取辅助函数 |

## 依赖与外部交互

### 导入依赖

无外部导入 - 这是纯配置模块。

### 被调用方

| 模块 | 路径 | 用途 |
|------|------|------|
| `permissionValidation.ts` | `./permissionValidation.js` | 验证权限规则时使用 |

### 导出 API

```typescript
export type ToolValidationConfig = {
  filePatternTools: string[]
  bashPrefixTools: string[]
  customValidation: {
    [toolName: string]: (content: string) => {
      valid: boolean
      error?: string
      suggestion?: string
      examples?: string[]
    }
  }
}

export const TOOL_VALIDATION_CONFIG: ToolValidationConfig

export function isFilePatternTool(toolName: string): boolean
export function isBashPrefixTool(toolName: string): boolean
export function getCustomValidation(toolName: string): Function | undefined
```

## 风险、边界与改进建议

### 风险点

1. **工具名硬编码**: 工具名在配置中硬编码，如果工具重命名需要同步更新。

2. **验证逻辑分散**: 自定义验证逻辑分散在配置对象中，复杂验证可能难以维护。

3. **扩展性**: 添加新工具类型需要修改此文件，可能不符合开闭原则。

### 边界情况

| 场景 | 行为 |
|------|------|
| 未知工具名 | `isFilePatternTool` 和 `isBashPrefixTool` 返回 false；`getCustomValidation` 返回 undefined |
| 工具名大小写 | 匹配是大小写敏感的 |
| 自定义验证返回 undefined | 调用方应视为无自定义验证 |

### 改进建议

1. **配置外部化**: 考虑将验证配置外部化为 JSON 或 YAML，便于非开发者修改
2. **插件扩展**: 考虑支持插件注册自定义验证规则
3. **工具元数据**: 考虑从工具定义中提取验证配置，而非单独维护
4. **正则验证**: 对于简单的模式匹配，考虑使用正则表达式配置
5. **文档生成**: 基于此配置自动生成权限规则文档

## 文件引用

- **本文件**: `src/utils/settings/toolValidationConfig.ts`
- **相关文件**:
  - `src/utils/settings/permissionValidation.ts` - 使用此配置进行验证

## 配置扩展示例

如需添加新工具的验证，按以下模式扩展：

```typescript
// 1. 添加到相应工具列表（如果需要）
filePatternTools: ['Read', 'Write', 'NewTool'],

// 2. 或添加自定义验证
customValidation: {
  NewTool: content => {
    if (/* 无效条件 */) {
      return {
        valid: false,
        error: '错误消息',
        suggestion: '修复建议',
        examples: ['NewTool(valid)', 'NewTool(also-valid)'],
      }
    }
    return { valid: true }
  },
}
```
