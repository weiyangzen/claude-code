# src/utils/immediateCommand.ts 研究文档

## 场景与职责

`immediateCommand.ts` 是一个极简的功能开关模块，用于控制推理配置相关命令（`/model`、`/fast`、`/effort`）是否应该在当前查询运行期间**立即执行**，而不是等待当前对话回合结束后再应用。

在 Claude Code 的交互模型中，用户可能在模型正在生成回复时输入斜杠命令。某些命令（如切换模型、切换快速模式、调整推理力度）若等到当前回合结束再执行，用户体验会较差。该模块为这些命令提供了一个统一的即时执行判定入口。

调用方包括：
- `src/commands/model/index.ts`
- `src/commands/fast/index.ts`
- `src/commands/effort/index.ts`
- `src/commands/fast/fast.tsx`

## 功能点目的

### `shouldInferenceConfigCommandBeImmediate`
单一导出函数，返回 `boolean`，判定逻辑：

```typescript
export function shouldInferenceConfigCommandBeImmediate(): boolean {
  return (
    process.env.USER_TYPE === 'ant' ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_immediate_model_command', false)
  )
}
```

判定规则：
1. **内部用户（ants）始终启用**：`USER_TYPE === 'ant'` 时无条件返回 `true`。
2. **外部用户受实验控制**：通过 GrowthBook 功能标志 `tengu_immediate_model_command` 控制，默认关闭（`false`）。

## 具体技术实现

### 实现极简性
整个模块仅 15 行代码，没有状态、没有副作用、没有缓存。每次调用都会实时评估环境变量和功能标志。

### GrowthBook 集成
使用 `getFeatureValue_CACHED_MAY_BE_STALE` 读取功能标志。函数名中的 `CACHED_MAY_BE_STALE` 暗示该值可能来自本地缓存，不一定反映 GrowthBook 服务器的最新状态。这对于即时命令判定是可接受的，因为：
- 功能标志切换不需要毫秒级实时性。
- 避免在每次按键或命令输入时发起网络请求。

### 环境变量依赖
`process.env.USER_TYPE` 在启动时由入口点（如 `hfi.tsx`）或构建流程注入，用于区分内部员工（`ant`）和外部用户。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/immediateCommand.ts:10-15` | `shouldInferenceConfigCommandBeImmediate` 判定函数 |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` 功能标志读取 |

## 依赖与外部交互

### 内部依赖
- `../services/analytics/growthbook.js`：`getFeatureValue_CACHED_MAY_BE_STALE`

### 调用方
- `src/commands/model/index.ts`：模型切换命令
- `src/commands/fast/index.ts`：快速模式命令
- `src/commands/fast/fast.tsx`：快速模式命令（TSX 变体）
- `src/commands/effort/index.ts`：推理力度命令

## 风险、边界与改进建议

### 风险与边界
1. **功能标志命名与语义不匹配**：标志名为 `tengu_immediate_model_command`，但实际影响 `/model`、`/fast`、`/effort` 三个命令。命名中的 `model` 可能让人误以为只控制 `/model` 命令。注释已说明这是"inference-config commands"，但标志名未更新。
2. **硬编码用户类型**：`USER_TYPE === 'ant'` 的判定将内部用户全部纳入，无法对内部用户进行灰度或 A/B 测试。若未来需要针对 ants 也做实验分组，需要修改代码。
3. **无上下文感知**：该函数不接受任何参数，无法根据当前会话状态（如是否正在流式接收响应、是否处于特定模式）做出更精细的判定。例如，在某些特殊模式（如工具执行中）立即切换模型可能存在风险。
4. **缓存陈旧风险**：虽然 `CACHED_MAY_BE_STALE` 是可接受的，但如果 GrowthBook 缓存长期未更新（如离线使用），用户可能永远无法获得新功能，而 ants 始终不受影响。

### 改进建议
1. **重命名功能标志**：将 `tengu_immediate_model_command` 重命名为更通用的 `tengu_immediate_inference_config_command`，以准确反映其控制的命令范围。
2. **参数化判定**：考虑让函数接受当前应用上下文（如 `isStreaming`, `currentMode`），以便在特定场景下禁用即时执行，提升安全性。
3. **日志记录**：在判定结果发生变化时（如从 `false` 变为 `true`），可以记录一个 debug 日志，便于排查用户反馈的"命令有时立即生效有时不生效"问题。
4. **文档化**：在命令处理代码中明确注释为什么某些命令需要即时执行，以及该模块的判定逻辑，帮助新开发者理解设计意图。
