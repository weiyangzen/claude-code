# use-multiple-choice-state.ts 研究文档

## 场景与职责

`use-multiple-choice-state.ts` 是 AskUserQuestionPermissionRequest 组件的状态管理核心 Hook。它使用 React 的 `useReducer` 来管理多选题交互的复杂状态，包括当前问题索引、用户答案、问题状态等。

### 核心职责
1. **状态管理**：管理多选题流程中的所有状态
2. **导航控制**：处理问题间的导航（上一题/下一题）
3. **答案记录**：记录用户对每个问题的答案
4. **问题状态跟踪**：跟踪每个问题的选中值和文本输入值
5. **输入模式管理**：管理是否处于文本输入模式

## 功能点目的

### 1. 多问题导航
- 支持多问题的顺序回答
- 提供 `nextQuestion` 和 `prevQuestion` 方法
- 自动边界检查（防止索引越界）

### 2. 答案管理
- 使用 `Record<string, AnswerValue>` 存储答案
- 键为问题文本，值为答案字符串
- 支持多选答案的逗号分隔存储

### 3. 问题状态管理
- 每个问题独立的状态：`selectedValue` 和 `textInputValue`
- 支持单选（string）和多选（string[]）的选中值
- 支持 "Other" 选项的文本输入

### 4. 输入模式追踪
- `isInTextInput` 标记是否处于文本输入模式
- 用于控制键盘事件的路由（Tabs 快捷键等）

## 具体技术实现

### 关键数据结构

```typescript
// 答案值类型
export type AnswerValue = string

// 单个问题的状态
export type QuestionState = {
  selectedValue?: string | string[]  // 选中的值（单选为 string，多选为 string[]）
  textInputValue: string             // 文本输入值（Other 选项）
}

// Reducer 状态
interface State {
  currentQuestionIndex: number                           // 当前问题索引
  answers: Record<string, AnswerValue>                   // 答案记录
  questionStates: Record<string, QuestionState>          // 问题状态记录
  isInTextInput: boolean                                 // 是否处于文本输入模式
}

// Action 类型
type Action =
  | { type: 'next-question' }
  | { type: 'prev-question' }
  | {
      type: 'update-question-state'
      questionText: string
      updates: Partial<QuestionState>
      isMultiSelect: boolean
    }
  | {
      type: 'set-answer'
      questionText: string
      answer: string
      shouldAdvance: boolean
    }
  | { type: 'set-text-input-mode'; isInInput: boolean }
```

### Reducer 实现

```typescript
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'next-question':
      return {
        ...state,
        currentQuestionIndex: state.currentQuestionIndex + 1,
        isInTextInput: false,  // 切换问题时退出文本输入模式
      }

    case 'prev-question':
      return {
        ...state,
        currentQuestionIndex: Math.max(0, state.currentQuestionIndex - 1),
        isInTextInput: false,
      }

    case 'update-question-state': {
      const existing = state.questionStates[action.questionText]
      const newState: QuestionState = {
        selectedValue:
          action.updates.selectedValue ??
          existing?.selectedValue ??
          (action.isMultiSelect ? [] : undefined),
        textInputValue:
          action.updates.textInputValue ?? existing?.textInputValue ?? '',
      }

      return {
        ...state,
        questionStates: {
          ...state.questionStates,
          [action.questionText]: newState,
        },
      }
    }

    case 'set-answer': {
      const newState = {
        ...state,
        answers: {
          ...state.answers,
          [action.questionText]: action.answer,
        },
      }

      if (action.shouldAdvance) {
        return {
          ...newState,
          currentQuestionIndex: newState.currentQuestionIndex + 1,
          isInTextInput: false,
        }
      }

      return newState
    }

    case 'set-text-input-mode':
      return {
        ...state,
        isInTextInput: action.isInInput,
      }
  }
}
```

### Hook 返回接口

```typescript
export interface MultipleChoiceState {
  currentQuestionIndex: number
  answers: Record<string, AnswerValue>
  questionStates: Record<string, QuestionState>
  isInTextInput: boolean
  nextQuestion: () => void
  prevQuestion: () => void
  updateQuestionState: (
    questionText: string,
    updates: Partial<QuestionState>,
    isMultiSelect: boolean,
  ) => void
  setAnswer: (
    questionText: string,
    answer: string,
    shouldAdvance?: boolean,
  ) => void
  setTextInputMode: (isInInput: boolean) => void
}
```

### Hook 实现

```typescript
export function useMultipleChoiceState(): MultipleChoiceState {
  const [state, dispatch] = useReducer(reducer, INITIAL_STATE)

  const nextQuestion = useCallback(() => {
    dispatch({ type: 'next-question' })
  }, [])

  const prevQuestion = useCallback(() => {
    dispatch({ type: 'prev-question' })
  }, [])

  const updateQuestionState = useCallback(
    (questionText: string, updates: Partial<QuestionState>, isMultiSelect: boolean) => {
      dispatch({
        type: 'update-question-state',
        questionText,
        updates,
        isMultiSelect,
      })
    },
    [],
  )

  const setAnswer = useCallback(
    (questionText: string, answer: string, shouldAdvance: boolean = true) => {
      dispatch({
        type: 'set-answer',
        questionText,
        answer,
        shouldAdvance,
      })
    },
    [],
  )

  const setTextInputMode = useCallback((isInInput: boolean) => {
    dispatch({ type: 'set-text-input-mode', isInInput })
  }, [])

  return {
    currentQuestionIndex: state.currentQuestionIndex,
    answers: state.answers,
    questionStates: state.questionStates,
    isInTextInput: state.isInTextInput,
    nextQuestion,
    prevQuestion,
    updateQuestionState,
    setAnswer,
    setTextInputMode,
  }
}
```

## 关键代码路径与文件引用

### 内部依赖
无（纯 React Hook，无内部文件依赖）

### 外部依赖
| 模块 | 用途 |
|------|------|
| `react` | useReducer, useCallback |

### 关键代码行
- **行 1**: React 导入
- **行 3-8**: 类型定义（AnswerValue, QuestionState）
- **行 10-15**: State 接口定义
- **行 17-33**: Action 类型定义
- **行 34-96**: Reducer 实现
- **行 98-103**: 初始状态
- **行 105-123**: Hook 返回类型定义
- **行 125-179**: Hook 实现

### 使用方
| 文件 | 用途 |
|------|------|
| `AskUserQuestionPermissionRequest.tsx` | 主权限请求组件 |

## 依赖与外部交互

### 与 AskUserQuestionPermissionRequest 的交互

```typescript
// 在 AskUserQuestionPermissionRequest.tsx 中使用
const state = useMultipleChoiceState()
const {
  currentQuestionIndex,
  answers,
  questionStates,
  isInTextInput,
  nextQuestion,
  prevQuestion,
  updateQuestionState,
  setAnswer,
  setTextInputMode,
} = state
```

### 状态流转示例

1. **初始化**：
   ```
   currentQuestionIndex: 0
   answers: {}
   questionStates: {}
   isInTextInput: false
   ```

2. **用户选择选项**：
   ```
   updateQuestionState(questionText, { selectedValue: 'option1' }, false)
   // questionStates[questionText].selectedValue = 'option1'
   ```

3. **用户输入 Other 文本**：
   ```
   updateQuestionState(questionText, { textInputValue: 'custom answer' }, false)
   // questionStates[questionText].textInputValue = 'custom answer'
   ```

4. **提交答案**：
   ```
   setAnswer(questionText, 'final answer', true)
   // answers[questionText] = 'final answer'
   // currentQuestionIndex += 1 (如果 shouldAdvance 为 true)
   ```

## 风险、边界与改进建议

### 潜在风险

1. **问题文本作为键**
   - 使用 `questionText` 作为 `answers` 和 `questionStates` 的键
   - 如果问题文本包含特殊字符或过长，可能有问题
   - 问题文本变更会导致状态丢失

2. **类型安全**
   - `selectedValue` 可以是 `string` 或 `string[]`
   - 需要调用方正确处理类型（通过 `isMultiSelect` 参数）
   - 运行时类型错误风险

3. **状态一致性**
   - `answers` 和 `questionStates` 是分开存储的
   - 可能存在不一致（如 questionStates 有值但 answers 没有）

### 边界情况

1. **空问题文本**
   - 如果 `questionText` 为空字符串，可能覆盖其他状态
   - 未对空字符串进行校验

2. **大量问题**
   - 状态对象会随着问题数量增长
   - 但通常问题数量很少（1-4个），影响不大

3. **并发更新**
   - `updateQuestionState` 和 `setAnswer` 可能快速连续调用
   - Reducer 的纯函数特性保证了状态一致性

### 改进建议

1. **键值优化**
   - 使用问题 ID 替代问题文本作为键
   - 添加问题文本的校验（非空、长度限制）

2. **类型安全**
   - 将 `QuestionState` 拆分为单选和多选两个类型
   - 使用泛型或联合类型替代 `string | string[]`

   ```typescript
   interface SingleSelectState {
     type: 'single'
     selectedValue?: string
     textInputValue: string
   }
   
   interface MultiSelectState {
     type: 'multi'
     selectedValue: string[]
     textInputValue: string
   }
   
   type QuestionState = SingleSelectState | MultiSelectState
   ```

3. **状态持久化**
   - 添加本地存储支持，防止页面刷新丢失
   - 添加状态导出/导入功能

4. **功能增强**
   - 添加 "跳转到问题" 功能
   - 添加答案历史/撤销功能
   - 支持问题的条件显示（基于前面问题的答案）

5. **测试覆盖**
   - 添加 Reducer 的单元测试
   - 测试边界情况（空问题、大量问题）
   - 测试状态流转的正确性

6. **性能优化**
   - 当前实现已使用 `useCallback` 缓存回调
   - 考虑使用 `useMemo` 缓存派生状态（如已完成问题数）
   - 对于大量问题，考虑使用 `useMemo` 缓存 `questionStates` 的查找

### 代码质量

1. **文档完善**
   - 为每个 Action 类型添加 JSDoc 注释
   - 说明 `shouldAdvance` 参数的行为

2. **错误处理**
   - 添加对无效 `questionText` 的警告
   - 在开发模式下验证状态一致性
