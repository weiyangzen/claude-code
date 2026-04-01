# migrateOpusToOpus1m.ts 研究文档

## 场景与职责

本迁移文件负责将使用 `opus` 别名的符合条件的用户迁移到 `opus[1m]`（Opus 1M 上下文版本）。这是 Opus 1M 体验合并策略的一部分，为 Max/Team Premium 用户提供统一的 Opus 体验。

**业务背景**：
- Opus 1M 合并功能将 Opus 和 Opus 1M 合并为单一选项
- 符合条件的用户（Max/Team Premium 第一方用户）自动获得 1M 上下文
- Pro 订阅者保持分离的 Opus 和 Opus 1M 选项
- CLI 的 `--model opus` 标志不受影响（运行时覆盖）

## 功能点目的

1. **体验统一**：为符合条件的用户自动启用 Opus 1M
2. **选择性迁移**：仅影响启用了合并功能的用户
3. **CLI 兼容**：保留 `--model opus` 标志的原始行为
4. **Pro 用户排除**：Pro 订阅者保持原有选项

## 具体技术实现

### 关键流程

```
1. 检查 Opus 1M 合并功能是否启用
2. 读取 userSettings 中的 model 字段
3. 检查是否精确等于 'opus'
4. 检查迁移后的模型是否与默认模型相同
5. 如不同，更新设置为 'opus[1m]'
6. 记录分析事件
```

### 资格检查

**isOpus1mMergeEnabled**（来自 `src/utils/model/model.ts`）：
```typescript
export function isOpus1mMergeEnabled(): boolean {
  // 1M 上下文被禁用
  if (is1mContextDisabled()) return false
  
  // Pro 订阅者保持分离选项
  if (isProSubscriber()) return false
  
  // 仅第一方用户
  if (getAPIProvider() !== 'firstParty') return false
  
  // 订阅类型未知时保守处理（避免 API 拒绝）
  if (isClaudeAISubscriber() && getSubscriptionType() === null) {
    return false
  }
  
  return true
}
```

### 核心代码逻辑

```typescript
export function migrateOpusToOpus1m(): void {
  // 检查功能启用
  if (!isOpus1mMergeEnabled()) {
    return
  }

  // 检查当前模型
  const model = getSettingsForSource('userSettings')?.model
  if (model !== 'opus') {
    return
  }

  // 确定目标模型
  const migrated = 'opus[1m]'
  const modelToSet =
    parseUserSpecifiedModel(migrated) ===
    parseUserSpecifiedModel(getDefaultMainLoopModelSetting())
      ? undefined  // 如果与默认相同，清除设置
      : migrated

  // 执行更新
  updateSettingsForSource('userSettings', { model: modelToSet })

  logEvent('tengu_opus_to_opus1m_migration', {})
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../services/analytics/index.js` | `logEvent` | 记录迁移事件 |
| `../utils/model/model.js` | `getDefaultMainLoopModelSetting`, `isOpus1mMergeEnabled`, `parseUserSpecifiedModel` | 模型检查和解析 |
| `../utils/settings/settings.js` | `getSettingsForSource`, `updateSettingsForSource` | 读写用户设置 |

### 依赖函数详解

**parseUserSpecifiedModel**：
- 解析模型别名（如 `opus`、`opus[1m]`）为完整模型 ID
- 支持 `[1m]` 后缀处理
- 用于比较迁移前后模型是否实际相同

**getDefaultMainLoopModelSetting**：
- 返回当前用户的默认模型设置
- Max/Team Premium 用户返回 `opus[1m]`（当合并启用时）
- 用于判断是否需要显式设置

## 依赖与外部交互

### 模型系统

迁移与模型解析系统紧密集成：
1. `parseUserSpecifiedModel('opus[1m]')` → 解析为完整模型 ID
2. `getDefaultMainLoopModelSetting()` → 获取用户默认模型
3. 比较两者，如相同则清除显式设置（`undefined`）

### 订阅系统

通过 `isOpus1mMergeEnabled` 间接依赖：
- `isProSubscriber()` - 排除 Pro 用户
- `isClaudeAISubscriber()` - 检查 Claude.ai 订阅状态
- `getSubscriptionType()` - 获取订阅类型

### 分析事件

| 事件名称 | 触发条件 | 元数据 |
|---------|---------|--------|
| `tengu_opus_to_opus1m_migration` | 迁移成功 | 无 |

## 风险、边界与改进建议

### 潜在风险

1. **Pro 用户误判**：如果订阅类型检测失败，可能错误地迁移 Pro 用户
   - 缓解：`isOpus1mMergeEnabled` 中的保守检查

2. **默认模型变化**：如果默认模型逻辑改变，迁移行为可能意外变化
   - 依赖：`getDefaultMainLoopModelSetting` 的行为

3. **设置清除逻辑**：当迁移后模型与默认相同时清除设置
   - 风险：如果默认模型后续变化，用户可能获得意外模型

### 边界情况

| 场景 | 行为 |
|-----|------|
| 功能开关禁用 | 跳过 |
| 当前模型 !== 'opus' | 跳过 |
| 已是 `opus[1m]` | 跳过 |
| 迁移后模型与默认相同 | 清除设置（`undefined`）|
| Pro 订阅者 | 跳过（`isOpus1mMergeEnabled` 返回 false）|
| 第三方用户 | 跳过 |
| 1M 上下文禁用 | 跳过 |

### 改进建议

1. **显式标记**：当前依赖模型值比较
   - 建议：添加 `hasMigratedToOpus1m` 标记，避免重复检查

2. **回滚支持**：如果用户想回到 `opus`
   - 当前：用户可手动更改，但迁移会再次执行
   - 建议：添加 `skipOpus1mMigration` 选项

3. **通知用户**：静默迁移可能让用户困惑
   - 建议：添加一次性通知告知模型升级

4. **CLI 标志处理**：注释说明 `--model opus` 不受影响
   - 验证：确保 `getMainLoopModelOverride` 优先级正确

5. **测试覆盖**：建议添加场景测试：
   - 不同订阅类型的行为
   - 默认模型匹配逻辑
   - 多次执行的幂等性
