# gitAvailability.ts 深度研究文档

## 场景与职责

`gitAvailability.ts` 提供 Git 可用性的备忘录化检查，是 Claude Code 插件系统与 GitHub 市场交互的前置条件验证模块。

核心场景：
1. **市场克隆前检查**：避免在 Git 不可用时尝试克隆操作
2. **macOS Xcode CLT 缺失处理**：处理 `/usr/bin/git` xcrun shim 的特殊情况
3. **会话级状态管理**：Git 可用性在单 CLI 会话中视为不变

## 功能点目的

### 1. Git 可用性检查 (`checkGitAvailable`)
- **备忘录化**：使用 `lodash.memoize` 缓存结果，同一会话重复调用返回缓存
- **安全检查**：使用 `which` 而非直接执行 git，避免执行不可信目录中的任意代码
- **PATH 检查**：仅检查 PATH 中是否存在 git 可执行文件
- **macOS 特殊情况**：`/usr/bin/git` xcrun shim 在缺少 Xcode CLT 时仍会返回存在

### 2. Git 不可用标记 (`markGitUnavailable`)
- **用途**：当 git 调用失败时（如 macOS xcrun 错误），强制剩余会话视 Git 为不可用
- **机制**：直接操作 `memoize` 缓存，将 `undefined` 键映射到 `Promise.resolve(false)`
- **触发场景**：
  ```
  xcrun: error: invalid active developer path
  ```

### 3. 缓存清理 (`clearGitAvailabilityCache`)
- **用途**：测试场景下重置缓存状态
- **实现**：调用 `checkGitAvailable.cache?.clear?.()`

## 具体技术实现

### 核心实现
```typescript
import memoize from 'lodash-es/memoize.js'
import { which } from '../which.js'

async function isCommandAvailable(command: string): Promise<boolean> {
  try {
    return !!(await which(command))
  } catch {
    return false
  }
}

export const checkGitAvailable = memoize(async (): Promise<boolean> => {
  return isCommandAvailable('git')
})

export function markGitUnavailable(): void {
  // lodash memoize 使用 undefined 作为无参缓存键
  checkGitAvailable.cache?.set?.(undefined, Promise.resolve(false))
}

export function clearGitAvailabilityCache(): void {
  checkGitAvailable.cache?.clear?.()
}
```

### 设计决策

| 决策 | 理由 |
|------|------|
| 使用 `which` 而非 `exec('git --version')` | 安全最佳实践，避免执行任意代码 |
| 备忘录化 | Git 可用性在单会话中极不可能变化 |
| `cache?.set?.(undefined, ...)` | lodash memoize 无参函数使用 `undefined` 作为缓存键 |

## 关键代码路径与文件引用

### 内部依赖
```
gitAvailability.ts
  └─ which.ts: which(command) → Promise<string | null>
```

### 外部调用方
| 调用方 | 用途 |
|--------|------|
| `pluginLoader.ts` | 加载插件前检查 Git 可用性 |
| `marketplaceManager.ts` | 克隆市场前检查 |
| `pluginInstallationHelpers.ts` | 安装插件时检查 |

### 调用链示例
```
marketplaceManager.ts: installMarketplace()
  ├─ checkGitAvailable()
  │   └─ which('git')
  └─ 若失败 → 调用方处理（如显示友好错误）
```

## 依赖与外部交互

### 依赖模块
| 模块 | 用途 |
|------|------|
| `lodash-es/memoize.js` | 结果备忘录化 |
| `which.ts` | 安全地检查命令是否存在 PATH 中 |

### which.ts 行为
- 使用 `which` npm 包或平台原生实现
- 返回可执行文件的完整路径
- 不存在时抛出错误（被捕获转为 `false`）

## 风险、边界与改进建议

### 已知风险

1. **macOS xcrun shim 误报**
   - **问题**：`/usr/bin/git` 存在且可执行，但实际运行时提示安装 Xcode CLT
   - **缓解**：`markGitUnavailable()` 允许调用方在首次失败后标记不可用
   - **局限**：首次失败仍会发生，用户体验有瑕疵

2. **PATH 变化未检测**
   - **问题**：用户在同一会话中修改 PATH 添加 git，但缓存返回旧结果
   - **现状**：设计接受，CLI 会话通常不修改 PATH
   - **缓解**：`clearGitAvailabilityCache()` 供特殊场景调用

3. **which 实现差异**
   - **问题**：不同平台 `which` 行为可能不一致
   - **现状**：`which.ts` 内部处理跨平台差异

### 边界条件

| 场景 | 行为 |
|------|------|
| git 在 PATH 中 | 返回 `true`，缓存结果 |
| git 不在 PATH 中 | 返回 `false`，缓存结果 |
| 重复调用 | 返回缓存结果，不重新检查 |
| `markGitUnavailable()` 后 | 强制返回 `false` |
| 缓存被清除后 | 下次调用重新检查 |

### 改进建议

1. **macOS xcrun 预检测**
   - 当前：失败后标记不可用
   - 建议：在 macOS 上执行 `xcrun --find git` 预检测，避免首次失败

2. **异步初始化模式**
   - 当前：首次调用触发检查
   - 建议：启动时预检，避免首次操作时的延迟

3. **更细粒度的错误信息**
   - 当前：仅返回 boolean
   - 建议：返回 `{available: boolean, reason?: string}`，帮助诊断

4. **环境变量覆盖**
   - 建议：支持 `CLAUDE_CODE_GIT_PATH` 环境变量显式指定 git 路径，跳过检测
