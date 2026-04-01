# 研究文档: src/services/remoteManagedSettings/securityCheck.tsx

## 场景与职责

本文件实现远程管理设置的安全检查功能，用于检测和提示用户关于"危险设置"的变更。当企业管理员通过远程管理设置推送可能执行任意代码或拦截提示词/响应的配置时，系统会向用户显示安全确认对话框，要求用户明确同意后才能应用这些设置。

**核心职责**：
1. 检测新远程设置中的危险配置（shell 命令、环境变量、hooks）
2. 比较新旧设置，识别危险设置的变更或新增
3. 在交互模式下显示阻塞式安全确认对话框
4. 记录用户接受/拒绝的遥测事件
5. 用户拒绝时执行优雅退出

## 功能点目的

### 1. 危险设置检测 (`checkManagedSettingsSecurity`)
- 提取新设置中的危险配置（通过 `extractDangerousSettings`）
- 检查危险设置是否发生变化（通过 `hasDangerousSettingsChanged`）
- 非交互模式下自动跳过对话框（与信任对话框行为一致）
- 返回三种状态：
  - `'approved'`: 用户接受
  - `'rejected'`: 用户拒绝
  - `'no_check_needed'`: 无需检查（无危险设置或危险设置未变更）

### 2. 安全对话框渲染
- 使用 React + Ink 渲染阻塞式对话框
- 包装在 `AppStateProvider` 和 `KeybindingSetup` 中提供上下文
- 使用 `ManagedSettingsSecurityDialog` 组件显示具体设置信息
- 记录遥测事件：`tengu_managed_settings_security_dialog_shown/accepted/rejected`

### 3. 结果处理 (`handleSecurityCheckResult`)
- 接受结果并决定是否继续
- 用户拒绝时调用 `gracefulShutdownSync(1)` 优雅退出
- 返回布尔值指示调用者是否应继续应用设置

## 具体技术实现

### 关键流程

#### 安全检查流程 (`checkManagedSettingsSecurity`)
```
1. 提取新设置中的危险设置
2. 检查是否存在危险设置
   - 不存在 → 返回 'no_check_needed'
3. 检查危险设置是否发生变化
   - 未变化 → 返回 'no_check_needed'
4. 检查是否为交互模式
   - 非交互 → 返回 'no_check_needed'
5. 记录遥测事件：dialog_shown
6. 渲染阻塞式安全对话框
7. 等待用户选择
8. 记录遥测事件：accepted/rejected
9. 返回结果
```

#### 对话框渲染结构
```tsx
<AppStateProvider>
  <KeybindingSetup>
    <ManagedSettingsSecurityDialog
      settings={newSettings}
      onAccept={() => { logEvent('accepted'); unmount(); resolve('approved'); }}
      onReject={() => { logEvent('rejected'); unmount(); resolve('rejected'); }}
    />
  </KeybindingSetup>
</AppStateProvider>
```

### 数据结构

#### SecurityCheckResult
```typescript
export type SecurityCheckResult = 'approved' | 'rejected' | 'no_check_needed';
```

### 危险设置定义

危险设置分为三类（定义在 `src/components/ManagedSettingsSecurityDialog/utils.ts`）：

1. **Shell 设置** (`DANGEROUS_SHELL_SETTINGS`)
   - `apiKeyHelper`: 输出认证值的脚本路径
   - `awsAuthRefresh`: 刷新 AWS 认证的命令
   - `awsCredentialExport`: 导出 AWS 凭证的脚本路径
   - `gcpAuthRefresh`: 刷新 GCP 认证的命令
   - `otelHeadersHelper`: 输出 OpenTelemetry Headers 的脚本路径
   - `statusLine`: 自定义状态栏命令

2. **环境变量** (`env`)
   - 任何不在 `SAFE_ENV_VARS` 白名单中的环境变量
   - 包括：代理设置、Base URL、认证令牌等

3. **Hooks**
   - 任何非空的 `hooks` 配置对象
   - 可在工具执行前后运行任意命令

## 关键代码路径与文件引用

### 核心函数

| 函数 | 行号 | 描述 |
|------|------|------|
| `checkManagedSettingsSecurity` | 22-61 | 主安全检查函数 |
| `handleSecurityCheckResult` | 67-73 | 处理检查结果 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `../../components/ManagedSettingsSecurityDialog/ManagedSettingsSecurityDialog.js` | 安全对话框 UI 组件 |
| `../../components/ManagedSettingsSecurityDialog/utils.js` | 危险设置提取和比较工具 |
| `../../bootstrap/state.js` | `getIsInteractive()` 检查 |
| `../../ink.js` | `render` 函数 |
| `../../state/AppState.js` | `AppStateProvider` |
| `../../keybindings/KeybindingProviderSetup.js` | `KeybindingSetup` |
| `../../utils/gracefulShutdown.js` | `gracefulShutdownSync` |
| `../../utils/renderOptions.js` | `getBaseRenderOptions` |
| `../analytics/index.js` | `logEvent` 遥测 |

### 依赖组件

#### ManagedSettingsSecurityDialog 组件
- 位置：`src/components/ManagedSettingsSecurityDialog/ManagedSettingsSecurityDialog.tsx`
- Props:
  - `settings: SettingsJson` - 要显示的新设置
  - `onAccept: () => void` - 用户接受回调
  - `onReject: () => void` - 用户拒绝回调
- 功能：显示危险设置列表，提供接受/拒绝选项

#### utils.ts 工具函数
- `extractDangerousSettings(settings)` - 从设置中提取危险配置
- `hasDangerousSettings(dangerous)` - 检查是否存在危险设置
- `hasDangerousSettingsChanged(old, new)` - 比较新旧设置的危险部分
- `formatDangerousSettingsList(dangerous)` - 格式化危险设置为可读列表

## 依赖与外部交互

### 被调用方

| 文件 | 调用函数 | 场景 |
|------|----------|------|
| `src/services/remoteManagedSettings/index.ts` | `checkManagedSettingsSecurity`, `handleSecurityCheckResult` | 获取新设置后执行安全检查 |

### 调用时序
```
index.ts:fetchAndLoadRemoteManagedSettings
  → 获取新设置
  → checkManagedSettingsSecurity(cachedSettings, newSettings)
    → 渲染 ManagedSettingsSecurityDialog
    → 返回 'approved' | 'rejected' | 'no_check_needed'
  → handleSecurityCheckResult(result)
    → 如果 rejected → gracefulShutdownSync(1)
    → 返回 boolean 指示是否继续
```

### 遥测事件

| 事件名 | 触发条件 |
|--------|----------|
| `tengu_managed_settings_security_dialog_shown` | 对话框显示时 |
| `tengu_managed_settings_security_dialog_accepted` | 用户接受时 |
| `tengu_managed_settings_security_dialog_rejected` | 用户拒绝时 |

## 风险、边界与改进建议

### 风险点

1. **非交互模式绕过**
   - 在 CI/CD 或 Agent SDK 场景中，非交互模式自动跳过安全对话框
   - 风险：恶意设置可能在无人知晓的情况下被应用
   - 缓解：企业应通过其他方式（如审计日志）监控设置应用

2. **对话框阻塞**
   - 安全对话框是阻塞式的，会暂停 CLI 初始化
   - 风险：自动化脚本可能因等待用户输入而挂起
   - 缓解：非交互模式自动跳过，30 秒加载 Promise 超时

3. **依赖 React/ Ink**
   - 需要渲染 React 组件，依赖完整的 UI 环境
   - 风险：在无头环境或测试环境中可能失败
   - 缓解：非交互模式检查在渲染前返回

4. **安全设置定义漂移**
   - 危险设置列表分散在多个文件（`managedEnvConstants.ts`, `utils.ts`）
   - 风险：新增危险设置类型时可能遗漏更新
   - 缓解：集中定义 `DANGEROUS_SHELL_SETTINGS` 和 `SAFE_ENV_VARS`

### 边界条件

1. **首次使用场景**
   - `cachedSettings` 为 `null` 时，任何危险设置都视为"新增"
   - 用户首次使用远程管理设置时会看到对话框

2. **设置移除场景**
   - 如果新设置移除了危险设置，不会触发对话框
   - 仅检测危险设置的"存在"和"变更"，不检测"移除"

3. **相同危险设置**
   - 如果危险设置内容完全相同，仅值变化，也会触发对话框
   - 比较基于 JSON 序列化后的字符串相等性

4. **并发场景**
   - 安全对话框使用 Promise 包装，确保单实例
   - 多次设置变更会排队处理

### 改进建议

1. **增强审计**
   - 记录危险设置的详细变更日志（不仅仅是事件）
   - 添加设置应用历史记录，供管理员审计

2. **细化控制**
   - 允许企业管理员配置"强制设置"，跳过用户确认
   - 添加设置分类（高/中/低风险），不同级别不同处理方式

3. **改进 UX**
   - 显示设置变更的详细差异（diff 视图）
   - 提供"查看设置来源"功能，显示是哪个管理员推送的

4. **安全增强**
   - 对危险设置进行数字签名验证
   - 添加设置哈希链，检测篡改

5. **测试覆盖**
   - 添加单元测试模拟用户接受/拒绝场景
   - 测试非交互模式下的自动跳过逻辑
   - 测试危险设置检测的边界条件
