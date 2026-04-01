# QuestionView.tsx 研究文档

## 场景与职责

`QuestionView.tsx` 是 AskUserQuestionPermissionRequest 组件的核心子组件，负责渲染单个问题的交互式界面。它是用户与多选题系统进行交互的主要入口，支持单选、多选、文本输入（Other选项）以及图片粘贴等复杂交互场景。

### 核心职责
1. **问题展示**：渲染问题标题和选项列表
2. **交互处理**：处理键盘导航、选项选择、文本输入
3. **模式切换**：支持 Plan 模式下的特殊交互（如 "Skip interview and plan immediately"）
4. **外部编辑器集成**：支持通过 Ctrl+G 打开外部编辑器编辑文本
5. **图片粘贴支持**：允许用户在文本输入区域粘贴图片

## 功能点目的

### 1. 多类型选项渲染
- **单选模式**：使用 `Select` 组件，用户只能选择一个选项
- **多选模式**：使用 `SelectMulti` 组件，用户可以选择多个选项
- **Other 选项**：自动添加的文本输入选项，允许用户输入自定义答案

### 2. 键盘导航
- 上下箭头/Ctrl+P/Ctrl+N：在选项间导航
- Enter：选择当前选项
- Esc：取消/退出
- Tab：在问题间切换（当有多问题时）
- 'n' 键：在 PreviewQuestionView 中聚焦笔记输入

### 3. Plan 模式支持
当处于 Plan 模式时，底部显示额外的操作选项：
- "Chat about this" - 与 Claude 继续对话
- "Skip interview and plan immediately" - 跳过访谈直接生成计划

### 4. 预览模式切换
当问题选项包含 `preview` 字段时，自动切换到 `PreviewQuestionView` 组件，提供左右分栏的预览界面。

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface Props {
  question: Question;                    // 当前问题
  questions: Question[];                 // 所有问题列表
  currentQuestionIndex: number;          // 当前问题索引
  answers: Record<string, string>;       // 已回答的答案
  questionStates: Record<string, QuestionState>;  // 问题状态（选中值、文本输入值）
  hideSubmitTab?: boolean;               // 是否隐藏提交标签
  planFilePath?: string;                 // Plan 文件路径
  pastedContents?: Record<number, PastedContent>;  // 粘贴的图片内容
  minContentHeight?: number;             // 最小内容高度
  minContentWidth?: number;              // 最小内容宽度
  onUpdateQuestionState: (questionText: string, updates: Partial<QuestionState>, isMultiSelect: boolean) => void;
  onAnswer: (questionText: string, label: string | string[], textInput?: string, shouldAdvance?: boolean) => void;
  onTextInputFocus: (isInInput: boolean) => void;
  onCancel: () => void;
  onSubmit: () => void;
  onTabPrev?: () => void;
  onTabNext?: () => void;
  onRespondToClaude: () => void;
  onFinishPlanInterview: () => void;
  onImagePaste?: (base64Image: string, mediaType?: string, filename?: string, dimensions?: ImageDimensions, sourcePath?: string) => void;
  onRemoveImage?: (id: number) => void;
}
```

### 关键流程

#### 1. 选项构建流程
```typescript
// 将 QuestionOption 转换为 Select 组件需要的格式
const textOptions = question.options.map(opt => ({
  type: "text" as const,
  value: opt.label,
  label: opt.label,
  description: opt.description
}));

// 添加 Other 选项（输入框）
const otherOption = {
  type: "input" as const,
  value: "__other__",
  label: "Other",
  placeholder: question.multiSelect ? "Type something" : "Type something.",
  initialValue: questionState?.textInputValue ?? "",
  onChange: (value) => onUpdateQuestionState(questionText, { textInputValue: value }, question.multiSelect ?? false)
};

const options = [...textOptions, otherOption];
```

#### 2. 外部编辑器集成
```typescript
const handleOpenEditor = async (currentValue, setValue) => {
  const result = await editPromptInEditor(currentValue);
  if (result.content !== null && result.content !== currentValue) {
    setValue(result.content);
    onUpdateQuestionState(questionText, {
      textInputValue: result.content
    }, question.multiSelect ?? false);
  }
};
```

#### 3. 键盘事件处理
```typescript
const handleKeyDown = (e) => {
  if (!isFooterFocused) return;
  
  if (e.key === "up" || e.ctrl && e.key === "p") {
    e.preventDefault();
    if (footerIndex === 0) {
      handleUpFromFooter();  // 返回选项列表
    } else {
      setFooterIndex(0);
    }
  }
  
  if (e.key === "return") {
    e.preventDefault();
    if (footerIndex === 0) {
      onRespondToClaude();
    } else {
      onFinishPlanInterview();
    }
  }
  
  if (e.key === "escape") {
    e.preventDefault();
    onCancel();
  }
};
```

#### 4. 答案提交处理
```typescript
// 多选模式
onChange={(values) => {
  onUpdateQuestionState(questionText, { selectedValue: values }, true);
  const textInput = values.includes("__other__") ? questionStates[questionText]?.textInputValue : undefined;
  const finalValues = values.filter(v => v !== "__other__").concat(textInput ? [textInput] : []);
  onAnswer(questionText, finalValues, undefined, false);
}}

// 单选模式
onChange={(value) => {
  onUpdateQuestionState(questionText, { selectedValue: value }, false);
  const textInput = value === "__other__" ? questionStates[questionText]?.textInputValue : undefined;
  onAnswer(questionText, value, textInput);
}}
```

### React Compiler 优化

代码使用了 React Compiler（通过 `_c` 函数），大量使用记忆化来避免不必要的重渲染：
- 使用 `Symbol.for("react.memo_cache_sentinel")` 作为缓存标记
- 通过 `$[n]` 数组存储和比较依赖
- 条件渲染时缓存 JSX 元素

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `PreviewQuestionView.tsx` | 带预览的问题视图（当选项有 preview 时切换） |
| `QuestionNavigationBar.tsx` | 问题导航栏（显示问题标签和进度） |
| `use-multiple-choice-state.ts` | 状态管理 Hook |
| `PermissionRequestTitle.tsx` | 权限请求标题组件 |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `../../../tools/AskUserQuestionTool/AskUserQuestionTool.js` | Question 类型定义 |
| `../../CustomSelect/index.js` | Select/SelectMulti 组件 |
| `../../../utils/editor.js` | 外部编辑器获取 |
| `../../../utils/promptEditor.js` | 编辑器集成 |
| `../../../state/AppState.js` | 应用状态（Plan 模式检测） |
| `../../../utils/config.js` | PastedContent 类型 |
| `../../../utils/imageResizer.js` | ImageDimensions 类型 |

### 关键代码行
- **行 43-67**: Props 解构和默认值处理
- **行 74-233**: 选项构建和外部编辑器处理
- **行 235-261**: 预览模式检测和切换
- **行 297-312**: Select/SelectMulti 渲染和事件绑定
- **行 341-446**: Footer 区域和键盘事件处理

## 依赖与外部交互

### 与父组件 AskUserQuestionPermissionRequest 的交互
通过 props 接收：
- `question`, `questions`: 问题数据
- `answers`, `questionStates`: 状态和答案
- 各种回调函数：`onAnswer`, `onCancel`, `onSubmit`, `onRespondToClaude`, `onFinishPlanInterview`

### 与 CustomSelect 的交互
- 使用 `Select` 组件（单选）
- 使用 `SelectMulti` 组件（多选）
- 传递 `onOpenEditor` 回调支持外部编辑器
- 传递 `onImagePaste` 和 `onRemoveImage` 支持图片粘贴

### 与状态管理的交互
通过 `onUpdateQuestionState` 更新问题状态：
- `selectedValue`: 选中的选项值
- `textInputValue`: 文本输入值（Other 选项）

## 风险、边界与改进建议

### 潜在风险

1. **React Compiler 依赖**
   - 代码严重依赖 React Compiler 的自动记忆化
   - 如果编译器配置变更，可能导致性能回退
   - 手动优化代码可读性较差

2. **键盘事件冲突**
   - 多个层级的键盘事件处理（Footer、选项、文本输入）
   - `isFooterFocused` 和 `isInTextInput` 状态管理复杂
   - 可能存在事件冒泡或优先级问题

3. **Other 选项硬编码**
   - `"__other__"` 作为魔法字符串硬编码
   - 如果多处使用不一致会导致 bug

4. **图片粘贴状态管理**
   - `pastedContents` 通过 props 传递，但删除操作需要回调
   - 状态同步可能存在延迟

### 边界情况

1. **空选项列表**：虽然 schema 要求最少 2 个选项，但组件未处理空列表
2. **超长文本输入**：Other 选项的文本输入没有长度限制
3. **终端尺寸变化**：minContentHeight/minContentWidth 是静态传入，不会随终端调整

### 改进建议

1. **代码可读性**
   - 将 React Compiler 生成的缓存逻辑抽取为自定义 Hook
   - 使用更具描述性的变量名（如 `t6`, `t12` 等）

2. **类型安全**
   - 将 `"__other__"` 提取为常量
   - 为 QuestionState 的 updates 参数添加更严格的类型

3. **性能优化**
   - 考虑使用 `useMemo` 缓存 options 构建
   - 减少不必要的回调函数重建

4. **测试覆盖**
   - 添加键盘导航的单元测试
   - 测试 Plan 模式下的特殊交互
   - 测试图片粘贴和删除流程

5. **功能增强**
   - 支持选项搜索/过滤
   - 支持选项分组
   - 添加选项禁用状态的支持
