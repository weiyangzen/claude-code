# types.ts 深度研究

## 场景与职责

本模块是 Claude Code CLI **服务器相关功能的类型定义中心**，提供 Direct Connect 和本地服务器模式所需的 TypeScript 类型定义和 Zod 验证 Schema。它是连接客户端与服务器之间的**类型契约层**，确保双方数据结构的一致性。

**核心场景：**
1. **Direct Connect 模式**: 定义会话创建响应格式、服务器配置类型
2. **本地服务器模式**: 定义会话状态、会话索引结构（用于会话持久化）
3. **类型安全**: 通过 Zod Schema 实现运行时验证，配合 TypeScript 类型提供编译时检查

## 功能点目的

### 1. 连接响应验证 (connectResponseSchema)

**目的：** 验证直连服务器创建会话后的响应格式

**验证字段：**
- `session_id`: 服务器分配的唯一会话标识符
- `ws_url`: WebSocket 连接端点地址
- `work_dir?`: 可选的工作目录建议

### 2. 服务器配置类型 (ServerConfig)

**目的：** 定义本地 Claude 服务器的配置结构

**配置项：**
- `port` / `host`: 服务器监听地址
- `authToken`: 访问认证令牌
- `unix?`: Unix 域套接字路径（可选）
- `idleTimeoutMs?`: 空闲会话超时（毫秒，0=永不超时）
- `maxSessions?`: 最大并发会话数
- `workspace?`: 默认工作目录

### 3. 会话状态管理 (SessionState, SessionInfo)

**目的：** 定义本地服务器中会话的生命周期状态

**会话状态：**
- `starting`: 正在启动
- `running`: 运行中
- `detached`: 已分离（后台运行）
- `stopping`: 正在停止
- `stopped`: 已停止

**会话信息：**
- 基础元数据：ID、状态、创建时间、工作目录
- 进程引用：关联的 ChildProcess（可能为 null）
- 可选会话密钥：用于会话恢复

### 4. 会话索引持久化 (SessionIndexEntry, SessionIndex)

**目的：** 支持会话在服务器重启后的恢复

**索引条目：**
- 服务器会话 ID 与转录会话 ID 的映射
- 工作目录、权限模式
- 创建时间和最后活跃时间

**存储位置：** `~/.claude/server-sessions.json`

## 具体技术实现

### 关键数据结构

#### 1. 连接响应 Schema

```typescript
export const connectResponseSchema = lazySchema(() =>
  z.object({
    session_id: z.string(),      // 必填：会话唯一标识
    ws_url: z.string(),          // 必填：WebSocket URL
    work_dir: z.string().optional(), // 可选：推荐工作目录
  }),
)
```

**使用场景：**
- `createDirectConnectSession.ts` 中验证服务器响应
- 确保必需的连接信息存在

#### 2. 服务器配置

```typescript
export type ServerConfig = {
  port: number                    // 监听端口
  host: string                    // 监听主机
  authToken: string               // 认证令牌
  unix?: string                   // Unix socket 路径（替代 TCP）
  idleTimeoutMs?: number          // 空闲超时（默认永不超时）
  maxSessions?: number            // 最大会话数限制
  workspace?: string              // 默认工作区
}
```

**配置策略：**
- 必填项：网络基础配置 + 安全认证
- 可选项：资源限制和默认值

#### 3. 会话状态机

```
┌──────────────────────────────────────────────────────────────┐
│                      会话状态流转                             │
│                                                               │
│   ┌─────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐ │
│   │ starting│───→│ running │───→│ detached │───→│stopping │ │
│   └─────────┘    └────┬────┘    └────┬─────┘    └────┬────┘ │
│                       │              │               │      │
│                       ↓              │               ↓      │
│                  ┌─────────┐         │          ┌─────────┐ │
│                  │ stopped │←────────┘          │ stopped │ │
│                  └─────────┘                    └─────────┘ │
│                                                               │
│  说明：                                                       │
│  - starting → running: 子进程启动成功                          │
│  - running → detached: 客户端断开但会话继续运行                 │
│  - running → stopping: 客户端请求停止                          │
│  - detached → stopping: 超时或显式停止                         │
└──────────────────────────────────────────────────────────────┘
```

#### 4. 会话信息结构

```typescript
export type SessionInfo = {
  id: string                     // 会话唯一标识
  status: SessionState           // 当前状态
  createdAt: number              // 创建时间戳（毫秒）
  workDir: string                // 工作目录
  process: ChildProcess | null   // 关联的子进程
  sessionKey?: string            // 可选的会话密钥
}
```

#### 5. 会话索引（持久化）

```typescript
export type SessionIndexEntry = {
  sessionId: string              // 服务器会话 ID
  transcriptSessionId: string    // 转录文件中的会话 ID
  cwd: string                    // 工作目录
  permissionMode?: string        // 权限模式设置
  createdAt: number              // 创建时间
  lastActiveAt: number           // 最后活跃时间
}

export type SessionIndex = Record<string, SessionIndexEntry>
```

**设计意图：**
- 分离服务器内部 ID 和转录会话 ID，支持不同恢复策略
- 记录权限模式，恢复时保持一致的安全策略
- 时间戳用于会话清理和 LRU 淘汰

### Schema 延迟加载机制

```typescript
import { lazySchema } from '../utils/lazySchema.js'

export const connectResponseSchema = lazySchema(() =>
  z.object({...})
)
```

**优势：**
- 避免模块加载时立即构建 Zod Schema（可能较耗时）
- 首次访问时才创建 Schema 实例
- 结果缓存，后续复用

## 关键代码路径与文件引用

### 本文件关键代码

| 行号 | 代码 | 说明 |
|------|------|------|
| 1 | `import type { ChildProcess }` | 子进程类型 |
| 2 | `import { z } from 'zod/v4'` | Zod v4 验证库 |
| 3 | `import { lazySchema }` | 延迟加载工具 |
| 5-11 | `connectResponseSchema` | 连接响应验证 Schema |
| 13-24 | `ServerConfig` 类型 | 服务器配置结构 |
| 26-32 | `SessionState` 联合类型 | 会话状态枚举 |
| 33-40 | `SessionInfo` 类型 | 会话完整信息 |
| 46-55 | `SessionIndexEntry` 类型 | 会话索引条目 |
| 57 | `SessionIndex` 类型 | 索引映射类型 |

### 依赖文件

| 文件路径 | 导入内容 | 用途 |
|----------|----------|------|
| `child_process` (Node.js) | `ChildProcess` | 子进程类型 |
| `zod/v4` | `z` | Schema 定义 |
| `../utils/lazySchema.js` | `lazySchema` | 延迟加载包装器 |

### 使用方

| 文件路径 | 使用内容 | 场景 |
|----------|----------|------|
| `src/server/createDirectConnectSession.ts` | `connectResponseSchema` | 验证会话创建响应 |
| `src/cli/structuredIO.ts` | `SessionInfo`, `SessionState` | 本地服务器会话管理 |
| `src/cli/transports/ccrClient.ts` | `ServerConfig` | CCR 客户端配置 |
| `src/cli/print.ts` | `SessionIndex`, `SessionIndexEntry` | 会话持久化 |
| `src/cli/remoteIO.ts` | `SessionInfo` | 远程 IO 管理 |
| `src/bridge/replBridgeTransport.ts` | `SessionInfo` | REPL 桥接传输 |
| `src/commands/session/session.tsx` | `SessionIndexEntry` | 会话命令实现 |
| `src/services/api/logging.ts` | `SessionInfo` | 日志服务 |
| `src/hooks/useTeleportResume.tsx` | `SessionIndexEntry` | Teleport 恢复 |
| `src/utils/sdkEventQueue.ts` | `SessionInfo` | SDK 事件队列 |
| `src/bootstrap/state.ts` | `SessionIndex` | 状态管理 |
| `src/entrypoints/sdk/coreSchemas.ts` | `SessionState` | SDK Schema |

## 依赖与外部交互

### 类型依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                        types.ts                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ connectResponse │  │   ServerConfig  │  │  SessionState   │ │
│  │    Schema       │  │                 │  │                 │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                    │                    │          │
│           ↓                    ↓                    ↓          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ createDirect    │  │ 本地服务器实现   │  │ 会话生命周期管理 │ │
│  │ ConnectSession  │  │                 │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │  SessionInfo    │  │ SessionIndex    │                      │
│  │                 │  │    Entry        │                      │
│  └────────┬────────┘  └────────┬────────┘                      │
│           │                    │                               │
│           ↓                    ↓                               │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │  本地服务器进程  │  │ ~/.claude/      │                      │
│  │   管理          │  │ server-sessions │                      │
│  │                 │  │    .json        │                      │
│  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

### 持久化流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│  会话创建/更新 │────→│ SessionIndex │────→│ ~/.claude/server-    │
│              │     │   内存对象    │     │ sessions.json        │
└──────────────┘     └──────────────┘     └──────────────────────┘
                                                │
                                                ↓
                                          ┌──────────────┐
                                          │ 服务器重启时  │
                                          │ 加载恢复会话  │
                                          └──────────────┘
```

## 风险、边界与改进建议

### 已知风险

1. **Schema 版本兼容性**
   - 风险：`connectResponseSchema` 与服务器协议版本不匹配时验证失败
   - 现状：无版本号字段，无法协商协议版本
   - 建议：添加 `protocol_version` 字段支持版本协商

2. **时间戳精度**
   - 风险：`createdAt` 和 `lastActiveAt` 使用毫秒时间戳，在跨时序源比较时可能有问题
   - 建议：明确使用 `Date.now()` 或标准化为 UTC

3. **SessionKey 唯一性**
   - 风险：`sessionKey` 为可选字符串，无格式约束，可能冲突
   - 建议：添加格式验证（如 UUID）

4. **权限模式字符串类型**
   - 风险：`permissionMode?: string` 无约束，可能存储无效值
   - 建议：使用联合类型限定有效值

### 边界情况

| 场景 | 当前行为 | 建议 |
|------|----------|------|
| `work_dir` 为空字符串 | 通过验证 | 应视为无效 |
| `ws_url` 格式错误 | 仅检查为字符串 | 应验证 URL 格式 |
| `sessionId` 与 `transcriptSessionId` 不一致 | 允许 | 需文档说明使用场景 |
| `lastActiveAt` 早于 `createdAt` | 允许 | 可添加验证约束 |

### 改进建议

1. **增强 Schema 验证**
   ```typescript
   export const connectResponseSchema = lazySchema(() =>
     z.object({
       session_id: z.string().uuid(),  // 验证 UUID 格式
       ws_url: z.string().url(),       // 验证 URL 格式
       work_dir: z.string().min(1).optional(),  // 非空验证
       protocol_version: z.string().optional(), // 添加版本
     }),
   )
   ```

2. **统一时间戳类型**
   ```typescript
   // 使用 branded type 明确时间戳单位
   type TimestampMs = number & { __brand: 'TimestampMs' }
   ```

3. **权限模式类型化**
   ```typescript
   export type PermissionMode = 'auto' | 'ask' | 'bypassPermissions'
   export type SessionIndexEntry = {
     // ...
     permissionMode?: PermissionMode
   }
   ```

4. **添加索引版本号**
   ```typescript
   export type SessionIndex = {
     version: number  // 用于迁移
     sessions: Record<string, SessionIndexEntry>
   }
   ```

5. **会话索引清理策略**
   ```typescript
   // 添加清理配置
   export type SessionIndexConfig = {
     maxAgeMs: number      // 最大保留时间
     maxEntries: number    // 最大条目数
   }
   ```

### 测试建议

- **Schema 测试**: 验证各种边界输入的验证结果
- **类型测试**: 使用 `tsd` 验证类型导出正确性
- **兼容性测试**: 验证新旧版本 Schema 的前向/后向兼容
