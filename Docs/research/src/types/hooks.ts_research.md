# hooks.ts 研究文档

## 场景与职责

`src/types/hooks.ts` 是 Claude Code CLI 的 Hook 系统核心类型定义文件。Hook 系统允许用户在特定生命周期事件点执行自定义脚本，实现：

1. **事件拦截与处理**: PreToolUse, PostToolUse, PermissionDenied 等工具生命周期事件
2. **会话生命周期管理**: SessionStart, SessionEnd, Setup, SubagentStart 等
3. **权限控制增强**: PermissionRequest, PermissionDenied 等安全相关事件
4. **用户交互扩展**: Elicitation（提示征询）, UserPromptSubmit 等
5. **文件监控**: FileChanged, CwdChanged 等工作目录变更事件

该文件定义了 Hook 的输入输出协议、响应 Schema、回调接口等完整类型体系。

## 功能点目的

### 1. Hook 事件类型系统
定义完整的 Hook 事件枚举（实际定义在 `src/entrypoints/agentSdkTypes.js`），包括：
- **工具生命周期**: PreToolUse, PostToolUse, PostToolUseFailure
- **会话管理**: SessionStart, SessionEnd, Setup, SubagentStart, SubagentStop
- **权限控制**: PermissionRequest, PermissionDenied
- **用户交互**: UserPromptSubmit, Elicitation, ElicitationResult
- **任务管理**: TaskCreated, TaskCompleted, TeammateIdle
- **文件监控**: FileChanged, CwdChanged
- **其他**: Notification, Stop, StopFailure, ConfigChange, InstructionsLoaded

### 2. Hook 响应协议（同步 vs 异步）
- **同步响应**: 直接返回处理结果，阻塞后续执行
- **异步响应**: 返回 `{ async: true }`，后台执行不阻塞
- **超时控制**: 支持 `asyncTimeout` 配置异步超时时间

### 3. 提示征询协议（Prompt Elicitation）
允许 Hook 向用户展示选项并获取选择：
```typescript
PromptRequest: { prompt: string; message: string; options: Array<{key, label, description}> }
PromptResponse: { prompt_response: string; selected: string }
```

### 4. 权限决策集成
Hook 可在 PreToolUse 和 PermissionRequest 事件中：
- 返回 `allow`/`deny`/`ask` 决策
- 提供决策理由
- 修改工具输入参数（updatedInput）

### 5. 类型安全机制
- 使用 Zod Schema 运行时验证 Hook 输出
- TypeScript 编译时类型检查与 Zod 运行时验证双重保障
- `type-fest` 的 `IsEqual` 确保 SDK 类型与 Zod 类型一致

## 具体技术实现

### 关键数据结构

```typescript
// 同步 Hook 响应 Schema
interface SyncHookJSONOutput {
  continue?: boolean           // 是否继续执行（默认 true）
  suppressOutput?: boolean     // 是否隐藏 stdout
  stopReason?: string          // continue=false 时显示的消息
  decision?: 'approve' | 'block'  // 权限决策
  reason?: string              // 决策解释
  systemMessage?: string       // 显示给用户的警告
  hookSpecificOutput?: {        // 事件特定输出
    hookEventName: HookEvent
    // ... 事件特定字段
  }
}

// 异步 Hook 响应
interface AsyncHookJSONOutput {
  async: true
  asyncTimeout?: number
}

// Hook 回调类型（内存中注册的 Hook）
interface HookCallback {
  type: 'callback'
  callback: (
    input: HookInput,
    toolUseID: string | null,
    abort: AbortSignal | undefined,
    hookIndex?: number,
    context?: HookCallbackContext,
  ) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean  // 是否排除在指标统计外
}

// Hook 执行结果
interface HookResult {
  message?: Message
  systemMessage?: Message
  blockingError?: HookBlockingError
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  // ... 其他结果字段
}

// 聚合 Hook 结果（多个 Hook 合并）
interface AggregatedHookResult {
  blockingErrors?: HookBlockingError[]
  additionalContexts?: string[]
  // ... 聚合字段
}
```

### Zod Schema 实现

```typescript
// 使用 lazySchema 延迟初始化，避免循环依赖
export const hookJSONOutputSchema = lazySchema(() => {
  const asyncHookResponseSchema = z.object({
    async: z.literal(true),
    asyncTimeout: z.number().optional(),
  })
  return z.union([asyncHookResponseSchema, syncHookResponseSchema()])
})

// 编译时类型一致性检查
type _assertSDKTypesMatch = Assert<IsEqual<SchemaHookJSONOutput, HookJSONOutput>>
```

### 关键流程

1. **Hook 执行流程**（`src/utils/hooks.ts:747-1000+`）
   - 构建 Hook 输入（`createBaseHookInput`）
   - 执行命令或回调
   - 解析输出（`parseHookOutput`）
   - 处理同步/异步响应
   - 聚合多个 Hook 结果

2. **权限决策流程**（`src/utils/hooks.ts:489-737`）
   - `processHookJSONOutput` 处理 Hook 返回的 JSON
   - 根据 `hookSpecificOutput.hookEventName` 路由到特定处理逻辑
   - 更新 `permissionBehavior`, `blockingError` 等结果字段

3. **提示征询流程**（`src/utils/hooks.ts:28-47`）
   - Hook 返回 `promptRequestSchema` 格式的请求
   - 系统渲染选项 UI
   - 用户选择后返回 `PromptResponse`

## 关键代码路径与文件引用

### 类型定义
- `src/types/hooks.ts` - 本文件，核心类型定义
- `src/entrypoints/agentSdkTypes.js` - SDK 暴露的 Hook 事件类型

### Hook 执行引擎
- `src/utils/hooks.ts` - 核心 Hook 执行逻辑（~1000+ 行）
- `src/utils/hooks/execAgentHook.ts` - Agent 类型 Hook 执行
- `src/utils/hooks/execPromptHook.ts` - Prompt 类型 Hook 执行
- `src/utils/hooks/execHttpHook.ts` - HTTP 类型 Hook 执行
- `src/utils/hooks/AsyncHookRegistry.ts` - 异步 Hook 注册管理

### Hook 配置与设置
- `src/utils/settings/types.ts` - Hook 配置类型定义
- `src/utils/hooks/hooksSettings.ts` - Hook 设置管理
- `src/utils/hooks/hooksConfigSnapshot.ts` - Hook 配置快照

### 事件触发点
- `src/services/tools/toolHooks.ts` - 工具 Hook 触发
- `src/query/stopHooks.ts` - Stop Hook 触发
- `src/services/tools/toolExecution.ts` - 工具执行 Hook

### UI 组件
- `src/components/hooks/PromptDialog.tsx` - Hook 提示对话框

## 依赖与外部交互

### 导入依赖
```typescript
import { z } from 'zod/v4'                                    // Schema 验证
import { lazySchema } from '../utils/lazySchema.js'           // 延迟 Schema
import { 
  type HookEvent, 
  HOOK_EVENTS, 
  type HookInput,
  type PermissionUpdate,
} from 'src/entrypoints/agentSdkTypes.js'                     // SDK 类型
import type { HookJSONOutput, AsyncHookJSONOutput, SyncHookJSONOutput } from 'src/entrypoints/agentSdkTypes.js'
import type { Message } from 'src/types/message.js'           // 消息类型
import type { PermissionResult } from 'src/utils/permissions/PermissionResult.js'  // 权限结果
import { permissionBehaviorSchema } from 'src/utils/permissions/PermissionRule.js' // 权限行为 Schema
import { permissionUpdateSchema } from 'src/utils/permissions/PermissionUpdateSchema.js' // 权限更新 Schema
import type { AppState } from '../state/AppState.js'          // 应用状态
import type { AttributionState } from '../utils/commitAttribution.js' // 归因状态
import type { IsEqual } from 'type-fest'                      // 类型工具
```

### 被依赖方（约 12 文件导入）
- Hook 执行系统（`src/utils/hooks.ts`）
- CLI 输出（`src/cli/print.ts`）
- 工具执行（`src/services/tools/toolExecution.ts`, `src/services/tools/toolHooks.ts`）
- REPL（`src/screens/REPL.tsx`）
- 停止 Hook（`src/query/stopHooks.ts`）
- 引导状态（`src/bootstrap/state.ts`）

## 风险、边界与改进建议

### 潜在风险

1. **Schema 验证性能**
   - `hookJSONOutputSchema` 是复杂的联合类型，每次 Hook 输出都需验证
   - 高频 Hook（如 PreToolUse）可能产生性能开销

2. **类型一致性维护**
   - `type-fest` 的 `IsEqual` 检查只在编译时生效
   - SDK 类型与 Zod Schema 的同步依赖人工维护

3. **Hook 超时控制**
   - 默认 10 分钟超时（`TOOL_HOOK_EXECUTION_TIMEOUT_MS`）
   - 异步 Hook 的 `asyncTimeout` 与注册超时可能重复计算

4. **权限决策冲突**
   - 多个 Hook 返回不同 `permissionBehavior` 时，聚合逻辑复杂
   - `AggregatedHookResult` 的 `permissionBehavior` 可能无法反映所有 Hook 决策

### 边界情况

1. **Hook 输出解析失败**
   - 非 JSON 输出视为纯文本
   - JSON 验证失败时保留原始输出并记录错误

2. **异步 Hook 状态丢失**
   - 进程重启后异步 Hook 状态无法恢复
   - 依赖 `AsyncHookRegistry` 的内存状态

3. **事件特定字段的灵活性**
   - `hookSpecificOutput` 使用联合类型，但新增事件类型需修改 Schema
   - 扩展性受限，插件难以定义自定义事件

### 改进建议

1. **性能优化**
   ```typescript
   // 建议：缓存已验证的 Hook 输出 Schema
   const validatedHookCache = new WeakMap<HookCallback, z.Schema>()
   ```

2. **类型安全增强**
   ```typescript
   // 建议：为每个 HookEvent 生成专用类型
   type HookOutputFor<T extends HookEvent> = T extends 'PreToolUse' 
     ? PreToolUseHookOutput 
     : T extends 'SessionStart' 
     ? SessionStartHookOutput 
     : GenericHookOutput
   ```

3. **Hook 调试支持**
   - 当前 `HookProgress` 仅包含基础信息
   - 建议增加 Hook 执行时间、输入输出大小等调试字段

4. **错误处理标准化**
   - `validationError` 和 `blockingError` 处理分散
   - 建议统一错误编码和国际化消息支持

5. **文档完善**
   - `HookCallbackContext` 的 `getAppState` 使用场景不明确
   - 建议增加 Hook 开发最佳实践文档
