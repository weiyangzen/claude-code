# StatusLine.tsx 研究文档

## 场景与职责

StatusLine 是 Claude Code CLI 的底部状态栏组件，负责在终端界面底部显示动态状态信息。它是全屏交互模式下的核心 UI 组件之一，为用户提供会话的实时状态概览。

### 核心职责
1. **状态信息展示**：显示当前模型、权限模式、工作目录、成本统计、上下文窗口使用情况等
2. **用户自定义状态栏**：支持通过 settings.json 配置自定义 shell 命令来生成状态栏内容
3. **性能优化**：通过防抖、缓存和记忆化减少不必要的渲染和计算
4. **安全控制**：遵守工作区信任设置，不信任环境下禁用自定义命令执行

### 使用场景
- 全屏交互模式 (CLAUDE_CODE_NO_FLICKER=1) 下的底部状态显示
- 需要实时监控会话成本、token 使用量、模型切换等场景
- 用户自定义工作流状态展示（通过 statusLine 配置）

---

## 功能点目的

### 1. 状态栏显示控制 (`statusLineShouldDisplay`)
- **目的**：决定是否渲染状态栏
- **逻辑**：
  - KAIROS 助手模式下隐藏（状态信息反映的是 REPL/daemon 进程而非 agent 子进程）
  - 需要 settings.statusLine 配置存在

### 2. 状态输入构建 (`buildStatusLineCommandInput`)
- **目的**：收集并结构化所有状态信息，供用户自定义命令使用
- **包含数据**：
  - **模型信息**：runtime 模型 ID 和显示名称
  - **工作区信息**：当前目录、项目目录、额外添加的目录
  - **版本信息**：MACRO.VERSION
  - **输出样式**：当前输出样式名称
  - **成本统计**：总成本、持续时间、API 调用时长、代码行变更统计
  - **上下文窗口**：输入/输出 token 数、窗口大小、使用百分比
  - **速率限制**：5小时和7天窗口的使用百分比和重置时间
  - **Vim 模式**：当前 Vim 模式状态（如启用）
  - **Agent 类型**：当前运行的 agent 名称
  - **远程模式**：远程会话 ID
  - **Worktree 信息**：git worktree 的名称、路径、分支等

### 3. 防抖更新机制 (`scheduleUpdate`, `doUpdate`)
- **目的**：避免频繁的状态更新导致性能问题
- **实现**：300ms 防抖延迟，取消进行中的请求

### 4. 条件重新计算
- **200k Token 检查**：仅在消息 ID 变化时重新计算是否超过 200k tokens
- **状态比较**：比较 messageId、permissionMode、vimMode、mainLoopModel 的变化

### 5. 信任与安全检查
- **工作区信任**：所有 hooks 需要用户接受工作区信任对话框
- **disableAllHooks**：支持通过设置完全禁用 hooks
- **通知机制**：当信任未接受时显示警告通知

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
type Props = {
  messagesRef: React.RefObject<Message[]>;  // 消息引用（只读）
  lastAssistantMessageId: string | null;     // 实际触发重新渲染的依赖
  vimMode?: VimMode;                         // 可选的 Vim 模式
};

// 状态栏命令输入（从 ../types/statusLine.js 导入）
interface StatusLineCommandInput {
  // 基础 Hook 输入（session_id, transcript_path, cwd 等）
  ...createBaseHookInput(),
  session_name?: string;
  model: { id: string; display_name: string };
  workspace: { current_dir: string; project_dir: string; added_dirs: string[] };
  version: string;
  output_style: { name: string };
  cost: {
    total_cost_usd: number;
    total_duration_ms: number;
    total_api_duration_ms: number;
    total_lines_added: number;
    total_lines_removed: number;
  };
  context_window: {
    total_input_tokens: number;
    total_output_tokens: number;
    context_window_size: number;
    current_usage: number;
    used_percentage: number;
    remaining_percentage: number;
  };
  exceeds_200k_tokens: boolean;
  rate_limits?: {
    five_hour?: { used_percentage: number; resets_at: string };
    seven_day?: { used_percentage: number; resets_at: string };
  };
  vim?: { mode: VimMode };
  agent?: { name: string };
  remote?: { session_id: string };
  worktree?: {
    name: string;
    path: string;
    branch: string;
    original_cwd: string;
    original_branch: string;
  };
}
```

### 关键流程

#### 1. 状态更新流程
```
useEffect 检测依赖变化 (lastAssistantMessageId, permissionMode, vimMode, mainLoopModel)
    ↓
scheduleUpdate() - 清除旧定时器，设置 300ms 新定时器
    ↓
doUpdate() - 实际执行更新
    ↓
1. 取消进行中的请求 (abortControllerRef.current?.abort())
2. 检查是否需要重新计算 exceeds200kTokens
3. 构建 StatusLineCommandInput
4. 调用 executeStatusLineCommand()
5. 更新 AppState 中的 statusLineText
```

#### 2. 命令执行流程 (`executeStatusLineCommand` in hooks.ts)
```
executeStatusLineCommand(statusInput, signal, timeoutMs, logResult)
    ↓
1. 检查 shouldDisableAllHooksIncludingManaged()
2. 检查 shouldSkipHookDueToTrust() - 安全控制
3. 根据 shouldAllowManagedHooksOnly() 获取对应 statusLine 配置
4. 验证 statusLine.type === 'command'
    ↓
execCommandHook() - 执行实际的 shell 命令
    ↓
返回 stdout 作为状态栏文本
```

#### 3. 渲染流程
```
StatusLine (memo 包裹)
    ↓
从 useAppState 获取 statusLineText
    ↓
从 settings 获取 padding 配置
    ↓
渲染 Box 容器
    ↓
条件渲染：
  - statusLineText 存在：显示 Ansi 格式的文本
  - 全屏模式且文本为空：显示空格占位（保持高度稳定）
  - 其他情况：null
```

### 性能优化策略

1. **React.memo**：避免父组件 PromptInputFooter 的频繁重新渲染影响 StatusLine
2. **useRef 模式**：settings、vimMode、permissionMode、addedDirs、mainLoopModel 都通过 ref 保持最新值，避免闭包问题
3. **previousStateRef**：缓存上一次的状态，避免重复计算
4. **防抖机制**：300ms 防抖避免频繁触发命令执行
5. **AbortController**：取消过时的请求，避免竞态条件
6. **条件更新**：setAppState 中比较新旧值，相同则不更新

---

## 关键代码路径与文件引用

### 主要文件
- `/src/components/StatusLine.tsx` - 主组件实现

### 依赖文件
| 文件路径 | 用途 |
|---------|------|
| `src/utils/hooks.ts` | `executeStatusLineCommand`, `createBaseHookInput` |
| `src/state/AppState.ts` | `useAppState`, `useSetAppState` |
| `src/hooks/useSettings.ts` | `useSettings`, `ReadonlySettings` |
| `src/hooks/useMainLoopModel.ts` | `useMainLoopModel` |
| `src/context/notifications.ts` | `useNotifications` |
| `src/types/statusLine.ts` | `StatusLineCommandInput` 类型定义 |
| `src/types/message.ts` | `Message` 类型 |
| `src/types/textInputTypes.ts` | `VimMode` 类型 |
| `src/utils/permissions/PermissionMode.ts` | `PermissionMode` 类型 |

### 工具函数依赖
| 文件路径 | 用途 |
|---------|------|
| `src/bootstrap/state.ts` | `getIsRemoteMode`, `getKairosActive`, `getMainThreadAgentType`, `getOriginalCwd`, `getSdkBetas`, `getSessionId` |
| `src/cost-tracker.ts` | 成本统计相关函数 |
| `src/services/claudeAiLimits.ts` | `getRawUtilization` - 获取 API 速率限制 |
| `src/utils/context.ts` | `calculateContextPercentages`, `getContextWindowForModel` |
| `src/utils/cwd.ts` | `getCwd` |
| `src/utils/sessionStorage.ts` | `getCurrentSessionTitle`, `getTranscriptPathForSession` |
| `src/utils/tokens.ts` | `doesMostRecentAssistantMessageExceed200k`, `getCurrentUsage` |
| `src/utils/worktree.ts` | `getCurrentWorktreeSession` |
| `src/utils/config.ts` | `checkHasTrustDialogAccepted` |
| `src/utils/messages.ts` | `getLastAssistantMessage` |
| `src/utils/model/model.ts` | `getRuntimeMainLoopModel`, `renderModelName` |
| `src/components/PromptInput/utils.ts` | `isVimModeEnabled` |
| `src/constants/outputStyles.ts` | `DEFAULT_OUTPUT_STYLE_NAME` |
| `src/utils/fullscreen.ts` | `isFullscreenEnvEnabled` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/services/analytics/index.ts` | `logEvent` |
| `src/ink.ts` | `Ansi`, `Box`, `Text` - Ink 渲染组件 |

### 调用方
- `src/components/PromptInput/PromptInputFooter.tsx` - 作为页脚组件嵌入

---

## 依赖与外部交互

### 状态管理
- **AppState**：读取 `toolPermissionContext.mode`, `toolPermissionContext.additionalWorkingDirectories`, `statusLineText`；写入 `statusLineText`
- **Settings**：读取 `statusLine.command`, `statusLine.padding`, `outputStyle`, `syntaxHighlightingDisabled`, `disableAllHooks`

### 外部系统交互
- **Shell 执行**：通过 `executeStatusLineCommand` → `execCommandHook` 执行用户配置的 shell 命令
- **Analytics**：挂载时记录 `tengu_status_line_mount` 事件
- **通知系统**：信任未接受时显示通知

### 环境变量依赖
- `CLAUDE_CODE_NO_FLICKER` - 控制全屏模式
- `CLAUDE_CODE_SHELL_PREFIX` - shell 命令前缀（hooks.ts 中处理）

---

## 风险、边界与改进建议

### 已知风险

1. **安全风险 - 代码注入**
   - 状态栏命令执行用户配置的任意 shell 代码
   - **缓解措施**：`shouldSkipHookDueToTrust()` 强制要求工作区信任
   - **潜在问题**：注释中提到 "executeStatusLineCommand returns undefined when trust is blocked — statusLineText stays undefined forever, user sees nothing, and tengu_status_line_mount above fires anyway so telemetry looks fine" - 存在 telemetry 与实际行为不一致的问题

2. **竞态条件**
   - 快速连续的状态变化可能导致过时的结果覆盖新结果
   - **缓解措施**：使用 AbortController 取消进行中的请求

3. **性能风险**
   - 每次消息变化都触发 200k token 检查，虽然已优化为仅在 messageId 变化时执行，但仍是大计算量操作
   - shell 命令执行有 5 秒超时，但大量并发可能影响性能

4. **内存泄漏**
   - `logNextResultRef` 和 `isFirstSettingsRender` 等 ref 在组件卸载后不再使用，但不会导致泄漏
   - `debounceTimerRef` 和 `abortControllerRef` 在卸载时正确清理

### 边界情况

1. **KAIROS 模式**：状态栏完全隐藏，因为状态信息反映的是错误的进程
2. **非交互模式**：信任检查跳过，允许执行
3. **空状态栏文本**：全屏模式下显示空格占位，防止布局抖动
4. **非常窄终端**：sliceAnsi 和 RawAnsi 处理宽度为 0 或负数的情况
5. **Settings 热重载**：`statusLineCommand` 变化时触发重新执行并记录日志

### 改进建议

1. **类型安全**
   - `StatusLineCommandInput` 类型定义文件 `src/types/statusLine.ts` 似乎不存在于源码中，可能是构建时生成。建议明确类型定义位置或添加注释说明

2. **错误处理**
   - 当前 `doUpdate` 中的 catch 块为空，建议至少记录错误详情
   - 命令执行失败时用户无感知，考虑添加错误状态显示

3. **可测试性**
   - 组件与大量外部系统耦合，建议提取纯函数逻辑（如 `buildStatusLineCommandInput`）以便单元测试
   - 时间相关的防抖逻辑难以测试，考虑使用可注入的定时器

4. **性能优化**
   - 考虑使用 useMemo 缓存 `buildStatusLineCommandInput` 的结果
   - 200k token 检查可以移到 Web Worker 或异步线程避免阻塞主线程

5. **用户体验**
   - 状态栏命令执行中无加载指示，用户可能不知道正在更新
   - 命令超时时无反馈，建议添加超时提示

6. **代码组织**
   - `buildStatusLineCommandInput` 函数较长（约 90 行），可以拆分为多个小函数
   - 大量的条件展开操作（`...(condition && { prop })`）可读性较差，考虑使用对象合并工具函数
