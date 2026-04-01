# useMemorySurvey.tsx 深度研究文档

## 文件元数据
- **路径**: `src/components/FeedbackSurvey/useMemorySurvey.tsx`
- **大小**: 30,418 bytes (编译后)
- **类型**: React Hook (TypeScript)
- **所属模块**: FeedbackSurvey 反馈调查系统

---

## 一、场景与职责

### 1.1 核心场景
`useMemorySurvey` 是 Claude Code 中专门用于**内存功能反馈调查**的 React Hook。当系统检测到用户在使用自动内存管理功能（如 MEMORY.md、自动记忆提取等）时，会以一定概率弹出反馈调查，收集用户对内存功能的满意度。

### 1.2 触发场景
- 用户对话中提到了 "memory" 或 "memories" 关键词
- 系统检测到读取了自动管理的内存文件（通过 `Read` 工具读取 `~/.claude/projects/*/memory/` 下的文件）
- 满足概率门槛（20% 的触发概率）
- 满足所有前置条件（GrowthBook 开关、策略限制、环境变量等）

### 1.3 职责边界
- **不负责**: 通用会话反馈调查（由 `useFeedbackSurvey` 处理）
- **不负责**: 压缩后反馈调查（由 `usePostCompactSurvey` 处理）
- **负责**: 专门收集与内存功能相关的用户反馈
- **负责**: 转录分享流程（可选的二次确认）

---

## 二、功能点目的

### 2.1 主要功能

| 功能 | 目的 | 用户价值 |
|------|------|----------|
| 内存功能满意度调查 | 收集用户对自动内存功能的反馈 | 帮助产品团队优化内存功能 |
| 智能触发 | 仅在真正使用内存功能时触发 | 避免打扰非目标用户 |
| 转录分享 | 允许用户分享会话转录以协助调试 | 帮助改进产品质量 |
| 概率控制 | 20% 触发概率 | 平衡数据收集与用户体验 |

### 2.2 调查流程状态机

```
closed → open → thanks → closed
           ↓
      transcript_prompt → submitting → submitted → closed
```

### 2.3 响应类型
基于 `FeedbackSurveyResponse` 类型（推断自代码）：
- `'bad'` - 功能表现不佳
- `'fine'` - 功能表现一般
- `'good'` - 功能表现良好
- `'dismissed'` - 用户关闭调查

---

## 三、具体技术实现

### 3.1 核心常量定义

```typescript
const HIDE_THANKS_AFTER_MS = 3000;           // 感谢消息显示时长
const MEMORY_SURVEY_GATE = 'tengu_dunwich_bell';  // GrowthBook 功能开关
const MEMORY_SURVEY_EVENT = 'tengu_memory_survey_event';  // 分析事件名
const SURVEY_PROBABILITY = 0.2;              // 20% 触发概率
const TRANSCRIPT_SHARE_TRIGGER = 'memory_survey';  // 转录分享触发标识
const MEMORY_WORD_RE = /\bmemor(?:y|ies)\b/i;  // 内存关键词正则
```

### 3.2 内存文件检测算法

```typescript
function hasMemoryFileRead(messages: Message[]): boolean {
  for (const message of messages) {
    if (message.type !== 'assistant') continue;
    const content = message.message.content;
    if (!Array.isArray(content)) continue;
    
    for (const block of content) {
      if (block.type !== 'tool_use' || block.name !== FILE_READ_TOOL_NAME) continue;
      
      const input = block.input as { file_path?: unknown };
      if (typeof input.file_path === 'string' && 
          isAutoManagedMemoryFile(input.file_path)) {
        return true;
      }
    }
  }
  return false;
}
```

**算法要点**:
1. 仅检查助手消息（`type === 'assistant'`）
2. 遍历消息内容块，查找 `tool_use` 类型
3. 检查工具名是否为 `Read`（`FILE_READ_TOOL_NAME`）
4. 验证文件路径是否在自动管理内存目录内

### 3.3 触发条件检查流程

```typescript
useEffect(() => {
  if (!enabled) return;
  if (messages.length === 0) { /* 重置 refs */ return; }
  if (state !== 'closed' || isLoading || hasActivePrompt) return;
  
  // 1. GrowthBook 功能开关检查
  if (!getFeatureValue_CACHED_MAY_BE_STALE(MEMORY_SURVEY_GATE, false)) return;
  
  // 2. 自动内存功能启用检查
  if (!isAutoMemoryEnabled()) return;
  
  // 3. 反馈调查全局禁用检查
  if (isFeedbackSurveyDisabled()) return;
  
  // 4. 组织策略限制检查
  if (!isPolicyAllowed('allow_product_feedback')) return;
  
  // 5. 环境变量禁用检查
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY)) return;
  
  // 6. 避免重复评估同一助手消息
  if (!lastAssistant || seenAssistantUuids.current.has(lastAssistant.uuid)) return;
  
  // 7. 文本内容关键词检查
  const text = extractTextContent(lastAssistant.message.content, ' ');
  if (!MEMORY_WORD_RE.test(text)) return;
  
  // 8. 标记已评估，执行内存文件扫描
  seenAssistantUuids.current.add(lastAssistant.uuid);
  if (!memoryReadSeen.current) {
    memoryReadSeen.current = hasMemoryFileRead(messages);
  }
  if (!memoryReadSeen.current) return;
  
  // 9. 概率触发
  if (Math.random() < SURVEY_PROBABILITY) {
    open();
  }
}, [/* deps */]);
```

### 3.4 Ref 管理策略

| Ref | 用途 | 生命周期 |
|-----|------|----------|
| `seenAssistantUuids` | 跟踪已评估的助手消息 UUID | 会话级 |
| `memoryReadSeen` | 缓存内存文件读取检测结果 | 会话级（直到 `/clear`）|
| `messagesRef` | 获取最新消息列表（用于转录分享）| 渲染同步 |

**重置时机**: 当 `messages.length === 0` 时（如执行 `/clear` 命令），重置所有 refs，避免跨会话污染。

### 3.5 转录分享流程

```typescript
const onTranscriptSelect = useCallback(async (
  appearanceId: string, 
  selected: TranscriptShareResponse
): Promise<boolean> => {
  // 1. 记录分析事件
  logEvent(MEMORY_SURVEY_EVENT, { 
    event_type: `transcript_share_${selected}`,
    appearance_id: appearanceId,
    trigger: TRANSCRIPT_SHARE_TRIGGER
  });
  
  // 2. 处理 "不再询问" 选项
  if (selected === 'dont_ask_again') {
    saveGlobalConfig(current => ({
      ...current,
      transcriptShareDismissed: true
    }));
  }
  
  // 3. 处理同意分享
  if (selected === 'yes') {
    const result = await submitTranscriptShare(
      messagesRef.current, 
      TRANSCRIPT_SHARE_TRIGGER, 
      appearanceId
    );
    // 4. 记录提交结果
    logEvent(MEMORY_SURVEY_EVENT, {
      event_type: result.success ? 'transcript_share_submitted' : 'transcript_share_failed'
    });
    return result.success;
  }
  return false;
}, []);
```

---

## 四、关键代码路径与文件引用

### 4.1 依赖关系图

```
useMemorySurvey.tsx
├── useSurveyState.tsx (核心状态管理)
├── submitTranscriptShare.ts (转录分享提交)
├── TranscriptSharePrompt.tsx (转录分享 UI)
├── utils.js (类型定义 - FeedbackSurveyResponse)
├── src/services/analytics/
│   ├── config.js (isFeedbackSurveyDisabled)
│   ├── growthbook.js (getFeatureValue_CACHED_MAY_BE_STALE)
│   └── index.js (logEvent)
├── src/memdir/paths.js (isAutoMemoryEnabled)
├── src/services/policyLimits/index.js (isPolicyAllowed)
├── src/tools/FileReadTool/prompt.js (FILE_READ_TOOL_NAME)
├── src/types/message.js (Message 类型)
├── src/utils/config.js (getGlobalConfig, saveGlobalConfig)
├── src/utils/envUtils.js (isEnvTruthy)
├── src/utils/memoryFileDetection.js (isAutoManagedMemoryFile)
├── src/utils/messages.js (extractTextContent, getLastAssistantMessage)
└── src/utils/telemetry/events.js (logOTelEvent)
```

### 4.2 关键文件路径

| 文件 | 作用 | 调用方式 |
|------|------|----------|
| `src/components/FeedbackSurvey/useSurveyState.tsx` | 提供基础调查状态管理 | Hook 组合 |
| `src/components/FeedbackSurvey/submitTranscriptShare.ts` | 提交转录分享到服务器 | 异步函数调用 |
| `src/services/analytics/growthbook.ts` | 功能开关检查 | `getFeatureValue_CACHED_MAY_BE_STALE` |
| `src/services/policyLimits/index.ts` | 组织策略检查 | `isPolicyAllowed('allow_product_feedback')` |
| `src/utils/memoryFileDetection.ts` | 内存文件路径检测 | `isAutoManagedMemoryFile()` |

---

## 五、依赖与外部交互

### 5.1 外部服务依赖

| 服务 | 用途 | 失败行为 |
|------|------|----------|
| GrowthBook | 功能开关控制 (`tengu_dunwich_bell`) | 默认关闭，不显示调查 |
| Analytics (1P) | 事件上报 (`tengu_memory_survey_event`) | 静默失败，不影响功能 |
| OTel Telemetry | 遥测事件 (`feedback_survey`) | 静默失败，不影响功能 |
| Policy Limits API | 组织策略检查 | 默认允许（fail open） |

### 5.2 配置依赖

```typescript
// GrowthBook 功能开关
const MEMORY_SURVEY_GATE = 'tengu_dunwich_bell';

// 全局配置项
transcriptShareDismissed?: boolean;  // 用户是否选择"不再询问"

// 环境变量
CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY?: string;  // 禁用所有反馈调查
NODE_ENV?: 'test';  // 测试环境自动禁用
```

### 5.3 策略限制集成

```typescript
// 检查组织是否允许产品反馈
if (!isPolicyAllowed('allow_product_feedback')) return;
```

**策略限制详情**:
- 位于 `src/services/policyLimits/index.ts`
- 支持企业/团队级策略控制
- 在 `essential-traffic-only` 模式下，如果缓存不可用，默认拒绝

---

## 六、风险、边界与改进建议

### 6.1 已知风险

| 风险 | 严重程度 | 描述 | 缓解措施 |
|------|----------|------|----------|
| 跨会话状态泄漏 | 中 | `/clear` 前的内存读取状态可能泄漏 | 在 messages.length === 0 时重置 refs |
| 概率重滚 | 低 | 依赖 useEffect deps 变化可能导致重复评估 | 使用 seenAssistantUuids 去重 |
| 性能问题 | 低 | 每次渲染扫描所有消息查找内存文件读取 | 使用 memoryReadSeen 缓存结果 |
| 第三方云禁用 | 低 | Bedrock/Vertex 用户无法使用 | 通过 GrowthBook 检测自动禁用 |

### 6.2 边界条件

```typescript
// 1. 消息为空时重置
if (messages.length === 0) {
  memoryReadSeen.current = false;
  seenAssistantUuids.current.clear();
  return;
}

// 2. 非 closed 状态不触发
if (state !== 'closed' || isLoading || hasActivePrompt) return;

// 3. 已评估过的助手消息跳过
if (seenAssistantUuids.current.has(lastAssistant.uuid)) return;

// 4. 仅当选择 'bad' 或 'good' 时显示转录分享
if (selected_0 !== 'bad' && selected_0 !== 'good') return false;

// 5. 用户选择"不再询问"后永久隐藏
if (getGlobalConfig().transcriptShareDismissed) return false;
```

### 6.3 改进建议

1. **增强检测准确性**
   - 当前仅检测 `Read` 工具，可扩展检测 `FileWriteTool` 对内存文件的写入
   - 考虑添加内存相关命令（如 `/remember`）的检测

2. **优化触发时机**
   - 当前在消息渲染时检测，可考虑在工具执行完成时立即检测
   - 添加冷却期，避免短时间内多次触发

3. **改进用户体验**
   - 添加调查原因的简要说明（"我们注意到您使用了内存功能..."）
   - 支持更细粒度的反馈（具体功能点评分）

4. **代码可维护性**
   - `FeedbackSurveyResponse` 和 `FeedbackSurveyType` 类型定义在 `utils.js` 中，但文件不存在，建议明确类型定义位置
   - 考虑将转录分享逻辑提取为独立 Hook，减少 `useMemorySurvey` 复杂度

5. **测试覆盖**
   - 添加单元测试覆盖各种触发条件组合
   - 模拟 GrowthBook 开关变化的行为
   - 测试 `/clear` 后的状态重置

### 6.4 监控指标建议

```typescript
// 建议添加的事件指标
{
  'memory_survey_eligible_sessions': '有资格显示调查的会话数',
  'memory_survey_triggered': '实际触发调查的次数',
  'memory_survey_response_rate': '调查响应率',
  'memory_survey_transcript_share_rate': '转录分享同意率',
  'memory_file_detection_hit_rate': '内存文件检测命中率'
}
```

---

## 七、相关文档链接

- [useSurveyState.tsx](./useSurveyState.tsx_research.md) - 基础调查状态管理
- [usePostCompactSurvey.tsx](./usePostCompactSurvey.tsx_research.md) - 压缩后反馈调查
- [submitTranscriptShare.ts](./submitTranscriptShare.ts_research.md) - 转录分享提交
- [GrowthBook 功能开关文档](../../services/analytics/growthbook.ts_research.md)
- [策略限制服务文档](../../services/policyLimits/index.ts_research.md)
