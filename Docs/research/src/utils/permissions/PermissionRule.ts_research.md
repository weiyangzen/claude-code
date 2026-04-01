# PermissionRule.ts 深度研究

## 场景与职责

`PermissionRule.ts` 是 Claude Code 权限系统的规则定义模块，负责定义权限规则的数据结构、类型和验证 schema。权限规则是 Claude Code 权限系统的核心概念，用于控制哪些工具可以在什么条件下被执行。

### 核心职责
1. **规则类型定义**: 定义权限规则的值结构、行为类型和来源类型
2. **Schema 验证**: 提供 Zod schema 用于运行时验证权限规则数据
3. **向后兼容**: 重新导出类型定义以保持与提取到 `src/types/permissions.ts` 的类型的兼容性

---

## 功能点目的

### 1. 权限行为类型 (`PermissionBehavior`)
定义三种基本的权限行为：
- `'allow'` - 允许工具执行
- `'deny'` - 拒绝工具执行
- `'ask'` - 要求用户确认

### 2. 权限规则值 (`PermissionRuleValue`)
表示单个权限规则的内容：
- `toolName`: 规则适用的工具名称
- `ruleContent`: 可选的规则内容（用于前缀匹配等高级规则）

### 3. 权限规则来源 (`PermissionRuleSource`)
定义规则可以来自哪些来源：
- `userSettings` - 用户全局设置
- `projectSettings` - 项目级设置
- `localSettings` - 本地设置（gitignored）
- `flagSettings` - 功能标志设置
- `policySettings` - 组织策略设置
- `cliArg` - 命令行参数
- `command` - 命令执行时设置
- `session` - 当前会话临时设置

### 4. 完整权限规则 (`PermissionRule`)
组合了规则值、行为和来源的完整规则定义。

---

## 具体技术实现

### 关键数据结构

```typescript
// 权限行为（已提取到 src/types/permissions.ts）
type PermissionBehavior = 'allow' | 'deny' | 'ask'

// 权限规则值（已提取到 src/types/permissions.ts）
interface PermissionRuleValue {
  toolName: string
  ruleContent?: string
}

// 权限规则来源（已提取到 src/types/permissions.ts）
type PermissionRuleSource = 
  | 'userSettings' 
  | 'projectSettings' 
  | 'localSettings' 
  | 'flagSettings' 
  | 'policySettings' 
  | 'cliArg' 
  | 'command' 
  | 'session'

// 完整权限规则（已提取到 src/types/permissions.ts）
interface PermissionRule {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

### Zod Schema 实现

```typescript
// 权限行为 schema
export const permissionBehaviorSchema = lazySchema(() =>
  z.enum(['allow', 'deny', 'ask']),
)

// 权限规则值 schema
export const permissionRuleValueSchema = lazySchema(() =>
  z.object({
    toolName: z.string(),
    ruleContent: z.string().optional(),
  }),
)
```

### 延迟加载模式 (`lazySchema`)

使用 `lazySchema` 包装 Zod schema 定义，实现：
1. **循环依赖解决**: 避免在模块加载时立即执行 schema 定义
2. **性能优化**: 延迟 schema 编译直到首次使用
3. **Tree-shaking 友好**: 未使用的 schema 不会被打包

---

## 关键代码路径与文件引用

### 类型定义源头

| 类型 | 源头文件 | 说明 |
|------|----------|------|
| `PermissionBehavior` | `src/types/permissions.ts` | 权限行为枚举 |
| `PermissionRule` | `src/types/permissions.ts` | 完整规则类型 |
| `PermissionRuleSource` | `src/types/permissions.ts` | 规则来源 |
| `PermissionRuleValue` | `src/types/permissions.ts` | 规则值 |

### Schema 使用方

| 使用方 | 路径 | 用途 |
|--------|------|------|
| `PermissionUpdateSchema.ts` | `src/utils/permissions/PermissionUpdateSchema.ts` | 在权限更新 schema 中引用 |
| 权限验证逻辑 | 多个位置 | 运行时验证规则数据 |

### 内部依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `lazySchema` | `src/utils/lazySchema.js` | 延迟加载 Zod schema |

---

## 依赖与外部交互

### 模块依赖图

```
PermissionRule.ts
    ↓
src/types/permissions.ts (类型定义源头)
    ↓
src/utils/lazySchema.js (延迟加载工具)
```

### 规则字符串格式

权限规则在设置文件和 CLI 中以字符串形式表示：

```
ToolName                    → { toolName: 'ToolName' }
ToolName(*)                 → { toolName: 'ToolName' }
ToolName(content)           → { toolName: 'ToolName', ruleContent: 'content' }
ToolName(prefix:*)          → { toolName: 'ToolName', ruleContent: 'prefix:*' }
```

解析逻辑位于 `permissionRuleParser.ts`。

### 规则匹配逻辑

规则匹配由 `permissions.ts` 中的以下函数处理：
- `toolMatchesRule()` - 检查工具是否匹配规则
- `toolAlwaysAllowedRule()` - 查找工具的允许规则
- `getDenyRuleForTool()` - 查找工具的拒绝规则
- `getAskRuleForTool()` - 查找工具的询问规则

---

## 风险、边界与改进建议

### 当前风险

1. **Schema 与类型不同步**:
   - Zod schema 和 TypeScript 类型是分开定义的
   - 如果修改类型但忘记更新 schema，可能导致运行时错误

2. **规则内容格式**:
   - `ruleContent` 是自由格式字符串，没有强类型约束
   - 不同工具可能有不同的内容格式约定（如 Bash 的前缀规则）

### 边界情况

1. **空字符串 ruleContent**:
   ```typescript
   { toolName: 'Bash', ruleContent: '' }
   ```
   空字符串与 `undefined` 在语义上可能不同，需要明确规范。

2. **特殊字符**:
   `ruleContent` 可能包含需要转义的字符（如括号），转义逻辑在 `permissionRuleParser.ts` 中处理。

3. **工具名称大小写**:
   工具名称是大小写敏感的，但某些旧代码可能使用不同的大小写。

### 改进建议

1. **Schema 与类型同步**:
   ```typescript
   // 使用 z.infer 从 schema 推导类型
   export const permissionRuleValueSchema = lazySchema(() =>
     z.object({
       toolName: z.string(),
       ruleContent: z.string().optional(),
     }),
   )
   
   // 从 schema 导出类型，确保同步
   export type PermissionRuleValue = z.infer<ReturnType<typeof permissionRuleValueSchema>>
   ```

2. **规则内容类型化**:
   ```typescript
   // 为不同工具定义特定的规则内容格式
   type BashRuleContent = 
     | { type: 'prefix'; prefix: string }
     | { type: 'exact'; command: string }
     | { type: 'wildcard'; pattern: string }
   
   type PermissionRuleValue<T = unknown> = {
     toolName: string
     ruleContent?: T extends unknown ? string : T
   }
   ```

3. **验证增强**:
   ```typescript
   export const permissionRuleValueSchema = lazySchema(() =>
     z.object({
       toolName: z.string().min(1, 'Tool name cannot be empty'),
       ruleContent: z.string()
         .refine(
           content => !content?.includes('\0'), 
           'Rule content cannot contain null bytes'
         )
         .optional(),
     }),
   )
   ```

4. **文档化规则格式**:
   ```typescript
   /**
    * Permission rule content format by tool:
    * 
    * - Bash: "prefix:*" for prefix matching, "exact command" for exact matching
    * - FileRead: "path/pattern" for path matching
    * - FileEdit: "path/pattern" for path matching
    * 
    * @example
    * Bash(python:*)     // Matches any python command
    * Bash(npm install)  // Matches exact "npm install" command
    * Read(src/**)       // Matches any file in src directory
    */
   ```

5. **迁移策略**:
   - 长期考虑将 schema 定义与类型定义完全合并
   - 使用 `zod-to-ts` 等工具自动生成 TypeScript 类型
   - 或者使用 TypeScript 装饰器/元数据生成 Zod schema

### 架构建议

随着权限系统复杂度增加，建议考虑：

1. **规则引擎抽象**: 将规则匹配逻辑抽象为可插拔的规则引擎
2. **规则版本控制**: 为规则格式添加版本号，支持平滑迁移
3. **规则验证器**: 在设置加载时进行全面的规则验证
4. **规则编辑器**: 提供结构化的规则编辑器，而非纯文本输入
