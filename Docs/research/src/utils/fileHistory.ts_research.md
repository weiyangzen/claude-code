# 研究文档：src/utils/fileHistory.ts

## 场景与职责

`fileHistory.ts` 是 Claude Code 的**文件检查点（file checkpointing）**核心模块，负责在对话过程中跟踪用户文件变更、创建快照备份，并支持按消息维度回滚（rewind）到历史状态。它相当于一个轻量级的、按 message UUID 索引的版本控制系统，集成在 REPL 主循环中，为 `/clear conversation`、`resume`、以及未来可能的 UI 回滚提供数据基础。

该模块同时服务于：
- **交互式会话**：在每次用户提交后自动快照。
- **SDK/非交互式会话**：通过环境变量显式控制开关。
- **会话恢复（resume）**：将历史快照从日志中还原，并硬链接备份文件到新 session。
- **VSCode 扩展联动**：通过 MCP 通知文件更新事件。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `fileHistoryTrackEdit` | 在文件被修改**之前**记录其 v1 备份，防止后续编辑覆盖原始内容。 |
| `fileHistoryMakeSnapshot` | 在每个用户消息（message UUID）处创建快照，备份所有被跟踪文件的当前版本。 |
| `fileHistoryRewind` | 按 message UUID 将磁盘文件恢复到该快照时的状态（删除/覆盖/保留）。 |
| `fileHistoryGetDiffStats` | 计算回滚到某快照时的变更统计（文件数、插入、删除），供 UI 展示。 |
| `fileHistoryHasAnyChanges` | 轻量版 diff 检查，只回答“是否有变化”，不做逐行 diff。 |
| `fileHistoryRestoreStateFromLog` | 从持久化日志中重建 `FileHistoryState`，支持 session resume。 |
| `copyFileHistoryForResume` | 跨 session 恢复时，通过硬链接（fallback 复制）迁移备份文件。 |
| `notifyVscodeSnapshotFilesUpdated` | 对比相邻快照，向 VSCode 发送 `file_updated` 通知。 |

## 具体技术实现

### 核心数据结构

```ts
type FileHistoryBackup = {
  backupFileName: string | null  // null 表示该版本文件不存在
  version: number
  backupTime: Date
}

type FileHistorySnapshot = {
  messageId: UUID
  trackedFileBackups: Record<string, FileHistoryBackup>
  timestamp: Date
}

type FileHistoryState = {
  snapshots: FileHistorySnapshot[]
  trackedFiles: Set<string>
  snapshotSequence: number  // 单调递增活动信号
}
```

- 用**相对路径**（`maybeShortenFilePath`）作为 `trackingPath`，减少 session storage 体积。
- `MAX_SNAPSHOTS = 100`，超出时滑动窗口丢弃最旧快照，但 `snapshotSequence` 继续递增，为 `useGitDiffStats` 提供非饱和的活动信号。

### 备份文件命名与存储

```ts
function getBackupFileName(filePath: string, version: number): string {
  const fileNameHash = createHash('sha256')
    .update(filePath)
    .digest('hex')
    .slice(0, 16)
  return `${fileNameHash}@v${version}`
}
```

- 存储目录：`~/.claude/file-history/{sessionId}/{hash}@v{N}`
- 使用 `fs/promises.copyFile` 直接复制（避免读入 JS heap），并 `chmod` 保留原权限。
- 懒创建目录：先 `copyFile`，遇 `ENOENT` 再 `mkdir` 重试。

### 变更检测流程（`checkOriginFileChanged`）

1. 比较 `mode` 和 `size`，不同则直接判定变更。
2. 若 `originalStats.mtimeMs < backupStats.mtimeMs`，判定未变更（优化路径）。
3. 否则读取双方内容做字符串比对。

### 三阶段状态更新模式

所有会修改状态的操作（`trackEdit`、`makeSnapshot`）都遵循：
1. **Capture**：通过 no-op updater 读取当前 state。
2. **Async I/O**：在 updater 外执行所有文件操作。
3. **Commit**：再次进入 updater，合并异步结果并生成新 state。

这种设计避免在 React state updater 中执行异步 IO，同时通过引用相等性优化（same-ref return = no-op）减少不必要的重渲染。

### 恢复流程（`applySnapshot`）

遍历 `state.trackedFiles`：
- `backupFileName === null` → `unlink(filePath)`（文件当时不存在）。
- `backupFileName === undefined` → 跳过（无法解析到备份，安全失败）。
- 其他 → 若 `checkOriginFileChanged` 为 true，则 `copyFile(backupPath, filePath)` 并恢复权限。

### Resume 迁移

`copyFileHistoryForResume`：
- 从 log 的 `fileHistorySnapshots` 读取旧 session 的备份清单。
- 对新 session 的备份目录先 `mkdir`。
- 对每个备份尝试 `link(old, new)`；若失败（`EEXIST` 跳过，`ENOENT` 报错，其他 fallback 到 `copyFile`）。
- 成功后把快照记录到新 session 的 storage。

## 关键代码路径与文件引用

### 调用方（入口）

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/utils/handlePromptSubmit.ts:527-537` | `fileHistoryMakeSnapshot` | 每次用户输入处理完成后，对每个 selectable user message 创建快照。 |
| `src/tools/FileEditTool/FileEditTool.ts` | `fileHistoryTrackEdit` | 编辑文件前跟踪原内容。 |
| `src/tools/FileWriteTool/FileWriteTool.ts` | `fileHistoryTrackEdit` | 写入文件前跟踪原内容。 |
| `src/tools/NotebookEditTool/NotebookEditTool.ts` | `fileHistoryTrackEdit` | 笔记本编辑前跟踪。 |
| `src/utils/conversationRecovery.ts:550` | `copyFileHistoryForResume` | 恢复会话时迁移备份。 |
| `src/utils/sessionRestore.ts:105` | `fileHistoryRestoreStateFromLog` | 从日志恢复 state 到 AppState。 |
| `src/hooks/useFileHistorySnapshotInit.ts` | 初始化/监听 | React hook 层对接。 |
| `src/QueryEngine.ts` / `src/screens/REPL.tsx` | `fileHistoryRewind` / `fileHistoryCanRestore` | UI 回滚操作。 |

### 被调用方（依赖）

- `src/bootstrap/state.js`：`getIsNonInteractiveSession`, `getOriginalCwd`, `getSessionId`
- `src/services/analytics/index.js`：`logEvent`
- `src/services/mcp/vscodeSdkMcp.js`：`notifyVscodeFileUpdated`
- `src/utils/sessionStorage.js`：`recordFileHistorySnapshot`
- `src/utils/config.js`：`getGlobalConfig`
- `src/utils/envUtils.js`：`getClaudeConfigHomeDir`, `isEnvTruthy`
- `src/utils/errors.js`：`getErrnoCode`, `isENOENT`
- `src/utils/file.js`：`pathExists`
- `src/utils/debug.js`：`logForDebugging`
- `src/utils/log.js`：`logError`
- 外部：`diff` 包的 `diffLines`

## 依赖与外部交互

### 环境变量控制

| 变量 | 作用 |
|------|------|
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | 全局关闭（任意 truthy 值）。 |
| `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` | 非交互式会话中显式开启。 |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | 同时影响交互式与非交互式。 |

### 配置项

- `getGlobalConfig().fileCheckpointingEnabled`：交互式会话的默认开关（`!== false` 即开启）。

### 持久化

- 备份文件落盘在 `~/.claude/file-history/{sessionId}/`。
- 快照元数据通过 `recordFileHistorySnapshot` 写入 session 的 transcript/log（JSONL），与消息链共存。

### VSCode 联动

- `notifyVscodeSnapshotFilesUpdated` 在 `makeSnapshot` 成功后 fire-and-forget 调用，对比旧/新快照内容，向 VSCode MCP 发送 `file_updated` 事件。

## 风险、边界与改进建议

### 风险

1. **竞态条件（Race）**：`trackEdit` 和 `makeSnapshot` 都可能在 async 窗口期间被并发调用。代码通过 commit 阶段重新检查 `mostRecentSnapshot.trackedFileBackups[trackingPath]` 来避免重复备份，但 state updater 的调度仍依赖 React 的批处理语义；若 updater 被延迟，理论上可能丢失 track。
2. **大文件 OOM**：虽然 `copyFile` 避免了读入 heap，但 `checkOriginFileChanged` 在 mtime 无法短路时仍会 `readFile` 整个文件到内存做字符串比较。对于 GB 级文件存在内存风险。
3. **备份目录膨胀**：每个被跟踪文件的每次变更都产生新版本，100 个快照 × N 个文件可能产生大量小文件；目前无自动清理旧 session 备份的机制（依赖 `src/utils/cleanup.ts` 的定期清理）。
4. **跨平台路径**：`maybeShortenFilePath` 使用 `relative(cwd, filePath)`，在 Windows 上可能产生反斜杠，而备份哈希基于原始路径字符串，不同路径表示可能产生不同哈希。
5. **ENOENT 误分类**：早期版本在 `createBackup` 中共享 catch 导致“文件删除”与“目录缺失”混淆，当前版本已修复为先 `stat` 再 `copyFile`。

### 边界

- 只跟踪被显式编辑/写入的文件；被动读取的文件不进入 `trackedFiles`。
- `MAX_SNAPSHOTS = 100` 是硬编码上限，无法配置。
- 快照只关联到**用户消息**的 UUID，不关联到 assistant/tool 消息。
- `fileHistoryHasAnyChanges` 使用 `checkOriginFileChanged`（stat + 可选内容比较），而 `fileHistoryGetDiffStats` 使用 `diffLines` 逐行计算，两者在“是否变更”的判定上理论上应一致，但 `diffLines` 对空文件 vs 零字节文件的处理有额外分支（`backupFileName === null && pathExists`）。

### 改进建议

1. **大文件内容比较优化**：对超过阈值（如 10MB）的文件改用流式 hash 比较，而非全量读入内存。
2. **配置化上限**：将 `MAX_SNAPSHOTS` 暴露为配置项，或按磁盘配额动态管理。
3. **自动清理策略**：在 session 过期/删除时联动清理 `file-history/{sessionId}` 目录，避免无限增长。
4. **并发控制**：考虑用 `AsyncLock` 或队列串行化同一文件的 `trackEdit` + `makeSnapshot`，彻底消除竞态。
5. **测试覆盖**：当前仓库中未找到针对 `fileHistory.ts` 的单元测试，建议补充对 `createBackup`、`applySnapshot`、`copyFileHistoryForResume` 的集成测试。
