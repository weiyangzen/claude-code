# useDynamicConfig.ts 研究文档

## 场景与职责

`useDynamicConfig` 是一个轻量级的 React Hook，用于在组件中**异步获取 GrowthBook 动态配置值**。它的设计目的是解决 GrowthBook 初始化是异步的这个问题：组件首次渲染时配置可能还没从服务器拉取回来，因此 Hook 先返回默认值，待 GrowthBook 初始化完成后再更新为真实值。

该 Hook 目前主要用于 `src/components/FeedbackSurvey/useFeedbackSurvey.tsx`，控制反馈问卷的展示逻辑（如问卷类型、触发条件等）。

## 功能点目的

1. **提供默认值兜底**：在 GrowthBook 尚未完成网络请求前，组件可以基于 `defaultValue` 正常渲染，避免阻塞 UI。

2. **异步拉取并更新**：在 `useEffect` 中调用 `getDynamicConfig_BLOCKS_ON_INIT`，待其 resolve 后通过 `setConfigValue` 更新状态，触发重渲染。

3. **测试环境保护**：当 `process.env.NODE_ENV === 'test'` 时，直接跳过异步拉取，防止测试挂起（因为测试环境中 GrowthBook 可能未初始化或网络请求被 mock）。

## 具体技术实现

### 源码实现

```ts
import React from 'react'
import { getDynamicConfig_BLOCKS_ON_INIT } from '../services/analytics/growthbook.js'

export function useDynamicConfig<T>(configName: string, defaultValue: T): T {
  const [configValue, setConfigValue] = React.useState<T>(defaultValue)

  React.useEffect(() => {
    if (process.env.NODE_ENV === 'test') {
      return
    }
    void getDynamicConfig_BLOCKS_ON_INIT<T>(configName, defaultValue).then(
      setConfigValue,
    )
  }, [configName, defaultValue])

  return configValue
}
```

### 实现要点

- **状态管理**：使用 `useState` 保存当前配置值，初始为 `defaultValue`。
- **副作用触发条件**：`useEffect` 的依赖数组是 `[configName, defaultValue]`。这意味着如果调用方传入了新的 `configName` 或不同的 `defaultValue`，会重新发起请求。
- **异步更新不阻塞渲染**：使用 `void` 忽略 Promise，让 React 继续正常渲染流程。
- **测试短路**：明确在测试环境中不执行任何异步操作，避免测试超时或需要额外 mock GrowthBook。

### 底层依赖：`getDynamicConfig_BLOCKS_ON_INIT`

该函数定义在 `src/services/analytics/growthbook.ts` 中，其实现是：
```ts
export async function getDynamicConfig_BLOCKS_ON_INIT<T>(
  configName: string,
  defaultValue: T,
): Promise<T> {
  // 内部调用 getFeatureValueInternal，会等待 initializeGrowthBook() 完成
  return getFeatureValueInternal(configName, defaultValue, false)
}
```

`getFeatureValueInternal` 的工作流程：
1. 检查环境变量覆盖 `CLAUDE_INTERNAL_FC_OVERRIDES`
2. 检查本地配置覆盖 `growthBookOverrides`
3. 若 GrowthBook 未启用，返回 `defaultValue`
4. 调用 `initializeGrowthBook()` 阻塞等待客户端初始化
5. 优先读取 `remoteEvalFeatureValues` 内存缓存，否则回退到 `growthBookClient.getFeatureValue()`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useDynamicConfig.ts` | 本 Hook 实现 |
| `src/components/FeedbackSurvey/useFeedbackSurvey.tsx` | 主要调用方，用于获取反馈问卷配置 |
| `src/services/analytics/growthbook.ts` | `getDynamicConfig_BLOCKS_ON_INIT`、`getFeatureValueInternal`、`initializeGrowthBook` |
| `src/services/analytics/firstPartyEventLogger.ts` | GrowthBook 实验曝光日志 |
| `src/utils/config.ts` | `getGlobalConfig`、`saveGlobalConfig`（存储 `cachedGrowthBookFeatures`、`growthBookOverrides`） |

## 依赖与外部交互

### 内部依赖
- **React**：`useState`、`useEffect`
- **GrowthBook SDK**：`@growthbook/growthbook`（在 `growthbook.ts` 中封装）

### 外部交互
- **Anthropic API / GrowthBook 远程评估服务**：`initializeGrowthBook()` 会发起网络请求到 `https://api.anthropic.com/`（或 `CLAUDE_CODE_GB_BASE_URL`），获取远程评估后的特性值。
- **本地磁盘缓存**：成功获取的配置会写入 `~/.claude.json` 的 `cachedGrowthBookFeatures` 字段，供下次启动时快速读取。

## 风险、边界与改进建议

### 风险与边界

1. **`defaultValue` 变化导致重复请求**：由于 `useEffect` 依赖了 `defaultValue`，如果调用方每次渲染都传入一个新的对象/数组字面量作为 `defaultValue`，会导致无限循环或频繁重新初始化 GrowthBook。虽然当前调用方（`useFeedbackSurvey`）传入的是基本类型，但这是潜在陷阱。

2. **测试环境硬编码短路**：`process.env.NODE_ENV === 'test'` 的判断虽然防止了测试挂起，但也意味着任何使用 `useDynamicConfig` 的组件在测试中都无法验证 GrowthBook 集成逻辑，除非额外 mock `getDynamicConfig_BLOCKS_ON_INIT`。

3. **无错误处理**：`getDynamicConfig_BLOCKS_ON_INIT` 的 Promise 没有 `.catch`，如果 GrowthBook 初始化失败（如网络超时、认证失败），错误会成为未捕获的 Promise rejection。虽然 `getFeatureValueInternal` 内部有 fallback 到 `defaultValue` 的逻辑，但如果 `initializeGrowthBook()` 抛出异常，错误会冒泡。

4. **无加载状态暴露**：Hook 只返回当前值，调用方无法知道配置是“已加载真实值”还是“仍在使用默认值”。对于某些需要等待配置才能做决策的场景（如 A/B 实验分组），这可能导致闪烁或错误分组。

5. **GrowthBook 初始化全局阻塞**：`getDynamicConfig_BLOCKS_ON_INIT` 名字中的 `_BLOCKS_ON_INIT` 表明它会等待全局 GrowthBook 初始化。如果多个组件同时调用，虽然 `initializeGrowthBook` 是 memoized 的，但首次调用仍可能阻塞所有组件的 `useEffect` 执行直到网络返回。

### 改进建议

1. **从依赖数组中移除 `defaultValue`**：通常 `defaultValue` 不应触发重新获取。如果确实需要支持动态 defaultValue，建议调用方自行控制，或在 Hook 内部使用 ref 保存初始默认值。
   ```ts
   const initialDefaultRef = useRef(defaultValue)
   // useEffect deps: [configName]
   ```

2. **增加错误边界处理**：为 Promise 添加 `.catch`，将错误记录到 `logError`，避免未捕获的 rejection：
   ```ts
   void getDynamicConfig_BLOCKS_ON_INIT<T>(configName, defaultValue)
     .then(setConfigValue)
     .catch(e => logError(e))
   ```

3. **暴露加载状态**：返回一个三元组 `[value, isLoading, error]`，让调用方可以更好地处理配置加载中的 UI 状态：
   ```ts
   export function useDynamicConfig<T>(configName: string, defaultValue: T) {
     const [state, setState] = useState({ value: defaultValue, isLoading: true, error: null })
     useEffect(() => { ... }, [configName])
     return state
   }
   ```

4. **使用非阻塞缓存优先读取**：对于热路径（如每次渲染都读取的配置），应优先使用 `getFeatureValue_CACHED_MAY_BE_STALE`（同步、读内存/磁盘缓存），只在需要精确值或首次加载时使用 `_BLOCKS_ON_INIT`。当前 `useDynamicConfig` 的设计更适合“ mount 时加载一次”的场景。

5. **文档化使用限制**：由于 GrowthBook 远程评估有 5 秒超时，且首次调用会阻塞，建议在代码注释中明确说明该 Hook 不适合在频繁渲染的组件中使用（如列表项、动画组件），只适合在顶层容器或对话框中使用。
