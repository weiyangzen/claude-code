# changeDetector.ts 研究文档

## 场景与职责

`changeDetector.ts` 是 Claude Code 设置系统的文件监控和变更通知核心模块。负责：

1. **文件系统监控** - 使用 chokidar 监控设置文件的变化
2. **MDM 设置轮询** - 定期轮询 MDM（移动设备管理）设置的变化
3. **变更通知** - 当检测到变更时，触发配置变更 hooks 并通知所有订阅者
4. **内部写入过滤** - 区分外部变更和 Claude Code 自身的写入操作
5. **删除-重建模式处理** - 处理文件删除后快速重建的常见模式

## 功能点目的

### 1. 文件系统监控 (`initialize`)
- **监控范围**: 所有设置源对应的目录（userSettings, projectSettings, localSettings, policySettings）
- **排除项**: flagSettings（临时文件可能包含特殊文件类型）、.git 目录、非文件/目录的特殊文件类型
- **稳定性等待**: 使用 `awaitWriteFinish` 避免处理部分写入

### 2. MDM 设置轮询 (`startMdmPoll`)
- **轮询间隔**: 30 分钟
- **监控内容**: macOS plist 和 Windows 注册表设置
- **快照比较**: 通过 JSON 序列化比较前后状态

### 3. 变更处理流程
- **变更事件**: 触发 ConfigChange hooks，如果被阻止则跳过应用
- **删除事件**: 使用宽限期（DELETION_GRACE_MS）处理删除-重建模式
- **添加事件**: 取消待处理的删除，视为变更处理

### 4. 内部写入过滤 (`consumeInternalWrite`)
- **时间窗口**: 5 秒
- **用途**: 避免 Claude Code 自身写入设置文件时触发不必要的重载

### 5. 变更分发 (`fanOut`)
- **缓存重置**: 在通知所有监听器前统一重置缓存（关键优化，避免 N 次磁盘重载）
- **信号机制**: 使用自定义信号系统通知订阅者

## 具体技术实现

### 关键常量

```typescript
const FILE_STABILITY_THRESHOLD_MS = 1000    // 文件写入稳定等待时间
const FILE_STABILITY_POLL_INTERVAL_MS = 500 // 稳定性检查轮询间隔
const INTERNAL_WRITE_WINDOW_MS = 5000       // 内部写入识别窗口
const MDM_POLL_INTERVAL_MS = 30 * 60 * 1000 // MDM 轮询间隔（30分钟）
const DELETION_GRACE_MS = 1700              // 删除宽限期（1000+500+200）
```

### 关键流程

```
initialize()
  ├── 检查远程模式（跳过）
  ├── startMdmPoll()                    // 启动 MDM 轮询
  ├── registerCleanup(dispose)          // 注册清理
  ├── getWatchTargets()                 // 获取监控目标
  │   ├── 遍历 SETTING_SOURCES
  │   ├── 跳过 flagSettings
  │   └── 收集目录和文件路径
  └── chokidar.watch()
      ├── 配置 ignored 回调（过滤逻辑）
      ├── 监听 'change' → handleChange
      ├── 监听 'unlink' → handleDelete
      └── 监听 'add' → handleAdd

handleChange(path)
  ├── getSourceForPath(path)            // 确定设置源
  ├── 检查并取消待处理的删除
  ├── consumeInternalWrite()            // 检查是否为内部写入
  ├── executeConfigChangeHooks()        // 执行配置变更 hooks
  └── 如果未被阻止 → fanOut(source)

fanOut(source)
  ├── resetSettingsCache()              // 统一重置缓存
  └── settingsChanged.emit(source)      // 通知所有订阅者

handleDelete(path)
  ├── 设置删除宽限期定时器
  └── 宽限期后执行 fanOut（如果未被重建）

startMdmPoll()
  ├── 捕获初始快照
  └── setInterval()
      ├── refreshMdmSettings()          // 刷新 MDM 设置
      ├── 比较快照
      └── 如果变化 → setMdmSettingsCache() + fanOut()
```

### 数据结构

```typescript
// 模块级状态
let watcher: FSWatcher | null = null
let mdmPollTimer: ReturnType<typeof setInterval> | null = null
let lastMdmSnapshot: string | null = null
let initialized = false
let disposed = false
const pendingDeletions = new Map<string, ReturnType<typeof setTimeout>>()
const settingsChanged = createSignal<[source: SettingSource]>()

// 测试覆盖
let testOverrides: {
  stabilityThreshold?: number
  pollInterval?: number
  mdmPollInterval?: number
  deletionGrace?: number
} | null = null
```

### 关键代码路径

| 函数 | 行号 | 说明 |
|------|------|------|
| `initialize` | 84-146 | 初始化文件监控 |
| `dispose` | 154-168 | 清理资源 |
| `handleChange` | 268-302 | 处理文件变更 |
| `handleDelete` | 330-360 | 处理文件删除（含宽限期） |
| `handleAdd` | 308-322 | 处理文件添加 |
| `fanOut` | 437-440 | 分发变更通知 |
| `startMdmPoll` | 381-418 | MDM 轮询 |
| `getWatchTargets` | 180-250 | 获取监控目标 |
| `getSourceForPath` | 362-375 | 路径到设置源的映射 |

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `chokidar` | 外部库 | 文件系统监控 |
| `getIsRemoteMode` | `../../bootstrap/state.js` | 检查远程模式 |
| `registerCleanup` | `../cleanupRegistry.js` | 注册清理函数 |
| `executeConfigChangeHooks` | `../hooks.js` | 执行配置变更 hooks |
| `createSignal` | `../signal.js` | 信号系统 |
| `SETTING_SOURCES` | `./constants.js` | 设置源常量 |
| `internalWrites` | `./internalWrites.js` | 内部写入跟踪 |
| `managedPath` | `./managedPath.js` | 托管设置路径 |
| `mdm/settings` | `./mdm/settings.js` | MDM 设置操作 |
| `settings` | `./settings.js` | 设置文件路径 |
| `settingsCache` | `./settingsCache.js` | 设置缓存 |

### 被调用方

- `src/main.tsx` - 应用启动时初始化
- `src/hooks/useSettingsChange.ts` - 订阅设置变更
- `src/cli/print.ts` - 无头模式订阅
- `src/services/settingsSync/index.ts` - 设置同步
- `src/services/remoteManagedSettings/index.ts` - 远程托管设置
- `src/tools/ConfigTool/ConfigTool.ts` - 配置工具
- `src/commands/reload-plugins/reload-plugins.ts` - 插件重载
- `src/utils/sandbox/sandbox-adapter.ts` - 沙盒适配器

### 导出 API

```typescript
export const settingsChangeDetector = {
  initialize,
  dispose,
  subscribe,        // 订阅设置变更
  notifyChange,     // 手动通知变更（程序化变更）
  resetForTesting,  // 测试重置
}
```

## 风险、边界与改进建议

### 风险点

1. **缓存重置位置关键**: `fanOut` 中的缓存重置必须在通知订阅者之前完成。之前有 bug 是每个订阅者自己重置，导致 N 次磁盘重载。

2. **flagSettings 排除**: 明确排除 flagSettings 监控，因为它们可能是 $TMPDIR 中的临时文件，包含 FIFO、socket 等特殊文件类型会导致监控挂起或报错。

3. **删除宽限期竞争**: `DELETION_GRACE_MS` 必须超过 chokidar 的 `awaitWriteFinish` 延迟，否则重建文件的写入稳定性检查可能在删除处理之前完成。

4. **路径标准化**: chokidar 在 Windows 上使用正斜杠，需要标准化为原生格式进行比较。

### 边界情况

| 场景 | 处理 |
|------|------|
| 远程模式 | 完全跳过初始化 |
| 文件不存在 | 跳过监控该目录 |
| 内部写入 | 5 秒内忽略 |
| 删除-重建 | 宽限期内重建视为变更，否则视为删除 |
| 符号链接 | 支持，但检查目标是否存在 |
| 特殊文件类型 | 忽略（socket、FIFO、设备） |
| .git 目录 | 忽略 |
| 非 .json 文件 | 在 drop-in 目录外忽略 |

### 改进建议

1. **错误处理增强**: 当前对 chokidar 错误的处理较简单，可考虑添加更详细的错误分类和恢复策略
2. **监控性能**: 对于大型项目，监控的目录数量可能增加，可考虑延迟初始化或按需监控
3. **测试覆盖**: `resetForTesting` 提供了测试覆盖点，确保所有时序覆盖都可测试
4. **MDM 轮询优化**: 30 分钟轮询可能过于频繁或稀疏，可考虑根据平台调整
5. **日志增强**: 关键路径的调试日志已存在，但可考虑添加结构化日志以便分析

## 文件引用

- **本文件**: `src/utils/settings/changeDetector.ts`
- **相关文件**:
  - `src/utils/settings/internalWrites.ts` - 内部写入跟踪
  - `src/utils/settings/settingsCache.ts` - 设置缓存
  - `src/utils/settings/mdm/settings.ts` - MDM 设置
  - `src/utils/hooks.js` - Hooks 执行
  - `src/utils/signal.js` - 信号系统
