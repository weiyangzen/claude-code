# agentMemorySnapshot.ts 深度研究文档

## 场景与职责

`agentMemorySnapshot.ts` 是 Claude Code 中负责**代理记忆快照**管理的核心模块。它实现了项目级代理记忆的团队共享机制，允许将代理记忆打包到项目中，使团队成员能够共享和同步代理的学习成果。

该模块解决的核心问题：
1. **团队记忆共享**：如何让团队成员共享代理的学习成果
2. **记忆初始化**：新成员如何快速获取团队的代理记忆
3. **记忆更新检测**：如何检测并提示用户有新的记忆快照可用
4. **记忆同步**：如何将项目快照同步到本地用户记忆

## 功能点目的

### 1. 快照存储结构
- 快照存储在 `.claude/agent-memory-snapshots/<agentType>/` 目录下
- 包含 `snapshot.json`（元数据）和实际的 `.md` 记忆文件
- 使用 `.snapshot-synced.json` 跟踪本地记忆的同步状态

### 2. 快照检测与比较
- `checkAgentMemorySnapshot()`：检查快照状态，返回三种操作类型
  - `'none'`：无需操作（已同步或无快照）
  - `'initialize'`：首次初始化（本地无记忆）
  - `'prompt-update'`：有更新可用（快照比本地新）

### 3. 快照操作
- `initializeFromSnapshot()`：首次从快照初始化本地记忆
- `replaceFromSnapshot()`：用快照替换本地记忆（删除旧文件后复制）
- `markSnapshotSynced()`：标记快照为已同步（不修改本地记忆）

### 4. 元数据管理
- 快照元数据包含 `updatedAt` 时间戳
- 同步元数据记录 `syncedFrom` 时间戳
- 使用时间戳比较确定是否需要更新

## 具体技术实现

### 关键数据类型

```typescript
// 快照操作结果类型
export type SnapshotAction = 'none' | 'initialize' | 'prompt-update'

// 快照检查结果
export type SnapshotCheckResult = {
  action: SnapshotAction
  snapshotTimestamp?: string
}

// 快照元数据（存储在 snapshot.json 中）
const snapshotMetaSchema = z.object({
  updatedAt: z.string().min(1),
})

// 同步元数据（存储在 .snapshot-synced.json 中）
const syncedMetaSchema = z.object({
  syncedFrom: z.string().min(1),
})
```

### 核心算法

**1. 快照状态检查**

```typescript
export async function checkAgentMemorySnapshot(
  agentType: string,
  scope: AgentMemoryScope,
): Promise<SnapshotCheckResult> {
  // 1. 读取快照元数据
  const snapshotMeta = await readJsonFile(
    getSnapshotJsonPath(agentType),
    snapshotMetaSchema(),
  )

  if (!snapshotMeta) {
    return { action: 'none' }
  }

  // 2. 检查本地记忆是否存在
  const localMemDir = getAgentMemoryDir(agentType, scope)
  let hasLocalMemory = false
  try {
    const dirents = await readdir(localMemDir, { withFileTypes: true })
    hasLocalMemory = dirents.some(d => d.isFile() && d.name.endsWith('.md'))
  } catch {
    // Directory doesn't exist
  }

  if (!hasLocalMemory) {
    return { action: 'initialize', snapshotTimestamp: snapshotMeta.updatedAt }
  }

  // 3. 读取同步元数据并比较时间戳
  const syncedMeta = await readJsonFile(
    getSyncedJsonPath(agentType, scope),
    syncedMetaSchema(),
  )

  if (
    !syncedMeta ||
    new Date(snapshotMeta.updatedAt) > new Date(syncedMeta.syncedFrom)
  ) {
    return {
      action: 'prompt-update',
      snapshotTimestamp: snapshotMeta.updatedAt,
    }
  }

  return { action: 'none' }
}
```

**2. 快照复制逻辑**

```typescript
async function copySnapshotToLocal(
  agentType: string,
  scope: AgentMemoryScope,
): Promise<void> {
  const snapshotMemDir = getSnapshotDirForAgent(agentType)
  const localMemDir = getAgentMemoryDir(agentType, scope)

  await mkdir(localMemDir, { recursive: true })

  try {
    const files = await readdir(snapshotMemDir, { withFileTypes: true })
    for (const dirent of files) {
      // 跳过 snapshot.json 元数据文件
      if (!dirent.isFile() || dirent.name === SNAPSHOT_JSON) continue
      
      const content = await readFile(join(snapshotMemDir, dirent.name), {
        encoding: 'utf-8',
      })
      await writeFile(join(localMemDir, dirent.name), content)
    }
  } catch (e) {
    logForDebugging(`Failed to copy snapshot to local agent memory: ${e}`)
  }
}
```

**3. 替换逻辑（带清理）**

```typescript
export async function replaceFromSnapshot(
  agentType: string,
  scope: AgentMemoryScope,
  snapshotTimestamp: string,
): Promise<void> {
  // 1. 删除现有的 .md 文件（避免孤儿文件）
  const localMemDir = getAgentMemoryDir(agentType, scope)
  try {
    const existing = await readdir(localMemDir, { withFileTypes: true })
    for (const dirent of existing) {
      if (dirent.isFile() && dirent.name.endsWith('.md')) {
        await unlink(join(localMemDir, dirent.name))
      }
    }
  } catch {
    // Directory may not exist yet
  }

  // 2. 复制新快照
  await copySnapshotToLocal(agentType, scope)
  
  // 3. 更新同步元数据
  await saveSyncedMeta(agentType, scope, snapshotTimestamp)
}
```

### 路径常量

```typescript
const SNAPSHOT_BASE = 'agent-memory-snapshots'
const SNAPSHOT_JSON = 'snapshot.json'
const SYNCED_JSON = '.snapshot-synced.json'
```

## 依赖与外部交互

### 依赖模块

| 模块路径 | 用途 |
|---------|------|
| `fs/promises` (Node.js) | 文件系统操作 (`mkdir`, `readdir`, `readFile`, `unlink`, `writeFile`) |
| `path` (Node.js) | 路径操作 (`join`) |
| `zod/v4` | 模式验证 |
| `../../utils/cwd.js` | 获取当前工作目录 (`getCwd`) |
| `../../utils/debug.js` | 调试日志 (`logForDebugging`) |
| `../../utils/lazySchema.js` | 懒加载模式 (`lazySchema`) |
| `../../utils/slowOperations.js` | JSON 序列化 (`jsonParse`, `jsonStringify`) |
| `./agentMemory.js` | 获取代理记忆目录 (`getAgentMemoryDir`, `AgentMemoryScope`) |

### 被调用方

通过 Grep 搜索，该模块被以下文件引用：
- `src/tools/AgentTool/loadAgentsDir.ts` - 初始化代理时检查和应用快照

### 调用时序

```
loadAgentsDir.ts
  └── initializeAgentMemorySnapshots()
        ├── checkAgentMemorySnapshot() - 检查状态
        ├── initializeFromSnapshot()   - 首次初始化
        └── replaceFromSnapshot()      - 更新快照
```

## 风险、边界与改进建议

### 已知风险

1. **并发写入风险**：多个进程同时操作快照文件可能导致数据不一致
2. **时间戳依赖**：依赖文件系统时间戳，在时钟不同步的系统上可能出现问题
3. **文件丢失风险**：`replaceFromSnapshot` 先删除后复制，如果复制失败会导致数据丢失
4. **无事务性**：操作不是原子性的，中断可能导致部分同步状态

### 边界情况

1. **空快照目录**：`copySnapshotToLocal` 会静默处理空目录情况
2. **权限错误**：文件操作失败会记录调试日志但继续执行
3. **损坏的 JSON**：使用 Zod 安全解析，损坏的元数据文件会被视为不存在
4. **时区问题**：时间戳比较使用 `new Date()`，应该能正确处理 ISO 格式时间戳

### 改进建议

1. **原子操作**：使用临时目录 + 重命名实现原子性替换
2. **备份机制**：在替换前创建备份，支持回滚
3. **校验和**：为快照文件添加校验和，确保完整性
4. **冲突解决**：当本地和快照都有更新时，提供合并选项
5. **增量同步**：只传输变更的文件，而非整个快照
6. **选择性同步**：允许用户选择要同步的特定记忆文件
7. **快照版本控制**：为快照添加版本号，支持多版本管理

### 代码质量建议

1. 添加更多错误处理，特别是磁盘空间不足的情况
2. 为长时间操作添加进度指示
3. 添加操作日志，便于审计和调试
4. 考虑使用数据库或键值存储替代文件系统操作

### 安全建议

1. 验证快照文件内容，防止恶意文件注入
2. 限制快照文件大小，防止磁盘耗尽攻击
3. 对 `.snapshot-synced.json` 添加完整性保护
4. 考虑对敏感记忆内容进行加密后再共享
