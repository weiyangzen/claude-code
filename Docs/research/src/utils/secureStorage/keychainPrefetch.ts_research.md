# keychainPrefetch.ts 深入研究

## 场景与职责

`keychainPrefetch.ts` 实现了**macOS Keychain 的预读取优化机制**，用于解决 Claude Code 启动时的性能瓶颈。这是启动优化架构的关键组件，与 `settings/mdm/rawRead.ts` 中的 MDM 预读取采用相同的设计模式。

### 核心职责

1. **并行化 Keychain 读取**：在 main.tsx 模块评估期间并行启动 Keychain 读取子进程
2. **减少启动延迟**：将顺序的 ~65ms Keychain 读取优化为与模块导入并行执行
3. **缓存预热**：提前将 Keychain 数据加载到内存缓存，供后续同步读取使用
4. **支持两种凭据类型**：同时预取 OAuth 凭据和 Legacy API Key

### 性能问题背景

在实现预读取之前，`isRemoteManagedSettingsEligible()` 函数会**顺序**读取两个 Keychain 条目：
- `"Claude Code-credentials"` (OAuth tokens) — ~32ms
- `"Claude Code"` (legacy API key) — ~33ms
- **总计**：~65ms 的同步阻塞时间

通过预读取，这两个操作与 main.tsx 的 ~65ms 模块导入并行执行，几乎消除了启动阻塞。

---

## 功能点目的

### 1. 异步预读取启动

```typescript
export function startKeychainPrefetch(): void
```

**目的**：在 main.tsx 顶层立即启动 Keychain 读取子进程，与后续模块导入并行执行。

**触发条件**：
- 仅在 macOS 平台 (`process.platform === 'darwin'`)
- 非裸模式 (`!isBareMode()`)
- 避免重复启动 (`!prefetchPromise`)

### 2. 预读取完成等待

```typescript
export async function ensureKeychainPrefetchCompleted(): Promise<void>
```

**目的**：在 main.tsx 的 preAction 阶段等待预读取完成，确保后续操作可以使用缓存数据。

**特点**：
- 如果预读取已完成，立即返回
- 如果预读取仍在进行，等待其完成
- 非 Darwin 平台立即返回

### 3. Legacy API Key 预读取结果获取

```typescript
export function getLegacyApiKeyPrefetchResult(): { stdout: string | null } | null
```

**目的**：供 `auth.ts` 中的 `getApiKeyFromConfigOrMacOSKeychain()` 使用，避免在预读取完成后仍进行同步 spawn。

### 4. 预读取缓存清理

```typescript
export function clearLegacyApiKeyPrefetch(): void
```

**目的**：在凭据更新后清理预读取缓存，防止陈旧数据影响后续读取。

---

## 具体技术实现

### 关键数据结构

```typescript
// 预读取超时时间：10秒
const KEYCHAIN_PREFETCH_TIMEOUT_MS = 10_000

// Legacy API Key 预读取结果（与 auth.ts 共享）
let legacyApiKeyPrefetch: { stdout: string | null } | null = null

// 预读取 Promise（防止重复启动）
let prefetchPromise: Promise<void> | null = null

// 子进程结果类型
type SpawnResult = { stdout: string | null; timedOut: boolean }
```

### 核心算法流程

#### 预读取启动流程
```
1. 检查平台是否为 darwin，不是则直接返回
2. 检查是否已启动（prefetchPromise 是否存在）
3. 检查是否为裸模式（isBareMode），是则返回
4. 并行启动两个子进程：
   - spawnSecurity(OAuth service name)
   - spawnSecurity(Legacy API Key service name)
5. 创建 Promise.all 等待两者完成
6. 对于每个结果：
   - 如果未超时，将结果写入缓存（primeKeychainCacheFromPrefetch）
   - 如果超时，不写入缓存（让同步读取重试）
```

#### 子进程执行流程
```typescript
function spawnSecurity(serviceName: string): Promise<SpawnResult> {
  return new Promise(resolve => {
    execFile(
      'security',
      ['find-generic-password', '-a', getUsername(), '-w', '-s', serviceName],
      { encoding: 'utf-8', timeout: KEYCHAIN_PREFETCH_TIMEOUT_MS },
      (err, stdout) => {
        resolve({
          stdout: err ? null : stdout?.trim() || null,
          timedOut: Boolean(err && 'killed' in err && err.killed),
        })
      },
    )
  })
}
```

**关键处理**：
- **Exit 44**：条目不存在是有效的"无密钥"结果，可以缓存为 null
- **超时** (`err.killed`)：Keychain 可能有密钥但我们无法获取，不缓存，让同步 spawn 重试

### 模块导入优化

文件头部注释详细说明了导入链的优化考虑：

```typescript
/**
 * Imports stay minimal: child_process + macOsKeychainHelpers.ts (NOT
 * macOsKeychainStorage.ts — that pulls in execa → human-signals →
 * cross-spawn, ~58ms of synchronous module init). The helpers file's own
 * import chain (envUtils, oauth constants, crypto) is already evaluated by
 * startupProfiler.ts at main.tsx:5, so no new module-init cost lands here.
 */
```

**优化要点**：
1. 使用 Node.js 原生 `child_process.execFile` 而非 `execa`
2. 从 `macOsKeychainHelpers.ts` 导入辅助函数（轻量）
3. 避免导入 `macOsKeychainStorage.ts`（会拉入 ~58ms 的模块初始化）

---

## 关键代码路径与文件引用

### 文件位置
- **主文件**：`src/utils/secureStorage/keychainPrefetch.ts`

### 依赖关系

```
keychainPrefetch.ts
├── imports:
│   ├── execFile from 'child_process' (Node.js 原生)
│   ├── isBareMode from '../envUtils.js'
│   ├── CREDENTIALS_SERVICE_SUFFIX from './macOsKeychainHelpers.js'
│   ├── getMacOsKeychainStorageServiceName from './macOsKeychainHelpers.js'
│   ├── getUsername from './macOsKeychainHelpers.js'
│   └── primeKeychainCacheFromPrefetch from './macOsKeychainHelpers.js'
├── exports:
│   ├── startKeychainPrefetch()
│   ├── ensureKeychainPrefetchCompleted()
│   ├── getLegacyApiKeyPrefetchResult()
│   └── clearLegacyApiKeyPrefetch()
└── used by:
    ├── main.tsx (启动时调用 startKeychainPrefetch)
    └── utils/auth.ts (获取 Legacy API Key 预读取结果)
```

### 调用链

#### 启动预读取
```
main.tsx (top-level)
├── import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
├── startKeychainPrefetch()  // 立即执行，非阻塞
│   ├── execFile('security', [...]) for OAuth credentials
│   └── execFile('security', [...]) for Legacy API Key
│
└── ... (继续导入其他模块，与预读取并行)
```

#### 等待预读取完成
```
main.tsx preAction
├── await ensureKeychainPrefetchCompleted()
│   └── 等待 prefetchPromise 完成（如果还在运行）
│
└── 继续执行，Keychain 数据已在缓存中
```

#### 消费预读取结果
```
utils/auth.ts getApiKeyFromConfigOrMacOSKeychain()
├── getLegacyApiKeyPrefetchResult()
│   └── 如果预读取已完成，直接使用结果
│   └── 如果未预读取或超时，执行同步 spawn
```

---

## 依赖与外部交互

### 直接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `execFile` | `child_process` (Node.js) | 执行 `security` CLI |
| `isBareMode` | `../envUtils.js` | 检测裸模式（跳过预读取） |
| `CREDENTIALS_SERVICE_SUFFIX` | `./macOsKeychainHelpers.js` | OAuth 凭据服务名后缀 |
| `getMacOsKeychainStorageServiceName` | `./macOsKeychainHelpers.js` | 生成服务名 |
| `getUsername` | `./macOsKeychainHelpers.js` | 获取用户名 |
| `primeKeychainCacheFromPrefetch` | `./macOsKeychainHelpers.js` | 写入缓存 |

### 外部系统交互

| 系统 | 交互方式 | 说明 |
|------|----------|------|
| macOS Keychain | `security find-generic-password` CLI | 读取凭据 |
| 进程调度 | `child_process.execFile` | 异步子进程 |

### 环境变量依赖

| 变量 | 用途 |
|------|------|
| `CLAUDE_CONFIG_DIR` | 影响服务名生成（通过 getMacOsKeychainStorageServiceName） |
| `USER` | 用户名（通过 getUsername） |
| `CLAUDE_CODE_SIMPLE` | 裸模式检测（通过 isBareMode） |

---

## 风险、边界与改进建议

### 已知风险

#### 1. 超时处理复杂性
- **场景**：预读取超时（10秒）
- **当前行为**：不写入缓存，让同步读取重试
- **风险**：如果 Keychain 持续缓慢，每次启动都会经历预读取超时 + 同步读取超时
- **缓解**：同步读取使用更长的超时时间

#### 2. 竞态条件
- **场景**：预读取正在进行时，用户执行 `/login` 更新凭据
- **风险**：预读取的 stale 数据可能覆盖新写入的凭据
- **缓解**：`primeKeychainCacheFromPrefetch` 只在缓存未初始化时写入（cachedAt === 0）

#### 3. 内存泄漏风险
- **场景**：`prefetchPromise` 和 `legacyApiKeyPrefetch` 是模块级变量
- **风险**：理论上可能持有对大型凭据对象的引用
- **缓解**：凭据对象通常很小，且进程生命周期内有效

### 边界情况

| 场景 | 行为 |
|------|------|
| 非 macOS 平台 | `startKeychainPrefetch()` 和 `ensureKeychainPrefetchCompleted()` 都是 no-op |
| 裸模式 (`--bare`) | 跳过预读取，避免不必要的 Keychain 访问 |
| 重复调用 `startKeychainPrefetch()` | 由于 `prefetchPromise` 检查，只执行一次 |
| 预读取超时 | 不缓存结果，同步读取会重试 |
| Keychain 条目不存在 (exit 44) | 缓存为 null，这是有效的"无密钥"结果 |
| 预读取完成前获取结果 | `getLegacyApiKeyPrefetchResult()` 返回 null |

### 改进建议

#### 1. 添加预读取指标收集
```typescript
// 建议添加
const prefetchMetrics = {
  startTime: number
  endTime: number
  oauthTimedOut: boolean
  legacyTimedOut: boolean
}

// 用于遥测，帮助识别 Keychain 性能问题
```

#### 2. 优化超时策略
```typescript
// 当前：固定 10 秒
const KEYCHAIN_PREFETCH_TIMEOUT_MS = 10_000

// 建议：根据历史数据动态调整
const ADAPTIVE_TIMEOUT_MS = Math.min(
  10_000,
  averageSuccessfulPrefetchTime * 3
)
```

#### 3. 添加预读取取消机制
```typescript
// 建议添加
let abortController: AbortController | null = null

export function cancelKeychainPrefetch(): void {
  abortController?.abort()
}
```

用于在确定不需要 Keychain 数据时（如使用 ANTHROPIC_API_KEY 环境变量）取消进行中的预读取。

#### 4. 改进错误分类
```typescript
// 当前：只区分超时和非超时
// 建议：更详细的错误分类
type PrefetchError = 
  | { type: 'timeout' }
  | { type: 'not_found' }  // exit 44
  | { type: 'keychain_locked' }  // exit 36
  | { type: 'other'; code: number }
```

#### 5. 支持部分成功
```typescript
// 当前：Promise.all 等待两者完成
// 建议：Promise.allSettled，允许部分成功
prefetchPromise = Promise.allSettled([oauthSpawn, legacySpawn]).then(...)
```

### 测试建议

1. **单元测试**：
   - 模拟 `security` CLI 的各种退出码
   - 验证超时处理逻辑
   - 验证重复启动保护

2. **性能测试**：
   - 测量预读取对启动时间的实际影响
   - 对比预读取 vs 同步读取的性能

3. **集成测试**：
   - 验证与 `main.tsx` 启动流程的集成
   - 验证与 `auth.ts` 的协作

4. **边界测试**：
   - 预读取期间凭据被更新的场景
   - Keychain 被锁定的场景
