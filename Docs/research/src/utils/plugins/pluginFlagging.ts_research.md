# pluginFlagging.ts 研究文档

## 场景与职责

`src/utils/plugins/pluginFlagging.ts` 负责**被自动移除插件的标记与生命周期管理**。当 `pluginBlocklist.ts` 检测到某个插件已从 marketplace 下架并自动卸载后，该模块：

1. **记录被标记的插件**：将插件 ID、标记时间写入 `~/.claude/plugins/flagged-plugins.json`。
2. **提供同步查询能力**：通过模块级内存缓存，使 React 渲染路径（如 `ManagePlugins.tsx`）能够以同步方式获取 flagged 列表，无需阻塞渲染等待磁盘 IO。
3. **支持“已查看”状态与自动过期**：用户查看 `/plugins` 中的 Flagged 区域后，记录 `seenAt`；48 小时后自动从列表中清除。
4. **支持用户手动移除标记**：用户在 UI 中 dismiss 某个 flagged 插件时，将其从 JSON 中删除。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `loadFlaggedPlugins()` | 从磁盘读取 `flagged-plugins.json`，解析并过滤掉已过期（seenAt 超过 48h）的条目，写回磁盘。是预热缓存的异步入口。 |
| `getFlaggedPlugins()` | 同步返回内存缓存中的 flagged 插件映射。若缓存未预热则返回 `{}`。 |
| `addFlaggedPlugin(pluginId)` | 将指定插件加入 flagged 列表，记录 `flaggedAt` 为当前 ISO 时间。 |
| `markFlaggedPluginsSeen(pluginIds)` | 为指定 flagged 插件设置 `seenAt`。已设置过的不再覆盖。 |
| `removeFlaggedPlugin(pluginId)` | 用户手动 dismiss 时，从缓存和磁盘中移除指定插件。 |

---

## 具体技术实现

### 关键流程

#### 1. 加载与过期清理流程

```
loadFlaggedPlugins()
  ├─> readFromDisk()
  │     ├─> readFile(flaggedPluginsPath, 'utf-8')
  │     └─> parsePluginsData(content)
  │           ├─> 验证顶层结构：必须包含 object 类型的 `plugins` 字段
  │           └─> 遍历每个条目，验证 `flaggedAt` 为 string，可选 `seenAt` 为 string
  ├─> 遍历解析结果
  │     └─> 若 entry.seenAt 存在且 now - seenAt >= 48h
  │           └─> delete all[id]; changed = true
  ├─> cache = all
  └─> if changed → writeToDisk(all)
```

#### 2. 写盘安全流程

```
writeToDisk(plugins)
  ├─> filePath = getFlaggedPluginsPath()
  ├─> tempPath = `${filePath}.${randomBytes(8).toString('hex')}.tmp`
  ├─> getFsImplementation().mkdir(getPluginsDirectory())   // 确保父目录存在
  ├─> writeFile(tempPath, jsonStringify({plugins}, null, 2), { mode: 0o600 })
  ├─> rename(tempPath, filePath)                           // 原子替换
  ├─> cache = plugins
  └─> catch error → logError + 尝试 unlink(tempPath)
```

### 数据结构

```typescript
export type FlaggedPlugin = {
  flaggedAt: string   // ISO 8601 时间戳
  seenAt?: string     // 用户首次在 UI 中看到的时间戳
}
```

- 磁盘文件格式：
  ```json
  {
    "plugins": {
      "plugin-name@marketplace": {
        "flaggedAt": "2026-04-01T10:00:00.000Z",
        "seenAt": "2026-04-01T11:00:00.000Z"
      }
    }
  }
  ```

- 模块级缓存：
  ```typescript
  let cache: Record<string, FlaggedPlugin> | null = null
  ```

### 过期策略

- `SEEN_EXPIRY_MS = 48 * 60 * 60 * 1000`（48 小时）。
- 仅在 `loadFlaggedPlugins()` 被调用时执行过期清理。若用户长期不打开 `/plugins`（即不触发 `loadFlaggedPlugins`）， flagged 条目可能长期保留在磁盘上。

---

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 用途 |
|------|------|
| `loadFlaggedPlugins(): Promise<void>` | 异步加载并清理过期条目 |
| `getFlaggedPlugins(): Record<string, FlaggedPlugin>` | 同步读取内存缓存 |
| `addFlaggedPlugin(pluginId): Promise<void>` | 新增标记 |
| `markFlaggedPluginsSeen(pluginIds): Promise<void>` | 标记为已查看 |
| `removeFlaggedPlugin(pluginId): Promise<void>` | 用户手动移除标记 |
| `FlaggedPlugin` | 类型定义 |

### 直接依赖文件

- `src/utils/plugins/pluginDirectories.ts`：`getPluginsDirectory`
- `src/utils/fsOperations.ts`：`getFsImplementation`
- `src/utils/slowOperations.ts`：`jsonParse`, `jsonStringify`
- `src/utils/debug.ts`：`logForDebugging`
- `src/utils/log.ts`：`logError`

### 调用方文件

- `src/utils/plugins/pluginBlocklist.ts`：在 `detectAndUninstallDelistedPlugins()` 中调用 `loadFlaggedPlugins()`（预热）、`getFlaggedPlugins()`（查重）、`addFlaggedPlugin()`（记录新下架插件）。
- `src/hooks/useManagePlugins.ts`：调用 `getFlaggedPlugins()` 检查是否有 flagged 插件，若有则弹出通知“Plugins flagged. Check /plugins”。
- `src/commands/plugin/ManagePlugins.tsx`：
  - 调用 `getFlaggedPlugins()` 获取列表用于渲染 Flagged 区域。
  - 调用 `markFlaggedPluginsSeen(flaggedIds)` 当 Flagged 区域被渲染时。
  - 调用 `removeFlaggedPlugin(pluginId)` 当用户选择 dismiss 某个 flagged 插件时。

---

## 依赖与外部交互

### 外部文件

- **`~/.claude/plugins/flagged-plugins.json`**：持久化存储 flagged 插件数据。
  - 写入时使用 `0o600` 权限，限制其他用户读取。
  - 使用“写临时文件 + rename”的原子写入策略，避免并发或崩溃导致文件损坏。

### 与 UI 的交互

- `ManagePlugins.tsx` 将 flagged 插件作为独立的 `type: 'flagged-plugin'` 项插入到统一安装列表的 `'flagged'` scope 分组中。
- 当用户浏览到这些项时，React effect 自动调用 `markFlaggedPluginsSeen`，启动 48 小时倒计时。

---

## 风险、边界与改进建议

### 风险与边界

1. **缓存未预热时返回空对象**
   - `getFlaggedPlugins()` 在 `cache === null` 时返回 `{}`。若调用方忘记先 `await loadFlaggedPlugins()`，会导致 UI 短暂或长期不显示 flagged 插件。当前 `useManagePlugins` 和 `ManagePlugins.tsx` 的调用顺序是正确的，但新增调用方时容易踩坑。

2. **过期清理的触发时机依赖 UI 打开**
   - `loadFlaggedPlugins` 主要在 `useManagePlugins`（REPL mount）和 `pluginBlocklist.ts`（delisting 检测）中调用。如果用户从不进入 `/plugins` 且没有发生新的 delisting 事件，过期的 flagged 条目不会自动清理，会永久留在磁盘上。

3. **并发写盘没有文件锁**
   - 虽然使用了原子 rename，但如果两个进程/线程同时执行 `writeToDisk`，后完成的 rename 会覆盖先完成的，导致数据丢失。Claude Code 通常是单进程，但快速重启或异常恢复时存在理论风险。

4. **无最大条目数限制**
   - 若用户长期运行且 marketplace 频繁下架插件，`flagged-plugins.json` 可能无限增长。虽然单个条目很小，但缺乏上限控制。

5. **`parsePluginsData` 的容错设计**
   - 当文件内容损坏或格式不符合预期时，函数返回 `{}` 而不是抛出异常。这保证了系统不会因为一个损坏的 JSON 而崩溃，但也意味着**数据损坏会被静默丢弃**。

### 改进建议

1. **增加缓存未预热的显式警告或自动回退**
   - 可在 `getFlaggedPlugins()` 内部检测到 `cache === null` 时自动触发一次 `loadFlaggedPlugins()`（异步，不阻塞返回），减少调用方遗漏 `await` 导致的 bug。

2. **启动时统一执行过期清理**
   - 将过期清理逻辑从 `loadFlaggedPlugins` 抽离，在 `main.tsx` 或 `installedPluginsManager.ts` 的初始化路径中固定调用一次，确保即使不打开 UI 也能定期清理旧数据。

3. **增加文件级乐观锁或版本号**
   - 在 `flagged-plugins.json` 中增加一个单调递增的 `version` 或 `seq` 字段，写盘时检查磁盘版本是否与读取时一致，防止并发覆盖导致的数据丢失。

4. **限制最大保留条目数**
   - 在 `loadFlaggedPlugins` 中增加逻辑：若总条目数超过某个阈值（如 100），按 `flaggedAt` 排序仅保留最新的 N 条，防止文件无限膨胀。

5. **数据损坏时备份旧文件**
   - 在 `parsePluginsData` 检测到格式非法时，不要直接返回 `{}`，而是将旧文件重命名为 `flagged-plugins.json.bak.<timestamp>`，便于用户排查和恢复。
