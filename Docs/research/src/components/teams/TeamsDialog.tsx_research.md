# TeamsDialog.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`TeamsDialog` 是一个复杂的 React 对话框组件，位于 `src/components/teams/TeamsDialog.tsx`，用于团队负责人（team lead）查看和管理 Agent Swarm 中的队友。提供两层级的界面：队友列表视图和队友详情视图。

### 1.2 使用场景
- **查看队友列表**: 显示团队中所有队友的名称、状态、权限模式
- **查看队友详情**: 查看特定队友的任务、提示词、模型等信息
- **管理队友**: 杀死队友进程、发送关闭请求、切换显示/隐藏
- **批量操作**: 批量修剪空闲队友、批量切换权限模式
- **权限管理**: 循环切换队友的权限模式（default → acceptEdits → plan → bypassPermissions → auto）

### 1.3 调用位置
- **PromptInput.tsx** (第110行): `import { TeamsDialog } from '../teams/TeamsDialog.js'`
- **渲染位置**: 当 `showTeamsDialog` 为 true 时渲染

```tsx
{showTeamsDialog && (
  <TeamsDialog
    initialTeams={cachedTeams}
    onDone={() => setShowTeamsDialog(false)}
  />
)}
```

## 2. 功能点目的

### 2.1 核心功能模块

| 模块 | 功能描述 |
|------|----------|
| **TeamDetailView** | 队友列表视图，显示所有队友的摘要信息 |
| **TeammateListItem** | 单个队友列表项，显示名称、状态、权限模式符号 |
| **TeammateDetailView** | 队友详情视图，显示任务、提示词、模型等详细信息 |
| **权限模式循环** | 支持单个或批量切换队友权限模式 |
| **进程管理** | 杀死队友、发送优雅关闭请求 |
| **可见性控制** | 隐藏/显示队友的 tmux/iTerm2 pane |

### 2.2 键盘快捷键

| 按键 | 列表视图 | 详情视图 |
|------|----------|----------|
| ↑/↓ | 选择队友 | - |
| ← | - | 返回列表 |
| Enter | 进入详情 | 查看 pane 输出 |
| k | 杀死选中队友 | 杀死当前队友 |
| s | 发送关闭请求 | 发送关闭请求并返回 |
| h | 切换可见性 | 切换可见性并返回 |
| H | 批量隐藏/显示所有 | - |
| p | 修剪所有空闲队友 | - |
| Shift+Tab | 循环权限模式 | 循环权限模式 |
| Esc | 关闭对话框 | 返回列表 |

## 3. 具体技术实现

### 3.1 数据结构

#### DialogLevel 类型（视图层级）
```typescript
type DialogLevel = 
  | { type: 'teammateList'; teamName: string }
  | { type: 'teammateDetail'; teamName: string; memberName: string }
```

#### TeammateStatus 类型（队友状态）
```typescript
type TeammateStatus = {
  name: string;
  agentId: string;
  agentType?: string;
  model?: string;
  prompt?: string;
  status: 'running' | 'idle' | 'unknown';
  color?: string;
  idleSince?: string;
  tmuxPaneId: string;
  cwd: string;
  worktreePath?: string;
  isHidden?: boolean;
  backendType?: PaneBackendType;
  mode?: string;
}
```

#### TeamSummary 类型（团队摘要）
```typescript
type TeamSummary = {
  name: string;
  memberCount: number;
  runningCount: number;
  idleCount: number;
}
```

### 3.2 关键流程

#### 3.2.1 初始化流程
```
1. 注册为 overlay (useRegisterOverlay('teams-dialog'))
2. 从 initialTeams 获取第一个团队名称
3. 初始化 dialogLevel 为 teammateList 视图
4. 初始化 selectedIndex = 0
5. 设置定时刷新 (useInterval 1000ms)
```

#### 3.2.2 队友状态获取
```typescript
const teammateStatuses = useMemo(() => {
  return getTeammateStatuses(dialogLevel.teamName);
}, [dialogLevel.teamName, refreshKey]);
```

- 从 `teamDiscovery.ts` 的 `getTeammateStatuses()` 获取
- 读取团队配置文件 `~/.claude/teams/{teamName}/config.json`
- 排除 team-lead，只返回队友列表

#### 3.2.3 权限模式循环流程
```
single teammate:
  1. 获取当前 mode (从 teammate.mode 解析)
  2. 调用 getNextPermissionMode(context) 计算下一个模式
  3. 调用 sendModeChangeToTeammate() 发送模式变更
  4. 更新 config.json 并发送 mailbox 消息

batch teammates:
  1. 检查所有队友模式是否相同
  2. 如果不相同，重置为 'default'
  3. 如果相同，计算下一个模式
  4. 调用 setMultipleMemberModes() 批量更新
  5. 为每个队友发送 mailbox 消息
```

#### 3.2.4 杀死队友流程
```typescript
async function killTeammate(
  paneId: string,
  backendType: PaneBackendType | undefined,
  teamName: string,
  teammateId: string,
  teammateName: string,
  setAppState: (f: (prev: AppState) => AppState) => void
): Promise<void>
```

```
1. 如果 backendType 存在:
   - 调用 ensureBackendsRegistered()
   - 调用 getBackendByType(backendType).killPane(paneId, !isInsideTmuxSync())
2. 调用 removeMemberFromTeam(teamName, paneId) 从团队文件移除
3. 调用 unassignTeammateTasks() 解除任务分配
4. 更新 AppState.teamContext.teammates 移除该队友
5. 发送 inbox 通知消息
```

### 3.3 后端抽象

#### PaneBackend 接口
```typescript
type PaneBackend = {
  readonly type: BackendType;
  readonly displayName: string;
  readonly supportsHideShow: boolean;
  
  // 核心方法
  isAvailable(): Promise<boolean>;
  isRunningInside(): Promise<boolean>;
  createTeammatePaneInSwarmView(name: string, color: AgentColorName): Promise<CreatePaneResult>;
  sendCommandToPane(paneId: PaneId, command: string, useExternalSession?: boolean): Promise<void>;
  setPaneBorderColor(paneId: PaneId, color: AgentColorName, useExternalSession?: boolean): Promise<void>;
  setPaneTitle(paneId: PaneId, name: string, color: AgentColorName, useExternalSession?: boolean): Promise<void>;
  killPane(paneId: PaneId, useExternalSession?: boolean): Promise<boolean>;
  hidePane(paneId: PaneId, useExternalSession?: boolean): Promise<boolean>;
  showPane(paneId: PaneId, targetWindowOrPane: string, useExternalSession?: boolean): Promise<boolean>;
  // ...
}
```

支持的后端:
- **tmux**: 通过 tmux 命令管理 pane
- **iterm2**: 通过 it2 CLI 管理 iTerm2 分割窗格

### 3.4 文件存储结构

```
~/.claude/
├── teams/
│   └── {teamName}/
│       ├── config.json          # 团队配置（成员列表、隐藏 pane IDs）
│       └── inboxes/
│           └── {agentName}.json # 队友收件箱（消息队列）
└── tasks/
    └── {teamName}/              # 团队任务列表
        └── {taskId}.json
```

## 4. 关键代码路径与文件引用

### 4.1 直接依赖
| 文件 | 用途 |
|------|------|
| `react/compiler-runtime` | React Compiler 缓存 |
| `crypto` (randomUUID) | 生成通知消息 ID |
| `figures` | 终端图标符号 |
| `react` / `usehooks-ts` (useInterval) | React 核心和定时器 |
| `../../context/overlayContext.js` | 注册为 overlay |
| `../../ink.js` (Box, Text, useInput) | 终端 UI 组件 |
| `../../keybindings/useKeybinding.js` | 快捷键绑定 |
| `../../keybindings/useShortcutDisplay.js` | 快捷键显示 |
| `../../state/AppState.js` | 全局状态 |
| `../../utils/swarm/teamHelpers.js` | 团队操作辅助函数 |
| `../../utils/teamDiscovery.js` | 队友状态发现 |
| `../../utils/teammateMailbox.js` | 邮箱消息发送 |
| `../design-system/Dialog.js` | 对话框基础组件 |

### 4.2 核心依赖模块
| 文件 | 用途 |
|------|------|
| `src/utils/swarm/backends/registry.ts` | 后端注册和获取 |
| `src/utils/swarm/backends/types.ts` | Backend 类型定义 |
| `src/utils/swarm/backends/detection.ts` | 后端检测 |
| `src/utils/swarm/teamHelpers.ts` | 团队成员管理 |
| `src/utils/teamDiscovery.ts` | 队友状态获取 |
| `src/utils/teammateMailbox.ts` | 消息协议 |
| `src/utils/tasks.ts` | 任务管理 |
| `src/utils/permissions/PermissionMode.ts` | 权限模式定义 |
| `src/utils/permissions/getNextPermissionMode.ts` | 权限模式切换逻辑 |

### 4.3 调用方文件
| 文件 | 行号 | 用途 |
|------|------|------|
| `src/components/PromptInput/PromptInput.tsx` | 110, ~1400 | 主输入组件 |
| `src/utils/swarm/teamHelpers.ts` | 引用 TeamsDialog 类型 | 团队辅助函数 |

## 5. 依赖与外部交互

### 5.1 状态交互

#### AppState 依赖
```typescript
const setAppState = useSetAppState();
const isBypassAvailable = useAppState(s => s.toolPermissionContext.isBypassPermissionsModeAvailable);
```

#### 本地状态
```typescript
const [dialogLevel, setDialogLevel] = useState<DialogLevel>({ type: 'teammateList', teamName: firstTeamName });
const [selectedIndex, setSelectedIndex] = useState(0);
const [refreshKey, setRefreshKey] = useState(0);
```

### 5.2 外部系统集成

#### tmux 集成
```typescript
// 查看队友输出
const args = isInsideTmuxSync() 
  ? ['select-pane', '-t', paneId] 
  : ['-L', getSwarmSocketName(), 'select-pane', '-t', paneId];
await execFileNoThrow(TMUX_COMMAND, args);
```

#### iTerm2 集成
```typescript
if (backendType === 'iterm2') {
  await execFileNoThrow(IT2_COMMAND, ['session', 'focus', '-s', paneId]);
}
```

#### 文件系统交互
- **读取**: `readTeamFile(teamName)` - 读取团队配置
- **写入**: `setMemberMode()`, `removeMemberFromTeam()` - 更新团队配置
- **锁定**: 使用 `lockfile` 模块防止并发写入冲突

### 5.3 消息协议

#### 模式设置请求
```typescript
type ModeSetRequestMessage = {
  type: 'mode_set_request';
  mode: PermissionMode;
  from: string;
}
```

#### 关闭请求
```typescript
type ShutdownRequestMessage = {
  type: 'shutdown_request';
  requestId: string;
  from: string;
  reason?: string;
  timestamp: string;
}
```

## 6. 风险、边界与改进建议

### 6.1 边界情况

| 场景 | 当前行为 | 风险等级 |
|------|----------|----------|
| 无队友 | 显示 "No teammates" | 低 |
| backendType 未定义 | 跳过 pane 杀死，仅更新配置 | 中（可能遗留孤儿 pane） |
| 团队文件不存在 | getTeammateStatuses 返回空数组 | 低 |
| 并发模式更新 | 使用 batch 更新减少竞态 | 低 |
| 隐藏 pane 不支持 | 后端 supportsHideShow=false 时 h/H 无效 | 低 |

### 6.2 潜在风险

1. **竞态条件**
   - 风险: 多个队友同时更新 config.json 可能导致数据丢失
   - 缓解: 使用文件锁和批量更新 API
   - 建议: 考虑使用原子写入或数据库替代 JSON 文件

2. **Pane 泄漏**
   - 风险: backendType 未定义时无法杀死 pane
   - 代码注释: "backendType undefined: old team files predating this field, or in-process"
   - 建议: 添加定期清理孤儿 pane 的机制

3. **状态不一致**
   - 风险: UI 显示的状态与实际 pane 状态可能不同步
   - 缓解: 使用 1 秒定时刷新
   - 建议: 使用文件系统监视器实现实时更新

4. **内存泄漏**
   - 风险: useInterval 和 useEffect 清理
   - 缓解: 组件卸载时自动清理

### 6.3 改进建议

1. **实时状态同步**
   ```typescript
   // 建议: 使用文件系统监视代替轮询
   useEffect(() => {
     const watcher = fs.watch(teamFilePath, () => {
       setRefreshKey(k => k + 1);
     });
     return () => watcher.close();
   }, [teamName]);
   ```

2. **操作确认对话框**
   - 当前: k 键直接杀死队友
   - 建议: 添加确认对话框防止误操作

3. **批量操作优化**
   - 当前: p 键修剪所有空闲队友
   - 建议: 支持多选后批量操作

4. **搜索和过滤**
   - 建议: 添加队友搜索功能
   - 建议: 按状态过滤（只看空闲/运行中）

5. **历史记录**
   - 建议: 记录队友加入/离开历史
   - 建议: 显示累计运行时间统计

6. **错误处理增强**
   ```typescript
   // 建议: 添加重试机制和用户反馈
   try {
     await killTeammate(...);
   } catch (error) {
     showNotification(`Failed to kill teammate: ${error.message}`);
     // 提供手动清理指导
   }
   ```

7. **测试覆盖**
   - 建议添加测试:
     - 视图切换逻辑
     - 权限模式循环
     - 键盘快捷键处理
     - 错误边界情况

### 6.4 性能优化

| 优化点 | 当前实现 | 建议 |
|--------|----------|------|
| 状态刷新 | 1秒轮询 | 文件系统事件驱动 |
| 列表渲染 | 全量渲染 | 虚拟列表（队友数量大时） |
| 任务查询 | 每次进入详情都查询 | 缓存和增量更新 |

### 6.5 安全考虑

1. **权限检查**
   - 当前: 假设调用者是 team lead
   - 建议: 验证当前会话是否有权限管理该团队

2. **输入验证**
   - 当前: 依赖 TypeScript 类型
   - 建议: 运行时验证 teamName 和 memberName

---

*文档生成时间: 2026-04-01*
*研究范围: 代码、配置、依赖关系、外部交互*
*文件大小: ~94KB (原始代码)*
