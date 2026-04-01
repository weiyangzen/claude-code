# LanguagePicker.tsx 深度研究文档

## 场景与职责

`LanguagePicker` 是 Claude Code 用于让用户设置首选响应语言和语音语言的输入组件。该组件负责：

1. **语言偏好设置**：允许用户输入自定义语言偏好
2. **自然语言输入**：支持自由文本输入（如 "Japanese", "日本語", "Español"）
3. **默认回退**：允许留空使用默认语言（英语）
4. **设置集成**：与 Claude Code 的设置系统集成

## 功能点目的

### 1. 语言偏好输入
- **目的**：收集用户的语言和语音输出偏好
- **输入方式**：
  - 自由文本输入（支持任何语言名称）
  - 支持 Unicode（日语、中文、阿拉伯语等）
  - 60 列宽度限制
- **默认值**：留空表示使用默认语言（英语）

### 2. 键盘交互支持
- **目的**：提供标准的输入交互体验
- **绑定按键**：
  - `confirm:no` (Esc) - 取消选择，保持当前设置
  - Enter - 提交输入
- **上下文**：Settings 上下文，确保 'n' 键不会触发取消（允许在输入中键入 'n'）

### 3. 输入状态管理
- **目的**：管理输入框状态和光标位置
- **状态**：
  - `language` - 当前输入值
  - `cursorOffset` - 光标位置，初始化为输入长度

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type Props = {
  initialLanguage: string | undefined;              // 初始语言值
  onComplete: (language: string | undefined) => void;  // 完成回调
  onCancel: () => void;                             // 取消回调
};

// 输入状态
const [language, setLanguage] = useState(initialLanguage);
const [cursorOffset, setCursorOffset] = useState((initialLanguage ?? "").length);
```

### 关键流程

1. **初始化流程**：
   ```
   1. 接收 initialLanguage prop
   2. 初始化 language state
   3. 初始化 cursorOffset 为字符串长度
   4. 注册 confirm:no 键盘绑定
   ```

2. **提交处理流程**：
   ```
   1. 用户按下 Enter 触发 handleSubmit
   2. 调用 language?.trim() 去除首尾空格
   3. 如果结果为空字符串，传递 undefined
   4. 否则传递 trimmed 值
   5. 调用 onComplete
   ```

3. **取消处理流程**：
   ```
   1. 用户按下 Esc 触发 confirm:no
   2. 调用 onCancel()
   3. 父组件关闭 picker，保持原设置
   ```

### 输入组件配置

```typescript
<TextInput
  value={language ?? ""}
  onChange={setLanguage}
  onSubmit={handleSubmit}
  focus={true}
  showCursor={true}
  placeholder={`e.g., Japanese, 日本語, Español${figures.ellipsis}`}
  columns={60}
  cursorOffset={cursorOffset}
  onChangeCursorOffset={setCursorOffset}
/>
```

## 关键代码路径与文件引用

### 本文件关键代码

```typescript
export function LanguagePicker({
  initialLanguage,
  onComplete,
  onCancel,
}: Props): React.ReactNode {
  const [language, setLanguage] = useState(initialLanguage);
  const [cursorOffset, setCursorOffset] = useState(
    (initialLanguage ?? '').length,
  );

  // 使用可配置的键盘绑定进行 ESC 取消
  // 使用 Settings 上下文，使 'n' 键不会触发取消（允许在输入中键入 'n'）
  useKeybinding('confirm:no', onCancel, { context: 'Settings' });

  function handleSubmit(): void {
    const trimmed = language?.trim();
    onComplete(trimmed || undefined);
  }

  return (
    <Box flexDirection="column" gap={1}>
      <Text>Enter your preferred response and voice language:</Text>
      <Box flexDirection="row" gap={1}>
        <Text>{figures.pointer}</Text>
        <TextInput
          value={language ?? ''}
          onChange={setLanguage}
          onSubmit={handleSubmit}
          focus={true}
          showCursor={true}
          placeholder={`e.g., Japanese, 日本語, Español${figures.ellipsis}`}
          columns={60}
          cursorOffset={cursorOffset}
          onChangeCursorOffset={setCursorOffset}
        />
      </Box>
      <Text dimColor>Leave empty for default (English)</Text>
    </Box>
  );
}
```

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `figures` | Unicode 图标（pointer, ellipsis） |
| `src/keybindings/useKeybinding.ts` | 键盘绑定钩子 |
| `src/components/TextInput.tsx` | 文本输入组件 |
| `src/ink.tsx` | Ink 渲染组件 (`Box`, `Text`) |

### 调用方

- `src/components/Settings/Settings.tsx` - 设置界面
- `src/commands/voice/voice.tsx` - 语音设置
- 可能的其他设置相关组件

## 依赖与外部交互

### 外部依赖

1. **React**：UI 组件库
2. **figures**：Unicode 图标库
3. **Ink**：终端 UI 渲染
4. **React Compiler**：编译时优化

### 内部服务交互

1. **键盘绑定系统**：
   - 使用 `useKeybinding` 钩子注册取消操作
   - 上下文为 "Settings"，隔离 'n' 键的影响

2. **输入系统**：
   - 使用 `TextInput` 组件提供标准输入体验
   - 支持光标移动和文本编辑

3. **设置系统**：
   - 通过 `onComplete` 回调将语言设置传递回父组件
   - 父组件负责持久化到配置

### 数据流

```
用户输入 → TextInput → language state → handleSubmit → 
trim() → onComplete(language | undefined) → 父组件持久化
```

## 风险、边界与改进建议

### 潜在风险

1. **语言识别依赖模型**：
   - 风险：输入的语言名称需要模型正确识别
   - 示例：用户输入 "中文"，模型需要理解为 Chinese
   - 建议：添加语言代码标准化（如 ISO 639-1）

2. **无效语言处理**：
   - 风险：用户可能输入模型不支持的语言
   - 示例：输入 "Klingon" 或乱码
   - 建议：添加语言验证或建议列表

3. **编码问题**：
   - 风险：某些终端可能不支持所有 Unicode 字符
   - 影响：非 ASCII 语言名称可能显示异常

### 边界情况

1. **空输入**：
   - 输入为空或仅空格时，传递 `undefined`
   - 表示使用默认语言（英语）

2. **极长输入**：
   - `columns={60}` 限制显示宽度
   - 但实际输入值可以更长
   - 建议：添加最大长度限制

3. **特殊字符**：
   - 输入可能包含表情符号或控制字符
   - 当前实现会原样传递
   - 建议：添加输入清理

### 改进建议

1. **添加语言选择列表**：
   ```typescript
   // 使用 Select 组件替代自由输入
   const COMMON_LANGUAGES = [
     { value: 'en', label: 'English' },
     { value: 'ja', label: 'Japanese / 日本語' },
     { value: 'es', label: 'Spanish / Español' },
     { value: 'zh', label: 'Chinese / 中文' },
     // ...
   ];
   
   <Select
     options={[
       { value: '', label: 'Default (English)' },
       ...COMMON_LANGUAGES,
       { value: 'custom', label: 'Other (specify)' },
     ]}
     onChange={handleLanguageSelect}
   />
   ```

2. **添加语言验证**：
   ```typescript
   function validateLanguage(input: string): boolean {
     // 检查是否为有效的 ISO 639-1 代码
     // 或匹配已知语言列表
     const validLanguages = ['en', 'ja', 'es', 'zh', 'fr', 'de', ...];
     return validLanguages.includes(input.toLowerCase());
   }
   ```

3. **自动检测语言**：
   ```typescript
   // 根据用户输入历史或系统设置自动建议
   function detectPreferredLanguage(): string | undefined {
     // 检查系统 locale
     const systemLang = process.env.LANG?.split('_')[0];
     // 或分析用户之前的输入
     return systemLang;
   }
   ```

4. **添加语音预览**：
   ```typescript
   // 添加 "Preview" 按钮，让用户听到语音效果
   <Box>
     <Button onClick={previewVoice}>Preview Voice</Button>
   </Box>
   ```

5. **分离响应语言和语音语言**：
   ```typescript
   // 允许分别设置文本响应语言和语音语言
   type LanguageSettings = {
     responseLanguage?: string;  // 文本响应
     voiceLanguage?: string;     // 语音合成
     autoDetect: boolean;        // 自动检测输入语言
   };
   ```

6. **添加最近使用**：
   ```typescript
   // 显示最近使用的语言
   const recentLanguages = getRecentLanguages();  // 从配置读取
   <Box>
     <Text dimColor>Recent:</Text>
     {recentLanguages.map(lang => (
       <Button key={lang} onClick={() => selectLanguage(lang)}>
         {lang}
       </Button>
     ))}
   </Box>
   ```

### 相关配置项

```typescript
// GlobalConfig 中的相关字段
interface GlobalConfig {
  preferredLanguage?: string;  // 用户首选语言
  voiceLanguage?: string;      // 语音合成语言（如果与响应语言不同）
}

// 或 Settings 中的字段
interface Settings {
  language?: {
    response?: string;
    voice?: string;
    autoDetect?: boolean;
  };
}
```

### 测试建议

1. **单元测试**：
   - 测试 `handleSubmit` 的 trim 逻辑
   - 测试空输入处理
   - 测试 Unicode 输入

2. **集成测试**：
   - 测试与 TextInput 的集成
   - 测试键盘绑定

3. **国际化测试**：
   - 测试各种语言名称的输入
   - 测试 RTL（从右到左）语言
   - 测试 CJK（中日韩）字符

4. **无障碍测试**：
   - 测试屏幕阅读器兼容性
   - 测试键盘导航

### 与相关组件的对比

| 特性 | LanguagePicker | ThemePicker | ModelPicker |
|-----|----------------|-------------|-------------|
| 输入方式 | 自由文本 | 选择列表 | 选择列表 |
| 验证 | 无 | 内置 | 内置 |
| 默认值 | undefined | 预设值 | 预设值 |
| 键盘上下文 | Settings | - | - |
| 即时应用 | 否（需提交） | 是 | 是 |

### 代码简化建议

当前组件相对简单，但可以考虑：

1. **提取通用 picker 模式**：
   ```typescript
   // 创建可复用的设置 picker 组件
   export function SettingPicker<T>({
     title,
     placeholder,
     initialValue,
     onComplete,
     onCancel,
   }: SettingPickerProps<T>);
   ```

2. **使用受控组件模式**：
   - 当前已经是受控组件
   - 确保父组件完全控制状态

3. **添加输入历史**：
   - 使用 `useHistory` 钩子支持上下键浏览历史
