# resetProToOpusDefault.ts 研究文档

## 场景与职责

本迁移文件负责处理 Pro 用户到 Opus 默认模型的过渡。这是 Opus 4.5 发布时的订阅者体验调整的一部分，确保 Pro 用户在第一方平台上获得适当的默认模型设置。

**业务背景**：
- Opus 4.5 发布时调整了默认模型策略
- Pro 第一方用户自动迁移到 Opus 4.5 默认
- 需要区分"使用默认模型"的用户和"自定义模型"的用户
- 仅针对第一方 Pro 用户

## 功能点目的

1. **默认模型升级**：为使用默认模型的 Pro 用户启用 Opus 默认
2. **用户分类**：区分使用默认的用户和自定义模型的用户
3. **通知支持**：为使用默认的用户设置时间戳以显示通知
4. **范围限制**：仅针对第一方 Pro 用户

## 具体技术实现

### 关键流程

```
1. 读取全局配置
2. 检查 opusProMigrationComplete 标记
3. 检查 API 提供商是否为 firstParty
4. 检查是否为 Pro 订阅者
5. 如不符合条件，设置标记并记录跳过事件
6. 读取用户设置
7. 如 model === undefined（使用默认），设置时间戳
8. 如 model !== undefined（自定义模型），仅设置标记
9. 记录分析事件
```

### 数据结构

**GlobalConfig 相关字段**：
```typescript
type GlobalConfig = {
  opusProMigrationComplete?: boolean    // 迁移完成标记
  opusProMigrationTimestamp?: number    // 迁移时间戳（用于通知）
  // ... 其他字段
}
```

**Settings 相关字段**：
```typescript
type SettingsJson = {
  model?: string  // 用户指定的模型，undefined 表示使用默认
  // ... 其他字段
}
```

### 核心代码逻辑

```typescript
export function resetProToOpusDefault(): void {
  const config = getGlobalConfig()

  // 检查完成标记
  if (config.opusProMigrationComplete) {
    return
  }

  const apiProvider = getAPIProvider()

  // 仅第一方 Pro 用户
  if (apiProvider !== 'firstParty' || !isProSubscriber()) {
    saveGlobalConfig(current => ({
      ...current,
      opusProMigrationComplete: true,
    }))
    logEvent('tengu_reset_pro_to_opus_default', { skipped: true })
    return
  }

  const settings = getSettings_DEPRECATED()

  // 检查是否使用默认模型
  if (settings?.model === undefined) {
    // 使用默认模型的用户：设置时间戳以显示通知
    const opusProMigrationTimestamp = Date.now()
    saveGlobalConfig(current => ({
      ...current,
      opusProMigrationComplete: true,
      opusProMigrationTimestamp,
    }))
    logEvent('tengu_reset_pro_to_opus_default', {
      skipped: false,
      had_custom_model: false,
    })
  } else {
    // 自定义模型的用户：仅设置标记
    saveGlobalConfig(current => ({
      ...current,
      opusProMigrationComplete: true,
    }))
    logEvent('tengu_reset_pro_to_opus_default', {
      skipped: false,
      had_custom_model: true,
    })
  }
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `src/services/analytics/index.js` | `logEvent` | 记录迁移事件 |
| `../utils/auth.js` | `isProSubscriber` | 检查 Pro 订阅状态 |
| `../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读写全局配置 |
| `../utils/model/providers.js` | `getAPIProvider` | 检查 API 提供商 |
| `../utils/settings/settings.js` | `getSettings_DEPRECATED` | 读取用户设置 |

### 依赖函数详解

**getSettings_DEPRECATED**（来自 `src/utils/settings/settings.ts`）：
```typescript
/**
 * @deprecated Use getInitialSettings() instead. This alias exists for backwards compatibility.
 */
export const getSettings_DEPRECATED = getInitialSettings
```
- 获取合并后的设置（所有源）
- 用于检查用户是否有自定义模型设置

**isProSubscriber**（来自 `src/utils/auth.ts`）：
- 检查用户是否为 Pro 订阅者
- 基于 OAuth 令牌中的订阅信息

## 依赖与外部交互

### 订阅系统

迁移依赖准确的订阅类型检测：
1. OAuth 登录时获取订阅信息
2. `isProSubscriber()` 解析订阅类型
3. 区分 Pro、Max、Team Premium 等不同订阅级别

### 模型系统

通过 `getSettings_DEPRECATED()` 检查模型设置：
- `model === undefined`：用户未指定，使用默认
- `model !== undefined`：用户有自定义模型偏好

### 通知机制

`opusProMigrationTimestamp` 时间戳用于：
- 识别需要显示模型升级通知的用户
- 仅针对使用默认模型的用户
- 自定义模型的用户不会收到通知（保留其选择）

### 分析事件

| 事件名称 | 触发条件 | 元数据 |
|---------|---------|--------|
| `tengu_reset_pro_to_opus_default` | 迁移执行 | `skipped`, `had_custom_model` |

事件变体：
- `{ skipped: true }`：不符合条件的用户
- `{ skipped: false, had_custom_model: false }`：使用默认模型的 Pro 用户
- `{ skipped: false, had_custom_model: true }`：自定义模型的 Pro 用户

## 风险、边界与改进建议

### 潜在风险

1. **订阅类型误判**：如果订阅检测失败
   - 风险：非 Pro 用户可能被错误分类
   - 缓解：`isProSubscriber()` 有缓存和验证

2. **模型设置检测**：使用 `getSettings_DEPRECATED()` 读取合并设置
   - 风险：项目级模型设置可能被误判为用户自定义
   - 当前行为：保守处理，有项目设置视为自定义

3. **时间戳含义**：`opusProMigrationTimestamp` 仅对使用默认的用户设置
   - 风险：查询时可能误解为所有用户的迁移时间
   - 缓解：结合 `had_custom_model` 分析事件理解

### 边界情况

| 场景 | 行为 |
|-----|------|
| 迁移已完成 | 立即返回 |
| 第三方用户 | 设置标记，记录 skipped: true |
| 非 Pro 用户 | 设置标记，记录 skipped: true |
| Pro + 使用默认模型 | 设置标记 + 时间戳，记录 had_custom_model: false |
| Pro + 自定义模型 | 仅设置标记，记录 had_custom_model: true |
| 设置读取失败 | 视为 undefined（使用默认）|

### 改进建议

1. **模型来源检查**：当前使用合并设置
   ```typescript
   // 建议：仅检查 userSettings
   const userModel = getSettingsForSource('userSettings')?.model
   ```

2. **错误处理**：当前无 try-catch
   - 建议：添加错误处理，特别是配置写入

3. **日志记录**：当前仅依赖分析事件
   - 建议：添加调试日志便于故障排除

4. **通知细化**：当前二元分类（默认/自定义）
   - 建议：考虑区分显式设置默认 vs 真正未设置

5. **与其他迁移协调**：存在多个模型相关迁移
   - 建议：确保执行顺序正确，避免冲突
