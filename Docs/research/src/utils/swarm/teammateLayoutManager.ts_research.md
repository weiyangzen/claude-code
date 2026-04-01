# teammateLayoutManager.ts 深度研究文档

## 场景与职责

`teammateLayoutManager.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**Teammate UI 布局管理模块**，负责管理 teammate 的颜色分配、pane 创建和终端后端交互。

### 核心场景

1. **颜色分配**：为每个 teammate 分配唯一的 UI 颜色（用于 pane 边框、消息标识等）
2. **Pane 管理**：在集群视图中创建和管理 teammate pane
3. **后端抽象**：自动检测并使用合适的终端后端（tmux 或 iTerm2）

### 职责边界

- 只负责 UI 布局和外观管理
- 不管理 teammate 生命周期（创建/终止由其他模块处理）
- 通过 `backends/registry.js` 自动检测和委托后端操作

---

## 功能点目的

### 1. 颜色分配

**函数**：`assignTeammateColor()` / `getTeammateColor()` / `clearTeammateColors()`

**目的**：为 teammate 分配和追踪 UI 颜色。

**分配策略**：
- 轮询（round-robin）从预定义调色板分配
- 每个 teammate ID 只分配一次颜色（缓存）
- 支持清除所有分配（团队清理时使用）

### 2. Pane 创建

**函数**：`createTeammatePaneInSwarmView()`

**目的**：在集群视图中为 teammate 创建新的 pane。

**布局策略**：
- **在 tmux 中运行**：分割当前窗口（领导在左 30%，teammates 在右 70%）
- **在 iTerm2 中运行（有 it2 CLI）**：使用原生 iTerm2 分割 pane
- **在 tmux/iTerm2 外运行**：创建外部 `claude-swarm` 会话

### 3. Pane 边框状态

**函数**：`enablePaneBorderStatus()`

**目的**：启用 pane 边框状态显示（在边框中显示 pane 标题）。

### 4. 命令发送

**函数**：`sendCommandToPane()`

**目的**：向特定 pane 发送命令执行。

---

## 具体技术实现

### 数据结构

#### 颜色分配状态

```typescript
// 模块级状态（每会话）
const teammateColorAssignments = new Map<string, AgentColorName>();
let colorIndex = 0;
```

#### AgentColorName

来自 `agentColorManager.js` 的预定义颜色调色板。

### 关键流程

#### 颜色分配流程

```
assignTeammateColor(teammateId)
├── 检查 teammateColorAssignments.has(teammateId)
│   └── 存在：返回已分配的颜色
└── 不存在：
    ├── AGENT_COLORS[colorIndex % AGENT_COLORS.length] 获取颜色
    ├── teammateColorAssignments.set(teammateId, color)
    ├── colorIndex++
    └── 返回颜色
```

#### Pane 创建流程

```
createTeammatePaneInSwarmView(teammateName, teammateColor)
├── detectAndGetBackend() 获取后端
│   └── 内部缓存，自动检测 tmux/iTerm2
└── backend.createTeammatePaneInSwarmView(name, color)
    ├── TmuxBackend：分割窗口
    ├── ITermBackend：iTerm2 原生分割
    └── 其他：外部会话
```

### 后端检测与委托

```typescript
// 获取后端（带缓存）
async function getBackend(): Promise<PaneBackend> {
  return (await detectAndGetBackend()).backend;
}

// 所有操作委托给检测到的后端
export async function createTeammatePaneInSwarmView(
  teammateName: string,
  teammateColor: AgentColorName,
): Promise<{ paneId: string; isFirstTeammate: boolean }> {
  const backend = await getBackend();
  return backend.createTeammatePaneInSwarmView(teammateName, teammateColor);
}
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `assignTeammateColor()` | Function | 为 teammate 分配颜色 |
| `getTeammateColor()` | Function | 获取已分配的颜色 |
| `clearTeammateColors()` | Function | 清除所有颜色分配 |
| `isInsideTmux()` | Function | 检查是否在 tmux 中运行 |
| `createTeammatePaneInSwarmView()` | Function | 创建 teammate pane |
| `enablePaneBorderStatus()` | Function | 启用 pane 边框状态 |
| `sendCommandToPane()` | Function | 向 pane 发送命令 |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | `assignTeammateColor` | 为 teammate 分配颜色 |
| `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | `clearTeammateColors` | 团队删除时清除颜色 |
| `src/tools/shared/spawnMultiAgent.ts` | 多个函数 | 创建 pane、发送命令 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/agentColorManager.ts` | `AgentColorName`, `AGENT_COLORS` |
| `src/utils/swarm/backends/registry.js` | `detectAndGetBackend` |
| `src/utils/swarm/backends/types.ts` | `PaneBackend` |

---

## 依赖与外部交互

### 模块依赖图

```
teammateLayoutManager.ts
├── AgentTool/agentColorManager.ts  # 颜色调色板
├── swarm/backends/registry.js      # 后端检测
└── swarm/backends/types.ts         # 后端类型
```

### 与 Backends 的交互

```typescript
// backends/types.ts
export type PaneBackend = {
  readonly type: BackendType;
  readonly displayName: string;
  readonly supportsHideShow: boolean;
  
  isAvailable(): Promise<boolean>;
  isRunningInside(): Promise<boolean>;
  createTeammatePaneInSwarmView(name: string, color: AgentColorName): Promise<CreatePaneResult>;
  sendCommandToPane(paneId: PaneId, command: string, useExternalSession?: boolean): Promise<void>;
  setPaneBorderColor(paneId: PaneId, color: AgentColorName, useExternalSession?: boolean): Promise<void>;
  setPaneTitle(paneId: PaneId, name: string, color: AgentColorName, useExternalSession?: boolean): Promise<void>;
  enablePaneBorderStatus(windowTarget?: string, useExternalSession?: boolean): Promise<void>;
  rebalancePanes(windowTarget: string, hasLeader: boolean): Promise<void>;
  killPane(paneId: PaneId, useExternalSession?: boolean): Promise<boolean>;
  hidePane(paneId: PaneId, useExternalSession?: boolean): Promise<boolean>;
  showPane(paneId: PaneId, targetWindowOrPane: string, useExternalSession?: boolean): Promise<boolean>;
};
```

### 与 spawnMultiAgent.ts 的交互

```typescript
// spawnMultiAgent.ts
import {
  assignTeammateColor,
  createTeammatePaneInSwarmView,
  enablePaneBorderStatus,
  sendCommandToPane,
} from '../../utils/swarm/teammateLayoutManager.js';

async function handleSpawnSplitPane(input, context) {
  // 分配颜色
  const teammateColor = assignTeammateColor(teammateId);
  
  // 创建 pane
  const { paneId, isFirstTeammate } = await createTeammatePaneInSwarmView(
    sanitizedName,
    teammateColor,
  );
  
  // 启用边框状态
  if (isFirstTeammate && insideTmux) {
    await enablePaneBorderStatus();
  }
  
  // 发送创建命令
  await sendCommandToPane(paneId, spawnCommand, !insideTmux);
}
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 颜色分配耗尽

**风险**：如果创建大量 teammate，可能耗尽预定义调色板的颜色。

**当前行为**：使用模运算循环使用颜色
```typescript
const color = AGENT_COLORS[colorIndex % AGENT_COLORS.length]!;
```

**改进建议**：
```typescript
// 动态生成颜色
function generateColor(index: number): AgentColorName {
  if (index < AGENT_COLORS.length) {
    return AGENT_COLORS[index]!;
  }
  
  // 动态生成更多颜色（HSL 色彩空间）
  const hue = (index * 137.508) % 360;  // 黄金角分布
  return `hsl(${hue}, 70%, 50%)` as AgentColorName;
}
```

#### 2. 后端检测缓存

**风险**：`detectAndGetBackend()` 内部缓存，如果环境变化（如用户安装 it2）不会自动更新。

**当前行为**：
```typescript
// detectAndGetBackend() caches internally — no need for a second cache here
async function getBackend(): Promise<PaneBackend> {
  return (await detectAndGetBackend()).backend;
}
```

**改进建议**：
```typescript
// 添加缓存失效机制
let backendCache: PaneBackend | null = null;
let backendCacheTime = 0;
const CACHE_TTL_MS = 60000;  // 1 分钟

async function getBackend(): Promise<PaneBackend> {
  const now = Date.now();
  if (backendCache && now - backendCacheTime < CACHE_TTL_MS) {
    return backendCache;
  }
  
  backendCache = (await detectAndGetBackend()).backend;
  backendCacheTime = now;
  return backendCache;
}

// 提供手动刷新
export function clearBackendCache(): void {
  backendCache = null;
  backendCacheTime = 0;
}
```

#### 3. 颜色分配持久化

**风险**：颜色分配存储在内存中，进程重启后可能重新分配不同颜色。

**当前行为**：
```typescript
const teammateColorAssignments = new Map<string, AgentColorName>();
```

**改进建议**：
```typescript
// 持久化到团队文件
export async function persistColorAssignments(teamName: string): Promise<void> {
  const teamFile = await readTeamFileAsync(teamName);
  if (!teamFile) return;
  
  teamFile.colorAssignments = Object.fromEntries(teammateColorAssignments);
  await writeTeamFileAsync(teamName, teamFile);
}

export async function restoreColorAssignments(teamName: string): Promise<void> {
  const teamFile = await readTeamFileAsync(teamName);
  if (!teamFile?.colorAssignments) return;
  
  for (const [id, color] of Object.entries(teamFile.colorAssignments)) {
    teammateColorAssignments.set(id, color as AgentColorName);
  }
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| 重复分配同一 teammate | 返回已缓存的颜色 |
| 调色板为空 | 运行时错误（理论上不会发生） |
| 后端检测失败 | 委托给 `detectAndGetBackend()` 处理 |
| pane 创建失败 | 后端返回错误，上层处理 |
| `clearTeammateColors()` 后 | 颜色索引重置为 0，可重新分配 |

### 改进建议

#### 1. 颜色主题支持

```typescript
// 支持不同的颜色主题
export type ColorTheme = 'default' | 'pastel' | 'highContrast';

const THEMES: Record<ColorTheme, AgentColorName[]> = {
  default: AGENT_COLORS,
  pastel: ['#FFB3BA', '#BAFFC9', '#BAE1FF', '#FFFFBA', '#FFDFBA'],
  highContrast: ['#FF0000', '#00FF00', '#0000FF', '#FFFF00', '#FF00FF'],
};

export function setColorTheme(theme: ColorTheme): void {
  currentTheme = theme;
  colorIndex = 0;  // 重置索引
}
```

#### 2. 颜色预览

```typescript
// 预览颜色分配而不实际分配
export function previewTeammateColor(teammateId: string): AgentColorName | undefined {
  // 检查已分配
  if (teammateColorAssignments.has(teammateId)) {
    return teammateColorAssignments.get(teammateId);
  }
  
  // 计算将要分配的颜色
  const colors = THEMES[currentTheme];
  const nextIndex = colorIndex % colors.length;
  return colors[nextIndex];
}
```

#### 3. 后端健康检查

```typescript
// 检查后端是否可用
export async function checkBackendHealth(): Promise<{
  available: boolean;
  backend: string;
  issues: string[];
}> {
  const detection = await detectAndGetBackend();
  const backend = detection.backend;
  
  const issues: string[] = [];
  
  if (!await backend.isAvailable()) {
    issues.push(`${backend.displayName} is not available`);
  }
  
  if (detection.needsIt2Setup) {
    issues.push('iTerm2 detected but it2 CLI not installed');
  }
  
  return {
    available: issues.length === 0,
    backend: backend.displayName,
    issues,
  };
}
```

#### 4. 布局预设

```typescript
// 支持不同的布局预设
export type LayoutPreset = 'horizontal' | 'vertical' | 'grid' | 'leaderFocus';

export async function applyLayoutPreset(
  preset: LayoutPreset,
  teammateCount: number
): Promise<void> {
  const backend = await getBackend();
  
  switch (preset) {
    case 'leaderFocus':
      // 领导占 50%，其余 teammate 平分剩余空间
      await backend.rebalancePanes('swarm-view', true);
      break;
    case 'grid':
      // 网格布局
      // ...
      break;
    // ...
  }
}
```

### 测试建议

1. **单元测试**：
   - 颜色分配逻辑
   - 颜色缓存行为
   - 清除功能

2. **集成测试**：
   - 与后端注册表的集成
   - 与 tmux/iTerm2 后端的集成

3. **视觉测试**：
   - 颜色对比度检查
   - 可访问性验证

4. **边界测试**：
   - 大量 teammate（超过调色板大小）
   - 快速创建/销毁
   - 后端切换
