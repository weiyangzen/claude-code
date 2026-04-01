# plainTextStorage.ts 深入研究

## 场景与职责

`plainTextStorage.ts` 实现了 Claude Code 的**明文文件存储后端**，作为 macOS Keychain 的降级方案和非 macOS 平台的主要存储实现。虽然名为 "plaintext"，但通过文件权限控制（0o600）提供了基本的安全保护。

### 核心职责

1. **跨平台兼容**：为 Linux 和 Windows 提供凭据存储功能
2. **macOS 回退**：当 Keychain 不可用时作为降级方案
3. **简单持久化**：将凭据以 JSON 格式存储在文件系统中
4. **权限控制**：设置文件权限为 0o600（仅所有者可读写）

### 使用场景

- **Linux/Windows**：主要存储实现（待实现 libsecret/Windows Credential Manager）
- **macOS**：作为 `FallbackStorage` 的 secondary 存储
- **容器/SSH 环境**：Keychain 不可用时自动回退
- **开发/测试**：快速设置，无需 Keychain 配置

---

## 功能点目的

### 1. 文件路径管理

```typescript
function getStoragePath(): { storageDir: string; storagePath: string }
```

**目的**：确定凭据文件的存储位置。

**路径**：
- 目录：`~/.claude/`（或 `CLAUDE_CONFIG_DIR` 指定的目录）
- 文件：`.credentials.json`

### 2. 同步读取 (read)

```typescript
read(): SecureStorageData | null
```

**目的**：从文件系统同步读取凭据。

**行为**：
- 文件不存在或读取失败时返回 `null`
- 解析 JSON 失败时返回 `null`（静默处理）

### 3. 异步读取 (readAsync)

```typescript
async readAsync(): Promise<SecureStorageData | null>
```

**目的**：提供非阻塞读取接口。

**行为**：与 `read()` 相同，但使用异步文件操作。

### 4. 安全写入 (update)

```typescript
update(data: SecureStorageData): { success: boolean; warning?: string }
```

**目的**：将凭据安全地写入文件。

**安全措施**：
1. 自动创建配置目录（如果不存在）
2. 使用 `writeFileSync_DEPRECATED` 写入文件
3. 设置文件权限为 `0o600`（仅所有者可读写）
4. 返回警告信息提示用户正在使用明文存储

### 5. 删除 (delete)

```typescript
delete(): boolean
```

**目的**：删除凭据文件。

**行为**：
- 文件不存在时返回 `true`（幂等）
- 删除失败时返回 `false`

---

## 具体技术实现

### 关键代码

```typescript
import { chmodSync } from 'fs'
import { join } from 'path'
import { getClaudeConfigHomeDir } from '../envUtils.js'
import { getErrnoCode } from '../errors.js'
import { getFsImplementation } from '../fsOperations.js'
import {
  jsonParse,
  jsonStringify,
  writeFileSync_DEPRECATED,
} from '../slowOperations.js'
import type { SecureStorage, SecureStorageData } from './types.js'

function getStoragePath(): { storageDir: string; storagePath: string } {
  const storageDir = getClaudeConfigHomeDir()
  const storageFileName = '.credentials.json'
  return { storageDir, storagePath: join(storageDir, storageFileName) }
}

export const plainTextStorage = {
  name: 'plaintext',
  
  read(): SecureStorageData | null {
    const { storagePath } = getStoragePath()
    try {
      const data = getFsImplementation().readFileSync(storagePath, {
        encoding: 'utf8',
      })
      return jsonParse(data)
    } catch {
      return null
    }
  },

  async readAsync(): Promise<SecureStorageData | null> {
    const { storagePath } = getStoragePath()
    try {
      const data = await getFsImplementation().readFile(storagePath, {
        encoding: 'utf8',
      })
      return jsonParse(data)
    } catch {
      return null
    }
  },

  update(data: SecureStorageData): { success: boolean; warning?: string } {
    try {
      const { storageDir, storagePath } = getStoragePath()
      
      // 创建目录（如果不存在）
      try {
        getFsImplementation().mkdirSync(storageDir)
      } catch (e: unknown) {
        const code = getErrnoCode(e)
        if (code !== 'EEXIST') {
          throw e
        }
      }

      // 写入文件
      writeFileSync_DEPRECATED(storagePath, jsonStringify(data), {
        encoding: 'utf8',
        flush: false,  // 不强制刷盘，依赖操作系统
      })
      
      // 设置权限为 0o600
      chmodSync(storagePath, 0o600)
      
      return {
        success: true,
        warning: 'Warning: Storing credentials in plaintext.',
      }
    } catch {
      return { success: false }
    }
  },

  delete(): boolean {
    const { storagePath } = getStoragePath()
    try {
      getFsImplementation().unlinkSync(storagePath)
      return true
    } catch (e: unknown) {
      const code = getErrnoCode(e)
      if (code === 'ENOENT') {
        return true  // 文件不存在视为成功
      }
      return false
    }
  },
} satisfies SecureStorage
```

### 文件权限设置

```typescript
chmodSync(storagePath, 0o600)
```

权限含义：
- `6` (所有者)：读 + 写
- `0` (组)：无权限
- `0` (其他)：无权限

这确保只有文件所有者可以访问凭据。

### 错误处理策略

| 操作 | 错误类型 | 处理 |
|------|----------|------|
| read | 文件不存在 | 返回 `null` |
| read | JSON 解析失败 | 返回 `null` |
| update | 目录创建失败 | 抛出异常，返回失败 |
| update | 写入失败 | 返回失败 |
| delete | 文件不存在 | 返回 `true`（幂等） |
| delete | 删除失败 | 返回 `false` |

---

## 关键代码路径与文件引用

### 文件位置
- **主文件**：`src/utils/secureStorage/plainTextStorage.ts`

### 依赖关系

```
plainTextStorage.ts
├── imports:
│   ├── chmodSync from 'fs' (Node.js 内置)
│   ├── join from 'path' (Node.js 内置)
│   ├── getClaudeConfigHomeDir from '../envUtils.js'
│   ├── getErrnoCode from '../errors.js'
│   ├── getFsImplementation from '../fsOperations.js'
│   ├── jsonParse, jsonStringify, writeFileSync_DEPRECATED from '../slowOperations.js'
│   └── SecureStorage types from './types.js'
├── exports:
│   └── plainTextStorage (SecureStorage 实现)
├── used by:
│   ├── index.ts (Linux/Windows 的主要存储)
│   ├── index.ts (macOS FallbackStorage 的 secondary)
│   └── fallbackStorage.ts (作为 secondary 存储)
```

### 调用链

#### Linux/Windows 凭据读取
```
getSecureStorage().read() [index.ts returns plainTextStorage]
├── plainTextStorage.read()
│   ├── getStoragePath() → ~/.claude/.credentials.json
│   ├── getFsImplementation().readFileSync()
│   └── jsonParse(data)
```

#### macOS 回退写入
```
getSecureStorage().update(data) [FallbackStorage]
├── macOsKeychainStorage.update(data) [失败]
└── plainTextStorage.update(data) [回退]
    ├── getFsImplementation().mkdirSync()
    ├── writeFileSync_DEPRECATED()
    ├── chmodSync(0o600)
    └── 返回 warning: "Storing credentials in plaintext."
```

---

## 依赖与外部交互

### 直接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `chmodSync` | `fs` (Node.js) | 设置文件权限 |
| `join` | `path` (Node.js) | 路径拼接 |
| `getClaudeConfigHomeDir` | `../envUtils.js` | 获取配置目录 |
| `getErrnoCode` | `../errors.js` | 错误码提取 |
| `getFsImplementation` | `../fsOperations.js` | 文件系统抽象 |
| `jsonParse` / `jsonStringify` | `../slowOperations.js` | JSON 操作 |
| `writeFileSync_DEPRECATED` | `../slowOperations.js` | 同步文件写入 |
| `SecureStorage` types | `./types.js` | 类型定义 |

### 外部系统交互

| 系统 | 交互方式 | 说明 |
|------|----------|------|
| 文件系统 | Node.js `fs` 模块 | 读写 `~/.claude/.credentials.json` |
| 权限系统 | `chmodSync` | 设置文件权限为 0o600 |

### 环境变量依赖

| 变量 | 用途 |
|------|------|
| `CLAUDE_CONFIG_DIR` | 覆盖默认配置目录路径 |
| `HOME` / `USERPROFILE` | 确定主目录（通过 `getClaudeConfigHomeDir`） |

---

## 风险、边界与改进建议

### 已知风险

#### 1. 明文存储安全风险
- **场景**：凭据以明文 JSON 存储在文件系统
- **后果**：
  - 具有文件系统访问权限的其他用户/进程可以读取
  - 备份软件可能备份凭据文件
  - 文件系统快照包含历史凭据
- **缓解**：
  - 文件权限 0o600
  - macOS 上优先使用 Keychain
  - 明确的警告信息

#### 2. 无加密保护
- **场景**：磁盘被盗或离线访问
- **后果**：凭据可直接读取
- **缓解**：
  - 依赖操作系统磁盘加密（FileVault、BitLocker、LUKS）
  - 未来可实现应用层加密

#### 3. 并发写入冲突
- **场景**：多个 Claude Code 实例同时写入
- **后果**：数据可能损坏或丢失
- **缓解**：
  - 上层调用者（如 MCP auth）使用文件锁
  - 简单的写入策略（无原子写入）

#### 4. 权限提升攻击
- **场景**：攻击者在写入后修改文件权限
- **后果**：其他用户可能获得读取权限
- **缓解**：每次写入后重新设置权限

### 边界情况

| 场景 | 行为 |
|------|------|
| 文件不存在 | `read()` 返回 `null`；`delete()` 返回 `true` |
| 目录不存在 | `update()` 自动创建 |
| JSON 解析失败 | `read()` 返回 `null`（静默处理） |
| 权限设置失败 | 数据已写入，但权限可能不正确 |
| 磁盘满 | `update()` 返回失败 |
| 符号链接 | 跟随链接，写入目标文件 |

### 改进建议

#### 1. 实现应用层加密
```typescript
// 建议：使用用户密码或系统密钥派生加密密钥
import { createCipheriv, createDecipheriv, scryptSync } from 'crypto'

const ENCRYPTION_KEY = scryptSync(getMachineId(), 'salt', 32)

function encrypt(data: string): string {
  const cipher = createCipheriv('aes-256-gcm', ENCRYPTION_KEY, iv)
  return cipher.update(data, 'utf8', 'hex') + cipher.final('hex')
}
```

#### 2. 实现原子写入
```typescript
// 建议：使用临时文件 + 重命名实现原子写入
function atomicWrite(filePath: string, data: string): void {
  const tempPath = `${filePath}.tmp.${Date.now()}`
  writeFileSync(tempPath, data)
  chmodSync(tempPath, 0o600)
  renameSync(tempPath, filePath)
}
```

#### 3. 添加文件锁
```typescript
// 建议：使用 proper-lockfile 或类似库
import lockfile from 'proper-lockfile'

async function lockedUpdate(data: SecureStorageData): Promise<void> {
  const { storagePath } = getStoragePath()
  const release = await lockfile.lock(storagePath)
  try {
    // 执行更新
  } finally {
    await release()
  }
}
```

#### 4. 实现 Linux libsecret 支持
```typescript
// 建议：添加 Linux 原生安全存储
import { setPassword, getPassword } from 'keytar'

export const linuxLibsecretStorage = {
  name: 'libsecret',
  read() { /* ... */ },
  update(data) { /* ... */ },
  delete() { /* ... */ },
} satisfies SecureStorage
```

#### 5. 实现 Windows Credential Manager 支持
```typescript
// 建议：添加 Windows 原生安全存储
import { setPassword, getPassword } from 'keytar'

export const windowsCredentialStorage = {
  name: 'windows-credential',
  read() { /* ... */ },
  update(data) { /* ... */ },
  delete() { /* ... */ },
} satisfies SecureStorage
```

#### 6. 添加完整性检查
```typescript
// 建议：添加 HMAC 或校验和
import { createHmac } from 'crypto'

function addIntegrityCheck(data: SecureStorageData): string {
  const json = jsonStringify(data)
  const hmac = createHmac('sha256', INTEGRITY_KEY).update(json).digest('hex')
  return jsonStringify({ data, hmac })
}
```

#### 7. 改进错误报告
```typescript
// 建议：提供更详细的错误信息
update(data: SecureStorageData): { success: boolean; warning?: string; error?: string } {
  try {
    // ...
  } catch (e) {
    return { 
      success: false, 
      error: `Failed to write credentials: ${errorMessage(e)}` 
    }
  }
}
```

### 测试建议

1. **单元测试**：
   - 文件路径生成（各种环境变量组合）
   - 权限设置验证
   - 错误处理（ENOENT、EACCES 等）

2. **集成测试**：
   - 完整读写删除周期
   - 并发写入行为
   - 磁盘满/权限不足场景

3. **安全测试**：
   - 验证文件权限
   - 验证其他用户无法读取
   - 符号链接攻击防护
