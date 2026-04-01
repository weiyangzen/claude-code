# migrateSonnet45ToSonnet46.ts 研究文档

## 场景与职责

本迁移文件负责将 Pro/Max/Team Premium 第一方用户从显式的 Sonnet 4.5 模型字符串迁移到 `sonnet` 别名（现在解析到 Sonnet 4.6）。这是 Sonnet 4.6 发布时的模型升级策略的一部分。

**业务背景**：
- `sonnet` 别名现在解析到 Sonnet 4.6
- 用户可能因以下原因固定在 Sonnet 4.5：
  - 之前的 `migrateSonnet1mToSonnet45` 迁移（`sonnet[1m]` → 显式 4.5[1m]）
  - 通过 `/model` 命令手动选择
- 仅针对 Pro/Max/Team Premium 第一方用户
- 新用户（`numStartups <= 1`）跳过通知

## 功能点目的

1. **模型升级**：将 Sonnet 4.5 用户升级到 Sonnet 4.6
2. **别名化**：使用 `sonnet` 别名而非显式版本号
3. **范围限制**：仅处理 `userSettings`，保留项目级固定
4. **用户通知**：为老用户设置时间戳以显示一次性通知

## 具体技术实现

### 关键流程

```
1. 检查 API 提供商是否为 firstParty
2. 检查用户是否为 Pro/Max/Team Premium 订阅者
3. 读取 userSettings 中的 model 字段
4. 检查是否匹配 Sonnet 4.5 字符串
5. 根据是否有 [1m] 后缀确定新值
6. 更新用户设置
7. 如非新用户，设置通知时间戳
8. 记录分析事件
```

### 目标模型字符串

```typescript
const SONNET_45_MODELS = [
  'claude-sonnet-4-5-20250929',
  'claude-sonnet-4-5-20250929[1m]',
  'sonnet-4-5-20250929',
  'sonnet-4-5-20250929[1m]',
]
```

### 映射规则

| 原模型 | 新模型 |
|-------|-------|
| `claude-sonnet-4-5-20250929` | `sonnet` |
| `sonnet-4-5-20250929` | `sonnet` |
| `claude-sonnet-4-5-20250929[1m]` | `sonnet[1m]` |
| `sonnet-4-5-20250929[1m]` | `sonnet[1m]` |

### 核心代码逻辑

```typescript
export function migrateSonnet45ToSonnet46(): void {
  // 仅第一方用户
  if (getAPIProvider() !== 'firstParty') {
    return
  }

  // 仅特定订阅类型
  if (!isProSubscriber() && !isMaxSubscriber() && !isTeamPremiumSubscriber()) {
    return
  }

  // 检查当前模型
  const model = getSettingsForSource('userSettings')?.model
  if (!SONNET_45_MODELS.includes(model)) {
    return
  }

  // 确定新模型（保留 [1m] 后缀）
  const has1m = model.endsWith('[1m]')
  updateSettingsForSource('userSettings', {
    model: has1m ? 'sonnet[1m]' : 'sonnet',
  })

  // 为老用户设置通知时间戳
  const config = getGlobalConfig()
  if (config.numStartups > 1) {
    saveGlobalConfig(current => ({
      ...current,
      sonnet45To46MigrationTimestamp: Date.now(),
    }))
  }

  // 记录分析事件
  logEvent('tengu_sonnet45_to_46_migration', {
    from_model: model,
    has_1m: has1m,
  })
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../services/analytics/index.js` | `logEvent`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 记录迁移事件 |
| `../utils/auth.js` | `isMaxSubscriber`, `isProSubscriber`, `isTeamPremiumSubscriber` | 订阅类型检查 |
| `../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读写全局配置 |
| `../utils/model/providers.js` | `getAPIProvider` | 检查 API 提供商 |
| `../utils/settings/settings.js` | `getSettingsForSource`, `updateSettingsForSource` | 读写用户设置 |

### 依赖函数详解

**订阅检查函数**（来自 `src/utils/auth.ts`）：
```typescript
export function isProSubscriber(): boolean
export function isMaxSubscriber(): boolean
export function isTeamPremiumSubscriber(): boolean
```
- 基于 OAuth 令牌中的订阅信息
- 缓存结果以提高性能

## 依赖与外部交互

### 订阅系统

迁移依赖准确的订阅类型检测：
1. OAuth 登录时获取订阅信息
2. 存储在 `GlobalConfig.oauthAccount`
3. `isProSubscriber()` 等函数解析订阅类型

### 通知机制

`sonnet45To46MigrationTimestamp` 时间戳用于：
- 识别需要显示升级通知的用户
- 新用户（`numStartups <= 1`）跳过通知
- 一次性通知，不重复显示

### 分析事件

| 事件名称 | 触发条件 | 元数据 |
|---------|---------|--------|
| `tengu_sonnet45_to_46_migration` | 迁移成功 | `from_model`, `has_1m` |

## 风险、边界与改进建议

### 潜在风险

1. **订阅类型检测失败**：如果订阅信息未加载或过期
   - 缓解：检查函数有缓存和默认值处理

2. **模型字符串匹配遗漏**：硬编码列表可能不完整
   - 风险：用户可能有其他格式的 4.5 字符串
   - 建议：使用正则或更灵活的匹配

3. **通知跳过逻辑**：新用户定义为 `numStartups > 1`
   - 风险：如果启动计数重置，可能错误通知
   - 缓解：时间戳本身也用于去重

### 边界情况

| 场景 | 行为 |
|-----|------|
| 第三方用户 | 跳过 |
| 非目标订阅类型 | 跳过 |
| 使用其他模型 | 跳过 |
| 使用 `sonnet` 别名 | 跳过（已是目标）|
| 新用户（首次启动） | 迁移但不设置时间戳 |
| 老用户 | 迁移并设置时间戳 |

### 改进建议

1. **模型匹配灵活性**：当前硬编码列表
   ```typescript
   // 建议使用更灵活的匹配
   const isSonnet45 = model.match(/sonnet-4-5|claude-sonnet-4-5/)
   ```

2. **幂等性标记**：当前依赖模型值检查
   - 建议：添加 `sonnet45To46MigrationComplete` 标记
   - 好处：避免重复检查，支持统计

3. **错误处理**：当前无 try-catch
   - 建议：添加错误处理和日志

4. **回滚支持**：如果用户想回到 4.5
   - 当前：用户可手动更改
   - 建议：考虑添加降级路径

5. **批量迁移优化**：当前逐个检查
   - 建议：考虑与其他模型迁移合并
