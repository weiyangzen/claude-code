# promptCategory.ts 深度研究文档

## 场景与职责

`promptCategory.ts` 是 Claude Code 中负责**提示分类（Prompt Category）**的工具模块。它根据 Agent 类型和输出样式设置确定查询来源（Query Source），用于分析系统中跟踪不同的使用模式。

### 核心职责
1. **Agent 查询来源**：根据 Agent 类型生成查询来源标识
2. **REPL 查询来源**：根据输出样式设置生成查询来源标识
3. **分析追踪**：为分析系统提供细粒度的使用模式追踪

### 使用场景
- 分析系统中跟踪 Agent 使用模式
- 输出样式使用统计
- 自定义 Agent vs 内置 Agent 使用分析

---

## 功能点目的

### 1. Agent 查询来源

```typescript
export function getQuerySourceForAgent(
  agentType: string | undefined,
  isBuiltInAgent: boolean,
): QuerySource
```

**分类规则**：

| Agent 类型 | 是否内置 | 查询来源 |
|-----------|---------|---------|
| 有类型名 | 是 | `agent:builtin:{agentType}` |
| 无类型名 | 是 | `agent:default` |
| 任意 | 否 | `agent:custom` |

### 2. REPL 查询来源

```typescript
export function getQuerySourceForREPL(): QuerySource
```

**分类规则**：

| 输出样式 | 查询来源 |
|---------|---------|
| 默认样式 | `repl_main_thread` |
| 内置样式 | `repl_main_thread:outputStyle:{style}` |
| 自定义样式 | `repl_main_thread:outputStyle:custom` |

---

## 具体技术实现

### Agent 查询来源实现

```typescript
export function getQuerySourceForAgent(
  agentType: string | undefined,
  isBuiltInAgent: boolean,
): QuerySource {
  if (isBuiltInAgent) {
    // TODO: avoid this cast
    return agentType
      ? (`agent:builtin:${agentType}` as QuerySource)
      : 'agent:default'
  } else {
    return 'agent:custom'
  }
}
```

**类型转换说明**：代码注释中提到需要避免类型转换，说明类型定义可能需要完善。

### REPL 查询来源实现

```typescript
export function getQuerySourceForREPL(): QuerySource {
  const settings = getSettings_DEPRECATED()
  const style = settings?.outputStyle ?? DEFAULT_OUTPUT_STYLE_NAME

  if (style === DEFAULT_OUTPUT_STYLE_NAME) {
    return 'repl_main_thread'
  }

  // 所有 OUTPUT_STYLE_CONFIG 中的样式都是内置的
  const isBuiltIn = style in OUTPUT_STYLE_CONFIG
  return isBuiltIn
    ? (`repl_main_thread:outputStyle:${style}` as QuerySource)
    : 'repl_main_thread:outputStyle:custom'
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/constants/querySource.js` | `QuerySource` 类型 |
| `../constants/outputStyles.js` | 输出样式配置 |
| `./settings/settings.js` | 用户设置获取 |

### 外部调用方

| 调用方 | 用途 |
|-------|------|
| `src/screens/REPL.tsx` | REPL 查询来源 |
| `src/tools/AgentTool/AgentTool.tsx` | Agent 查询来源 |
| `src/tools/AgentTool/resumeAgent.ts` | Agent 恢复查询来源 |

---

## 依赖与外部交互

### 运行时依赖

1. **内部模块**：
   - `settings/settings.js` - 获取用户设置
   - `constants/outputStyles.js` - 输出样式配置

### 类型定义

```typescript
// 来自 src/constants/querySource.js
type QuerySource = 
  | 'repl_main_thread'
  | `repl_main_thread:outputStyle:${string}`
  | 'agent:default'
  | `agent:builtin:${string}`
  | 'agent:custom'
  // ... 其他来源
```

---

## 风险、边界与改进建议

### 已知风险

1. **类型转换**
   - 使用 `as QuerySource` 类型断言
   - 编译时无法验证字符串格式

2. **硬编码字符串**
   - 查询来源前缀硬编码
   - 变更时需要修改多处

3. **设置依赖**
   - 依赖 `getSettings_DEPRECATED()`
   - 使用已弃用的 API

4. **扩展性限制**
   - 新增查询来源类型需要修改类型定义

### 边界条件

| 场景 | 处理 |
|------|------|
| agentType 为空字符串 | 视为 undefined，返回 `agent:default` |
| 输出样式未设置 | 使用默认样式 |
| 未知内置样式 | 视为自定义样式 |

### 改进建议

1. **类型安全**
   ```typescript
   // 使用模板字面量类型
   type BuiltinAgentSource = `agent:builtin:${string}`
   type OutputStyleSource = `repl_main_thread:outputStyle:${string}`
   
   // 验证函数
   function isValidQuerySource(source: string): source is QuerySource
   ```

2. **常量定义**
   ```typescript
   export const QUERY_SOURCE_PREFIXES = {
     AGENT_BUILTIN: 'agent:builtin',
     AGENT_CUSTOM: 'agent:custom',
     REPL: 'repl_main_thread',
     REPL_STYLE: 'repl_main_thread:outputStyle',
   } as const
   ```

3. **迁移到新设置 API**
   ```typescript
   // 替换 getSettings_DEPRECATED
   import { getSettings } from './settings/newSettings.js'
   ```

4. **更多分类维度**
   ```typescript
   export function getQuerySourceForTool(toolName: string): QuerySource
   export function getQuerySourceForCommand(command: string): QuerySource
   ```

5. **分析元数据**
   ```typescript
   export interface QuerySourceMetadata {
     source: QuerySource
     timestamp: Date
     sessionId: string
     userId?: string
   }
   ```

### 维护注意事项

1. **类型定义同步**：更新 `QuerySource` 类型定义
2. **分析系统协调**：确保分析系统支持新的查询来源
3. **文档更新**：更新分析追踪文档
4. **测试覆盖**：测试各种 Agent 类型和输出样式组合

### 使用示例

```typescript
import { getQuerySourceForAgent, getQuerySourceForREPL } from './promptCategory.js'

// Agent 查询来源
const builtinSource = getQuerySourceForAgent('code-reviewer', true)
console.log(builtinSource)  // 'agent:builtin:code-reviewer'

const defaultSource = getQuerySourceForAgent(undefined, true)
console.log(defaultSource)  // 'agent:default'

const customSource = getQuerySourceForAgent('my-agent', false)
console.log(customSource)   // 'agent:custom'

// REPL 查询来源
const replSource = getQuerySourceForREPL()
console.log(replSource)  // 'repl_main_thread' 或 'repl_main_thread:outputStyle:...'
```
