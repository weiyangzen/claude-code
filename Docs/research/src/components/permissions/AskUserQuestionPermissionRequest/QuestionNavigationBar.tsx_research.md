# QuestionNavigationBar.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`QuestionNavigationBar` 是一个**问题导航标签栏组件**，用于在多问题问卷场景中显示当前进度和导航状态。它以标签页（tab）形式展示所有问题，标记已回答和当前激活的问题，并提供左右导航指示器。

### 1.2 使用场景
- **多问题问卷**：当 `AskUserQuestionTool` 包含 2-4 个问题时显示导航栏
- **进度可视化**：通过复选框图标（☐/☑）直观显示哪些问题已回答
- **快速导航**：用户可通过 Tab/箭头键在问题间切换
- **提交入口**：最后一个标签是 "Submit"，用于进入答案审核页面

### 1.3 调用关系
- **调用方**：
  - `AskUserQuestionPermissionRequest.tsx` - 主权限请求组件
  - `QuestionView.tsx` - 普通问题视图
  - `PreviewQuestionView.tsx` - 预览问题视图
  - `SubmitQuestionsView.tsx` - 提交确认视图
- **无子组件调用**：纯展示组件，不依赖其他子组件

---

## 2. 功能点目的

### 2.1 标签导航展示
- **问题标签**：显示每个问题的简短标题（`header` 字段）或默认 "Q{N}"
- **当前问题高亮**：当前激活的问题使用 `permission` 背景色高亮
- **回答状态标记**：已回答的问题显示 `☑`，未回答显示 `☐`

### 2.2 自适应布局
- **宽度感知**：根据终端宽度动态计算可用空间
- **智能截断**：空间不足时截断非当前问题的标签文本
- **当前问题优先**：确保当前问题标签有足够显示空间

### 2.3 导航指示器
- **左右箭头**：`←` 和 `→` 指示可导航方向
- **智能隐藏**：单问题且隐藏提交标签时隐藏箭头
- **禁用状态**：已在第一个/最后一个问题时淡化对应箭头

### 2.4 提交标签
- **可选显示**：通过 `hideSubmitTab` 控制是否显示
- **激活状态**：当前在提交页面时高亮显示

---

## 3. 具体技术实现

### 3.1 组件架构

```
QuestionNavigationBar
├── Left Arrow (←) [条件渲染]
├── Question Tabs
│   ├── Tab 1 [☐ Q1] / [☑ Q1] / [☑ Q1] (active)
│   ├── Tab 2 [☐ Q2] / [☑ Q2] / [☑ Q2] (active)
│   └── ...
├── Submit Tab [☑ Submit] / [☑ Submit] (active) [条件渲染]
└── Right Arrow (→) [条件渲染]
```

### 3.2 关键数据结构

```typescript
interface Props {
  questions: Question[];           // 问题列表
  currentQuestionIndex: number;    // 当前问题索引 (0-based)
  answers: Record<string, string>; // 答案映射
  hideSubmitTab?: boolean;         // 是否隐藏提交标签
}

// 内部计算
interface TabLayout {
  tabHeaders: string[];      // 原始标签文本
  tabDisplayTexts: string[]; // 截断后的显示文本
  hideArrows: boolean;       // 是否隐藏箭头
}
```

### 3.3 核心布局算法

#### 3.3.1 空间计算
```javascript
const submitText = hideSubmitTab ? "" : ` ${figures.tick} Submit `;
const fixedWidth = stringWidth("← ") + stringWidth(" →") + stringWidth(submitText);
const availableForTabs = columns - fixedWidth;
```

#### 3.3.2 标签宽度分配策略
```javascript
// 1. 计算理想宽度（每个标签 4 + 文本宽度）
const idealWidths = tabHeaders.map(h => 4 + stringWidth(h));
const totalIdealWidth = idealWidths.reduce((sum, w) => sum + w, 0);

// 2. 空间充足：使用完整标签
if (totalIdealWidth <= availableForTabs) {
  return tabHeaders;
}

// 3. 空间不足：优先保证当前标签，其他均分剩余空间
const currentHeader = tabHeaders[currentQuestionIndex] || "";
const currentIdealWidth = 4 + stringWidth(currentHeader);
const currentTabWidth = Math.min(currentIdealWidth, availableForTabs / 2);
const remainingWidth = availableForTabs - currentTabWidth;
const otherTabCount = questions.length - 1;
const widthPerOtherTab = Math.max(6, Math.floor(remainingWidth / Math.max(otherTabCount, 1)));

// 4. 应用截断
return tabHeaders.map((header, index) => {
  const maxTextWidth = index === currentQuestionIndex
    ? currentTabWidth - 4  // 当前标签
    : widthPerOtherTab - 4; // 其他标签
  return truncateToWidth(header, maxTextWidth);
});
```

### 3.4 标签渲染逻辑

```javascript
questions.map((q, index) => {
  const isSelected = index === currentQuestionIndex;
  const isAnswered = q?.question && !!answers[q.question];
  const checkbox = isAnswered ? figures.checkboxOn : figures.checkboxOff;
  const displayText = tabDisplayTexts[index] || q?.header || `Q${index + 1}`;
  
  return (
    <Box key={q?.question || `question-${index}`}>
      {isSelected ? (
        <Text backgroundColor="permission" color="inverseText">
          {" "}{checkbox} {displayText}{" "}
        </Text>
      ) : (
        <Text>{" "}{checkbox} {displayText}{" "}</Text>
      )}
    </Box>
  );
});
```

### 3.5 箭头渲染逻辑

```javascript
// 左箭头
!hideArrows && (
  <Text color={currentQuestionIndex === 0 ? "inactive" : undefined}>
    ←{" "}
  </Text>
);

// 右箭头
!hideArrows && (
  <Text color={currentQuestionIndex === questions.length ? "inactive" : undefined}>
    {" "}→
  </Text>
);
```

### 3.6 提交标签渲染

```javascript
!hideSubmitTab && (
  <Box key="submit">
    {currentQuestionIndex === questions.length ? (
      <Text backgroundColor="permission" color="inverseText">
        {" "}{figures.tick} Submit{" "}
      </Text>
    ) : (
      <Text> {figures.tick} Submit </Text>
    )}
  </Box>
);
```

---

## 4. 关键代码路径与文件引用

### 4.1 组件文件
| 文件路径 | 职责 |
|---------|------|
| `src/components/permissions/AskUserQuestionPermissionRequest/QuestionNavigationBar.tsx` | 主组件实现 |

### 4.2 依赖工具
| 文件路径 | 功能 |
|---------|------|
| `src/hooks/useTerminalSize.ts` | `useTerminalSize()` - 终端宽度监听 |
| `src/ink/stringWidth.ts` | `stringWidth()` - 显示宽度计算 |
| `src/utils/format.ts` | `truncateToWidth()` - 文本截断 |

### 4.3 类型定义
| 文件路径 | 类型 |
|---------|------|
| `src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx` | `Question` 类型 |

### 4.4 调用链
```
AskUserQuestionPermissionRequest.tsx
├── QuestionView.tsx / PreviewQuestionView.tsx
│   └── QuestionNavigationBar (当前问题导航)
└── SubmitQuestionsView.tsx
    └── QuestionNavigationBar (提交页面导航)
```

---

## 5. 依赖与外部交互

### 5.1 React 依赖
- **Hooks**: `useMemo` - 缓存布局计算结果
- **React Compiler**: 使用 `_c(39)` 缓存机制（39 个缓存槽位）

### 5.2 Ink 组件
```typescript
import { Box, Text } from '../../../ink.js';
```

### 5.3 外部库
```typescript
import figures from 'figures';  // 终端符号（☐, ☑, ✓ 等）
```

### 5.4 工具函数
```typescript
import { useTerminalSize } from '../../../hooks/useTerminalSize.js';
import { stringWidth } from '../../../ink/stringWidth.js';
import { truncateToWidth } from '../../../utils/format.js';
```

### 5.5 类型导入
```typescript
import type { Question } from '../../../tools/AskUserQuestionTool/AskUserQuestionTool.js';
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 布局计算复杂度
- **缓存依赖多**：布局计算依赖 `columns`, `currentQuestionIndex`, `hideSubmitTab`, `questions` 四个变量
- **频繁重计算**：终端宽度变化时触发完整重新计算

#### 6.1.2 极端宽度情况
- **极小终端**：当 `availableForTabs <= 0` 时，回退到只显示当前问题缩略
- **截断过度**：非当前问题的标签可能被截断到只剩 "..."

#### 6.1.3 标签文本来源不一致
```javascript
// 截断时使用 tabHeaders（来自 header 或 Q{N}）
const displayText = tabDisplayTexts[index] || q?.header || `Q${index + 1}`;
```
这里存在潜在的不一致：`tabHeaders` 和 fallback 逻辑可能产生不同结果。

### 6.2 边界情况

| 场景 | 当前行为 | 潜在问题 |
|-----|---------|---------|
| `availableForTabs <= 0` | 只显示当前问题的前 3 个字符 | 用户无法看到其他问题 |
| 问题 header 超长 | 被截断 | 可能丢失关键信息 |
| 所有问题已回答 | 所有标签显示 ☑ | 视觉拥挤 |
| `hideSubmitTab=true` 且单问题 | 隐藏箭头 | 用户可能不知道可以导航 |
| `currentQuestionIndex` 越界 | 可能渲染异常 | 需要边界检查 |

### 6.3 改进建议

#### 6.3.1 代码结构优化
```typescript
// 建议：提取布局计算为独立函数，便于测试
function calculateTabLayout(
  questions: Question[],
  currentIndex: number,
  availableWidth: number,
  hideSubmitTab: boolean
): TabLayout {
  // 布局计算逻辑
}

// 建议：使用配置对象替代魔法数字
const TAB_PADDING = 4;  // 2 spaces + checkbox + space
const MIN_TAB_WIDTH = 6; // 最小可识别宽度
```

#### 6.3.2 性能优化
```typescript
// 当前：每次渲染都重新计算布局
// 建议：添加深度比较，questions 引用不变时跳过计算
const questionsKey = useMemo(() => 
  questions.map(q => q.header || q.question.slice(0, 20)).join('|'),
  [questions]
);
```

#### 6.3.3 可访问性改进
- 添加当前问题索引的文本提示（如 "Question 2 of 4"）
- 为色盲用户提供额外的视觉指示（如 `*` 标记当前）

#### 6.3.4 功能扩展
- **悬停提示**：当标签被截断时显示完整文本 tooltip
- **进度百分比**：添加进度条或百分比显示
- **问题分组**：支持问题分组和子导航

### 6.4 测试建议

#### 6.4.1 单元测试
```typescript
describe('QuestionNavigationBar', () => {
  it('should highlight current question', () => {});
  it('should show checkmark for answered questions', () => {});
  it('should truncate tabs when space is limited', () => {});
  it('should prioritize current question width', () => {});
  it('should hide arrows when appropriate', () => {});
});
```

#### 6.4.2 边界测试
- 终端宽度 80/40/20/10 列的不同表现
- 1/2/3/4 个问题的布局
- 超长/超短 header 的显示

### 6.5 代码审查点

#### 6.5.1 潜在 Bug
```javascript
// 行 33-55：availableForTabs <= 0 时的处理
if (availableForTabs <= 0) {
  t3 = questions.map((q, index) => {
    const header = q?.header || `Q${index + 1}`;
    return index === currentQuestionIndex ? header.slice(0, 3) : "";
  });
}
```
问题：当 `availableForTabs <= 0` 时，非当前问题返回空字符串 `""`，这会导致渲染空标签，可能产生布局问题。

#### 6.5.2 不一致的 key 生成
```javascript
// 行 118
<Box key={q_1?.question || `question-${index_2}`}>
```
使用 `question` 文本作为 key 在问题文本变化时会导致重新渲染，但这是预期行为。

#### 6.5.3 React Compiler 产物
代码是 React Compiler 编译后的产物，包含大量缓存检查：
```javascript
if ($[0] !== columns || $[1] !== currentQuestionIndex || ...) {
  // 重新计算
}
```
修改源码后需要重新编译。

---

## 7. 相关类型定义

```typescript
// 来自 AskUserQuestionTool.tsx
interface Question {
  question: string;      // 完整问题文本
  header: string;        // 短标题（用于导航栏）
  options: QuestionOption[];
  multiSelect?: boolean;
}

// 组件内部使用的布局类型
interface TabLayoutResult {
  tabDisplayTexts: string[];
  hideArrows: boolean;
}
```

---

## 8. 总结

`QuestionNavigationBar` 是一个精巧的导航组件，通过智能的布局算法在有限的终端空间内提供清晰的问题导航体验：

### 8.1 设计亮点
1. **空间自适应**：根据终端宽度动态调整标签显示
2. **优先级策略**：当前问题优先获得显示空间
3. **状态可视化**：通过复选框和颜色直观展示回答状态
4. **轻量实现**：无子组件依赖，纯计算+渲染

### 8.2 核心算法
- **两阶段布局**：先计算理想宽度，再处理空间不足情况
- **截断策略**：`truncateToWidth` 确保 grapheme 安全
- **缓存优化**：React Compiler 自动处理 39 个缓存点

### 8.3 注意事项
- 极端终端宽度下的降级处理需要验证
- `availableForTabs <= 0` 时的空字符串标签可能有问题
- 布局计算依赖较多变量，需要确保缓存有效性

该组件虽然代码量不大，但在多问题问卷场景中承担了关键的导航和状态展示职责，是用户体验的重要组成部分。
