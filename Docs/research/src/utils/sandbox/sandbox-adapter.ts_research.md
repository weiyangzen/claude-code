# sandbox-adapter.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位

`sandbox-adapter.ts` 是 Claude Code CLI 中 **Sandbox 运行时（@anthropic-ai/sandbox-runtime）的适配层**，负责将外部沙箱运行时包与 Claude CLI 的设置系统、工具集成和附加功能进行桥接。

### 1.2 主要职责

| 职责领域 | 说明 |
|---------|------|
| **配置转换** | 将 Claude Code 的 settings.json 格式转换为 sandbox-runtime 所需的 `SandboxRuntimeConfig` 格式 |
| **路径解析** | 处理 Claude Code 特有的路径模式（`//path`、`/path`、`~/path` 等） |
| **权限集成** | 将权限规则（Edit/Read/WebFetch）映射到沙箱的文件系统和网络限制 |
| **生命周期管理** | 初始化、配置刷新、重置沙箱状态 |
| **安全检查** | 检测平台支持性、依赖可用性、提供沙箱不可用的原因 |
| **Git Worktree 支持** | 检测并处理 Git worktree 场景下的写入路径 |
| **安全加固** | 防止 bare-repo 攻击、阻止对 settings.json 的写入 |

### 1.3 使用场景

1. **Bash 命令执行**：通过 `Shell.ts` 调用，决定是否使用沙箱包裹命令
2. **权限检查**：与 `permissions.ts` 协作，确定工具使用权限
3. **设置变更响应**：监听 settings 变化，动态更新沙箱配置
4. **Doctor 诊断**：提供沙箱状态检查，用于故障排查

---

## 2. 功能点目的

### 2.1 配置转换 (`convertToSandboxRuntimeConfig`)

**目的**：将分散在多个设置源（user/project/local/policy/flag）的权限规则统一转换为沙箱运行时配置。

**转换内容**：
- **网络域**：从 WebFetch 权限规则提取 `allowedDomains` / `deniedDomains`
- **文件系统**：从 Edit/Read 权限规则提取 `allowWrite` / `denyWrite` / `denyRead` / `allowRead`
- **Ripgrep 配置**：传递 rg 路径和参数给沙箱内的搜索工具
- **特殊设置**：Unix socket、本地绑定、代理端口等

**关键默认行为**：
- 始终允许写入当前目录 `.` 和 Claude 临时目录
- 始终拒绝写入所有 settings.json 文件（防止沙箱逃逸）
- 始终拒绝写入 `.claude/skills` 目录

### 2.2 路径解析 (`resolvePathPatternForSandbox` vs `resolveSandboxFilesystemPath`)

**两套路径语义**：

| 函数 | `//path` | `/path` | `~/path` | `./path` |
|------|---------|---------|----------|----------|
| `resolvePathPatternForSandbox` | 绝对路径 `/path` | settings 相对路径 | 传递给沙箱处理 | 传递给沙箱处理 |
| `resolveSandboxFilesystemPath` | 绝对路径 `/path`（兼容） | 绝对路径（原样） | 展开为 home | settings 相对路径 |

**设计原因**：Permission 规则使用 `/path` 表示 settings-relative，而 `sandbox.filesystem.*` 设置使用标准路径语义（`/path` = 绝对路径）。这是 issue #30067 的修复。

### 2.3 平台与依赖检查

| 检查项 | 函数 | 说明 |
|--------|------|------|
| 平台支持 | `isSupportedPlatform()` | 支持 macOS、Linux、WSL2+（WSL1 不支持） |
| 平台白名单 | `isPlatformInEnabledList()` | 通过 `enabledPlatforms` 设置限制沙箱仅在特定平台启用 |
| 依赖检查 | `checkDependencies()` | 检查 bubblewrap、socat 等依赖（使用 memoize 缓存） |
| 不可用原因 | `getSandboxUnavailableReason()` | 向用户解释为什么沙箱未启用 |

### 2.4 Git Worktree 支持

**问题**：在 Git worktree 中，`.git` 是一个指向主仓库的文本文件，沙箱需要写入主仓库的 `.git` 目录（index.lock 等）。

**解决方案**：
- `detectWorktreeMainRepoPath()`：解析 `.git` 文件中的 `gitdir:` 路径
- 在 `convertToSandboxRuntimeConfig` 中将主仓库路径加入 `allowWrite`
- 缓存结果（worktree 状态在会话期间不变）

### 2.5 Bare Repo 安全防护 (anthropics/claude-code#29316)

**攻击向量**：攻击者在 cwd 放置 `HEAD`、`objects/`、`refs/` 等文件，Git 的 `is_git_directory()` 会将其识别为 bare repo，配合 `core.fsmonitor` 配置实现沙箱逃逸。

**防护机制**：
1. 配置时检查 bare repo 文件是否存在
2. 存在的文件加入 `denyWrite`（ro-bind）
3. 不存在的文件路径加入 `bareGitRepoScrubPaths`
4. 命令执行后调用 `scrubBareGitRepoFiles()` 删除可能新创建的文件

---

## 3. 具体技术实现

### 3.1 核心数据流

```
settings.json (多个源)
    ↓
getSettings_DEPRECATED() / getSettingsForSource()
    ↓
convertToSandboxRuntimeConfig()
    ├─ 解析权限规则 → allowWrite/denyWrite/denyRead/allowRead
    ├─ 解析网络规则 → allowedDomains/deniedDomains
    ├─ 处理 worktree → 额外 allowWrite
    ├─ 处理 bare repo → denyWrite + scrubPaths
    └─ 获取 ripgrep 配置 → ripgrepConfig
    ↓
BaseSandboxManager.initialize/updateConfig
```

### 3.2 关键数据结构

```typescript
// 从 @anthropic-ai/sandbox-runtime 导入
interface SandboxRuntimeConfig {
  network: {
    allowedDomains: string[]
    deniedDomains: string[]
    allowUnixSockets?: string[]
    allowAllUnixSockets?: boolean
    allowLocalBinding?: boolean
    httpProxyPort?: number
    socksProxyPort?: number
  }
  filesystem: {
    denyRead: string[]
    allowRead: string[]
    allowWrite: string[]
    denyWrite: string[]
  }
  ignoreViolations?: Record<string, string[]>
  enableWeakerNestedSandbox?: boolean
  enableWeakerNetworkIsolation?: boolean
  ripgrep?: { command: string; args?: string[]; argv0?: string }
}

// Claude CLI 接口
interface ISandboxManager {
  initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void>
  isSandboxingEnabled(): boolean
  isSandboxRequired(): boolean
  wrapWithSandbox(command: string, binShell?: string, customConfig?: Partial<SandboxRuntimeConfig>, abortSignal?: AbortSignal): Promise<string>
  // ... 其他方法
}
```

### 3.3 初始化流程

```typescript
async function initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void> {
  // 1. 检查沙箱是否启用
  if (!isSandboxingEnabled()) return

  // 2. 包装回调以强制执行 allowManagedDomainsOnly 策略
  const wrappedCallback = sandboxAskCallback ? async (hostPattern) => {
    if (shouldAllowManagedSandboxDomainsOnly()) return false
    return sandboxAskCallback(hostPattern)
  } : undefined

  // 3. 创建初始化 Promise（同步创建防止竞态）
  initializationPromise = (async () => {
    // 3.1 检测 worktree（只执行一次）
    if (worktreeMainRepoPath === undefined) {
      worktreeMainRepoPath = await detectWorktreeMainRepoPath(getCwdState())
    }

    // 3.2 转换配置并初始化
    const settings = getSettings_DEPRECATED()
    const runtimeConfig = convertToSandboxRuntimeConfig(settings)
    await BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)

    // 3.3 订阅设置变更
    settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
      const newConfig = convertToSandboxRuntimeConfig(getSettings_DEPRECATED())
      BaseSandboxManager.updateConfig(newConfig)
    })
  })()
}
```

### 3.4 权限规则解析

```typescript
// 本地实现避免循环依赖
function permissionRuleValueFromString(ruleString: string): PermissionRuleValue {
  const matches = ruleString.match(/^([^(]+)\(([^)]+)\)$/)
  if (!matches) {
    return { toolName: ruleString }
  }
  return {
    toolName: matches[1],
    ruleContent: matches[2]
  }
}

// 提取前缀用于 Bash 权限规则
function permissionRuleExtractPrefix(permissionRule: string): string | null {
  const match = permissionRule.match(/^(.+):\*$/)
  return match?.[1] ?? null
}
```

### 3.5 排除命令处理

```typescript
export function addToExcludedCommands(
  command: string,
  permissionUpdates?: Array<{ type: string; rules: Array<{ toolName: string; ruleContent?: string }> }>
): string {
  // 1. 从 permissionUpdates 提取 Bash 规则的模式
  // 2. 如果规则是 "Bash(command:*)" 格式，提取前缀
  // 3. 将模式添加到 localSettings.sandbox.excludedCommands
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件路径

| 文件 | 角色 |
|------|------|
| `src/utils/sandbox/sandbox-adapter.ts` | 本文件，适配层实现 |
| `src/entrypoints/sandboxTypes.ts` | SandboxSettings Schema 定义 |
| `src/utils/settings/settings.ts` | 设置读取/合并逻辑 |
| `src/utils/settings/constants.ts` | SETTING_SOURCES 定义 |
| `src/utils/settings/types.ts` | SettingsJson 类型定义 |
| `src/utils/permissions/filesystem.ts` | 文件系统权限检查，getClaudeTempDir |
| `src/utils/permissions/permissions.ts` | 权限决策逻辑 |
| `src/utils/ripgrep.ts` | Ripgrep 配置获取 |
| `src/utils/path.ts` | expandPath 等路径工具 |

### 4.2 调用方文件

| 文件 | 使用方式 |
|------|----------|
| `src/utils/Shell.ts` | `SandboxManager.wrapWithSandbox()` 包裹命令 |
| `src/tools/BashTool/BashTool.tsx` | `shouldUseSandbox()` 检查是否使用沙箱 |
| `src/tools/BashTool/shouldUseSandbox.ts` | 调用 `SandboxManager.isSandboxingEnabled()` 等 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | PowerShell 命令沙箱处理 |
| `src/commands/sandbox-toggle/` | `/sandbox` 命令实现 |
| `src/components/sandbox/` | 沙箱相关 UI 组件 |
| `src/screens/REPL.tsx` | 初始化沙箱 |
| `src/cli/print.ts` | 打印模式沙箱初始化 |
| `src/main.tsx` | 应用启动时初始化 |

### 4.3 关键代码行号

```
行  48-  60: Settings Converter 区域，导入和本地权限解析函数
行  99-119: resolvePathPatternForSandbox - 权限规则路径解析
行 138-146: resolveSandboxFilesystemPath - 文件系统设置路径解析
行 172-381: convertToSandboxRuntimeConfig - 核心配置转换函数
行 387-397: Claude CLI 特有状态（initializationPromise, worktreeMainRepoPath, bareGitRepoScrubPaths）
行 404-414: scrubBareGitRepoFiles - bare repo 清理
行 422-445: detectWorktreeMainRepoPath - worktree 检测
行 451-457: checkDependencies - 依赖检查（memoized）
行 491-526: isSupportedPlatform / isPlatformInEnabledList - 平台检查
行 562-592: getSandboxUnavailableReason - 不可用原因诊断
行 669-691: setSandboxSettings - 设置沙箱配置
行 704-725: wrapWithSandbox - 命令包裹
行 730-792: initialize - 沙箱初始化
行 798-803: refreshConfig - 配置刷新
行 808-822: reset - 状态重置
行 828-874: addToExcludedCommands - 排除命令添加
行 927-967: SandboxManager 对象实现
行 973-985: 类型和 Schema 重导出
```

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 |
|------|------|
| `@anthropic-ai/sandbox-runtime` | 核心沙箱运行时，提供 BaseSandboxManager、类型定义 |
| `lodash-es` | `memoize` 用于缓存依赖检查和平台检查结果 |

### 5.2 内部模块依赖

```typescript
// 设置系统
import { getSettings_DEPRECATED, getSettingsForSource, ... } from '../settings/settings.js'
import { SETTING_SOURCES } from '../settings/constants.js'
import { settingsChangeDetector } from '../settings/changeDetector.js'

// 权限系统
import { getClaudeTempDir } from '../permissions/filesystem.js'
import type { PermissionRuleValue } from '../permissions/PermissionRule.js'

// 工具相关
import { BASH_TOOL_NAME } from 'src/tools/BashTool/toolName.js'
import { FILE_EDIT_TOOL_NAME } from 'src/tools/FileEditTool/constants.js'
import { FILE_READ_TOOL_NAME } from 'src/tools/FileReadTool/prompt.js'
import { WEB_FETCH_TOOL_NAME } from 'src/tools/WebFetchTool/prompt.js'

// 其他工具
import { expandPath } from '../path.js'
import { ripgrepCommand } from '../ripgrep.js'
import { getPlatform } from '../platform.js'
```

### 5.3 与 @anthropic-ai/sandbox-runtime 的交互

```typescript
// 导入
import {
  SandboxManager as BaseSandboxManager,
  SandboxRuntimeConfigSchema,
  SandboxViolationStore,
} from '@anthropic-ai/sandbox-runtime'

// 调用点
BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)
BaseSandboxManager.updateConfig(newConfig)
BaseSandboxManager.wrapWithSandbox(command, binShell, customConfig, abortSignal)
BaseSandboxManager.checkDependencies({ command: rgPath, args: rgArgs })
BaseSandboxManager.isSupportedPlatform()
BaseSandboxManager.cleanupAfterCommand()
BaseSandboxManager.reset()
```

---

## 6. 风险、边界与改进建议

### 6.1 安全风险

| 风险点 | 说明 | 缓解措施 |
|--------|------|----------|
| **Bare Repo 攻击** | 攻击者在 cwd 放置 Git bare repo 文件绕过沙箱 | 已缓解：denyWrite + scrubBareGitRepoFiles |
| **Settings 文件写入** | 沙箱内修改 settings.json 实现逃逸 | 已缓解：始终 denyWrite settings.json |
| **Worktree 写入** | worktree 需要写入主仓库 .git | 已缓解：自动检测并允许主仓库路径 |
| **路径解析不一致** | 权限规则和文件系统设置路径语义不同 | 已缓解：两套路径解析函数 |
| **Memoize 缓存失效** | 依赖检查结果缓存可能过期 | 风险较低：依赖在会话期间不变 |

### 6.2 边界情况

1. **WSL1 不支持**：`isSupportedPlatform()` 会返回 false，但用户可能期望沙箱工作
2. **依赖缺失**：bubblewrap/socat 缺失时静默禁用沙箱（除非 `failIfUnavailable: true`）
3. **Glob 模式限制**：Linux/WSL 上 bubblewrap 不支持 glob，会产生警告
4. **Worktree 路径解析**：`gitdir:` 路径可能是相对路径，需要 `resolve(cwd, gitdir)`
5. **并发初始化**：`initializationPromise` 同步创建防止竞态

### 6.3 改进建议

#### 6.3.1 可观测性
- **建议**：增加沙箱决策的详细日志（哪些路径被允许/拒绝，为什么）
- **位置**：`convertToSandboxRuntimeConfig` 中添加结构化日志

#### 6.3.2 错误处理
- **建议**：`getSandboxUnavailableReason` 可以返回更详细的诊断信息（具体缺失哪个依赖）
- **现状**：只返回 "dependencies are missing"，不具体说明

#### 6.3.3 配置验证
- **建议**：在 `convertToSandboxRuntimeConfig` 后使用 `SandboxRuntimeConfigSchema` 验证配置
- **现状**：Schema 被导入但未在转换后验证

#### 6.3.4 路径解析统一
- **建议**：考虑统一 `resolvePathPatternForSandbox` 和 `resolveSandboxFilesystemPath` 的语义
- **风险**：需要向后兼容，可能影响现有用户配置

#### 6.3.5 缓存策略
- **建议**：`checkDependencies` 和 `isSupportedPlatform` 的缓存可能在长时间运行的会话中过期
- **方案**：添加缓存时间戳或提供手动刷新接口

#### 6.3.6 测试覆盖
- **建议**：增加以下场景的测试
  - Worktree 检测（各种 gitdir 格式）
  - Bare repo 文件清理
  - `enabledPlatforms` 限制
  - Policy settings 覆盖行为

### 6.4 技术债务

1. **循环依赖处理**：`permissionRuleValueFromString` 和 `permissionRuleExtractPrefix` 在文件内重复定义，避免与 `permissions.ts` 循环依赖
2. **Deprecated API**：`getSettings_DEPRECATED` 仍被广泛使用，应逐步迁移到 `getInitialSettings`
3. **类型断言**：多处使用 `as const` 和类型断言，可能引入运行时错误
