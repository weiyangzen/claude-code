# teammateInit.ts 深度研究文档

## 场景与职责

`teammateInit.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**Teammate 初始化模块**，负责为作为 teammate 运行的 Claude Code 实例初始化必要的钩子和配置。

### 核心场景

当 Claude Code 实例作为 teammate 加入团队时，需要：

1. **应用团队级权限**：从团队文件读取 `teamAllowedPaths` 并应用到当前会话
2. **注册生命周期钩子**：在 teammate 停止时通知团队领导
3. **建立通信机制**：通过 Mailbox 系统与领导通信

### 职责边界

- 只负责初始化，不管理后续运行时行为
- 只在 teammate 模式下执行（领导跳过）
- 与 `reconnection.ts` 协作完成 teammate 启动流程

---

## 功能点目的

### 1. Teammate 钩子初始化

**函数**：`initializeTeammateHooks()`

**目的**：为作为 teammate 运行的实例注册必要的钩子。

**关键步骤**：
1. 读取团队文件获取领导 ID
2. 应用团队级允许路径（`teamAllowedPaths`）
3. 查找领导名称
4. 如果是领导则跳过（不注册钩子）
5. 注册 `Stop` 钩子以通知领导 teammate 变为空闲

### 2. 团队权限应用

**目的**：将团队配置中的允许路径应用到当前 teammate 的权限上下文。

**处理逻辑**：
- 对于绝对路径（以 `/` 开头）：添加 `//path/**` 模式
- 对于相对路径：添加 `path/**` 模式

---

## 具体技术实现

### 数据结构

#### 团队信息参数

```typescript
{
  teamName: string;    // 团队名称
  agentId: string;     // 代理 ID
  agentName: string;   // 代理名称
}
```

### 关键流程

#### 初始化流程

```
initializeTeammateHooks(setAppState, sessionId, teamInfo)
├── readTeamFile(teamName) 读取团队配置
│   └── 如果失败，记录调试日志并返回
├── 获取 leadAgentId
├── 应用 teamAllowedPaths（如果存在）
│   ├── 遍历 teamFile.teamAllowedPaths
│   ├── 构建规则内容：
│   │   ├── 绝对路径：`//${path}/**`
│   │   └── 相对路径：`${path}/**`
│   └── 通过 applyPermissionUpdate() 应用规则
├── 查找领导成员获取领导名称
├── 检查是否为领导（agentId === leadAgentId）
│   └── 是：记录日志并返回（跳过钩子注册）
└── 注册 Stop 钩子
    ├── addFunctionHook(setAppState, sessionId, 'Stop', ...)
    ├── 钩子回调：
    │   ├── setMemberActive(teamName, agentName, false) 标记为空闲
    │   ├── createIdleNotification() 创建空闲通知
    │   ├── getLastPeerDmSummary(messages) 获取最后消息摘要
    │   └── writeToMailbox(leadAgentName, notification) 发送通知
    └── 超时：10 秒
```

### 权限规则构建

```typescript
// 绝对路径处理
const ruleContent = allowedPath.path.startsWith('/')
  ? `/${allowedPath.path}/**`    // 添加 // 前缀
  : `${allowedPath.path}/**`;    // 直接使用

// 应用规则
setAppState(prev => ({
  ...prev,
  toolPermissionContext: applyPermissionUpdate(
    prev.toolPermissionContext,
    {
      type: 'addRules',
      rules: [{ toolName: allowedPath.toolName, ruleContent }],
      behavior: 'allow',
      destination: 'session',
    }
  ),
}));
```

### Stop 钩子实现

```typescript
addFunctionHook(
  setAppState,
  sessionId,
  'Stop',           // 钩子类型
  '',               // 匹配器（空表示匹配所有）
  async (messages, _signal) => {
    // 标记为空闲（fire and forget）
    void setMemberActive(teamName, agentName, false);
    
    // 创建并发送空闲通知
    const notification = createIdleNotification(agentName, {
      idleReason: 'available',
      summary: getLastPeerDmSummary(messages),
    });
    
    await writeToMailbox(leadAgentName, {
      from: agentName,
      text: jsonStringify(notification),
      timestamp: new Date().toISOString(),
      color: getTeammateColor(),
    });
    
    return true;  // 不阻止 Stop
  },
  'Failed to send idle notification to team leader',  // 错误消息
  { timeout: 10000 }  // 超时 10 秒
);
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `initializeTeammateHooks()` | Function | 初始化 teammate 钩子 |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/hooks/useSwarmInitialization.ts` | `initializeTeammateHooks` | 在 teammate 启动时调用 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/state/AppState.ts` | `AppState` 类型 |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/hooks/sessionHooks.ts` | `addFunctionHook` |
| `src/utils/permissions/PermissionUpdate.ts` | `applyPermissionUpdate` |
| `src/utils/slowOperations.ts` | `jsonStringify` |
| `src/utils/teammate.ts` | `getTeammateColor` |
| `src/utils/teammateMailbox.ts` | `createIdleNotification`, `getLastPeerDmSummary`, `writeToMailbox` |
| `src/utils/swarm/teamHelpers.ts` | `readTeamFile`, `setMemberActive` |

---

## 依赖与外部交互

### 模块依赖图

```
teammateInit.ts
├── AppState.ts              # 状态类型
├── debug.ts                 # 调试日志
├── hooks/sessionHooks.ts    # 钩子注册
├── permissions/PermissionUpdate.ts  # 权限更新
├── slowOperations.ts        # JSON 序列化
├── teammate.ts              # 代理颜色
├── teammateMailbox.ts       # Mailbox 操作
└── swarm/teamHelpers.ts     # 团队文件读取
```

### 与 useSwarmInitialization 的交互

```typescript
// useSwarmInitialization.ts
import { initializeTeammateHooks } from '../utils/swarm/teammateInit.js';

// 在 teammate 初始化流程中
export function useSwarmInitialization() {
  // ...
  
  // 如果是 teammate，初始化钩子
  if (isTeammate() && teamContext) {
    initializeTeammateHooks(
      setAppState,
      sessionId,
      {
        teamName: teamContext.teamName,
        agentId: teamContext.selfAgentId!,
        agentName: teamContext.selfAgentName,
      }
    );
  }
  
  // ...
}
```

### 与 sessionHooks 的交互

```typescript
// hooks/sessionHooks.ts
export type FunctionHook = {
  type: 'Stop' | 'Start' | string;
  matcher: string;
  handler: (messages: Message[], signal: AbortSignal) => Promise<boolean>;
  errorMessage: string;
  timeout?: number;
};

export function addFunctionHook(
  setAppState: SetAppStateFn,
  sessionId: string,
  type: string,
  matcher: string,
  handler: FunctionHook['handler'],
  errorMessage: string,
  options?: { timeout?: number }
): void;
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 团队文件读取失败

**风险**：如果团队文件在初始化时不可用，teammate 将无法应用团队权限和注册钩子。

**当前行为**：
```typescript
const teamFile = readTeamFile(teamName);
if (!teamFile) {
  logForDebugging(`[TeammateInit] Team file not found for team: ${teamName}`);
  return;  // 直接返回，不注册钩子
}
```

**改进建议**：
```typescript
// 添加重试机制
export async function initializeTeammateHooks(
  setAppState: SetAppStateFn,
  sessionId: string,
  teamInfo: { teamName: string; agentId: string; agentName: string },
  options?: { retries?: number; retryDelayMs?: number }
): Promise<void> {
  const { retries = 3, retryDelayMs = 1000 } = options ?? {};
  
  let teamFile: TeamFile | null = null;
  for (let i = 0; i < retries; i++) {
    teamFile = readTeamFile(teamInfo.teamName);
    if (teamFile) break;
    
    if (i < retries - 1) {
      logForDebugging(`[TeammateInit] Retry ${i + 1}/${retries} reading team file...`);
      await delay(retryDelayMs);
    }
  }
  
  if (!teamFile) {
    logError(new Error(`[TeammateInit] Failed to read team file after ${retries} retries`));
    // 使用默认配置继续
    teamFile = createDefaultTeamFile(teamInfo.teamName);
  }
  
  // ... 继续初始化
}
```

#### 2. 权限规则冲突

**风险**：团队级允许路径可能与用户本地配置冲突。

**当前行为**：直接应用团队规则，可能覆盖用户设置

**改进建议**：
```typescript
// 添加冲突检测
function detectPermissionConflicts(
  existingRules: PermissionRule[],
  newRules: PermissionRule[]
): Array<{ existing: PermissionRule; new: PermissionRule }> {
  const conflicts = [];
  for (const newRule of newRules) {
    const conflict = existingRules.find(r => 
      r.toolName === newRule.toolName && 
      r.ruleContent === newRule.ruleContent &&
      r.behavior !== newRule.behavior
    );
    if (conflict) conflicts.push({ existing: conflict, new: newRule });
  }
  return conflicts;
}

// 在应用前检测
const conflicts = detectPermissionConflicts(
  prev.toolPermissionContext.rules,
  teamFile.teamAllowedPaths.map(p => ({
    toolName: p.toolName,
    ruleContent: buildRuleContent(p.path),
  }))
);

if (conflicts.length > 0) {
  logForDebugging(`[TeammateInit] Detected ${conflicts.length} permission conflicts`);
}
```

#### 3. 钩子注册失败

**风险**：`addFunctionHook` 可能失败，导致领导无法收到空闲通知。

**当前保护**：钩子系统内部有错误处理

**改进建议**：
```typescript
// 验证钩子注册成功
export function initializeTeammateHooks(...): boolean {
  try {
    // ... 读取团队文件等
    
    const hookId = addFunctionHook(...);
    
    // 验证钩子已注册
    const hooks = getFunctionHooks(sessionId);
    if (!hooks.some(h => h.id === hookId)) {
      logError(new Error('[TeammateInit] Failed to register Stop hook'));
      return false;
    }
    
    logForDebugging(`[TeammateInit] Successfully registered Stop hook: ${hookId}`);
    return true;
  } catch (error) {
    logError(error);
    return false;
  }
}
```

#### 4. Mailbox 写入超时

**风险**：如果领导 Mailbox 不可用，空闲通知可能超时。

**当前保护**：钩子有 10 秒超时

**改进建议**：
```typescript
// 添加本地队列作为后备
const pendingNotifications: IdleNotificationMessage[] = [];

async function sendIdleNotificationWithRetry(
  leadAgentName: string,
  notification: IdleNotificationMessage,
  retries = 3
): Promise<void> {
  for (let i = 0; i < retries; i++) {
    try {
      await writeToMailbox(leadAgentName, ...);
      return;
    } catch (error) {
      if (i === retries - 1) {
        // 保存到本地队列，稍后重试
        pendingNotifications.push(notification);
        logForDebugging(`[TeammateInit] Queued notification for later delivery`);
      }
      await delay(1000 * (i + 1));
    }
  }
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| 团队文件不存在 | 记录调试日志，直接返回（不注册钩子） |
| `teamAllowedPaths` 为空 | 跳过权限应用，继续注册钩子 |
| 无 `teamAllowedPaths` 字段 | 视为空数组，跳过权限应用 |
| 领导调用（agentId === leadAgentId） | 记录日志并返回（跳过钩子注册） |
| 找不到领导成员 | 使用默认名称 'team-lead' |
| Stop 钩子执行失败 | 记录错误，但不阻止 Stop 事件 |
| Mailbox 写入超时 | 钩子超时（10秒），继续执行 |

### 改进建议

#### 1. 初始化状态报告

```typescript
// 返回详细的初始化结果
export type TeammateInitResult = {
  success: boolean;
  hooksRegistered: boolean;
  permissionsApplied: number;
  errors: string[];
};

export function initializeTeammateHooks(...): TeammateInitResult {
  const result: TeammateInitResult = {
    success: true,
    hooksRegistered: false,
    permissionsApplied: 0,
    errors: [],
  };
  
  try {
    // ... 初始化逻辑
    result.hooksRegistered = true;
    result.permissionsApplied = teamFile.teamAllowedPaths?.length ?? 0;
  } catch (error) {
    result.success = false;
    result.errors.push(String(error));
  }
  
  return result;
}
```

#### 2. 动态权限更新

```typescript
// 监听团队文件变化，动态更新权限
export function watchTeamPermissions(
  teamName: string,
  onUpdate: (paths: TeamAllowedPath[]) => void
): () => void {
  const watcher = watch(getTeamFilePath(teamName), async () => {
    const teamFile = await readTeamFileAsync(teamName);
    if (teamFile?.teamAllowedPaths) {
      onUpdate(teamFile.teamAllowedPaths);
    }
  });
  
  return () => watcher.close();
}
```

#### 3. 多钩子支持

```typescript
// 除了 Stop 钩子，还可以注册其他钩子
export function initializeTeammateHooks(setAppState, sessionId, teamInfo) {
  // ... 现有初始化
  
  // 注册 Start 钩子
  addFunctionHook(
    setAppState,
    sessionId,
    'Start',
    '',
    async () => {
      await setMemberActive(teamInfo.teamName, teamInfo.agentName, true);
      return true;
    },
    'Failed to mark teammate as active',
  );
  
  // 注册 Error 钩子
  addFunctionHook(
    setAppState,
    sessionId,
    'Error',
    '',
    async (messages) => {
      // 发送错误通知给领导
      await sendErrorNotification(teamInfo, messages);
      return true;
    },
    'Failed to send error notification',
  );
}
```

### 测试建议

1. **单元测试**：
   - 团队文件读取成功/失败
   - 权限规则正确构建
   - 领导检测逻辑

2. **集成测试**：
   - 与 `sessionHooks` 的集成
   - 与 `teammateMailbox` 的集成
   - 与 `teamHelpers` 的集成

3. **边界测试**：
   - 空团队文件
   - 缺失字段
   - 特殊字符路径

4. **故障测试**：
   - 文件系统错误
   - Mailbox 不可用
   - 钩子注册失败
