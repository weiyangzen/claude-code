# 研究文档：src/utils/sessionFileAccessHooks.ts

## 场景与职责

`sessionFileAccessHooks.ts` 是 Claude Code 的**会话文件访问分析钩子注册器**。它通过 `PostToolUse` hook 机制，在用户通过工具（Read、Grep、Glob、Edit、Write）访问特定类型的文件时，记录分析事件（analytics events）。这些事件用于：

1. **会话内存（session memory）访问追踪**：监控模型是否读取/修改了 `~/.claude/session-memory/` 下的文件。
2. **会话转录（session transcript）访问追踪**：监控对 `~/.claude/projects/*.jsonl` 的访问。
3. **自动记忆（memdir）访问追踪**：监控对自动记忆目录的读/写/编辑。
4. **团队记忆（team memory）访问追踪**：在 `TEAMMEM` feature flag 开启时，追踪团队记忆文件并通知同步 watcher。
5. **记忆形状遥测（MEMORY_SHAPE_TELEMETRY）**：记录记忆写入的结构特征。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `isMemoryFileAccess(toolName, toolInput)` | 判断某次工具调用是否构成了对 memory 文件的访问（包括 session memory 和 memdir）。 |
| `handleSessionFileAccess(input, toolUseID, signal)` | `PostToolUse` 回调：解析工具输入，检测文件类型，发射对应的 analytics events。 |
| `registerSessionFileAccessHooks()` | 在 CLI 初始化时注册 `PostToolUse` 回调，绑定到 Read/Grep/Glob/Edit/Write 五种工具。 |

---

## 具体技术实现

### 1. 文件路径提取

```ts
function getFilePathFromInput(toolName: string, toolInput: unknown): string | null
```

支持的工具：
- `FILE_READ_TOOL_NAME` → 解析 `file_path`
- `FILE_EDIT_TOOL_NAME` → 解析 `file_path`
- `FILE_WRITE_TOOL_NAME` → 解析 `file_path`

使用对应工具的 `inputSchema.safeParse()` 做安全解析，失败返回 `null`。

### 2. 会话文件类型检测

```ts
function getSessionFileTypeFromInput(toolName, toolInput): 'session_memory' | 'session_transcript' | null
```

- **Read**：直接检测 `file_path`
- **Grep**：先检测 `path`，再检测 `glob`
- **Glob**：先检测 `path`，再检测 `pattern`

检测逻辑委托给 `memoryFileDetection.ts` 的 `detectSessionFileType` 和 `detectSessionPatternType`。

### 3. `isMemoryFileAccess`

判断逻辑：
1. 若 `getSessionFileTypeFromInput(...) === 'session_memory'` → `true`
2. 否则提取 `filePath`，若 `isAutoMemFile(filePath)` 或 `teamMemPaths!.isTeamMemFile(filePath)` → `true`
3. 否则 `false`

### 4. `handleSessionFileAccess`

仅在 `hook_event_name === 'PostToolUse'` 时生效。事件发射矩阵：

| 检测到的文件类型 | 发射的事件 |
|------------------|-----------|
| `session_memory` | `tengu_session_memory_accessed` |
| `session_transcript` | `tengu_transcript_accessed` |
| auto-mem file (read) | `tengu_memdir_accessed` + `tengu_memdir_file_read` |
| auto-mem file (edit) | `tengu_memdir_accessed` + `tengu_memdir_file_edit` |
| auto-mem file (write) | `tengu_memdir_accessed` + `tengu_memdir_file_write` |
| team-mem file (read) | `tengu_team_mem_accessed` + `tengu_team_mem_file_read` |
| team-mem file (edit) | `tengu_team_mem_accessed` + `tengu_team_mem_file_edit` + `notifyTeamMemoryWrite()` |
| team-mem file (write) | `tengu_team_mem_accessed` + `tengu_team_mem_file_write` + `notifyTeamMemoryWrite()` |

所有事件都附带 `subagent_name`（若存在）。

### 5. `MEMORY_SHAPE_TELEMETRY` 分支

```ts
if (feature('MEMORY_SHAPE_TELEMETRY') && filePath) {
  const scope = memoryScopeForPath(filePath)
  if (scope !== null && (tool === FILE_EDIT_TOOL_NAME || tool === FILE_WRITE_TOOL_NAME)) {
    memoryShapeTelemetry!.logMemoryWriteShape(tool, toolInput, filePath, scope)
  }
}
```

### 6. Hook 注册

```ts
export function registerSessionFileAccessHooks(): void {
  const hook: HookCallback = {
    type: 'callback',
    callback: handleSessionFileAccess,
    timeout: 1, // 极短，只是日志
    internal: true,
  }

  registerHookCallbacks({
    PostToolUse: [
      { matcher: FILE_READ_TOOL_NAME, hooks: [hook] },
      { matcher: GREP_TOOL_NAME, hooks: [hook] },
      { matcher: GLOB_TOOL_NAME, hooks: [hook] },
      { matcher: FILE_EDIT_TOOL_NAME, hooks: [hook] },
      { matcher: FILE_WRITE_TOOL_NAME, hooks: [hook] },
    ],
  })
}
```

- `timeout: 1` 表示该 hook 几乎不占用 hook 执行时间预算。
- `internal: true` 标记为内部 hook，不对用户暴露配置。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sessionFileAccessHooks.ts:49-69` | `getFilePathFromInput` 路径提取。 |
| `src/utils/sessionFileAccessHooks.ts:75-116` | `getSessionFileTypeFromInput` 会话文件类型检测。 |
| `src/utils/sessionFileAccessHooks.ts:123-141` | `isMemoryFileAccess` 公共判断函数。 |
| `src/utils/sessionFileAccessHooks.ts:146-227` | `handleSessionFileAccess` 回调与事件发射。 |
| `src/utils/sessionFileAccessHooks.ts:233-250` | `registerSessionFileAccessHooks` 注册入口。 |
| `src/utils/memoryFileDetection.ts` | 被依赖：session memory / memdir / team mem 检测逻辑。 |
| `src/setup.ts:362-363` | 调用方：CLI 启动时注册 hooks。 |
| `src/utils/hooks.ts` | Hook 执行框架。 |
| `src/utils/attribution.ts` | 相关分析模块。 |

---

## 依赖与外部交互

- **内部依赖**：
  - `bun:bundle` 的 `feature` → 条件加载 `TEAMMEM` 和 `MEMORY_SHAPE_TELEMETRY` 模块
  - `../bootstrap/state.js`：`registerHookCallbacks`
  - `../entrypoints/agentSdkTypes.js`：`HookInput`, `HookJSONOutput`
  - `../services/analytics/index.js`：`logEvent`
  - `../tools/FileReadTool/FileReadTool.js` 等：工具 schema 与常量
  - `./agentContext.js`：`getSubagentLogName`
  - `./memoryFileDetection.js`：`detectSessionFileType`, `isAutoMemFile`, `memoryScopeForPath`
- **调用方**：
  - `src/setup.ts`（启动注册）
  - 间接调用方：所有使用 Read/Grep/Glob/Edit/Write 工具的用户操作

---

## 风险、边界与改进建议

### 风险与边界

1. **`feature('TEAMMEM')` 的条件 `require`**：模块顶部使用 `require` 动态加载 `teamMemPaths.js` 和 `watcher.js`。若 `TEAMMEM` feature flag 在运行时切换（如通过 GrowthBook 热更新），模块加载时的静态判断可能过时。不过当前 `feature()` 在 bundled 模式下是编译时常量，在 Node.js 模式下是启动时读取的环境/配置，运行时切换不常见。

2. **`safeParse` 失败即忽略**：`getFilePathFromInput` 和 `getSessionFileTypeFromInput` 在 `safeParse` 失败时返回 `null`。这意味着如果模型输出了非法参数（如 `file_path: 123`），该次访问不会被记录。这在 analytics 层面是数据丢失，但不会造成功能错误。

3. **Grep/Glob 的 pattern 检测局限性**：`detectSessionPatternType` 基于字符串匹配（如是否包含 `session-memory`、`.jsonl` 等），可能被误触发（如用户 grep 一个恰好包含 `session-memory` 字符串的普通目录路径）。虽然这是低概率事件，但 analytics 的 false positive 会影响数据质量。

4. **Hook timeout 极短**：`timeout: 1` 意味着如果 `logEvent` 内部意外阻塞（如 analytics sink 同步 flush），可能导致 hook 被中断。但 `logEvent` 设计为 fire-and-forget 队列写入，实际上不会阻塞。

5. **无幂等性保证**：若同一工具调用因重试或批处理被触发多次 `PostToolUse`，事件会被重复记录。当前没有基于 `toolUseID` 的去重机制。

### 改进建议

1. **增加 `toolUseID` 级别的去重缓存**：使用一个大小受限的 `Set<string>` 记录最近已上报的 `toolUseID`，避免重复事件。考虑到 `PostToolUse` 通常只触发一次，这更多是防御性编程。

2. **将 `safeParse` 失败记录为 `tengu_session_file_access_parse_failed`**：在 `getFilePathFromInput` 返回 `null` 时，可记录一个低优先级的 debug/telemetry 事件，帮助发现模型输出格式异常导致的 analytics 盲区。

3. **Grep/Glob pattern 检测增强**：对于 Grep/Glob 工具，除了字符串匹配外，可尝试将 `glob`/`pattern` 解析为实际路径前缀，再做精确的 `startsWith` 比较，降低 false positive。

4. **将条件 require 改为稳定的类型导入**：若 `TEAMMEM` 和 `MEMORY_SHAPE_TELEMETRY` 逐渐成为稳定功能，可移除条件 `require`，改为常规 ES import，提升类型安全和 tree-shaking 友好性。

5. **增加单元测试**：测试矩阵应覆盖：
   - Read session memory → 触发 `tengu_session_memory_accessed`
   - Edit memdir → 触发 `tengu_memdir_file_edit`
   - Write team mem → 触发 `tengu_team_mem_file_write` + `notifyTeamMemoryWrite`
   - 非法 tool input → 不触发事件
   - `isMemoryFileAccess` 的 true/false 边界
