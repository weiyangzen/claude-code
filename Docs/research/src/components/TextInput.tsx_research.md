# TextInput.tsx 深度研究文档

## 场景与职责

TextInput 是 Claude Code CLI 中 **核心的文本输入组件**，用于用户命令输入、搜索、配置等多种交互场景。它是 PromptInput 的基础组件，支持丰富的输入功能和视觉效果。

该组件的核心职责：
1. **文本输入处理**：支持单行/多行文本输入
2. **光标效果**：标准反色光标 + 语音模式下的波形动画光标
3. **键盘交互**：支持各种编辑快捷键（Emacs 风格）
4. **语音集成**：与语音输入模式集成，显示音频波形
5. **剪贴板支持**：支持图片粘贴检测和提示

## 功能点目的

### 1. 语音模式光标动画
当语音输入激活时，光标变为动态音频波形：
- 使用音频电平数据驱动动画
- 平滑过渡效果（EMA 滤波）
- 彩虹色渐变（HSL 色彩空间）
- 静音检测（灰色显示）

### 2. 文本编辑功能
通过 `useTextInput` hook 提供完整的编辑能力：
- 光标移动（方向键、Home/End）
- 文本选择（Shift + 方向键）
- 删除操作（Backspace、Delete）
- Emacs 快捷键（Ctrl+A/E/K 等）
- 历史导航（↑/↓）

### 3. 无障碍支持
- `CLAUDE_CODE_ACCESSIBILITY` 环境变量控制
- 禁用光标动画和视觉效果
- 简化界面元素

### 4. 图片粘贴提示
当终端获得焦点且剪贴板中有图片时，显示粘贴提示。

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
export type Props = BaseTextInputProps & {
  highlights?: TextHighlight[];  // 文本高亮
};

// 语音相关常量
const BARS = ' ▁▂▃▄▅▆▇█';  // 8 级波形条
const CURSOR_WAVEFORM_WIDTH = 1;  // 波形光标宽度
const SMOOTH = 0.7;  // 平滑系数（EMA）
const LEVEL_BOOST = 1.8;  // 电平增强
const SILENCE_THRESHOLD = 0.15;  // 静音阈值
```

### 核心流程

#### 1. 语音状态获取
```typescript
const voiceState = feature('VOICE_MODE') 
  ? useVoiceState(s => s.voiceState) 
  : 'idle' as const;
const isVoiceRecording = voiceState === 'recording';

const audioLevels = feature('VOICE_MODE') 
  ? useVoiceState(s => s.voiceAudioLevels) 
  : [];
```

#### 2. 波形光标渲染
```typescript
if (isVoiceRecording && !reducedMotion) {
  // 获取最新音频电平
  const raw = audioLevels.length > 0 
    ? audioLevels[audioLevels.length - 1] ?? 0 
    : 0;
  
  // EMA 平滑
  const target = Math.min(raw * LEVEL_BOOST, 1);
  smoothed[0] = (smoothed[0] ?? 0) * SMOOTH + target * (1 - SMOOTH);
  const displayLevel = smoothed[0] ?? 0;
  
  // 映射到波形字符
  const barIndex = Math.max(1, Math.min(
    Math.round(displayLevel * (BARS.length - 1)), 
    BARS.length - 1
  ));
  
  // 静音检测
  const isSilent = raw < SILENCE_THRESHOLD;
  
  // 彩虹色计算
  const hue = animTime / 1000 * 90 % 360;
  const { r, g, b } = isSilent 
    ? { r: 128, g: 128, b: 128 }  // 静音灰色
    : hueToRgb(hue);  // 彩虹色
  
  invert = () => chalk.rgb(r, g, b)(BARS[barIndex]!);
} else {
  invert = chalk.inverse;  // 标准反色光标
}
```

#### 3. 文本输入状态管理
```typescript
const textInputState = useTextInput({
  value: props.value,
  onChange: props.onChange,
  onSubmit: props.onSubmit,
  onExit: props.onExit,
  // ... 其他回调
  cursorChar: props.showCursor ? ' ' : '',
  invert,  // 光标渲染函数
  themeText: color('text', theme),
  columns: props.columns,
  maxVisibleLines: props.maxVisibleLines,
  onImagePaste: props.onImagePaste,
  // ... 其他配置
});
```

#### 4. 剪贴板图片提示
```typescript
useClipboardImageHint(isTerminalFocused, !!props.onImagePaste);
```

## 关键代码路径与文件引用

### 本文件导出
- `TextInput` - 默认导出，文本输入组件
- `Props` - 组件 Props 类型

### 依赖文件
| 文件路径 | 用途 |
|---------|------|
| `bun:bundle` | `feature` 编译时特性开关 |
| `chalk` | 终端颜色处理 |
| `../context/voice.js` | 语音状态管理 |
| `../hooks/useClipboardImageHint.js` | 剪贴板图片检测 |
| `../hooks/useSettings.js` | 用户设置（reducedMotion） |
| `../hooks/useTextInput.js` | 文本输入逻辑 hook |
| `../ink.js` | Ink UI 组件和动画 |
| `../types/textInputTypes.js` | 类型定义 |
| `../utils/envUtils.js` | 环境变量工具 |
| `../utils/textHighlighting.js` | 文本高亮类型 |
| `./BaseTextInput.js` | 基础输入组件 |
| `./Spinner/utils.js` | `hueToRgb` 颜色工具 |

### 调用方
TextInput 被广泛使用于：
- `src/components/PromptInput/PromptInput.tsx` - 主输入框
- `src/components/VimTextInput.tsx` - Vim 模式输入
- `src/components/HistorySearchInput.tsx` - 历史搜索
- `src/components/CustomSelect/select-input-option.tsx` - 选择输入
- 以及各种命令步骤组件（OAuth、API Key 等）

### useTextInput Hook

```typescript
export function useTextInput({
  value,
  onChange,
  onSubmit,
  onExit,
  onHistoryUp,
  onHistoryDown,
  cursorChar,
  invert,
  columns,
  // ...
}): TextInputState {
  const cursor = Cursor.fromText(value, columns, offset);
  
  // 键盘输入处理
  function onInput(input: string, key: Key): void {
    const nextCursor = mapKey(key)(input);
    if (nextCursor && !cursor.equals(nextCursor)) {
      onChange(nextCursor.text);
      setOffset(nextCursor.offset);
    }
  }
  
  return {
    onInput,
    renderedValue: cursor.render(cursorChar, mask, invert, ghostText),
    offset,
    cursorLine,
    cursorColumn,
    // ...
  };
}
```

### BaseTextInput 组件

TextInput 将处理后的状态传递给 BaseTextInput 进行实际渲染：
```typescript
return (
  <Box ref={animRef}>
    <BaseTextInput 
      inputState={textInputState} 
      terminalFocus={isTerminalFocused}
      highlights={props.highlights}
      invert={invert}
      hidePlaceholderText={isVoiceRecording}
      {...props} 
    />
  </Box>
);
```

## 依赖与外部交互

### 语音状态流转

```
用户按下语音快捷键
    ↓
voiceState = 'recording'
    ↓
TextInput 检测到 isVoiceRecording = true
    ↓
启用波形光标动画
    ↓
audioLevels 更新 → 波形高度变化
    ↓
用户停止录音
    ↓
voiceState = 'processing' → 'idle'
    ↓
恢复标准光标
```

### 光标渲染策略

| 条件 | 光标样式 |
|-----|---------|
| `!canShowCursor` | 无光标（纯文本） |
| `isVoiceRecording && !reducedMotion` | 彩虹波形 |
| 其他情况 | 反色块（chalk.inverse） |

### 动画性能优化

1. **条件动画帧**：`useAnimationFrame(needsAnimation ? 50 : null)`
   - 非语音模式不订阅动画帧
   - 减少不必要的重渲染

2. **平滑滤波**：EMA (Exponential Moving Average)
   - 减少音频电平的抖动
   - 提供更流畅的视觉效果

3. **React Compiler 优化**：使用 `_c` 函数进行自动记忆化

## 风险、边界与改进建议

### 已知风险

1. **Hook 条件调用警告**
   ```typescript
   // biome-ignore lint/correctness/useHookAtTopLevel
   useVoiceState(s => s.voiceState)
   ```
   - 使用 `feature()` 作为条件导致 lint 警告
   - 注释说明 `feature()` 是编译时常量

2. **音频电平校准**
   - `LEVEL_BOOST = 1.8` 和 `SILENCE_THRESHOLD = 0.15` 是经验值
   - 不同麦克风和环境可能需要不同参数

3. **波形宽度限制**
   - `CURSOR_WAVEFORM_WIDTH = 1` 仅支持单字符波形
   - 无法展示更丰富的波形效果

### 边界情况

1. **高频音频更新**
   - 如果音频电平更新频率高于 50ms
   - 部分更新可能被跳过

2. **终端颜色支持**
   - `chalk.rgb` 需要终端支持真彩色
   - 旧终端可能显示异常

3. **语音模式快速切换**
   - 快速开始/停止录音可能导致动画状态不一致

### 改进建议

1. **自适应静音阈值**
   ```typescript
   // 根据环境噪声动态调整阈值
   const adaptiveThreshold = useMemo(() => {
     const baseline = audioLevels.slice(0, 10).reduce((a, b) => a + b, 0) / 10;
     return Math.max(0.1, baseline * 1.5);
   }, []);
   ```

2. **多字符波形**
   ```typescript
   const CURSOR_WAVEFORM_WIDTH = 3;  // 3 字符宽度
   
   // 显示最近 3 个电平
   const waveform = audioLevels.slice(-3).map((level, i) => {
     const smoothed = smoothedRef.current[i] * SMOOTH + level * (1 - SMOOTH);
     // ...
   });
   ```

3. **语音可视化增强**
   ```typescript
   // 添加音量历史可视化
   const volumeHistory = useRef<number[]>([]);
   
   useEffect(() => {
     if (isVoiceRecording) {
       volumeHistory.current.push(currentLevel);
       if (volumeHistory.current.length > 20) {
         volumeHistory.current.shift();
       }
     }
   }, [audioLevels]);
   ```

4. **配置化参数**
   ```typescript
   // 从设置读取用户偏好
   const settings = useSettings();
   const levelBoost = settings.voiceLevelBoost ?? LEVEL_BOOST;
   const smoothFactor = settings.voiceSmoothFactor ?? SMOOTH;
   ```

5. **降级方案**
   ```typescript
   // 检测终端颜色支持
   const supportsRGB = process.env.COLORTERM === 'truecolor' || 
                       process.env.TERM?.includes('256color');
   
   const invert = !supportsRGB && isVoiceRecording
     ? () => chalk.yellow(BARS[barIndex]!)  // 使用 16 色降级
     : () => chalk.rgb(r, g, b)(BARS[barIndex]!);
   ```

6. **性能监控**
   ```typescript
   // 监控渲染性能
   const renderCount = useRef(0);
   const lastLog = useRef(Date.now());
   
   useEffect(() => {
     renderCount.current++;
     if (Date.now() - lastLog.current > 5000) {
       logForDebugging(`TextInput render rate: ${renderCount.current / 5} fps`);
       renderCount.current = 0;
       lastLog.current = Date.now();
     }
   });
   ```

### 测试建议

1. **单元测试**：
   - 光标渲染函数正确性
   - 颜色计算准确性
   - 语音状态切换

2. **视觉回归测试**：
   - 波形动画截图对比
   - 不同主题下的显示效果

3. **性能测试**：
   - 高频音频更新下的渲染性能
   - 长时间语音输入的内存占用

---

**文档生成时间**：2026-04-01  
**组件路径**：`src/components/TextInput.tsx`  
**关联研究文件**：`src/hooks/useTextInput.ts`, `src/components/BaseTextInput.tsx`, `src/context/voice.tsx`
