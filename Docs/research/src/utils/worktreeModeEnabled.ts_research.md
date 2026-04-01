# worktreeModeEnabled.ts 研究文档

## 场景与职责

`worktreeModeEnabled.ts` 是一个极简的功能开关模块，用于控制 Claude Code CLI 的 worktree 模式是否启用。

**历史背景（来自代码注释）：**
- 此前通过 GrowthBook flag `'tengu_worktree_mode'` 控制
- 但 `CACHED_MAY_BE_STALE` 模式在首次启动时返回默认值（false），导致 `--worktree` 参数被静默忽略
- 参见 GitHub Issue #27044

**当前状态：**
- Worktree 模式现在无条件为所有用户启用
- 该模块保留作为功能开关的抽象层，便于未来需要时重新引入条件控制

## 功能点目的

### 功能开关 (`isWorktreeModeEnabled`)
- **目的**：提供统一的 worktree 模式启用状态查询
- **当前实现**：始终返回 `true`
- **历史实现**：曾检查 GrowthBook feature flag

## 具体技术实现

### 核心代码

```typescript
/**
 * Worktree mode is now unconditionally enabled for all users.
 *
 * Previously gated by GrowthBook flag 'tengu_worktree_mode', but the
 * CACHED_MAY_BE_STALE pattern returns the default (false) on first launch
 * before the cache is populated, silently swallowing --worktree.
 * See https://github.com/anthropics/claude-code/issues/27044.
 */
export function isWorktreeModeEnabled(): boolean {
  return true
}
```

### 设计模式

这是一个典型的"功能开关"（Feature Toggle）模式：

```
调用方 → isWorktreeModeEnabled() → boolean
                              ↓
                    当前: 始终返回 true
                    未来: 可能重新引入条件检查
```

## 关键代码路径与文件引用

### 导出函数
- `src/utils/worktreeModeEnabled.ts:9` - `isWorktreeModeEnabled()`

### 调用方（预期）
- CLI 参数解析（`--worktree` 处理）
- 功能可用性检查
- UI 显示控制（如显示/隐藏 worktree 相关选项）

## 依赖与外部交互

### 外部依赖
无外部依赖。

### 内部依赖
无内部依赖。

### 历史依赖（已移除）
- ~~GrowthBook feature flag~~
- ~~`tengu_worktree_mode`~~

## 风险、边界与改进建议

### 已知风险

1. **无条件启用风险**
   - 所有用户现在都可以使用 worktree 功能
   - 如果功能有 bug，影响范围是 100%
   - 但鉴于功能已经成熟，风险可控

2. **无法临时禁用**
   - 没有环境变量或配置项可以禁用 worktree 模式
   - 如果用户遇到问题，无法通过配置规避

### 边界情况

1. **非 Git 仓库**
   - `isWorktreeModeEnabled()` 返回 `true` 不代表 worktree 一定可用
   - 实际创建工作区时还需要检查是否在 git 仓库中
   - 这是设计行为，功能开关与功能可用性分离

### 改进建议

1. **保留条件控制接口**
   ```typescript
   export function isWorktreeModeEnabled(): boolean {
     // 为未来预留：环境变量覆盖
     if (process.env.CLAUDE_CODE_WORKTREE_MODE === 'false') {
       return false
     }
     return true
   }
   ```

2. **遥测统计**
   - 添加 worktree 功能使用统计
   - 监控功能采用率和潜在问题

3. **文档更新**
   - 更新用户文档说明 worktree 模式始终可用
   - 移除关于功能开关的过时说明

4. **代码清理**
   - 如果确定不再需要条件控制，考虑内联此函数
   - 但保留作为抽象层有助于代码可读性

5. **A/B 测试支持**
   - 如果未来需要重新引入条件控制，考虑支持渐进式发布
   - 例如按用户 ID 哈希值百分比启用
