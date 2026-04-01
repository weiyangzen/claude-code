# powershellToolUseOptions.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`powershellToolUseOptions.tsx` 是 PowerShell 权限请求对话框的选项生成模块。它负责根据当前的权限上下文、用户反馈模式和建议规则，动态生成用户可选择的权限选项列表。

### 1.2 使用场景
- **标准权限请求**：生成基本的 Yes/No 选项
- **带反馈的权限请求**：生成支持输入模式的 Yes/No 选项
- **带前缀编辑的权限请求**：生成可编辑命令前缀的"不再询问"选项
- **应用建议规则**：生成应用系统建议权限规则的选项

### 1.3 与 Bash 选项生成的区别
虽然与 `bashToolUseOptions.tsx` 共享相似的设计模式，但 PowerShell 版本有以下特点：
- 不支持沙盒切换（Windows 不支持沙盒）
- 不支持分类器审查选项（仅 Bash 的 ANT 内部功能）
- 使用 PowerShell 特定的工具名称常量

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| 基础选项 | 提供 Yes/No 两种基本决策选项 |
| 反馈模式 | 支持展开输入框收集用户额外指令 |
| 前缀编辑 | 允许用户自定义"不再询问"的命令前缀 |
| 建议应用 | 支持一键应用系统推荐的权限规则 |
| 权限控制 | 根据 `allowManagedPermissionRulesOnly` 设置控制选项显示 |

### 2.2 选项类型定义

```typescript
export type PowerShellToolUseOption = 
  | 'yes'           // 允许执行
  | 'yes-apply-suggestions'  // 允许并应用建议规则
  | 'yes-prefix-edited'      // 允许并保存自定义前缀规则
  | 'no';            // 拒绝执行
```

---

## 3. 具体技术实现

### 3.1 函数签名

```typescript
export function powershellToolUseOptions({
  suggestions = [],
  onRejectFeedbackChange,
  onAcceptFeedbackChange,
  yesInputMode = false,
  noInputMode = false,
  editablePrefix,
  onEditablePrefixChange,
}: {
  suggestions?: PermissionUpdate[];
  onRejectFeedbackChange: (value: string) => void;
  onAcceptFeedbackChange: (value: string) => void;
  yesInputMode?: boolean;
  noInputMode?: boolean;
  editablePrefix?: string;
  onEditablePrefixChange?: (value: string) => void;
}): OptionWithDescription<PowerShellToolUseOption>[]
```

### 3.2 选项生成逻辑

#### 3.2.1 Yes 选项生成
```typescript
if (yesInputMode) {
  // 反馈模式：显示输入框
  options.push({
    type: 'input',
    label: 'Yes',
    value: 'yes',
    placeholder: 'and tell Claude what to do next',
    onChange: onAcceptFeedbackChange,
    allowEmptySubmitToCancel: true
  });
} else {
  // 普通模式：简单选项
  options.push({
    label: 'Yes',
    value: 'yes'
  });
}
```

#### 3.2.2 "不再询问"选项生成（核心逻辑）
```typescript
// 行 44-73: 条件判断
if (shouldShowAlwaysAllowOptions() && suggestions.length > 0) {
  // 检查是否包含非 PowerShell 建议（如目录权限、Read 工具规则）
  const hasNonPowerShellSuggestions = suggestions.some(
    s => s.type === 'addDirectories' || 
         (s.type === 'addRules' && s.rules?.some(r => r.toolName !== POWERSHELL_TOOL_NAME))
  );

  // 优先使用可编辑前缀输入
  if (editablePrefix !== undefined && 
      onEditablePrefixChange && 
      !hasNonPowerShellSuggestions) {
    options.push({
      type: 'input',
      label: 'Yes, and don't ask again for',
      value: 'yes-prefix-edited',
      placeholder: 'command prefix (e.g., Get-Process:*)',
      initialValue: editablePrefix,
      onChange: onEditablePrefixChange,
      allowEmptySubmitToCancel: true,
      showLabelWithValue: true,
      labelValueSeparator: ': ',
      resetCursorOnUpdate: true
    });
  } else {
    // 回退到非可编辑标签
    const label = generateShellSuggestionsLabel(suggestions, POWERSHELL_TOOL_NAME);
    if (label) {
      options.push({
        label,
        value: 'yes-apply-suggestions'
      });
    }
  }
}
```

#### 3.2.3 No 选项生成
```typescript
if (noInputMode) {
  options.push({
    type: 'input',
    label: 'No',
    value: 'no',
    placeholder: 'and tell Claude what to do differently',
    onChange: onRejectFeedbackChange,
    allowEmptySubmitToCancel: true
  });
} else {
  options.push({
    label: 'No',
    value: 'no'
  });
}
```

### 3.3 数据结构

#### 3.3.1 OptionWithDescription 类型
```typescript
interface OptionWithDescription<T> {
  type?: 'input' | 'default';  // 选项类型
  label: string | ReactNode;   // 显示标签
  value: T;                    // 选项值
  placeholder?: string;        // 输入框占位符（type='input'时）
  onChange?: (value: string) => void;  // 输入变化回调
  allowEmptySubmitToCancel?: boolean;  // 允许空提交取消
  initialValue?: string;       // 初始值
  showLabelWithValue?: boolean;        // 是否显示标签与值
  labelValueSeparator?: string;        // 标签与值的分隔符
  resetCursorOnUpdate?: boolean;       // 更新时重置光标
}
```

#### 3.3.2 PermissionUpdate 类型
```typescript
type PermissionUpdate =
  | { type: 'addRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'replaceRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'removeRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'setMode'; mode: PermissionMode; destination: PermissionUpdateDestination }
  | { type: 'addDirectories'; directories: string[]; destination: PermissionUpdateDestination }
  | { type: 'removeDirectories'; directories: string[]; destination: PermissionUpdateDestination };
```

### 3.4 关键决策逻辑

#### 3.4.1 可编辑前缀 vs 建议标签的选择
```
使用可编辑前缀输入的条件：
1. editablePrefix !== undefined（有可用前缀）
2. onEditablePrefixChange !== undefined（有变化处理器）
3. !hasNonPowerShellSuggestions（没有非 PowerShell 建议）

使用非可编辑标签的条件：
- 上述任一条件不满足时回退
- 包含目录权限建议时
- 包含 Read 工具规则时
```

#### 3.4.2 非 PowerShell 建议检测
```typescript
const hasNonPowerShellSuggestions = suggestions.some(s => 
  s.type === 'addDirectories' || 
  (s.type === 'addRules' && s.rules?.some(r => r.toolName !== POWERSHELL_TOOL_NAME))
);
```

**原因**：可编辑前缀输入只能表示命令前缀规则，无法表示目录权限或 Read 工具规则。当存在这些复杂建议时，回退到 `generateShellSuggestionsLabel` 生成描述性标签。

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖文件

```
src/components/permissions/PowerShellPermissionRequest/
├── powershellToolUseOptions.tsx       # 本文件
└── PowerShellPermissionRequest.tsx    # 主要调用方

依赖的同级组件：
src/components/permissions/
└── shellPermissionHelpers.tsx         # generateShellSuggestionsLabel

依赖的工具常量：
src/tools/PowerShellTool/
└── toolName.ts                        # POWERSHELL_TOOL_NAME

依赖的权限类型：
src/utils/permissions/
├── PermissionUpdateSchema.js          # PermissionUpdate 类型
└── permissionsLoader.js               # shouldShowAlwaysAllowOptions

依赖的 UI 组件：
src/components/CustomSelect/
└── select.tsx                         # OptionWithDescription 类型
```

### 4.2 调用链

**调用路径 1: PowerShellPermissionRequest**
```
PowerShellPermissionRequest.tsx
  → powershellToolUseOptions({
      suggestions: toolUseConfirm.permissionResult.suggestions,
      onRejectFeedbackChange: setRejectFeedback,
      onAcceptFeedbackChange: setAcceptFeedback,
      yesInputMode,
      noInputMode,
      editablePrefix,
      onEditablePrefixChange
    })
    → 返回 OptionWithDescription[]
    → Select.tsx 渲染选项
```

**调用路径 2: 建议标签生成（回退路径）**
```
powershellToolUseOptions()
  → shellPermissionHelpers.tsx:generateShellSuggestionsLabel(suggestions, POWERSHELL_TOOL_NAME)
    → 返回 ReactNode 标签
    → 包装为 OptionWithDescription
```

### 4.3 关键代码路径

**路径 1: 选项生成流程**
```
行 24: 初始化 options 数组
行 25-39: 添加 Yes 选项（普通或输入模式）
行 49-73: 条件添加"不再询问"选项（可编辑前缀或建议标签）
行 74-88: 添加 No 选项（普通或输入模式）
行 89: 返回 options 数组
```

**路径 2: 权限控制流程**
```
permissionsLoader.ts:shouldShowAlwaysAllowOptions()
  → 检查 policySettings.allowManagedPermissionRulesOnly
  → 返回 boolean
  → powershellToolUseOptions() 决定是否显示"不再询问"选项
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `POWERSHELL_TOOL_NAME` | `../../../tools/PowerShellTool/toolName.js` | PowerShell 工具名称常量 |
| `PermissionUpdate` | `../../../utils/permissions/PermissionUpdateSchema.js` | 权限更新类型 |
| `shouldShowAlwaysAllowOptions` | `../../../utils/permissions/permissionsLoader.js` | 权限控制检查 |
| `OptionWithDescription` | `../../CustomSelect/select.js` | 选项类型定义 |
| `generateShellSuggestionsLabel` | `../shellPermissionHelpers.js` | 建议标签生成 |

### 5.2 与 shellPermissionHelpers.tsx 的交互

```typescript
// 当无法使用可编辑前缀时，调用此函数生成标签
import { generateShellSuggestionsLabel } from '../shellPermissionHelpers.js';

const label = generateShellSuggestionsLabel(suggestions, POWERSHELL_TOOL_NAME);
```

`generateShellSuggestionsLabel` 功能：
- 解析建议规则（addRules, addDirectories）
- 分离 Read 工具规则和 PowerShell 规则
- 生成人类可读的描述文本
- 处理单/多路径、单/多命令的格式化

### 5.3 与权限系统的交互

```typescript
// 检查是否允许显示"始终允许"选项
import { shouldShowAlwaysAllowOptions } from '../../../utils/permissions/permissionsLoader.js';

if (shouldShowAlwaysAllowOptions() && suggestions.length > 0) {
  // 显示"不再询问"选项
}
```

**权限控制逻辑**：
- 当 `allowManagedPermissionRulesOnly` 启用时，隐藏"不再询问"选项
- 这是为了企业/管理场景，只允许使用预配置的权限规则

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 建议类型混合风险
```typescript
const hasNonPowerShellSuggestions = suggestions.some(s => 
  s.type === 'addDirectories' || 
  (s.type === 'addRules' && s.rules?.some(r => r.toolName !== POWERSHELL_TOOL_NAME))
);
```
**风险**：当建议混合了 PowerShell 规则和其他类型规则时，回退到非可编辑标签。但 `generateShellSuggestionsLabel` 可能生成过于简化的描述，用户可能不理解实际会应用哪些规则。

#### 6.1.2 空建议列表处理
```typescript
if (shouldShowAlwaysAllowOptions() && suggestions.length > 0) {
  // 显示"不再询问"选项
}
```
**边界**：当 `suggestions` 为空时，完全不显示"不再询问"选项。这在某些场景下可能限制用户体验（如用户明确想要创建规则）。

#### 6.1.3 可编辑前缀验证缺失
```typescript
options.push({
  type: 'input',
  label: 'Yes, and don't ask again for',
  value: 'yes-prefix-edited',
  placeholder: 'command prefix (e.g., Get-Process:*)',
  initialValue: editablePrefix,
  onChange: onEditablePrefixChange,
  // ... 没有验证逻辑
});
```
**风险**：用户可以输入任意前缀，包括过于宽泛或危险的前缀，没有实时验证或警告。

### 6.2 边界情况

| 场景 | 当前行为 |
|------|----------|
| `editablePrefix` 为 undefined | 隐藏可编辑前缀选项，尝试显示建议标签 |
| `onEditablePrefixChange` 未提供 | 回退到建议标签模式 |
| 存在目录权限建议 | 强制使用建议标签，不使用可编辑前缀 |
| `shouldShowAlwaysAllowOptions()` 返回 false | 完全不显示"不再询问"选项 |
| `suggestions` 为空数组 | 不显示任何"不再询问"选项 |
| 建议标签生成为 null | 不添加该选项 |

### 6.3 改进建议

#### 6.3.1 添加前缀验证
```typescript
// 建议：在选项生成时添加前缀验证
function validatePrefix(prefix: string): { valid: boolean; warning?: string } {
  // 检查前缀格式
  if (!prefix.includes(':')) {
    return { valid: false, warning: '前缀必须包含冒号（如 Command:*）' };
  }
  
  // 检查是否过于宽泛
  const [cmd] = prefix.split(':');
  if (cmd && !cmd.includes('-') && !cmd.includes(' ')) {
    return { valid: true, warning: '注意：此规则将匹配该命令的所有用法' };
  }
  
  return { valid: true };
}

// 在组件中使用
const validation = validatePrefix(editablePrefix);
if (!validation.valid) {
  // 显示警告或禁用选项
}
```

#### 6.3.2 增强混合建议的展示
```typescript
// 建议：当存在混合建议时，提供更详细的说明
if (hasNonPowerShellSuggestions) {
  const label = generateShellSuggestionsLabel(suggestions, POWERSHELL_TOOL_NAME);
  options.push({
    label: (
      <Text>
        {label}
        <Text dimColor> (包含目录权限)</Text>
      </Text>
    ),
    value: 'yes-apply-suggestions',
    description: '将应用命令规则和目录访问权限'  // 添加详细说明
  });
}
```

#### 6.3.3 支持手动创建规则
```typescript
// 建议：即使没有系统建议，也允许用户手动创建规则
if (shouldShowAlwaysAllowOptions()) {
  if (suggestions.length > 0) {
    // 现有逻辑
  } else if (editablePrefix) {
    // 添加手动创建规则选项
    options.push({
      type: 'input',
      label: 'Yes, and don't ask again for',
      value: 'yes-prefix-edited',
      placeholder: 'command prefix (e.g., Get-Process:*)',
      initialValue: editablePrefix,
      onChange: onEditablePrefixChange,
      allowEmptySubmitToCancel: true,
      showLabelWithValue: true,
      labelValueSeparator: ': ',
      resetCursorOnUpdate: true
    });
  }
}
```

#### 6.3.4 代码注释补充
```typescript
// 行 41-42 的注释可以扩展
// Note: No sandbox toggle for PowerShell - sandbox is not supported on Windows
// Reason: Windows 沙盒技术与 Unix 的 chroot/jail 机制不同，实现复杂度高，
// 且 PowerShell 主要用于 Windows 环境，沙盒价值有限。

// Note: No classifier-reviewed option for PowerShell (ANT-ONLY feature for Bash)
// Reason: Bash 分类器是基于大量 Bash 命令训练的内部模型，
// PowerShell 命令模式差异较大，需要单独训练分类器。
```

### 6.4 测试建议

建议增加以下测试用例：

1. **选项生成测试**
   - 标准 Yes/No 选项生成
   - 输入模式下的 Yes/No 选项
   - 可编辑前缀选项生成条件
   - 建议标签回退逻辑

2. **权限控制测试**
   - `shouldShowAlwaysAllowOptions()` 返回 false 时的行为
   - 混合建议的处理
   - 空建议列表的处理

3. **交互测试**
   - 输入模式切换
   - 前缀编辑回调触发
   - 选项值正确传递

---

## 7. 附录：完整类型定义

```typescript
// PowerShellToolUseOption 联合类型
export type PowerShellToolUseOption = 
  | 'yes'
  | 'yes-apply-suggestions' 
  | 'yes-prefix-edited'
  | 'no';

// 函数参数类型
interface PowerShellToolUseOptionsParams {
  suggestions?: PermissionUpdate[];
  onRejectFeedbackChange: (value: string) => void;
  onAcceptFeedbackChange: (value: string) => void;
  yesInputMode?: boolean;
  noInputMode?: boolean;
  editablePrefix?: string;
  onEditablePrefixChange?: (value: string) => void;
}

// 简化后的选项结构
interface InputOption {
  type: 'input';
  label: string;
  value: PowerShellToolUseOption;
  placeholder: string;
  onChange: (value: string) => void;
  allowEmptySubmitToCancel: boolean;
  initialValue?: string;
  showLabelWithValue?: boolean;
  labelValueSeparator?: string;
  resetCursorOnUpdate?: boolean;
}

interface DefaultOption {
  label: ReactNode;
  value: PowerShellToolUseOption;
}

type Option = InputOption | DefaultOption;
```
