# FeedbackSurveyView.tsx 研究文档

## 场景与职责

`FeedbackSurveyView.tsx` 是 Claude Code CLI 反馈调查系统的核心 UI 组件，负责渲染会话质量调查的交互界面。它是 `FeedbackSurvey.tsx` 的子组件，专门处理用户评分输入的展示和收集。

该组件的主要职责：
1. **渲染调查问卷界面**: 显示调查问题（默认或自定义消息）和评分选项
2. **处理数字输入**: 捕获用户输入的数字（0-3）并映射为反馈响应
3. **提供视觉反馈**: 使用颜色编码（青色）突出显示可选项
4. **防抖处理**: 集成 `useDebouncedDigitInput` 防止意外触发

## 功能点目的

### 1. 评分选项映射
将数字输入映射为语义化的反馈响应：
- `0` → `dismissed` (关闭/忽略调查)
- `1` → `bad` (差评)
- `2` → `fine` (一般)
- `3` → `good` (好评)

### 2. 可自定义的调查消息
支持通过 `message` 属性自定义调查问题，默认消息为：
> "How is Claude doing this session? (optional)"

### 3. 输入防抖机制
使用 `useDebouncedDigitInput` Hook 防止用户输入编号列表（如 "1. First item"）时意外触发评分选择。

### 4. 输入验证
通过 `isValidResponseInput` 函数验证输入是否为有效数字（0-3），该函数被导出供父组件使用。

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
type Props = {
  onSelect: (option: FeedbackSurveyResponse) => void;
  inputValue: string;
  setInputValue: (value: string) => void;
  message?: string;
};

// 响应输入类型（字面量类型）
const RESPONSE_INPUTS = ['0', '1', '2', '3'] as const;
type ResponseInput = (typeof RESPONSE_INPUTS)[number];

// 输入到响应的映射表
const inputToResponse: Record<ResponseInput, FeedbackSurveyResponse> = {
  '0': 'dismissed',
  '1': 'bad',
  '2': 'fine',
  '3': 'good'
} as const;
```

### 关键流程

#### 1. 输入处理流程
```
用户输入字符 → useDebouncedDigitInput 防抖（400ms）
  → 验证是否为有效数字（isValidResponseInput）
  → 从输入中移除该数字（setInputValue(trimmed)）
  → 映射为响应类型（inputToResponse[digit]）
  → 调用 onSelect 回调
```

#### 2. 渲染结构
```
Box (flexDirection="column", marginTop=1)
  ├── Box
  │     ├── Text (color="ansi:cyan") ●
  │     └── Text (bold) {message}
  └── Box (marginLeft=2)
        ├── Box (width=10) Text: 1: Bad
        ├── Box (width=10) Text: 2: Fine
        ├── Box (width=10) Text: 3: Good
        └── Box Text: 0: Dismiss
```

### 输入验证实现

```typescript
// 行 20
export const isValidResponseInput = (input: string): input is ResponseInput => 
  (RESPONSE_INPUTS as readonly string[]).includes(input);
```

该函数使用 TypeScript 类型谓词（type predicate）确保类型安全，将字符串输入缩小为 `ResponseInput` 类型。

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `./useDebouncedDigitInput.js` | 防抖数字输入处理 Hook |
| `./utils.js` | FeedbackSurveyResponse 类型定义 |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `src/ink.js` | Ink 渲染组件 (Box, Text) |

### 关键代码片段

#### 防抖输入配置
```typescript
// 行 31-54
useDebouncedDigitInput({
  inputValue,
  setInputValue,
  isValidDigit: isValidResponseInput,
  onDigit: digit => onSelect(inputToResponse[digit])
});
```

#### UI 渲染
```typescript
// 行 62-106
<Box flexDirection="column" marginTop={1}>
  <Box>
    <Text color="ansi:cyan">● </Text>
    <Text bold>{message}</Text>
  </Box>
  <Box marginLeft={2}>
    <Box width={10}><Text><Text color="ansi:cyan">1</Text>: Bad</Text></Box>
    <Box width={10}><Text><Text color="ansi:cyan">2</Text>: Fine</Text></Box>
    <Box width={10}><Text><Text color="ansi:cyan">3</Text>: Good</Text></Box>
    <Box><Text><Text color="ansi:cyan">0</Text>: Dismiss</Text></Box>
  </Box>
</Box>
```

### React Compiler 优化
代码使用 React Compiler 自动优化：
- 使用 `_c(15)` 创建 15 个缓存槽位的 memoization cache
- 静态内容（如选项文本）使用 `Symbol.for("react.memo_cache_sentinel")` 标记
- 动态内容（如 `message`）基于依赖变化决定是否更新

## 依赖与外部交互

### 输入依赖
1. **输入值**: `inputValue` - 当前输入框内容
2. **状态设置器**: `setInputValue` - 用于清除已处理的数字
3. **选择回调**: `onSelect` - 当用户选择选项时调用
4. **自定义消息**: `message` - 可选的调查问题文本

### 输出交互
1. **回调调用**: 通过 `onSelect` 返回 `FeedbackSurveyResponse` 类型值
2. **输入修改**: 通过 `setInputValue` 移除已处理的数字

### 类型依赖关系
```
FeedbackSurveyView.tsx
  ├── FeedbackSurveyResponse (from ./utils.js)
  └── ResponseInput (本地定义)
```

## 风险、边界与改进建议

### 已知风险

1. **硬编码布局尺寸**
   - 行 72, 78, 84: `width={10}` 固定宽度
   - 如果选项文本翻译为其他语言，可能溢出或显示不完整

2. **默认消息硬编码**
   - 行 21: `const DEFAULT_MESSAGE = 'How is Claude doing this session? (optional)';`
   - 不支持国际化（i18n）

3. **颜色硬编码**
   - 使用 `ansi:cyan` 作为强调色
   - 在某些终端主题下可能对比度不足

### 边界情况

1. **快速连续输入**: 防抖机制确保只有停顿 400ms 后的输入才被处理
2. **非数字输入**: 通过 `isValidDigit` 过滤，非 0-3 的数字被忽略
3. **空输入**: 组件正常渲染，等待有效输入
4. **输入包含多个数字**: 仅处理最后一个字符

### 改进建议

1. **支持国际化**
   ```typescript
   // 建议：通过配置或上下文获取本地化文本
   const DEFAULT_MESSAGE = t('feedback.survey.default_message');
   const OPTIONS = {
     '1': t('feedback.survey.bad'),
     '2': t('feedback.survey.fine'),
     '3': t('feedback.survey.good'),
     '0': t('feedback.survey.dismiss')
   };
   ```

2. **动态布局**
   ```typescript
   // 建议：根据内容自动调整宽度
   <Box flexDirection="row" gap={2}>
     {OPTIONS.map(([key, label]) => (
       <Box key={key}><Text>...</Text></Box>
     ))}
   </Box>
   ```

3. **增强可访问性**
   - 添加键盘导航支持（上下箭头键选择）
   - 为屏幕阅读器添加适当的标签

4. **提取静态内容为配置**
   ```typescript
   // 建议：将选项配置提取为常量
   export const SURVEY_OPTIONS = [
     { key: '1', value: 'bad', label: 'Bad' },
     { key: '2', value: 'fine', label: 'Fine' },
     { key: '3', value: 'good', label: 'Good' },
     { key: '0', value: 'dismissed', label: 'Dismiss' }
   ] as const;
   ```

5. **优化 React Compiler 缓存**
   - 当前使用 15 个缓存槽位，部分静态内容可以进一步合并
   - 考虑将选项渲染提取为独立组件

### 测试建议

1. **输入映射测试**: 验证 0-3 分别正确映射为对应响应
2. **防抖测试**: 验证 400ms 防抖延迟行为
3. **回调测试**: 验证 `onSelect` 和 `setInputValue` 被正确调用
4. **自定义消息测试**: 验证 `message` 属性覆盖默认消息
5. **类型测试**: 验证 `isValidResponseInput` 的类型谓词行为
