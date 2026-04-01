# FeedbackSurvey.tsx 研究文档

## 场景与职责

`FeedbackSurvey.tsx` 是 Claude Code CLI 的反馈调查系统的主 UI 组件，负责根据当前调查状态渲染不同的反馈界面。它是反馈调查功能的核心展示层，协调多个子组件（FeedbackSurveyView、TranscriptSharePrompt）以及自定义 Hook（useDebouncedDigitInput）来提供完整的用户反馈体验。

该组件主要服务于以下场景：
1. **会话质量调查**：定期询问用户对 Claude 会话质量的评价
2. **转录分享提示**：在用户给出好评或差评后，询问是否愿意分享会话转录
3. **感谢页面**：用户完成反馈后显示感谢信息，并提供后续操作指引

## 功能点目的

### 1. 状态驱动的条件渲染
组件根据 `state` 属性决定渲染内容：
- `closed`: 不渲染任何内容
- `open`: 显示主调查问卷（FeedbackSurveyView）
- `thanks`: 显示感谢页面，包含可选的后续反馈引导
- `transcript_prompt`: 显示转录分享提示（TranscriptSharePrompt）
- `submitting`: 显示转录分享中状态
- `submitted`: 显示转录分享成功确认

### 2. 输入验证与过滤
- 通过 `isValidResponseInput` 验证用户输入是否为有效数字（0-3）
- 对转录分享提示的输入进行验证（1-3）
- 无效输入时组件返回 null，避免干扰用户正常输入

### 3. 感谢页面的智能引导
- 好评（good）后显示可选的详细反馈入口（按 1 键）
- 差评（bad）后提示使用 `/issue` 报告问题
- 其他情况提示使用 `/feedback` 分享详细反馈

### 4. 分析事件追踪
在感谢页面显示时记录 `followup_accepted` 事件，用于分析用户参与后续反馈的意愿。

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
type Props = {
  state: 'closed' | 'open' | 'thanks' | 'transcript_prompt' | 'submitting' | 'submitted';
  lastResponse: FeedbackSurveyResponse | null;
  handleSelect: (selected: FeedbackSurveyResponse) => void;
  handleTranscriptSelect?: (selected: TranscriptShareResponse) => void;
  inputValue: string;
  setInputValue: (value: string) => void;
  onRequestFeedback?: () => void;
  message?: string;
};

// 感谢页面 Props
type ThanksProps = {
  lastResponse: FeedbackSurveyResponse | null;
  inputValue: string;
  setInputValue: (value: string) => void;
  onRequestFeedback?: () => void;
};
```

### 关键流程

#### 1. 主渲染流程
```
state === 'closed' → return null
state === 'thanks' → render FeedbackSurveyThanks
state === 'submitted' → render success message
state === 'submitting' → render loading message
state === 'transcript_prompt' → render TranscriptSharePrompt
inputValue 无效 → return null
其他情况 → render FeedbackSurveyView
```

#### 2. 感谢页面交互流程
```
显示条件: onRequestFeedback 存在且 lastResponse === 'good'
验证函数: isFollowUpDigit (仅接受 '1')
触发动作: 
  - 记录 analytics 事件
  - 调用 onRequestFeedback()
```

### React Compiler 优化
代码使用 React Compiler 自动优化，通过 `_c` 函数创建 memoization cache：
- 使用 `$[n]` 访问缓存槽位
- 使用 `Symbol.for("react.memo_cache_sentinel")` 作为未初始化标记
- 条件渲染路径均有独立的缓存槽位管理

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `./FeedbackSurveyView.js` | 主调查问卷 UI 组件 |
| `./TranscriptSharePrompt.js` | 转录分享提示 UI 组件 |
| `./useDebouncedDigitInput.js` | 防抖数字输入处理 Hook |
| `./utils.js` | FeedbackSurveyResponse 类型定义 |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `src/services/analytics/index.js` | 分析事件记录 (logEvent) |
| `src/ink.js` | Ink 渲染组件 (Box, Text) |

### 关键代码片段

#### 状态路由逻辑
```typescript
// 行 32-102
if (state === "closed") return null;
if (state === "thanks") return <FeedbackSurveyThanks ... />;
if (state === "submitted") return <Box ...>Thanks for sharing...</Box>;
if (state === "submitting") return <Box ...>Sharing transcript...</Box>;
if (state === "transcript_prompt") {
  if (!handleTranscriptSelect) return null;
  if (inputValue && !["1", "2", "3"].includes(inputValue)) return null;
  return <TranscriptSharePrompt ... />;
}
if (inputValue && !isValidResponseInput(inputValue)) return null;
return <FeedbackSurveyView ... />;
```

#### 感谢页面防抖输入
```typescript
// 行 136-154
useDebouncedDigitInput({
  inputValue,
  setInputValue,
  isValidDigit: isFollowUpDigit,
  enabled: Boolean(showFollowUp),
  once: true,
  onDigit: () => {
    logEvent("tengu_feedback_survey_event", { event_type: "followup_accepted", ... });
    onRequestFeedback?.();
  }
});
```

#### 反馈命令选择
```typescript
// 行 155
const feedbackCommand = false ? "/issue" : "/feedback";
// 注：当前硬编码为 "/feedback"，预留条件分支可能用于未来扩展
```

## 依赖与外部交互

### 输入依赖
1. **状态管理**: 通过 Props 接收来自 `useSurveyState` 的状态和控制函数
2. **输入值**: 通过 `inputValue` 和 `setInputValue` 与父组件共享输入状态
3. **回调函数**: 
   - `handleSelect`: 处理调查选项选择
   - `handleTranscriptSelect`: 处理转录分享选择
   - `onRequestFeedback`: 处理后续反馈请求

### 输出交互
1. **Analytics**: 通过 `logEvent` 记录用户交互事件
   - 事件名: `tengu_feedback_survey_event`
   - 事件类型: `followup_accepted`
2. **UI 渲染**: 使用 Ink 组件渲染终端 UI

### 类型依赖关系
```
FeedbackSurvey.tsx
  ├── FeedbackSurveyResponse (from ./utils.js)
  ├── TranscriptShareResponse (from ./TranscriptSharePrompt.js)
  └── AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS (from analytics)
```

## 风险、边界与改进建议

### 已知风险

1. **硬编码反馈命令**
   - 行 155: `const feedbackCommand = false ? "/issue" : "/feedback";`
   - 条件永远为 false，"/issue" 分支不可达，可能是遗留代码或未完成的功能

2. **输入验证的提前返回**
   - 行 73-75, 88-90: 无效输入时直接返回 null
   - 这可能导致用户输入数字后界面突然消失，体验不够友好

3. **转录分享回调缺失处理**
   - 行 70-72: `handleTranscriptSelect` 不存在时返回 null
   - 静默失败可能导致用户困惑

### 边界情况

1. **空输入处理**: 当 `inputValue` 为空字符串时，组件正常渲染调查界面
2. **状态不一致**: 如果 `state` 为 `transcript_prompt` 但 `handleTranscriptSelect` 未提供，组件静默返回 null
3. **并发输入**: 使用 `useDebouncedDigitInput` 防止用户在输入列表项（如 "1. First item"）时意外触发选择

### 改进建议

1. **移除或启用条件分支**
   ```typescript
   // 建议：根据实际业务逻辑决定
   const feedbackCommand = lastResponse === 'bad' ? "/issue" : "/feedback";
   // 或完全移除死代码
   const feedbackCommand = "/feedback";
   ```

2. **增强无效输入反馈**
   ```typescript
   // 当前：直接返回 null
   if (inputValue && !isValidResponseInput(inputValue)) return null;
   
   // 建议：提供视觉反馈
   // 在 FeedbackSurveyView 内部处理无效输入提示
   ```

3. **添加错误边界**
   - 为转录分享回调缺失添加警告日志
   - 在开发模式下抛出错误以便及时发现集成问题

4. **优化缓存策略**
   - 当前 React Compiler 缓存槽位较多（16 个），可考虑拆分组件减少复杂度
   - `FeedbackSurveyThanks` 可作为独立组件文件提取

5. **类型安全增强**
   - `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型使用繁琐
   - 考虑为常用事件类型创建辅助函数减少重复转换

### 测试建议

1. **状态转换测试**: 验证所有状态组合的正确渲染
2. **输入防抖测试**: 验证 `useDebouncedDigitInput` 的防抖行为
3. **回调触发测试**: 验证 `onRequestFeedback` 仅在好评后且按 1 时触发
4. **边界条件测试**: 
   - 空 `inputValue`
   - 无效 `inputValue`（如字母、符号）
   - 缺失可选回调
