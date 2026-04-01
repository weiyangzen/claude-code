# types.ts 深度研究文档

## 场景与职责

types.ts 是 Agent Swarm 后端系统的 **核心类型定义模块**，定义了所有后端相关的接口、类型和类型守卫。它是整个后端架构的基础，确保各模块之间的类型安全和接口一致性。

**核心定位：**
- 后端系统的类型单一可信源
- PaneBackend 和 TeammateExecutor 接口定义
- 配置和消息类型定义
- 类型守卫函数

**类型体系：**
```
BackendType ('tmux' | 'iterm2' | 'in-process')
    ├── PaneBackendType ('tmux' | 'iterm2')
    │       └── PaneBackend (接口)
    │               ├── TmuxBackend (实现)
    │               └── ITermBackend (实现)
    │
    └── TeammateExecutor (接口)
            ├── InProcessBackend (实现)
            └── PaneBackendExecutor (包装 PaneBackend)
```

---

## 功能点目的

### 1. 后端类型定义
- `BackendType`: 所有后端类型的联合类型
- `PaneBackendType`: 窗口后端类型的子集
- `PaneId`: 窗口标识符类型

### 2. PaneBackend 接口
- 定义窗口管理后端的完整接口
- 包含创建、样式、生命周期管理等方法
- 被 TmuxBackend 和 ITermBackend 实现

### 3. TeammateExecutor 接口
- 定义队友执行器的统一接口
- 抽象 PaneBackend 和 InProcessBackend 的差异
- 被 PaneBackendExecutor 和 InProcessBackend 实现

### 4. 配置和消息类型
- `TeammateSpawnConfig`: 队友创建配置
- `TeammateSpawnResult`: 队友创建结果
- `TeammateMessage`: 队友间消息格式
- `TeammateIdentity`: 队友身份信息

### 5. 类型守卫
- `isPaneBackend()`: 检查是否为窗口后端类型

---

## 具体技术实现

### 关键类型定义

#### 1. 后端类型

```typescript
// 所有后端类型
export type BackendType = 'tmux' | 'iterm2' | 'in-process'

// 窗口后端子集
export type PaneBackendType = 'tmux' | 'iterm2'

// 窗口标识符（tmux: %1, iTerm2: session UUID）
export type PaneId = string
```

#### 2. 窗口创建结果

```typescript
export type CreatePaneResult = {
  paneId: PaneId           // 新窗口的 ID
  isFirstTeammate: boolean // 是否为首个队友（影响布局）
}
```

#### 3. PaneBackend 接口

```typescript
export type PaneBackend = {
  // 元数据
  readonly type: BackendType
  readonly displayName: string
  readonly supportsHideShow: boolean  // 是否支持隐藏/显示

  // 可用性检测
  isAvailable(): Promise<boolean>
  isRunningInside(): Promise<boolean>

  // 窗口创建
  createTeammatePaneInSwarmView(
    name: string,
    color: AgentColorName,
  ): Promise<CreatePaneResult>

  // 命令执行
  sendCommandToPane(
    paneId: PaneId,
    command: string,
    useExternalSession?: boolean,
  ): Promise<void>

  // 视觉样式
  setPaneBorderColor(
    paneId: PaneId,
    color: AgentColorName,
    useExternalSession?: boolean,
  ): Promise<void>

  setPaneTitle(
    paneId: PaneId,
    name: string,
    color: AgentColorName,
    useExternalSession?: boolean,
  ): Promise<void>

  enablePaneBorderStatus(
    windowTarget?: string,
    useExternalSession?: boolean,
  ): Promise<void>

  // 布局管理
  rebalancePanes(
    windowTarget: string,
    hasLeader: boolean,
  ): Promise<void>

  // 生命周期
  killPane(paneId: PaneId, useExternalSession?: boolean): Promise<boolean>
  hidePane(paneId: PaneId, useExternalSession?: boolean): Promise<boolean>
  showPane(
    paneId: PaneId,
    targetWindowOrPane: string,
    useExternalSession?: boolean,
  ): Promise<boolean>
}
```

**设计决策：**

- `useExternalSession`: tmux 特有，用于区分用户会话和 swarm 会话
- `supportsHideShow`: iTerm2 不支持，因此需要标志位
- 所有方法返回 Promise，统一异步接口

#### 4. 检测结果类型

```typescript
export type BackendDetectionResult = {
  backend: PaneBackend
  isNative: boolean        // 是否在原生环境中运行
  needsIt2Setup?: boolean  // iTerm2 特有：是否需要 it2 设置
}
```

#### 5. 队友身份信息

```typescript
export type TeammateIdentity = {
  name: string
  teamName: string
  color?: AgentColorName
  planModeRequired?: boolean
}
```

#### 6. 队友创建配置

```typescript
export type TeammateSpawnConfig = TeammateIdentity & {
  prompt: string                    // 初始提示
  cwd: string                       // 工作目录
  model?: string                    // 模型覆盖
  systemPrompt?: string             // 系统提示
  systemPromptMode?: 'default' | 'replace' | 'append'
  worktreePath?: string             // Git worktree 路径
  parentSessionId: string           // 父会话 ID
  permissions?: string[]            // 工具权限
  allowPermissionPrompts?: boolean  // 是否允许权限提示
}
```

#### 7. 队友创建结果

```typescript
export type TeammateSpawnResult = {
  success: boolean
  agentId: string
  error?: string
  abortController?: AbortController  // in-process 特有
  taskId?: string                    // in-process 特有
  paneId?: PaneId                    // pane-based 特有
}
```

**注意：** `abortController` 和 `taskId` 是 in-process 特有，`paneId` 是 pane-based 特有。调用方需要根据实际情况处理。

#### 8. 队友消息格式

```typescript
export type TeammateMessage = {
  text: string
  from: string
  color?: string
  timestamp?: string
  summary?: string  // 5-10 字摘要，用于 UI 预览
}
```

#### 9. TeammateExecutor 接口

```typescript
export type TeammateExecutor = {
  readonly type: BackendType

  isAvailable(): Promise<boolean>
  spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult>
  sendMessage(agentId: string, message: TeammateMessage): Promise<void>
  terminate(agentId: string, reason?: string): Promise<boolean>
  kill(agentId: string): Promise<boolean>
  isActive(agentId: string): Promise<boolean>
}
```

**与 PaneBackend 的区别：**

| 特性 | PaneBackend | TeammateExecutor |
|-----|-------------|------------------|
| 抽象层级 | 低（窗口操作） | 高（队友生命周期） |
| 实现 | TmuxBackend, ITermBackend | InProcessBackend, PaneBackendExecutor |
| 标识 | PaneId | AgentId |
| 使用方 | PaneBackendExecutor | TeammateTool |

#### 10. 类型守卫

```typescript
export function isPaneBackend(type: BackendType): type is 'tmux' | 'iterm2' {
  return type === 'tmux' || type === 'iterm2'
}
```

**用途：**

```typescript
const type: BackendType = getBackendType()

if (isPaneBackend(type)) {
  // TypeScript 知道这里 type 是 'tmux' | 'iterm2'
  // 可以安全地调用 PaneBackend 特有方法
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/tools/AgentTool/agentColorManager.ts` | `AgentColorName` 类型 |

### 关键代码位置

- **后端类型**: 行 9-15
- **PaneId 和创建结果**: 行 18-32
- **PaneBackend 接口**: 行 39-168
- **检测结果类型**: 行 173-180
- **队友身份信息**: 行 191-200
- **创建配置**: 行 205-225
- **创建结果**: 行 230-254
- **消息类型**: 行 259-270
- **TeammateExecutor 接口**: 行 279-300
- **类型守卫**: 行 306-311

---

## 依赖与外部交互

### 与 AgentColorManager 的交互

```typescript
import type { AgentColorName } from '../../../tools/AgentTool/agentColorManager.js'
```

颜色定义集中管理，确保 UI 一致性。

### 类型使用模式

**实现接口：**

```typescript
// TmuxBackend.ts
export class TmuxBackend implements PaneBackend {
  readonly type = 'tmux' as const
  readonly displayName = 'tmux'
  readonly supportsHideShow = true
  
  async createTeammatePaneInSwarmView(...): Promise<CreatePaneResult> {
    // 实现
  }
  // ...
}
```

**使用接口：**

```typescript
// PaneBackendExecutor.ts
export class PaneBackendExecutor implements TeammateExecutor {
  constructor(private backend: PaneBackend) {}
  
  async spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult> {
    const { paneId } = await this.backend.createTeammatePaneInSwarmView(...)
    // ...
  }
}
```

---

## 类型架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         BackendType                              │
│              ('tmux' | 'iterm2' | 'in-process')                  │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
          ▼                                       ▼
┌─────────────────────┐                 ┌─────────────────────┐
│   PaneBackendType   │                 │   (in-process)      │
│  ('tmux' | 'iterm2')│                 │                     │
└─────────────────────┘                 └─────────────────────┘
          │                                       │
          ▼                                       ▼
┌─────────────────────┐                 ┌─────────────────────┐
│    PaneBackend      │                 │  TeammateExecutor   │
│     (interface)     │                 │     (interface)     │
├─────────────────────┤                 ├─────────────────────┤
│ + type              │                 │ + type              │
│ + displayName       │                 │ + isAvailable()     │
│ + supportsHideShow  │                 │ + spawn()           │
│ + isAvailable()     │                 │ + sendMessage()     │
│ + createTeammate... │                 │ + terminate()       │
│ + sendCommand...    │                 │ + kill()            │
│ + setPaneBorder...  │                 │ + isActive()        │
│ + killPane()        │                 └─────────────────────┘
│ + hidePane()        │                           │
│ + showPane()        │                           │
└─────────────────────┘                           │
          │                                       │
    ┌─────┴─────┐                                 │
    │           │                                 │
    ▼           ▼                                 ▼
┌────────┐ ┌────────┐                    ┌─────────────────┐
│ Tmux   │ │ ITerm  │                    │ InProcessBackend│
│Backend │ │Backend │                    │                 │
└────────┘ └────────┘                    └─────────────────┘
                                                  │
                                                  │ wraps
                                                  ▼
                                         ┌─────────────────┐
                                         │PaneBackendExecutor
                                         │                 │
                                         │ - backend:      │
                                         │   PaneBackend   │
                                         └─────────────────┘
```

---

## 风险、边界与改进建议

### 已知风险

1. **类型兼容性**
   - `TeammateSpawnResult` 的字段是可选的，调用方需要知道实际类型
   - **缓解**: 通过 `type` 字段区分，或使用类型守卫

2. **接口膨胀**
   - PaneBackend 接口有 12 个方法，可能过于庞大
   - **影响**: 新后端实现成本高

3. **可选参数复杂性**
   - `useExternalSession` 是 tmux 特有，但出现在通用接口中
   - **影响**: iTerm2 实现需要忽略该参数

4. **类型与实现漂移**
   - 接口变更需要同步更新所有实现
   - **缓解**: TypeScript 编译时检查

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| 未知 BackendType | TypeScript 编译错误 |
| PaneBackend 方法未实现 | 运行时错误或空实现 |
| 可选字段缺失 | 使用类型守卫或默认值 |

### 改进建议

1. **接口拆分**
   - 将 PaneBackend 拆分为更小的接口
   - 例如：PaneLifecycle、PaneStyling、PaneLayout

2. **更精确的类型**
   - 为不同后端类型定义特定的 SpawnResult
   - 使用 discriminated union：

```typescript
type TeammateSpawnResult = 
  | { type: 'in-process'; abortController: AbortController; taskId: string }
  | { type: 'pane'; paneId: PaneId }
  | { type: 'error'; error: string }
```

3. **移除 tmux 特有参数**
   - 将 `useExternalSession` 从接口中移除
   - 通过构造函数或配置传递

4. **添加文档注释**
   - 为每个接口方法添加 JSDoc
   - 说明参数含义和返回值

5. **运行时类型检查**
   - 添加 io-ts 或 zod 进行运行时验证
   - 捕获配置错误

6. **版本控制**
   - 为接口添加版本号
   - 支持向后兼容的演进
