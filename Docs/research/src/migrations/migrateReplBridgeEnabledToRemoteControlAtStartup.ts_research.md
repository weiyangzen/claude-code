# migrateReplBridgeEnabledToRemoteControlAtStartup.ts 研究文档

## 场景与职责

本迁移文件负责将全局配置中的 `replBridgeEnabled` 键迁移到新的 `remoteControlAtStartup` 键。这是远程控制功能配置重构的一部分，将内部实现细节泄漏的配置键替换为用户友好的命名。

**业务背景**：
- `replBridgeEnabled` 是内部实现细节（REPL Bridge）
- 该配置实际上控制"启动时远程控制"功能
- 新名称 `remoteControlAtStartup` 更准确地反映用户意图
- 属于配置命名规范化和用户体验改进

## 功能点目的

1. **命名规范化**：将内部实现细节替换为用户友好的配置名
2. **值复制**：将旧值复制到新键
3. **配置清理**：移除旧的实现细节键
4. **幂等性**：仅在旧键存在且新键不存在时执行

## 具体技术实现

### 关键流程

```
1. 调用 saveGlobalConfig 并传入 updater 函数
2. 在 updater 中：
   a. 通过类型转换访问旧键（已从类型定义中移除）
   b. 检查旧键是否存在
   c. 检查新键是否已设置
   d. 如条件满足，复制值并删除旧键
3. 返回更新后的配置
```

### 数据结构

**旧键**（已从 `GlobalConfig` 类型中移除）：
```typescript
// 不再在类型定义中，通过类型转换访问
replBridgeEnabled?: boolean
```

**新键**（`GlobalConfig` 中）：
```typescript
remoteControlAtStartup?: boolean  // 控制启动时是否启用远程控制
```

### 核心代码逻辑

```typescript
export function migrateReplBridgeEnabledToRemoteControlAtStartup(): void {
  saveGlobalConfig(prev => {
    // 旧键已从类型定义中移除，通过类型转换访问
    const oldValue = (prev as Record<string, unknown>)['replBridgeEnabled']
    
    // 旧键不存在，无需迁移
    if (oldValue === undefined) return prev
    
    // 新键已设置，保留用户显式设置
    if (prev.remoteControlAtStartup !== undefined) return prev
    
    // 执行迁移：复制值并删除旧键
    const next = { ...prev, remoteControlAtStartup: Boolean(oldValue) }
    delete (next as Record<string, unknown>)['replBridgeEnabled']
    return next
  })
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../utils/config.js` | `saveGlobalConfig` | 原子性配置更新 |

### 调用上下文

该迁移在应用启动时执行，通常在配置系统初始化之后。

## 依赖与外部交互

### 配置系统

**saveGlobalConfig** 特性：
- 提供原子性更新（通过 updater 函数模式）
- 内部使用文件锁防止竞态条件
- 支持缓存更新

### 类型处理

由于 `replBridgeEnabled` 已从 `GlobalConfig` 类型中移除：
```typescript
// 使用类型转换访问
const oldValue = (prev as Record<string, unknown>)['replBridgeEnabled']
```

这是 TypeScript 中处理已弃用字段迁移的常见模式。

### 幂等性保证

```typescript
if (oldValue === undefined) return prev  // 已迁移或从未设置
if (prev.remoteControlAtStartup !== undefined) return prev  // 新键已存在
```

通过这两个检查确保：
1. 已完成的迁移不会重复执行
2. 用户显式设置的新值不会被覆盖

## 风险、边界与改进建议

### 潜在风险

1. **类型安全**：使用 `as Record<string, unknown>` 绕过类型检查
   - 风险：如果字段名拼写错误，编译器不会报错
   - 缓解：仔细测试，注释说明

2. **竞态条件**：虽然 `saveGlobalConfig` 有锁，但读取-修改-写入周期可能与其他进程冲突
   - 缓解：updater 函数模式确保原子性，但非事务性

3. **值转换**：使用 `Boolean(oldValue)` 转换
   - 风险：非布尔值（如字符串 `"false"`）会被转换为 `true`
   - 当前：原字段类型为 `boolean`，风险较低

### 边界情况

| 场景 | 行为 |
|-----|------|
| `replBridgeEnabled` 不存在 | 不操作 |
| `replBridgeEnabled === undefined` | 不操作 |
| `remoteControlAtStartup` 已设置 | 保留现有值，仅删除旧键 |
| `replBridgeEnabled === true` | 设置为 `true` |
| `replBridgeEnabled === false` | 设置为 `false` |
| 配置写入失败 | `saveGlobalConfig` 内部处理错误 |

### 改进建议

1. **验证值类型**：当前直接转换为 Boolean
   ```typescript
   // 建议添加类型验证
   const oldValue = (prev as Record<string, unknown>)['replBridgeEnabled']
   if (typeof oldValue !== 'boolean') {
     // 记录警告或使用默认值
   }
   ```

2. **迁移标记**：虽然设计为幂等，但添加标记可：
   - 支持迁移统计
   - 便于调试和故障排除

3. **日志记录**：当前无日志
   - 建议：添加调试日志记录迁移行为

4. **向后兼容性**：当前立即删除旧键
   - 建议：考虑保留旧键一段时间，或添加备份机制

5. **文档化**：注释已说明设计意图
   - 建议：在配置文档中添加迁移说明
