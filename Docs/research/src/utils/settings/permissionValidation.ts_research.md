# permissionValidation.ts 研究文档

## 场景与职责

`permissionValidation.ts` 负责验证权限规则（permission rules）的格式和内容。权限规则是 Claude Code 中控制工具使用的核心机制，格式为 `ToolName` 或 `ToolName(pattern)`。该模块确保用户输入的权限规则符合语法要求，并提供友好的错误提示。

## 功能点目的

### 1. 权限规则验证 (`validatePermissionRule`)
- **语法验证**: 括号匹配、转义字符处理
- **工具特定验证**: 不同工具有不同的模式要求
  - **Bash 工具**: 支持 `:*` 前缀语法和 `*` 通配符
  - **文件工具**: 支持 glob 模式（`*.ts`, `src/**`）
  - **MCP 工具**: 不支持括号内的模式
  - **WebSearch**: 不支持通配符
  - **WebFetch**: 必须使用 `domain:` 前缀

### 2. Zod Schema 集成 (`PermissionRuleSchema`)
- **用途**: 在设置文件验证时集成 Zod schema
- **行为**: 验证失败时添加详细的错误信息、建议和示例

## 具体技术实现

### 关键辅助函数

```typescript
// 检查字符是否被转义（前面有奇数个反斜杠）
function isEscaped(str: string, index: number): boolean

// 计算字符串中未转义的字符数量
function countUnescapedChar(str: string, char: string): number

// 检查是否有未转义的空括号 "()"
function hasUnescapedEmptyParens(str: string): boolean
```

### 验证流程

```
validatePermissionRule(rule)
  ├── 空规则检查
  ├── 括号匹配检查（仅未转义的括号）
  ├── 空括号检查
  ├── 解析规则 (permissionRuleValueFromString)
  ├── MCP 验证
  │   └── 检查是否有括号内容（MCP 不支持）
  ├── 工具名验证
  │   └── 必须以大写字母开头
  ├── 自定义验证（如果配置）
  ├── Bash 特定验证
  │   ├── :* 必须在末尾
  │   ├── :* 不能单独使用
  │   └── 通配符位置灵活
  └── 文件工具验证
      ├── 禁止 :* 语法
      └── 通配符位置警告
```

### 工具特定验证配置

验证逻辑依赖 `toolValidationConfig.ts` 中的配置：

```typescript
// 文件模式工具
filePatternTools: ['Read', 'Write', 'Edit', 'Glob', 'NotebookRead', 'NotebookEdit']

// Bash 前缀工具
bashPrefixTools: ['Bash']

// 自定义验证
customValidation: {
  WebSearch: (content) => { /* 禁止通配符 */ },
  WebFetch: (content) => { /* 必须 domain: 前缀 */ }
}
```

### 关键代码路径

| 函数 | 行号 | 说明 |
|------|------|------|
| `isEscaped` | 15-23 | 检查字符是否被转义 |
| `countUnescapedChar` | 29-37 | 计算未转义字符数 |
| `hasUnescapedEmptyParens` | 43-53 | 检查空括号 |
| `validatePermissionRule` | 58-239 | 主验证函数 |
| `PermissionRuleSchema` | 244-262 | Zod schema |

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `z` | `zod/v4` | Schema 验证 |
| `mcpInfoFromString` | `../../services/mcp/mcpStringUtils.js` | MCP 工具名解析 |
| `lazySchema` | `../lazySchema.js` | 延迟加载 schema |
| `permissionRuleValueFromString` | `../permissions/permissionRuleParser.js` | 解析权限规则 |
| `capitalize` | `../stringUtils.js` | 字符串首字母大写 |
| `getCustomValidation` | `./toolValidationConfig.js` | 获取自定义验证 |
| `isBashPrefixTool` | `./toolValidationConfig.js` | 检查 Bash 工具 |
| `isFilePatternTool` | `./toolValidationConfig.js` | 检查文件工具 |

### 被调用方

| 模块 | 路径 | 用途 |
|------|------|------|
| `validation.ts` | `./validation.js` | 设置文件验证时过滤无效规则 |
| `types.ts` | `./types.js` | PermissionRuleSchema 用于 PermissionsSchema |
| `skills/bundled/updateConfig.ts` | `../../skills/bundled/updateConfig.js` | 权限规则更新 |

### 导出 API

```typescript
export function validatePermissionRule(rule: string): {
  valid: boolean
  error?: string
  suggestion?: string
  examples?: string[]
}

export const PermissionRuleSchema: () => z.ZodSchema<string>
```

## 风险、边界与改进建议

### 风险点

1. **转义逻辑复杂**: 括号转义逻辑（`isEscaped`, `countUnescapedChar`）容易出错，特别是边界情况如 `\\(`。

2. **MCP 工具名解析限制**: 注释中提到 `mcp__server__tool` 格式如果服务器名包含 `__` 会解析错误。

3. **通配符验证宽松**: 文件工具的通配符位置检查是启发式的，可能误报或漏报。

4. **Bash 引号不验证**: 注释明确说明不验证引号平衡，因为 bash 引号规则复杂。

### 边界情况

| 场景 | 行为 |
|------|------|
| 空规则 | 返回错误 |
| 括号不匹配 | 返回错误 |
| 空括号 `()` | 返回错误，建议移除括号或添加内容 |
| 转义的括号 `\(\)` | 视为内容的一部分 |
| 工具名小写 | 返回错误，建议首字母大写 |
| MCP 规则带括号 | 返回错误，MCP 不支持模式 |
| Bash `:*` 不在末尾 | 返回错误 |
| WebFetch 无 domain: 前缀 | 返回错误 |

### 改进建议

1. **测试覆盖**: 确保所有转义场景都有单元测试
2. **MCP 解析增强**: 考虑改进 MCP 工具名解析，处理服务器名包含 `__` 的情况
3. **错误消息国际化**: 当前错误消息是硬编码英文
4. **性能优化**: 如果验证成为瓶颈，考虑缓存常见规则的结果
5. **文档**: 添加更多关于权限规则语法的文档和示例

## 文件引用

- **本文件**: `src/utils/settings/permissionValidation.ts`
- **相关文件**:
  - `src/utils/settings/toolValidationConfig.ts` - 工具验证配置
  - `src/utils/settings/validation.ts` - 设置验证
  - `src/utils/permissions/permissionRuleParser.ts` - 权限规则解析
  - `src/services/mcp/mcpStringUtils.ts` - MCP 工具名处理
