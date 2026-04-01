# PowerShellPermissionRequest.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`PowerShellPermissionRequest.tsx` 是 Claude Code 中 PowerShell 工具权限请求对话框的 React 组件实现。当用户执行 PowerShell 命令需要权限确认时，此组件负责渲染交互式权限请求界面，收集用户决策（允许/拒绝/永久允许），并处理相关的权限规则更新。

### 1.2 使用场景
- **PowerShell 命令执行前权限确认**：当 PowerShellTool 检测到需要用户确认的命令时
- **权限规则管理**：用户可以通过对话框添加新的权限规则（如"以后不再询问此类命令"）
- **反馈收集**：用户可以提供额外的指令或反馈给 Claude
- **调试信息展示**：在调试模式下显示权限决策的详细信息

### 1.3 与 BashPermissionRequest 的关系
该组件与 `BashPermissionRequest` 共享大量设计模式和钩子（如 `useShellPermissionFeedback`），但针对 PowerShell 的语法特性（如大小写不敏感、Cmdlet 命名规范等）有专门的处理。

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| 命令展示 | 显示待执行的 PowerShell 命令及其描述 |
| 权限选项 | 提供 Yes/No/Yes with prefix 三种决策选项 |
| 可编辑前缀 | 允许用户编辑"不再询问"的命令前缀规则 |
| 反馈输入 | 支持用户在允许或拒绝时附加说明 |
| 解释器集成 | 集成权限解释器，说明命令的风险级别 |
| 调试模式 | 显示权限决策的详细技术信息 |

### 2.2 安全功能
- **破坏性命令警告**：通过 `getDestructiveCommandWarning` 检测并警告潜在危险操作
- 支持 `tengu_destructive_command_warning` 功能开关控制

---

## 3. 具体技术实现

### 3.1 组件架构

```typescript
// 主组件入口
export function PowerShellPermissionRequest(props: PermissionRequestProps): React.ReactNode
```

**Props 结构** (`PermissionRequestProps`):
- `toolUseConfirm`: 工具使用确认对象，包含命令输入、权限结果、回调函数
- `toolUseContext`: 工具使用上下文，包含消息历史和选项
- `onDone`: 对话框完成回调
- `onReject`: 拒绝回调
- `workerBadge`: 工作进程徽章信息

### 3.2 关键流程

#### 3.2.1 初始化流程
```
1. 解析命令输入 (PowerShellTool.inputSchema.parse)
2. 初始化权限解释器状态 (usePermissionExplainerUI)
3. 初始化反馈状态 (useShellPermissionFeedback)
4. 计算破坏性命令警告
5. 初始化可编辑前缀状态
```

#### 3.2.2 可编辑前缀计算流程
```typescript
// 行 72-90: 前缀提取逻辑
const [editablePrefix, setEditablePrefix] = useState<string | undefined>(
  command.includes('\n') ? undefined : command
);

useEffect(() => {
  getCompoundCommandPrefixesStatic(command, element => isAllowlistedCommand(element, element.text))
    .then(prefixes => {
      if (prefixes.length > 0) {
        setEditablePrefix(`${prefixes[0]}:*`);
      }
    });
}, [command]);
```

**关键设计决策**：
- 单行命令：初始化为完整命令，异步优化为提取前缀
- 多行命令：设为 `undefined`，隐藏"不再询问"选项（因为多行命令通常是单次使用）
- 使用 `isAllowlistedCommand` 过滤已自动允许的子命令

#### 3.2.3 用户决策处理流程
```typescript
// 行 117-194: onSelect 处理函数
function onSelect(value: string) {
  // 1. 记录分析事件
  logEvent('tengu_permission_request_option_selected', {...});
  
  // 2. 根据选项类型处理
  switch (value) {
    case 'yes-prefix-edited':
      // 处理带前缀编辑的允许
      // 创建 PermissionUpdate 规则
      toolUseConfirm.onAllow(toolUseConfirm.input, prefixUpdates);
      break;
    case 'yes':
      // 普通允许，可选反馈
      toolUseConfirm.onAllow(toolUseConfirm.input, [], trimmedFeedback || undefined);
      break;
    case 'yes-apply-suggestions':
      // 应用系统建议的权限规则
      toolUseConfirm.onAllow(toolUseConfirm.input, permissionUpdates);
      break;
    case 'no':
      // 拒绝，可选反馈
      handleReject(trimmedFeedback || undefined);
      break;
  }
}
```

### 3.3 数据结构

#### 3.3.1 PermissionUpdate 结构
```typescript
// 用于添加新权限规则
type PermissionUpdate = {
  type: 'addRules';
  rules: Array<{
    toolName: string;      // 'PowerShell'
    ruleContent: string;   // 命令前缀，如 'Get-Process:*'
  }>;
  behavior: 'allow';
  destination: 'localSettings'; // 或 userSettings/projectSettings
};
```

#### 3.3.2 分析事件映射
```typescript
const optionIndex: Record<string, number> = {
  yes: 1,
  'yes-apply-suggestions': 2,
  'yes-prefix-edited': 2,
  no: 3
};
```

### 3.4 关键 Hook 依赖

| Hook | 来源 | 用途 |
|------|------|------|
| `usePermissionExplainerUI` | PermissionExplanation.tsx | 管理权限解释器的显示状态和异步获取 |
| `useShellPermissionFeedback` | useShellPermissionFeedback.ts | 管理 Yes/No 的反馈输入模式和状态 |
| `usePermissionRequestLogging` | hooks.ts | 记录权限请求的分析事件 |
| `useKeybinding` | useKeybinding.ts | 绑定调试切换快捷键 (Ctrl+D) |

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖文件

```
src/components/permissions/PowerShellPermissionRequest/
├── PowerShellPermissionRequest.tsx    # 本文件
└── powershellToolUseOptions.tsx       # 选项生成逻辑

依赖的上级组件：
src/components/permissions/
├── PermissionRequest.tsx              # 权限请求分发器
├── PermissionDialog.tsx               # 对话框容器
├── PermissionExplanation.tsx          # 权限解释器
├── PermissionRuleExplanation.tsx      # 规则解释
├── PermissionDecisionDebugInfo.tsx    # 调试信息
├── hooks.ts                           # 通用权限钩子
├── useShellPermissionFeedback.ts      # 反馈状态管理
└── shellPermissionHelpers.tsx         # Shell 权限辅助函数

依赖的工具实现：
src/tools/PowerShellTool/
├── PowerShellTool.ts                  # 工具主实现
├── toolName.ts                        # 工具名称常量
├── destructiveCommandWarning.ts       # 破坏性命令检测
├── readOnlyValidation.ts              # 只读命令验证
└── powershellPermissions.ts           # 权限检查逻辑

依赖的通用工具：
src/utils/powershell/
├── staticPrefix.ts                    # 静态前缀提取
└── parser.ts                          # PowerShell 解析器
```

### 4.2 关键代码路径

**路径 1: 组件渲染流程**
```
PermissionRequest.tsx:permissionComponentForTool() 
  → PowerShellPermissionRequest.tsx:PowerShellPermissionRequest()
    → PermissionDialog.tsx:PermissionDialog
      → 渲染命令展示、解释器、选项选择器
```

**路径 2: 前缀提取流程**
```
PowerShellPermissionRequest.tsx
  → staticPrefix.ts:getCompoundCommandPrefixesStatic()
    → parser.ts:parsePowerShellCommand()
    → readOnlyValidation.ts:isAllowlistedCommand()
    → staticPrefix.ts:extractPrefixFromElement()
      → bash/registry.ts:getCommandSpec()  // 复用 Bash 的 fig spec
      → shell/specPrefix.ts:buildPrefix()
```

**路径 3: 决策处理流程**
```
Select.tsx:onChange()
  → PowerShellPermissionRequest.tsx:onSelect()
    → toolUseConfirm.onAllow() / handleReject()
      → 更新权限规则 / 记录分析事件
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 类型 | 说明 |
|------|------|------|
| `../../../ink.js` | UI | Ink React 组件库 (Box, Text, useTheme) |
| `../../../keybindings/useKeybinding.js` | 交互 | 键盘快捷键绑定 |
| `../../../services/analytics/index.js` | 分析 | 分析事件记录 |
| `../../../services/analytics/metadata.js` | 分析 | 工具名称清理 |
| `../../../services/analytics/growthbook.js` | 功能开关 | 破坏性命令警告开关 |
| `../../../tools/PowerShellTool/*` | 业务逻辑 | PowerShell 工具相关实现 |
| `../../../utils/powershell/staticPrefix.js` | 工具 | 前缀提取 |
| `../../../utils/permissions/PermissionUpdateSchema.js` | 类型 | 权限更新类型定义 |
| `../../CustomSelect/select.js` | UI | 选择器组件 |
| `../hooks.js` | 钩子 | 权限请求日志记录 |
| `../PermissionDecisionDebugInfo.js` | UI | 调试信息组件 |
| `../PermissionDialog.js` | UI | 对话框容器 |
| `../PermissionExplanation.js` | UI | 解释器组件 |
| `../PermissionRequest.js` | 类型 | Props 类型定义 |
| `../PermissionRuleExplanation.js` | UI | 规则解释组件 |
| `../useShellPermissionFeedback.js` | 钩子 | 反馈状态管理 |
| `../utils.js` | 工具 | 权限事件日志工具 |

### 5.2 与 PowerShellTool 的交互

```typescript
// 输入解析
const { command, description } = PowerShellTool.inputSchema.parse(toolUseConfirm.input);

// 消息渲染
PowerShellTool.renderToolUseMessage({ command, description }, { theme, verbose: true });

// 工具名称
const toolNameForAnalytics = sanitizeToolNameForAnalytics(toolUseConfirm.tool.name);
```

### 5.3 与权限系统的交互

组件通过 `toolUseConfirm` 对象与权限系统交互：
- `toolUseConfirm.permissionResult`: 获取权限检查结果
- `toolUseConfirm.onAllow(input, updates, feedback)`: 允许执行
- `toolUseConfirm.onReject(feedback)`: 拒绝执行
- `toolUseConfirm.onUserInteraction()`: 通知用户交互

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 前缀提取安全风险
```typescript
// 行 80: 过滤已自动允许的子命令
getCompoundCommandPrefixesStatic(command, element => isAllowlistedCommand(element, element.text))
```
**风险**：如果 `isAllowlistedCommand` 存在误判，可能导致危险命令被错误地包含在前缀建议中。

#### 6.1.2 多行命令处理
```typescript
// 行 72: 多行命令不显示"不再询问"选项
command.includes('\n') ? undefined : command
```
**边界**：多行命令无法创建前缀规则，这可能导致用户体验不一致。

#### 6.1.3 用户编辑前缀风险
用户可以自由编辑前缀，可能创建过于宽泛的规则（如 `Remove-Item:*` 会匹配所有删除操作）。

### 6.2 边界情况

| 场景 | 处理方式 |
|------|----------|
| 空命令 | 已在 PowerShellTool 层处理，不会到达此组件 |
| 解析失败 | 前缀保持为原始命令，不显示编辑选项 |
| 所有子命令已自动允许 | 前缀列表为空，不显示"不再询问"选项 |
| 用户快速切换选项 | 通过 `hasUserEditedPrefix` ref 防止异步更新覆盖用户编辑 |
| 网络延迟（分析事件） | 使用 `usePermissionRequestLogging` 的 ref 防护避免重复记录 |

### 6.3 改进建议

#### 6.3.1 前缀验证增强
```typescript
// 建议：添加前缀安全性检查
function validatePrefix(prefix: string): { valid: boolean; warning?: string } {
  // 检查是否过于宽泛
  if (prefix.endsWith(':*') && !prefix.includes(' ')) {
    return { valid: true, warning: '此规则将匹配所有该命令的子命令' };
  }
  // 检查是否包含危险命令
  const dangerousCmds = ['Remove-Item', 'Invoke-Expression', 'Start-Process'];
  if (dangerousCmds.some(cmd => prefix.toLowerCase().includes(cmd.toLowerCase()))) {
    return { valid: true, warning: '此规则涉及潜在危险命令' };
  }
  return { valid: true };
}
```

#### 6.3.2 多行命令支持
考虑支持多行命令的前缀提取，或提供更清晰的说明为什么无法创建规则。

#### 6.3.3 性能优化
```typescript
// 当前：每次渲染都重新计算选项
const options = useMemo(() => powershellToolUseOptions({...}), [...]);

// 建议：将前缀提取也加入 useMemo 或移至单独的 hook
const { editablePrefix, isLoading } = useEditablePrefix(command);
```

#### 6.3.4 测试覆盖
建议增加以下测试场景：
1. 复合命令的前缀提取（`Get-Process; git status`）
2. 用户编辑前缀后的行为
3. 多行命令的处理
4. 调试模式的切换
5. 分析事件的正确记录

### 6.4 相关 Issue/PR 参考

根据代码注释，以下历史修复值得注意：
- **前缀提取优化**：使用 `getCompoundCommandPrefixesStatic` 替代 LLM 调用
- **多行命令处理**：明确禁用多行命令的"不再询问"选项
- **用户编辑保护**：使用 `hasUserEditedPrefix` ref 防止异步更新覆盖用户输入

---

## 7. 附录：关键类型定义

```typescript
// PermissionRequestProps 完整定义
interface PermissionRequestProps<Input extends AnyObject = AnyObject> {
  toolUseConfirm: ToolUseConfirm<Input>;
  toolUseContext: ToolUseContext;
  onDone(): void;
  onReject(): void;
  verbose: boolean;
  workerBadge: WorkerBadgeProps | undefined;
  setStickyFooter?: (jsx: React.ReactNode | null) => void;
}

// ToolUseConfirm 关键字段
interface ToolUseConfirm<Input extends AnyObject = AnyObject> {
  assistantMessage: AssistantMessage;
  tool: Tool<Input>;
  description: string;
  input: z.infer<Input>;
  toolUseContext: ToolUseContext;
  toolUseID: string;
  permissionResult: PermissionDecision;
  onUserInteraction(): void;
  onAllow(updatedInput: z.infer<Input>, permissionUpdates: PermissionUpdate[], feedback?: string): void;
  onReject(feedback?: string): void;
}
```
