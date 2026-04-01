# WorkerPendingPermission.tsx 研究文档

## 场景与职责

`WorkerPendingPermission.tsx` 是 Claude Code CLI 多智能体（Swarm）系统中的**等待状态指示组件**。当团队成员（teammate/worker）发起权限请求并等待团队负责人（team lead）审批时，该组件在工作者的界面上显示一个视觉指示器，告知用户当前正在等待权限审批。

该组件解决了多智能体协作场景中的**状态透明性问题**：
- 工作者需要向用户表明它正在等待权限审批，而不是卡死或无响应
- 显示正在等待的工具名称和操作描述
- 提供团队上下文信息（团队名称、工作者身份）

## 功能点目的

### 1. 等待状态可视化
- 显示旋转的 Spinner 动画
- 显示"Waiting for team lead approval"提示文本
- 使用警告色（warning）边框强调等待状态

### 2. 工作者身份展示
- 显示工作者徽章（WorkerBadge）
- 标识当前是哪个工作者在等待审批

### 3. 操作信息展示
- 显示正在等待的工具名称（Tool）
- 显示操作描述（Action）
- 显示权限请求发送的目标团队

## 具体技术实现

### 核心数据结构

```typescript
// 组件 Props
type Props = {
  toolName: string;      // 工具名称（如 "Bash", "FileEdit"）
  description: string;   // 操作描述
};

// 从 teammate 工具获取的信息
import { getAgentName, getTeammateColor, getTeamName } from '../../utils/teammate.js';
```

### 渲染结构

```
Box (flexDirection="column", borderStyle="round", borderColor="warning")
  ├── Box (marginBottom={1})
  │     ├── Spinner
  │     └── Text (color="warning", bold) "Waiting for team lead approval"
  ├── Box (marginBottom={1}) [条件渲染]
  │     └── WorkerBadge (name={agentName}, color={agentColor})
  ├── Box
  │     ├── Text (dimColor) "Tool: "
  │     └── Text {toolName}
  ├── Box
  │     ├── Text (dimColor) "Action: "
  │     └── Text {description}
  └── Box (marginTop={1}) [条件渲染]
        └── Text (dimColor) "Permission request sent to team \"{teamName}\" leader"
```

### 关键代码路径

```typescript
// 团队成员工具
import { 
  getAgentName,      // 获取当前工作者名称
  getTeammateColor,  // 获取当前工作者颜色
  getTeamName        // 获取团队名称
} from '../../utils/teammate.js';

// UI 组件
import { Box, Text } from '../../ink.js';
import { Spinner } from '../Spinner.js';
import { WorkerBadge } from './WorkerBadge.js';
```

### React Compiler 优化

组件使用 React Compiler 的缓存机制：
- 缓存 `teamName`、`agentName`、`agentColor` 的获取结果
- 缓存静态 JSX 元素（Spinner + 等待文本）
- 缓存条件渲染的 WorkerBadge
- 缓存所有标签文本元素

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `Box`, `Text` | `../../ink.js` | Ink UI 组件 |
| `getAgentName` | `../../utils/teammate.js` | 获取工作者名称 |
| `getTeammateColor` | `../../utils/teammate.js` | 获取工作者颜色 |
| `getTeamName` | `../../utils/teammate.js` | 获取团队名称 |
| `Spinner` | `../Spinner.js` | 加载动画 |
| `WorkerBadge` | `./WorkerBadge.js` | 工作者徽章 |

### Teammate 工具详解

`src/utils/teammate.ts` 提供了团队成员身份管理功能：

```typescript
// 动态团队上下文（运行时加入团队）
let dynamicTeamContext: {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId?: string
} | null = null

// 优先级：AsyncLocalStorage（进程内）> dynamicTeamContext（tmux）
export function getAgentName(): string | undefined {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.agentName
  return dynamicTeamContext?.agentName
}

export function getTeammateColor(): string | undefined {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.color
  return dynamicTeamContext?.color
}

export function getTeamName(teamContext?: { teamName: string }): string | undefined {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.teamName
  if (dynamicTeamContext?.teamName) return dynamicTeamContext.teamName
  return teamContext?.teamName
}
```

### 被调用方

该组件在多智能体权限流程中被调用：

```
工作者发起工具调用
  ↓
权限系统检测到需要审批
  ↓
权限请求发送到团队负责人
  ↓
工作者界面渲染 WorkerPendingPermission
  ↓
等待负责人响应...
  ↓
收到响应后继续/取消操作
```

具体调用位置通常在：
- 工作者的工具执行循环中
- 权限拦截器（permission interceptor）中

### 数据流

```
工作者进程
  ↓
Teammate 初始化（设置 dynamicTeamContext）
  ↓
工具调用触发权限检查
  ↓
权限系统
  ↓
WorkerPendingPermission (toolName, description)
  ↓
getAgentName() / getTeammateColor() / getTeamName()
  ↓
渲染等待界面
```

## 风险、边界与改进建议

### 当前风险

1. **团队信息依赖运行时状态**：
   - `dynamicTeamContext` 是运行时设置的
   - 如果设置失败或时机不对，组件可能无法显示正确的团队信息

2. **硬编码的英文文本**：
   - 所有 UI 文本都是硬编码的
   - 不支持国际化

3. **无限等待风险**：
   - 组件本身没有超时机制
   - 如果负责人永远不响应，工作者将永远等待

### 边界情况

1. **非团队成员环境**：
   - 如果 `getAgentName()` 和 `getTeammateColor()` 都返回 undefined
   - WorkerBadge 不会渲染（条件渲染）
   - 但等待文本仍然显示

2. **空团队名称**：
   - 如果 `getTeamName()` 返回 undefined
   - 最后一行提示不会渲染

3. **长文本截断**：
   - `toolName` 和 `description` 可能很长
   - Ink 的 Text 组件会自动处理，但可能影响布局

### 改进建议

1. **添加超时机制**：
   ```typescript
   // 添加超时提示
   const [waitTime, setWaitTime] = useState(0);
   useEffect(() => {
     const interval = setInterval(() => setWaitTime(t => t + 1), 1000);
     return () => clearInterval(interval);
   }, []);
   
   // 显示等待时间
   {waitTime > 30 && (
     <Text dimColor>Waiting for {waitTime}s...</Text>
   )}
   ```

2. **取消功能**：
   - 添加取消按钮，允许用户主动取消等待
   - 避免无限期阻塞

3. **国际化支持**：
   - 提取所有字符串到翻译文件
   - 支持动态语言切换

4. **重试机制**：
   - 如果请求超时，提供重试选项
   - 或者自动重试一定次数

5. **负责人信息**：
   - 显示当前负责人的名称（如果已知）
   - 帮助用户知道谁在审批

6. **队列位置**：
   - 如果有多个权限请求排队，显示当前位置
   - "Position 2 of 5 in queue"

7. **通知集成**：
   - 当权限被批准/拒绝时发送桌面通知
   - 用户可以在等待时做其他事情

8. **进度指示**：
   - 如果可能，显示审批进度
   - 如负责人正在查看相关信息

9. **视觉改进**：
   - 考虑使用不同的边框样式区分不同类型的等待
   - 添加微妙的动画效果吸引注意力
