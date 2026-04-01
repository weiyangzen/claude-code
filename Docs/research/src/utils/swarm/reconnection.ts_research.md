# reconnection.ts 深度研究文档

## 场景与职责

`reconnection.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**会话恢复与初始化模块**，负责处理 teammate（工作代理）在两种场景下的团队上下文初始化：

1. **全新启动**：通过 CLI 参数（`--agent-id`、`--team-name` 等）初始化 teammate
2. **会话恢复**：从 transcript（会话记录）中存储的 teamName/agentName 恢复 teammate 上下文

### 核心场景

#### 场景一：全新启动（Fresh Spawns）

当用户通过 `TeammateTool` 或 `AgentTool` 创建新的 teammate 时：
- CLI 参数通过 `main.tsx` 中的 `dynamicTeamContext` 传递
- `computeInitialTeamContext()` 在首次渲染前同步计算 teamContext
- 消除对 `useEffect` 的依赖，避免渲染闪烁

#### 场景二：会话恢复（Resumed Sessions）

当用户恢复一个之前保存的会话时：
- Transcript 中存储了 `teamName` 和 `agentName`
- `initializeTeammateContextFromSession()` 从团队文件中读取信息
- 重建 `teamContext` 使心跳和其他集群功能正常工作

### 职责边界

- 只负责**初始化** teamContext，不管理后续状态更新
- 不处理 UI 渲染，只提供数据层接口
- 区分领导（leader）和 teammate 的身份识别

---

## 功能点目的

### 1. 初始团队上下文计算

**函数**：`computeInitialTeamContext()`

**目的**：在应用首次渲染前同步计算 teammate 的初始团队上下文。

**关键设计**：
- 同步执行（非异步），确保在 React 首次渲染前完成
- 从 `dynamicTeamContext` 读取 CLI 参数
- 读取团队文件验证团队存在并获取领导 ID
- 区分领导和 teammate 身份

**返回值**：
```typescript
{
  teamName: string;
  teamFilePath: string;
  leadAgentId: string;
  selfAgentId: string | undefined;  // 领导为 undefined
  selfAgentName: string;
  isLeader: boolean;
  teammates: {};  // 初始为空
}
```

### 2. 会话恢复初始化

**函数**：`initializeTeammateContextFromSession()`

**目的**：从恢复的会话中初始化 teammate 上下文。

**关键设计**：
- 异步操作（需要读取团队文件）
- 通过 `setAppState` 更新状态
- 在团队成员列表中查找 agentName 对应的成员
- 处理成员可能已被移除的情况

---

## 具体技术实现

### 数据结构

#### AppState['teamContext']

```typescript
{
  teamName: string;           // 团队名称
  teamFilePath: string;       // 团队配置文件路径
  leadAgentId: string;        // 领导代理 ID
  selfAgentId?: string;       // 自身代理 ID（领导为 undefined）
  selfAgentName: string;      // 自身代理名称
  isLeader: boolean;          // 是否为领导
  teammates: Record<string, TeammateInfo>;  // 队友信息映射
}
```

### 关键流程

#### 全新启动流程

```
1. main.tsx 解析 CLI 参数
2. setDynamicTeamContext() 设置上下文
3. computeInitialTeamContext() 被调用
   ├── getDynamicTeamContext() 获取 CLI 参数
   ├── 验证 teamName 和 agentName 存在
   ├── readTeamFile(teamName) 读取团队配置
   ├── 确定 isLeader（agentId 是否存在）
   └── 返回完整的 teamContext 对象
4. teamContext 作为 initialState 的一部分
5. 应用首次渲染时 teamContext 已就绪
```

#### 会话恢复流程

```
1. 用户恢复会话
2. transcript 中包含 teamName 和 agentName
3. initializeTeammateContextFromSession() 被调用
   ├── readTeamFile(teamName) 读取团队配置
   ├── 在 members 中查找 agentName
   ├── 获取 agentId（可能为 undefined 如果成员被移除）
   └── setAppState() 更新 teamContext
4. 心跳和其他集群功能恢复正常
```

### 身份识别逻辑

```typescript
// 领导识别：没有 agentId 或 ID 为 'team-lead'
const isLeader = !agentId;

// 或者通过 teamContext 识别
const isLeader = agentId === teamFile.leadAgentId;
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `computeInitialTeamContext()` | Function | 计算初始团队上下文（同步） |
| `initializeTeammateContextFromSession()` | Function | 从会话恢复初始化（异步） |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/main.tsx` | `computeInitialTeamContext` | 应用启动时初始化 teamContext |
| `src/hooks/useSwarmInitialization.ts` | `initializeTeammateContextFromSession`, `readTeamFile` | 会话恢复时重建上下文 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/state/AppState.ts` | `AppState` 类型定义 |
| `src/utils/teammate.ts` | `getDynamicTeamContext` 获取 CLI 参数 |
| `src/utils/swarm/teamHelpers.ts` | `getTeamFilePath`, `readTeamFile` 团队文件操作 |
| `src/utils/debug.ts` | `logForDebugging` 调试日志 |
| `src/utils/log.ts` | `logError` 错误日志 |

---

## 依赖与外部交互

### 模块依赖图

```
reconnection.ts
├── AppState.ts              # 状态类型定义
├── teammate.ts              # dynamicTeamContext 获取
├── teamHelpers.ts           # 团队文件读取
├── debug.ts                 # 调试日志
└── log.ts                   # 错误日志
```

### 与 main.tsx 的交互

```typescript
// main.tsx
import { computeInitialTeamContext } from './utils/swarm/reconnection.js';

// 解析 CLI 参数后设置 dynamicTeamContext
setDynamicTeamContext({
  agentId: parsedArgs.agentId,
  agentName: parsedArgs.agentName,
  teamName: parsedArgs.teamName,
  // ...
});

// 计算初始 teamContext
const initialTeamContext = computeInitialTeamContext();

// 作为 initialState 的一部分
const initialState: AppState = {
  // ...
  teamContext: initialTeamContext,
  // ...
};
```

### 与 useSwarmInitialization 的交互

```typescript
// useSwarmInitialization.ts
import { initializeTeammateContextFromSession } from '../utils/swarm/reconnection.js';

// 在会话恢复时调用
if (resumedSession.hasTeamContext) {
  initializeTeammateContextFromSession(
    setAppState,
    resumedSession.teamName,
    resumedSession.agentName
  );
}
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 团队文件丢失

**风险**：如果团队文件在会话恢复期间被删除，`initializeTeammateContextFromSession()` 将失败。

**当前行为**：
- 记录错误日志
- 不设置 teamContext
- teammate 将以独立模式运行

**改进建议**：
```typescript
// 添加恢复机制或优雅降级
export function initializeTeammateContextFromSession(
  setAppState: SetAppStateFn,
  teamName: string,
  agentName: string,
): void {
  const teamFile = readTeamFile(teamName);
  if (!teamFile) {
    // 当前：仅记录错误
    logError(new Error(`[initializeTeammateContextFromSession] Could not read team file...`));
    
    // 建议：尝试从备份恢复或通知用户
    notifyUserOfMissingTeamFile(teamName, agentName);
    return;
  }
  // ...
}
```

#### 2. 成员被移除后的恢复

**风险**：如果 teammate 在会话恢复前被领导从团队中移除，agentId 将为 undefined。

**当前行为**：
- 记录调试日志
- 继续初始化，但 agentId 为 undefined
- 可能导致后续操作失败

**改进建议**：
```typescript
// 明确处理成员被移除的情况
const member = teamFile.members.find(m => m.name === agentName);
if (!member) {
  logForDebugging(`[Reconnection] Member ${agentName} not found...`);
  
  // 建议：提供选项让用户选择
  // 1. 以独立模式继续
  // 2. 请求重新加入团队
  // 3. 取消会话恢复
  handleOrphanedTeammate(teamName, agentName);
  return;
}
```

#### 3. 同步文件 I/O 阻塞

**风险**：`computeInitialTeamContext()` 使用同步文件读取 `readTeamFile()`，可能阻塞启动。

**当前实现**：
```typescript
// sync IO: called from sync context
export function readTeamFile(teamName: string): TeamFile | null {
  try {
    const content = readFileSync(getTeamFilePath(teamName), 'utf-8');
    return jsonParse(content) as TeamFile;
  } catch (e) {
    // ...
  }
}
```

**权衡**：这是有意的设计选择——必须在首次渲染前完成，避免 `useEffect` 导致的闪烁。

### 边界条件

| 场景 | 行为 |
|------|------|
| 无 dynamicTeamContext | `computeInitialTeamContext()` 返回 `undefined` |
| 缺少 teamName | 返回 `undefined`，记录调试日志 |
| 缺少 agentName | 返回 `undefined`，记录调试日志 |
| 团队文件不存在 | `initializeTeammateContextFromSession()` 记录错误并返回 |
| 成员不在团队中 | agentId 为 `undefined`，记录调试日志 |
| agentId 为空字符串 | 视为领导（`isLeader = true`） |

### 改进建议

#### 1. 添加版本检查

```typescript
// 检查团队文件版本兼容性
const MIN_TEAM_FILE_VERSION = '1.0';

export function computeInitialTeamContext(): AppState['teamContext'] | undefined {
  // ...
  const teamFile = readTeamFile(teamName);
  if (teamFile?.version && semver.lt(teamFile.version, MIN_TEAM_FILE_VERSION)) {
    logError(new Error(`Team file version ${teamFile.version} is incompatible...`));
    return undefined;
  }
  // ...
}
```

#### 2. 支持部分恢复

```typescript
// 即使某些信息缺失，也尝试恢复
export function initializeTeammateContextFromSession(
  setAppState: SetAppStateFn,
  teamName: string,
  agentName: string,
): void {
  const teamFile = readTeamFile(teamName);
  
  // 即使找不到成员，也设置基本的 teamContext
  const member = teamFile?.members.find(m => m.name === agentName);
  
  setAppState(prev => ({
    ...prev,
    teamContext: {
      teamName,
      teamFilePath: getTeamFilePath(teamName),
      leadAgentId: teamFile?.leadAgentId ?? '',
      selfAgentId: member?.agentId,  // 可能为 undefined
      selfAgentName: agentName,
      isLeader: false,
      teammates: {},
      // 标记为部分恢复
      isPartiallyRestored: !member,
    },
  }));
}
```

#### 3. 添加健康检查

```typescript
// 验证恢复的 teamContext 是否有效
export function validateTeamContext(
  teamContext: AppState['teamContext']
): { valid: boolean; issues: string[] } {
  const issues: string[] = [];
  
  if (!teamContext.teamName) {
    issues.push('Missing teamName');
  }
  
  if (!teamContext.selfAgentName) {
    issues.push('Missing selfAgentName');
  }
  
  if (!teamContext.isLeader && !teamContext.selfAgentId) {
    issues.push('Non-leader missing selfAgentId');
  }
  
  return { valid: issues.length === 0, issues };
}
```

### 测试建议

1. **正常启动测试**：验证 CLI 参数正确转换为 teamContext
2. **会话恢复测试**：验证从 transcript 恢复 teamContext
3. **边界测试**：
   - 团队文件丢失
   - 成员被移除
   - 空 agentId
   - 特殊字符的团队名
4. **性能测试**：测量同步文件 I/O 对启动时间的影响
