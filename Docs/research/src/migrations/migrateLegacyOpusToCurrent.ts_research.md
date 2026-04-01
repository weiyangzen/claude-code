# migrateLegacyOpusToCurrent.ts 研究文档

## 场景与职责

本迁移文件负责将使用显式 Opus 4.0/4.1 模型字符串的第一方（first-party）用户迁移到当前的 Opus 别名（`opus`）。这是模型版本升级策略的一部分，确保用户在模型更新后获得最佳体验。

**业务背景**：
- Opus 4.0/4.1 已从第一方 API 下线（与 Claude.ai 一致）
- `opus` 别名已解析到 Opus 4.6
- 仍在使用显式 4.0/4.1 字符串的用户是在 4.5 发布前固定的设置
- `parseUserSpecifiedModel` 已在运行时静默重映射，此迁移清理设置文件

## 功能点目的

1. **模型升级**：将旧版 Opus 模型引用升级到当前版本
2. **设置清理**：更新设置文件使 `/model` 命令显示正确的模型
3. **通知支持**：设置时间戳以便 REPL 显示一次性通知
4. **选择性迁移**：仅影响第一方用户，保留第三方用户设置

## 具体技术实现

### 关键流程

```
1. 检查 API 提供商是否为 firstParty
2. 检查旧版模型重映射功能是否启用
3. 读取 userSettings 中的 model 字段
4. 检查是否匹配旧版 Opus 字符串
5. 更新设置为 'opus'
6. 记录迁移时间戳到 global config
7. 记录分析事件
```

### 旧版模型字符串

```typescript
const LEGACY_OPUS_MODELS = [
  'claude-opus-4-20250514',
  'claude-opus-4-1-20250805',
  'claude-opus-4-0',
  'claude-opus-4-1',
]
```

### 核心代码逻辑

```typescript
export function migrateLegacyOpusToCurrent(): void {
  // 仅第一方用户
  if (getAPIProvider() !== 'firstParty') {
    return
  }

  // 检查功能开关
  if (!isLegacyModelRemapEnabled()) {
    return
  }

  const model = getSettingsForSource('userSettings')?.model
  
  // 检查是否匹配旧版模型
  if (
    model !== 'claude-opus-4-20250514' &&
    model !== 'claude-opus-4-1-20250805' &&
    model !== 'claude-opus-4-0' &&
    model !== 'claude-opus-4-1'
  ) {
    return
  }

  // 执行迁移
  updateSettingsForSource('userSettings', { model: 'opus' })
  
  // 记录时间戳（用于通知）
  saveGlobalConfig(current => ({
    ...current,
    legacyOpusMigrationTimestamp: Date.now(),
  }))
  
  // 记录分析事件
  logEvent('tengu_legacy_opus_migration', { from_model: model })
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../services/analytics/index.js` | `logEvent`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 记录迁移事件 |
| `../utils/config.js` | `saveGlobalConfig` | 保存迁移时间戳 |
| `../utils/model/model.js` | `isLegacyModelRemapEnabled` | 检查功能开关 |
| `../utils/model/providers.js` | `getAPIProvider` | 检查 API 提供商 |
| `../utils/settings/settings.js` | `getSettingsForSource`, `updateSettingsForSource` | 读写用户设置 |

### 依赖函数详解

**isLegacyModelRemapEnabled**（来自 `src/utils/model/model.ts`）：
```typescript
export function isLegacyModelRemapEnabled(): boolean {
  return !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP)
}
```
- 通过环境变量 `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` 控制
- 默认启用（返回 `true`）

**getAPIProvider**（来自 `src/utils/model/providers.ts`）：
```typescript
export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : 'firstParty'
}
```

## 依赖与外部交互

### 运行时重映射

`parseUserSpecifiedModel` 函数（`src/utils/model/model.ts`）已在运行时处理旧版模型：
```typescript
if (
  getAPIProvider() === 'firstParty' &&
  isLegacyOpusFirstParty(modelString) &&
  isLegacyModelRemapEnabled()
) {
  return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
}
```

此迁移的目的是**清理设置文件**，使显示正确并支持通知。

### 通知机制

`legacyOpusMigrationTimestamp` 存储在 `GlobalConfig` 中，用于：
- REPL 检测是否需要显示一次性模型升级通知
- 避免重复通知

### 分析事件

| 事件名称 | 触发条件 | 元数据 |
|---------|---------|--------|
| `tengu_legacy_opus_migration` | 迁移成功 | `from_model` |

## 风险、边界与改进建议

### 潜在风险

1. **第三方用户被错误排除**：当前明确跳过第三方用户
   - 原因：第三方提供商可能尚未有 4.6 容量
   - 风险：如果第三方也下线旧模型，这些用户会失败

2. **时间戳污染**：`legacyOpusMigrationTimestamp` 可能与其他逻辑冲突
   - 当前仅用于通知，风险较低

3. **[1m] 后缀丢失**：迁移到 `opus` 会丢失 `[1m]` 后缀
   - 设计决策：运行时 `parseUserSpecifiedModel` 会根据用户订阅重新添加

### 边界情况

| 场景 | 行为 |
|-----|------|
| 第三方用户 | 跳过 |
| 功能开关禁用 | 跳过 |
| 使用 `opus` 别名 | 跳过（已是目标）|
| 使用其他模型 | 跳过 |
| 设置读取失败 | 跳过（`?.` 可选链）|

### 改进建议

1. **第三方处理**：添加第三方提供商的模型可用性检查
   - 当第三方也有 4.6 时自动迁移

2. **[1m] 保留**：当前迁移会丢失 `[1m]`
   ```typescript
   // 建议检查并保留
   const has1m = model.endsWith('[1m]')
   updateSettingsForSource('userSettings', { 
     model: has1m ? 'opus[1m]' : 'opus' 
   })
   ```

3. **批量迁移**：当前逐个检查，性能可优化
   - 使用正则或 Set 提高匹配效率

4. **回滚支持**：如果用户想回退到旧模型
   - 当前无回滚机制
   - 建议添加备份或回滚标记

5. **通知内容**：时间戳仅支持简单通知
   - 建议添加迁移详情到配置，支持更丰富的通知内容
