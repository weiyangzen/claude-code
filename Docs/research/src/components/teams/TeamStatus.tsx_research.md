# TeamStatus.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`TeamStatus` 是一个 React 函数组件，位于 `src/components/teams/TeamStatus.tsx`，用于在终端 UI 的底部状态栏显示当前团队的队友数量指示器。它是 Agent Swarms 功能的 UI 组成部分，让团队负责人（team lead）能够快速了解当前有多少队友正在运行。

### 1.2 使用场景
- **团队状态展示**: 当用户创建了 Agent Swarm 团队并添加了队友时，在底部状态栏显示队友数量
- **导航提示**: 当用户选中 teams pill 时，显示 "Enter to view" 提示
- **与 BackgroundTaskStatus 并列**: 类似于后台任务状态指示器，但专门用于显示队友信息

### 1.3 调用位置
- **PromptInputFooterLeftSide.tsx** (第24行): `import { TeamStatus } from '../teams/TeamStatus.js'`
- **渲染位置** (第368行): 作为 footer items 的一部分渲染

```tsx
...(isAgentSwarmsEnabled() && hasTeams ? [<TeamStatus key="teams" teamsSelected={teamsSelected} showHint={showHint && !hasBackgroundTasks} />] : [])
```

## 2. 功能点目的

### 2.1 核心功能
| 功能 | 描述 |
|------|------|
| 队友计数 | 从 `teamContext.teammates` 中统计非 team-lead 的队友数量 |
| 状态文本 | 根据数量显示单复数形式: "1 teammate" / "N teammates" |
| 选中状态 | 支持 `teamsSelected` 属性，选中时显示反色背景 |
| 提示文本 | 当 `showHint && teamsSelected` 时显示 "· Enter to view" |

### 2.2 Props 接口
```typescript
type Props = {
  teamsSelected: boolean;  // 是否被选中（用于导航）
  showHint: boolean;       // 是否显示提示文本
}
```

## 3. 具体技术实现

### 3.1 数据结构

**Teammate 数据结构** (来自 AppState.teamContext):
```typescript
type TeammateContext = {
  teamName: string;
  teammates: Record<string, {
    name: string;
    agentId: string;
    color?: string;
    // ... 其他字段
  }>;
}
```

**过滤逻辑**:
- 排除 `name === "team-lead"` 的条目（团队负责人自己）
- 只统计实际的队友数量

### 3.2 关键流程

```
1. 通过 useAppState 订阅 teamContext
2. 计算 totalTeammates:
   - teamContext ? Object.values(teamContext.teammates).filter(t => t.name !== "team-lead").length : 0
3. 如果 totalTeammates === 0，返回 null（不渲染）
4. 构建状态文本: `${totalTeammates} ${totalTeammates === 1 ? "teammate" : "teammates"}`
5. 构建提示文本（条件渲染）
6. 渲染 Text 组件，根据 teamsSelected 设置 inverse 和 key
```

### 3.3 React Compiler 优化
代码经过 React Compiler 编译，使用 `_c` (cache) 机制优化渲染性能:
- `$[0]` - `$[13]` 用于缓存计算结果和 JSX 元素
- 通过比较依赖项决定是否复用缓存

### 3.4 渲染输出示例

**未选中状态**:
```
[background color] 3 teammates
```

**选中状态** (teamsSelected=true):
```
[background color inverse] 3 teammates · Enter to view
```

## 4. 关键代码路径与文件引用

### 4.1 直接依赖
| 文件 | 用途 |
|------|------|
| `react/compiler-runtime` | React Compiler 缓存机制 |
| `react` | React 核心 |
| `../../ink.js` (Text) | 终端文本渲染组件 |
| `../../state/AppState.js` (useAppState) | 全局状态订阅 |

### 4.2 间接依赖（通过 AppState）
| 文件 | 用途 |
|------|------|
| `src/state/AppState.tsx` | 全局状态管理 |
| `src/state/AppStateStore.js` | 状态存储定义 |

### 4.3 调用方文件
| 文件 | 行号 | 用途 |
|------|------|------|
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 24, 368 | Footer 左侧状态栏 |

## 5. 依赖与外部交互

### 5.1 状态依赖
```typescript
const teamContext = useAppState(s => s.teamContext)
```

**teamContext 来源**:
- 当创建团队时由 `TeamCreateTool` 设置
- 当队友加入时由 `useTeammateHandshake` 更新
- 存储在 AppState 中，跨组件共享

### 5.2 条件渲染依赖
- `isAgentSwarmsEnabled()`: 功能开关
- `hasTeams`: 是否有团队（基于 teamContext 计算）
- `hasBackgroundTasks`: 是否有后台任务（影响 hint 显示）

### 5.3 与 Footer 导航的集成
- `teamsSelected`: 来自 AppState 的 `footerSelection === 'teams'`
- 通过 `selectFooterItem('teams')` 设置选中状态
- 使用 ↓/→ 键在 footer items 间导航

## 6. 风险、边界与改进建议

### 6.1 边界情况

| 场景 | 行为 |
|------|------|
| 无团队 | 返回 null，不渲染任何内容 |
| 只有 team-lead | 返回 null（team-lead 被过滤） |
| teamContext 为 undefined | 显示 0 teammates |
| 队友被隐藏 | 仍计入总数（只按 teammates 对象计数） |

### 6.2 潜在风险

1. **状态同步延迟**
   - 风险: teamContext 更新可能有延迟，导致计数不准确
   - 缓解: AppState 使用同步更新，延迟最小

2. **内存泄漏**
   - 风险: useAppState 订阅未正确清理
   - 缓解: React 的 useSyncExternalStore 自动处理清理

3. **性能问题**
   - 风险: 每次 teamContext 变化都重新计算
   - 缓解: React Compiler 缓存优化，且计算复杂度为 O(n)

### 6.3 改进建议

1. **显示更多信息**
   ```typescript
   // 建议: 显示活跃/空闲队友数量
   const activeCount = teammates.filter(t => t.status === 'running').length;
   const idleCount = teammates.filter(t => t.status === 'idle').length;
   // 显示: "3 teammates (2 active, 1 idle)"
   ```

2. **点击交互**
   - 当前: 仅显示提示 "Enter to view"
   - 建议: 支持鼠标点击打开 TeamsDialog

3. **颜色区分**
   - 当前: 统一使用 background 颜色
   - 建议: 根据队友状态改变颜色（如所有队友空闲时变暗）

4. **测试覆盖**
   - 建议添加单元测试:
     - 无团队时返回 null
     - 正确过滤 team-lead
     - 单复数文本正确
     - 选中状态样式正确

### 6.4 相关配置

**功能开关**:
- `isAgentSwarmsEnabled()`: 控制是否启用 Agent Swarms 功能

**环境变量**:
- `CLAUDE_CODE_TEAM_NAME`: 当前团队名称
- `CLAUDE_CODE_AGENT_NAME`: 当前代理名称

---

*文档生成时间: 2026-04-01*
*研究范围: 代码、配置、依赖关系*
