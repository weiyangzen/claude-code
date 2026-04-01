# ShowInIDEPrompt.tsx 研究文档

## 场景与职责

`ShowInIDEPrompt` 是一个 IDE 集成相关的权限提示组件，用于在文件变更已在 IDE 中打开 diff 视图时，向用户确认是否应用这些编辑。它是文件权限对话框 (`FilePermissionDialog`) 的配套组件，专门处理 IDE diff 场景。

**使用场景：**
- 文件权限对话框 (`FilePermissionDialog`) 中当 diff 已在 IDE 中显示时
- 用户通过 IDE 插件查看文件变更后确认操作
- 支持符号链接目标文件的警告提示

## 功能点目的

### 1. IDE Diff 确认
- 当文件变更已在 IDE 中打开时显示确认提示
- 告知用户保存文件以继续操作

### 2. 权限选项处理
- 支持接受/拒绝/接受一次等多种权限选项
- 处理反馈文本输入（用于拒绝或接受一次时的注释）

### 3. 符号链接警告
- 检测并显示符号链接目标文件警告
- 区分工作目录内外的符号链接目标

### 4. VS Code 终端适配
- 检测 VS Code 终端环境
- 显示相应的保存文件提示

## 具体技术实现

### 关键数据结构

```typescript
type Props<A> = {
  filePath: string;                           // 文件路径
  input: A;                                   // 工具输入数据
  onChange: (option: PermissionOption, args: A, feedback?: string) => void;
  options: PermissionOptionWithLabel[];       // 权限选项列表
  ideName: string;                            // IDE 名称（如 "VS Code"）
  symlinkTarget?: string | null;              // 符号链接目标路径
  rejectFeedback: string;                     // 拒绝反馈文本
  acceptFeedback: string;                     // 接受反馈文本
  setFocusedOption: (value: string) => void;  // 设置聚焦选项
  onInputModeToggle: (value: string) => void; // 输入模式切换回调
  focusedOption: string;                      // 当前聚焦的选项
  yesInputMode: boolean;                      // 是/否输入模式状态
  noInputMode: boolean;                       // 否输入模式状态
};
```

### 核心渲染逻辑

**IDE 打开提示：**
```typescript
const ideOpenText = <Text bold color="permission">
  Opened changes in {ideName} ⧉
</Text>;
```

**符号链接警告：**
```typescript
const symlinkWarning = symlinkTarget && (
  <Text color="warning">
    {relative(getCwd(), symlinkTarget).startsWith("..") 
      ? `This will modify ${symlinkTarget} (outside working directory) via a symlink`
      : `Symlink target: ${symlinkTarget}`
    }
  </Text>
);
```

**VS Code 终端提示：**
```typescript
const vscodeHint = isSupportedVSCodeTerminal() && (
  <Text dimColor>Save file to continue…</Text>
);
```

### 选项变更处理

```typescript
const handleChange = (value: string) => {
  const selected = options.find(opt => opt.value === value);
  if (selected) {
    // 拒绝选项：传递拒绝反馈
    if (selected.option.type === "reject") {
      const trimmedFeedback = rejectFeedback.trim();
      onChange(selected.option, input, trimmedFeedback || undefined);
      return;
    }
    // 接受一次选项：传递接受反馈
    if (selected.option.type === "accept-once") {
      const trimmedFeedback = acceptFeedback.trim();
      onChange(selected.option, input, trimmedFeedback || undefined);
      return;
    }
    // 其他选项：直接传递
    onChange(selected.option, input);
  }
};
```

### 取消处理

```typescript
const handleCancel = () => onChange({ type: "reject" }, input);
```

### 快捷键提示

```typescript
const amendHint = (focusedOption === "yes" && !yesInputMode || 
                   focusedOption === "no" && !noInputMode) 
  && " · Tab to amend";

<Box marginTop={1}>
  <Text dimColor>Esc to cancel{amendHint}</Text>
</Box>
```

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/components/ShowInIDEPrompt.tsx` - 组件实现

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/components/permissions/FilePermissionDialog/FilePermissionDialog.tsx` - 文件权限对话框

### 依赖文件
- `/home/sansha/Github/claude-code-instructkr/src/utils/cwd.js` - 获取当前工作目录
- `/home/sansha/Github/claude-code-instructkr/src/utils/ide.js` - IDE 检测工具
  - `isSupportedVSCodeTerminal()` - 检测 VS Code 终端

### 依赖组件
- `../ink.js` - Box, Text
- `./CustomSelect/index.js` - Select 组件
- `./design-system/Pane.js` - 面板容器

### 类型定义
- `/home/sansha/Github/claude-code-instructkr/src/components/permissions/FilePermissionDialog/permissionOptions.ts` - 权限选项类型

## 依赖与外部交互

### IDE 集成
- **isSupportedVSCodeTerminal**: 检测是否在 VS Code 终端中运行
- **ideName**: 从父组件传入，显示当前使用的 IDE 名称

### 权限系统
- **PermissionOption**: 权限选项类型（reject, accept-once, accept-all 等）
- **PermissionOptionWithLabel**: 带标签的选项类型
- **onChange**: 回调函数，将用户选择传递给父组件

### 文件系统
- **symlinkTarget**: 符号链接目标路径，用于显示警告
- **relative(getCwd(), symlinkTarget)**: 检测目标是否在工作目录外

### 选择器组件
- **Select**: 自定义选择器组件，支持内联描述
- 支持聚焦、取消、输入模式切换等回调

## 风险、边界与改进建议

### 边界情况

1. **符号链接检测**: 如果 `symlinkTarget` 为 null 或 undefined，不显示警告
2. **反馈文本**: 拒绝和接受一次时传递反馈文本，其他情况不传
3. **输入模式**: 根据 `yesInputMode` 和 `noInputMode` 显示 "Tab to amend" 提示

### 潜在风险

1. **硬编码字符串**: "Tab to amend" 和 "Esc to cancel" 是硬编码的英文文本
2. **IDE 检测**: 仅检测 VS Code，其他 IDE 可能也需要类似的保存提示
3. **路径安全**: `basename(filePath)` 用于显示文件名，但可能暴露敏感路径信息

### 改进建议

1. **国际化支持**:
   - 提取所有用户可见的字符串到 i18n 配置
   - 支持多语言显示

2. **扩展 IDE 支持**:
   ```typescript
   const saveFileHint = isSupportedJetBrainsTerminal() 
     ? "Press Ctrl+S to save" 
     : isSupportedVSCodeTerminal() 
       ? "Save file to continue…" 
       : null;
   ```

3. **安全改进**:
   - 对显示的文件路径进行脱敏处理
   - 避免在 UI 中显示完整绝对路径

4. **UX 增强**:
   - 添加文件内容预览（前 N 行）
   - 显示变更统计（添加/删除行数）
   - 添加快捷键提示（如 "Enter to apply"）

5. **代码重构**:
   - 将选项处理逻辑提取为独立函数
   - 将符号链接警告提取为独立组件

6. **错误处理**:
   - 处理 `options.find()` 返回 undefined 的情况
   - 添加选项类型验证

7. **可访问性**:
   - 添加颜色以外的视觉指示器
   - 支持键盘导航的明确指示
