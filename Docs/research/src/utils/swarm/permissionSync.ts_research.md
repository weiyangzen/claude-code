# permissionSync.ts 深度研究文档

## 场景与职责

`permissionSync.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**权限同步中枢模块**，负责协调工作代理（Worker Agent）与团队领导（Team Leader）之间的权限请求与响应流程。

### 核心场景

1. **工作代理权限委托**：当工作代理需要执行敏感操作（如 Bash、Edit 等工具）时，将权限请求转发给团队领导
2. **领导端权限审批**：团队领导通过 UI 审批或拒绝工作代理的权限请求
3. **沙箱网络权限**：处理沙箱运行时的网络访问权限请求（如访问外部主机）
4. **跨代理消息传递**：通过 Mailbox 系统实现权限请求和响应的异步通信

### 职责边界

- 不直接处理 UI 渲染，只提供数据层接口
- 不管理代理生命周期，只处理权限相关状态
- 支持两种存储后端：文件系统（pending/resolved 目录）和 Mailbox（消息队列）

---

## 功能点目的

### 1. 权限请求管理

**目的**：为工作代理提供标准化的权限请求创建和提交流程。

**关键功能**：
- `createPermissionRequest()`: 创建标准化的权限请求对象
- `writePermissionRequest()`: 将请求写入 pending 目录（带文件锁）
- `sendPermissionRequestViaMailbox()`: 通过 Mailbox 发送请求给领导

### 2. 权限响应处理

**目的**：让工作代理能够轮询和接收领导的权限决策。

**关键功能**：
- `readPendingPermissions()`: 领导读取所有待处理请求
- `readResolvedPermission()`: 工作代理读取已解决的请求
- `resolvePermission()`: 领导解决（批准/拒绝）权限请求
- `pollForResponse()`: 工作代理轮询响应的便捷函数

### 3. 沙箱权限管理

**目的**：处理沙箱运行时的网络访问权限（独立于普通工具权限）。

**关键功能**：
- `sendSandboxPermissionRequestViaMailbox()`: 发送沙箱网络权限请求
- `sendSandboxPermissionResponseViaMailbox()`: 领导响应沙箱权限请求
- `generateSandboxRequestId()`: 生成沙箱请求唯一 ID

### 4. 角色识别

**目的**：区分当前实例是领导还是工作代理。

**关键功能**：
- `isTeamLeader()`: 检查是否为团队领导（无 agentId 或 ID 为 'team-lead'）
- `isSwarmWorker()`: 检查是否为集群工作代理

### 5. 清理维护

**目的**：防止已解决权限文件无限累积。

**关键功能**：
- `cleanupOldResolutions()`: 清理超过指定时间的已解决权限文件（默认 1 小时）

---

## 具体技术实现

### 数据结构

#### SwarmPermissionRequest

```typescript
{
  id: string;                    // 唯一请求 ID (格式: perm-{timestamp}-{random})
  workerId: string;              // 工作代理 ID
  workerName: string;            // 工作代理名称
  workerColor?: string;          // 工作代理颜色（UI 展示）
  teamName: string;              // 团队名称
  toolName: string;              // 请求权限的工具名（如 "Bash"）
  toolUseId: string;             // 原始 toolUseID
  description: string;           // 人类可读的描述
  input: Record<string, unknown>; // 工具输入参数
  permissionSuggestions: unknown[]; // 建议的权限规则
  status: 'pending' | 'approved' | 'rejected';
  resolvedBy?: 'worker' | 'leader';
  resolvedAt?: number;
  feedback?: string;             // 拒绝时的反馈
  updatedInput?: Record<string, unknown>; // 修改后的输入
  permissionUpdates?: unknown[]; // 应用的权限更新
  createdAt: number;
}
```

#### PermissionResolution

```typescript
{
  decision: 'approved' | 'rejected';
  resolvedBy: 'worker' | 'leader';
  feedback?: string;
  updatedInput?: Record<string, unknown>;
  permissionUpdates?: PermissionUpdate[];
}
```

### 存储路径结构

```
~/.claude/teams/{teamName}/
├── permissions/
│   ├── pending/           # 待处理请求
│   │   ├── .lock         # 目录级锁文件
│   │   └── {requestId}.json
│   └── resolved/          # 已解决请求
│       └── {requestId}.json
└── inboxes/               # Mailbox 消息
    └── {agentName}.json
```

### 关键流程

#### 权限请求流程（Worker → Leader）

```
1. Worker 遇到需要权限的工具调用
2. createPermissionRequest() 创建请求对象
3. sendPermissionRequestViaMailbox() 发送给 Leader
   └── createPermissionRequestMessage() 创建消息
   └── writeToMailbox() 写入领导 Mailbox
4. Leader 的 useInboxPoller 检测到 permission_request 消息
5. 领导 UI 展示权限提示
6. 用户批准/拒绝
7. sendPermissionResponseViaMailbox() 发送响应
   └── createPermissionResponseMessage() 创建响应消息
   └── writeToMailbox() 写入 Worker Mailbox
8. Worker 轮询到响应，继续执行
```

#### 文件锁机制

```typescript
// 使用目录级锁确保原子写入
const lockFilePath = join(lockDir, '.lock');
await writeFile(lockFilePath, '', 'utf-8');
const release = await lockfile.lock(lockFilePath);
try {
  await writeFile(pendingPath, jsonStringify(request, null, 2), 'utf-8');
} finally {
  await release();
}
```

### 协议设计

#### PermissionRequestMessage（Mailbox 协议）

```typescript
{
  type: 'permission_request';
  request_id: string;
  agent_id: string;        // 工作代理名称
  tool_name: string;
  tool_use_id: string;
  description: string;
  input: Record<string, unknown>;
  permission_suggestions: unknown[];
}
```

#### PermissionResponseMessage（Mailbox 协议）

```typescript
// 成功响应
{
  type: 'permission_response';
  request_id: string;
  subtype: 'success';
  response?: {
    updated_input?: Record<string, unknown>;
    permission_updates?: unknown[];
  };
}

// 错误响应
{
  type: 'permission_response';
  request_id: string;
  subtype: 'error';
  error: string;
}
```

#### SandboxPermissionRequestMessage

```typescript
{
  type: 'sandbox_permission_request';
  requestId: string;
  workerId: string;
  workerName: string;
  workerColor?: string;
  hostPattern: { host: string };
  createdAt: number;
}
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `SwarmPermissionRequestSchema` | Zod Schema | 请求数据验证 |
| `SwarmPermissionRequest` | Type | 请求类型定义 |
| `PermissionResolution` | Type | 决议类型定义 |
| `createPermissionRequest()` | Function | 创建请求对象 |
| `writePermissionRequest()` | Function | 写入 pending 目录 |
| `readPendingPermissions()` | Function | 读取待处理请求 |
| `readResolvedPermission()` | Function | 读取已解决请求 |
| `resolvePermission()` | Function | 解决权限请求 |
| `pollForResponse()` | Function | 轮询响应 |
| `isTeamLeader()` | Function | 角色检查 |
| `isSwarmWorker()` | Function | 角色检查 |
| `sendPermissionRequestViaMailbox()` | Function | Mailbox 发送请求 |
| `sendPermissionResponseViaMailbox()` | Function | Mailbox 发送响应 |
| `sendSandboxPermissionRequestViaMailbox()` | Function | 沙箱权限请求 |
| `sendSandboxPermissionResponseViaMailbox()` | Function | 沙箱权限响应 |
| `generateSandboxRequestId()` | Function | 生成沙箱请求 ID |
| `cleanupOldResolutions()` | Function | 清理旧决议 |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` | 权限相关函数 | 处理工作代理权限提示 |
| `src/hooks/useSwarmPermissionPoller.ts` | 权限相关函数 | 领导端权限轮询 |
| `src/hooks/useInboxPoller.ts` | `sendPermissionResponseViaMailbox` | 发送权限响应 |
| `src/screens/REPL.tsx` | `isSwarmWorker`, `sendSandboxPermissionRequestViaMailbox` | 沙箱权限处理 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/utils/teammateMailbox.ts` | Mailbox 消息创建和写入 |
| `src/utils/teammate.ts` | 获取当前代理身份信息 |
| `src/utils/swarm/teamHelpers.ts` | 读取团队文件获取领导名称 |
| `src/utils/lockfile.ts` | 文件锁实现 |
| `src/utils/slowOperations.ts` | JSON 解析/序列化 |

---

## 依赖与外部交互

### 模块依赖图

```
permissionSync.ts
├── teammateMailbox.ts      # Mailbox 消息操作
├── teammate.ts             # 代理身份获取
├── teamHelpers.ts          # 团队文件读取
├── lockfile.ts             # 文件锁
├── slowOperations.ts       # JSON 操作
├── debug.ts                # 调试日志
├── log.ts                  # 错误日志
└── errors.ts               # 错误处理
```

### 环境变量依赖

| 变量 | 来源 | 用途 |
|------|------|------|
| `CLAUDE_CODE_AGENT_ID` | `teammate.ts` | 识别工作代理 ID |
| `CLAUDE_CODE_AGENT_NAME` | `teammate.ts` | 识别工作代理名称 |
| `CLAUDE_CODE_TEAM_NAME` | `teammate.ts` | 识别团队名称 |
| `CLAUDE_CODE_AGENT_COLOR` | `teammate.ts` | 获取代理颜色 |

### 外部系统集成

1. **Mailbox 系统** (`teammateMailbox.ts`)
   - 通过 `writeToMailbox()` 发送消息
   - 通过 `createPermissionRequestMessage()` 等创建结构化消息

2. **文件系统**
   - 使用 `fs/promises` 进行异步文件操作
   - 使用 `lockfile.ts` 实现并发控制

---

## 风险、边界与改进建议

### 已知风险

#### 1. 文件锁竞争

**风险**：高并发场景下多个代理同时写入 pending 目录可能导致锁竞争。

**缓解措施**：
- 使用 `lockfile.ts` 的异步锁 API
- 配置重试策略（10 次，5-100ms 退避）

**改进建议**：
```typescript
// 当前：固定重试配置
const LOCK_OPTIONS = {
  retries: { retries: 10, minTimeout: 5, maxTimeout: 100 }
};

// 建议：根据团队规模动态调整
const getLockOptions = (teamSize: number) => ({
  retries: { retries: Math.max(10, teamSize * 2), minTimeout: 5, maxTimeout: 200 }
});
```

#### 2. 权限文件累积

**风险**：resolved 目录中的文件可能无限增长，占用磁盘空间。

**缓解措施**：
- `cleanupOldResolutions()` 函数可定期清理
- 默认保留 1 小时

**改进建议**：
- 添加自动清理定时器
- 配置化保留时间

#### 3. 请求 ID 冲突

**风险**：`generateRequestId()` 使用 `Math.random()`，理论上存在冲突可能。

**当前实现**：
```typescript
return `perm-${Date.now()}-${Math.random().toString(36).substring(2, 9)}`;
```

**改进建议**：
- 使用 UUID 或添加进程 ID / 序列号

#### 4. 无超时处理

**风险**：`pollForResponse()` 可能无限期轮询，如果领导无响应。

**改进建议**：
- 添加轮询超时机制
- 支持取消请求

### 边界条件

| 场景 | 行为 |
|------|------|
| 团队名称不存在 | `getPermissionDir()` 仍返回路径，但后续操作可能失败 |
| 请求 ID 不存在 | `readResolvedPermission()` 返回 `null` |
| 文件解析失败 | 记录错误日志，返回 `null` 或空数组 |
| 并发写入同一请求 | 文件锁确保串行化，但后写入者覆盖前者 |
| 领导不存在 | `sendPermissionRequestViaMailbox()` 返回 `false` |

### 性能考虑

1. **文件 I/O**：每次权限请求涉及多次文件操作（写 pending、读 pending、写 resolved）
2. **轮询开销**：工作代理需要轮询 resolved 目录检查响应
3. **序列化成本**：使用 `jsonStringify` 进行格式化输出（2 空格缩进）

### 改进建议

#### 1. 内存缓存层

```typescript
// 添加内存缓存减少文件 I/O
const pendingCache = new Map<string, SwarmPermissionRequest>();
const resolvedCache = new Map<string, SwarmPermissionRequest>();
```

#### 2. 事件驱动替代轮询

```typescript
// 使用文件系统监听替代轮询
import { watch } from 'fs/promises';

export async function watchForResponse(requestId: string): Promise<PermissionResponse> {
  const resolvedPath = getResolvedRequestPath(teamName, requestId);
  const watcher = watch(resolvedPath);
  // ... 等待文件创建
}
```

#### 3. 批量操作支持

```typescript
// 支持批量解决权限请求
export async function resolvePermissionsBatch(
  resolutions: Array<{ requestId: string; resolution: PermissionResolution }>
): Promise<boolean[]> {
  // 单次锁内完成多个解决操作
}
```

#### 4. 请求去重

```typescript
// 检测重复请求（相同 toolUseId）
export function isDuplicateRequest(
  newRequest: SwarmPermissionRequest,
  existingRequests: SwarmPermissionRequest[]
): boolean {
  return existingRequests.some(r => 
    r.toolUseId === newRequest.toolUseId && 
    r.workerId === newRequest.workerId &&
    Date.now() - r.createdAt < 60000 // 1 分钟内视为重复
  );
}
```

### 测试建议

1. **并发测试**：模拟多个工作代理同时发送权限请求
2. **故障恢复**：测试文件系统故障后的行为
3. **性能基准**：测量高负载下的延迟和吞吐量
4. **边界测试**：测试空团队名、超长请求 ID 等边界情况
