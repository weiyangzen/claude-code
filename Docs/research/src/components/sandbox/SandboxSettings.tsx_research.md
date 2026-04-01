# SandboxSettings.tsx 深度研究文档

## 1. 场景与职责

**SandboxSettings** 是 Claude Code CLI 的 `/sandbox` 命令的主界面组件，是沙箱配置的中心化管理界面。

### 核心职责

1. **沙箱模式配置**: 启用/禁用沙箱，配置自动允许模式
2. **标签页管理**: 整合 Mode、Overrides、Config、Dependencies 四个标签页
3. **依赖状态感知**: 根据依赖检查结果动态调整界面
4. **键盘交互**: 提供完整的键盘导航支持

### 使用场景

- **首次配置**: 用户首次设置沙箱时选择工作模式
- **模式切换**: 在 auto-allow、regular、disabled 之间切换
- **故障排查**: 通过 Dependencies 标签页检查依赖状态
- **高级配置**: 通过 Overrides 和 Config 进行精细调整

### 界面结构

```
┌─────────────────────────────────────┐
│ Sandbox: [Mode] [Overrides] [Config]│  ← Tabs 标题行
├─────────────────────────────────────┤
│                                     │
│  Mode 标签页内容                     │  ← 默认显示
│  - 模式选择器                        │
│  - 模式说明                          │
│                                     │
└─────────────────────────────────────┘
```

当存在依赖错误或警告时，显示 Dependencies 标签页：

```
┌─────────────────────────────────────┐
│ Sandbox: [Mode] [Overrides]         │
│          [Config] [Dependencies] ⚠️ │  ← 警告标记
├─────────────────────────────────────┤
│                                     │
│  Dependencies 标签页内容             │  ← 优先显示
│  - 依赖检查状态                      │
│  - 安装指导                          │
│                                     │
└─────────────────────────────────────┘
```

---

## 2. 功能点目的

### 2.1 沙箱模式

| 模式 | 配置 | 行为 |
|------|------|------|
| Auto-allow | `enabled: true, autoAllowBashIfSandboxed: true` | 命令优先在沙箱执行，失败时自动允许非沙箱执行 |
| Regular | `enabled: true, autoAllowBashIfSandboxed: false` | 命令在沙箱执行，失败时询问用户 |
| Disabled | `enabled: false` | 不使用沙箱 |

### 2.2 标签页动态显示

```typescript
// 有错误时：Dependencies 标签页放在首位
hasErrors 
  ? [DependenciesTab]
  : [ModeTab, ...(hasWarnings ? [DependenciesTab] : []), OverridesTab, ConfigTab]
```

### 2.3 Socket 警告

当依赖检查有警告且未配置 `allowAllUnixSockets` 时显示：

```typescript
const showSocketWarning = hasWarnings && !allowAllUnixSockets
```

---

## 3. 具体技术实现

### 3.1 组件架构

```tsx
export function SandboxSettings({ onComplete, depCheck }: Props): React.ReactNode {
  const [theme] = useTheme()
  const currentEnabled = SandboxManager.isSandboxingEnabled()
  const currentAutoAllow = SandboxManager.isAutoAllowBashIfSandboxedEnabled()
  const hasWarnings = depCheck.warnings.length > 0
  const hasErrors = depCheck.errors.length > 0
  
  // 获取设置检查 socket 警告
  const settings = getSettings_DEPRECATED()
  const allowAllUnixSockets = settings.sandbox?.network?.allowAllUnixSockets
  const showSocketWarning = hasWarnings && !allowAllUnixSockets
  
  // 确定当前模式
  const getCurrentMode = (): SandboxMode => {
    if (!currentEnabled) return 'disabled'
    if (currentAutoAllow) return 'auto-allow'
    return 'regular'
  }
  
  // 构建选项
  const options = [
    { label: 'Sandbox BashTool, with auto-allow (current)', value: 'auto-allow' },
    { label: 'Sandbox BashTool, with regular permissions', value: 'regular' },
    { label: 'No Sandbox', value: 'disabled' }
  ]
  
  // 处理模式选择
  const handleSelect = async (value: SandboxMode) => {
    switch (value) {
      case 'auto-allow':
        await SandboxManager.setSandboxSettings({ 
          enabled: true, 
          autoAllowBashIfSandboxed: true 
        })
        break
      case 'regular':
        await SandboxManager.setSandboxSettings({ 
          enabled: true, 
          autoAllowBashIfSandboxed: false 
        })
        break
      case 'disabled':
        await SandboxManager.setSandboxSettings({ 
          enabled: false, 
          autoAllowBashIfSandboxed: false 
        })
        break
    }
    onComplete(成功消息)
  }
  
  // 构建标签页
  const modeTab = <Tab title="Mode"><SandboxModeTab ... /></Tab>
  const overridesTab = <Tab title="Overrides"><SandboxOverridesTab ... /></Tab>
  const configTab = <Tab title="Config"><SandboxConfigTab /></Tab>
  
  const tabs = hasErrors 
    ? [<Tab title="Dependencies"><SandboxDependenciesTab ... /></Tab>]
    : [modeTab, ...(hasWarnings ? [dependenciesTab] : []), overridesTab, configTab]
  
  return (
    <Pane color="permission">
      <Tabs title="Sandbox:" color="permission" defaultTab="Mode">
        {tabs}
      </Tabs>
    </Pane>
  )
}
```

### 3.2 SandboxModeTab 子组件

```tsx
function SandboxModeTab({ showSocketWarning, options, onSelect, onComplete }: ModeTabProps) {
  const { headerFocused, focusHeader } = useTabHeaderFocus()
  
  return (
    <Box flexDirection="column" paddingY={1}>
      {showSocketWarning && (
        <Text color="warning">
          Cannot block unix domain sockets (see Dependencies tab)
        </Text>
      )}
      <Text bold>Configure Mode:</Text>
      <Select 
        options={options} 
        onChange={onSelect}
        onCancel={() => onComplete(undefined, { display: 'skip' })}
        onUpFromFirstItem={focusHeader}
        isDisabled={headerFocused}
      />
      <说明文本>
        <Text dimColor>
          <Text bold>Auto-allow mode:</Text> Commands will try to run in the 
          sandbox automatically, and attempts to run outside of the sandbox 
          fallback to regular permissions...
        </Text>
      </说明文本>
    </Box>
  )
}
```

### 3.3 React Compiler 记忆化

**主组件**（34 个槽位）：
- `$[0]`: settings 缓存
- `$[1-2]`: currentIndicator 主题颜色
- `$[3-12]`: 三个模式选项缓存
- `$[13-14]`: handleSelect 处理器
- `$[15-16]`: 键盘绑定配置
- `$[17]`: keybindings context
- `$[18-22]`: modeTab 缓存
- `$[23-24]`: overridesTab 缓存
- `$[25]`: configTab 缓存
- `$[26-31]`: tabs 数组缓存
- `$[32-33]`: 最终渲染结果

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入 | 路径 | 用途 |
|------|------|------|
| React | 'react' | JSX 运行时 |
| Box, color, Link, Text, useTheme | '../../ink.js' | Ink UI 组件 |
| useKeybindings | '../../keybindings/useKeybinding.js' | 键盘绑定 |
| CommandResultDisplay | '../../types/command.js' | 类型定义 |
| SandboxDependencyCheck | '../../utils/sandbox/sandbox-adapter.js' | 类型定义 |
| SandboxManager | '../../utils/sandbox/sandbox-adapter.js' | 沙箱管理器 |
| getSettings_DEPRECATED | '../../utils/settings/settings.js' | 设置读取 |
| Select | '../CustomSelect/select.js' | 选择器组件 |
| Pane | '../design-system/Pane.js' | 面板容器 |
| Tab, Tabs, useTabHeaderFocus | '../design-system/Tabs.js' | Tabs 组件 |
| SandboxConfigTab | './SandboxConfigTab.js' | Config 标签页 |
| SandboxDependenciesTab | './SandboxDependenciesTab.js' | Dependencies 标签页 |
| SandboxOverridesTab | './SandboxOverridesTab.js' | Overrides 标签页 |

### 4.2 设置更新流程

```
用户选择模式
  ↓
handleSelect(mode)
  ↓
SandboxManager.setSandboxSettings({ enabled, autoAllowBashIfSandboxed })
  ↓
updateSettingsForSource('localSettings', { sandbox: {...} })
  ↓
写入 .claude/settings.local.json
  ↓
resetSettingsCache()  // 清除缓存
  ↓
触发 settingsChangeDetector
  ↓
SandboxManager.refreshConfig()
  ↓
BaseSandboxManager.updateConfig(newConfig)  // 更新运行时
```

### 4.3 标签页渲染条件

```typescript
// 有错误：只显示 Dependencies 标签页
if (hasErrors) {
  return [<Tab key="dependencies" title="Dependencies">...</Tab>]
}

// 正常或警告：显示所有相关标签页
return [
  modeTab,
  ...(hasWarnings ? [dependenciesTab] : []),
  overridesTab,
  configTab
]
```

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 |
|------|------|
| react | React 框架 |
| ink | 终端渲染框架 |
| @anthropic-ai/sandbox-runtime | 沙箱运行时 |

### 5.2 内部模块依赖

```
SandboxSettings.tsx
├── src/ink.ts                              # Ink 渲染层
├── src/keybindings/useKeybinding.ts        # 键盘绑定
├── src/types/command.ts                    # 命令类型
├── src/utils/sandbox/sandbox-adapter.ts    # 沙箱管理器
│   ├── src/utils/settings/settings.ts      # 设置管理
│   ├── src/utils/settings/changeDetector.ts # 变更检测
│   └── @anthropic-ai/sandbox-runtime       # 外部运行时
├── src/components/CustomSelect/select.tsx  # 选择器
├── src/components/design-system/Pane.tsx   # 面板
├── src/components/design-system/Tabs.tsx   # Tabs
├── SandboxConfigTab.tsx                    # Config 标签页
├── SandboxDependenciesTab.tsx              # Dependencies 标签页
└── SandboxOverridesTab.tsx                 # Overrides 标签页
```

### 5.3 设置系统集成

```typescript
// 读取设置
const settings = getSettings_DEPRECATED()

// 关键设置项
interface SandboxSettings {
  enabled?: boolean
  autoAllowBashIfSandboxed?: boolean
  allowUnsandboxedCommands?: boolean
  failIfUnavailable?: boolean
  network?: {
    allowedDomains?: string[]
    allowAllUnixSockets?: boolean
    allowUnixSockets?: string[]
    ...
  }
  filesystem?: {
    allowWrite?: string[]
    denyWrite?: string[]
    denyRead?: string[]
    allowRead?: string[]
  }
  excludedCommands?: string[]
  ...
}
```

---

## 6. 风险、边界与改进建议

### 6.1 安全风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 模式降级攻击 | 恶意项目通过 settings.json 禁用沙箱 | projectSettings 不控制沙箱启用状态 |
| 配置竞争 | 快速切换模式可能导致配置不一致 | 使用 Promise 串行处理 |
| 敏感信息泄露 | Config 标签页展示路径信息 | 仅展示当前用户可见的配置 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 依赖错误 | 强制显示 Dependencies 标签页，阻止其他操作 |
| 依赖警告 | 显示 Dependencies 标签页，但允许访问其他标签页 |
| 平台不支持 | SandboxManager.isSandboxingEnabled() 返回 false，Disabled 模式 |
| 设置读取失败 | 使用默认值，记录调试日志 |
| 设置写入失败 | onComplete 不调用，错误由设置系统处理 |
| 取消操作 | 调用 onComplete(undefined, { display: 'skip' }) |

### 6.3 改进建议

1. **向导模式**: 首次使用沙箱时提供交互式配置向导
2. **配置验证**: 保存前验证配置组合的有效性
3. **预览模式**: 显示配置变更的预览效果
4. **导入/导出**: 支持配置的导入导出
5. **配置模板**: 提供常见场景的预设配置（如"开发环境"、"CI环境"）
6. **变更历史**: 显示最近的配置变更记录

### 6.4 测试要点

- 三种模式的正确切换
- 依赖错误/警告/正常状态的标签页显示
- 键盘导航（Tab、方向键）
- 取消操作的处理
- 设置写入的成功/失败
- 主题颜色变化
- 多平台行为一致性

### 6.5 性能考虑

- 每次渲染读取 settings 可能较频繁
- 建议：
  - 使用 React Context 缓存设置
  - 监听 settingsChangeDetector 更新缓存
  - 避免在渲染路径中同步读取文件

### 6.6 可访问性

- 完整的键盘导航支持
- Tabs 组件的 `useTabHeaderFocus` 集成
- Select 组件的 `onUpFromFirstItem` 支持
- 颜色不仅用于传达信息（有文本标签）

---

## 附录：相关文件索引

| 文件 | 描述 |
|------|------|
| `src/components/sandbox/SandboxConfigTab.tsx` | 配置展示标签页 |
| `src/components/sandbox/SandboxDependenciesTab.tsx` | 依赖检查标签页 |
| `src/components/sandbox/SandboxOverridesTab.tsx` | 覆盖设置标签页 |
| `src/components/sandbox/SandboxDoctorSection.tsx` | Doctor 诊断组件 |
| `src/utils/sandbox/sandbox-adapter.ts` | 沙箱管理器核心 |
| `src/entrypoints/sandboxTypes.ts` | 沙箱类型定义 |
| `src/utils/settings/settings.ts` | 设置管理系统 |
| `src/utils/settings/types.ts` | 设置类型定义 |
