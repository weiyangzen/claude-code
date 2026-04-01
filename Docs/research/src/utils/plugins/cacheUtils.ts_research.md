# cacheUtils.ts 深度研究文档

## 场景与职责

`cacheUtils.ts` 是 Claude Code 插件系统的缓存管理核心模块，负责插件全生命周期的缓存控制与孤儿版本清理。该模块在以下场景发挥关键作用：

1. **插件安装/更新/卸载时**：清理相关缓存确保状态一致性
2. **会话启动时**：执行孤儿版本后台清理（7天延迟删除策略）
3. **用户执行 `/reload-plugins` 时**：批量刷新所有插件相关缓存
4. **ZIP缓存模式**：跳过目录遍历清理（避免误删ZIP文件）

## 功能点目的

### 1. 全量缓存清理 (`clearAllPluginCaches` / `clearAllCaches`)
- **目的**：在插件状态变更时确保所有层级缓存同步失效
- **清理范围**：
  - 插件主缓存 (`clearPluginCache`)
  - 命令缓存 (`clearPluginCommandCache`)
  - Agent定义缓存 (`clearPluginAgentCache`)
  - Hook缓存 (`clearPluginHookCache`) + 孤儿Hook剪枝 (`pruneRemovedPluginHooks`)
  - 选项缓存 (`clearPluginOptionsCache`)
  - 输出样式缓存 (`clearPluginOutputStyleCache`)
  - 全局输出样式缓存 (`clearAllOutputStylesCache`)
  - 核心命令缓存 (`clearCommandsCache`)
  - Agent定义缓存 (`clearAgentDefinitionsCache`)
  - Prompt缓存 (`clearPromptCache`)
  - 已发送技能名记录 (`resetSentSkillNames`)

### 2. 孤儿版本标记与清理
- **标记机制**：插件卸载/更新时写入 `.orphaned_at` 文件记录时间戳
- **延迟清理**：7天宽限期 (`CLEANUP_AGE_MS = 7 * 24 * 60 * 60 * 1000`)
- **双阶段清理**：
  - Pass 1：清除已重新安装版本的过期标记
  - Pass 2：处理真正的孤儿版本（创建标记或执行删除）

### 3. ZIP缓存模式适配
- 检测到 `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` 时跳过目录清理
- 原因：ZIP模式下插件以 `.zip` 文件存储，`readSubdirs` 会误判目录为空

## 具体技术实现

### 关键数据结构
```typescript
const ORPHANED_AT_FILENAME = '.orphaned_at'
const CLEANUP_AGE_MS = 7 * 24 * 60 * 60 * 1000 // 7天
```

### 核心算法流程

**孤儿版本清理算法**：
```
cleanupOrphanedPluginVersionsInBackground()
  ├─ 检查ZIP缓存模式（是则直接返回）
  ├─ 获取已安装版本路径集合 (getInstalledVersionPaths)
  │   └─ 遍历 installed_plugins.json 所有安装项
  ├─ Pass 1: 清理已安装版本的.orphaned_at标记
  │   └─ Promise.all 并行删除标记文件
  └─ Pass 2: 遍历缓存目录三级结构
      ├─ marketplace/
      │   └─ plugin/
      │       └─ version/ ← 处理层级
      └─ 对每个非安装版本调用 processOrphanedPluginVersion
          ├─ 无标记 → 创建标记（兼容旧版本）
          ├─ 有标记且超7天 → rm -rf 删除
          └─ 有标记未超期 → 保留
```

**路径遍历安全**：
- 使用 `readSubdirs` 过滤仅目录项
- 逐级清理空目录 (`removeIfEmpty`)
- 错误隔离：每个版本独立try-catch，单点失败不影响整体

## 关键代码路径与文件引用

### 内部调用图
```
cacheUtils.ts
  ├─ clearAllPluginCaches()
  │   ├─ pluginLoader.ts: clearPluginCache()
  │   ├─ loadPluginCommands.ts: clearPluginCommandCache()
  │   ├─ loadPluginAgents.ts: clearPluginAgentCache()
  │   ├─ loadPluginHooks.ts: clearPluginHookCache(), pruneRemovedPluginHooks()
  │   ├─ pluginOptionsStorage.ts: clearPluginOptionsCache()
  │   ├─ loadPluginOutputStyles.ts: clearPluginOutputStyleCache()
  │   ├─ constants/outputStyles.ts: clearAllOutputStylesCache()
  │   └─ commands.ts: clearCommandsCache()
  ├─ clearAllCaches()
  │   ├─ clearAllPluginCaches()
  │   ├─ commands.ts: clearCommandsCache()
  │   ├─ tools/AgentTool/loadAgentsDir.ts: clearAgentDefinitionsCache()
  │   ├─ tools/SkillTool/prompt.ts: clearPromptCache()
  │   └─ attachments.ts: resetSentSkillNames()
  └─ cleanupOrphanedPluginVersionsInBackground()
      ├─ zipCache.ts: isPluginZipCacheEnabled()
      ├─ installedPluginsManager.ts: loadInstalledPluginsFromDisk()
      └─ pluginLoader.ts: getPluginCachePath()
```

### 外部调用方
- `marketplaceManager.ts`：插件安装/更新后调用 `markPluginVersionOrphaned`
- `pluginInstallationHelpers.ts`：安装流程中调用缓存清理
- `init.ts` / `headless.ts`：启动时调用 `cleanupOrphanedPluginVersionsInBackground`

## 依赖与外部交互

### 依赖模块
| 模块 | 用途 |
|------|------|
| `fs/promises` | 文件系统操作（readdir, rm, stat, unlink, writeFile） |
| `path.join` | 路径拼接 |
| `commands.ts` | 核心命令缓存清理 |
| `constants/outputStyles.ts` | 输出样式缓存 |
| `tools/AgentTool/loadAgentsDir.ts` | Agent定义缓存 |
| `tools/SkillTool/prompt.ts` | Prompt缓存 |
| `attachments.ts` | 技能名发送记录 |
| `debug.ts` | 调试日志 |
| `errors.ts` | 错误码提取 (`getErrnoCode`) |
| `log.ts` | 错误日志 |
| `installedPluginsManager.ts` | 已安装插件数据读取 |
| `loadPluginAgents.ts` | Agent缓存清理 |
| `loadPluginCommands.ts` | 命令缓存清理 |
| `loadPluginHooks.ts` | Hook缓存清理与剪枝 |
| `loadPluginOutputStyles.ts` | 输出样式缓存清理 |
| `pluginLoader.ts` | 插件主缓存清理与缓存路径获取 |
| `pluginOptionsStorage.ts` | 选项缓存清理 |
| `zipCache.ts` | ZIP缓存模式检测 |

## 风险、边界与改进建议

### 已知风险

1. **ZIP缓存模式下的目录误判**
   - 风险：若ZIP检测与实际操作模式不一致，可能导致数据丢失
   - 缓解：通过 `isPluginZipCacheEnabled()` 统一判断，与存储层保持一致

2. **孤儿清理的竞态条件**
   - 风险：用户同时执行卸载和重新安装，可能导致新安装被误判为孤儿
   - 缓解：Pass 1 优先清理已安装版本的标记，确保重新安装的版本不会被删除

3. **macOS xcrun shim 问题**
   - 风险：git存在但无法执行时，清理操作可能失败
   - 相关：`gitAvailability.ts` 提供 `markGitUnavailable()` 机制

4. **大缓存目录遍历性能**
   - 风险：缓存目录极深时，递归遍历可能成为启动瓶颈
   - 现状：后台异步执行，不阻塞主流程

### 边界条件

| 场景 | 行为 |
|------|------|
| ZIP缓存模式 | 完全跳过孤儿清理 |
| 无已安装插件数据 | 返回空Set，所有版本视为孤儿 |
| .orphaned_at 读取失败 | 记录调试日志，跳过该版本 |
| 目录删除失败 | 记录调试日志，继续处理其他版本 |
| 7天边界精确值 | 使用 `mtimeMs` 比较，`>` 严格大于 |

### 改进建议

1. **增量清理优化**
   - 当前：每次启动遍历整个缓存目录树
   - 建议：维护一个待清理队列，避免全量扫描

2. **可配置清理策略**
   - 当前：硬编码7天宽限期
   - 建议：支持环境变量覆盖（如 `CLAUDE_CODE_PLUGIN_ORPHAN_TTL_DAYS`）

3. **清理进度可观测性**
   - 当前：仅调试日志
   - 建议：添加 telemetry 事件，监控清理量和耗时

4. **孤儿版本空间统计**
   - 当前：仅删除，无回收空间统计
   - 建议：计算并上报回收的磁盘空间
