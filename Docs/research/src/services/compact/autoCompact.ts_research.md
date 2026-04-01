# autoCompact.ts 深度研究文档

## 场景与职责

`autoCompact.ts` 是 Claude Code 自动上下文压缩（Auto-Compact）功能的核心实现模块。它负责在对话上下文接近模型限制时，自动触发上下文压缩流程，避免用户因上下文溢出而中断工作流。

该模块在以下关键位置被调用：
- 主查询循环（`query.ts`）- 每次用户输入后检查是否需要压缩
- Token 警告组件（`TokenWarning.tsx`）- 计算并显示上下文使用状态
- Swarm 运行器（`inProcessRunner.ts`）- 多 Agent 场景下的上下文管理

## 功能点目的

### 1. 自动压缩触发判断 (`shouldAutoCompact`)
- 基于当前消息列表的 token 估算和模型上下文窗口，判断是否需要触发自动压缩
- 支持多种功能开关的协调（DISABLE_COMPACT、DISABLE_AUTO_COMPACT、用户配置等）
- 集成 Session Memory 压缩和 Context Collapse 的互斥逻辑

### 2. 自动压缩执行 (`autoCompactIfNeeded`)
- 执行完整的自动压缩流程，包括 Session Memory 压缩尝试和传统压缩回退
- 实现熔断机制，防止在不可恢复的场景下无限重试
- 管理压缩状态跟踪（连续失败计数、turn ID 等）

### 3. Token 警告状态计算 (`calculateTokenWarningState`)
- 计算上下文使用百分比和剩余空间
- 判断多个阈值状态：警告阈值、错误阈值、自动压缩阈值、阻塞限制

### 4. 有效上下文窗口计算 (`getEffectiveContextWindowSize`)
- 考虑模型输出 token 预留、环境变量覆盖等因素
- 为自动压缩阈值计算提供基础

## 具体技术实现

### 关键常量定义

```typescript
// 缓冲 token 数
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000        // 自动压缩触发缓冲
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000  // 警告阈值缓冲
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000    // 错误阈值缓冲
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000      // 手动压缩阻塞缓冲

// 熔断配置
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3         // 最大连续失败次数

// 摘要输出预留
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000           // 基于 p99.99 的摘要输出
```

### 核心流程

#### 1. 自动压缩触发判断流程

```
shouldAutoCompact(messages, model, querySource, snipTokensFreed)
  │
  ├─> 递归保护检查
  │   ├─ querySource === 'session_memory' → false
  │   ├─ querySource === 'compact' → false
  │   ├─ querySource === 'marble_origami' (ctx-agent) → false
  │   └─ 防止死锁和状态污染
  │
  ├─> 功能开关检查
  │   ├─ DISABLE_COMPACT → false
  │   ├─ DISABLE_AUTO_COMPACT → false
  │   └─ userConfig.autoCompactEnabled → 必须 true
  │
  ├─> 响应式压缩模式检查 (REACTIVE_COMPACT)
  │   └─ 若启用且 GB flag 为 true → false (抑制主动压缩)
  │
  ├─> 上下文折叠模式检查 (CONTEXT_COLLAPSE)
 │   └─ 若启用且 isContextCollapseEnabled() → false
  │       (避免与 collapse 竞争)
  │
  ├─> Token 计算
  │   └─ tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  │
  └─> 阈值比较
      └─ tokenCount >= getAutoCompactThreshold(model) → true/false
```

#### 2. 自动压缩执行流程

```
autoCompactIfNeeded(messages, toolUseContext, cacheSafeParams, querySource, tracking)
  │
  ├─> 全局禁用检查 (DISABLE_COMPACT)
  │
  ├─> 熔断检查
  │   └─ consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES → 跳过
  │
  ├─> 触发判断 (shouldAutoCompact)
  │   └─ false → 返回 { wasCompacted: false }
  │
  ├─> 构建 RecompactionInfo
  │   ├─ isRecompactionInChain: 是否链式压缩
  │   ├─ turnsSincePreviousCompact: 距上次压缩的轮数
  │   ├─ previousCompactTurnId: 上次压缩的 turn ID
  │   └─ autoCompactThreshold: 当前阈值
  │
  ├─> 尝试 Session Memory 压缩
  │   └─ trySessionMemoryCompaction()
  │       ├─ 成功 → 清理状态、返回结果
  │       └─ 失败 → 继续传统压缩
  │
  ├─> 传统压缩 (compactConversation)
  │   ├─ 参数: isAutoCompact=true (抑制用户问题)
  │   ├─ 成功 → 返回结果、重置失败计数
  │   └─ 失败 → 捕获错误、递增失败计数
  │
  └─> 返回结果
```

#### 3. Token 警告状态计算

```typescript
export function calculateTokenWarningState(tokenUsage: number, model: string) {
  const autoCompactThreshold = getAutoCompactThreshold(model)
  const threshold = isAutoCompactEnabled() ? autoCompactThreshold : effectiveWindow
  
  // 计算剩余百分比
  const percentLeft = Math.max(0, Math.round(((threshold - tokenUsage) / threshold) * 100))
  
  // 子阈值计算
  const warningThreshold = threshold - WARNING_THRESHOLD_BUFFER_TOKENS  // threshold - 20K
  const errorThreshold = threshold - ERROR_THRESHOLD_BUFFER_TOKENS      // threshold - 20K
  const blockingLimit = effectiveWindow - MANUAL_COMPACT_BUFFER_TOKENS  // window - 3K
  
  return {
    percentLeft,              // 剩余百分比（用于 UI 显示）
    isAboveWarningThreshold,  // 是否超过警告阈值
    isAboveErrorThreshold,    // 是否超过错误阈值
    isAboveAutoCompactThreshold, // 是否应触发自动压缩
    isAtBlockingLimit,        // 是否达到阻塞限制（禁止手动压缩）
  }
}
```

### 数据结构

```typescript
// 自动压缩跟踪状态
export type AutoCompactTrackingState = {
  compacted: boolean      // 当前链是否已压缩
  turnCounter: number     // 轮数计数器
  turnId: string          // 当前 turn 唯一 ID
  consecutiveFailures?: number  // 连续失败计数（熔断用）
}

// Recompaction 诊断信息
export type RecompactionInfo = {
  isRecompactionInChain: boolean
  turnsSincePreviousCompact: number
  previousCompactTurnId?: string
  autoCompactThreshold: number
  querySource?: QuerySource
}
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/bootstrap/state.js` | `markPostCompaction`, `getSdkBetas` |
| `src/utils/config.js` | `getGlobalConfig` (用户配置) |
| `src/utils/context.js` | `getContextWindowForModel` |
| `src/utils/tokens.js` | `tokenCountWithEstimation` |
| `src/services/api/claude.js` | `getMaxOutputTokensForModel` |
| `src/services/compact/compact.js` | `compactConversation` |
| `src/services/compact/sessionMemoryCompact.js` | `trySessionMemoryCompaction` |
| `src/services/compact/postCompactCleanup.js` | `runPostCompactCleanup` |

### 外部调用方

| 调用方 | 路径 | 调用函数 |
|-------|------|---------|
| `query/deps.ts` | `src/query/deps.ts:3,37` | `autoCompactIfNeeded` |
| `TokenWarning.tsx` | `src/components/TokenWarning.tsx:7` | `calculateTokenWarningState`, `isAutoCompactEnabled` |
| `inProcessRunner.ts` | `src/utils/swarm/inProcessRunner.ts:26,1076` | `getAutoCompactThreshold` |
| `Notifications.tsx` | `src/components/PromptInput/Notifications.tsx` | `isAutoCompactEnabled` |
| `attachments.ts` | `src/utils/attachments.ts` | `isAutoCompactEnabled` |
| `analyzeContext.ts` | `src/utils/analyzeContext.ts` | `shouldAutoCompact` (注释引用) |

### 关键调用链

```
query.ts:mainLoop
  └─> autoCompactIfNeeded (via deps)
      ├─> trySessionMemoryCompaction (优先)
      │   └─> createCompactionResultFromSessionMemory
      │
      └─> compactConversation (回退)
          ├─> streamCompactSummary
          │   ├─> runForkedAgent (缓存共享路径)
          │   └─> queryModelWithStreaming (流式回退)
          │
          └─> createPostCompactFileAttachments
              └─> generateFileAttachment
```

## 依赖与外部交互

### 环境变量

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `DISABLE_COMPACT` | boolean | 完全禁用压缩功能 |
| `DISABLE_AUTO_COMPACT` | boolean | 仅禁用自动压缩（保留手动 /compact） |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | number | 覆盖上下文窗口大小 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | number | 百分比阈值覆盖（测试用） |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | number | 阻塞限制覆盖（测试用） |

### GrowthBook Feature Flags

| Flag | 用途 |
|------|------|
| `REACTIVE_COMPACT` | 响应式压缩模式（抑制主动压缩） |
| `CONTEXT_COLLAPSE` | 上下文折叠模式（与自动压缩互斥） |
| `tengu_cobalt_raccoon` | 响应式压缩的子开关 |

### 用户配置

```typescript
// ~/.claude/config.json
{
  "autoCompactEnabled": true  // 用户级自动压缩开关
}
```

## 风险、边界与改进建议

### 已知风险

1. **熔断机制局限性**: 
   - 仅跟踪连续失败次数，不区分失败原因
   - 建议：区分可重试错误（网络）和不可重试错误（prompt_too_long）

2. **Session Memory 压缩失败回退**:
   - 当 SM 压缩因阈值超限返回 null 时，直接回退到传统压缩
   - 可能在极端场景下导致双重压缩开销

3. **Context Collapse 互斥逻辑复杂**:
   - 通过 `require()` 动态导入避免循环依赖
   - 增加了代码复杂度和测试难度

4. **Snip Token 计算**:
   - `snipTokensFreed` 是粗略估算，可能与实际不符
   - 可能导致压缩时机判断偏差

### 边界条件

| 场景 | 行为 |
|------|------|
| 连续失败 3 次 | 熔断触发，后续跳过自动压缩 |
| `snipTokensFreed > 0` | 从 token 计数中扣除，延迟压缩触发 |
| `querySource='session_memory'` | 完全跳过，防止死锁 |
| `querySource='compact'` | 完全跳过，防止递归 |
| `querySource='marble_origami'` | 跳过（ctx-agent 共享状态） |
| 环境变量覆盖百分比 | 仅在 0-100 范围内生效 |

### 改进建议

1. **智能熔断策略**:
   ```typescript
   // 区分错误类型
   if (isRetryableError(error)) {
     consecutiveFailures++
   } else {
     permanentFailure = true  // 永久跳过
   }
   ```

2. **压缩策略优先级明确化**:
   - 当前：Session Memory → 传统压缩
   - 建议：添加 Reactive Compact 到优先级队列

3. **阈值动态调整**:
   - 基于历史压缩效果动态调整缓冲值
   - 考虑模型特定的输出模式

4. **监控增强**:
   - 添加压缩触发时机与实际需求的偏差指标
   - 跟踪熔断触发频率和恢复情况

5. **代码简化**:
   - 考虑将 Context Collapse 互斥逻辑提取到独立的策略协调器
   - 减少 `require()` 动态导入的使用
