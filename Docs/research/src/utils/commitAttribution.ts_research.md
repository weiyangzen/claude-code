# commitAttribution.ts 研究文档

> 文件路径：`src/utils/commitAttribution.ts`  
> 行数：961 行  
> 研究日期：2026-04-01

---

## 场景与职责

`commitAttribution.ts` 是 Claude Code **提交归因（commit attribution）** 子系统的核心引擎。其目标是在用户通过 Claude 修改代码并执行 `git commit` 时，量化 Claude 对该次提交内容的贡献比例，并将这些元数据写入 git notes 或 commit message trailer 中。

**业务背景：**
- 用户可能在与 Claude 的多轮对话中逐步修改文件；Claude 需要追踪“哪些字符是 Claude 写的，哪些是人类写的”。
- 归因数据用于生成 `Co-Authored-By`、PR 描述中的贡献百分比、以及内部 telemetry。
- 对于外部（public/open-source）仓库，必须避免泄露内部模型代号（codename），因此需要模型名称的“消毒”（sanitize）逻辑。

**主要职责：**
1. 维护每轮会话的归因状态（`AttributionState`）。
2. 在文件被修改/创建/删除时，计算 Claude 的字符贡献量。
3. 在提交时，对比会话基线与暂存区状态，生成最终的 `AttributionData`。
4. 提供模型名称消毒、仓库内外部判定、git 暂存区查询等辅助能力。

---

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `createEmptyAttributionState` | 创建新的归因状态，包含文件状态映射、会话基线、surface、prompt 计数等 |
| `trackFileModification` | 在 Edit/Write 工具完成后调用，计算并累加单文件的字符贡献 |
| `trackFileCreation` / `trackFileDeletion` | 处理通过 Bash 等非工具路径创建/删除的文件 |
| `trackBulkFileChanges` | 批量处理大量文件变更（优化 Map 拷贝，避免 O(n²)） |
| `calculateCommitAttribution` | 核心：给定多个会话的归因状态和暂存文件列表，计算最终归因数据 |
| `getStagedFiles` / `isFileDeleted` / `getGitDiffSize` | git 辅助：获取暂存区文件、判断删除、估算 diff 大小 |
| `isInternalModelRepo` / `isInternalModelRepoCached` | 判断当前仓库是否为内部私有仓库，决定能否使用内部模型名 |
| `sanitizeModelName` / `sanitizeSurfaceKey` / `buildSurfaceKey` | 将内部模型名映射为公开名称，防止 codename 泄露 |
| `stateToSnapshotMessage` / `restoreAttributionStateFromSnapshots` | 归因状态的持久化与恢复（用于会话恢复/压实） |
| `incrementPromptCount` | 每次用户发送消息后增加 prompt 计数并保存快照 |
| `isGitTransientState` | 检测是否处于 rebase/merge/cherry-pick 等暂态，避免错误归因 |

---

## 具体技术实现

### 3.1 归因状态 `AttributionState`

```ts
export type AttributionState = {
  fileStates: Map<string, FileAttributionState>
  sessionBaselines: Map<string, { contentHash: string; mtime: number }>
  surface: string
  startingHeadSha: string | null
  promptCount: number
  promptCountAtLastCommit: number
  permissionPromptCount: number
  permissionPromptCountAtLastCommit: number
  escapeCount: number
  escapeCountAtLastCommit: number
}
```

- `fileStates`：以**相对于 cwd 的归一化路径**为键，记录每个文件的累计 Claude 贡献字符数、内容哈希、修改时间。
- `sessionBaselines`：会话开始时的文件状态快照，用于计算 net change。
- `surface`：用户入口点（如 `cli`、`vscode`、`desktop` 等），与模型名组合成 `surface/model` 的归因标签。
- `promptCount` 等：用于计算 "steers"（用户引导次数 = promptCount - 1），写入 commit trailer。

### 3.2 单文件贡献计算 `computeFileModificationState`

**核心算法：公共前后缀匹配**

对于旧内容 `oldContent` 和新内容 `newContent`：
1. 从头扫描找到最长公共前缀 `prefixEnd`。
2. 从尾扫描找到最长公共后缀 `suffixLen`。
3. 实际变更区域长度：
   - `oldChangedLen = oldContent.length - prefixEnd - suffixLen`
   - `newChangedLen = newContent.length - prefixEnd - suffixLen`
4. Claude 贡献 = `max(oldChangedLen, newChangedLen)`

**为什么用 max？**
- 若 Claude 把 `abc` 替换为 `xyz`，前后缀均为 0，贡献为 `max(3, 3) = 3`。
- 若 Claude 删除一段内容，旧内容变长大于新内容，贡献以删除的字符数计。
- 若 Claude 新增内容，贡献以新增字符数计。
- 该算法能正确处理等长替换（如 `"Esc" → "esc"`），而简单的 `abs(newLen - oldLen)` 会返回 0。

**边界情况：**
- 新建文件（`oldContent === ''`）：贡献 = `newContent.length`
- 完全删除（`newContent === ''`）：贡献 = `oldContent.length`

### 3.3 批量变更优化 `trackBulkFileChanges`

在处理大型 git 操作（如 `jj` 可能触及数十万文件）时，若对每个文件都复制一次 `Map`，时间复杂度为 O(n²)。`trackBulkFileChanges` 采用**单次 Map 拷贝 + 原地修改**策略：

```ts
const newFileStates = new Map(state.fileStates)
for (const change of changes) {
  // 直接修改 newFileStates
  newFileStates.set(normalizedPath, newFileState)
}
return { ...state, fileStates: newFileStates }
```

### 3.4 提交时归因计算 `calculateCommitAttribution`

输入：
- `states: AttributionState[]` — 可能跨多个会话（如长会话被压实后恢复）
- `stagedFiles: string[]` — 当前 git 暂存区文件列表

流程：
1. **合并多会话数据**
   - 合并 `sessionBaselines`：最早基线优先。
   - 合并 `fileStates`：同一文件的贡献字符数**累加**。
2. **并行处理每个暂存文件**
   - 跳过生成文件（`isGeneratedFile(file)`）。
   - 判断文件是否被删除（`isFileDeleted`）。
   - **若文件在 `mergedFileStates` 中**：Claude 贡献 = `fileState.claudeContribution`，人类贡献 = 0。
   - **若文件不在 `mergedFileStates` 中但在 `mergedBaselines` 中**：人类修改，贡献通过 `git diff --cached --stat` 估算（`getGitDiffSize`）。
   - **若文件既不在状态也不在基线中**：视为人类新建文件，人类贡献 = 文件当前大小（`stat.size`）。
3. **聚合结果**
   - 计算总 Claude 百分比：`round(totalClaudeChars / totalChars * 100)`
   - 按 surface 拆分贡献。
   - 返回 `AttributionData`（版本 1，含 summary、files、surfaceBreakdown、excludedGenerated、sessions）。

### 3.5 内部/外部仓库判定 `isInternalModelRepo`

```ts
const INTERNAL_MODEL_REPOS = [
  'github.com:anthropics/claude-cli-internal',
  'github.com/anthropics/anthropic',
  // ... 约 30+ 条
]
```

- 通过 `getRemoteUrlForDir(cwd)` 获取当前仓库 remote URL。
- 使用 `sequential()` 包装，保证进程内只执行一次异步检查。
- 结果缓存到 `repoClassCache: 'internal' | 'external' | 'none'`。
- **注意**：这是 repo 级白名单，而非 org 级。因为 `anthropics` 和 `anthropic-experimental` 组织下有 public repo（如 `claude-code`、`sandbox-runtime`），不能一刀切放行。

### 3.6 模型名称消毒 `sanitizeModelName`

```ts
export function sanitizeModelName(shortName: string): string {
  if (shortName.includes('opus-4-6')) return 'claude-opus-4-6'
  if (shortName.includes('opus-4-5')) return 'claude-opus-4-5'
  if (shortName.includes('opus-4')) return 'claude-opus-4'
  if (shortName.includes('sonnet-4-6')) return 'claude-sonnet-4-6'
  // ...
  return 'claude'  // 未知模型兜底
}
```

- 基于子串包含匹配，顺序很重要（更具体的型号必须排在更通用的前面，如 `opus-4-1` 必须在 `opus-4` 之前）。
- 注释中带有 `@[MODEL LAUNCH]` 标记，提示每次发布新模型时必须更新映射。

### 3.7 状态恢复陷阱与修复

`restoreAttributionStateFromSnapshots` 在早期版本中存在一个严重 bug：
- 快照是**全量状态 dump**（不是增量 delta）。
- 旧实现遍历所有快照并对 `fileStates` 求和，导致恢复时贡献字符数呈**二次增长**（如 837 个快照 × 280 个文件 → 对一个 5KB 文件追踪到 1.15 千万亿字符）。

**修复方式：** 只取最后一个快照（`snapshots[snapshots.length - 1]`），因为 `fileStates` 只会增长不会收缩，最后一个快照已经包含最新状态。

---

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.js` | `getOriginalCwd`, `getSessionId` |
| `src/types/logs.js` | `AttributionSnapshotMessage`, `FileAttributionState` |
| `src/utils/cwd.js` | `getCwd`（支持 AsyncLocalStorage 工作树覆盖） |
| `src/utils/debug.js` | `logForDebugging` |
| `src/utils/execFileNoThrow.js` | `execFileNoThrowWithCwd` — 执行 git 命令 |
| `src/utils/fsOperations.js` | `getFsImplementation` |
| `src/utils/generatedFiles.js` | `isGeneratedFile` — 跳过生成文件 |
| `src/utils/git/gitFilesystem.js` | `getRemoteUrlForDir`, `resolveGitDir` |
| `src/utils/git.js` | `findGitRoot`, `gitExe` |
| `src/utils/log.js` | `logError` |
| `src/utils/model/model.js` | `getCanonicalName`, `ModelName` |
| `src/utils/sequential.js` | `sequential` — 保证 `isInternalModelRepo` 只执行一次 |

### 调用方
| 文件 | 调用场景 |
|------|----------|
| `src/utils/attribution.ts` | 封装归因文本生成（`getAttributionTexts`），调用 `calculateCommitAttribution` 等 |
| `src/commands/commit.ts` | `git commit` 命令执行时计算归因并写入 commit message |
| `src/commands/commit-push-pr.ts` | commit + push + PR 流程 |
| `src/state/AppStateStore.ts` | 维护全局归因状态，调用 `trackFileModification` |
| `src/QueryEngine.ts` | 查询引擎中触发归因计算 |
| `src/screens/REPL.tsx` | REPL 中文件修改后更新归因状态 |
| `src/main.tsx` | 应用启动时恢复归因状态 |
| `src/utils/sessionRestore.ts` | 会话恢复时重建归因状态 |
| `src/utils/undercover.ts` | 判断是否处于 undercover 模式（影响归因文本） |
| `src/setup.ts` | 初始化设置 |
| `src/Tool.ts` | 工具层可能触发归因追踪 |
| `src/tools/BashTool/prompt.ts` | Bash 工具执行后追踪文件创建/删除 |

---

## 依赖与外部交互

### 外部系统交互
- **Git 命令行**：通过 `execFileNoThrowWithCwd` 调用 `git diff --cached --stat`、`git diff --cached --name-only`、`git diff --cached --name-status` 等。
- **文件系统**：使用 `fs/promises.stat` 获取文件大小（作为字符数代理），使用 `fs.realpathSync` 处理 macOS `/tmp` → `/private/tmp` 等符号链接差异。
- **进程环境**：读取 `CLAUDE_CODE_ENTRYPOINT` 确定 surface；读取 `USER_TYPE` 判断内部构建。

### 数据持久化
- 归因快照通过 `stateToSnapshotMessage` 序列化为 `AttributionSnapshotMessage`，写入会话日志（transcript JSONL）。
- 会话恢复时从日志中读取快照数组，调用 `restoreAttributionStateFromSnapshots` 重建状态。

---

## 风险、边界与改进建议

### 风险与边界

1. **字符数代理精度**
   - 使用 `stats.size`（字节数）代替真实字符数。对于纯 ASCII 文件这是准确的，但对于 UTF-8 多字节字符（如中文），字节数 > 字符数，会导致人类贡献被高估。

2. **git diff size 估算粗糙**
   - `getGitDiffSize` 将 insertions + deletions 乘以固定系数 40（假设平均每行 40 字符）。对于长行或二进制文件，估算误差可能很大。

3. **路径归一化跨平台**
   - `normalizeFilePath` 使用 `fs.realpathSync` 解析符号链接，然后计算相对路径。若文件在会话期间被移动或链接关系变化，路径键可能不一致，导致贡献分散到两个键上。

4. **模型消毒映射维护负担**
   - 每次新模型发布都必须手动更新 `sanitizeModelName` 和 `INTERNAL_MODEL_REPOS`，容易遗漏。遗漏会导致外部仓库 commit trailer 中出现内部 codename。

5. **并发提交与 transient state**
   - `isGitTransientState` 检测 rebase/merge/cherry-pick，但调用方是否在每个 commit 前都检查并不明确。如果在 rebase 过程中提交，归因数据可能被附加到错误的 commit。

### 改进建议

1. **精确字符计数**
   将 `stats.size` 替换为 `Buffer.toString('utf-8').length` 或更高效的流式字符计数器，减少 UTF-8 非 ASCII 内容的误差。

2. **自动化模型映射检查**
   在 CI 中增加一个测试，扫描 `src/utils/model/model.ts` 中定义的所有模型名，确保每个都在 `sanitizeModelName` 中有对应映射；否则构建失败。

3. **路径键稳定性**
   考虑使用 `git ls-files --full-name` 或 `git rev-parse --show-prefix` 来归一化路径，而不是依赖 `fs.realpathSync`，以更好地与 git 内部路径保持一致。

4. **批量 diff size 优化**
   当前 `calculateCommitAttribution` 对每个暂存文件单独调用 `getGitDiffSize`（即单独执行一次 `git diff --cached --stat -- <file>`）。可改为单次 `git diff --cached --stat` 然后解析所有文件的变更行数，减少子进程开销。

5. **增加单测覆盖**
   重点测试：
   - `computeFileModificationState` 的等长替换、全删、全增场景
   - `restoreAttributionStateFromSnapshots` 的多快照恢复（只取最后一条）
   - `calculateCommitAttribution` 对生成文件排除、删除文件处理、人类修改估算
