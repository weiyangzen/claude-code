# fallbackStorage.ts 深入研究

## 场景与职责

`fallbackStorage.ts` 实现了一个**双重回退安全存储机制**，用于在 macOS 上提供高可靠性的凭据存储。它是 Claude Code 安全存储架构的核心组件之一，确保即使主存储（macOS Keychain）暂时不可用，凭据也能被安全地保存和读取。

### 核心职责

1. **存储冗余**：当主存储（macOS Keychain）写入失败时，自动回退到次要存储（明文文件）
2. **数据迁移**：当主存储从空状态成功写入后，自动清理次要存储中的旧数据
3. **读写一致性**：确保读取时优先使用主存储，但写入失败时不会丢失数据
4. **防止循环登录**：修复了因主存储中存在旧凭据导致的"/login 循环"问题（#30337）

### 使用场景

- macOS 系统上 Keychain 被锁定或不可用时
- 容器/虚拟机环境中共享 `.claude` 目录时
- 首次迁移到 Keychain 存储时的平滑过渡

---

## 功能点目的

### 1. 双存储回退机制

```typescript
export function createFallbackStorage(
  primary: SecureStorage,      // macOS Keychain
  secondary: SecureStorage,    // 明文文件存储
): SecureStorage
```

**目的**：提供一个统一的 `SecureStorage` 接口，内部协调两个存储后端的读写操作。

### 2. 智能读写路由

| 操作 | 策略 | 原因 |
|------|------|------|
| `read()` | 优先主存储，失败回退次要 | Keychain 更安全，应优先使用 |
| `update()` | 先写主存储，失败再写次要 | 尽量使用安全存储 |
| `delete()` | 两边都删除 | 确保彻底清理 |

### 3. 数据迁移与清理

当主存储首次成功写入（之前为空）时，自动删除次要存储中的数据：
- **目的**：避免容器/主机共享目录时的凭据泄露
- **引用**：GitHub Issue #1414

### 4. 防循环登录修复

当主存储写入失败但次要存储写入成功时：
- 检测主存储中是否已存在旧凭据
- 如果存在，删除主存储中的旧凭据
- **目的**：防止旧凭据（如已失效的 refresh token）覆盖新凭据导致的登录循环
- **引用**：GitHub Issue #30337

---

## 具体技术实现

### 关键数据结构

```typescript
interface SecureStorage {
  name: string
  read(): SecureStorageData | null
  readAsync(): Promise<SecureStorageData | null>
  update(data: SecureStorageData): { success: boolean; warning?: string }
  delete(): boolean
}

type SecureStorageData = {
  mcpOAuth?: Record<string, McpOAuthData>
  pluginSecrets?: Record<string, Record<string, string>>
  // 其他敏感数据...
}
```

### 核心算法流程

#### Read 流程
```
1. 尝试从 primary.read() 读取
2. 如果结果非 null/undefined，直接返回
3. 否则返回 secondary.read() 或 {}
```

#### Update 流程
```
1. 记录 primary 更新前的状态 (primaryDataBefore)
2. 尝试 primary.update(data)
3. 如果成功：
   - 如果 primaryDataBefore === null（首次写入），删除 secondary
   - 返回成功
4. 如果失败，尝试 secondary.update(data)
5. 如果 secondary 成功：
   - 如果 primaryDataBefore !== null（主存储有旧数据），删除 primary
   - 返回成功（带警告）
6. 如果都失败，返回失败
```

#### Delete 流程
```
1. 执行 primary.delete()
2. 执行 secondary.delete()
3. 返回任一成功的结果
```

### 关键代码路径

```typescript
// 行 27-62: update 方法的完整实现
update(data: SecureStorageData): { success: boolean; warning?: string } {
  // Capture state before update
  const primaryDataBefore = primary.read()

  const result = primary.update(data)

  if (result.success) {
    // Delete secondary when migrating to primary for the first time
    if (primaryDataBefore === null) {
      secondary.delete()
    }
    return result
  }

  const fallbackResult = secondary.update(data)

  if (fallbackResult.success) {
    // Primary write failed but primary may still hold an *older* valid entry
    if (primaryDataBefore !== null) {
      primary.delete()  // 防止旧凭据导致登录循环
    }
    return {
      success: true,
      warning: fallbackResult.warning,
    }
  }

  return { success: false }
}
```

---

## 关键代码路径与文件引用

### 文件位置
- **主文件**：`src/utils/secureStorage/fallbackStorage.ts`

### 依赖关系

```
fallbackStorage.ts
├── imports:
│   └── types.ts (SecureStorage, SecureStorageData)
├── used by:
│   └── index.ts (getSecureStorage 创建 FallbackStorage 实例)
├── primary storage:
│   └── macOsKeychainStorage.ts
└── secondary storage:
    └── plainTextStorage.ts
```

### 调用链

```
main.tsx
├── getSecureStorage() [index.ts]
│   └── createFallbackStorage(macOsKeychainStorage, plainTextStorage)
│       └── fallbackStorage.ts (本文件)
│
├── services/mcp/auth.ts
│   └── getSecureStorage() 用于 MCP OAuth 凭据存储
│
├── utils/plugins/pluginOptionsStorage.ts
│   └── getSecureStorage() 用于插件敏感配置存储
│
└── utils/plugins/mcpbHandler.ts
    └── getSecureStorage() 用于 MCPB 用户配置存储
```

---

## 依赖与外部交互

### 直接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `SecureStorage` interface | `./types.js` | 类型定义 |
| `SecureStorageData` type | `./types.js` | 数据类型 |

### 运行时依赖（通过参数传入）

| 依赖 | 类型 | 职责 |
|------|------|------|
| `primary` | `macOsKeychainStorage` | macOS Keychain 存储 |
| `secondary` | `plainTextStorage` | 明文文件存储（~/.claude/.credentials.json） |

### 外部系统交互

- **macOS Keychain**：通过 `security` CLI 命令交互
- **文件系统**：通过 `plainTextStorage` 写入 `~/.claude/.credentials.json`

---

## 风险、边界与改进建议

### 已知风险

#### 1. 双存储不一致风险
- **场景**：主存储写入失败，数据写入次要存储后，下次读取时主存储恢复
- **后果**：可能读到主存储中的旧数据
- **缓解**：update 成功后会检测并删除主存储中的旧数据

#### 2. 明文存储泄露风险
- **场景**：当 Keychain 不可用时，凭据以明文形式存储在文件系统
- **后果**：其他用户或进程可能读取凭据文件
- **缓解**：文件权限设置为 `0o600`，且一旦 Keychain 恢复会自动迁移

#### 3. 并发写入冲突
- **场景**：多个 Claude Code 实例同时写入
- **后果**：后写入的数据可能覆盖先写入的数据
- **缓解**：上层调用者（如 MCP auth）使用文件锁机制

### 边界情况

| 场景 | 行为 |
|------|------|
| primary 和 secondary 都失败 | 返回 `{ success: false }` |
| primary 读取失败 | 回退到 secondary 读取 |
| secondary 读取也失败 | 返回 `{}`（空对象） |
| primary 为空，secondary 有数据 | 返回 secondary 的数据 |
| 首次写入 primary 成功 | 自动删除 secondary 中的数据 |

### 改进建议

#### 1. 添加健康检查机制
```typescript
// 建议添加
function checkPrimaryHealth(): boolean {
  // 快速检测 Keychain 是否可用
}
```

#### 2. 增强并发控制
- 当前依赖上层调用者的文件锁
- 可考虑在 fallbackStorage 层添加简单的版本号/时间戳检测

#### 3. 改进警告信息
- 当前仅返回 `warning?: string`
- 可考虑返回更详细的诊断信息，帮助用户排查 Keychain 问题

#### 4. 定期迁移检查
- 当前仅在写入时触发迁移
- 可考虑在读取时检测 secondary 有数据但 primary 为空的情况，主动迁移

#### 5. 监控与遥测
- 添加指标收集：fallback 使用频率、迁移成功率
- 帮助识别 Keychain 可靠性问题

### 测试建议

1. **单元测试**：
   - 模拟 primary 失败，验证 secondary 写入
   - 验证首次写入后的自动清理行为
   - 验证旧数据删除逻辑（#30337 修复）

2. **集成测试**：
   - 锁定 Keychain 后的完整登录流程
   - 容器环境下的凭据共享场景

3. **边界测试**：
   - 两个存储都失败的错误处理
   - 大数据量（接近 4KB Keychain 限制）的写入
