# 文件研究文档: src/commands/sandbox-toggle/sandbox-toggle.tsx

## 场景与职责

该文件是 `/sandbox` 命令的核心实现文件，负责提供交互式沙盒配置界面。它是一个本地 JSX 命令，使用 React 和 Ink 渲染终端 UI，允许用户：

1. 查看和修改沙盒启用状态（禁用/启用/自动允许）
2. 配置命令排除规则（excludedCommands）
3. 查看依赖状态和网络/文件系统限制
4. 配置覆盖选项（是否允许非沙盒回退）

**核心职责：**
- 处理 `/sandbox` 命令的各种调用形式（无参数、子命令）
- 验证平台支持和依赖状态
- 提供交互式配置界面（通过 `SandboxSettings` 组件）
- 实现 `exclude` 子命令用于添加命令排除规则

## 功能点目的

### 1. 平台与依赖验证
在执行任何操作前，验证：
- 平台是否支持沙盒（macOS、Linux、WSL2，**不支持 WSL1**）
- 平台是否在 `enabledPlatforms` 列表中
- 沙盒设置是否被策略锁定

### 2. 交互式配置界面
当不带参数调用 `/sandbox` 时，显示多标签配置界面：
- **Mode 标签**: 选择沙盒模式（禁用/启用/自动允许）
- **Overrides 标签**: 配置是否允许非沙盒命令回退
- **Dependencies 标签**: 查看依赖状态（有警告/错误时显示）
- **Config 标签**: 查看当前沙盒配置详情

### 3. 命令排除功能
通过 `/sandbox exclude "command pattern"` 子命令，允许用户将特定命令模式添加到排除列表，这些命令将不会运行在沙盒中。

### 4. 错误处理与反馈
提供清晰的错误消息和成功反馈，包括：
- 平台不支持的错误提示
- 依赖缺失的警告
- 策略锁定的提示
- 排除规则添加成功的确认

## 具体技术实现

### 核心函数: `call()`
```typescript
export async function call(
  onDone: (result?: string) => void,
  _context: unknown,
  args?: string,
): Promise<React.ReactNode | null>
```

**执行流程：**
1. 获取当前设置和主题
2. 检查平台支持（WSL1 特殊处理）
3. 检查依赖状态
4. 检查平台启用列表
5. 检查策略锁定状态
6. 解析参数：
   - 无参数 → 显示交互式界面
   - `exclude` 子命令 → 添加排除规则
   - 未知子命令 → 返回错误

### 子命令处理: `exclude`
```typescript
if (subcommand === 'exclude') {
  const commandPattern = trimmedArgs.slice('exclude '.length).trim()
  // 去除引号
  const cleanPattern = commandPattern.replace(/^["']|["']$/g, '')
  // 添加到排除列表
  addToExcludedCommands(cleanPattern)
  // 返回成功消息
}
```

### 数据结构
```typescript
// 沙盒模式类型
type SandboxMode = 'auto-allow' | 'regular' | 'disabled'

// 覆盖模式类型
type OverrideMode = 'open' | 'closed'

// 组件 Props
interface Props {
  onComplete: (result?: string, options?: { display?: CommandResultDisplay }) => void
  depCheck: SandboxDependencyCheck
}
```

### 依赖检查结构
```typescript
interface SandboxDependencyCheck {
  errors: string[]    // 导致沙盒无法运行的错误
  warnings: string[]  // 不影响运行但功能受限的警告
}
```

## 关键代码路径与文件引用

### 当前文件路径
```
src/commands/sandbox-toggle/sandbox-toggle.tsx
```

### 直接依赖文件
| 文件路径 | 用途 |
|---------|------|
| `src/bootstrap/state.ts` | `getCwdState()` 获取当前工作目录 |
| `src/components/sandbox/SandboxSettings.tsx` | 主配置界面组件 |
| `src/ink.ts` | `color()` 主题颜色函数 |
| `src/utils/platform.ts` | `getPlatform()` 平台检测 |
| `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager`, `addToExcludedCommands()` |
| `src/utils/settings/settings.ts` | `getSettings_DEPRECATED()`, `getSettingsFilePathForSource()` |
| `src/utils/theme.ts` | `ThemeName` 类型 |

### 组件调用链
```
call() 入口
    ↓
无参数 → <SandboxSettings />
    ↓
    ├── <Tabs>
    │     ├── <SandboxModeTab />      (Mode 标签)
    │     ├── <SandboxOverridesTab /> (Overrides 标签)
    │     ├── <SandboxDependenciesTab /> (Dependencies 标签 - 条件渲染)
    │     └── <SandboxConfigTab />    (Config 标签)
    │
exclude 子命令 → addToExcludedCommands()
    ↓
updateSettingsForSource('localSettings', { sandbox: { excludedCommands: [...] }})
```

### 沙盒设置组件层次
```
SandboxSettings.tsx
    ├── SandboxModeTab (内联在 SandboxSettings.tsx)
    ├── SandboxOverridesTab.tsx
    ├── SandboxDependenciesTab.tsx
    └── SandboxConfigTab.tsx
```

## 依赖与外部交互

### 设置系统交互
通过 `SandboxManager.setSandboxSettings()` 更新设置：
```typescript
await SandboxManager.setSandboxSettings({
  enabled: boolean,
  autoAllowBashIfSandboxed: boolean,
  allowUnsandboxedCommands: boolean,
})
```

设置保存到 `localSettings` 源（`.claude/settings.local.json`）。

### 平台检测逻辑
```typescript
const platform = getPlatform()
if (!SandboxManager.isSupportedPlatform()) {
  // WSL1 特殊错误提示
  const errorMessage = platform === 'wsl' 
    ? 'Error: Sandboxing requires WSL2. WSL1 is not supported.'
    : 'Error: Sandboxing is currently only supported on macOS, Linux, and WSL2.'
}
```

### 策略锁定检查
```typescript
if (SandboxManager.areSandboxSettingsLockedByPolicy()) {
  // 检查 flagSettings 或 policySettings 是否设置了沙盒相关配置
  // 这些高优先级设置会覆盖 localSettings
}
```

### 依赖检查
```typescript
const depCheck = SandboxManager.checkDependencies()
// 返回 { errors: string[], warnings: string[] }
// errors: 缺少 ripgrep, bwrap, socat 等
// warnings: 缺少 seccomp 过滤器（仅影响 Unix socket 阻断）
```

## 风险、边界与改进建议

### 潜在风险

1. **设置竞态条件**
   - 问题：多个并发调用可能同时修改 `excludedCommands`
   - 缓解：设置系统使用文件锁和合并逻辑

2. **路径解析问题**
   - 代码使用 `relative(getCwdState(), localSettingsPath)` 显示相对路径
   - 如果工作目录在会话中改变，显示的路径可能不准确

3. **错误处理不完整**
   - `exclude` 子命令没有验证命令模式的有效性
   - 用户可以添加任意字符串，可能导致匹配问题

### 边界情况

1. **WSL1 检测**
   ```typescript
   // WSL1 用户会看到特定错误
   platform === 'wsl' && !isSupportedPlatform()
   // → "Sandboxing requires WSL2. WSL1 is not supported."
   ```

2. **空参数处理**
   ```typescript
   const trimmedArgs = args?.trim() || ''
   if (!trimmedArgs) { /* 显示交互界面 */ }
   ```

3. **引号处理**
   ```typescript
   // 支持带引号和不带引号的参数
   const cleanPattern = commandPattern.replace(/^["']|["']$/g, '')
   ```

4. **策略锁定场景**
   - 当 `flagSettings` 或 `policySettings` 设置了沙盒配置时
   - 本地修改无效，UI 显示锁定状态

### 改进建议

1. **输入验证**
   ```typescript
   // 建议：验证 exclude 模式的有效性
   if (!isValidCommandPattern(cleanPattern)) {
     onDone(color('error', themeName)('Error: Invalid command pattern'))
     return null
   }
   ```

2. **批量排除支持**
   ```typescript
   // 当前：一次只能添加一个
   // 建议：支持 /sandbox exclude "pattern1" "pattern2" ...
   ```

3. **排除规则预览**
   - 在添加排除规则前显示当前列表
   - 允许用户确认或取消

4. **更好的错误恢复**
   ```typescript
   // 当前：设置更新失败时返回通用错误
   // 建议：区分文件系统错误、权限错误、JSON 解析错误
   ```

5. **命令补全支持**
   - 为 `exclude` 子命令提供历史命令补全
   - 帮助用户快速选择要排除的命令模式

### 测试建议

1. **平台检测测试**
   - 在 macOS、Linux、WSL1、WSL2、Windows 上验证行为

2. **策略锁定测试**
   - 设置 `flagSettings.sandbox.enabled` 后验证本地修改被阻止

3. **并发修改测试**
   - 同时执行多个 `exclude` 命令，验证数据一致性

4. **边界输入测试**
   - 空字符串、特殊字符、超长模式等

### 相关配置项

| 配置项 | 说明 |
|--------|------|
| `sandbox.enabled` | 是否启用沙盒 |
| `sandbox.autoAllowBashIfSandboxed` | 是否自动允许沙盒内的 bash 命令 |
| `sandbox.allowUnsandboxedCommands` | 是否允许非沙盒命令回退 |
| `sandbox.excludedCommands` | 排除命令列表（不运行在沙盒中） |
| `sandbox.enabledPlatforms` | 限制沙盒仅在特定平台启用 |
| `sandbox.failIfUnavailable` | 沙盒不可用时是否报错退出 |
