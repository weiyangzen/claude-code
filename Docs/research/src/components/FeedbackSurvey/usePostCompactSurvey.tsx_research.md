# usePostCompactSurvey.tsx 深度研究文档

## 文件元数据
- **路径**: `src/components/FeedbackSurvey/usePostCompactSurvey.tsx`
- **大小**: 24,071 bytes (编译后，含 React Compiler 转换)
- **类型**: React Hook (TypeScript)
- **所属模块**: FeedbackSurvey 反馈调查系统

---

## 一、场景与职责

### 1.1 核心场景
`usePostCompactSurvey` 是 Claude Code 中用于**会话压缩后反馈调查**的 React Hook。当会话历史被压缩（compaction）以管理上下文窗口大小时，系统会以一定概率显示反馈调查，收集用户对压缩后会话质量的满意度。

### 1.2 触发场景
- 会话执行了内存压缩（session memory compaction）
- 压缩边界（compact boundary）被创建
- 用户在压缩边界后有新的交互（用户或助手消息）
- 满足概率门槛（20% 触发概率）
- GrowthBook 功能开关 `tengu_post_compact_survey` 已启用

### 1.3 职责边界
- **不负责**: 通用会话反馈调查（由 `useFeedbackSurvey` 处理）
- **不负责**: 内存功能反馈调查（由 `useMemorySurvey` 处理）
- **负责**: 专门收集与会话压缩相关的用户反馈
- **负责**: 跟踪压缩边界并检测边界后的用户活动

---

## 二、功能点目的

### 2.1 主要功能

| 功能 | 目的 | 用户价值 |
|------|------|----------|
| 压缩后满意度调查 | 收集用户对会话压缩质量的反馈 | 帮助优化压缩算法和策略 |
| 边界活动检测 | 确保用户真正体验了压缩后的会话 | 避免对未体验压缩的用户打扰 |
| 会话内存压缩关联 | 与 `tengu_sm_compact` 功能集成 | 支持实验性功能的数据收集 |
| 概率控制 | 20% 触发概率 | 平衡数据收集与用户体验 |

### 2.2 压缩边界检测逻辑

```typescript
function hasMessageAfterBoundary(messages: Message[], boundaryUuid: string): boolean {
  const boundaryIndex = messages.findIndex(msg => msg.uuid === boundaryUuid);
  if (boundaryIndex === -1) return false;
  
  // 检查边界后是否有用户或助手消息
  for (let i = boundaryIndex + 1; i < messages.length; i++) {
    const msg = messages[i];
    if (msg && (msg.type === 'user' || msg.type === 'assistant')) {
      return true;
    }
  }
  return false;
}
```

### 2.3 调查流程

```
1. 检测到新的压缩边界
2. 记录边界 UUID 到 pendingCompactBoundaryUuid
3. 等待用户在边界后发送消息
4. 满足条件后以 20% 概率触发调查
5. 显示调查 UI 收集反馈
6. 记录分析事件
```

---

## 三、具体技术实现

### 3.1 核心常量定义

```typescript
const HIDE_THANKS_AFTER_MS = 3000;           // 感谢消息显示时长
const POST_COMPACT_SURVEY_GATE = 'tengu_post_compact_survey';  // GrowthBook 开关
const SURVEY_PROBABILITY = 0.2;              // 20% 触发概率
```

### 3.2 状态管理

```typescript
// 使用 useState 跟踪 GrowthBook 开关状态
const [gateEnabled, setGateEnabled] = useState<boolean | null>(null);

// 使用 useRef 跟踪已见的压缩边界
const seenCompactBoundaries = useRef<Set<string>>(new Set());

// 使用 useRef 跟踪待处理的压缩边界
const pendingCompactBoundaryUuid = useRef<string | null>(null);
```

### 3.3 压缩边界检测算法

```typescript
// 从消息中提取当前所有压缩边界
const currentCompactBoundaries = useMemo(() => {
  return new Set(
    messages
      .filter(msg => isCompactBoundaryMessage(msg))
      .map(msg => msg.uuid)
  );
}, [messages]);

// 在 useEffect 中检测新边界
useEffect(() => {
  if (!enabled) return;
  
  // 获取新出现的边界（不在 seenCompactBoundaries 中）
  const newBoundaries = Array.from(currentCompactBoundaries)
    .filter(uuid => !seenCompactBoundaries.current.has(uuid));
  
  if (newBoundaries.length > 0) {
    // 更新已见边界集合
    seenCompactBoundaries.current = new Set(currentCompactBoundaries);
    // 记录最新的边界为待处理
    pendingCompactBoundaryUuid.current = newBoundaries[newBoundaries.length - 1];
  }
}, [currentCompactBoundaries, enabled]);
```

### 3.4 触发条件检查流程

```typescript
useEffect(() => {
  if (!enabled) return;
  if (state !== 'closed' || isLoading) return;
  if (hasActivePrompt) return;
  if (gateEnabled !== true) return;  // GrowthBook 开关检查
  if (isFeedbackSurveyDisabled()) return;
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY)) return;
  
  // 检查待处理边界后的用户活动
  if (pendingCompactBoundaryUuid.current !== null) {
    if (hasMessageAfterBoundary(messages, pendingCompactBoundaryUuid.current)) {
      // 用户在边界后有活动，重置待处理状态
      pendingCompactBoundaryUuid.current = null;
      
      // 概率触发调查
      if (Math.random() < SURVEY_PROBABILITY) {
        open();
      }
    }
  }
}, [
  enabled, currentCompactBoundaries, state, isLoading, 
  hasActivePrompt, gateEnabled, messages, open
]);
```

### 3.5 GrowthBook 开关初始化

```typescript
useEffect(() => {
  if (!enabled) return;
  
  // 使用 checkStatsigFeatureGate_CACHED_MAY_BE_STALE 检查开关
  // 这是 Statsig 迁移到 GrowthBook 期间的兼容函数
  setGateEnabled(
    checkStatsigFeatureGate_CACHED_MAY_BE_STALE(POST_COMPACT_SURVEY_GATE)
  );
}, [enabled]);
```

### 3.6 事件上报

```typescript
// 调查显示时上报
const onOpen = useCallback((appearanceId: string) => {
  const smCompactionEnabled = shouldUseSessionMemoryCompaction();
  
  logEvent('tengu_post_compact_survey_event', {
    event_type: 'appeared',
    appearance_id: appearanceId,
    session_memory_compaction_enabled: smCompactionEnabled
  });
  
  logOTelEvent('feedback_survey', {
    event_type: 'appeared',
    appearance_id: appearanceId,
    survey_type: 'post_compact'
  });
}, []);

// 用户响应时上报
const onSelect = useCallback((appearanceId: string, selected: FeedbackSurveyResponse) => {
  const smCompactionEnabled = shouldUseSessionMemoryCompaction();
  
  logEvent('tengu_post_compact_survey_event', {
    event_type: 'responded',
    appearance_id: appearanceId,
    response: selected,
    session_memory_compaction_enabled: smCompactionEnabled
  });
  
  logOTelEvent('feedback_survey', {
    event_type: 'responded',
    appearance_id: appearanceId,
    response: selected,
    survey_type: 'post_compact'
  });
}, []);
```

---

## 四、关键代码路径与文件引用

### 4.1 依赖关系图

```
usePostCompactSurvey.tsx
├── useSurveyState.tsx (核心状态管理)
├── src/services/analytics/
│   ├── config.js (isFeedbackSurveyDisabled)
│   ├── growthbook.js (checkStatsigFeatureGate_CACHED_MAY_BE_STALE)
│   └── index.js (logEvent)
├── src/services/compact/sessionMemoryCompact.js (shouldUseSessionMemoryCompaction)
├── src/types/message.js (Message 类型)
├── src/utils/envUtils.js (isEnvTruthy)
├── src/utils/messages.js (isCompactBoundaryMessage)
└── src/utils/telemetry/events.js (logOTelEvent)
```

### 4.2 关键文件路径

| 文件 | 作用 | 调用方式 |
|------|------|----------|
| `src/components/FeedbackSurvey/useSurveyState.tsx` | 提供基础调查状态管理 | Hook 组合 |
| `src/services/analytics/growthbook.ts` | GrowthBook/Statsig 开关检查 | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` |
| `src/services/compact/sessionMemoryCompact.ts` | 会话内存压缩功能检查 | `shouldUseSessionMemoryCompaction()` |
| `src/utils/messages.ts` | 压缩边界消息检测 | `isCompactBoundaryMessage()` |

### 4.3 压缩边界消息类型

压缩边界消息是 `SystemMessage` 的特殊子类型，标记会话压缩发生的位置：

```typescript
// 位于 src/types/message.ts
interface SystemCompactBoundaryMessage extends SystemMessage {
  type: 'system';
  systemMessageType: 'compact_boundary';
  compactMetadata: {
    trigger: 'auto' | 'manual';
    preCompactTokenCount: number;
    // ... 其他元数据
  };
}
```

---

## 五、依赖与外部交互

### 5.1 外部服务依赖

| 服务 | 用途 | 失败行为 |
|------|------|----------|
| GrowthBook/Statsig | 功能开关控制 (`tengu_post_compact_survey`) | 默认关闭，不显示调查 |
| Analytics (1P) | 事件上报 (`tengu_post_compact_survey_event`) | 静默失败，不影响功能 |
| OTel Telemetry | 遥测事件 (`feedback_survey`) | 静默失败，不影响功能 |

### 5.2 功能开关

```typescript
// GrowthBook/Statsig 功能开关
const POST_COMPACT_SURVEY_GATE = 'tengu_post_compact_survey';

// 依赖的会话内存压缩开关（仅用于分析上报）
// tengu_session_memory + tengu_sm_compact
```

### 5.3 环境变量

```typescript
// 禁用所有反馈调查
CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY?: string;

// 启用/禁用会话内存压缩（测试用）
ENABLE_CLAUDE_CODE_SM_COMPACT?: string;
DISABLE_CLAUDE_CODE_SM_COMPACT?: string;

// 测试环境自动禁用
NODE_ENV?: 'test';
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险

| 风险 | 严重程度 | 描述 | 缓解措施 |
|------|----------|------|----------|
| 边界消息丢失 | 中 | 如果压缩边界消息被意外删除，调查可能无法触发 | 依赖消息列表的稳定性 |
| 概率聚集 | 低 | 多个边界连续出现时可能多次触发 | 使用 pendingCompactBoundaryUuid 确保一次只处理一个 |
| 用户活动误判 | 低 | 系统消息可能被误认为用户活动 | 仅检查 'user' 和 'assistant' 类型消息 |
| 开关状态滞后 | 低 | gateEnabled 只在 enabled 变化时更新 | 用户需重新加载才能获取最新开关状态 |

### 6.2 边界条件

```typescript
// 1. 未找到边界消息时
const boundaryIndex = messages.findIndex(msg => msg.uuid === boundaryUuid);
if (boundaryIndex === -1) return false;

// 2. 边界后没有用户活动时
if (pendingCompactBoundaryUuid.current !== null) {
  if (hasMessageAfterBoundary(messages, pendingCompactBoundaryUuid.current)) {
    // 只有这时才触发
  }
}

// 3. 非 closed 状态不触发
if (state !== 'closed' || isLoading) return;

// 4. 有活动提示时不触发
if (hasActivePrompt) return;

// 5. GrowthBook 开关未启用
if (gateEnabled !== true) return;
```

### 6.3 React Compiler 转换说明

编译后的代码使用了 React Compiler (`react/compiler-runtime`)，主要转换包括：

```typescript
// 原始代码（推断）
const seenCompactBoundaries = useRef(new Set());

// 编译后代码
let t4;
if ($[2] === Symbol.for("react.memo_cache_sentinel")) {
  t4 = new Set();
  $[2] = t4;
} else {
  t4 = $[2];
}
const seenCompactBoundaries = useRef(t4);
```

这种转换优化了 Hook 的缓存行为，但增加了代码阅读难度。

### 6.4 改进建议

1. **增强边界检测**
   - 当前仅检测 UUID 匹配，可考虑添加时间戳验证
   - 支持检测多个连续边界，选择最合适的一个触发调查

2. **优化触发时机**
   - 当前在边界后第一条消息就触发，可等待更多交互后再触发
   - 添加最小交互次数要求（如边界后至少 3 条消息）

3. **改进用户体验**
   - 添加调查上下文说明（"您刚刚经历了会话压缩..."）
   - 支持更细粒度的反馈（压缩质量、信息保留程度等）

4. **代码可维护性**
   - 考虑将 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` 替换为纯 GrowthBook API
   - 添加更多注释解释 React Compiler 转换后的代码逻辑

5. **测试覆盖**
   - 添加单元测试模拟压缩边界创建和检测
   - 测试边界消息被删除后的行为
   - 测试多个连续边界的处理

### 6.5 监控指标建议

```typescript
// 建议添加的事件指标
{
  'post_compact_survey_eligible_compacts': '有资格触发调查的压缩次数',
  'post_compact_survey_triggered': '实际触发调查的次数',
  'post_compact_survey_response_rate': '调查响应率',
  'post_compact_survey_boundary_miss': '边界消息未找到的次数',
  'post_compact_survey_user_activity_time': '边界到用户活动的平均时间'
}
```

---

## 七、与会话内存压缩的集成

### 7.1 集成关系

```
sessionMemoryCompact.ts
├── 执行会话内存压缩
├── 创建压缩边界消息
├── 设置 lastSummarizedMessageId
└── 触发 compact 事件

usePostCompactSurvey.tsx
├── 监听压缩边界消息
├── 检测边界后的用户活动
└── 触发反馈调查
```

### 7.2 数据流

1. `trySessionMemoryCompaction()` 执行压缩
2. `createCompactBoundaryMessage()` 创建边界消息
3. 消息列表更新，包含新的边界消息
4. `usePostCompactSurvey` 检测到新边界
5. 等待用户活动
6. 触发调查并上报 `session_memory_compaction_enabled` 状态

---

## 八、相关文档链接

- [useSurveyState.tsx](./useSurveyState.tsx_research.md) - 基础调查状态管理
- [useMemorySurvey.tsx](./useMemorySurvey.tsx_research.md) - 内存功能反馈调查
- [sessionMemoryCompact.ts](../../services/compact/sessionMemoryCompact.ts_research.md) - 会话内存压缩
- [GrowthBook 功能开关文档](../../services/analytics/growthbook.ts_research.md)
