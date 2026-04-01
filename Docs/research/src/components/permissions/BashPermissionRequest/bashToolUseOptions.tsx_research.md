# bashToolUseOptions.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`bashToolUseOptions.tsx` 是 Bash 权限请求对话框的**选项生成引擎**，负责根据权限检查结果、用户配置和分类器状态，动态生成用户可交互的选项列表。它是连接后端权限决策与前端用户界面的关键桥梁。

### 1.2 使用场景
- **基础权限确认**: 生成 "Yes" / "No" 基础选项
- **规则建议展示**: 将后端生成的权限建议转换为可读的选项标签
- **前缀规则编辑**: 提供可编辑的输入框让用户自定义权限规则
- **分类器描述审查**: 当分类器生成描述时，允许用户审查并保存为 prompt-based 规则
- **反馈收集**: 支持 Tab 键切换输入模式，收集用户执行指令或反馈

### 1.3 架构位置

```
src/components/permissions/
├── BashPermissionRequest/
│   ├── BashPermissionRequest.tsx      # 主组件，调用 bashToolUseOptions()
│   └── bashToolUseOptions.tsx         # 本文件 - 选项生成逻辑
├── shellPermissionHelpers.tsx         # 选项标签生成辅助函数
└── useShellPermissionFeedback.ts      # 反馈状态管理
```

---

## 2. 功能点目的

### 2.1 选项类型矩阵

| 选项值 | 显示条件 | 用户交互 | 结果行为 |
|--------|---------|---------|---------|
| `yes` | 始终显示 | 直接选择或 Tab 输入反馈 | 执行命令，可选反馈 |
| `yes-apply-suggestions` | 有后端建议时 | 直接选择 | 执行命令并应用建议规则 |
| `yes-prefix-edited` | 有可编辑前缀且无非 Bash 建议时 | 编辑前缀后选择 | 执行命令并保存自定义前缀规则 |
| `yes-classifier-reviewed` | 分类器启用、描述非空、未重复、非分类器触发时 | 编辑描述后选择 | 执行命令并保存 prompt-based 规则 |
| `no` | 始终显示 | 直接选择或 Tab 输入反馈 | 拒绝执行，可选反馈 |

### 2.2 功能设计原则

1. **渐进式披露**: 优先显示简单选项，高级选项（编辑前缀）在有需要时显示
2. **避免重复**: 检测已存在的允许描述，避免生成重复规则
3. **互斥性**: `yes-prefix-edited` 和 `yes-classifier-reviewed` 不会同时显示，避免用户困惑
4. **安全默认**: 当 `allowManagedPermissionRulesOnly` 启用时，隐藏所有"始终允许"选项

---

## 3. 具体技术实现

### 3.1 核心函数签名

```typescript
export type BashToolUseOption = 
  | 'yes' 
  | 'yes-apply-suggestions' 
  | 'yes-prefix-edited' 
  | 'yes-classifier-reviewed' 
  | 'no';

export function bashToolUseOptions({
  suggestions = [],                    // 后端建议的权限更新
  decisionReason,                      // 决策原因
  onRejectFeedbackChange,              // 拒绝反馈变更回调
  onAcceptFeedbackChange,              // 接受反馈变更回调
  onClassifierDescriptionChange,       // 分类器描述变更回调
  classifierDescription,               // 分类器生成的描述
  initialClassifierDescriptionEmpty,   // 初始描述是否为空
  existingAllowDescriptions = [],      // 已存在的允许描述列表
  yesInputMode = false,                // Yes 选项是否处于输入模式
  noInputMode = false,                 // No 选项是否处于输入模式
  editablePrefix,                      // 可编辑的前缀规则内容
  onEditablePrefixChange,              // 前缀变更回调
}: {
  // ... 类型定义
}): OptionWithDescription<BashToolUseOption>[]
```

### 3.2 选项生成算法

```typescript
export function bashToolUseOptions(params): OptionWithDescription<BashToolUseOption>[] {
  const options: OptionWithDescription<BashToolUseOption>[] = [];
  
  // ========== 1. Yes 选项 ==========
  if (yesInputMode) {
    // 输入模式：显示带输入框的 Yes
    options.push({
      type: 'input',
      label: 'Yes',
      value: 'yes',
      placeholder: 'and tell Claude what to do next',
      onChange: onAcceptFeedbackChange,
      allowEmptySubmitToCancel: true,
    });
  } else {
    // 普通模式：简单 Yes
    options.push({ label: 'Yes', value: 'yes' });
  }
  
  // ========== 2. "始终允许"选项（受 allowManagedPermissionRulesOnly 限制）==========
  if (shouldShowAlwaysAllowOptions()) {
    
    // 2a. 可编辑前缀选项（优先级最高）
    const hasNonBashSuggestions = suggestions.some(s => 
      s.type === 'addDirectories' || 
      (s.type === 'addRules' && s.rules?.some(r => r.toolName !== BASH_TOOL_NAME))
    );
    
    if (editablePrefix !== undefined && 
        onEditablePrefixChange && 
        !hasNonBashSuggestions && 
        suggestions.length > 0) {
      options.push({
        type: 'input',
        label: 'Yes, and don't ask again for',
        value: 'yes-prefix-edited',
        placeholder: 'command prefix (e.g., npm run:*)',
        initialValue: editablePrefix,
        onChange: onEditablePrefixChange,
        allowEmptySubmitToCancel: true,
        showLabelWithValue: true,
        labelValueSeparator: ': ',
        resetCursorOnUpdate: true,
      });
    } 
    // 2b. 应用建议选项（无可编辑前缀时）
    else if (suggestions.length > 0) {
      const label = generateShellSuggestionsLabel(suggestions, BASH_TOOL_NAME, stripBashRedirections);
      if (label) {
        options.push({ label, value: 'yes-apply-suggestions' });
      }
    }
    
    // 2c. 分类器审查选项（ANT-ONLY，特定条件）
    const editablePrefixShown = options.some(o => o.value === 'yes-prefix-edited');
    if ("external" === 'ant' &&           // 编译时条件，ANT-ONLY
        !editablePrefixShown && 
        isClassifierPermissionsEnabled() && 
        onClassifierDescriptionChange && 
        !initialClassifierDescriptionEmpty && 
        !descriptionAlreadyExists(classifierDescription ?? '', existingAllowDescriptions) &&
        decisionReason?.type !== 'classifier') {
      options.push({
        type: 'input',
        label: 'Yes, and don't ask again for',
        value: 'yes-classifier-reviewed',
        placeholder: 'describe what to allow...',
        initialValue: classifierDescription ?? '',
        onChange: onClassifierDescriptionChange,
        allowEmptySubmitToCancel: true,
        showLabelWithValue: true,
        labelValueSeparator: ': ',
        resetCursorOnUpdate: true,
      });
    }
  }
  
  // ========== 3. No 选项 ==========
  if (noInputMode) {
    options.push({
      type: 'input',
      label: 'No',
      value: 'no',
      placeholder: 'and tell Claude what to do differently',
      onChange: onRejectFeedbackChange,
      allowEmptySubmitToCancel: true,
    });
  } else {
    options.push({ label: 'No', value: 'no' });
  }
  
  return options;
}
```

### 3.3 辅助函数实现

#### 3.3.1 描述去重检测

```typescript
/**
 * Check if a description already exists in the allow list.
 * Compares lowercase and trailing-whitespace-trimmed versions.
 */
function descriptionAlreadyExists(description: string, existingDescriptions: string[]): boolean {
  const normalized = description.toLowerCase().trimEnd();
  return existingDescriptions.some(existing => 
    existing.toLowerCase().trimEnd() === normalized
  );
}
```

**设计考量**:
- 大小写不敏感："Run Tests" 和 "run tests" 视为重复
- 尾部空格忽略：避免因格式差异导致的误判
- 时间复杂度 O(n)，n 为已有描述数量（通常很小）

#### 3.3.2 重定向剥离

```typescript
/**
 * Strip output redirections so filenames don't show as commands in the label.
 */
function stripBashRedirections(command: string): string {
  const { commandWithoutRedirections, redirections } = extractOutputRedirections(command);
  // Only use stripped version if there were actual redirections
  return redirections.length > 0 ? commandWithoutRedirections : command;
}
```

**使用场景**:
- 输入: `"python script.py > output.txt"`
- 输出: `"python script.py"`
- 目的：在选项标签中只显示命令本身，不显示输出文件

### 3.4 选项配置数据结构

```typescript
// OptionWithDescription 类型（来自 select.tsx）
type OptionWithDescription<T> = 
  | (BaseOption<T> & { type?: 'text' })
  | (BaseOption<T> & { 
      type: 'input';
      onChange: (value: string) => void;
      placeholder?: string;
      initialValue?: string;
      allowEmptySubmitToCancel?: boolean;  // 空提交视为取消
      showLabelWithValue?: boolean;        // 始终显示标签
      labelValueSeparator?: string;        // 标签与值分隔符
      resetCursorOnUpdate?: boolean;       // 更新时重置光标
    });

type BaseOption<T> = {
  label: ReactNode;      // 显示标签
  value: T;              // 选项值
  description?: string;  // 描述（可选）
  dimDescription?: boolean;
  disabled?: boolean;    // 是否禁用
};
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 文件路径 | 用途 | 关键导出 |
|---------|------|---------|
| `src/tools/BashTool/toolName.js` | Bash 工具名常量 | `BASH_TOOL_NAME` |
| `src/utils/bash/commands.js` | 重定向提取 | `extractOutputRedirections()` |
| `src/utils/permissions/bashClassifier.js` | 分类器检测 | `isClassifierPermissionsEnabled()` |
| `src/utils/permissions/PermissionResult.js` | 决策原因类型 | `PermissionDecisionReason` |
| `src/utils/permissions/PermissionUpdateSchema.js` | 权限更新类型 | `PermissionUpdate` |
| `src/utils/permissions/permissionsLoader.js` | 权限加载 | `shouldShowAlwaysAllowOptions()` |
| `src/components/CustomSelect/select.js` | 选项类型 | `OptionWithDescription` |
| `src/components/permissions/shellPermissionHelpers.tsx` | 标签生成 | `generateShellSuggestionsLabel()` |

### 4.2 调用链

```
BashPermissionRequest.tsx
    │
    ├── 准备参数
    │   ├── suggestions = toolUseConfirm.permissionResult.suggestions
    │   ├── decisionReason = toolUseConfirm.permissionResult.decisionReason
    │   ├── classifierDescription (来自 useState)
    │   ├── initialClassifierDescriptionEmpty (来自 useState)
    │   ├── existingAllowDescriptions = getBashPromptAllowDescriptions(context)
    │   ├── yesInputMode / noInputMode (来自 useShellPermissionFeedback)
    │   ├── editablePrefix (来自 useState)
    │   └── onEditablePrefixChange (useCallback)
    │
    ▼
bashToolUseOptions({...params})
    │
    ├── 检查 shouldShowAlwaysAllowOptions()
    │   └── 如果 allowManagedPermissionRulesOnly 启用，跳过所有"始终允许"选项
    │
    ├── 检查 hasNonBashSuggestions
    │   ├── addDirectories 类型建议 → 包含非 Bash 规则
    │   └── Read 工具规则 → 包含非 Bash 规则
    │
    ├── 决定显示哪种"始终允许"选项
    │   ├── editablePrefix 可用 + 无非 Bash 建议 → yes-prefix-edited
    │   ├── 有建议 → yes-apply-suggestions (通过 generateShellSuggestionsLabel)
    │   └── 无建议 → 不显示"始终允许"选项
    │
    ├── 检查分类器选项条件（ANT-ONLY）
    │   ├── "external" === 'ant'（编译时 false，外部构建禁用）
    │   ├── editablePrefix 未显示
    │   ├── isClassifierPermissionsEnabled() = true
    │   ├── 描述非空
    │   ├── 描述未重复
    │   └── 决策原因不是 classifier
    │   └── 全部满足 → yes-classifier-reviewed
    │
    └── 返回 OptionWithDescription[]
            │
            ▼
    Select 组件渲染选项列表
            │
            ▼
    用户选择 → onSelect(value) → BashPermissionRequestInner
```

---

## 5. 依赖与外部交互

### 5.1 权限系统集成

```
bashToolUseOptions
    │
    ├── 输入: PermissionUpdate[] (来自后端权限检查)
    │   ├── addRules: 添加规则建议
    │   ├── addDirectories: 添加目录权限
    │   └── setMode: 设置权限模式
    │
    ├── 输入: PermissionDecisionReason (决策原因)
    │   ├── type: 'rule' | 'classifier' | 'subcommandResults' | ...
    │   └── 影响: yes-classifier-reviewed 的显示条件
    │
    └── 输出: OptionWithDescription[]
        └── 被 Select 组件渲染为交互选项
```

### 5.2 分类器集成（ANT-ONLY）

```typescript
// 编译时条件，外部构建中恒为 false
if ("external" === 'ant' && ...) {
  options.push({
    value: 'yes-classifier-reviewed',
    // ...
  });
}
```

**说明**:
- `"external" === 'ant'` 是编译时常量表达式
- 在外部构建中，字符串 `"external"` 永远不会等于 `'ant'`，因此整个条件块被 DCE（死代码消除）
- 这是 feature flag 模式的一种实现，确保分类器功能只在内部构建中可用

### 5.3 配置影响

| 配置项 | 来源 | 影响 |
|--------|------|------|
| `allowManagedPermissionRulesOnly` | policySettings | 为 true 时隐藏所有"始终允许"选项 |
| `BASH_CLASSIFIER` | feature flag | 启用时可能显示 yes-classifier-reviewed |
| `USER_TYPE === 'ant'` | 环境变量 | 启用内部调试功能 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 逻辑风险

| 风险点 | 描述 | 缓解措施 |
|--------|------|---------|
| 选项顺序依赖 | 代码依赖 `options.some(o => o.value === 'yes-prefix-edited')` 检测 | 确保检查在 push 之后进行 |
| 条件竞争 | `editablePrefix` 和 `suggestions` 可能异步变化 | 调用方确保状态一致性 |
| 标签过长 | `generateShellSuggestionsLabel` 可能生成超长标签 | 使用 `commandListDisplayTruncated` 截断 |

#### 6.1.2 用户体验风险

| 风险点 | 描述 | 缓解措施 |
|--------|------|---------|
| 选项过多 | 最多可能显示 5 个选项，造成选择困难 | 优先级排序，最重要的在前 |
| 输入模式困惑 | Tab 切换输入模式可能不被用户发现 | 提示文本 "Tab to amend" |
| 描述重复误判 | 大小写/空格差异导致重复规则 | `descriptionAlreadyExists` 规范化比较 |

### 6.2 边界条件

```typescript
// 1. 空建议
suggestions = [] 
→ 不显示 yes-apply-suggestions 或 yes-prefix-edited
→ 只显示 Yes / No

// 2. 纯目录建议
suggestions = [{ type: 'addDirectories', directories: ['/path'] }]
→ hasNonBashSuggestions = true（addDirectories 被视为非 Bash）
→ 不显示 yes-prefix-edited
→ 显示 yes-apply-suggestions（通过 generateShellSuggestionsLabel）

// 3. 混合建议
suggestions = [
  { type: 'addRules', rules: [{ toolName: 'Read', ... }] },
  { type: 'addRules', rules: [{ toolName: 'Bash', ... }] }
]
→ hasNonBashSuggestions = true
→ 不显示 yes-prefix-edited

// 4. 空描述
classifierDescription = ''
→ initialClassifierDescriptionEmpty = true
→ 不显示 yes-classifier-reviewed

// 5. 重复描述
existingAllowDescriptions = ['run tests']
classifierDescription = 'Run Tests'
→ descriptionAlreadyExists 返回 true
→ 不显示 yes-classifier-reviewed

// 6. 分类器触发
decisionReason.type = 'classifier'
→ 不显示 yes-classifier-reviewed（避免与分类器决策冲突）

// 7. 管理权限模式
shouldShowAlwaysAllowOptions() = false
→ 跳过所有"始终允许"选项
→ 只显示 Yes / No
```

### 6.3 改进建议

#### 6.3.1 短期优化

1. **选项优先级动态调整**: 基于用户历史选择行为，动态调整选项顺序
2. **描述重复检测增强**: 使用模糊匹配（如 Levenshtein 距离）检测相似描述
3. **输入模式提示优化**: 在输入框下方显示更明确的操作提示

#### 6.3.2 中期改进

1. **智能前缀建议**: 基于命令历史，推荐最常用的前缀规则
2. **批量规则编辑**: 支持在选项中直接编辑多个建议规则
3. **规则预览**: 悬停时显示规则将匹配哪些历史命令

#### 6.3.3 长期架构

1. **选项配置化**: 将选项生成逻辑配置化，支持插件扩展
2. **A/B 测试框架**: 支持不同选项布局的 A/B 测试
3. **个性化推荐**: 基于用户角色和项目类型推荐选项

### 6.4 测试建议

```typescript
// 关键测试用例

describe('bashToolUseOptions', () => {
  // 基础选项
  it('应始终返回 Yes 和 No 选项', () => {});
  
  // 建议选项
  it('有建议时显示 yes-apply-suggestions', () => {});
  it('建议包含非 Bash 规则时显示 yes-apply-suggestions', () => {});
  it('建议为空时不显示始终允许选项', () => {});
  
  // 前缀编辑选项
  it('editablePrefix 可用时显示 yes-prefix-edited', () => {});
  it('有非 Bash 建议时不显示 yes-prefix-edited', () => {});
  
  // 分类器选项（ANT-ONLY）
  it('分类器启用且描述非空时显示 yes-classifier-reviewed', () => {});
  it('描述重复时不显示 yes-classifier-reviewed', () => {});
  it('决策原因是 classifier 时不显示 yes-classifier-reviewed', () => {});
  
  // 管理权限模式
  it('allowManagedPermissionRulesOnly 启用时隐藏始终允许选项', () => {});
  
  // 输入模式
  it('yesInputMode=true 时 Yes 变为输入类型', () => {});
  it('noInputMode=true 时 No 变为输入类型', () => {});
  
  // 重定向剥离
  it('应剥离输出重定向生成标签', () => {});
});
```

---

## 7. 附录

### 7.1 类型定义完整版

```typescript
// PermissionUpdate 类型（简化）
type PermissionUpdate =
  | { type: 'addRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'addDirectories'; directories: string[]; destination: PermissionUpdateDestination }
  | { type: 'setMode'; mode: PermissionMode; destination: PermissionUpdateDestination }
  | { type: 'replaceRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'removeRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'removeDirectories'; directories: string[]; destination: PermissionUpdateDestination };

// PermissionDecisionReason 类型（简化）
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'safetyCheck'; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'other'; reason: string };
```

### 7.2 代码统计

- **文件大小**: ~147 行
- **核心逻辑**: ~100 行
- **辅助函数**: 2 个（`descriptionAlreadyExists`, `stripBashRedirections`）
- **类型定义**: 1 个（`BashToolUseOption`）

### 7.3 变更历史

| 日期 | 变更 | 说明 |
|------|------|------|
| 2024-Q4 | 初始实现 | 基础选项生成逻辑 |
| 2025-Q1 | 添加分类器选项 | `yes-classifier-reviewed`（ANT-ONLY） |
| 2025-Q1 | 添加前缀编辑选项 | `yes-prefix-edited` 替代 Haiku 建议 |
| 2025-Q2 | 管理权限模式支持 | `shouldShowAlwaysAllowOptions()` 检查 |
