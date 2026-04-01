# TranscriptSharePrompt.tsx 研究文档

## 场景与职责

`TranscriptSharePrompt.tsx` 是 Claude Code CLI 反馈调查系统的转录分享提示组件，负责在用户完成会话质量调查后，询问用户是否愿意分享会话转录给 Anthropic 以改进 Claude Code。

该组件的主要职责：
1. **请求转录分享权限**: 向用户解释数据用途并请求分享同意
2. **提供多种响应选项**: 支持同意、拒绝或永久不再询问
3. **隐私透明**: 提供文档链接让用户了解数据使用政策
4. **一致的交互体验**: 使用与主调查相同的数字输入模式

## 功能点目的

### 1. 数据收集同意机制
在收集用户会话数据前获取明确的用户同意，符合隐私保护最佳实践：
- **Yes**: 同意分享当前会话转录
- **No**: 拒绝分享，但未来可能再次询问
- **Don't ask again**: 永久拒绝，不再显示此提示

### 2. 隐私政策透明
- 显示明确的问题说明："Can Anthropic look at your session transcript to help us improve Claude Code?"
- 提供官方文档链接：https://code.claude.com/docs/en/data-usage#session-quality-surveys

### 3. 响应映射
将数字输入映射为语义化的分享响应：
- `1` → `yes` (同意分享)
- `2` → `no` (拒绝分享)
- `3` → `dont_ask_again` (不再询问)

### 4. 视觉一致性
- 使用 `BLACK_CIRCLE` 符号作为提示标记
- 与主调查问卷相同的颜色方案（青色强调）
- 统一的布局结构

## 具体技术实现

### 关键数据结构

```typescript
// 分享响应类型定义
export type TranscriptShareResponse = 'yes' | 'no' | 'dont_ask_again';

// Props 定义
type Props = {
  onSelect: (option: TranscriptShareResponse) => void;
  inputValue: string;
  setInputValue: (value: string) => void;
};

// 响应输入类型
const RESPONSE_INPUTS = ['1', '2', '3'] as const;
type ResponseInput = (typeof RESPONSE_INPUTS)[number];

// 输入到响应的映射表
const inputToResponse: Record<ResponseInput, TranscriptShareResponse> = {
  '1': 'yes',
  '2': 'no',
  '3': 'dont_ask_again'
} as const;
```

### 关键流程

#### 1. 输入处理流程
```
用户输入字符 → useDebouncedDigitInput 防抖（默认 400ms）
  → 验证是否为有效数字（isValidResponseInput: 1-3）
  → 从输入中移除该数字
  → 映射为分享响应类型
  → 调用 onSelect 回调
```

#### 2. 渲染结构
```
Box (flexDirection="column", marginTop=1)
  ├── Box
  │     ├── Text (color="ansi:cyan") {BLACK_CIRCLE}
  │     └── Text (bold) Can Anthropic look at your session transcript...
  ├── Box (marginLeft=2)
  │     └── Text (dimColor) Learn more: https://code.claude.com/docs/...
  └── Box (marginLeft=2)
        ├── Box (width=10) Text: 1: Yes
        ├── Box (width=10) Text: 2: No
        └── Box Text: 3: Don't ask again
```

### 输入验证实现

```typescript
// 行 19
const isValidResponseInput = (input: string): input is ResponseInput => 
  (RESPONSE_INPUTS as readonly string[]).includes(input);
```

与 `FeedbackSurveyView.tsx` 类似，使用 TypeScript 类型谓词确保类型安全。

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `./useDebouncedDigitInput.js` | 防抖数字输入处理 Hook |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `src/constants/figures.js` | 符号常量 (BLACK_CIRCLE) |
| `src/ink.js` | Ink 渲染组件 (Box, Text) |

### 关键代码片段

#### 防抖输入配置
```typescript
// 行 27-50
useDebouncedDigitInput({
  inputValue,
  setInputValue,
  isValidDigit: isValidResponseInput,
  onDigit: t1  // digit => onSelect(inputToResponse[digit])
});
```

#### UI 渲染
```typescript
// 行 51-86
<Box flexDirection="column" marginTop={1}>
  <Box>
    <Text color="ansi:cyan">{BLACK_CIRCLE} </Text>
    <Text bold>Can Anthropic look at your session transcript...</Text>
  </Box>
  <Box marginLeft={2}>
    <Text dimColor>Learn more: https://code.claude.com/docs/...</Text>
  </Box>
  <Box marginLeft={2}>
    <Box width={10}><Text><Text color="ansi:cyan">1</Text>: Yes</Text></Box>
    <Box width={10}><Text><Text color="ansi:cyan">2</Text>: No</Text></Box>
    <Box><Text><Text color="ansi:cyan">3</Text>: Don't ask again</Text></Box>
  </Box>
</Box>
```

### React Compiler 优化
代码使用 React Compiler 自动优化：
- 使用 `_c(11)` 创建 11 个缓存槽位的 memoization cache
- 静态内容（标题、选项、链接）使用 `Symbol.for("react.memo_cache_sentinel")` 标记
- 动态回调通过依赖比较决定是否更新

## 依赖与外部交互

### 输入依赖
1. **输入值**: `inputValue` - 当前输入框内容
2. **状态设置器**: `setInputValue` - 用于清除已处理的数字
3. **选择回调**: `onSelect` - 当用户选择选项时调用，返回 `TranscriptShareResponse`

### 输出交互
1. **回调调用**: 通过 `onSelect` 返回 `TranscriptShareResponse` 类型值
2. **输入修改**: 通过 `setInputValue` 移除已处理的数字

### 类型导出
```typescript
export type { TranscriptShareResponse } from './TranscriptSharePrompt.js';
```

该类型被以下文件使用：
- `FeedbackSurvey.tsx`: Props 类型定义
- `useSurveyState.tsx`: 回调函数类型
- `useFeedbackSurvey.tsx`: 处理函数类型
- `useMemorySurvey.tsx`: 处理函数类型

## 风险、边界与改进建议

### 已知风险

1. **硬编码文档链接**
   - 行 52: `https://code.claude.com/docs/en/data-usage#session-quality-surveys`
   - 链接变更需要代码更新
   - 不支持多语言文档

2. **选项 3 的持久化逻辑在父组件**
   - `dont_ask_again` 的实际处理（写入全局配置）在 `useFeedbackSurvey.tsx` 中
   - 组件本身只负责触发回调，不处理业务逻辑

3. **符号显示兼容性**
   - `BLACK_CIRCLE` 在 macOS 上显示为 `⏺`，其他平台为 `●`
   - 某些终端可能无法正确显示这些符号

### 边界情况

1. **输入范围限制**: 仅接受 1-3，0 被忽略（与主调查不同）
2. **快速连续输入**: 防抖机制确保只有停顿后的输入才被处理
3. **空输入**: 组件正常渲染，等待有效输入

### 改进建议

1. **配置化文档链接**
   ```typescript
   // 建议：从配置或环境变量获取
   const DOCS_URL = process.env.CLAUDE_DOCS_URL || 
     'https://code.claude.com/docs/en/data-usage#session-quality-surveys';
   ```

2. **支持国际化**
   ```typescript
   // 建议：文本内容配置化
   const MESSAGES = {
     title: t('transcript_share.title'),
     options: {
       yes: t('transcript_share.yes'),
       no: t('transcript_share.no'),
       dontAskAgain: t('transcript_share.dont_ask_again')
     },
     learnMore: t('transcript_share.learn_more')
   };
   ```

3. **增强可访问性**
   - 添加选项编号的高对比度显示
   - 考虑为色盲用户提供额外的视觉提示（如符号前缀）

4. **提取静态配置**
   ```typescript
   // 建议：与 FeedbackSurveyView 共享配置模式
   export const TRANSCRIPT_SHARE_OPTIONS = [
     { key: '1', value: 'yes', label: 'Yes' },
     { key: '2', value: 'no', label: 'No' },
     { key: '3', value: 'dont_ask_again', label: "Don't ask again" }
   ] as const;
   ```

5. **优化布局适应性**
   ```typescript
   // 建议：使用 flex 布局替代固定宽度
   <Box flexDirection="row" gap={2}>
     {OPTIONS.map(opt => (
       <Box key={opt.key}><Text>...</Text></Box>
     ))}
   </Box>
   ```

### 测试建议

1. **响应映射测试**: 验证 1-3 分别正确映射为对应响应
2. **防抖测试**: 验证防抖延迟行为
3. **回调测试**: 验证 `onSelect` 和 `setInputValue` 被正确调用
4. **类型测试**: 验证 `TranscriptShareResponse` 类型导出正确
5. **渲染测试**: 验证所有静态文本正确显示
