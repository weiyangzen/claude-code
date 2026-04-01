# migrateBypassPermissionsAcceptedToSettings.ts 研究文档

## 场景与职责

本迁移文件负责将全局配置中的 `bypassPermissionsModeAccepted` 标志迁移到 `settings.json` 中的 `skipDangerousModePermissionPrompt` 设置。这是权限系统配置重构的一部分，旨在将用户权限首选项统一到 `settings.json` 文件中。

**业务背景**：
- `bypassPermissionsModeAccepted` 标记用户是否接受了"绕过权限模式"（危险模式）
- 迁移后使用更语义化的 `skipDangerousModePermissionPrompt` 名称
- 这是将配置从 `~/.claude.json` 迁移到 `settings.json` 的系列工作之一

## 功能点目的

1. **配置统一化**：将权限相关的用户首选项集中到 `settings.json`
2. **命名规范化**：使用更清晰的 `skipDangerousModePermissionPrompt` 名称
3. **向后兼容**：迁移后清理旧配置字段
4. **避免重复设置**：检查目标设置是否已存在，避免覆盖

## 具体技术实现

### 关键流程

```
1. 读取全局配置
2. 检查 bypassPermissionsModeAccepted 是否存在
3. 检查 settings 中是否已有 skipDangerousModePermissionPrompt
4. 如未设置，写入 userSettings
5. 记录分析事件
6. 从全局配置中移除旧字段
```

### 数据结构

**GlobalConfig 相关字段**：
```typescript
type GlobalConfig = {
  bypassPermissionsModeAccepted?: boolean  // 旧字段
  // ... 其他字段
}
```

**Settings 相关字段**：
```typescript
type SettingsJson = {
  skipDangerousModePermissionPrompt?: boolean  // 新字段
  // ... 其他字段
}
```

### 核心代码逻辑

```typescript
// 检查旧标志是否存在
if (!globalConfig.bypassPermissionsModeAccepted) {
  return  // 无需迁移
}

// 避免重复设置
try {
  if (!hasSkipDangerousModePermissionPrompt()) {
    updateSettingsForSource('userSettings', {
      skipDangerousModePermissionPrompt: true,
    })
  }

  // 记录事件
  logEvent('tengu_migrate_bypass_permissions_accepted', {})

  // 清理旧配置
  saveGlobalConfig(current => {
    if (!('bypassPermissionsModeAccepted' in current)) return current
    const { bypassPermissionsModeAccepted: _, ...updatedConfig } = current
    return updatedConfig
  })
} catch (error) {
  logError(new Error(`Failed to migrate bypass permissions accepted: ${error}`))
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `src/services/analytics/index.js` | `logEvent` | 记录迁移事件 |
| `../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读写全局配置 |
| `../utils/log.js` | `logError` | 错误日志记录 |
| `../utils/settings/settings.js` | `hasSkipDangerousModePermissionPrompt`, `updateSettingsForSource` | 检查和更新设置 |

### 依赖函数详解

**hasSkipDangerousModePermissionPrompt**（来自 `src/utils/settings/settings.ts`）：
```typescript
export function hasSkipDangerousModePermissionPrompt(): boolean {
  return !!(
    getSettingsForSource('userSettings')?.skipDangerousModePermissionPrompt ||
    getSettingsForSource('localSettings')?.skipDangerousModePermissionPrompt ||
    getSettingsForSource('flagSettings')?.skipDangerousModePermissionPrompt ||
    getSettingsForSource('policySettings')?.skipDangerousModePermissionPrompt
  )
}
```
- 检查多个设置源，确保不重复设置
- 排除 `projectSettings`（安全风险：恶意项目可能自动绕过权限对话框）

## 依赖与外部交互

### 配置系统

1. **读取路径**：`getGlobalConfig()` → `~/.claude.json`
2. **写入路径**：`updateSettingsForSource('userSettings')` → `~/.claude/settings.json`
3. **清理路径**：`saveGlobalConfig()` → `~/.claude.json`

### 安全考虑

- `projectSettings` 被排除在检查范围外
- 原因：防止恶意项目通过提交 `.claude/settings.json` 自动绕过权限确认

## 风险、边界与改进建议

### 潜在风险

1. **权限绕过风险**：如果迁移逻辑有误，可能导致用户意外进入危险模式
   - 缓解：严格的条件检查和多重验证

2. **配置不一致**：如果迁移中断，新旧配置可能同时存在
   - 缓解：先检查新配置，再写入，最后清理旧配置

### 边界情况

| 场景 | 行为 |
|-----|------|
| `bypassPermissionsModeAccepted === undefined` | 不迁移 |
| `bypassPermissionsModeAccepted === false` | 不迁移 |
| `skipDangerousModePermissionPrompt` 已存在 | 仅清理旧字段，不覆盖新字段 |
| 写入失败 | 记录错误，保留旧配置 |
| 清理失败 | 已写入新配置，但旧配置残留 |

### 改进建议

1. **原子性操作**：当前实现非原子，可能留下不一致状态
   - 建议：添加迁移标记，支持幂等重试

2. **验证机制**：迁移后验证新配置生效
   - 建议：添加迁移后断言检查

3. **用户通知**：静默迁移可能让用户困惑
   - 建议：添加一次性通知告知配置迁移

4. **回滚机制**：如果新配置导致问题，缺乏回滚能力
   - 建议：保留备份或添加回滚标记
