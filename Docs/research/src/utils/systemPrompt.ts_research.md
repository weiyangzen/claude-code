# systemPrompt.ts 研究文档

## 场景与职责

`systemPrompt.ts` 负责构建 Claude Code 的有效系统提示词（system prompt）。该模块实现了复杂的提示词优先级逻辑，支持多种模式（覆盖模式、协调器模式、Agent 模式、自定义提示词、默认提示词），并处理 proactive 模式的特殊需求。

## 功能点目的

### 系统提示词优先级系统
构建有效系统提示词的优先级（从高到低）：

1. **覆盖系统提示词** (`overrideSystemPrompt`): 如果设置，替换所有其他提示词
2. **协调器系统提示词** (`coordinator mode`): 如果协调器模式激活且没有 Agent
3. **Agent 系统提示词** (`mainThreadAgentDefinition`): 
   - Proactive 模式：追加到默认提示词
   - 其他模式：替换默认提示词
4. **自定义系统提示词** (`customSystemPrompt`): 通过 `--system-prompt` 指定
5. **默认系统提示词** (`defaultSystemPrompt`): 标准 Claude Code 提示词
6. **追加系统提示词** (`appendSystemPrompt`): 总是追加在最后（覆盖模式除外）

### Proactive 模式支持
- 在 proactive 模式下，Agent 提示词追加到默认提示词而非替换
- Agent 在 proactive 默认提示词基础上添加领域特定行为

### 分析遥测
- 当主循环 Agent 有 memory 时，记录 `tengu_agent_memory_loaded` 事件

## 具体技术实现

### 核心函数

```typescript
export function buildEffectiveSystemPrompt({
  mainThreadAgentDefinition,
  toolUseContext,
  customSystemPrompt,
  defaultSystemPrompt,
  appendSystemPrompt,
  overrideSystemPrompt,
}: {
  mainThreadAgentDefinition: AgentDefinition | undefined
  toolUseContext: Pick<ToolUseContext, 'options'>
  customSystemPrompt: string | undefined
  defaultSystemPrompt: string[]
  appendSystemPrompt: string | undefined
  overrideSystemPrompt?: string | null
}): SystemPrompt
```

### 优先级逻辑实现

```typescript
// 1. 覆盖模式
if (overrideSystemPrompt) {
  return asSystemPrompt([overrideSystemPrompt])
}

// 2. 协调器模式
if (feature('COORDINATOR_MODE') && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE) && !mainThreadAgentDefinition) {
  const { getCoordinatorSystemPrompt } = require('../coordinator/coordinatorMode.js')
  return asSystemPrompt([
    getCoordinatorSystemPrompt(),
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}

// 获取 Agent 提示词
const agentSystemPrompt = mainThreadAgentDefinition
  ? isBuiltInAgent(mainThreadAgentDefinition)
    ? mainThreadAgentDefinition.getSystemPrompt({ toolUseContext: { options: toolUseContext.options } })
    : mainThreadAgentDefinition.getSystemPrompt()
  : undefined

// 3. Proactive 模式：Agent 提示词追加
if (agentSystemPrompt && (feature('PROACTIVE') || feature('KAIROS')) && isProactiveActive_SAFE_TO_CALL_ANYWHERE()) {
  return asSystemPrompt([
    ...defaultSystemPrompt,
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}

// 4-5. 标准优先级
return asSystemPrompt([
  ...(agentSystemPrompt
    ? [agentSystemPrompt]
    : customSystemPrompt
      ? [customSystemPrompt]
      : defaultSystemPrompt),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

### 延迟加载模式

```typescript
// Dead code elimination: conditional import for proactive mode
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? (require('../proactive/index.js') as typeof import('../proactive/index.js'))
    : null

function isProactiveActive_SAFE_TO_CALL_ANYWHERE(): boolean {
  return proactiveModule?.isProactiveActive() ?? false
}
```

## 关键代码路径与文件引用

### 本文件导出
- `buildEffectiveSystemPrompt(params)`: 构建有效系统提示词
- `asSystemPrompt`, `SystemPrompt`: 从 `./systemPromptType.js` 重新导出

### 依赖模块

| 模块 | 用途 |
|------|------|
| `bun:bundle` | `feature` 死代码消除 |
| `../services/analytics/index.js` | `logEvent` |
| `../Tool.js` | `ToolUseContext` 类型 |
| `../tools/AgentTool/loadAgentsDir.js` | `AgentDefinition`, `isBuiltInAgent` |
| `./envUtils.js` | `isEnvTruthy` |
| `./systemPromptType.js` | `asSystemPrompt`, `SystemPrompt` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/utils/queryContext.ts` | 构建查询上下文时使用 |

## 依赖与外部交互

### 与 Agent 系统的集成
- 接收 `AgentDefinition` 对象
- 区分内置 Agent 和自定义 Agent（调用签名不同）
- 内置 Agent 接收 `toolUseContext`，自定义 Agent 不需要

### 与 Proactive 模式的集成
- 延迟加载 proactive 模块
- 使用 `feature()` 进行死代码消除
- 检测 proactive 激活状态决定提示词组合方式

### 与协调器模式的集成
- 检查 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量
- 延迟加载协调器模块避免循环依赖

### 与分析系统的集成
- 记录 Agent memory 加载事件
- 仅在 `USER_TYPE === 'ant'` 时包含详细元数据

## 风险、边界与改进建议

### 潜在风险

1. **循环依赖**: 协调器模块的延迟加载是为了避免循环依赖，但仍需谨慎
2. **延迟加载失败**: `require()` 可能失败，需要错误处理
3. **优先级复杂性**: 多种模式的组合可能导致意外的提示词结果

### 边界情况

1. **空提示词**: 所有提示词参数都为空时返回空数组
2. **Agent 与自定义提示词冲突**: Agent 提示词优先级高于自定义提示词
3. **Proactive 与非 Proactive 切换**: 运行时切换可能导致不一致行为

### 改进建议

1. **提示词预览功能**: 添加调试模式显示最终组合的系统提示词
```typescript
if (isDebugMode()) {
  logForDebugging(`Effective system prompt: ${JSON.stringify(prompt)}`)
}
```

2. **配置验证**: 验证提示词参数组合的有效性
```typescript
function validatePromptParams(params: BuildParams): void {
  if (params.overrideSystemPrompt && params.appendSystemPrompt) {
    console.warn('appendSystemPrompt is ignored when overrideSystemPrompt is set')
  }
}
```

3. **提示词模板**: 支持模板变量替换
```typescript
const promptTemplate = 'You are {{agentName}}. Current directory: {{cwd}}'
const prompt = renderTemplate(promptTemplate, { agentName: 'Claude', cwd: process.cwd() })
```

4. **版本控制**: 为系统提示词添加版本号，便于追踪变化
```typescript
interface SystemPrompt {
  readonly __brand: 'SystemPrompt'
  version: string
  content: readonly string[]
}
```

5. **A/B 测试支持**: 支持系统提示词的 A/B 测试
```typescript
export function buildEffectiveSystemPromptWithExperiment(
  params: BuildParams,
  experimentGroup?: 'control' | 'treatment'
): SystemPrompt
```

6. **文档生成**: 从代码自动生成系统提示词组合规则文档
