# skillLoadedEvent.ts 深度研究文档

## 场景与职责

`skillLoadedEvent.ts` 是 Claude Code 技能系统遥测模块，负责在会话启动时记录所有可用技能（skills）的加载事件。该模块的核心职责包括：

1. **技能可用性追踪**：记录会话启动时可用的所有技能，用于分析技能使用分布和覆盖率
2. **技能元数据收集**：收集技能名称、来源、加载位置、字符预算等关键元数据
3. **隐私安全的数据分类**：使用 TypeScript 类型系统强制区分 PII 数据和非敏感数据
4. **支持技能发现分析**：为产品团队提供技能加载模式的洞察，优化技能推荐和发现

该模块将会话启动时的技能快照发送到分析后端，是理解用户可用功能集和技能采用率的关键数据源。

## 功能点目的

### 1. 技能加载事件记录 (`logSkillsLoaded`)
- **目的**：为每个可用技能生成 `tengu_skill_loaded` 分析事件
- **触发时机**：会话启动时（`main.tsx` 中的 `logSessionTelemetry` 函数）
- **数据用途**：
  - 技能使用分布分析（哪些技能被加载得最频繁）
  - 技能来源分析（bundled、skills 目录、plugin、MCP 等）
  - 字符预算影响分析（技能列表大小对上下文窗口的影响）

### 2. 隐私安全的数据分类
- **`AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`**：标记可路由到特权 BQ 列的数据
  - 用于 `_PROTO_skill_name`：技能名称原样发送到受控的 `skill_name` BQ 列
  - 该列有严格的访问控制，适合存储未脱敏的标识符
  
- **`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`**：标记非敏感元数据
  - 用于 `skill_source`、`skill_loaded_from`、`skill_kind`
  - 确保这些字段不包含代码片段或文件路径

### 3. 字符预算计算
- **目的**：记录当前会话的技能列表字符预算
- **计算逻辑**：基于上下文窗口大小的 1%（`SKILL_BUDGET_CONTEXT_PERCENT = 0.01`）
- **用途**：分析技能列表大小与模型性能的关系

## 具体技术实现

### 关键流程

```
logSkillsLoaded(cwd, contextWindowTokens)
  ├── getSkillToolCommands(cwd)           // 获取所有可用技能命令
  ├── getCharBudget(contextWindowTokens)  // 计算字符预算
  │
  └── 遍历 skills
        ├── 跳过非 'prompt' 类型技能
        └── 对每个技能调用 logEvent('tengu_skill_loaded', {...})
              ├── _PROTO_skill_name: 技能名称（PII 标记）
              ├── skill_source: 技能来源
              ├── skill_loaded_from: 加载位置
              ├── skill_budget: 字符预算
              └── skill_kind: 技能类型（如果有）
```

### 数据结构

```typescript
// 技能加载事件元数据
{
  // PII 标记字段 - 路由到特权 BQ 列
  _PROTO_skill_name: string as AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED,
  
  // 非敏感元数据
  skill_source: string as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  skill_loaded_from: string as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  skill_budget: number,
  skill_kind?: string as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
}
```

### 技能来源分类

| 来源值 | 含义 |
|--------|------|
| `bundled` | 内置捆绑技能 |
| `skills` | 用户技能目录（`.claude/skills/`） |
| `plugin` | 插件提供的技能 |
| `mcp` | MCP 服务器提供的技能 |
| `commands_DEPRECATED` | 传统命令（已弃用） |

### 字符预算计算

```typescript
// src/tools/SkillTool/prompt.ts
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 1% 的上下文窗口
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000          // 回退值：1% of 200k × 4

export function getCharBudget(contextWindowTokens?: number): number {
  if (Number(process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET)) {
    return Number(process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET)
  }
  if (contextWindowTokens) {
    return Math.floor(
      contextWindowTokens * CHARS_PER_TOKEN * SKILL_BUDGET_CONTEXT_PERCENT,
    )
  }
  return DEFAULT_CHAR_BUDGET
}
```

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `src/utils/telemetry/skillLoadedEvent.ts` | 本文件，技能加载事件生成 |
| `src/commands.ts` | `getSkillToolCommands` - 获取所有可用技能 |
| `src/tools/SkillTool/prompt.ts` | `getCharBudget` - 字符预算计算 |
| `src/services/analytics/index.ts` | `logEvent` - 分析事件接口 |

### 调用方
| 文件 | 调用位置 |
|------|----------|
| `src/main.tsx` | `logSessionTelemetry()` 函数，会话启动时调用 |

**具体调用代码**（`main.tsx`）：
```typescript
function logSessionTelemetry(): void {
  const model = parseUserSpecifiedModel(getInitialMainLoopModel() ?? getDefaultMainLoopModel());
  void logSkillsLoaded(getCwd(), getContextWindowForModel(model, getSdkBetas()));
  // ... 其他遥测记录
}
```

### 被调用方/依赖
| 文件 | 用途 |
|------|------|
| `src/commands.ts` | `getSkillToolCommands` - 获取技能列表；`getCharBudget` - 预算计算 |
| `src/services/analytics/index.ts` | `logEvent` - 发送分析事件 |

## 依赖与外部交互

### 分析服务集成

通过 `src/services/analytics/index.ts` 的 `logEvent` 函数发送事件：

```typescript
// 分析事件接口
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // 仅限 boolean | number | undefined，禁止字符串防止泄露代码/路径
): void
```

**重要设计**：`logEvent` 的 `metadata` 参数类型故意限制为 `boolean | number | undefined`，禁止字符串值。这是为了防止意外记录代码片段或文件路径。如果需要记录字符串，必须使用类型断言：

```typescript
// 正确：显式标记已验证
skill_source: skill.source as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS

// 错误：直接传递字符串会被类型系统阻止
skill_source: skill.source  // TypeScript 错误
```

### _PROTO_ 前缀机制

`_PROTO_skill_name` 使用特殊前缀将数据路由到特权 BigQuery 列：

1. **sink.ts** 在发送到 Datadog 之前会剥离所有 `_PROTO_*` 键
2. **firstPartyEventLoggingExporter** 识别 `_PROTO_*` 键，将其值提升到顶层 proto 字段
3. 这样可以确保未脱敏的技能名称只进入受控的 BQ 列，而不会泄露到其他后端

### 技能命令过滤

`getSkillToolCommands` 过滤逻辑（来自 `commands.ts`）：

```typescript
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&                    // 仅 prompt 类型
        !cmd.disableModelInvocation &&              // 允许模型调用
        cmd.source !== 'builtin' &&                 // 排除内置命令
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

## 风险、边界与改进建议

### 风险点

1. **技能名称隐私风险**
   - 技能名称可能包含敏感信息（如项目名称、内部工具名）
   - **缓解**：使用 `_PROTO_` 前缀路由到受控 BQ 列，Datadog 不会收到

2. **事件重复发送风险**
   - 如果 `logSessionTelemetry` 被多次调用，会生成重复事件
   - **当前状态**：`main.tsx` 中仅在会话启动调用一次

3. **技能列表获取失败**
   - `getSkillToolCommands` 是异步操作，可能失败
   - **当前处理**：使用 `void` 调用忽略 Promise，失败静默

4. **类型安全依赖人工审查**
   - `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型断言依赖开发者正确验证
   - **风险**：错误使用可能导致敏感数据泄露

### 边界条件

1. **空技能列表**
   - 如果没有可用技能（如 `--bare` 模式），不会发送任何事件
   - 这是预期行为，表示"没有技能可用"

2. **大量技能**
   - 每个技能生成一个独立事件，大量技能可能导致事件突发
   - 但技能数量通常 < 100，不会构成性能问题

3. **字符预算为零**
   - 如果 `contextWindowTokens` 为 0 或未定义，使用 `DEFAULT_CHAR_BUDGET = 8000`

### 改进建议

1. **批量事件发送**
   - 当前每个技能发送一个独立事件
   - 建议：添加 `tengu_skills_loaded` 批量事件，包含技能计数和列表摘要
   - 减少事件数量，降低后端压力

2. **失败重试机制**
   - 当前使用 `void` 忽略 Promise，失败无感知
   - 建议：添加错误日志或重试机制

3. **技能变更检测**
   - 当前仅在会话启动记录
   - 建议：在技能动态加载/卸载时也记录事件

4. **更细粒度的来源分类**
   - 当前 `skill_source` 和 `skill_loaded_from` 可能有重叠
   - 建议：统一为清晰的层次结构（如 `source: 'plugin'`, `pluginName: 'xxx'`）

5. **技能使用关联**
   - 当前只记录"加载了哪些技能"
   - 建议：在后续工具调用中记录"实际使用了哪些技能"，计算加载-使用转化率
