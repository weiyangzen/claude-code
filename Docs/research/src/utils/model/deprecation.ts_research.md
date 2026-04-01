# Research: src/utils/model/deprecation.ts

## 场景与职责

本模块集中管理“模型弃用（deprecation）”元数据。当用户当前使用的模型已被官方标记为即将退役时，在 UI 中弹出高优先级警告通知，提醒用户切换到更新的模型。

## 功能点目的

- 维护一份硬编码的弃用模型清单 `DEPRECATED_MODELS`，记录每个模型的友好名称及各 provider（firstParty / bedrock / vertex / foundry）的退役日期。
- 提供 `getModelDeprecationWarning(modelId)` 供外部调用：若模型 ID（不区分大小写）包含清单中的 key，且当前 provider 存在退役日期，则返回格式化警告字符串；否则返回 `null`。

## 具体技术实现

### 数据结构

```ts
type DeprecationEntry = {
  modelName: string
  retirementDates: Record<APIProvider, string | null>
}
```

当前清单包含：
- `claude-3-opus` → firstParty: 2026-01-05, bedrock: 2026-01-15, vertex/foundry: 2026-01-05
- `claude-3-7-sonnet` → firstParty: 2026-02-19, bedrock: 2026-04-28, vertex: 2026-05-11, foundry: 2026-02-19
- `claude-3-5-haiku` → firstParty: 2026-02-19, 其余 provider 未弃用 (`null`)

### 匹配逻辑

`getDeprecatedModelInfo(modelId)` 将输入转小写后，遍历 `DEPRECATED_MODELS` 的每个 key，使用 `String.prototype.includes(key)` 做子串匹配（因此 `claude-3-opus-4-6` 不会误匹配 `claude-3-opus`，因为 key 更长时顺序遍历也能正确命中，但存在理论上更短 key 先匹配的风险）。

### 外部接口

- `getModelDeprecationWarning(modelId: string | null): string | null`

## 关键代码路径与文件引用

- **入口**: `src/utils/model/deprecation.ts`
- **被调用方**:
  - `src/hooks/notifs/useDeprecationWarningNotification.tsx` — React hook，在模型切换时触发通知。
  - `src/main.tsx` — 启动时检查当前模型并可能输出警告。
- **依赖**:
  - `src/utils/model/providers.ts` (`getAPIProvider`)

## 依赖与外部交互

- 仅依赖 `getAPIProvider()` 判断当前 provider，从而选取对应的 `retirementDate`。
- 无网络请求、无磁盘 IO、无缓存。

## 风险、边界与改进建议

- **风险**: 子串匹配 `includes` 未按 key 长度排序，若未来出现 `claude-3` 这样的短 key 与 `claude-3-7-sonnet` 这样的长 key 共存，短 key 可能先匹配导致错误结果。建议按 key 长度降序遍历。
- **边界**: 对 `null` 输入直接返回 `null`，不做报错。
- **边界**: 若某 provider 的退役日期为 `null`，则视为该 provider 下此模型未弃用。
- **改进建议**: 退役日期目前为硬编码字符串，未来可考虑从远程配置或 API 拉取，以减少每次模型发布后的代码修改。若保持本地硬编码，建议增加单元测试确保日期与官方公告一致。
- **改进建议**: 当前返回的警告文本固定为英文，若产品需要多语言，应将文案外置到 i18n 资源中。
