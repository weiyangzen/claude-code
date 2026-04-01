# AskUserQuestionPermissionRequest 组件研究文档

## 1. 场景与职责

### 1.1 组件定位

`AskUserQuestionPermissionRequest.tsx` 是 Claude Code CLI 的权限请求系统中的一个核心组件，专门用于渲染**AskUserQuestion 工具**的权限请求界面。该组件负责向用户展示多选题问卷，收集用户选择，并将答案返回给系统。

### 1.2 业务场景

- **需求澄清**: 在 Plan Mode 下向用户询问实现细节和偏好选择
- **决策收集**: 帮助 AI 在多个实现方案中做出符合用户偏好的选择
- **配置确认**: 收集用户对方案、库、方法等的偏好
- **交互式问卷**: 支持单选、多选、文本输入等多种问答形式

### 1.3 核心职责

1. 解析和验证 AskUserQuestionTool 的输入数据
2. 管理多问题的导航状态（当前问题索引、答案记录）
3. 渲染问题视图（支持带预览的选项展示）
4. 处理用户输入（键盘导航、选择、文本输入、图片粘贴）
5. 收集答案并提交，或拒绝/取消操作
6. 支持 Plan Mode 特有的"回复 Claude"和"完成访谈"功能

---

## 2. 功能点目的

### 2.1 问题展示与导航

| 功能 | 目的 |
|------|------|
| 多问题支持 | 允许一次询问 1-4 个问题，提高效率 |
| 问题导航栏 | 显示问题进度，支持 Tab/方向键切换 |
| 答案状态指示 | 用复选框显示哪些问题已回答 |
| 提交视图 | 最后展示所有答案供用户确认 |

### 2.2 选项交互模式

| 模式 | 说明 |
|------|------|
| 单选 (Select) | 标准单选，选择后自动进入下一题 |
| 多选 (SelectMulti) | 支持选择多个选项，需手动提交 |
| 文本输入 (Other) | 每个问题自动提供"Other"选项，允许自由输入 |
| 预览模式 | 选项带预览内容时，使用左右分栏布局 |

### 2.3 Plan Mode 集成

| 功能 | 目的 |
|------|------|
| 回复 Claude | 允许用户不直接回答，而是与 Claude 继续对话澄清 |
| 完成访谈 | 跳过剩余问题，直接用已收集的信息生成计划 |
| 计划文件路径显示 | 在界面中显示当前计划文件位置 |

### 2.4 图片支持

- 支持用户粘贴图片作为答案的一部分
- 图片会被缓存、存储，并最终转换为 Anthropic SDK 的 ImageBlockParam 格式

---

## 3. 具体技术实现

### 3.1 组件架构

```
AskUserQuestionPermissionRequest (入口)
├── AskUserQuestionWithHighlight (语法高亮支持)
└── AskUserQuestionPermissionRequestBody (核心逻辑)
    ├── QuestionView (普通问题视图)
    │   ├── QuestionNavigationBar (导航栏)
    │   ├── PermissionRequestTitle (标题)
    │   ├── Select/SelectMulti (选项组件)
    │   └── PreviewQuestionView (预览模式)
    │       ├── PreviewBox (预览内容展示)
    │       └── TextInput (笔记输入)
    └── SubmitQuestionsView (提交确认视图)
```

### 3.2 关键数据结构

#### 3.2.1 问题定义 (来自 AskUserQuestionTool)

```typescript
// src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx
type Question = {
  question: string;        // 问题文本
  header: string;          // 短标签（最多12字符）
  options: QuestionOption[]; // 2-4个选项
  multiSelect?: boolean;   // 是否多选
}

type QuestionOption = {
  label: string;           // 选项显示文本
  description?: string;    // 选项描述
  preview?: string;        // 预览内容（markdown/HTML）
}
```

#### 3.2.2 组件内部状态

```typescript
// 状态管理通过 useMultipleChoiceState hook
type State = {
  currentQuestionIndex: number;           // 当前问题索引
  answers: Record<string, string>;        // 问题文本 -> 答案
  questionStates: Record<string, QuestionState>; // 每题详细状态
  isInTextInput: boolean;                 // 是否在文本输入模式
}

type QuestionState = {
  selectedValue?: string | string[];      // 选中的值
  textInputValue: string;                 // 文本输入值
}

// 图片粘贴内容
const [pastedContentsByQuestion, setPastedContentsByQuestion] = 
  useState<Record<string, Record<number, PastedContent>>>({})
```

### 3.3 关键流程

#### 3.3.1 初始化流程

```typescript
// 1. 解析输入数据
const result = AskUserQuestionTool.inputSchema.safeParse(toolUseConfirm.input);
const questions = result.success ? result.data.questions || [] : [];

// 2. 计算内容区域尺寸
const maxAllowedHeight = Math.max(MIN_CONTENT_HEIGHT, terminalRows - CONTENT_CHROME_OVERHEAD);
// 遍历所有问题，计算带预览时的最大高度和宽度

// 3. 初始化状态
const state = useMultipleChoiceState();
```

#### 3.3.2 答案提交流程

```typescript
const submitAnswers = async (answersToSubmit: Record<string, string>) => {
  // 1. 记录分析事件
  logEvent("tengu_ask_user_question_accepted", { ... });

  // 2. 构建 annotations（包含预览内容和笔记）
  const annotations: Record<string, { preview?: string; notes?: string }> = {};
  for (const q of questions) {
    const answer = answersToSubmit[q.question];
    const notes = questionStates[q.question]?.textInputValue;
    const selectedOption = q.options.find(opt => opt.label === answer);
    if (selectedOption?.preview || notes?.trim()) {
      annotations[q.question] = {
        ...(preview && { preview }),
        ...(notes?.trim() && { notes: notes.trim() })
      };
    }
  }

  // 3. 转换图片为 SDK blocks
  const contentBlocks = await convertImagesToBlocks(allImageAttachments);

  // 4. 调用 onAllow 完成提交
  toolUseConfirm.onAllow(updatedInput, [], undefined, contentBlocks);
};
```

#### 3.3.3 图片处理流程

```typescript
// 粘贴图片
const onImagePaste = (questionText, base64Image, mediaType, filename, dimensions) => {
  const newContent: PastedContent = {
    id: ++nextPasteIdRef.current,
    type: "image",
    content: base64Image,
    mediaType: mediaType || "image/png",
    filename: filename || "Pasted image",
    dimensions
  };
  cacheImagePath(newContent);
  storeImage(newContent);
  setPastedContentsByQuestion(prev => ({
    ...prev,
    [questionText]: { ...prev[questionText], [pasteId]: newContent }
  }));
};

// 转换为 SDK blocks
async function convertImagesToBlocks(images: PastedContent[]): Promise<ImageBlockParam[] | undefined> {
  return Promise.all(images.map(async img => {
    const block: ImageBlockParam = {
      type: 'image',
      source: {
        type: 'base64',
        media_type: img.mediaType as Base64ImageSource['media_type'],
        data: img.content
      }
    };
    const resized = await maybeResizeAndDownsampleImageBlock(block);
    return resized.block;
  }));
}
```

### 3.4 键盘交互协议

| 按键 | 功能 |
|------|------|
| ↑/↓ 或 Ctrl+P/N | 选项间导航 |
| Tab / Shift+Tab | 问题间切换 |
| Enter | 选择当前选项 |
| 1-9 | 直接选择对应选项 |
| n | 进入笔记输入模式（预览模式） |
| Ctrl+G | 在外部编辑器中编辑文本 |
| Esc | 取消/退出 |

### 3.5 布局计算算法

```typescript
// 计算带预览问题的内容尺寸
for (const q of questions) {
  const hasPreview = q.options.some(opt => opt.preview);
  if (hasPreview) {
    // 计算预览区域高度
    const maxPreviewContentLines = Math.max(1, maxAllowedHeight - 11);
    let maxPreviewBoxHeight = 0;
    for (const opt of q.options) {
      if (opt.preview) {
        const rendered = applyMarkdown(opt.preview, theme, highlight);
        const previewLines = rendered.split("\n");
        const isTruncated = previewLines.length > maxPreviewContentLines;
        const displayedLines = isTruncated ? maxPreviewContentLines : previewLines.length;
        maxPreviewBoxHeight = Math.max(maxPreviewBoxHeight, displayedLines + (isTruncated ? 1 : 0) + 2);
      }
    }
    // 左右分栏布局高度 = max(左侧面板, 右侧面板) + 额外开销
    const rightPanelHeight = maxPreviewBoxHeight + 2;
    const leftPanelHeight = q.options.length + 2;
    maxHeight = Math.max(maxHeight, Math.max(leftPanelHeight, rightPanelHeight) + 7);
  }
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件 | 职责 |
|------|------|
| `AskUserQuestionPermissionRequest.tsx` | 主组件，状态管理，答案提交 |
| `QuestionView.tsx` | 普通问题渲染，键盘处理，Footer 交互 |
| `PreviewQuestionView.tsx` | 带预览的问题渲染，左右分栏布局 |
| `PreviewBox.tsx` | 预览内容展示框（带边框、截断指示） |
| `QuestionNavigationBar.tsx` | 问题导航栏，响应式宽度计算 |
| `SubmitQuestionsView.tsx` | 提交确认视图 |
| `use-multiple-choice-state.ts` | 状态管理 Hook（reducer 模式） |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx` | 工具定义，输入/输出 Schema |
| `src/tools/AskUserQuestionTool/prompt.ts` | 工具提示文本，预览功能说明 |
| `src/components/CustomSelect/select.tsx` | 单选组件 (Select) |
| `src/components/CustomSelect/SelectMulti.tsx` | 多选组件 (SelectMulti) |
| `src/components/permissions/PermissionRequest.tsx` | 父组件，工具到权限组件的映射 |
| `src/components/permissions/PermissionRequestTitle.tsx` | 标题组件 |
| `src/components/permissions/PermissionRuleExplanation.tsx` | 权限规则说明 |

### 4.3 工具调用链

```
Tool Execution
└── PermissionRequest.tsx (permissionComponentForTool)
    └── AskUserQuestionPermissionRequest (当 tool === AskUserQuestionTool)
        ├── AskUserQuestionTool.inputSchema.safeParse() // 验证输入
        ├── useMultipleChoiceState() // 初始化状态
        ├── QuestionView / PreviewQuestionView // 渲染问题
        └── submitAnswers() // 提交答案
            └── toolUseConfirm.onAllow() // 回调完成
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
// React 生态
import React, { Suspense, use, useCallback, useMemo, useRef, useState } from 'react';

// Ink (终端 UI 库)
import { useTheme } from '../../../ink.js';
import { Box, Text } from '../../../ink.js';

// 应用状态
import { useAppState } from '../../../state/AppState.js';
import { useSettings } from '../../../hooks/useSettings.js';
import { useTerminalSize } from '../../../hooks/useTerminalSize.js';
import { useKeybindings } from '../../../keybindings/useKeybinding.js';

// 工具定义
import { AskUserQuestionTool, type Question } from '../../../tools/AskUserQuestionTool/AskUserQuestionTool.js';

// 图片处理
import { maybeResizeAndDownsampleImageBlock } from '../../../utils/imageResizer.js';
import { cacheImagePath, storeImage } from '../../../utils/imageStore.js';

// Markdown 渲染
import { applyMarkdown } from '../../../utils/markdown.js';
import { getCliHighlightPromise, type CliHighlight } from '../../../utils/cliHighlight.js';

// 分析
import { logEvent, type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS } from '../../../services/analytics/index.js';

// Plan Mode
import { isPlanModeInterviewPhaseEnabled } from '../../../utils/planModeV2.js';
import { getPlanFilePath } from '../../../utils/plans.js';
```

### 5.2 子组件依赖

```typescript
// 同目录子组件
import { QuestionView } from './QuestionView.js';
import { SubmitQuestionsView } from './SubmitQuestionsView.js';
import { useMultipleChoiceState } from './use-multiple-choice-state.js';

// 父目录共享组件
import type { PermissionRequestProps } from '../PermissionRequest.js';
import { PermissionRequestTitle } from '../PermissionRequestTitle.js';
import { PermissionRuleExplanation } from '../PermissionRuleExplanation.js';

// 通用组件
import { Select, SelectMulti, type OptionWithDescription } from '../../CustomSelect/index.js';
import { Divider } from '../../design-system/Divider.js';
import { FilePathLink } from '../../FilePathLink.js';
```

### 5.3 与 AskUserQuestionTool 的契约

```typescript
// 输入 Schema（工具调用时传入）
interface Input {
  questions: Question[];           // 1-4 个问题
  answers?: Record<string, string>; // 预填充答案（通常为空）
  annotations?: Record<string, { preview?: string; notes?: string }>;
  metadata?: { source?: string };   // 分析追踪
}

// 输出 Schema（工具返回）
interface Output {
  questions: Question[];
  answers: Record<string, string>;
  annotations?: Record<string, { preview?: string; notes?: string }>;
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 输入验证失败 | 如果输入数据不符合 Schema，questions 会变为空数组 | 使用 `safeParse` 并回退到空数组 |
| 终端高度不足 | 小终端窗口可能导致内容被截断 | 计算 `maxAllowedHeight` 并设置最小值 |
| 图片过大 | 粘贴的大图片可能超出 API 限制 | 使用 `maybeResizeAndDownsampleImageBlock` 压缩 |
| 状态不一致 | 多选题和单选题状态管理方式不同 | 通过 `isMultiSelect` 参数区分处理 |

### 6.2 边界情况

1. **单问题单选**: 自动提交，不显示提交标签页 (`hideSubmitTab = true`)
2. **空选项预览**: 预览框显示 "No preview available"
3. **无答案提交**: 允许提交空答案，但会显示警告
4. **Plan Mode 切换**: 通过 `useAppState` 监听模式变化，动态显示 Plan Mode 特有功能
5. **语法高亮禁用**: 通过 `useSettings` 检测，禁用时代码块不应用高亮

### 6.3 改进建议

#### 6.3.1 性能优化

```typescript
// 当前：每次渲染都重新计算所有问题的尺寸
// 建议：使用 useMemo 缓存尺寸计算结果
const contentDimensions = useMemo(() => {
  return calculateDimensions(questions, terminalRows, theme, highlight);
}, [questions, terminalRows, theme, highlight]);
```

#### 6.3.2 代码组织

- `AskUserQuestionPermissionRequestBody` 函数过长（600+ 行），建议拆分为更小的子组件或自定义 hooks
- 图片处理逻辑 (`convertImagesToBlocks`) 可以提取为共享工具函数

#### 6.3.3 可访问性

- 当前键盘导航依赖视觉指示器，建议增加屏幕阅读器支持
- 考虑为预览内容添加滚动提示

#### 6.3.4 测试覆盖

- 需要增加对以下场景的单元测试：
  - 多问题导航逻辑
  - 图片粘贴和提交
  - Plan Mode 特有功能（回复 Claude、完成访谈）
  - 预览模式布局计算
  - 边界情况（空输入、超大预览内容）

#### 6.3.5 类型安全

- `QuestionState` 中的 `selectedValue` 类型为 `string | string[]`，在运行时需要通过 `Array.isArray()` 检查，建议拆分为两个更明确的类型

### 6.4 相关配置

| 配置项 | 位置 | 影响 |
|--------|------|------|
| `syntaxHighlightingDisabled` | useSettings | 禁用代码高亮 |
| `toolPermissionContext.mode` | AppState | 启用 Plan Mode 功能 |
| `getQuestionPreviewFormat()` | bootstrap/state | 决定预览格式（markdown/HTML） |

---

## 7. 附录

### 7.1 文件清单

```
src/components/permissions/AskUserQuestionPermissionRequest/
├── AskUserQuestionPermissionRequest.tsx    # 主组件 (645 行)
├── QuestionView.tsx                         # 普通问题视图 (465 行)
├── PreviewQuestionView.tsx                  # 预览问题视图 (328 行)
├── PreviewBox.tsx                           # 预览框组件 (229 行)
├── QuestionNavigationBar.tsx                # 导航栏 (178 行)
├── SubmitQuestionsView.tsx                  # 提交视图 (144 行)
└── use-multiple-choice-state.ts             # 状态管理 (179 行)
```

### 7.2 变更历史参考

- 该组件经历了 React Compiler 转换，代码中可见 `_c` 缓存机制和 `$[n]` 模式
- 支持语法高亮的异步加载（`Suspense` + `getCliHighlightPromise`）
- 图片粘贴功能是后期添加的，通过 `pastedContents` 和 `onImagePaste` 回调实现
