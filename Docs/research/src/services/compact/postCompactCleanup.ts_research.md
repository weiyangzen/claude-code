# postCompactCleanup.ts 深度研究文档

## 场景与职责

`postCompactCleanup.ts` 是 Claude Code 压缩(compact)机制的核心清理模块，负责在对话压缩完成后执行一系列状态清理操作。该模块解决了压缩操作后各种缓存和追踪状态可能失效的问题，确保系统状态与压缩后的对话历史保持一致。

**核心场景：**
1. **自动压缩(auto-compact)** 后的清理 - 当对话token数超过阈值时自动触发
2. **手动压缩(manual /compact)** 后的清理 - 用户主动执行 `/compact` 命令
3. **Session Memory 压缩后的清理** - 使用 session memory 替代传统压缩时的清理

**关键设计决策：**
- 该模块**不**清理 `invoked_skill` 内容，因为 skill 内容需要在多次压缩间保持，以便 `createSkillAttachmentIfNeeded()` 能在后续压缩附件中包含完整的 skill 文本
- 区分主线程(main thread)和子代理(subagent)的清理范围，避免子代理压缩时破坏主线程状态

## 功能点目的

### 1. 主线程状态清理 (Main Thread State Cleanup)
- **目的**：仅在主线程压缩时重置模块级状态，避免子代理压缩破坏主线程状态
- **实现**：通过 `querySource` 参数判断是否为子代理 (`agent:*`)，子代理与主线程共享进程和模块级状态

### 2. 微压缩状态重置 (Microcompact State Reset)
- **目的**：重置微压缩相关的追踪状态
- **调用**：`resetMicrocompactState()`

### 3. 上下文崩溃重置 (Context Collapse Reset)
- **目的**：当 `CONTEXT_COLLAPSE` 特性启用时，重置上下文崩溃模块的状态
- **条件**：仅主线程压缩时执行，使用动态 require 避免循环依赖

### 4. 用户上下文缓存清理 (User Context Cache Clear)
- **目的**：清理 `getUserContext` 的 memoized 缓存
- **重要性**：如果只清理内部的 `getMemoryFiles` 缓存，下次请求会命中 `getUserContext` 缓存而不会触发 `InstructionsLoaded` hook

### 5. 内存文件缓存重置 (Memory Files Cache Reset)
- **目的**：重置 `getMemoryFiles` 的 one-shot hook 标志
- **调用**：`resetGetMemoryFilesCache('compact')`

### 6. 系统提示区段清理 (System Prompt Sections Clear)
- **目的**：清理系统提示的各个区段缓存
- **调用**：`clearSystemPromptSections()`

### 7. 分类器批准清理 (Classifier Approvals Clear)
- **目的**：清理 bash 分类器和自动模式分类器的批准记录
- **调用**：`clearClassifierApprovals()`

### 8. 推测性检查清理 (Speculative Checks Clear)
- **目的**：清理 bash 权限的推测性检查状态
- **调用**：`clearSpeculativeChecks()`

### 9. Beta 追踪状态清理 (Beta Tracing State Clear)
- **目的**：清理 beta session tracing 的哈希追踪状态
- **调用**：`clearBetaTracingState()`

### 10. 归属钩子文件内容缓存清理 (Attribution Hooks Cache Sweep)
- **目的**：当 `COMMIT_ATTRIBUTION` 特性启用时，清理文件内容缓存
- **实现**：动态导入 `attributionHooks.js` 并调用 `sweepFileContentCache()`

### 11. Session Messages 缓存清理 (Session Messages Cache Clear)
- **目的**：清理 session storage 中的消息缓存
- **调用**：`clearSessionMessagesCache()`

## 具体技术实现

### 核心函数签名

```typescript
export function runPostCompactCleanup(querySource?: QuerySource): void
```

**参数说明：**
- `querySource`: 压缩请求的源标识，用于区分主线程和子代理
  - `undefined`: 主线程（向后兼容）
  - `'repl_main_thread'`: 主线程
  - `'sdk'`: SDK 模式
  - `'agent:*'`: 子代理

### 主线程检测逻辑

```typescript
const isMainThreadCompact =
  querySource === undefined ||
  querySource.startsWith('repl_main_thread') ||
  querySource === 'sdk'
```

### 清理执行顺序

1. **无条件执行**：`resetMicrocompactState()`
2. **条件执行 (CONTEXT_COLLAPSE + 主线程)**：`resetContextCollapse()`
3. **条件执行 (主线程)**：`getUserContext.cache.clear()` 和 `resetGetMemoryFilesCache('compact')`
4. **无条件执行**：
   - `clearSystemPromptSections()`
   - `clearClassifierApprovals()`
   - `clearSpeculativeChecks()`
   - `clearBetaTracingState()`
   - `clearSessionMessagesCache()`
5. **条件执行 (COMMIT_ATTRIBUTION)**：`sweepFileContentCache()`

### 动态导入模式

为避免循环依赖和特性标志检查，使用动态 require：

```typescript
if (feature('CONTEXT_COLLAPSE')) {
  if (isMainThreadCompact) {
    ;(
      require('../contextCollapse/index.js') as typeof import('../contextCollapse/index.js')
    ).resetContextCollapse()
  }
}

if (feature('COMMIT_ATTRIBUTION')) {
  void import('../../utils/attributionHooks.js').then(m =>
    m.sweepFileContentCache(),
  )
}
```

## 关键代码路径与文件引用

### 调用方 (Callers)

| 文件路径 | 调用场景 | 参数传递 |
|---------|---------|---------|
| `src/services/compact/autoCompact.ts:297` | 自动压缩成功后 | `querySource` |
| `src/services/compact/autoCompact.ts:326` | 传统压缩成功后 | `querySource` |
| `src/commands/compact/compact.ts:64` | 手动 `/compact` 命令 (session memory 路径) | 无参数 |
| `src/commands/compact/compact.ts:118` | 手动 `/compact` 命令 (传统路径) | 无参数 |
| `src/commands/compact/compact.ts:201` | 响应式压缩路径 | 无参数 |
| `src/screens/REPL.tsx` | REPL 中的压缩处理 | `querySource` |
| `src/utils/hooks.ts` | Hook 执行后的清理 | 无参数 |
| `src/commands/clear/caches.ts:74` | `/clear` 命令 | 无参数 |

### 被调用方 (Callees)

| 被调用函数 | 来源文件 | 用途 |
|-----------|---------|------|
| `resetMicrocompactState()` | `microCompact.ts` | 重置微压缩状态 |
| `resetContextCollapse()` | `contextCollapse/index.ts` | 重置上下文崩溃状态 |
| `getUserContext.cache.clear()` | `context.ts` | 清理用户上下文缓存 |
| `resetGetMemoryFilesCache()` | `claudemd.ts` | 重置内存文件缓存 |
| `clearSystemPromptSections()` | `systemPromptSections.ts` | 清理系统提示区段 |
| `clearClassifierApprovals()` | `classifierApprovals.ts` | 清理分类器批准 |
| `clearSpeculativeChecks()` | `bashPermissions.ts` | 清理推测性检查 |
| `clearBetaTracingState()` | `betaSessionTracing.ts` | 清理 beta 追踪状态 |
| `sweepFileContentCache()` | `attributionHooks.ts` | 清理归属缓存 |
| `clearSessionMessagesCache()` | `sessionStorage.ts` | 清理 session 消息缓存 |

## 依赖与外部交互

### 导入依赖

```typescript
import { feature } from 'bun:bundle'
import type { QuerySource } from '../../constants/querySource.js'
import { clearSystemPromptSections } from '../../constants/systemPromptSections.js'
import { getUserContext } from '../../context.js'
import { clearSpeculativeChecks } from '../../tools/BashTool/bashPermissions.js'
import { clearClassifierApprovals } from '../../utils/classifierApprovals.js'
import { resetGetMemoryFilesCache } from '../../utils/claudemd.js'
import { clearSessionMessagesCache } from '../../utils/sessionStorage.js'
import { clearBetaTracingState } from '../../utils/telemetry/betaSessionTracing.js'
import { resetMicrocompactState } from './microCompact.js'
```

### 特性标志依赖

| 特性标志 | 用途 |
|---------|------|
| `CONTEXT_COLLAPSE` | 控制上下文崩溃重置 |
| `COMMIT_ATTRIBUTION` | 控制归属钩子缓存清理 |

### 配置/环境变量

- 无直接依赖，通过 `querySource` 参数控制行为

## 风险、边界与改进建议

### 已知风险

1. **子代理状态污染风险**
   - 子代理与主线程共享模块级状态
   - 如果子代理压缩时错误地重置了主线程状态，会导致状态不一致
   - 缓解：`isMainThreadCompact` 检查确保仅在主线程时重置关键状态

2. **Skill 内容保留的副作用**
   - 故意不调用 `resetSentSkillNames()` 以保留 skill 内容
   - 这可能导致旧的 skill 引用在新压缩后的对话中仍然存在
   - 缓解：`skillChangeDetector` 和 `cacheUtils` 重置处理动态添加

3. **动态导入失败风险**
   - `attributionHooks.js` 的动态导入失败时静默处理（`void`）
   - 如果清理失败可能导致内存泄漏

### 边界情况

1. **QuerySource 为 undefined**
   - 向后兼容处理，视为 main thread
   - 仅用于 `/compact` 和 `/clear` 等主线程专用调用点

2. **重复调用**
   - 函数设计为幂等，多次调用不会导致问题
   - 但会重复执行清理操作，可能影响性能

3. **并发压缩**
   - 如果多个压缩同时触发（理论上不应发生），清理可能交错执行
   - 依赖调用方的同步保证

### 改进建议

1. **添加日志记录**
   ```typescript
   // 建议添加调试日志
   logForDebugging(`Post-compact cleanup: isMainThread=${isMainThreadCompact}, querySource=${querySource}`)
   ```

2. **错误处理增强**
   - 当前动态导入失败静默处理，建议添加错误日志
   - 考虑使用 `Promise.catch` 记录导入失败

3. **类型安全增强**
   - `querySource` 的类型检查可以更严格
   - 考虑使用字面量联合类型替代 `string`

4. **测试覆盖**
   - 添加单元测试验证主线程/子代理的清理范围差异
   - 测试各种特性标志组合下的行为

5. **文档化调用约定**
   - 明确文档化哪些调用点应该传递 `querySource`
   - 添加 lint 规则确保新调用点正确传递参数
