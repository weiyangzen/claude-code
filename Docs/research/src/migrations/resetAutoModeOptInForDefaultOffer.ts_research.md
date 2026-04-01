# resetAutoModeOptInForDefaultOffer.ts 研究文档

## 场景与职责

本迁移文件负责为特定用户重置 `skipAutoPermissionPrompt` 设置，使他们重新看到 Auto Mode Opt-In 对话框。这是 Auto Mode 功能推广策略的一部分，针对之前接受旧版对话框但默认模式不是 auto 的用户。

**业务背景**：
- Auto Mode 是 Claude Code 的自动权限模式（类似 YOLO 模式的安全版本）
- 早期版本的 Opt-In 对话框只有 2 个选项
- 新版本增加了"设为默认模式"选项
- 需要让之前选择旧版对话框的用户看到新选项

## 功能点目的

1. **重新提示**：让符合条件的用户重新看到 Opt-In 对话框
2. **条件限制**：仅针对启用了 Auto Mode 的用户（`enabled` 状态）
3. **目标筛选**：仅针对 `skipAutoPermissionPrompt=true` 但默认模式不是 `auto` 的用户
4. **一次性**：使用 GlobalConfig 标记确保只执行一次

## 具体技术实现

### 关键流程

```
1. 检查 TRANSCRIPT_CLASSIFIER 功能标志
2. 读取全局配置
3. 检查 hasResetAutoModeOptInForDefaultOffer 标记
4. 检查 getAutoModeEnabledState() === 'enabled'
5. 读取 userSettings
6. 检查 skipAutoPermissionPrompt && defaultMode !== 'auto'
7. 如条件满足，清除 skipAutoPermissionPrompt
8. 记录分析事件
9. 设置 hasResetAutoModeOptInForDefaultOffer 标记
```

### 数据结构

**GlobalConfig 标记**：
```typescript
type GlobalConfig = {
  hasResetAutoModeOptInForDefaultOffer?: boolean  // 迁移完成标记
  // ... 其他字段
}
```

**Settings 相关字段**：
```typescript
type SettingsJson = {
  skipAutoPermissionPrompt?: boolean      // 是否跳过权限提示
  permissions?: {
    defaultMode?: 'auto' | 'default' | ... // 默认权限模式
  }
  // ... 其他字段
}
```

### 核心代码逻辑

```typescript
export function resetAutoModeOptInForDefaultOffer(): void {
  // 功能标志检查
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const config = getGlobalConfig()
    
    // 完成标记检查
    if (config.hasResetAutoModeOptInForDefaultOffer) return
    
    // Auto Mode 状态检查
    if (getAutoModeEnabledState() !== 'enabled') return

    try {
      const user = getSettingsForSource('userSettings')
      
      // 目标用户筛选
      if (
        user?.skipAutoPermissionPrompt &&
        user?.permissions?.defaultMode !== 'auto'
      ) {
        // 清除跳过标记，重新显示对话框
        updateSettingsForSource('userSettings', {
          skipAutoPermissionPrompt: undefined,
        })
        logEvent('tengu_migrate_reset_auto_opt_in_for_default_offer', {})
      }

      // 设置完成标记
      saveGlobalConfig(c => {
        if (c.hasResetAutoModeOptInForDefaultOffer) return c
        return { ...c, hasResetAutoModeOptInForDefaultOffer: true }
      })
    } catch (error) {
      logError(new Error(`Failed to reset auto mode opt-in: ${error}`))
    }
  }
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `bun:bundle` | `feature` | 功能标志检查 |
| `src/services/analytics/index.js` | `logEvent` | 记录迁移事件 |
| `../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读写全局配置 |
| `../utils/log.js` | `logError` | 错误日志记录 |
| `../utils/permissions/permissionSetup.js` | `getAutoModeEnabledState` | 检查 Auto Mode 状态 |
| `../utils/settings/settings.js` | `getSettingsForSource`, `updateSettingsForSource` | 读写用户设置 |

### 依赖函数详解

**getAutoModeEnabledState**（来自 `src/utils/permissions/permissionSetup.ts`）：
```typescript
export function getAutoModeEnabledState(): 'enabled' | 'opt-in' | 'disabled' {
  // 检查功能开关和配置
  // 返回 Auto Mode 的当前状态
}
```

**feature('TRANSCRIPT_CLASSIFIER')**：
- Bun 构建时功能标志
- 控制是否包含 Auto Mode 相关代码
- 生产环境通常启用

## 依赖与外部交互

### Auto Mode 系统

迁移与 Auto Mode 权限系统紧密集成：
1. `getAutoModeEnabledState()` 检查功能是否启用
2. `skipAutoPermissionPrompt` 控制是否显示 Opt-In 对话框
3. `permissions.defaultMode` 存储用户的默认模式选择

### 安全考虑

**重要设计决策**（来自注释）：
```typescript
/**
 * Only runs when tengu_auto_mode_config.enabled === 'enabled'. For 'opt-in'
 * users, clearing skipAutoPermissionPrompt would remove auto from the carousel
 * (permissionSetup.ts:988) — the dialog would become unreachable and the
 * migration would defeat itself.
 */
```

- 仅对 `'enabled'` 状态的用户执行
- `'opt-in'` 用户跳过，因为清除标记会使对话框不可达

### 分析事件

| 事件名称 | 触发条件 | 元数据 |
|---------|---------|--------|
| `tengu_migrate_reset_auto_opt_in_for_default_offer` | 重置成功 | 无 |

## 风险、边界与改进建议

### 潜在风险

1. **功能标志依赖**：代码包裹在 `feature('TRANSCRIPT_CLASSIFIER')` 中
   - 风险：如果构建时标志未启用，整个迁移被排除
   - 缓解：生产构建通常启用

2. **状态竞争**：如果用户在迁移执行期间更改设置
   - 风险：可能覆盖用户新选择
   - 缓解：通常启动时执行，用户交互前完成

3. **标记位置**：完成标记在 GlobalConfig 而非设置中
   - 风险：设置重置后不会重新执行
   - 设计决策：有意为之，避免无限循环

### 边界情况

| 场景 | 行为 |
|-----|------|
| TRANSCRIPT_CLASSIFIER 禁用 | 整个代码块被排除 |
| 迁移已完成 | 立即返回 |
| Auto Mode 状态 !== 'enabled' | 跳过（包括 'opt-in' 和 'disabled'）|
| skipAutoPermissionPrompt === false | 不重置 |
| defaultMode === 'auto' | 不重置（已是目标状态）|
| 设置读取失败 | 记录错误，不设置标记 |

### 改进建议

1. **错误恢复**：当前 catch 块仅记录错误
   ```typescript
   // 建议：区分可恢复和不可恢复错误
   // 考虑在失败时重试或延迟执行
   ```

2. **用户通知**：静默重置可能让用户困惑
   - 建议：在重新显示的对话框中说明原因

3. **条件细化**：当前仅检查 `defaultMode !== 'auto'`
   - 建议：考虑检查用户是否实际看到过旧版对话框

4. **测试覆盖**：建议添加场景测试：
   - 各种 Auto Mode 状态组合
   - 设置读写失败处理
   - 多次执行的幂等性

5. **文档化**：注释已详细说明设计意图
   - 建议：在功能文档中添加迁移说明
