# useDebouncedDigitInput.ts 研究文档

## 场景与职责

`useDebouncedDigitInput.ts` 是 Claude Code CLI 反馈调查系统的核心交互 Hook，负责检测并处理用户在主输入框中输入的数字字符。该 Hook 解决了用户在输入编号列表（如 "1. First item"）时意外触发调查选项选择的问题。

该 Hook 的主要职责：
1. **防抖处理**: 延迟响应数字输入，避免意外触发
2. **输入验证**: 验证输入字符是否为有效数字
3. **自动清理**: 从输入中移除已处理的数字
4. **灵活配置**: 支持启用/禁用、单次触发等选项

## 功能点目的

### 1. 防止意外触发
用户在 CLI 中输入内容时，可能会以数字开头（如编号列表）。防抖机制确保：
- 用户停顿 400ms 后才视为有意输入
- 快速连续输入时自动取消之前的待处理操作

### 2. 全角数字支持
通过 `normalizeFullWidthDigits` 支持日文/中文 IME 输入的全角数字（０-９），提升国际化体验。

### 3. 灵活的验证机制
通过 `isValidDigit` 回调允许调用者定义什么是"有效数字"：
- 主调查：0-3 有效
- 转录分享提示：1-3 有效
- 后续反馈：仅 1 有效

### 4. 单次触发模式
`once: true` 模式确保特定操作只执行一次（如感谢页面的后续反馈入口）。

## 具体技术实现

### 关键数据结构

```typescript
// Hook 配置选项
interface UseDebouncedDigitInputOptions<T extends string> {
  inputValue: string;                    // 当前输入值
  setInputValue: (value: string) => void; // 设置输入值的函数
  isValidDigit: (char: string) => char is T; // 数字验证函数
  onDigit: (digit: T) => void;           // 数字确认后的回调
  enabled?: boolean;                     // 是否启用（默认 true）
  once?: boolean;                        // 是否只触发一次（默认 false）
  debounceMs?: number;                   // 防抖延迟（默认 400ms）
}

// 默认防抖延迟
const DEFAULT_DEBOUNCE_MS = 400;
```

### 关键流程

#### 1. 初始化流程
```
useDebouncedDigitInput(options)
  ├── 创建 refs: initialInputValue, hasTriggeredRef, debounceRef
  ├── 创建 callbacksRef（Latest-ref pattern）
  └── 返回 void
```

#### 2. 输入处理流程
```
inputValue 变化 → useEffect 触发
  ├── 检查 enabled 和 once 条件
  ├── 清除之前的定时器
  ├── 比较当前值与 initialInputValue
  ├── 提取最后一个字符并全角转半角
  ├── 验证是否为有效数字
  │     └── 是 → 设置定时器
  └── 返回清理函数（清除定时器）
```

#### 3. 定时器触发流程
```
定时器触发（debounceMs 后）
  ├── 清除定时器引用
  ├── 标记已触发（hasTriggeredRef.current = true）
  ├── 从输入中移除该数字（setInputValue(trimmed)）
  └── 调用 onDigit 回调
```

### Latest-ref Pattern 实现

```typescript
// 行 41-42
const callbacksRef = useRef({ setInputValue, isValidDigit, onDigit });
callbacksRef.current = { setInputValue, isValidDigit, onDigit };
```

该模式确保：
- 调用者可以传递内联回调函数而不会导致 effect 重新运行
- 始终使用最新的回调函数
- 防抖定时器不会被不必要的重置

### 全角数字转换

```typescript
// 行 55
const lastChar = normalizeFullWidthDigits(inputValue.slice(-1));
```

`normalizeFullWidthDigits` 函数（来自 `../../utils/stringUtils.js`）：
```typescript
export function normalizeFullWidthDigits(input: string): string {
  return input.replace(/[０-９]/g, ch =>
    String.fromCharCode(ch.charCodeAt(0) - 0xfee0),
  );
}
```

## 关键代码路径与文件引用

### 外部依赖
| 文件 | 用途 |
|------|------|
| `react` | useEffect, useRef |
| `../../utils/stringUtils.js` | normalizeFullWidthDigits |

### 关键代码片段

#### Hook 实现
```typescript
// 行 18-82
export function useDebouncedDigitInput<T extends string = string>({
  inputValue,
  setInputValue,
  isValidDigit,
  onDigit,
  enabled = true,
  once = false,
  debounceMs = DEFAULT_DEBOUNCE_MS,
}: UseDebouncedDigitInputOptions<T>): void {
  const initialInputValue = useRef(inputValue);
  const hasTriggeredRef = useRef(false);
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  
  // Latest-ref pattern
  const callbacksRef = useRef({ setInputValue, isValidDigit, onDigit });
  callbacksRef.current = { setInputValue, isValidDigit, onDigit };
  
  useEffect(() => {
    if (!enabled || (once && hasTriggeredRef.current)) {
      return;
    }
    
    // 清除之前的定时器
    if (debounceRef.current !== null) {
      clearTimeout(debounceRef.current);
      debounceRef.current = null;
    }
    
    // 检查输入变化并验证最后一个字符
    if (inputValue !== initialInputValue.current) {
      const lastChar = normalizeFullWidthDigits(inputValue.slice(-1));
      if (callbacksRef.current.isValidDigit(lastChar)) {
        const trimmed = inputValue.slice(0, -1);
        debounceRef.current = setTimeout(
          (debounceRef, hasTriggeredRef, callbacksRef, trimmed, lastChar) => {
            debounceRef.current = null;
            hasTriggeredRef.current = true;
            callbacksRef.current.setInputValue(trimmed);
            callbacksRef.current.onDigit(lastChar);
          },
          debounceMs,
          debounceRef, hasTriggeredRef, callbacksRef, trimmed, lastChar
        );
      }
    }
    
    return () => {
      if (debounceRef.current !== null) {
        clearTimeout(debounceRef.current);
        debounceRef.current = null;
      }
    };
  }, [inputValue, enabled, once, debounceMs]);
}
```

## 依赖与外部交互

### 输入依赖
1. **inputValue**: 当前输入框内容，用于检测变化
2. **setInputValue**: 用于清除已处理的数字
3. **isValidDigit**: 验证函数，确定字符是否为有效数字
4. **onDigit**: 回调函数，在数字确认后调用
5. **enabled**: 控制 Hook 是否活跃
6. **once**: 控制是否只触发一次
7. **debounceMs**: 自定义防抖延迟

### 输出交互
1. **输入修改**: 通过 `setInputValue` 移除已处理的数字
2. **回调调用**: 通过 `onDigit` 返回确认的数字

### 使用场景

#### 1. 主调查问卷（FeedbackSurveyView.tsx）
```typescript
useDebouncedDigitInput({
  inputValue,
  setInputValue,
  isValidDigit: isValidResponseInput,  // 接受 0-3
  onDigit: digit => onSelect(inputToResponse[digit])
});
```

#### 2. 转录分享提示（TranscriptSharePrompt.tsx）
```typescript
useDebouncedDigitInput({
  inputValue,
  setInputValue,
  isValidDigit: isValidResponseInput,  // 接受 1-3
  onDigit: digit => onSelect(inputToResponse[digit])
});
```

#### 3. 感谢页面后续反馈（FeedbackSurvey.tsx）
```typescript
useDebouncedDigitInput({
  inputValue,
  setInputValue,
  isValidDigit: isFollowUpDigit,  // 仅接受 '1'
  enabled: Boolean(showFollowUp),
  once: true,  // 只触发一次
  onDigit: () => { ... }
});
```

## 风险、边界与改进建议

### 已知风险

1. **定时器参数传递方式**
   - 行 58-71: 使用 `setTimeout` 的参数传递方式传递 refs 和值
   - 这种方式在旧版 Node.js 中可能不被支持
   - 现代浏览器和 Node.js 支持，但不够直观

2. **initialInputValue 不更新**
   - `initialInputValue` 只在 Hook 首次渲染时设置
   - 如果输入值在 Hook 外部被重置，可能导致意外行为

3. **并发输入处理**
   - 如果用户在防抖期间继续输入多个数字，只有最后一个数字被处理
   - 这可能不是期望的行为（如快速输入 "12" 只会处理 "2"）

### 边界情况

1. **空输入**: 如果 `inputValue` 为空，不会触发任何处理
2. **非单字符变化**: 如果输入一次性变化多个字符（如粘贴），只处理最后一个字符
3. **组件卸载**: 清理函数确保定时器被清除，避免内存泄漏
4. **快速启用/禁用**: `enabled` 变化会触发 effect 清理和重新设置

### 改进建议

1. **简化定时器回调**
   ```typescript
   // 当前：使用 setTimeout 参数传递
   debounceRef.current = setTimeout(
     (debounceRef, hasTriggeredRef, callbacksRef, trimmed, lastChar) => { ... },
     debounceMs,
     debounceRef, hasTriggeredRef, callbacksRef, trimmed, lastChar
   );
   
   // 建议：使用闭包（更直观）
   debounceRef.current = setTimeout(() => {
     debounceRef.current = null;
     hasTriggeredRef.current = true;
     callbacksRef.current.setInputValue(trimmed);
     callbacksRef.current.onDigit(lastChar);
   }, debounceMs);
   ```

2. **支持多数字处理**
   ```typescript
   // 建议：处理输入中的所有有效数字
   const digits = inputValue.split('').filter(char => 
     callbacksRef.current.isValidDigit(normalizeFullWidthDigits(char))
   );
   if (digits.length > 0) {
     // 处理所有数字或仅最后一个，根据配置决定
   }
   ```

3. **添加重置机制**
   ```typescript
   // 建议：允许调用者重置触发状态
   const reset = useCallback(() => {
     hasTriggeredRef.current = false;
   }, []);
   
   // 返回 reset 函数供调用者使用
   return { reset };
   ```

4. **支持连续数字模式**
   ```typescript
   // 建议：添加模式支持多位数输入
   interface Options<T> {
     // ...现有选项
     allowMultiDigit?: boolean;  // 是否允许多位数
     maxDigits?: number;         // 最大位数
   }
   ```

5. **增强调试能力**
   ```typescript
   // 建议：添加调试日志
   if (process.env.DEBUG_FEEDBACK) {
     console.log('[useDebouncedDigitInput]', { 
       inputValue, lastChar, isValid: callbacksRef.current.isValidDigit(lastChar) 
     });
   }
   ```

6. **考虑使用 useCallback**
   ```typescript
   // 建议：将处理逻辑提取为 memoized 回调
   const handleDigit = useCallback((digit: T) => {
     const trimmed = inputValue.slice(0, -1);
     setInputValue(trimmed);
     onDigit(digit);
   }, [inputValue, setInputValue, onDigit]);
   ```

### 测试建议

1. **防抖延迟测试**:
   - 验证在延迟期间输入不会触发回调
   - 验证延迟后正确触发回调

2. **输入清理测试**:
   - 验证数字被正确从输入中移除
   - 验证非数字输入不受影响

3. **全角数字测试**:
   - 验证全角数字（０-９）被正确识别和处理

4. **once 模式测试**:
   - 验证 `once: true` 时只触发一次
   - 验证后续输入被忽略

5. **enabled 切换测试**:
   - 验证禁用时不处理输入
   - 验证启用后恢复正常处理

6. **边界测试**:
   - 空字符串输入
   - 超长字符串输入
   - 快速连续输入
