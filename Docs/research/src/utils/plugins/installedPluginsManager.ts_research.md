# installedPluginsManager.ts 深度研究文档

## 场景与职责

`installedPluginsManager.ts` 是 Claude Code 插件系统的核心状态管理模块，负责管理 `installed_plugins.json` 中的插件安装元数据。该模块将**安装状态**（全局）与**启用/禁用状态**（每仓库）分离，支持多作用域（user/project/local/managed）的插件安装。

核心场景：
1. **插件安装/更新/卸载**：记录安装元数据到磁盘
2. **会话启动**：加载安装状态，检测待更新插件
3. **V1→V2 迁移**：平滑升级旧格式数据
4. **背景更新**：磁盘状态与会话状态分离，支持后台更新

## 功能点目的

### 1. 安装元数据管理
- **存储位置**：`~/.claude/plugins/installed_plugins.json`
- **V2 格式**：支持多作用域，每插件可有多条安装记录
  ```typescript
  {
    version: 2,
    plugins: {
      "plugin@marketplace": [
        {
          scope: 'user' | 'project' | 'local' | 'managed',
          installPath: string,
          version: string,
          installedAt: string,
          lastUpdated: string,
          gitCommitSha?: string,
          projectPath?: string // project/local 作用域
        }
      ]
    }
  }
  ```

### 2. 双状态管理（内存 vs 磁盘）
- **`inMemoryInstalledPlugins`**：会话启动时的快照，整个会话使用
- **磁盘状态**：背景更新器直接修改，不影响运行会话
- **待更新检测**：比较内存与磁盘状态，提示用户重启

### 3. V1→V2 迁移
- **单文件合并**：`installed_plugins_v2.json` → `installed_plugins.json`
- **格式转换**：V1 扁平结构 → V2 数组结构
- **作用域默认**：V1 无作用域概念，默认迁移为 `user` 作用域
- **遗留缓存清理**：删除非版本化缓存目录

### 4. 作用域感知查询
- **`isPluginInstalled`**：检查当前项目上下文相关安装
- **`isPluginGloballyInstalled`**：检查 user/managed 全局安装
- **项目相关性判断**：`isInstallationRelevantToCurrentProject`

## 具体技术实现

### 缓存与状态变量
```typescript
// 备忘录化缓存（文件修改时清除）
let installedPluginsCacheV2: InstalledPluginsFileV2 | null = null

// 会话级快照（背景更新不修改）
let inMemoryInstalledPlugins: InstalledPluginsFileV2 | null = null

// 迁移状态（防止重复执行）
let migrationCompleted = false
```

### V1→V2 迁移算法
```
migrateToSinglePluginFile()
  ├─ 若 migrationCompleted 已设置 → 返回
  ├─ 尝试直接重命名 v2→main
  │   ├─ 成功 → 清理遗留缓存，标记完成
  │   └─ ENOENT → v2 文件不存在，继续
  ├─ 尝试读取 main 文件
  │   ├─ ENOENT → 无文件，标记完成
  │   └─ 读取成功 → 解析版本号
  ├─ 若 version === 1
  │   ├─ 解析为 V1 格式
  │   ├─ 转换为 V2 格式（默认 user 作用域）
  │   ├─ 计算版本化缓存路径
  │   ├─ 原子写入 V2 格式
  │   └─ 清理遗留缓存
  └─ 标记 migrationCompleted = true
```

### 双状态比较（待更新检测）
```typescript
export function hasPendingUpdates(): boolean {
  const memoryState = getInMemoryInstalledPlugins()
  const diskState = loadInstalledPluginsFromDisk()
  
  for (const [pluginId, diskInstallations] of Object.entries(diskState.plugins)) {
    const memoryInstallations = memoryState.plugins[pluginId]
    if (!memoryInstallations) continue
    
    for (const diskEntry of diskInstallations) {
      const memoryEntry = memoryInstallations.find(
        m => m.scope === diskEntry.scope && m.projectPath === diskEntry.projectPath
      )
      if (memoryEntry && memoryEntry.installPath !== diskEntry.installPath) {
        return true // 磁盘有新版本
      }
    }
  }
  return false
}
```

### 磁盘-only 更新（背景更新器使用）
```typescript
export function updateInstallationPathOnDisk(
  pluginId: string,
  scope: PersistableScope,
  projectPath: string | undefined,
  newPath: string,
  newVersion: string,
  gitCommitSha?: string
): void {
  const diskData = loadInstalledPluginsFromDisk() // 直接读磁盘
  // ... 修改 diskData ...
  writeFileSync_DEPRECATED(filePath, jsonStringify(diskData, null, 2))
  installedPluginsCacheV2 = null // 清除缓存
  // 注意：inMemoryInstalledPlugins 不更新！
}
```

### 从 settings 迁移（初始化时）
```
migrateFromEnabledPlugins()
  ├─ 获取 merged settings 中的 enabledPlugins
  ├─ 若无插件 → 返回
  ├─ 检查文件存在性和格式
  │   └─ 若 V2 且所有插件存在 → 跳过（优化路径）
  ├─ 从各 settings 源构建 pluginId → scope 映射
  │   └─ 优先级：local > project > user
  ├─ 加载现有数据或创建空对象
  ├─ 对每个 pluginId
  │   ├─ 若存在且 scope 匹配 → 更新 scope
  │   └─ 若不存在 → 查询市场，创建新条目
  │       ├─ 获取 installPath, version, gitCommitSha
  │       └─ 添加到 v2Plugins
  └─ 若有变更 → 保存到磁盘
```

## 关键代码路径与文件引用

### 内部依赖
```
installedPluginsManager.ts
  ├─ path: dirname, join
  ├─ debug.ts: logForDebugging
  ├─ errors.ts: errorMessage, isENOENT, toError
  ├─ fsOperations.ts: getFsImplementation
  ├─ log.ts: logError
  ├─ slowOperations.ts: jsonParse, jsonStringify, writeFileSync_DEPRECATED
  ├─ pluginDirectories.ts: getPluginsDirectory
  ├─ schemas.ts: InstalledPluginsFileSchemaV1/V2, PluginInstallationEntry, etc.
  ├─ bootstrap/state.ts: getOriginalCwd
  ├─ cwd.ts: getCwd
  ├─ git/gitFilesystem.ts: getHeadForDir → getGitCommitSha
  ├─ settings/constants.ts: EditableSettingSource
  ├─ settings/settings.ts: getSettings_DEPRECATED, getSettingsForSource
  ├─ marketplaceManager.ts: getPluginById
  ├─ pluginIdentifier.ts: parsePluginIdentifier, settingSourceToScope
  └─ pluginLoader.ts: getPluginCachePath, getVersionedCachePath
```

### 外部调用方
| 调用方 | 调用函数 | 用途 |
|--------|----------|------|
| `cacheUtils.ts` | `loadInstalledPluginsFromDisk` | 孤儿版本清理 |
| `pluginInstallationHelpers.ts` | `addInstalledPlugin`, `getGitCommitSha` | 安装流程 |
| `pluginOperations.ts` | `removeInstalledPlugin`, `deletePluginCache` | 卸载流程 |
| `marketplaceManager.ts` | `removeAllPluginsForMarketplace` | 市场移除 |
| `init.ts` | `initializeVersionedPlugins` | 启动初始化 |
| `backgroundUpdater.ts` | `updateInstallationPathOnDisk`, `hasPendingUpdates` | 背景更新 |
| `UI 组件` | `isPluginInstalled`, `isPluginGloballyInstalled` | 安装状态查询 |

### 调用链：初始化流程
```
initializeVersionedPlugins()
  ├─ migrateToSinglePluginFile() // V1/V2 迁移
  ├─ migrateFromEnabledPlugins() // settings → installed_plugins 同步
  └─ getInMemoryInstalledPlugins() // 加载会话快照
```

## 依赖与外部交互

### 文件系统交互
- **主文件**：`~/.claude/plugins/installed_plugins.json`
- **遗留 V2 文件**：`~/.claude/plugins/installed_plugins_v2.json`（迁移后删除）
- **写入模式**：`writeFileSync_DEPRECATED`（同步 + flush）

### 与其他状态的关系
```
┌─────────────────────────────────────────────────────────────┐
│                     插件状态全景                              │
├─────────────────────────────────────────────────────────────┤
│ installed_plugins.json                                      │
│   └─ 安装元数据（全局，多作用域）                              │
│        ├─ installPath                                       │
│        ├─ version                                           │
│        └─ scope (user/project/local/managed)                │
├─────────────────────────────────────────────────────────────┤
│ .claude/settings.json (各层级)                              │
│   └─ enabledPlugins                                         │
│        ├─ true → 启用                                        │
│        ├─ false → 显式禁用                                   │
│        └─ string[] → 版本约束                                │
├─────────────────────────────────────────────────────────────┤
│ 内存状态 (inMemoryInstalledPlugins)                         │
│   └─ 会话启动时的磁盘快照                                    │
│        └─ 背景更新不修改此状态                               │
└─────────────────────────────────────────────────────────────┘
```

## 风险、边界与改进建议

### 已知风险

1. **双状态不一致**
   - 风险：用户看到"有可用更新"但重启后状态不同
   - 现状：设计如此，背景更新原子性提交到磁盘
   - 缓解：清晰的用户提示

2. **同步文件写入阻塞**
   - 风险：`writeFileSync_DEPRECATED` 在慢磁盘上可能阻塞
   - 现状：安装/卸载操作本身较重，可接受
   - 建议：考虑异步写入 + 崩溃恢复

3. **V1 迁移遗留问题**
   - 风险：极端并发场景下迁移可能重复执行
   - 现状：`migrationCompleted` 标志防止同会话重复
   - 局限：跨进程无锁保护

4. **作用域冲突**
   - 风险：同一插件多作用域安装时逻辑复杂
   - 现状：数组结构支持，但 UI 展示可能混淆

### 边界条件

| 场景 | 行为 |
|------|------|
| 文件不存在 | 返回空 V2 结构 |
| 文件损坏 | 记录错误，返回空结构 |
| V1 格式 | 内存中转换，下次写入 V2 |
| 同一插件多作用域 | 数组中多条记录 |
| 背景更新时进程崩溃 | 依赖文件系统原子写入保证一致性 |
| projectPath 不匹配当前 | `isPluginInstalled` 返回 false |

### 改进建议

1. **异步写入优化**
   - 当前：同步写入 + flush
   - 建议：批量异步写入，退出时强制 flush

2. **迁移锁机制**
   - 当前：进程内标志
   - 建议：文件锁或原子 rename 检测

3. **安装历史记录**
   - 当前：仅最新状态
   - 建议：保留安装/更新/卸载历史，便于审计

4. **校验和验证**
   - 当前：JSON 解析验证
   - 建议：添加文件校验和，检测静默损坏

5. **作用域继承可视化**
   - 当前：用户难以理解作用域优先级
   - 建议：UI 展示 effective scope 计算过程
