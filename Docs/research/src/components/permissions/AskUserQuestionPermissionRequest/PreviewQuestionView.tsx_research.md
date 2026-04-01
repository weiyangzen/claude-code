# PreviewQuestionView.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`PreviewQuestionView` 是一个**分屏式问题视图组件**，专门用于展示带有预览内容的多选题。它采用左右分栏布局：左侧显示选项列表，右侧显示当前聚焦选项的详细预览。这是 `AskUserQuestionPermissionRequest` 流程中处理预览型问题的核心 UI 组件。

### 1.2 使用场景
- **设计选择场景**：用户需要在多个设计方案中选择（如 UI 布局、代码架构）
- **代码对比场景**：展示不同实现方案的代码片段供用户选择
- **配置选择场景**：显示配置文件预览帮助用户决策
- **Plan 模式访谈**：在计划模式中与用户进行交互式问答

### 1.3 调用关系
- **调用方**：`QuestionView.tsx` 在检测到 `hasAnyPreview` 时渲染此组件
- **被调用方**：
  - `QuestionNavigationBar` - 问题导航标签栏
  - `PreviewBox` - 右侧预览内容展示
  - `PermissionRequestTitle` - 问题标题
  - `TextInput` - Notes 输入框

---

## 2. 功能点目的

### 2.1 分屏交互布局
- **左侧面板（30列宽）**：垂直选项列表，带数字索引和选中标记
- **右侧面板（自适应）**：预览内容 + Notes 输入区
- **响应式设计**：根据终端宽度自动调整右侧面板宽度

### 2.2 键盘导航系统
- **方向键导航**：↑/↓ 或 Ctrl+P/Ctrl+N 在选项间移动
- **数字快捷键**：1-9 直接跳转到对应选项
- **Tab 切换**：左右箭头或 Tab 键切换问题（配合父组件）
- **Footer 导航**：从选项列表进入底部操作区

### 2.3 Notes 备注功能
- **快速进入**：按 `n` 键聚焦 Notes 输入框
- **外部编辑器**：Ctrl+G 打开系统编辑器编辑长文本
- **自动保存**：输入内容实时同步到 questionState

### 2.4 Plan 模式支持
- **双 Footer 选项**：
  - "Chat about this" - 向 Claude 反馈澄清
  - "Skip interview and plan immediately" - 跳过访谈直接生成计划
- **条件渲染**：仅在 Plan 模式下显示第二个选项

### 2.5 视觉反馈
- **聚焦指示器**：`figures.pointer` (▸) 显示当前聚焦项
- **选中状态**：`figures.tick` (✔) 标记已选选项
- **颜色编码**：使用主题色区分聚焦、选中、默认状态

---

## 3. 具体技术实现

### 3.1 组件架构

```
PreviewQuestionView
├── QuestionNavigationBar (顶部导航)
├── PermissionRequestTitle (问题标题)
├── Side-by-side Layout
│   ├── Left Panel (选项列表)
│   │   ├── Option 1 [1. label] [✔]
│   │   ├── Option 2 [2. label] [▸]
│   │   └── ...
│   └── Right Panel (预览 + Notes)
│       ├── PreviewBox (预览内容)
│       └── Notes Input (备注输入)
├── Footer Section
│   ├── Divider
│   ├── "Chat about this" option
│   └── "Skip interview..." option (Plan mode)
└── Help Text (快捷键提示)
```

### 3.2 关键数据结构

```typescript
interface Props {
  question: Question;                    // 当前问题
  questions: Question[];                 // 所有问题列表
  currentQuestionIndex: number;          // 当前问题索引
  answers: Record<string, string>;       // 已回答的答案
  questionStates: Record<string, QuestionState>; // 问题状态
  hideSubmitTab?: boolean;               // 是否隐藏提交标签
  minContentHeight?: number;             // 最小内容高度
  minContentWidth?: number;              // 最小内容宽度
  
  // 回调函数
  onUpdateQuestionState: (questionText: string, updates: Partial<QuestionState>, isMultiSelect: boolean) => void;
  onAnswer: (questionText: string, label: string | string[], textInput?: string, shouldAdvance?: boolean) => void;
  onTextInputFocus: (isInInput: boolean) => void;
  onCancel: () => void;
  onTabPrev?: () => void;
  onTabNext?: () => void;
  onRespondToClaude: () => void;
  onFinishPlanInterview: () => void;
}

// 内部状态
const [isFooterFocused, setIsFooterFocused] = useState(false);
const [footerIndex, setFooterIndex] = useState(0);
const [isInNotesInput, setIsInNotesInput] = useState(false);
const [cursorOffset, setCursorOffset] = useState(0);
const [focusedIndex, setFocusedIndex] = useState(0);
```

### 3.3 核心交互流程

#### 3.3.1 选项导航流程
```javascript
const handleNavigate = useCallback((direction: 'up' | 'down' | number) => {
  if (isInNotesInput) return;  // Notes 模式下禁用
  
  let newIndex: number;
  if (typeof direction === 'number') {
    newIndex = direction;  // 数字快捷键
  } else if (direction === 'up') {
    newIndex = focusedIndex > 0 ? focusedIndex - 1 : focusedIndex;
  } else {
    newIndex = focusedIndex < allOptions.length - 1 ? focusedIndex + 1 : focusedIndex;
  }
  
  if (newIndex >= 0 && newIndex < allOptions.length) {
    setFocusedIndex(newIndex);
  }
}, [focusedIndex, allOptions.length, isInNotesInput]);
```

#### 3.3.2 选项选择流程
```javascript
const handleSelectOption = useCallback((index: number) => {
  const option = allOptions[index];
  if (!option) return;
  
  setFocusedIndex(index);
  onUpdateQuestionState(questionText, {
    selectedValue: option.label
  }, false);
  onAnswer(questionText, option.label);
}, [allOptions, questionText, onUpdateQuestionState, onAnswer]);
```

#### 3.3.3 键盘事件处理
```javascript
const handleKeyDown = useCallback((e: KeyboardEvent) => {
  // Footer 区域键盘处理
  if (isFooterFocused) {
    if (e.key === 'up' || e.ctrl && e.key === 'p') {
      // 返回选项列表或切换 Footer 项
    }
    if (e.key === 'return') {
      // 执行 Footer 操作
      footerIndex === 0 ? onRespondToClaude() : onFinishPlanInterview();
    }
    return;
  }
  
  // Notes 输入模式
  if (isInNotesInput) {
    if (e.key === 'escape') {
      handleNotesExit();
    }
    return;
  }
  
  // 选项导航模式
  switch (e.key) {
    case 'up': case 'ctrl+p':
      handleNavigate('up'); break;
    case 'down': case 'ctrl+n':
      // 到达底部时进入 Footer
      focusedIndex === allOptions.length - 1 
        ? handleDownFromPreview() 
        : handleNavigate('down');
      break;
    case 'return':
      handleSelectOption(focusedIndex); break;
    case 'n':
      // 进入 Notes 模式
      setIsInNotesInput(true); break;
    case '1'...'9':
      // 数字快捷键
      handleNavigate(parseInt(e.key) - 1); break;
  }
}, [...]);
```

### 3.4 外部编辑器集成
```javascript
useKeybinding('chat:externalEditor', async () => {
  const currentValue = questionState?.textInputValue || '';
  const result = await editPromptInEditor(currentValue);
  if (result.content !== null && result.content !== currentValue) {
    onUpdateQuestionState(questionText, {
      textInputValue: result.content
    }, false);
  }
}, {
  context: 'Chat',
  isActive: isInNotesInput && !!editor
});
```

### 3.5 尺寸计算策略
```javascript
const LEFT_PANEL_WIDTH = 30;
const GAP = 4;
const { columns } = useTerminalSize();
const previewMaxWidth = columns - LEFT_PANEL_WIDTH - GAP;

// 预览区域可用行数计算
const PREVIEW_OVERHEAD = 11; // 边框、Notes、Footer、Help 等占用
const previewMaxLines = minContentHeight 
  ? Math.max(1, minContentHeight - PREVIEW_OVERHEAD) 
  : undefined;
```

---

## 4. 关键代码路径与文件引用

### 4.1 组件文件
| 文件路径 | 职责 |
|---------|------|
| `src/components/permissions/AskUserQuestionPermissionRequest/PreviewQuestionView.tsx` | 主组件实现 |

### 4.2 依赖组件
| 文件路径 | 功能 |
|---------|------|
| `PreviewBox.tsx` | 右侧预览内容展示 |
| `QuestionNavigationBar.tsx` | 顶部问题导航标签 |
| `PermissionRequestTitle.tsx` | 问题标题显示 |
| `TextInput.tsx` | Notes 输入框 |
| `Divider.tsx` | 分隔线组件 |

### 4.3 依赖工具
| 文件路径 | 功能 |
|---------|------|
| `src/hooks/useTerminalSize.ts` | 终端尺寸监听 |
| `src/state/AppState.ts` | Plan 模式检测 |
| `src/keybindings/useKeybinding.ts` | 快捷键绑定 |
| `src/utils/editor.ts` | 外部编辑器检测 |
| `src/utils/promptEditor.ts` | 外部编辑器调用 |
| `src/utils/ide.ts` | IDE 名称转换 |

### 4.4 类型定义
| 文件路径 | 类型 |
|---------|------|
| `src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx` | `Question`, `QuestionOption` |
| `src/components/permissions/AskUserQuestionPermissionRequest/use-multiple-choice-state.ts` | `QuestionState` |

### 4.5 调用链
```
AskUserQuestionPermissionRequest.tsx
└── QuestionView.tsx (检测 hasAnyPreview)
    └── PreviewQuestionView.tsx
        ├── QuestionNavigationBar (导航)
        ├── PermissionRequestTitle (标题)
        ├── Box (左右分栏)
        │   ├── Box (左侧面板 - 选项列表)
        │   └── Box (右侧面板)
        │       ├── PreviewBox (预览)
        │       └── TextInput (Notes)
        └── Box (Footer)
```

---

## 5. 依赖与外部交互

### 5.1 React 依赖
- **Hooks**: `useCallback`, `useMemo`, `useRef`, `useState`
- **React Compiler**: 使用 `_c(n)` 缓存机制优化渲染

### 5.2 Ink 组件
```typescript
import { Box, Text } from '../../../ink.js';
import type { KeyboardEvent } from '../../../ink/events/keyboard-event.js';
```

### 5.3 外部库
```typescript
import figures from 'figures';  // 终端符号（✔, ▸, ←, → 等）
```

### 5.4 应用状态
```typescript
import { useAppState } from '../../../state/AppState.js';
// 检测是否在 Plan 模式
const isInPlanMode = useAppState(s => s.toolPermissionContext.mode) === 'plan';
```

### 5.5 快捷键系统
```typescript
import { useKeybinding, useKeybindings } from '../../../keybindings/useKeybinding.js';

// Tab 导航快捷键
useKeybindings({
  'tabs:previous': () => onTabPrev?.(),
  'tabs:next': () => onTabNext?.()
}, {
  context: 'Tabs',
  isActive: !isInNotesInput && !isFooterFocused
});
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 状态管理复杂性
- **多层级状态**：`focusedIndex`, `isFooterFocused`, `footerIndex`, `isInNotesInput` 四个相关状态
- **状态同步风险**：切换问题时需要重置 `focusedIndex`，使用 `useRef` 检测问题变化

#### 6.1.2 键盘事件冲突
- **事件优先级**：子组件的 `useInput` 处理器先于父组件注册
- **当前方案**：在子组件中重复实现 Tab 导航逻辑以确保可靠性

#### 6.1.3 终端尺寸限制
- **最小宽度要求**：左侧面板固定 30 列，右侧至少需要一定空间
- **极端情况**：终端宽度 < 40 时布局可能异常

### 6.2 边界情况

| 场景 | 当前行为 | 建议 |
|-----|---------|------|
| 无预览选项 | 显示 "No preview available" | 考虑隐藏右侧面板 |
| 单选项问题 | 仍显示导航 | 简化 UI |
| 终端高度不足 | 内容截断 | 添加滚动提示 |
| 外部编辑器未配置 | 不显示 Ctrl+G 提示 | 正确检测并隐藏 |
| Notes 内容超长 | 正常处理 | 添加字符限制提示 |

### 6.3 改进建议

#### 6.3.1 代码结构优化
```typescript
// 建议：将键盘处理拆分为独立 hook
function usePreviewNavigation(options: Option[], {
  onSelect, onNavigate, onEnterNotes, onEnterFooter
}) {
  // 封装导航逻辑
}

// 建议：将 Footer 提取为独立组件
function PreviewFooter({
  isInPlanMode, onRespondToClaude, onFinishPlanInterview
}) {
  // 封装 Footer 渲染和逻辑
}
```

#### 6.3.2 可访问性改进
- 添加 ARIA 标签支持（Ink 限制，需评估可行性）
- 为屏幕阅读器提供选项摘要
- 添加声音反馈选项

#### 6.3.3 功能扩展
- **搜索过滤**：在选项多时支持搜索
- **预览历史**：记住之前查看的选项预览
- **全屏预览**：支持展开预览到全屏

#### 6.3.4 性能优化
```typescript
// 当前：每次渲染都创建新的回调函数
// 建议：使用 useMemo 缓存选项列表渲染
const optionList = useMemo(() => 
  allOptions.map((option, index) => (
    <OptionItem key={option.label} ... />
  )),
  [allOptions, focusedIndex, selectedValue]
);
```

### 6.4 测试建议
- **单元测试**：
  - 键盘导航逻辑（各方向键行为）
  - Footer 状态切换
  - Notes 进入/退出
- **集成测试**：
  - 与 PreviewBox 协作
  - 与父组件状态同步
- **E2E 测试**：
  - 完整的问题回答流程
  - Plan 模式特殊路径

### 6.5 已知问题

#### 6.5.1 注释中的代码注释
代码中有关于事件优先级的详细注释：
```javascript
// This must be in the child component (not just the parent) because child useInput
// handlers register first on the event emitter and fire before parent handlers.
// Without this, the parent's useKeybindings may not fire reliably depending on
// listener ordering in the event emitter.
```
这表明开发者已经意识到事件系统的复杂性。

#### 6.5.2 React Compiler 产物
代码是 React Compiler 编译后的产物，包含大量 `_c(n)` 缓存检查。直接修改源码需要注意编译器会重新生成这些代码。

---

## 7. 相关类型定义

```typescript
// 来自 AskUserQuestionTool.tsx
interface Question {
  question: string;
  header: string;
  options: QuestionOption[];
  multiSelect?: boolean;
}

interface QuestionOption {
  label: string;
  description: string;
  preview?: string;
}

// 来自 use-multiple-choice-state.ts
interface QuestionState {
  selectedValue?: string | string[];
  textInputValue: string;
}
```

---

## 8. 总结

`PreviewQuestionView` 是一个复杂但设计精良的交互组件，成功实现了：

1. **分屏布局**：左右分栏，信息密度高但不拥挤
2. **流畅导航**：键盘操作完整，支持多种快捷方式
3. **灵活扩展**：Notes 功能和 Plan 模式支持
4. **视觉清晰**：颜色、符号、布局共同提供良好的视觉层次

主要关注点：
- 状态管理复杂，需要小心维护
- 键盘事件系统依赖 Ink 的内部实现细节
- 终端尺寸限制下的布局适配

该组件是 `AskUserQuestionPermissionRequest` 系统中用户体验的关键部分，直接影响用户对预览型问题的交互感受。
