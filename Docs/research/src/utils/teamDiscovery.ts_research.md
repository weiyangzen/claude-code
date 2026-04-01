# teamDiscovery.ts 研究文档

## 场景与职责

`teamDiscovery.ts` 提供了团队发现和队友状态查询功能。该模块扫描 `~/.claude/teams/` 目录，发现当前会话作为 leader 的团队，并为 Teams UI 提供队友状态信息。

## 功能点目的

### 团队发现
- **目标**: 发现当前会话作为 leader 的所有团队
- **用途**: Teams UI 页脚显示团队状态
- **数据来源**: `~/.claude/teams/` 目录下的团队配置文件

### 队友状态查询
- **状态计算**: 基于 `isActive` 字段确定队友状态
- **信息丰富**: 提供名称、模型、提示词、颜色、工作状态等
- **隐藏状态**: 支持标记隐藏的 pane

## 具体技术实现

### 核心类型定义

```typescript
export type TeamSummary = {
  name: string
  memberCount: number
  runningCount: number
  idleCount: number
}

export type TeammateStatus = {
  name: string
  agentId: string
  agentType?: string
  model?: string
  prompt?: string
  status: 'running' | 'idle' | 'unknown'
  color?: string
  idleSince?: string  // ISO 时间戳
  tmuxPaneId: string
  cwd: string
  worktreePath?: string
  isHidden?: boolean  // 是否在 swarm 视图中隐藏
  backendType?: PaneBackendType
  mode?: string  // 权限模式
}
```

### 队友状态获取

```typescript
export function getTeammateStatuses(teamName: string): TeammateStatus[] {
  const teamFile = readTeamFile(teamName)
  if (!teamFile) {
    return []
  }

  const hiddenPaneIds = new Set(teamFile.hiddenPaneIds ?? [])
  const statuses: TeammateStatus[] = []

  for (const member of teamFile.members) {
    // 排除 team-lead
    if (member.name === 'team-lead') {
      continue
    }

    // 基于 isActive 确定状态
    const isActive = member.isActive !== false
    const status: 'running' | 'idle' = isActive ? 'running' : 'idle'

    statuses.push({
      name: member.name,
      agentId: member.agentId,
      agentType: member.agentType,
      model: member.model,
      prompt: member.prompt,
      status,
      color: member.color,
      tmuxPaneId: member.tmuxPaneId,
      cwd: member.cwd,
      worktreePath: member.worktreePath,
      isHidden: hiddenPaneIds.has(member.tmuxPaneId),
      backendType: isPaneBackend(member.backendType) ? member.backendType : undefined,
      mode: member.mode,
    })
  }

  return statuses
}
```

## 关键代码路径与文件引用

### 本文件导出
- `getTeammateStatuses(teamName)`: 获取队友状态列表
- `TeamSummary`, `TeammateStatus`: 类型定义

### 依赖模块

| 模块 | 用途 |
|------|------|
| `./swarm/backends/types.js` | `PaneBackendType`, `isPaneBackend` |
| `./swarm/teamHelpers.js` | `readTeamFile` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/components/teams/TeamsDialog.tsx` | Teams 对话框 UI |
| `src/components/PromptInput/PromptInput.tsx` | 提示输入组件 |

## 依赖与外部交互

### 与团队文件系统的集成
- 通过 `readTeamFile` 读取团队配置
- 团队文件路径: `~/.claude/teams/{teamName}/config.json`
- 同步读取，适用于 React 渲染路径

### 与后端类型的集成
- 使用 `isPaneBackend` 类型守卫验证后端类型
- 支持 `tmux` 和 `iterm2` 两种 pane 后端

### 与 UI 的集成
- 返回的数据直接用于 Teams UI 渲染
- `isHidden` 用于控制 pane 在 swarm 视图中的可见性
- `idleSince` 用于显示相对空闲时间（使用 `formatRelativeTimeAgo`）

## 风险、边界与改进建议

### 潜在风险

1. **同步 I/O**: `readTeamFile` 是同步的，大量团队可能影响性能
2. **文件格式变更**: 团队文件格式变更需要同步更新此模块
3. **状态延迟**: `isActive` 可能不是实时状态

### 边界情况

1. **团队不存在**: 返回空数组
2. **空团队**: 只有 team-lead 时返回空数组
3. **缺失字段**: 使用可选链和默认值处理

### 改进建议

1. **异步版本**: 添加异步 API 避免阻塞
```typescript
export async function getTeammateStatusesAsync(teamName: string): Promise<TeammateStatus[]>
```

2. **批量查询**: 支持一次查询多个团队
```typescript
export function getTeammateStatusesForTeams(teamNames: string[]): Map<string, TeammateStatus[]>
```

3. **实时更新**: 集成文件 watcher 实现实时更新
```typescript
export function subscribeToTeamChanges(teamName: string, callback: () => void): () => void
```

4. **状态聚合**: 添加团队级别统计
```typescript
export function getTeamStats(teamName: string): {
  total: number
  running: number
  idle: number
  hidden: number
}
```

5. **过滤和排序**: 支持过滤和排序选项
```typescript
export interface GetTeammateOptions {
  filter?: { status?: 'running' | 'idle'; backendType?: PaneBackendType }
  sortBy?: 'name' | 'status' | 'joinedAt'
}
```

6. **缓存机制**: 添加短期缓存减少文件读取
```typescript
const cache = new Map<string, { data: TeammateStatus[]; timestamp: number }>()
```
