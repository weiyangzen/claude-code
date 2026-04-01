# teamHelpers.ts 深度研究文档

## 场景与职责

`teamHelpers.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**团队管理核心模块**，提供团队文件的 CRUD 操作、成员管理、权限模式同步以及团队生命周期管理功能。

### 核心场景

1. **团队创建与配置**：创建团队文件，定义团队结构和元数据
2. **成员管理**：添加、移除、更新团队成员信息
3. **权限模式同步**：同步 teammate 的权限模式到团队文件
4. **团队清理**：会话结束时清理团队目录和 git worktree
5. **隐藏 Pane 管理**：管理 UI 中隐藏/显示的 pane

### 职责边界

- 管理团队文件的持久化存储
- 不直接处理 UI 渲染，只提供数据层接口
- 支持同步和异步两种 I/O 模式
- 处理 git worktree 的生命周期

---

## 功能点目的

### 1. 团队文件管理

**函数**：`readTeamFile()` / `readTeamFileAsync()` / `writeTeamFileAsync()`

**目的**：读写团队配置文件（`~/.claude/teams/{teamName}/config.json`）。

**设计特点**：
- 提供同步和异步两种读取方式
- 同步版本用于 React 渲染路径
- 异步版本用于工具处理器

### 2. 成员管理

**函数**：
- `removeTeammateFromTeamFile()` - 通过 agent ID 或名称移除成员
- `removeMemberFromTeam()` - 通过 tmux pane ID 移除成员
- `removeMemberByAgentId()` - 通过 agent ID 移除成员（用于进程内 teammate）

**目的**：从团队中移除成员，支持多种标识方式。

### 3. 隐藏 Pane 管理

**函数**：`addHiddenPaneId()` / `removeHiddenPaneId()`

**目的**：管理 UI 中隐藏的 pane 列表。

### 4. 权限模式管理

**函数**：
- `setMemberMode()` - 设置单个成员的权限模式
- `setMultipleMemberModes()` - 批量设置成员权限模式
- `syncTeammateMode()` - 同步当前 teammate 的权限模式到配置文件

**目的**：让团队领导能够看到并管理 teammate 的权限模式。

### 5. 活跃状态管理

**函数**：`setMemberActive()`

**目的**：标记成员为活跃（正在工作）或空闲（等待任务）。

### 6. 团队清理

**函数**：
- `cleanupTeamDirectories()` - 清理团队目录和任务目录
- `cleanupSessionTeams()` - 清理会话创建的所有团队
- `killOrphanedTeammatePanes()` - 终止孤立的 teammate pane

**目的**：会话结束时清理资源，防止文件累积和进程孤儿。

---

## 具体技术实现

### 数据结构

#### TeamFile

```typescript
{
  name: string;                    // 团队名称
  description?: string;            // 团队描述
  createdAt: number;               // 创建时间戳
  leadAgentId: string;             // 领导代理 ID
  leadSessionId?: string;          // 领导的实际会话 UUID
  hiddenPaneIds?: string[];        // 隐藏的 pane ID 列表
  teamAllowedPaths?: TeamAllowedPath[];  // 团队级允许路径
  members: Array<{
    agentId: string;               // 代理 ID（格式：name@team）
    name: string;                  // 代理名称
    agentType?: string;            // 代理类型/角色
    model?: string;                // 使用的模型
    prompt?: string;               // 初始提示
    color?: string;                // 分配的颜色
    planModeRequired?: boolean;    // 是否需要计划模式
    joinedAt: number;              // 加入时间
    tmuxPaneId: string;            // tmux pane ID
    cwd: string;                   // 工作目录
    worktreePath?: string;         // git worktree 路径
    sessionId?: string;            // 会话 ID
    subscriptions: string[];       // 订阅列表
    backendType?: BackendType;     // 后端类型
    isActive?: boolean;            // 是否活跃
    mode?: PermissionMode;         // 当前权限模式
  }>;
}
```

#### TeamAllowedPath

```typescript
{
  path: string;        // 目录路径（绝对路径）
  toolName: string;    // 适用的工具（如 "Edit"、"Write"）
  addedBy: string;     // 添加此规则的代理名称
  addedAt: number;     // 添加时间戳
}
```

### 存储路径结构

```
~/.claude/
├── teams/
│   └── {sanitizedTeamName}/
│       ├── config.json          # 团队配置文件
│       ├── permissions/         # 权限请求（由 permissionSync.ts 使用）
│       │   ├── pending/
│       │   └── resolved/
│       └── inboxes/             # Mailbox 消息
│           └── {agentName}.json
└── tasks/
    └── {sanitizedTeamName}/     # 任务存储
```

### 关键流程

#### 团队成员移除流程

```
removeTeammateFromTeamFile(teamName, identifier)
├── 验证 identifier 存在
├── readTeamFile(teamName) 读取团队文件
├── 过滤 members 数组
├── 如果长度未变，记录调试日志并返回 false
├── writeTeamFile(teamName, updatedTeamFile)
└── 返回 true
```

#### 团队清理流程

```
cleanupTeamDirectories(teamName)
├── sanitizeName(teamName) → sanitizedName
├── readTeamFile(teamName) 获取 worktree 路径
├── 遍历 members，收集 worktreePaths
├── 对每个 worktreePath 调用 destroyWorktree()
│   ├── 尝试读取 .git 文件获取主仓库路径
│   ├── 尝试 git worktree remove --force
│   └── 失败则 rm -rf
├── rm -rf ~/.claude/teams/{sanitizedTeamName}/
└── rm -rf ~/.claude/tasks/{sanitizedTeamName}/
```

#### 会话团队清理流程

```
cleanupSessionTeams()
├── getSessionCreatedTeams() 获取会话创建的团队列表
├── 如果为空，直接返回
├── 记录调试日志
├── Promise.allSettled(teams.map(killOrphanedTeammatePanes))
│   └── 动态导入 backends/registry.js 和 backends/detection.js
│   └── 对每个 pane-backed 成员调用 backend.killPane()
├── Promise.allSettled(teams.map(cleanupTeamDirectories))
└── sessionCreatedTeams.clear()
```

### 名称处理

```typescript
// 清理名称用于文件路径和 tmux 窗口名
export function sanitizeName(name: string): string {
  return name.replace(/[^a-zA-Z0-9]/g, '-').toLowerCase();
}

// 清理代理名称用于确定性 agent ID
export function sanitizeAgentName(name: string): string {
  return name.replace(/@/g, '-');  // 防止与 name@team 格式冲突
}
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `inputSchema` | Zod Schema | TeamCreateTool 输入验证 |
| `SpawnTeamOutput` / `CleanupOutput` | Type | 工具输出类型 |
| `TeamAllowedPath` / `TeamFile` | Type | 团队数据结构 |
| `sanitizeName()` | Function | 清理团队名称 |
| `sanitizeAgentName()` | Function | 清理代理名称 |
| `getTeamDir()` | Function | 获取团队目录路径 |
| `getTeamFilePath()` | Function | 获取团队文件路径 |
| `readTeamFile()` | Function | 同步读取团队文件 |
| `readTeamFileAsync()` | Function | 异步读取团队文件 |
| `writeTeamFileAsync()` | Function | 异步写入团队文件 |
| `removeTeammateFromTeamFile()` | Function | 移除团队成员 |
| `removeMemberFromTeam()` | Function | 通过 pane ID 移除成员 |
| `removeMemberByAgentId()` | Function | 通过 agent ID 移除成员 |
| `addHiddenPaneId()` / `removeHiddenPaneId()` | Function | 隐藏 pane 管理 |
| `setMemberMode()` | Function | 设置成员权限模式 |
| `setMultipleMemberModes()` | Function | 批量设置权限模式 |
| `syncTeammateMode()` | Function | 同步权限模式 |
| `setMemberActive()` | Function | 设置成员活跃状态 |
| `registerTeamForSessionCleanup()` | Function | 注册团队清理 |
| `unregisterTeamForSessionCleanup()` | Function | 取消注册团队清理 |
| `cleanupSessionTeams()` | Function | 清理会话团队 |
| `cleanupTeamDirectories()` | Function | 清理团队目录 |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/cli/print.ts` | `removeTeammateFromTeamFile` | CLI 打印模式成员移除 |
| `src/hooks/useSwarmInitialization.ts` | `readTeamFile` | 会话恢复时读取团队 |
| `src/hooks/useInboxPoller.ts` | `readTeamFileAsync` | 收件箱轮询 |
| `src/components/teams/TeamsDialog.tsx` | 多个函数 | 团队管理 UI |
| `src/screens/REPL.tsx` | `setMemberActive` | 设置成员活跃状态 |
| `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | 多个函数 | 团队删除工具 |
| `src/tools/shared/spawnMultiAgent.ts` | 多个函数 | 创建 teammate |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `readTeamFileAsync` | 发送消息 |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | `TeamFile`, 多个函数 | 团队创建工具 |
| `src/utils/teamDiscovery.ts` | `readTeamFile` | 团队发现 |
| `src/utils/attachments.ts` | `removeTeammateFromTeamFile` | 附件处理 |
| `src/utils/swarm/permissionSync.ts` | `readTeamFileAsync` | 获取领导名称 |
| `src/utils/swarm/spawnInProcess.ts` | `removeMemberByAgentId` | 终止进程内 teammate |
| `src/utils/swarm/teammateInit.ts` | `readTeamFile`, `setMemberActive` | teammate 初始化 |
| `src/components/PromptInput/PromptInput.tsx` | `syncTeammateMode` | 同步权限模式 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.ts` | `getSessionCreatedTeams` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/envUtils.ts` | `getTeamsDir` |
| `src/utils/errors.ts` | `errorMessage`, `getErrnoCode` |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrowWithCwd` |
| `src/utils/git.ts` | `gitExe` |
| `src/utils/lazySchema.ts` | `lazySchema` |
| `src/utils/permissions/PermissionMode.ts` | `PermissionMode` |
| `src/utils/slowOperations.ts` | `jsonParse`, `jsonStringify` |
| `src/utils/tasks.ts` | `getTasksDir`, `notifyTasksUpdated` |
| `src/utils/teammate.ts` | `getAgentName`, `getTeamName`, `isTeammate` |
| `src/utils/swarm/backends/types.ts` | `BackendType`, `isPaneBackend` |
| `src/utils/swarm/constants.ts` | `TEAM_LEAD_NAME` |

---

## 依赖与外部交互

### 模块依赖图

```
teamHelpers.ts
├── bootstrap/state.ts       # 会话创建的团队集合
├── debug.ts                 # 调试日志
├── envUtils.ts              # 团队目录获取
├── errors.ts                # 错误处理
├── execFileNoThrow.ts       # 命令执行
├── git.ts                   # git 命令
├── lazySchema.ts            # Zod schema
├── permissions/PermissionMode.ts  # 权限模式
├── slowOperations.ts        # JSON 操作
├── tasks.ts                 # 任务目录
├── teammate.ts              # 代理身份
├── swarm/backends/types.ts  # 后端类型
└── swarm/constants.ts       # 常量
```

### 与 gracefulShutdown 的集成

```typescript
// init.ts 中注册
import { cleanupSessionTeams } from './utils/swarm/teamHelpers.js';

gracefulShutdown.registerCleanup('cleanupSessionTeams', cleanupSessionTeams);
```

### 与 TeamCreateTool 的交互

```typescript
// TeamCreateTool.ts
import { 
  inputSchema, 
  writeTeamFileAsync, 
  sanitizeName,
  registerTeamForSessionCleanup 
} from '../../utils/swarm/teamHelpers.js';

// 创建团队文件
await writeTeamFileAsync(teamName, teamFile);
registerTeamForSessionCleanup(teamName);
```

### 与 TeamDeleteTool 的交互

```typescript
// TeamDeleteTool.ts
import { 
  cleanupTeamDirectories,
  unregisterTeamForSessionCleanup 
} from '../../utils/swarm/teamHelpers.js';

// 删除团队时清理
await cleanupTeamDirectories(teamName);
unregisterTeamForSessionCleanup(teamName);
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 同步文件 I/O 阻塞

**风险**：`readTeamFile()` 和 `writeTeamFile()` 使用同步 I/O，可能阻塞事件循环。

**当前使用场景**：
- React 渲染路径（需要同步数据）
- 清理流程（可以接受阻塞）

**改进建议**：
```typescript
// 添加缓存层减少 I/O
const teamFileCache = new Map<string, { data: TeamFile; mtime: number }>();

export function readTeamFile(teamName: string): TeamFile | null {
  const path = getTeamFilePath(teamName);
  const stats = statSync(path, { throwIfNoEntry: false });
  
  if (!stats) return null;
  
  const cached = teamFileCache.get(teamName);
  if (cached && cached.mtime === stats.mtimeMs) {
    return cached.data;
  }
  
  const content = readFileSync(path, 'utf-8');
  const data = jsonParse(content) as TeamFile;
  teamFileCache.set(teamName, { data, mtime: stats.mtimeMs });
  return data;
}
```

#### 2. Git Worktree 清理失败

**风险**：`destroyWorktree()` 可能无法正确清理 worktree，导致磁盘空间泄漏。

**当前回退策略**：
```typescript
// 尝试 git worktree remove，失败则 rm -rf
if (mainRepoPath) {
  const result = await execFileNoThrowWithCwd(...);
  if (result.code === 0) return;
}
await rm(worktreePath, { recursive: true, force: true });
```

**改进建议**：
```typescript
// 添加重试和验证
async function destroyWorktree(worktreePath: string, retries = 3): Promise<void> {
  for (let i = 0; i < retries; i++) {
    try {
      // 尝试 git worktree remove
      // ...
      
      // 验证清理成功
      const exists = await access(worktreePath).then(() => true).catch(() => false);
      if (!exists) return;
      
      // 等待后重试
      await delay(100 * (i + 1));
    } catch (error) {
      if (i === retries - 1) throw error;
    }
  }
}
```

#### 3. 并发修改冲突

**风险**：多个进程同时修改团队文件可能导致数据丢失。

**当前保护**：无显式锁机制

**改进建议**：
```typescript
// 使用文件锁
import * as lockfile from '../lockfile.js';

export async function writeTeamFileAsync(
  teamName: string, 
  teamFile: TeamFile
): Promise<void> {
  const lockPath = `${getTeamFilePath(teamName)}.lock`;
  const release = await lockfile.lock(lockPath);
  try {
    // 重新读取并合并
    const current = await readTeamFileAsync(teamName);
    const merged = mergeTeamFiles(current, teamFile);
    await writeFile(getTeamFilePath(teamName), jsonStringify(merged, null, 2));
  } finally {
    await release();
  }
}
```

#### 4. 会话清理遗漏

**风险**：如果进程异常退出，`cleanupSessionTeams()` 可能无法执行。

**当前保护**：通过 `gracefulShutdown` 注册

**改进建议**：
```typescript
// 添加定期清理孤儿团队
export async function cleanupOrphanTeams(maxAgeHours: number = 24): Promise<void> {
  const teamsDir = getTeamsDir();
  const entries = await readdir(teamsDir, { withFileTypes: true });
  
  for (const entry of entries) {
    if (!entry.isDirectory()) continue;
    
    const teamFilePath = join(teamsDir, entry.name, 'config.json');
    const stats = await stat(teamFilePath).catch(() => null);
    
    if (!stats) {
      // 无团队文件，清理目录
      await rm(join(teamsDir, entry.name), { recursive: true });
      continue;
    }
    
    const ageHours = (Date.now() - stats.mtimeMs) / (1000 * 60 * 60);
    if (ageHours > maxAgeHours) {
      await cleanupTeamDirectories(entry.name);
    }
  }
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| 团队文件不存在 | `readTeamFile()` 返回 `null` |
| 团队文件解析失败 | 记录错误，返回 `null` |
| 成员不存在 | 移除函数返回 `false` |
| pane ID 不存在 | `removeMemberFromTeam()` 返回 `false` |
| 重复添加隐藏 pane | `addHiddenPaneId()` 静默忽略 |
| 移除不存在的隐藏 pane | `removeHiddenPaneId()` 静默忽略 |
| 非 teammate 调用 `syncTeammateMode()` | 无操作（early return） |
| git worktree 不存在 | `destroyWorktree()` 静默完成 |

### 改进建议

#### 1. 团队文件版本控制

```typescript
// 添加版本号支持迁移
export type TeamFile = {
  version: string;  // 添加版本号
  name: string;
  // ...
};

const CURRENT_VERSION = '1.0';

export function migrateTeamFile(data: unknown): TeamFile {
  const file = data as Partial<TeamFile>;
  const version = file.version ?? '0.0';
  
  if (version === '0.0') {
    // 从旧版本迁移
    return { ...file, version: CURRENT_VERSION, teamAllowedPaths: [] };
  }
  
  return file as TeamFile;
}
```

#### 2. 批量操作优化

```typescript
// 批量添加成员
export async function addMembersBatch(
  teamName: string,
  members: TeamFile['members']
): Promise<void> {
  const teamFile = await readTeamFileAsync(teamName);
  if (!teamFile) throw new Error(`Team ${teamName} not found`);
  
  // 检查重复
  const existingIds = new Set(teamFile.members.map(m => m.agentId));
  const newMembers = members.filter(m => !existingIds.has(m.agentId));
  
  teamFile.members.push(...newMembers);
  await writeTeamFileAsync(teamName, teamFile);
}
```

#### 3. 审计日志

```typescript
// 记录团队变更历史
export type TeamAuditLog = {
  timestamp: number;
  action: 'member_added' | 'member_removed' | 'mode_changed' | 'path_allowed';
  actor: string;
  details: Record<string, unknown>;
};

export async function appendAuditLog(
  teamName: string,
  entry: Omit<TeamAuditLog, 'timestamp'>
): Promise<void> {
  const logPath = join(getTeamDir(teamName), 'audit.json');
  const logs: TeamAuditLog[] = await readFile(logPath)
    .then(c => jsonParse(c.toString()))
    .catch(() => []);
  
  logs.push({ ...entry, timestamp: Date.now() });
  
  // 保留最近 1000 条
  if (logs.length > 1000) logs.splice(0, logs.length - 1000);
  
  await writeFile(logPath, jsonStringify(logs, null, 2));
}
```

### 测试建议

1. **单元测试**：
   - 团队文件读写
   - 成员添加/移除
   - 权限模式同步

2. **集成测试**：
   - 与 git worktree 的集成
   - 与会话清理的集成

3. **并发测试**：
   - 同时修改团队文件
   - 同时添加/移除成员

4. **故障测试**：
   - 磁盘满
   - 权限拒绝
   - 文件损坏
