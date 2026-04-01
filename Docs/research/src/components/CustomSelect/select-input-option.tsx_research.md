# select-input-option.tsx 研究文档

## 场景与职责

`select-input-option.tsx` 是 Claude Code CLI 中选择组件的**输入类型选项**实现。它在选择列表中提供一个可交互的文本输入框，支持：

1. **文本输入**：在选择列表中嵌入可编辑的文本输入框
2. **图片粘贴**：支持从剪贴板粘贴图片作为附件
3. **图片附件管理**：显示、选择、删除已粘贴的图片
4. **外部编辑器集成**：通过快捷键（Ctrl+G）打开外部编辑器编辑内容
5. **标签显示模式**：支持在输入框旁显示标签（label）

该组件主要用于需要用户在选择列表中输入自定义内容的场景，如权限确认时的自定义理由输入、配置参数编辑等。

## 功能点目的

### 1. 文本输入集成
- 在选择列表项中嵌入 `TextInput` 组件
- 支持 placeholder、多行输入、光标控制
- 两种布局模式：`compact`（紧凑）和 `expanded`（展开）

### 2. 标签显示控制
- `showLabel` 属性控制是否在输入框旁显示标签
- 支持 `labelValueSeparator` 自定义分隔符（默认 `, `）
- `showLabelWithValue` 强制显示标签（即使全局设置不显示）

### 3. 外部编辑器集成
- `onOpenEditor` 回调支持打开外部编辑器（如 VS Code）
- 快捷键：`chat:externalEditor`（默认 Ctrl+G）
- 编辑器保存后通过 `setValue` 回调更新内容

### 4. 图片粘贴功能
- `onImagePaste` 回调处理图片粘贴
- 使用 `getImageFromClipboard()` 获取剪贴板图片
- 支持 base64 编码的图片数据

### 5. 图片附件管理
- 显示已粘贴的图片附件列表（`ClickableImageRef`）
- 支持图片选择模式（`imagesSelected`）
- 快捷键导航图片：`→` 下一个，`←` 上一个
- 快捷键删除图片：`backspace`
- 快捷键退出选择模式：`esc`

### 6. 光标位置管理
- `resetCursorOnUpdate` 属性：值更新时自动将光标移到行尾
- 解决异步更新时的光标位置错乱问题
- 使用 `isUserEditing` ref 跟踪用户是否正在编辑

## 具体技术实现

### Props 定义

```typescript
type Props<T> = {
  option: Extract<OptionWithDescription<T>, { type: 'input' }>;
  isFocused: boolean;                    // 是否聚焦
  isSelected: boolean;                   // 是否选中
  shouldShowDownArrow: boolean;          // 是否显示向下滚动箭头
  shouldShowUpArrow: boolean;            // 是否显示向上滚动箭头
  maxIndexWidth: number;                 // 索引最大宽度（对齐用）
  index: number;                         // 选项索引（1-based）
  inputValue: string;                    // 当前输入值
  onInputChange: (value: string) => void; // 值变化回调
  onSubmit: (value: string) => void;     // 提交回调
  onExit?: () => void;                   // 退出回调
  layout: 'compact' | 'expanded';        // 布局模式
  children?: ReactNode;                  // 子元素（如复选框）
  showLabel?: boolean;                   // 是否显示标签
  onOpenEditor?: (currentValue: string, setValue: (value: string) => void) => void;
  resetCursorOnUpdate?: boolean;         // 更新时重置光标
  onImagePaste?: (base64Image: string, mediaType?: string, filename?: string, dimensions?: ImageDimensions, sourcePath?: string) => void;
  pastedContents?: Record<number, PastedContent>;  // 粘贴的内容
  onRemoveImage?: (id: number) => void;  // 删除图片回调
  imagesSelected?: boolean;              // 图片选择模式
  selectedImageIndex?: number;           // 当前选中的图片索引
  onImagesSelectedChange?: (selected: boolean) => void;  // 切换选择模式
  onSelectedImageIndexChange?: (index: number) => void;  // 切换选中图片
};
```

### 核心状态

```typescript
// 光标偏移量（控制光标位置）
const [cursorOffset, setCursorOffset] = useState(inputValue.length);

// 用户是否正在编辑（防止 resetCursorOnUpdate 干扰）
const isUserEditing = useRef(false);

// 从 pastedContents 过滤出图片附件
const imageAttachments = pastedContents 
  ? Object.values(pastedContents).filter(c => c.type === "image") 
  : [];

// 是否显示标签
const showLabel = showLabelProp || option.showLabelWithValue === true;
```

### 光标位置管理

```typescript
// 当 focused 或 inputValue 变化时，重置光标位置
useEffect(() => {
  if (resetCursorOnUpdate && isFocused) {
    if (isUserEditing.current) {
      isUserEditing.current = false;
    } else {
      setCursorOffset(inputValue.length);
    }
  }
}, [inputValue.length, isFocused, resetCursorOnUpdate]);
```

### 键盘快捷键绑定

#### 外部编辑器快捷键
```typescript
const openEditor = () => {
  onOpenEditor?.(inputValue, onInputChange);
};
useKeybinding("chat:externalEditor", openEditor, {
  context: "Chat",
  isActive: isFocused && !!onOpenEditor
});
```

#### 图片粘贴快捷键
```typescript
const pasteImage = () => {
  getImageFromClipboard().then(imageData => {
    if (imageData) {
      onImagePaste(imageData.base64, imageData.mediaType, undefined, imageData.dimensions);
    }
  });
};
useKeybinding("chat:imagePaste", pasteImage, {
  context: "Chat",
  isActive: isFocused && !!onImagePaste
});
```

#### 附件管理快捷键
```typescript
// 非图片选择模式下：删除最后一个图片
useKeybinding("attachments:remove", removeLastImage, {
  context: "Attachments",
  isActive: isFocused && !imagesSelected && inputValue === "" && imageAttachments.length > 0
});

// 图片选择模式下：导航、删除、退出
useKeybindings({
  "attachments:next": () => { /* 下一个图片 */ },
  "attachments:previous": () => { /* 上一个图片 */ },
  "attachments:remove": () => { /* 删除选中图片 */ },
  "attachments:exit": () => { /* 退出选择模式 */ }
}, {
  context: "Attachments",
  isActive: isFocused && !!imagesSelected
});
```

### 渲染结构

```
Box (flexDirection: column)
├── SelectOption (输入行包装)
│   └── Box (flexDirection: row)
│       ├── 索引文本 (如 "1.  ")
│       ├── children (复选框等)
│       └── 输入区域
│           ├── 显示标签模式：
│           │   ├── Text (label)
│           │   ├── Text (分隔符)
│           │   └── TextInput (输入框)
│           └── 不显示标签模式：
│               └── TextInput (输入框，placeholder 显示 label)
├── 描述文本 (可选)
│   └── Box (paddingLeft)
│       └── Text (option.description)
├── 图片附件区域 (如果有)
│   └── Box (flexDirection: row)
│       ├── ClickableImageRef[] (图片引用)
│       └── 快捷键提示
└── 空行 (expanded 布局)
```

### TextInput 配置

```typescript
<TextInput
  value={inputValue}
  onChange={value => {
    isUserEditing.current = true;
    onInputChange(value);
    option.onChange(value);  // 同时调用 option 的 onChange
  }}
  onSubmit={onSubmit}
  onExit={onExit}
  placeholder={option.placeholder || option.label}
  focus={!imagesSelected}    // 图片选择模式下不聚焦输入框
  showCursor={true}
  multiline={true}
  cursorOffset={cursorOffset}
  onChangeCursorOffset={setCursorOffset}
  columns={80}
  onImagePaste={onImagePaste}
  onPaste={handleTextPaste}   // 处理文本粘贴
/>
```

## 关键代码路径与文件引用

### 直接依赖

| 文件 | 用途 |
|------|------|
| `../../ink.js` | Ink 组件（Box, Text, useInput）|
| `../../keybindings/useKeybinding.js` | 快捷键绑定（useKeybinding, useKeybindings）|
| `../../utils/config.js` | PastedContent 类型 |
| `../../utils/imagePaste.js` | getImageFromClipboard 函数 |
| `../../utils/imageResizer.js` | ImageDimensions 类型 |
| `../ClickableImageRef.js` | 可点击的图片引用组件 |
| `../ConfigurableShortcutHint.js` | 快捷键提示组件 |
| `../design-system/Byline.js` | 底部提示行组件 |
| `../TextInput.js` | 文本输入组件 |
| `./select.js` | OptionWithDescription 类型 |
| `./select-option.js` | SelectOption 包装组件 |

### 被依赖文件

| 文件 | 用途 |
|------|------|
| `select.tsx` | 单选组件中使用 SelectInputOption |
| `SelectMulti.tsx` | 多选组件中使用 SelectInputOption |

## 依赖与外部交互

### 图片粘贴流程

```
用户按下粘贴快捷键
    ↓
getImageFromClipboard() 读取剪贴板
    ↓
返回 { base64, mediaType, dimensions }
    ↓
onImagePaste(base64, mediaType, undefined, dimensions)
    ↓
父组件存储到 pastedContents
    ↓
组件重新渲染，显示 ClickableImageRef
```

### 外部编辑器流程

```
用户按下 Ctrl+G
    ↓
onOpenEditor(currentValue, setValue)
    ↓
父组件打开外部编辑器
    ↓
用户编辑并保存
    ↓
setValue(newValue) 更新输入值
    ↓
onInputChange 通知上游
```

### 图片附件选择模式

```
用户按下 ↓（在 input 上且有图片附件）
    ↓
onImagesSelectedChange(true) 进入选择模式
    ↓
imagesSelected = true
    ↓
TextInput focus=false（输入框失焦）
    ↓
用户可用 ← → 切换选中图片
    ↓
用户可用 backspace 删除选中图片
    ↓
用户可用 esc 退出选择模式
```

## 风险、边界与改进建议

### 风险

1. **React Compiler 编译后代码复杂**
   - 大量 `$[n]` 缓存数组操作，调试困难
   - 需要对照源码理解逻辑

2. **光标位置管理复杂性**
   - `cursorOffset`、`isUserEditing`、`resetCursorOnUpdate` 三者交互复杂
   - 异步更新时可能出现光标跳动

3. **图片附件状态分散**
   - `pastedContents` 存储在父组件
   - `imagesSelected`、`selectedImageIndex` 需要父组件管理
   - 状态同步复杂

4. **快捷键冲突风险**
   - 同时监听 `useKeybinding` 和 `useInput`
   - 不同 context 的快捷键可能冲突

### 边界情况

1. **空输入值**
   - 输入框显示 placeholder 或 label
   - 提交时可能触发 onCancel（取决于 allowEmptySubmitToCancel）

2. **无图片附件**
   - `imageAttachments` 为空数组
   - 图片相关快捷键不激活

3. **失去焦点**
   - 自动退出图片选择模式（通过 useEffect）
   - 输入框停止响应键盘事件

4. **大量图片附件**
   - 水平排列可能超出屏幕宽度
   - 没有实现水平滚动

### 改进建议

1. **性能优化**
   - 图片附件过滤使用 `useMemo` 缓存
   - 考虑虚拟化大量图片附件的渲染

2. **代码组织**
   - 将图片附件管理逻辑提取为自定义 Hook
   - 分离光标管理逻辑

3. **可访问性**
   - 添加屏幕阅读器支持
   - 图片附件添加 alt 文本

4. **功能增强**
   - 支持拖拽排序图片附件
   - 支持图片预览（hover 显示大图）
   - 添加图片大小限制提示

5. **类型安全**
   ```typescript
   // 建议添加更严格的类型
   type ImageAttachment = Extract<PastedContent, { type: 'image' }>;
   
   // 建议添加 props 验证
   if (option.type !== 'input') {
     throw new Error('SelectInputOption requires input type option');
   }
   ```

6. **错误处理**
   - `getImageFromClipboard()` 失败时添加错误提示
   - 图片加载失败时的降级显示
