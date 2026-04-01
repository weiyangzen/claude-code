# useFeedbackSurvey.tsx 研究文档

## 场景与职责

`useFeedbackSurvey.tsx` 是 Claude Code CLI 反馈调查系统的核心状态管理 Hook，负责控制调查问卷的显示时机、处理用户响应、管理转录分享流程。它是整个反馈调查功能的业务逻辑中枢，协调多个子系统和外部服务。

该 Hook 的主要职责：
1. **智能显示控制**: 基于多种条件（时间、消息数、概率等）决定是否显示调查
2. **状态管理**: 管理调查的完整生命周期（closed → open → thanks/transcript_prompt → closed）
3. **响应处理**: 处理用户评分选择，记录分析事件
4. **转录分享协调**: 在适当的时候询问并处理转录分享
5. **跨会话持久化**: 保存调查状态到全局配置，实现跨会话的显示频率控制

## 功能点目的

### 1. 智能显示策略
通过多维度条件控制调查显示频率，避免过度打扰用户：
- **时间控制**: 会话开始 10 分钟后首次显示，之后至少间隔 1 小时
- **消息数控制**: 至少 5 条用户消息后首次显示，之后至少间隔 10 条消息
- **全局控制**: 跨所有会话至少间隔 ~27.7 小时
- **概率控制**: 默认 0.5% 的显示概率，可通过配置调整

### 2. 动态配置支持
通过 GrowthBook 动态配置实现无需发版的策略调整：
- `tengu_feedback_survey_config`: 主调查配置
- `tengu_bad_survey_transcript_ask_config`: 差评后转录分享概率
- `tengu_good_survey_transcript_ask_config`: 好评后转录分享概率

### 3. 分析事件追踪
完整的用户交互追踪：
- `appeared`: 调查展示
- `responded`: 用户响应
- `transcript_prompt_appeared`: 转录分享提示展示
- `transcript_share_yes/no/dont_ask_again`: 转录分享选择
- `transcript_share_submitted/failed`: 转录分享结果

### 4. 组织策略合规
检查并遵守组织级策略限制：
- `allow_product_feedback`: 检查组织是否允许产品反馈
- ZDR (Zero Data Retention) 组织禁用反馈收集

## 具体技术实现

### 关键数据结构

```typescript
// 调查配置
interface FeedbackSurveyConfig {
  minTimeBeforeFeedbackMs: number;        // 首次显示前最小时间（默认 10 分钟）
  minTimeBetweenFeedbackMs: number;       // 显示间隔最小时间（默认 1 小时）
  minTimeBetweenGlobalFeedbackMs: number; // 全局显示间隔（默认 ~27.7 小时）
  minUserTurnsBeforeFeedback: number;     // 首次显示前最小用户消息数（默认 5）
  minUserTurnsBetweenFeedback: number;    // 显示间隔最小用户消息数（默认 10）
  hideThanksAfterMs: number;              // 感谢页面显示时长（默认 3 秒）
  onForModels: string[];                  // 允许显示的模型列表
  probability: number;                    // 显示概率（默认 0.005）
}

// 转录分享配置
interface TranscriptAskConfig {
  probability: number;  // 转录分享提示显示概率
}

// 默认配置
const DEFAULT_FEEDBACK_SURVEY_CONFIG: FeedbackSurveyConfig = {
  minTimeBeforeFeedbackMs: 600000,
  minTimeBetweenFeedbackMs: 3600000,
  minTimeBetweenGlobalFeedbackMs: 100000000,
  minUserTurnsBeforeFeedback: 5,
  minUserTurnsBetweenFeedback: 10,
  hideThanksAfterMs: 3000,
  onForModels: ['*'],
  probability: 0.005
};

// 返回值
interface UseFeedbackSurveyReturn {
  state: 'closed' | 'open' | 'thanks' | 'transcript_prompt' | 'submitting' | 'submitted';
  lastResponse: FeedbackSurveyResponse | null;
  handleSelect: (selected: FeedbackSurveyResponse) => boolean;
  handleTranscriptSelect: (selected: TranscriptShareResponse) => void;
}
```

### 关键流程

#### 1. 初始化流程
```
useFeedbackSurvey(messages, isLoading, submitCount, surveyType, hasActivePrompt)
  ├── 初始化 refs: lastAssistantMessageIdRef, sessionStartTime, submitCountAtSessionStart
  ├── 初始化 state: feedbackSurvey (timeLastShown, submitCountAtLastAppearance)
  ├── 获取动态配置: config, badTranscriptAskConfig, goodTranscriptAskConfig
  ├── 获取用户设置: settingsRate
  ├── 初始化概率控制 refs: probabilityPassedRef, lastEligibleSubmitCountRef
  └── 使用 useSurveyState 管理调查状态
```

#### 2. 显示决策流程（shouldOpen）
```
计算 shouldOpen
  ├── 状态检查: state !== 'closed' → false
  ├── 加载检查: isLoading → false
  ├── 活跃提示检查: hasActivePrompt → false
  ├── 强制显示检查: CLAUDE_FORCE_DISPLAY_SURVEY → true
  ├── 模型允许检查: isModelAllowed
  ├── 环境禁用检查: CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY → false
  ├── 功能禁用检查: isFeedbackSurveyDisabled() → false
  ├── 组织策略检查: isPolicyAllowed('allow_product_feedback') → true
  ├── 会话本地节奏检查:
  │     ├── 首次显示: 检查时间 >= 10min && 消息数 >= 5
  │     └── 后续显示: 检查时间 >= 1h && 消息数 >= 10
  ├── 概率检查: Math.random() <= probability
  └── 全局节奏检查: 检查跨会话时间 >= ~27.7h
```

#### 3. 转录分享决策流程
```
shouldShowTranscriptPrompt(selected)
  ├── 仅好评/差评触发: selected === 'bad' || selected === 'good'
  ├── 用户未选择不再询问: !transcriptShareDismissed
  ├── 组织策略允许: isPolicyAllowed('allow_product_feedback')
  └── 概率检查: Math.random() <= config.probability
```

#### 4. 响应处理流程
```
handleSelect(selected)
  ├── 记录分析事件: tengu_feedback_survey_event/responded
  ├── 更新最后显示时间
  ├── 检查是否显示转录分享提示
  │     ├── 是 → 状态变为 transcript_prompt
  │     └── 否 → 显示感谢页面
  └── 返回是否显示转录分享

handleTranscriptSelect(selected)
  ├── 记录分析事件
  ├── 处理 dont_ask_again: 保存全局配置
  ├── 处理 yes: 调用 submitTranscriptShare
  │     ├── 成功 → 状态变为 submitted
  │     └── 失败 → 显示感谢页面
  └── 处理 no: 显示感谢页面
```

### 概率控制机制

```typescript
// 行 72-73, 265-270
const probabilityPassedRef = useRef(false);
const lastEligibleSubmitCountRef = useRef<number | null>(null);

// 只在 submitCount 变化时重新计算概率
if (lastEligibleSubmitCountRef.current !== submitCount) {
  lastEligibleSubmitCountRef.current = submitCount;
  probabilityPassedRef.current = Math.random() <= (settingsRate ?? config.probability);
}
```

该机制确保：
- 每个符合条件的提交只计算一次概率
- 避免每次渲染重新随机导致几乎必然触发
- 用户设置（settingsRate）优先于动态配置

### 跨会话持久化

```typescript
// 行 85-92
if (getGlobalConfig().feedbackSurveyState?.lastShownTime !== timestamp) {
  saveGlobalConfig(current => ({
    ...current,
    feedbackSurveyState: { lastShownTime: timestamp }
  }));
}
```

保存到全局配置：
- 路径: `~/.claude/config.json`
- 字段: `feedbackSurveyState.lastShownTime`

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `./submitTranscriptShare.js` | 转录分享提交函数 |
| `./TranscriptSharePrompt.js` | TranscriptShareResponse 类型 |
| `./useSurveyState.js` | 调查状态管理 Hook |
| `./utils.js` | FeedbackSurveyResponse 和 FeedbackSurveyType 类型 |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `src/hooks/useDynamicConfig.js` | 动态配置获取 |
| `src/services/analytics/config.js` | isFeedbackSurveyDisabled |
| `src/services/analytics/index.js` | logEvent |
| `src/services/policyLimits/index.js` | isPolicyAllowed |
| `src/utils/config.js` | getGlobalConfig, saveGlobalConfig |
| `src/utils/envUtils.js` | isEnvTruthy |
| `src/utils/messages.js` | getLastAssistantMessage |
| `src/utils/model/model.js` | getMainLoopModel |
| `src/utils/settings/settings.js` | getInitialSettings |
| `src/utils/telemetry/events.js` | logOTelEvent |

### 关键代码片段

#### 显示决策逻辑
```typescript
// 行 209-283
const shouldOpen = useMemo(() => {
  if (state !== 'closed') return false;
  if (isLoading) return false;
  if (hasActivePrompt) return false;
  
  // 强制显示（测试用）
  if (process.env.CLAUDE_FORCE_DISPLAY_SURVEY && !feedbackSurvey.timeLastShown) {
    return true;
  }
  
  // 各种禁用检查
  if (!isModelAllowed) return false;
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY)) return false;
  if (isFeedbackSurveyDisabled()) return false;
  if (!isPolicyAllowed('allow_product_feedback')) return false;
  
  // 会话本地节奏
  if (feedbackSurvey.timeLastShown) {
    const timeSinceLastShown = Date.now() - feedbackSurvey.timeLastShown;
    if (timeSinceLastShown < config.minTimeBetweenFeedbackMs) return false;
    if (submitCount < feedbackSurvey.submitCountAtLastAppearance + config.minUserTurnsBetweenFeedback) {
      return false;
    }
  } else {
    const timeSinceSessionStart = Date.now() - sessionStartTime.current;
    if (timeSinceSessionStart < config.minTimeBeforeFeedbackMs) return false;
    if (submitCount < submitCountAtSessionStart.current + config.minUserTurnsBeforeFeedback) {
      return false;
    }
  }
  
  // 概率检查（每个 eligibility window 只计算一次）
  if (lastEligibleSubmitCountRef.current !== submitCount) {
    lastEligibleSubmitCountRef.current = submitCount;
    probabilityPassedRef.current = Math.random() <= (settingsRate ?? config.probability);
  }
  if (!probabilityPassedRef.current) return false;
  
  // 全局节奏（读取文件系统，放最后）
  const globalFeedbackState = getGlobalConfig().feedbackSurveyState;
  if (globalFeedbackState?.lastShownTime) {
    const timeSinceGlobalLastShown = Date.now() - globalFeedbackState.lastShownTime;
    if (timeSinceGlobalLastShown < config.minTimeBetweenGlobalFeedbackMs) return false;
  }
  
  return true;
}, [/* 依赖数组 */]);
```

#### 转录分享决策
```typescript
// 行 124-143
const shouldShowTranscriptPrompt = useCallback((selected: FeedbackSurveyResponse) => {
  if (selected !== 'bad' && selected !== 'good') return false;
  if (getGlobalConfig().transcriptShareDismissed) return false;
  if (!isPolicyAllowed('allow_product_feedback')) return false;
  
  const probability = selected === 'bad' 
    ? badTranscriptAskConfig.probability 
    : goodTranscriptAskConfig.probability;
  return Math.random() <= probability;
}, [badTranscriptAskConfig.probability, goodTranscriptAskConfig.probability]);
```

#### 转录分享处理
```typescript
// 行 159-184
const onTranscriptSelect = useCallback(async (appearanceId: string, selected: TranscriptShareResponse, surveyResponse: FeedbackSurveyResponse | null): Promise<boolean> => {
  const trigger: TranscriptShareTrigger = surveyResponse === 'good' ? 'good_feedback_survey' : 'bad_feedback_survey';
  
  logEvent('tengu_feedback_survey_event', {
    event_type: `transcript_share_${selected}`,
    appearance_id: appearanceId,
    trigger
  });
  
  if (selected === 'dont_ask_again') {
    saveGlobalConfig(current => ({ ...current, transcriptShareDismissed: true }));
  }
  
  if (selected === 'yes') {
    const result = await submitTranscriptShare(messagesRef.current, trigger, appearanceId);
    logEvent('tengu_feedback_survey_event', {
      event_type: result.success ? 'transcript_share_submitted' : 'transcript_share_failed',
      appearance_id: appearanceId,
      trigger
    });
    return result.success;
  }
  
  return false;
}, [surveyType]);
```

## 依赖与外部交互

### 输入依赖
1. **messages**: 当前会话消息数组，用于转录分享
2. **isLoading**: 是否正在加载，加载时不显示调查
3. **submitCount**: 用户提交次数，用于节奏控制
4. **surveyType**: 调查类型（默认为 'session'）
5. **hasActivePrompt**: 是否有活跃提示，有时不显示调查

### 输出交互
1. **状态**: 返回当前调查状态（closed/open/thanks/transcript_prompt/submitting/submitted）
2. **响应处理**: 返回 `handleSelect` 和 `handleTranscriptSelect` 回调
3. **分析事件**: 通过 `logEvent` 和 `logOTelEvent` 记录用户交互
4. **配置持久化**: 通过 `saveGlobalConfig` 保存跨会话状态
5. **API 调用**: 通过 `submitTranscriptShare` 上传转录

### 依赖关系图
```
useFeedbackSurvey.tsx
  ├── useSurveyState (本地状态管理)
  ├── submitTranscriptShare (转录上传)
  ├── useDynamicConfig (动态配置)
  ├── analytics (分析事件)
  ├── policyLimits (组织策略)
  ├── config (全局配置读写)
  ├── settings (用户设置)
  ├── telemetry (OTel 事件)
  └── model (当前模型信息)
```

## 风险、边界与改进建议

### 已知风险

1. **文件系统读取在 useMemo 中**
   - 行 275: `getGlobalConfig()` 在 `useMemo` 中读取文件
   - 每次依赖变化都可能导致磁盘读取，性能开销较大
   - 注释已说明"Leave this till last because it reads from the filesystem which is expensive"

2. **概率计算的竞态条件**
   - `probabilityPassedRef` 和 `lastEligibleSubmitCountRef` 是 refs
   - 在严格模式或并发渲染下可能有意外行为

3. **硬编码的调查类型**
   - 行 43: `surveyType: FeedbackSurveyType = 'session'`
   - 虽然支持传入，但默认和主要使用场景是 'session'

4. **动态配置的加载延迟**
   - `useDynamicConfig` 初始返回默认值，异步获取真实配置
   - 可能导致首次显示使用默认概率而非配置值

### 边界情况

1. **消息数组为空**: 正常处理，`getLastAssistantMessage` 返回 undefined
2. **submitCount 回退**: 如果 `submitCount` 异常减少，概率会重新计算
3. **时间回退**: 系统时间回退可能导致节奏控制异常
4. **全局配置读写失败**: 静默失败，不影响调查显示
5. **转录分享失败**: 记录失败事件，显示感谢页面而非错误

### 改进建议

1. **缓存全局配置读取**
   ```typescript
   // 建议：使用 ref 缓存全局配置读取结果
   const globalConfigRef = useRef(getGlobalConfig().feedbackSurveyState);
   useEffect(() => {
     globalConfigRef.current = getGlobalConfig().feedbackSurveyState;
   }, [/* 适当依赖 */]);
   ```

2. **优化概率计算**
   ```typescript
   // 建议：使用 useMemo 缓存概率计算
   const probabilityCheck = useMemo(() => {
     if (lastEligibleSubmitCountRef.current !== submitCount) {
       lastEligibleSubmitCountRef.current = submitCount;
       return Math.random() <= (settingsRate ?? config.probability);
     }
     return probabilityPassedRef.current;
   }, [submitCount, settingsRate, config.probability]);
   ```

3. **添加调试日志**
   ```typescript
   // 建议：在显示决策关键点添加调试日志
   if (process.env.DEBUG_FEEDBACK_SURVEY) {
     console.log('[useFeedbackSurvey] shouldOpen check:', { 
       state, isLoading, hasActivePrompt, isModelAllowed, ...
     });
   }
   ```

4. **支持配置热更新**
   ```typescript
   // 建议：监听配置变化并重新评估
   useEffect(() => {
     if (config.probability !== previousConfig?.probability) {
       probabilityPassedRef.current = false; // 重置概率检查
     }
   }, [config]);
   ```

5. **提取显示决策逻辑**
   ```typescript
   // 建议：将 shouldOpen 逻辑提取为纯函数，便于测试
   function shouldShowSurvey(params: ShouldShowParams): boolean {
     // 纯逻辑实现
   }
   ```

6. **优化依赖数组**
   - 当前 `useMemo` 依赖数组包含 15+ 项
   - 考虑使用 reducer 模式或状态机简化逻辑

### 测试建议

1. **显示决策测试**:
   - 验证所有禁用条件（环境变量、组织策略等）
   - 验证时间节奏控制
   - 验证消息数节奏控制
   - 验证概率计算

2. **转录分享测试**:
   - 验证仅在好评/差评后显示
   - 验证 `dont_ask_again` 持久化
   - 验证分享成功/失败处理

3. **事件追踪测试**:
   - 验证所有事件正确触发
   - 验证事件参数正确

4. **边界测试**:
   - 空消息数组
   - 时间边界（刚好达到阈值）
   - 配置加载延迟场景
