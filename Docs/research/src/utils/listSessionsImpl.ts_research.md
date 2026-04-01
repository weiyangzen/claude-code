# src/utils/listSessionsImpl.ts 研究文档

## 场景与职责

`listSessionsImpl.ts` 是 Claude Code Agent SDK 的会话列表功能的核心实现。它的设计目标非常明确：**保持最小依赖、可移植、无 CLI 初始化副作用**。模块顶部注释强调它不依赖 `bootstrap/state.ts`、analytics、`bun:bundle` 或模块级可变状态，因此可以安全地从 SDK 入口点导入。

主要职责：
1. **会话发现**：扫描项目目录下的 `.jsonl` 会话文件。
2. **元数据提取**：通过廉价的 `stat` + 头/尾读取（而非完整 JSONL 解析）提取会话摘要、标题、分支、标签等信息。
3. **分页与排序**：支持 `limit`/`offset` 分页，以及按 `lastModified` 降序排序。
4. **Git worktree 感知**：当指定 `dir` 时，可以跨 git worktree 查找相关会话。
5. **性能优化**：在分页场景下，先用 `stat` -only 路径筛选候选文件，再对排序后的前 N 个文件做昂贵的头/尾读取，避免对大量会话文件进行全量 I/O。

调用方：
- `src/utils/sessionStorage.ts`：CLI 的会话列表功能
- `src/services/autoDream/consolidationLock.ts`：自动梦境（autoDream）服务

## 功能点目的

### 1. `listSessionsImpl`
主入口函数，接受 `ListSessionsOptions`：
- `dir?: string`：指定项目目录，返回该目录（及 worktree）的会话。
- `limit?: number`：最大返回数量（0 表示无限制）。
- `offset?: number`：跳过的记录数。
- `includeWorktrees?: boolean`：是否包含 git worktree 中的会话（默认 `true`）。

核心逻辑分支：
- 若 `limit > 0` 或 `offset > 0`，启用 `doStat = true` 路径：先 `stat` 所有候选文件排序，再批量读取前 N 个。
- 否则 `doStat = false`：读取所有候选文件的头/尾，然后统一排序（保持与旧实现相同的 I/O 成本）。

### 2. `parseSessionInfoFromLite`
从 `LiteSessionFile`（头、尾、stat 信息）中解析 `SessionInfo`：
- **过滤 sidechain 会话**：检查第一行是否包含 `"isSidechain":true`。
- **标题提取优先级**：`customTitle`（用户设置） > `aiTitle`（AI 生成） > `lastPrompt`（最后提示） > `summary` > `firstPrompt`。
- **时间戳**：从第一条记录的 `timestamp` 字段解析 `createdAt`。
- **标签提取**：专门查找 `{"type":"tag"}` 开头的 JSONL 行，避免与工具参数中的 `tag` 字段冲突。
- **过滤无意义会话**：若没有任何可提取的摘要/标题/提示，返回 `null`。

### 3. `listCandidates`
在单个项目目录中列出候选 `.jsonl` 文件：
- 过滤非 `.jsonl` 结尾的文件。
- 验证文件名前 36 字符是否为有效 UUID。
- 可选 `stat` 获取 `mtime`。

### 4. `applySortAndLimit`
分页路径的排序和限制应用：
- 按 `mtime desc` 排序，相同 `mtime` 按 `sessionId desc` 稳定排序。
- 批量读取（`READ_BATCH_SIZE = 32`），并发读取一批候选文件的头/尾。
- 去重：同一 `sessionId` 只保留第一个（即最新）的有效副本。
- 支持 `offset` 跳过前 N 条记录。

### 5. `readAllAndSort`
无分页路径的全量读取和排序：
- 并发读取所有候选文件。
- 使用 `Map<string, SessionInfo>` 去重，保留 `lastModified` 最新的副本。
- 最后统一排序。

### 6. `gatherProjectCandidates` / `gatherAllCandidates`
- `gatherProjectCandidates`：处理单项目 + worktree 的候选收集。涉及路径匹配和 sanitize 逻辑。
- `gatherAllCandidates`：处理跨所有项目的候选收集。

## 具体技术实现

### 性能优化策略
分页场景的核心优化：
```typescript
const doStat = (limit !== undefined && limit > 0) || off > 0
```

当 `doStat = true` 时：
1. `listCandidates` 对每个 `.jsonl` 文件调用 `stat`，获取 `mtime`。
2. `applySortAndLimit` 先按 `mtime` 排序。
3. 然后以 32 个文件为一批，并发调用 `readCandidate`（读取头/尾并解析）。
4. 一旦收集到足够的有效会话（`sessions.length >= want`）就提前停止。

这意味着：如果一个目录有 1000 个会话文件，但用户只请求前 20 个，系统大约只需要 1000 次 `stat` + 约 20 次头/尾读取，而不是 1000 次完整读取。

### Worktree 匹配逻辑
```typescript
const isMatch =
  dirName === prefix ||
  (prefix.length >= MAX_SANITIZED_LENGTH &&
    dirName.startsWith(prefix + '-'))
```

- 短路径要求精确匹配（避免 `/root/project` 错误匹配 `/root/project-foo`）。
- 长路径（超过 `MAX_SANITIZED_LENGTH`，即被截断并加了 hash 后缀的路径）允许 `startsWith(prefix + '-')` 的模糊匹配。

### 元数据提取的鲁棒性
`parseSessionInfoFromLite` 大量使用了 `sessionStoragePortable.ts` 提供的字符串模式匹配函数（`extractJsonStringField`、`extractLastJsonStringField` 等），而不是完整的 `JSON.parse`。这使得即使头/尾读取截断了某条 JSONL 记录，仍然能正确提取字段。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/listSessionsImpl.ts:79-149` | `parseSessionInfoFromLite` 元数据解析 |
| `src/utils/listSessionsImpl.ts:169-198` | `listCandidates` 候选发现 |
| `src/utils/listSessionsImpl.ts:235-271` | `applySortAndLimit` 分页排序 |
| `src/utils/listSessionsImpl.ts:278-299` | `readAllAndSort` 全量排序 |
| `src/utils/listSessionsImpl.ts:309-401` | `gatherProjectCandidates` 项目 + worktree 收集 |
| `src/utils/listSessionsImpl.ts:439-454` | `listSessionsImpl` 主入口 |
| `src/utils/sessionStoragePortable.ts` | `readSessionLite`, `extractJsonStringField`, `validateUuid`, `sanitizePath`, `getProjectsDir`, `findProjectDir`, `canonicalizePath` |
| `src/utils/getWorktreePathsPortable.ts` | `getWorktreePathsPortable` |

## 依赖与外部交互

### 内部依赖
- `./sessionStoragePortable.js`：大量便携式会话存储工具
- `./getWorktreePathsPortable.js`：git worktree 路径获取
- `fs/promises`：`readdir`, `stat`
- `path`：`join`, `basename`
- `fs`：`Dirent` 类型

### 外部依赖
- 无第三方依赖。

### 调用方
- `src/utils/sessionStorage.ts`
- `src/services/autoDream/consolidationLock.ts`

## 风险、边界与改进建议

### 风险与边界
1. **`stat` 时间不可靠**：`mtime` 可能因文件复制、同步、备份工具操作而改变，不完全反映会话最后活动时间。虽然这是现有设计接受的权衡，但在某些边缘场景下排序可能不符合用户预期。
2. **`firstTimestamp` 解析依赖 ISO 格式**：若会话文件的第一条记录时间戳不是标准 ISO 格式（如旧版本使用其他格式），`Date.parse` 会返回 `NaN`，`createdAt` 会被设为 `undefined`。
3. **Worktree 匹配的安全边界**：`startsWith(prefix + '-')` 的模糊匹配虽然解决了长路径截断问题，但理论上仍可能产生误匹配（虽然概率极低，因为 hash 后缀提供了足够的熵）。
4. **无文件内容校验**：`listCandidates` 只检查文件名是否为 UUID 且以 `.jsonl` 结尾，不验证文件内容是否真的是有效的 JSONL。若目录中有空文件或损坏文件，它们会参与 `stat` 排序，但在 `readCandidate` 阶段可能被过滤掉，导致实际返回数量少于 `limit`。
5. **`readAllAndSort` 的内存使用**：在无分页路径下，所有候选文件的头/尾会并发读取。若项目有数万个会话文件，可能产生大量并发 I/O 和内存占用。

### 改进建议
1. **增加文件大小过滤**：在 `listCandidates` 中过滤掉 0 字节文件，减少无效 I/O。
2. **`stat` 排序的替代方案**：考虑维护一个轻量级的索引文件（如 `~/.claude/sessions/index.jsonl`），记录每个会话的 `sessionId`、`lastModified`、`summary`，避免每次列表时都扫描整个目录。
3. **更精确的 worktree 匹配**：对于长路径，可以将完整路径的 hash 作为目录名的一部分，确保匹配是确定性的而非基于 `startsWith`。
4. **并发限制**：`readAllAndSort` 中的 `Promise.all(candidates.map(readCandidate))` 在候选数量极大时可能导致 EMFILE（打开文件过多）。可以引入 `p-limit` 或类似的并发控制。
5. **元数据缓存**：`parseSessionInfoFromLite` 的结果可以按 `filePath + mtime` 做短期缓存，避免在短时间内重复列表时重复解析相同的文件。
