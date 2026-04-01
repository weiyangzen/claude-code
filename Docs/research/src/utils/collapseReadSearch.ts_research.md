# collapseReadSearch.ts 研究文档

> 文件路径：`src/utils/collapseReadSearch.ts`  
> 行数：1109 行  
> 研究日期：2026-04-01

---

## 场景与职责

`collapseReadSearch.ts` 是 Claude Code 消息渲染管道的核心工具之一，负责将对话流中**连续的搜索/读取类工具调用**折叠成紧凑的摘要组（`CollapsedReadSearchGroup`）。在 REPL/TUI 中，Claude 可能一次性发出数十个 `Read`、`Grep`、`Glob`、`Bash` 等工具调用；若每个都单独渲染，消息列表会极度冗长。该模块通过“折叠”机制，把同类操作聚合成类似 **"Read 5 files, searched for 3 patterns"** 的摘要，并在 `Ctrl+O` 详细模式下保留原始消息的可展开视图。

**主要使用场景：**
- REPL 消息列表渲染（`Messages.tsx`、`MessageRow.tsx`）
- 子代理（LocalAgentTask）进度摘要
- 流式输出（streamlined）中的活动总结
- 转录搜索（transcript search）中识别被折叠的消息组

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `collapseReadSearchGroups` | 主入口：遍历 `RenderableMessage[]`，把连续的搜索/读取工具 use 及其结果折叠成 `collapsed_read_search` 消息组 |
| `getToolSearchOrReadInfo` | 判断某个工具调用是否属于可折叠的搜索/读取/列表/REPL/MCP/内存操作 |
| `getSearchReadSummaryText` | 根据计数生成人类可读的摘要文本（时态支持：进行中 vs 已完成） |
| `summarizeRecentActivities` | 为 Agent UI 和 REPL 提供“最近活动”单行摘要 |
| `getToolUseIdsFromCollapsedGroup` / `hasAnyToolInProgress` | 供外部组件查询折叠组内包含哪些 tool_use_id，以及是否有工具仍在执行 |
| `getDisplayMessageFromCollapsed` | 获取折叠组用于展示时间戳/模型的底层消息 |

**折叠规则核心：**
1. **什么能折叠**：`Read`、`Grep`、`Glob`、`Bash` 中的搜索/读取命令、`ls/tree/du` 列表命令、REPL 调用、MCP 查询、内存文件读写等。
2. **什么会打断组**：助理文本消息、非折叠类工具调用（如 `Write`、`Edit`）、用户非相关 tool_result、系统消息（除 `PreToolUse` hook summary 外）。
3. **可跳过但不打断组**：思考块（thinking/redacted_thinking）、附件消息（除 `nested_memory` 外）、系统消息——它们会被延迟到折叠组之后输出，保持视觉顺序。

---

## 具体技术实现

### 3.1 关键数据结构

#### `SearchOrReadResult`
```ts
export type SearchOrReadResult = {
  isCollapsible: boolean
  isSearch: boolean
  isRead: boolean
  isList: boolean
  isREPL: boolean
  isMemoryWrite: boolean
  isAbsorbedSilently: boolean
  mcpServerName?: string
  isBash?: boolean
}
```
每个工具调用都会经过 `getToolSearchOrReadInfo` 分类，得到上述标签。标签决定该消息在折叠组中的计数方式。

#### `GroupAccumulator`（内部折叠状态）
折叠过程中维护的累加器，包含：
- `messages: CollapsibleMessage[]` — 组内原始消息
- `searchCount / readFilePaths / readOperationCount / listCount` — 各类操作计数
- `memorySearchCount / memoryReadFilePaths / memoryWriteCount` — 个人内存操作计数
- `teamMemorySearchCount / teamMemoryReadFilePaths / teamMemoryWriteCount` — 团队内存操作计数（`TEAMMEM` feature gate）
- `mcpCallCount / mcpServerNames` — MCP 调用计数
- `bashCount / bashCommands` — 非搜索/读取的 Bash 命令计数（fullscreen 模式）
- `commits / pushes / branches / prs` — 从 Bash 结果中扫描出的 Git 操作元数据
- `hookTotalMs / hookCount / hookInfos` — 吸收的 `PreToolUse` hook 耗时信息
- `relevantMemories` — 自动注入的 relevant_memories 附件

#### `CollapsedReadSearchGroup`（输出结构）
由 `createCollapsedGroup` 生成，关键字段：
- `searchCount / readCount / listCount / replCount` — 普通操作数量
- `memorySearchCount / memoryReadCount / memoryWriteCount` — 内存操作数量
- `readFilePaths` — 去重后的非内存读取文件路径列表
- `searchArgs` — 非内存搜索的 pattern 列表
- `latestDisplayHint` — 最近一条操作的提示文本（用于 ⤿ 行展示）
- `messages` — 组内全部原始消息（verbose 模式迭代用）
- `displayMessage` — 用于提取时间戳/模型的首条消息

### 3.2 关键流程

#### 主折叠流程 `collapseReadSearchGroups(messages, tools)`
```
初始化 result = [], currentGroup = 空, deferredSkippable = []
遍历每条消息 msg:
  ├─ 若是可折叠 tool_use:
  │   根据 getCollapsibleToolInfo 分类
  │   ├─ memoryWrite → 计入 memoryWriteCount 或 teamMemoryWriteCount
  │   ├─ absorbedSilently (Snip/ToolSearch/REPL) → 不计数，仅入组
  │   ├─ mcpServerName → mcpCallCount++
  │   ├─ isBash (fullscreen 非搜索 bash) → bashCount++，记录 command hint
  │   ├─ isList → listCount++
  │   ├─ isSearch → searchCount++，区分 memory/teamMemory/regular
  │   └─ 否则视为 read → 收集 file_path，去重；无 path 则 readOperationCount++
  │   收集 tool_use_id，msg 入组
  ├─ 若是可折叠 tool_result (tool_use_id 匹配当前组):
  │   msg 入组；fullscreen 模式下扫描 bash 结果中的 git SHA/PR URL
  ├─ 若是 PreToolUse hook summary 且组非空:
  │   吸收 hook 耗时和计数到 currentGroup
  ├─ 若是 relevant_memories attachment 且组非空:
  │   入 relevantMemories（不污染 readFilePaths，避免破坏 bash-only fallback）
  ├─ 若是 shouldSkipMessage (thinking/attachment/system):
  │   组非空时延迟输出（deferredSkippable），否则直接入 result
  ├─ 若是文本消息:
  │   flushGroup()，然后入 result
  ├─ 若是非折叠 tool_use:
  │   flushGroup()，然后入 result
  └─ 否则:
       flushGroup()，然后入 result
flushGroup() 收尾
```

#### Git 操作扫描 `scanBashResultForGitOps`
在 fullscreen 模式下，非搜索/读取的 Bash 命令及其结果会被保留。当收到 Bash 的 tool_result 时，调用 `detectGitOperation(command, combinedOutput)`（来自 `src/tools/shared/gitOperationTracking.ts`）解析出 commit SHA、push branch、merge/rebase ref、PR number/URL，并累加到组的 `commits`/`pushes`/`branches`/`prs` 数组中。这些元数据最终在 `CollapsedReadSearchContent.tsx` 中以 "Committed abc123, created PR #42" 的形式展示。

#### 摘要文本生成 `getSearchReadSummaryText`
按固定优先级拼接动词短语：
1. Memory 操作（Recalling / Searching / Writing memories）
2. Team Memory 操作（`TEAMMEM` 下由 `teamMemoryOps.appendTeamMemorySummaryParts` 拼接）
3. Search（Searching for N patterns）
4. Read（Reading N files）
5. List（Listing N directories）
6. REPL（REPL'ing N times）
时态根据 `isActive` 切换：进行时用现在分词，完成时用过去式。

### 3.3 特殊处理与 feature gate

- **`feature('TEAMMEM')`**：团队内存相关判断和计数通过 `require('./teamMemoryOps.js')` 动态加载，避免外部构建打包内部代码。
- **`feature('HISTORY_SNIP')`**：`SnipTool` 被识别为 `isAbsorbedSilently`，不打破折叠组。
- **`isFullscreenEnvEnabled()`**：开启后，非搜索/读取的 Bash 命令也视为可折叠（`isBash`），并启用 `ToolSearch` 的静默吸收；同时启用 Git 操作扫描。
- **REPL 工具**：REPL 调用本身被静默吸收（`isAbsorbedSilently`），其内部产生的虚拟消息（`Read`、`Grep`、`Bash`）会作为独立消息再次流经本函数，从而被正常折叠。

---

## 关键代码路径与文件引用

### 直接依赖（被调用方）
| 文件 | 用途 |
|------|------|
| `src/Tool.js` | `findToolByName`, `Tools` 类型 |
| `src/tools/BashTool/commentLabel.ts` | `extractBashCommentLabel` — 提取 Bash 命令中的注释标签 |
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` |
| `src/tools/FileEditTool/constants.js` | `FILE_EDIT_TOOL_NAME` |
| `src/tools/FileWriteTool/prompt.js` | `FILE_WRITE_TOOL_NAME` |
| `src/tools/REPLTool/constants.js` | `REPL_TOOL_NAME` |
| `src/tools/REPLTool/primitiveTools.js` | `getReplPrimitiveTools` — REPL 内嵌原始工具 fallback |
| `src/tools/shared/gitOperationTracking.js` | `detectGitOperation` — 扫描 git commit/push/PR |
| `src/tools/ToolSearchTool/prompt.js` | `TOOL_SEARCH_TOOL_NAME` |
| `src/utils/file.js` | `getDisplayPath` — 缩短路径用于展示 |
| `src/utils/fullscreen.js` | `isFullscreenEnvEnabled` |
| `src/utils/memoryFileDetection.js` | `isAutoManagedMemoryFile`, `isMemoryDirectory`, `isShellCommandTargetingMemory` |
| `src/utils/teamMemoryOps.js` (动态 require) | `isTeamMemorySearch`, `isTeamMemoryWriteOrEdit`, `appendTeamMemorySummaryParts` |

### 调用方（上游入口）
| 文件 | 调用点 |
|------|--------|
| `src/utils/streamlinedTransform.ts` | 引用 `getSearchReadSummaryText` 生成流式摘要 |
| `src/components/Messages.tsx` | 在消息列表渲染前调用 `collapseReadSearchGroups` |
| `src/components/MessageRow.tsx` | 渲染折叠消息行 |
| `src/components/messages/CollapsedReadSearchContent.tsx` | 渲染折叠组详细内容，调用 `getToolUseIdsFromCollapsedGroup` |
| `src/components/messages/AttachmentMessage.tsx` | 处理被吸收的 attachment |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 子代理进度展示 |
| `src/components/Spinner/TeammateSpinnerLine.tsx` | 队友旋转提示 |
| `src/components/TaskListV2.tsx` | 任务列表 |
| `src/utils/transcriptSearch.ts` | 转录搜索中识别折叠组 |

---

## 依赖与外部交互

### 运行时依赖
- **Bun bundle feature gate** (`bun:bundle`)：用于 `feature('TEAMMEM')`、`feature('HISTORY_SNIP')`、`feature('UDS_INBOX')` 等条件编译。
- **Node `crypto` UUID**：生成折叠组 UUID（`collapsed-${firstMsg.uuid}`）。
- **工具元数据**：依赖各工具的 `isSearchOrReadCommand` 方法（如 `BashTool`、`ReadTool`、`GrepTool` 等）进行精确分类。

### 与 UI 的交互契约
- 输出 `CollapsedReadSearchGroup` 类型消息，下游 `CollapsedReadSearchContent.tsx` 负责将其渲染为带 ⤿ 提示的折叠卡片。
- `verbose` 模式下，UI 会遍历 `group.messages` 并逐个渲染原始 `VerboseToolUse`。
- `hasAnyToolInProgress` 被 UI 用来决定是否显示加载动画。

---

## 风险、边界与改进建议

### 已知风险与边界

1. **路径大小写与分隔符（Windows）**
   - `memoryFileDetection.ts` 中已做 `toComparable` 处理（转 `/`、Windows 转小写），但 `collapseReadSearch.ts` 自身对 `file_path` 是原样存入 `Set`。如果同一文件通过不同大小写被读取，Windows 下会重复计数。建议在读文件路径去重时也经过 `normalizeFilePath` 或 `toComparable`。

2. **Bash-only read 的回退计数**
   - 当读取操作没有 `file_path`（如 `Bash: cat file.txt`）时，使用 `readOperationCount` 作为回退。但如果组内同时存在带 path 的 Read 和无 path 的 Bash read，`totalReadCount` 只会取 `readFilePaths.size`，导致 Bash read 被忽略。这是设计上的有意取舍，但可能让用户觉得“少读了一个文件”。

3. **relevantMemories 的路径隔离**
   - `relevantMemories` 被刻意**不**加入 `readFilePaths`/`memoryReadFilePaths`，以防止破坏 `readOperationCount` 回退。这导致 `readFilePaths` 中看不到这些内存路径，但 `memoryReadCount` 会额外加上 `relevantMemories.length`。逻辑正确但分散在两个地方，维护成本高。

4. **MCP 工具 fallback**
   - MCP 工具通过 `tool.isMcp` 和 `tool.mcpInfo?.serverName` 识别。如果 MCP 工具没有正确实现 `isSearchOrReadCommand`，会被误判为不可折叠，从而打断组。

5. **动态 require 的 tree-shaking 边界**
   - `teamMemoryOps.js` 和 `SnipTool/prompt.js` 使用动态 `require` 包裹在 `feature()` 后面。如果 bundler 的 DCE 不够精确，可能把内部模块残留进外部构建。

### 改进建议

1. **统一路径归一化**
   在 `getFilePathsFromReadMessage` 返回路径前调用 `normalizeFilePath` 或至少 `toPosix`，确保跨平台去重一致。

2. **提取计数逻辑**
   `collapseReadSearchGroups` 主循环中的 `if/else` 计数分支已非常长（~80 行）。可考虑把各分类的计数逻辑拆成策略函数表（`Record<CollapsibleKind, (msg, group) => void>`），降低圈复杂度。

3. **类型安全**
   `CollapsedReadSearchGroup` 目前只在 `createCollapsedGroup` 中隐式构造，没有独立的 TS interface/type 定义文件。建议在 `src/types/message.ts`（或等效位置）显式导出，方便跨模块引用和编译器检查。

4. **测试覆盖**
   该文件逻辑分支极多（REPL、MCP、fullscreen、team memory、relevant memories、hook summaries 等），建议增加单元测试覆盖以下场景：
   - 混合 memory/non-memory read 的去重
   - Bash git 操作扫描的边界（stderr vs stdout）
   - `deferredSkippable` 在 thinking/attachment 之间的顺序保持
   - `summarizeRecentActivities` 的 trailing search/read 计数
