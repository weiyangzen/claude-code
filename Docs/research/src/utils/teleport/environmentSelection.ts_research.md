# Research: src/utils/teleport/environmentSelection.ts

## 场景与职责

`src/utils/teleport/environmentSelection.ts` 负责将**后端 Environment API** 返回的可用环境与**本地用户设置**（`settings.remote.defaultEnvironmentId`）进行撮合，得出“当前应当使用哪个远程环境”以及“这个选择来自哪一份设置”。

它是连接“环境列表”与“用户偏好”的薄适配层，主要服务于：
- `/remote-env` 命令的交互式选择对话框（`RemoteEnvironmentDialog`）。
- `teleportToRemote` 在创建会话时的环境选择逻辑（通过 `getSettings_DEPRECATED` 直接读取，不经过本文件）。

注意：虽然 `teleportToRemote`（`src/utils/teleport.tsx`）没有直接导入本函数，但它复用了相同的选择逻辑；本文件将其抽离出来，专门供 UI 和设置展示使用。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `EnvironmentSelectionInfo` | 类型定义：可用环境数组、已选环境、已选环境的设置来源。 |
| `getEnvironmentSelectionInfo` | 异步获取环境选择全景信息：拉取环境列表 → 按设置 defaultEnvironmentId 匹配 → 回退到默认启发式 → 追溯设置来源。 |

## 具体技术实现

### 选择优先级（`getEnvironmentSelectionInfo` 内部逻辑）
1. **拉取环境**：调用 `fetchEnvironments()` 获取全部环境。
2. **无环境提前返回**：若数组为空，返回三字段均为空/null。
3. **默认启发式**：当没有 `defaultEnvironmentId` 或匹配失败时，
   - 优先选第一个 `kind !== 'bridge'` 的环境；
   - 若全是 `bridge`，则回退到 `environments[0]`。
4. **按设置匹配**：若 `getSettings_DEPRECATED().remote.defaultEnvironmentId` 存在且能在列表中命中，则使用该环境。
5. **来源追溯**：当 `defaultEnvironmentId` 命中时，从低到高遍历 `SETTING_SOURCES`，找到最后一个（优先级最高）包含该 `defaultEnvironmentId` 的来源，作为 `selectedEnvironmentSource`。
   - 显式跳过 `flagSettings`，注释说明其为 "not a normal source we check"。

### 数据结构
```ts
export type EnvironmentSelectionInfo = {
  availableEnvironments: EnvironmentResource[]
  selectedEnvironment: EnvironmentResource | null
  selectedEnvironmentSource: SettingSource | null
}
```

## 关键代码路径与文件引用

- **本文件**：`src/utils/teleport/environmentSelection.ts`（77 行）
- **直接调用方**：
  - `src/components/RemoteEnvironmentDialog.tsx` — Ink 组件，渲染环境选择对话框，展示当前使用环境及其来源。
  - `src/commands/remote-env/remote-env.tsx` — `/remote-env` 命令的入口，简单包裹上述对话框。
- **相关但非直接调用方**：
  - `src/utils/teleport.tsx` — `teleportToRemote` 函数内部实现了与本文件几乎一致的选择逻辑（读取 `settings.remote.defaultEnvironmentId` → 匹配 → 回退 `cloudEnv` / 非 `bridge` / 首个），但没有复用本函数，存在逻辑重复。

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `src/utils/settings/constants.ts` | `SETTING_SOURCES` 数组、`SettingSource` 类型、来源显示名称。 |
| `src/utils/settings/settings.ts` | `getSettings_DEPRECATED`（读取合并后的设置）、`getSettingsForSource`（按来源读取原始设置，用于追溯）。 |
| `src/utils/teleport/environments.ts` | `fetchEnvironments`、`EnvironmentResource` 类型。 |

外部 API：
- 通过 `fetchEnvironments()` 间接调用 `GET /v1/environment_providers`。

## 风险、边界与改进建议

### 风险
1. **逻辑重复**：`teleportToRemote`（`src/utils/teleport.tsx` 第 1054-1093 行）与本文件的选择逻辑高度相似，但未调用 `getEnvironmentSelectionInfo`。若默认启发式或来源追溯规则变更，需要同时修改两处，容易遗漏。
2. **`flagSettings` 被跳过**：用户在命令行通过 `--settings` 传入的 `remote.defaultEnvironmentId` 不会被 `selectedEnvironmentSource` 识别，UI 上可能显示为“来自其他设置”或回退到默认环境，造成困惑。
3. **无缓存**：`RemoteEnvironmentDialog` 每次挂载都会触发 `fetchEnvironments()`，而该函数本身无缓存；用户快速打开/关闭对话框会产生重复 API 请求。

### 边界
- `bridge` 环境在默认启发式中被明确降级。这是有意为之：bridge 用于 `remote-control` 等本地代理场景，不适合作为普通远程会话的默认环境。
- 当 `defaultEnvironmentId` 指向一个已不存在的环境时，函数不会报错，而是静默回退到默认启发式；`selectedEnvironmentSource` 为 `null`。

### 改进建议
- **统一入口**：让 `teleportToRemote` 直接调用 `getEnvironmentSelectionInfo`，消除逻辑重复。
- **支持 `flagSettings` 追溯**：若命令行显式指定了环境，应在 UI 中正确显示来源（如 "CLI flag"）。
- **引入本地缓存**：在 `RemoteEnvironmentDialog` 或 `fetchEnvironments` 层增加短周期缓存（如 30s），减少重复请求。
- **增加不匹配提示**：当用户配置的 `defaultEnvironmentId` 不在可用列表中时，在 `RemoteEnvironmentDialog` 的 subtitle 中给出提示（如 "Configured default not available"）。
