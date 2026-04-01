# useDiffInIDE.ts 研究文档

## 场景与职责

`useDiffInIDE` 是 Claude Code 中负责**将文件编辑差异（diff）投射到连接的 IDE 编辑器**中的核心 Hook。它主要服务于 `FilePermissionDialog` 组件，在用户需要确认文件编辑（如 `FileEditTool`）时，如果检测到已连接支持的 IDE（VS Code 及其衍生版、JetBrains 系列），则不再在终端内展示 diff，而是直接在 IDE 中打开一个 diff 标签页，让用户在熟悉的编辑器环境中审阅、修改并决定接受或拒绝变更。

该 Hook 的引入是为了提升用户体验：对于在 IDE 内置终端中运行 Claude Code 的用户，IDE 的 diff 视图比终端文本 diff 更直观、可交互（可直接在 diff 中修改）。

## 功能点目的

1. **条件性启用 IDE Diff**：仅在同时满足以下条件时启用：
   - MCP clients 中存在已连接的 IDE client（`hasAccessToIDEExtensionDiffFeature`）
   - 全局配置 `diffTool === 'auto'`（`src/utils/config.ts` 中 `GlobalConfig.diffTool`）
   - 不是 `.ipynb` 文件（Notebook 编辑不支持 IDE diff）

2. **在 IDE 中打开 Diff 标签页**：通过 MCP RPC `openDiff` 向 IDE 发送旧内容、新内容和自定义标签名，IDE 扩展负责渲染 diff 视图。

3. **监听用户操作并回传结果**：
   - 用户保存文件 → 视为接受，读取 IDE 回传的新内容，重新计算 edits
   - 用户关闭标签页 → 视为接受（但内容未变）
   - 用户点击拒绝 → 视为拒绝，恢复旧内容

4. **清理与关闭标签页**：在组件卸载或用户拒绝时，通过 RPC `close_tab` 关闭 IDE 中的 diff 标签页，避免残留。

5. **WSL 路径转换**：当 Claude Code 运行在 WSL 而 IDE 运行在 Windows 时，自动进行 WSL↔Windows 路径转换（`WindowsToWSLConverter`）。

## 具体技术实现

### 关键流程

#### 1. 初始化与条件判断（`useDiffInIDE` 主 Hook）
```ts
const shouldShowDiffInIDE =
  hasAccessToIDEExtensionDiffFeature(toolUseContext.options.mcpClients) &&
  getGlobalConfig().diffTool === 'auto' &&
  !filePath.endsWith('.ipynb')
```
- `hasAccessToIDEExtensionDiffFeature`（`src/utils/ide.ts`）遍历 `mcpClients`，检查是否存在 `type === 'connected' && name === 'ide'` 的 client。

#### 2. 打开 Diff（`showDiffInIDE` 异步函数）
- 读取本地文件旧内容（`readFileSync`）
- 通过 `getPatchForEdits`（`src/tools/FileEditTool/utils.ts`）将 edits 应用到旧内容，得到 `updatedFile`
- 获取 `ideClient`（`getConnectedIdeClient`），若不存在或状态不对则抛错
- **WSL 路径转换**：若平台为 WSL、IDE 运行在 Windows、且 `WSL_DISTRO_NAME` 存在，使用 `WindowsToWSLConverter.toIDEPath(oldFilePath)` 转换路径
- 调用 `callIdeRpc('openDiff', { old_file_path, new_file_path, new_file_contents, tab_name }, ideClient)`
- 注册清理监听器：`abortController.signal`（用户按 Esc 取消）和 `process.on('beforeExit')`

#### 3. 结果解析
RPC 返回的数据结构通过三个类型守卫函数解析：
- `isSaveMessage(data)`：`[{ text: 'FILE_SAVED' }, { text: string }]` → 返回新内容
- `isClosedMessage(data)`：`[{ text: 'TAB_CLOSED' }]` → 返回 `updatedFile`
- `isRejectedMessage(data)`：`[{ text: 'DIFF_REJECTED' }]` → 返回 `oldContent`

#### 4. Edits 重计算（`computeEditsFromContents`）
当用户保存并在 IDE diff 中做了额外修改时，需要基于返回的新旧内容重新计算 edits：
- 调用 `getPatchFromContents`（`src/utils/diff.ts`）生成结构化 patch（`StructuredPatchHunk[]`）
- 调用 `getEditsForPatch`（`src/tools/FileEditTool/utils.ts`）将 patch 转回 `FileEdit[]`
- 支持 `singleHunk` 模式校验（若期望单 hunk 但得到多个则报错）

#### 5. 关闭标签页（`closeTabInIDE`）
通过 `callIdeRpc('close_tab', { tab_name }, ideClient)` 异步关闭。错误被吞掉（`logError`），因为这是清理操作。

### 数据结构

- **Props**：
  - `onChange(option: PermissionOption, input: { file_path: string; edits: FileEdit[] })`：将用户决策和更新后的 edits 回传给父组件
  - `toolUseContext: ToolUseContext`：包含 `abortController` 和 `mcpClients`
  - `filePath: string`：被编辑文件路径
  - `edits: FileEdit[]`：原始编辑列表
  - `editMode: 'single' | 'multiple'`：影响 patch 生成方式

- **返回值**：
  - `closeTabInIDE: () => void`：手动关闭标签页
  - `showingDiffInIDE: boolean`：是否正在 IDE 中展示 diff
  - `ideName: string`：连接的 IDE 名称（如 "VS Code"）
  - `hasError: boolean`：是否发生错误（如 RPC 失败）

### 标签名生成
```ts
const tabName = `✻ [Claude Code] ${basename(filePath)} (${sha}) ⧉`
```
使用 `randomUUID().slice(0, 6)` 生成唯一后缀，避免同名文件冲突。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useDiffInIDE.ts` | 本 Hook 实现 |
| `src/components/permissions/FilePermissionDialog/FilePermissionDialog.tsx` | 主要调用方，在文件编辑权限对话框中使用 |
| `src/utils/ide.ts` | `hasAccessToIDEExtensionDiffFeature`、`getConnectedIdeClient`、`callIdeRpc`、`initializeIdeIntegration` |
| `src/utils/idePathConversion.ts` | `WindowsToWSLConverter` 路径转换 |
| `src/tools/FileEditTool/utils.ts` | `getPatchForEdits`、`getEditsForPatch`、quote normalization 等 |
| `src/utils/diff.ts` | `getPatchFromContents`、结构化 diff 生成 |
| `src/utils/config.ts` | `getGlobalConfig`、全局配置读取（含 `diffTool`） |
| `src/services/mcp/client.ts` | `callIdeRpc` 实际 RPC 调用实现 |
| `src/services/mcp/types.ts` | `MCPServerConnection`、`ConnectedMCPServer`、`McpSSEIDEServerConfig` 等类型 |

## 依赖与外部交互

### 内部依赖
- **React**：`useEffect`、`useMemo`、`useRef`、`useState`
- **Node 内置**：`crypto.randomUUID`、`path.basename`
- **文件系统**：`src/utils/fileRead.js` 的 `readFileSync`

### 外部交互
- **IDE MCP Client**：通过 `callIdeRpc` 与 IDE 扩展通信，使用的 RPC 方法：
  - `openDiff`：打开 diff 视图
  - `close_tab`：关闭指定标签页
- **IDE 扩展协议**：期望返回特定格式的消息数组（`FILE_SAVED`、`TAB_CLOSED`、`DIFF_REJECTED`）

### 配置依赖
- `GlobalConfig.diffTool`：必须为 `'auto'` 才启用
- `process.env.WSL_DISTRO_NAME`：WSL 路径转换所需

## 风险、边界与改进建议

### 风险与边界

1. **RPC 协议隐式耦合**：`isSaveMessage` / `isClosedMessage` / `isRejectedMessage` 的类型守卫硬编码了 IDE 扩展返回的消息格式。如果 IDE 扩展协议变更，这里会静默失败或抛错（`throw new Error('Not accepted')`）。

2. **WSL 路径转换的脆弱性**：依赖 `wslpath` 命令（`src/utils/idePathConversion.ts` 中通过 `execFileSync` 调用）。如果 `wslpath` 不可用或跨 distro 路径不匹配，会 fallback 到原始路径，可能导致 IDE 找不到文件。

3. **单 hunk 校验可能误报**：`computeEditsFromContents` 中 `singleHunk && patch.length > 1` 仅 `logError` 而不阻止流程，但下游消费方可能假设只有一个 hunk。

4. **竞态条件**：`showDiffInIDE` 内部使用 `isCleanedUp` 标志防止重复清理，但 `cleanup` 被同时注册到 `abortController.signal` 和 `process.beforeExit`，存在多入口触发可能。当前实现通过 `if (isCleanedUp) return` 做了防护。

5. **无超时机制**：代码注释中明确写了 `TODO: Time out after 5 mins of inactivity?`。如果用户长时间不操作 diff 标签页，Hook 会永远挂起等待。

6. **Notebook 文件被硬排除**：`.ipynb` 被写死在代码中，未来若支持 Notebook diff 需要修改此处。

### 改进建议

1. **协议版本化**：建议 IDE 扩展 RPC 返回增加 `version` 或 `method` 字段，避免纯文本匹配带来的协议脆弱性。

2. **增加 diff 会话超时**：在 `showDiffInIDE` 中加入 `setTimeout`，例如 5 分钟无响应后自动 `cleanup()` 并 reject，防止资源泄漏。

3. **统一路径转换层**：当前 WSL 转换逻辑散落在 `useDiffInIDE.ts` 和 `src/utils/ide.ts`（`detectIDEs`）中，建议抽象为统一的 `toIDERpcPath` / `fromIDERpcPath` 工具函数。

4. **错误恢复增强**：当 `showDiffInIDE` 抛错时，目前仅设置 `hasError = true` 并 fallback 到终端 diff。可考虑增加重试逻辑或更详细的错误分类（网络错误 vs 协议错误 vs 文件不存在）。

5. **类型安全提升**：`callIdeRpc` 返回类型为 `unknown`，建议为 `openDiff` 定义严格的返回类型契约，替代当前的运行时类型守卫。
