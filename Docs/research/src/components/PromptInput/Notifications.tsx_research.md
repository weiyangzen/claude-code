# Notifications.tsx 研究文档

## 场景与职责

`Notifications.tsx` 是 Claude Code CLI 中 PromptInput 组件子模块的核心通知系统，负责**底部状态栏的综合信息展示**。它聚合了多种系统状态、用户提示和警告信息，在终端界面的底部区域向用户展示关键反馈。

### 使用场景
- **API 密钥状态提示**：登录状态、密钥验证失败
- **Token 使用警告**：接近上下文窗口限制时提醒
- **IDE 连接状态**：显示与 IDE 的集成状态
- **自动更新通知**：新版本可用或更新中
- **语音模式指示**：录音/处理状态
- **环境钩子反馈**：文件变更等事件通知
- **超额使用提示**：超出配额时的提醒

## 功能点目的

### 1. 通知队列管理
- 通过 `useNotifications` hook 管理通知队列
- 支持优先级（low/medium/high/immediate）
- 自动超时和队列处理

### 2. 多维度状态聚合
- **认证状态**：API 密钥验证状态
- **Token 使用**：当前 token 计数和警告阈值
- **IDE 集成**：IDE 连接状态和选择信息
- **自动更新**：更新进度和结果
- **语音模式**：录音/处理状态指示
- **内存使用**：内存占用指示

### 3. 动态提示管理
- 外部编辑器提示（`external-editor-hint`）
- 环境钩子通知（`env-hook`）
- API Key Helper 慢响应警告

### 4. 视觉层次控制
- 根据 `isNarrow` 属性调整对齐方式
- 使用 SentryErrorBoundary 包裹，防止通知区域崩溃影响主应用

## 具体技术实现

### 关键数据结构

```typescript
type Props = {
  apiKeyStatus: VerificationStatus;           // API 密钥验证状态
  autoUpdaterResult: AutoUpdaterResult | null; // 自动更新结果
  isAutoUpdating: boolean;                    // 是否正在更新
  debug: boolean;                             // 调试模式
  verbose: boolean;                           // 详细模式
  messages: Message[];                        // 消息列表（用于 token 计算）
  onAutoUpdaterResult: (result: AutoUpdaterResult) => void;
  onChangeIsUpdating: (isUpdating: boolean) => void;
  ideSelection: IDESelection | undefined;     // IDE 选择信息
  mcpClients?: MCPServerConnection[];         // MCP 客户端连接
  isInputWrapped?: boolean;                   // 输入是否换行
  isNarrow?: boolean;                         // 是否窄屏模式
};

// 通知数据结构
interface Notification {
  key: string;
  priority: 'low' | 'medium' | 'high' | 'immediate';
  timeoutMs?: number;
  text?: string;
  jsx?: React.ReactNode;
  color?: keyof Theme;
  invalidates?: string[];
  fold?: (accumulator: Notification, incoming: Notification) => Notification;
}
```

### 关键流程

#### 1. Token 使用监控流程

```tsx
// 计算 token 使用量
const messagesForTokenCount = getMessagesAfterCompactBoundary(messages);
const tokenUsage = tokenCountFromLastAPIResponse(messagesForTokenCount);

// 计算警告状态
const tokenWarningState = calculateTokenWarningState(tokenUsage, mainLoopModel);
const isShowingCompactMessage = tokenWarningState.isAboveWarningThreshold;
```

#### 2. 环境钩子通知设置

```tsx
useEffect(() => {
  setEnvHookNotifier((text, isError) => {
    addNotification({
      key: "env-hook",
      text,
      color: isError ? "error" : undefined,
      priority: isError ? "medium" : "low",
      timeoutMs: isError ? 8000 : 5000
    });
  });
  return () => setEnvHookNotifier(null);
}, [addNotification]);
```

#### 3. 外部编辑器提示管理

```tsx
useEffect(() => {
  if (shouldShowExternalEditorHint && editor) {
    logEvent("tengu_external_editor_hint_shown", {});
    addNotification({
      key: "external-editor-hint",
      jsx: <Text dimColor>
        <ConfigurableShortcutHint 
          action="chat:externalEditor" 
          context="Chat" 
          fallback="ctrl+g" 
          description={`edit in ${toIDEDisplayName(editor)}`} 
        />
      </Text>,
      priority: "immediate",
      timeoutMs: 5000
    });
  } else {
    removeNotification("external-editor-hint");
  }
}, [shouldShowExternalEditorHint, editor, addNotification, removeNotification]);
```

#### 4. 通知内容渲染逻辑

```tsx
function NotificationContent({...props}): ReactNode {
  // API Key Helper 慢响应检测
  const [apiKeyHelperSlow, setApiKeyHelperSlow] = useState<string | null>(null);
  useEffect(() => {
    if (!getConfiguredApiKeyHelper()) return;
    const interval = setInterval(() => {
      const ms = getApiKeyHelperElapsedMs();
      const next = ms >= 10_000 ? formatDuration(ms) : null;
      setSlow(prev => next === prev ? prev : next);
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  // 语音状态（VOICE_MODE 构建）
  const voiceState = feature('VOICE_MODE') ? useVoiceState(s => s.voiceState) : 'idle';
  const voiceEnabled = feature('VOICE_MODE') ? useVoiceEnabled() : false;
  const voiceError = feature('VOICE_MODE') ? useVoiceState(s => s.voiceError) : null;

  // 语音活跃时只显示语音指示器
  if (feature('VOICE_MODE') && voiceEnabled && 
      (voiceState === 'recording' || voiceState === 'processing')) {
    return <VoiceIndicator voiceState={voiceState} />;
  }

  return (
    <>
      <IdeStatusIndicator ideSelection={ideSelection} mcpClients={mcpClients} />
      {/* 当前通知 */}
      {notifications.current && ('jsx' in notifications.current 
        ? <Text wrap="truncate" key={notifications.current.key}>{notifications.current.jsx}</Text>
        : <Text color={notifications.current.color} dimColor={!notifications.current.color} wrap="truncate">
            {notifications.current.text}
          </Text>
      )}
      {/* 超额使用提示 */}
      {isInOverageMode && !isTeamOrEnterprise && (
        <Box><Text dimColor wrap="truncate">Now using extra usage</Text></Box>
      )}
      {/* API Key Helper 慢响应 */}
      {apiKeyHelperSlow && (
        <Box>
          <Text color="warning" wrap="truncate">apiKeyHelper is taking a while </Text>
          <Text dimColor wrap="truncate">({apiKeyHelperSlow})</Text>
        </Box>
      )}
      {/* 认证错误 */}
      {(apiKeyStatus === 'invalid' || apiKeyStatus === 'missing') && (
        <Box>
          <Text color="error" wrap="truncate">
            {isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) 
              ? 'Authentication error · Try again' 
              : 'Not logged in · Run /login'}
          </Text>
        </Box>
      )}
      {/* 调试模式 */}
      {debug && <Box><Text color="warning" wrap="truncate">Debug mode</Text></Box>}
      {/* Token 计数（详细模式） */}
      {apiKeyStatus !== 'invalid' && apiKeyStatus !== 'missing' && verbose && (
        <Box><Text dimColor wrap="truncate">{tokenUsage} tokens</Text></Box>
      )}
      {/* Token 警告 */}
      {!isBriefOnly && <TokenWarning tokenUsage={tokenUsage} model={mainLoopModel} />}
      {/* 自动更新包装器 */}
      {shouldShowAutoUpdater && <AutoUpdaterWrapper ... />}
      {/* 语音错误 */}
      {feature('VOICE_MODE') ? voiceEnabled && voiceError && (
        <Box><Text color="error" wrap="truncate">{voiceError}</Text></Box>
      ) : null}
      <MemoryUsageIndicator />
      <SandboxPromptFooterHint />
    </>
  );
}
```

### 特性开关（Feature Flags）

组件大量使用 `feature()` 进行条件编译：

```typescript
// VOICE_MODE：语音功能
const VoiceIndicator = feature('VOICE_MODE') 
  ? require('./VoiceIndicator.js').VoiceIndicator 
  : () => null;

// KAIROS / KAIROS_BRIEF：简化模式
const isBriefOnly = feature('KAIROS') || feature('KAIROS_BRIEF')
  ? useAppState(s => s.isBriefOnly)
  : false;
```

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| React Compiler Runtime | `"react/compiler-runtime"` | 渲染优化 |
| bun:bundle feature | `'bun:bundle'` | 特性开关 |
| useNotifications | `src/context/notifications.js` | 通知队列管理 |
| logEvent | `src/services/analytics/index.js` | 分析事件上报 |
| useAppState | `src/state/AppState.js` | 全局状态访问 |
| useVoiceState | `../../context/voice.js` | 语音状态 |
| useIdeConnectionStatus | `../../hooks/useIdeConnectionStatus.js` | IDE 连接状态 |
| useMainLoopModel | `../../hooks/useMainLoopModel.js` | 当前模型 |
| useVoiceEnabled | `../../hooks/useVoiceEnabled.js` | 语音启用状态 |
| useClaudeAiLimits | `../../services/claudeAiLimitsHook.js` | AI 使用限制 |
| calculateTokenWarningState | `../../services/compact/autoCompact.js` | Token 警告计算 |
| Box, Text | `../../ink.js` | UI 组件 |

### 子组件

| 组件 | 路径 | 用途 |
|------|------|------|
| AutoUpdaterWrapper | `../AutoUpdaterWrapper.js` | 自动更新 UI |
| ConfigurableShortcutHint | `../ConfigurableShortcutHint.js` | 快捷键提示 |
| IdeStatusIndicator | `../IdeStatusIndicator.js` | IDE 状态指示 |
| MemoryUsageIndicator | `../MemoryUsageIndicator.js` | 内存使用指示 |
| SentryErrorBoundary | `../SentryErrorBoundary.js` | 错误边界 |
| TokenWarning | `../TokenWarning.js` | Token 警告 |
| SandboxPromptFooterHint | `./SandboxPromptFooterHint.js` | 沙箱提示 |
| VoiceIndicator | `./VoiceIndicator.js` | 语音状态指示 |

### 调用方

- **PromptInputFooter.tsx** (`src/components/PromptInput/PromptInputFooter.tsx`)
  - 作为底部状态栏的核心组件被调用
  - 传递所有必要的 props

### 工具函数

| 函数 | 路径 | 用途 |
|------|------|------|
| getApiKeyHelperElapsedMs | `../../utils/auth.js` | 获取 API Key Helper 耗时 |
| getConfiguredApiKeyHelper | `../../utils/auth.js` | 获取配置的 Helper |
| getSubscriptionType | `../../utils/auth.js` | 获取订阅类型 |
| getExternalEditor | `../../utils/editor.js` | 获取外部编辑器 |
| isEnvTruthy | `../../utils/envUtils.js` | 环境变量检查 |
| formatDuration | `../../utils/format.js` | 时长格式化 |
| setEnvHookNotifier | `../../utils/hooks/fileChangedWatcher.js` | 设置环境钩子通知器 |
| toIDEDisplayName | `../../utils/ide.js` | IDE 名称转换 |
| getMessagesAfterCompactBoundary | `../../utils/messages.js` | 获取压缩边界后的消息 |
| tokenCountFromLastAPIResponse | `../../utils/tokens.js` | 从 API 响应获取 token 数 |

## 风险、边界与改进建议

### 潜在风险

1. **特性开关复杂性**
   - 大量使用 `feature()` 条件编译，代码分支多
   - 不同构建配置下行为差异大，测试覆盖困难
   - 建议：建立特性矩阵测试，确保各组合正常工作

2. **Hook 条件调用**
   - 使用 `// biome-ignore lint/correctness/useHookAtTopLevel` 注释
   - `feature()` 被假定为编译时常量，但如果运行时变化会导致 Hook 规则违反
   - 建议：确保 `feature()` 在构建时完全内联

3. **定时器管理**
   - 使用模块级变量 `currentTimeoutId` 管理通知超时
   - 可能存在内存泄漏风险（虽然代码中做了清理）
   - 建议：使用 ref 或更健壮的定时器管理方案

4. **Token 计算性能**
   - `getMessagesAfterCompactBoundary` 和 `tokenCountFromLastAPIResponse` 可能在每次渲染时执行
   - 消息量大时可能影响性能
   - 建议：使用 memoization 优化

### 边界情况

1. **通知优先级冲突**
   - `immediate` 优先级会中断当前通知
   - 如果频繁收到 immediate 通知，低优先级通知可能永远无法显示

2. **窄屏模式**
   - `isNarrow` 影响对齐方式
   - 在极窄终端下可能出现布局问题

3. **语音模式覆盖**
   - 语音活跃时会完全替换其他通知
   - 重要通知可能在语音处理期间被隐藏

4. **API Key Helper 检测**
   - 仅当配置了 Helper 时才启动检测
   - 10秒阈值是硬编码的

### 改进建议

1. **通知持久化**
   - 重要通知（如错误）应该持久显示直到用户确认
   - 当前所有通知都有超时，可能错过关键信息

2. **通知历史**
   - 添加通知历史功能，用户可以查看错过的通知
   - 可通过快捷键或命令访问

3. **性能优化**
   - 使用 `useMemo` 缓存 token 计算结果
   - 优化通知队列处理，减少不必要的重渲染

4. **可配置性**
   - 允许用户配置通知超时时间
   - 允许禁用特定类型的通知

5. **更好的错误处理**
   - 当前使用 SentryErrorBoundary 包裹整个通知区域
   - 可以考虑为单个通知组件添加错误边界，隔离故障

6. **国际化**
   - 当前所有文本都是英文
   - 添加 i18n 支持

7. **测试覆盖**
   - 添加单元测试覆盖各种通知场景
   - 测试特性开关的不同组合

### 相关文件引用

- 实现文件：`src/components/PromptInput/Notifications.tsx`
- 调用方：`src/components/PromptInput/PromptInputFooter.tsx`
- 通知上下文：`src/context/notifications.tsx`
- 状态管理：`src/state/AppState.tsx`
- Token 计算：`src/services/compact/autoCompact.ts`
- 语音指示器：`src/components/PromptInput/VoiceIndicator.tsx`
