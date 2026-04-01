# migrateFennecToOpus.ts 研究文档

## 场景与职责

本迁移文件负责将已弃用的 "fennec" 模型别名迁移到新的 Opus 4.6 别名。这是内部模型代号（codename）清理工作的一部分，仅针对 Anthropic 内部员工（`USER_TYPE === 'ant'`）。

**业务背景**：
- "fennec" 是 Claude Opus 4.6 的内部开发代号
- 内部员工可能在其设置中使用了这些别名
- 需要将 fennec 别名映射到正式的 Opus 别名，同时保留特殊功能（如 fast mode）

## 功能点目的

1. **别名映射**：将 fennec 别名转换为正式的 Opus 别名
2. **功能保留**：fennec-fast-latest 需要同时启用 fast mode
3. **内部限制**：仅对 Anthropic 员工执行迁移
4. **幂等性**：通过读写同一数据源实现无标记幂等

## 具体技术实现

### 关键流程

```
1. 检查 USER_TYPE === 'ant'（仅内部员工）
2. 读取 userSettings
3. 检查 model 字段
4. 根据模型字符串匹配规则进行替换：
   - fennec-latest[1m] → opus[1m]
   - fennec-latest → opus
   - fennec-fast-latest / opus-4-5-fast → opus[1m] + fastMode: true
```

### 映射规则

| 原模型字符串 | 新模型字符串 | 附加设置 |
|-----------|------------|---------|
| `fennec-latest[1m]` | `opus[1m]` | 无 |
| `fennec-latest` | `opus` | 无 |
| `fennec-fast-latest` | `opus[1m]` | `fastMode: true` |
| `opus-4-5-fast` | `opus[1m]` | `fastMode: true` |

### 核心代码逻辑

```typescript
export function migrateFennecToOpus(): void {
  // 仅内部员工
  if (process.env.USER_TYPE !== 'ant') {
    return
  }

  const settings = getSettingsForSource('userSettings')
  const model = settings?.model

  if (typeof model === 'string') {
    if (model.startsWith('fennec-latest[1m]')) {
      updateSettingsForSource('userSettings', { model: 'opus[1m]' })
    } else if (model.startsWith('fennec-latest')) {
      updateSettingsForSource('userSettings', { model: 'opus' })
    } else if (
      model.startsWith('fennec-fast-latest') ||
      model.startsWith('opus-4-5-fast')
    ) {
      updateSettingsForSource('userSettings', {
        model: 'opus[1m]',
        fastMode: true,
      })
    }
  }
}
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../utils/settings/settings.js` | `getSettingsForSource`, `updateSettingsForSource` | 读写用户设置 |

### 调用上下文

该迁移在应用启动时执行，通常在模型系统初始化之后。

## 依赖与外部交互

### 设置系统

- **目标源**：`userSettings`（`~/.claude/settings.json`）
- **故意排除**：`projectSettings`、`localSettings`、`policySettings`
- **原因**：无法重写这些来源的设置，且读取合并设置会导致无限重运行和全局提升问题

### 安全考虑

**重要设计决策**：
```typescript
/**
 * Only touches userSettings. Reading and writing the same source keeps this
 * idempotent without a completion flag. Fennec aliases in project/local/policy
 * settings are left alone — we can't rewrite those, and reading merged
 * settings here would cause infinite re-runs + silent global promotion.
 */
```

## 风险、边界与改进建议

### 潜在风险

1. **内部代码泄露**：文件包含内部代号（fennec）
   - 缓解：代号已在发布前通过构建流程处理，且文件仅对内部员工执行

2. **部分匹配问题**：使用 `startsWith` 可能匹配意外字符串
   - 例如：`fennec-latest-custom` 会被匹配
   - 当前行为：按顺序检查，先检查 `[1m]` 变体

3. **Fast Mode 冲突**：如果用户已有 fastMode 设置
   - 当前行为：直接覆盖为 `true`
   - 潜在问题：可能覆盖用户显式禁用的 fast mode

### 边界情况

| 场景 | 行为 |
|-----|------|
| `USER_TYPE !== 'ant'` | 立即返回 |
| `model` 不存在 | 不操作 |
| `model` 不是字符串 | 不操作 |
| 部分匹配（如 `fennec-latest-custom`） | 匹配 `fennec-latest` 规则 |
| 设置写入失败 | 静默失败（无错误处理）|

### 改进建议

1. **精确匹配**：当前 `startsWith` 可能过于宽松
   ```typescript
   // 建议改为精确匹配或更严格的模式
   if (model === 'fennec-latest[1m]' || model.startsWith('fennec-latest[1m]:'))
   ```

2. **Fast Mode 处理**：当前直接覆盖
   - 建议：检查现有 fastMode 值，仅在未设置时启用

3. **错误处理**：无 try-catch
   - 建议：添加错误处理和日志记录

4. **迁移标记**：虽然设计为幂等，但添加标记可：
   - 避免重复检查（性能优化）
   - 支持迁移统计

5. **文档化**：内部代号应在代码中明确标注为内部使用
   - 当前：注释说明，但代码中仍有硬编码字符串
