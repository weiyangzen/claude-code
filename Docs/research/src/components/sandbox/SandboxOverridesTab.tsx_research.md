# SandboxOverridesTab.tsx 深度研究文档

## 1. 场景与职责

**SandboxOverridesTab** 是 Claude Code CLI 沙箱设置界面的"Overrides"标签页组件，负责管理沙箱的覆盖策略：

### 核心职责

1. **配置覆盖模式**: 允许用户在"开放模式"和"严格模式"之间选择
2. **策略锁定检测**: 检测设置是否被高优先级配置（如托管策略）锁定
3. **用户引导**: 解释不同模式的安全影响和适用场景

### 使用场景

- **企业环境**: 管理员通过 managed-settings.json 强制沙箱策略
- **安全敏感场景**: 用户希望禁止任何非沙箱执行
- **开发调试**: 临时允许命令在沙箱外执行以排查问题

### 覆盖模式说明

| 模式 | 配置值 | 行为 |
|------|--------|------|
| Allow unsandboxed fallback | `allowUnsandboxedCommands: true` | 命令在沙箱中失败时，可尝试在沙箱外执行 |
| Strict sandbox mode | `allowUnsandboxedCommands: false` | 所有命令必须在沙箱内执行，除非在 `excludedCommands` 中 |

---

## 2. 功能点目的

### 2.1 模式选择器

提供两个互斥选项：

```typescript
type OverrideMode = 'open' | 'closed'

const options = [
  { 
    label: 'Allow unsandboxed fallback (current)', 
    value: 'open' 
  },
  { 
    label: 'Strict sandbox mode', 
    value: 'closed' 
  }
]
```

### 2.2 策略锁定检测

```typescript
const isLocked = SandboxManager.areSandboxSettingsLockedByPolicy()
```

当设置被锁定时：
- 显示只读状态
- 告知用户设置由更高优先级配置管理
- 展示当前生效的设置值

### 2.3 状态检查

```typescript
const isEnabled = SandboxManager.isSandboxingEnabled()
const currentAllowUnsandboxed = SandboxManager.areUnsandboxedCommandsAllowed()
```

- 沙箱未启用时：提示用户先启用沙箱
- 沙箱已启用时：展示模式选择器

---

## 3. 具体技术实现

### 3.1 组件架构

```tsx
export function SandboxOverridesTab({ onComplete }: Props): React.ReactNode {
  const isEnabled = SandboxManager.isSandboxingEnabled()
  const isLocked = SandboxManager.areSandboxSettingsLockedByPolicy()
  const currentAllowUnsandboxed = SandboxManager.areUnsandboxedCommandsAllowed()
  
  // 沙箱未启用
  if (!isEnabled) {
    return <提示启用沙箱 />
  }
  
  // 设置被锁定
  if (isLocked) {
    return <只读状态展示 />
  }
  
  // 正常模式选择
  return <OverridesSelect onComplete={onComplete} currentMode={currentMode} />
}

// 内层组件：模式选择器
function OverridesSelect({ onComplete, currentMode }: SelectProps) {
  const [theme] = useTheme()
  const { headerFocused, focusHeader } = useTabHeaderFocus()
  
  const currentIndicator = color("success", theme)("(current)")
  
  const options = [
    { 
      label: currentMode === 'open' 
        ? `Allow unsandboxed fallback ${currentIndicator}` 
        : 'Allow unsandboxed fallback',
      value: 'open' 
    },
    { 
      label: currentMode === 'closed' 
        ? `Strict sandbox mode ${currentIndicator}` 
        : 'Strict sandbox mode',
      value: 'closed' 
    }
  ]
  
  const handleSelect = async (value: OverrideMode) => {
    await SandboxManager.setSandboxSettings({
      allowUnsandboxedCommands: value === 'open'
    })
    onComplete(成功消息)
  }
  
  return (
    <Box flexDirection="column" paddingY={1}>
      <Select options={options} onChange={handleSelect} ... />
      <说明文本 />
    </Box>
  )
}
```

### 3.2 关键设计决策

**组件拆分原因**（代码注释）：

```tsx
// Split so useTabHeaderFocus() only runs when the Select renders. Calling it
// above the early returns registers a down-arrow opt-in even when we return
// static text — pressing ↓ then blurs the header with no way back.
```

将 `OverridesSelect` 拆分为独立组件，确保 `useTabHeaderFocus()` 只在实际渲染 Select 时调用，避免在静态文本返回时注册键盘事件监听。

### 3.3 React Compiler 记忆化

**外层组件**（5 个槽位）：
- `$[0]`: 沙箱未启用提示
- `$[1]`: 设置锁定提示
- `$[2]`: 锁定状态完整展示
- `$[3-4]`: OverridesSelect 组件

**内层组件**（25 个槽位）：
- 主题颜色缓存
- 选项标签缓存
- 选项数组缓存
- 选择处理器缓存
- 各 JSX 片段缓存

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入 | 路径 | 用途 |
|------|------|------|
| React | 'react' | JSX 运行时 |
| Box, color, Link, Text, useTheme | '../../ink.js' | Ink UI 组件和主题 |
| CommandResultDisplay | '../../types/command.js' | 类型定义 |
| SandboxManager | '../../utils/sandbox/sandbox-adapter.js' | 沙箱管理器 |
| Select | '../CustomSelect/select.js' | 选择器组件 |
| useTabHeaderFocus | '../design-system/Tabs.js' | Tab 焦点管理 |

### 4.2 设置更新流程

```
用户选择模式
  ↓
handleSelect(value)
  ↓
SandboxManager.setSandboxSettings({ allowUnsandboxedCommands: boolean })
  ↓
updateSettingsForSource('localSettings', settings)
  ↓
写入 .claude/settings.local.json
  ↓
触发 settingsChangeDetector
  ↓
SandboxManager.refreshConfig()
  ↓
BaseSandboxManager.updateConfig(newConfig)
```

### 4.3 策略锁定检测

```typescript
function areSandboxSettingsLockedByPolicy(): boolean {
  const overridingSources = ['flagSettings', 'policySettings'] as const
  
  for (const source of overridingSources) {
    const settings = getSettingsForSource(source)
    if (
      settings?.sandbox?.enabled !== undefined ||
      settings?.sandbox?.autoAllowBashIfSandboxed !== undefined ||
      settings?.sandbox?.allowUnsandboxedCommands !== undefined
    ) {
      return true
    }
  }
  
  return false
}
```

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 |
|------|------|
| react | React 框架 |
| ink | 终端渲染框架 |
| @anthropic-ai/sandbox-runtime | 沙箱运行时（间接） |

### 5.2 内部模块依赖

```
SandboxOverridesTab.tsx
├── src/ink.ts                           # Ink 渲染层
│   └── 主题系统 (src/utils/theme.ts)
├── src/types/command.ts                 # 命令类型定义
├── src/utils/sandbox/sandbox-adapter.ts # 沙箱管理器
│   ├── src/utils/settings/settings.ts   # 设置读写
│   └── src/utils/settings/constants.ts  # 设置源常量
├── src/components/CustomSelect/select.tsx # 选择器组件
└── src/components/design-system/Tabs.tsx  # Tabs 焦点管理
```

### 5.3 设置优先级

```
高优先级（锁定来源）
├── policySettings    (managed-settings.json)
└── flagSettings      (CLI 参数)

低优先级（被锁定覆盖）
├── localSettings     (.claude/settings.local.json)
├── projectSettings   (.claude/settings.json)
└── userSettings      (~/.claude/settings.json)
```

---

## 6. 风险、边界与改进建议

### 6.1 安全风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 权限提升 | `allowUnsandboxedCommands: true` 允许绕过沙箱 | 明确提示安全风险 |
| 策略绕过 | 本地设置可能覆盖企业策略 | 策略锁定检测 |
| 配置漂移 | 设置更新后未立即生效 | 实时刷新配置 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 沙箱未启用 | 显示提示："Sandbox is not enabled..." |
| 设置被 policySettings 锁定 | 显示只读状态，告知由策略管理 |
| 设置被 flagSettings 锁定 | 同上 |
| 当前模式为 open | "Allow unsandboxed fallback" 标记 (current) |
| 当前模式为 closed | "Strict sandbox mode" 标记 (current) |
| 选择取消 | 调用 onComplete(undefined, { display: 'skip' }) |

### 6.3 改进建议

1. **模式说明增强**: 
   - 添加具体示例说明两种模式的区别
   - 显示当前会话中每种模式的使用统计

2. **企业策略提示**:
   - 显示具体哪个策略源锁定了设置
   - 提供联系管理员的指引

3. **快速切换**:
   - 添加键盘快捷键快速切换模式
   - 支持命令行参数直接设置

4. **模式验证**:
   - 切换到 strict 模式前验证 excludedCommands 是否配置正确
   - 警告用户可能导致常用命令失败

5. **审计日志**:
   - 记录模式切换操作
   - 显示最近切换历史

### 6.4 测试要点

- 沙箱启用/禁用状态的展示
- 策略锁定状态的检测和展示
- 模式切换的成功/失败处理
- 当前模式指示器的正确显示
- 取消操作的处理
- 主题颜色变化时的更新

### 6.5 可访问性考虑

- 使用 `useTabHeaderFocus` 确保键盘导航正常
- Select 组件支持 `onUpFromFirstItem` 返回 Tab 头部
- 颜色不仅传达信息（current 指示器有文本标记）

---

## 附录：相关文件索引

| 文件 | 描述 |
|------|------|
| `src/components/sandbox/SandboxSettings.tsx` | 沙箱设置主界面 |
| `src/components/sandbox/SandboxConfigTab.tsx` | 配置展示标签页 |
| `src/utils/sandbox/sandbox-adapter.ts` | 沙箱管理器 |
| `src/utils/settings/settings.ts` | 设置读写实现 |
| `src/utils/settings/constants.ts` | 设置源定义 |
| `src/components/CustomSelect/select.tsx` | 选择器组件 |
| `src/components/design-system/Tabs.tsx` | Tabs 组件 |
