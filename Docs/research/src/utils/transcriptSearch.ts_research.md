# 研究文档：src/utils/transcriptSearch.ts

## 场景与职责

本模块为 REPL 转录本的 **`/` 实时搜索功能**提供可搜索文本的提取与缓存。在大型会话中，消息对象可能包含嵌套的 content block、tool result、attachment 等复杂结构。若每次按键都重新遍历并拼接这些结构，会导致严重的性能问题（历史上曾出现“按 Backspace 卡住”的现象，因为每次按键都重新 lowercase 约 1.5MB 文本）。本模块通过 `WeakMap` 缓存展平后的 lowercase 文本，将计算成本摊销到消息创建时。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `renderableSearchText(msg)` | 主入口：从 `RenderableMessage` 提取可搜索文本，带 `WeakMap` 缓存，返回已小写的字符串。 |
| `computeSearchText(msg)` | 实际展平逻辑，按 `msg.type` 分发处理。 |
| `toolUseSearchText(input)` | 从 `tool_use` 的 input 对象中提取用户可见的搜索字段（如 `command`、`file_path`、`prompt` 等）。 |
| `toolResultSearchText(r)` | 从工具原生输出（`toolUseResult`）中提取用户实际看到的文本，避免索引到仅面向模型的 system reminder 或 persisted-output 包装。 |

## 具体技术实现

### 1. WeakMap 缓存策略

```ts
const searchTextCache = new WeakMap<RenderableMessage, string>()
```

- 键为 `RenderableMessage` 对象本身；值为已转小写的完整搜索文本。
- `RenderableMessage` 在应用中被视为 append-only 且 immutable（替换时生成新对象），因此缓存一旦写入永远有效，无需失效逻辑。
- 旧实现中，调用方在拿到结果后会再次 `.toLowerCase()`，导致每次按键都处理 megabytes 级文本；本模块将 lowercase 前置到缓存写入阶段，消除了该热点。

### 2. 按消息类型的展平逻辑

#### `user` 消息
- 若 `content` 为字符串：直接取文本，但排除 `INTERRUPT_MESSAGE` 和 `INTERRUPT_MESSAGE_FOR_TOOL_USE` 这两个 UI sentinel（它们在屏幕上会被渲染为 `Interrupted · /issue...`，原始文本不应被搜索到）。
- 若 `content` 为数组：遍历 block。
  - `text` block：同样排除 sentinel 后取文本。
  - `tool_result` block：**不直接使用 `b.content`**（那是面向模型的序列化，包含 `<persisted-output>`、system-reminder、backgroundInfo 等 phantom 文本），而是 duck-type `msg.toolUseResult`（工具的原生输出对象），调用 `toolResultSearchText` 提取真正渲染在 UI 上的文本。

#### `assistant` 消息
- 遍历 content array：
  - `text` block：取 `b.text`。
  - `tool_use` block：调用 `toolUseSearchText(b.input)` 提取命令/路径/模式等可见字段。
  - 跳过 `thinking` block（历史 thinking 在 transcript mount 时会被 `hidePastThinking` 隐藏，不应被搜索）。

#### `attachment` 消息
- `relevant_memories`：取所有 memory 的 `content` 拼接（对应 `AttachmentMessage.tsx` 中的 `<Ansi>` 渲染）。
- `queued_command`：取 prompt 文本，但排除 `task-notification` 和 meta 命令（与 `VirtualMessageList.tsx` 的 `stickyPromptText` 逻辑保持一致）。

#### `collapsed_read_search`
- 若存在 `msg.relevantMemories`，拼接其 `content`（这些 relevant memories 已被 collapse 组吸收，但在 transcript mode 下通过 `CollapsedReadSearchContent` 可见）。

#### 其他类型
- `grouped_tool_use`、`system` 等：无可搜索文本，返回空。

### 3. System-reminder 剥离

无论哪种类型，最终原始文本都会经过一段循环，剥离所有 `<system-reminder>...</system-reminder>` 标签及其内容。这是因为在某些场景下（如 `cc -c` resume），system reminder 会插入到用户 prompt 行之间，但这些内容对用户不可见，不应产生搜索命中。

### 4. Tool Use 搜索文本提取（duck-type）

`toolUseSearchText(input)` 对 input 对象进行白名单字段提取：

- 单字符串字段：`command`、`pattern`、`file_path`、`path`、`prompt`、`description`、`query`、`url`、`skill`。
- 字符串数组字段：`args`、`files`（拼接为空格分隔）。

这些字段覆盖了 `Bash`、`Grep`、`Read`、`Edit`、`Agent`、`Skill` 等常见工具的 UI 渲染主参数。未匹配的字段被忽略（under-count > phantom 的设计哲学）。

### 5. Tool Result 搜索文本提取（duck-type）

`toolResultSearchText(r)` 优先匹配已知工具输出形状：

- `{ stdout, stderr? }` → `stdout + stderr`（Bash/Shell 类工具）。
- `{ file: { content } }` → `file.content`（Read 工具）。
- 通用字符串字段白名单：`content`、`output`、`result`、`text`、`message`。
- 字符串数组白名单：`filenames`、`lines`、`results`（换行拼接）。

未知形状返回空字符串，避免索引到 `rawOutputPath`、`backgroundTaskId` 等仅面向模型或内部的元数据。

## 关键代码路径与文件引用

- **主实现**：`src/utils/transcriptSearch.ts`（202 行）
- **调用方（消息列表搜索）**：`src/components/VirtualMessageList.tsx`、`src/components/Messages.tsx`（均调用 `renderableSearchText`）
- **Sentinel 定义**：`src/utils/messages.ts`（`INTERRUPT_MESSAGE`、`INTERRUPT_MESSAGE_FOR_TOOL_USE`）
- **RenderableMessage 类型**：`src/types/message.ts`
- **Attachment 渲染逻辑**：`src/components/AttachmentMessage.tsx`（relevant_memories、queued_command 的可见性规则）
- **Sticky prompt 逻辑**：`src/components/VirtualMessageList.tsx`（`stickyPromptText` 的 guard 与本模块镜像）

## 依赖与外部交互

- `src/types/message.js`：`RenderableMessage` 类型。
- `src/utils/messages.js`：`INTERRUPT_MESSAGE`、`INTERRUPT_MESSAGE_FOR_TOOL_USE`。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **Duck-type 导致的 under-count**：`toolResultSearchText` 和 `toolUseSearchText` 对未知工具形状返回空字符串，这意味着用户引入的自定义工具（如 MCP 工具）如果输出结构不匹配任何已知模式，其内容将完全无法通过 `/` 搜索找到。这在 MCP 生态扩展后可能成为明显缺陷。
2. **`toolUseResult` 与 `b.content` 的语义漂移**：模块假设 `msg.toolUseResult` 是 UI 渲染的真实来源，但如果某工具在 `mapToolResultToToolResultBlockParam` 中做了大量转换（如注入安全提示、格式化表格），而 `toolUseResult` 保留原始对象，则搜索文本可能与用户实际看到的文本不一致。
3. **WeakMap 的 GC 依赖**：缓存绑定在消息对象上，若消息对象被 React 保留引用（如虚拟列表的缓冲池），缓存不会被回收。但由于缓存的是字符串而非对象树，内存增长与消息文本量成正比，通常可控。

### 边界

- **大小写不敏感**：缓存阶段已完成 `toLowerCase()`，因此搜索匹配不区分大小写，但这也意味着原始大小写信息丢失，无法支持未来的大小写敏感搜索选项。
- **仅支持单语言（Unicode）**：`toLowerCase()` 对土耳其语等特殊大小写规则处理不完全（应使用 `toLocaleLowerCase('en')` 或明确指定）。
- **System-reminder 全剥离**：使用字符串扫描而非 DOM/AST 解析，若用户消息本身包含字面量 `<system-reminder>` 标签，也会被错误剥离（虽然这种情况极为罕见）。

### 改进建议

1. **Per-tool extractSearchText 接口**：源码注释中已提到 TODO——在 `Tool` 接口上增加 `extractSearchText(toolUseResult): string` 方法，让每种工具自行声明可搜索文本。这能彻底解决 duck-type 的 under-count 问题，并支持 MCP 工具的自定义输出格式。
2. **支持大小写敏感搜索**：将缓存拆分为原始文本和 lowercased 文本两个 WeakMap，或缓存原始文本由调用方决定大小写策略。
3. **更精确的 system-reminder 剥离**：仅剥离由系统注入的、位于特定位置的 reminder，而非全局字符串替换。可考虑给 system-reminder 添加不可见标记（如零宽字符），以便精确识别。
4. **增量更新缓存**：当前缓存是“全量展平”。对于超长的 Bash stdout（如数万行），可以考虑只缓存前 N KB 的摘要，因为终端搜索通常不会翻到那么深。
