# FileEditTool.ts 研究文档

## 场景与职责

`FileEditTool.ts` 是 Claude Code 核心文件编辑工具的主实现文件，负责提供基于字符串替换（string-replacement）的文件修改能力。它是模型与用户文件系统之间最关键的可写交互通道之一，名称对外暴露为 `'Edit'`。该工具在 `src/tools.ts` 中被全局注册，并在 `src/utils/api.ts` 的 `normalizeFileEditInput` 等路径中被间接调用。

核心职责包括：
- 接收模型输入（`file_path`、`old_string`、`new_string`、`replace_all`），执行精确字符串替换。
- 在编辑前执行多层校验：文件存在性、大小限制、权限规则、是否已读取（read-before-write）、文件是否被外部修改（timestamp + content 双重校验）、Jupyter Notebook 拦截、Claude 设置文件 schema 校验、团队记忆密钥泄露检查等。
- 执行原子写操作，并同步通知 LSP（语言服务器协议）、VSCode MCP diff 视图、诊断追踪、文件历史备份、技能目录发现等周边系统。
- 输出结构化 diff（`StructuredPatchHunk[]`）供 UI 渲染，同时支持远程 Git diff 附件。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `validateInput` | 在工具调用前进行防御性校验，防止误写、覆盖外部变更、写入非法路径。 |
| `call` | 执行实际编辑：原子读写、生成 patch、写盘、通知周边服务、返回结果。 |
| `readFileForEdit` | 同步读取文件元数据（内容、编码、换行符），为原子替换做准备。 |
| `backfillObservableInput` | 将 `file_path` 展开为绝对路径，防止通过 `~` 或相对路径绕过 hook 白名单。 |
| `checkPermissions` | 委托 `checkWritePermissionForTool` 进行文件系统写权限判定。 |
| `inputsEquivalent` / `mapToolResultToToolResultBlockParam` | 支持工具去重（dedup）和结果序列化到 API。 |

## 具体技术实现

### 1. 工具定义与构建

文件通过 `buildTool({ ... }) satisfies ToolDef<...>` 构建完整 Tool 对象。关键字段：
- `name: FILE_EDIT_TOOL_NAME`（即 `'Edit'`）
- `strict: true` — 启用 API 严格模式。
- `maxResultSizeChars: 100_000`
- `getPath(input)` 返回 `input.file_path`，用于权限系统的路径提取。

### 2. 输入校验流程 (`validateInput`)

校验按顺序执行，任一失败即返回 `{ result: false, behavior: 'ask', ... }`：

1. **路径展开**：`expandPath(file_path)` 统一处理 `~`、相对路径、Windows 路径分隔符。
2. **团队记忆密钥检查**：`checkTeamMemSecrets(fullFilePath, new_string)` — 防止向团队记忆文件写入敏感密钥。
3. **无变化拦截**：`old_string === new_string` 直接拒绝，errorCode=1。
4. **deny 规则检查**：`matchingRuleForInput(..., 'edit', 'deny')` — 若命中权限系统的 deny 规则，直接拒绝，errorCode=2。
5. **UNC 路径安全跳过**：`\\` 或 `//` 开头的路径跳过后续 fs 操作（防止 NTLM 凭证泄露），但返回 `{ result: true }` 让权限层处理。
6. **文件大小限制**：`MAX_EDIT_FILE_SIZE = 1 GiB`，通过 `fs.stat` 检查，超大文件返回 errorCode=10。
7. **文件读取与编码检测**：先读 bytes，通过 BOM (`0xff 0xfe`) 判断 `utf16le`，否则 `utf8`；内容统一将 `\r\n` 替换为 `\n`。
8. **新文件创建**：`fileContent === null && old_string === ''` 允许创建新文件。
9. **已存在文件但 `old_string === ''`**：若文件非空则拒绝（errorCode=3），空文件则允许。
10. **Jupyter Notebook 拦截**：`.ipynb` 文件强制要求使用 `NotebookEditTool`，errorCode=5。
11. **read-before-write 校验**：`readFileState.get(fullFilePath)` 必须存在且非 partial view，errorCode=6。
12. **外部修改检测（timestamp + content fallback）**：
    - 取文件 `mtimeMs`（`Math.floor`）与 `readFileState` 中的 `timestamp` 比较。
    - 若 `lastWriteTime > lastRead.timestamp`，则进入 content fallback：仅当该次读取为完整读取（`offset === undefined && limit === undefined`）且内容一致时，才放行；否则拒绝，errorCode=7。
13. **字符串存在性校验**：`findActualString(file, old_string)` 尝试精确匹配或归一化引号后匹配；未找到则 errorCode=8。
14. **多匹配拦截**：若匹配数 `> 1` 且 `replace_all === false`，拒绝并提示增加上下文或开启 `replace_all`，errorCode=9。
15. **Claude 设置文件 schema 校验**：`validateInputForSettingsFileEdit` 对 `.claude/settings.json` 等文件进行编辑后 schema 校验，防止写入非法配置。

### 3. 执行流程 (`call`)

`call` 接收 `ToolUseContext` 和 `parentMessage`，执行以下步骤：

1. **技能目录发现**：非 simple mode 下，调用 `discoverSkillDirsForPaths` 和 `activateConditionalSkillsForPaths`，动态加载与当前文件路径匹配的技能。
2. **诊断基线捕获**：`diagnosticTracker.beforeFileEdited(absoluteFilePath)` 在编辑前向 IDE MCP 获取诊断基线。
3. **目录创建与文件历史备份**：
   - `fs.mkdir(dirname(...))` 确保父目录存在。
   - `fileHistoryTrackEdit(...)` 若启用则备份原文件内容。
4. **原子读-改-写临界区**：
   - `readFileForEdit` 再次读取文件（同步）。
   - 重复 `mtimeMs` 校验（与 `validateInput` 逻辑一致），失败抛 `FILE_UNEXPECTEDLY_MODIFIED_ERROR`。
   - `findActualString` 确定实际被替换字符串 `actualOldString`。
   - `preserveQuoteStyle` 根据文件实际引号风格调整 `new_string`。
   - `getPatchForEdit` 生成结构化 diff 和更新后内容。
   - `writeTextContent(absoluteFilePath, updatedFile, encoding, endings)` 原子写盘（temp file + rename 回退策略）。
5. **LSP 通知**：
   - `clearDeliveredDiagnosticsForFile` 清除旧诊断。
   - `lspManager.changeFile(...)` 发送 `textDocument/didChange`。
   - `lspManager.saveFile(...)` 发送 `textDocument/didSave`。
   两者均 fire-and-forget（`.catch` 捕获错误）。
6. **VSCode diff 通知**：`notifyVscodeFileUpdated(...)` 向 VSCode MCP 推送原/新内容，用于 IDE 内 diff 视图。
7. **更新 readFileState**：将新内容和最新 `mtimeMs` 写回 `readFileState`，防止后续 stale write。
8. **遥测与统计**：
   - `tengu_write_claudemd`（若文件名为 `CLAUDE.md`）
   - `countLinesChanged(patch)`
   - `logFileOperation({ operation: 'edit', tool: 'FileEditTool', ... })`
   - `tengu_edit_string_lengths`
9. **Git diff 附件（远程模式）**：当 `CLAUDE_CODE_REMOTE` 为真且 GrowthBook flag `tengu_quartz_lantern` 开启时，异步调用 `fetchSingleFileGitDiff` 获取单文件 git diff。
10. **返回结果**：`FileEditOutput` 对象，包含 `filePath`、`oldString`、`newString`、`originalFile`、`structuredPatch`、`userModified`、`replaceAll`、`gitDiff`（可选）。

### 4. 关键数据结构

```ts
// 输入（zod 校验后）
FileEditInput = {
  file_path: string
  old_string: string
  new_string: string
  replace_all?: boolean
}

// 输出
FileEditOutput = {
  filePath: string
  oldString: string
  newString: string
  originalFile: string
  structuredPatch: StructuredPatchHunk[]
  userModified: boolean
  replaceAll: boolean
  gitDiff?: GitDiff
}
```

## 关键代码路径与文件引用

- **工具注册入口**：`src/tools.ts` 导入 `FileEditTool` 并加入全局 `tools` 数组。
- **权限系统**：`src/utils/permissions/filesystem.ts` 中的 `checkWritePermissionForTool`、`matchingRuleForInput`。
- **设置文件校验**：`src/utils/settings/validateEditTool.ts`。
- **团队记忆密钥守卫**：`src/services/teamMemorySync/teamMemSecretGuard.ts`。
- **LSP 管理器**：`src/services/lsp/manager.ts` → `getLspServerManager()`。
- **诊断追踪**：`src/services/diagnosticTracking.ts`。
- **文件读写工具**：`src/utils/file.ts`（`writeTextContent`、`readFileSyncCached`、`getFileModificationTime` 等）。
- **Diff 生成**：`src/utils/diff.ts`（`getPatchFromContents`、`countLinesChanged`）。
- **UI 组件**：`src/components/FileEditToolUpdatedMessage.tsx`、`src/components/FileEditToolUseRejectedMessage.tsx`。
- **引用方**：`src/utils/api.ts`（`normalizeFileEditInput`）、`src/tools/REPLTool/primitiveTools.ts`、`src/components/agents/ToolSelector.tsx` 等。

## 依赖与外部交互

| 依赖模块 | 交互方式 | 说明 |
|----------|----------|------|
| `permissions/filesystem.ts` | 函数调用 | 写权限判定、deny/ask/allow 规则匹配。 |
| `settings/validateEditTool.ts` | 函数调用 | Claude 设置文件编辑后 schema 校验。 |
| `teamMemorySync/teamMemSecretGuard.ts` | 函数调用 | 防止密钥泄露到团队记忆文件。 |
| `lsp/manager.ts` | 单例获取 + 异步通知 | 发送 didChange/didSave 到语言服务器。 |
| `diagnosticTracking.ts` | 单例方法调用 | 编辑前捕获诊断基线。 |
| `mcp/vscodeSdkMcp.ts` | 函数调用 | 通知 VSCode 更新 diff 视图。 |
| `skills/loadSkillsDir.ts` | 异步调用 | 动态发现/激活条件技能。 |
| `utils/file.ts` | 函数调用 | 文件读写、时间戳获取、路径建议。 |
| `utils/diff.ts` | 函数调用 | Patch 生成与行变更统计。 |
| `utils/gitDiff.ts` | 异步调用 | 远程模式下获取 git diff。 |
| `utils/fileHistory.ts` | 函数调用 | 编辑前文件历史备份。 |

## 风险、边界与改进建议

### 风险与边界

1. **TOCTOU（Time-of-check-time-of-use）**：`validateInput` 与 `call` 之间存在时间窗口，文件可能被外部进程修改。当前通过 `call` 内部的二次 `mtimeMs` + content fallback 缓解，但在高并发写场景下仍非绝对安全。
2. **同步文件读写阻塞事件循环**：`readFileForEdit` 和 `writeTextContent` 均使用同步 API（`readFileSyncWithMetadata`、`writeFileSyncAndFlush_DEPRECATED`）。虽然注释强调“避免 async yield 破坏原子性”，但在大文件或慢磁盘上会阻塞主线程。
3. **UNC 路径跳过的语义缝隙**：UNC 路径在 `validateInput` 中跳过 `fs.stat` 和文件读取，直接返回 `{ result: true }`，依赖权限层后续拦截。若权限层存在漏洞，可能产生绕过。
4. **Windows 时间戳抖动**：`mtimeMs` 在 Windows 上可能因云同步、杀毒软件触发无内容变更的更新，导致误报“文件已被修改”。当前已有 content fallback，但仅对完整读取生效；partial read 无 fallback。
5. **LSP 通知 fire-and-forget 错误静默**：`changeFile`/`saveFile` 的 `.catch` 仅记录日志，用户无法感知 LSP 同步失败。
6. **1 GiB 硬上限**：`MAX_EDIT_FILE_SIZE` 为固定值，无法通过配置调整，对特定大文件编辑场景不够灵活。

### 改进建议

1. **引入文件级锁或乐观并发版本号**：在 `readFileState` 中增加内容哈希（如 SHA-256）替代或补充 `mtimeMs` 比较，降低 TOCTOU 和 timestamp 抖动风险。
2. **评估异步原子写的可行性**：研究是否可以在 `call` 的临界区内使用 `fs.promises` 配合显式文件锁（如 `fcntl`/`LockFile`）实现非阻塞原子写。
3. **增强错误码文档化**：当前 errorCode 1~10 为魔法数字，建议在 `constants.ts` 中增加具名常量，并在测试和 UI 中统一引用。
4. **LSP 通知失败的上报**：可将 LSP 同步失败作为 `meta` 或 warning 附加到 `FileEditOutput`，让 UI 有机会提示用户。
5. **GrowthBook flag 驱动的文件大小上限**：将 `MAX_EDIT_FILE_SIZE` 改为可配置（或至少可通过 feature flag 调整），以适配不同用户场景。
