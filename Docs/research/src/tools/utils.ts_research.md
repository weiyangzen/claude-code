# 研究文档：src/tools/utils.ts

> 研究范围：代码、调用方、被调用方、配置、测试、脚本及必要实现上下文。
> 研究日期：2026-04-01
> 目标文件：`src/tools/utils.ts`

---

## 1. 场景与职责

`src/tools/utils.ts` 是 `src/tools/` 目录下的一个轻量级公共工具模块，当前仅包含 **2 个导出函数**：

- `tagMessagesWithToolUseID(...)`
- `getToolUseIDFromParentMessage(...)`

其核心职责是 **为工具调用（Tool Use）产生的用户消息（UserMessage）打上“来源工具调用 ID”标记**，从而让这些消息在 UI 层保持“瞬态（transient）”，直到对应工具真正执行完毕。这避免了在工具执行期间出现重复的 "is running" 或中间状态消息。

当前该模块的**唯一调用方**是 `src/tools/SkillTool/SkillTool.ts`（Skill 工具）。因此，本文件的研究必须结合 SkillTool 的调用上下文、消息渲染管线以及底层消息类型系统一并理解。

---

## 2. 功能点目的

### 2.1 `tagMessagesWithToolUseID`

**目的**：将一批消息（`UserMessage | AttachmentMessage | SystemMessage`）中的 `UserMessage` 打上 `sourceToolUseID` 字段，使其与某个工具调用（tool_use）建立关联。

**业务背景**：
- 当 SkillTool 被模型调用时，它会解析并执行对应的 skill（本地 skill 或远程 skill）。
- 执行过程中可能产生新的用户消息（例如 skill 的 prompt 内容、inline 注入的 meta 消息等）。
- 如果这些消息在工具尚未返回结果前就被 UI 静态渲染出来，用户会看到重复的“正在运行”提示或中间草稿。
- 通过打上 `sourceToolUseID`，UI 层的渲染逻辑可以识别出这些消息属于“尚未完成的工具调用”，从而将它们保持为 transient（动态/未固化），直到工具返回最终结果。

### 2.2 `getToolUseIDFromParentMessage`

**目的**：从父级 `AssistantMessage` 的消息内容块（content blocks）中，按工具名提取对应的 `tool_use` ID。

**业务背景**：
- SkillTool 被调用时，模型会生成一个 `assistant` 消息，其中包含一个 `tool_use` 块（name = `'Skill'`）。
- SkillTool 需要知道这个 `tool_use` 块的 `id`，才能将后续产生的用户消息正确关联到本次工具调用。
- 该函数通过遍历 `parentMessage.message.content`，找到 `type === 'tool_use' && name === toolName` 的块并返回其 `id`。

---

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 源码逐行分析

```typescript
import type {
  AssistantMessage,
  AttachmentMessage,
  SystemMessage,
  UserMessage,
} from 'src/types/message.js'
```

- 类型来源：`src/types/message.js`（在仓库工作树中未找到对应的 `.ts` 源文件，推断为构建时生成或 Bun 运行时通过路径映射解析的虚拟/生成模块；全仓库大量文件均以此路径导入消息类型）。

#### `tagMessagesWithToolUseID`

```typescript
export function tagMessagesWithToolUseID(
  messages: (UserMessage | AttachmentMessage | SystemMessage)[],
  toolUseID: string | undefined,
): (UserMessage | AttachmentMessage | SystemMessage)[] {
  if (!toolUseID) {
    return messages
  }
  return messages.map(m => {
    if (m.type === 'user') {
      return { ...m, sourceToolUseID: toolUseID }
    }
    return m
  })
}
```

- **防御性处理**：若 `toolUseID` 为 falsy，直接原样返回，避免无意义的对象拷贝。
- **仅对 `user` 类型消息打标**：`AttachmentMessage` 和 `SystemMessage` 不会被添加 `sourceToolUseID`。
- **不可变更新**：使用 spread `...m` 创建新对象，符合函数式风格。
- **动态字段注入**：`sourceToolUseID` 并非 `createUserMessage` 的显式参数，但运行时通过对象扩展直接附加；`UserMessage` 类型系统允许该字段存在（可能通过索引签名或生成类型中的可选字段）。

#### `getToolUseIDFromParentMessage`

```typescript
export function getToolUseIDFromParentMessage(
  parentMessage: AssistantMessage,
  toolName: string,
): string | undefined {
  const toolUseBlock = parentMessage.message.content.find(
    block => block.type === 'tool_use' && block.name === toolName,
  )
  return toolUseBlock && toolUseBlock.type === 'tool_use'
    ? toolUseBlock.id
    : undefined
}
```

- **输入**：`parentMessage` 是模型生成的 assistant 消息；`toolName` 是期望匹配的工具名（如 `'Skill'`）。
- **查找逻辑**：在 `message.content` 数组中查找第一个满足 `type === 'tool_use' && block.name === toolName` 的内容块。
- **类型收窄**：`find` 返回的类型是 `ContentBlock` 联合类型，二次判断 `toolUseBlock.type === 'tool_use'` 是为了在 TypeScript 中安全地访问 `.id` 属性（`ToolUseBlock` 特有的字段）。
- **返回值**：匹配成功返回 `id`（`string`），否则 `undefined`。

### 3.2 调用方上下文：SkillTool 中的使用

`src/tools/SkillTool/SkillTool.ts` 在第 64–66 行导入这两个函数：

```typescript
import {
  getToolUseIDFromParentMessage,
  tagMessagesWithToolUseID,
} from '../utils.js'
```

#### 第一次调用（Prompt Skill 路径，约第 728–755 行）

```typescript
const toolUseID = getToolUseIDFromParentMessage(parentMessage, SKILL_TOOL_NAME)

const newMessages = tagMessagesWithToolUseID(
  processedCommand.messages.filter(
    (m): m is UserMessage | AttachmentMessage | SystemMessage => {
      if (m.type === 'progress') return false
      // 过滤掉 command-message，因为 SkillTool 自己处理展示
      if (m.type === 'user' && 'message' in m) {
        const content = m.message.content
        if (
          typeof content === 'string' &&
          content.includes(`<${COMMAND_MESSAGE_TAG}>`)
        ) {
          return false
        }
      }
      return true
    },
  ),
  toolUseID,
)
```

- `processedCommand.messages` 是执行 skill 后产生的一批消息，可能包含 `progress`、`user`、`attachment`、`system` 等类型。
- 先过滤掉 `progress` 消息和带有 `<command_message>` 标签的用户消息。
- 然后对剩余消息中的 `user` 消息打上 `sourceToolUseID = toolUseID`。

#### 第二次调用（Remote/Inline Skill 路径，约第 1097–1106 行）

```typescript
const toolUseID = getToolUseIDFromParentMessage(parentMessage, SKILL_TOOL_NAME)
return {
  data: { success: true, commandName, status: 'inline' },
  newMessages: tagMessagesWithToolUseID(
    [createUserMessage({ content: finalContent, isMeta: true })],
    toolUseID,
  ),
}
```

- 远程 skill 加载后，将其内容包装成一个 `isMeta: true` 的用户消息直接注入上下文。
- 同样通过 `tagMessagesWithToolUseID` 打上 `sourceToolUseID`，保证这条 meta 消息在 SkillTool 返回前不会提前固化显示。

### 3.3 下游消费：`sourceToolUseID` 如何影响消息渲染

#### 3.3.1 `getToolUseID`（`src/utils/messages.ts:2765–2793`）

```typescript
export function getToolUseID(message: NormalizedMessage): string | null {
  switch (message.type) {
    // ...
    case 'user':
      if (message.sourceToolUseID) {
        return message.sourceToolUseID
      }
      if (message.message.content[0]?.type !== 'tool_result') {
        return null
      }
      return message.message.content[0].tool_use_id
    // ...
  }
}
```

- `getToolUseID` 是消息系统的核心关联函数，用于将任意消息映射到其所属的工具调用 ID。
- 对于普通 `tool_result` 用户消息，它从内容块中读取 `tool_use_id`；而对于被 `tagMessagesWithToolUseID` 标记过的消息，它优先返回 `sourceToolUseID`。
- 这意味着：**即使该用户消息不是标准的 `tool_result`，也能被 UI 识别为属于某个 tool_use**。

#### 3.3.2 `shouldRenderStatically`（`src/components/Messages.tsx:779–810`）

```typescript
export function shouldRenderStatically(
  message: RenderableMessage,
  streamingToolUseIDs: Set<string>,
  inProgressToolUseIDs: Set<string>,
  siblingToolUseIDs: ReadonlySet<string>,
  screen: Screen,
  lookups: ReturnType<typeof buildMessageLookups>
): boolean {
  if (screen === 'transcript') return true
  // ...
  const toolUseID = getToolUseID(message)
  if (!toolUseID) return true
  if (streamingToolUseIDs.has(toolUseID)) return false
  if (inProgressToolUseIDs.has(toolUseID)) return false
  if (hasUnresolvedHooksFromLookup(toolUseID, 'PostToolUse', lookups)) return false
  return every(siblingToolUseIDs, lookups.resolvedToolUseIDs)
}
```

- **关键链路**：`tagMessagesWithToolUseID` → `sourceToolUseID` → `getToolUseID` → `shouldRenderStatically`。
- 如果 `toolUseID` 仍在 `streamingToolUseIDs` 或 `inProgressToolUseIDs` 集合中，`shouldRenderStatically` 返回 `false`，消息保持 **transient/dynamic** 渲染。
- 这就解释了注释中所说的：*"prevents the 'is running' message from being duplicated in the UI"*。

#### 3.3.3 `expandKey`（`src/components/Messages.tsx:725–727`）

```typescript
function expandKey(msg: RenderableMessage): string {
  return (msg.type === 'assistant' || msg.type === 'user' ? getToolUseID(msg) : null) ?? msg.uuid
}
```

- `sourceToolUseID` 还用于消息点击展开时的 key 计算，确保同一 tool_use 下的消息（tool_use + 后续 user 消息）可以联动展开/折叠。

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖图

```
src/tools/utils.ts
├── 导入类型 ──► src/types/message.js (生成/虚拟模块)
├── 被调用方 ──► src/tools/SkillTool/SkillTool.ts (唯一调用方)
└── 间接消费 ──► src/utils/messages.ts (getToolUseID 读取 sourceToolUseID)
                └── src/components/Messages.tsx (shouldRenderStatically 决定渲染策略)
```

### 4.2 核心调用链（Call Chain）

1. **模型发起 Skill 调用** → `assistant` 消息含 `tool_use(name='Skill', id=xxx)`
2. **`SkillTool.ts`** 接收 `parentMessage`
   - 调用 `getToolUseIDFromParentMessage(parentMessage, 'Skill')` → 得到 `toolUseID`
3. **Skill 执行** 产生新的用户消息（prompt 内容、meta 消息等）
4. **`tagMessagesWithToolUseID(newMessages, toolUseID)`** → 为 `user` 消息附加 `sourceToolUseID`
5. **消息返回** 进入消息队列
6. **UI 渲染** (`Messages.tsx`)
   - `getToolUseID(msg)` 读取 `sourceToolUseID`
   - `shouldRenderStatically(...)` 判断工具是否仍在执行中
   - 若仍在执行，消息保持 transient，不固化显示
7. **工具返回结果** → `toolUseID` 被标记为 resolved → 消息转为 static 渲染

### 4.3 相关文件清单

| 文件路径 | 作用 |
|---------|------|
| `src/tools/utils.ts` | 本研究目标：提供 `tagMessagesWithToolUseID` 和 `getToolUseIDFromParentMessage` |
| `src/tools/SkillTool/SkillTool.ts` | 唯一调用方，分别在 prompt skill 路径（~L729）和 inline skill 路径（~L1097）使用 |
| `src/tools/SkillTool/constants.ts` | 定义 `SKILL_TOOL_NAME = 'Skill'` |
| `src/utils/messages.ts` | 核心消息工具库；`getToolUseID` 消费 `sourceToolUseID`（L2778） |
| `src/components/Messages.tsx` | UI 消息列表；`shouldRenderStatically`（L779）决定消息是否 transient |
| `src/components/MessageRow.tsx` | 消息行组件；使用 `getToolUseID` 计算 `toolUseID`（L207, 307, 332） |

---

## 5. 依赖与外部交互

### 5.1 类型依赖

`src/tools/utils.ts` 仅依赖以下类型（全部来自 `src/types/message.js`）：

- `AssistantMessage`：父级 assistant 消息，包含 `message.content`（`ContentBlock[]`）。
- `UserMessage`：用户消息，运行时会被扩展 `sourceToolUseID`。
- `AttachmentMessage`：附件消息，当前仅作为参数类型出现，不会被修改。
- `SystemMessage`：系统消息，当前仅作为参数类型出现，不会被修改。

> **注意**：`src/types/message.js` 在仓库源文件树中不存在，推断为构建时生成（可能是 Bun bundle 的虚拟解析或预生成类型声明）。全仓库有超过 100 处导入该路径，说明这是项目的标准做法。

### 5.2 运行时依赖

- **无外部 npm 包依赖**：本文件不导入任何第三方库。
- **无 Node.js/Bun 原生 API 依赖**：纯函数，无副作用。

### 5.3 调用方依赖

- **唯一调用方**：`src/tools/SkillTool/SkillTool.ts`
- 其他工具（如 `BashTool`、`FileEditTool`、`WebFetchTool` 等）虽然也有各自的 `utils.ts`（如 `src/tools/BashTool/utils.ts`），但**均不调用** `src/tools/utils.ts` 中的函数。
- 这意味着 `src/tools/utils.ts` 的当前设计是 **为 SkillTool 量身定制的**，命名上虽然放在 `tools/` 根目录下显得通用，但实际通用性有限。

---

## 6. 风险、边界与改进建议

### 6.1 风险与边界

#### 6.1.1 唯一调用方与过度泛化风险

- `src/tools/utils.ts` 位于 `src/tools/` 根目录，文件名非常通用（`utils.ts`），但当前只有两个函数且**仅被 SkillTool 使用**。
- 风险：新开发者可能误以为这是所有工具的公共工具库，将不相关的工具逻辑放入此处，导致该文件变成“杂物抽屉”。

#### 6.1.2 `sourceToolUseID` 的类型安全边界

- `tagMessagesWithToolUseID` 通过 `{ ...m, sourceToolUseID: toolUseID }` 动态注入字段。由于 `src/types/message.js` 源文件不可见，无法直接确认 `UserMessage` 类型是否显式声明了 `sourceToolUseID?: string`。
- 如果未来类型声明收紧（例如移除索引签名），此处可能在编译期报错。
- 当前 `createUserMessage`（`src/utils/messages.ts:460`）的签名中**没有** `sourceToolUseID` 参数，说明该字段是事后附加的，而非消息构造的原生一环。

#### 6.1.3 `getToolUseIDFromParentMessage` 的查找边界

- 该函数只返回**第一个**匹配的 `tool_use` 块的 `id`（`find` 而非 `filter`）。
- 如果未来某个 assistant 消息包含**多个同名工具调用**（例如一次请求中模型连续调用两次 `Skill`），该函数只能提取第一个的 ID，第二个 Skill 调用产生的新消息将无法正确关联。
- 当前 Anthropic API 的行为通常是一个 assistant 消息只包含一个 `tool_use`，但这不是协议层面的绝对保证。

#### 6.1.4 空 `toolUseID` 的静默跳过

- `tagMessagesWithToolUseID` 在 `!toolUseID` 时直接返回原数组，不做任何警告或报错。
- 如果 `getToolUseIDFromParentMessage` 因为某种原因（如模型输出格式异常）返回 `undefined`，后续消息将**不带 `sourceToolUseID`** 直接流入 UI，可能导致 transient 行为失效，出现消息闪烁或重复。

### 6.2 改进建议

#### 6.2.1 文件重命名或合并（高优先级）

考虑到当前该文件的唯一调用方是 SkillTool，建议：

- **方案 A**：将这两个函数迁移到 `src/tools/SkillTool/utils.ts` 或 `src/tools/SkillTool/skillToolUtils.ts`，与 SkillTool 的其他辅助逻辑放在一起，消除“通用工具库”的误导性。
- **方案 B**：如果预计未来会有其他工具需要类似功能（如 `AgentTool` 也可能需要为子代理消息打标），保留在 `src/tools/utils.ts`，但应在文件头添加明确注释说明当前使用范围。

#### 6.2.2 显式声明 `sourceToolUseID` 类型（中优先级）

- 在 `src/utils/messages.ts` 的 `createUserMessage` 参数列表中增加 `sourceToolUseID?: string`，使该字段成为 `UserMessage` 构造的一等公民，而不是通过运行时 spread 注入。
- 或者，在 `src/types/message.js`（或其生成源）中显式为 `UserMessage` 添加 `sourceToolUseID?: string` 字段声明。

#### 6.2.3 增加防御性日志（低优先级）

在 `tagMessagesWithToolUseID` 中，当 `toolUseID` 为空时，可在开发/调试模式下输出一条 `logForDebugging` 或 `console.warn`，帮助排查消息未正确关联的问题：

```typescript
if (!toolUseID) {
  // 建议：在 DEBUG 模式下记录警告
  return messages
}
```

#### 6.2.4 考虑泛化到更多工具（低优先级）

- `AgentTool`（`src/tools/AgentTool/AgentTool.tsx`）在 fork 子代理时也会产生新消息。检查其是否也需要 `sourceToolUseID` 标记来避免类似的 UI 重复问题。
- 如果确实需要，可将这两个函数提取为真正的跨工具公共能力，并统一在 `src/tools/utils.ts` 维护。

---

## 7. 结论

`src/tools/utils.ts` 是一个小而精的辅助模块，通过 **"为 user 消息附加 `sourceToolUseID`"** 这一关键机制，支撑了 SkillTool 的 transient 消息行为，防止工具执行期间 UI 出现重复或过早固化的消息。其技术链路清晰：

> `SkillTool` → `getToolUseIDFromParentMessage` → `tagMessagesWithToolUseID` → `sourceToolUseID` → `getToolUseID`（messages.ts）→ `shouldRenderStatically`（Messages.tsx）

尽管当前通用性有限（仅 SkillTool 使用），但它在消息-工具关联体系中扮演了不可或缺的桥梁角色。未来若需扩展，应优先考虑类型安全性和模块归属的清晰度。
