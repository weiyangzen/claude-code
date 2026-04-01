# skillChangeDetector.ts 深度研究文档

## 1. 场景与职责

### 1.1 定位
`skillChangeDetector.ts` 是 Claude Code 的**技能文件变更检测与热重载模块**，负责监视用户技能目录（skills/commands）的文件变化，并在检测到变更时触发缓存清理和命令列表刷新。

### 1.2 核心职责
| 职责 | 说明 |
|------|------|
| **文件监视** | 使用 chokidar 监视技能目录的增删改事件 |
| **防抖聚合** | 将短时间内的多次变更聚合为单次重载，避免级联刷新 |
| **缓存失效** | 触发技能缓存、命令缓存的清理 |
| **事件通知** | 通过 Signal 机制通知订阅者技能已变更 |
| **生命周期管理** | 提供初始化、销毁、测试重置等生命周期方法 |

### 1.3 使用场景
- **开发时热重载**：用户编辑 `~/.claude/skills/my-skill/SKILL.md` 后自动刷新
- **Git 操作后同步**：切换分支、pull 后技能文件变化自动感知
- **动态技能发现**：文件操作触发嵌套技能目录的发现与加载
- **多会话协作**：其他会话修改技能目录时当前会话自动更新

---

## 2. 功能点目的

### 2.1 文件稳定性检测（File Stability）
```typescript
const FILE_STABILITY_THRESHOLD_MS = 1000
const FILE_STABILITY_POLL_INTERVAL_MS = 500
```
**目的**：等待文件写入稳定后再处理，避免读取到不完整内容。

### 2.2 重载防抖（Reload Debounce）
```typescript
const RELOAD_DEBOUNCE_MS = 300
```
**目的**：将短时间内的多次变更（如 git checkout 批量修改）聚合为单次重载，防止事件循环死锁。

### 2.3 Bun 死锁规避（Polling Mode）
```typescript
const USE_POLLING = typeof Bun !== 'undefined'
const POLLING_INTERVAL_MS = 2000
```
**目的**：规避 Bun 的 `fs.watch()` PathWatcherManager 死锁问题（oven-sh/bun#27469），使用 stat 轮询代替原生监听。

### 2.4 钩子拦截机制
```typescript
const results = await executeConfigChangeHooks('skills', paths[0]!)
if (hasBlockingResult(results)) return
```
**目的**：允许用户通过 `ConfigChange` 钩子拦截或响应技能变更事件。

---

## 3. 具体技术实现

### 3.1 关键数据结构

```typescript
// 模块级状态
let watcher: FSWatcher | null = null           // chokidar 实例
let reloadTimer: ReturnType<typeof setTimeout> | null = null  // 防抖定时器
const pendingChangedPaths = new Set<string>()  // 待处理的变更路径
let initialized = false                        // 初始化标志
let disposed = false                           // 销毁标志

// Signal 事件总线（来自 ../signal.js）
const skillsChanged = createSignal()
```

### 3.2 关键流程

#### 3.2.1 初始化流程 (`initialize()`)
```
1. 检查 initialized/disposed 标志，防止重复初始化
2. 注册动态技能加载回调（onDynamicSkillsLoaded）
   - 回调内清理命令缓存（clearCommandMemoizationCaches）
   - 触发 skillsChanged.emit() 通知订阅者
3. 获取可监视路径（getWatchablePaths）
   - 用户技能目录：~/.claude/skills
   - 用户命令目录：~/.claude/commands
   - 项目技能目录：./.claude/skills
   - 项目命令目录：./.claude/commands
   - 额外目录（--add-dir）：<dir>/.claude/skills
4. 创建 chokidar watcher
   - depth: 2（技能使用 skill-name/SKILL.md 格式）
   - awaitWriteFinish: 等待写入稳定
   - usePolling: Bun 环境下启用轮询
   - ignored: 忽略 .git 目录和特殊文件类型
5. 注册事件处理器：add/change/unlink -> handleChange
6. 注册清理函数到 cleanupRegistry
```

#### 3.2.2 变更处理流程 (`handleChange -> scheduleReload`)
```
handleChange(path)
  ├── 记录调试日志
  ├── 发送分析事件（tengu_skill_file_changed）
  └── 调用 scheduleReload(path)

scheduleReload(changedPath)
  ├── 将路径加入 pendingChangedPaths
  ├── 清除现有定时器（防抖）
  └── 设置新定时器（RELOAD_DEBOUNCE_MS 后执行）
      └── 定时器回调：
          ├── 收集所有待处理路径
          ├── 执行 ConfigChange 钩子
          │   └── 如果被拦截（blocking），直接返回
          ├── 清理技能缓存（clearSkillCaches）
          ├── 清理命令缓存（clearCommandsCache）
          ├── 重置已发送技能名（resetSentSkillNames）
          └── 触发 skillsChanged.emit()
```

#### 3.2.3 销毁流程 (`dispose()`)
```
1. 设置 disposed = true
2. 从 cleanupRegistry 注销
3. 关闭 watcher（watcher.close()）
4. 清除定时器
5. 清空待处理路径集合
6. 清空 Signal 监听器
```

### 3.3 协议与命令

#### Signal 协议（来自 ../signal.ts）
```typescript
type Signal<Args extends unknown[] = []> = {
  subscribe: (listener: (...args: Args) => void) => () => void
  emit: (...args: Args) => void
  clear: () => void
}
```
- **subscribe**: 返回取消订阅函数
- **emit**: 同步调用所有监听器
- **clear**: 清空所有监听器（用于 dispose）

#### 缓存清理协议
| 函数 | 来源 | 作用 |
|------|------|------|
| `clearSkillCaches()` | `loadSkillsDir.ts` | 清理技能目录命令缓存、条件技能状态 |
| `clearCommandsCache()` | `commands.ts` | 清理命令缓存、插件缓存、技能索引缓存 |
| `resetSentSkillNames()` | `attachments.ts` | 重置已发送技能名集合，允许重新发送 skill_listing |

---

## 4. 关键代码路径与文件引用

### 4.1 模块依赖图

```
skillChangeDetector.ts
├── chokidar (外部库)
├── platformPath (node:path)
├── ../../bootstrap/state.ts
│   └── getAdditionalDirectoriesForClaudeMd()
├── ../../commands.ts
│   ├── clearCommandMemoizationCaches()
│   └── clearCommandsCache()
├── ../../services/analytics/index.ts
│   └── logEvent()
├── ../../skills/loadSkillsDir.ts
│   ├── clearSkillCaches()
│   ├── getSkillsPath()
│   └── onDynamicSkillsLoaded()
├── ../attachments.ts
│   └── resetSentSkillNames()
├── ../cleanupRegistry.ts
│   └── registerCleanup()
├── ../debug.ts
│   └── logForDebugging()
├── ../fsOperations.ts
│   └── getFsImplementation()
├── ../hooks.ts
│   ├── executeConfigChangeHooks()
│   └── hasBlockingResult()
└── ../signal.ts
    └── createSignal()
```

### 4.2 调用方（Consumers）

| 文件 | 调用方式 | 用途 |
|------|----------|------|
| `src/main.tsx:424` | `skillChangeDetector.initialize()` | 应用启动时初始化（非 bare 模式） |
| `src/hooks/useSkillsChange.ts:43` | `skillChangeDetector.subscribe()` | React 钩子订阅变更，刷新命令列表 |
| `src/cli/print.ts:1824` | `skillChangeDetector.subscribe()` | CLI 模式订阅变更，热重载命令 |
| `src/services/compact/postCompactCleanup.ts:68` | 注释引用 | 说明 compaction 不重置技能名 |

### 4.3 关键代码片段

#### 4.3.1 防抖重载实现
```typescript
function scheduleReload(changedPath: string): void {
  pendingChangedPaths.add(changedPath)
  if (reloadTimer) clearTimeout(reloadTimer)
  reloadTimer = setTimeout(async () => {
    reloadTimer = null
    const paths = [...pendingChangedPaths]
    pendingChangedPaths.clear()
    
    // 执行钩子检查
    const results = await executeConfigChangeHooks('skills', paths[0]!)
    if (hasBlockingResult(results)) {
      logForDebugging(`ConfigChange hook blocked skill reload (${paths.length} paths)`)
      return
    }
    
    // 清理缓存并通知
    clearSkillCaches()
    clearCommandsCache()
    resetSentSkillNames()
    skillsChanged.emit()
  }, testOverrides?.reloadDebounce ?? RELOAD_DEBOUNCE_MS)
}
```

#### 4.3.2 可监视路径收集
```typescript
async function getWatchablePaths(): Promise<string[]> {
  const fs = getFsImplementation()
  const paths: string[] = []

  // 用户级目录
  const userSkillsPath = getSkillsPath('userSettings', 'skills')
  const userCommandsPath = getSkillsPath('userSettings', 'commands')
  
  // 项目级目录
  const projectSkillsPath = getSkillsPath('projectSettings', 'skills')
  const projectCommandsPath = getSkillsPath('projectSettings', 'commands')
  
  // 额外目录（--add-dir）
  for (const dir of getAdditionalDirectoriesForClaudeMd()) {
    const additionalSkillsPath = platformPath.join(dir, '.claude', 'skills')
    // ...stat 检查
  }
  
  return paths
}
```

#### 4.3.3 Bun 死锁规避注释
```typescript
/**
 * Bun's native fs.watch() has a PathWatcherManager deadlock (oven-sh/bun#27469,
 * #26385): closing a watcher on the main thread while the File Watcher thread
 * is delivering events can hang both threads in __ulock_wait2 forever. Chokidar
 * with depth: 2 on large skill trees (hundreds of subdirs) triggers this
 * reliably when a git operation touches many directories at once — chokidar
 * internally closes/reopens per-directory FSWatchers as dirs are added/removed.
 *
 * Workaround: use stat() polling under Bun. No FSWatcher = no deadlock.
 * The fix is pending upstream; remove this once the Bun PR lands.
 */
const USE_POLLING = typeof Bun !== 'undefined'
```

---

## 5. 依赖与外部交互

### 5.1 外部库依赖

| 库 | 用途 | 版本约束 |
|----|------|----------|
| `chokidar` | 跨平台文件监视 | ^3.x |

### 5.2 内部模块交互

#### 5.2.1 与 loadSkillsDir.ts 的交互
- **导入**: `clearSkillCaches`, `getSkillsPath`, `onDynamicSkillsLoaded`
- **机制**: 
  - `onDynamicSkillsLoaded` 注册回调，在动态技能加载时触发
  - `clearSkillCaches` 清理技能目录命令的 memoize 缓存
  - `getSkillsPath` 获取不同来源（userSettings/projectSettings）的技能路径

#### 5.2.2 与 commands.ts 的交互
- **导入**: `clearCommandMemoizationCaches`, `clearCommandsCache`
- **机制**:
  - `clearCommandMemoizationCaches` 仅清理 lodash memoize 缓存（用于动态技能添加）
  - `clearCommandsCache` 全面清理包括插件缓存、技能索引缓存

#### 5.2.3 与 attachments.ts 的交互
- **导入**: `resetSentSkillNames`
- **机制**: 重置已发送技能名集合，允许在变更后重新发送 skill_listing 附件

#### 5.2.4 与 hooks.ts 的交互
- **导入**: `executeConfigChangeHooks`, `hasBlockingResult`
- **机制**: 在重载前执行用户定义的 ConfigChange 钩子，支持拦截或自定义处理

### 5.3 配置与环境变量

| 变量/配置 | 影响 |
|-----------|------|
| `Bun` 全局变量 | 决定是否启用轮询模式（USE_POLLING） |
| `--bare` 模式 | 跳过技能变更检测器初始化 |
| `--add-dir` 参数 | 通过 `getAdditionalDirectoriesForClaudeMd()` 影响监视路径 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 Bun 死锁风险（已缓解）
- **风险**: Bun 的 `fs.watch()` 在大量目录变更时可能死锁
- **缓解**: 使用 `usePolling: true` 在 Bun 环境下启用 stat 轮询
- **代价**: 2 秒轮询间隔增加技能变更感知延迟

#### 6.1.2 事件循环阻塞风险
- **风险**: 大量技能文件同时变更时，级联的缓存清理可能阻塞事件循环
- **缓解**: 300ms 防抖聚合多次变更
- **边界**: 如果单次清理耗时过长，仍可能阻塞

#### 6.1.3 钩子执行风险
- **风险**: `executeConfigChangeHooks` 是异步的，可能抛出异常
- **现状**: 未在 `scheduleReload` 中捕获钩子异常
- **潜在问题**: 钩子异常可能导致重载流程中断

### 6.2 边界条件

| 边界场景 | 行为 |
|----------|------|
| 技能目录不存在 | `getWatchablePaths` 中捕获 stat 错误，跳过该路径 |
| 初始化后重复调用 `initialize()` | 直接返回，无操作 |
| `dispose()` 后调用 `initialize()` | 直接返回，需先调用 `resetForTesting()` |
| 文件写入过程中触发变更 | `awaitWriteFinish` 等待 1000ms 稳定期 |
| 批量 Git 操作（数百文件） | 防抖聚合为单次重载，避免 30+ 次全量刷新 |

### 6.3 改进建议

#### 6.3.1 异常处理增强
```typescript
// 建议：在 scheduleReload 中添加异常捕获
try {
  const results = await executeConfigChangeHooks('skills', paths[0]!)
} catch (error) {
  logForDebugging(`ConfigChange hook failed: ${error}`)
  // 继续执行重载，或根据策略中断
}
```

#### 6.3.2 性能优化：增量刷新
- **现状**: 任何变更都触发全量缓存清理和重新扫描
- **建议**: 支持基于路径的增量刷新，仅重新加载变更的技能

#### 6.3.3 可观测性增强
- **现状**: 仅记录调试日志和分析事件
- **建议**: 
  - 添加重载耗时指标（从变更到完成的时间）
  - 添加待处理路径队列大小指标
  - 添加钩子执行耗时指标

#### 6.3.4 Bun 死锁长期修复
- **现状**: 依赖轮询规避
- **建议**: 跟踪 Bun 上游修复（oven-sh/bun#27469），修复后移除轮询回退

#### 6.3.5 测试覆盖增强
- **现状**: 无直接测试文件（搜索未发现 `*.test.ts`）
- **建议**: 
  - 添加单元测试覆盖防抖逻辑
  - 添加集成测试验证完整变更检测流程
  - 使用 `resetForTesting` 提供干净的测试隔离

### 6.4 相关 Issue/PR 参考

| 引用 | 说明 |
|------|------|
| oven-sh/bun#27469 | Bun fs.watch() PathWatcherManager 死锁问题 |
| oven-sh/bun#26385 | 相关死锁报告 |
| 代码注释中的 `#22257` | 调试日志写入性能优化 |
| 代码注释中的 `gh-32730` | 会话创建团队清理 |

---

## 7. 总结

`skillChangeDetector.ts` 是 Claude Code 技能系统的核心基础设施，负责将文件系统变更转化为应用状态更新。其设计体现了以下工程权衡：

1. **可靠性优先**：通过防抖、稳定性检测、轮询回退等机制确保变更检测的可靠性
2. **性能折中**：300ms 防抖和 2s 轮询间隔换取系统稳定性
3. **扩展性**：通过 Signal 机制和钩子系统支持灵活的订阅和拦截
4. **测试友好**：提供 `resetForTesting` 和 `testOverrides` 支持测试隔离

理解该模块对于维护技能系统、诊断热重载问题、以及扩展技能相关功能至关重要。
