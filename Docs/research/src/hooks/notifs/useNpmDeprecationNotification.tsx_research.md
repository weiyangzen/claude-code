# useNpmDeprecationNotification.tsx 深度研究

## 场景与职责

`useNpmDeprecationNotification` 是一个 React Hook，用于在用户使用 npm 安装的 Claude Code 时显示弃用警告通知。Claude Code 已从 npm 分发迁移到原生安装程序，此 Hook 负责提示用户进行迁移。

### 核心场景
1. **npm 安装检测**：检测用户当前是否通过 npm 使用 Claude Code
2. **迁移引导**：提示用户运行 `claude install` 切换到原生安装
3. **文档链接**：提供链接到官方文档获取更多安装选项
4. **开发环境排除**：在开发环境中不显示通知

## 功能点目的

### 1. 安装方式检测
- 检查是否在捆绑模式（bundled mode）下运行
- 检查是否设置了 `DISABLE_INSTALLATION_CHECKS` 环境变量
- 通过 `getCurrentInstallationType()` 获取当前安装类型
- 排除开发环境（`installationType === "development"`）

### 2. 弃用通知显示
- 显示明确的弃用消息
- 提供迁移命令（`claude install`）
- 链接到官方文档获取更多信息
- 使用较长的超时时间（15 秒）确保用户看到

### 3. 高优先级警告
- 使用 `priority: 'high'` 确保用户注意到
- 使用 `color: 'warning'` 提供视觉提示

### 4. 启动时一次性执行
- 使用 `useStartupNotification` 确保只在启动时执行
- 避免重复提醒

## 具体技术实现

### 关键数据结构

```typescript
// 安装类型
 type InstallationType = 
   | 'npm-global' 
   | 'npm-local' 
   | 'native' 
   | 'package-manager' 
   | 'development' 
   | 'unknown'

// 通知配置
interface Notification {
  key: string
  text: string
  color: 'warning' | 'error' | 'suggestion' | 'text'
  priority: 'low' | 'medium' | 'high' | 'immediate'
  timeoutMs?: number
}
```

### 核心流程

```
useStartupNotification 初始化
    ↓
检查是否在捆绑模式 (isInBundledMode)
    如果是，返回 null（不需要显示）
    ↓
检查是否禁用安装检查 (DISABLE_INSTALLATION_CHECKS)
    如果是，返回 null
    ↓
获取当前安装类型 (getCurrentInstallationType)
    ↓
如果是开发环境 (installationType === "development")
    返回 null
    ↓
返回通知对象
```

### 关键代码路径

```typescript
// 弃用消息常量
const NPM_DEPRECATION_MESSAGE = 
  'Claude Code has switched from npm to native installer. ' +
  'Run `claude install` or see ' +
  'https://docs.anthropic.com/en/docs/claude-code/getting-started ' +
  'for more options.'

// 主 Hook
export function useNpmDeprecationNotification() {
  useStartupNotification(async () => {
    // 检查捆绑模式
    if (isInBundledMode()) {
      return null
    }
    
    // 检查环境变量禁用
    if (isEnvTruthy(process.env.DISABLE_INSTALLATION_CHECKS)) {
      return null
    }
    
    // 获取安装类型
    const installationType = await getCurrentInstallationType()
    
    // 排除开发环境
    if (installationType === "development") {
      return null
    }
    
    // 返回通知
    return {
      timeoutMs: 15000, // 15 秒超时
      key: "npm-deprecation-warning",
      text: NPM_DEPRECATION_MESSAGE,
      color: "warning",
      priority: "high"
    }
  })
}
```

### 安装类型检测逻辑

`getCurrentInstallationType()` 的检测逻辑（在 `src/utils/doctorDiagnostic.ts` 中）：

```typescript
export async function getCurrentInstallationType(): Promise<InstallationType> {
  // 开发环境优先
  if (process.env.NODE_ENV === 'development') {
    return 'development'
  }
  
  const [invokedPath] = getNormalizedPaths()
  
  // 检查捆绑模式
  if (isInBundledMode()) {
    // 检查是否由包管理器安装
    if (detectHomebrew() || detectWinget() || /* ... */) {
      return 'package-manager'
    }
    return 'native'
  }
  
  // 检查本地 npm 安装
  if (isRunningFromLocalInstallation()) {
    return 'npm-local'
  }
  
  // 检查全局 npm 安装路径
  const npmGlobalPaths = [
    '/usr/local/lib/node_modules',
    '/usr/lib/node_modules',
    '/opt/homebrew/lib/node_modules',
    // ...
  ]
  if (npmGlobalPaths.some(path => invokedPath.includes(path))) {
    return 'npm-global'
  }
  
  // 检查 npm 前缀
  const npmConfigResult = await execa('npm config get prefix', { shell: true, reject: false })
  const globalPrefix = npmConfigResult.exitCode === 0 ? npmConfigResult.stdout.trim() : null
  if (globalPrefix && invokedPath.startsWith(globalPrefix)) {
    return 'npm-global'
  }
  
  return 'unknown'
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `isInBundledMode` | `src/utils/bundledMode.js` | 检测捆绑模式 |
| `getCurrentInstallationType` | `src/utils/doctorDiagnostic.js` | 获取当前安装类型 |
| `isEnvTruthy` | `src/utils/envUtils.js` | 环境变量检查 |
| `useStartupNotification` | `./useStartupNotification.js` | 启动通知基类 |

### 依赖模块详解

#### 1. isInBundledMode (src/utils/bundledMode.ts)
检测是否运行在 Bun 编译的独立可执行文件中：
```typescript
export function isInBundledMode(): boolean {
  return (
    typeof Bun !== 'undefined' &&
    Array.isArray(Bun.embeddedFiles) &&
    Bun.embeddedFiles.length > 0
  )
}
```

#### 2. getCurrentInstallationType (src/utils/doctorDiagnostic.ts)
检测 Claude Code 的安装方式：
- 开发环境检测
- 捆绑模式检测
- 包管理器检测（Homebrew、Winget 等）
- npm 全局/本地安装检测

#### 3. useStartupNotification (src/hooks/notifs/useStartupNotification.ts)
提供启动时一次性通知的基础设施：
- 远程模式检查
- 单次执行保证
- 异步计算支持
- 错误处理

## 风险、边界与改进建议

### 潜在风险

1. **检测误判**
   - `getCurrentInstallationType` 依赖路径匹配，可能在某些自定义安装场景下误判
   - 例如：用户手动复制 npm 安装的文件到非标准位置

2. **异步检测延迟**
   - `getCurrentInstallationType` 包含异步操作（`execa`）
   - 如果检测耗时较长，可能影响启动体验

3. **环境变量覆盖**
   - `DISABLE_INSTALLATION_CHECKS` 可以完全禁用此通知
   - 用户可能无意中设置此变量而错过重要提示

4. **消息长度**
   - 通知消息较长（包含 URL），在小屏幕终端可能被截断

### 边界情况

1. **捆绑模式**
   - 如果用户已通过 `claude install` 安装，运行在捆绑模式下
   - 不显示通知（预期行为）

2. **包管理器安装**
   - 如果通过 Homebrew、Winget 等包管理器安装
   - `getCurrentInstallationType` 返回 `'package-manager'` 或 `'native'`
   - 不显示 npm 弃用通知

3. **开发环境**
   - 开发者在本地开发时不会看到此通知
   - 避免干扰开发工作流

4. **未知安装类型**
   - 如果无法确定安装类型，返回 `'unknown'`
   - 当前代码会显示通知（保守策略）

### 改进建议

1. **细化安装类型处理**
   ```typescript
   // 明确处理未知类型
   if (installationType === 'unknown') {
     // 可以选择显示更通用的消息，或不显示
     logForDebugging('[NpmDeprecation] Unknown installation type')
     return null
   }
   ```

2. **添加分析事件**
   ```typescript
   logEvent('tengu_npm_deprecation_shown', {
     installationType,
     isBundled: isInBundledMode()
   })
   ```

3. **缩短消息或添加折叠**
   ```typescript
   // 考虑在短屏幕上使用更简洁的消息
   const SHORT_MESSAGE = 'Claude Code: Run `claude install` to switch from npm'
   ```

4. **添加"不再提示"选项**
   ```typescript
   // 记录用户选择
   if (getGlobalConfig().dismissedNpmDeprecation) {
     return null
   }
   
   // 在通知中添加操作
   actions: [
     { label: 'Install', command: '/install' },
     { label: 'Dismiss', callback: () => saveGlobalConfig(c => ({ ...c, dismissedNpmDeprecation: true })) }
   ]
   ```

5. **优化检测性能**
   ```typescript
   // 考虑缓存安装类型检测结果
   const cachedType = getSessionCache('installationType')
   if (cachedType) {
     return processCachedType(cachedType)
   }
   ```

6. **区分 npm-global 和 npm-local**
   ```typescript
   // 可以针对不同 npm 安装类型显示不同的引导
   if (installationType === 'npm-global') {
     return { text: 'Uninstall: npm -g uninstall @anthropic-ai/claude-code, then run: claude install' }
   } else if (installationType === 'npm-local') {
     return { text: 'Remove ~/.claude/local, then run: claude install' }
   }
   ```

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useNpmDeprecationNotification.tsx`
- **启动通知基类**: `src/hooks/notifs/useStartupNotification.ts`
- **捆绑模式检测**: `src/utils/bundledMode.ts`
- **安装类型检测**: `src/utils/doctorDiagnostic.ts`
- **环境工具**: `src/utils/envUtils.ts`
