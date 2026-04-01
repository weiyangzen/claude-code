# thinking.ts 研究文档

## 场景与职责

`thinking.ts` 是 Claude Code CLI 的模型思考（thinking）功能管理模块，负责处理与 Claude 模型思考能力相关的配置、检测和高亮功能。该模块是 Claude 4+ 模型扩展能力（extended thinking）的核心配置中心。

主要使用场景：
1. **Ultrathink 功能**：检测用户输入中的 "ultrathink" 关键词并触发特殊处理
2. **模型思考能力检测**：判断当前模型是否支持 thinking 和 adaptive thinking
3. **默认配置**：确定是否默认启用 thinking 功能
4. **彩虹高亮**：为 ultrathink 关键词提供彩虹色高亮效果

## 功能点目的

### 1. Ultrathink 功能控制 (`isUltrathinkEnabled`)
- **双层控制**：
  - 构建时控制：`feature('ULTRATHINK')` 决定代码是否包含在构建中
  - 运行时控制：GrowthBook 功能开关 `tengu_turtle_carbon`
- **目的**：允许逐步推出功能，紧急情况下可快速禁用

### 2. 关键词检测与高亮定位
- `hasUltrathinkKeyword(text)`：快速检测文本是否包含 "ultrathink"
- `findThinkingTriggerPositions(text)`：精确定位所有关键词位置，用于 UI 高亮

### 3. 模型能力检测
- `modelSupportsThinking(model)`：检测模型是否支持 extended thinking
- `modelSupportsAdaptiveThinking(model)`：检测模型是否支持 adaptive thinking（动态预算）

### 4. 默认启用策略
- `shouldEnableThinkingByDefault()`：根据环境变量和设置决定是否默认启用

## 具体技术实现

### 核心类型

```typescript
export type ThinkingConfig =
  | { type: 'adaptive' }                    // 自适应思考（动态预算）
  | { type: 'enabled'; budgetTokens: number }  // 固定预算
  | { type: 'disabled' }                    // 禁用
```

### Ultrathink 检测

```typescript
export function hasUltrathinkKeyword(text: string): boolean {
  return /\bultrathink\b/i.test(text)
}
```

**正则说明**：
- `\b`：单词边界，避免匹配 "superultrathink" 或 "ultrathinking"
- `i`：不区分大小写

**位置定位实现**：
```typescript
export function findThinkingTriggerPositions(text: string): Array<{
  word: string
  start: number
  end: number
}>
```

**关键注释**：
```typescript
// Fresh /g literal each call — String.prototype.matchAll copies lastIndex
// from the source regex, so a shared instance would leak state from
// hasUltrathinkKeyword's .test() into this call on the next render.
const matches = text.matchAll(/\bultrathink\b/gi)
```

**问题背景**：
- JavaScript 的 `RegExp` 带有 `lastIndex` 状态
- `hasUltrathinkKeyword` 使用 `.test()` 会修改 `lastIndex`
- 如果共享同一个 `/gi` 正则实例，`findThinkingTriggerPositions` 可能从错误位置开始匹配

### 彩虹高亮颜色

```typescript
const RAINBOW_COLORS: Array<keyof Theme> = [
  'rainbow_red', 'rainbow_orange', 'rainbow_yellow',
  'rainbow_green', 'rainbow_blue', 'rainbow_indigo', 'rainbow_violet',
]

const RAINBOW_SHIMMER_COLORS: Array<keyof Theme> = [
  'rainbow_red_shimmer', /* ... */
]

export function getRainbowColor(charIndex: number, shimmer: boolean = false): keyof Theme {
  const colors = shimmer ? RAINBOW_SHIMMER_COLORS : RAINBOW_COLORS
  return colors[charIndex % colors.length]!
}
```

**使用方式**：对 "ultrathink" 的每个字符依次应用彩虹色，产生彩虹文字效果。

### 模型能力检测

#### `modelSupportsThinking`

**检测逻辑**（优先级从高到低）：

1. **3P 覆盖配置**：`get3PModelCapabilityOverride(model, 'thinking')`
   - 允许通过配置覆盖第三方模型的能力检测

2. **内部用户特殊处理**：
   ```typescript
   if (process.env.USER_TYPE === 'ant') {
     if (resolveAntModel(model.toLowerCase())) return true
   }
   ```

3. **基于提供商和模型名称**：
   - 1P 和 Foundry：所有 Claude 4+ 模型（不包括 Claude 3.x）
   - 3P（Bedrock/Vertex）：仅 Opus 4+ 和 Sonnet 4+

**重要注释**：
```typescript
// IMPORTANT: Do not change thinking support without notifying the model
// launch DRI and research. This can greatly affect model quality and bashing.
```

#### `modelSupportsAdaptiveThinking`

**Adaptive Thinking**：模型动态决定思考预算，而非固定值。

**支持检测**：
- 明确支持：`opus-4-6`、`sonnet-4-6`（4.6 版本）
- 明确不支持：其他 opus/sonnet/haiku 变体
- 未知模型：1P 和 Foundry 默认 true，其他默认 false

**关键注释**：
```typescript
// IMPORTANT: Do not change adaptive thinking support without notifying the
// model launch DRI and research. This can greatly affect model quality and
// bashing.

// Newer models (4.6+) are all trained on adaptive thinking and MUST have it
// enabled for model testing. DO NOT default to false for first party, otherwise
// we may silently degrade model quality.
```

### 默认启用策略

```typescript
export function shouldEnableThinkingByDefault(): boolean {
  // 环境变量优先
  if (process.env.MAX_THINKING_TOKENS) {
    return parseInt(process.env.MAX_THINKING_TOKENS, 10) > 0
  }

  // 设置覆盖
  const { settings } = getSettingsWithErrors()
  if (settings.alwaysThinkingEnabled === false) {
    return false
  }

  // 默认启用
  return true
}
```

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/QueryEngine.ts` | 查询引擎中的 thinking 配置 |
| `src/cli/print.ts` | CLI 打印输出 |
| `src/state/AppStateStore.ts` | 应用状态管理 |
| `src/components/tasks/RemoteSessionProgress.tsx` | 远程会话进度显示 |
| `src/buddy/useBuddyNotification.tsx` | Buddy 通知 |
| `src/services/api/claude.ts` | API 调用中的 thinking 参数 |
| `src/screens/ResumeConversation.tsx` | 恢复对话 |
| `src/screens/REPL.tsx` | REPL 主界面 |
| `src/services/api/withRetry.ts` | 重试逻辑中的 thinking 处理 |
| `src/Tool.ts` | 工具定义中的 thinking 支持 |
| `src/main.tsx` | 主入口 |
| `src/components/messages/HighlightedThinkingText.tsx` | 思考文本高亮 |
| `src/components/PromptInput/PromptInput.tsx` | 输入框中的 ultrathink 检测 |
| `src/utils/queryContext.ts` | 查询上下文 |
| `src/utils/effort.ts` | 工作量估算 |
| `src/utils/attachments.ts` | 附件处理 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `bun:bundle` (feature) | 构建时功能开关 |
| `../services/analytics/growthbook.js` | 运行时功能开关 |
| `./model/model.js` | 模型名称解析 |
| `./model/modelSupportOverrides.js` | 3P 模型能力覆盖 |
| `./model/providers.js` | API 提供商检测 |
| `./settings/settings.js` | 用户设置 |
| `./theme.js` | 彩虹颜色定义 |

## 依赖与外部交互

### 与 GrowthBook 的集成

```typescript
export function isUltrathinkEnabled(): boolean {
  if (!feature('ULTRATHINK')) {  // 构建时检查
    return false
  }
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_turtle_carbon', true)  // 运行时检查
}
```

**双层控制的意义**：
- 构建时：从外部构建中完全移除功能代码（代码体积优化）
- 运行时：无需重新构建即可开关功能（运营灵活性）

### 与模型系统的集成

**能力检测链**：
1. 配置覆盖（最高优先级）
2. 内部用户特殊模型
3. 提供商 + 模型名称规则

**模型版本识别**：
```typescript
const canonical = getCanonicalName(model)  // 标准化模型名称
// 示例：
// "claude-opus-4-6-20251022" → "claude-opus-4-6"
// "us.anthropic.claude-opus-4-6-v1:0" → "claude-opus-4-6"
```

### 与设置系统的集成

```typescript
const { settings } = getSettingsWithErrors()
if (settings.alwaysThinkingEnabled === false) {
  return false
}
```

用户可以通过设置禁用 thinking：
```json
{
  "alwaysThinkingEnabled": false
}
```

## 风险、边界与改进建议

### 潜在风险

1. **模型能力检测硬编码**
   - 模型版本字符串匹配是硬编码的
   - 新模型发布时需要更新代码
   - 注释 `@[MODEL LAUNCH]` 标记了需要更新的位置

2. **正则状态泄漏**
   - 虽然已修复，但类似的 `lastIndex` 问题可能存在于其他模块
   - JavaScript 的 `RegExp` 全局匹配行为容易出错

3. **3P 模型检测不准确**
   - 依赖模型名称字符串匹配
   - 用户自定义部署 ID（如 Foundry）可能无法正确识别

4. **功能开关缓存**
   - `getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期值
   - 功能状态变化可能有延迟

### 边界条件

| 场景 | 行为 |
|-----|------|
| 空文本 | `hasUltrathinkKeyword('')` → false |
| 大小写混合 | `"UlTrAtHiNk"` → 匹配（/i 标志）|
| 子串匹配 | `"superultrathink"` → 不匹配（\b 边界）|
| 未知模型 | 1P/Foundry 默认启用 thinking，3P 默认禁用 |
| `MAX_THINKING_TOKENS=0` | `shouldEnableThinkingByDefault()` → false |
| `MAX_THINKING_TOKENS=abc` | `parseInt` 返回 NaN，视为 false |

### 改进建议

1. **模型能力配置化**
   ```typescript
   // 建议：从 GrowthBook 或配置文件读取模型能力
   const MODEL_CAPABILITIES = getFeatureValue('model_capabilities', {
     'claude-opus-4-6': { thinking: true, adaptive: true },
     // ...
   })
   ```

2. **更灵活的默认策略**
   ```typescript
   // 建议：支持按模型类型设置默认
   interface ThinkingDefaultPolicy {
     opus: boolean
     sonnet: boolean
     haiku: boolean
   }
   ```

3. **Ultrathink 变体支持**
   ```typescript
   // 支持更多关键词变体
   const ULTRATHINK_VARIANTS = [
     /\bultrathink\b/i,
     /\bdeepthink\b/i,
     /\bthinkhard\b/i,
   ]
   ```

4. **彩虹高亮配置**
   ```typescript
   // 允许用户自定义彩虹颜色
   export function setRainbowColors(colors: Array<keyof Theme>): void
   ```

5. **Thinking 预算建议**
   ```typescript
   // 根据上下文大小建议预算
   export function suggestThinkingBudget(contextTokens: number): number {
     return Math.min(32000, Math.max(1024, contextTokens / 4))
   }
   ```

6. **A/B 测试支持**
   ```typescript
   // 支持 thinking 参数的 A/B 测试
   export function getThinkingConfigForExperiment(
     experimentId: string,
   ): ThinkingConfig
   ```

### 测试建议

应覆盖以下场景：
- 各种 ultrathink 变体的匹配/不匹配
- `matchAll` 和 `test` 的 lastIndex 隔离
- 各模型版本的能力检测结果
- 3P 覆盖配置的优先级
- 环境变量和设置的组合效果
- 彩虹颜色循环（charIndex > colors.length）
