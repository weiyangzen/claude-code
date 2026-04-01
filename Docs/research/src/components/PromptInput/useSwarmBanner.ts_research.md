# useSwarmBanner.ts 研究文档

## 场景与职责

为 PromptInput 顶部/边框区域提供 Swarm Banner 数据。根据当前身份（teammate/leader/standalone agent/named agent/--agent CLI）返回 banner 文本和背景色，让用户清楚自己正在向谁输入。

## 功能点目的

- 身份可视化：teammate > leader with teammates > viewing background agent > standalone agent > --agent CLI > null
- 环境自适应：外部 tmux 时提示 attach 命令；in-process/native pane 时显示查看的 teammate 名
- 颜色与 teammate pill、任务列表等 UI 保持一致

## 具体技术实现

### 关键流程

1. 读取 `teamContext`、`standaloneAgentContext`、`agent`、`viewingAgentTaskId` 等 AppState
2. `useEffect` 异步检测 `isInsideTmux()` 并缓存结果
3. 按优先级判断：
   - **Teammate**（非 in-process）：`@${agentName}` + 自身颜色
   - **Leader 有 teammates**：
     - 外部 tmux：`View teammates: tmux -L ${socket} a` + 查看的 teammate 颜色
     - in-process/native pane 且正在查看 teammate：`@${viewedTeammate.identity.agentName}` + 其颜色
   - **Viewing background agent**：通过 `agentNameRegistry` 反查名称，`@${name}` 或 task description + agent 颜色
   - **Standalone agent**：名称 + 颜色（无 @team）
   - **--agent CLI**：`@${agent}` + 从 `agentDefinitions` 查找的颜色
   - 都不满足 → `null`
4. `toThemeColor(colorName, fallback)` 用 `AGENT_COLORS.includes()` 校验，无效时回退

### 数据结构

```ts
type SwarmBannerInfo = { text: string; bgColor: keyof Theme } | null
```

### 协议/命令

- `isTeammate()` / `getAgentName()` / `getTeamName()` / `getTeammateColor()` → `src/utils/teammate.ts`
- `isInProcessTeammate()` → `src/utils/teammateContext.ts`
- `isInsideTmux()` / `isInProcessEnabled()` / `getCachedDetectionResult()` → `src/utils/swarm/backends/registry.ts` / `detection.ts`
- `getSwarmSocketName()` → `src/utils/swarm/constants.ts`
- `getActiveAgentForInput()` / `getViewedTeammateTask()` → `src/state/selectors.ts`
- `getStandaloneAgentName()` → `src/utils/standaloneAgent.ts`
- `getAgentColor()` / `AGENT_COLOR_TO_THEME_COLOR` → `src/tools/AgentTool/agentColorManager.ts`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/useSwarmBanner.ts` | 本 hook |
| `src/components/PromptInput/PromptInput.tsx` | 调用方，渲染 banner |
| `src/state/selectors.ts` | `getActiveAgentForInput`、`getViewedTeammateTask` |
| `src/utils/teammate.ts` | teammate 身份工具 |
| `src/utils/teammateContext.ts` | in-process teammate 检测 |
| `src/utils/swarm/backends/registry.ts` | `isInProcessEnabled`、`getCachedDetectionResult` |
| `src/utils/swarm/backends/detection.ts` | `isInsideTmux` |
| `src/utils/swarm/constants.ts` | `getSwarmSocketName` |
| `src/utils/standaloneAgent.ts` | `getStandaloneAgentName` |
| `src/tools/AgentTool/agentColorManager.ts` | 颜色映射 |
| `src/utils/theme.ts` | `Theme` 类型 |

## 依赖与外部交互

- 订阅 AppState 多字段，任何相关变化都会触发重渲染
- `isInsideTmux()` 异步，首帧 `insideTmux` 为 `null`，可能短暂 fall through
- `agentNameRegistry` 是 `Map<string, string>`，由 CoordinatorAgentStatus 维护

## 风险、边界与改进建议

1. **首帧闪烁**：`insideTmux` 初始为 `null`，leader 首帧可能无法正确判断环境，导致 banner 短暂缺失
2. **颜色静默回退**：非法颜色回退到 cyan，用户可能困惑为何颜色未生效。建议在设置颜色时就校验
3. **逻辑复杂**：涉及五种身份和三种后端环境，分支极多。建议将每种身份判定提取为独立 selector 或策略函数
4. **standalone 与 swarm 互斥**：`getStandaloneAgentName()` 内部检查 `getTeamName()`，互斥逻辑分散在多个文件
5. **测试建议**：mock `isTeammate`/`isInProcessTeammate`、mock `isInsideTmux` true/false、构造 `agentNameRegistry` 验证 named_agent 反查、验证非法颜色回退
