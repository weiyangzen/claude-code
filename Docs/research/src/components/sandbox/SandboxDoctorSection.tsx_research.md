# SandboxDoctorSection.tsx 深度研究文档

## 1. 场景与职责

**SandboxDoctorSection** 是 Claude Code CLI 的 `/doctor` 命令中的一个诊断组件，负责在 Doctor 界面中展示沙箱系统的健康状态。

### 核心职责

1. **快速状态概览**: 在 Doctor 界面中提供沙箱系统的整体健康状态
2. **问题聚合展示**: 集中展示依赖错误和配置警告
3. **用户引导**: 当存在问题时，引导用户运行 `/sandbox` 获取详细帮助

### 使用场景

- **用户执行 `/doctor` 命令**: 作为系统诊断报告的一部分
- **启动时健康检查**: 可能作为启动诊断的一部分调用
- **问题排查**: 用户遇到沙箱相关问题时快速查看状态

### 与 SandboxSettings 的区别

| 特性 | SandboxDoctorSection | SandboxSettings |
|------|---------------------|-----------------|
| 位置 | `/doctor` 命令输出 | `/sandbox` 命令界面 |
| 详细程度 | 概览，仅显示问题 | 完整配置展示 |
| 交互性 | 只读展示 | 可交互修改设置 |
| 标签页 | 无（独立段落） | 多标签页（Mode/Overrides/Config/Dependencies） |

---

## 2. 功能点目的

### 2.1 状态判断逻辑

组件通过三层检查决定是否展示：

```typescript
// 1. 平台支持检查
if (!SandboxManager.isSupportedPlatform()) {
  return null  // 不展示沙箱段落
}

// 2. 设置启用检查
if (!SandboxManager.isSandboxEnabledInSettings()) {
  return null  // 用户未启用沙箱
}

// 3. 问题存在检查
const depCheck = SandboxManager.checkDependencies()
const hasErrors = depCheck.errors.length > 0
const hasWarnings = depCheck.warnings.length > 0

if (!hasErrors && !hasWarnings) {
  return null  // 一切正常，不展示
}
```

### 2.2 状态展示

| 状态 | 颜色 | 文本 |
|------|------|------|
| 有错误 | error (红色) | "Missing dependencies" |
| 仅警告 | warning (黄色) | "Available (with warnings)" |

### 2.3 问题列表展示

- **错误**: 每条错误以 `└` 前缀展示，使用 `error` 颜色
- **警告**: 每条警告以 `└` 前缀展示，使用 `warning` 颜色
- **引导**: 错误存在时显示 "└ Run /sandbox for install instructions"

---

## 3. 具体技术实现

### 3.1 组件架构

```tsx
export function SandboxDoctorSection(): React.ReactNode {
  // 平台支持检查
  if (!SandboxManager.isSupportedPlatform()) {
    return null
  }
  
  // 设置启用检查
  if (!SandboxManager.isSandboxEnabledInSettings()) {
    return null
  }
  
  // React Compiler 记忆化块
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    const depCheck = SandboxManager.checkDependencies()
    const hasErrors = depCheck.errors.length > 0
    const hasWarnings = depCheck.warnings.length > 0
    
    // 无问题则返回 null
    if (!hasErrors && !hasWarnings) {
      $[1] = Symbol.for("react.early_return_sentinel")
      break bb0
    }
    
    // 确定状态样式
    const statusColor = hasErrors ? "error" : "warning"
    const statusText = hasErrors 
      ? "Missing dependencies" 
      : "Available (with warnings)"
    
    // 渲染诊断段落
    $[0] = <Box flexDirection="column">
      <Text bold>Sandbox</Text>
      <Text>└ Status: <Text color={statusColor}>{statusText}</Text></Text>
      {depCheck.errors.map((e, i) => <Text key={i} color="error">└ {e}</Text>)}
      {depCheck.warnings.map((w, i) => <Text key={i} color="warning">└ {w}</Text>)}
      {hasErrors && <Text dimColor>└ Run /sandbox for install instructions</Text>}
    </Box>
  }
  
  return $[0]
}
```

### 3.2 React Compiler 优化

使用 2 个记忆化槽位：

```typescript
const $ = _c(2)

// $[0]: 缓存渲染结果
// $[1]: 早期返回标记（用于无问题时的 null 返回）
```

使用 `Symbol.for("react.early_return_sentinel")` 标记早期返回，避免重复计算。

### 3.3 条件渲染策略

组件采用"静默成功"策略：
- **正常状态**: 完全不渲染（返回 null），保持 Doctor 输出简洁
- **异常状态**: 详细展示问题，引导用户解决

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入 | 路径 | 用途 |
|------|------|------|
| React | 'react' | JSX 运行时 |
| Box, Text | '../../ink.js' | Ink 终端 UI 组件 |
| SandboxManager | '../../utils/sandbox/sandbox-adapter.js' | 沙箱管理器 |

### 4.2 调用链

```
SandboxDoctorSection.tsx
  ↓ 调用
SandboxManager.isSupportedPlatform()
  ↓ 委托
BaseSandboxManager.isSupportedPlatform() (@anthropic-ai/sandbox-runtime)
  ↓ 系统检测
检查 process.platform (darwin/linux/win32)
检查 /proc/version (WSL 检测)

SandboxManager.isSandboxEnabledInSettings()
  ↓ 读取
settings.sandbox.enabled (来自 settings.json)

SandboxManager.checkDependencies()
  ↓ 委托
BaseSandboxManager.checkDependencies() (@anthropic-ai/sandbox-runtime)
  ↓ 系统调用
检查 rg, bwrap, socat, seccomp 等依赖
```

### 4.3 在 Doctor 中的集成

组件被导入到 Doctor 命令的实现中：

```typescript
// 伪代码示例
import { SandboxDoctorSection } from './SandboxDoctorSection'

function DoctorCommand() {
  return (
    <Box flexDirection="column">
      <GitDoctorSection />
      <SandboxDoctorSection />  {/* 沙箱诊断 */}
      <PermissionsDoctorSection />
      {/* ... 其他诊断段落 */}
    </Box>
  )
}
```

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 |
|------|------|
| @anthropic-ai/sandbox-runtime | 底层沙箱运行时 |
| react | React 框架 |
| ink | 终端渲染框架 |

### 5.2 内部模块依赖

```
SandboxDoctorSection.tsx
├── src/ink.ts                    # Ink 渲染层
├── src/utils/sandbox/sandbox-adapter.ts
│   ├── 平台检测 (src/utils/platform.ts)
│   ├── 设置读取 (src/utils/settings/settings.ts)
│   └── 依赖检测 (@anthropic-ai/sandbox-runtime)
└── src/components/design-system/  # 设计系统（间接）
```

### 5.3 设置系统集成

```typescript
// isSandboxEnabledInSettings() 实现
function getSandboxEnabledSetting(): boolean {
  try {
    const settings = getSettings_DEPRECATED()
    return settings?.sandbox?.enabled ?? false
  } catch (error) {
    logForDebugging(`Failed to get settings for sandbox check: ${error}`)
    return false
  }
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 静默失败 | 设置读取失败时返回 false，可能误导用户 | 记录调试日志 |
| 平台误判 | 新型平台或容器环境可能识别错误 | 保守返回 null |
| 缓存过期 | 依赖状态可能在上次检测后改变 | 每次渲染重新检测 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| Windows 平台 | 返回 null，不展示沙箱段落 |
| 沙箱未启用 | 返回 null，不展示 |
| 依赖检测出错 | 通过 try-catch 捕获，返回 null |
| 仅 warnings 无 errors | 展示警告状态，不显示安装指引 |
| errors 和 warnings 同时存在 | 优先显示错误状态（红色） |

### 6.3 改进建议

1. **成功状态展示**: 可选配置显示 "Sandbox: ✓ Ready" 增强用户信心
2. **快速修复**: 在 Doctor 界面直接提供修复选项（如 "Install missing dependencies"）
3. **详细模式**: 添加 verbose 模式展示完整配置，不仅限于问题
4. **历史状态**: 记录上次检测时间，提示状态可能已过期
5. **一键诊断**: 集成 `/sandbox` 命令的快速入口

### 6.4 测试要点

- 各平台（macOS/Linux/WSL/Windows）的正确处理
- 沙箱启用/禁用状态
- 依赖错误/警告/正常三种状态
- 错误和警告同时存在时的优先级
- 空错误/警告数组的处理
- 设置读取异常的处理

### 6.5 性能考虑

- 每次渲染都调用 `checkDependencies()` 可能较耗时
- 建议考虑：
  - 添加缓存机制（如 5 秒内不重复检测）
  - 延迟检测（使用 useEffect）
  - 后台检测，前端展示缓存结果

---

## 附录：相关文件索引

| 文件 | 描述 |
|------|------|
| `src/components/sandbox/SandboxSettings.tsx` | 完整沙箱设置界面 |
| `src/components/sandbox/SandboxDependenciesTab.tsx` | 详细依赖展示 |
| `src/utils/sandbox/sandbox-adapter.ts` | 沙箱管理器实现 |
| `src/utils/platform.ts` | 平台检测 |
| `src/utils/doctorDiagnostic.ts` | Doctor 诊断框架 |
