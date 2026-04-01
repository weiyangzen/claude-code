# PromptInput.tsx 深度研究文档

> **研究对象**: `src/components/PromptInput/PromptInput.tsx`  
> **研究范围**: 包含其同目录下的所有子组件、Hooks、工具函数及相关依赖  
> **执行器**: kimi (k2p5)  
> **研究日期**: 2026-04-01

---

## 1. 场景与职责

### 1.1 核心定位

`PromptInput.tsx` 是 Claude Code CLI 应用中**最核心的用户输入组件**，负责：

1. **用户输入接收**: 处理用户通过键盘、粘贴、语音等方式输入的文本
2. **多模态内容处理**: 支持文本、图片、长文本粘贴等多种内容类型
3. **智能建议系统**: 集成命令补全、文件路径补全、历史记录搜索等功能
4. **权限模式切换**: 管理工具执行权限模式（default/plan/auto等）
5. **团队协作支持**: 支持 Agent Swarms 模式下的队友通信和任务管理
6. **输入状态管理**: 处理输入缓冲、撤销、外部编辑器集成等高级功能

### 1.2 应用场景

| 场景 | 说明 |
|------|------|
| **普通对话模式** | 用户输入自然语言与 Claude 对话 |
| **Bash 模式** | 以 `!` 开头的命令直接执行 shell 命令 |
| **命令模式** | 以 `/` 开头的斜杠命令（如 `/help`, `/model`） |
| **Agent Swarms** | 多 Agent 协作场景下的队友间通信 |
| **图片粘贴** | 支持从剪贴板粘贴图片并发送给 Claude |
| **外部编辑器** | 支持在 `$EDITOR` 中编辑复杂输入 |

---

## 2. 功能点目的

### 2.1 输入模式管理 (`inputModes.ts`)

```typescript
// 三种输入模式
export type PromptInputMode = 'prompt' | 'bash' | 'command'

// 模式检测逻辑
export function getModeFromInput(input: string): HistoryMode {
  if (input.startsWith('!')) {
    return 'bash'
  }
  return 'prompt'
}
```

- **目的**: 根据输入前缀自动切换模式，提供不同的处理逻辑和 UI 反馈
- **bash 模式**: 直接执行 shell 命令，绕过 AI 处理
- **prompt 模式**: 标准 AI 对话模式

### 2.2 图片粘贴处理

```typescript
function onImagePaste(image: string, mediaType?: string, filename?: string, dimensions?: ImageDimensions, sourcePath?: string) {
  const pasteId = nextPasteIdRef.current++
  const newContent: PastedContent = {
    id: pasteId,
    type: 'image',
    content: image,
    mediaType: mediaType || 'image/png',
    filename: filename || 'Pasted image',
    dimensions,
    sourcePath
  }
  // ...
  insertTextAtCursor(prefix + formatImageRef(pasteId))
}
```

- **目的**: 支持用户从剪贴板粘贴图片，将图片转换为 `[Image #N]` 引用格式
- **图片存储**: 异步存储到磁盘，避免阻塞 UI
- **引用管理**: 通过 `pastedContents` 状态管理所有粘贴内容

### 2.3 长文本截断 (`useMaybeTruncateInput.ts`)

```typescript
const TRUNCATION_THRESHOLD = 10000 // 字符数阈值
const PREVIEW_LENGTH = 1000 // 保留的预览长度

export function maybeTruncateInput(
  input: string,
  pastedContents: Record<number, PastedContent>,
): { newInput: string; newPastedContents: Record<number, PastedContent> } {
  // 超过阈值时，保留开头和结尾，中间用占位符替代
}
```

- **目的**: 防止超长输入导致性能问题
- **实现**: 保留前后各 500 字符，中间内容存储为独立的 `PastedContent`

### 2.4 输入缓冲区与撤销 (`useInputBuffer`)

```typescript
const {
  pushToBuffer,
  undo,
  canUndo,
  clearBuffer
} = useInputBuffer({
  maxBufferSize: 50,
  debounceMs: 1000
})
```

- **目的**: 提供类似编辑器的撤销功能
- **机制**: 每次输入变更后 1 秒将状态推入缓冲区，支持最多 50 步撤销

### 2.5 提示建议系统 (`usePromptSuggestion`)

```typescript
const {
  suggestion: promptSuggestion,
  markAccepted,
  logOutcomeAtSubmission,
  markShown
} = usePromptSuggestion({
  inputValue: input,
  isAssistantResponding: isLoading
})
```

- **目的**: 基于用户历史行为和当前上下文提供输入建议
- **集成**: 与 `speculation` 系统配合，实现预生成内容的快速提交

### 2.6 页脚导航与任务管理

```typescript
// Footer pills 导航
const footerItems = useMemo(() => [
  tasksFooterVisible && 'tasks',
  tmuxFooterVisible && 'tmux', 
  bagelFooterVisible && 'bagel',
  teamsFooterVisible && 'teams',
  bridgeFooterVisible && 'bridge',
  companionFooterVisible && 'companion'
].filter(Boolean) as FooterItem[], [/* deps */])
```

- **目的**: 提供快速访问任务、团队、桥接等功能的入口
- **交互**: 支持 ↑/↓ 导航，Enter 打开选中项

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 Props 定义

```typescript
type Props = {
  debug: boolean
  ideSelection: IDESelection | undefined
  toolPermissionContext: ToolPermissionContext
  setToolPermissionContext: (ctx: ToolPermissionContext) => void
  apiKeyStatus: VerificationStatus
  commands: Command[]
  agents: AgentDefinition[]
  isLoading: boolean
  verbose: boolean
  messages: Message[]
  onAutoUpdaterResult: (result: AutoUpdaterResult) => void
  autoUpdaterResult: AutoUpdaterResult | null
  input: string
  onInputChange: (value: string) => void
  mode: PromptInputMode
  onModeChange: (mode: PromptInputMode) => void
  stashedPrompt: {
    text: string
    cursorOffset: number
    pastedContents: Record<number, PastedContent>
  } | undefined
  setStashedPrompt: (value: ...) => void
  submitCount: number
  onShowMessageSelector: () => void
  onMessageActionsEnter?: () => void
  mcpClients: MCPServerConnection[]
  pastedContents: Record<number, PastedContent>
  setPastedContents: React.Dispatch<...>
  vimMode: VimMode
  setVimMode: (mode: VimMode) => void
  showBashesDialog: string | boolean
  setShowBashesDialog: (show: string | boolean) => void
  onExit: () => void
  getToolUseContext: (...) => ProcessUserInputContext
  onSubmit: (input: string, helpers: PromptInputHelpers, ...) => Promise<void>
  onAgentSubmit?: (input: string, task: ..., helpers: ...) => Promise<void>
  // ... 更多属性
}
```

#### 3.1.2 PastedContent 类型

```typescript
type PastedContent = {
  id: number
  type: 'image' | 'text'
  content: string  // base64 (image) 或文本内容
  mediaType?: string  // e.g., 'image/png'
  filename?: string
  dimensions?: ImageDimensions
  sourcePath?: string
}
```

### 3.2 关键流程

#### 3.2.1 输入提交流程

```
用户按下 Enter
    ↓
onSubmit 回调触发
    ↓
1. 检查 footer 是否有选中项 → 打开对应对话框
2. 检查是否在 agent 选择模式 → 确认选择
3. 检查是否匹配 promptSuggestion → 接受建议
4. 检查是否是 @name 直接消息 → 发送给队友
5. 检查是否有图片但无文本 → 允许提交
6. 检查是否有未处理的建议 → 阻止提交
7. 路由到对应 Agent 或 Leader 提交
    ↓
clearBuffer() + resetHistory() + setCursorOffset(0)
```

#### 3.2.2 图片粘贴流程

```
用户粘贴图片
    ↓
onImagePaste(imageData)
    ↓
1. 生成递增的 pasteId
2. 创建 PastedContent 对象
3. cacheImagePath() - 缓存路径
4. storeImage() - 异步存储到磁盘
5. setPastedContents() - 更新状态
6. insertTextAtCursor(`[Image #${pasteId}]`)
    ↓
输入框显示 [Image #N] 占位符
```

#### 3.2.3 权限模式切换流程

```
用户按下 Shift+Tab
    ↓
handleCycleMode()
    ↓
1. 检查是否是首次进入 auto 模式
   → 是: 显示 AutoModeOptInDialog
   → 否: 直接切换
2. 调用 cyclePermissionMode() 计算下一模式
3. 更新 AppState 中的 toolPermissionContext
4. 调用 setToolPermissionContext() 触发重新检查
5. syncTeammateMode() 同步到队友配置
```

### 3.3 高亮系统实现

```typescript
const combinedHighlights = useMemo((): TextHighlight[] => {
  const highlights: TextHighlight[] = []
  
  // 1. 图片引用高亮（选中时反色）
  for (const ref of imageRefPositions) {
    if (cursorOffset === ref.start) {
      highlights.push({ start: ref.start, end: ref.end, inverse: true, priority: 8 })
    }
  }
  
  // 2. 历史搜索匹配高亮
  if (isSearchingHistory && historyMatch) {
    highlights.push({ start: cursorOffset, end: cursorOffset + historyQuery.length, color: 'warning', priority: 20 })
  }
  
  // 3. btw 触发器高亮（黄色）
  for (const trigger of btwTriggers) {
    highlights.push({ start: trigger.start, end: trigger.end, color: 'warning', priority: 15 })
  }
  
  // 4. 斜杠命令高亮（蓝色）
  for (const trigger of slashCommandTriggers) {
    highlights.push({ start: trigger.start, end: trigger.end, color: 'suggestion', priority: 5 })
  }
  
  // 5. @name 提及高亮（队友颜色）
  for (const mention of memberMentionHighlights) {
    highlights.push({ start: mention.start, end: mention.end, color: mention.themeColor, priority: 5 })
  }
  
  // 6. ultrathink 彩虹高亮
  if (isUltrathinkEnabled()) {
    for (const trigger of thinkTriggers) {
      for (let i = trigger.start; i < trigger.end; i++) {
        highlights.push({ start: i, end: i + 1, color: getRainbowColor(i - trigger.start), shimmerColor: getRainbowColor(i - trigger.start, true), priority: 10 })
      }
    }
  }
  
  return highlights
}, [/* deps */])
```

### 3.4 键盘快捷键绑定

```typescript
// Chat 上下文快捷键
const chatHandlers = useMemo(() => ({
  'chat:undo': handleUndo,
  'chat:newline': handleNewline,
  'chat:externalEditor': handleExternalEditor,
  'chat:stash': handleStash,
  'chat:modelPicker': handleModelPicker,
  'chat:thinkingToggle': handleThinkingToggle,
  'chat:cycleMode': handleCycleMode,
  'chat:imagePaste': handleImagePaste
}), [/* deps */])

useKeybindings(chatHandlers, { context: 'Chat', isActive: !isModalOverlayActive })

// Footer 导航快捷键
useKeybindings({
  'footer:up': () => { /* ... */ },
  'footer:down': () => { /* ... */ },
  'footer:openSelected': () => { /* ... */ },
  // ...
}, { context: 'Footer', isActive: !!footerItemSelected && !isModalOverlayActive })
```

---

## 4. 关键代码路径与文件引用

### 4.1 组件文件结构

```
src/components/PromptInput/
├── PromptInput.tsx                 # 主组件 (~2340 lines)
├── PromptInputFooter.tsx           # 页脚组件
├── PromptInputFooterLeftSide.tsx   # 页脚左侧（模式指示器）
├── PromptInputFooterSuggestions.tsx # 建议列表 UI
├── PromptInputHelpMenu.tsx         # 帮助菜单
├── PromptInputModeIndicator.tsx    # 模式指示器（❯ 符号）
├── PromptInputQueuedCommands.tsx   # 队列命令显示
├── PromptInputStashNotice.tsx      # Stash 提示
├── Notifications.tsx               # 通知系统
├── inputModes.ts                   # 输入模式工具函数
├── inputPaste.ts                   # 粘贴处理逻辑
├── useMaybeTruncateInput.ts        # 长文本截断 Hook
├── usePromptInputPlaceholder.ts    # 占位符 Hook
├── useShowFastIconHint.ts          # Fast 图标提示 Hook
├── useSwarmBanner.ts               # Swarm 横幅 Hook
├── utils.ts                        # 通用工具函数
├── HistorySearchInput.tsx          # 历史搜索输入
├── IssueFlagBanner.tsx             # Issue 标记横幅
├── SandboxPromptFooterHint.tsx     # Sandbox 提示
├── ShimmeredInput.tsx              # 闪光输入效果
└── VoiceIndicator.tsx              # 语音指示器
```

### 4.2 核心依赖关系

```
PromptInput.tsx
├── 子组件
│   ├── PromptInputFooter
│   ├── PromptInputModeIndicator
│   ├── PromptInputQueuedCommands
│   ├── PromptInputStashNotice
│   └── Notifications (内联渲染)
├── Hooks
│   ├── useInputBuffer (撤销功能)
│   ├── useMaybeTruncateInput (长文本截断)
│   ├── usePromptInputPlaceholder (动态占位符)
│   ├── usePromptSuggestion (提示建议)
│   ├── useTypeahead (自动补全)
│   ├── useArrowKeyHistory (历史导航)
│   ├── useHistorySearch (历史搜索)
│   ├── useSwarmBanner (Swarm 横幅)
│   └── useCommandQueue (命令队列)
├── 状态管理
│   ├── useAppState / useAppStateStore
│   └── useSetAppState
├── 上下文
│   ├── useNotifications
│   └── useKeybindings / useKeybinding
└── 工具函数
    ├── inputModes.ts
    ├── inputPaste.ts
    └── utils.ts
```

### 4.3 调用方分析

| 调用方 | 文件路径 | 用途 |
|--------|----------|------|
| **REPL** | `src/screens/REPL.tsx` | 主渲染入口，传递所有 Props |
| **FullscreenLayout** | `src/components/FullscreenLayout.tsx` | 全屏布局下的输入处理 |
| **HelpV2/General** | `src/components/HelpV2/General.tsx` | 帮助系统中的输入引用 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 模块 | 用途 |
|------|------|
| `bun:bundle` | Feature flag 系统 (`feature()`) |
| `chalk` | 终端颜色格式化 |
| `strip-ansi` | ANSI 转义码清理 |
| `figures` | 终端图标 (❯ 等) |
| `react` / `ink` | React 组件和终端渲染 |

### 5.2 内部依赖

| 模块路径 | 用途 |
|----------|------|
| `src/state/AppState.js` | 全局状态管理 |
| `src/context/notifications.js` | 通知系统 |
| `src/keybindings/useKeybinding.js` | 键盘快捷键 |
| `src/hooks/useTypeahead.tsx` | 自动补全逻辑 |
| `src/hooks/useCommandQueue.js` | 命令队列 |
| `src/utils/config.js` | 配置管理 (`getGlobalConfig`) |
| `src/utils/imagePaste.js` | 图片粘贴工具 |
| `src/utils/imageStore.js` | 图片存储 |
| `src/utils/permissions/permissionSetup.js` | 权限模式切换 |
| `src/utils/swarm/teamHelpers.js` | 团队协作工具 |

### 5.3 与 REPL.tsx 的交互

```typescript
// REPL.tsx 中的关键调用
<PromptInput
  debug={debug}
  ideSelection={ideSelection}
  toolPermissionContext={toolPermissionContext}
  setToolPermissionContext={setToolPermissionContext}
  apiKeyStatus={apiKeyStatus}
  commands={commands}
  agents={agents}
  isLoading={isLoading}
  // ... 更多 props
  onSubmit={handlePromptSubmitWrapper}
  onAgentSubmit={handleAgentSubmit}
/>
```

**数据流**:
1. REPL 管理全局状态（`messages`, `isLoading` 等）
2. PromptInput 接收用户输入并通过 `onSubmit` 回调传递
3. REPL 的 `handlePromptSubmitWrapper` 处理提交，调用 `handlePromptSubmit`
4. 处理结果通过状态更新反馈到 PromptInput

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 性能风险

| 风险点 | 说明 | 缓解措施 |
|--------|------|----------|
| **长文本输入** | 超过 10,000 字符的输入可能导致渲染卡顿 | `useMaybeTruncateInput` 自动截断 |
| **频繁状态更新** | 每次按键都触发大量 hooks 重新计算 | 使用 `useMemo` 和 `useCallback` 优化 |
| **图片粘贴** | 大图片 base64 编码占用内存 | 异步存储到磁盘，UI 只保留引用 |
| **建议列表** | 大量建议项（>100）时渲染缓慢 | 虚拟滚动 + 最大显示限制 (OVERLAY_MAX_ITEMS = 5) |

#### 6.1.2 状态一致性风险

```typescript
// 风险示例：footerSelection 可能指向已不存在的 pill
const footerItemSelected = rawFooterSelection && footerItems.includes(rawFooterSelection) 
  ? rawFooterSelection 
  : null

useEffect(() => {
  if (rawFooterSelection && !footerItemSelected) {
    setAppState(prev => ({ ...prev, footerSelection: null }))
  }
}, [rawFooterSelection, footerItemSelected])
```

**说明**: Footer pills 可能因外部状态变化（如任务完成）而消失，需要及时清理选择状态。

#### 6.1.3 竞态条件

```typescript
// 图片粘贴和文本输入的竞态
pendingSpaceAfterPillRef.current = true  // 在 onImagePaste 中设置
// 如果用户在图片粘贴后立即输入，lazySpaceInputFilter 会处理
// 但如果输入发生在 ref 更新之前，可能导致空格丢失
```

### 6.2 边界情况

| 边界情况 | 处理逻辑 |
|----------|----------|
| **空输入提交** | 阻止提交，除非有图片附件 |
| **历史搜索无匹配** | 显示 `historyFailedMatch` 提示 |
| **Vim 模式冲突** | 通过 `isVimModeEnabled()` 检测，使用 `VimTextInput` 替代 `TextInput` |
| **多行输入** | 检测 `isCursorOnFirstLine` / `isCursorOnLastLine` 决定 ↑/↓ 行为 |
| **外部输入变更** | 通过 `lastInternalInputRef` 检测，自动移动光标到末尾 |
| **模态对话框打开** | `isModalOverlayActive` 禁用大部分输入处理 |

### 6.3 改进建议

#### 6.3.1 代码组织

1. **拆分超大组件**: `PromptInput.tsx` 超过 2300 行，建议拆分为：
   - `PromptInputCore.tsx` - 核心输入逻辑
   - `PromptInputDialogs.tsx` - 对话框管理
   - `PromptInputHandlers.ts` - 事件处理函数

2. **提取常量**: 将分散的魔法数字提取为命名常量：
   ```typescript
   // 当前分散在代码中
   const FOOTER_TEMPORARY_STATUS_TIMEOUT = 5000
   const MAX_VISIBLE_NOTIFICATIONS = 3
   const PASTE_THRESHOLD = 500  // 需要查找定义位置
   ```

#### 6.3.2 性能优化

1. **虚拟列表**: 建议列表使用虚拟滚动处理大量项
2. **防抖优化**: 当前 `useInputBuffer` 使用 1s 防抖，可根据输入速度动态调整
3. **Memo 优化**: `combinedHighlights` 计算较复杂，可考虑使用 `useMemo` 的自定义比较函数

#### 6.3.3 可测试性

1. **提取纯函数**: 将 `onSubmit` 中的复杂逻辑提取为可独立测试的纯函数
2. **Mock 接口**: 为 `useTypeahead`, `usePromptSuggestion` 等 hooks 提供 mock 接口
3. **边界测试**: 增加对超长输入、快速输入、并发粘贴等场景的测试

#### 6.3.4 可访问性

1. **屏幕阅读器支持**: 当前依赖视觉高亮（如图片引用反色），需要增加 ARIA 属性
2. **键盘导航**: Footer pills 的导航逻辑较复杂，建议统一使用 `useFocusManager`

#### 6.3.5 类型安全

```typescript
// 当前存在一些 any 类型和类型断言
const suggestionText = promptSuggestionState.text as string  // 可改进

// 建议定义更严格的类型
interface PromptSuggestionState {
  text: string | null
  shownAt: number
  // ...
}
```

---

## 7. 附录

### 7.1 关键常量汇总

| 常量 | 值 | 用途 |
|------|-----|------|
| `PROMPT_FOOTER_LINES` | 5 | 页脚预留行数 |
| `MIN_INPUT_VIEWPORT_LINES` | 3 | 输入区域最小可见行数 |
| `FOOTER_TEMPORARY_STATUS_TIMEOUT` | 5000ms | 页脚临时状态显示时长 |
| `MAX_VISIBLE_NOTIFICATIONS` | 3 | 最大可见通知数量 |
| `TRUNCATION_THRESHOLD` | 10000 | 长文本截断阈值 |
| `PREVIEW_LENGTH` | 1000 | 截断后保留的预览长度 |
| `OVERLAY_MAX_ITEMS` | 5 | 建议列表最大显示项 |

### 7.2 调试技巧

```typescript
// 启用调试日志
import { logForDebugging } from 'src/utils/debug.js'

// 在关键路径添加日志
logForDebugging(`[onSubmit] early return: suggestions showing (count=${suggestionsState.suggestions.length})`)
```

### 7.3 相关文档

- `src/hooks/useTypeahead.tsx` - 自动补全系统详细实现
- `src/utils/permissions/permissionSetup.js` - 权限模式系统
- `src/utils/swarm/teamHelpers.js` - Agent Swarms 协作机制
- `src/state/AppState.js` - 全局状态管理

---

*文档结束*
