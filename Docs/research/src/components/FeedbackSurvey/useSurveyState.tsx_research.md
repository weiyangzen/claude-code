# useSurveyState.tsx 深度研究文档

## 文件元数据
- **路径**: `src/components/FeedbackSurvey/useSurveyState.tsx`
- **大小**: 14,800 bytes (编译后)
- **类型**: React Hook (TypeScript)
- **所属模块**: FeedbackSurvey 反馈调查系统

---

## 一、场景与职责

### 1.1 核心场景
`useSurveyState` 是 Claude Code 反馈调查系统的**核心状态管理 Hook**。它为所有具体的调查 Hook（`useFeedbackSurvey`、`useMemorySurvey`、`usePostCompactSurvey`）提供统一的状态管理和流程控制。

### 1.2 设计定位
作为底层基础 Hook，`useSurveyState` 实现了：
- **状态机管理**: 调查生命周期的完整状态流转
- **回调抽象**: 统一的回调接口供上层 Hook 注入业务逻辑
- **转录分享支持**: 可选的二次确认流程（转录分享）
- **防重复触发**: 基于 appearanceId 的事件追踪

### 1.3 职责边界
- **负责**: 调查状态（open/closed/thanks 等）的管理
- **负责**: 用户选择的处理和响应
- **负责**: 转录分享流程的状态控制
- **不负责**: 具体的触发条件判断（由上层 Hook 控制）
- **不负责**: 具体的事件上报逻辑（由上层 Hook 通过回调实现）

---

## 二、功能点目的

### 2.1 主要功能

| 功能 | 目的 | 技术实现 |
|------|------|----------|
| 状态机管理 | 控制调查的显示/隐藏生命周期 | useState + 状态流转逻辑 |
| 回调注入 | 允许上层 Hook 自定义行为 | 通过 options 对象传入回调 |
| 转录分享流程 | 支持用户同意分享会话转录 | 条件渲染 + 异步提交 |
| 自动关闭 | 感谢消息后自动关闭调查 | setTimeout 定时器 |
| 唯一标识 | 追踪每次调查展示 | crypto.randomUUID |

### 2.2 状态定义

```typescript
type SurveyState = 
  | 'closed'      // 关闭状态，等待触发
  | 'open'        // 显示调查问卷
  | 'thanks'      // 显示感谢消息
  | 'transcript_prompt'  // 显示转录分享确认
  | 'submitting'  // 正在提交转录
  | 'submitted';  // 转录已提交，显示确认
```

### 2.3 流程图

```
                    ┌─────────┐
                    │ closed  │
                    └────┬────┘
                         │ open()
                         ▼
                    ┌─────────┐
         ┌─────────│  open   │◄──────────────────┐
         │         └────┬────┘                   │
         │              │ handleSelect()         │
         │              ▼                        │
         │    ┌─────────────────┐                │
         │    │  selected ===   │                │
         │    │  'dismissed'?   │                │
         │    └────────┬────────┘                │
         │         yes │    no                   │
         │             ▼    ▼                    │
         │      ┌────────┐  ┌─────────────────┐ │
         │      │ closed │  │ shouldShowTrans │ │
         │      └────────┘  │ criptPrompt?    │ │
         │                  └────────┬────────┘ │
         │                     yes   │    no    │
         │                      ▼    ▼    ▼     │
         │         ┌─────────────────┐  ┌──────┴───┐
         │         │ transcript_     │  │  thanks  │
         │         │ prompt          │  │  (定时器) │
         │         └────────┬────────┘  └────┬─────┘
         │                  │ handleTranscriptSelect()
         │                  ▼
         │    ┌──────────────────────────────┐
         │    │ selected_0 === 'yes'?        │
         │    └─────────────┬────────────────┘
         │       yes        │         no
         │        ▼         ▼          ▼
         │   ┌─────────┐ ┌─────────┐ ┌─────────┐
         │   │submitting│ │  thanks │ │  thanks │
         │   └────┬────┘ └────┬────┘ └────┬────┘
         │        │ 提交完成   │          │
         │        ▼           │          │
         │   ┌─────────┐      │          │
         │   │submitted│      │          │
         │   └────┬────┘      │          │
         │        │ 定时器    │ 定时器   │ 定时器
         └────────┴──────────┴──────────┴──────► closed
```

---

## 三、具体技术实现

### 3.1 接口定义

```typescript
interface UseSurveyStateOptions {
  hideThanksAfterMs: number;           // 感谢消息显示时长
  onOpen: (appearanceId: string) => void | Promise<void>;  // 打开回调
  onSelect: (appearanceId: string, selected: FeedbackSurveyResponse) => void | Promise<void>;  // 选择回调
  shouldShowTranscriptPrompt?: (selected: FeedbackSurveyResponse) => boolean;  // 是否显示转录分享
  onTranscriptPromptShown?: (appearanceId: string, surveyResponse: FeedbackSurveyResponse) => void;  // 转录提示显示回调
  onTranscriptSelect?: (appearanceId: string, selected: TranscriptShareResponse, surveyResponse: FeedbackSurveyResponse | null) => boolean | Promise<boolean>;  // 转录选择回调
}

interface UseSurveyStateReturn {
  state: SurveyState;                           // 当前状态
  lastResponse: FeedbackSurveyResponse | null;  // 最后响应
  open: () => void;                             // 打开调查
  handleSelect: (selected: FeedbackSurveyResponse) => boolean;  // 处理选择
  handleTranscriptSelect: (selected: TranscriptShareResponse) => void;  // 处理转录选择
}
```

### 3.2 核心状态管理

```typescript
export function useSurveyState(options: UseSurveyStateOptions): UseSurveyStateReturn {
  const [state, setState] = useState<SurveyState>('closed');
  const [lastResponse, setLastResponse] = useState<FeedbackSurveyResponse | null>(null);
  
  // 每次调查的唯一标识
  const appearanceId = useRef(randomUUID());
  
  // 用于转录分享回调访问最新响应
  const lastResponseRef = useRef<FeedbackSurveyResponse | null>(null);
  
  // ... 实现
}
```

### 3.3 状态流转实现

#### 打开调查

```typescript
const open = useCallback(() => {
  if (state !== 'closed') return;  // 防止重复打开
  
  setState('open');
  appearanceId.current = randomUUID();  // 生成新的唯一标识
  void onOpen(appearanceId.current);    // 触发上层回调
}, [state, onOpen]);
```

#### 处理用户选择

```typescript
const handleSelect = useCallback((selected: FeedbackSurveyResponse): boolean => {
  setLastResponse(selected);
  lastResponseRef.current = selected;
  
  // 触发上层回调
  void onSelect(appearanceId.current, selected);
  
  if (selected === 'dismissed') {
    // 用户关闭调查，直接关闭
    setState('closed');
    setLastResponse(null);
  } else if (shouldShowTranscriptPrompt?.(selected)) {
    // 需要显示转录分享确认
    setState('transcript_prompt');
    onTranscriptPromptShown?.(appearanceId.current, selected);
    return true;  // 表示进入转录分享流程
  } else {
    // 显示感谢消息后关闭
    showThanksThenClose();
  }
  return false;
}, [showThanksThenClose, onSelect, shouldShowTranscriptPrompt, onTranscriptPromptShown]);
```

#### 处理转录分享选择

```typescript
const handleTranscriptSelect = useCallback((selected: TranscriptShareResponse) => {
  switch (selected) {
    case 'yes':
      setState('submitting');
      void (async () => {
        try {
          const success = await onTranscriptSelect?.(
            appearanceId.current, 
            selected, 
            lastResponseRef.current
          );
          if (success) {
            showSubmittedThenClose();  // 提交成功
          } else {
            showThanksThenClose();     // 提交失败，显示感谢
          }
        } catch {
          showThanksThenClose();       // 异常，显示感谢
        }
      })();
      break;
      
    case 'no':
    case 'dont_ask_again':
      // 不分享或不再询问，触发回调后关闭
      void onTranscriptSelect?.(appearanceId.current, selected, lastResponseRef.current);
      showThanksThenClose();
      break;
  }
}, [showThanksThenClose, showSubmittedThenClose, onTranscriptSelect]);
```

### 3.4 定时器管理

```typescript
// 显示感谢消息后自动关闭
const showThanksThenClose = useCallback(() => {
  setState('thanks');
  setTimeout((setState_0, setLastResponse_0) => {
    setState_0('closed');
    setLastResponse_0(null);
  }, hideThanksAfterMs, setState, setLastResponse);
}, [hideThanksAfterMs]);

// 显示提交成功后自动关闭
const showSubmittedThenClose = useCallback(() => {
  setState('submitted');
  setTimeout(setState, hideThanksAfterMs, 'closed');
}, [hideThanksAfterMs]);
```

**注意**: 使用 `setTimeout` 的回调参数传递 `setState` 和 `setLastResponse`，避免闭包捕获过期状态。

---

## 四、关键代码路径与文件引用

### 4.1 依赖关系图

```
useSurveyState.tsx
├── crypto (randomUUID)
├── react (useCallback, useRef, useState)
├── ./TranscriptSharePrompt.js (TranscriptShareResponse 类型)
└── ./utils.js (FeedbackSurveyResponse 类型)
```

### 4.2 类型定义来源

| 类型 | 来源 | 定义位置 |
|------|------|----------|
| `FeedbackSurveyResponse` | `import type { FeedbackSurveyResponse } from './utils.js'` | 推断: `'bad' \| 'fine' \| 'good' \| 'dismissed'` |
| `TranscriptShareResponse` | `import type { TranscriptShareResponse } from './TranscriptSharePrompt.js'` | `'yes' \| 'no' \| 'dont_ask_again'` |

### 4.3 被调用方

| 调用方 | 用途 |
|--------|------|
| `useFeedbackSurvey.tsx` | 通用会话反馈调查 |
| `useMemorySurvey.tsx` | 内存功能反馈调查 |
| `usePostCompactSurvey.tsx` | 压缩后反馈调查 |

---

## 五、依赖与外部交互

### 5.1 标准库依赖

```typescript
import { randomUUID } from 'crypto';  // Node.js crypto 模块
```

### 5.2 React 依赖

```typescript
import { useCallback, useRef, useState } from 'react';
```

### 5.3 无外部服务依赖

`useSurveyState` 是一个纯前端状态管理 Hook，不直接依赖任何外部服务（如 Analytics、GrowthBook 等）。这些依赖通过回调函数注入。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

| 风险 | 严重程度 | 描述 | 缓解措施 |
|------|----------|------|----------|
| 定时器泄漏 | 低 | 组件卸载时定时器可能仍在运行 | 当前实现未清理定时器，但影响有限（仅更新状态） |
| 回调异常 | 中 | 上层回调抛出异常可能影响流程 | 使用 `void` 关键字确保异步回调不阻塞 |
| 状态竞态 | 低 | 快速连续调用可能导致状态不一致 | `open()` 检查 `state !== 'closed'` |
| 内存泄漏 | 低 | lastResponseRef 可能长期持有引用 | 在关闭时设置为 null |

### 6.2 边界条件

```typescript
// 1. 非 closed 状态调用 open() 被忽略
if (state !== 'closed') return;

// 2. dismissed 响应立即关闭
if (selected === 'dismissed') {
  setState('closed');
  setLastResponse(null);
}

// 3. 转录分享提交异常处理
try {
  const success = await onTranscriptSelect?.(...);
} catch {
  showThanksThenClose();  // 异常时优雅降级
}

// 4. 定时器回调使用参数而非闭包
setTimeout((setState_0, setLastResponse_0) => {
  setState_0('closed');      // 使用传入的参数
  setLastResponse_0(null);   // 而非闭包捕获的函数
}, hideThanksAfterMs, setState, setLastResponse);
```

### 6.3 改进建议

1. **定时器清理**
   ```typescript
   // 建议添加清理逻辑
   useEffect(() => {
     return () => {
       if (timeoutRef.current) {
         clearTimeout(timeoutRef.current);
       }
     };
   }, []);
   ```

2. **状态机验证**
   - 当前状态流转依赖编码约定，可添加显式状态机验证
   - 使用 TypeScript 的穷尽检查确保所有状态都被处理

3. **回调错误处理**
   - 当前仅对 `onTranscriptSelect` 进行 try-catch
   - 建议统一包装所有回调的错误处理

4. **类型安全增强**
   ```typescript
   // 建议明确 FeedbackSurveyResponse 类型定义
   export type FeedbackSurveyResponse = 'bad' | 'fine' | 'good' | 'dismissed';
   
   // 建议明确 TranscriptShareResponse 类型定义  
   export type TranscriptShareResponse = 'yes' | 'no' | 'dont_ask_again';
   ```

5. **测试覆盖**
   - 状态流转的单元测试
   - 回调调用的顺序和参数验证
   - 边界条件（快速连续调用、异常回调等）

### 6.4 设计模式分析

`useSurveyState` 使用了**策略模式**（Strategy Pattern）通过回调注入业务逻辑：

```typescript
// 抽象策略接口
interface SurveyStrategy {
  onOpen: (appearanceId: string) => void;
  onSelect: (appearanceId: string, selected: FeedbackSurveyResponse) => void;
  shouldShowTranscriptPrompt?: (selected: FeedbackSurveyResponse) => boolean;
  // ...
}

// 具体策略实现由上层 Hook 提供
useSurveyState({
  onOpen: (id) => { /* useMemorySurvey 的实现 */ },
  onSelect: (id, selected) => { /* useMemorySurvey 的实现 */ },
  // ...
});
```

这种设计使得 `useSurveyState` 保持通用和可复用，同时允许上层 Hook 定制具体行为。

---

## 七、使用示例

### 7.1 基础使用

```typescript
function useMemorySurvey(messages: Message[], isLoading: boolean) {
  const { state, lastResponse, open, handleSelect } = useSurveyState({
    hideThanksAfterMs: 3000,
    onOpen: (appearanceId) => {
      logEvent('memory_survey_appeared', { appearanceId });
    },
    onSelect: (appearanceId, selected) => {
      logEvent('memory_survey_responded', { appearanceId, selected });
    }
  });
  
  // 触发逻辑...
  useEffect(() => {
    if (shouldTrigger) {
      open();
    }
  }, [shouldTrigger, open]);
  
  return { state, lastResponse, handleSelect };
}
```

### 7.2 带转录分享的使用

```typescript
function useFeedbackSurvey(messages: Message[], isLoading: boolean) {
  const { state, handleSelect, handleTranscriptSelect } = useSurveyState({
    hideThanksAfterMs: 3000,
    onOpen: (id) => { /* ... */ },
    onSelect: (id, selected) => { /* ... */ },
    shouldShowTranscriptPrompt: (selected) => {
      return selected === 'bad' || selected === 'good';
    },
    onTranscriptPromptShown: (id, response) => { /* ... */ },
    onTranscriptSelect: async (id, selected, surveyResponse) => {
      if (selected === 'yes') {
        const result = await submitTranscriptShare(messages, trigger, id);
        return result.success;
      }
      return false;
    }
  });
  
  return { state, handleSelect, handleTranscriptSelect };
}
```

---

## 八、相关文档链接

- [useFeedbackSurvey.tsx](./useFeedbackSurvey.tsx_research.md) - 通用会话反馈调查
- [useMemorySurvey.tsx](./useMemorySurvey.tsx_research.md) - 内存功能反馈调查
- [usePostCompactSurvey.tsx](./usePostCompactSurvey.tsx_research.md) - 压缩后反馈调查
- [TranscriptSharePrompt.tsx](./TranscriptSharePrompt.tsx_research.md) - 转录分享 UI
