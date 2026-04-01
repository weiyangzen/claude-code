# Research: src/utils/model/contextWindowUpgradeCheck.ts

## 场景与职责

本模块负责在 UI 中向用户提示“上下文窗口升级”选项。当用户当前使用的模型支持更大的 1M context 变体，且用户账户具备访问权限时，在 TokenWarning 组件和 /compact 命令输出中展示升级引导（例如 `/model opus[1m]` 或提示语）。

## 功能点目的

- **getAvailableUpgrade()**: 内部函数，判断当前用户指定的模型（通过 `getUserSpecifiedModelSetting()`）是否为 `opus` 或 `sonnet`，并分别调用 `checkOpus1mAccess()` / `checkSonnet1mAccess()` 校验账户权限。若通过，返回包含 alias、name、multiplier（固定为 5）的对象。
- **getUpgradeMessage(context)**: 导出函数，根据调用场景返回不同文案：
  - `'warning'` → 返回简洁的 `/model <alias>` 指令，用于 TokenWarning 的紧凑提示。
  - `'tip'` → 返回完整提示语 `"Tip: You have access to <name> with <multiplier>x more context"`，用于 /compact 成功后的友好提示。

## 具体技术实现

- 依赖 `src/utils/model/check1mAccess.ts` 中的权限检查函数和 `src/utils/model/model.ts` 中的 `getUserSpecifiedModelSetting()`。
- 无状态、无副作用，纯函数式判断。
- 代码顶部有 `@[MODEL LAUNCH]` 注释，提示新增支持 1M 的模型时需要在此添加分支。

## 关键代码路径与文件引用

- **入口**: `src/utils/model/contextWindowUpgradeCheck.ts`
- **被调用方**:
  - `src/components/TokenWarning.tsx` — 在上下文使用率告警区域展示升级快捷指令。
  - `src/commands/compact/compact.ts` — 在手动 compact 成功后以 tip 形式提示用户。
- **依赖**:
  - `src/utils/model/check1mAccess.ts` (`checkOpus1mAccess`, `checkSonnet1mAccess`)
  - `src/utils/model/model.ts` (`getUserSpecifiedModelSetting`)

## 依赖与外部交互

- 读取当前 session 的模型覆盖状态（来自 `bootstrap/state.ts` 的 `mainLoopModelOverride`）。
- 读取全局配置中的 extraUsage 禁用原因（`check1mAccess.ts` 内部通过 `getGlobalConfig().cachedExtraUsageDisabledReason` 判断）。
- 不直接发起网络请求，所有权限判断均为本地同步计算。

## 风险、边界与改进建议

- **边界**: 若 `getUserSpecifiedModelSetting()` 返回 `undefined`（用户未指定模型或模型被 allowlist 拦截），则不会返回任何升级提示。
- **边界**: 仅对 alias 为 `opus` 和 `sonnet` 生效；对 `haiku`、`best`、`opusplan` 等 alias 不提示升级。
- **风险**: 每次 UI render 都会调用 `getUpgradeMessage('warning')`，虽然本身是纯函数，但会级联触发 `getUserSpecifiedModelSetting()` 和 `check*1mAccess()` 的调用链；当前调用链较轻，但需避免未来引入重型计算。
- **改进建议**: 如果未来更多模型支持 1M context，可将模型→升级配置的映射抽成表驱动，而非继续增加 `if/else` 分支。
