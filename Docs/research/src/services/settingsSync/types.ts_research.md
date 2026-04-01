# Settings Sync Types - 研究文档

## 场景与职责

本文件 (`src/services/settingsSync/types.ts`) 定义了 Claude Code 设置同步功能的**类型系统与数据校验模式**。它是 Settings Sync 服务的契约层，负责：

1. **API 契约定义**：定义与后端 API (`/api/claude_code/user_settings`) 交互的数据结构
2. **运行时类型安全**：使用 Zod 模式进行运行时数据验证
3. **同步键管理**：定义本地文件与远程存储之间的映射键
4. **延迟加载优化**：通过 `lazySchema` 避免模块初始化时的性能开销

该模块是设置同步功能的**基础依赖**，被 `index.ts` 以及需要理解同步数据格式的其他模块引用。

---

## 功能点目的

### 1. 用户同步数据结构 (`UserSyncData`)

定义从后端 API 获取的完整用户设置数据格式：

| 字段 | 类型 | 说明 |
|------|------|------|
| `userId` | `string` | 用户唯一标识 |
| `version` | `number` | 数据版本号，用于乐观并发控制 |
| `lastModified` | `string` | ISO 8601 格式的时间戳 |
| `checksum` | `string` | MD5 哈希，用于完整性校验 |
| `content` | `UserSyncContent` | 实际的同步内容（键值对） |

### 2. 同步内容结构 (`UserSyncContent`)

采用**扁平键值存储**设计：
- **键**：不透明字符串，通常是模拟的文件路径（如 `~/.claude/settings.json`）
- **值**：UTF-8 字符串内容（JSON、Markdown 等）

这种设计允许灵活扩展，无需修改 API 契约即可支持新的设置类型。

### 3. 同步键常量 (`SYNC_KEYS`)

定义四类同步条目的键名生成规则：

| 类别 | 键名格式 | 对应本地文件 |
|------|----------|--------------|
| 用户设置 | `~/.claude/settings.json` | `~/.claude/settings.json` |
| 用户记忆 | `~/.claude/CLAUDE.md` | `~/.claude/CLAUDE.md` |
| 项目设置 | `projects/{projectId}/.claude/settings.local.json` | `.claude/settings.local.json` |
| 项目记忆 | `projects/{projectId}/CLAUDE.local.md` | `CLAUDE.local.md` |

**关键设计**：项目级键需要传入 `projectId`（来自 `getRepoRemoteHash()` 的 16 位十六进制字符串），从而在不同项目之间隔离同步数据。

### 4. 操作结果类型

- **`SettingsSyncFetchResult`**：下载操作结果，包含 `success`、`data`、`isEmpty`（404 表示无数据）、`error`、`skipRetry` 字段
- **`SettingsSyncUploadResult`**：上传操作结果，包含 `success`、`checksum`、`lastModified`、`error` 字段

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### lazySchema 延迟加载机制

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

**实现原理**：
- 返回一个工厂函数而非直接执行 Zod 模式定义
- 首次调用时执行工厂函数并缓存结果
- 后续调用直接返回缓存值

**性能优势**：避免在模块加载时立即构建复杂的 Zod 模式对象，减少启动时间。

### Zod 模式定义

```typescript
export const UserSyncContentSchema = lazySchema(() =>
  z.object({
    entries: z.record(z.string(), z.string()),
  }),
)

export const UserSyncDataSchema = lazySchema(() =>
  z.object({
    userId: z.string(),
    version: z.number(),
    lastModified: z.string(),
    checksum: z.string(),
    content: UserSyncContentSchema(),
  }),
)
```

**注意**：使用 `zod/v4` 版本，模式通过函数调用 `UserSyncDataSchema()` 获取。

### 类型推断

```typescript
export type UserSyncData = z.infer<ReturnType<typeof UserSyncDataSchema>>
```

由于 `lazySchema` 返回函数，需要使用 `ReturnType` 提取实际的 Zod 模式类型。

---

## 关键代码路径与文件引用

### 导出内容

| 导出名称 | 类型 | 说明 |
|----------|------|------|
| `UserSyncContentSchema` | `() => ZodSchema` | 内容部分 Zod 模式（延迟加载） |
| `UserSyncDataSchema` | `() => ZodSchema` | 完整数据 Zod 模式（延迟加载） |
| `UserSyncData` | `type` | 用户同步数据类型 |
| `SettingsSyncFetchResult` | `type` | 下载结果类型 |
| `SettingsSyncUploadResult` | `type` | 上传结果类型 |
| `SYNC_KEYS` | `const` | 同步键常量对象 |

### 被调用方（使用者）

| 文件路径 | 使用内容 |
|----------|----------|
| `src/services/settingsSync/index.ts` | 全部类型和常量 |

---

## 依赖与外部交互

### 外部依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `zod/v4` | `z` | Zod 模式定义 |
| `../../utils/lazySchema.js` | `lazySchema` | 延迟加载工具函数 |

### 无外部副作用

本模块**纯类型定义**，无网络请求、无文件 I/O、无全局状态修改。

---

## 风险、边界与改进建议

### 当前风险与边界

1. **Zod v4 版本依赖**
   - 使用 `zod/v4` 而非默认导出，如果未来升级 Zod 版本可能需要修改导入路径。

2. **MD5 校验和**
   - `checksum` 字段使用 MD5，虽然仅用于完整性校验而非安全场景，但 MD5 已被证明存在碰撞漏洞。

3. **时间戳格式无校验**
   - `lastModified` 定义为普通 `string`，未使用 `z.string().datetime()` 进行 ISO 8601 格式校验。

4. **版本号语义未定义**
   - `version` 字段仅定义为 `number`，未说明是单调递增、时间戳还是其他语义。

5. **键名硬编码风险**
   - `SYNC_KEYS` 中的键名（如 `~/.claude/settings.json`）是模拟路径，如果本地实际路径变化（如 XDG 目录支持），需要同步修改。

### 改进建议

1. **增强模式校验**
   ```typescript
   lastModified: z.string().datetime(), // 严格 ISO 8601 校验
   checksum: z.string().length(32), // MD5 固定长度
   ```

2. **键名与路径解耦**
   - 考虑将 `SYNC_KEYS` 与实际的 `getSettingsFilePathForSource` 逻辑关联，避免路径变更时不同步。

3. **文档化版本号语义**
   - 在注释中说明 `version` 的递增规则（乐观并发控制中的使用方式）。

4. **考虑使用更安全的哈希**
   - 虽然不影响安全性，但可考虑迁移到 SHA-256 以符合现代标准。
