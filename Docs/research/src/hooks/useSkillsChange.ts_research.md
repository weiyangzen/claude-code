# useSkillsChange.ts 深度研究文档

## 场景与职责

`useSkillsChange` 是一个 React Hook，用于监听技能文件的变化和 GrowthBook 配置的刷新，确保命令列表始终保持最新。当技能内容或功能开关状态变化时，它会重新加载命令列表。

### 核心职责

1. **技能文件变化监听**: 监听技能文件系统的变化
2. **GrowthBook 刷新监听**: 监听远程功能开关的刷新
3. **命令缓存管理**: 清除缓存并重新加载命令
4. **错误处理**: 处理重新加载过程中的错误

### 使用场景

- **技能开发**: 开发者在编辑技能文件时自动刷新命令
- **功能开关**: GrowthBook 功能开关状态变化时更新命令可用性
- **动态技能**: 运行时添加或移除技能

---

## 功能点目的

### 1. 技能文件变化处理

当技能文件在磁盘上变化时：
- 清除所有命令缓存（`clearCommandsCache`）
- 重新扫描磁盘加载技能
- 获取更新后的命令列表

### 2. GrowthBook 刷新处理

当 GrowthBook 初始化或刷新时：
- 仅清除记忆化缓存（`clearCommandMemoizationCaches`）
- 重新评估 `isEnabled()` 谓词
- 获取更新后的命令列表

### 3. 错误恢复

重新加载过程中的错误不会中断应用：
- 捕获并记录错误
- 保持现有命令列表继续运行

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface UseSkillsChangeProps {
  cwd: string | undefined                    // 当前工作目录
  onCommandsChange: (commands: Command[]) => void  // 命令变化回调
}

// Command 类型
interface Command {
  name: string
  isEnabled: () => boolean                   // 功能开关检查
  // ... 其他字段
}
```

### 核心流程

#### 1. 技能文件变化处理流程
```
skillChangeDetector 检测到变化
  ↓
调用 handleChange
  ↓
检查 cwd 是否存在
  ↓
clearCommandsCache()  // 清除所有缓存
  ↓
getCommands(cwd)  // 重新加载命令
  ↓
onCommandsChange(commands)  // 通知父组件
```

**代码实现**（行 28-41）：
```typescript
const handleChange = useCallback(async () => {
  if (!cwd) return
  try {
    // Clear all command caches to ensure fresh load
    clearCommandsCache()
    const commands = await getCommands(cwd)
    onCommandsChange(commands)
  } catch (error) {
    // Errors during reload are non-fatal - log and continue
    if (error instanceof Error) {
      logError(error)
    }
  }
}, [cwd, onCommandsChange])

useEffect(() => skillChangeDetector.subscribe(handleChange), [handleChange])
```

#### 2. GrowthBook 刷新处理流程
```
GrowthBook 初始化/刷新
  ↓
调用 handleGrowthBookRefresh
  ↓
检查 cwd 是否存在
  ↓
clearCommandMemoizationCaches()  // 仅清除记忆化缓存
  ↓
getCommands(cwd)  // 重新加载命令
  ↓
onCommandsChange(commands)  // 通知父组件
```

**代码实现**（行 45-61）：
```typescript
const handleGrowthBookRefresh = useCallback(async () => {
  if (!cwd) return
  try {
    clearCommandMemoizationCaches()
    const commands = await getCommands(cwd)
    onCommandsChange(commands)
  } catch (error) {
    if (error instanceof Error) {
      logError(error)
    }
  }
}, [cwd, onCommandsChange])

useEffect(
  () => onGrowthBookRefresh(handleGrowthBookRefresh),
  [handleGrowthBookRefresh],
)
```

### 关键代码路径

#### 缓存清除的区别

**技能文件变化**（行 32）：
```typescript
clearCommandsCache()  // 完全清除，包括磁盘缓存
```

**GrowthBook 刷新**（行 48）：
```typescript
clearCommandMemoizationCaches()  // 仅清除记忆化缓存
```

区别原因（来自注释）：
- 技能文件变化：内容在磁盘上改变，需要完全重新扫描
- GrowthBook 刷新：只有 `isEnabled()` 谓词可能变化，内容不变

#### GrowthBook 时序问题

注释解释了特殊处理的原因：
```typescript
// Skill file changes (watcher) — full cache clear + disk re-scan, since
// skill content changed on disk.
// GrowthBook init/refresh — memo-only clear, since only `isEnabled()`
// predicates may have changed. Handles commands like /btw whose gate
// reads a flag that isn't in the disk cache yet on first session after
// a flag rename: getCommands() runs before GB init (main.tsx:2855 vs
// showSetupScreens at :3106), so the memoized list is baked with the
// default. Once init populates remoteEvalFeatureValues, re-filter.
```

关键点：
- `getCommands()` 在 GrowthBook 初始化之前运行
- 初始命令列表使用功能开关的默认值
- GrowthBook 初始化后需要重新过滤

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `../commands.js` | `Command`, `clearCommandsCache`, `clearCommandMemoizationCaches`, `getCommands` |
| `../services/analytics/growthbook.js` | `onGrowthBookRefresh` |
| `../utils/log.js` | `logError` |
| `../utils/skills/skillChangeDetector.js` | `skillChangeDetector` |

### 外部交互

1. **skillChangeDetector**: 
   - 订阅技能文件系统变化
   - 变化时触发 `handleChange`

2. **GrowthBook**: 
   - 订阅功能开关刷新
   - 刷新时触发 `handleGrowthBookRefresh`

3. **命令系统**: 
   - `getCommands(cwd)`: 加载所有可用命令
   - `clearCommandsCache()`: 完全清除缓存
   - `clearCommandMemoizationCaches()`: 仅清除记忆化缓存

4. **父组件**: 
   - `onCommandsChange`: 通知命令列表更新

---

## 风险、边界与改进建议

### 已知风险

1. **重复加载**: 技能变化和 GrowthBook 刷新同时发生时可能导致两次加载
2. **竞态条件**: 快速连续的变化可能导致过期的命令列表覆盖新的
3. **错误静默**: 加载错误仅记录日志，用户无感知

### 边界情况

1. **cwd 变化**: 工作目录变化时的处理
2. **空命令列表**: 加载失败或返回空列表时的行为
3. **并发加载**: 多个变化事件同时到达时的处理

### 改进建议

1. **防抖处理**: 对快速连续的变化进行防抖，避免重复加载
2. **加载状态**: 添加加载状态指示，让用户知道命令正在更新
3. **错误提示**: 加载失败时向用户显示提示
4. **增量更新**: 只重新加载变化的技能，而不是全部
5. **取消机制**: 支持取消过期的加载请求
6. **乐观更新**: 先更新 UI，后台异步验证

### 测试关注点

1. 技能文件变化时的命令重新加载
2. GrowthBook 刷新时的命令更新
3. 两种触发器同时发生时的行为
4. 加载错误时的错误处理
5. cwd 变化时的处理
6. 组件卸载时的取消订阅
