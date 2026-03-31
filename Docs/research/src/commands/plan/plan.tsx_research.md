# 研究文档：src/commands/plan/plan.tsx

## 场景与职责

`src/commands/plan/plan.tsx` 是 Claude Code `/plan` 斜杠命令的**核心实现文件**。它负责处理用户输入 `/plan` 后的全部交互逻辑，包括：

1. **进入 Plan Mode**：如果当前会话不在 plan 模式，切换权限上下文并启用 plan 模式。
2. **查看当前计划**：如果已在 plan 模式，读取并展示当前会话的计划文件内容。
3. **外部编辑器打开**：支持 `/plan open` 子命令，调用用户配置的外部编辑器（如 VS Code、vim）打开计划文件。

该文件是一个 `local-jsx` 类型命令的实现模块，导出 `call` 函数，并内部定义了一个 React 组件 `PlanDisplay` 用于在终端中渲染计划内容。

## 功能点目的

### 1. Plan Mode 切换

Plan Mode 是 Claude Code 的一种特殊权限模式。进入该模式后，系统会要求用户先撰写计划（plan），然后才能继续与模型交互。`/plan` 命令（不带参数或带描述）是进入该模式的入口之一。

### 2. 计划内容展示

当用户已经在 plan 模式时，再次输入 `/plan` 会显示当前已写入的计划文件路径和内容，帮助用户回顾或确认计划状态。

### 3. 外部编辑器集成

用户可以通过 `/plan open` 在外部编辑器中打开计划文件。系统支持 GUI 编辑器（如 VS Code、Cursor）和终端编辑器（如 vim、nano），并会自动处理 Ink TUI 的暂停/恢复逻辑。

## 具体技术实现

### 关键流程

#### `call` 函数主流程

```typescript
export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
  args: string,
): Promise<React.ReactNode>
```

流程分支如下：

1. **获取当前权限模式**：`const currentMode = appState.toolPermissionContext.mode`
2. **不在 plan 模式**：
   - 调用 `handlePlanModeTransition(currentMode, 'plan')` 记录状态转换副作用。
   - 调用 `prepareContextForPlanMode(prev.toolPermissionContext)` 准备权限上下文（保存进入 plan 前的模式到 `prePlanMode`，并可能根据设置启用 auto 模式）。
   - 应用权限更新：`applyPermissionUpdate(..., { type: 'setMode', mode: 'plan', destination: 'session' })`。
   - 如果用户输入了非 `open` 的描述文本，调用 `onDone('Enabled plan mode', { shouldQuery: true })`，触发模型继续对话；否则仅显示启用消息。
3. **已在 plan 模式**：
   - 读取计划内容：`getPlan()` 和计划路径：`getPlanFilePath()`。
   - 如果计划为空，提示 "No plan written yet"。
   - 如果参数首词为 `open`，调用 `editFileInEditor(planPath)` 打开外部编辑器，并根据结果调用 `onDone` 报告成功或失败。
   - 否则，构建 `PlanDisplay` JSX 组件，通过 `renderToString` 将其渲染为纯文本字符串，再通过 `onDone(output)` 输出到终端。

#### `PlanDisplay` 组件

这是一个纯展示型 React 函数组件，使用 React Compiler（通过 `import { c as _c } from "react/compiler-runtime"` 和 `_c(11)` 缓存钩子）进行渲染优化。组件接收三个 props：

- `planContent`: 计划文件内容
- `planPath`: 计划文件绝对路径
- `editorName`: 外部编辑器显示名称（可选）

输出结构为纵向排列的 Ink `Box`/`Text`：
- 标题：`Current Plan`（粗体）
- 路径：`planPath`（暗淡色）
- 内容：`planContent`（带顶部边距）
- 提示：如果检测到外部编辑器，显示 `"/plan open" to edit this plan in {editorName}`

### 关键数据结构

- **`LocalJSXCommandContext`**（来自 `src/types/command.ts`）：包含 `getAppState`、`setAppState` 等状态访问器，以及 `setMessages`、`options` 等 TUI 上下文。
- **`LocalJSXCommandOnDone`**：命令完成回调，可传入结果字符串和选项（如 `shouldQuery` 控制是否继续向模型发请求）。
- **`EditorResult`**（来自 `src/utils/promptEditor.ts`）：`{ content: string | null, error?: string }`。

## 关键代码路径与文件引用

### 直接依赖

| 路径 | 导入符号 | 作用 |
|------|----------|------|
| `../../bootstrap/state.js` | `handlePlanModeTransition` | 记录 plan 模式进/出的全局状态副作用（如设置 `needsPlanModeExitAttachment`）。 |
| `../../commands.js` | `LocalJSXCommandContext` | 命令上下文类型定义。 |
| `../../ink.js` | `Box`, `Text` | Ink UI 组件，用于终端渲染。 |
| `../../types/command.js` | `LocalJSXCommandOnDone` | 命令完成回调类型。 |
| `../../utils/editor.js` | `getExternalEditor` | 获取用户配置的外部编辑器（`$VISUAL`、`$EDITOR` 或系统默认）。 |
| `../../utils/ide.js` | `toIDEDisplayName` | 将编辑器命令字符串映射为人类可读的显示名称（如 `code` → `VS Code`）。 |
| `../../utils/permissions/PermissionUpdate.js` | `applyPermissionUpdate` | 将权限更新（如模式切换）应用到权限上下文。 |
| `../../utils/permissions/permissionSetup.js` | `prepareContextForPlanMode` | 在进入 plan 模式前预处理权限上下文（保存 `prePlanMode`，处理 auto 模式过渡）。 |
| `../../utils/plans.js` | `getPlan`, `getPlanFilePath` | 读取计划文件内容和路径。 |
| `../../utils/promptEditor.ts` | `editFileInEditor` | 同步调用外部编辑器打开文件，处理 Ink 暂停/恢复。 |
| `../../utils/staticRender.tsx` | `renderToString` | 将 React 节点渲染为纯文本字符串（剥离 ANSI）。 |

### 间接依赖与调用链

- **`prepareContextForPlanMode`**（`src/utils/permissions/permissionSetup.ts`）
  - 内部可能调用 `shouldPlanUseAutoMode()`、`stripDangerousPermissionsForAutoMode()`、`restoreDangerousPermissions()` 以及 `autoModeStateModule?.setAutoModeActive()`。
  - 当当前模式为 `auto` 且设置允许时，会保存 `prePlanMode: 'auto'`；当设置要求 plan 模式使用 auto 时，会激活 auto 模式并剥离危险权限。

- **`getPlan` / `getPlanFilePath`**（`src/utils/plans.js`）
  - `getPlanFilePath()` → `getPlanSlug(getSessionId())` → 从 `planSlugCache` 获取或生成一个单词 slug（如 `happy-fox`）。
  - 计划文件存储在 `~/.claude/plans/{slug}.md`（或 `settings.json` 中配置的 `plansDirectory`）。
  - 支持子代理（subagent）通过 `agentId` 参数生成 `{slug}-agent-{agentId}.md`。

- **`editFileInEditor`**（`src/utils/promptEditor.ts`）
  - 获取 Ink 实例：`instances.get(process.stdout)`。
  - 判断编辑器类型（GUI vs Terminal）：GUI 编辑器调用 `inkInstance.pause()` + `suspendStdin()`；终端编辑器调用 `enterAlternateScreen()`。
  - 使用 `execSync_DEPRECATED` 阻塞式启动编辑器，等待用户保存退出。
  - 最后恢复 Ink 状态并读取文件内容返回。

- **`renderToString`**（`src/utils/staticRender.tsx`）
  - 使用 `renderToAnsiString` 将 JSX 渲染到 `PassThrough` 流，捕获 ANSI 输出。
  - 通过 `stripAnsi` 去除 ANSI 转义序列，得到纯文本字符串。
  - 用于让 `local-jsx` 命令像 `local` 命令一样通过 `onDone` 输出文本结果。

## 依赖与外部交互

### 状态系统

`call` 函数深度依赖 `context.getAppState()` 和 `context.setAppState()` 来读取和修改全局应用状态，特别是 `toolPermissionContext`：

```typescript
setAppState(prev => ({
  ...prev,
  toolPermissionContext: applyPermissionUpdate(
    prepareContextForPlanMode(prev.toolPermissionContext),
    { type: 'setMode', mode: 'plan', destination: 'session' }
  )
}))
```

### 文件系统

通过 `src/utils/plans.js` 和 `src/utils/fsOperations.js` 间接与文件系统交互：
- 读取计划文件：`getFsImplementation().readFileSync(filePath, { encoding: 'utf-8' })`
- 计划目录创建：`getFsImplementation().mkdirSync(plansPath)`

### 外部进程

通过 `editFileInEditor` 启动用户配置的外部编辑器进程，涉及 `child_process` 的 `spawn`/`spawnSync`（在 `editor.ts` 和 `promptEditor.ts` 中封装）。

### TUI / Ink

- 渲染 `PlanDisplay` 组件时使用 `Box` 和 `Text`（来自 `src/ink.js`，实际为 `ThemedBox` 和 `ThemedText`）。
- `editFileInEditor` 需要直接操作 Ink 实例（`instances.get(process.stdout)`）来暂停/恢复渲染和 stdin 处理。

## 风险、边界与改进建议

### 风险与边界

1. **`await editFileInEditor` 的类型不匹配**
   - `editFileInEditor` 在 `promptEditor.ts` 中声明为同步函数：`export function editFileInEditor(filePath: string): EditorResult`。
   - 但在 `plan.tsx` 中使用了 `await editFileInEditor(planPath)`。虽然 JavaScript 中 `await` 一个非 Promise 值会立即返回该值，不会报错，但这是一种**类型/语义不一致**。如果未来 `editFileInEditor` 被改为真正的异步函数，代码行为不变；但如果有人误以为它已经是异步的，可能会在相关代码审查中造成困惑。

2. **计划文件读取的静默失败**
   - `getPlan()` 在 `plans.ts` 中对 `ENOENT` 返回 `null`，对其他错误仅调用 `logError(error)` 后也返回 `null`。如果计划文件因权限问题无法读取，用户只会看到 "No plan written yet"，而不会收到权限 denied 的明确提示。

3. **`renderToString` 的列宽问题**
   - `renderToString(display)` 未传入 `columns` 参数，会回退到 Ink 的默认宽度（通常是 80 列或终端实际宽度，取决于 `PassThrough` 流的 `columns` 属性）。对于宽屏终端用户，渲染出的计划内容可能会被不必要地换行。不过 `plan.tsx` 中随后将字符串传给 `onDone`，最终由消息组件渲染，影响相对有限。

4. **外部编辑器不可用时的用户体验**
   - 当 `getExternalEditor()` 返回 `undefined` 时，`editorName` 为 `undefined`，`PlanDisplay` 不会显示 `"/plan open"` 提示。用户可能不知道可以通过 `/plan open` 编辑计划。如果用户确实输入了 `/plan open`，`editFileInEditor` 会返回 `{ content: null }`（因为 `getExternalEditor()` 在内部再次检查并返回无编辑器），但 `plan.tsx` 中对 `result.error` 的判断会将其视为成功（`error` 未定义），导致 `onDone` 报告 "Opened plan in editor"，实际上编辑器并未打开。
   - **具体代码**：
     ```typescript
     const result = await editFileInEditor(planPath);
     if (result.error) {
       onDone(`Failed to open plan in editor: ${result.error}`);
     } else {
       onDone(`Opened plan in editor: ${planPath}`); // 这里在 editor 为 null 时也会执行
     }
     ```
   - 实际上 `editFileInEditor` 在 `getExternalEditor()` 返回 null 时返回 `{ content: null }`，`error` 为 undefined，所以确实会错误地报告成功。

5. **Plan Mode 与 Auto Mode 的复杂交互**
   - `prepareContextForPlanMode` 内部逻辑复杂，涉及 `TRANSCRIPT_CLASSIFIER` feature flag、`shouldPlanUseAutoMode()`、危险权限的剥离与恢复。如果用户从 `auto` 模式进入 plan 模式，再退出 plan 模式，权限恢复的正确性高度依赖 `prePlanMode` 字段和 `applyPermissionUpdate` 的实现。任何状态不一致都可能导致用户意外留在 `auto` 模式或丢失权限规则。

### 改进建议

1. **修复 `/plan open` 无编辑器时的错误报告**
   - 在 `editFileInEditor` 返回 `content: null` 且无 `error` 时（表示无可用编辑器或文件不存在），应明确向用户报告 "No external editor configured" 或 "Plan file not found"。

2. **统一 `editFileInEditor` 的同步/异步语义**
   - 建议要么将 `editFileInEditor` 改为 `async` 函数（如果内部未来需要异步 I/O），要么在 `plan.tsx` 中移除不必要的 `await`，以消除语义歧义。

3. **增强计划文件读取失败的反馈**
   - 在 `getPlan()` 返回 `null` 时，可以区分 "文件不存在" 和 "读取失败"，向用户展示更准确的提示信息。

4. **为 `renderToString` 传入终端宽度**
   - 如果 `context` 或全局状态中可获取当前终端列宽，可以将其传入 `renderToString(display, columns)`，使静态渲染的换行行为与 TUI 实际显示更一致。

5. **考虑将 `PlanDisplay` 的渲染逻辑与 `call` 函数解耦**
   - 当前 `PlanDisplay` 组件仅用于被 `renderToString` 转换为字符串。如果未来计划内容的展示需要更复杂的交互（如滚动、复制按钮），可能需要直接在 Ink 中渲染组件而不是先转字符串。不过目前这种 "JSX → String → onDone" 的模式是 `local-jsx` 命令与 `local` 命令输出对齐的标准做法，无需立即改动。
