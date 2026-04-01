# permissions.ts 研究文档

## 场景与职责

`src/types/permissions.ts` 是 Claude Code CLI 的权限系统核心类型定义文件。该文件被显式设计为"纯类型定义文件"以打破循环依赖，使得其他模块可以在不引入重量级实现的情况下使用权限类型。核心职责：

1. **权限模式定义**: 定义用户可配置的权限模式（acceptEdits, bypassPermissions, default, dontAsk, plan, auto, bubble）
2. **权限规则系统**: 定义规则的来源、值和行为（allow/deny/ask）
3. **权限决策类型**: 定义 allow/ask/deny/passthrough 四种决策结果的完整结构
4. **权限决策理由**: 定义决策来源的详细追踪（rule, mode, hook, classifier 等）
5. **工具权限上下文**: 定义工具执行时的权限检查上下文

该文件是整个权限系统的类型基础，被 25+ 文件直接导入，间接影响所有工具的执行流程。

## 功能点目的

### 1. 权限模式分层
- **ExternalPermissionMode**: 用户可直接设置的模式（acceptEdits, bypassPermissions, default, dontAsk, plan）
- **InternalPermissionMode**: 内部使用的扩展模式（增加 auto, bubble）
- **PermissionMode**: 完整的模式联合类型

运行时验证使用 `INTERNAL_PERMISSION_MODES` 数组，支持特性开关控制（如 `TRANSCRIPT_CLASSIFIER` 特性控制 auto 模式的可用性）。

### 2. 权限规则系统
- **PermissionRuleSource**: 规则来源（userSettings, projectSettings, localSettings, flagSettings, policySettings, cliArg, command, session）
- **PermissionRuleValue**: 规则值（toolName + 可选的 ruleContent）
- **PermissionRule**: 完整规则（source + behavior + value）

### 3. 权限更新操作
支持六种更新类型：
- `addRules`: 添加规则
- `replaceRules`: 替换规则
- `removeRules`: 删除规则
- `setMode`: 设置模式
- `addDirectories`: 添加额外工作目录
- `removeDirectories`: 删除额外工作目录

### 4. 权限决策结果
完整的决策类型体系：
- **PermissionAllowDecision**: 允许执行，可携带 updatedInput（修改后的输入）
- **PermissionAskDecision**: 询问用户，包含消息、建议、阻塞路径等
- **PermissionDenyDecision**: 拒绝执行
- **PermissionResult**: 联合类型，增加 passthrough 行为

### 5. 决策理由追踪（PermissionDecisionReason）
支持 10+ 种决策来源：
- `rule`: 匹配到具体规则
- `mode`: 由当前模式决定
- `subcommandResults`: 子命令结果聚合
- `permissionPromptTool`: 权限提示工具的结果
- `hook`: Hook 干预
- `asyncAgent`: 异步 Agent 决策
- `sandboxOverride`: 沙箱覆盖
- `classifier`: AI 分类器决策
- `workingDir`: 工作目录检查
- `safetyCheck`: 安全检查
- `other`: 其他原因

### 6. Bash 分类器类型
- **ClassifierResult**: 分类结果（匹配、置信度、理由）
- **ClassifierBehavior**: 分类器行为（deny/ask/allow）
- **YoloClassifierResult**: YOLO 分类器完整结果（包含 token 使用、延迟、两阶段执行等）

### 7. 权限解释器类型
- **RiskLevel**: 风险等级（LOW/MEDIUM/HIGH）
- **PermissionExplanation**: 权限解释（风险等级、解释、推理、风险描述）

## 具体技术实现

### 关键数据结构

```typescript
// 权限模式
const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',      // 自动接受编辑类操作
  'bypassPermissions', // 完全绕过权限检查（危险）
  'default',          // 默认模式
  'dontAsk',          // 不询问（自动允许安全操作）
  'plan',             // 计划模式
] as const

type ExternalPermissionMode = typeof EXTERNAL_PERMISSION_MODES[number]
type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'

// 运行时验证数组（支持特性开关）
const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? ['auto'] : []),
] as const
```

### 权限决策类型

```typescript
// 允许决策
interface PermissionAllowDecision<Input = Record<string, unknown>> {
  behavior: 'allow'
  updatedInput?: Input        // Hook/分类器修改后的输入
  userModified?: boolean      // 用户是否修改了输入
  decisionReason?: PermissionDecisionReason
  toolUseID?: string
  acceptFeedback?: string     // 用户反馈
  contentBlocks?: ContentBlockParam[]  // 附加内容块
}

// 询问决策
interface PermissionAskDecision<Input = Record<string, unknown>> {
  behavior: 'ask'
  message: string
  updatedInput?: Input
  decisionReason?: PermissionDecisionReason
  suggestions?: PermissionUpdate[]  // 建议的权限更新
  blockedPath?: string              // 被阻塞的路径
  metadata?: PermissionMetadata     // 命令元数据
  isBashSecurityCheckForMisparsing?: boolean  // Bash 安全标记
  pendingClassifierCheck?: PendingClassifierCheck  // 待处理的分类器检查
  contentBlocks?: ContentBlockParam[]
}

// 拒绝决策
interface PermissionDenyDecision {
  behavior: 'deny'
  message: string
  decisionReason: PermissionDecisionReason
  toolUseID?: string
}

// 完整结果（增加 passthrough）
type PermissionResult<Input = Record<string, unknown>> =
  | PermissionDecision<Input>
  | {
      behavior: 'passthrough'
      message: string
      decisionReason?: PermissionDecisionReason['decisionReason']
      suggestions?: PermissionUpdate[]
      blockedPath?: string
      pendingClassifierCheck?: PendingClassifierCheck
    }
```

### 决策理由类型

```typescript
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'other'; reason: string }
```

### YOLO 分类器结果

```typescript
interface YoloClassifierResult {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean   // 上下文超出窗口
  model: string
  usage?: ClassifierUsage       // Token 使用统计
  durationMs?: number
  promptLengths?: {
    systemPrompt: number
    toolCalls: number
    userPrompts: number
  }
  errorDumpPath?: string        // 错误提示转储路径
  stage?: 'fast' | 'thinking'   // 两阶段分类器的阶段
  stage1Usage?: ClassifierUsage
  stage1DurationMs?: number
  stage1RequestId?: string      // API request_id（用于服务端关联）
  stage1MsgId?: string          // API message id（用于分析关联）
  stage2Usage?: ClassifierUsage
  stage2DurationMs?: number
  stage2RequestId?: string
  stage2MsgId?: string
}
```

### 工具权限上下文

```typescript
interface ToolPermissionContext {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource
  readonly shouldAvoidPermissionPrompts?: boolean
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  readonly prePlanMode?: PermissionMode
}
```

## 关键代码路径与文件引用

### 类型定义
- `src/types/permissions.ts` - 本文件，核心类型定义

### 权限实现
- `src/utils/permissions/PermissionResult.ts` - 结果类型重导出和工具函数
- `src/utils/permissions/PermissionRule.ts` - 规则 Schema 定义
- `src/utils/permissions/PermissionUpdate.ts` - 权限更新逻辑
- `src/utils/permissions/PermissionUpdateSchema.ts` - 更新 Schema
- `src/utils/permissions/PermissionMode.ts` - 模式管理
- `src/utils/permissions/yoloClassifier.ts` - YOLO 分类器实现

### 工具权限检查
- `src/tools/BashTool/bashPermissions.ts` - Bash 工具权限
- `src/tools/PowerShellTool/powershellPermissions.ts` - PowerShell 权限
- `src/tools/PowerShellTool/pathValidation.ts` - 路径验证
- `src/tools/WebFetchTool/WebFetchTool.ts` - Web 获取权限

### UI 组件
- `src/components/PromptInput/PromptInput.tsx` - 提示输入（权限模式显示）
- `src/components/sandbox/SandboxSettings.tsx` - 沙箱设置
- `src/components/sandbox/SandboxOverridesTab.tsx` - 沙箱覆盖

### Hook 集成
- `src/hooks/toolPermission/PermissionContext.ts` - 权限上下文
- `src/hooks/toolPermission/handlers/coordinatorHandler.ts` - 协调器处理
- `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` - Swarm 工作器处理

### 其他使用者
- `src/cli/print.ts` - CLI 输出
- `src/Tool.ts` - 工具基类
- `src/services/tools/toolHooks.ts` - 工具 Hook
- `src/utils/conversationRecovery.ts` - 对话恢复
- `src/utils/processUserInput/processTextPrompt.ts` - 文本提示处理
- `src/utils/processUserInput/processUserInput.ts` - 用户输入处理
- `src/utils/messages.ts` - 消息处理
- `src/utils/swarm/inProcessRunner.ts` - Swarm 运行器
- `src/utils/teleport.tsx` - 传送功能

## 依赖与外部交互

### 导入依赖
```typescript
import { feature } from 'bun:bundle'                                    // 特性开关
import type { ContentBlockParam } from '@anthropic-ai/sdk/resources/messages.mjs'  // Anthropic SDK
```

### 被依赖方（25+ 文件）
主要分布：
- 权限系统实现（`src/utils/permissions/*`）
- 工具实现（`src/tools/*`）
- UI 组件（`src/components/*`）
- 输入处理（`src/utils/processUserInput/*`）

## 风险、边界与改进建议

### 潜在风险

1. **特性开关与类型不同步**
   - `INTERNAL_PERMISSION_MODES` 使用运行时 `feature()` 检查
   - 但 TypeScript 类型在编译时确定
   - 可能导致类型系统认为 `auto` 可用但实际运行时不可用

2. **泛型默认参数的风险**
   ```typescript
   PermissionDecision<Input = Record<string, unknown>>
   ```
   默认参数可能导致类型推断过于宽松，丢失具体输入类型。

3. **PermissionResult 的 passthrough 行为**
   - passthrough 不是真正的决策，而是"传递处理"
   - 但类型上与 allow/ask/deny 并列，容易混淆

4. **decisionReason 的可选性不一致**
   - `PermissionAllowDecision` 的 `decisionReason` 是可选的
   - `PermissionDenyDecision` 的 `decisionReason` 是必需的
   - 这种不一致增加了使用复杂度

### 边界情况

1. **classifierApprovable 标记**
   ```typescript
   // safetyCheck 决策理由包含 classifierApprovable 标记
   // true: 敏感文件路径，分类器可以评估
   // false: Windows 路径绕过或跨机器桥接消息
   ```

2. **pendingClassifierCheck**
   - 用于非阻塞式分类器检查
   - 允许在分类器评估的同时显示权限提示
   - 分类器结果可能先于用户响应到达

3. **两阶段分类器**
   - `stage` 字段标识 fast/thinking 阶段
   - 第一阶段快速筛选，第二阶段深度分析
   - 需要关联两个阶段的 request_id 用于服务端分析

4. **transcriptTooLong 处理**
   - 分类器上下文超出窗口时设置
   - 调用方应回退到正常提示而非重试
   - 确定性错误（相同输入总是失败）

### 改进建议

1. **特性开关类型安全**
   ```typescript
   // 建议：使用条件类型
   type AvailableModes = typeof EXTERNAL_PERMISSION_MODES[number]
   type FeatureModes = typeof FEATURE_FLAGS['TRANSCRIPT_CLASSIFIER'] extends true 
     ? 'auto' 
     : never
   type PermissionMode = AvailableModes | FeatureModes | 'bubble'
   ```

2. **决策类型重构**
   ```typescript
   // 建议：分离 passthrough
   type PermissionDecision<Input> = AllowDecision<Input> | AskDecision<Input> | DenyDecision
   type PermissionHandlingResult<Input> = 
     | { type: 'decision'; decision: PermissionDecision<Input> }
     | { type: 'passthrough'; message: string; /* ... */ }
   ```

3. **决策理由标准化**
   - 统一 `decisionReason` 的可选性
   - 为每种理由类型增加版本字段，支持演进

4. **分类器结果优化**
   ```typescript
   // 建议：提取公共接口
   interface ClassifierStageResult {
     usage: ClassifierUsage
     durationMs: number
     requestId: string
     msgId: string
   }
   ```

5. **工具权限上下文精简**
   - 当前 10+ 个字段，部分字段用途重叠
   - 建议按职责分组（如分离规则上下文和模式上下文）

6. **文档完善**
   - 增加权限决策流程图
   - 说明各模式的适用场景和转换规则
   - 提供自定义权限规则的示例
