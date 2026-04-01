# macOsKeychainHelpers.ts 深入研究

## 场景与职责

`macOsKeychainHelpers.ts` 是 macOS Keychain 存储模块的**轻量级辅助工具库**，被设计为 `keychainPrefetch.ts` 和 `macOsKeychainStorage.ts` 的共享依赖。该模块经过精心优化，确保在启动早期（main.tsx 顶层）导入时不会引入昂贵的模块初始化开销。

### 核心职责

1. **服务名生成**：根据配置目录和 OAuth 环境生成唯一的 Keychain 服务名
2. **用户名获取**：安全地获取当前系统用户名
3. **缓存状态管理**：维护 Keychain 读取操作的共享缓存状态
4. **缓存操作**：提供缓存清理和预读取结果注入功能

### 设计约束

该模块有严格的导入限制，文件头部注释明确说明：

```typescript
/**
 * This module MUST NOT import execa, execFileNoThrow, or
 * execFileNoThrowPortable. keychainPrefetch.ts fires at the very top of
 * main.tsx (before the ~65ms of module evaluation it parallelizes), and Bun's
 * __esm wrapper evaluates the ENTIRE module when any symbol is accessed —
 * so a heavy transitive import here defeats the prefetch. The execa →
 * human-signals → cross-spawn chain alone is ~58ms of synchronous init.
 */
```

---

## 功能点目的

### 1. 服务名生成

```typescript
export function getMacOsKeychainStorageServiceName(
  serviceSuffix: string = '',
): string
```

**目的**：生成唯一的 Keychain 服务名，支持：
- 多环境隔离（prod/staging/local OAuth）
- 多配置目录隔离（通过目录哈希）
- 向后兼容（默认目录不使用哈希后缀）

**服务名格式**：
```
Claude Code{OAUTH_FILE_SUFFIX}{SERVICE_SUFFIX}{DIR_HASH}
```

示例：
- 默认目录生产环境：`"Claude Code"`
- 默认目录 OAuth 凭据：`"Claude Code-credentials"`
- 自定义目录 OAuth 凭据：`"Claude Code-credentials-a1b2c3d4"`

### 2. 用户名获取

```typescript
export function getUsername(): string
```

**目的**：获取当前系统用户名，用于 Keychain 条目的 account 字段。

**回退策略**：
1. 优先使用 `process.env.USER`
2. 失败时使用 `os.userInfo().username`
3. 再失败时回退到 `'claude-code-user'`

### 3. 缓存状态管理

```typescript
export const keychainCacheState: {
  cache: { data: SecureStorageData | null; cachedAt: number }
  generation: number
  readInFlight: Promise<SecureStorageData | null> | null
}
```

**目的**：
- **cache**：存储读取的数据和时间戳
- **generation**：缓存版本号，用于检测并发更新
- **readInFlight**：去重并发读取请求

**TTL 策略**：
```typescript
export const KEYCHAIN_CACHE_TTL_MS = 30_000  // 30秒
```

选择 30 秒的原因（代码注释详细说明）：
- OAuth token 有效期以小时计
- 跨进程写入者只有另一个 CC 实例的 `/login` 或刷新
- 过短的 TTL 会导致 MCP 连接器启动时的"读取风暴"（曾观察到 5.5s 的事件循环阻塞）

### 4. 缓存清理

```typescript
export function clearKeychainCache(): void
```

**目的**：在写入或删除操作前清理缓存，确保后续读取获取最新数据。

**操作**：
- 重置缓存数据为 null
- 递增 generation（使进行中的读取失效）
- 清除进行中的读取 Promise

### 5. 预读取结果注入

```typescript
export function primeKeychainCacheFromPrefetch(stdout: string | null): void
```

**目的**：将 `keychainPrefetch.ts` 的预读取结果注入缓存。

**安全机制**：
- 只在缓存未初始化时写入（`cachedAt === 0`）
- 如果同步读取或更新已运行，丢弃预读取结果（它们的结果更权威）
- 解析失败时静默返回，让同步读取重试

---

## 具体技术实现

### 关键常量

```typescript
// OAuth 凭据服务名后缀（与 Legacy API Key 区分）
// DO NOT change this value — it's part of the keychain lookup key
export const CREDENTIALS_SERVICE_SUFFIX = '-credentials'

// 缓存 TTL：30秒
export const KEYCHAIN_CACHE_TTL_MS = 30_000
```

### 服务名生成算法

```typescript
export function getMacOsKeychainStorageServiceName(
  serviceSuffix: string = '',
): string {
  const configDir = getClaudeConfigHomeDir()
  const isDefaultDir = !process.env.CLAUDE_CONFIG_DIR

  // 使用配置目录路径的哈希创建唯一但稳定的后缀
  // 只有非默认目录才添加后缀，保持向后兼容
  const dirHash = isDefaultDir
    ? ''
    : `-${createHash('sha256').update(configDir).digest('hex').substring(0, 8)}`
  return `Claude Code${getOauthConfig().OAUTH_FILE_SUFFIX}${serviceSuffix}${dirHash}`
}
```

### 缓存状态结构

```typescript
export const keychainCacheState: {
  // 缓存数据和时间戳（cachedAt 为 0 表示无效）
  cache: { data: SecureStorageData | null; cachedAt: number }
  
  // 缓存版本号，每次清理时递增
  // readAsync() 在读取前捕获此值，如果读取期间 generation 变化，
  // 说明缓存已被更新，丢弃子进程结果以防止陈旧数据覆盖新数据
  generation: number
  
  // 进行中的读取 Promise，用于去重并发读取
  // TTL 过期时，多个并发读取会共享同一个 Promise，只 spawn 一个子进程
  readInFlight: Promise<SecureStorageData | null> | null
} = {
  cache: { data: null, cachedAt: 0 },
  generation: 0,
  readInFlight: null,
}
```

### 缓存清理实现

```typescript
export function clearKeychainCache(): void {
  keychainCacheState.cache = { data: null, cachedAt: 0 }
  keychainCacheState.generation++
  keychainCacheState.readInFlight = null
}
```

### 预读取结果注入

```typescript
export function primeKeychainCacheFromPrefetch(stdout: string | null): void {
  // 只在缓存未初始化时写入
  if (keychainCacheState.cache.cachedAt !== 0) return
  
  let data: SecureStorageData | null = null
  if (stdout) {
    try {
      // 注意：这里直接使用 JSON.parse 而非 jsonParse()
      // 因为 jsonParse() 会拉入 slowOperations → lodash-es/cloneDeep
      // 增加启动时的模块初始化开销
      data = JSON.parse(stdout)
    } catch {
      // 解析失败，让同步读取重试
      return
    }
  }
  keychainCacheState.cache = { data, cachedAt: Date.now() }
}
```

---

## 关键代码路径与文件引用

### 文件位置
- **主文件**：`src/utils/secureStorage/macOsKeychainHelpers.ts`

### 依赖关系

```
macOsKeychainHelpers.ts
├── imports (轻量，已预加载):
│   ├── createHash from 'crypto' (Node.js 内置)
│   ├── userInfo from 'os' (Node.js 内置)
│   ├── getOauthConfig from 'src/constants/oauth.js'
│   ├── getClaudeConfigHomeDir from '../envUtils.js'
│   └── SecureStorageData type from './types.js'
├── exports:
│   ├── CREDENTIALS_SERVICE_SUFFIX
│   ├── getMacOsKeychainStorageServiceName()
│   ├── getUsername()
│   ├── KEYCHAIN_CACHE_TTL_MS
│   ├── keychainCacheState (可变状态对象)
│   ├── clearKeychainCache()
│   └── primeKeychainCacheFromPrefetch()
├── used by:
│   ├── keychainPrefetch.ts (启动预读取)
│   └── macOsKeychainStorage.ts (主存储实现)
└── used via macOsKeychainStorage.ts:
    ├── services/mcp/auth.ts
    ├── utils/auth.ts
    └── utils/authPortable.ts
```

### 调用链

#### 服务名生成
```
macOsKeychainStorage.ts read/update/delete
├── getMacOsKeychainStorageServiceName(CREDENTIALS_SERVICE_SUFFIX)
│   ├── getClaudeConfigHomeDir()
│   └── getOauthConfig().OAUTH_FILE_SUFFIX
│       └── '' (prod) | '-staging-oauth' | '-local-oauth'
```

#### 缓存操作
```
macOsKeychainStorage.ts update/delete
├── clearKeychainCache()
│   ├── keychainCacheState.cache = { data: null, cachedAt: 0 }
│   ├── keychainCacheState.generation++
│   └── keychainCacheState.readInFlight = null
│
macOsKeychainStorage.ts readAsync
├── keychainCacheState.readInFlight (检查/设置)
└── keychainCacheState.generation (版本检测)
│
keychainPrefetch.ts (预读取完成)
├── primeKeychainCacheFromPrefetch(stdout)
│   └── keychainCacheState.cache = { data, cachedAt: Date.now() }
```

---

## 依赖与外部交互

### 直接依赖

| 依赖 | 来源 | 用途 | 加载成本 |
|------|------|------|----------|
| `createHash` | `crypto` (Node.js) | SHA256 哈希 | 低（内置） |
| `userInfo` | `os` (Node.js) | 获取用户名 | 低（内置） |
| `getOauthConfig` | `src/constants/oauth.js` | OAuth 配置 | 已预加载 |
| `getClaudeConfigHomeDir` | `../envUtils.js` | 配置目录 | 已预加载 |
| `SecureStorageData` | `./types.js` | 类型定义 | 类型-only |

### 显式避免的依赖

| 依赖 | 原因 | 成本 |
|------|------|------|
| `execa` | 拉入 human-signals → cross-spawn | ~58ms 同步初始化 |
| `execFileNoThrow` | 依赖 execa | ~58ms |
| `execFileNoThrowPortable` | 依赖 execa | ~58ms |
| `jsonParse` from slowOperations | 拉入 lodash-es/cloneDeep | 高 |

### 外部系统交互

本模块本身不直接与外部系统交互，只提供辅助函数和状态管理。

---

## 风险、边界与改进建议

### 已知风险

#### 1. 全局可变状态
- **场景**：`keychainCacheState` 是模块级可变对象
- **风险**：多个代码路径同时修改可能导致竞态条件
- **缓解**：
  - 使用 generation 版本号检测并发修改
  - readInFlight 去重并发读取
  - 模块设计为单例模式，一个进程只有一个实例

#### 2. 缓存 TTL 权衡
- **当前**：30 秒 TTL
- **风险**：
  - 过长：跨进程凭据更新（如另一实例登录）延迟可见
  - 过短：导致读取风暴（曾观察到 5.5s 阻塞）
- **缓解**：30 秒是经验值，OAuth token 有效期以小时计

#### 3. 服务名哈希冲突
- **场景**：两个不同目录的 SHA256 前 8 位相同
- **概率**：1/2^32（极低）
- **后果**：不同目录共享同一个 Keychain 条目
- **缓解**：8 位十六进制提供 32 位熵，冲突概率可接受

### 边界情况

| 场景 | 行为 |
|------|------|
| `CLAUDE_CONFIG_DIR` 未设置 | 不使用目录哈希，保持向后兼容 |
| `CLAUDE_CONFIG_DIR` 变化 | 服务名变化，使用不同的 Keychain 条目 |
| `USER` 环境变量未设置 | 回退到 `userInfo().username` |
| `userInfo()` 抛出异常 | 回退到 `'claude-code-user'` |
| 预读取时缓存已初始化 | `primeKeychainCacheFromPrefetch` 不执行任何操作 |
| 并发清理和读取 | generation 检测防止陈旧数据写入 |

### 改进建议

#### 1. 添加缓存统计
```typescript
// 建议添加
export const keychainCacheStats = {
  hits: 0,
  misses: 0,
  prefetchesUsed: 0,
  prefetchesRejected: 0,
}
```

用于遥测和性能分析。

#### 2. 支持持久化缓存
```typescript
// 建议添加
export function persistCacheToDisk(): void {
  // 将缓存写入 ~/.claude/.keychain-cache.json
  // 用于进程重启后的快速恢复
}
```

#### 3. 改进目录哈希算法
```typescript
// 当前：SHA256 前 8 位
// 建议：使用更短的哈希或编码
const dirHash = isDefaultDir
  ? ''
  : `-${createHash('xxhash').update(configDir).digest('hex').substring(0, 6)}`
```

xxhash 更快，6 位提供足够熵。

#### 4. 添加缓存一致性检查
```typescript
// 建议添加
export function validateCacheConsistency(): boolean {
  // 检查 cache.cachedAt 与 generation 的一致性
  // 检测潜在的竞态条件
}
```

#### 5. 支持多 Keychain 条目分片
```typescript
// 建议：当数据量超过 4KB 限制时，分片存储
const MAX_KEYCHAIN_ENTRY_SIZE = 4096 - 64  // security -i 限制

export function getShardingKey(index: number): string {
  return `${getMacOsKeychainStorageServiceName()}-${index}`
}
```

### 测试建议

1. **单元测试**：
   - 服务名生成（各种配置组合）
   - 用户名获取（各种回退场景）
   - 缓存状态操作（清理、注入、并发）

2. **集成测试**：
   - 与 keychainPrefetch.ts 的协作
   - 与 macOsKeychainStorage.ts 的协作

3. **性能测试**：
   - 验证模块导入时间（应 < 1ms）
   - 验证缓存命中率
