# src/state/onChangeAppState.ts 研究文档

## 场景与职责

`onChangeAppState.ts` 是 Claude Code 全局状态变更的 **统一副作用闸门（choke point）**。它作为 `createStore` 的 `onChange` 回调注册，在 **每次 `setState` 导致状态实际变化后** 执行，负责将 AppState 的变更同步到外部系统：

1. **权限模式同步** — 当 `toolPermissionContext.mode` 变化时，通知 CCR（Claude.ai Remote）与 SDK 消费者。
2. **主模型持久化** — `mainLoopModel` 在 null 与非 null 之间切换时，自动写入/清除用户设置。
3. **视图状态持久化** — `expandedView` 变化时映射回旧版全局配置字段（`showExpandedTodos`、`showSpinnerTree`）。
4. **verbose 持久化** — 同步到全局配置。
5. **Tungsten 面板可见性** — ant-only 的 tmux 面板 sticky toggle 持久化。
6. **设置变更清理** — 当 `settings` 对象引用变化时，清除 API Key / AWS / GCP 凭证缓存，并在 `settings.env` 变化时重新应用环境变量。

该文件的核心设计哲学是：**与其在 8+ 个 mutation 路径上各自手动同步，不如在单一 diff 点统一处理**，从而消除遗漏。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `onChangeAppState` | Store 的 `onChange` 回调，接收 `{ newState, oldState }`，在所有 listener 之前执行。 |
| `externalMetadataToAppState` | 逆向操作：将 CCR 的 `SessionExternalMetadata` 还原为 AppState patch，用于 worker 重启或远程恢复。 |
| 权限模式 diff | 解决历史上仅 2/8+ 条 mutation 路径通知 CCR 的缺陷，确保任何 `setAppState` 改 mode 都能同步到外部。 |
| 模型设置双向同步 | `--model` 临时切换与持久化设置之间的桥梁；null 时清除，非 null 时保存。 |
| 设置变更缓存清理 | 保证 `apiKeyHelper`、云凭证修改后立即生效，无需重启进程。 |

## 具体技术实现

### 1. onChangeAppState 主流程

```ts
export function onChangeAppState({ newState, oldState }: { newState: AppState; oldState: AppState }) {
  // 1. 权限模式同步
  const prevMode = oldState.toolPermissionContext.mode
  const newMode = newState.toolPermissionContext.mode
  if (prevMode !== newMode) {
    const prevExternal = toExternalPermissionMode(prevMode)
    const newExternal = toExternalPermissionMode(newMode)
    if (prevExternal !== newExternal) {
      const isUltraplan = newExternal === 'plan' && newState.isUltraplanMode && !oldState.isUltraplanMode ? true : null
      notifySessionMetadataChanged({ permission_mode: newExternal, is_ultraplan_mode: isUltraplan })
    }
    notifyPermissionModeChanged(newMode)
  }

  // 2. mainLoopModel 清除
  if (newState.mainLoopModel !== oldState.mainLoopModel && newState.mainLoopModel === null) {
    updateSettingsForSource('userSettings', { model: undefined })
    setMainLoopModelOverride(null)
  }

  // 3. mainLoopModel 写入
  if (newState.mainLoopModel !== oldState.mainLoopModel && newState.mainLoopModel !== null) {
    updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
    setMainLoopModelOverride(newState.mainLoopModel)
  }

  // 4. expandedView 持久化
  if (newState.expandedView !== oldState.expandedView) { ... }

  // 5. verbose 持久化
  if (newState.verbose !== oldState.verbose && getGlobalConfig().verbose !== newState.verbose) { ... }

  // 6. tungstenPanelVisible 持久化 (ant-only)
  if (process.env.USER_TYPE === 'ant') { ... }

  // 7. settings 变更 → 清理缓存 + 应用环境变量
  if (newState.settings !== oldState.settings) { ... }
}
```

### 2. 权限模式同步的详细逻辑

这是文件中最关键、注释最详尽的代码块：

```ts
const prevExternal = toExternalPermissionMode(prevMode)
const newExternal = toExternalPermissionMode(newMode)
if (prevExternal !== newExternal) {
  const isUltraplan =
    newExternal === 'plan' && newState.isUltraplanMode && !oldState.isUltraplanMode
      ? true
      : null
  notifySessionMetadataChanged({
    permission_mode: newExternal,
    is_ultraplan_mode: isUltraplan,
  })
}
notifyPermissionModeChanged(newMode)
```

- **外部化（externalize）**：CCR 不应接收内部模式名（如 `bubble`、`ungated auto`），因此先调用 `toExternalPermissionMode` 映射。
- **去噪**：若外部化后相同（如 `default → bubble → default` 都映射为 `'default'`），则跳过 CCR 通知，避免无意义的 PUT。
- **Ultraplan 门控**：`isUltraplanMode` 仅在首次 plan cycle 的 control_request 中被原子设置，因此用 `!oldState.isUltraplanMode` 来精确判断是否是第一次进入 ultraplan。
- **双通道通知**：
  - `notifySessionMetadataChanged` → CCR `external_metadata` PUT（通过 `ccrClient.reportMetadata`）。
  - `notifyPermissionModeChanged` → SDK 状态流（`print.ts` 中注册监听器）。

### 3. externalMetadataToAppState 逆向恢复

```ts
export function externalMetadataToAppState(metadata: SessionExternalMetadata): (prev: AppState) => AppState {
  return prev => ({
    ...prev,
    ...(typeof metadata.permission_mode === 'string'
      ? { toolPermissionContext: { ...prev.toolPermissionContext, mode: permissionModeFromString(metadata.permission_mode) } }
      : {}),
    ...(typeof metadata.is_ultraplan_mode === 'boolean'
      ? { isUltraplanMode: metadata.is_ultraplan_mode }
      : {}),
  })
}
```

- 返回一个 updater 函数，可直接传给 `setAppState`。
- 目前仅恢复 `permission_mode` 与 `is_ultraplan_mode`，是 worker 重启或远程会话恢复时的状态重建入口。

### 4. 设置变更的缓存清理

```ts
if (newState.settings !== oldState.settings) {
  try {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
    clearGcpCredentialsCache()
    if (newState.settings.env !== oldState.settings.env) {
      applyConfigEnvironmentVariables()
    }
  } catch (error) {
    logError(toError(error))
  }
}
```

- 使用引用相等 (`!==`) 判断设置是否变化，这意味着设置对象必须被整体替换（immutable update）。
- 环境变量重新应用是 **additive-only**：只添加/覆盖，不删除已有变量。

## 关键代码路径与文件引用

| 代码路径 | 说明 |
|----------|------|
| `onChangeAppState` (L43) | 被 `src/state/AppState.tsx` 的 `AppStateProvider` 作为 `createStore(initialState, onChangeAppState)` 的第二个参数传入。 |
| `externalMetadataToAppState` (L24) | 被 `src/main.tsx` 用于从远程元数据恢复状态；也可能被 worker 恢复逻辑使用。 |
| `notifySessionMetadataChanged` (L86) | 来自 `../utils/sessionState.js`，触发 CCR 侧元数据更新。 |
| `notifyPermissionModeChanged` (L91) | 来自同一文件，触发 SDK 权限模式变更事件。 |
| `updateSettingsForSource` (L100, L110) | 来自 `../utils/settings/settings.js`，持久化模型设置。 |
| `setMainLoopModelOverride` (L101, L111) | 来自 `../bootstrap/state.js`，设置进程级的模型覆盖值。 |
| `saveGlobalConfig` (L122, L137, L150) | 来自 `../utils/config.js`，持久化全局配置（verbose、expandedView、tungstenPanelVisible）。 |
| `applyConfigEnvironmentVariables` (L165) | 来自 `../utils/managedEnv.js`，重新应用 `settings.env`。 |

## 依赖与外部交互

### 直接依赖

- `./AppStateStore.js` → `AppState` 类型
- `../bootstrap/state.js` → `setMainLoopModelOverride`
- `../utils/auth.js` → `clearApiKeyHelperCache`, `clearAwsCredentialsCache`, `clearGcpCredentialsCache`
- `../utils/config.js` → `getGlobalConfig`, `saveGlobalConfig`
- `../utils/errors.js` → `toError`
- `../utils/log.js` → `logError`
- `../utils/managedEnv.js` → `applyConfigEnvironmentVariables`
- `../utils/permissions/PermissionMode.js` → `permissionModeFromString`, `toExternalPermissionMode`
- `../utils/sessionState.js` → `notifyPermissionModeChanged`, `notifySessionMetadataChanged`, `SessionExternalMetadata`
- `../utils/settings/settings.js` → `updateSettingsForSource`

### 调用方（上游）

- `src/components/App.tsx`：将 `onChangeAppState` 作为 prop 传给 `AppStateProvider`。
- `src/main.tsx`：直接导入并用于 `createStore`；也使用 `externalMetadataToAppState`。
- `src/state/AppState.tsx`：`AppStateProvider` 内部调用 `createStore(..., onChangeAppState)`。

## 风险、边界与改进建议

### 风险

1. **单点故障与性能瓶颈**  
   所有 `setState` 都要经过本文件的 diff 逻辑。虽然当前操作都是 O(1) 的字段比较，但随着状态增长和副作用增多，这里可能成为性能瓶颈。任何本文件中的异常（如 `saveGlobalConfig` 抛错）都会阻断后续 listener 通知。

2. **引用相等判断的脆弱性**  
   `newState.settings !== oldState.settings` 依赖上层 updater 做不可变更新。若某处不小心 mutate 了 `settings` 对象再传给 `setState`，diff 会失败，导致缓存不清理、环境变量不更新。

3. **tungstenPanelVisible 的 ant-only 硬编码**  
   `process.env.USER_TYPE === 'ant'` 的判定在编译期固定，外部构建中这段逻辑会被保留但永远为 false；虽无运行时开销，但增加了代码分支的维护负担。

4. **expandedView 的双向映射债务**  
   `expandedView` 需要同步到 `showExpandedTodos` 和 `showSpinnerTree` 两个旧字段，说明存在配置层面的 legacy debt。未来若移除旧字段，需要同步修改读取端。

### 边界

- `onChange` 只在 `Object.is(next, prev)` 为 false 时触发（由 `store.ts` 保证），因此若 updater 返回相同引用，本文件逻辑不会执行。
- `notifySessionMetadataChanged` 中的 `is_ultraplan_mode: null` 遵循 RFC 7396 JSON Merge Patch 语义，表示删除该键。
- `applyConfigEnvironmentVariables` 是 additive-only，意味着已删除的 `settings.env` 键不会从 `process.env` 中移除；这是设计上的限制。

### 改进建议

1. **引入结构化 diff 或订阅分片**  
   与其在单一函数中检查所有字段，可考虑让 `createStore` 支持按字段/路径的订阅，或将副作用注册为独立的“状态反应器”（如 effect system），降低单点复杂度。

2. **将持久化逻辑下沉到各自领域**  
   `saveGlobalConfig` 的调用分散在多个 if 块中，可考虑将 verbose、expandedView、tungstenPanelVisible 的持久化封装成独立的“持久化选择器”，由统一的持久化层处理。

3. **增加不可变更新的开发时校验**  
   在开发/测试构建中，可在 `createStore` 或 `onChangeAppState` 前加入 `Object.freeze` 或 deep freeze，捕获意外 mutate。

4. **清理 mainLoopModel 的双向同步**  
   当前模型设置既在 `settings.json` 中又在 `AppState.mainLoopModel` 中维护，可考虑将 `mainLoopModel` 变为纯派生状态（selector），以 `settings` 为唯一数据源，消除双向同步的复杂性。
