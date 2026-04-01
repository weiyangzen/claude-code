# permissionRuleParser.ts 研究文档

## 场景与职责

Permission Rule Parser（权限规则解析器）是 Claude Code 权限系统的核心组件，负责解析和序列化权限规则字符串。权限规则用于定义哪些工具操作被允许、拒绝或需要询问，格式为 `"ToolName"` 或 `"ToolName(content)"`。

该模块处理：
1. **规则解析**: 将字符串规则解析为结构化对象
2. **规则序列化**: 将结构化对象序列化为字符串
3. **内容转义**: 处理规则内容中的特殊字符（括号、反斜杠）
4. **遗留工具名映射**: 将旧工具名映射到新的规范名称

## 功能点目的

### 1. 规则解析 (`permissionRuleValueFromString`)

将权限规则字符串解析为 `PermissionRuleValue` 对象：

**支持的格式**:
- `"Bash"` → `{ toolName: 'Bash' }`
- `"Bash(npm install)"` → `{ toolName: 'Bash', ruleContent: 'npm install' }`
- `"Bash(python -c \"print\\(1\\)\")"` → `{ toolName: 'Bash', ruleContent: 'python -c "print(1)"' }`

**处理逻辑**:
1. 查找第一个未转义的开括号 `(`
2. 查找最后一个未转义的闭括号 `)`
3. 验证括号位置和匹配
4. 提取工具名和内容
5. 对内容进行反转义
6. 规范化遗留工具名

### 2. 规则序列化 (`permissionRuleValueToString`)

将 `PermissionRuleValue` 对象序列化为字符串：

**处理逻辑**:
1. 如果没有 `ruleContent`，只返回工具名
2. 对内容进行转义（反斜杠 → `\\`, `(` → `\(`, `)` → `\)`）
3. 组合为 `"ToolName(content)"` 格式

### 3. 内容转义 (`escapeRuleContent`)

转义规则内容中的特殊字符：

**转义顺序**（重要）:
1. 先转义反斜杠: `\` → `\\`
2. 再转义开括号: `(` → `\(`
3. 最后转义闭括号: `)` → `\)`

**示例**:
```typescript
escapeRuleContent('psycopg2.connect()')
// => 'psycopg2.connect\\(\\)'

escapeRuleContent('echo "test\\nvalue"')
// => 'echo "test\\\\nvalue"'
```

### 4. 内容反转义 (`unescapeRuleContent`)

反转义规则内容：

**反序**（与转义相反）:
1. 先反转义括号: `\(` → `(`, `\)` → `)`
2. 最后反转义反斜杠: `\\` → `\`

### 5. 遗留工具名映射

处理工具重命名后的向后兼容性：

**映射表** (`LEGACY_TOOL_NAME_ALIASES`):
- `Task` → `Agent` (AgentTool)
- `KillShell` → `TaskStop` (TaskStopTool)
- `AgentOutputTool` → `TaskOutput` (TaskOutputTool)
- `BashOutputTool` → `TaskOutput` (TaskOutputTool)
- `Brief` → `BRIEF_TOOL_NAME` (Kairos 功能，条件编译)

**双向查找**:
- `normalizeLegacyToolName(name)`: 旧名 → 新名
- `getLegacyToolNames(canonicalName)`: 新名 → 所有旧名

## 具体技术实现

### 关键流程

```
permissionRuleValueFromString(ruleString)
├── 查找第一个未转义的 '('
│   └── findFirstUnescapedChar(ruleString, '(')
│       └── 计算前面反斜杠数量，偶数=未转义
├── 未找到 → 规范化工具名返回
├── 查找最后一个未转义的 ')'
│   └── findLastUnescapedChar(ruleString, ')')
├── 验证括号位置和匹配
├── 提取 toolName 和 rawContent
├── 处理特殊情况
│   ├── 空内容 → 视为工具级规则
│   ├── 通配符 '*' → 视为工具级规则
│   └── 空工具名 → 视为畸形，返回原字符串
├── 反转义内容
│   └── unescapeRuleContent(rawContent)
└── 规范化工具名返回

permissionRuleValueToString(ruleValue)
├── 无 ruleContent → 返回 toolName
├── 转义内容
│   └── escapeRuleContent(ruleContent)
└── 返回 `${toolName}(${escapedContent})`
```

### 数据结构

```typescript
// 权限规则值（来自 types/permissions.ts）
type PermissionRuleValue = {
  toolName: string
  ruleContent?: string
}

// 遗留工具名映射表
const LEGACY_TOOL_NAME_ALIASES: Record<string, string> = {
  Task: AGENT_TOOL_NAME,           // 'Agent'
  KillShell: TASK_STOP_TOOL_NAME,  // 'TaskStop'
  AgentOutputTool: TASK_OUTPUT_TOOL_NAME,  // 'TaskOutput'
  BashOutputTool: TASK_OUTPUT_TOOL_NAME,   // 'TaskOutput'
  Brief: BRIEF_TOOL_NAME,          // 条件编译
}
```

### 转义算法详解

#### 查找未转义字符

```typescript
function findFirstUnescapedChar(str: string, char: string): number {
  for (let i = 0; i < str.length; i++) {
    if (str[i] === char) {
      // 计算前面反斜杠数量
      let backslashCount = 0
      let j = i - 1
      while (j >= 0 && str[j] === '\\') {
        backslashCount++
        j--
      }
      // 偶数=未转义，奇数=已转义
      if (backslashCount % 2 === 0) {
        return i
      }
    }
  }
  return -1
}
```

#### 转义顺序的重要性

```typescript
// 正确顺序：先转义反斜杠
'psycopg2.connect()' 
  → 'psycopg2.connect\\(\\)'  // 正确

// 错误顺序：如果先转义括号
'psycopg2.connect()'
  → 'psycopg2.connect\(\)'
  → 'psycopg2.connect\\(\\)'  // 结果相同，但逻辑混乱

// 更复杂的例子
'echo "test\\nvalue"'
  // 正确：先转义反斜杠
  → 'echo "test\\\\nvalue"'  // \\n 被保留为字面量
  
  // 错误：如果先转义括号（虽然这里没有括号）
  // 但如果有：'cmd \\(args\\)'
  // 先转义括号 → 'cmd \\\(args\\\)'
  // 再转义反斜杠 → 'cmd \\\\\\(args\\\\\)'
  // 过度转义！
```

## 关键代码路径与文件引用

### 内部依赖

- `bun:bundle` 的 `feature` 函数：条件编译支持
- 工具名常量（条件导入）:
  - `src/tools/AgentTool/constants.js`: `AGENT_TOOL_NAME`
  - `src/tools/TaskOutputTool/constants.js`: `TASK_OUTPUT_TOOL_NAME`
  - `src/tools/TaskStopTool/prompt.js`: `TASK_STOP_TOOL_NAME`
  - `src/tools/BriefTool/prompt.js`: `BRIEF_TOOL_NAME` (Kairos 功能)

### 外部调用方

- `src/tools/BashTool/bashPermissions.ts`: Bash 权限检查
- `src/utils/settings/permissionValidation.ts`: 设置验证
- `src/utils/doctorContextWarnings.ts`: 诊断警告
- `src/tools/AgentTool/agentToolUtils.ts`: Agent 工具工具函数
- `src/utils/hooks.ts`: 钩子系统
- `src/utils/sandbox/sandbox-adapter.ts`: 沙箱适配器
- `src/utils/permissions/permissionSetup.ts`: 权限设置
- `src/utils/permissions/permissionsLoader.ts`: 权限加载器
- `src/utils/permissions/permissions.ts`: 核心权限逻辑
- `src/utils/permissions/PermissionUpdate.ts`: 权限更新
- `src/components/permissions/hooks.ts`: 权限钩子
- `src/components/permissions/rules/PermissionRuleList.tsx`: 规则列表 UI
- `src/components/permissions/rules/PermissionRuleInput.tsx`: 规则输入 UI
- `src/components/permissions/rules/AddPermissionRules.tsx`: 添加规则 UI
- `src/components/permissions/PermissionRuleExplanation.tsx`: 规则解释 UI
- `src/components/permissions/PermissionDecisionDebugInfo.tsx`: 调试信息 UI

### 类型定义

- `src/utils/permissions/PermissionRule.ts`: `PermissionRuleValue` 类型（注意：实际定义在 `types/permissions.ts`）
- `src/types/permissions.ts`: 权威类型定义位置

## 依赖与外部交互

### 运行时依赖

```typescript
import { feature } from 'bun:bundle'
import { AGENT_TOOL_NAME } from '../../tools/AgentTool/constants.js'
import { TASK_OUTPUT_TOOL_NAME } from '../../tools/TaskOutputTool/constants.js'
import { TASK_STOP_TOOL_NAME } from '../../tools/TaskStopTool/prompt.js'
import type { PermissionRuleValue } from './PermissionRule.js'

// 条件导入（Kairos 功能）
const BRIEF_TOOL_NAME: string | null =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? require('../../tools/BriefTool/prompt.js').BRIEF_TOOL_NAME
    : null
```

### 条件编译说明

```typescript
// 使用条件编译避免在正式构建中包含 Ant-only 工具名
const LEGACY_TOOL_NAME_ALIASES: Record<string, string> = {
  // ... 标准映射
  ...((feature('KAIROS') || feature('KAIROS_BRIEF')) && BRIEF_TOOL_NAME
    ? { Brief: BRIEF_TOOL_NAME }
    : {}),
}
```

这种设计确保：
- 外部构建不包含 Ant-only 工具名字符串
- 静态导入总是被打包，所以使用条件 `require()`

### 解析失败的处理

```typescript
// 畸形规则的处理策略：返回原字符串作为工具名
if (closeParenIndex === -1 || closeParenIndex <= openParenIndex) {
  return { toolName: normalizeLegacyToolName(ruleString) }
}

if (closeParenIndex !== ruleString.length - 1) {
  return { toolName: normalizeLegacyToolName(ruleString) }
}

if (!toolName) {
  return { toolName: normalizeLegacyToolName(ruleString) }
}
```

这种"容错"设计确保：
- 用户输入的畸形规则不会导致崩溃
- 尽可能地将输入解释为工具名

## 风险、边界与改进建议

### 潜在风险

1. **转义顺序错误**:
   - 虽然当前实现正确，但未来的修改可能破坏顺序
   - 建议添加单元测试验证转义/反转义的对称性

2. **Unicode 和特殊字符**:
   - 当前只处理 ASCII 反斜杠和括号
   - 其他特殊字符（如 Unicode 变体）可能产生意外行为

3. **性能问题**:
   - `findFirstUnescapedChar` 和 `findLastUnescapedChar` 是 O(n) 算法
   - 对于极长的规则字符串可能影响性能
   - 但通常规则字符串很短，这不是实际问题

4. **遗留工具名扩散**:
   - 映射表分散在多个地方（此处、工具定义、其他解析代码）
   - 可能产生不一致

### 边界情况

1. **嵌套括号**:
   ```typescript
   // 当前实现只处理最外层括号
   'Bash(echo $(date))'
   // 解析为 toolName='Bash', content='echo $(date)'
   // 这是正确的，因为内部括号是内容的一部分
   ```

2. **连续反斜杠**:
   ```typescript
   'Bash(\\\\)'  // 4 个反斜杠
   // 查找算法：每个字符前面有偶数个反斜杠
   // 所以 '(' 是未转义的
   ```

3. **空内容和通配符**:
   ```typescript
   'Bash()'   → { toolName: 'Bash' }
   'Bash(*)'  → { toolName: 'Bash' }
   // 这两种情况都被视为工具级规则
   ```

4. **条件编译的复杂性**:
   - `BRIEF_TOOL_NAME` 在运行时可能为 `null`
   - 映射表构建需要考虑这一点

### 改进建议

1. **单元测试覆盖**:
   ```typescript
   // 建议添加的测试用例
describe('permissionRuleParser', () => {
     test('escape/unescape symmetry', () => {
       const cases = [
         'simple',
         'with(parens)',
         'with\\backslash',
         'with\\(both\\)',
         '',
         '()',
         '(*)'
       ]
       cases.forEach(c => {
         expect(unescapeRuleContent(escapeRuleContent(c))).toBe(c)
       })
     })
     
     test('parse/serialize symmetry', () => {
       const cases = [
         { toolName: 'Bash' },
         { toolName: 'Bash', ruleContent: 'npm install' },
         { toolName: 'Bash', ruleContent: 'python -c "print(1)"' }
       ]
       cases.forEach(c => {
         expect(permissionRuleValueFromString(permissionRuleValueToString(c))).toEqual(c)
       })
     })
   })
   ```

2. **验证函数**:
   ```typescript
   // 添加规则格式验证
   export function isValidPermissionRule(ruleString: string): boolean {
     try {
       const parsed = permissionRuleValueFromString(ruleString)
       return parsed.toolName.length > 0
     } catch {
       return false
     }
   }
   ```

3. **类型安全增强**:
   ```typescript
   // 使用 branded type 区分已解析和未解析的规则
   type ParsedRule = PermissionRuleValue & { __brand: 'parsed' }
   type RawRule = string & { __brand: 'raw' }
   
   export function parseRule(raw: RawRule): ParsedRule
   export function serializeRule(parsed: ParsedRule): RawRule
   ```

4. **文档生成**:
   ```typescript
   // 自动生成遗留工具名映射文档
   export const LEGACY_TOOL_NAME_DOCUMENTATION = Object.entries(LEGACY_TOOL_NAME_ALIASES)
     .map(([old, new_]) => `- \`${old}\` → \`${new_}\``)
     .join('\n')
   ```

5. **性能优化**（如果需要）:
   ```typescript
   // 对于大量规则解析，考虑使用正则表达式
   const RULE_PATTERN = /^(.*?)(?:\((.*)\))?$/
   // 但需要先处理转义，所以可能不更简单
   ```

6. **错误报告改进**:
   ```typescript
   // 当前只是容错，建议增加警告日志
   if (closeParenIndex !== ruleString.length - 1) {
     logForDebugging(`Malformed permission rule: ${ruleString} (content after closing paren)`)
     return { toolName: normalizeLegacyToolName(ruleString) }
   }
   ```

7. **标准化**:
   ```typescript
   // 添加规则规范化函数
   export function normalizePermissionRule(ruleString: string): string {
     const parsed = permissionRuleValueFromString(ruleString)
     return permissionRuleValueToString(parsed)
   }
   // 'Bash()' → 'Bash'
   // 'Bash(*)' → 'Bash'
   // 'Task' → 'Agent' (遗留名映射)
   ```
