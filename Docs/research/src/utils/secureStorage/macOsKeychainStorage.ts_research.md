# macOsKeychainStorage.ts 深入研究

## 场景与职责

`macOsKeychainStorage.ts` 是 Claude Code 在 macOS 上的**主要安全存储实现**，通过 macOS Keychain 系统服务提供凭据的安全存储。它是 `SecureStorage` 接口的完整实现，支持同步和异步读取、写入和删除操作。

### 核心职责

1. **Keychain 读写**：通过 `security` CLI 与 macOS Keychain 交互
2. **缓存管理**：使用 TTL 缓存减少重复的 Keychain 访问
3. **错误恢复**：实现"stale-while-error"策略，在读取失败时返回缓存数据
4. **并发控制**：去重并发读取请求，避免读取风暴
5. **安全写入**：使用十六进制编码和 stdin 输入避免命令注入和进程监控
6. **Keychain 状态检测**：检测 Keychain 是否被锁定（常见于 SSH 会话）

### 关键设计约束

#### 4KB 限制
macOS `security -i` 使用 4096 字节的 fgets() 缓冲区。超过此限制的命令行会被截断，导致凭据损坏（#30337）。

```typescript
const SECURITY_STDIN_LINE_LIMIT = 4096 - 64  // 4032 字节安全限制
```

---

## 功能点目的

### 1. 同步读取 (read)

```typescript
read(): SecureStorageData | null
```

**目的**：提供同步读取接口，满足 `SecureStorage` 契约。

**策略**：
1. 检查 TTL 缓存，如果未过期直接返回
2. 执行同步 `security find-generic-password` 调用
3. 解析 JSON 结果并更新缓存
4. 如果读取失败但缓存有数据，返回陈旧缓存（stale-while-error）

### 2. 异步读取 (readAsync)

```typescript
async readAsync(): Promise<SecureStorageData | null>
```

**目的**：提供非阻塞读取接口，避免在事件循环中阻塞。

**策略**：
1. 检查 TTL 缓存
2. 如果有进行中的读取，复用其 Promise（去重）
3. 捕获当前 generation，读取完成后验证
4. 如果 generation 已变化，丢弃结果（防止陈旧数据覆盖）

### 3. 安全写入 (update)

```typescript
update(data: SecureStorageData): { success: boolean; warning?: string }
```

**目的**：安全地将凭据写入 Keychain。

**安全特性**：
- **十六进制编码**：将 JSON 数据转换为十六进制，避免转义问题
- **stdin 优先**：使用 `security -i` 通过 stdin 输入，进程监控工具（如 CrowdStrike）只能看到 "security -i"，看不到实际数据（INC-3028）
- **大小限制**：超过限制时回退到命令行参数（仍使用十六进制编码）

### 4. 删除 (delete)

```typescript
delete(): boolean
```

**目的**：从 Keychain 删除凭据条目。

### 5. Keychain 锁定检测

```typescript
export function isMacOsKeychainLocked(): boolean
```

**目的**：检测 Keychain 是否被锁定（常见于 SSH 会话），用于 UI 提示用户解锁。

**缓存**：进程生命周期内缓存结果，避免重复的 ~27ms 同步调用。

---

## 具体技术实现

### 关键常量

```typescript
// security -i 缓冲区限制（4096 - 64 字节安全边距）
const SECURITY_STDIN_LINE_LIMIT = 4096 - 64

// macOsKeychainStorage 对象
export const macOsKeychainStorage = {
  name: 'keychain',
  read(): SecureStorageData | null { ... },
  async readAsync(): Promise<SecureStorageData | null> { ... },
  update(data: SecureStorageData): { success: boolean; warning?: string } { ... },
  delete(): boolean { ... },
} satisfies SecureStorage
```

### 同步读取实现

```typescript
read(): SecureStorageData | null {
  const prev = keychainCacheState.cache
  
  // 1. 检查 TTL 缓存
  if (Date.now() - prev.cachedAt < KEYCHAIN_CACHE_TTL_MS) {
    return prev.data
  }

  try {
    // 2. 执行同步 security 调用
    const storageServiceName = getMacOsKeychainStorageServiceName(CREDENTIALS_SERVICE_SUFFIX)
    const username = getUsername()
    const result = execSyncWithDefaults_DEPRECATED(
      `security find-generic-password -a "${username}" -w -s "${storageServiceName}"`,
    )
    
    if (result) {
      const data = jsonParse(result)
      keychainCacheState.cache = { data, cachedAt: Date.now() }
      return data
    }
  } catch (_e) {
    // fall through
  }
  
  // 3. Stale-while-error：如果之前有数据，返回陈旧数据
  if (prev.data !== null) {
    logForDebugging('[keychain] read failed; serving stale cache', { level: 'warn' })
    keychainCacheState.cache = { data: prev.data, cachedAt: Date.now() }
    return prev.data
  }
  
  keychainCacheState.cache = { data: null, cachedAt: Date.now() }
  return null
}
```

### 异步读取实现

```typescript
async readAsync(): Promise<SecureStorageData | null> {
  const prev = keychainCacheState.cache
  
  // 1. 检查 TTL 缓存
  if (Date.now() - prev.cachedAt < KEYCHAIN_CACHE_TTL_MS) {
    return prev.data
  }
  
  // 2. 复用进行中的读取
  if (keychainCacheState.readInFlight) {
    return keychainCacheState.readInFlight
  }

  const gen = keychainCacheState.generation
  const promise = doReadAsync().then(data => {
    // 3. 验证 generation，防止陈旧数据覆盖
    if (gen === keychainCacheState.generation) {
      // Stale-while-error
      if (data === null && prev.data !== null) {
        logForDebugging('[keychain] readAsync failed; serving stale cache', { level: 'warn' })
      }
      const next = data ?? prev.data
      keychainCacheState.cache = { data: next, cachedAt: Date.now() }
      keychainCacheState.readInFlight = null
      return next
    }
    return data
  })
  
  keychainCacheState.readInFlight = promise
  return promise
}
```

### 安全写入实现

```typescript
update(data: SecureStorageData): { success: boolean; warning?: string } {
  // 1. 清理缓存
  clearKeychainCache()

  try {
    const storageServiceName = getMacOsKeychainStorageServiceName(CREDENTIALS_SERVICE_SUFFIX)
    const username = getUsername()
    const jsonString = jsonStringify(data)

    // 2. 转换为十六进制避免转义问题
    const hexValue = Buffer.from(jsonString, 'utf-8').toString('hex')

    // 3. 构建命令
    const command = `add-generic-password -U -a "${username}" -s "${storageServiceName}" -X "${hexValue}"\n`

    let result
    // 4. 优先使用 stdin（安全）
    if (command.length <= SECURITY_STDIN_LINE_LIMIT) {
      result = execaSync('security', ['-i'], {
        input: command,
        stdio: ['pipe', 'pipe', 'pipe'],
        reject: false,
      })
    } else {
      // 5. 超过限制时使用命令行参数
      logForDebugging(
        `Keychain payload (${jsonString.length}B JSON) exceeds security -i stdin limit; using argv`,
        { level: 'warn' },
      )
      result = execaSync('security', [
        'add-generic-password',
        '-U',
        '-a', username,
        '-s', storageServiceName,
        '-X', hexValue,
      ], { stdio: ['ignore', 'pipe', 'pipe'], reject: false })
    }

    if (result.exitCode !== 0) {
      return { success: false }
    }

    // 6. 更新缓存
    keychainCacheState.cache = { data, cachedAt: Date.now() }
    return { success: true }
  } catch (_e) {
    return { success: false }
  }
}
```

### Keychain 锁定检测

```typescript
let keychainLockedCache: boolean | undefined

export function isMacOsKeychainLocked(): boolean {
  // 进程生命周期缓存
  if (keychainLockedCache !== undefined) return keychainLockedCache
  
  if (process.platform !== 'darwin') {
    keychainLockedCache = false
    return false
  }

  try {
    const result = execaSync('security', ['show-keychain-info'], {
      reject: false,
      stdio: ['ignore', 'pipe', 'pipe'],
    })
    // Exit code 36 表示 Keychain 被锁定
    keychainLockedCache = result.exitCode === 36
  } catch {
    keychainLockedCache = false
  }
  return keychainLockedCache
}
```

---

## 关键代码路径与文件引用

### 文件位置
- **主文件**：`src/utils/secureStorage/macOsKeychainStorage.ts`

### 依赖关系

```
macOsKeychainStorage.ts
├── imports:
│   ├── execaSync from 'execa'
│   ├── logForDebugging from '../debug.js'
│   ├── execFileNoThrow from '../execFileNoThrow.js'
│   ├── execSyncWithDefaults_DEPRECATED from '../execFileNoThrowPortable.js'
│   ├── jsonParse, jsonStringify from '../slowOperations.js'
│   ├── helpers from './macOsKeychainHelpers.js'
│   └── SecureStorage types from './types.js'
├── exports:
│   ├── macOsKeychainStorage (SecureStorage 实现)
│   └── isMacOsKeychainLocked()
├── used by:
│   ├── index.ts (组合到 FallbackStorage)
│   ├── services/mcp/auth.ts (MCP OAuth 凭据)
│   ├── utils/auth.ts (Legacy API Key)
│   └── components/messages/AssistantTextMessage.tsx (Keychain 锁定检测)
```

### 调用链

#### 凭据读取
```
getSecureStorage().read() [FallbackStorage]
├── macOsKeychainStorage.read()
│   ├── keychainCacheState.cache 检查
│   ├── execSyncWithDefaults_DEPRECATED('security find-generic-password ...')
│   │   └── execaSync (shell: true)
│   └── jsonParse(result)
└── 如果失败，回退到 plainTextStorage.read()
```

#### 凭据写入
```
getSecureStorage().update(data) [FallbackStorage]
├── macOsKeychainStorage.update(data)
│   ├── clearKeychainCache()
│   ├── jsonStringify(data)
│   ├── Buffer.toString('hex') 编码
│   ├── execaSync('security', ['-i'], { input: command })
│   └── 更新缓存
└── 如果失败，回退到 plainTextStorage.update(data)
```

---

## 依赖与外部交互

### 直接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `execaSync` | `execa` | 执行 security CLI |
| `logForDebugging` | `../debug.js` | 调试日志 |
| `execFileNoThrow` | `../execFileNoThrow.js` | 异步执行（readAsync） |
| `execSyncWithDefaults_DEPRECATED` | `../execFileNoThrowPortable.js` | 同步执行（read） |
| `jsonParse` / `jsonStringify` | `../slowOperations.js` | JSON 操作（带性能监控） |
| `macOsKeychainHelpers` | `./macOsKeychainHelpers.js` | 服务名、用户名、缓存状态 |
| `SecureStorage` types | `./types.js` | 类型定义 |

### 外部系统交互

| 系统 | 交互方式 | 说明 |
|------|----------|------|
| macOS Keychain | `security` CLI | 所有存储操作 |
| 系统日志 | `logForDebugging` | 错误和警告日志 |

### security CLI 命令

| 操作 | 命令 |
|------|------|
| 读取 | `security find-generic-password -a "<username>" -w -s "<service>"` |
| 写入 | `security -i` + `add-generic-password -U -a "<username>" -s "<service>" -X "<hex>"` |
| 删除 | `security delete-generic-password -a "<username>" -s "<service>"` |
| 锁定检测 | `security show-keychain-info` |

---

## 风险、边界与改进建议

### 已知风险

#### 1. 4KB 限制
- **场景**：JSON 序列化后的数据超过 ~2013 字节（十六进制编码后 ~4032 字节）
- **后果**：必须回退到命令行参数，进程监控工具可见
- **缓解**：
  - 当前 MCP OAuth 数据设计紧凑
  - 超过限制时记录警告日志
  - 十六进制编码仍提供基本的明文隐藏

#### 2. 同步调用阻塞
- **场景**：`read()` 使用 `execSyncWithDefaults_DEPRECATED`
- **后果**：阻塞事件循环 ~500ms（每次 security spawn）
- **缓解**：
  - 30 秒 TTL 缓存
  - 提供 `readAsync()` 非阻塞接口
  - keychainPrefetch.ts 预读取减少启动阻塞

#### 3. 命令注入风险
- **场景**：用户名或服务名包含特殊字符
- **缓解**：
  - 使用 `execaSync` 的数组参数形式（自动转义）
  - 十六进制编码避免 payload 中的特殊字符问题

#### 4. Keychain 锁定
- **场景**：SSH 会话中 Keychain 未解锁
- **后果**：所有 Keychain 操作失败
- **缓解**：
  - `isMacOsKeychainLocked()` 检测
  - FallbackStorage 回退到明文存储
  - UI 提示用户解锁 Keychain

### 边界情况

| 场景 | 行为 |
|------|------|
| Keychain 条目不存在 | `security` 返回 exit 44，读取返回 null |
| Keychain 被锁定 | `security` 返回 exit 36，操作失败 |
| 数据超过 4KB 限制 | 回退到命令行参数，记录警告 |
| 并发读取 | 通过 `readInFlight` 去重，只 spawn 一个子进程 |
| 读取期间缓存被清理 | generation 检测丢弃陈旧结果 |
| 写入期间失败 | 返回 `{ success: false }`，不更新缓存 |

### 改进建议

#### 1. 实现数据分片
```typescript
// 建议：当数据超过限制时，分片存储到多个 Keychain 条目
function shardData(data: SecureStorageData): string[] {
  const json = jsonStringify(data)
  const chunks: string[] = []
  for (let i = 0; i < json.length; i += MAX_CHUNK_SIZE) {
    chunks.push(json.slice(i, i + MAX_CHUNK_SIZE))
  }
  return chunks
}
```

#### 2. 添加重试机制
```typescript
// 建议：对 transient 失败（如 Keychain 暂时锁定）添加重试
async function readWithRetry(retries = 3): Promise<SecureStorageData | null> {
  for (let i = 0; i < retries; i++) {
    const result = await doReadAsync()
    if (result !== null) return result
    await sleep(100 * (i + 1))
  }
  return null
}
```

#### 3. 改进锁定检测
```typescript
// 建议：不仅检测锁定，还尝试自动解锁（如果可能）
export async function ensureKeychainUnlocked(): Promise<boolean> {
  if (!isMacOsKeychainLocked()) return true
  // 尝试使用系统默认钥匙串解锁
  // 或提示用户输入密码
}
```

#### 4. 添加操作指标
```typescript
// 建议：收集操作统计
const keychainMetrics = {
  reads: { total: 0, cacheHits: 0, errors: 0 },
  writes: { total: 0, fallbackToArgv: 0, errors: 0 },
  deletes: { total: 0, errors: 0 },
}
```

#### 5. 支持 Touch ID / 生物识别
```typescript
// 建议：使用 -T 选项支持生物识别验证
const command = `add-generic-password -U -a "${username}" -s "${service}" -T "" -X "${hexValue}"`
// -T "" 允许任何应用访问，但需要生物识别验证
```

### 测试建议

1. **单元测试**：
   - 模拟 security CLI 的各种退出码
   - 验证缓存行为（TTL、generation、stale-while-error）
   - 验证 4KB 限制处理

2. **集成测试**：
   - 实际 Keychain 操作（在隔离的测试钥匙串上）
   - 锁定/解锁场景
   - 并发读取场景

3. **性能测试**：
   - 测量 security spawn 延迟
   - 验证缓存命中率
   - 测试大负载下的行为
