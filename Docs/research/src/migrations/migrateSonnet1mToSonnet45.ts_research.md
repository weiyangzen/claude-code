# migrateSonnet1mToSonnet45.ts 研究文档

## 场景与职责

本迁移文件负责将使用 `sonnet[1m]` 别名的用户迁移到显式的 `sonnet-4-5-20250929[1m]` 模型字符串。这是 Sonnet 4.6 发布时的模型别名策略调整的一部分，确保原有用户保留其预期的模型行为。

**业务背景**：
- `sonnet` 别名现在解析到 Sonnet 4.6
- 之前设置 `sonnet[1m]` 的用户实际使用的是 Sonnet 4.5 1M
- Sonnet 4.6 1M 向不同的用户群体提供，与 4.5 1M 不完全相同
- 需要将现有 `sonnet[1m]` 用户固定到显式的 4.5 版本

## 功能点目的

1. **模型固定**：将 `sonnet[1m]` 用户固定到 Sonnet 4.5 1M 显式版本
2. **行为保留**：确保用户继续获得与之前相同的模型行为
3. **范围限制**：仅处理 `userSettings`，避免提升项目级设置
4. **内存覆盖**：同时迁移内存中的模型覆盖（如果已设置）

## 具体技术实现

### 关键流程

```
1. 读取全局配置
2. 检查 sonnet1m45MigrationComplete 标记
3. 如已完成，直接返回
4. 读取 userSettings 中的 model 字段
5. 如等于 'sonnet[1m]'，更新为 'sonnet-4-5-20250929[1m]'
6. 检查内存中的 mainLoopModelOverride
7. 如等于 'sonnet[1m]'，更新覆盖
8. 设置 sonnet1m45MigrationComplete 标记
```

### 数据结构

**GlobalConfig 迁移标记**：
```typescript
type GlobalConfig = {
  sonnet1m45MigrationComplete?: boolean  // 迁移完成标记
  // ... 其他字段
}
```

**模型值**：
```typescript
// 旧值
'sonnet[1m]'

// 新值
'sonnet-4-5-20250929[1m]'
```

### 核心代码逻辑

```typescript
export function migrateSonnet1mToSonnet45(): void {
  const config = getGlobalConfig()
  
  // 检查完成标记
  if (config.sonnet1m45MigrationComplete) {
    return
  }

  // 迁移用户设置
  const model = getSettingsForSource('userSettings')?.model
  if (model === 'sonnet[1m]') {
    updateSettingsForSource('userSettings', {
      model: 'sonnet-4-5-20250929[1m]',
    })
  }

  // 迁移内存覆盖
  const override = getMainLoopModelOverride()
  if (override === 'sonnet[1m]') {
    setMainLoopModelOverride('sonnet-4-5-20250929[1m]')
  }

  // 设置完成标记
  saveGlobalConfig(current => ({
    ...current,
    sonnet1m45MigrationComplete: true,
  }))
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../bootstrap/state.js` | `getMainLoopModelOverride`, `setMainLoopModelOverride` | 读写内存模型覆盖 |
| `../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读写全局配置 |
| `../utils/settings/settings.js` | `getSettingsForSource`, `updateSettingsForSource` | 读写用户设置 |

### 依赖函数详解

**getMainLoopModelOverride / setMainLoopModelOverride**（来自 `src/bootstrap/state.ts`）：
```typescript
export function getMainLoopModelOverride(): ModelSetting | undefined {
  return STATE.mainLoopModelOverride
}

export function setMainLoopModelOverride(model: ModelSetting | undefined): void {
  STATE.mainLoopModelOverride = model
}
```
- 内存中的模型覆盖（来自 `--model` CLI 标志或 `/model` 命令）
- 优先级高于设置文件

## 依赖与外部交互

### 模型覆盖优先级

模型选择优先级（从高到低）：
1. `getMainLoopModelOverride()` - 运行时覆盖（`/model` 命令）
2. `--model` CLI 标志
3. `ANTHROPIC_MODEL` 环境变量
4. 设置文件中的 `model`
5. 内置默认

### 迁移范围限制

**重要设计决策**：
```typescript
/**
 * Reads from userSettings specifically (not merged settings) so we don't
 * promote a project-scoped "sonnet[1m]" to the global default.
 */
```

- 仅读取 `userSettings`，不读取合并设置
- 原因：避免将项目级设置提升为全局默认

### 完成标记

使用 `sonnet1m45MigrationComplete` 标记确保：
- 幂等性：多次执行安全
- 性能：避免重复检查
- 可追溯性：可查询迁移状态

## 风险、边界与改进建议

### 潜在风险

1. **项目级设置未迁移**：故意排除 `projectSettings`
   - 风险：项目级 `sonnet[1m]` 设置会解析到 4.6
   - 缓解：注释说明这是有意为之

2. **内存覆盖竞争**：如果迁移期间用户更改模型
   - 风险：可能覆盖用户新选择
   - 缓解：通常启动时执行，用户交互前完成

3. **硬编码模型 ID**：`sonnet-4-5-20250929[1m]` 硬编码
   - 风险：如果模型 ID 格式变化，迁移会失败
   - 缓解：模型 ID 通常稳定

### 边界情况

| 场景 | 行为 |
|-----|------|
| 迁移已完成 | 立即返回 |
| 模型 !== 'sonnet[1m]' | 跳过设置更新 |
| 覆盖 !== 'sonnet[1m]' | 跳过覆盖更新 |
| 两者都不匹配 | 仅设置完成标记 |
| 设置写入失败 | 异常抛出（无 try-catch）|

### 改进建议

1. **错误处理**：当前无 try-catch
   ```typescript
   // 建议添加错误处理
   try {
     // 迁移逻辑
   } catch (error) {
     logError(error)
     // 决定是否继续设置标记
   }
   ```

2. **部分失败处理**：如果设置更新成功但覆盖更新失败
   - 当前：继续执行并设置标记
   - 建议：考虑事务性语义

3. **日志记录**：当前无日志
   - 建议：添加调试日志记录迁移行为

4. **模型 ID 常量**：硬编码字符串
   - 建议：使用 `getModelStrings()` 中的常量

5. **后续迁移路径**：用户后续可能想升级到 4.6 1M
   - 当前：固定到 4.5
   - 建议：提供手动升级路径或后续自动迁移
