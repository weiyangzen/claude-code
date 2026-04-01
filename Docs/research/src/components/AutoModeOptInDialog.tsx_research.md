# AutoModeOptInDialog.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`AutoModeOptInDialog` 是一个**功能启用确认对话框**，用于向用户介绍并获取 Auto Mode（自动模式）的启用同意。Auto Mode 允许 Claude 自动处理权限提示，无需用户逐一手动确认。

### 1.2 使用场景
- **首次使用引导**：用户首次启动或首次使用 Auto Mode 时显示
- **功能升级提示**：从旧版本升级后需要重新确认
- **启动时检查**：`declineExits` 模式用于启动门禁（拒绝则退出进程）

### 1.3 法律合规
```typescript
// NOTE: This copy is legally reviewed — do not modify without Legal team approval.
export const AUTO_MODE_DESCRIPTION = "Auto mode lets Claude handle permission prompts automatically..."
```
⚠️ **重要**：描述文本经过法务审核，修改需获得法务团队批准。

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| 功能介绍 | 向用户解释 Auto Mode 的工作原理和风险 |
| 选项提供 | 提供 "启用并设为默认"、"仅启用"、"拒绝" 三个选项 |
| 配置持久化 | 将用户选择保存到 settings.json |
| 分析追踪 | 记录用户选择用于产品分析 |
| 启动门禁 | `declineExits` 模式支持拒绝时退出进程 |

### 2.2 Props 接口

```typescript
type Props = {
  onAccept(): void;      // 接受回调（启用 Auto Mode）
  onDecline(): void;     // 拒绝回调
  declineExits?: boolean; // 拒绝是否退出进程（启动门禁模式）
}
```

### 2.3 用户选项

| 选项 | 值 | 行为 |
|------|-----|------|
| "Yes, and make it my default mode" | `accept-default` | 启用并设为默认模式 |
| "Yes, enable auto mode" | `accept` | 仅启用，不更改默认 |
| "No, exit" / "No, go back" | `decline` | 拒绝启用 |

---

## 3. 具体技术实现

### 3.1 Auto Mode 描述文本

```typescript
export const AUTO_MODE_DESCRIPTION = 
  "Auto mode lets Claude handle permission prompts automatically — " +
  "Claude checks each tool call for risky actions and prompt injection before executing. " +
  "Actions Claude identifies as safe are executed, while actions Claude identifies as risky " +
  "are blocked and Claude may try a different approach. Ideal for long-running tasks. " +
  "Sessions are slightly more expensive. Claude can make mistakes that allow harmful commands " +
  "to run, it's recommended to only use in isolated environments. Shift+Tab to change mode."
```

**关键信息**：
- 自动处理权限提示
- Claude 会检查风险操作和提示注入
- 安全操作自动执行，风险操作被阻止
- 适合长时间运行任务
- 会话成本略高
- 存在误执行风险，建议在隔离环境使用

### 3.2 用户选择处理

```typescript
function onChange(value: 'accept' | 'accept-default' | 'decline') {
  switch (value) {
    case 'accept':
      logEvent('tengu_auto_mode_opt_in_dialog_accept', {})
      updateSettingsForSource('userSettings', {
        skipAutoPermissionPrompt: true
      })
      onAccept()
      break
      
    case 'accept-default':
      logEvent('tengu_auto_mode_opt_in_dialog_accept_default', {})
      updateSettingsForSource('userSettings', {
        skipAutoPermissionPrompt: true,
        permissions: {
          defaultMode: 'auto'
        }
      })
      onAccept()
      break
      
    case 'decline':
      logEvent('tengu_auto_mode_opt_in_dialog_decline', {})
      onDecline()
  }
}
```

### 3.3 配置更新

**settings.json 结构**：
```json
{
  "skipAutoPermissionPrompt": true,
  "permissions": {
    "defaultMode": "auto"
  }
}
```

**updateSettingsForSource** (`src/utils/settings/settings.ts`):
- 支持多源配置：userSettings, projectSettings, localSettings, policySettings
- 使用 lodash mergeWith 进行深度合并
- 自动处理数组替换逻辑

### 3.4 分析事件

| 事件 | 触发条件 |
|------|----------|
| `tengu_auto_mode_opt_in_dialog_shown` | 对话框显示时（useEffect） |
| `tengu_auto_mode_opt_in_dialog_accept` | 选择 "Yes, enable auto mode" |
| `tengu_auto_mode_opt_in_dialog_accept_default` | 选择 "Yes, and make it my default mode" |
| `tengu_auto_mode_opt_in_dialog_decline` | 选择拒绝选项 |

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置
```
src/components/AutoModeOptInDialog.tsx
```

### 4.2 依赖图

```
AutoModeOptInDialog.tsx
├── react (React 核心)
├── src/services/analytics/index.js
│   └── logEvent
├── ../ink.js (Box, Link, Text)
├── ../utils/settings/settings.js
│   └── updateSettingsForSource
├── ./CustomSelect/index.js
│   └── Select
└── ./design-system/Dialog.js
    └── Dialog
```

### 4.3 配置存储路径

```
~/.claude/settings.json (userSettings)
└── skipAutoPermissionPrompt: boolean
└── permissions.defaultMode: "auto" | "default"
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| React | 'react' | 组件运行时 |
| logEvent | 'src/services/analytics/index.js' | 分析追踪 |
| Box, Link, Text | '../ink.js' | UI 组件 |
| updateSettingsForSource | '../utils/settings/settings.js' | 配置更新 |
| Select | './CustomSelect/index.js' | 选择器 |
| Dialog | './design-system/Dialog.js' | 对话框容器 |

### 5.2 数据流

```
对话框显示
    │
    ▼
logEvent('tengu_auto_mode_opt_in_dialog_shown')
    │
    ▼
用户选择
    │
    ├── accept-default ──┬── updateSettingsForSource
    │                    │    ├── skipAutoPermissionPrompt: true
    │                    │    └── permissions.defaultMode: 'auto'
    │                    ├── logEvent('...accept_default')
    │                    └── onAccept()
    │
    ├── accept ──────────┬── updateSettingsForSource
    │                    │    └── skipAutoPermissionPrompt: true
    │                    ├── logEvent('...accept')
    │                    └── onAccept()
    │
    └── decline ─────────┬── logEvent('...decline')
                         └── onDecline()
                                  │
                                  ▼
                         declineExits ? exit() : goBack()
```

### 5.3 权限模式关联

```
AutoModeOptInDialog
        │
        ▼
updateSettingsForSource
        │
        ▼
settings.json
        │
        ▼
getInitialSettings() (启动时读取)
        │
        ▼
PermissionMode 初始化
        │
        ▼
ToolPermissionContext (运行时权限检查)
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 严重程度 |
|------|------|----------|
| 法务合规 | 描述文本修改需法务审核 | 高 |
| 误启用风险 | 用户可能不理解风险就启用 | 中 |
| 配置冲突 | 多源配置可能产生冲突 | 低 |
| 分析丢失 | 网络问题导致分析事件丢失 | 低 |

### 6.2 边界情况

1. **重复显示**：用户已启用但仍可能看到对话框（需检查 `skipAutoPermissionPrompt`）
2. **配置验证失败**：settings.json 格式错误时的降级处理
3. **多实例竞争**：多个 Claude 进程同时修改配置
4. **取消操作**：用户按 Esc 触发 `onCancel`，调用 `onDecline`

### 6.3 改进建议

1. **法务合规自动化**：
   ```typescript
   // 添加文本哈希验证
   const LEGAL_APPROVED_HASH = 'sha256:abc123...'
   if (hash(AUTO_MODE_DESCRIPTION) !== LEGAL_APPROVED_HASH) {
     console.warn('AutoMode description has been modified')
   }
   ```

2. **用户体验优化**：
   - 添加 "了解更多" 链接到详细文档
   - 显示当前环境是否为隔离环境的提示
   - 添加 "稍后再说" 选项（临时跳过）

3. **安全性增强**：
   ```typescript
   // 添加企业策略覆盖
   if (policySettings.disableAutoModeOptIn) {
     return null // 不显示对话框，使用策略配置
   }
   ```

4. **可观察性**：
   ```typescript
   // 添加更多分析维度
   logEvent('tengu_auto_mode_opt_in_dialog_shown', {
     isFirstTime: !hasSeenDialogBefore,
     currentMode: getCurrentPermissionMode(),
     source: declineExits ? 'startup_gate' : 'settings'
   })
   ```

5. **国际化**：
   - 将描述文本提取到 i18n 文件
   - 支持多语言显示

6. **测试覆盖**：
   - 单元测试三种选择路径
   - 集成测试配置持久化
   - E2E 测试对话框交互

### 6.4 相关配置项

```typescript
// ~/.claude/settings.json
type UserSettings = {
  skipAutoPermissionPrompt?: boolean
  permissions?: {
    defaultMode?: 'auto' | 'default'
    allow?: string[]
    deny?: string[]
    ask?: string[]
  }
}
```

### 6.5 相关功能标志

- `TRANSCRIPT_CLASSIFIER`: 启用新的 Auto Mode 实现
- `tengu_auto_mode_opt_in_dialog_*`: 分析事件前缀
