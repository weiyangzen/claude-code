# SkillImprovementSurvey.tsx 研究文档

## 场景与职责

`SkillImprovementSurvey` 是一个技能改进建议调查组件，用于在 AI 检测到用户对当前执行技能的改进建议时，向用户展示这些建议并收集反馈。它是 Claude Code CLI 技能系统的一部分，支持用户通过简单的数字输入（0/1）快速响应。

**使用场景：**
- 主 REPL 屏幕 (`REPL.tsx`) 中的技能改进提示
- 用户使用技能过程中 AI 检测到可改进点时
- 用户通过 `useSkillImprovementSurvey` hook 触发的调查

## 功能点目的

### 1. 技能改进建议展示
- 显示技能名称和改进建议列表
- 每项建议包含变更描述

### 2. 快速反馈收集
- 支持数字输入（1=应用，0=忽略）
- 实时响应用户输入

### 3. 输入验证
- 验证输入是否为有效数字（0 或 1）
- 支持全角数字自动转换

### 4. 条件渲染
- 仅在 `isOpen=true` 时显示
- 无效输入时隐藏（防止错误渲染）

## 具体技术实现

### 关键数据结构

```typescript
// 主组件 Props
type Props = {
  isOpen: boolean;                                    // 是否打开
  skillName: string;                                  // 技能名称
  updates: SkillUpdate[];                             // 改进建议列表
  handleSelect: (selected: FeedbackSurveyResponse) => void;  // 选择回调
  inputValue: string;                                 // 当前输入值
  setInputValue: (value: string) => void;             // 设置输入值
};

// SkillUpdate 类型（来自 skillImprovement.ts）
type SkillUpdate = {
  section: string;    // 要修改的部分
  change: string;     // 变更描述
  reason: string;     // 原因/依据
};

// 视图组件 Props
type ViewProps = {
  skillName: string;
  updates: SkillUpdate[];
  onSelect: (option: FeedbackSurveyResponse) => void;
  inputValue: string;
  setInputValue: (value: string) => void;
};
```

### 有效输入定义

```typescript
// 仅接受 1（应用）和 0（忽略）
const VALID_INPUTS = ['0', '1'] as const;
type ResponseInput = (typeof VALID_INPUTS)[number];

function isValidInput(input: string): boolean {
  return (VALID_INPUTS as readonly string[]).includes(input);
}
```

### 输入处理逻辑

```typescript
const initialInputValue = useRef(inputValue);

useEffect(() => {
  if (inputValue !== initialInputValue.current) {
    // 获取最后一个字符并转换为半角
    const lastChar = normalizeFullWidthDigits(inputValue.slice(-1));
    
    if (isValidInput(lastChar)) {
      // 移除已处理的字符
      setInputValue(inputValue.slice(0, -1));
      // 1=good（应用），其他=dismissed（忽略）
      onSelect(lastChar === "1" ? "good" : "dismissed");
    }
  }
}, [inputValue, onSelect, setInputValue]);
```

### 渲染结构

```typescript
// 标题行
<Box>
  <Text color="ansi:cyan">{BLACK_CIRCLE} </Text>
  <Text bold>Skill improvement suggested for "{skillName}"</Text>
</Box>

// 建议列表
<Box flexDirection="column" marginLeft={2}>
  {updates.map((u, i) => (
    <Text key={i} dimColor>{BULLET_OPERATOR} {u.change}</Text>
  ))}
</Box>

// 操作提示
<Box marginLeft={2} marginTop={1}>
  <Box width={12}>
    <Text><Text color="ansi:cyan">1</Text>: Apply</Text>
  </Box>
  <Box width={14}>
    <Text><Text color="ansi:cyan">0</Text>: Dismiss</Text>
  </Box>
</Box>
```

### 条件渲染逻辑

```typescript
export function SkillImprovementSurvey(props: Props) {
  // 未打开时不渲染
  if (!props.isOpen) {
    return null;
  }
  
  // 输入无效时不渲染（防止错误状态）
  if (props.inputValue && !isValidResponseInput(props.inputValue)) {
    return null;
  }
  
  return <SkillImprovementSurveyView {...props} />;
}
```

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/components/SkillImprovementSurvey.tsx` - 组件实现

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/screens/REPL.tsx` - 主 REPL 屏幕
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useSkillImprovementSurvey.ts` - Skill Improvement Survey Hook

### 依赖文件
- `/home/sansha/Github/claude-code-instructkr/src/utils/hooks/skillImprovement.ts` - 技能改进核心逻辑
  - `SkillUpdate` 类型定义
  - `applySkillImprovement()` - 应用技能改进
- `/home/sansha/Github/claude-code-instructkr/src/utils/stringUtils.ts` - 字符串工具
  - `normalizeFullWidthDigits()` - 全角数字转半角
- `/home/sansha/Github/claude-code-instructkr/src/components/FeedbackSurvey/FeedbackSurveyView.ts` - 反馈调查视图
  - `isValidResponseInput()` - 输入验证
- `/home/sansha/Github/claude-code-instructkr/src/components/FeedbackSurvey/utils.ts` - 反馈调查工具
  - `FeedbackSurveyResponse` 类型

### 依赖组件
- `../ink.js` - Box, Text

### 常量
- `/home/sansha/Github/claude-code-instructkr/src/constants/figures.js` - 图形符号
  - `BLACK_CIRCLE` (●)
  - `BULLET_OPERATOR` (∙)

## 依赖与外部交互

### 技能改进系统
- **SkillUpdate**: 包含改进建议的数据结构
- **applySkillImprovement**: 异步应用改进到技能文件
- **useSkillImprovementSurvey**: 管理调查状态和逻辑的 hook

### 输入处理
- **normalizeFullWidthDigits**: 将全角数字（０-９）转换为半角（0-9），支持 CJK 输入法
- **isValidResponseInput**: 验证输入是否为有效响应（0 或 1）

### 状态管理
- **AppState**: 通过 `useAppState` 和 `useSetAppState` 管理技能改进建议状态
- **技能建议状态**: `state.skillImprovement.suggestion`

### 反馈调查系统
- **FeedbackSurveyResponse**: 'good' | 'dismissed' 等响应类型
- 与通用的反馈调查组件共享验证逻辑

## 风险、边界与改进建议

### 边界情况

1. **输入处理**: 只处理输入值的最后一个字符，之前的字符被忽略
2. **全角转换**: 支持 CJK 用户的全角数字输入自动转换
3. **初始值引用**: 使用 `useRef` 存储初始输入值，避免重复处理
4. **空建议列表**: 如果 `updates` 为空数组，仍显示标题和操作提示

### 潜在风险

1. **竞态条件**: 如果用户快速输入多个数字，可能只处理最后一个
2. **输入丢失**: `setInputValue(inputValue.slice(0, -1))` 可能与其他输入处理冲突
3. **硬编码响应值**: "1" 和 "0" 是硬编码的，与其他调查组件（使用 0-3）不一致
4. **缺少确认**: 用户输入后立即执行，没有二次确认机制

### 改进建议

1. **输入队列处理**:
   ```typescript
   // 处理所有累积的输入字符
   const newChars = inputValue.slice(initialInputValue.current.length);
   for (const char of newChars) {
     const normalized = normalizeFullWidthDigits(char);
     if (isValidInput(normalized)) {
       onSelect(normalized === "1" ? "good" : "dismissed");
       break;  // 处理第一个有效输入
     }
   }
   ```

2. **添加撤销机制**:
   - 应用改进后显示确认消息
   - 提供撤销操作（如按 'u' 撤销）

3. **统一响应值**:
   - 与其他调查组件保持一致，支持 0-3 的评分
   - 或明确区分简单二元选择和详细评分

4. **增强 UX**:
   - 添加倒计时自动关闭
   - 显示改进的详细说明（reason 字段）
   - 支持查看修改前后的对比

5. **错误处理**:
   - 处理 `applySkillImprovement` 失败的情况
   - 显示错误提示和重试选项

6. **可访问性**:
   - 添加屏幕阅读器支持
   - 提供键盘导航的明确指示

7. **分析追踪**:
   - 记录用户响应时间
   - 追踪改进接受率
   - 收集用户反馈文本（可选）

8. **代码重构**:
   - 提取输入处理逻辑为独立 hook
   - 将视图组件与容器组件分离
