# src/query/config.ts 研究文档

## 场景与职责

`config.ts` 是 Claude Code 查询模块的配置中心，负责在每次 `query()` 调用入口点一次性快照化不可变配置。该设计将配置与可变的 State 结构以及 ToolUseContext 分离，使得未来的 `step()` 提取成为可能——纯 reducer 可以接收 `(state, event, config)` 形式的参数，其中 config 是纯数据。

### 核心设计原则
1. **不可变性**: 配置值在 `query()` 入口点一次性捕获，避免在循环迭代中重复读取可能变化的配置
2. **与 feature() 门控分离**: 有意排除 `feature()` 门控，因为这些是 tree-shaking 边界，必须保持在受保护块内以实现死代码消除
3. **懒加载优化**: 避免引入重型模块依赖（如 fastMode.ts 及其 axios、settings、auth 等依赖树）到测试分片

## 功能点目的

### QueryConfig 类型
定义查询所需的运行时配置结构：
- `sessionId`: 当前会话唯一标识，用于追踪和日志
- `gates`: 运行时功能门控（Statsig/环境变量控制）
  - `streamingToolExecution`: 是否启用流式工具执行
  - `emitToolUseSummaries`: 是否发送工具使用摘要
  - `isAnt`: 是否为内部 Ant 用户（启用调试功能）
  - `fastModeEnabled`: 快速模式是否启用

### buildQueryConfig() 函数
构建 QueryConfig 实例，从以下源读取配置：
- `getSessionId()`: 从 bootstrap state 获取会话ID
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`: 从 GrowthBook/Statsig 获取功能开关（接受可能过时的缓存值）
- `isEnvTruthy()`: 读取环境变量布尔值
- `process.env.USER_TYPE`: 用户类型判断

## 具体技术实现

### 关键流程

```typescript
export function buildQueryConfig(): QueryConfig {
  return {
    sessionId: getSessionId(),
    gates: {
      streamingToolExecution: checkStatsigFeatureGate_CACHED_MAY_BE_STALE(
        'tengu_streaming_tool_execution2',
      ),
      emitToolUseSummaries: isEnvTruthy(
        process.env.CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES,
      ),
      isAnt: process.env.USER_TYPE === 'ant',
      fastModeEnabled: !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE),
    },
  }
}
```

### 数据结构

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

### 依赖模块

| 依赖 | 用途 |
|------|------|
| `../bootstrap/state.js` | `getSessionId()` 获取会话ID |
| `../services/analytics/growthbook.js` | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` 功能门控查询 |
| `../types/ids.js` | `SessionId` 类型定义 |
| `../utils/envUtils.js` | `isEnvTruthy()` 环境变量解析 |

## 关键代码路径与文件引用

### 调用方
- `src/query.ts:295` - `queryLoop()` 中调用 `buildQueryConfig()` 创建配置快照
  ```typescript
  const config = buildQueryConfig()
  ```

### 配置使用点
- `src/query.ts:561` - 流式工具执行门控检查
  ```typescript
  const useStreamingToolExecution = config.gates.streamingToolExecution
  ```
- `src/query.ts:588` - Ant 用户调试功能（dumpPromptsFetch）
  ```typescript
  const dumpPromptsFetch = config.gates.isAnt
    ? createDumpPromptsFetch(toolUseContext.agentId ?? config.sessionId)
    : undefined
  ```
- `src/query.ts:671` - 快速模式参数传递
  ```typescript
  ...(config.gates.fastModeEnabled && {
    fastMode: appState.fastMode,
  }),
  ```
- `src/query.ts:1416` - 工具使用摘要生成门控
  ```typescript
  if (config.gates.emitToolUseSummaries && ...)
  ```

## 依赖与外部交互

### 上游依赖
1. **bootstrap/state.ts**: 提供会话状态管理
2. **services/analytics/growthbook.ts**: 提供功能开关服务
3. **utils/envUtils.ts**: 环境变量工具

### 下游消费
1. **query.ts**: 主查询循环消费配置进行运行时决策

### 环境变量
| 变量名 | 作用 |
|--------|------|
| `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` | 控制工具摘要生成 |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | 禁用快速模式 |
| `USER_TYPE` | 用户类型标识（ant/3p） |

## 风险、边界与改进建议

### 风险点
1. **缓存过期**: `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` 明确接受可能过时的值，这在长会话中可能导致功能开关不一致
2. **配置漂移**: 配置在 `query()` 入口点快照化，如果在长会话中门控值发生变化，需要重启才能生效

### 边界情况
1. **测试隔离**: 该文件有意避免引入重型模块（如 fastMode.ts），确保测试分片不加载不必要的依赖
2. **Tree-shaking**: `feature()` 门控不放在 config 中，确保外部构建可以正确消除死代码

### 改进建议
1. **动态重载**: 考虑为长会话添加配置刷新机制，或明确文档化配置快照的行为
2. **配置验证**: 添加配置值的运行时验证，确保环境变量解析符合预期
3. **文档化门控**: 建议将各门控的默认值和优先级（环境变量 vs Statsig）文档化
