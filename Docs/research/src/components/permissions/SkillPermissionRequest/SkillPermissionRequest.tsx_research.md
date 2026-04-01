# SkillPermissionRequest.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位

`SkillPermissionRequest` 是 Claude Code 权限系统中的一个专用 React 组件，负责处理 **Skill 工具（SkillTool）** 的权限请求交互。当 Claude 尝试执行某个 skill（如 `commit`, `review-pr`, `pdf` 等）且需要用户确认时，该组件渲染权限请求对话框。

### 1.2 核心职责

1. **权限请求展示**：向用户展示 Skill 执行请求，包括 skill 名称和描述
2. **用户决策处理**：提供多种用户选项（同意、拒绝、永久同意等）
3. **权限规则管理**：支持用户添加"总是允许"规则（精确匹配或前缀匹配）
4. **分析事件上报**：记录用户交互事件用于遥测分析
5. **反馈收集**：支持用户提交接受/拒绝的反馈信息

### 1.3 触发场景

- 当 `SkillTool.checkPermissions()` 返回 `behavior: 'ask'` 时
- 用户未配置对应 skill 的自动允许规则
- 该 skill 不在安全属性白名单内

---

## 2. 功能点目的

### 2.1 选项类型

组件提供四种用户选项：

| 选项值 | 含义 | 行为 |
|--------|------|------|
| `yes` | 仅本次允许 | 执行 skill，不添加规则 |
| `yes-exact` | 总是允许此 skill | 添加精确匹配规则到 localSettings |
| `yes-prefix` | 总是允许此前缀 skill | 添加前缀匹配规则（如 `review:*`） |
| `no` | 拒绝 | 拒绝执行，可选提供反馈 |

### 2.2 "总是允许"选项的条件显示

通过 `shouldShowAlwaysAllowOptions()` 控制：
- 当 `allowManagedPermissionRulesOnly` 启用时隐藏（企业策略模式）
- 防止在受管环境中用户绕过策略限制

### 2.3 前缀匹配逻辑

当 skill 名称包含空格（如 `review-pr 123`）时：
- 提取空格前的部分作为前缀（如 `review-pr`）
- 生成前缀规则格式：`{prefix}:*`（如 `review-pr:*`）
- 允许匹配该前缀的所有 skill 调用

---

## 3. 具体技术实现

### 3.1 关键数据结构

```typescript
// 选项值类型
type SkillOptionValue = 'yes' | 'yes-exact' | 'yes-prefix' | 'no';

// PermissionPromptOption 结构（来自 PermissionPrompt.tsx）
type PermissionPromptOption<T extends string> = {
  value: T;
  label: ReactNode;
  feedbackConfig?: {
    type: 'accept' | 'reject';
    placeholder?: string;
  };
  keybinding?: KeybindingAction;
};

// ToolAnalyticsContext 结构
type ToolAnalyticsContext = {
  toolName: string;
  isMcp: boolean;
};
```

### 3.2 输入解析流程

```typescript
// 使用 SkillTool.inputSchema 解析输入
const parseInput = (input: unknown): string => {
  const result = SkillTool.inputSchema.safeParse(input);
  if (!result.success) {
    logError(new Error(`Failed to parse skill tool input: ${result.error.message}`));
    return "";
  }
  return result.data.skill;
};
```

输入格式（来自 SkillTool.ts）：
```typescript
{
  skill: string;  // skill 名称，如 "commit", "review-pr"
  args?: string;  // 可选参数
}
```

### 3.3 权限更新协议

当用户选择 "yes-exact" 或 "yes-prefix" 时，生成 `PermissionUpdate`：

```typescript
// 精确匹配规则
{
  type: "addRules",
  rules: [{
    toolName: SKILL_TOOL_NAME,  // "Skill"
    ruleContent: skill          // 如 "commit"
  }],
  behavior: "allow",
  destination: "localSettings"
}

// 前缀匹配规则
{
  type: "addRules",
  rules: [{
    toolName: SKILL_TOOL_NAME,
    ruleContent: `${commandPrefix}:*`  // 如 "review:*"
  }],
  behavior: "allow",
  destination: "localSettings"
}
```

### 3.4 分析事件上报

使用 `logUnaryEvent` 记录事件：

```typescript
// 接受事件
logUnaryEvent({
  completion_type: "tool_use_single",
  event: "accept",
  metadata: {
    language_name: "none",
    message_id: toolUseConfirm.assistantMessage.message.id,
    platform: env.platform
  }
});

// 拒绝事件
logUnaryEvent({
  completion_type: "tool_use_single",
  event: "reject",
  metadata: { /* ... */ }
});
```

### 3.5 React Compiler 缓存模式

组件使用 React Compiler（`_c` 函数）进行自动记忆化：
- 使用 `$[n]` 数组存储缓存值
- `Symbol.for("react.memo_cache_sentinel")` 作为未初始化标记
- 比较依赖变化决定是否重新计算

---

## 4. 关键代码路径与文件引用

### 4.1 组件入口

**文件**: `src/components/permissions/SkillPermissionRequest/SkillPermissionRequest.tsx`

```typescript
export function SkillPermissionRequest(props: PermissionRequestProps) {
  const { toolUseConfirm, onDone, onReject, workerBadge } = props;
  // ...
}
```

### 4.2 调用链

```
PermissionRequest.tsx (PermissionRequest 组件)
  └── permissionComponentForTool(tool)
       └── case SkillTool: return SkillPermissionRequest
```

### 4.3 核心依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/components/permissions/PermissionRequest.tsx` | 父组件，负责路由到具体权限请求组件 |
| `src/components/permissions/PermissionPrompt.tsx` | 通用权限提示 UI 组件 |
| `src/components/permissions/PermissionDialog.tsx` | 权限对话框容器组件 |
| `src/components/permissions/PermissionRuleExplanation.tsx` | 权限规则解释展示 |
| `src/components/permissions/hooks.ts` | `usePermissionRequestLogging` Hook |
| `src/tools/SkillTool/SkillTool.ts` | Skill 工具实现 |
| `src/tools/SkillTool/constants.ts` | `SKILL_TOOL_NAME` 常量 |
| `src/utils/permissions/permissionsLoader.ts` | `shouldShowAlwaysAllowOptions()` |
| `src/utils/permissions/PermissionResult.ts` | PermissionDecision 类型 |
| `src/utils/permissions/PermissionUpdateSchema.ts` | PermissionUpdate 类型 |
| `src/utils/unaryLogging.ts` | `logUnaryEvent` 函数 |
| `src/services/analytics/metadata.ts` | `sanitizeToolNameForAnalytics` |
| `src/bootstrap/state.ts` | `getOriginalCwd()` |
| `src/utils/env.ts` | `env.platform` |

### 4.4 类型定义

**文件**: `src/types/permissions.ts`

```typescript
export type PermissionAskDecision<Input extends { [key: string]: unknown } = { [key: string]: unknown }> = {
  behavior: 'ask';
  message: string;
  updatedInput?: Input;
  decisionReason?: PermissionDecisionReason;
  suggestions?: PermissionUpdate[];
  metadata?: PermissionMetadata;  // 包含 command 对象
};

export type PermissionUpdate = {
  type: 'addRules';
  destination: PermissionUpdateDestination;
  rules: PermissionRuleValue[];
  behavior: PermissionBehavior;
};
```

---

## 5. 依赖与外部交互

### 5.1 Props 接口

```typescript
type PermissionRequestProps<Input extends AnyObject = AnyObject> = {
  toolUseConfirm: ToolUseConfirm<Input>;
  toolUseContext: ToolUseContext;
  onDone(): void;
  onReject(): void;
  verbose: boolean;
  workerBadge: WorkerBadgeProps | undefined;
};

type ToolUseConfirm<Input extends AnyObject = AnyObject> = {
  assistantMessage: AssistantMessage;
  tool: Tool<Input>;
  description: string;
  input: z.infer<Input>;
  toolUseContext: ToolUseContext;
  toolUseID: string;
  permissionResult: PermissionDecision;
  permissionPromptStartTimeMs: number;
  onAllow(updatedInput, permissionUpdates, feedback?, contentBlocks?): void;
  onReject(feedback?, contentBlocks?): void;
  // ... 其他字段
};
```

### 5.2 外部 Hook 依赖

**`usePermissionRequestLogging`** (来自 `hooks.ts`):
- 在组件挂载时记录 `tengu_tool_use_show_permission_request` 分析事件
- 更新应用状态中的 `permissionPromptCount`
- 记录 unary 事件用于使用分析

### 5.3 权限规则持久化

通过 `toolUseConfirm.onAllow()` 回调，将新规则传递给父组件：
- 父组件调用 `addPermissionRulesToSettings()` (来自 `permissionsLoader.ts`)
- 规则被写入 `localSettings`（通常是 `.claude/settings.json`）

### 5.4 SkillTool 的权限检查逻辑

**文件**: `src/tools/SkillTool/SkillTool.ts`

```typescript
async checkPermissions({ skill, args }, context): Promise<PermissionDecision> {
  // 1. 解析 skill 名称
  const commandName = trimmed.startsWith('/') ? trimmed.substring(1) : trimmed;
  
  // 2. 检查 deny 规则
  const denyRules = getRuleByContentsForTool(permissionContext, SkillTool, 'deny');
  // ...
  
  // 3. 检查 allow 规则
  const allowRules = getRuleByContentsForTool(permissionContext, SkillTool, 'allow');
  // ...
  
  // 4. 检查是否为安全属性 skill
  if (skillHasOnlySafeProperties(commandObj)) {
    return { behavior: 'allow', ... };
  }
  
  // 5. 默认返回 ask，附带建议规则
  return {
    behavior: 'ask',
    message: `Execute skill: ${commandName}`,
    suggestions: [
      { type: 'addRules', rules: [{ toolName: SKILL_TOOL_NAME, ruleContent: commandName }], ... },
      { type: 'addRules', rules: [{ toolName: SKILL_TOOL_NAME, ruleContent: `${commandName}:*` }], ... }
    ],
    metadata: commandObj ? { command: commandObj } : undefined
  };
}
```

### 5.5 安全属性白名单

**文件**: `src/tools/SkillTool/SkillTool.ts` (lines 875-907)

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames', 'model', 'effort',
  'source', 'pluginInfo', 'disableNonInteractive', 'skillRoot', 'context',
  'agent', 'getPromptForCommand', 'frontmatterKeys', 'name', 'description',
  // ... 等其他安全属性
]);
```

只有包含白名单内属性的 skill 才会被自动允许。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 输入解析失败处理

```typescript
const parseInput = (input) => {
  const result = SkillTool.inputSchema.safeParse(input);
  if (!result.success) {
    logError(new Error(`Failed to parse skill tool input: ${result.error.message}`));
    return "";  // 返回空字符串，组件会继续渲染但显示空 skill 名
  }
  return result.data.skill;
};
```

**风险**: 输入解析失败时返回空字符串，UI 会显示 `"Use skill ""?"`，用户体验不佳。

#### 6.1.2 前缀匹配逻辑限制

```typescript
const spaceIndex = skill.indexOf(" ");
if (spaceIndex > 0) {
  const commandPrefix = skill.substring(0, spaceIndex);
  // ...
}
```

**风险**: 仅处理第一个空格，对于复杂参数格式（如 `skill "arg with space"`）可能无法正确提取前缀。

#### 6.1.3 规则目的地硬编码

```typescript
destination: "localSettings"  // 硬编码，不支持用户选择其他目的地
```

**限制**: 用户无法选择将规则保存到 `userSettings` 或 `projectSettings`。

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| skill 名称为空字符串 | 显示 `"Use skill ""?"`，可能令人困惑 |
| skill 名称以 `/` 开头 | SkillTool.validateInput 会处理，但 UI 显示原始名称 |
| 企业策略模式 (`allowManagedPermissionRulesOnly`) | "总是允许"选项被隐藏 |
| MCP skill | `isMcp` 标志会传递给分析事件 |
| 无描述信息 | `commandObj?.description` 为 undefined，不显示描述区域 |

### 6.3 改进建议

#### 6.3.1 增强错误处理

```typescript
// 建议：添加输入验证和错误状态
const skill = parseInput(toolUseConfirm.input);
if (!skill) {
  return <ErrorDialog message="Invalid skill input" onDone={onDone} />;
}
```

#### 6.3.2 支持自定义规则目的地

允许用户选择规则保存位置（用户级、项目级、本地级）：

```typescript
// 潜在改进：添加目的地选择器
const [ruleDestination, setRuleDestination] = useState<EditableSettingSource>('localSettings');
```

#### 6.3.3 规则预览

在添加"总是允许"规则前，展示规则的具体内容和影响范围：

```typescript
// 潜在改进：规则预览组件
<RulePreview 
  rule={{ toolName: SKILL_TOOL_NAME, ruleContent: skill }}
  behavior="allow"
/>
```

#### 6.3.4 支持正则/通配符匹配

当前仅支持精确匹配和前缀匹配（`:*`），可考虑支持更灵活的匹配模式：

```typescript
// 潜在改进：支持 glob 模式
ruleContent: "review-*"  // 匹配 review-pr, review-code 等
```

#### 6.3.5 分析事件增强

当前仅记录基本接受/拒绝事件，可扩展记录：
- 用户选择的规则类型（exact vs prefix）
- skill 来源（built-in, bundled, plugin, MCP）
- 是否提供了反馈信息

### 6.4 测试建议

1. **单元测试**：测试 `parseInput` 对各种输入的处理
2. **集成测试**：测试选项选择与 `onAllow` 回调的交互
3. **边界测试**：空 skill 名、超长 skill 名、特殊字符
4. **策略测试**：`allowManagedPermissionRulesOnly` 模式下的选项显示

---

## 7. 相关代码片段

### 7.1 选项构建逻辑

```typescript
// 基础选项：仅本次允许
const baseOptions = [{
  label: "Yes",
  value: "yes",
  feedbackConfig: { type: "accept" }
}];

// "总是允许"选项（条件显示）
const alwaysAllowOptions = [];
if (showAlwaysAllowOptions) {
  // 精确匹配选项
  alwaysAllowOptions.push({
    label: <Text>Yes, and don't ask again for {skill} in {originalCwd}</Text>,
    value: "yes-exact"
  });
  
  // 前缀匹配选项（如果 skill 包含空格）
  if (spaceIndex > 0) {
    alwaysAllowOptions.push({
      label: <Text>Yes, and don't ask again for {commandPrefix}:* commands in {originalCwd}</Text>,
      value: "yes-prefix"
    });
  }
}

// 拒绝选项
const noOption = {
  label: "No",
  value: "no",
  feedbackConfig: { type: "reject" }
};

const options = [...baseOptions, ...alwaysAllowOptions, noOption];
```

### 7.2 选择处理逻辑

```typescript
const handleSelect = (value: SkillOptionValue, feedback?: string) => {
  switch (value) {
    case "yes":
      logUnaryEvent({ event: "accept", ... });
      toolUseConfirm.onAllow(toolUseConfirm.input, [], feedback);
      onDone();
      break;
      
    case "yes-exact":
      logUnaryEvent({ event: "accept", ... });
      toolUseConfirm.onAllow(toolUseConfirm.input, [{
        type: "addRules",
        rules: [{ toolName: SKILL_TOOL_NAME, ruleContent: skill }],
        behavior: "allow",
        destination: "localSettings"
      }]);
      onDone();
      break;
      
    case "yes-prefix":
      // 类似 yes-exact，但使用 `${commandPrefix}:*`
      break;
      
    case "no":
      logUnaryEvent({ event: "reject", ... });
      toolUseConfirm.onReject(feedback);
      onReject();
      onDone();
      break;
  }
};
```

---

## 8. 总结

`SkillPermissionRequest` 是一个职责单一、结构清晰的权限请求组件。它：

1. **专注于 Skill 工具**：只处理 `SkillTool` 的权限请求
2. **提供灵活的允许选项**：支持单次允许、永久允许（精确/前缀）
3. **遵循权限系统规范**：使用标准的 `PermissionPrompt` 和 `PermissionDialog`
4. **支持分析遥测**：完整记录用户交互事件

主要改进空间在于：
- 输入验证和错误状态处理
- 规则目的地的可配置性
- 更丰富的规则预览和说明
