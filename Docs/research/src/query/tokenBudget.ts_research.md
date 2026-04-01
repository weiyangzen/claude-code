# src/query/tokenBudget.ts 研究文档

## 场景与职责

`tokenBudget.ts` 是 Claude Code 查询模块的 Token 预算管理器，实现基于用户指定预算的自动对话延续机制。当用户在提示中使用类似 "+500k" 或 "use 2M tokens" 的语法时，系统会追踪已使用的 token 并在达到预算前自动继续对话。

### 核心设计原则
1. **预算追踪**: 追踪每轮对话的 token 使用量和延续次数
2. **收益递减检测**: 当连续多轮 token 增量低于阈值时，认为达到收益递减点，停止自动延续
3. **阈值控制**: 使用完成阈值（90%）和收益递减阈值（500 tokens）控制延续行为
4. **非代理限制**: 预算功能不适用于子代理（agentId 存在时直接返回 stop）

## 功能点目的

### BudgetTracker 类型
追踪预算使用状态：
- `continuationCount`: 自动延续次数
- `lastDeltaTokens`: 上次检查的 token 增量
- `lastGlobalTurnTokens`: 上次检查时的累计 token 数
- `startedAt`: 追踪开始时间戳

### checkTokenBudget 函数
核心决策函数，决定是继续对话还是停止：
- **ContinueDecision**: 继续对话，包含提示消息
- **StopDecision**: 停止对话，可选包含完成事件详情

### createBudgetTracker 函数
创建新的预算追踪器实例，初始化所有计数器。

## 具体技术实现

### 关键流程

```typescript
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // 1. 子代理或无预算时直接停止
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  // 2. 计算当前百分比和增量
  const turnTokens = globalTurnTokens
  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 3. 检测收益递减
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // 4. 未达阈值且非收益递减时继续
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++
    tracker.lastDeltaTokens = deltaSinceLastCheck
    tracker.lastGlobalTurnTokens = globalTurnTokens
    return {
      action: 'continue',
      nudgeMessage: getBudgetContinuationMessage(pct, turnTokens, budget),
      ...
    }
  }

  // 5. 收益递减或已延续过时停止
  if (isDiminishing || tracker.continuationCount > 0) {
    return {
      action: 'stop',
      completionEvent: { ..., diminishingReturns: isDiminishing, ... }
    }
  }

  return { action: 'stop', completionEvent: null }
}
```

### 数据结构

```typescript
export type BudgetTracker = {
  continuationCount: number
  lastDeltaTokens: number
  lastGlobalTurnTokens: number
  startedAt: number
}

type ContinueDecision = {
  action: 'continue'
  nudgeMessage: string
  continuationCount: number
  pct: number
  turnTokens: number
  budget: number
}

type StopDecision = {
  action: 'stop'
  completionEvent: {
    continuationCount: number
    pct: number
    turnTokens: number
    budget: number
    diminishingReturns: boolean
    durationMs: number
  } | null
}

export type TokenBudgetDecision = ContinueDecision | StopDecision
```

### 常量定义

```typescript
const COMPLETION_THRESHOLD = 0.9      // 90% 预算完成阈值
const DIMINISHING_THRESHOLD = 500     // 收益递减检测阈值（tokens）
```

### 依赖模块

| 依赖 | 用途 |
|------|------|
| `../utils/tokenBudget.js` | `getBudgetContinuationMessage()` 生成延续提示消息 |

## 关键代码路径与文件引用

### 调用方
- `src/query.ts:111` - 导入函数
  ```typescript
  import { createBudgetTracker, checkTokenBudget } from './query/tokenBudget.js'
  ```

- `src/query.ts:280` - 创建预算追踪器（feature-gated）
  ```typescript
  const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
  ```

- `src/query.ts:1308-1354` - 检查预算并决定是否继续
  ```typescript
  if (feature('TOKEN_BUDGET')) {
    const decision = checkTokenBudget(
      budgetTracker!,
      toolUseContext.agentId,
      getCurrentTurnTokenBudget(),
      getTurnOutputTokens(),
    )

    if (decision.action === 'continue') {
      incrementBudgetContinuationCount()
      // ... 创建延续状态并继续循环
    }

    if (decision.completionEvent) {
      // ... 记录完成事件
    }
  }
  ```

### 上游状态管理（bootstrap/state.ts）
- `getCurrentTurnTokenBudget()`: 获取当前轮次的预算限制
- `getTurnOutputTokens()`: 获取当前轮次已使用的输出 token 数
- `incrementBudgetContinuationCount()`: 增加延续计数
- `snapshotOutputTokensForTurn()`: 在轮次开始时快照 token 数

### 消息生成（utils/tokenBudget.ts）
- `getBudgetContinuationMessage()`: 生成给模型的提示消息
  ```typescript
  export function getBudgetContinuationMessage(
    pct: number,
    turnTokens: number,
    budget: number,
  ): string {
    const fmt = (n: number): string => new Intl.NumberFormat('en-US').format(n)
    return `Stopped at ${pct}% of token target (${fmt(turnTokens)} / ${fmt(budget)}). Keep working — do not summarize.`
  }
  ```

## 依赖与外部交互

### 上游依赖
1. **utils/tokenBudget.ts**: 提供消息生成和预算解析功能
2. **bootstrap/state.ts**: 提供 token 计数和预算状态管理

### 下游消费
1. **query.ts**: 主查询循环根据决策结果控制对话流程

### 预算语法解析（utils/tokenBudget.ts）
```typescript
// 支持的预算语法
"+500k"                    // 简写，开头或结尾
"use 2M tokens"           // 详细语法
"spend 1.5b tokens"       // 支持 k/m/b 后缀
```

## 风险、边界与改进建议

### 风险点
1. **精度问题**: 使用 `Math.round()` 进行百分比计算，在极大预算时可能有精度损失
2. **收益递减误判**: 固定 500 tokens 阈值可能不适用于所有场景
3. **无预算上限**: 用户可指定任意大的预算（包括 billions），可能导致意外的大量 API 调用

### 边界情况
1. **子代理禁用**: `agentId` 存在时预算功能完全禁用，防止子代理意外消耗大量 token
2. **零或负预算**: 视为无预算，直接返回 stop
3. **首次检查**: `continuationCount === 0` 且未达阈值时直接停止（无延续历史）

### 改进建议
1. **预算上限**: 添加合理的预算上限（如 10M tokens），防止用户误输入导致巨额费用
2. **动态阈值**: 考虑根据预算大小动态调整收益递减阈值
3. **用户确认**: 在超大预算或大量延续时添加用户确认提示
4. **预算进度**: 添加实时预算使用进度显示（如进度条或百分比）
5. **预算持久化**: 考虑在会话恢复时保留预算状态

### 代码质量观察
1. **简洁性**: 93 行代码，职责单一，逻辑清晰
2. **纯函数**: `checkTokenBudget` 是纯函数，易于测试
3. **类型安全**: 使用联合类型区分 Continue/Stop 决策
4. **文档**: 注释清晰解释了设计意图和阈值选择原因
