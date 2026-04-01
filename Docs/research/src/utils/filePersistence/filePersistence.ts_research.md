# 研究文档：src/utils/filePersistence/filePersistence.ts

## 场景与职责

`filePersistence.ts` 是 Claude Code 项目中的**文件持久化编排器（File Persistence Orchestrator）**。它的核心职责是在每个 turn（对话轮次）结束时，检测并持久化本地 `outputs` 目录中被修改过的文件。

该模块主要服务于 **BYOC（Bring Your Own Compute）模式**的远程会话：
- 在 BYOC 模式下，用户代码运行在自己的机器上，但需要通过 Anthropic Public Files API 将生成的文件（如图表、报告、下载的内容等）上传回云端，以便在会话恢复或跨设备访问时能够获取这些文件。
- 在 **1P/Cloud 模式**下，文件同步由 `rclone` 等外部工具处理，当前模块仅预留了接口，尚未实现具体的 xattr 读取逻辑。

调用入口位于 `src/cli/print.ts` 的主循环中，每次 `ask()` 完成一轮对话后，如果启用了 `FILE_PERSISTENCE` 特性开关，就会触发 `executeFilePersistence()`。

---

## 功能点目的

### 1. `runFilePersistence(turnStartTime, signal?)`
**主入口函数**，负责组装配置、协调扫描与上传流程，并返回持久化结果。

关键前置检查：
- 环境类型必须是 `byoc`（通过 `getEnvironmentKind()`）
- 必须能获取到 session access token（通过 `getSessionIngressAuthToken()`）
- 必须设置环境变量 `CLAUDE_CODE_REMOTE_SESSION_ID`
- 支持 `AbortSignal` 取消

### 2. `executeBYOCPersistence(turnStartTime, config, outputsDir, signal?)`
**BYOC 模式的具体实现**：
- 调用 `findModifiedFiles()` 扫描 `{cwd}/{sessionId}/outputs` 目录
- 如果修改文件数超过 `FILE_COUNT_LIMIT`（1000），则拒绝上传并记录超限事件
- 对文件路径进行安全过滤（跳过解析到 `outputs` 目录外部的文件，防止路径遍历）
- 调用 `uploadSessionFiles()` 并行上传文件
- 分离成功/失败结果，构造 `FilesPersistedEventData`

### 3. `executeCloudPersistence()`
**Cloud/1P 模式的占位实现**。当前仅记录调试日志说明 "xattr-based file ID reading not yet implemented"，直接返回空结果。

### 4. `executeFilePersistence(turnStartTime, signal, onResult)`
**带回调的包装器**，供 `print.ts` 调用。内部捕获异常，避免文件持久化失败影响主流程。成功时通过 `onResult` 回调将 `files_persisted` 系统事件注入消息流。

### 5. `isFilePersistenceEnabled()`
**特性开关检查**。要求同时满足：
- `feature('FILE_PERSISTENCE')` 为 true
- 环境类型为 `byoc`
- 有有效的 session access token
- `CLAUDE_CODE_REMOTE_SESSION_ID` 已设置

这确保了只有 public-api/sessions 用户（如 CCR 远程会话）才会触发文件持久化，普通的 Claude Code CLI 用户不会受影响。

---

## 具体技术实现

### 关键流程

```
print.ts (turn end)
  └─> executeFilePersistence(turnStartTime, signal, onResult)
        └─> runFilePersistence(turnStartTime, signal)
              ├─> 检查 environmentKind === 'byoc'
              ├─> 获取 sessionAccessToken
              ├─> 读取 CLAUDE_CODE_REMOTE_SESSION_ID
              ├─> 构造 outputsDir = join(getCwd(), sessionId, OUTPUTS_SUBDIR)
              ├─> 检查 signal.aborted
              ├─> 记录 analytics: tengu_file_persistence_started
              └─> executeBYOCPersistence(...)
                    ├─> findModifiedFiles(turnStartTime, outputsDir)
                    ├─> 检查 FILE_COUNT_LIMIT (1000)
                    ├─> 路径安全过滤 (relativePath.startsWith('..'))
                    ├─> uploadSessionFiles(filesToProcess, config, DEFAULT_UPLOAD_CONCURRENCY)
                    └─> 分离 success/failure，构造 FilesPersistedEventData
              └─> 记录 analytics: tengu_file_persistence_completed
```

### 数据结构

来自 `./types.js`（根据代码引用及项目上下文重建）：

```typescript
export type TurnStartTime = number

export type PersistedFile = {
  filename: string   // 相对路径（相对于 outputs 目录）
  file_id: string    // Files API 返回的文件 ID
}

export type FailedPersistence = {
  filename: string
  error: string
}

export type FilesPersistedEventData = {
  files: PersistedFile[]
  failed: FailedPersistence[]
}

export const OUTPUTS_SUBDIR = 'outputs'
export const FILE_COUNT_LIMIT = 1000
export const DEFAULT_UPLOAD_CONCURRENCY = 5
```

### 协议与命令

- **Files API 上传**：通过 `src/services/api/filesApi.ts` 中的 `uploadSessionFiles()` 实现，使用 Anthropic Public Files API 的 `POST /v1/files` 端点，采用 Bearer OAuth 认证，multipart/form-data 格式上传。
- **Analytics 事件**：
  - `tengu_file_persistence_started`：持久化开始
  - `tengu_file_persistence_completed`：持久化完成（含 success_count, failure_count, duration_ms, mode, error）
  - `tengu_file_persistence_limit_exceeded`：文件数超限

### 安全设计

- **路径遍历防护**：使用 `relative(outputsDir, filePath)` 计算相对路径，如果结果以 `..` 开头则跳过该文件。
- **符号链接跳过**：在 `outputsScanner.ts` 的 `findModifiedFiles()` 中，通过 `entry.isSymbolicLink()` 和 `stat.isSymbolicLink()` 双重检查跳过符号链接，防止读取敏感文件。

---

## 关键代码路径与文件引用

### 本文件内部

| 函数/常量 | 行号 | 说明 |
|-----------|------|------|
| `runFilePersistence` | 51 | 主入口 |
| `executeBYOCPersistence` | 150 | BYOC 上传逻辑 |
| `executeCloudPersistence` | 247 | Cloud 占位 |
| `executeFilePersistence` | 256 | 回调包装器 |
| `isFilePersistenceEnabled` | 278 | 开关检查 |

### 上游调用方

| 文件 | 行号 | 调用方式 |
|------|------|----------|
| `src/cli/print.ts` | 2256 | `void executeFilePersistence(turnStartTime, abortController.signal, result => output.enqueue({ type: 'system', subtype: 'files_persisted', ... }))` |

`print.ts` 中 `turnStartTime` 的生成逻辑（行 2134）：
```typescript
const turnStartTime = feature('FILE_PERSISTENCE') ? Date.now() : undefined
```
在每次进入 `ask()` 前记录当前时间戳，作为后续判断文件是否在本轮被修改的基准。

### 下游依赖方

| 文件 | 导入内容 | 作用 |
|------|----------|------|
| `src/services/api/filesApi.ts` | `uploadSessionFiles`, `FilesApiConfig` | 实际执行 Files API 上传 |
| `src/utils/filePersistence/outputsScanner.ts` | `findModifiedFiles`, `getEnvironmentKind`, `logDebug` | 扫描修改文件、环境检测、调试日志 |
| `src/utils/sessionIngressAuth.ts` | `getSessionIngressAuthToken` | 获取会话 OAuth token |
| `src/utils/cwd.ts` | `getCwd` | 获取当前工作目录 |
| `src/utils/filePersistence/types.js` | 常量与类型 | `OUTPUTS_SUBDIR`, `FILE_COUNT_LIMIT`, `DEFAULT_UPLOAD_CONCURRENCY`, `TurnStartTime`, `FilesPersistedEventData`, `PersistedFile`, `FailedPersistence` |
| `src/services/analytics/index.js` | `logEvent` | 埋点上报 |
| `src/utils/log.js` | `logError` | 错误日志 |
| `src/utils/errors.js` | `errorMessage` | 错误信息提取 |
| `src/utils/debug.js` | `logForDebugging` | 调试日志（间接通过 outputsScanner） |

### SDK Schema 定义

`src/entrypoints/sdk/coreSchemas.ts`（行 1672）定义了 `files_persisted` 事件的 Zod Schema：

```typescript
export const SDKFilesPersistedEventSchema = lazySchema(() =>
  z.object({
    type: z.literal('system'),
    subtype: z.literal('files_persisted'),
    files: z.array(z.object({ filename: z.string(), file_id: z.string() })),
    failed: z.array(z.object({ filename: z.string(), error: z.string() })),
    processed_at: z.string(),
    uuid: UUIDPlaceholder(),
    session_id: z.string(),
  }),
)
```

---

## 依赖与外部交互

### 环境变量依赖

| 环境变量 | 用途 | 来源/设置方 |
|----------|------|-------------|
| `CLAUDE_CODE_ENVIRONMENT_KIND` | 判断运行环境（`byoc` / `anthropic_cloud`） | 远程会话启动时注入 |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | 构造 outputs 目录路径、Files API 调用 | 远程会话启动时注入 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | OAuth Bearer token（优先） | 远程会话启动时注入，或由 REPL bridge 动态更新 |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | 遗留的 token 读取方式（文件描述符） | 远程会话启动时注入 |
| `CLAUDE_SESSION_INGRESS_TOKEN_FILE` | token 备用文件路径 | 可选 |

### 外部 API 交互

通过 `src/services/api/filesApi.ts` 与 Anthropic Public Files API 交互：
- **端点**：`POST /v1/files`
- **认证**：`Authorization: Bearer {oauthToken}`
- **Beta Header**：`files-api-2025-04-14,oauth-2025-04-20`
- **上传方式**：multipart/form-data，文件 purpose 为 `user_data`
- **大小限制**：单文件最大 500MB（`MAX_FILE_SIZE_BYTES = 500 * 1024 * 1024`）

---

## 风险、边界与改进建议

### 已知风险

1. **Cloud 模式完全未实现**
   `executeCloudPersistence()` 是一个空壳。如果未来在 `anthropic_cloud` 环境下启用文件持久化，必须补充基于 xattr 的 file_id 读取逻辑，否则文件变更无法被追踪。

2. **文件数量硬上限**
   `FILE_COUNT_LIMIT = 1000` 是硬性限制。如果用户在一次 turn 中生成了超过 1000 个文件，所有文件都会被拒绝上传，且错误信息只包含目录路径和数量，不便于用户定位问题。

3. **turnStartTime 的时序边界**
   `turnStartTime` 在 `ask()` 开始前记录。如果文件在 turn 进行期间被修改（这是正常场景），会被正确捕获。但如果 `findModifiedFiles()` 执行期间又有新文件写入，这些文件也会被包含，因为扫描和过滤不是原子操作。

4. **大文件上传失败成本高**
   文件大小检查在 `filesApi.ts` 的 `uploadFile()` 中才进行（读取完整文件内容到内存后）。对于超过 500MB 的文件，已经消耗了 I/O 和内存才返回错误。编排层没有预检查。

5. **路径遍历过滤的局限性**
   使用 `relativePath.startsWith('..')` 过滤路径遍历，对于某些复杂的符号链接或规范化前的路径可能不够严谨。虽然 `outputsScanner.ts` 已经跳过了符号链接，但仍需注意不同操作系统下的路径行为差异。

6. **没有子目录结构保留的显式协议**
   `relativePath` 保留了相对于 `outputs` 的子目录结构，这在 `uploadFile()` 中作为 `filename` 传入 API。如果服务端对 `filename` 中的路径分隔符处理不一致，可能导致目录结构丢失。

### 改进建议

1. **增加文件大小预检查**：在 `executeBYOCPersistence()` 中，对 `modifiedFiles` 进行批量 `stat` 获取大小，提前过滤掉超过 500MB 的文件，避免无效的内存分配和上传尝试。

2. **细化超限错误信息**：当 `FILE_COUNT_LIMIT` 被触发时，除了返回总数，还可以列出前 N 个文件的名称，帮助用户快速定位问题。

3. **实现 Cloud 模式**：补充 `executeCloudPersistence()` 的实现，读取本地文件的 xattr（如 `user.anthropic.file_id`）来识别已同步文件的 file_id。

4. **增加上传进度/流式反馈**：当前上传是批量并行，没有中间进度。对于大文件或慢网络，可以考虑在上传层支持进度回调，并在 `filePersistence.ts` 中透传给上层。

5. **考虑增量/差异持久化**：当前每次 turn 都扫描整个 `outputs` 目录。对于大型项目，可以维护一个本地索引（如 SQLite 或 JSON 文件），记录上次持久化的文件哈希或 mtime，减少全量扫描开销。

6. **增强测试覆盖**：当前代码库中未找到针对 `filePersistence.ts` 或 `outputsScanner.ts` 的单元测试。建议补充测试，覆盖以下场景：
   - 正常上传流程（mock Files API）
   - `FILE_COUNT_LIMIT` 超限
   - 路径遍历攻击防护
   - `AbortSignal` 取消
   - 符号链接跳过
   - `turnStartTime` 边界条件
