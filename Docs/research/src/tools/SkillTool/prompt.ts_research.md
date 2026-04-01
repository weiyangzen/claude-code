# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 SkillTool 的**提示生成模块**，负责生成提供给 AI 模型的工具使用说明。它实现了技能列表的格式化、字符预算管理和提示模板生成，确保模型能够：

1. **发现可用技能** - 在上下文窗口预算内展示技能列表
2. **理解调用方式** - 学习如何正确调用 SkillTool
3. **遵循调用约束** - 了解何时必须使用工具而非直接回复

该模块被 `SkillTool.ts` 的 `prompt` 方法调用，也用于系统提示组装。

## 功能点目的

### 1. 字符预算管理 (`getCharBudget`)

控制技能列表占用的上下文窗口大小：

- 默认占用上下文窗口的 1%（`SKILL_BUDGET_CONTEXT_PERCENT = 0.01`）
- 按 4 字符/token 估算（`CHARS_PER_TOKEN = 4`）
- 支持环境变量覆盖（`SLASH_COMMAND_TOOL_CHAR_BUDGET`）
- 默认回退值 8000 字符（对应 200K 上下文的 1%）

### 2. 技能列表格式化 (`formatCommandsWithinBudget`)

智能截断技能描述以适应预算：

- 优先保留 bundled 技能的完整描述（永不截断）
- 非 bundled 技能按需截断描述
- 极端情况下仅显示技能名称（无描述）
- 单条描述硬上限 250 字符（`MAX_LISTING_DESC_CHARS`）

### 3. 提示模板生成 (`getPrompt`)

生成给模型的 SkillTool 使用说明：

- 工具用途说明
- 调用语法示例
- 重要约束（阻塞要求、禁止重复调用等）

### 4. 技能信息统计 (`getSkillToolInfo`, `getSkillInfo`)

提供技能数量统计用于上下文分析：

- 总技能数
- 包含在提示中的技能数

## 具体技术实现

### 关键常量

```typescript
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 1% 上下文预算
export const CHARS_PER_TOKEN = 4                   // 字符/token 比例
export const DEFAULT_CHAR_BUDGET = 8_000          // 默认预算（200K 上下文的 1%）
export const MAX_LISTING_DESC_CHARS = 250         // 单条描述上限
const MIN_DESC_LENGTH = 20                        // 最小描述长度阈值
```

### 预算计算流程

```
getCharBudget(contextWindowTokens?)
├── 检查环境变量 SLASH_COMMAND_TOOL_CHAR_BUDGET
│   └── 如设置，直接返回
├── 如有传入 contextWindowTokens
│   └── 计算: tokens × 4 × 0.01
└── 否则返回 DEFAULT_CHAR_BUDGET (8000)
```

### 技能列表截断算法

```
formatCommandsWithinBudget(commands, contextWindowTokens?)
├── 计算预算
├── 尝试完整描述
│   └── 如总长度 ≤ 预算，直接返回
├── 分离 bundled 和非 bundled 技能
├── 计算 bundled 技能占用空间
├── 计算剩余预算给非 bundled
├── 计算每条非 bundled 技能可用描述长度
│   ├── 如 < MIN_DESC_LENGTH (20)
│   │   └── 仅显示名称（bundled 保留描述）
│   │   └── 记录遥测: names_only 模式
│   └── 否则
│       └── 截断描述至可用长度
│       └── 记录遥测: description_trimmed 模式
└── 返回格式化列表
```

### 提示模板

```typescript
export const getPrompt = memoize(async (_cwd: string): Promise<string> => {
  return `Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match. Skills provide specialized capabilities and domain knowledge.

When users reference a "slash command" or "/<something>" (e.g., "/commit", "/review-pr"), they are referring to a skill. Use this tool to invoke it.

How to invoke:
- Use this tool with the skill name and optional arguments
- Examples:
  - \`skill: "pdf"\` - invoke the pdf skill
  - \`skill: "commit", args: "-m 'Fix bug'"\` - invoke with arguments
  - \`skill: "review-pr", args: "123"\` - invoke with arguments
  - \`skill: "ms-office-suite:pdf"\` - invoke using fully qualified name

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task
- NEVER mention a skill without actually calling this tool
- Do not invoke a skill that is already running
- Do not use this tool for built-in CLI commands (like /help, /clear, etc.)
- If you see a <${COMMAND_NAME_TAG}> tag in the current conversation turn, the skill has ALREADY been loaded - follow the instructions directly instead of calling this tool again
`
})
```

关键约束说明：
- **BLOCKING REQUIREMENT**: 必须优先调用工具，而非直接回复
- **NEVER mention without calling**: 禁止只提及不调用
- **COMMAND_NAME_TAG 检查**: 避免重复加载已激活技能

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `lodash-es` | `memoize` | 提示缓存 |
| `src/commands.ts` | `Command`, `getCommandName`, `getSkillToolCommands`, `getSlashCommandToolSkills` | 技能查询与格式化 |
| `src/constants/xml.ts` | `COMMAND_NAME_TAG` | 提示模板中的 XML 标签 |
| `src/ink/stringWidth.ts` | `stringWidth` | 准确计算字符串显示宽度（处理 Unicode）|
| `src/services/analytics/index.ts` | `logEvent`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 遥测记录 |
| `src/utils/array.ts` | `count` | 数组计数 |
| `src/utils/debug.ts` | `logForDebugging` | 调试日志 |
| `src/utils/errors.ts` | `toError` | 错误转换 |
| `src/utils/format.ts` | `truncate` | 字符串截断 |
| `src/utils/log.ts` | `logError` | 错误日志 |

### 被调用方

| 文件路径 | 调用方式 | 用途 |
|---------|---------|------|
| `src/tools/SkillTool/SkillTool.ts` | `import { getPrompt } from './prompt.js'` | 工具提示方法 |
| `src/commands.ts` | `getSkillToolCommands` 等 | 技能列表获取 |

### 导出函数

```typescript
// 主要导出
export function getCharBudget(contextWindowTokens?: number): number
export function formatCommandsWithinBudget(commands: Command[], contextWindowTokens?: number): string
export const getPrompt: ((cwd: string) => Promise<string>) & { cache?: Map<string, Promise<string>> | undefined }
export function getSkillToolInfo(cwd: string): Promise<{ totalCommands: number; includedCommands: number }>
export function getLimitedSkillToolCommands(cwd: string): Promise<Command[]>
export function clearPromptCache(): void
export function getSkillInfo(cwd: string): Promise<{ totalSkills: number; includedSkills: number }>
```

## 依赖与外部交互

### 核心算法依赖

1. **字符串宽度计算**
   - 使用 `stringWidth` 而非 `String.length`
   - 正确处理 CJK 字符、emoji 等宽字符

2. **缓存机制**
   - `getPrompt` 使用 `memoize` 缓存
   - `clearPromptCache` 提供手动清除
   - 缓存键为 `cwd`（当前工作目录）

### 遥测集成

```typescript
// 描述截断时记录遥测（仅 ant 用户）
if (process.env.USER_TYPE === 'ant') {
  logEvent('tengu_skill_descriptions_truncated', {
    skill_count: commands.length,
    budget,
    full_total: fullTotal,
    truncation_mode: 'names_only' | 'description_trimmed',
    max_desc_length: maxDescLen,
    bundled_count: bundledIndices.size,
    bundled_chars: bundledChars,
    truncated_count?: number,
  })
}
```

### 技能来源处理

```typescript
// 格式化单条技能描述
function formatCommandDescription(cmd: Command): string {
  const displayName = getCommandName(cmd)
  // 调试日志：检查 userFacingName 与 name 不一致的情况
  if (cmd.name !== displayName && cmd.type === 'prompt' && cmd.source === 'plugin') {
    logForDebugging(`Skill prompt: showing "${cmd.name}" (userFacingName="${displayName}")`)
  }
  return `- ${cmd.name}: ${getCommandDescription(cmd)}`
}
```

## 风险、边界与改进建议

### 已知风险

1. **预算计算不准确**
   - 使用固定 4 字符/token 比例，实际比例因内容而异
   - 可能导致预算超支或利用不足
   - 建议：使用实际 tokenizer 计算

2. **缓存失效问题**
   - `getPrompt` 缓存基于 `cwd`，但技能列表可能变化
   - 添加/删除技能后提示不会自动更新
   - 建议：添加技能变更监听或缩短缓存时间

3. **截断策略争议**
   - bundled 技能永不截断，可能在极端情况下占用过多预算
   - 非 bundled 技能可能完全无描述
   - 建议：添加最小描述保证（如至少 10 字符）

### 边界条件

1. **空技能列表**
   ```typescript
   if (commands.length === 0) return ''
   ```

2. **极端预算不足**
   - 当 `maxDescLen < MIN_DESC_LENGTH` (20) 时
   - 非 bundled 技能仅显示名称

3. **全 bundled 技能**
   - `restCommands.length === 0` 时
   - 直接返回完整描述，无需截断计算

4. **单条描述过长**
   - 超过 `MAX_LISTING_DESC_CHARS` (250) 时
   - 使用 `truncate` 截断并添加省略号

### 改进建议

1. **动态预算调整**
   ```typescript
   // 建议：基于实际 token 计数调整
   export async function getDynamicCharBudget(
     commands: Command[],
     contextWindowTokens: number
   ): Promise<number> {
     const baseBudget = getCharBudget(contextWindowTokens)
     // 使用实际 tokenizer 估算
     const estimatedTokens = await estimateTokens(commands)
     return Math.min(baseBudget, estimatedTokens * CHARS_PER_TOKEN)
   }
   ```

2. **优先级排序**
   - 当前按原始顺序处理
   - 建议：按使用频率或相关性排序，优先保留高频技能描述

3. **渐进式披露**
   ```typescript
   // 建议：添加技能分类或搜索提示
   export function formatCommandsWithCategories(commands: Command[]): string {
     // 按 source 分组展示
     // 添加 "...and N more" 提示
   }
   ```

4. **缓存优化**
   ```typescript
   // 建议：添加版本号或哈希校验
   export const getPrompt = memoize(async (cwd: string, skillEtag?: string): Promise<string> => {
     // 技能变更时使缓存失效
   })
   ```

5. **国际化支持**
   - 当前提示为硬编码英文
   - 建议：支持多语言提示模板

6. **测试覆盖**
   - 预算计算边界条件
   - 截断算法各种模式
   - Unicode 字符宽度计算
   - 缓存清除功能

### 性能考虑

- `formatCommandsWithinBudget` 时间复杂度 O(n)，n 为技能数
- 多次调用 `stringWidth` 可能影响大列表性能
- `memoize` 缓存提示生成结果，避免重复计算

### 与系统其他部分的协作

```
system prompt assembly
    │
    ├── getPrompt() ──────────────────────► SkillTool 使用说明
    │
    ├── getSkillToolCommands() ───────────► 可用技能列表
    │   └── formatCommandsWithinBudget() ► 格式化并截断
    │
    └── getSkillInfo() ───────────────────► 上下文分析统计
```
