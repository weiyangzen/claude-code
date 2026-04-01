# useFileHistorySnapshotInit.ts 研究文档

## 场景与职责

`useFileHistorySnapshotInit` 是 Claude Code **文件历史（File History / Checkpoint）系统**的初始化 Hook。它的职责是在 REPL 挂载时，如果会话是从 `--continue` 或 `--resume` 恢复的，将之前会话持久化的文件历史快照（`FileHistorySnapshot[]`）重新加载到当前的 `FileHistoryState` 中。

文件历史系统允许用户在对话过程中回溯到任意消息节点，恢复当时的文件系统状态。该 Hook 确保恢复会话时，文件历史状态不会丢失，从而支持跨会话的 rewind 操作。

## 功能点目的

1. **会话恢复时重建文件历史**：当 `initialFileHistorySnapshots` 存在时（来自 resumed log），将其合并到当前的 `fileHistoryState` 中。

2. **防止重复初始化**：使用 `initialized` ref 确保初始化逻辑只执行一次，避免在 `fileHistoryState` 或 `onUpdateState` 引用变化时重复恢复。

3. **尊重功能开关**：如果 `fileHistoryEnabled()` 返回 `false`（如非交互式会话、用户显式禁用、环境变量关闭），则跳过初始化。

## 具体技术实现

### 源码实现

```ts
import { useEffect, useRef } from 'react'
import {
  type FileHistorySnapshot,
  type FileHistoryState,
  fileHistoryEnabled,
  fileHistoryRestoreStateFromLog,
} from '../utils/fileHistory.js'

export function useFileHistorySnapshotInit(
  initialFileHistorySnapshots: FileHistorySnapshot[] | undefined,
  fileHistoryState: FileHistoryState,
  onUpdateState: (newState: FileHistoryState) => void,
): void {
  const initialized = useRef(false)

  useEffect(() => {
    if (!fileHistoryEnabled() || initialized.current) {
      return
    }
    initialized.current = true
    if (initialFileHistorySnapshots) {
      fileHistoryRestoreStateFromLog(initialFileHistorySnapshots, onUpdateState)
    }
  }, [fileHistoryState, initialFileHistorySnapshots, onUpdateState])
}
```

### 设计要点

- **`initialized` ref 的防重入机制**：
  由于 `useEffect` 的依赖数组包含了 `fileHistoryState` 和 `onUpdateState`，而这些引用在 React 生命周期中可能变化，使用 ref 保证 `fileHistoryRestoreStateFromLog` 只被调用一次。

- **`fileHistoryEnabled()` 前置检查**：
  该函数（`src/utils/fileHistory.ts`）检查：
  - 非交互式会话时，需要 `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` 为真
  - 交互式会话时，`globalConfig.fileCheckpointingEnabled !== false` 且 `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` 未设置

- **`fileHistoryRestoreStateFromLog` 的恢复逻辑**：
  该函数（`src/utils/fileHistory.ts`）执行以下操作：
  1. 遍历所有快照
  2. 将快照中的绝对路径转换为相对路径（`maybeShortenFilePath`），减少存储空间
  3. 重建 `trackedFiles` Set
  4. 调用 `onUpdateState` 写入新的 `FileHistoryState`

### 数据结构

- **FileHistorySnapshot**：
  ```ts
  export type FileHistorySnapshot = {
    messageId: UUID
    trackedFileBackups: Record<string, FileHistoryBackup>
    timestamp: Date
  }
  ```

- **FileHistoryState**：
  ```ts
  export type FileHistoryState = {
    snapshots: FileHistorySnapshot[]
    trackedFiles: Set<string>
    snapshotSequence: number
  }
  ```

- **FileHistoryBackup**：
  ```ts
  export type FileHistoryBackup = {
    backupFileName: string | null  // null 表示文件在该版本中不存在
    version: number
    backupTime: Date
  }
  ```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useFileHistorySnapshotInit.ts` | 本 Hook 实现 |
| `src/screens/REPL.tsx` | 唯一调用方，在 REPL 挂载时传入 `initialFileHistorySnapshots` |
| `src/utils/fileHistory.ts` | `fileHistoryEnabled`、`fileHistoryRestoreStateFromLog`、`FileHistoryState`、`FileHistorySnapshot` |
| `src/utils/sessionStorage.ts` | `recordFileHistorySnapshot`：持久化快照到会话存储 |
| `src/utils/config.ts` | `getGlobalConfig`：读取 `fileCheckpointingEnabled` |

## 依赖与外部交互

### 内部依赖
- **React**：`useEffect`、`useRef`
- **文件历史工具**：`src/utils/fileHistory.ts`

### 外部交互
- **会话存储（Session Storage）**：`initialFileHistorySnapshots` 来自 `restoreSessionStateFromLog` 或 `deserializeMessages` 等会话恢复逻辑，这些数据最终从 `~/.claude/sessions/` 或 log 文件中读取。
- **文件系统备份目录**：`fileHistoryRestoreStateFromLog` 恢复状态后，实际的备份文件还需要通过 `copyFileHistoryForResume`（在 `REPL.tsx` 的恢复流程中调用）从旧 session 目录硬链接到新 session 目录。

## 风险、边界与改进建议

### 风险与边界

1. **`fileHistoryState` 在依赖数组中可能导致过早触发**：虽然 `initialized` ref 防止了重复执行，但如果 `fileHistoryEnabled()` 在首次渲染时为 `false`（如配置尚未加载），后续 `fileHistoryState` 变化也不会重新触发，因为 `initialized` 已经被设为 `true`。这意味着如果功能开关在 mount 后变为启用，恢复逻辑将永远被跳过。

2. **`onUpdateState` 的调用时机**：`fileHistoryRestoreStateFromLog` 是同步调用的，它直接调用 `onUpdateState` 设置新状态。如果此时 React 的渲染流程正在进行，可能导致状态更新与初始状态设置冲突。

3. **与 `copyFileHistoryForResume` 的时序依赖**：本 Hook 只恢复了内存中的 `FileHistoryState`，但实际的备份文件（在 `~/.claude/file-history/{sessionId}/` 下）的迁移是由 `REPL.tsx` 在另一个 `useEffect` 中调用 `copyFileHistoryForResume` 完成的。如果这两个操作时序错乱（如用户立即尝试 rewind），可能发现内存状态指向的备份文件尚未迁移。

4. **`initialFileHistorySnapshots` 的路径迁移**：`fileHistoryRestoreStateFromLog` 内部会将快照中的路径从绝对路径缩短为相对路径。如果旧快照中的路径已经是相对路径，或使用了不同的 cwd，可能导致路径解析错误。

5. **无错误处理**：如果 `initialFileHistorySnapshots` 数据损坏（如 `timestamp` 不是有效的 `Date` 对象），`fileHistoryRestoreStateFromLog` 没有 try/catch，异常会直接抛出到 React 的 effect 执行中。

### 改进建议

1. **延迟设置 `initialized` 标志**：将 `initialized.current = true` 移到 `fileHistoryRestoreStateFromLog` 成功执行之后，这样如果首次尝试失败（如 `fileHistoryEnabled()` 为 false），后续还有机会重试。
   ```ts
   useEffect(() => {
     if (!fileHistoryEnabled() || initialized.current) return
     if (initialFileHistorySnapshots) {
       fileHistoryRestoreStateFromLog(initialFileHistorySnapshots, onUpdateState)
     }
     initialized.current = true
   }, [...])
   ```

2. **将文件迁移也纳入 Hook 职责**：当前文件历史恢复逻辑分散在 `useFileHistorySnapshotInit`（恢复内存状态）和 `REPL.tsx` 的 `useEffect`（迁移备份文件）中。建议将两者统一到一个 Hook 或一个恢复函数中，确保原子性。

3. **增加数据校验**：在恢复前对 `initialFileHistorySnapshots` 进行轻量级校验（如检查 `messageId` 是否存在、`trackedFileBackups` 是否为对象），避免损坏数据导致崩溃。

4. **使用 `useLayoutEffect` 替代 `useEffect`**：由于这是一个需要在用户可见交互前完成的初始化操作，使用 `useLayoutEffect` 可以确保状态在首次绘制前就已经恢复，避免一闪而过的空状态。

5. **日志与可观测性**：增加恢复成功的日志记录（如恢复了几个快照、跟踪了多少文件），便于排查会话恢复相关的问题。
