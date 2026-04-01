# AskUserQuestionTool.tsx 研究文档

## 场景与职责

`AskUserQuestionTool.tsx` 是 Claude Code 交互式 CLI 中的核心工具之一，职责是**在代码执行过程中向用户发起结构化多选问卷**，以收集偏好、澄清歧义、获取实现决策或提供方向选择。该工具是模型与用户之间的"双向确认通道"，在 Plan Mode（计划模式）和常规执行模式下均被广泛使用。

典型使用场景包括：
- 用户请求存在多义性时，Claude 通过该工具询问具体偏好（如"使用哪种缓存策略？"）
- Plan Mode 中，Claude 在制定计划前收集用户约束条件
- 多方案对比时，通过 `preview` 字段展示代码片段/配置差异/ASCII 示意图，供用户可视化比较
- 支持多选（`multiSelect`）和自定义文本输入（"Other"选项）

该工具属于**延迟加载（deferred）**工具，且**必须用户交互（`requiresUserInteraction()`）**，因此不会出现在初始系统提示的完整 schema 中，需通过 `ToolSearch` 触发；同时在异步代理（async agents）和通道模式（channels）下会被禁用，避免无人值守时挂起。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| 多问卷支持（1-4 题） | 允许一次 tool call 收集多个相关决策，减少往返次数 |
| 每题 2-4 个选项 | 强制提供有限且互斥的选择，降低用户认知负担 |
| `multiSelect` | 当选项非互斥时，允许用户多选，答案以逗号分隔 |
| `preview` 预览 | 为单选题提供 side-by-side 的可视化对比（markdown 或 HTML 片段） |
| "Other" 自由输入 | 每个问题自动附加自定义输入选项，避免模型遗漏边缘情况 |
| `annotations` | 记录用户对预览选项的备注（notes）及所选 preview 内容，回传给模型 |
| 图片粘贴 | 在文本输入模式下支持粘贴图片，作为 `ImageBlockParam` 随答案提交 |
| Plan Mode 专属 footer | 提供"Chat about this"和"Skip interview and plan immediately"两个快捷动作 |
| HTML 预览校验 | 防止模型输出完整 HTML 文档或注入 `<script>`/`<style>` 标签 |
| 通道安全禁用 | 当用户通过 Telegram/Discord 等通道交互时，禁用该工具防止 TUI 挂起 |

---

## 具体技术实现

### 1. 数据结构与 Zod Schema

文件使用 `zod/v4` 定义了四层 schema，并通过 `lazySchema` 延迟初始化以打破循环引用/降低启动开销：

- **`questionOptionSchema`**：选项结构
  - `label`（展示文本，1-5 词）
  - `description`（选项含义说明）
  - `preview`（可选，预览内容）
- **`questionSchema`**：问题结构
  - `question`（问题文本，必须以问号结尾）
  - `header`（chip/tag，最大 12 字符）
  - `options`（2-4 个选项数组）
  - `multiSelect`（是否多选，默认 false）
- **`annotationsSchema`**：可选的每题注解
  - `preview`（所选选项的 preview 内容）
  - `notes`（用户自由备注）
- **`inputSchema` / `outputSchema`**：顶层入参/出参
  - `questions`（1-4 题数组）
  - `answers`（问题文本 → 答案字符串）
  - `annotations`（注解字典）
  - `metadata.source`（追踪来源，如 `"remember"`）

**唯一性校验（`UNIQUENESS_REFINE`）**：通过 `.refine()` 确保所有 `question` 文本全局唯一，且每题内部 `label` 唯一。

### 2. 工具定义（`buildTool`）

```ts
export const AskUserQuestionTool = buildTool({
  name: ASK_USER_QUESTION_TOOL_NAME,      // 'AskUserQuestion'
  searchHint: 'prompt the user with a multiple-choice question',
  maxResultSizeChars: 100_000,
  shouldDefer: true,
  // ...
})
```

关键钩子实现：
- **`prompt()`**：动态拼接基础提示与 preview 格式说明。若 `getQuestionPreviewFormat()` 返回 `'markdown'` 或 `'html'`，则追加对应格式的 `PREVIEW_FEATURE_PROMPT`。
- **`isEnabled()`**：当 `KAIROS` 或 `KAIROS_CHANNELS` feature 开启且 `getAllowedChannels().length > 0` 时返回 `false`，防止在通道模式下弹出 TUI 多选对话框导致挂起。
- **`requiresUserInteraction()`**：返回 `true`，标记为交互型工具。
- **`validateInput()`**：仅在 `getQuestionPreviewFormat() === 'html'` 时，对每个选项的 `preview` 调用 `validateHtmlPreview()` 进行轻量正则校验：
  - 禁止 `<html>`、`<body>`、`<!DOCTYPE>`
  - 禁止 `<script>`、`<style>` 标签（要求使用 inline `style`）
  - 要求内容中至少包含一个 HTML 标签
- **`checkPermissions()`**：固定返回 `behavior: 'ask'`，必须经过用户显式确认。
- **`call()`**：纯透传，将 `questions`、`answers`、`annotations` 原样返回。
- **`mapToolResultToToolResultBlockParam()`**：将答案格式化为模型可读的文本，例如：
  ```
  User has answered your questions: "Which library?"="lodash", selected preview: <div>...</div>, user notes: prefer v4. You can now continue...
  ```
- **渲染方法**：
  - `renderToolResultMessage`：使用 `AskUserQuestionResultMessage` 组件，在对话历史中展示 "User answered Claude's questions: · Q → A"
  - `renderToolUseRejectedMessage`：展示 "User declined to answer questions"

### 3. 权限/交互组件链路

该工具没有走通用的 `FallbackPermissionRequest`，而是注册了专用的 React 组件树：

```
PermissionRequest.tsx
  └── AskUserQuestionPermissionRequest.tsx
        ├── QuestionView.tsx            (常规单题：Select/SelectMulti + Other输入)
        │     └── 若含 preview → PreviewQuestionView.tsx
        │           └── PreviewBox.tsx  (带边框的 markdown/HTML 预览盒)
        ├── SubmitQuestionsView.tsx     (最终确认页)
        └── QuestionNavigationBar.tsx   (顶部 tab 导航)
```

状态管理由 `use-multiple-choice-state.ts` 提供 reducer：
- `currentQuestionIndex`：当前题号
- `answers`：已收集答案
- `questionStates`：每题的 `selectedValue` 和 `textInputValue`
- `isInTextInput`：是否处于文本输入焦点状态（用于禁用全局 tab 快捷键）

### 4. 图片附件流程

在 `AskUserQuestionPermissionRequest.tsx` 中：
1. `onImagePaste` 接收 base64、mediaType、filename、dimensions
2. 生成递增 `pasteId`，存入 `pastedContentsByQuestion`（按问题文本分组）
3. `cacheImagePath` + `storeImage` 持久化
4. 提交时 `convertImagesToBlocks()` 将图片转为 `ImageBlockParam[]`，经 `maybeResizeAndDownsampleImageBlock` 压缩后，作为 `contentBlocks` 参数传给 `toolUseConfirm.onAllow()`

### 5. Plan Mode 特殊逻辑

- `isInPlanMode` 通过 `useAppState(s => s.toolPermissionContext.mode) === 'plan'` 判断
- 底部 footer 在 plan 模式下额外显示 "Skip interview and plan immediately"
- 选择该选项会调用 `handleFinishPlanInterview`，向模型发送特定 feedback：
  > "The user has indicated they have provided enough answers for the plan interview. Stop asking clarifying questions and proceed to finish the plan..."
- "Chat about this" 则发送另一段 feedback，提示模型继续追问澄清

### 6. 分析追踪

在允许/拒绝/澄清/结束面试四个动作中，均通过 `logEvent` 上报：
- `tengu_ask_user_question_accepted`
- `tengu_ask_user_question_rejected`
- `tengu_ask_user_question_respond_to_claude`
- `tengu_ask_user_question_finish_plan_interview`

事件携带 `source`、`questionCount`、`isInPlanMode`、`interviewPhaseEnabled` 等字段。

---

## 关键代码路径与文件引用

### 本文件内部
- **schema 定义**：行 14-83（`questionOptionSchema`、`questionSchema`、`annotationsSchema`、`inputSchema`、`outputSchema`）
- **工具构建**：行 109-245（`buildTool` 调用）
- **HTML 校验**：行 250-265（`validateHtmlPreview`）
- **结果消息组件**：行 83-108（`AskUserQuestionResultMessage`）

### 直接依赖（被调用方）
| 文件 | 用途 |
|------|------|
| `src/Tool.ts` | `Tool`、`ToolDef`、`buildTool`、类型定义 |
| `src/utils/lazySchema.js` | `lazySchema` 延迟构造 zod schema |
| `src/bootstrap/state.ts` | `getAllowedChannels`、`getQuestionPreviewFormat` |
| `src/components/MessageResponse.js` | 结果消息展示容器 |
| `src/constants/figures.js` | `BLACK_CIRCLE` 符号 |
| `src/utils/permissions/PermissionMode.js` | `getModeColor` 颜色主题 |
| `./prompt.ts` | 工具名、描述、prompt 常量 |

### 调用方与注册点
| 文件 | 用途 |
|------|------|
| `src/tools.ts` | 将 `AskUserQuestionTool` 加入全局 tools 数组 |
| `src/components/permissions/PermissionRequest.tsx` | `permissionComponentForTool` 映射到专用权限组件 |
| `src/constants/tools.ts` | 列入 `ALL_AGENT_DISALLOWED_TOOLS`，禁止异步代理使用 |
| `src/utils/permissions/classifierDecision.ts` | 列入 `SAFE_YOLO_ALLOWLISTED_TOOLS`，自动模式免分类 |
| `src/constants/prompts.ts` | 检测该工具是否启用，调整系统提示 |
| `src/utils/messages.ts` | 引用 `ASK_USER_QUESTION_TOOL_NAME` |
| `src/components/tasks/RemoteSessionDetailDialog.tsx` | 引用工具名 |
| `src/skills/bundled/batch.ts` | skill prompt 引用 |
| `src/skills/bundled/scheduleRemoteAgents.ts` | skill prompt 引用 |
| `src/tools/EnterPlanModeTool/prompt.ts` | 在 EnterPlanMode 的 prompt 中引用该工具 |

### 配套 UI 组件
| 文件 | 职责 |
|------|------|
| `src/components/permissions/AskUserQuestionPermissionRequest/AskUserQuestionPermissionRequest.tsx` | 主权限组件，统筹状态、图片、提交、analytics |
| `src/components/permissions/AskUserQuestionPermissionRequest/QuestionView.tsx` | 单题渲染（Select/SelectMulti、Other、图片粘贴） |
| `src/components/permissions/AskUserQuestionPermissionRequest/PreviewQuestionView.tsx` | 带 preview 的 side-by-side 布局 |
| `src/components/permissions/AskUserQuestionPermissionRequest/PreviewBox.tsx` | 预览内容的带边框渲染盒（markdown + syntax highlight） |
| `src/components/permissions/AskUserQuestionPermissionRequest/SubmitQuestionsView.tsx` | 答案回顾与最终提交页 |
| `src/components/permissions/AskUserQuestionPermissionRequest/QuestionNavigationBar.tsx` | 问题 tab 导航与作答状态指示 |
| `src/components/permissions/AskUserQuestionPermissionRequest/use-multiple-choice-state.ts` | reducer 与 hook，管理答题进度状态 |

---

## 依赖与外部交互

### 运行时依赖
- **React + Ink**：所有 UI 渲染基于 `ink`（React for CLI），文件头部可见 `react/compiler-runtime` 的编译后产物
- **Zod v4**：输入输出校验与类型推断
- **Bun bundle feature flags**：`feature('KAIROS')`、`feature('KAIROS_CHANNELS')` 来自 `bun:bundle`
- **全局状态（bootstrap/state.ts）**：
  - `getQuestionPreviewFormat()` 决定 preview 是 markdown/html/无
  - `getAllowedChannels()` 决定是否在通道模式下禁用该工具

### 与权限系统的交互
- 该工具的 `checkPermissions` 固定返回 `behavior: 'ask'`，因此每次调用都会进入 `PermissionRequest` 流程
- `interactiveHandler.ts`（权限处理器）对 `requiresUserInteraction()` 为 true 的工具会跳过通道转发（channel relay），确保本地 TUI 弹窗是唯一的交互路径

### 与模型提示系统的交互
- `prompt()` 方法在运行时被调用，动态决定注入到模型上下文中的 tool description
- 若 SDK 消费者未配置 preview format，则 omit preview 相关 guidance

### 与 Agent/Skill 系统的交互
- 被列入 `ALL_AGENT_DISALLOWED_TOOLS`，因此**子代理/后台代理无法直接调用该工具**
- 多个 bundled skill（batch、scheduleRemoteAgents）的 prompt 模板中引用了该工具名，用于指导 coordinator 何时向用户提问

---

## 风险、边界与改进建议

### 风险与边界

1. **通道模式下的静默禁用**
   - `isEnabled()` 在 `getAllowedChannels().length > 0` 时返回 `false`。这意味着当用户通过 Telegram/Discord 使用 Claude 时，模型**完全无法**调用该工具，可能导致模型在需要澄清时陷入循环或做出错误假设。
   - 当前没有 fallback 机制（如将问题文本通过 `SendMessageTool` 转发到通道）。

2. **HTML Preview 的安全边界**
   - `validateHtmlPreview` 只是轻量正则检查，不是真正的 HTML parser。它无法阻止所有 XSS 向量（例如内联事件处理器 `onclick=`、伪协议 `javascript:`）。
   - 注释中明确提到："Inline event handlers are still possible; consumers should sanitize." 这意味着安全责任部分外泄到了 SDK 消费者端。

3. **图片粘贴的生命周期**
   - 图片通过 `cacheImagePath` + `storeImage` 存入内存/磁盘，但没有显见的清理逻辑。如果用户频繁粘贴大图，可能导致状态膨胀或磁盘占用增长。

4. **"Other" 选项与图片的耦合逻辑复杂**
   - 在 `handleQuestionAnswer` 中，当 `label === "__other__"` 且没有文本输入但有图片时，答案会被格式化为 `"(Image attached)"`，这种隐式转换对模型来说可能不够直观。

5. **Plan Mode 的硬编码反馈文本**
   - `handleRespondToClaude` 和 `handleFinishPlanInterview` 中的 feedback 字符串是硬编码的英文模板，未走 i18n 或配置化，未来多语言支持时需要重构。

6. **无单元测试覆盖**
   - 仓库中未找到针对 `AskUserQuestionTool` 或其 UI 组件的 `.test.ts(x)` 文件。schema 校验、HTML 校验、reducer 逻辑均缺乏自动化测试保护。

### 改进建议

1. **通道 fallback 机制**
   - 当 `isEnabled()` 为 false 时，可考虑在系统提示中告知模型"当前处于通道模式，无法使用多选问卷，请直接陈述问题或调用 SendMessageTool"，而不是让模型在不知情的情况下无法调用该工具。

2. **增强 HTML 校验**
   - 引入一个更严格的 allowlist-based sanitizer（如 DOMPurify 的轻量替代方案）在服务端对 preview HTML 进行清洗，而不是仅依赖正则。

3. **图片清理与预算**
   - 在 `onDone` 或组件 unmount 时清理已粘贴但未被采纳的图片；或对单张图片大小、总数量设置上限。

4. **提取 feedback 模板**
   - 将 Plan Mode 的 feedback 文本提取到 `prompt.ts` 或专门的文案配置文件中，便于后续 A/B 测试和国际化。

5. **补充测试**
   - 为 `validateHtmlPreview` 编写边界测试（合法 fragment、非法 tag、无 tag）
   - 为 `useMultipleChoiceState` reducer 编写状态流转测试
   - 为 `inputSchema` 的 `.refine()` 唯一性校验编写测试

6. **性能优化**
   - `AskUserQuestionPermissionRequestBody` 被 React Compiler 编译后产生了大量 memo cache 槽位（`$[0]` ~ `$[114]`），虽然保证了渲染性能，但也使代码可读性极差。建议在关键业务逻辑（如 `submitAnswers`）中提取为独立 hook，减少单组件体积。
